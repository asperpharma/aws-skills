---
name: aws-cdk-lambda-deployment
description: AWS CDK expert for deploying Lambda functions with production-ready bundling, layers, versions/aliases, and traffic-shifting (canary, linear, all-at-once). Use when deploying Lambda via CDK, configuring NodejsFunction or PythonFunction bundling, defining Lambda layers, wiring CodeDeploy aliases for blue/green or canary rollouts, optimising cold starts (memory/SnapStart/provisioned concurrency), or setting up Lambda observability (Powertools, X-Ray, log retention).
context: fork
skills:
  - aws-mcp-setup
  - aws-cdk-development
allowed-tools:
  - mcp__cdk__*
  - mcp__aws-mcp__*
  - mcp__awsdocs__*
  - Bash(cdk *)
  - Bash(npm *)
  - Bash(npx *)
  - Bash(aws lambda *)
  - Bash(aws cloudformation *)
  - Bash(aws sts get-caller-identity)
hooks:
  PreToolUse:
    - matcher: Bash(cdk deploy*)
      command: aws sts get-caller-identity --query Account --output text
      once: true
---

# AWS CDK Lambda Deployment

This skill focuses on production-ready deployment of AWS Lambda functions defined in CDK. It complements `aws-cdk-development` (which covers general CDK practices) by going deep on Lambda packaging, versioning, traffic shifting, runtime tuning, and observability.

## When to Use This Skill

Use this skill when:

- Adding or refactoring Lambda functions in a CDK stack
- Choosing the right Lambda construct (`NodejsFunction`, `PythonFunction`, `DockerImageFunction`, container image)
- Bundling TypeScript/Python with esbuild or `PythonFunction` with Poetry/Pip
- Publishing immutable versions, aliases, and traffic-shifting deployments
- Reducing cold-start latency (memory tuning, SnapStart, provisioned concurrency, ARM64)
- Wiring observability (Powertools, X-Ray, log retention, structured logs)
- Granting least-privilege IAM permissions to a Lambda
- Sharing dependencies via Lambda layers

## AWS Documentation Requirement

Always verify Lambda runtime support, regional availability, and current quotas using `mcp__aws-mcp__*` or `mcp__*awsdocs*__*` before recommending a runtime, memory, or concurrency setting. The `aws-mcp-setup` skill is loaded as a dependency — if MCP tools are missing, drive the user through that skill first.

## 1. Pick the Right Lambda Construct

| Use case | Construct | Notes |
|---|---|---|
| TypeScript/JavaScript handler | `aws-cdk-lib/aws-lambda-nodejs` `NodejsFunction` | esbuild bundling, tree-shaking, source maps |
| Python handler with deps | `@aws-cdk/aws-lambda-python-alpha` `PythonFunction` | Builds via Docker, supports Pip/Poetry/Pipenv |
| Container image (>250 MB unzipped or custom OS deps) | `aws-cdk-lib/aws-lambda` `DockerImageFunction` | Up to 10 GB image, ECR-backed |
| Pre-built artifact | `lambda.Function` with `Code.fromAsset` | When CI builds the zip |

Default to `NodejsFunction` / `PythonFunction` so bundling lives inside `cdk synth` and is reproducible.

```typescript
import { NodejsFunction, OutputFormat } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Architecture, Runtime, Tracing } from 'aws-cdk-lib/aws-lambda';
import { RetentionDays } from 'aws-cdk-lib/aws-logs';
import { Duration } from 'aws-cdk-lib';

const api = new NodejsFunction(this, 'OrdersApi', {
  entry: 'src/handlers/orders.ts',
  handler: 'handler',
  runtime: Runtime.NODEJS_20_X,
  architecture: Architecture.ARM_64,
  memorySize: 1024,
  timeout: Duration.seconds(10),
  tracing: Tracing.ACTIVE,
  logRetention: RetentionDays.ONE_MONTH,
  bundling: {
    minify: true,
    sourceMap: true,
    target: 'node20',
    externalModules: ['@aws-sdk/*'], // AWS SDK v3 is in the runtime
    format: OutputFormat.ESM,
  },
  environment: {
    POWERTOOLS_SERVICE_NAME: 'orders',
    POWERTOOLS_METRICS_NAMESPACE: 'Orders',
    LOG_LEVEL: 'INFO',
  },
});
```

