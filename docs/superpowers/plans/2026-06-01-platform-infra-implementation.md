# Platform Infrastructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a multi-region GKE platform across `us-central1` and `europe-west1` using Terragrunt + Terraform, with GKE Fleet, Config Sync, Workload Identity Federation, Policy Controller, and Binary Authorization.

**Architecture:** Terragrunt layered structure (bootstrap → networking/iam → clusters → addons), two GCP projects (`plt-bootstrap` permanent, `mrd-dev` ephemeral), all new skills are hands-on — you build it, you learn it.

**Tech Stack:** Terragrunt, Terraform, GCP (GKE, Fleet, Config Sync, Workload Identity, Binary Authorization, Policy Controller), GitHub Actions (OIDC keyless auth)

> **Learning project note:** Tasks tell you WHAT to build and WHY, with verification commands and doc links. They do not give you HCL to paste in — that's intentional. Look things up, make mistakes, figure it out.

---

## File Map

Files you will create across all phases:

```
platform-infra/
├── terragrunt.hcl                          # Phase 2 — root backend/provider generation
├── .pre-commit-config.yaml                 # Phase 0
├── .gitignore                              # Phase 0
├── .github/
│   ├── CODEOWNERS                          # Phase 0
│   ├── pull_request_template.md            # Phase 0
│   └── workflows/
│       ├── tf-plan.yml                     # Phase 8
│       └── tf-apply.yml                    # Phase 8
├── bootstrap/
│   ├── terragrunt.hcl                      # Phase 1
│   ├── main.tf                             # Phase 1
│   ├── variables.tf                        # Phase 1
│   └── outputs.tf                          # Phase 1
├── platform/
│   ├── networking/
│   │   ├── terragrunt.hcl                  # Phase 3
│   │   ├── main.tf                         # Phase 3
│   │   ├── variables.tf                    # Phase 3
│   │   └── outputs.tf                      # Phase 3
│   └── iam/
│       ├── terragrunt.hcl                  # Phase 4
│       ├── main.tf                         # Phase 4
│       ├── variables.tf                    # Phase 4
│       └── outputs.tf                      # Phase 4
├── clusters/
│   ├── us-central1/
│   │   └── terragrunt.hcl                  # Phase 5
│   └── europe-west1/
│       └── terragrunt.hcl                  # Phase 5
├── addons/
│   ├── config-sync/
│   │   ├── terragrunt.hcl                  # Phase 7
│   │   ├── main.tf                         # Phase 7
│   │   └── variables.tf                    # Phase 7
│   ├── policy-controller/
│   │   ├── terragrunt.hcl                  # Phase 7
│   │   ├── main.tf                         # Phase 7
│   │   ├── variables.tf                    # Phase 7
│   │   └── constraints/
│   │       ├── require-resource-limits.yaml   # Phase 7
│   │       ├── disallow-latest-tag.yaml        # Phase 7
│   │       └── require-artifact-registry.yaml  # Phase 7
│   └── binary-authorization/
│       ├── terragrunt.hcl                  # Phase 7
│       ├── main.tf                         # Phase 7
│       └── variables.tf                    # Phase 7
└── modules/
    ├── gke-cluster/
    │   ├── main.tf                         # Phase 5
    │   ├── variables.tf                    # Phase 5
    │   └── outputs.tf                      # Phase 5
    ├── vpc/
    │   ├── main.tf                         # Phase 3
    │   ├── variables.tf                    # Phase 3
    │   └── outputs.tf                      # Phase 3
    └── workload-identity/
        ├── main.tf                         # Phase 6
        ├── variables.tf                    # Phase 6
        └── outputs.tf                      # Phase 6
```

---

## Phase 0: Repository & Local Toolchain

### Task 1: Set up local toolchain

