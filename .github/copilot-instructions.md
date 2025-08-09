# Copilot Instructions for retail-store-sample-app

## Big Picture Architecture
- This is a microservices-based retail store app for AWS EKS, using GitOps and Infrastructure as Code.
- Major services: UI (Java), Catalog (Go), Cart (Java), Orders (Java), Checkout (Node.js).
- Each service is in its own folder under `src/`, with its own Dockerfile, Helm chart (`chart/`), and README.
- Data persistence varies: Cart uses DynamoDB, Catalog/Orders use MySQL, Checkout uses Redis.
- Infrastructure is managed via Terraform (`terraform/`), including EKS, networking, security, and ArgoCD setup.
- ArgoCD watches the repo and deploys Helm charts to EKS automatically.

## Developer Workflows
- **Build**: Each service has its own build system (Maven for Java, Go for Catalog, npm for Checkout).
  - Example: `./mvnw clean package` in `src/cart`, `go build` in `src/catalog`, `npm run build` in `src/checkout`.
- **Test**: Run tests using the service's build tool (e.g., `./mvnw test`, `go test ./...`, `npm test`).
- **Containerization**: Build Docker images using the provided Dockerfile in each service directory.
- **Helm Charts**: Deploy/update services via Helm charts in each service's `chart/` folder. Key templates: `deployment.yaml`, `service.yaml`, and values in `values.yaml`.
- **GitOps**: Push changes to GitHub; ArgoCD auto-syncs and deploys to EKS. No manual `kubectl apply` needed for app resources.
- **Terraform**: Use to provision/update infrastructure. Example commands: `terraform init`, `terraform apply` in `terraform/`.

## Project-Specific Patterns & Conventions
- **Service Endpoints**: Each service exposes utility endpoints for chaos testing (`/chaos/status/{code}`, `/chaos/latency/{delay}`, `/chaos/health`).
- **Environment Variables**: Each service is configured via env vars (see each service's README for details).
- **Persistence Providers**: Services support multiple backends (e.g., Cart: in-memory or DynamoDB, Catalog: in-memory or MySQL).
- **Helm Values**: Image tags and resource settings are managed via `values.yaml` and updated by CI/CD pipelines.
- **Branch Strategy**: `main` for public/simple deployments, `gitops` for production with private images and CI/CD.

## Integration Points & Communication
- **Service Discovery**: Services communicate via Kubernetes Service DNS (e.g., `cart.retail-store.svc.cluster.local`).
- **External Dependencies**: DynamoDB, MySQL, Redis, SQS, RabbitMQ (see service READMEs for config).
- **CI/CD**: GitHub Actions and Jenkins pipelines build, test, push images, update Helm charts, and trigger deployments.
- **Monitoring/Observability**: Optional integration with OpenTelemetry and metrics (see Helm chart annotations).

## Key Files & Directories
- `src/<service>/` — Source code, Dockerfile, Helm chart, README for each service.
- `src/<service>/chart/` — Helm chart templates and values.
- `terraform/` — Infrastructure as Code for AWS EKS and dependencies.
- `argocd/applications/` — ArgoCD Application manifests for GitOps deployment.
- `README.md` — Project overview, architecture, and workflow details.

## Example: Deploying a Change
1. Edit code in `src/cart/` and update as needed.
2. Build and test locally (`./mvnw clean package && ./mvnw test`).
3. Build/push Docker image, update `values.yaml` image tag.
4. Commit/push changes to GitHub.
5. ArgoCD auto-syncs and deploys the new version to EKS.

---

For service-specific patterns, see each service's README in `src/<service>/README.md`.
For infrastructure, see `terraform/README.md`.
For deployment, see ArgoCD manifests in `argocd/applications/`.

If any section is unclear or missing, please provide feedback to improve these instructions.
