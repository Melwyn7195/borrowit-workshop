---
title : "Container image and registry"
date : 2026-07-28
weight : 1
chapter : false
pre : " <b> 3.6.1 </b> "
---

`BorrowitApp` will not deploy until an image exists in ECR, so this comes first.

The application's Dockerfile is not the subject here — what matters is how the
image reaches ECR, how ECS is authorised to pull it, and how versioning affects
your ability to roll back.

#### Authenticate Docker to ECR

ECR uses **IAM** rather than stored registry credentials. `get-login-password`
exchanges your AWS identity for a short-lived token:

```powershell
$env:ACCOUNT  = (aws sts get-caller-identity --query Account --output text)
$env:REGION   = "ap-southeast-1"
$env:REGISTRY = "$env:ACCOUNT.dkr.ecr.$env:REGION.amazonaws.com"

aws ecr get-login-password --region $env:REGION | docker login --username AWS --password-stdin $env:REGISTRY
```

Expected output: `Login Succeeded`.

The token expires in 12 hours. Nothing long-lived is stored on the machine, which
is the main reason ECR was chosen over Docker Hub in
[3.3.4](../../3.3-Architecture/3.3.4-delivery-and-secrets/).

#### Build for the right architecture

```powershell
cd ..\Renting-Online-Backend-main
docker build --platform linux/amd64 -t borrowit-be:v1 .
```

{{% notice warning %}}
`--platform linux/amd64` is not optional. The Fargate task definition targets
X86_64. An image built on an ARM machine without this flag pushes successfully and
then fails at task start with `exec format error` — a failure that appears at
runtime, not at build or deploy time.
{{% /notice %}}

<!-- TODO(prose): Fargate can run ARM64 natively via runtimePlatform, which is
     cheaper than X86_64. If you considered it and stayed on X86_64, say why. -->

#### Tag and push

```powershell
docker tag borrowit-be:v1 "$env:REGISTRY/borrowit-be:v1"
docker push "$env:REGISTRY/borrowit-be:v1"
```

{{% notice note %}}
Use real version tags rather than `latest`. With `latest`, there is no way to roll
back to a known-good image, because the tag no longer points at it. The stack reads
the tag from context: `npx cdk deploy BorrowitApp -c imageTag=v2`.
{{% /notice %}}

#### How ECS is authorised to pull

Two IAM roles are involved, and confusing them is a common source of failures:

| Role | Used by | Needs |
|---|---|---|
| **Task execution role** | The ECS agent, before the container starts | Pull from ECR, read Secrets Manager, write to CloudWatch Logs |
| **Task role** | The application code, while running | Whatever AWS APIs the app itself calls — here, `s3:PutObject` on the uploads bucket |

```typescript
props.uploadsBucket.grantPut(taskDefinition.taskRole);
```

CDK creates and populates the execution role automatically. The task role starts
empty and is granted exactly one permission — the application uploads images to
S3 and does nothing else with AWS.

<!-- TODO(prose): worth a sentence - this replaces static access keys in the
     application with a role assumed at runtime, so there are no AWS credentials
     anywhere in the codebase. -->

#### Verify in ECR

<!-- SCREENSHOT: /images/3-Workshop/3.6-Compute/3.6.1-image-and-registry/ecr-image.png
     Image listed with tag, size, pushed-at time. -->

`imageScanOnPush: true` runs a basic vulnerability scan automatically.

<!-- TODO(prose): report what the scan actually found. If the base image has
     findings, say so and say what you would do - rebuilding on a newer base tag
     is usually the whole answer. Do not claim a clean scan you did not get. -->
