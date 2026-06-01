# Platform Infrastructure Design
**Date:** 2026-06-01
**Project:** `platform-infra`
**Status:** Approved

---

## Overview

A multi-region GKE platform across `us-central1` and `europe-west1`, managed entirely with Terragrunt + Terraform. Built as a learning/portfolio project with enterprise-grade structure, naming conventions, and security practices. Cost is controlled by using spot VMs and spinning clusters up/down per session.

The primary learning goals are:
- Terragrunt (layered infra, dependency graph, DRY backends)
- GKE Fleet + Config Sync (multi-cluster GitOps)
- Workload Identity Federation (keyless auth for pods and CI/CD)
- Private GKE clusters
- Policy Controller + Binary Authorization
- GCP bootstrap pattern (state bucket persistence across destroys)

Initial workload: `my-go-gopher`. Future phase: multi-tier app with Kafka and Redis.

---

## GCP Project Structure

Two GCP projects with distinct lifecycles:

| Project ID | Purpose | Lifecycle |
|---|---|---|
| `plt-bootstrap` | Terraform state (GCS), Terraform SA, CI/CD Workload Identity | **Permanent** — never destroyed |
| `mrd-dev` | VPC, GKE clusters, Fleet, all platform resources | **Ephemeral** — spin up to work, destroy when done |

The separation exists for two reasons:
1. `plt-bootstrap` must survive `terraform destroy` on `mrd-dev` — it holds the state files for everything
2. The Terraform service account cannot live inside the project it manages (circular destroy problem)

GCP project IDs must be globally unique. Append a short random suffix at creation time (e.g., `plt-bootstrap-a3f2`, `mrd-dev-a3f2`). The suffix is set once and never changes.

---

## Repository Structure

```
platform-infra/
├── terragrunt.hcl                        # Root: generates GCS backend + GCP provider for all units
├── bootstrap/
│   └── terragrunt.hcl                    # Creates plt-bootstrap project, state bucket, Terraform SA
├── platform/
│   ├── iam/
│   │   └── terragrunt.hcl                # Service accounts, IAM bindings for mrd-dev
│   └── networking/
│       └── terragrunt.hcl                # VPC, subnets, Cloud NAT, firewall rules
├── clusters/
│   ├── us-central1/
│   │   └── terragrunt.hcl                # Private GKE cluster, Fleet enrollment, node pools
│   └── europe-west1/
│       └── terragrunt.hcl                # Private GKE cluster, Fleet enrollment, node pools
├── addons/
│   ├── config-sync/
│   │   └── terragrunt.hcl                # Fleet Config Sync — gitops/ repo as source of truth
│   ├── policy-controller/
│   │   └── terragrunt.hcl                # OPA-based policy enforcement via Fleet
│   └── binary-authorization/
│       └── terragrunt.hcl                # Container image signing policy
├── modules/                              # Reusable local Terraform modules
│   ├── gke-cluster/                      # Private GKE cluster with Workload Identity
│   ├── vpc/                              # VPC + subnets + NAT
│   └── workload-identity/                # Workload Identity binding (SA ↔ KSA)
├── .github/
│   ├── workflows/
│   │   ├── tf-plan.yml                   # terragrunt run-all plan on PR (keyless OIDC auth)
│   │   └── tf-apply.yml                  # terragrunt run-all apply on merge to main
│   ├── CODEOWNERS
│   └── pull_request_template.md
└── docs/
    └── superpowers/specs/
```

### How Terragrunt wires this together

The root `terragrunt.hcl` generates two blocks automatically for every unit:
1. **Remote state backend** — GCS bucket path derived from the unit's directory path (no repeated `backend` blocks)
2. **GCP provider** — project ID and region injected from root inputs

Each unit's `terragrunt.hcl` declares `dependency` blocks pointing to units it needs outputs from. Example: `clusters/us-central1` declares a dependency on `platform/networking` to get the subnet self-link. Terragrunt resolves the full dependency graph and can apply all units in correct order with `terragrunt run-all apply`.

---

## Naming Conventions

**Pattern:** `{prefix}-{resource-type}-{descriptor}-{region-short}`

