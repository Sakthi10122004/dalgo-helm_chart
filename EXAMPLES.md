# Multi-Tenant Helm Deployment Examples

This file contains ready-to-use configuration examples for deploying multiple Dalgo instances in the same Kubernetes namespace.

## Example 1: Two Organizations with Ingress

### Namespace Setup
```bash
kubectl create namespace saas-platform
```

### Organization 1 Values (values-acme.yaml)
```yaml
environment: production
debug: "False"
djangoSecret: "acme-django-secret-$(openssl rand -hex 16)"
allowedHosts: "acme.example.com,api-acme.example.com"
corsAllowedOrigins: "https://acme.example.com"

postgres:
  enabled: true
  user: postgres
  password: "acme-db-pwd-$(openssl rand -hex 12)"
  db: dalgo_acme
  persistence:
    enabled: true
    size: 10Gi
    storageClass: "fast-ssd"

redis:
  enabled: true
  port: 6379

webapp:
  enabled: true
  replicaCount: 2
  port: 3000
  nextAuthSecret: "acme-nextauth-secret-$(openssl rand -hex 16)"
  backendUrl: "https://api-acme.example.com"
  websocketUrl: "wss://api-acme.example.com"
  nextAuthUrl: "https://acme.example.com"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: acme.example.com
      paths:
        - path: /
          pathType: Prefix
    - host: api-acme.example.com
      paths:
        - path: /
          pathType: Prefix

airbyte:
  deployInCluster: false
  host: "airbyte.company.internal"
  port: 8000
  token: "$(echo -n 'airbyte:password' | base64)"
```

### Organization 2 Values (values-globex.yaml)
```yaml
environment: production
debug: "False"
djangoSecret: "globex-django-secret-$(openssl rand -hex 16)"
allowedHosts: "globex.example.com,api-globex.example.com"
corsAllowedOrigins: "https://globex.example.com"

postgres:
  enabled: true
  user: postgres
  password: "globex-db-pwd-$(openssl rand -hex 12)"
  db: dalgo_globex
  persistence:
    enabled: true
    size: 15Gi
    storageClass: "fast-ssd"

redis:
  enabled: true
  port: 6379

webapp:
  enabled: true
  replicaCount: 3
  port: 3000
  nextAuthSecret: "globex-nextauth-secret-$(openssl rand -hex 16)"
  backendUrl: "https://api-globex.example.com"
  websocketUrl: "wss://api-globex.example.com"
  nextAuthUrl: "https://globex.example.com"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: globex.example.com
      paths:
        - path: /
          pathType: Prefix
    - host: api-globex.example.com
      paths:
        - path: /
          pathType: Prefix

airbyte:
  deployInCluster: false
  host: "airbyte.company.internal"
  port: 8000
  token: "$(echo -n 'airbyte:password' | base64)"
```

### Deployment Commands
```bash
# Deploy ACME organization
helm install dalgo-acme ./dalgo \
  -f values-acme.yaml \
  --namespace saas-platform \
  --create-namespace

# Deploy Globex organization
helm install dalgo-globex ./dalgo \
  -f values-globex.yaml \
  --namespace saas-platform

# Verify deployments
helm list -n saas-platform
kubectl get pods -n saas-platform
kubectl get pvc -n saas-platform
kubectl get svc -n saas-platform
```

---

## Example 2: Development, Staging, and Production

### Development Namespace
```bash
kubectl create namespace dalgo-dev
```

**values-dev.yaml:**
```yaml
environment: development
debug: "True"
djangoSecret: "dev-secret"

postgres:
  enabled: true
  user: postgres
  password: "dev-password"
  db: dalgo_dev
  persistence:
    size: 2Gi

redis:
  enabled: true

webapp:
  enabled: true
  replicaCount: 1
  nextAuthSecret: "dev-nextauth-secret"

airbyte:
  deployInCluster: false
  host: "localhost"
  port: 8000
```

**Deploy to Dev:**
```bash
helm install dalgo-dev ./dalgo \
  -f values-dev.yaml \
  -n dalgo-dev \
  --create-namespace
```

---

### Staging Namespace
```bash
kubectl create namespace dalgo-staging
```

