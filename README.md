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

Built the VPC (public/private subnets, internet gateway, route table) — left
running, since none of these pieces cost anything hourly. Built an EC2 instance
and security group in the public subnet, verified it in CloudTrail, then
destroyed just those two resources with a targeted `terraform destroy`, leaving
the VPC intact.

Added a second private subnet (RDS requires a DB subnet group spanning at least
two Availability Zones) and built RDS alongside a rebuilt EC2 instance, so the
security boundary between them would mean something. RDS's security group only
allows its port from the EC2 security group, not from any IP range. Verified
that boundary two ways: in CloudTrail (the `AuthorizeSecurityGroupIngress` event
shows the RDS security group authorized specifically against the EC2 security
group's ID) and empirically (`nc` from my own laptop to the RDS endpoint timed
out, since my IP isn't in the allow list). Destroyed both EC2 and RDS afterward
with a targeted `terraform destroy`, again leaving the VPC intact.

Built the app-tier S3 bucket, fully private (all four public-access-block
settings on), plus an IAM role, policy, and instance profile scoped to exactly
that bucket — `GetObject`/`PutObject`/`ListBucket` only, nothing broader. This
is unlike EC2/RDS: S3 isn't inside the VPC at all, so access is controlled by
IAM identity rather than network location (no security group applies here).
Left the bucket and IAM resources running (no hourly cost); EC2 and RDS remain
torn down until a deliberate end-to-end pass attaches the instance profile to
a running EC2 instance and proves it can actually use the role.

**Not started yet:** the CI/CD pipeline, and the full end-to-end proof that EC2
can assume its role and read/write the S3 bucket. App logic for the EC2 tier
is TBD.

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
  practice bucket used earlier. Fully private; access granted only via an IAM
  role/instance profile meant for the EC2 tier, not a security group — S3
  lives outside the VPC entirely
- **CI/CD** — GitHub Actions runs `terraform plan` on every PR, `terraform
  apply` on merge to main

## Lessons learned

- `terraform destroy` with no arguments destroys **everything** in the state
  file, not just what you're currently working on. Learned this the hard way
  when destroying the EC2 instance almost took the VPC down with it. Use
  `-target` to scope a destroy to specific resources — but treat that as an
  exception for iterating on one layer at a time, not the default workflow.
  A real team's state file usually represents a shared environment, so an
  untargeted destroy there has a much bigger blast radius than "redo a lab."
- For anything holding real data (RDS, in particular), always confirm a final
  snapshot is enabled before destroying — an EC2 instance is annoying to
  recreate, but a database without a snapshot means the data itself is gone.
- One syntax mistake early in `main.tf` (an unterminated string) cascaded into
  dozens of unrelated-looking `terraform validate` errors for every line after
  it. When facing a wall of errors, fix the *first* one reported and re-check
  before assuming the rest are real — most of them were fallout, not separate
  bugs.
