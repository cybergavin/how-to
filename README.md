---
description: >-
  This page explains how to use OpenTofu's early evaluation feature to simplify
  your environment-specific configurations.
---

# Use early evaluation with OpenTofu

Here’s how to configure your infrastructure using **OpenTofu** for separate AWS accounts and dynamic backend configurations.

***

#### **Steps to Configure OpenTofu for Your Requirement**

**1. Directory Structure**

Maintain a well-organized directory structure:

```plaintext
├── environments/
│   ├── dev/
│   │   └── terraform.tfvars
│   ├── prd/
│   │   └── terraform.tfvars
│   ├── stg/
│       └── terraform.tfvars
├── main.tf
├── provider.tf
└── variables.tf
```

***

**2. Environment-Specific Variables (`terraform.tfvars`)**

Define environment-specific variables to configure the backend and other AWS-specific settings.

**Example: `environments/dev/terraform.tfvars`**

```hcl
environment     = "dev"
aws_profile     = "dev-account"
aws_region      = "us-west-2"
s3_bucket       = "my-terraform-state-dev"
s3_key          = "state/dev/terraform.tfstate"
dynamodb_table  = "terraform-lock-table-dev"
```

**Example: `environments/prd/terraform.tfvars`**

```hcl
environment     = "prd"
aws_profile     = "prd-account"
aws_region      = "us-east-1"
s3_bucket       = "my-terraform-state-prd"
s3_key          = "state/prd/terraform.tfstate"
dynamodb_table  = "terraform-lock-table-prd"
```

***

**3. Dynamic Backend Configuration in `main.tf`**

Use OpenTofu's **early evaluation** to dynamically configure the backend based on variables or local values.

**Example: `main.tf`**

```hcl
locals {
  environment     = var.environment
  backend_bucket  = var.s3_bucket
  backend_key     = var.s3_key
  backend_region  = var.aws_region
  backend_dynamo  = var.dynamodb_table
}

terraform {
  backend "s3" {
    bucket         = local.backend_bucket
    key            = local.backend_key
    region         = local.backend_region
    dynamodb_table = local.backend_dynamo
    encrypt        = true
  }
}

module "vpc" {
  source      = "./modules/vpc"
  cidr_block  = var.vpc_cidr_block
}

module "ec2" {
  source        = "./modules/ec2"
  instance_type = var.instance_type
  vpc_id        = module.vpc.vpc_id
}
```

***

**4. Provider Configuration (`provider.tf`)**

Configure the AWS provider dynamically for each environment.

**Example: `provider.tf`**

```hcl
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile
}
```

***

**5. Variable Definitions (`variables.tf`)**

Define all required variables for your infrastructure.

**Example: `variables.tf`**

```hcl
variable "environment" {
  description = "The environment name (e.g., dev, prd)"
  type        = string
}

variable "aws_profile" {
  description = "AWS CLI profile name"
  type        = string
}

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
}

variable "s3_bucket" {
  description = "S3 bucket for backend state storage"
  type        = string
}

variable "s3_key" {
  description = "S3 key for the backend state file"
  type        = string
}

variable "dynamodb_table" {
  description = "DynamoDB table for backend state locking"
  type        = string
}

```

***

**6. Running OpenTofu Commands**

You can now initialize and deploy your infrastructure using OpenTofu, leveraging dynamic backend configurations.

**Example Commands for the `dev` Environment:**

```bash
cd /path/to/your/project

# Initialize the backend with the dev configuration
tofu init -var-file="environments/dev/terraform.tfvars"

# Preview changes
tofu plan -var-file="environments/dev/terraform.tfvars"

# Apply changes
tofu apply -var-file="environments/dev/terraform.tfvars"
```

**Example Commands for the `prd` Environment:**

```bash
# Initialize the backend with the prd configuration
tofu init -var-file="environments/prd/terraform.tfvars"

# Preview changes
tofu plan -var-file="environments/prd/terraform.tfvars"

# Apply changes
tofu apply -var-file="environments/prd/terraform.tfvars"
```

***

**7. CI/CD Pipeline Integration**

You can integrate OpenTofu into your CI/CD pipeline with dynamic backend configurations.

**Example GitHub Actions Workflow:**

```yaml
name: OpenTofu Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Environment to deploy (e.g., dev, prd)"
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Setup OpenTofu
        run: |
          curl -fsSL https://opentofu.org/install.sh | bash
          tofu --version

      - name: Deploy Environment
        run: |
          ENV=${{ inputs.environment }}
          tofu init -var-file="environments/$ENV/terraform.tfvars"
          tofu apply -var-file="environments/$ENV/terraform.tfvars" -auto-approve
```

***

#### **Benefits of This Approach**

1. **Dynamic Backend Configuration**: Using OpenTofu's early evaluation, you avoid hardcoding backend values or relying on external scripts.
2. **AWS Account Isolation**: Each environment uses its own AWS account and backend configuration.
3. **Scalability**: Easily add more environments by creating new `terraform.tfvars` files.
4. **CI/CD Integration**: Automate deployments across environments with seamless backend management.
