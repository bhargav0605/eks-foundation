# eks-foundation

A reusable Terraform project for spinning up production-ready Amazon EKS clusters.

Designed to be deployed for any AWS account by editing **one file** — `terraform.tfvars`.

---

## What This Is

A starter foundation you can use to:

- Provision EKS clusters quickly and consistently across multiple AWS accounts or clients
- Customize every important setting (region, VPC, instance types, node groups, add-ons) from a single input file
- Extend over time with custom components (Cilium, Karpenter, service meshes) without rewriting the base

**Built from scratch.** No community Terraform modules anywhere. Every AWS resource is written explicitly so you always know what's being created and can change anything without fighting abstractions.

---

## How It's Organized

The project is broken into small, focused modules. The root composes them together:

```
eks-foundation/
│
├── terraform.tfvars.example   ← Copy this to terraform.tfvars and fill in
├── terraform.tfvars           ← Your per-deployment values (gitignored)
│
├── versions.tf                ← Terraform + provider versions
├── providers.tf               ← AWS provider setup
├── variables.tf               ← All input variable definitions
├── locals.tf                  ← Common tags + computed values
├── main.tf                    ← Wires the modules together (added in Phase 2)
├── outputs.tf                 ← Cluster endpoint, etc. (added in Phase 4)
├── backend.tf                 ← Remote state (added in Phase 7)
│
└── modules/
    ├── vpc/                   ← Networking (VPC, subnets, NAT, routing)
    ├── existing-vpc-tagger/   ← Tags an existing VPC's subnets for EKS
    ├── iam/                   ← Cluster + node IAM roles
    ├── eks-cluster/           ← Control plane, security groups, KMS, OIDC
    ├── eks-nodes/             ← Managed worker node groups
    └── eks-addons/            ← vpc-cni, kube-proxy, coredns
```

Each module is **self-contained** — its own `variables.tf`, `outputs.tf`, and resources. You can copy any module folder into another project and use it on its own.

---

## How to Use It

### 1. Prerequisites

- Terraform `>= 1.6.0`
- AWS CLI configured with credentials for your target account (`aws configure` or `AWS_PROFILE`)
- `kubectl` (only needed once the cluster is up)

### 2. Configure your deployment

```bash
cd eks-foundation
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
```

### 3. Deploy

```bash
terraform init
terraform plan       # review what will be created
terraform apply      # create it
```

That's it. Every per-deployment setting lives in `terraform.tfvars` — the `.tf` files never need to change between deployments.

---

## What You Can Configure

All from `terraform.tfvars`:

| Setting | Example |
|---|---|
| AWS region | `us-east-1`, `ap-south-1`, anything |
| Cluster name & environment | `client-a-prod`, `dev`, etc. |
| Tags | Layered: common + per-module + per-resource |
| VPC | Create a new one *or* use an existing VPC you already have |
| Subnet CIDRs | Public + private, one CIDR per AZ |
| NAT gateway | Single (cheap) or one per AZ (HA) |
| Kubernetes version | Whatever EKS supports (e.g. `1.35`) |
| API endpoint access | Public, private, or both — with CIDR allowlist |
| Control plane encryption | KMS envelope encryption toggle |
| Node groups | Any number; each with its own instance types, capacity (on-demand/spot), AMI, disk, scaling, labels, taints |
| Add-ons | Toggle and version-pin `vpc-cni`, `kube-proxy`, `coredns` individually |

---

## Bring Your Own VPC

If a deployment target already has a VPC you want to use instead of creating a new one, set `create_vpc = false` in your tfvars and provide the existing VPC ID and subnet IDs.

The project will skip VPC creation and use what you provide. Optionally, it can auto-apply the EKS-required tags to your existing subnets so load balancers work out of the box.

---

## Reusing Modules in Other Projects

The modules under `modules/` are designed to be portable. Three ways to reuse them:

1. **Copy the folder.** Drop `modules/vpc/` into another project's `modules/` folder and reference it as `source = "./modules/vpc"`. Simplest.
2. **Shared local directory.** Keep modules in a central place (e.g. `~/Developer/terraform-modules/`) and reference them with relative paths from each project.
3. **Git source.** Once you publish modules to a Git repo, reference them with `source = "git::https://github.com/<you>/terraform-modules.git//vpc?ref=v1.0.0"` for version-pinned, immutable usage across projects.

---

## Build Status

This project is being built incrementally. Each phase ends with `terraform validate` clean before the next begins.

| Phase | Scope | Status |
|---:|---|:---:|
| 1 | Root skeleton (providers, variables, tagging) | ✅ done |
| 2 | `modules/vpc` + bring-your-own-VPC support | next |
| 3 | `modules/iam` (cluster + node roles) | pending |
| 4 | `modules/eks-cluster` (control plane, SGs, KMS, OIDC) | pending |
| 5 | `modules/eks-nodes` (managed node groups) | pending |
| 6 | `modules/eks-addons` (vpc-cni, kube-proxy, coredns) | pending |
| 7 | Remote state (S3 + DynamoDB) | pending |
| 8+ | Cilium, Karpenter, ALB controller, observability, more | future |

---

## Design Principles

- **Everything raw.** No third-party Terraform modules. Every `aws_*` resource is explicit and readable.
- **One file per deployment.** All per-deployment values live in `terraform.tfvars`. Code files never change between deployments.
- **Layered tagging.** Common tags everywhere + per-module overlays + optional per-resource extras. Filter resources in the AWS console by client, component, or tier.
- **Composable.** Each module has a narrow contract (inputs / outputs). Adding a new capability = adding a module and wiring it into `main.tf`.
- **Built to extend.** The base is intentionally minimal so Cilium, Karpenter, ingress controllers, and observability stacks can be layered on without conflict.