Region abbreviations: `usc1` = `us-central1`, `euw1` = `europe-west1`

| Resource | Name |
|---|---|
| Bootstrap GCP project | `plt-bootstrap-{suffix}` |
| Platform GCP project | `mrd-dev-{suffix}` |
| Terraform state bucket | `plt-tfstate-{suffix}` |
| Terraform service account | `plt-sa-terraform` |
| VPC | `mrd-vpc-global` |
| Subnet (us-central1) | `mrd-subnet-gke-usc1` |
| Subnet (europe-west1) | `mrd-subnet-gke-euw1` |
| Cloud NAT (us-central1) | `mrd-nat-usc1` |
| Cloud NAT (europe-west1) | `mrd-nat-euw1` |
| GKE cluster (us-central1) | `mrd-gke-usc1` |
| GKE cluster (europe-west1) | `mrd-gke-euw1` |
| GKE node SA | `mrd-sa-gke-nodes` |
| Fleet membership | `mrd-gke-usc1`, `mrd-gke-euw1` |
| Billing budget | `mrd-budget-dev` |

All names: lowercase, hyphens only, no underscores (GCP convention). Max 63 characters.

---

## Bootstrap Process

The classic Terraform chicken-and-egg problem: Terraform needs a GCS bucket to store state, but Terraform creates the bucket.

**One-time manual steps (not in Terraform):**
1. `gcloud projects create plt-bootstrap-{suffix}`
2. Link billing account via console or `gcloud beta billing projects link`
3. Enable the `cloudresourcemanager` and `iam` APIs manually

**Then Terraform takes over:**
1. Run `bootstrap/` with **local state** — creates GCS state bucket and Terraform SA
2. Migrate local state to the newly created GCS bucket (`terraform init -migrate-state`)
3. All subsequent layers use the GCS backend automatically via Terragrunt root config

The `mrd-dev` project itself is created by the `bootstrap/` layer, so `terraform destroy` on higher layers never touches it. Only an explicit destroy of `bootstrap/` would remove it.

---

## Platform Layer: Networking

A single global VPC with regional subnets — simpler than per-region VPCs for a two-region setup and sufficient for GKE private clusters.

**VPC:** `mrd-vpc-global`
- Two subnets, one per region, with secondary IP ranges for GKE pods and services (VPC-native clusters require this)
- Private Google Access enabled on all subnets (nodes can reach Google APIs without public IPs)
- Cloud NAT per region for outbound internet access from private nodes
- Firewall: default-deny ingress, allow only what's explicitly needed

**Why VPC-native (alias IP)?** Required for private clusters. It also allows GKE to allocate pod IPs directly from the VPC, making them routable within the network without NAT.

---

## Clusters Layer: GKE

Two private Standard GKE clusters, one per region. Standard (not Autopilot) is used because it exposes more configuration knobs — better for learning.

**Both clusters share this config:**
- Private cluster: nodes have no public IP, control plane access restricted to authorized networks
- Workload Identity enabled at cluster level
- Release channel: `REGULAR` (stable, monthly updates)
- Fleet enrollment at creation time
- Two node pools:
  - `system`: 1× `e2-standard-2`, on-demand (system pods need stability)
  - `apps`: 0–2× `e2-standard-2`, **spot VMs**, autoscaler enabled (scale to 0 when idle)

**Workload Identity:** Each application service account gets a Kubernetes Service Account (KSA) bound to a GCP Service Account (GSA) via IAM. Pods assume GCP identities without SA key files. This is the keyless authentication pattern for workloads.

**Fleet enrollment:** Both clusters register as Fleet members under `mrd-dev`. The Fleet host project is the same project as the clusters — `plt-bootstrap` stays out of it entirely. This unlocks Fleet-level features: Config Sync, Policy Controller, and a unified multi-cluster view in the GCP console.

---

## Addons Layer

Managed via Terraform after clusters exist. All addons are Fleet-level features — configured once, applied to all member clusters.

### Config Sync
Replaces standalone ArgoCD for platform config delivery. Configured at the Fleet level to point at the existing `gitops/` repo. Both clusters automatically sync from the same Git source. This is the key difference from your current setup — you configure sync once at the Fleet level rather than installing ArgoCD into each cluster separately.

