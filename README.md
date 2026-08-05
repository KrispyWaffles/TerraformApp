# TerraformApp

This project is my hands-on introduction to Terraform and AWS. Rather than just
reading docs or watching tutorials, I'm using it to actually provision
infrastructure — starting small (a single S3 bucket) and building up to a full
3-tier AWS application (VPC, EC2, RDS, S3, CI/CD). The goal is to understand how
real apps get built and hosted on cloud infrastructure by doing it myself, not
just following along.

## Status

**Done:** Terraform installed and AWS CLI configured. Ran a full practice loop —
declared an S3 bucket in `main.tf`, created it with `terraform apply`, verified
it existed, then tore it down with `terraform destroy`. Confirmed the full
create/destroy cycle in AWS CloudTrail.

**Not started yet:** VPC, EC2, RDS, the app-tier S3 bucket, and the CI/CD
pipeline. App logic for the EC2 tier is TBD.

## Prerequisites

- macOS with [Homebrew](https://brew.sh) installed
- [Git](https://git-scm.com/) installed, to clone this repo
- Terraform, installed via:
  ```bash
  brew tap hashicorp/tap && brew install hashicorp/tap/terraform
  ```
- AWS CLI installed and configured (`aws configure`) with an IAM user's access
  keys — not root, and not just a console login
- An AWS account with permissions to create the resources in this repo
- Confirm both are working:
  ```bash
  terraform version
  aws sts get-caller-identity
  ```

## How to run it

1. Configure AWS credentials (one-time):
   ```bash
   aws configure
   ```
2. Create a `.gitignore` (already done in this repo) to protect state files
3. Write `main.tf` declaring the resource(s) to create
4. Run the loop:
   ```bash
   terraform init      # downloads the AWS provider plugin
   terraform plan      # preview what will be created — read this output
   terraform apply     # type "yes" to actually create it
   ```
5. Verify the resource exists (`aws s3 ls`, or check CloudTrail)
6. Tear it down when done:
   ```bash
   terraform destroy   # type "yes" — don't skip this
   ```

## Architecture (planned)

- **VPC** — the network boundary everything else lives inside, split into a
  public and private subnet
- **EC2** (public subnet) — runs the application, reachable from the internet
- **RDS** (private subnet) — the application's database, reachable only from
  EC2, never directly from the internet
- **S3** — storage for app assets (files, images, etc.), separate from the
  practice bucket used earlier
- **CI/CD** — GitHub Actions runs `terraform plan` on every PR, `terraform
  apply` on merge to main
