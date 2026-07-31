# Getting Started — Terraform, from Zero

This project is Portfolio Project #1 (see `~/workspace/cloud-camp/PROJECTS.md`): a
Terraform-provisioned 3-tier AWS app with CI/CD. This doc is the on-ramp for someone who
has never used Terraform before.

---

## Terraform in plain terms

- **Provider** — the plugin that lets Terraform talk to a cloud's API (AWS, Azure, GCP, etc.)
- **Resource** — a thing you want to exist (an S3 bucket, an EC2 instance, a VPC). You describe it in a `.tf` file, Terraform makes it real.
- **State** (`terraform.tfstate`) — Terraform's memory of what it already created. This is how it knows what to change or destroy later. Never hand-edit this file. Never commit it to git.
- **The loop:**
  - `terraform init` — downloads the provider plugins
  - `terraform plan` — shows what *would* change, without touching anything
  - `terraform apply` — actually makes the change
  - `terraform destroy` — tears it all down

That's the whole mental model. Everything past this is just learning which resource blocks exist.

---

## Step 1 — Install Terraform

```bash
brew tap hashicorp/tap && brew install hashicorp/tap/terraform
terraform version
```

## Step 2 — Confirm AWS credentials work

```bash
aws sts get-caller-identity
```
Should return your account ID and IAM user ARN. If this fails, fix AWS CLI auth before touching Terraform.

## Step 3 — `.gitignore` before anything else

State files and the `.terraform/` cache should never hit GitHub — state can contain
sensitive resource data. Create `.gitignore` in the repo root:
```
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
crash.log
```

## Step 4 — Prove the loop works (don't skip this)

One file, one resource, run the full loop once so you trust it before building anything
real.

`main.tf`:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "first_bucket" {
  bucket = "wess-terraform-first-bucket-<add-random-suffix>"
}
```
(S3 bucket names are globally unique — add initials + date to the suffix.)

```bash
terraform init
terraform plan     # read this output — it tells you exactly what it's about to create
terraform apply    # type "yes" when prompted
```
Verify it exists (`aws s3 ls` or the console), then:
```bash
terraform destroy  # type "yes" — don't skip this, avoid leaving stray resources around
```

---

## Real project scope (after the loop feels natural)

Build in this order, one resource type at a time — don't write the whole thing in one file on day one:

1. **VPC** — public + private subnets, internet gateway, route tables
2. **EC2** — launched into the public subnet
3. **RDS** — database in the private subnet
4. **S3** — bucket for app assets/storage
5. **CI/CD** — GitHub Actions workflow that runs `terraform plan` on PR, `terraform apply` on merge to main

Definition of Done for this project lives in `~/workspace/cloud-camp/PROJECTS.md` —
README + architecture diagram + IaC + CI/CD + proof it runs + torn down when not actively
iterating.

## Cost discipline

- Nothing in this project is fully free-tier by default once you get to RDS/NAT Gateway — check before leaving anything running overnight
- Run `terraform destroy` at the end of any session where you're not actively iterating
- Set a billing alarm on the AWS account if you haven't already
