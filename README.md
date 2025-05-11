# ALTCHA Sentinel AWS Deployment

This CloudFormation template deploys [ALTCHA Sentinel](https://altcha.org/) as an ECS Fargate service with an Application Load Balancer (ALB) on AWS.

## Overview

The template creates the following AWS resources:
- **ECS Fargate Service** running on ARM64 architecture
- **Application Load Balancer** with HTTPS support (when certificate provided)
- **VPC with public subnets** across two Availability Zones
- **EFS storage** mounted to the container at `/data`
- **All required IAM roles and security groups**

## Prerequisites

1. AWS CLI installed and configured
2. (Optional) Domain name and ACM certificate if using custom domain

## Deployment

### Basic Deployment (without custom domain)
```bash
aws cloudformation deploy \
  --template-file altcha-sentinel-aws-ecs.yml \
  --stack-name altcha-sentinel-stack \
  --capabilities CAPABILITY_IAM
```

### Deployment with Custom Domain
```bash
aws cloudformation deploy \
  --template-file altcha-sentinel-aws-ecs.yml \
  --stack-name altcha-sentinel-stack \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      DomainName=your.domain.com \
      CertificateArn=arn:aws:acm:region:account:certificate/id
```

## Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ImageURI` | `public.ecr.aws/n6m6b4n8/altcha-org/sentinel:<version>` | Container image URI |
| `ServiceName` | `altcha-sentinel` | Name for the ECS service |
| `DomainName` | (empty) | Optional custom domain name |
| `CertificateArn` | (empty) | ACM certificate ARN for HTTPS |
| `TaskCPU` | `2048` | CPU units (1024 = 1 vCPU) |
| `TaskMemory` | `4096` | Memory in MiB |

## Available Task Sizes

### CPU Options
- 1024 (1 vCPU)
- 2048 (2 vCPU)
- 4096 (4 vCPU)

### Memory Options
- 2048 MB
- 3072 MB
- 4096 MB
- 5120 MB
- 6144 MB
- 7168 MB
- 8192 MB
- 16384 MB

Note: Memory must be compatible with CPU choice.

## Accessing the Service

After deployment, the service will be available at:
- The ALB DNS name (if no custom domain specified)
- Your custom domain (if configured)

The output will display the service URL.

## Storage

The deployment includes an EFS volume mounted at `/data` in the container with encryption enabled.

## Cleaning Up

To delete the stack and all resources:
```bash
aws cloudformation delete-stack --stack-name altcha-sentinel-stack
```

## Outputs

The stack provides the following outputs:
- `ALBDNSName`: DNS name of the load balancer
- `ServiceURL`: Full URL to access the service
- `TaskSize`: Configured CPU and memory allocation