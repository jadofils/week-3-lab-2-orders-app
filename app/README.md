# Onward order application

A single full-stack Spring Boot application. The REST backend and the browser UI are packaged in **one executable JAR**. The UI is plain HTML, CSS and JavaScript that calls the API with AJAX (`fetch`), with no server-side templates.

## What the page shows

- Your **full name** and the **lab name** ("Lab 2 - Delivering Order System Updates Safely").
- **Which server answered**: the ID, Availability Zone and private IP of the container that answered. On Fargate these come from the ECS task metadata endpoint (`ECS_CONTAINER_METADATA_URI_V4`), which replaces Lab 1's EC2 instance metadata (IMDSv2). Off AWS, the app shows the host name instead.
- **Which server is answering?** This panel polls once a second through the load balancer and counts answers per server, so you can watch servers join and leave as capacity changes.
- **Sales-day CPU spike.** This panel starts a bounded CPU spike on the servers that receive the requests, which makes Auto Scaling add servers.
- Sample orders, held in memory on each server.

## API

| Method and path | Purpose |
| --- | --- |
| `GET /health` | Load balancer health check. Returns `{status, server, availabilityZone}`. |
| `GET /api/info` | Name, lab name, app name, server ID, zone and private IP |
| `GET /api/orders`, `POST /api/orders` | List or create sample orders |
| `POST /api/load/spike` `{"seconds": 420}` | Start a CPU spike on the answering server |
| `POST /api/load/stop`, `GET /api/load/status` | Stop a spike, or check its state |

**Why the spike trigger is safe to expose:**
- Its length is limited to 30–600 seconds.
- Each server runs at most one spike at a time.
- It takes no other input.
- It can be switched off with `LOAD_DEMO_ENABLED=false`.

See the [live-review checks](https://github.com/jadofils/week-3-lab-2-aws-containerization/blob/main/docs/demo/README.md#5-the-service-adds-containers-under-load-and-removes-them-afterwards) for how the spike is used to show scaling.

## Structure (layered MVC, SOLID)

```text
src/main/java/com/onward/orders/
├── config/        @ConfigurationProperties: app.info.*, app.orders.*, app.identity.*, app.load.*
├── controller/    Thin REST controllers: Order, Info, Health, Load
├── dto/           request/ and response/ JSON shapes
├── exception/     BadRequestException family + @RestControllerAdvice returning ProblemDetail
├── mapper/        Domain model to response DTO
├── model/         Order, OrderStatus, ServerIdentity
├── repository/    OrderRepository + in-memory implementation
└── service/       Interfaces + impl/: OrderServiceImpl, InfoServiceImpl, DefaultOrderValidator,
                   EcsTaskMetadataServerIdentityService, CpuSpikeLoadGeneratorService
src/main/resources/static/   index.html, css/styles.css, js/api.js (AJAX) · view.js (DOM) · app.js (controller)
```

## Run and test

```powershell
mvn spring-boot:run   # http://localhost:8080
mvn verify            # 20 tests: validator, repository, CPU spike limits, ECS task identity, full HTTP contract
```

## Release

Merging to `main` is the only manual step.

1. **CI**, [`.github/workflows/ci.yml`](../.github/workflows/ci.yml), runs on every pull request and push: a credential scan, `mvn verify`, and a check that the image builds. It has no AWS access.
2. **CD**, [`.github/workflows/build-and-push.yml`](../.github/workflows/build-and-push.yml), starts only after CI passes on `main`. It builds the commit CI tested from the [`Dockerfile`](../Dockerfile) (Java 21 JRE, non-root, port 8080).
3. It signs in to AWS with **GitHub OIDC**, assuming the role in the `AWS_ROLE_ARN` repository variable. No AWS keys are stored in GitHub.
4. It pushes the image to the ECR repository `onward-dev-orders`, tagged `sha-<commit>` (immutable) and `latest`.
5. The `latest` push fires an EventBridge rule. CodePipeline and CodeDeploy then release it blue/green, with no downtime and automatic rollback on errors.

The pipeline and everything it deploys live in the infrastructure repository. See its [architecture page](https://github.com/jadofils/week-3-lab-2-aws-containerization/blob/main/docs/architecture/README.md#4-how-a-change-reaches-customers).
