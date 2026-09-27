# Infrastructure for breakfix

This file states what must exist in Amazon Web Services (AWS). Every value comes from the
application code. The file names the source file for each value that is not obvious.

## 1. What runs where

Triage is a long-lived container. It polls Sentry on the cron expression `*/5 * * * *`, which is
every 5 minutes. It serves two HTTP routes on `TRIAGE_PORT`, default `3100`.

1. `POST /triage/tick` starts one poll immediately.
2. `GET /triage/health` probes the database. It answers `503` when the database is silent.

Run triage as one Amazon Elastic Container Service (ECS) service on AWS Fargate. One task is
enough. Two tasks poll the same rows twice.

Scout is a one-shot container. AWS Batch starts one container for each Sentry Issue. The container
exits when the investigation ends. AWS Batch then destroys the container. Nothing survives the
container except the log stream.

Warning: `POST /triage/tick` has no authentication. Keep triage inside the Virtual Private Cloud
(VPC). Do not attach a public load balancer to it.

## 2. AWS Batch on AWS Fargate

Provision one compute environment, one job queue, and one job definition.

Compute environment:

- `type = "MANAGED"`
- `compute_resources.type = "FARGATE"`
- `compute_resources.max_vcpus = 6`
- Private subnets, and one security group with egress only.

Job queue:

- The job queue name must equal the value of `BATCH_JOB_QUEUE`. The example file uses
  `breakfix-scout`.
- Attach the compute environment with priority 1.

Job definition:

- The job definition name must equal the value of `BATCH_JOB_DEFINITION`. The example file uses
  `breakfix-scout`.
- `type = "container"`
- `platform_capabilities = ["FARGATE"]`
- `resourceRequirements`: type `VCPU` with value `2`, and type `MEMORY` with value `4096`. That is
  2 virtual central processing units (vCPU) and 4096 mebibytes (MiB), which is 4 gibibytes (GiB).
- `retry_strategy { attempts = 2 }`
- `timeout { attempt_duration_seconds = 2700 }`
- `runtimePlatform.operatingSystemFamily = "LINUX"`
- `fargatePlatformConfiguration.platformVersion = "LATEST"`

The compute environment holds 6 virtual central processing units (vCPU). Each job takes 2 virtual
central processing units (vCPU). 6 divided by 2 gives 3 concurrent investigations. A fourth Sentry
Issue waits in the job queue.

The value `2700` seconds is 45 minutes. Scout's own limit `MAX_RUN_MS` is 1800000 milliseconds,
which is 30 minutes. The application limit sits deliberately below the AWS Batch limit. Scout's
timeout fires first, and scout still returns a partial report. The AWS Batch timeout kills the
container and returns nothing.

Triage calls `SubmitJob` in `apps/triage/src/batch/batch.service.ts`. The call sends:

- `jobName`: `<shortId>-<runId>`, with every character outside `A-Za-z0-9_-` replaced by `-`, and
  cut to 128 characters.
- `jobQueue`: the value of `BATCH_JOB_QUEUE`.
- `jobDefinition`: the value of `BATCH_JOB_DEFINITION`.
- `containerOverrides.environment`: three overrides, `RUN_ID`, `ISSUE_ID`, and `KIND`.
- `tags`: `issueId`, `shortId`, and `projectSlug`, for search in the console.

The three overrides carry the whole job input. The job definition needs no default value for them.

## 3. Identity and Access Management (IAM)

Create the service-linked role `AWSServiceRoleForBatch` once for the account. AWS Batch cannot
create a compute environment without it. Terraform creates it with `aws_iam_service_linked_role`
for the service `batch.amazonaws.com`. The role may already exist in the account. An existing role
makes the create call fail, so import the role or guard it with a variable.

All three roles below trust the service `ecs-tasks.amazonaws.com`.

### Scout job execution role

AWS Batch uses this role. Scout's own code never uses it.

- `ecr:GetAuthorizationToken`
- `ecr:BatchCheckLayerAvailability`
- `ecr:GetDownloadUrlForLayer`
- `ecr:BatchGetImage`
- `logs:CreateLogStream`
- `logs:PutLogEvents`
- `secretsmanager:GetSecretValue` on the three secrets in section 4. The execution role needs this
  action only when the job definition maps a secret into an environment variable. Section 4
  explains why that mapping is the only path that works today.

### Scout job role

Scout's own code uses this role.

- `secretsmanager:GetSecretValue` on `breakfix/sentry`, `breakfix/github`, and `breakfix/model`.
- `kms:Decrypt` on the Key Management Service (KMS) key, when a customer managed key encrypts the
  secrets.

Scout opens no database connection today. Section 8 explains that gap. Grant no database permission
to this role now.

### Triage task role

- `batch:SubmitJob` on two Amazon Resource Names (ARN): the job queue and the job definition.
- `secretsmanager:GetSecretValue` on `breakfix/sentry`.
- `kms:Decrypt` on the Key Management Service (KMS) key, when a customer managed key encrypts the
  secret.

