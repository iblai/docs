# AMI-Based Launch Pipeline

Automated pipeline for launching isolated staging environments from pre-built AMIs, running E2E tests, and tearing down — all via GitHub Actions.

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub Actions Workflow                       │
│                                                                 │
│  ┌──────────────┐   ┌──────────────┐                           │
│  │ Build        │   │ Launch EC2   │                           │
│  │ Playwright   │   │ from AMI     │   (parallel)              │
│  │ Image (OCIR) │   │ + Service    │                           │
│  └──────┬───────┘   │   Update     │                           │
│         │           └──────┬───────┘                           │
│         │                  │                                    │
│         └────────┬─────────┘                                    │
│                  ▼                                              │
│         ┌──────────────┐                                       │
│         │ Run Playwright│  (OCI Container Instances             │
│         │ Tests         │   hit mentorai.stgX.iblai.org)        │
│         └──────┬───────┘                                       │
│                ▼                                                │
│         ┌──────────────┐                                       │
│         │ Terminate    │                                       │
│         │ EC2 Instance │                                       │
│         └──────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

## Architecture

Each staging environment (stg1–stg4) has permanent AWS infrastructure:

| Resource | Purpose | Persists between launches |
|----------|---------|--------------------------|
| VPC + Subnets | Networking | Yes |
| ALB + Target Group | Load balancer with TLS termination | Yes |
| ACM Certificates | SSL for `*.stgX.iblai.org` | Yes |
| Route53 Records | DNS → ALB | Yes |
| Security Groups | Firewall rules | Yes |
| S3 Buckets | Media + static storage | Yes |
| **EC2 Instance** | **Platform server** | **No — ephemeral** |

The EC2 is the only component created and destroyed per pipeline run. Everything else is pre-provisioned via Terraform and reused.

## Pre-Built AMI Contents

Each AMI is a snapshot of a fully configured staging environment:

- **OS**: Ubuntu 22.04 with Docker, pyenv, Python 3.11.8, AWS CLI
- **Platform CLI**: iblai-cli-ops installed via iblai-prod-images
- **Services** (Docker containers):
  - iblai-dm-pro (Django, PostgreSQL, Redis, Celery, Langfuse, ClickHouse, MinIO)
  - iblai-edx-pro (LMS, CMS, MySQL, MongoDB, Redis, Elasticsearch, MFE)
  - Auth SPA, Agent SPA, Skills SPA
  - Nginx reverse proxy
- **Data**: Test platforms, users, RBAC, analytics views pre-seeded
- **Config**: S3 buckets, AWS credentials, TimescaleDB enabled

## Pipeline Steps — Detailed

### Step 1: Build Playwright Image

**What**: Builds a Docker image containing the Playwright test suite from the mentorai repo and pushes it to Oracle Cloud Container Registry (OCIR).

**Where**: GitHub Actions runner (ubuntu-latest) → OCIR

**Image**: `iad.ocir.io/idcwyla5j5cr/ibl-agent-playwright:{tag}`

**Contents**: Playwright browsers (Chromium, Firefox, WebKit), test specs from `e2e/journeys/`, page objects, test utilities, AWS CLI for S3 log upload.

**Caching**: Checks if image with the same tag already exists — skips build if so.

### Step 2: Launch EC2 from AMI

**What**: Provisions a fresh EC2 instance from the pre-built AMI into the existing VPC/subnet/security group.

**How** (via boto3 in the iblai-infra-cli tool):
1. `ec2:RunInstances` with the AMI ID, instance type (t3.2xlarge), 200GB gp3 volume
2. Wait for instance to enter `running` state
3. Get public IP address

**Security**: The workflow opens port 22 on the security group for the GitHub Actions runner IP, and revokes it after completion (always, even on failure).

### Step 3: Service Update (Ansible)

**What**: Ensures all services on the launched EC2 are running and configured correctly.

**Tool**: `iblai infra service-update --host <IP>` from [iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)

**Ansible Playbook** (`service_update_playbook.yml`, 2 roles):

#### Role: ibl_cli_ops
- Installs latest `iblai-prod-images` package from `iblai/iblai-prod-images@main`
- This pins all container image versions and includes ibl-cli-ops

