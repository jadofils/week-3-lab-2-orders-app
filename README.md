# Onward Logistics order application

Lab 2 (Week 3, Compute & Delivery on AWS), by Jado Fils SEZIKEYE. This is the **application repository**. The infrastructure, the release pipeline and the docs are in [week-3-lab-2-aws-containerization](https://github.com/jadofils/week-3-lab-2-aws-containerization).

One Spring Boot application, with the order API and the browser UI packaged in one container image. It runs on Amazon ECS with Fargate, in private subnets behind an Application Load Balancer.

## What is here

| Path | What it is |
| --- | --- |
| [`app/`](app/README.md) | The Spring Boot application (Java 21): API, UI, tests |
| [`deploy/`](deploy/README.md) | The deployment files CodeDeploy uses: `taskdef.json` and `appspec.yaml`. CD zips them into the S3 deployment bundle. |
| [`Dockerfile`](Dockerfile) | Two-stage build: Maven builds the JAR, then a Java 21 JRE image runs it as a non-root user on port 8080 |
| [`.github/workflows/ci.yml`](.github/workflows/ci.yml) | **CI**, on every pull request and push: credential scan, `mvn verify`, and a check that the image builds. No AWS access. |
| [`.github/workflows/build-and-push.yml`](.github/workflows/build-and-push.yml) | **CD**, only after CI passes on `main`: builds the tested commit and pushes it to Amazon ECR with OIDC |

## How a change reaches customers

Merging to `main` is the only manual step.

```text
merge to main
  └─▶ CI (ci.yml): credential scan · mvn verify · image builds          ── fails → nothing is released
        └─▶ CD (build-and-push.yml), only if CI passed: docker build of the same commit
              └─▶ OIDC token → AWS role onward-dev-github-actions-role (no stored keys)
                    └─▶ ECR onward-dev-orders: sha-<commit> (immutable)
                    └─▶ S3 deploy-bundle.zip (deploy/taskdef.json + deploy/appspec.yaml)
                    └─▶ ECR: move latest last
                          └─▶ EventBridge rule (push of latest) → CodePipeline → CodeDeploy blue/green
```

1. **CI** (`ci.yml`) runs on every pull request and every push: gitleaks over the full history, a check that no workflow uses AWS keys, `mvn verify`, and a `docker build` without pushing. It has no AWS access.
2. When CI passes on a push to `main`, it triggers **CD** (`build-and-push.yml`), which checks out exactly the commit CI tested. CD requests a GitHub OIDC token and exchanges it for temporary AWS credentials on the role in the `AWS_ROLE_ARN` variable. That role trusts only `repo:jadofils@114186160/week-3-lab-2-orders-app@1460005073:ref:refs/heads/main` (GitHub's immutable subject: owner and repository names with their permanent numeric IDs).
3. It pushes the image as `sha-<commit>`, uploads `deploy-bundle.zip` (the two files in `deploy/`, with the account ID and Region filled in) to the S3 bucket `onward-dev-orders-deploy-bundle-<account-id>`, and then moves `latest` to the new image.
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