The `SubmitJob` call sends tags. AWS Batch may reject a tagged call without `batch:TagResource`.
Add that action to the triage task role when the call fails.

Triage also needs an execution role for the Amazon Elastic Container Service (ECS) task. Give it
the same image pull actions and log actions as the scout job execution role.

## 4. Secrets

Create three secrets in AWS Secrets Manager. The code looks for the exact identifiers below. The
identifiers live in `apps/scout/src/secrets/secrets.service.ts` and in
`apps/triage/src/secrets/secrets.service.ts`.

| Secret identifier | Reader | Contents |
|---|---|---|
| `breakfix/sentry` | triage and scout | A Sentry authentication token. |
| `breakfix/github` | scout | A GitHub token that can clone the target repositories. |
| `breakfix/model` | scout | The API key of the model gateway. Scout treats a read failure as no key. |

### The Sentry token

`GAPS.md` item 6 decides the contents of `breakfix/sentry`. The file states:

> A Sentry organization auth token starts with `sntrys_`. Its scope list is fixed to `org:ci`.

A `sntrys_` organization token CANNOT do scout's work. It fails on the issue details endpoint, on
the single event endpoint, on the trace endpoint, and on the events and logs endpoint.

> **What we do.** Use an internal integration token, or a user auth token. Grant `event:read`,
> `org:read`, and `project:releases`.

Put that token in `breakfix/sentry`. A `sntrys_` value there breaks both services.

### How the container gets a secret today

`SecretsService` reads the environment variable first. It reads `SENTRY_TOKEN`, `GITHUB_TOKEN`, or
`MODEL_API_KEY`. It calls AWS Secrets Manager only when the environment variable is absent and
`NODE_ENV` equals `production`. That AWS Secrets Manager path is not implemented. It throws an
error in both services.

So the environment variable is the only path that works. Map each secret into an environment
variable with the `secrets` block of the job definition, and with the `secrets` block of the triage
task definition. AWS Batch and Amazon Elastic Container Service (ECS) then read the secret with the
execution role, and the code finds the value in the environment.

## 5. The database

Provision one PostgreSQL instance on the Amazon Relational Database Service (RDS). Both triage and
scout must reach it. Scout does not use it yet. Section 8 explains that gap.

- Triage opens a connection pool of `DATABASE_POOL_MAX` clients, default 5.
- Triage creates the table itself at start. The file `apps/triage/src/db/schema.sql` uses
  `create table if not exists`. Terraform provisions the instance, the database, and the
  credentials only. Terraform runs no migration.
- The table is `scout_run`. Its primary key is `issue_id`. It holds one unique index on `run_id`,
  and one index on `(state, issue_last_seen)`.
- Build `DATABASE_URL` in the form `postgres://<user>:<password>@<host>:5432/<database>`. Store it
  in AWS Secrets Manager. Map it into the environment of the triage task.
- Set `DATABASE_SSL=true` when the instance forces Transport Layer Security (TLS). The code then
  sets `rejectUnauthorized: false`, so it does not check the server certificate.

## 6. Networking

Place both containers in private subnets. Give the subnets a route to a Network Address Translation
(NAT) gateway. AWS Fargate needs that route to pull the image and to reach the internet. A public
subnet with `assign_public_ip = "ENABLED"` also works, and it costs less.

Triage needs outbound access to:

1. The Sentry API at `SENTRY_BASE_URL`, over port 443.
2. The AWS Batch API, over port 443.
3. The PostgreSQL instance, over port 5432.

Scout needs outbound access to:

1. The Sentry API at `SENTRY_BASE_URL`, over port 443.
2. GitHub, over port 443.
3. The model gateway at `MODEL_BASE_URL`, over the port of that URL.

Scout clones a repository with `git clone --filter=blob:none`. A partial clone fetches more objects
during the run. Scout therefore needs real egress to GitHub for the whole run. A cached mirror is
not enough.

`OPENCODE_PORT`, default `4096`, is a loopback port inside the scout container. Write no security
group rule for it.

Give the PostgreSQL security group one inbound rule from the triage security group on port 5432.

## 7. Environment variables

Every row comes from `apps/triage/src/config/config.service.ts` or from
`apps/scout/src/config/config.service.ts`. A service fails at start when a required variable is
absent.

