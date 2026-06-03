# Automate AWS Infrastructure with Terraform

**An end-to-end cloud infrastructure automation pipeline using Terraform, Python, and Bash.**

> Built by Mishael Eluehike | June 2026

---

## What This Project Does

This project automates the full lifecycle of deploying and validating AWS cloud infrastructure without any manual clicking in the AWS console. A single Bash command initializes Terraform, provisions real AWS resources, and runs a Python validation script that confirms everything deployed correctly — printing a PASS/FAIL report at the end.

The goal was to replace a manual, error-prone deployment process with a repeatable, automated pipeline that any engineer could run consistently.

---

## Pipeline Overview

```
Run deploy.sh
     |
     v
[1/4] terraform init      -- Downloads the AWS provider plugin
     |
     v
[2/4] terraform plan      -- Previews what will be created
     |
     v
[3/4] terraform apply     -- Provisions S3 bucket + DynamoDB table on AWS
     |
     v
[4/4] python validate.py  -- Validates resources exist via boto3 API calls
     |
     v
Result: ALL CHECKS PASSED
```

---

## Tools and Technologies

| Tool | Version | Purpose |
|---|---|---|
| Terraform | v1.15.5 | Infrastructure as Code — provisioning AWS resources |
| Python | 3.14.5 | Automated validation of deployed resources |
| AWS CLI | 2.34.57 | Authentication and credential management |
| boto3 | 1.43.20 | AWS SDK for Python — used in the validation script |
| Git Bash | Latest | Bash scripting and pipeline orchestration |
| VS Code | Latest | Development environment |

---

## AWS Resources Provisioned

| Resource | Name | Service |
|---|---|---|
| S3 Bucket | devops-automation-bucket (random suffix appended by AWS) | Amazon S3 |
| DynamoDB Table | devops-automation-table | Amazon DynamoDB |

The S3 bucket uses `bucket_prefix` rather than a fixed name because AWS requires all bucket names to be globally unique. Using a prefix allows AWS to automatically append a random suffix, guaranteeing uniqueness without manual naming.

---

## Project Structure

```
infra-automation/
|
|-- main.tf          <- Defines the AWS provider, region, and resources to create
|-- variables.tf     <- Declares input variables (e.g. AWS region)
|-- outputs.tf       <- Returns resource names and region to the Python validator
|-- validate.py      <- Python script that checks each resource exists via boto3
|-- deploy.sh        <- Bash script that runs the full pipeline end to end
```

---

## How It Works: Step by Step

### Step 1 — IAM Setup and Authentication

A dedicated IAM user called `terraform-user` was created in AWS instead of using root credentials. The user was granted the necessary permissions and configured via the AWS CLI. Running `aws sts get-caller-identity` confirms the correct identity is authenticated before any infrastructure is touched.

![AWS IAM Identity Confirmed](/aws_terraform_user_identity.png)

---

### Step 2 — Installing the Toolchain

Python, Terraform, and the AWS CLI were installed and verified via PowerShell before starting the project.

![Tool Versions Confirmed](/py_terraform_aws_version.png)

---

### Step 3 — Writing the Terraform Files

Three Terraform files define the infrastructure:

**variables.tf** declares the AWS region as an input variable so it can be changed without editing the main configuration.

**main.tf** defines the AWS provider, its version, the deployment region, and the two resources to create: an S3 bucket and a DynamoDB table. Both are tagged with Environment, ManagedBy, and Name labels for tracking.

**outputs.tf** exposes the S3 bucket name, DynamoDB table name, and AWS region as output values so the Python validation script can consume them programmatically.

![Terraform Files in VS Code](/display_of_my_terraform_files.png)

---

### Step 4 — Deploying the Infrastructure

The Terraform workflow runs in three commands:

- `terraform init` downloads the AWS provider plugin and initializes the backend
- `terraform plan` previews the exact resources that will be created before anything is deployed
- `terraform apply` provisions the resources in AWS

The plan output showed 2 resources to add (the S3 bucket and DynamoDB table) with 0 changes and 0 destroys. After apply, both resources were confirmed created with the correct names and configuration.

![Terraform Init and Plan](/terraform_init_plan_apply1.png)
![Terraform Apply Complete](/terraform_init_plan_apply2.png)

