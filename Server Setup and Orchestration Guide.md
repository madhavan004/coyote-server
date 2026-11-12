# Server Setup and Orchestration Guide

This document outlines the remote data engineering and self-hosted personal cloud environment running on the `coyote` Ubuntu server, including the Apache Airflow stack, Nextcloud file storage, Immich photo backup, local Python/Pandas development workflow, and remote accessibility via Tailscale.

## 1. System Overview

### Host Details

* **Operating System:** Ubuntu Server (Linux 6.8+ x86_64)
* **Host Name:** `coyote`
* **Local LAN IP:** `192.168.1.19`
* **Tailscale IP:** `100.96.65.65`
* **Tailscale Hostname:** `coyote.tail09a9ed.ts.net`

### Software Stack

| Component | Tool / Service | Version / Notes | Purpose | 
| :--- | :--- | :--- | :--- | 
| Container Runtime | Docker Engine + Compose | Latest stable | Runs Airflow, Code-Server, Nextcloud, and Immich containers | 
| Workflow Orchestrator | Apache Airflow | 3.0+ via Astro CLI | DAG scheduling and workflow execution | 
| Local Developer Framework | Astro CLI | Latest stable | Airflow project lifecycle and local dev workflow | 
| Metadata & App Databases | PostgreSQL / MariaDB | Containerized (Postgres v16+, MariaDB 10.6) | Airflow/Immich metadata and Nextcloud storage backend | 
| Web IDE | Code-Server (VS Code) | `codercom/code-server:latest` | Browser-based development environment | 
| File Sync & Cloud Storage | Nextcloud | `nextcloud:latest` (Docker) | Private file sync and cross-device storage | 
| Photo & Media Backup | Immich | Self-hosted Docker stack | Open-source Google Photos alternative for media backup | 
| Python Dev Environment | Python 3 + `venv` | Ubuntu native `python3` + Pandas | Data science and Pandas practice workflows | 
| Remote Networking | Tailscale | Mesh VPN | Secure access without public exposure | 

## 2. Project & Workspace Structure

```text
/home/coyote/
├── python/                         # Local Python virtual environment & practice scripts
│   └── venv/                       # Isolated Python 3 virtual environment (Pandas, etc.)
└── workspace/
    ├── my-airflow-project/
    │   ├── .astro/
    │   │   └── config.yaml         # Astro configuration and webserver settings
    │   ├── dags/
    │   │   ├── example_test_dag.py # Basic BashOperator validation pipeline
    │   │   └── etl_pipeline.py     # Open-Meteo ETL workflow using TaskFlow API
    │   ├── plugins/                # Custom Airflow plugins
    │   ├── include/                # SQL scripts, helpers, or reference data
    │   ├── tests/                  # DAG/unit tests
    │   ├── Dockerfile              # Custom Astro/Airflow container setup
    │   ├── airflow_settings.yaml   # Airflow connections, variables, and pools
    │   ├── packages.txt            # OS package dependencies
    │   ├── requirements.txt        # Python package dependencies
    │   ├── README.md               # Project documentation
    │   └── .env.example            # Example environment variables (if used)
    ├── nextcloud/
    │   └── docker-compose.yml      # Nextcloud + MariaDB stack configuration
    └── immich/
        ├── docker-compose.yml      # Immich services stack configuration
        ├── .env                    # Storage paths (`UPLOAD_LOCATION=./backup`) and DB config
        ├── postgres/               # Immich PostgreSQL database volume
        └── backup/                 # Media storage directory (`library/`, `thumbs/`, `upload/`, etc.)
```

## 3. Setup Process

### Step 1: Free Conflicting Ports

Stop the host-level PostgreSQL service so Docker can bind to port `5432` without conflict.

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Confirm the port is free
sudo lsof -i :5432
```

### Step 2: Configure & Start the Airflow Environment

Create an Astro config to avoid auto-browser launch, set the webserver port, and launch the stack.

```bash
cd ~/workspace/my-airflow-project
mkdir -p .astro

cat << 'EOF' > .astro/config.yaml
project:
  name: my-airflow-project

webserver:
  port: 8080
EOF

