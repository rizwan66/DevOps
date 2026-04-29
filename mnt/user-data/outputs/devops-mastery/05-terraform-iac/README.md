# Module 5 — Terraform & infrastructure as code

> *"Infrastructure provisioning via Terraform for cloud-native applications serving a large user base."*

The job description mentions Terraform specifically. This module assumes you've seen Terraform before but want to use it well, not just acceptably.

## What infrastructure-as-code actually changes

Before IaC, infrastructure was provisioned by clicking around in cloud consoles. The state of your infrastructure existed in three contradictory places: the cloud provider, the wiki page where someone documented it, and the head of the engineer who set it up. Reproducing it on a new account took weeks.

IaC says: the **only** way to change infrastructure is by changing code. The code is the source of truth. Drift between code and reality is treated as a bug.

This is exactly the same insight as GitOps for Kubernetes — declarative state, versioned in Git, reconciled. Terraform is GitOps for cloud infrastructure, except the reconciliation is manual (you run `terraform apply`) rather than continuous.

## Terraform core mental model

Terraform has three things you must keep separate in your head:

1. **The configuration** (`.tf` files): what you want.
2. **The state** (`terraform.tfstate`): what Terraform thinks currently exists.
3. **The real world**: what actually exists in the cloud.

`terraform plan` compares configuration ↔ state and shows you the diff.  
`terraform apply` makes real-world ↔ state changes and updates state.  
`terraform refresh` (mostly deprecated) updates state ↔ real world.

The job of a Terraform engineer is largely about keeping these three things in sync. Drift between them is the source of 90% of Terraform pain.

## The state file: where most production accidents happen

The state file contains everything Terraform needs to map your config to real resources, including potentially **secrets** (database passwords, API keys, certificates). Treat it like crown jewels.

### Three rules for state, in order of importance:

1. **Use remote state with locking, always.** Local `terraform.tfstate` is for tutorials only. In a team, two people running `apply` simultaneously will corrupt the state.
2. **Encrypt state at rest.** S3 with SSE-KMS, GCS with CMEK, or Terraform Cloud.
3. **Restrict access.** State is more sensitive than your code. Lock down the bucket like you lock down production credentials.

Standard pattern for AWS:

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"
    key            = "prod/network/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "mycompany-tfstate-locks"
    encrypt        = true
    kms_key_id     = "alias/terraform-state"
  }
}
```

For GCP:
```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-tfstate"
    prefix = "prod/network"
  }
}
```

GCS provides locking via object generation; you don't need a separate locks table.

### State file separation

Don't put everything in one state file. The blast radius of a corrupted state file is the entire scope of that file. Separate by:

- **Environment** (dev/staging/prod each have their own state)
- **Layer** (network, compute, data, app — different state files)
- **Lifecycle** (rarely-changed things separate from frequently-changed things)

A typical layout:
```
mycompany-tfstate/
├── prod/
│   ├── network/terraform.tfstate
│   ├── eks/terraform.tfstate
│   ├── databases/terraform.tfstate
│   └── apps/terraform.tfstate
├── staging/
│   └── ... same structure
└── dev/
    └── ...
```

## Modules: the unit of reuse

A Terraform module is just a directory with `.tf` files. Any directory can be a module. Modules can be called from other modules.

### When to extract a module

The "rule of three": don't extract a module the first time, or even the second. The third time you'd write nearly the same configuration, *that's* when to extract.

Premature module extraction creates a module with the wrong abstraction, which is harder to fix than copy-pasted code.

### Module structure

```
modules/eks-cluster/
├── README.md                  # required — usage examples
├── versions.tf                # required Terraform/provider versions
├── variables.tf               # inputs
├── outputs.tf                 # outputs
├── main.tf                    # primary resources
├── iam.tf                     # IAM-specific resources
├── networking.tf              # subnets, security groups
└── examples/
    └── basic/
        └── main.tf