---

### Step 5 — Validating with Python
(/pyhtonValidation_script.png)

Instead of manually checking the AWS console, a Python script using the boto3 SDK validates each resource programmatically and produces a PASS/FAIL report.

The `validate_s3_bucket` function calls `s3.head_bucket(Bucket=bucket_name)`. If the bucket exists and is accessible the call succeeds. If not, the exception block catches the failure and reports FAIL.

The `validate_dynamodb_table` function calls `dynamodb.describe_table(TableName=table_name)` and checks that the table status is ACTIVE.

The script receives the bucket name, table name, and region directly from Terraform outputs at runtime so there is no hardcoding.

![Python Validation Report](/python_validation_checks.png)

---

### Step 6 — Orchestrating Everything with Bash

A single Bash script (`deploy.sh`) ties the entire pipeline together. Running `./deploy.sh` executes all four stages in sequence and prints a structured status report at each step.

`set -e` is used at the top of the script to ensure that if any step fails, the pipeline stops immediately rather than continuing with a broken state.

![Full Pipeline Running](/using_bash_to_automate_evrything.png)

---

## Key Concepts Demonstrated

| Concept | Where it appears |
|---|---|
| Infrastructure as Code | Terraform files define all resources declaratively |
| Least-privilege IAM | Dedicated terraform-user instead of root credentials |
| Automated validation | Python boto3 script replaces manual console checks |
| Pipeline orchestration | Single Bash script runs the full deploy-validate workflow |
| Output passing | Terraform outputs feed directly into the Python validator |
| Error handling | set -e stops the pipeline on any failure |
| Resource tagging | All AWS resources tagged with Environment, ManagedBy, Name |

---

## How to Run This Yourself

### Prerequisites

- AWS account with an IAM user configured via `aws configure`
- Terraform installed: [terraform.io/downloads](https://developer.hashicorp.com/terraform/downloads)
- Python 3.x installed with boto3: `pip install boto3`
- Git Bash installed (Windows) or any Bash shell (Linux/Mac)

### Steps

```bash
# Clone the repo
git clone https://github.com/E-Mishael/aws-terraform-automation
cd aws-terraform-automation/infra-automation

# Make the deploy script executable
chmod +x deploy.sh

# Run the full pipeline
./deploy.sh
```

Expected output:
```
DevOps Automation Pipeline
==========================
[1/4] Initializing Terraform...
[2/4] Planning infrastructure...
[3/4] Deploying infrastructure...
[4/4] Validating infrastructure...

=== Infrastructure Validation Report ===
[PASS] S3 bucket exists and is accessible.
[PASS] DynamoDB table is ACTIVE

Result: ALL CHECKS PASSED
Pipeline complete!
```

### Cleaning Up

To destroy all AWS resources created by this project and avoid charges:

```bash
terraform destroy
```

Type `yes` when prompted to confirm.

---

## Lessons Learned

**Naming S3 buckets:** AWS S3 bucket names must be globally unique across all AWS accounts worldwide. Using `bucket_prefix` and letting AWS append a random suffix is a cleaner solution than manually generating unique names.

**IAM least privilege:** Using a dedicated IAM user for Terraform instead of root credentials is the correct security practice. In a production environment, the policy would be scoped down further to only the specific S3 and DynamoDB permissions required.

**set -e in Bash:** Without this flag a Bash script will continue running even after a command fails. For a deployment pipeline this could mean applying broken infrastructure or skipping validation. Always use `set -e` in automation scripts.

**Terraform state:** Terraform tracks what it has deployed in a state file. Running the pipeline a second time shows 0 changes because Terraform compares the state file against your configuration and finds they already match.

---

## About

**Mishael Eluehike**
Cybersecurity Analyst | CompTIA Security+ | Cisco CyberOps Associate | BSc Computer Science (First-Class Honours)

[LinkedIn](https://www.linkedin.com/in/mishael-eluehike-791696259) | [GitHub](https://github.com/E-Mishael) | [Cybersecurity Homelab](https://github.com/E-Mishael/cybersecurity-homelab)

---

*This project is part of an ongoing series building practical DevOps and cloud automation skills. More projects coming.*