**Read first:**
- [Terragrunt install](https://terragrunt.gruntwork.io/docs/getting-started/install/)
- [tflint](https://github.com/terraform-linters/tflint) — lints Terraform for errors and bad practices
- [tfsec](https://github.com/aquasecurity/tfsec) — static security analysis for Terraform
- [pre-commit](https://pre-commit.com/) — runs checks automatically before every commit

**Why:** You want these running locally before CI catches them. Faster feedback loop.

- [ ] Install Terragrunt: `winget install --id Gruntwork.Terragrunt` or download from GitHub releases
- [ ] Install tflint: `winget install --id TerraformLinters.tflint`
- [ ] Install tfsec: `winget install --id Aquasecurity.tfsec`
- [ ] Install pre-commit: `pip install pre-commit` or `winget install Python.Python.3` first if needed
- [ ] Verify all installs:
  ```
  terragrunt --version
  tflint --version
  tfsec --version
  pre-commit --version
  ```
  Expected: each prints a version string without error
- [ ] Create `.pre-commit-config.yaml` in repo root. Configure these hooks: `terraform_fmt`, `terraform_validate`, `tflint`, `tfsec`. Reference: [pre-commit-terraform hooks](https://github.com/antonbabenko/pre-commit-terraform)
- [ ] Create `.gitignore`. Include: `.terraform/`, `*.tfstate`, `*.tfstate.backup`, `.terragrunt-cache/`, `*.tfvars` (you'll use root inputs instead), `override.tf`
- [ ] Run `pre-commit install` in repo root — this wires the hooks into git
- [ ] Commit:
  ```
  git add .pre-commit-config.yaml .gitignore
  git commit -m "chore: add pre-commit hooks and gitignore"
  ```

---

### Task 2: GitHub repo setup and branch protection

**Why:** Enterprise repos enforce code review via branch protection. You should practice working against this constraint from day one — all your Terraform changes will go through PRs.

- [ ] Create `platform-infra` repo at github.com/esketch545 — set to **private**
- [ ] Add remote and push:
  ```
  git remote add origin https://github.com/esketch545/platform-infra.git
  git push -u origin master
  ```
- [ ] Create `CODEOWNERS` at `.github/CODEOWNERS`. Set `* @esketch545` — this requires your review on all PRs
- [ ] Create `.github/pull_request_template.md`. Include sections: `## What`, `## Why`, `## Testing done`, `## Checklist` with items: tflint passes, tfsec passes, plan reviewed
- [ ] In GitHub repo settings → Branches → Add rule for `main`:
  - Require pull request before merging
  - Require 1 approval (you can approve your own PRs as sole owner)
  - Require status checks (add these once CI is set up in Phase 8)
- [ ] Rename default branch from `master` to `main`:
  ```
  git branch -m master main
  git push -u origin main
  ```
- [ ] Commit and push CODEOWNERS + PR template:
  ```
  git add .github/
  git commit -m "chore: add CODEOWNERS and PR template"
  git push
  ```

---

## Phase 1: Bootstrap

### Task 3: Manual GCP prerequisites

**Read first:**
- [gcloud CLI install](https://cloud.google.com/sdk/docs/install)
- [GCP project creation](https://cloud.google.com/resource-manager/docs/creating-managing-projects)

**Why:** Terraform can't create its own state bucket — something has to exist first. These manual steps are the only things in this entire project you'll do outside of Terraform/Terragrunt.

- [ ] Install and initialize gcloud:
  ```
  gcloud init
  gcloud auth login
  gcloud auth application-default login
  ```
- [ ] Pick a 4-char random suffix (e.g., run `openssl rand -hex 2` or just make one up). Write it down — you'll use it in every resource name
- [ ] Create the bootstrap GCP project (replace `{suffix}` with yours):
  ```
  gcloud projects create plt-bootstrap-{suffix} --name="Platform Bootstrap"
  ```
- [ ] List your billing accounts:
  ```
  gcloud beta billing accounts list
  ```
- [ ] Link billing to the bootstrap project:
  ```
  gcloud beta billing projects link plt-bootstrap-{suffix} \
    --billing-account=YOUR_BILLING_ACCOUNT_ID
  ```
- [ ] Enable the two APIs Terraform needs to bootstrap itself:
  ```
  gcloud services enable cloudresourcemanager.googleapis.com \
    --project=plt-bootstrap-{suffix}
  gcloud services enable iam.googleapis.com \
    --project=plt-bootstrap-{suffix}
  ```
- [ ] Verify:
  ```
  gcloud projects describe plt-bootstrap-{suffix}
  gcloud services list --project=plt-bootstrap-{suffix} --enabled
  ```
  Expected: project shows `lifecycleState: ACTIVE`, both APIs listed as enabled

---

### Task 4: Write bootstrap Terraform

**Read first:**
- [google_project resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project)
- [google_storage_bucket resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/storage_bucket)
- [google_service_account resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_service_account)
- [google_project_service resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_service) (for enabling APIs)
- [google_iam_workload_identity_pool](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool) (for GitHub Actions auth)
- [google_billing_budget](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/billing_budget)

**Why:** The bootstrap layer is run once with local state, creates the permanent infrastructure (state bucket, Terraform SA), and creates the `mrd-dev` project that all other layers will use.

**Files to create:** `bootstrap/main.tf`, `bootstrap/variables.tf`, `bootstrap/outputs.tf`, `bootstrap/terragrunt.hcl`

- [ ] Create `bootstrap/terragrunt.hcl`. For the bootstrap unit only, you want a **local backend** initially (not GCS — the bucket doesn't exist yet). Set `generate` block for the Terraform backend with `path = "backend.tf"` and `if_exists = "overwrite"`, but point it to `local` backend type. You'll change this after the bucket exists.
- [ ] Create `bootstrap/variables.tf`. Variables needed: `project_id` (plt-bootstrap-{suffix}), `billing_account`, `suffix`, `region` (default `us-central1`), `github_repo` (for Workload Identity — format: `esketch545/platform-infra`)
- [ ] Create `bootstrap/main.tf`. This file should create:
  1. **GCS state bucket** — name `plt-tfstate-{suffix}`, region `us-central1`, versioning enabled, uniform bucket-level access, `force_destroy = false`
  2. **Terraform service account** — `plt-sa-terraform` in `plt-bootstrap-{suffix}`, used by CI/CD to run terragrunt
  3. **IAM bindings** for the Terraform SA — needs `roles/owner` on `mrd-dev` project (broad for learning; you'd tighten this in prod)
  4. **mrd-dev GCP project** — name `mrd-dev-{suffix}`, link to same billing account
  5. **APIs to enable on mrd-dev** — minimum set: `container.googleapis.com` (GKE), `compute.googleapis.com`, `iam.googleapis.com`, `gkehub.googleapis.com` (Fleet), `anthos.googleapis.com`, `mesh.googleapis.com` (Config Sync), `binaryauthorization.googleapis.com`
  6. **Workload Identity Pool + Provider** for GitHub Actions — pool name `plt-wif-pool`, provider name `plt-wif-github`, attribute mapping for GitHub OIDC claims (`attribute.repository`, `assertion.repository`)
  7. **IAM binding** — grant the Terraform SA `roles/iam.workloadIdentityUser` to the GitHub Actions principal (`principalSet://iam.googleapis.com/...`)
  8. **Billing budget** — `mrd-budget-dev`, threshold $20, email notification to your address
- [ ] Create `bootstrap/outputs.tf`. Output: `state_bucket_name`, `terraform_sa_email`, `mrd_project_id`, `workload_identity_provider` (the full provider resource name used in GitHub Actions)
- [ ] Commit:
  ```
  git add bootstrap/
  git commit -m "feat: add bootstrap Terraform layer"
  ```

---

### Task 5: Apply bootstrap and migrate state

**Why:** This is the chicken-and-egg resolution. You'll apply with local state, then move that state into the bucket you just created. After migration, the bootstrap layer is managed like every other layer.

- [ ] Change into bootstrap directory and initialize Terraform (not Terragrunt yet):
  ```
  cd bootstrap
  terraform init
  ```
- [ ] Create a `terraform.tfvars` file locally (do NOT commit this — it contains your billing account ID). Set all variables.
- [ ] Run plan, review output carefully — make sure you understand every resource being created:
  ```
  terraform plan -var-file=terraform.tfvars
  ```
- [ ] Apply:
  ```
  terraform apply -var-file=terraform.tfvars
  ```
- [ ] Verify the bucket exists:
  ```
  gsutil ls gs://plt-tfstate-{suffix}
  ```
- [ ] Verify the Terraform SA was created:
  ```
  gcloud iam service-accounts list --project=plt-bootstrap-{suffix}
  ```
- [ ] Verify mrd-dev project was created:
  ```
  gcloud projects describe mrd-dev-{suffix}
  ```
- [ ] Now update `bootstrap/terragrunt.hcl` — change the backend from `local` to `gcs`, pointing at the bucket you just created, with key `bootstrap/terraform.tfstate`
- [ ] Migrate state:
  ```
  terraform init -migrate-state
  ```
  When prompted "Do you want to copy existing state to the new backend?" — answer yes
- [ ] Verify state is now in GCS:
  ```
  gsutil cat gs://plt-tfstate-{suffix}/bootstrap/terraform.tfstate | head -20
  ```
  Expected: JSON with your resource state
- [ ] Delete the local `terraform.tfstate` files (they're now in GCS):
  ```
  rm terraform.tfstate terraform.tfstate.backup
  ```
- [ ] Commit the updated terragrunt.hcl:
  ```
  git add bootstrap/terragrunt.hcl
  git commit -m "feat: migrate bootstrap state to GCS backend"
  ```

---

## Phase 2: Root Terragrunt Config

### Task 6: Write root terragrunt.hcl

**Read first:**
- [Terragrunt DRY backends](https://terragrunt.gruntwork.io/docs/features/keep-your-remote-state-configuration-dry/)
- [Terragrunt generate blocks](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#generate)
- [Terragrunt inputs](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#inputs)
- [Terragrunt read_terragrunt_config](https://terragrunt.gruntwork.io/docs/reference/built-in-functions/#read_terragrunt_config)

**Why:** The root `terragrunt.hcl` is the DRY core of this project. Every unit beneath it automatically gets a GCS backend (keyed by the unit's directory path) and a GCP provider — you never write these again. This is Terragrunt's primary value proposition.

**File to create:** `terragrunt.hcl` (repo root)

- [ ] Create `terragrunt.hcl` at repo root. It needs three things:
  1. **`remote_state` block** — configure GCS backend. Use `path_relative_to_include()` as the GCS object prefix so each unit gets a unique state path (e.g., `platform/networking/terraform.tfstate`). Bucket name comes from a local variable or `run_cmd` reading bootstrap outputs.
  2. **`generate "provider"` block** — generates `provider.tf` for every unit with the `google` and `google-beta` providers. Project and region come from `inputs`.
  3. **`inputs` block** — set `project_id = "mrd-dev-{suffix}"` and `region` defaults. Units that need a different project (like bootstrap itself) override this locally.
- [ ] Update `bootstrap/terragrunt.hcl` to `include` the root config using `include "root"` block. Override the backend to keep pointing at the bootstrap GCS path (not the auto-generated path from root).
- [ ] Test that Terragrunt can read the root config from the bootstrap unit:
  ```
  cd bootstrap
  terragrunt validate
  ```
  Expected: no errors
- [ ] Commit:
  ```
  git add terragrunt.hcl bootstrap/terragrunt.hcl
  git commit -m "feat: add root terragrunt config with DRY backend and provider generation"
  ```

---

## Phase 3: Networking

### Task 7: Write modules/vpc

**Read first:**
- [google_compute_network](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_network)
- [google_compute_subnetwork](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_subnetwork) — pay special attention to `secondary_ip_range` blocks, these are required for VPC-native GKE
- [google_compute_router + google_compute_router_nat](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_router_nat)
- [GKE VPC-native clusters docs](https://cloud.google.com/kubernetes-engine/docs/concepts/alias-ips)

**Why:** This is a reusable module — both regions share the same VPC structure. The secondary IP ranges are the tricky part: GKE needs separate ranges for pods and services that don't overlap with your node subnet.

**Files to create:** `modules/vpc/main.tf`, `modules/vpc/variables.tf`, `modules/vpc/outputs.tf`

- [ ] Design your IP address ranges before writing any code. Sketch this out:
  - `mrd-subnet-gke-usc1`: nodes `10.0.0.0/24`, pods `10.10.0.0/20`, services `10.20.0.0/24`
  - `mrd-subnet-gke-euw1`: nodes `10.1.0.0/24`, pods `10.11.0.0/20`, services `10.21.0.0/24`
  - Make sure none of these overlap
- [ ] Create `modules/vpc/variables.tf` — inputs: `project_id`, `network_name`, `subnets` (list of objects with region, name, cidr, pod_cidr, service_cidr), `nat_regions` (list)
- [ ] Create `modules/vpc/main.tf` — resources:
  1. `google_compute_network` — custom subnet mode, NOT auto-create-subnetworks
  2. `google_compute_subnetwork` — one per entry in `var.subnets`, each with two `secondary_ip_range` blocks (pods + services), `private_ip_google_access = true`
  3. `google_compute_router` — one per region in `var.nat_regions`
  4. `google_compute_router_nat` — attached to each router, `NAT_ALL_SUBNETWORK_IP_RANGES`
- [ ] Create `modules/vpc/outputs.tf` — output: `network_self_link`, `subnet_self_links` (map of region → self_link), `pod_cidr_names` (map of region → secondary range name), `service_cidr_names` (same)
- [ ] Commit:
  ```
  git add modules/vpc/
  git commit -m "feat: add vpc module with VPC-native subnets and Cloud NAT"
  ```

---

### Task 8: Write and apply platform/networking

**Read first:**
- [Terragrunt dependency blocks](https://terragrunt.gruntwork.io/docs/reference/config-blocks-and-attributes/#dependency)

**Files to create:** `platform/networking/terragrunt.hcl`, `platform/networking/main.tf`, `platform/networking/variables.tf`, `platform/networking/outputs.tf`

- [ ] Create `platform/networking/terragrunt.hcl`. Include root config. No `dependency` blocks yet — networking has no upstream dependencies. Pass inputs matching your vpc module's variables.
- [ ] Create `platform/networking/main.tf` — call the `modules/vpc` module, passing the subnet definitions for both regions
- [ ] Create `platform/networking/outputs.tf` — re-export the vpc module outputs so `clusters/` can read them via dependency blocks
- [ ] Run plan from the networking directory:
  ```
  cd platform/networking
  terragrunt plan
  ```
  Expected: plan shows VPC, 2 subnets, 2 routers, 2 NATs — no errors
- [ ] Apply:
  ```
  terragrunt apply
  ```
- [ ] Verify in GCP console or via gcloud:
  ```
  gcloud compute networks list --project=mrd-dev-{suffix}
  gcloud compute networks subnets list --project=mrd-dev-{suffix}
  ```
  Expected: `mrd-vpc-global`, `mrd-subnet-gke-usc1`, `mrd-subnet-gke-euw1` all present
- [ ] Commit:
  ```
  git add platform/networking/
  git commit -m "feat: add platform networking layer (VPC, subnets, NAT)"
  ```

---

## Phase 4: IAM

### Task 9: Write and apply platform/iam

**Read first:**
- [google_service_account](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_service_account)
- [GKE node SA recommended roles](https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster#use_least_privilege_sa)
- [Workload Identity overview](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) — read this fully before writing any code

**Why:** The GKE node service account should have minimal permissions (not the Compute Engine default SA). You're also pre-creating the GCP service account that `my-go-gopher` will use via Workload Identity — this is the GSA (GCP Service Account) side of the WI binding.

**Files to create:** `platform/iam/terragrunt.hcl`, `platform/iam/main.tf`, `platform/iam/variables.tf`, `platform/iam/outputs.tf`

- [ ] Create `platform/iam/main.tf`. Resources:
  1. `google_service_account` — `mrd-sa-gke-nodes` — used by GKE node VMs
  2. IAM bindings for `mrd-sa-gke-nodes`: `roles/logging.logWriter`, `roles/monitoring.metricWriter`, `roles/monitoring.viewer`, `roles/stackdriver.resourceMetadata.writer`, `roles/artifactregistry.reader`
  3. `google_service_account` — `mrd-sa-my-go-gopher` — the GSA for your app (you'll bind it to a KSA in Phase 6)
- [ ] Create `platform/iam/outputs.tf` — output: `gke_node_sa_email`, `my_go_gopher_sa_email`
- [ ] Create `platform/iam/terragrunt.hcl` — include root, no dependencies
- [ ] Apply:
  ```
  cd platform/iam
  terragrunt apply
  ```
- [ ] Verify:
  ```
  gcloud iam service-accounts list --project=mrd-dev-{suffix}
  ```
  Expected: both SAs listed
- [ ] Commit:
  ```
  git add platform/iam/
  git commit -m "feat: add platform IAM layer (GKE node SA, app SA)"
  ```

---

## Phase 5: GKE Clusters

### Task 10: Write modules/gke-cluster

**Read first:**
- [google_container_cluster resource](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_cluster) — this is a long page, read it fully
- [Private cluster configuration](https://cloud.google.com/kubernetes-engine/docs/how-to/private-cluster-create)
- [GKE Fleet registration via Terraform](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/gke_hub_membership)
- [GKE node pools](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/container_node_pool)
- [Spot VMs on GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/spot-vms)

**Why:** This module is instantiated twice (one per region). The private cluster config is the tricky part — you need to understand the difference between `enable_private_nodes` and `enable_private_endpoint`, and why you need authorized networks.

**Files to create:** `modules/gke-cluster/main.tf`, `modules/gke-cluster/variables.tf`, `modules/gke-cluster/outputs.tf`

- [ ] Create `modules/gke-cluster/variables.tf`. Variables: `cluster_name`, `project_id`, `region`, `network_self_link`, `subnet_self_link`, `pod_range_name`, `service_range_name`, `node_sa_email`, `master_authorized_networks_cidr` (your home IP or `0.0.0.0/0` for learning — read why this matters)
- [ ] Create `modules/gke-cluster/main.tf`. Resources:
  1. `google_container_cluster` — remove the default node pool (`remove_default_node_pool = true`, `initial_node_count = 1`), enable private nodes, enable Workload Identity (`workload_pool = "${project_id}.svc.id.goog"`), set release channel to `REGULAR`, enable Fleet registration (`fleet { project = "projects/${project_id}" }`)
  2. `google_container_node_pool` named `system` — `e2-standard-2`, 1 node, on-demand, `service_account = var.node_sa_email`, workload metadata config `GKE_METADATA`
  3. `google_container_node_pool` named `apps` — `e2-standard-2`, spot VMs (`spot = true`), autoscaling min=0 max=2, workload metadata config `GKE_METADATA`
- [ ] Create `modules/gke-cluster/outputs.tf` — output: `cluster_name`, `cluster_id`, `cluster_endpoint`, `cluster_ca_certificate`, `fleet_membership_id`
- [ ] Commit:
  ```
  git add modules/gke-cluster/
  git commit -m "feat: add gke-cluster module with private nodes, Fleet enrollment, spot VMs"
  ```

---

### Task 11: Write cluster units and apply

**Files to create:** `clusters/us-central1/terragrunt.hcl`, `clusters/europe-west1/terragrunt.hcl`

- [ ] Create `clusters/us-central1/terragrunt.hcl`. Include root. Add `dependency` blocks for:
  - `platform/networking` — to get `subnet_self_links["us-central1"]`, `pod_cidr_names`, `service_cidr_names`
  - `platform/iam` — to get `gke_node_sa_email`
  Set inputs using those dependency outputs. Cluster name: `mrd-gke-usc1`, region: `us-central1`.
- [ ] Create `clusters/europe-west1/terragrunt.hcl` — same pattern, different region/name: `mrd-gke-euw1`, `europe-west1`
- [ ] Run plan for both (from repo root):
  ```
  terragrunt run-all plan --terragrunt-include-dir clusters
  ```
  Expected: plan for 2 clusters with no errors. Review the plan carefully — GKE clusters take 8-12 min to create.
- [ ] Apply — this will take ~15-20 minutes total:
  ```
  terragrunt run-all apply --terragrunt-include-dir clusters
  ```
- [ ] Once applied, get kubeconfig for both clusters:
  ```
  gcloud container clusters get-credentials mrd-gke-usc1 \
    --region us-central1 --project mrd-dev-{suffix}
  gcloud container clusters get-credentials mrd-gke-euw1 \
    --region europe-west1 --project mrd-dev-{suffix}
  ```
- [ ] Verify both clusters are running and Fleet members:
  ```
  kubectl config get-contexts
  gcloud container fleet memberships list --project=mrd-dev-{suffix}
  ```
  Expected: two contexts in kubeconfig, two Fleet memberships listed
- [ ] Commit:
  ```
  git add clusters/
  git commit -m "feat: add cluster units for us-central1 and europe-west1"
  ```

---

## Phase 6: Workload Identity for my-go-gopher

### Task 12: Write modules/workload-identity and bind the app SA

**Read first:**
- [Workload Identity how-to](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) — follow this end to end at least once manually before Terraforming it
- [google_service_account_iam_member](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_service_account_iam_member)

**Why:** Workload Identity is the correct way for pods to authenticate to GCP — no SA key JSON files anywhere. The module creates the IAM binding between a KSA (in a Kubernetes namespace) and a GSA (in GCP). The annotation on the KSA is applied via Terraform using the Kubernetes provider.

**Files to create:** `modules/workload-identity/main.tf`, `modules/workload-identity/variables.tf`, `modules/workload-identity/outputs.tf`

- [ ] Create `modules/workload-identity/variables.tf` — inputs: `project_id`, `gsa_email` (the GCP SA), `ksa_name` (Kubernetes SA name), `ksa_namespace`, `cluster_name`, `cluster_location`
- [ ] Create `modules/workload-identity/main.tf` — resources:
  1. `google_service_account_iam_member` — grant the KSA principal (`serviceAccount:{project_id}.svc.id.goog[{ksa_namespace}/{ksa_name}]`) the role `roles/iam.workloadIdentityUser` on the GSA
  2. `kubernetes_service_account` — create the KSA in the specified namespace with annotation `iam.gke.io/gcp-service-account = var.gsa_email`
- [ ] **Note:** the Kubernetes provider needs cluster credentials. Instantiate it using the cluster endpoint and CA cert from the cluster module outputs. This is a good challenge — figure out how to pass kubeconfig from Terragrunt dependency outputs into the provider.
- [ ] Apply the workload-identity module for `my-go-gopher` on both clusters. You can do this from within `platform/iam` by calling the module twice (once per cluster) or create a separate `workload-identity/` unit.
- [ ] Verify the binding works:
  ```
  kubectl describe serviceaccount my-go-gopher -n default
  ```
  Expected: annotation `iam.gke.io/gcp-service-account` pointing at your GSA
- [ ] Commit:
  ```
  git add modules/workload-identity/
  git commit -m "feat: add workload-identity module and bind my-go-gopher SA"
  ```

---

## Phase 7: Addons

### Task 13: Config Sync (Fleet-level GitOps)

**Read first:**
- [Config Sync overview](https://cloud.google.com/anthos-config-management/docs/config-sync/overview)
- [Enable Config Sync via Fleet](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync)
- [google_gke_hub_feature](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/gke_hub_feature) — `configmanagement` feature
- [google_gke_hub_feature_membership](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/gke_hub_feature_membership)

**Why:** This replaces standalone ArgoCD for platform config delivery. You configure sync once at the Fleet level and both clusters receive it. The `gitops/` repo becomes the source of truth for both clusters simultaneously.

**Files to create:** `addons/config-sync/terragrunt.hcl`, `addons/config-sync/main.tf`, `addons/config-sync/variables.tf`

- [ ] Create `addons/config-sync/terragrunt.hcl`. Dependencies: `clusters/us-central1`, `clusters/europe-west1`
- [ ] Create `addons/config-sync/main.tf`:
  1. `google_gke_hub_feature` — enable the `configmanagement` feature on the Fleet
  2. Two `google_gke_hub_feature_membership` resources — one per cluster. Each points the `sync.git` config at your `gitops/` GitHub repo (branch `main`, sync directory `apps/`). Set `policy_controller.enabled = false` (separate resource in next task)
- [ ] Apply:
  ```
  cd addons/config-sync
  terragrunt apply
  ```
- [ ] Verify Config Sync is syncing on both clusters:
  ```
  gcloud alpha anthos config sync repo describe \
    --managed-resources=all --sync-name=root-sync \
    --sync-namespace=config-management-system \
    --project=mrd-dev-{suffix}
  ```
  Or check Fleet status in GCP console → Anthos → Config Management
- [ ] Commit:
  ```
  git add addons/config-sync/
  git commit -m "feat: enable Fleet Config Sync pointing at gitops repo"
  ```

---

### Task 14: Policy Controller

**Read first:**
- [Policy Controller overview](https://cloud.google.com/anthos-config-management/docs/concepts/policy-controller)
- [Constraint Templates](https://cloud.google.com/anthos-config-management/docs/reference/constraint-template-library) — browse the built-in library
- [Writing custom constraints](https://cloud.google.com/anthos-config-management/docs/how-to/write-a-constraint-template)

**Why:** Policy Controller is OPA/Gatekeeper managed by Google via Fleet. You define policies as `Constraint` + `ConstraintTemplate` YAML and they're enforced as admission webhooks on all Fleet clusters. This is how real platform teams enforce standards.

**Files to create:** `addons/policy-controller/terragrunt.hcl`, `addons/policy-controller/main.tf`, `addons/policy-controller/variables.tf`, plus 3 constraint YAML files

- [ ] Enable Policy Controller in `addons/policy-controller/main.tf` via `google_gke_hub_feature_membership` with `policy_controller.enabled = true`, `referential_rules_enabled = true`
- [ ] Apply:
  ```
  cd addons/policy-controller
  terragrunt apply
  ```
- [ ] Wait for Policy Controller to be ready on both clusters (check Fleet status or `kubectl get pods -n gatekeeper-system`)
- [ ] Write `addons/policy-controller/constraints/require-resource-limits.yaml` — use the built-in `K8sRequiredResources` template to require `requests` and `limits` on all containers. Set `enforcementAction: warn` first (so it logs without blocking).
- [ ] Write `addons/policy-controller/constraints/disallow-latest-tag.yaml` — use `K8sDisallowedTags` to block the `latest` image tag
- [ ] Write `addons/policy-controller/constraints/require-artifact-registry.yaml` — use `K8sAllowedRepos` to only allow images from `{region}-docker.pkg.dev` (placeholder for now, activate after Binary Auth phase)
- [ ] Apply the constraint YAMLs directly with kubectl (these are Kubernetes resources, not Terraform):
  ```
  kubectl apply -f addons/policy-controller/constraints/ --context=mrd-gke-usc1
  kubectl apply -f addons/policy-controller/constraints/ --context=mrd-gke-euw1
  ```
- [ ] Test a constraint is working — try deploying a pod with no resource limits:
  ```
  kubectl run test --image=nginx --context=mrd-gke-usc1
  kubectl describe pod test --context=mrd-gke-usc1
  ```
  Expected: pod runs but you see a Policy Controller warning in events (because `enforcementAction: warn`)
- [ ] Commit:
  ```
  git add addons/policy-controller/
  git commit -m "feat: enable Policy Controller with resource limits and image tag constraints"
  ```

---

### Task 15: Binary Authorization

**Read first:**
- [Binary Authorization overview](https://cloud.google.com/binary-authorization/docs/overview)
- [Binary Authorization with GKE](https://cloud.google.com/binary-authorization/docs/setting-up)
- [google_binary_authorization_policy](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/binary_authorization_policy)

**Why:** Binary Authorization is an admission controller that checks whether container images satisfy your policy before allowing them to run. Starting in `WARN` mode lets you see violations without breaking your workload, then you graduate to enforced once you understand the signing flow.

**Files to create:** `addons/binary-authorization/terragrunt.hcl`, `addons/binary-authorization/main.tf`, `addons/binary-authorization/variables.tf`

- [ ] Create `addons/binary-authorization/main.tf`:
  1. `google_binary_authorization_policy` — set `global_policy_evaluation_mode = "ENABLE"`, `default_admission_rule` with `evaluation_mode = "ALWAYS_ALLOW"` and `enforcement_mode = "WARN"` (warn-only to start). This enables BinAuth without blocking anything yet.
- [ ] Apply and verify Binary Authorization is enabled on both clusters:
  ```
  gcloud container clusters describe mrd-gke-usc1 \
    --region us-central1 --project mrd-dev-{suffix} \
    --format="value(binaryAuthorization.evaluationMode)"
  ```
  Expected: `PROJECT_SINGLETON_POLICY_ENFORCE`
- [ ] Test a violation is logged — deploy any image and check Cloud Logging for Binary Authorization audit entries:
  ```
  gcloud logging read 'resource.type="k8s_cluster" protoPayload.methodName="io.k8s.core.v1.pods.create"' \
    --project=mrd-dev-{suffix} --limit=5
  ```
- [ ] Commit:
  ```
  git add addons/binary-authorization/
  git commit -m "feat: enable Binary Authorization in warn mode"
  ```

---

## Phase 8: CI/CD with Keyless Auth

### Task 16: GitHub Actions tf-plan workflow

**Read first:**
- [GitHub Actions OIDC docs](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform)
- [google-github-actions/auth action](https://github.com/google-github-actions/auth) — this is the action that exchanges the GitHub OIDC token for a GCP access token
- [Terragrunt in CI](https://terragrunt.gruntwork.io/docs/features/autoretry/)

**Why:** No SA key files in GitHub secrets. The `auth` action uses the Workload Identity Pool you created in the bootstrap layer to exchange a short-lived GitHub OIDC token for a short-lived GCP access token. Zero long-lived credentials anywhere.

**File to create:** `.github/workflows/tf-plan.yml`

- [ ] Create `.github/workflows/tf-plan.yml`. The workflow should:
  1. Trigger on: `pull_request` targeting `main`
  2. Permissions: `contents: read`, `id-token: write` (required for OIDC), `pull-requests: write` (to post plan comment)
  3. Step: `google-github-actions/auth@v2` — set `workload_identity_provider` to the output value from your bootstrap layer, `service_account` to `plt-sa-terraform@plt-bootstrap-{suffix}.iam.gserviceaccount.com`
  4. Step: Install Terraform and Terragrunt (use `hashicorp/setup-terraform` action + direct Terragrunt binary download)
  5. Step: `terragrunt run-all plan --terragrunt-non-interactive` from repo root
  6. Step: Post plan output as PR comment (use `actions/github-script` or similar)
- [ ] Push to a feature branch, open a PR, verify the workflow runs and posts a plan:
  ```
  git checkout -b feat/test-ci
  echo "# test" >> README.md
  git add README.md && git commit -m "test: trigger CI"
  git push origin feat/test-ci
  ```
  Then open a PR on GitHub and watch the Actions tab
- [ ] Expected: workflow authenticates to GCP without errors, plan runs, output posted as comment
- [ ] Merge the PR after verifying
- [ ] Commit the workflow file itself in a separate PR:
  ```
  git checkout -b feat/ci-plan-workflow
  git add .github/workflows/tf-plan.yml
  git commit -m "feat: add Terraform plan GitHub Actions workflow with OIDC auth"
  git push origin feat/ci-plan-workflow
  ```

---

### Task 17: GitHub Actions tf-apply workflow

**File to create:** `.github/workflows/tf-apply.yml`

- [ ] Create `.github/workflows/tf-apply.yml`. The workflow should:
  1. Trigger on: `push` to `main` (i.e., after PR merge)
  2. Same OIDC auth setup as tf-plan
  3. Step: `terragrunt run-all apply --terragrunt-non-interactive`
  4. Add a concurrency group (`group: terraform-apply, cancel-in-progress: false`) so parallel merges don't run concurrent applies
- [ ] Add a manual `workflow_dispatch` trigger as well — useful for re-running apply without a code change
- [ ] Make a small non-infra change (e.g., update a comment in a `.tf` file), push to a branch, merge via PR, and verify apply runs cleanly on merge
- [ ] Commit:
  ```
  git add .github/workflows/tf-apply.yml
  git commit -m "feat: add Terraform apply GitHub Actions workflow"
  ```

---

## Phase 9: Deploy my-go-gopher via Config Sync

### Task 18: Add my-go-gopher to gitops repo

**Read first:**
- [Config Sync repository structure](https://cloud.google.com/anthos-config-management/docs/concepts/configs)
- Your existing `gitops/` repo at `D:\Code\gitops`

**Why:** Config Sync is now watching your `gitops/` repo. Adding `my-go-gopher` manifests there will cause it to automatically deploy to both clusters — you don't `kubectl apply` anything manually.

- [ ] In your `gitops/` repo, create `apps/my-go-gopher/` directory
- [ ] Add a `kustomization.yaml` that references the app's base manifests
- [ ] Add a `deployment.yaml` for `my-go-gopher` — use the image from your existing Docker Hub or build a new one. Make sure:
  - `serviceAccountName` references the KSA you created in Phase 6
  - Resource `requests` and `limits` are set (Policy Controller requires it)
  - Image tag is NOT `latest` (Policy Controller requires it)
- [ ] Commit and push to `gitops/` main branch
- [ ] Watch Config Sync pick it up (takes up to 30 seconds):
  ```
  kubectl get pods -n default --context=mrd-gke-usc1
  kubectl get pods -n default --context=mrd-gke-euw1
  ```
  Expected: `my-go-gopher` pod running on both clusters
- [ ] Verify the pod is using Workload Identity correctly (no SA key file mounted):
  ```
  kubectl exec -it <pod-name> --context=mrd-gke-usc1 -- env | grep GOOGLE
  ```

---

## Phase 10: Harden and Verify

### Task 19: Tighten Policy Controller to enforce mode

**Why:** You've been running in `warn` mode — now that `my-go-gopher` satisfies all constraints, switch to `deny` to see real enforcement.

- [ ] Update each constraint YAML to change `enforcementAction: warn` → `enforcementAction: deny`
- [ ] Apply to both clusters:
  ```
  kubectl apply -f addons/policy-controller/constraints/ --context=mrd-gke-usc1
  kubectl apply -f addons/policy-controller/constraints/ --context=mrd-gke-euw1
  ```
- [ ] Test enforcement — try to deploy a pod with `image: nginx:latest`:
  ```
  kubectl run bad-pod --image=nginx:latest --context=mrd-gke-usc1
  ```
  Expected: error from Policy Controller: `[disallow-latest-tag] container <bad-pod> uses a disallowed tag <latest>`
- [ ] Commit:
  ```
  git add addons/policy-controller/constraints/
  git commit -m "feat: set policy controller constraints to enforce mode"
  ```

---

### Task 20: Verify full destroy and re-create cycle

**Why:** This is the most important test for this architecture. Can you destroy `mrd-dev` and rebuild it from scratch without losing state or config? This verifies the bootstrap separation actually works.

- [ ] From repo root, destroy only the ephemeral layers (NOT bootstrap):
  ```
  terragrunt run-all destroy --terragrunt-exclude-dir bootstrap \
    --terragrunt-non-interactive
  ```
  This should destroy addons → clusters → networking/iam in reverse dependency order
- [ ] Verify `plt-bootstrap` and state bucket are untouched:
  ```
  gsutil ls gs://plt-tfstate-{suffix}/
  gcloud projects describe plt-bootstrap-{suffix}
  ```
  Expected: state files still exist, bootstrap project still alive
- [ ] Re-apply everything from scratch:
  ```
  terragrunt run-all apply --terragrunt-exclude-dir bootstrap \
    --terragrunt-non-interactive
  ```
- [ ] Verify `my-go-gopher` comes back up on both clusters via Config Sync (no manual kubectl apply needed)
- [ ] Commit a `RUNBOOK.md` at repo root documenting the spin-up and tear-down commands:
  ```
  git add RUNBOOK.md
  git commit -m "docs: add spin-up and tear-down runbook"
  ```

---

## Self-Review Checklist

**Spec coverage:**
- [x] Terragrunt layered structure — Phases 2–5
- [x] GCP bootstrap pattern (chicken-and-egg) — Phase 1
- [x] plt-bootstrap permanent / mrd-dev ephemeral — Tasks 3–5
- [x] VPC-native networking, Cloud NAT, private subnets — Phase 3
- [x] Private GKE clusters, spot VMs, Fleet enrollment — Phase 5
- [x] Workload Identity (pods) — Phase 6
- [x] Config Sync (Fleet-level GitOps) — Task 13
- [x] Policy Controller with 3 constraints — Task 14
- [x] Binary Authorization (warn mode) — Task 15
- [x] GitHub Actions OIDC → GCP keyless auth — Phase 8
- [x] $20 billing budget alert — Task 4
- [x] Naming conventions (mrd-, plt- prefixes) — enforced via variable defaults
- [x] my-go-gopher deployed via Config Sync — Task 18
- [x] Destroy/re-create cycle verified — Task 20
- [x] pre-commit hooks + tflint/tfsec — Task 1
- [x] Branch protection + CODEOWNERS + PR template — Task 2