Follow the resource-naming rule from `aws-cdk-development`: do **not** set `functionName` so multiple environments can coexist.

## 2. Bundling Best Practices

### NodejsFunction

- Mark AWS SDK v3 packages as `externalModules` for Node 18+/20+ runtimes — they ship with the runtime and excluding them shrinks the bundle by megabytes.
- Enable `sourceMap: true` and set `NODE_OPTIONS=--enable-source-maps` so stack traces map back to TypeScript.
- Prefer ESM (`format: OutputFormat.ESM`) for smaller bundles and top-level `await` support.
- Pin `target: 'node20'` (or matching runtime) so esbuild emits compatible syntax.

### PythonFunction

- Use a `requirements.txt`, `Pipfile`, or `pyproject.toml` next to the entry; bundling runs in a Lambda-compatible Docker image automatically.
- Set `bundling: { assetExcludes: ['.venv', '__pycache__', 'tests'] }` to keep the zip lean.

### DockerImageFunction

- Use AWS-provided base images (`public.ecr.aws/lambda/python:3.12`) and multi-stage builds.
- Tag images by content hash (CDK does this automatically with `DockerImageAsset`).

## 3. Versions, Aliases, and Traffic Shifting

New code should always roll out behind an **alias** so that integrations (API Gateway, EventBridge, SNS, SQS) point at a stable target while a new version is canaried.

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Duration } from 'aws-cdk-lib';
import { LambdaDeploymentConfig, LambdaDeploymentGroup } from 'aws-cdk-lib/aws-codedeploy';
import { Alarm, ComparisonOperator } from 'aws-cdk-lib/aws-cloudwatch';

const version = api.currentVersion; // immutable, content-hashed
const live = new lambda.Alias(this, 'Live', {
  aliasName: 'live',
  version,
});

const errorAlarm = new Alarm(this, 'ApiErrorsAlarm', {
  metric: api.metricErrors({ period: Duration.minutes(1) }),
  threshold: 1,
  evaluationPeriods: 2,
  comparisonOperator: ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
});

new LambdaDeploymentGroup(this, 'OrdersDeploy', {
  alias: live,
  deploymentConfig: LambdaDeploymentConfig.CANARY_10PERCENT_5MINUTES,
  alarms: [errorAlarm],
  autoRollback: { failedDeployment: true, deploymentInAlarm: true },
});
```

Deployment-config options:

- `ALL_AT_ONCE` — fastest, no safety margin; only for pre-prod or low-risk paths.
- `LINEAR_10PERCENT_EVERY_1MINUTE` (or 2/3/10 minute variants) — ramps traffic linearly.
- `CANARY_10PERCENT_5MINUTES` (or 10/15/30 minute variants) — send 10% to the new version, hold, then shift the rest if alarms stay green.

Always pair traffic shifting with at least one CloudWatch alarm (errors, throttles, p99 latency). Without alarms, auto-rollback never triggers.

## 4. Cold-Start and Cost Optimisation

- **Architecture**: Default to `Architecture.ARM_64` — ~20% cheaper and often faster.
- **Memory tuning**: Memory sets CPU too. Use the [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) state machine to find the cost/performance sweet spot — verify the latest deployment instructions via `awsdocs` MCP.
- **SnapStart**: Available for Java and (more recently) .NET and Python runtimes — always confirm current runtime coverage via the `awsdocs` MCP before enabling. Turn on with `snapStart: lambda.SnapStartConf.ON_PUBLISHED_VERSIONS` to cut cold starts dramatically.
- **Provisioned concurrency**: Attach to an alias for predictable warm capacity:
  ```typescript
  live.addAutoScaling({ minCapacity: 5, maxCapacity: 50 })
    .scaleOnUtilization({ utilizationTarget: 0.7 });
  ```
- **Keep bundles small** (see §2) — init time is dominated by code size and module graph.
- **Avoid VPC unless needed** — ENI attachment adds cold-start latency. If a VPC is required, use Hyperplane ENIs (default in current Lambda) and right-size subnets.

## 5. Lambda Layers

Use layers when:

- Multiple functions share large dependencies (e.g. `aws-lambda-powertools`, `pydantic`).
- You want to ship binaries (e.g. `ffmpeg`, `chromium`) once.

Define and attach:

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import { Stack } from 'aws-cdk-lib';

const powertools = lambda.LayerVersion.fromLayerVersionArn(
  this,
  'Powertools',
  // Verify the latest ARN/version for your region via awsdocs MCP
  `arn:aws:lambda:${Stack.of(this).region}:017000801446:layer:AWSLambdaPowertoolsTypeScriptV2:14`,
);
api.addLayers(powertools);
```