**values-staging.yaml:**
```yaml
environment: staging
debug: "False"
djangoSecret: "$(openssl rand -base64 32)"

postgres:
  enabled: true
  user: postgres
  password: "$(openssl rand -base64 24)"
  db: dalgo_staging
  persistence:
    size: 5Gi

redis:
  enabled: true

webapp:
  enabled: true
  replicaCount: 2
  nextAuthSecret: "$(openssl rand -base64 32)"
  backendUrl: "https://staging-api.example.com"
  nextAuthUrl: "https://staging.example.com"

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: staging.example.com
      paths:
        - path: /
```

**Deploy to Staging:**
```bash
helm install dalgo-staging ./dalgo \
  -f values-staging.yaml \
  -n dalgo-staging \
  --create-namespace
```

---

### Production Namespace
```bash
kubectl create namespace dalgo-prod
```

**values-prod.yaml:**
```yaml
environment: production
debug: "False"
djangoSecret: "$(openssl rand -base64 32)"

postgres:
  enabled: true
  user: postgres
  password: "$(openssl rand -base64 24)"
  db: dalgo_prod
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "prod-ssd"

redis:
  enabled: true
  port: 6379

webapp:
  enabled: true
  replicaCount: 5
  nextAuthSecret: "$(openssl rand -base64 32)"
  backendUrl: "https://api.example.com"
  websocketUrl: "wss://api.example.com"
  nextAuthUrl: "https://example.com"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
  hosts:
    - host: example.com
      paths:
        - path: /
    - host: api.example.com
      paths:
        - path: /

airbyte:
  deployInCluster: true
  persistence:
    size: 100Gi
    storageClass: "prod-ssd"
```

**Deploy to Production:**
```bash
helm install dalgo-prod ./dalgo \
  -f values-prod.yaml \
  -n dalgo-prod \
  --create-namespace
```

---

## Example 3: Multi-Tenant with Airbyte In-Cluster

**values-multitenant-airbyte.yaml:**
```yaml
environment: production
debug: "False"
djangoSecret: "$(openssl rand -base64 32)"

postgres:
  enabled: true
  user: postgres
  password: "$(openssl rand -base64 24)"
  db: dalgo_org
  persistence:
    enabled: true
    size: 20Gi
    storageClass: "fast-ssd"

redis:
  enabled: true

webapp:
  enabled: true
  replicaCount: 2
  nextAuthSecret: "$(openssl rand -base64 32)"

# Deploy Airbyte in the same cluster
airbyte:
  deployInCluster: true
  username: "airbyte"
  password: "$(openssl rand -base64 16)"
  persistence:
    enabled: true
    size: 50Gi
    storageClass: "fast-ssd"

ingress:
  enabled: true
  className: "nginx"
  hosts:
    - host: org-name.example.com
      paths:
        - path: /
```

**Deploy multiple organizations with in-cluster Airbyte:**
```bash
kubectl create namespace dalgo-orgs

# Org 1
helm install dalgo-org1 ./dalgo \
  -f values-multitenant-airbyte.yaml \
  -n dalgo-orgs \
  --set-string postgres.db=dalgo_org1 \
  --set-string postgres.password="org1-pwd-$(openssl rand -hex 8)"

# Org 2
helm install dalgo-org2 ./dalgo \
  -f values-multitenant-airbyte.yaml \
  -n dalgo-orgs \
  --set-string postgres.db=dalgo_org2 \
  --set-string postgres.password="org2-pwd-$(openssl rand -hex 8)"
```

---

## Example 4: Using Helm Values Overrides for Quick Deployment

```bash
# Deploy with inline value overrides (useful for scripting)
helm install dalgo-custom ./dalgo \
  -n dalgo-prod \
  --create-namespace \
  --set environment=production \
  --set debug=false \
  --set postgres.db=dalgo_custom \
  --set postgres.persistence.size=25Gi \
  --set webapp.replicaCount=3 \
  --set ingress.enabled=true \
  --set 'ingress.hosts[0].host=custom.example.com' \
  --set 'ingress.hosts[0].paths[0].path=/' \
  --set 'ingress.hosts[0].paths[0].pathType=Prefix'
```

---

## Example 5: Automated Organization Deployment Script

