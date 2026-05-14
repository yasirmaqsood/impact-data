# Galaxy Server Deployment Guide

A step-by-step guide for deploying a production-ready [Galaxy](https://galaxyproject.org/) server (version 25.1) on Ubuntu 24.04 LTS. This document covers cloning the source, setting up the database, configuring Galaxy, enabling file uploads via TUS, managing the process with systemd, and optional extras like OIDC authentication and Pulsar remote execution.

---

## Table of Contents


- [System Packages](#system-packages)
- [PostgreSQL Database](#postgresql-database)
- [Clone Galaxy Source](#clone-galaxy-source)
- [Python Virtual Environment](#python-virtual-environment)
- [Core Configuration (`galaxy.yml`)](#core-configuration-galaxyyml)
- [Data Directories](#data-directories)
- [Tool Configuration (`tool_conf.xml`)](#tool-configuration-tool_confxml)
- [Job Configuration (`job_conf.yml`)](#job-configuration-job_confyml)
- [First Start (Manual)](#first-start-manual)
- [systemd Service](#systemd-service)
- [CVMFS for Reference Data (Optional)](#cvmfs-for-reference-data-optional)
- [OIDC / SSO Authentication (Optional)](#oidc--sso-authentication-optional)
- [Installing Tools from ToolShed](#installing-tools-from-toolshed)
- [Remote Job Execution with Pulsar (Optional)](#remote-job-execution-with-pulsar-optional)
- [Maintenance & Troubleshooting](#maintenance--troubleshooting)

---


## System Packages

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

## PostgreSQL Database

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

---

## Clone Galaxy Source

```bash
git clone -b release_25.1 https://github.com/galaxyproject/galaxy.git
cd galaxy/
```

The repository includes all built-in tools, the client build system, and sample configurations.

---

## Python Virtual Environment

Galaxy ships a helper script (`scripts/common_startup.sh`) that bootstraps a virtual environment, but you can also create one explicitly:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

The first run of `run.sh` will automatically install Galaxy's Python dependencies into `.venv/`.

---

## Core Configuration (`galaxy.yml`)

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
| `database_connection` | SQLAlchemy URI pointing to PostgreSQL database |
| `file_path` | Where Galaxy stores uploaded/generated dataset files |
| `job_working_directory` | Temporary workspace for running jobs |
| `conda_auto_init` | Galaxy will install Miniconda on first start to resolve tool deps |
| `admin_users` | Email(s) that gain admin panel access |
| `gravity` | Process manager config — Gunicorn serves the web app |
| `tusd` | Enables resumable, large file uploads via the TUS protocol |

---

## Data Directories

Create the directories referenced in `galaxy.yml`:

```bash
sudo mkdir -p /data/galaxy/database/{files,tmp,tmp/tus,jobs_directory}
sudo chown -R $(whoami):$(whoami) /data/galaxy
```

Ensure the disk mounted at `/data` has sufficient space for datasets.

---

## Tool Configuration (`tool_conf.xml`)

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

## Job Configuration (`job_conf.yml`)

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

This tells Galaxy to run all jobs locally with 4 worker threads. See [Remote Job Execution with Pulsar](#remote-job-execution-with-pulsar-optional) for distributed execution.

---

## First Start (Manual)

```bash
cd /path-to-galaxy/galaxy
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

## systemd Service

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
WorkingDirectory=path-to-galaxy
ExecStart=path-to-galaxy/run.sh --daemon
ExecStop=path-to-galaxy/run.sh --stop-daemon
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


## CVMFS for Reference Data (Optional)

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

## OIDC / SSO Authentication (Optional)

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

## Installing Tools from ToolShed

Once Galaxy is running and you have admin access:

1. Navigate to **Admin → Install and Uninstall** in the Galaxy UI.
2. Search the [Galaxy ToolShed](https://toolshed.g2.bx.psu.edu/) for the tool you need (e.g., `clustalo`, `bwa`, `samtools`).
3. Click **Install** — Galaxy downloads the tool XML, test data, and resolves dependencies via Conda.

Installed tools appear in `config/shed_tool_conf.xml` automatically and become available to all users.

---

## Remote Job Execution with Pulsar (Optional)

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

## Maintenance & Troubleshooting

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
/path-to-galaxy/galaxy/
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
