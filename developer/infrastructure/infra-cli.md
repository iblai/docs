# Infrastructure CLI

Interactive CLI for provisioning and configuring the [ibl.ai platform](https://github.com/iblai/iblai-infra-cli) on AWS or GCP.

---

## Overview

**iblai-infra-cli** automates end-to-end provisioning and configuration of the ibl.ai platform. It provisions infrastructure with Terraform and performs the full application setup with Ansible, guiding you through both with interactive prompts that handle resource creation, networking, and service deployment.

It supports single-server deployments on **AWS** and **GCP**, and can also bootstrap an existing server — any other cloud, or bare metal — without running Terraform at all. In that mode only the setup step runs, over SSH.

The repository contains the installation and provisioning tooling. Access to the platform container images and codebase requires a license.

---

## Prerequisites

| | Requirement |
|---|---|
| **Always** | Python 3.11+, [uv](https://docs.astral.sh/uv/) or pip, [Terraform](https://developer.hashicorp.com/terraform/install) on PATH, a domain you control, and licensed access to the private packages |
| **AWS** | An account with EC2, ELB, S3, ACM, Route53, IAM, and STS permissions |
| **GCP** | A project with billing enabled, the `compute` and `dns` APIs on, and the `compute.admin`, `dns.admin`, and `iam.serviceAccountUser` roles |
| **Existing server** | SSH access to an Ubuntu 22.04 machine with at least 100 GB of disk, and DNS you can point at it |

AWS credentials are required on every deployment — including bare metal and GCP — because the platform container images are pulled from ECR and S3 is used for object storage.

---

## Repository

- **GitHub**: [iblai/iblai-infra-cli](https://github.com/iblai/iblai-infra-cli)
- **License**: MIT

---

## Getting Started

Five steps from zero to a running platform:

```bash
# 1. Install
git clone https://github.com/iblai/iblai-infra-cli.git && cd iblai-infra-cli
uv sync                        # AWS only
uv sync --extra gcp            # AWS + GCP support

# 2. Verify your cloud credentials have what's needed
uv run iblai infra permissions --check

# 3. Provision the infrastructure (interactive wizard)
uv run iblai infra provision

# 4. Install the platform on the new VM (Ansible over SSH)
uv run iblai infra setup <project-name>
```

The platform is then available at `https://learn.<your-domain>`. The permissions check prints the exact IAM policy required, so you can hand it to a cloud administrator before provisioning. Steps 3 and 4 also run non-interactively from a `.env` file if you would rather script the deployment.

---

## Related

- [Infra CLI overview](/developer/repos/infra-cli) — the full repository reference
- [edX SSO Setup](/developer/infrastructure/edx-sso-setup) — add identity providers after deployment
