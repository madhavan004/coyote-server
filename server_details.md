# Server Setup and Orchestration Guide

This document outlines the remote data engineering environment running on the `coyote` Ubuntu server, including the Apache Airflow stack, local development workflow, and remote accessibility via Tailscale.

---

## 1. System Overview

### Host Details

- **Operating System:** Ubuntu Server (Linux 6.8+ x86_64)
- **Host Name:** `coyote`
- **Local LAN IP:** `192.168.1.19`
- **Tailscale IP:** `100.96.65.65`
- **Tailscale Hostname:** `coyote.tail09a9ed.ts.net`

### Software Stack

| Component | Tool / Service | Version / Notes | Purpose |
| :--- | :--- | :--- | :--- |
| Container Runtime | Docker Engine + Compose | Latest stable | Runs Airflow and Code-Server containers |
| Workflow Orchestrator | Apache Airflow | 3.0+ via Astro CLI | DAG scheduling and workflow execution |
| Local Developer Framework | Astro CLI | Latest stable | Airflow project lifecycle and local dev workflow |
| Metadata Database | PostgreSQL | Containerized (v16+) | Airflow metadata and task tracking |
| Web IDE | Code-Server (VS Code) | `codercom/code-server:latest` | Browser-based development environment |
| Remote Networking | Tailscale | Mesh VPN | Secure access without public exposure |

---

## 2. Project Structure

```text
/home/coyote/workspace/my-airflow-project/
├── .astro/
│   └── config.yaml               # Astro configuration and webserver settings
├── dags/
│   ├── example_test_dag.py       # Basic BashOperator validation pipeline
│   └── etl_pipeline.py            # Open-Meteo ETL workflow using TaskFlow API
├── plugins/                       # Custom Airflow plugins
├── include/                       # SQL scripts, helpers, or reference data
├── tests/                         # DAG/unit tests
├── Dockerfile                     # Custom Astro/Airflow container setup
├── airflow_settings.yaml          # Airflow connections, variables, and pools
├── packages.txt                   # OS package dependencies
├── requirements.txt               # Python package dependencies
├── README.md                      # Project documentation
└── .env.example                   # Example environment variables (if used)
```

---

## 3. Setup Process

### Step 1: Free Conflicting Ports

Stop the host-level PostgreSQL service so Docker can bind to port `5432` without conflict.

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Confirm the port is free
sudo lsof -i :5432
```

### Step 2: Configure the Astro Project

Create an Astro config to avoid auto-browser launch and explicitly set the webserver port.

```bash
cd ~/workspace/my-airflow-project
mkdir -p .astro

cat << 'EOF' > .astro/config.yaml
project:
  name: my-airflow-project

webserver:
  port: 8080
EOF
```

### Step 3: Start the Airflow Environment

Launch the environment with Astro CLI, disabling browser startup and extending the health check timeout.

```bash
cd ~/workspace/my-airflow-project
astro dev start --no-proxy --no-browser --wait 3m
```

### Step 4: Run Code-Server (VS Code in the Browser)

Start a persistent VS Code container pointing to the Airflow project directory.

```bash
docker run -d \
  --name code-server \
  --restart unless-stopped \
  -p 8443:8080 \
  -v "$HOME/workspace/my-airflow-project:/home/coder/project" \
  -e PASSWORD="<your-password>" \
  codercom/code-server:latest
```

> Replace the placeholder password with a secure value appropriate for your environment.

---

## 4. Access Endpoints

These services are intended to be accessed through the Tailnet VPN only.

- **Airflow UI:** `http://coyote.tail09a9ed.ts.net:8080`
  - Default credentials: `admin / admin`
- **Code-Server:** `http://coyote.tail09a9ed.ts.net:8443`

---

## 5. Useful Operational Commands

```bash
# Check the status of Airflow services
astro dev ps

# Inspect logs for the API server and scheduler
astro dev logs --api-server
astro dev logs --scheduler

# Stop the local Airflow stack
astro dev stop

# Remove unused Docker images and networks
docker system prune -f

# View active network interfaces
ip a
```

---

## 6. Issues Encountered and Resolutions

| Issue | Root Cause | Fix |
| :--- | :--- | :--- |
| Airflow startup failed with port `5432` already in use | Host PostgreSQL was still running | Stopped and disabled the system service with `sudo systemctl stop postgresql` and `sudo systemctl disable postgresql` |
| `404` page when opening Airflow via raw Tailscale IP | Astro reverse proxy expected a valid Host header and rejected direct IP addressing | Used the Tailscale hostname instead: `http://coyote.tail09a9ed.ts.net:8080` |
| API-server health check timed out during `astro dev start` | Initial Airflow database migration takes longer than the default timeout | Increased startup wait time using `--wait 3m` |
| Docker name resolution failure for `postgres` | The Postgres container had crashed during initial setup and left the network in a bad state | Stopped the stack, cleaned Docker state with `docker system prune`, and restarted cleanly |
| `unknown flag: --clean` from Astro CLI | The flag combination was unsupported in the installed Astro version | Switched to `astro dev stop` followed by `docker system prune -a --volumes -f` |
| Terminal unexpectedly opened a text browser (Lynx) during startup | Astro attempted to open a local browser in headless mode | Added `--no-browser` and exited the browser safely with `q` and `y` |

---

## 7. Summary

This environment provides a reliable way to run Apache Airflow locally in Docker while keeping the development stack accessible through Tailscale. The combination of Astro CLI for orchestration and Code-Server for browser-based editing creates a clean remote engineering setup for DAG development and experimentation.
