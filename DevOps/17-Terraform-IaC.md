# Terraform: Infrastructure as Code (IaC)

> **DevOps – Lecture 17 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Where this fits:** so far we automated **apps**: building ([Docker](05-Docker.md)), running ([Kubernetes](08-Kubernetes-Basics.md)) and shipping ([CI/CD](15-CICD-with-GitHub-Actions.md)). Terraform automates the **infrastructure underneath**: servers, networks, buckets, databases, even the Kubernetes cluster itself.
>
> **Practice:** [Terraform & AWS MCQs](Terraform-AWS-MCQs.md) · **Next:** [Lecture 18 – AWS Core Services & VPC](18-AWS-VPC-and-Core-Services.md).

---

## Table of Contents
1. [The Big Picture: What Is IaC?](#1-the-big-picture-what-is-iac)
2. [Why Terraform?](#2-why-terraform)
3. [Setting Up: Terraform + AWS Credentials](#3-setting-up-terraform--aws-credentials)
4. [HCL Basics & Project Layout](#4-hcl-basics--project-layout)
5. [The `terraform` Block & Providers](#5-the-terraform-block--providers)
6. [Resources](#6-resources)
7. [Variables](#7-variables)
8. [Outputs](#8-outputs)
9. [➕ Locals & Data Sources](#9--locals--data-sources)
10. [➕ State: How Terraform Remembers](#10--state-how-terraform-remembers)
11. [The Core Workflow: init → fmt → validate → plan → apply → destroy](#11-the-core-workflow-init--fmt--validate--plan--apply--destroy)
12. [Hands-on: Create an S3 Bucket on AWS](#12-hands-on-create-an-s3-bucket-on-aws)
13. [➕ Modules: Reusable Infrastructure](#13--modules-reusable-infrastructure)
14. [Troubleshooting & FAQs](#14-troubleshooting--faqs)
15. [Quick Revision Cheat Sheet](#15-quick-revision-cheat-sheet)
16. [Likely Exam / Interview Questions](#16-likely-exam--interview-questions)

---

## 1. The Big Picture: What Is IaC?

**The old way (ClickOps):** log in to the AWS console, click through 15 screens to create a server, a bucket and a network. Then do it again for staging, and again for prod. Every time slightly differently. Nobody remembers exactly what was clicked.

**Infrastructure as Code (IaC):** describe infrastructure in **machine-readable text files**, keep them in **Git**, and let a tool **create, change and delete** the real resources to match.

| ClickOps (manual) | IaC |
|-------------------|-----|
| Slow, repetitive | One command builds everything |
| Environments drift apart | dev / staging / prod built from the **same code** |
| No history of who changed what | **Git history + code review (PRs)** for infrastructure |
| Hard to rebuild after a disaster | Re-run the code → identical infrastructure |
| Knowledge lives in people's heads | The code **is** the documentation |

### Declarative vs. imperative
| Imperative ("how") | Declarative ("what") |
|--------------------|----------------------|
| A list of **steps**: "create a bucket, then enable versioning, then…" | The **end result**: "a bucket with versioning should exist" |
| Running twice may create **duplicates** | Running twice changes **nothing** the second time (**idempotent**) |
| Shell scripts, AWS CLI commands | **Terraform**, Kubernetes YAML, CloudFormation |

> 💡 **Simple words mein:** Terraform ko ghar ka **naksha (blueprint)** do. Ghar kaise banana hai woh khud dekhega. Naksha badlo to sirf utna hissa badlega jitna naksha mein badla, poora ghar dobara nahi banega.
>
> It's the same **desired-state** idea as Kubernetes ([L8 §1](08-Kubernetes-Basics.md#1-the-big-picture-why-kubernetes)): you declare what you want, and the tool makes reality match.

---

## 2. Why Terraform?

**Terraform** (by **HashiCorp**) is an IaC tool. You write configuration in **HCL** (HashiCorp Configuration Language), and Terraform calls cloud APIs to make it real.

| Benefit | What it means |
|---------|---------------|
| **Multi-cloud** | One tool and one language for **AWS, Azure, GCP**, plus Kubernetes, GitHub, Cloudflare, Datadog … (thousands of **providers**) |
| **Automation** | `terraform apply` builds an entire environment in minutes |
| **Versioning** | `.tf` files live in Git: history, PR reviews, rollback by reverting a commit |
| **Plan before apply** | `terraform plan` shows **exactly** what will change **before** anything happens |
| **Reusable modules** | Package a setup (e.g. "VPC + subnets") once, reuse it for dev/staging/prod |
| **Dependency graph** ➕ | Terraform works out the **order** itself: it creates the network before the server that uses it |

**➕ How it compares:**
| Tool | Type | Clouds | Notes |
|------|------|--------|-------|
| **Terraform** | Provisioning, declarative | **Any** | Industry standard for IaC |
| AWS CloudFormation | Provisioning, declarative | AWS only | JSON/YAML, AWS-native |
| Pulumi | Provisioning | Any | Real languages (Python, TypeScript) instead of HCL |
| Ansible | **Configuration management** | Any | Configures software **inside** servers (install nginx, edit files). Often used **with** Terraform. |
| OpenTofu | Provisioning | Any | Open-source **fork of Terraform**, created after HashiCorp changed Terraform's license in 2023. Same HCL and commands (`tofu` instead of `terraform`). |

> **Terraform vs. Ansible in one line:** Terraform **builds the house** (servers, networks). Ansible **furnishes the rooms** (software and config inside the servers).

---

## 3. Setting Up: Terraform + AWS Credentials

### Install & verify Terraform
```bash
brew tap hashicorp/tap && brew install hashicorp/tap/terraform   # macOS (see developer.hashicorp.com for others)
terraform version                                                # Terraform v1.x.x
```

### AWS CLI & credentials
```bash
brew install awscli
aws configure
# AWS Access Key ID:     AKIA...
# AWS Secret Access Key: ****
# Default region name:   us-east-1
# Default output format: json
aws sts get-caller-identity          # ✅ prints your account + user = credentials work
```
`aws configure` saves the keys in `~/.aws/credentials` and `~/.aws/config`.

> ⚠️ **Precision point:** Terraform doesn't call the `aws` command itself. The **AWS provider reads the same credentials** (`~/.aws/credentials`, or the env vars `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_PROFILE`). The AWS CLI is the easy way to **set up and test** those credentials, which is why the class installed it.
>
> ⚠️ **Never put access keys in `.tf` files** (they'd end up in Git). Use an **IAM user/role with only the permissions needed** (least privilege), not the root account.

### ➕ VS Code helper
Install the **HashiCorp Terraform** extension for syntax highlighting, autocomplete and format-on-save.

---

## 4. HCL Basics & Project Layout

### HCL syntax in 30 seconds
```hcl
# comment
<BLOCK_TYPE> "<LABEL>" "<LABEL>" {      # e.g. resource "aws_s3_bucket" "my_bucket" {
  argument = "string"
  number   = 3
  enabled  = true
  list     = ["a", "b"]
  map      = { env = "dev", team = "devops" }

  nested_block {
    key = value
  }
}
```
- **Blocks** (`provider`, `resource`, `variable`, `output` …) contain **arguments** (`key = value`).
- References use dots: `var.aws_region`, `aws_s3_bucket.my_bucket.id`.
- Strings can include expressions: `"${var.project}-bucket"`.

### Project layout (convention)
Terraform reads **every `.tf` file in the folder** as one configuration. The file names are just for humans:
```
s3-demo/
├── providers.tf        # terraform {} block + provider "aws" {}
├── variables.tf        # variable blocks
├── main.tf             # resources (the class put everything here, which also works)
├── outputs.tf          # output blocks
├── terraform.tfvars    # ➕ values for the variables
├── .terraform/         # created by init: downloaded providers (don't commit)
├── .terraform.lock.hcl # created by init: exact provider versions (DO commit)
└── terraform.tfstate   # created by apply: what exists (don't commit, see §10)
```

---

## 5. The `terraform` Block & Providers

A **provider** is a **plugin that knows how to talk to one platform's API** (AWS, Azure, GCP, Kubernetes …). `terraform init` downloads it.

```hcl
terraform {
  required_version = ">= 1.5.0"            # which Terraform CLI versions may run this code

  required_providers {
    aws = {
      source  = "hashicorp/aws"            # from the Terraform Registry
      version = "~> 6.0"                   # any 6.x, never 7.0 (protects against breaking changes)
    }
  }
}

provider "aws" {
  region = var.aws_region                  # from class: the region comes from a variable
}
```

| Version constraint | Allows |
|--------------------|--------|
| `= 6.2.0` | Exactly 6.2.0 |
| `>= 6.0` | 6.0 or anything newer (risky: a future 7.0 could break you) |
| `~> 6.0` | ≥ 6.0 and < 7.0 (**the common choice**) |
| `~> 6.2.0` | ≥ 6.2.0 and < 6.3.0 |

> ➕ The same code can use **several providers** at once, e.g. `aws` for servers and `cloudflare` for DNS. That's what "multi-cloud" looks like in practice.

---

## 6. Resources

A **resource** is **one piece of infrastructure** Terraform creates and manages: a bucket, a server, a security group, a DNS record.

```hcl
resource "aws_s3_bucket" "bucket_name" {
#        └── TYPE ──┘   └ LOCAL NAME ┘   (TYPE = provider prefix "aws_" + kind of thing)
  bucket = "my-bucket-name"              # arguments: the real-world settings
}
```

| Part | Meaning |
|------|---------|
| `aws_s3_bucket` | **Resource type**, defined by the provider (see its docs on registry.terraform.io) |
| `bucket_name` | **Local name**, only used inside your Terraform code to refer to it |
| `bucket = "..."` | The **actual AWS bucket name** (must be globally unique!) |

**Referencing one resource from another** (`TYPE.NAME.ATTRIBUTE`):
```hcl
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.bucket_name.id        # ← reference = automatic dependency
  versioning_configuration {
    status = "Enabled"
  }
}
```
Because of that reference, Terraform knows it must **create the bucket first**. You never write the order yourself.

> ➕ Attributes like `.id` and `.arn` are only **known after apply**. `plan` shows them as `(known after apply)`.

---

## 7. Variables

**Input variables** keep values **out of the code**, so the same config works for different regions, environments or names.

```hcl
variable "aws_region" {
  description = "AWS region to deploy into"     # ➕ good practice: document it
  type        = string
  default     = "us-east-1"                     # used if no other value is given
}

variable "environment" {
  type = string
  validation {                                  # ➕ reject bad values early
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging or prod."
  }
}

variable "tags" {
  type    = map(string)
  default = {}
}
```
Use it as **`var.<name>`**: `region = var.aws_region`.

| Type | Example |
|------|---------|
| `string` | `"us-east-1"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b"]` |
| `map(string)` | `{ env = "dev" }` |

### Ways to set a value (➕ later ones win)
```bash
# 1. default in the variable block
# 2. environment var:   export TF_VAR_environment=dev
# 3. terraform.tfvars (loaded automatically)
#      aws_region  = "ap-south-1"
#      environment = "dev"
# 4. *.auto.tfvars files (loaded automatically, alphabetical order)
# 5. command line:      terraform apply -var-file="prod.tfvars"  /  -var="environment=prod"
#                       (both count as command line; if you give several, the last one wins)
# 6. not set anywhere and no default → Terraform PROMPTS you for it
```
> ➕ Mark secrets with `sensitive = true` so they're hidden in plan/apply output. Don't put real secrets in a committed `.tfvars` file.

---

## 8. Outputs

**Outputs** print useful values **after apply** (IDs, URLs, IP addresses). Other tools or modules can also read them.

```hcl
output "bucket_id" {
  description = "Name/ID of the S3 bucket"
  value       = aws_s3_bucket.bucket_name.id
}

output "bucket_arn" {
  value = aws_s3_bucket.bucket_name.arn
}
```
```bash
terraform apply
# Outputs:
# bucket_arn = "arn:aws:s3:::aarham-demo-bucket-3f9a1c2e"
# bucket_id  = "aarham-demo-bucket-3f9a1c2e"

terraform output                  # show all again later
terraform output -raw bucket_id   # ➕ just the value, handy in shell scripts / CI
```

| Block | Direction | Analogy (a function) |
|-------|-----------|----------------------|
| `variable` | **Input** to the configuration | Function parameters |
| `resource` | The work | Function body |
| `output` | **Output** from the configuration | Return value |

---

## 9. ➕ Locals & Data Sources

**Locals:** named values computed inside the config, to avoid repeating yourself:
```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"     # e.g. "shop-dev"
  common_tags = { Project = var.project, Env = var.environment, ManagedBy = "terraform" }
}
# use: local.name_prefix, local.common_tags
```

**Data sources:** **read** information about things that already exist (Terraform doesn't create or manage them):
```hcl
data "aws_caller_identity" "current" {}           # who am I?

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```
| `resource` | `data` |
|------------|--------|
| **Creates and manages** something | **Only reads** something that already exists |

---

## 10. ➕ State: How Terraform Remembers

After `apply`, Terraform writes **`terraform.tfstate`**: a JSON file that maps **your code** to the **real resources** (`aws_s3_bucket.bucket_name` = the real bucket `aarham-demo-bucket-3f9a1c2e`).

```
   .tf files (desired)  ──┐
                          ├──▶ terraform plan compares them ──▶ "+1 to add, ~1 to change, -0 to destroy"
   state + real cloud  ──┘
```

**Why it matters:**
- `plan` compares **code ↔ state ↔ reality** to work out the changes.
- Without state, Terraform doesn't know a bucket is "its own", and would try to create it again.

| Rule | Why |
|------|-----|
| **Never edit `terraform.tfstate` by hand** | Easy to corrupt. Use `terraform state list / show / rm` instead. |
| **Don't commit it to Git** | It can contain **secrets in plain text** (DB passwords, keys) |
| **Teams: use remote state** | Store it in an **S3 bucket** (with state locking) or HCP Terraform, so everyone shares one state and two people can't `apply` at the same time |

```hcl
terraform {
  backend "s3" {                                   # ➕ remote state
    bucket       = "my-company-tf-state"
    key          = "s3-demo/terraform.tfstate"
    region       = "us-east-1"
    use_lockfile = true                            # S3-native locking (older setups used a DynamoDB table)
  }
}
```

**➕ Drift:** someone changes a resource **manually in the console**. The next `terraform plan` notices the difference and offers to put it back as the code says. That's another reason to make **all** changes through code.

**`.gitignore` for Terraform:**
```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars          # if they contain secrets
crash.log
```
(Do **commit** `.terraform.lock.hcl`.)

---

## 11. The Core Workflow: init → fmt → validate → plan → apply → destroy

```
  write .tf ──▶ init ──▶ fmt ──▶ validate ──▶ plan ──▶ apply ──▶ (change code → plan → apply) ──▶ destroy
               (once)   (style)   (syntax)    (preview)  (build)                                    (tear down)
```

| Command | What it does | When |
|---------|--------------|------|
| `terraform init` | **Initialise** the folder: download providers into `.terraform/`, write the lock file, set up the backend | First time, and after adding a provider/module/backend |
| `terraform fmt` | **Rewrites** files into the standard style (indentation, aligned `=`) | Before committing. ➕ `fmt -check` only **checks** (use it in CI). `fmt -recursive` does sub-folders too. |
| `terraform validate` | Checks **syntax and internal consistency** (typos, wrong types, missing arguments). It doesn't contact AWS. | After editing |
| `terraform plan` | **Preview**: compares code vs. state vs. reality and lists every change. **Changes nothing.** | Always, before apply |
| `terraform apply` | Shows the plan again, asks **`yes`**, then **makes the changes** | To build or update |
| `terraform destroy` | **Deletes everything** this configuration manages (asks `yes` first) | Tear down labs and temporary environments |

> ⚠️ **Correction to the class notes:** "`terraform fmt`: automatic formatting **checks**". Plain `fmt` **changes** your files. Only `fmt -check` just checks.

### Reading a plan
```
Terraform will perform the following actions:

  # aws_s3_bucket.bucket_name will be created
  + resource "aws_s3_bucket" "bucket_name" {
      + bucket = "aarham-demo-bucket-3f9a1c2e"
      + arn    = (known after apply)
      + id     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```
| Symbol | Meaning |
|--------|---------|
| `+` | **Create** |
| `~` | **Update in place** (e.g. change a tag) |
| `-` | **Destroy** |
| `-/+` | **Destroy and re-create** (the argument can't be changed in place, e.g. a bucket's name) ⚠️ watch for these |

> 💡 **Analogy from class:** `terraform apply` = building a whole apartment block from the blueprints. `terraform destroy` = demolishing the whole block at once. And `plan` = the architect walking you through the drawings **before** any brick is laid.

### ➕ Other handy commands
```bash
terraform show                       # human-readable current state
terraform state list                 # every resource Terraform manages
terraform output                     # print outputs
terraform plan -out=tfplan           # save the plan...
terraform apply tfplan               # ...and apply EXACTLY that plan (used in CI)
terraform apply -auto-approve        # skip the "yes" prompt (CI only, never casually)
terraform destroy -target=aws_s3_bucket.bucket_name   # just one resource (use sparingly)
```

---

## 12. Hands-on: Create an S3 Bucket on AWS

**Goal:** a uniquely named, versioned, private S3 bucket, with the name and region as variables and the bucket ID as an output.

### `providers.tf`
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    random = {                                   # ➕ to generate a unique suffix
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### `variables.tf`
```hcl
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "bucket_prefix" {
  description = "Start of the bucket name: lowercase letters, numbers, hyphens"
  type        = string
  default     = "aarham-demo-bucket"
}
```

### `main.tf`
```hcl
# Random 8-hex-character suffix → avoids "BucketAlreadyExists"
resource "random_id" "suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "bucket_name" {
  bucket = "${var.bucket_prefix}-${random_id.suffix.hex}"   # e.g. aarham-demo-bucket-3f9a1c2e

  tags = {
    Name      = "Terraform demo"
    ManagedBy = "terraform"
  }
}

# ➕ Keep old versions of objects (undo accidental overwrites/deletes)
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.bucket_name.id
  versioning_configuration {
    status = "Enabled"
  }
}

# ➕ Make sure nothing in the bucket can ever be made public
resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.bucket_name.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### `outputs.tf`
```hcl
output "bucket_id" {
  value = aws_s3_bucket.bucket_name.id
}

output "bucket_arn" {
  value = aws_s3_bucket.bucket_name.arn
}
```

### Run it
```bash
cd s3-demo
terraform init          # downloads hashicorp/aws + hashicorp/random
terraform fmt
terraform validate      # Success! The configuration is valid.
terraform plan          # Plan: 4 to add, 0 to change, 0 to destroy.
terraform apply         # type: yes
# Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
# Outputs:
# bucket_id = "aarham-demo-bucket-3f9a1c2e"

# Verify with the CLI (class rule: CLI only, no console)
aws s3 ls | grep aarham-demo-bucket
aws s3api get-bucket-versioning --bucket "$(terraform output -raw bucket_id)"   # "Status": "Enabled"
echo "hello" > hello.txt && aws s3 cp hello.txt "s3://$(terraform output -raw bucket_id)/"

# Change something: add a tag in main.tf, then
terraform plan          # ~ update in-place: 1 to change ✅ (no re-create)

# Clean up (avoid charges)
aws s3 rm "s3://$(terraform output -raw bucket_id)" --recursive   # a bucket must be empty to delete
terraform destroy       # type: yes → Resources: 4 destroyed.
```
> ⚠️ Because versioning is on, `aws s3 rm` leaves **old object versions** behind, and `destroy` fails with `BucketNotEmpty`. For labs, either skip versioning or add `force_destroy = true` to the `aws_s3_bucket` resource, which lets Terraform empty the bucket on destroy. Never use that on real data.

---

## 13. ➕ Modules: Reusable Infrastructure

A **module** is a **folder of `.tf` files used as a building block**. Every Terraform folder is already a module (the "root module"). You call other modules to reuse them.

```
modules/
└── s3-bucket/          # variables.tf, main.tf, outputs.tf (like §12)
envs/
├── dev/main.tf
└── prod/main.tf
```
```hcl
# envs/dev/main.tf
module "logs_bucket" {
  source        = "../../modules/s3-bucket"   # or a registry module, e.g. "terraform-aws-modules/s3-bucket/aws"
  bucket_prefix = "shop-dev-logs"
}

output "logs_bucket_id" {
  value = module.logs_bucket.bucket_id        # read the module's outputs
}
```
**Same module, different inputs** → identical dev/staging/prod setups. It's the Terraform version of a Helm chart with values files ([L13 §15](13-Kubernetes-Troubleshooting-and-Helm.md#15-upgrade-rollback--environments)).

---

## 14. Troubleshooting & FAQs

| Problem | Cause | Fix |
|---------|-------|-----|
| `BucketAlreadyExists` | **S3 bucket names are globally unique across ALL AWS accounts** | Add a unique identifier: your name, account ID, or `random_id` (§12) |
| `InvalidBucketName` ➕ | Uppercase letters, underscores, or wrong length | Use 3–63 chars: **lowercase letters, numbers, hyphens** |
| `No valid credential sources found` / `ExpiredToken` | AWS credentials not set up, or wrong profile | `aws configure`, check `aws sts get-caller-identity`, `export AWS_PROFILE=...` |
| `AccessDenied` ➕ | Your IAM user lacks that permission | Ask for the needed policy (e.g. S3 permissions) |
| `Error: Inconsistent dependency lock file` / provider not found | Skipped `init`, or added a provider afterwards | `terraform init` (or `init -upgrade`) |
| `BucketNotEmpty` on destroy ➕ | Objects or old versions still inside | Empty it first, or `force_destroy = true` (labs only) |
| Plan wants to **re-create** (`-/+`) something ➕ | You changed an argument that can't be updated in place | Read the plan carefully. Avoid it for anything holding data. |
| Two people applied at once / state conflict ➕ | Local state, no locking | Remote backend with locking (§10) |

**FAQ (from class): why CLI only, without console access?** To build command-line skills, and because in real teams **all changes go through code** (Terraform + Git). Manual console changes cause drift ([§10](#10--state-how-terraform-remembers)).

---

## 15. Quick Revision Cheat Sheet

```
IAC           infra as machine-readable code in Git · declarative (WHAT) · idempotent · no ClickOps
TERRAFORM     HashiCorp · HCL · multi-cloud (AWS/Azure/GCP + thousands of providers) · plan before apply
              vs CloudFormation (AWS only) · vs Ansible (config INSIDE servers) · OpenTofu = open fork ➕
SETUP         terraform version · aws configure (keys → ~/.aws/credentials) · aws sts get-caller-identity
              provider reads the same creds · never put keys in .tf · least-privilege IAM
FILES         all *.tf in a folder = one config · providers.tf · variables.tf · main.tf · outputs.tf · terraform.tfvars
TERRAFORM {}  required_version · required_providers { aws = { source = "hashicorp/aws", version = "~> 6.0" } }
PROVIDER      plugin for one API · provider "aws" { region = var.aws_region } · downloaded by init
RESOURCE      resource "TYPE" "NAME" { args } · reference TYPE.NAME.ATTR → automatic dependency/order
VARIABLE      variable "x" { type, default, description, validation ➕, sensitive ➕ } → var.x
              set via default < TF_VAR_x < terraform.tfvars < *.auto.tfvars < -var / -var-file · no value → prompt
OUTPUT        output "x" { value = ... } · terraform output [-raw x]
LOCALS/DATA ➕ local.x = computed value · data "..." = READ existing things
STATE ➕      terraform.tfstate maps code ↔ real resources · don't edit · don't commit (secrets)
              teams → remote backend (S3 + locking) · drift = manual changes detected by plan
WORKFLOW      init → fmt (rewrites; -check only checks) → validate → plan → apply (yes) → destroy
PLAN SYMBOLS  + create · ~ update in place · - destroy · -/+ replace ⚠️
S3 LAB        bucket names GLOBALLY unique, lowercase · random_id suffix · versioning + public access block
              empty before destroy (or force_destroy for labs)
MODULES ➕    reusable folder of .tf · module "x" { source = ..., inputs } · module.x.output
ANALOGY       apply = build the apartment block from blueprints · destroy = demolish it · plan = review drawings
```

---

## 16. Likely Exam / Interview Questions

1. **What is Infrastructure as Code, and what are its benefits?** Defining infrastructure in machine-readable files instead of manual setup. It's repeatable, version-controlled, reviewable and fast, and keeps environments consistent.
2. **Declarative vs. imperative IaC? Which is Terraform?** Declarative describes the end state; imperative lists the steps. Terraform is declarative and idempotent.
3. **Why choose Terraform over CloudFormation?** It's multi-cloud (one tool and language for AWS, Azure, GCP and more), has a huge provider ecosystem, and has the `plan` preview and modules.
4. **Terraform vs. Ansible?** Terraform provisions infrastructure. Ansible configures software inside servers. They're often used together.
5. **What is a provider? Give an example.** A plugin that lets Terraform talk to a platform's API, e.g. `hashicorp/aws`, downloaded by `terraform init`.
6. **Explain `resource "aws_s3_bucket" "bucket_name"`.** The resource type comes from the AWS provider, and the local name is used to reference it in code (`aws_s3_bucket.bucket_name.id`).
7. **How does Terraform know the order to create resources in?** From references between resources. It builds a dependency graph automatically.
8. **Variables vs. outputs?** Variables are inputs (`var.x`), like function parameters. Outputs are values returned after apply, like return values.
9. **List the ways to set a variable's value, and which wins.** Default → `TF_VAR_name` env var → `terraform.tfvars` → `*.auto.tfvars` → `-var` / `-var-file` on the command line. Later ones override earlier ones.
10. **What does `version = "~> 6.0"` mean?** Any 6.x version, but not 7.0.
11. **Explain init, fmt, validate, plan, apply and destroy.** Set up and download providers / format the code / check the syntax / preview the changes / make the changes / delete everything.
12. **Does `terraform fmt` only check formatting?** No, it rewrites files. `terraform fmt -check` only checks, which is useful in CI.
13. **What do `+`, `~`, `-` and `-/+` mean in a plan?** Create, update in place, destroy, replace.
14. **What is the Terraform state file, and why shouldn't you commit it?** It maps the code to real resources and is used by plan. It can contain secrets, and teams need shared, locked remote state instead.
15. **Why might `terraform apply` fail with `BucketAlreadyExists`?** S3 bucket names are globally unique across all AWS accounts. Add a unique suffix.
16. **Does Terraform need the AWS CLI?** Not strictly. The AWS provider uses the same credentials (files or env vars). The CLI is a convenient way to configure and test them.
17. **What is drift, and how does Terraform handle it?** Real infrastructure that has been changed outside Terraform. `terraform plan` detects the difference and `apply` restores the coded state.
18. **What is a Terraform module, and why use one?** A reusable folder of Terraform config, called with different inputs per environment for consistent dev, staging and prod.
