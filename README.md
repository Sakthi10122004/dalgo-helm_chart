# Dalgo Helm Chart

This repository contains a Helm chart to deploy the Dalgo platform (backend, webapp, databases, Celery workers, and supporting services) on Kubernetes (k3s/k3d/local clusters) and documentation for local Docker-based development.

## Quick links

- Chart: [dalgo/Chart.yaml](dalgo/Chart.yaml)
- Default values: [dalgo/values.yaml](dalgo/values.yaml)
- K3s / Helm deploy guide: [Docs/k3s_deployment.md](Docs/k3s_deployment.md)
- Local Docker deploy guide: [Docs/docker_deployment.md](Docs/docker_deployment.md)

## Requirements

- Kubernetes cluster (k3s, k3d, minikube, or managed cluster)
- Helm v3
- kubectl configured to target your cluster
- Docker (for building local images when using k3s/k3d)

## Quick Start (Helm)

1. Review and adjust configuration in [dalgo/values.yaml](dalgo/values.yaml).
2. Install the chart from the repository root:

```bash
helm install dalgo ./dalgo -n dalgo --create-namespace
# or provide a custom values file:
helm install my-release ./dalgo -f dalgo/values.yaml -n dalgo --create-namespace
```

3. Monitor the deployment:

```bash
kubectl get pods -n dalgo -w
```

Note: The chart runs an initialization job (init-job) to perform DB migrations and seed data — wait for that job to complete before relying on the API.

## Configuration

All configurable values live in [dalgo/values.yaml](dalgo/values.yaml). Key sections include:

- `images` — container images for `backend`, `webapp`, `postgres`, `redis`, etc.
- `postgres` — credentials, persistence and storage settings
- `redis` — enable/disable and port
- `airbyte` — settings for connecting to Airbyte (in-cluster or external)
- `ingress` — host, annotations and TLS settings

Adjust these values for your environment before installing/upgrading the chart.

## Chart structure and templates

Primary templates located under `dalgo/templates/`:

- [dalgo/templates/backend.yaml](dalgo/templates/backend.yaml)
- [dalgo/templates/webapp.yaml](dalgo/templates/webapp.yaml)
- [dalgo/templates/init-job.yaml](dalgo/templates/init-job.yaml)
- [dalgo/templates/postgres.yaml](dalgo/templates/postgres.yaml)
- [dalgo/templates/redis.yaml](dalgo/templates/redis.yaml)
- [dalgo/templates/airbyte-platform.yaml](dalgo/templates/airbyte-platform.yaml)
- [dalgo/templates/ingress.yaml](dalgo/templates/ingress.yaml)
- [dalgo/templates/celery.yaml](dalgo/templates/celery.yaml)
- [dalgo/templates/prefect-proxy.yaml](dalgo/templates/prefect-proxy.yaml)

See the template files for service-level details and hooks.

## Local development and Docker

For local development and Docker-based deployment, see [Docs/docker_deployment.md](Docs/docker_deployment.md). That guide covers:

- Running Postgres and Redis via Docker
- Starting Airbyte via its Docker Compose/start script
- Running a lightweight Prefect proxy mock for development
- Running the backend and frontend locally

## K3s / k3d deployment

For local Kubernetes using k3s/k3d, see [Docs/k3s_deployment.md](Docs/k3s_deployment.md). The guide explains:

- Creating a k3d cluster and port mappings
- Building/importing local Docker images into k3d
- Helm install/upgrade commands and tips

## Useful Helm commands

```bash
# Install
helm install dalgo ./dalgo -n dalgo --create-namespace

# Upgrade after editing values / templates
helm upgrade dalgo ./dalgo -n dalgo

# Uninstall
helm uninstall dalgo -n dalgo
```

## Contributing

1. Open an issue describing the change or bug.
2. Create a branch and a clear PR with the change.

## License

This repository does not include a license file. Add a `LICENSE` if you intend to publish or share under an open-source license.

---
Generated README for the Dalgo Helm chart and deployment documentation.
