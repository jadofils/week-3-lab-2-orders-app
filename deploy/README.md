# Deployment files

CodeDeploy releases each new image blue/green using these two files. On every release, CD (`.github/workflows/build-and-push.yml`) zips them into `deploy-bundle.zip` and uploads it to the S3 bucket `onward-dev-orders-deploy-bundle-<account-id>`. CodePipeline reads the bundle from S3, and the image from ECR.

| File | What it is |
| --- | --- |
| [`taskdef.json`](taskdef.json) | The ECS task definition for each release: CPU and memory, roles, port, environment and logging. |
| [`appspec.yaml`](appspec.yaml) | Tells CodeDeploy which container (`orders`) and port (`8080`) receive traffic from the load balancer. |

## Placeholders

| Placeholder | Filled by | With |
| --- | --- | --- |
| `<AWS_ACCOUNT_ID>`, `<AWS_REGION>` | CD, before zipping | The account and Region CD signed in to, so no account ID is stored in Git |
| `<IMAGE1_NAME>` | CodeDeploy | The exact image digest of the release (from the pipeline's ECR source) |
| `<TASK_DEFINITION>` | CodeDeploy | The new task definition revision it registers from `taskdef.json` |

## Keep in step with the infrastructure

The role names, log group and container name follow the `onward-dev-*` names created by the infrastructure stacks. CPU and memory here are what every release runs; the service stack's own task definition (`infrastructure/templates/3-service.yaml`) is only used when the service is first created. Change both together.
