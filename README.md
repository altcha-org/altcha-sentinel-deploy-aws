# ALTCHA Sentinel AWS Deployment

This CloudFormation template deploys [ALTCHA Sentinel](https://altcha.org/) as an ECS Fargate service with an Application Load Balancer (ALB) on AWS.

## Overview

The template creates the following AWS resources:
- **ECS Fargate Service** running on ARM64 architecture
- **Application Load Balancer** with HTTPS support (when certificate provided)
- **VPC with public subnets** across two Availability Zones
- **(Optional) EFS storage** mounted to the container at `/data`
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
| `EnablePersistence` | `false` | Attach an EFS-backed volume (at `/data`) that survives task recreation |
| `DatabaseUrl` | (empty) | Connection string for an external database |
| `DatabaseUrlVarName` | `POSTGRES_URL` | Which env var `DatabaseUrl` is passed as - one of `POSTGRES_URL`, `MYSQL_URL`, `MARIADB_URL`, `MSSQL_URL`, `LIBSQL_URL` |
| `SecretSeed` | (empty) | `SECRET_SEED` - deterministically derives `JWT_SECRET`, `ALTCHA_HMAC_SECRET`, `CODE_CHALLENGE_SECRET`, `EXOTDB_HMAC_SECRET` and `HASHING_SALT`, kept stable across redeploys |
| `NodeId` | (empty) | `NODE_ID` - unique node identifier |
| `BaseUrl` | (empty) | `BASE_URL` - used for generating absolute URLs |
| `AllowedHosts` | (empty) | `ALLOWED_HOSTS` - comma-separated hostnames (supports wildcards) |
| `ExtraEnvName1/2/3`, `ExtraEnvValue1/2/3` | (empty) | Up to 3 arbitrary additional env var name/value pairs |

Any parameter left at its default (empty string) is simply not passed to the container — ALTCHA
Sentinel falls back to its own defaults/auto-generated values for that variable. See ALTCHA's
[environment variable reference](https://altcha.org/docs/v2/sentinel/advanced/env/) for the full list
of variables it understands.

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

By default, no volume is attached and the container relies on its local, ephemeral task storage.
This is the recommended setting when ALTCHA Sentinel is configured with an external database, since
no local persistence is needed in that case.

This template always runs a single task (`DesiredCount: 1`). If you're instead using a local,
file-based database (e.g. SQLite), deploy with `EnablePersistence=true` to mount an encrypted EFS
volume at `/data`:

```bash
aws cloudformation deploy \
  --template-file altcha-sentinel-aws-ecs.yml \
  --stack-name altcha-sentinel-stack \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides EnablePersistence=true
```

Unlike a per-task volume, EFS exists independently of any task's lifecycle, so `/data` survives
task recreation — deployments, scaling events, and crash restarts. When persistence is enabled, the
service also switches to a "recreate" deployment strategy (the running task is stopped before its
replacement starts) so two tasks never mount the volume and write to a local database file at the
same time.

This setup assumes a single task. Scaling `DesiredCount` beyond 1 would have every task share the
same EFS mount and directory, which is unsafe for a local file-based database — use an external
database instead if you need to run more than one task.

## Environment Variables

Pass container environment variables using `--parameter-overrides` on the same `aws cloudformation
deploy` command as any other parameter. For example, to point ALTCHA Sentinel at an external Postgres
database (and skip the EFS volume, since it's not needed once there's an external database):

```bash
aws cloudformation deploy \
  --template-file altcha-sentinel-aws-ecs.yml \
  --stack-name altcha-sentinel-stack \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
      DatabaseUrl="postgresql://user:password@your-db-host:5432/altcha_sentinel" \
      DatabaseUrlVarName=POSTGRES_URL
```

By default, Sentinel auto-generates its required secrets (`JWT_SECRET`, `ALTCHA_HMAC_SECRET`,
`CODE_CHALLENGE_SECRET`, `EXOTDB_HMAC_SECRET`, `HASHING_SALT`) with random values on startup, which
makes them unstable across redeploys and invalidates existing sessions/tokens every time the task
restarts. Set `SecretSeed` to have Sentinel derive all of them deterministically from one fixed value
instead:

```bash
aws cloudformation deploy \
  --template-file altcha-sentinel-aws-ecs.yml \
  --stack-name altcha-sentinel-stack \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides SecretSeed="$(openssl rand -hex 32)"
```

For anything not covered by a named parameter, use one of the three generic slots:

```bash
--parameter-overrides ExtraEnvName1=SOME_VAR ExtraEnvValue1=some-value
```

Parameters holding secrets (`DatabaseUrl`, `SecretSeed`, `ExtraEnvValue1/2/3`) are declared `NoEcho`,
so CloudFormation masks them in the console and in `describe-stacks` output — but they are still
passed to `deploy` as plain CLI arguments, which is worth knowing before running this from a shared
shell history or CI log.

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