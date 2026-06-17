# Dalgo K3s & Helm Deployment Guide

This guide explains how to deploy the Dalgo platform onto a local **K3s** or **k3d** (K3s in Docker) cluster using the custom **Dalgo Helm Chart**.

---

## Prerequisites

1. **Docker Desktop** (running and configured on your host).
2. **Helm v3** installed:
   * *macOS/Linux:* `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
   * *Windows:* `choco install kubernetes-helm` or download the binary.
3. **kubectl** installed.

---

## Step 1: Set up a Local K3s Cluster (using k3d)

`k3d` is a wrapper tool that runs K3s as lightweight containers in Docker. It is the easiest way to run K3s locally on Windows or macOS.

1. **Install k3d**:
   * *Windows (PowerShell):* `choco install k3d` or `winget install k3d`
   * *macOS (Homebrew):* `brew install k3d`
   * *Linux:* `wget -qO- https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | tag=v5.6.0 bash`

2. **Create the K3s cluster**:
   Map ports `3000` (frontend webapp) and `8002` (backend API) from the cluster load balancer to localhost:
   ```bash
   k3d cluster create dalgo-cluster -p "8002:8002@loadbalancer" -p "3000:3000@loadbalancer"
   ```

3. **Verify the cluster status**:
   ```bash
   kubectl cluster-info
   kubectl get nodes
   ```

---

## Step 2: Build and Import Dalgo Backend and Frontend Images

Since the backend image (`dalgo_backend:0.1`) and frontend image (`dalgo_webapp:latest`) are built locally, you must import them directly into the K3s container runtime so Kubernetes can resolve them.

### 1. Build and Import Dalgo Backend
1. Build the Docker image from the root of `DDP_backend`:
   ```bash
   docker build -f Docker/Dockerfile.dev.deploy -t dalgo_backend:0.1 .
   ```
2. Import the image into K3s (k3d):
   ```bash
   k3d image import dalgo_backend:0.1 -c dalgo-cluster
   ```
   *(Or for native K3s/WSL: `docker save dalgo_backend:0.1 | sudo k3s ctr images import -`)*

### 2. Build and Import Dalgo Frontend (Webapp)
1. Set up the environment configuration by copying `.env.local` to `Docker/.env` and build the image from the `webapp` directory:
   ```bash
   cp webapp/.env.local webapp/Docker/.env
   cd webapp
   ./docker-build.sh dalgo_webapp:latest
   ```
2. Import the image into K3s (k3d):
   ```bash
   k3d image import dalgo_webapp:latest -c dalgo-cluster
   ```
   *(Or for native K3s/WSL: `docker save dalgo_webapp:latest | sudo k3s ctr images import -`)*

---

## Step 3: Deploy the Dalgo Helm Chart

The Dalgo Helm chart is located at `charts/dalgo/`.

1. **Review and edit configurations** in the `charts/dalgo/values.yaml` file if necessary. You have two options for **Airbyte**:

   * **Option A: Deploy Airbyte Inside the Cluster (In-Cluster)**:
     Set the values to run the full Airbyte stack inside k3s:
     ```yaml
     airbyte:
       deployInCluster: true
       createBridgeService: false
       host: "airbyte-proxy"
     ```
     *Note: Ensure your k3s nodes have mounted `/var/run/docker.sock` correctly. The chart will spin up a `docker-proxy` to interface with the node's docker daemon for sync jobs.*

   * **Option B: Connect to External Airbyte (Docker Compose)**:
     Use your existing Docker Compose instance running on the host machine by configuring a bridge service:
     ```yaml
     airbyte:
       deployInCluster: false
       createBridgeService: true
       externalIP: "172.28.195.198" # Set to your WSL node/host IP address
       host: "airbyte-proxy"
     ```

2. **Install the chart** (replace `dalgo-1` with your preferred release name):
   ```bash
   helm install dalgo-1 ./charts/dalgo -n dalgo --create-namespace
   ```

3. **Monitor the deployment**:
   ```bash
   kubectl get pods -n dalgo -w
   ```
   *Note: During startup, Helm will run a post-install Job (`dalgo-1-initdb-*`) to execute database migrations and seed data. Once that job succeeds, the backend and celery pods will start.*

---

## Step 4: Access and Configure the Platform

1. **Create an admin user and organization**:
   Execute the Django management command directly inside the running backend container (adjust `dalgo-1-backend` according to your Helm release name):
   ```bash
   kubectl exec -it deployments/dalgo-1-backend -n dalgo -- python manage.py createorganduser "YourOrg" "admin@example.com" "password123" --role super-admin
   ```

2. **Verify API connectivity**:
   Test the Django Swagger/Ninja documentation endpoint on the mapped port:
   ```bash
   curl -I http://localhost:8002/api/docs
   ```

3. **Access the Frontend**:
   Open your browser and navigate to:
   👉 **http://localhost:3000**

---

## Helm Chart Management Commands

### Upgrade Configuration
If you modify `values.yaml` or any templates:
```bash
helm upgrade dalgo-1 ./charts/dalgo -n dalgo
```

### Delete/Teardown Deployment
```bash
helm uninstall dalgo-1 -n dalgo
```

### Check Logs of Backend or Worker Pods
```bash
kubectl logs -l app=dalgo-backend -n dalgo --tail=100 -f
kubectl logs -l app=dalgo-celery-worker-default -n dalgo --tail=100 -f
```
