# BlakTail AWS deploy

Public buyer artifacts for self-hosting BlakTail in Sydney (`ap-southeast-2`).
This repository is Terraform and instructions only. There is no hosted product
and no marketplace image: you build the images from the public source repo and
run them in your own AWS account. Your organisation holds the data, keys and logs.

- Source: <https://github.com/jusso-dev/BlakTail>
- Landing site: set `PUBLIC_SITE_URL` when you publish it
- License: Apache-2.0

## What it creates

VPC (default VPC for the first deploy; move to a dedicated VPC for prod),
Application + Network load balancers, ECS Fargate (console, coordinator, relay),
RDS PostgreSQL, ECR repositories, Secrets Manager secrets and security groups —
all pinned to `ap-southeast-2`. The relay refuses non-Australian regions.

## Prerequisites

- Terraform >= 1.6, AWS CLI v2, Docker
- An AWS account with a Sydney default VPC (first deploy) or your own VPC plan
- The source repo checked out (images build from source, nothing is pulled prebuilt)

```sh
git clone https://github.com/jusso-dev/BlakTail && cd BlakTail
git clone https://github.com/jusso-dev/blaktail-aws-deploy.git
cd blaktail-aws-deploy/terraform
cp terraform.tfvars.example terraform.tfvars
```

Edit `terraform.tfvars`:

1. Leave `region` at `ap-southeast-2`. Anything else fails validation.
2. Paste `coord_tls_cert_pem` / `coord_tls_key_pem` (TLS chain the coordinator presents to agents).
3. Leave image URIs empty for the first apply — Terraform creates the ECR repos, then you build and push.
4. Leave `console_acm_certificate_arn` empty for an HTTP-only first launch; add an ACM ARN for HTTPS.

## Deploy

```sh
terraform init
terraform apply -var-file=terraform.tfvars   # coord replicas stay 0 for migration
```

Build and push images from the source repo (prints digests for the record):

```sh
cd ../BlakTail
./scripts/aws-e2e/build-images.sh "$(terraform -chdir=../blaktail-aws-deploy/terraform output -json ecr_repos)"
```

Set the three image variables in `terraform.tfvars`, run the migration task, then scale
up:

```sh
terraform apply -var-file=terraform.tfvars
terraform output console_url coord_endpoint relay_endpoint
```

Open `console_url`, register the first owner, and enrol agents with `blaktaild up`
against `coord_endpoint`. Approve each device fingerprint in the browser.

## Hardening for production

- Set `db_multi_az = true`, `coord_desired_count >= 2`, ACM cert + HTTPS console URL.
- Restrict ALB ingress to your office/VPN CIDRs. Do not use `0.0.0.0/0` without a reason.
- Keep TLS proxy, logs, backups and DNS in Australia — residency is a deployment duty.
- Never commit `terraform.tfvars`, PEM keys or `terraform.tfstate` to git.

## Destroy

```sh
terraform destroy -var-file=terraform.tfvars
```

Every run-scoped resource is tagged `Project = blaktail`. Verify in the console that
no load balancers, RDS instances or ECR images remain.