astro dev start --no-proxy --no-browser --wait 3m
```

### Step 3: Run Code-Server (VS Code in the Browser)

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

### Step 4: Deploy Personal Cloud Services (Nextcloud & Immich)

#### Nextcloud (File Sync & Storage)

```bash
mkdir -p ~/workspace/nextcloud && cd ~/workspace/nextcloud
# Configure docker-compose.yml mapping port 8081:80 with MariaDB 10.6
docker compose up -d
```

#### Immich (Photo & Media Backup)

```bash
mkdir -p ~/workspace/immich && cd ~/workspace/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
# Configured UPLOAD_LOCATION=./backup and DB_DATA_LOCATION=./postgres in .env
docker compose up -d
```

### Step 5: Set Up Python Virtual Environment & Docker Hub Workflow

```bash
# Create and activate Python 3 virtual environment for Pandas practice
cd ~/python
python3 -m venv venv
source venv/bin/activate
pip install pandas

# Tag and push custom local images to Docker Hub
docker tag docker-practice:latest <dockerhub-username>/docker-practice:latest
docker push <dockerhub-username>/docker-practice:latest
```

## 4. Network Ports, Endpoints & Credentials Reference

All web interfaces below are exposed over **HTTP** (unencrypted) and are intended to be accessed exclusively inside the private Tailnet VPN (`100.96.65.65` / `coyote.tail09a9ed.ts.net`).

### 4.1 Port Allocation Matrix

| Host Port | Container / Internal Port | Protocol | Service / Stack | Exposure Scope | Notes & Conflict Rules |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`22`** | N/A (Host OS) | SSH | OpenSSH Server | LAN / Tailnet | Remote terminal administration (`ssh coyote@coyote.tail09a9ed.ts.net`) |
| **`2283`** | `2283` | HTTP | Immich Server | Tailnet / LAN | Web UI and mobile app sync endpoint |
| **`5432`** | `5432` | TCP | Airflow Metadata DB (Postgres) | Docker / Localhost | **Must remain free on host OS.** Native `postgresql` service is stopped/disabled to prevent port collision |
| **`8080`** | `8080` | HTTP | Apache Airflow Webserver | Tailnet / LAN | Requires Tailscale hostname (`coyote.tail09a9ed.ts.net`) due to Astro reverse proxy Host header checks |
| **`8081`** | `80` | HTTP | Nextcloud Web App | Tailnet / LAN | Mapped from Apache HTTP port `80` inside `nextcloud-app` container |
| **`8443`** | `8080` | HTTP | Code-Server (VS Code) | Tailnet / LAN | Serves browser IDE over plain HTTP despite port number `8443` |
| *Internal* | `3306` | TCP | Nextcloud MariaDB (`nextcloud-db`) | Docker Bridge Only | Not published to host; isolated inside Nextcloud Compose network |
| *Internal* | `5432` / `6379` | TCP | Immich Postgres & Redis | Docker Bridge Only | Isolated inside Immich Compose network; stored in `./postgres` |

### 4.2 Service Endpoints & Authentication Details

#### 1. Apache Airflow (Astro CLI)
* **Primary URL:** `http://coyote.tail09a9ed.ts.net:8080` *(Do not use raw IP; causes 404 Host header error)*
* **Web UI Username:** `admin`
* **Web UI Password:** `admin` *(Default Astro dev credentials)*
* **Internal Metadata DB:**
  * **Host / Port:** `postgres:5432` (internal) / `localhost:5432` (host)
  * **Database / User / Password:** `postgres` / `postgres` / `postgres`

#### 2. Code-Server (Browser VS Code)
* **Primary URL:** `http://coyote.tail09a9ed.ts.net:8443` (or `http://100.96.65.65:8443`)
* **Authentication Method:** Single password (no username required)
* **Password Location:** Defined via `-e PASSWORD="<your-password>"` in the `docker run` command (or viewable inside the container at `~/.config/code-server/config.yaml`)
* **Mounted Workspace:** `/home/coyote/workspace/my-airflow-project` $\rightarrow$ `/home/coder/project`

