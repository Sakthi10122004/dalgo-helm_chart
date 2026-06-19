# Dalgo Helm Chart - Multi-Tenant Deployment Guide

## Overview

This Helm chart is now configured to support deploying multiple instances of Dalgo in the same Kubernetes namespace, with each instance having its own isolated resources. This is ideal for:

- **Multi-organization deployments**: Each organization gets its own Dalgo instance
- **Multi-tenant SaaS**: Deploy separate instances per customer/tenant
- **Development and testing**: Run multiple versions simultaneously

## How Multi-Tenancy Works

The chart uses Helm Release names to isolate resources for each instance. Each release creates separate:

- **Databases & PVCs**: Unique PostgreSQL databases and persistent volumes for each organization
- **Services**: Isolated backend, frontend, and helper services
- **Deployments**: Separate pods for backend, celery workers, redis, etc.
- **Configurations & Secrets**: Organization-specific config and credentials

## Deployment Architecture

```
Namespace (org-team-dev)
├── Release: dalgo-org1
│   ├── Deployments
│   │   ├── dalgo-org1-backend
│   │   ├── dalgo-org1-celery-worker-default
│   │   ├── dalgo-org1-celery-worker-canvas
│   │   ├── dalgo-org1-postgres
│   │   ├── dalgo-org1-redis
│   │   ├── dalgo-org1-webapp
│   │   ├── dalgo-org1-prefect-proxy
│   │   └── [optional] dalgo-org1-airbyte-* (if deployInCluster)
│   ├── Services
│   │   ├── dalgo-org1-backend
│   │   ├── dalgo-org1-postgres
│   │   ├── dalgo-org1-redis
│   │   └── ...
│   ├── PVCs
│   │   ├── dalgo-org1-postgres-pvc
│   │   └── [optional] dalgo-org1-airbyte-db-pvc
│   ├── ConfigMaps & Secrets
│   │   ├── dalgo-org1-config
│   │   ├── dalgo-org1-secrets
│   │   └── dalgo-org1-webapp-config
│   └── Jobs
│       └── dalgo-org1-initdb
│
└── Release: dalgo-org2
    ├── Deployments
    │   ├── dalgo-org2-backend
    │   ├── dalgo-org2-celery-worker-default
    │   ├── dalgo-org2-postgres
    │   └── ...
    ├── Services
    └── PVCs
        ├── dalgo-org2-postgres-pvc
        └── ...
```

## Quick Start: Deploy Multiple Instances

### Prerequisites

```bash
# Install Helm (if not already installed)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Have a Kubernetes cluster ready
kubectl cluster-info
```

### 1. Create a Namespace for Your Deployments

```bash
kubectl create namespace org-team-dev
```

### 2. Deploy First Organization Instance

```bash
# Create a custom values file for org1
cat > values-org1.yaml << 'EOF'
environment: production
debug: "False"
djangoSecret: "org1-django-secret-change-me-in-production"
allowedHosts: "org1.example.com"
corsAllowedOrigins: "https://org1.example.com"

postgres:
  enabled: true
  user: postgres
  password: "org1-db-password"  # Change in production
  db: dalgo_org1
  persistence:
    size: 5Gi

webapp:
  replicaCount: 2
  backendUrl: "https://org1-api.example.com"
  nextAuthSecret: "org1-nextauth-secret-change-me"

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: org1.example.com
      paths:
        - path: /
          pathType: Prefix

airbyte:
  deployInCluster: false  # Set to true if you want to deploy Airbyte
  host: "airbyte.company.internal"
  port: 8000
EOF

# Deploy the first instance
helm install dalgo-org1 ./dalgo \
  -f values-org1.yaml \
  --namespace org-team-dev \
  --create-namespace
```

### 3. Deploy Second Organization Instance (Same Namespace)

