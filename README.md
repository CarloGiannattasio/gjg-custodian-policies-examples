# gjg-custodian-policies-examples

[![Custodian policies (Moto)](https://github.com/CarloGiannattasio/gjg-custodian-policies-examples/actions/workflows/custodian-policies.yml/badge.svg)](https://github.com/CarloGiannattasio/gjg-custodian-policies-examples/actions/workflows/custodian-policies.yml)

Staged idle cleanups for RDS, load balancers, and IAM roles, plus an EC2 off-hours schedule. Each stage notifies before it marks, stops, or deletes.

Opt-out tags skip a resource. Slack and email alerts go through c7n-mailer.

The pipeline validates every policy and dry-runs it with Moto, so tests need no AWS credentials.

Example [Cloud Custodian](https://cloudcustodian.io/) policies you can copy, adapt, and run in your own AWS (and other cloud) accounts.

Cloud Custodian is a rules engine for managing public cloud resources. Policies are YAML files that describe what to find (filters) and what to do (actions)—for example, tag untagged resources, stop idle instances, or enforce encryption settings.

## What’s in this repo

This repository collects **ready-to-use policy examples** for common governance and cost-optimization scenarios. Each policy is meant as a starting point: adjust account IDs, tags, regions, and actions to match your organization’s standards.

Policies are grouped by cloud provider under `policies/aws/`.

### Mailer (Slack + email)

All policies in this repo use the **`notify`** action with **Slack** (`slack://#cloud-custodian-alerts`) and **email** (`cloud-custodian@example.com`, plus `resource-owner` where tags allow).

1. Deploy shared **[c7n-mailer](https://cloudcustodian.io/docs/tools/c7n-mailer.html)** from **`mailer/`** (`mailer.example.yml` → `mailer.yml`).
2. Set the same SQS **`queue`** URL in each policy’s `notify-transport` vars (template in **`policies/aws/shared/notify-vars.yml`**).

### AWS service quota usage (25% discovery → 70% warnings)

Policy file: `policies/aws/service-quota-usage.yml` (two policies, no scripts).

| Policy | Cadence | Purpose |
|--------|---------|---------|
| `service-quota-discovery-25` | **Weekly** | Full scan; save quotas **≥ 25%** to `-s logs/service-quota-discovery` |
| `service-quota-warning-70` | **Daily** | Read that discovery list; warn on **≥ 70%** into `-s logs/service-quota-warnings` |

The daily job reuses the discovery `resources.json` from the last weekly run (same paths per region). Run both from the repo root with the same `-r` (typically `-r all`).

**Weekly — discovery:**

```bash
custodian run -s logs/service-quota-discovery -r all \
  policies/aws/service-quota-usage.yml --policy service-quota-discovery-25
```

**Daily — warnings:**

```bash
custodian run -s logs/service-quota-warnings -r all \
  policies/aws/service-quota-usage.yml --policy service-quota-warning-70
```

Discovery files (with `-r all`): `logs/service-quota-discovery/<region>/service-quota-discovery-25/resources.json`.

### EC2 off-hours / on-hours (`maid_offhours` tag)

Two policy files — pick **opt-in** or **opt-out** (do not run both in the same account unless you intend to).

| File | Mode | Behavior |
|------|------|----------|
| `policies/aws/off-hours/ec2-offhours-schedule.yml` | **Opt-in** (`opt-out: false`) | Only instances **with** `maid_offhours` are scheduled |
| `policies/aws/off-hours/ec2-offhours-schedule-opt-out.yml` | **Opt-out** (`opt-out: true`) | **All** instances are scheduled unless `maid_offhours=off` |

Both use Custodian **`offhour`** / **`onhour`** filters (defaults: 8–20 Eastern, weekends off). Tag values: `on` or empty for defaults, `off` to exclude (opt-in: untagged instances are already excluded), or a custom schedule string. Run **about once per hour**.

```bash
# Opt-in
custodian run -s logs/ec2-schedule -r us-east-1 policies/aws/off-hours/ec2-offhours-schedule.yml

# Opt-out
custodian run -s logs/ec2-schedule-opt-out -r us-east-1 policies/aws/off-hours/ec2-offhours-schedule-opt-out.yml
```

See [Cloud Custodian offhours](https://cloudcustodian.io/docs/aws/examples/offhours.html) for tag syntax and other resource types (ASG, RDS, etc.).

### RDS idle cleanup (Slack + email)

Policy file: `policies/aws/rds-idle-cleanup/rds-idle-cleanup.yml`

Staged lifecycle for **RDS instances** (standalone) and **RDS/Aurora clusters** using `DatabaseConnections` metrics, `mark-for-op`, and **notify** (Slack + email via [c7n-mailer](https://cloudcustodian.io/docs/tools/c7n-mailer.html)):

| Stage | Policy (examples) | Action |
|-------|-------------------|--------|
| 7d idle | `rds-idle-notify-7d`, `rds-cluster-idle-notify-7d` | Notify, tag notice |
| Mark stop 14d | `rds-idle-mark-stop-14d`, `rds-cluster-idle-mark-stop-14d` | Notify + mark stop |
| Stop → mark delete 30d | `rds-idle-stop-mark-delete-30d`, `rds-cluster-idle-stop-mark-delete-30d` | Stop, mark delete, notify |
| Ensure stopped | `rds-idle-enforce-stopped-before-delete`, `rds-cluster-idle-enforce-stopped-before-delete` | Stop if still running |
| Cluster pre-delete | `rds-cluster-idle-start-predelete-1d` | **Start** cluster 1 day before delete (daily schedule) |
| Delete | `rds-idle-delete`, `rds-cluster-idle-delete` | Delete + notify |
| Active again | `rds-idle-unmark-on-activity`, `rds-cluster-idle-unmark-on-activity` | Clear tags |

Opt out with tag **`custodian_rds_idle_exempt`** or **`custodian_rds_idle_ignore`** (any value). Align the policy `notify` queue with **`mailer/mailer.example.yml`** (see **`mailer/README.md`**).

Run policies daily (or on a schedule that matches your `mark-for-op` windows):

```bash
custodian run -s logs/rds-idle -r all policies/aws/rds-idle-cleanup/rds-idle-cleanup.yml
```

### ELB idle cleanup (Slack + email)

Policy file: `policies/aws/elb-idle-cleanup/elb-idle-cleanup.yml`

Covers **Application/Network load balancers** (`app-elb`) and **Classic ELBs** (`elb`):

| Stage | Policies | Action |
|-------|----------|--------|
| 7d idle | `app-elb-idle-notify-7d`, `elb-idle-notify-7d` | Notify, tag notice |
| Mark delete 14d | `app-elb-idle-mark-delete-14d`, `elb-idle-mark-delete-14d` | Notify + mark delete |
| Delete | `app-elb-idle-delete`, `elb-idle-delete` | Delete + notify |

Opt out: **`custodian_elb_idle_exempt`** or **`custodian_elb_idle_ignore`**. Mailer: **`mailer/mailer.example.yml`**. ELBv2 delete skips load balancers with **deletion protection** enabled.

```bash
custodian run -s logs/elb-idle -r all policies/aws/elb-idle-cleanup/elb-idle-cleanup.yml
```

### IAM role idle cleanup (Slack + email)

Policy file: `policies/aws/iam-role-idle-cleanup/iam-role-idle-cleanup.yml`

IAM is **global** — run in **`us-east-1`**. Unused = not attached (EC2/Lambda/ECS) and no IAM Access Advisor activity (`usage` filter).

| Day (approx.) | Policy | Action |
|---------------|--------|--------|
| Start | `iam-role-idle-mark-year-notify` | Age > 1y, notify, mark 21-day lifecycle |
| +7 | `iam-role-idle-notify-7d` | Second notify |
| +14 | `iam-role-idle-disable-mark-delete-7d` | Detach policies (disable), mark delete +7d |
| +21 | `iam-role-idle-delete` | Delete (`force: true`) + notify |

Opt out: **`custodian_iam_role_idle_exempt`** or **`custodian_iam_role_idle_ignore`**. Skips roles under `/aws-service-role/`.

```bash
custodian run -s logs/iam-role-idle -r us-east-1 policies/aws/iam-role-idle-cleanup/iam-role-idle-cleanup.yml
```

## Using these policies

1. [Install Cloud Custodian](https://cloudcustodian.io/docs/installation.html) and configure credentials for your cloud provider.
2. Copy or clone the policy YAML you need.
3. Run in dry-run mode first to see what would change:

   ```bash
   custodian run -s output/ policy.yml
   ```

4. Review the output, then apply when you are satisfied with the scope and actions.

Refer to the [Cloud Custodian documentation](https://cloudcustodian.io/docs/index.html) for policy structure, filters, actions, and modes (periodic, cloud trail, etc.).

## Contributing

More examples will be added over time. If you use or extend a policy here, treat it as sample code: test in a non-production account before rolling out broadly.

### CI and local tests (Moto)

GitHub Actions workflow **Custodian policies (Moto)** runs on **pull requests** and **manual dispatch** (`workflow_dispatch`). It validates every policy YAML and dry-runs policies in-process against [Moto](https://github.com/getmoto/moto) (no AWS credentials). Service quota policies are validate-only because Moto does not implement the Service Quotas API.

```bash
pip install -r requirements-dev.txt
pytest tests/ -v
```

Local Custodian output can go under **`tmp/`** (gitignored except `tmp/README.md`):

```bash
custodian run -s tmp/c7n-out -r us-east-1 policies/aws/off-hours/ec2-offhours-schedule.yml
```
