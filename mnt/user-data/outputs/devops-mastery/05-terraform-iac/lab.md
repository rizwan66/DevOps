# Module 5 — Lab: Terraform module for an EKS cluster

You'll build a small but realistic Terraform setup: a network module, an EKS cluster module, and a top-level configuration that wires them together. You'll then go through a refactor that demonstrates correct use of `moved` blocks.

## Choice of cloud

This lab uses AWS as a target, but the concepts are identical for GCP/Azure. If you don't have an AWS account, you can run almost everything against [LocalStack](https://localstack.cloud/) instead — point your AWS provider at the LocalStack endpoint.

## Prerequisites

```bash
brew install terraform tflint  # or download manually
terraform --version             # >= 1.6
```

If using AWS for real:
```bash
aws configure  # set up credentials
aws sts get-caller-identity  # verify
```

## Step 1 — Bootstrap the state backend

You can't use S3+DynamoDB for state until those resources exist. The chicken-and-egg solution: bootstrap them with local state, then migrate.

`bootstrap/main.tf`:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "eu-west-1" }

variable "company_name" { type = string }

resource "aws_s3_bucket" "tfstate" {
  bucket = "${var.company_name}-tfstate"
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_public_access_block" "tfstate" {
  bucket                  = aws_s3_bucket.tfstate.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "tfstate_locks" {
  name         = "${var.company_name}-tfstate-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
cd bootstrap
terraform init
terraform apply -var="company_name=mycompany"
```

## Step 2 — Build a network module

`modules/network/versions.tf`:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
```

`modules/network/variables.tf`:
```hcl
variable "name" {
  description = "Name prefix for tagging resources"
  type        = string
}

variable "cidr_block" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones for subnets"
  type        = list(string)
}
```

`modules/network/main.tf`:
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags                 = { Name = var.name }
}

resource "aws_subnet" "public" {
  for_each                = toset(var.availability_zones)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, index(var.availability_zones, each.value))
  availability_zone       = each.value
  map_public_ip_on_launch = true
  tags = {
    Name                     = "${var.name}-public-${each.value}"
    "kubernetes.io/role/elb" = "1"
  }
}

resource "aws_subnet" "private" {
  for_each          = toset(var.availability_zones)
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, index(var.availability_zones, each.value) + 10)
  availability_zone = each.value
  tags = {
    Name                              = "${var.name}-private-${each.value}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = { Name = var.name }
}

# (NAT gateways, route tables, etc. omitted for brevity but you should add them)
```

`modules/network/outputs.tf`:
```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}

output "public_subnet_ids" {
  value = [for s in aws_subnet.public : s.id]
}

output "private_subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}
```

Note the use of `for_each` rather than `count`. If you remove an availability zone, only that one subnet gets destroyed — not all of them.

## Step 3 — Wire it up in an environment

`environments/dev/backend.tf`:
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"
    key            = "dev/network/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "mycompany-tfstate-locks"
    encrypt        = true
  }
}
```

`environments/dev/main.tf`:
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" { region = "eu-west-1" }

module "network" {
  source = "../../modules/network"
  name   = "dev"
  availability_zones = ["eu-west-1a", "eu-west-1b", "eu-west-1c"]
}
```

```bash
cd environments/dev
terraform init
terraform plan
terraform apply
```

## Step 4 — Add an EKS module (skeleton)

`modules/eks/main.tf` (truncated; the real module is much larger):
```hcl
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn
  version  = var.kubernetes_version

  vpc_config {
    subnet_ids = concat(var.public_subnet_ids, var.private_subnet_ids)
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_AmazonEKSClusterPolicy
  ]
}

resource "aws_iam_role" "cluster" {
  name = "${var.cluster_name}-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cluster_AmazonEKSClusterPolicy" {
  role       = aws_iam_role.cluster.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```

In production, you would not write this from scratch — use `terraform-aws-modules/eks/aws`, a battle-tested community module. But writing your own once is educational.

## Step 5 — The refactor exercise

This is the most important task in the lab. You're going to learn how to refactor without destroying resources.

### Setup

You have your network module structured as above, with subnets created via `for_each`. Imagine the team decides public subnets should be in their own sub-module (e.g. `modules/public-subnets/`).

### The wrong way

You move the resources, run `apply`, and Terraform says: "I will destroy these 3 subnets and create 3 new ones." You apply. Your VPC's load balancers have just lost their subnets. Production goes down.

### The right way: `moved {}` blocks

Add to your config:
```hcl
moved {
  from = module.network.aws_subnet.public["eu-west-1a"]
  to   = module.network.module.public_subnets.aws_subnet.this["eu-west-1a"]
}

moved {
  from = module.network.aws_subnet.public["eu-west-1b"]
  to   = module.network.module.public_subnets.aws_subnet.this["eu-west-1b"]
}

moved {
  from = module.network.aws_subnet.public["eu-west-1c"]
  to   = module.network.module.public_subnets.aws_subnet.this["eu-west-1c"]
}
```

Now run `terraform plan`. You should see "0 to add, 0 to change, 0 to destroy" — Terraform recognises the resources moved and updates state in place.

### Tasks

1. Implement the refactor. Verify with `plan` that it shows zero changes.
2. Add a *real* change after the refactor (change a tag, say). Run `plan` again. Now you see only the genuine change.
3. Once applied successfully, the `moved` blocks can stay forever (they're harmless) or be removed in a later commit. Decide which and why.

## Step 6 — Multi-environment with shared modules

Replicate `environments/dev/` to `environments/staging/` and `environments/prod/`. Each gets its own state file in S3. Each calls the same modules with different parameters.

Tasks:
- Write a `Makefile` or shell script that lets you `make plan-dev`, `make plan-staging`, etc., to avoid `cd`'ing around.
- Set up a GitHub Actions workflow that runs `terraform plan` on every PR and comments the plan output on the PR.

## Reflection

1. Your colleague says: "I'll just `terraform import` the EKS cluster I created in the console last week." What questions would you ask before letting them do it?
2. The state file for production now contains a plaintext password. How did it get there? How do you remove it from the state file safely without it surviving in S3 versioning?
3. If your bootstrap state (the local one from step 1) got deleted, what would you lose? How would you recover the S3 bucket and DynamoDB table for state?
4. Argue both sides: should `terraform apply` be allowed in CI, or should it always require a human to run it manually after a plan review?
5. The job description says "infrastructure provisioning via Terraform for cloud-native applications serving a large user base". What changes when "small" becomes "large"? Module structure? State separation? Plan review process?
