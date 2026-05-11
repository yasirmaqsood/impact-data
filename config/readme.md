# Galaxy Server Deployment Guide

A step-by-step guide for deploying a production-ready [Galaxy](https://galaxyproject.org/) server (version 25.1) on Ubuntu 24.04 LTS. This document covers cloning the source, setting up the database, configuring Galaxy, enabling file uploads via TUS, managing the process with systemd, and optional extras like OIDC authentication and Pulsar remote execution.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [System Packages](#2-system-packages)
3. [PostgreSQL Database](#3-postgresql-database)
4. [Clone Galaxy Source](#4-clone-galaxy-source)
5. [Python Virtual Environment](#5-python-virtual-environment)
6. [Core Configuration (`galaxy.yml`)](#6-core-configuration-galaxyyml)
7. [Data Directories](#7-data-directories)
8. [Tool Configuration (`tool_conf.xml`)](#8-tool-configuration-tool_confxml)
9. [Job Configuration (`job_conf.yml`)](#9-job-configuration-job_confyml)
10. [First Start (Manual)](#10-first-start-manual)
11. [systemd Service](#11-systemd-service)
12. [Reverse Proxy (Nginx + TLS)](#12-reverse-proxy-nginx--tls)
13. [CVMFS for Reference Data (Optional)](#13-cvmfs-for-reference-data-optional)
14. [OIDC / SSO Authentication (Optional)](#14-oidc--sso-authentication-optional)
15. [Installing Tools from ToolShed](#15-installing-tools-from-toolshed)
16. [Remote Job Execution with Pulsar (Optional)](#16-remote-job-execution-with-pulsar-optional)
17. [Maintenance & Troubleshooting](#17-maintenance--troubleshooting)

---

## 1. Prerequisites

| Requirement | Minimum |
|---|---|
| OS | Ubuntu 22.04 or 24.04 LTS (x86_64) |
| RAM | 4 GB (8 GB+ recommended) |
| Disk | 50 GB free (more for datasets) |
| Python | 3.10 – 3.12 |
| PostgreSQL | 14+ (16 recommended) |
| Network | Outbound HTTPS for ToolShed, Conda, PyPI |

You need `sudo` access for installing system packages and setting up services.

---

## 2. System Packages

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  build-essential \
  python3 python3-dev python3-venv python3-pip \
  git \
  postgresql postgresql-client libpq-dev \
  nginx certbot python3-certbot-nginx \
  curl wget \
  libcurl4-openssl-dev libssl-dev \
  zlib1g-dev libbz2-dev liblzma-dev \
  libffi-dev libsqlite3-dev \
  node-gyp nodejs npm
```

These cover compilation of C extensions, database drivers, and web proxy components.

---

## 3. PostgreSQL Database

Galaxy uses PostgreSQL for all persistent metadata (users, histories, jobs, datasets).

```bash
# Switch to the postgres system user
sudo -u postgres psql
```

Inside the `psql` shell:

```sql
-- Create a dedicated Galaxy role with a strong password
CREATE ROLE galaxy LOGIN PASSWORD 'CHANGE_ME_STRONG_PASSWORD';

-- Create the Galaxy database owned by that role
CREATE DATABASE galaxy OWNER galaxy;

-- Exit
\q
```

Verify connectivity:

```bash
psql -h localhost -U galaxy -d galaxy -c "SELECT 1;"
```

> **Tip**: For password-free connections from your Galaxy process, create a `~/.pgpass` file:
>
> ```
> localhost:5432:galaxy:galaxy:CHANGE_ME_STRONG_PASSWORD
> ```
>
> Then `chmod 600 ~/.pgpass`.

---

## 4. Clone Galaxy Source

```bash
cd /home/ubuntu
git clone -b release_25.1 https://github.com/galaxyproject/galaxy.git galaxy_25.1/galaxy
cd galaxy_25.1/galaxy
```

The repository includes all built-in tools, the client build system, and sample configurations.

---

## 5. Python Virtual Environment

Galaxy ships a helper script (`scripts/common_startup.sh`) that bootstraps a virtual environment, but you can also create one explicitly:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

The first run of `run.sh` will automatically install Galaxy's Python dependencies into `.venv/`.

---

## 6. Core Configuration (`galaxy.yml`)

Copy the sample and edit:

```bash
cp config/galaxy.yml.sample config/galaxy.yml
```

Minimal production settings:

```yaml
galaxy:
  # Database
  database_connection: postgresql://galaxy:CHANGE_ME_STRONG_PASSWORD@localhost:5432/galaxy
  install_database_connection: postgresql://galaxy:CHANGE_ME_STRONG_PASSWORD@localhost:5432/galaxy

  # Data paths
  file_path: /data/galaxy/database/files
  new_file_path: /data/galaxy/database/tmp
  job_working_directory: /data/galaxy/database/jobs_directory

  # TUS resumable uploads
  tus_upload_store: /data/galaxy/database/tmp/tus

  # Job configuration
  job_config_file: config/job_conf.yml

  # Tool configuration
  tool_config_file: config/tool_conf.xml

  # Conda (auto-resolve tool dependencies)
  conda_auto_init: true
  conda_auto_install: true

  # Admin users (comma-separated emails)
  admin_users: your-email@example.com

  # Public URL (used for callbacks, TUS hooks, etc.)
  galaxy_infrastructure_url: https://galaxy.yourdomain.org

  # Keep job dirs for debugging (set to "always" in production to clean up)
  cleanup_job: never

gravity:
  gunicorn:
    bind: 0.0.0.0:8080
  tusd:
    enable: true
```

### Key settings explained

| Setting | Purpose |
|---|---|
| `database_connection` | SQLAlchemy URI pointing to your PostgreSQL database |
| `file_path` | Where Galaxy stores uploaded/generated dataset files |
| `job_working_directory` | Temporary workspace for running jobs |
| `conda_auto_init` | Galaxy will install Miniconda on first start to resolve tool deps |
| `admin_users` | Email(s) that gain admin panel access |
| `gravity` | Process manager config — Gunicorn serves the web app |
| `tusd` | Enables resumable, large file uploads via the TUS protocol |

---

## 7. Data Directories

Create the directories referenced in `galaxy.yml`:

```bash
sudo mkdir -p /data/galaxy/database/{files,tmp,tmp/tus,jobs_directory}
sudo chown -R $(whoami):$(whoami) /data/galaxy
```

Ensure the disk mounted at `/data` has sufficient space for datasets.

---

## 8. Tool Configuration (`tool_conf.xml`)

The default `config/tool_conf.xml.sample` includes Galaxy's built-in tools (Get Data, Text Manipulation, etc.). To use it:

```bash
cp config/tool_conf.xml.sample config/tool_conf.xml
```

You can add custom tool sections by editing this file:

```xml
<toolbox monitor="true">
  <!-- Built-in sections ... -->
  
  <section id="custom_tools" name="My Custom Tools">
    <tool file="tools/my_tool/my_tool.xml" />
  </section>
</toolbox>
```

Tools installed from the ToolShed (via the admin UI) are tracked separately in `config/shed_tool_conf.xml`, which Galaxy manages automatically.

---

## 9. Job Configuration (`job_conf.yml`)

For a basic single-server setup with local execution:

```bash
cp config/job_conf.sample.yml config/job_conf.yml
```

Minimal working configuration:

```yaml
runners:
  local:
    load: galaxy.jobs.runners.local:LocalJobRunner
    workers: 4

execution:
  default: local_dest
  environments:
    local_dest:
      runner: local
```

This tells Galaxy to run all jobs locally with 4 worker threads. See [Section 16](#16-remote-job-execution-with-pulsar-optional) for distributed execution.

---

## 10. First Start (Manual)

```bash
cd /home/ubuntu/galaxy_25.1/galaxy
./run.sh
```

On the first launch Galaxy will:

1. Install Python dependencies into `.venv/`
2. Run database migrations (creating all tables)
3. Initialize Conda (if `conda_auto_init: true`)
4. Start the Gunicorn WSGI server on port 8080

Watch the logs (`galaxy.log`) for:

```
serving on http://0.0.0.0:8080
```

Test locally:

```bash
curl -s http://localhost:8080/api/version
```

You should see: `{"version_major":"25.1", ...}`

Stop with `Ctrl+C` or `./run.sh --stop-daemon` if backgrounded.

---

## 11. systemd Service

Create a unit file so Galaxy starts on boot and can be managed with `systemctl`:

```bash
sudo tee /etc/systemd/system/galaxy.service > /dev/null <<'EOF'
[Unit]
Description=Galaxy Server
After=network.target postgresql.service

[Service]
Type=oneshot
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/galaxy_25.1/galaxy
ExecStart=/home/ubuntu/galaxy_25.1/galaxy/run.sh --daemon
ExecStop=/home/ubuntu/galaxy_25.1/galaxy/run.sh --stop-daemon
RemainAfterExit=yes
TimeoutStartSec=300
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable galaxy
sudo systemctl start galaxy
```

Check status:

```bash
sudo systemctl status galaxy
journalctl -u galaxy -f
```

---

## 12. Reverse Proxy (Nginx + TLS)

Galaxy's Gunicorn should not face the internet directly. Use Nginx as a reverse proxy with TLS termination.

### Nginx site configuration

```bash
sudo tee /etc/nginx/sites-available/galaxy > /dev/null <<'EOF'
upstream galaxy_app {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name galaxy.yourdomain.org;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name galaxy.yourdomain.org;

    ssl_certificate     /etc/letsencrypt/live/galaxy.yourdomain.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/galaxy.yourdomain.org/privkey.pem;

    client_max_body_size 50G;

    # TUS upload endpoint
    location /api/upload/resumable_upload {
        proxy_pass http://127.0.0.1:1080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Galaxy application
    location / {
        proxy_pass http://galaxy_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/galaxy /etc/nginx/sites-enabled/galaxy
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

### TLS certificate (Let's Encrypt)

```bash
sudo certbot --nginx -d galaxy.yourdomain.org
```

Certbot will automatically configure auto-renewal.

---

## 13. CVMFS for Reference Data (Optional)

[CernVM-FS](https://cvmfs.readthedocs.io/) provides read-only access to Galaxy reference genomes, tool-data tables, and pre-built Singularity containers without local disk storage.

### Install CVMFS client

```bash
wget https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest_all.deb
sudo dpkg -i cvmfs-release-latest_all.deb
sudo apt update && sudo apt install -y cvmfs

sudo cvmfs_config setup
```

### Configure Galaxy repositories

```bash
sudo tee /etc/cvmfs/default.local > /dev/null <<'EOF'
CVMFS_REPOSITORIES="data.galaxyproject.org,singularity.galaxyproject.org"
CVMFS_HTTP_PROXY="DIRECT"
CVMFS_QUOTA_LIMIT=10000
EOF

sudo cvmfs_config probe
```

### Point Galaxy to CVMFS tool-data

In `galaxy.yml`, add:

```yaml
tool_data_table_config_path: /cvmfs/data.galaxyproject.org/byhand/location/tool_data_table_conf.xml,/cvmfs/data.galaxyproject.org/managed/location/tool_data_table_conf.xml
```

This gives Galaxy access to pre-indexed reference genomes (hg38, mm10, etc.) without downloading them locally.

---

## 14. OIDC / SSO Authentication (Optional)

Galaxy supports OpenID Connect for institutional single sign-on.

### Enable in `galaxy.yml`

```yaml
galaxy:
  enable_oidc: true
  oidc_config_file: config/oidc_config.xml
  oidc_backends_config_file: config/oidc_backends_config.xml
```

### Configure an OIDC provider

Create `config/oidc_backends_config.xml`:

```xml
<?xml version="1.0"?>
<OIDC>
  <provider name="keycloak">
    <url>https://keycloak.yourdomain.org/realms/your-realm</url>
    <client_id>galaxy-client</client_id>
    <client_secret>YOUR_CLIENT_SECRET</client_secret>
    <redirect_uri>https://galaxy.yourdomain.org/authnz/keycloak/callback</redirect_uri>
  </provider>
</OIDC>
```

Refer to the [Galaxy OIDC docs](https://docs.galaxyproject.org/en/latest/admin/authentication.html) for full details on supported providers (Keycloak, Google, Elixir AAI, etc.).

---

## 15. Installing Tools from ToolShed

Once Galaxy is running and you have admin access:

1. Navigate to **Admin → Install and Uninstall** in the Galaxy UI.
2. Search the [Galaxy ToolShed](https://toolshed.g2.bx.psu.edu/) for the tool you need (e.g., `clustalo`, `bwa`, `samtools`).
3. Click **Install** — Galaxy downloads the tool XML, test data, and resolves dependencies via Conda.

Installed tools appear in `config/shed_tool_conf.xml` automatically and become available to all users.

---

## 16. Remote Job Execution with Pulsar (Optional)

[Pulsar](https://pulsar.readthedocs.io/) allows Galaxy to offload jobs to remote compute nodes communicating via a message queue (e.g., RabbitMQ).

### Architecture

```
Galaxy Server  ──(AMQP/TLS)──►  RabbitMQ Broker  ──(AMQP/TLS)──►  Pulsar Node
     │                                                                    │
     └── submits job ─────────────────────────────────────────────────────┘
                              (files transferred via HTTP)
```

### Galaxy-side configuration (`job_conf.yml`)

```yaml
runners:
  local:
    load: galaxy.jobs.runners.local:LocalJobRunner
    workers: 4

  pulsar_mq_remote:
    load: galaxy.jobs.runners.pulsar:PulsarMQJobRunner
    amqp_url: amqps://user:pass@broker-host:5671/vhost
    galaxy_url: https://galaxy.yourdomain.org
    amqp_acknowledge: true
    manager: _default_
    # TLS certs (if using mutual TLS)
    amqp_connect_ssl_ca_certs: /path/to/ca-cert.pem
    amqp_connect_ssl_keyfile: /path/to/client-key.pem
    amqp_connect_ssl_certfile: /path/to/client-cert.pem
    amqp_connect_ssl_cert_reqs: cert_required
    amqp_consumer_timeout: 2

execution:
  default: local_dest
  environments:
    local_dest:
      runner: local

    pulsar_remote_dest:
      runner: pulsar_mq_remote
      jobs_directory: /var/opt/pulsar/staging
      dependency_resolution: remote
      rewrite_parameters: true
      # For Singularity/Apptainer execution:
      singularity_enabled: true
      singularity_cmd: /usr/bin/apptainer
      singularity_default_container_id: docker://quay.io/biocontainers/your-tool:tag
      require_container: true

  tools:
    - id: your_tool_id
      environment: pulsar_remote_dest
```

### Pulsar-side setup (remote node)

```bash
# Install Pulsar
pip install pulsar-app

# Create config directory
mkdir -p /srv/pulsar/config

# app.yml
cat > /srv/pulsar/config/app.yml <<'EOF'
message_queue_url: amqps://user:pass@broker-host:5671/vhost
staging_directory: /var/opt/pulsar/staging
persistence_directory: /var/opt/pulsar/persisted_data
EOF

# Start Pulsar
pulsar --mode webless --config /srv/pulsar/config/app.yml
```

This is a simplified overview. Refer to the [Pulsar documentation](https://pulsar.readthedocs.io/) for advanced container resolution, dependency management, and multi-queue setups.

---

## 17. Maintenance & Troubleshooting

### Checking logs

```bash
# Galaxy application log
tail -f /home/ubuntu/galaxy_25.1/galaxy/galaxy.log

# systemd journal
journalctl -u galaxy -f

# PostgreSQL logs
sudo journalctl -u postgresql -f
```

### Common issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| `FATAL: password authentication failed` on startup | `.pgpass` missing or wrong permissions | Create `~/.pgpass` with `chmod 600` |
| Port 8080 already in use | Previous Galaxy instance still running | `./run.sh --stop-daemon` then restart |
| Tools greyed out / not visible | `tool_conf.xml` path wrong in `galaxy.yml` | Verify the path and restart |
| Uploads fail silently | `tus_upload_store` dir missing or wrong owner | `mkdir -p` and `chown` the TUS directory |
| Database migration errors | Schema out of date after upgrade | `./manage_db.sh upgrade` |

### Upgrading Galaxy

```bash
cd /home/ubuntu/galaxy_25.1/galaxy
git fetch origin
git checkout release_25.2  # or the next release branch

# Run database migrations
./manage_db.sh upgrade

# Restart
sudo systemctl restart galaxy
```

### Backup checklist

- **PostgreSQL**: `pg_dump -U galaxy galaxy > galaxy_backup.sql`
- **Dataset files**: `/data/galaxy/database/files/`
- **Configuration**: `config/galaxy.yml`, `config/job_conf.yml`, `config/tool_conf.xml`
- **Shed tools metadata**: `config/shed_tool_conf.xml`

---

## Quick Reference — File Layout

```
/home/ubuntu/galaxy_25.1/galaxy/
├── config/
│   ├── galaxy.yml              # Main Galaxy configuration
│   ├── job_conf.yml            # Job routing and runners
│   ├── tool_conf.xml           # Built-in tool registry
│   └── shed_tool_conf.xml      # ToolShed-installed tools (auto-managed)
├── .venv/                      # Python virtual environment
├── run.sh                      # Start/stop Galaxy
├── manage_db.sh                # Database migration utility
├── tools/                      # Built-in tool XML definitions
└── galaxy.log                  # Application log

/data/galaxy/database/
├── files/                      # Stored datasets
├── tmp/                        # Temporary and TUS upload staging
└── jobs_directory/             # Per-job working directories

/etc/systemd/system/
└── galaxy.service              # systemd unit

/etc/nginx/sites-available/
└── galaxy                      # Reverse proxy config
```

---

## Summary

With these steps you have a fully functional Galaxy 25.1 instance running on Ubuntu 24.04 with:

- PostgreSQL-backed metadata storage
- Gunicorn serving the web application
- Resumable file uploads via TUS
- Conda-based automatic tool dependency resolution
- systemd process management
- Nginx reverse proxy with TLS

From here you can install bioinformatics tools from the ToolShed, configure OIDC for institutional authentication, mount CVMFS for shared reference data, and add Pulsar nodes for distributed job execution.

---

*Last updated: 2026-05-11*
