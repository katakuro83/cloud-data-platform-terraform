# Cloud Data Platform as Code

Fully reproducible cloud data platform warehouse, storage, IAM, and networking defined and deployed via Infrastructure as Code, with CI/CD.

**Stack:** Terraform · AWS · GitHub Actions

> Implemented against AWS as the primary provider. The module boundaries (`networking` / `storage` / `iam` / `warehouse`) are deliberately provider agnostic in *design*.
---

## Architecture

```
                    ┌────────────────────────────────────────────┐
                    │                    VPC                     │
                    │  ┌────────────┐        ┌────────────────┐  │
                    │  │  Private   │        │  Public        │  │
                    │  │  subnets   │        │  subnets       │  │
                    │  │ (warehouse)│        │  (NAT gateway) │  │
                    │  └────────────┘        └────────────────┘  │
                    └────────────────────────────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
┌───────────────┐         ┌───────────────────┐          ┌──────────────────┐
│  S3 storage   │         │Redshift Serverless│          │  IAM roles &     │
│  raw / curated│         │warehouse          │          │  policies        │
│  buckets,     │         │(VPC private)      │          │  least privilege │
│  versioned,   │         │                   │          │  per persona     │
│  encrypted    │         │                   │          │ (engineer/reader)│
└───────────────┘         └───────────────────┘          └──────────────────┘
```

| Module | Provisions | Notes |
|---|---|---|
| `modules/networking` | VPC, public/private subnets, NAT gateway, security groups | Warehouse lives in private subnets no direct internet exposure |
| `modules/storage` | S3 buckets (`raw`, `curated`), versioning, SSE KMS encryption, lifecycle rules | Mirrors the raw/curated split used across the other projects in this portfolio |
| `modules/iam` | Least privilege roles: `data engineer` (read/write), `data reader` (read only), warehouse service role | No wildcard (`*`) permissions anywhere |
| `modules/warehouse` | Redshift Serverless namespace + workgroup, deployed inside the private subnets | Serverless keeps idle cost near zero, matching the pattern used for the Snowflake warehouse project |

---

## Project Structure

```
cloud-data-platform-terraform/
├── bootstrap/                  # One time: creates the S3 backend + DynamoDB lock table
├── modules/
│   ├── networking/
│   ├── storage/
│   ├── iam/
│   └── warehouse/
├── environments/
│   ├── dev/                    # Small footprint, fast iteration
│   └── prod/                   # Full size, deletion protection enabled
├── .github/workflows/
│   ├── terraform-plan.yml      # Runs on every PR: fmt, validate, plan (posts plan as PR comment)
│   └── terraform-apply.yml     # Runs on merge to main: apply against prod
├── architecture/architecture.md
└── docs/setup_guide.md
```

Each environment is a thin composition root it wires the shared modules together with environment specific sizing/variables, so `dev` and `prod` can never drift into different *architectures*, only different *scale*.

---

## CI/CD Pipeline

| Trigger | Workflow | What happens |
|---|---|---|
| Pull request touching `environments/**` or `modules/**` | `terraform plan.yml` | `terraform fmt check`, `terraform validate`, `terraform plan`, plan output posted as a PR comment for review |
| Merge to `main` | `terraform apply.yml` | `terraform apply` runs automatically against `prod` using the reviewed plan |

Authentication uses **GitHub OIDC** GitHub Actions assumes an AWS IAM role directly, with no long lived AWS access keys stored as repo secrets. This is set up once as part of `bootstrap/`.

---

## Design Highlights

- **State isolation per environment** `dev` and `prod` use separate state files (separate S3 keys), so a `terraform apply` in one can never touch the other.
- **Plan before apply, always** every change is visible as a plan on the PR before anything is applied; nothing reaches `prod` without review.
- **No long lived cloud credentials in CI** OIDC federation means there's no AWS access key to leak from a compromised Actions run or a misconfigured secret.
- **Least privilege IAM by persona**, not by convenience `data engineer` and `data reader` roles are scoped to exactly the S3/Redshift actions they need, not broad `*` grants.
- **Everything reproducible from zero** tearing down and re running `terraform apply` from a clean account should produce an equivalent platform; nothing is created via console click-ops.

---

## Possible Extensions

- Add an Azure implementation of each module (`modules/networking azure`, etc.) using the same variable contracts, selected via a `cloud_provider` input
- Add Terraform Cloud / Atlantis for policy as code (Sentinel/OPA) gating on `terraform plan`
- Add automated cost estimation (Infracost) as a PR check alongside the plan
- Extend `modules/warehouse` with a Snowflake provider variant, reusing the networking/IAM modules unchanged

---
