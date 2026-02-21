# EKS Terraform GitHub Actions

Production-ready Amazon EKS cluster deployment using Terraform with CI/CD automation via GitHub Actions and Jenkins.

## Overview

This repository provisions a complete AWS EKS infrastructure including:

- **VPC** with public and private subnets across multiple availability zones
- **EKS Cluster** (v1.33) with private endpoint access
- **Managed Node Groups** - On-Demand and Spot instances for cost optimization
- **Essential Add-ons** - VPC CNI, CoreDNS, kube-proxy, EBS CSI Driver
- **IAM Roles & Policies** - Cluster role, node group role, and OIDC provider
- **Security Groups** - Network isolation for the EKS cluster

## Architecture

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                         VPC                             │
                    │                    (10.16.0.0/16)                       │
                    │                                                         │
                    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
                    │   │   Public    │  │   Public    │  │   Public    │    │
                    │   │  Subnet 1   │  │  Subnet 2   │  │  Subnet 3   │    │
                    │   │ ap-south-1a │  │ ap-south-1b │  │ ap-south-1c │    │
                    │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
                    │          │                │                │           │
                    │          └────────────────┼────────────────┘           │
                    │                           │                            │
                    │                    ┌──────┴──────┐                     │
                    │                    │   Internet  │                     │
                    │                    │   Gateway   │                     │
                    │                    └──────┬──────┘                     │
                    │                           │                            │
                    │                    ┌──────┴──────┐                     │
                    │                    │     NAT     │                     │
                    │                    │   Gateway   │                     │
                    │                    └──────┬──────┘                     │
                    │                           │                            │
                    │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
                    │   │   Private   │  │   Private   │  │   Private   │    │
                    │   │  Subnet 1   │  │  Subnet 2   │  │  Subnet 3   │    │
                    │   │ ap-south-1a │  │ ap-south-1b │  │ ap-south-1c │    │
                    │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
                    │          │                │                │           │
                    │          └────────────────┼────────────────┘           │
                    │                           │                            │
                    │                  ┌────────┴────────┐                   │
                    │                  │   EKS Cluster   │                   │
                    │                  │   (Private)     │                   │
                    │                  └────────┬────────┘                   │
                    │                           │                            │
                    │            ┌──────────────┴──────────────┐             │
                    │            │                             │             │
                    │   ┌────────┴────────┐         ┌─────────┴────────┐    │
                    │   │  On-Demand Node │         │   Spot Node      │    │
                    │   │     Group       │         │     Group        │    │
                    │   │  (t3.small)     │         │   (t3.small)     │    │
                    │   └─────────────────┘         └──────────────────┘    │
                    │                                                        │
                    └────────────────────────────────────────────────────────┘
```

## Project Structure

```
.
├── eks/                          # Main Terraform configuration
│   ├── main.tf                   # Module invocation
│   ├── variables.tf              # Variable declarations
│   ├── backend.tf                # S3 backend & provider config
│   └── dev.tfvars                # Development environment values
├── module/                       # Reusable Terraform modules
│   ├── vpc.tf                    # VPC, subnets, route tables, NAT
│   ├── eks.tf                    # EKS cluster and node groups
│   ├── iam.tf                    # IAM roles and policies
│   ├── gather.tf                 # Data sources
│   └── variables.tf              # Module variables
├── .github/workflows/
│   └── validate-terraform.yml    # GitHub Actions CI pipeline
├── Jenkinsfile                   # Jenkins CI/CD pipeline
└── README.md
```

## Prerequisites

- **AWS Account** with appropriate permissions
- **Terraform** >= 1.9.3
- **AWS CLI** configured with credentials
- **S3 Bucket** for Terraform state (update `backend.tf`)
- **DynamoDB Table** for state locking (named `Lock-Files`)

## Configuration

Key configuration in `eks/dev.tfvars`:

| Parameter | Default Value | Description |
|-----------|---------------|-------------|
| `aws-region` | `ap-south-1` | AWS region for deployment |
| `cluster-version` | `1.33` | EKS Kubernetes version |
| `ondemand_instance_types` | `t3.small` | Instance type for on-demand nodes |
| `spot_instance_types` | `t3.small` | Instance type for spot nodes |
| `endpoint-public-access` | `false` | Public API endpoint (disabled for security) |
| `endpoint-private-access` | `true` | Private API endpoint |

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/pratyush934/EKS-Terraform-GitHub-Actions.git
cd EKS-Terraform-GitHub-Actions
```

### 2. Configure Backend

Update `eks/backend.tf` with your S3 bucket details:

```hcl
backend "s3" {
  bucket         = "your-terraform-state-bucket"
  region         = "ap-south-1"
  key            = "eks/terraform.tfstate"
  dynamodb_table = "Lock-Files"
  encrypt        = true
}
```

### 3. Initialize and Deploy

```bash
cd eks

# Initialize Terraform
terraform init

# Review the plan
terraform plan -var-file=dev.tfvars

# Apply the configuration
terraform apply -var-file=dev.tfvars -auto-approve
```

### 4. Configure kubectl

```bash
aws eks update-kubeconfig --name dev-ap-medium-eks-cluster --region ap-south-1
```

## CI/CD Pipelines

### GitHub Actions

Automatically runs on every push to validate Terraform code:

- Terraform Init
- Terraform Format Check
- Terraform Validate
- Terraform Plan

Configure secrets in your GitHub repository:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

### Jenkins Pipeline

Supports three actions via parameters:
- **plan** - Preview infrastructure changes
- **apply** - Deploy infrastructure
- **destroy** - Tear down infrastructure

## EKS Add-ons Included

| Add-on | Version | Purpose |
|--------|---------|---------|
| VPC CNI | v1.20.0-eksbuild.1 | Pod networking |
| CoreDNS | v1.12.2-eksbuild.4 | DNS resolution |
| kube-proxy | v1.33.0-eksbuild.2 | Network proxy |
| EBS CSI Driver | v1.46.0-eksbuild.1 | Persistent storage |

## Cleanup

To destroy all resources:

```bash
cd eks
terraform destroy -var-file=dev.tfvars -auto-approve
```

## Cost Optimization

This setup uses cost-effective configurations:
- **t3.small instances** - Free tier eligible
- **Spot instances** - Up to 90% savings for fault-tolerant workloads
- **Private endpoint only** - No additional NAT costs for API access

## Security Features

- Private EKS endpoint (no public API access)
- Nodes deployed in private subnets
- Security groups restricting cluster access
- OIDC provider for pod-level IAM roles
- Encrypted Terraform state

## License

This project is licensed under the Apache 2.0 License. See the [LICENSE](LICENSE) file for details.