| Name | Service | Required or default | Source of the value |
|---|---|---|---|
| `TRIAGE_PORT` | triage | Default `3100` | Terraform, and it must match the container port |
| `TRIAGE_POLL_CRON` | triage | Default `*/5 * * * *` | Terraform, only to change the poll period |
| `TRIAGE_WATERMARK_MARGIN_MINUTES` | triage | Default `15` | Terraform, only to change the margin |
| `TRIAGE_COLD_START_DAYS` | triage | Default `7` | Terraform, only for the first run |
| `TRIAGE_MAX_ISSUES_PER_TICK` | triage | Default `50` | Terraform, and it caps the queue growth |
| `TRIAGE_REPROCESS_SUBSTATUSES` | triage | Default `regressed,escalating` | Terraform, comma separated |
| `TRIAGE_SUBMIT_ENABLED` | triage | Default `true` | Terraform. `false`, `0`, or `no` gives a dry run |
| `SENTRY_BASE_URL` | triage and scout | Default `https://sentry.io` | Terraform, for a self-hosted Sentry |
| `SENTRY_ORG` | triage and scout | Required | Terraform variable, the Sentry organization slug |
| `SENTRY_ENVIRONMENT` | triage and scout | Default `staging` | Terraform variable for each deployment |
| `DATABASE_URL` | triage | Required | AWS Secrets Manager, mapped into the environment |
| `DATABASE_SSL` | triage | Default `false` | Terraform. Set `true` for the Amazon Relational Database Service (RDS) |
| `DATABASE_POOL_MAX` | triage | Default `5` | Terraform, and it must stay under the instance limit |
| `AWS_REGION` | triage | Required | Terraform, the region of the job queue |
| `BATCH_JOB_QUEUE` | triage | Required | Terraform output, the job queue name |
| `BATCH_JOB_DEFINITION` | triage | Required | Terraform output, the job definition name |
| `PORT` | scout | Default `3000` | Terraform. Section 8 explains why scout still opens a port |
| `GITHUB_TOKEN` | scout | Optional | AWS Secrets Manager `breakfix/github` |
| `MODEL_PROVIDER_ID` | scout | Default `gateway` | Terraform |
| `MODEL_PROVIDER_NPM` | scout | Default `@ai-sdk/openai-compatible` | Terraform |
| `MODEL_BASE_URL` | scout | Required, and it must be a URL | Terraform, the address of the model gateway |
| `MODEL_ID` | scout | Required | Terraform, the model name |
| `MODEL_API_KEY` | scout | Optional | AWS Secrets Manager `breakfix/model` |
| `MAX_TOKENS` | scout | Default `500000` | Terraform, the token budget of one run |
| `MAX_TOOL_CALLS` | scout | Default `40` | Terraform, the tool call budget of one run |
| `MAX_RUN_MS` | scout | Default `1800000` | Terraform. Keep it under `attempt_duration_seconds` |
| `WORK_DIR` | scout | Default `/work` | Terraform, and the path must be writable |
| `OPENCODE_PORT` | scout | Default `4096` | Terraform, a loopback port inside the container |

`ConfigService` does not read the variables below. The container still needs them.

| Name | Service | Required or default | Source of the value |
|---|---|---|---|
| `SENTRY_TOKEN` | triage and scout | Required, read by `SecretsService` | AWS Secrets Manager `breakfix/sentry` |
| `NODE_ENV` | triage and scout | Set the value to `production` | Terraform |
| `RUN_ID` | scout | Required, sent by `SubmitJob` | The `containerOverrides` of the triage submit call |
| `ISSUE_ID` | scout | Required, sent by `SubmitJob` | The `containerOverrides` of the triage submit call |
| `KIND` | scout | Required, sent by `SubmitJob` | The `containerOverrides` of the triage submit call |

`GAPS.md` item 5 also names three OpenCode variables for the scout container:
`OPENCODE_DISABLE_PROJECT_CONFIG=1`, `OPENCODE_DISABLE_EXTERNAL_SKILLS=1`, and
`OPENCODE_DISABLE_AUTOUPDATE=1`. Without them, OpenCode writes into the cloned repository.

## 8. Still open

The application does not do the work below yet. Do not build infrastructure that promises it.

1. **Scout has no repository map.** The field `repos` in `apps/scout/src/config/config.service.ts`
   is an empty object. `repoForProject` returns `undefined` for every project slug. Every scout run
   fails today. Do not size the compute environment against a working scout.
2. **Scout's entry point is still a server.** The file `apps/scout/src/main.ts` calls
   `app.listen(config.port)`. It does not read `RUN_ID`, `ISSUE_ID`, or `KIND`. Warning: an AWS
   Batch job of this image starts a server and never exits. It holds 2 virtual central processing
   units (vCPU) for the full 2700 seconds, and `attempts = 2` repeats that. Keep
   `TRIAGE_SUBMIT_ENABLED=false` until somebody repairs the entry point.
3. **Nothing stores the scout result.** No code writes the report, the usage, or the final state
   into the `scout_run` row. A row stays in the state `running` after the container exits.
4. **Nothing uploads the evidence.** No code copies the contents of `WORK_DIR` out of the container
   before AWS Batch destroys the container. Add an Amazon Simple Storage Service (S3) bucket, and
   grant `s3:PutObject` to the scout job role, when that code exists. Do not create the bucket now.
5. **AWS Secrets Manager is not implemented.** `SecretsService` throws on the AWS Secrets Manager
   path in both services. Section 4 names the environment variable path that works today.