#### Role: ibl_service_update
1. **Restore postgres data dir ownership** to uid 999 (fixes chown from pre-tasks)
2. **ECR login** — authenticate Docker with AWS ECR (using server's existing AWS creds)
3. **Save platform config** — `ibl config save` regenerates all compose files
4. **Save edX tutor config** — `ibl tutor config save`
5. **Ensure edX running** — `ibl edx start -d`
6. **Wait for LMS** — curl `localhost:8600/heartbeat` (40 retries × 15s)
7. **Ensure DM containers running** — `docker compose up -d` in background (avoids timeout on collectstatic)
8. **Wait for DM** — curl `localhost:8400` (60 retries × 15s = 15 min max for collectstatic)
9. **Run DM migrations** — `docker compose exec web ./manage.py migrate --noinput`
10. **Restart SPAs** — `docker compose down; docker compose up -d` for auth, agent, skills (with auto-restart for Agent empty reply)
11. **OAuth/OIDC integrations** — `ibl launch --ibl-oauth --ibl-oidc --ibl-edx-manager` + `ibl dm auth-setup`
12. **Sync edX users** — `ibl edx sync-with-manager --users`
13. **Sync SSO credentials** — reads `spa-sso` and `ibl_web` client IDs from LMS database, writes to config, restarts Auth SPA
14. **Reload proxy + restart nginx**

### Step 4: Register in ALB Target Group

**What**: Deregisters any existing targets from the ALB target group, then registers the new EC2 instance.

**Why deregister first**: Prevents split-brain routing where the ALB sends some requests to an old instance with stale OAuth credentials.

**Health check**: ALB verifies the instance returns HTTP 200-399 on `/` before routing traffic.

### Step 5: Run Playwright Tests (OCI)

**What**: Launches Docker containers on Oracle Cloud Infrastructure (OCI) Container Instances that run the Playwright test suite against the staging environment.

**Test target**: `mentorai.stgX.iblai.org` (via ALB → EC2)

**Configuration**:
- Browsers: chrome, firefox, safari, edge (configurable, default: all 4 parallel)
- Workers: 3 per browser
- Max wait: 5400s (90 minutes)
- Retries: 2 per test

**Test users**: Each browser has its own dedicated test user to avoid conflicts:
- Chrome: `iblaiuserchromenew`
- Firefox: `iblaiuserfirefoxnew`
- Safari: `iblaiusersafarinew`
- Edge: `iblaiuseredgenew`

**Results**: Uploaded to S3 for resumption on subsequent runs.

### Step 6: Terminate EC2

**What**: `aws ec2 terminate-instances --instance-ids <id>`

**When**: Always runs, even if tests fail. The `if: always()` condition ensures cleanup.

**What persists**: VPC, ALB, Route53, S3 buckets — all reused on next launch.

## Timing

| Step | Duration |
|------|----------|
| Build Playwright image | 2-5 min (cached: instant) |
| Launch EC2 | ~20s |
| SSH ready | ~45s |
| Service update (Ansible) | 20-40 min (DM collectstatic dominates) |
| ALB health check | ~30s |
| Playwright tests (4 browsers) | 15-90 min |
| Terminate | instant |
| **Total** | **40-90 min** |

## Repository Map

| Repository | Role |
|------------|------|
| [iblai-infra-cli](https://github.com/iblai/iblai-infra-cli) | CLI tool with `service-update` command, Ansible playbooks, Terraform templates |
| [iblai-web-ops](https://github.com/iblai/iblai-web-ops) | Reusable GitHub Actions workflows (OCI test runner, Docker builds, domain locking) |
| [iblai-prod-images](https://github.com/iblai/iblai-prod-images) | Container image version pins (DM, edX, SPAs) |
| [mentorai](https://github.com/iblai/mentorai) | SPA source code, Playwright tests, PR validation workflows |

## Secrets & Variables

### Variables (on mentorai repo)

| Variable | Example |
|----------|---------|
| `STG1_AMI_ID` | `ami-02dff3992891505ba` |
| `STG1_SUBNET_ID` | `subnet-022ff062fe90b23b1` |
| `STG1_SG_ID` | `sg-0d56a7433d4b2a364` |
| `STG1_TG_ARN` | `arn:aws:elasticloadbalancing:...` |
| `STG1_KEY_PAIR` | `stg1-staging-key` |

Repeat for STG2, STG3, STG4.

### Secrets

| Secret | Purpose |
|--------|---------|
| `SERVICE_UPDATE_ACCESS_KEY` | AWS IAM key for EC2 launch/terminate + SG rule management |
| `SERVICE_UPDATE_SECRET_KEY` | AWS IAM secret |
| `STG1_SSH_KEY` – `STG4_SSH_KEY` | SSH private keys for each stg environment |
| `GIT_TOKEN` | GitHub PAT for private repo access |
| `SSH_PRIVATE_DEPLOY_OPS` | SSH key for OCI/deployment operations |
| OCI secrets | Oracle Cloud credentials for container instances |
| S3 secrets | AWS credentials for test log storage |

### IAM Policy (SERVICE_UPDATE keys)

```json
{
  "Statement": [
    {
      "Action": [
        "ec2:RunInstances", "ec2:DescribeInstances", "ec2:DescribeImages",
        "ec2:CreateTags", "ec2:TerminateInstances",
        "ec2:AuthorizeSecurityGroupIngress", "ec2:RevokeSecurityGroupIngress"
      ],
      "Resource": "*"
    },
    {
      "Action": [
        "elasticloadbalancing:RegisterTargets",
        "elasticloadbalancing:DeregisterTargets",
        "elasticloadbalancing:DescribeTargetHealth"
      ],
      "Resource": "*"
    }
  ]
}
```

## Known Behaviors

### DM collectstatic (15-20 min cold boot)
The DM container entrypoint runs `collectstatic --noinput` before starting gunicorn. This takes 15-20 minutes on a fresh AMI boot at 100% CPU. The service-update flow uses `docker compose up -d` (idempotent, no recreate) to avoid triggering collectstatic unnecessarily.

### Agent SPA empty reply
Agent SPA occasionally returns empty HTTP replies for 60-90s after startup despite reporting "Ready". The service-update role detects this and auto-restarts the container, with `ignore_errors` so the pipeline continues.

### ALB split-brain routing
If old EC2 instances remain registered in the ALB target group, the ALB load-balances between old and new instances with different OAuth credentials — causing intermittent 409 auth errors. The pipeline deregisters all existing targets before registering the new instance.

### OAuth credential sync
`ibl config save` regenerates `auth.yml` but doesn't preserve SSO credentials. The pipeline reads `spa-sso` and `ibl_web` client credentials directly from the LMS database and writes them to config before restarting the Auth SPA.

## Creating New AMIs

When the platform or test data changes, create new AMIs:

1. Launch a stg env from an existing AMI
2. Make changes (add platforms, users, config)
3. Verify all services healthy
4. Create AMI from the EC2 instance
5. Update `STGx_AMI_ID` variables on mentorai (and skillsai)

AMI requirements:
- All containers must be in a startable state (they may not be running — the service-update handles startup)
- S3 config must be baked in (`ENABLE_S3_BUCKET_STORAGE=True`, bucket names, region, credentials)
- Test platforms and users must be pre-seeded
- `iblai-cli-ops` virtualenv must exist with pyenv