```

`versions.tf`:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Module interface design

The module's variables and outputs are its public API. Treat changes to them like API changes.

```hcl
# Bad — leaky abstraction
variable "vpc_id" { type = string }
variable "subnet_ids" { type = list(string) }
variable "security_group_ids" { type = list(string) }
variable "node_instance_profile" { type = string }
# ... 25 more

# Better — structured input
variable "network" {
  type = object({
    vpc_id     = string
    subnet_ids = list(string)
  })
}

variable "compute" {
  type = object({
    instance_type     = string
    desired_capacity  = number
    max_capacity      = number
  })
}
```

Use object types to group related inputs. It's much easier to extend later without breaking callers.

## Workspaces vs branches vs directories

Terraform has a "workspace" feature. **Don't use it for environments.** It's a footgun. Workspaces share the same backend config, so they all live in the same state bucket / different state file. They look like a clean way to have dev/staging/prod separation, but:

- It's easy to be in the wrong workspace and apply prod changes by accident.
- The "current workspace" is a local concept, not a Git concept — there's no PR review of "switch to prod".
- Backend config is shared, so you can't have different KMS keys, different buckets, etc. per environment.

Use **separate directories** per environment, each with its own backend config. This is more verbose but strictly safer.

```
infrastructure/
├── modules/                  # reusable, generic
│   └── eks-cluster/
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── terraform.tfvars
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

Workspaces are useful for *short-lived* state separation — feature branches, ephemeral environments. Not for long-lived environments.

## Provider versioning

Pin everything. Loose version constraints (`>= 4.0`) seem flexible but mean a `terraform init` on a different machine months later can pull a different version that breaks things.

```hcl
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.70"  # 5.70.x but not 5.71+
  }
}
```

The `.terraform.lock.hcl` file (committed to Git) records exact versions and hashes. Treat it like `package-lock.json` or `Cargo.lock`.

## DRY without HCL gymnastics

A common impulse: "I want to share variables across all environments." Then you discover Terraform doesn't have great built-in support for this. Common solutions:

1. **Just accept some duplication.** Three environments, three sets of `.tfvars` files. Verbose but readable.
2. **Use a wrapper tool.** Terragrunt is the most popular — it's HCL on top of Terraform that handles backend config, variable sharing, and dependency wiring.
3. **Use a deployment tool.** Spacelift, Atlantis, env0, Terraform Cloud — these run Terraform on your behalf with shared config.

For solo projects or small teams, option 1 is fine. For more than ~5 environments, options 2 or 3 are worth the investment.

## CI/CD for Terraform

Don't run `terraform apply` from a developer's laptop in production. The pattern that works:

1. Developer opens a PR with Terraform changes.
2. CI runs `terraform plan` and posts the plan as a PR comment.
3. Reviewer reads the plan (this is the security gate).
4. PR is merged.
5. CI on the main branch runs `terraform apply`.

Tools that do this well: Atlantis (open source), Spacelift, Terraform Cloud, Env0. You can roll your own with GitHub Actions, but the failure modes are subtle (concurrent applies, state corruption on cancelled runs).

## Common pitfalls

- **`terraform import` for everything.** Importing existing resources is fine, but if you import 200 resources by hand instead of refactoring or using `terraformer`, you'll lose your weekend.
- **`count` vs `for_each`.** `count` uses indices — removing item 2 from a list shifts items 3, 4, 5 and Terraform thinks they need to be destroyed and recreated. `for_each` uses keys — much safer for resources that might come and go.
- **`null_resource` with `local-exec` provisioners.** You're using Terraform to run shell scripts. Stop. Find a real resource or use a different tool.
- **Hard-coded secrets in `.tf` files.** Use `aws_secretsmanager_secret`, Vault, GCP Secret Manager, or environment variables. Never plain text.
- **Cross-state references via `data` sources.** `data "terraform_remote_state"` couples states tightly and creates implicit dependencies. Prefer SSM Parameter Store / Secret Manager / GCP equivalents — explicit named outputs that other states read.
- **Big-bang refactors.** Moving a resource between modules without `moved` blocks (or `terraform state mv`) destroys and recreates it. Always use `moved {}` blocks for refactors.

## See `lab.md` for hands-on exercise.