Layers count against the 250 MB unzipped quota — if you approach it, switch to `DockerImageFunction`.

## 6. IAM, Networking, and Secrets

- Grant least privilege using resource-level helpers:
  ```typescript
  ordersTable.grantReadWriteData(api);
  uploadsBucket.grantPut(api);
  ```
  Never attach `AdministratorAccess` or wildcard `*` resources.
- Pull secrets from Secrets Manager / SSM Parameter Store at runtime via the [Parameters & Secrets Lambda Extension](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets_lambda.html), not as plaintext env vars.
- Tag every function with `Environment`, `Owner`, and `CostCenter` via `Tags.of(this).add(...)` for downstream cost reporting (see `aws-cost-ops`).

## 7. Observability

- Enable `tracing: Tracing.ACTIVE` so X-Ray captures cold starts, downstream calls, and SDK timings.
- Use `logRetention` (default is **never expire** — a silent cost trap). One month is a reasonable default; choose per workload.
- Emit structured logs and EMF metrics with [AWS Lambda Powertools](https://docs.powertools.aws.dev/lambda/typescript/latest/) (TypeScript and Python).
- Add CloudWatch alarms for `Errors`, `Throttles`, `Duration` (p99), and `ConcurrentExecutions` near account quota — see `aws-cost-ops` for alarm patterns.

## 8. Pre-Deployment Validation Workflow

Follow the multi-layer strategy from `aws-cdk-development`, plus Lambda-specific checks:

1. `npm run build` (or `pytest`, etc.) — typecheck and unit tests pass.
2. `cdk synth` with `cdk-nag` aspects enabled — in particular review:
   - **AwsSolutions-L1**: Lambda runtime is current (not deprecated).
   - **AwsSolutions-IAM4 / IAM5**: no managed policies / wildcard resources.
3. Inspect the synthesised template for:
   - `MemorySize` and `Timeout` set deliberately (defaults of 128 MB / 3 s are rarely correct).
   - `LogRetentionInDays` present.
   - `TracingConfig: { Mode: Active }` if X-Ray is expected.
4. `cdk diff` against the target environment — confirm only intended changes.
5. For new alias rollouts, confirm the deployment group references at least one alarm.

A sample post-synth check (extend `scripts/validate-stack.sh`):

```bash
jq -r '.Resources[]
  | select(.Type == "AWS::Lambda::Function")
  | "\(.Properties.FunctionName // "<auto>")\t\(.Properties.MemorySize)\t\(.Properties.Timeout)"' \
  cdk.out/*.template.json
```

## 9. Workflow Summary

1. **Verify** runtime, regional availability, and quotas via `awsdocs` MCP.
2. **Define** the function with `NodejsFunction` / `PythonFunction`, ARM64, sensible memory/timeout, log retention, X-Ray.
3. **Bundle** with esbuild externals or PythonFunction asset excludes; aim for <5 MB zipped where possible.
4. **Wrap** the function in an `Alias` for stable downstream wiring.
5. **Roll out** via `LambdaDeploymentGroup` with canary or linear config and CloudWatch alarms.
6. **Observe** with Powertools, structured logs, and alarms wired to `aws-cost-ops` dashboards.
7. **Validate** via `cdk-nag`, `cdk synth`, and `cdk diff` before `cdk deploy`.

## Related Skills and References

- **`aws-cdk-development`** (auto-loaded dependency): general CDK principles, validation strategy, and patterns reference.
- **`aws-cost-operations`**: cost estimation for Lambda + alarms patterns.
- **`aws-serverless-eda`**: when the function is part of a wider event-driven workflow.
- **AWS Documentation MCP**: always the source of truth for runtime support, quotas, and pricing.

## GitHub Actions Integration

If the repository defines `.github/workflows/`, ensure the Lambda-related jobs (build, test, `cdk synth`) pass before committing. Cold-start and bundle-size regressions are easy to introduce — consider adding a workflow step that asserts the synthesised zip stays under a budget you choose.
