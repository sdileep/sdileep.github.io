# Maintain multi-environment, multi-region infrastructure on AWS using Terraform


In one of my recent engagements, I had to work out an approach to manage AWS infrastructure across multiple regions, and for various environments, using Terraform.

As any sane copy-paste-tweak developer would, I did "google" for _inspiration_ but ended up finding content that solved partially, either only for multi-environment or multi-region scenarios, or wasn't thought through (for example, no isolation of state between regions). For anyone in a similar need, here's something to build upon.

## Prerequisites

An understanding of [Terraform](https://www.terraform.io/docs/configuration/index.html) and the concepts of [Modules](https://www.terraform.io/docs/modules/usage.html), [Backends](https://www.terraform.io/docs/backends/index.html), [Workspaces](https://www.terraform.io/docs/state/workspaces.html), [Remote State](https://www.terraform.io/docs/state/remote.html) & [AWS provider](https://www.terraform.io/docs/providers/aws/index.html) would be required to make sense of the content in this post:

The following tools would be required to experiment with the provided sample code :

- [AWS Command Line Interface](https://aws.amazon.com/cli) with access to AWS configured
- [Terraform](https://www.terraform.io/downloads.html)

## What do we have to play with?

### Module

Terraform's `module` system helps us create configurable infrastructure templates that could be reused across various environments (_product-a_ deployed to _development/production_) or across various products (standard _s3/DynamoDB/SNS/etc_ templates)

### Backend

Terraform's `backend` configuration for AWS `s3` remote state uses the following configuration variables to organize infrastructure state:

- `bucket`: name of the `s3` bucket where state would be stored
- `workspace_key_prefix`: custom prefix on state file path
- `workspace`: name of the workspace
- `key`: state file name

In `s3`, state file could then be located at `<bucket>/<workspace_key_prefix>/<workspace>/<key>`. If we substitute `workspace` with _ap-southeast-1_ or _ap-southeast-2_, if we substitute the variables `workspace_key_prefix` with _product-a_ and `key` with _terraform.tfstate_, we end up with state files stored as:

```
bucket
    └──product-a
        ├──ap-southeast-1
        │   └── terraform.tfstate
        └──ap-southeast-2
            └── terraform.tfstate
```

This sets up grouping infrastructure states at a product/project level while establishing isolation between deployments to different regions while storing all those states conveniently in one place.

## Approach

Using the terraform module and backend systems, the infrastructure-as-source code repository layout & Terraform backend configuration snippet described in the section provides us with a way to:

- establish a structure in which common or a product/project's infrastructure is templatised for reuse across various enviroments
- fine tune product/project's infrastructure at an environment level while even adding environment specific infrastructure for those non-ideal cases
- maintain state at a region level so that we could have better isolation, canary deploy, etc.,

#### Source Layout

```
├── environments
│   ├── development
│   |   ├── ap-southeast-1.tfvars
│   |   ├── ap-southeast-2.tfvars
│   |   ├── variables.tf
│   |   ├── main.tf
│   |   ├── provider.tf
│   |   ├── terraform.tf
│   |   └── terraform.tfvars
│   ├── test
│   |   ├── ap-southeast-1.tfvars
│   |   ├── ap-southeast-2.tfvars
│   |   └── ...
│   ├── stage
│   |   ├── ap-southeast-1.tfvars
│   |   └── ...
│   └── production
│   |   └── ...
└── modules
    ├── aws-s3
    │   ├── main.tf
    │   ├── provider.tf
    │   └── variables.tf
    ├── product-a
    │   ├── main.tf
    │   ├── provider.tf
    │   └── variables.tf
    └── sub-system-x
        ├── main.tf
        ├── provider.tf
        └── variables.tf
```

- `environments`: folder to isolate various environment (_development_/_test_/_stage_/_production_) specific configuration. This also helps with flexibility of maintaining environment specific infrastructure for those, common, non-ideal scenarios.
- `modules`: folder to host reusable resource sets grouped at product/project or at a sub-system or common infrastructure components level. This folder doesn't have to exist in the same repository - it does here as an example and might very well serve the purpose of more than handful of usecases.

Region specific configurations are managed through their respective `<workpace>.tfvars` file. For example, `environments/development/ap-southeast-2.tfvars` file for _ap-southeast-2_ region in _development_ environment.

Also, `terraform.tfvars` file found inside _development_/_test_/_stage_/_production_ folder under _environments_ could be used to set common configuration for a given environment, across all regions.

#### Backend Configuration

```
terraform {
  required_version = "~> 0.12.6"

  backend "s3" {
    bucket               = "terraform-state-bucket"
    dynamodb_table       = "terraform-state-lock-table"
    encrypt              = true
    key                  = "terraform.tfstate"
    region               = "ap-southeast-2"
    workspace_key_prefix = "product-a"
  }
}
```

_Note_ : The configuration described in this post and the included sample presume state bucket per environment. But, if the need is to store state from all environments in a common bucket, we could update `workspace_key_prefix` value to include environment in it. For example, with _product-a/development_ or _product-a/production_, we end up with state under following path in `s3`:

```
bucket
    └──product-a
        ├──development
        │   ├──ap-southeast-1
        │   │   └── terraform.tfstate
        │   └──ap-southeast-2
        │       └── terraform.tfstate
        └──production
            ├──ap-southeast-1
            │   └── terraform.tfstate
            └──ap-southeast-2
                └── terraform.tfstate
```

## Repository

Source for a sample setup could be found at [here](https://www.terraform.io/docs/configuration/index.html).

## Working with the setup

Navigate to the environment folder, `development` for example, on the terminal.
_Note:_ Working configuration to access AWS environment is presumed

**Initialize terraform**

To get started, first initialize your local terraform state information

```sh
terraform init
```

**List out the available workspaces**

```sh
terraform workspace list
```

**Create a new workspace (if it doesn't exist already)**

```sh
#terraform workspace new <workspace-name>
terraform workspace new ap-southeast-2
```

**Select a workspace**

```sh
#terraform workspace select <workspace-name>
terraform workspace select ap-southeast-2
```

**Plan & apply changes**

```sh
#terraform plan -var-file=<workspace-name>.tfvars
#terraform apply -var-file=<workspace-name>.tfvars

terraform plan -var-file=ap-southeast-2.tfvars
terraform apply -var-file=ap-southeast-2.tfvars
```

**Repeat for other regions**

For _ap-southeast-1_ region:

```sh
terraform workspace new ap-southeast-1
terraform workspace select ap-southeast-1
terraform plan -var-file=ap-southeast-1.tfvars
terraform apply -var-file=ap-southeast-1.tfvars
```

Hopefully, this note helps a mate out!