#### 3. Nextcloud (File Sync & Storage)
* **Primary URL:** `http://coyote.tail09a9ed.ts.net:8081` (or `http://100.96.65.65:8081`)
* **Web / Mobile App Admin Account:** Created during initial first-boot web setup wizard
* **Mobile Client Note:** Explicitly include `http://` when entering the server URL and allow insecure/HTTP connections in the app settings to prevent `SSL initialization failed` errors
* **Internal Database Credentials (`nextcloud-db` / MariaDB 10.6):**
  * **Database Name:** `nextcloud`
  * **Database User:** `nextcloud`
  * **Database Password:** `nextcloud_db_pass`
  * **Root Password:** `nextcloud_root_pass`
  * *(Defined in `~/workspace/nextcloud/docker-compose.yml`)*

#### 4. Immich (Photo & Media Backup)
* **Primary URL:** `http://coyote.tail09a9ed.ts.net:2283` (or `http://100.96.65.65:2283`)
* **Web / Mobile App Admin Account:** Registered on first visit to port `2283` (stored under `~/workspace/immich/backup/library/admin`)
* **Internal Database Credentials (Immich Postgres):**
  * **Database Name:** `immich`
  * **Database User:** `postgres`
  * **Database Password:** `postgres` *(Defined via `DB_PASSWORD` in `~/workspace/immich/.env`)*

#### 5. Host Server & Registry Accounts
* **Ubuntu Host SSH User:** `coyote` (password-authenticated with `sudo` privileges)
* **Docker Hub Registry:** Authenticated locally via `docker login` for pushing custom images (`<dockerhub-username>/docker-practice:latest`)

## 5. Useful Operational Commands

```bash
# Check the status of Airflow services
astro dev ps

# Inspect logs for the API server and scheduler
astro dev logs --api-server
astro dev logs --scheduler

# Stop the local Airflow stack
astro dev stop

# Manage Nextcloud or Immich stacks (run inside respective directories)
docker compose up -d
docker compose pull && docker compose up -d

# Check all active listening ports on the host
sudo ss -tulpn

# Check native PostgreSQL service status
sudo systemctl status postgresql

# Remove unused Docker images and networks
docker system prune -f

# View active network interfaces
ip a
```

## 6. Issues Encountered and Resolutions

| Issue | Root Cause | Fix | 
| :--- | :--- | :--- | 
| Airflow startup failed with port `5432` already in use | Host PostgreSQL was still running | Stopped and disabled the system service with `sudo systemctl stop postgresql` and `sudo systemctl disable postgresql` | 
| `404` page when opening Airflow via raw Tailscale IP | Astro reverse proxy expected a valid Host header and rejected direct IP addressing | Used the Tailscale hostname instead: `http://coyote.tail09a9ed.ts.net:8080` | 
| API-server health check timed out during `astro dev start` | Initial Airflow database migration takes longer than the default timeout | Increased startup wait time using `--wait 3m` | 
| Docker name resolution failure for `postgres` | The Postgres container had crashed during initial setup and left the network in a bad state | Stopped the stack, cleaned Docker state with `docker system prune`, and restarted cleanly | 
| `unknown flag: --clean` from Astro CLI | The flag combination was unsupported in the installed Astro version | Switched to `astro dev stop` followed by `docker system prune -a --volumes -f` | 
| Terminal unexpectedly opened a text browser (Lynx) during startup | Astro attempted to open a local browser in headless mode | Added `--no-browser` and exited the browser safely with `q` and `y` | 
| `Command 'python' not found` when creating virtual environment | Ubuntu uses `python3` by default rather than an unversioned `python` binary | Used `python3 -m venv venv` to create the virtual environment | 
| Nextcloud mobile app threw `SSL initialization failed` | Nextcloud container runs over plain HTTP (`8081`), while mobile clients expect HTTPS by default | Explicitly specified `http://` with the Tailscale IP/port and enabled insecure/HTTP connections in the app (and added Tailscale host to `trusted_domains`) | 

## 7. Summary

This environment combines a containerized data engineering stack (Apache Airflow via Astro CLI and Code-Server) with self-hosted personal cloud infrastructure (Nextcloud for file sync and Immich for automated photo/video backups) alongside an isolated Python 3 virtual environment for data analysis with Pandas. All services run cleanly on `coyote` via Docker and remain securely accessible over Tailscale.