```bash
# Create values for org2
cat > values-org2.yaml << 'EOF'
environment: production
debug: "False"
djangoSecret: "org2-django-secret-change-me-in-production"
allowedHosts: "org2.example.com"
corsAllowedOrigins: "https://org2.example.com"

postgres:
  enabled: true
  user: postgres
  password: "org2-db-password"
  db: dalgo_org2
  persistence:
    size: 5Gi

webapp:
  replicaCount: 2
  backendUrl: "https://org2-api.example.com"
  nextAuthSecret: "org2-nextauth-secret-change-me"

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: org2.example.com
      paths:
        - path: /
          pathType: Prefix
EOF

# Deploy the second instance
helm install dalgo-org2 ./dalgo \
  -f values-org2.yaml \
  --namespace org-team-dev
```

## Verification

### Check Deployed Resources

```bash
# List all Helm releases
helm list -n org-team-dev

# Check resources for org1
kubectl get all -n org-team-dev -l app.kubernetes.io/instance=dalgo-org1

# Check PVCs
kubectl get pvc -n org-team-dev

# Check logs
kubectl logs -n org-team-dev deployment/dalgo-org1-backend
```

### Access the Applications

```bash
# Port-forward for local testing (if no ingress)
# For org1 backend
kubectl port-forward -n org-team-dev svc/dalgo-org1-backend 8002:8002

# For org1 webapp
kubectl port-forward -n org-team-dev svc/dalgo-org1-webapp 3000:3000

# For org2 backend
kubectl port-forward -n org-team-dev svc/dalgo-org2-backend 8003:8002

# For org2 webapp
kubectl port-forward -n org-team-dev svc/dalgo-org2-webapp 3001:3000
```

## Data Isolation

Each release instance has:

- **Separate Database**: Each organization has its own PostgreSQL database
- **Separate PVC**: Each organization has its own persistent volume for database data
- **Isolated Secrets**: Unique API keys, tokens, and passwords per organization
- **Isolated Configuration**: Organization-specific environment variables
- **No Cross-Talk**: Services communicate only within the same release via DNS names like `dalgo-org1-postgres`

## Scaling & Management

### Update an Instance

```bash
# Update only org1 with new values
helm upgrade dalgo-org1 ./dalgo \
  -f values-org1.yaml \
  --namespace org-team-dev
```

### Scale Replicas for a Specific Organization

```bash
# Scale org1 backend to 3 replicas
kubectl scale deployment dalgo-org1-backend -n org-team-dev --replicas=3

# Scale org2 celery workers
kubectl scale deployment dalgo-org2-celery-worker-default -n org-team-dev --replicas=5
```

### Delete an Organization Instance

```bash
# Delete org1 completely (including PVCs if configured)
helm uninstall dalgo-org1 --namespace org-team-dev

# Note: PVCs are not deleted by default to prevent data loss
# To delete PVCs as well:
kubectl delete pvc -n org-team-dev -l app.kubernetes.io/instance=dalgo-org1
```

## Resource Naming Convention

All resources follow this naming pattern: `{release-name}-{component}`

| Component | Resource Name | Example |
|-----------|--------------|---------|
| Backend | `{release}-backend` | `dalgo-org1-backend` |
| Database | `{release}-postgres` | `dalgo-org1-postgres` |
| Database PVC | `{release}-postgres-pvc` | `dalgo-org1-postgres-pvc` |
| Redis | `{release}-redis` | `dalgo-org1-redis` |
| Celery Worker (default) | `{release}-celery-worker-default` | `dalgo-org1-celery-worker-default` |
| Celery Worker (canvas) | `{release}-celery-worker-canvas` | `dalgo-org1-celery-worker-canvas` |
| Webapp | `{release}-webapp` | `dalgo-org1-webapp` |
| Prefect Proxy | `{release}-prefect-proxy` | `dalgo-org1-prefect-proxy` |
| Airbyte DB | `{release}-airbyte-db` | `dalgo-org1-airbyte-db` |
| Airbyte DB PVC | `{release}-airbyte-db-pvc` | `dalgo-org1-airbyte-db-pvc` |

## Important Configuration Notes

### 1. **Database Credentials**

Always use different credentials for each organization:

```yaml
postgres:
  user: postgres
  password: "UNIQUE_PASSWORD_FOR_THIS_ORG"  # Change per instance
  db: dalgo_org_name  # Unique database name
```