### Policy Controller
OPA-based policy enforcement, managed by Fleet. Enforces constraints across all member clusters. Initial policies to write:
- Require all containers to have resource `requests` and `limits`
- Disallow `latest` image tags
- Require images to come from Artifact Registry only (once Binary Authorization is set up)

### Binary Authorization
Requires container images to be signed before GKE admits them. Enforced at the cluster level. For the learning phase, start with `WARN` mode (logs violations but doesn't block) then graduate to `ENFORCED` once you have signing set up for `my-go-gopher`.

---

## CI/CD: GitHub Actions with Keyless Auth

No service account keys anywhere. GitHub Actions authenticates to GCP using **OIDC → Workload Identity Federation**:

1. GitHub Actions job requests a short-lived OIDC token from GitHub
2. GCP Workload Identity Pool exchanges it for a short-lived GCP access token
3. Terraform runs with that token — no JSON key file, no secrets stored in GitHub

**Workflows:**
- `tf-plan.yml`: triggers on PR, runs `terragrunt run-all plan` across all units, posts plan output as PR comment
- `tf-apply.yml`: triggers on merge to `main`, runs `terragrunt run-all apply` in dependency order

The Workload Identity Pool and Provider for GitHub Actions is itself created in the `bootstrap/` layer — so it bootstraps the CI/CD auth as part of the bootstrap process.

---

## Cost Controls

| Control | Detail |
|---|---|
| Spot VMs for app node pool | 70–90% cheaper than on-demand |
| `e2-standard-2` machine type | Smallest that runs GKE system pods comfortably |
| App node pool scales to 0 | No app workload = no app node cost |
| Daily destroy workflow | Clusters only exist during active sessions |
| Billing budget alert | **$20/month** — catches forgotten running clusters |
| No Cloud SQL / Memorystore | `my-go-gopher` doesn't need managed databases |

Approximate cost when running: ~$3–5/hour for both clusters. A full 8-hour session ≈ $25–40. With daily destroy, monthly spend depends entirely on how many sessions you run.

The $20 budget alert is intentionally set below a full day of running — it will fire if you forget to destroy overnight. Treat it as your "did you leave the stove on?" check.

---

## New Skills Introduced

| Skill | Where you'll use it |
|---|---|
| **Terragrunt** | Root config, `dependency` blocks, `run-all` commands |
| **GCP bootstrap pattern** | `bootstrap/` layer, local → remote state migration |
| **Private GKE clusters** | `clusters/` layer, VPC-native networking |
| **GKE Fleet** | Enrollment in `clusters/`, unified view across regions |
| **Config Sync** | `addons/config-sync/`, Fleet-level GitOps |
| **Workload Identity Federation** | Pod auth in `modules/workload-identity/`, CI/CD auth in `bootstrap/` |
| **Policy Controller** | `addons/policy-controller/`, writing Constraint Templates |
| **Binary Authorization** | `addons/binary-authorization/`, image signing policy |
| **GitHub Actions OIDC → GCP** | `tf-plan.yml` + `tf-apply.yml` workflows |

---

## GitHub Setup

The PAT provided does not have repository creation permission (fine-grained PATs require explicit `Administration: write` scope for repo creation). Steps to set up:

1. Create `platform-infra` repo manually at github.com/esketch545
2. Set visibility to **private**
3. Push the local repo: `git remote add origin https://github.com/esketch545/platform-infra.git`
4. For GitHub Actions to work, generate a new PAT or update the existing one with `Actions: write` and `Contents: write` scopes, or use the default `GITHUB_TOKEN`

---

## Future Phases

- **Multi-tier workload:** Replace `my-go-gopher` with a frontend + backend + Redis + Kafka stack to demonstrate Fleet-level service routing and cross-cluster communication
- **Terragrunt module registry:** Extract modules to a separate `terraform-modules` repo and version them
- **Cloud Armor:** WAF + DDoS protection in front of the GKE ingress
- **Multi-region ingress:** Global HTTP(S) Load Balancer routing traffic to the nearest healthy cluster
- **Production project:** Add `mrd-prod` alongside `mrd-dev` to practice environment promotion
