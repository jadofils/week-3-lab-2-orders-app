# Onward Logistics order application

Lab 2 (Week 3, Compute & Delivery on AWS), by Jado Fils SEZIKEYE. This is the **application repository**. The infrastructure, the release pipeline and the docs are in [week-3-lab-2-aws-containerization](https://github.com/jadofils/week-3-lab-2-aws-containerization).

One Spring Boot application, with the order API and the browser UI packaged in one container image. It runs on Amazon ECS with Fargate, in private subnets behind an Application Load Balancer.

## What is here

| Path | What it is |
| --- | --- |
| [`app/`](app/README.md) | The Spring Boot application (Java 21): API, UI, tests |
| [`Dockerfile`](Dockerfile) | Two-stage build: Maven builds the JAR, then a Java 21 JRE image runs it as a non-root user on port 8080 |
| [`.github/workflows/build-and-push.yml`](.github/workflows/build-and-push.yml) | Tests every pull request; on `main`, builds the image and pushes it to Amazon ECR with OIDC |

## How a change reaches customers

Merging to `main` is the only manual step.

```text
merge to main
  └─▶ GitHub Actions: mvn verify → docker build
        └─▶ OIDC token → AWS role onward-dev-github-actions-role (no stored keys)
              └─▶ ECR onward-dev-orders: sha-<commit> (immutable) + latest
                    └─▶ EventBridge rule (push of latest) → CodePipeline → CodeDeploy blue/green
```

1. The **Test** job runs `mvn verify` on every pull request and every push.
2. On `main`, the **Build and push** job requests a GitHub OIDC token and exchanges it for temporary AWS credentials on the role in the `AWS_ROLE_ARN` variable. That role trusts only `repo:jadofils@114186160/week-3-lab-2-orders-app@1460005073:ref:refs/heads/main` (GitHub's immutable subject: owner and repository names with their permanent numeric IDs).
3. It pushes the image as `sha-<commit>` and then moves `latest` to it.
4. The `latest` push fires the EventBridge rule in the infrastructure stack. CodePipeline pins the image by digest, and CodeDeploy shifts traffic 10% → 100% with automatic rollback on errors.

## One-time setup

After the infrastructure repository's `onward-dev-registry` stack exists, open *Settings → Secrets and variables → Actions → **Variables*** and add:

| Variable | Value |
| --- | --- |
| `AWS_ROLE_ARN` | Output `GitHubActionsRoleArn` of the `onward-dev-registry` stack |
| `AWS_REGION` | `eu-west-1` |

These are variables, not secrets: neither is a credential. Then run *Actions → Build and push image → Run workflow* on `main` to push the first image.

## Run locally

```powershell
cd app
mvn spring-boot:run       # http://localhost:8080
mvn verify                # tests

# or as the container that runs on Fargate
docker build -t onward-orders .
docker run --rm -p 8080:8080 onward-orders
```

Off ECS, the page shows the host name instead of the task ID and zone.