### 2. **Secrets Management**

```yaml
djangoSecret: "UNIQUE_SECRET_PER_ORG"
webapp:
  nextAuthSecret: "UNIQUE_SECRET_PER_ORG"
airbyte:
  token: "BASE64_ENCODED_UNIQUE_TOKEN"
```

### 3. **Ingress Configuration**

Each organization should have its own hostname:

```yaml
ingress:
  enabled: true
  hosts:
    - host: org1.example.com
      paths:
        - path: /
```

### 4. **Persistence (PVC) Configuration**

Each organization gets its own PVC automatically. Customize per organization:

```yaml
postgres:
  persistence:
    enabled: true
    size: 5Gi  # Adjust based on expected data volume
    storageClass: ""  # Use default or specify a storage class
```

For Airbyte (when `deployInCluster: true`):

```yaml
airbyte:
  deployInCluster: true
  persistence:
    size: 20Gi
```

## Monitoring and Debugging

### Check Resource Usage

```bash
# Monitor CPU and memory usage
kubectl top nodes -n org-team-dev
kubectl top pods -n org-team-dev

# Check PVC usage
kubectl exec -it deployment/dalgo-org1-postgres -n org-team-dev -- df -h /var/lib/postgresql/data
```

### View Logs

```bash
# Backend logs
kubectl logs -f -n org-team-dev deployment/dalgo-org1-backend --tail=100

# Database initialization
kubectl logs -n org-team-dev job/dalgo-org1-initdb

# Celery worker logs
kubectl logs -f -n org-team-dev deployment/dalgo-org1-celery-worker-default
```

### Describe Resources

```bash
# Get detailed info about PVC
kubectl describe pvc dalgo-org1-postgres-pvc -n org-team-dev

# Get detailed info about deployment
kubectl describe deployment dalgo-org1-backend -n org-team-dev
```

## Troubleshooting

### Issue: Pods are stuck in Pending state

```bash
# Check if PVC is bound
kubectl get pvc -n org-team-dev
kubectl describe pvc dalgo-org1-postgres-pvc -n org-team-dev

# Check node capacity
kubectl describe nodes
```

### Issue: Database migration failed

```bash
# Check init job logs
kubectl logs -n org-team-dev job/dalgo-org1-initdb

# Re-run the init job if needed
kubectl delete job dalgo-org1-initdb -n org-team-dev
helm upgrade dalgo-org1 ./dalgo -f values-org1.yaml --namespace org-team-dev
```

### Issue: Services can't connect to database

```bash
# Verify database pod is running
kubectl get pods -n org-team-dev -l app=dalgo-org1-postgres

# Test DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -n org-team-dev -- \
  nslookup dalgo-org1-postgres

# Verify service exists
kubectl get svc -n org-team-dev dalgo-org1-postgres
```

## Best Practices

1. **Use Separate Namespace Per Environment**
   ```bash
   kubectl create namespace org-team-dev
   kubectl create namespace org-team-staging
   kubectl create namespace org-team-prod
   ```

2. **Version Your Values Files**
   ```bash
   git checkout values-org1-v1.0.yaml
   git checkout values-org1-v1.1.yaml
   ```

3. **Backup Important Data**
   ```bash
   # Backup org1 database
   kubectl exec -n org-team-dev deployment/dalgo-org1-postgres -- \
     pg_dump -U postgres dalgo_org1 > dalgo_org1_backup.sql
   ```

4. **Use Resource Requests/Limits**
   ```yaml
   # In values.yaml, add:
   resources:
     requests:
       memory: "512Mi"
       cpu: "250m"
     limits:
       memory: "1Gi"
       cpu: "500m"
   ```

5. **Monitor PVC Capacity**
   - Set up alerts for PVC usage nearing limits
   - Plan for storage growth per organization

## Support

For issues or questions, please refer to:
- Helm Chart Repository: [GitHub URL]
- Dalgo Documentation: [Documentation URL]
- Kubernetes Documentation: https://kubernetes.io/docs/
