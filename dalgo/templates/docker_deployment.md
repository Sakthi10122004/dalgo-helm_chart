# Dalgo Local & Docker Deployment Guide

This document details the steps to set up, deploy, and troubleshoot Dalgo's core services using Docker and local development tools.

---

## Architecture Overview

A complete local deployment of Dalgo comprises:
1. **Core Databases**: Postgres (Dalgo metadata) and Redis (caching and task queues).
2. **Airbyte**: Data integration engine (running in Docker Compose).
3. **Prefect Proxy**: Orchestration proxy for Prefect workflows.
4. **Dalgo Backend**: Django REST API ASGI application.
5. **Dalgo Webapp**: Next.js user interface.

---

## Step 1: Run Core Databases (Postgres & Redis)

Start PostgreSQL and Redis containers for local data persistence:

```bash
# Start Postgres
docker run --name postgres-db -e POSTGRES_PASSWORD=docker -e POSTGRES_DB=dalgo -p 5432:5432 -d postgres:latest

# Start Redis
docker run --name redis-server -p 6379:6379 -d redis:latest
```

---

## Step 2: Install and Start Airbyte (v0.58.0)

Airbyte runs inside its own Docker Compose network.

1. **Create an isolated directory** for Airbyte assets:
   ```bash
   mkdir -p airbyte
   cd airbyte
   ```

2. **Download the Airbyte startup script** (v0.58.0 is verified compatible):
   ```bash
   curl -LSO https://raw.githubusercontent.com/airbytehq/airbyte/v0.58.0/run-ab-platform.sh
   ```

3. **Start Airbyte in background/detached mode**:
   * *On Windows:* Run this command via Git Bash or WSL (Windows Subsystem for Linux):
     ```bash
     bash ./run-ab-platform.sh -b --dnt
     ```
   * *On macOS/Linux:* 
     ```bash
     chmod +x run-ab-platform.sh
     ./run-ab-platform.sh -b --dnt
     ```

4. **Verify Airbyte is running**:
   Access the dashboard at [http://localhost:8000](http://localhost:8000) (default credentials: `airbyte` / `password`).

---

## Step 3: Set up Prefect Proxy Mock (Development Mode)

During client onboarding, the Django backend makes API requests to the Prefect Proxy to register orchestration blocks. If you do not have a fully configured Prefect Server running, you can run a lightweight mock proxy:

1. **Create `mock_prefect_proxy.py`** in the application directory:
   ```python
   import http.server
   import json
   import socketserver

   PORT = 8085

   class MockPrefectProxyHandler(http.server.BaseHTTPRequestHandler):
       def log_message(self, format, *args):
           print(f"[MOCK PROXY] {format % args}")

       def do_GET(self):
           self.send_response(200)
           self.send_header("Content-Type", "application/json")
           self.end_headers()
           if "/proxy/blocks/airbyte/server/" in self.path:
               self.wfile.write(json.dumps({"block_id": None}).encode("utf-8"))
               return
           self.wfile.write(json.dumps({"status": "ok"}).encode("utf-8"))

       def do_POST(self):
           content_length = int(self.headers.get('Content-Length', 0))
           body = self.rfile.read(content_length).decode('utf-8')
           self.send_response(200)
           self.send_header("Content-Type", "application/json")
           self.end_headers()
           if "/proxy/blocks/airbyte/server/" in self.path:
               try:
                   data = json.loads(body)
                   block_name = data.get("blockName", "mock-server")
               except Exception:
                   block_name = "mock-server"
               response = {
                   "block_id": "mock-prefect-block-id-12345",
                   "cleaned_block_name": block_name
               }
               self.wfile.write(json.dumps(response).encode("utf-8"))
               return
           self.wfile.write(json.dumps({"status": "created"}).encode("utf-8"))

       def do_PUT(self):
           self.send_response(200)
           self.send_header("Content-Type", "application/json")
           self.end_headers()
           self.wfile.write(json.dumps({"status": "updated"}).encode("utf-8"))

   if __name__ == "__main__":
       socketserver.TCPServer.allow_reuse_address = True
       with socketserver.TCPServer(("", PORT), MockPrefectProxyHandler) as httpd:
           print(f"Starting Mock Prefect Proxy on port {PORT}")
           httpd.serve_forever()
   ```

2. **Run the mock proxy in the background**:
   ```bash
   python mock_prefect_proxy.py
   ```

---

## Step 4: Configure Backend Environment (`.env`)

In `DDP_backend/.env`, update the Airbyte and Prefect connections to match the local setup:

```env
# === AIRBYTE CONFIGURATION ===
AIRBYTE_SERVER_HOST=localhost
AIRBYTE_SERVER_PORT=8000
AIRBYTE_SERVER_APIVER=v1
AIRBYTE_API_TOKEN=YWlyYnl0ZTpwYXNzd29yZA== # base64 encoding of airbyte:password

# === PREFECT CONFIGURATION ===
PREFECT_PROXY_API_URL=http://localhost:8085
```

---

## Step 5: Start the Dalgo Backend

1. **Activate Virtual Environment & Sync DB**:
   ```bash
   .\.venv\Scripts\python.exe manage.py migrate
   .\.venv\Scripts\python.exe manage.py loaddata seed/*.json
   ```

2. **Create your first Org and Super-Admin**:
   Provide the password as the third argument to run non-interactively:
   ```bash
   .\.venv\Scripts\python.exe manage.py createorganduser "YourOrgName" "your-email@example.com" "yourpassword" --role super-admin
   ```

3. **Start the ASGI Backend Server**:
   ```bash
   .\.venv\Scripts\python.exe -m uvicorn ddpui.asgi:application --host 0.0.0.0 --port 8002
   ```

---

## Step 6: Start the Dalgo Frontend (Webapp)

1. **Configure Frontend Environment (`.env.local`)**:
   In `webapp/.env.local`, configure the variables pointing to your local backend and NextAuth secrets (see `webapp/.env.example` for reference):
   ```env
   NEXT_PUBLIC_BACKEND_URL="http://localhost:8002"
   NEXT_PUBLIC_WEBSOCKET_URL="ws://localhost:8002"
   NEXTAUTH_SECRET="local-nextauth-secret-change-me"
   NEXTAUTH_URL="http://localhost:3000"
   NEXT_PUBLIC_AIRBYTE_URL="http://localhost:8000"
   ```

2. **Start the Frontend Development Server**:
   Navigate to the `webapp` folder, install dependencies, and run the development server:
   ```bash
   cd webapp
   yarn install
   yarn dev
   ```
   Open your browser and navigate to [http://localhost:3000](http://localhost:3000) to access the platform interface.

---

## Troubleshooting

### Issue: `Invalid URL 'http://:/api//source_definitions/list_for_workspace': No host supplied`
* **Symptom**: The UI/Network Calls return a `500 connection error` or stack trace complaining about an empty hostname / port.
* **Root Cause**: The uvicorn backend process was started before the `.env` configuration file was populated with the Airbyte settings. Because environment variables are read once at process startup, modifying `.env` afterwards will not update the running uvicorn container/server.
* **Resolution**:
  1. Find the PID of the process using port `8002`:
     ```powershell
     # Windows PowerShell
     netstat -ano | findstr :8002
     ```
  2. Kill the stale uvicorn process (replace `<PID>` with the actual process ID):
     ```powershell
     taskkill /F /PID <PID>
     ```
  3. Restart the uvicorn server:
     ```powershell
     .\.venv\Scripts\python.exe -m uvicorn ddpui.asgi:application --host 0.0.0.0 --port 8002
     ```
