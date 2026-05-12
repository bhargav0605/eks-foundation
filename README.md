# eks-foundation

A raw-Terraform, modular foundation for provisioning Amazon EKS clusters across multiple AWS accounts / clients. Built **without any third-party Terraform modules** — every `aws_*` resource is defined explicitly so the codebase stays transparent, debuggable, and extendable (Cilium, Karpenter, service meshes, etc. layer cleanly on top).

Each per-client deployment is driven by a single `terraform.tfvars` file at the project root.

---

## Status

| Phase | Scope | Status |
|------:|-------|:------:|
| 1 | Root skeleton (versions, providers, core variables) | **in progress** |
| 2 | `modules/vpc` + bring-your-own-VPC support | pending |
| 3 | `modules/iam` | pending |
| 4 | `modules/eks-cluster` (control plane, SGs, KMS, OIDC) | pending |
| 5 | `modules/eks-nodes` (managed node groups) | pending |
| 6 | `modules/eks-addons` (vpc-cni, kube-proxy, coredns) | pending |
| 7 | Remote state backend (S3 + DynamoDB) | pending |
| 8+ | Cilium, Karpenter, ALB controller, observability, etc. | future |

Each phase is independently `terraform validate`-clean before moving on.

---

## Project Layout

```
eks-foundation/
├── versions.tf              # Terraform + provider version pins
├── providers.tf             # AWS provider configuration
├── variables.tf             # Root-level input variables
├── locals.tf                # Computed common tags
├── terraform.tfvars.example # Template — copy to terraform.tfvars
├── main.tf                  # (added in Phase 2) Module composition
├── outputs.tf               # (added in Phase 4) Cluster outputs
├── backend.tf               # (added in Phase 7) S3 remote state
└── modules/
    ├── vpc/                 # Phase 2
    ├── existing-vpc-tagger/ # Phase 2 (BYO-VPC support)
    ├── iam/                 # Phase 3
    ├── eks-cluster/         # Phase 4
    ├── eks-nodes/           # Phase 5
    └── eks-addons/          # Phase 6
```

---

## Requirements

- Terraform `>= 1.6.0`
- AWS CLI configured with credentials for the target account (e.g. `aws configure` or `AWS_PROFILE`)
- `kubectl` (for verifying the cluster once Phase 4+ is built)

---

## Quickstart (current — Phase 1)

```bash
cd eks-foundation

# Copy the example tfvars and edit values for your deployment
cp terraform.tfvars.example terraform.tfvars
$EDITOR terraform.tfvars

# Initialize and validate
terraform init
terraform validate
```

Phase 1 only verifies the skeleton compiles — no resources are created yet. The first apply happens at the end of Phase 2 (VPC).

---

## Reusing These Modules Elsewhere

Each module under `modules/` is fully self-contained (its own `versions.tf`, `variables.tf`, `outputs.tf`). Three reuse patterns:

1. **Copy the folder.** Copy `modules/vpc/` into another project and reference it as `source = "./modules/vpc"`.
2. **Shared local path.** Keep modules in a central directory (e.g. `~/Developer/terraform-modules/`) and reference with `source = "../../terraform-modules/vpc"`.
3. **Git source (future).** Publish modules to a private Git repo and reference with `source = "git::https://github.com/<you>/terraform-modules.git//vpc?ref=v1.0.0"`. Version-pinned and immutable per deployment.

---

## Design Principles

- **Raw resources only.** No `terraform-aws-modules/*` or other community modules. Every resource is explicit so you know exactly what's deployed.
- **One input file per deployment.** All per-client values live in `terraform.tfvars`. The `.tf` files never change between clients.
- **Layered tagging.** `common_tags` (everywhere) → per-module tags → per-resource overlays. Easy to filter resources by client, component, or tier in the AWS console.
- **Composable phases.** Each module has a narrow, well-defined contract (inputs/outputs). Add a new module = wire it into `main.tf` and add its variables.
# eks-foundation
