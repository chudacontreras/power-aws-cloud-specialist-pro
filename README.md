# AWS Cloud Specialist Pro — Kiro Power

A [Kiro](https://kiro.dev) Power that turns the agent into a tier-2/tier-3 **AWS Cloud Specialist**,
reasoning through the lens of the [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
across its six pillars: Operational Excellence, Security, Reliability, Performance Efficiency,
Cost Optimization, and Sustainability.

This is the hardened ("pro") revision of the original AWS Cloud Specialist power. It fixes the
broken MCP servers from the original (yanked PyPI packages, `PATH` resolution failures) and adds
first-class support for multiple AWS CLI profiles.

## What's included

### MCP servers

| Server key | Package / Image | Purpose | Needs AWS creds? |
|------------|-----------------|---------|------------------|
| `aws-documentation` | `awslabs.aws-documentation-mcp-server` (uvx) | Search official AWS docs & references | No |
| `aws-pricing` | `awslabs.aws-pricing-mcp-server` (uvx) | AWS service catalog + pricing discovery | Yes |
| `terraform-registry` | `hashicorp/terraform-mcp-server` (Docker) | Terraform Registry: providers, modules, best practices | No |
| `aws-pricing-<profile>` | `awslabs.aws-pricing-mcp-server` (uvx) | Per-profile pricing server (optional, one per AWS profile) | Yes |

### Steering guides

On-demand topic guides loaded based on what you're working on:

- `architecture-wellarchitected.md` — architecture design, multi-region, HA
- `cicd-devops.md` — CI/CD pipelines, CodePipeline, ArgoCD
- `containers-kubernetes.md` — Docker, ECS, EKS, Kubernetes, Helm
- `cost-optimization.md` — cost review, rightsizing, savings plans
- `networking-vpc.md` — VPC, networking, load balancers, DNS
- `observability-monitoring.md` — CloudWatch, alarms, X-Ray, logging
- `security-compliance.md` — IAM, KMS, Secrets, WAF, GuardDuty
- `terraform-iac.md` — Terraform code, modules, state

## Requirements

- [Kiro IDE](https://kiro.dev)
- [`uv` / `uvx`](https://docs.astral.sh/uv/getting-started/installation/) — runs the AWS MCP servers
- [Docker](https://www.docker.com/) — runs the Terraform Registry MCP server
- AWS CLI v2 with configured profiles — for any account-scoped (pricing/cost) work

## Installation & setup

1. Copy `power-aws-cloud-specialist-pro/` into your Kiro powers directory.
2. **Adapt `mcp.json` to your machine.** The config uses **absolute binary paths** so the servers
   launch even when the IDE has a minimal `PATH`. These paths are machine-specific — update them:
   - `command` for the uvx servers → run `command -v uvx` and use that path.
   - `command` for `terraform-registry` → run `command -v docker` and use that path.
   - The `env.PATH` values accordingly.
3. **Set your AWS profile.** The `aws-pricing` server binds to one profile via `AWS_PROFILE`
   (default: `default`). Change it to a profile that exists on your machine. The
   `aws-pricing-<profile>` block is a template for additional per-profile pricing servers —
   duplicate it, rename the key, set its `AWS_PROFILE`, and set `"disabled": false`.
4. Reconnect the MCP servers (or fully restart the IDE).

> **Note:** `mcp.json` contains no credentials — only profile *names* and local paths. AWS
> credentials are resolved by the AWS SDK from your `~/.aws` config at runtime. Never commit real
> credentials to this repo.

## Multiple AWS profiles

For ad-hoc multi-profile work, prefer the AWS CLI with an explicit `--profile`:

```bash
aws ce get-cost-and-usage --profile <profile> --region us-east-1 \
  --time-period Start=2026-10-01,End=2026-11-01 \
  --granularity MONTHLY --metrics "UnblendedCost"
```

Reserve dedicated `aws-pricing-<profile>` MCP servers for the one or two profiles you use most.

## Troubleshooting

**"Connection closed" / server shows no tools** — almost always the launch `PATH`. Verify the
binaries still live where `mcp.json` expects (`command -v uvx`, `command -v docker`). After editing
`mcp.json`, fully restart the IDE.

**Terraform server logs a TFE credentials warning** — harmless. It just means no HCP Terraform /
Terraform Enterprise token is configured. Public Registry tools still work.

**Pricing server errors on account data** — check the bound `AWS_PROFILE` has valid credentials:
`aws sts get-caller-identity --profile <profile>`.

---

*This power follows AWS Well-Architected Framework best practices and reflects real-world
enterprise cloud operations experience.*
