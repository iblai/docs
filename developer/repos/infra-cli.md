# Infra CLI (`iblai/iblai-infra-cli`)

> **GitHub: [github.com/iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)** — Interactive CLI for provisioning and configuring the ibl.ai platform on AWS: end-to-end infrastructure creation with Terraform and full application setup with Ansible. Python 3.11+ · Terraform · Ansible · Proprietary license.

## Overview

The Infra CLI is how you **deploy the ibl.ai platform on your own infrastructure**. It handles end-to-end infrastructure creation on AWS with Terraform and full application setup with Ansible — and it can also bootstrap existing servers on any cloud or bare metal, without Terraform at all.

In the five-repo family, this is the **deployment layer**: `iblai/api` operates a running platform headlessly via its REST API, `iblai/vibe` builds new applications on the backend, and `iblai/os` and `iblai/lms` are complete sample applications you can take and modify — the Infra CLI is what stands that backend up on infrastructure you control.

It is for platform engineers doing self-hosted or sovereign deployments. Access to the ibl.ai Docker images and platform codebase requires a license — reach out at [ibl.ai/contact](https://ibl.ai/contact) to get started.

## Prerequisites

- **Python 3.11+**
- **[uv](https://docs.astral.sh/uv/)** (recommended) or pip
- **[Terraform](https://developer.hashicorp.com/terraform/install)** installed and on PATH
- **AWS account** with EC2, ELB, S3, ACM, Route53, IAM, and STS permissions
- **SSH access** to the target EC2 instance (key is generated or provided during provisioning)

Installed automatically as Python dependencies: **ansible-core** (>= 2.15, used by `iblai infra setup`) and **boto3** (AWS SDK). Terraform is called as a subprocess and must be installed separately.

## Usage

Run `iblai infra` to see all available commands and a getting-started guide.

#### 1. Check IAM permissions

Before provisioning, verify your AWS credentials have the required permissions:

```bash
iblai infra permissions              # Show required IAM policy JSON
iblai infra permissions --check      # Dry-run verification against active credentials
```

#### 2. Provision infrastructure

```bash
iblai infra provision
```

An interactive wizard walks you through AWS credentials (profile, access keys, or environment variables), project and compute settings (name, environment, instance type, volume size), network and SSH (VPC CIDR, VPN IP, key setup), and domain and certificates (base domain, Route53, ACM/upload/none) — then shows a full review before applying. Terraform runs with real-time progress showing each resource as it's created.

#### 3. Set up the platform

```bash
iblai infra setup              # Set up an existing server (any provider, bare metal)
iblai infra setup <name>       # Set up a Terraform-provisioned environment by name
```

Both paths run the same Ansible playbook. With a project name, inputs (IP, domain, SSH key, AWS credentials) auto-populate from Terraform state; without one, the CLI prompts interactively — no Terraform required. The playbook runs 9 sequential roles:

| Role | What it does |
|------|-------------|
| `docker` | Installs Docker Engine, docker compose, and apache2-utils |
| `awscli` | Installs AWS CLI v2 for ECR and S3 access |
| `python` | Installs pyenv and Python 3.11.8 |
| `ibl_cli_ops` | Clones and installs `iblai-cli-ops` in a virtualenv (private repo — requires access) |
| `ibl_platform` | Configures base domain, environment, image tags, CORS, RBAC, unified API gateway, and service defaults |
| `ibl_dm` | Launches iblai-dm-pro (PostgreSQL with pgvector, Redis, Django, Celery, Langfuse, Minio) |
| `ibl_edx` | Launches iblai-edx-pro (LMS, CMS, MySQL, MongoDB, Redis, Elasticsearch, MFE) |
| `ibl_spa` | Creates OAuth2 apps, configures and launches the Auth, Mentor AI, and Skills AI SPAs |
| `final_steps` | Reloads proxy, OAuth/OIDC setup, syncs edX with DM, creates super admins, seeds CSRF domains, flows, LLM registry, agents, and RBAC data |

The setup wizard prompts for the target host IP and SSH key path, base domain and environment config, Docker image tags for each service, whether to enable AI features, an optional OpenAI API key, super admin credentials, and the GitHub PAT + AWS credentials for the VM.

#### 4. Manage environments

```bash
iblai infra list                # List all managed environments
iblai infra status <name>       # Show infrastructure details and outputs
iblai infra auth                # Switch AWS credentials
iblai infra destroy <name>      # Tear down infrastructure or remove bootstrap project
```

## What Gets Created

#### AWS infrastructure (Terraform)

- VPC with 2 public subnets across availability zones (10.0.0.0/16)
- EC2 instance (Ubuntu 22.04) with encrypted EBS volume (AES-256)
- Application Load Balancer with TLS 1.2/1.3 termination
- ACM certificates (RSA 2048-bit, DNS-validated, auto-renewed)
- Security groups (SSH restricted to VPN CIDR, HTTP/HTTPS from ALB only)
- 3 S3 buckets with server-side encryption (backups, media, static)
- Route53 hosted zone with 19 subdomain A-records

#### Platform services (Ansible)

- **iblai-edx-pro** — LMS, CMS, workers, MySQL 8.0, Redis, MongoDB, Elasticsearch, Forum, Notes, Meilisearch, SMTP relay, Caddy
- **iblai-dm-pro** — Django web, ASGI, Celery worker/beat, PostgreSQL 16, Redis, Flowise AI
- **iblai-web-frontend** — the platform single-page applications (Auth, Mentor AI, Skills AI)
- **Monitoring** — Prometheus, Grafana, AlertManager, metric exporters
- **Nginx** reverse proxy

## Authentication

The CLI always lets you choose how to authenticate — it never silently auto-detects credentials, and it walks you through the choice interactively on first use. Supported methods: AWS profiles from `~/.aws/config` and `~/.aws/credentials` (type to filter), environment variables (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`), or manual entry with masked input. Your session is saved and reused across commands until you switch credentials or it expires.

## Workspace

All Terraform state, SSH keys, and project configuration are stored at:

```
~/.iblai-infra/projects/<project-name>/
```

## Quick Start

Using [uv](https://docs.astral.sh/uv/) (recommended):

```bash
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone the repo
git clone https://github.com/iblai/iblai-infra-cli.git
cd iblai-infra-cli

# Create a virtual environment and install
uv venv
source .venv/bin/activate
uv pip install .
```

Using pip:

```bash
pip install .
```

Verify the installation and its toolchain:

```bash
iblai --version
ansible-playbook --version
terraform --version
```

## How It Fits With the Other Repos

The Infra CLI is the deployment layer of the ibl.ai open-source family: it stands up, on your own AWS account or existing servers, the same backend platform that the other four repos build on, operate, and run against.

- **[github.com/iblai/api](https://github.com/iblai/api)** — agent skills + a chat MCP server to headlessly manage and operate the entire ibl.ai platform via its REST API (`npx skills add iblai/api`)
- **[github.com/iblai/vibe](https://github.com/iblai/vibe)** — how to "vibe code" new applications on top of the ibl.ai backend (Next.js SDK + Claude Code skills; `npx skills add iblai/vibe`)
- **[github.com/iblai/os](https://github.com/iblai/os)** — a complete sample application on the backend: the open-source AI agent platform running at os.ibl.ai, yours to fork and modify
- **[github.com/iblai/lms](https://github.com/iblai/lms)** — another complete sample application: an open-source skills intelligence platform (courses, competencies, credentials), yours to fork and modify

## Related Documentation

- [Infrastructure CLI](/developer/infrastructure/infra-cli) — the existing infra CLI reference page
- [Infrastructure Architecture](/developer/infrastructure/infra-architecture) — single-server and multi-server AWS architecture diagrams
- [AMI Launch Pipeline](/developer/infrastructure/ami-launch-pipeline) — the AMI build and launch pipeline
- [Claw Server Setup](/developer/claw-setup/server-setup) — setting up agent sandbox servers