**deploy-org.sh:**
```bash
#!/bin/bash

# Usage: ./deploy-org.sh acme 10.0.0.1 100Gi

ORG_NAME=$1
API_HOST=$2
STORAGE_SIZE=${3:-10Gi}

if [ -z "$ORG_NAME" ]; then
    echo "Usage: $0 <org-name> <api-host> [storage-size]"
    exit 1
fi

NAMESPACE="dalgo-orgs"
RELEASE_NAME="dalgo-$ORG_NAME"

# Create namespace if it doesn't exist
kubectl create namespace $NAMESPACE 2>/dev/null || true

# Generate random secrets
DJANGO_SECRET=$(openssl rand -base64 32)
DB_PASSWORD=$(openssl rand -base64 24)
NEXTAUTH_SECRET=$(openssl rand -base64 32)

# Deploy
helm install $RELEASE_NAME ./dalgo \
  --namespace $NAMESPACE \
  --set environment=production \
  --set debug=false \
  --set djangoSecret="$DJANGO_SECRET" \
  --set postgres.password="$DB_PASSWORD" \
  --set postgres.db="dalgo_$ORG_NAME" \
  --set postgres.persistence.size="$STORAGE_SIZE" \
  --set webapp.nextAuthSecret="$NEXTAUTH_SECRET" \
  --set webapp.backendUrl="https://$API_HOST" \
  --set ingress.enabled=true \
  --set "ingress.hosts[0].host=$ORG_NAME.example.com"

echo "Deployment complete for organization: $ORG_NAME"
echo "Release name: $RELEASE_NAME"
echo "Namespace: $NAMESPACE"
```

**Usage:**
```bash
chmod +x deploy-org.sh
./deploy-org.sh acme api-acme.example.com 20Gi
./deploy-org.sh globex api-globex.example.com 50Gi
```

---

## Example 6: Backup and Restore for Multi-Tenant Setup

### Backup Single Organization
```bash
#!/bin/bash
ORG=$1
NAMESPACE=${2:-dalgo-orgs}
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="./backups/$ORG"

mkdir -p $BACKUP_DIR

# Backup database
kubectl exec -n $NAMESPACE deployment/dalgo-$ORG-postgres -- \
  pg_dump -U postgres dalgo_$ORG | \
  gzip > $BACKUP_DIR/db_$TIMESTAMP.sql.gz

# Backup PVC data
kubectl exec -n $NAMESPACE deployment/dalgo-$ORG-postgres -- \
  tar -czf - -C /var/lib/postgresql/data . | \
  cat > $BACKUP_DIR/pvc_$TIMESTAMP.tar.gz

echo "Backup completed for $ORG at $BACKUP_DIR"
```

### Restore Single Organization
```bash
#!/bin/bash
ORG=$1
BACKUP_FILE=$2
NAMESPACE=${3:-dalgo-orgs}

# Restore database
gzip -dc $BACKUP_FILE | \
kubectl exec -i -n $NAMESPACE deployment/dalgo-$ORG-postgres -- \
  psql -U postgres dalgo_$ORG

echo "Restore completed for $ORG"
```

---

## Monitoring Multi-Tenant Deployments

```bash
# Watch all instances in a namespace
kubectl get pods -n dalgo-orgs -w

# Check resource usage per organization
kubectl top pods -n dalgo-orgs -l app.kubernetes.io/instance=dalgo-org1
kubectl top pods -n dalgo-orgs -l app.kubernetes.io/instance=dalgo-org2

# Check PVC status
kubectl get pvc -n dalgo-orgs

# View logs from specific organization
kubectl logs -f -n dalgo-orgs deployment/dalgo-org1-backend
kubectl logs -f -n dalgo-orgs deployment/dalgo-org2-backend

# List all resources per organization
kubectl get all -n dalgo-orgs -l app.kubernetes.io/instance=dalgo-org1
```

---

## Upgrade Multi-Tenant Instances

```bash
# Upgrade all instances
for release in $(helm list -n dalgo-orgs -q); do
    helm upgrade $release ./dalgo -n dalgo-orgs -f values-$release.yaml
done

# Upgrade specific instance
helm upgrade dalgo-org1 ./dalgo -n dalgo-orgs -f values-org1.yaml
```

---

## Delete Organization Instance

```bash
# Delete release (keeps PVCs for data safety)
helm uninstall dalgo-org1 -n dalgo-orgs

# Delete with PVCs (caution: data loss!)
kubectl delete pvc -n dalgo-orgs -l app.kubernetes.io/instance=dalgo-org1
```
