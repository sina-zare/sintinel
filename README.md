# Sintinel — centralized backups with restic + rest-server

**Small, containerized backup agent that connects to remote hosts over SSH, creates PostgreSQL dumps or archives directories, and pushes encrypted, deduplicated snapshots to a central rest-server repository using Restic.**

---

* **Repo:** containerized `rest-server` + `sintinel` agent via `docker-compose` and a `.env`-driven configuration
* **Goal:** simple, repeatable backups of remote hosts (psql dumps or directory archives) into a single centralized Restic HTTP repository

---

## Table of contents

* [Overview](#overview)
* [Features](#features)
* [Requirements](#requirements)
* [Quick start (5 minutes)](#quick-start)
* [Configuration (`.env`)](#configuration-env)
* [`sintinel_clients.txt` format](#sintinel_clientstxt-format)
* [Docker Compose (example)](#docker-compose-example)
* [Keys, permissions and security notes](#keys-permissions-and-security-notes)
* [How backups work (flow)](#how-backups-work-flow)
* [Monitoring & metrics](#monitoring--metrics)
* [Troubleshooting & FAQ](#troubleshooting--faq)
* [Security recommendations](#security-recommendations)
* [License & Contributing](#license--contributing)

---

## Overview

* **restserver** — `restic/rest-server`: HTTP endpoint that stores restic repositories.
* **sintinel** — backup agent container that reads `sintinel_clients.txt`, connects to each client via SSH, performs either a `pg_dump` (for PostgreSQL backups) or archives a directory, then pushes snapshots to the `rest-server` repository using Restic.

Snapshots are encrypted, compressed and deduplicated by Restic.

---

## Features

* Agentless backups over SSH (no agent required on the client)
* PostgreSQL dumps (`pg_dump`) or directory archives (`tar`)
* Pushes encrypted Restic snapshots to central `rest-server`
* Retention policy support (`restic forget + prune`)
* Optional metrics pushed to Prometheus Pushgateway

---

## Requirements

* Docker and Docker Compose on the host that runs the stack
* SSH access from the `sintinel` container to each client (private key mounted into container)
* `sintinel_clients.txt` (one line per client)
* Reasonable filesystem permissions on mounted volumes for persistence

---

## Quick start

1. Clone this repo to the host that will run the stack.
2. Place your private SSH key on the host (example path used by compose: `/sintinel/id_ed25519_sintinel`).
3. Create or adapt a `.env` (example below). Ensure `SSH_KEY_NAME` matches the filename you mount.
   
4. Start the restserver:

```bash
# from the directory containing docker-compose.yml
docker compose up -d restserver
```

5. Create restserver users (each host to be backed up needs a restserver user, password, repository and repository password):

```bash
# Create restserver user
docker compose exec -it restserver create_user username password

# Delete restserver user
docker compose exec -it restserver delete_user username
```
6. Create `sintinel_clients.txt` in `/sintinel` (or the path you mount). See format in [sintinel_clients.txt format](#sintinel_clientstxt-format).
7. Give appropriate permissions to the repo parent directory (example `sintinel` user inside container has uid 1000):

```bash
sudo chown -R 1000:1000 /opt/sintinel
```

Usage examples (from the folder with `docker-compose.yml`):

```bash
# Get backup of a single node
docker compose run --rm sintinel backup

# Get backup of all nodes
docker compose run --rm sintinel backup all

# List backups of a single node
docker compose run --rm sintinel list

# List backups of all nodes
docker compose run --rm sintinel list all

# Restore (single node): accepts ID | latest | all
docker compose run --rm sintinel restore

# Restore latest backup of all nodes
docker compose run --rm sintinel restore all

# Remove (single node): accepts ID | latest | all
docker compose run --rm sintinel remove

# Remove backups of all nodes
docker compose run --rm sintinel remove all
```

Optional: Add a shell alias to shorten commands (example edits `/etc/bash.bashrc`):

```bash
sudo tee -a /etc/bash.bashrc > /dev/null <<'EOF'
alias sintinel='sudo docker compose -f /opt/sintinel/docker-compose.yml run --rm sintinel'
EOF

# reload
source /etc/bash.bashrc

# then use
sintinel backup
sintinel list all
```



# Important Notes

To ensure successful backups with **Sintinel**, please follow these prerequisites on all client machines.


## Client Requirements

### 1. SSH Connection

Ensure SSH key-based authentication is configured properly between the Sintinel server and backup clients.

```bash
# Create SSH key pairs using the ed25519 algorithm
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_sintinel -C "backup_automation" -N ""

# Output:
# ~/.ssh/id_ed25519_sintinel      → your private key
# ~/.ssh/id_ed25519_sintinel.pub  → your public key
```

Now, from the Sintinel server, connect to each client and append the server's public key to the client's `authorized_keys`:

```bash
ssh username@client-ip "mkdir -p ~/.ssh && chmod 700 ~/.ssh && printf 'no-port-forwarding,no-agent-forwarding,no-X11-forwarding %s\n' \"$(<~/.ssh/id_ed25519_sintinel.pub)\" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

### 2. PostgreSQL Dumps

On each PostgreSQL database server, save the `postgres` credentials in the `.pgpass` file of the `sysops` user. Limit file access permissions to the `sysops` user only.

Replace `secret` with your actual PostgreSQL password.

```bash
echo "localhost:5432:*:postgres:secret" > ~/.pgpass
chmod 600 ~/.pgpass
```

This allows non-interactive access for automated backups.

---

### 3. Directory Backups with Sudo Access

If the directories to be backed up require **sudo** access that the `SSH_USER` does not normally have, you must grant passwordless sudo access for the `tar` command.

```bash
# Create a new sudoers file for your SSH_USER
sudo visudo -f /etc/sudoers.d/ssh-user-tar

# Add the following line inside the file:
SSH_USER ALL=(ALL) NOPASSWD: /usr/bin/tar
```

This allows Sintinel to execute directory backups without manual intervention.





## Checking Logs

To monitor logs and ensure proper operation:

### Rest Server Logs

```bash
docker compose logs -f restserver
```

### Sintinel Logs

```bash
tail -f /sintinel/sintinel-data/logs/sintinel.logs
```


**Tip:** Always verify connectivity and permissions before scheduling production backups.



---

## Configuration (`.env`)

Create a `.env` file next to `docker-compose.yml`. Example:

```ini
# --- REST Server ---
RESTSERVER_HOST=restserver                              # Hostname or IP of rest-server (as resolvable by the container) (HTTP protocol used for simplicity)
RESTSERVER_PORT=8000                                    # port where rest-server listens (default 8000)
 
# --- SSH config ---
SSH_USER=ubuntu                                         # default SSH user for connecting to clients
SSH_KEY_NAME=id_ed25519_sintinel                        # filename of the private key as mounted into the container
 
# --- Backup retention policy ---
BACKUP_RETENTION_POLICY=last:5,daily:5                  # Example: last:5,daily:7,weekly:4,monthly:1  (keep 5 last snapshots, 7 daily, 4 weekly, 1 monthly)
 
# --- Logging ---
LOG_LEVEL=INFO                                          # log level
LOG_MAX_MEGABYTES=10                                    # maximum log file size
LOG_BACKUP_COUNT=3                                      # keeps up to {LOG_BACKUP_COUNT} log files if one grows over {LOG_MAX_MEGABYTES} size.
 
# --- User Output ---
HOST_RESTORE_DIR=/opt/sintinel/sintinel-data/restore    # optional path to copy restored data for users
 
# --- PushGateway (optional) ---
PUSHGATEWAY_URL=http://pushgateway.example.com:9091     # pushgateway URL
PUSHGATEWAY_JOB=(optional)                              # defaults to docker_containers
PUSHGATEWAY_INSTANCE=(optional)                         # defaults to sintinel
PUSHGATEWAY_TARGET=(optional)                           # defaults to sintinel_clients
```

**Notes**:

* `RESTSERVER_HOST` should be resolvable from inside the `sintinel` container. When using `docker-compose` the service name `restserver` resolves automatically.
* `SSH_KEY_NAME` must match the filename you mount into `/home/sintinel/.ssh/` in the compose file.

---

## `sintinel_clients.txt` format

Each line defines one remote client. Fields are colon-separated.

**Format (for reference)**

```
# Hostname:Host_IP:Restic_User:Restic_Password:Repo_Password:Backup_Type:Backup_Dir
```

**Example**

```
my-db:192.0.2.10:restic-user:RESTIC_USER_PASSWORD:REPO_PASSWORD:psql:
my-fileserver:192.0.2.11:restic-user:RESTIC_USER_PASSWORD:REPO_PASSWORD:dir:/var/www
```

**Fields**

* `Hostname` — logical name used for the restic repository and labels.
* `Host_IP` — IP or hostname reachable via SSH from the `sintinel` container.
* `Restic_User` — username used within the restserver for rest-server authentication (created above).
* `Restic_Password` — password used to authenticate the `Restic_User`.
* `Repo_Password` — repository password used to encrypt the restic repo for that host (managed by Sintinel).
* `Backup_Type` — `psql` or `dir`.

  * `psql`: Sintinel will run a `pg_dump` on the remote host (SSH user must have rights).
  * `dir`: Sintinel archives `Backup_Dir` on the remote host.
* `Backup_Dir` — full path to backup for `dir` entries. Leave empty for `psql`.

**Security**: `sintinel_clients.txt` contains sensitive passwords. Keep file permissions strict (e.g. `chmod 600`).

---

## Docker Compose (example)

The example `docker-compose.yml` contains two services: `restserver` and `sintinel`. `restserver` exposes port `8000` and stores data under `./sintinel-data/rest-data` on the host. `sintinel` mounts `sintinel_clients.txt`, SSH private key, and local directories for cache, restore and logs.

```yaml
services:
  restserver:
    image: restic/rest-server:latest
    container_name: restserver
    ports:
      - "8000:8000"
    environment:
      OPTIONS: "--prometheus --prometheus-no-auth --private-repos"
    volumes:
      - ./sintinel-data/rest-data:/data
    restart: always
 
  sintinel:
    image: sintinel:0.1
    container_name: sintinel
    env_file:
      - .env
    depends_on:
      - restserver
    volumes:
      # clients list (read-only)
      - /sintinel/sintinel_clients.txt:/opt/scripts/sintinel_clients.txt:ro
 
      # mount private key at the same filename that SSH_KEY_NAME points to
      - /sintinel/id_ed25519_sintinel:/home/sintinel/.ssh/id_ed25519_sintinel:ro
 
      # persist restore/cache/logs on host
      - /sintinel/sintinel-data/restore:/opt/sintinel/restore
      - /sintinel/sintinel-data/cache:/opt/sintinel/cache
      - /sintinel/sintinel-data/logs:/var/log/sintinel
    restart: on-failure
```

---

## Keys, permissions and security notes

* Put the private key on the host and mount it read-only into the container as shown above.
* Private key file should be `600` and owned by the host user that runs Docker: `chmod 600 /sintinel/id_ed25519_sintinel`.
* Inside the container the key path must match `SSH_KEY_NAME`.

---

## How backups work (flow)

1. Sintinel reads `sintinel_clients.txt` line by line.
2. For each entry it SSHes into the client using `SSH_USER` and the mounted key.
3. If `Backup_Type=psql`, a `pg_dump` is executed on the remote host and streamed back into the container.
4. If `Backup_Type=dir`, files under `Backup_Dir` are streamed (tar/stream) and added to a restic repository.
5. Restic pushes the snapshot to the `rest-server` repository under a repo name derived from the `Hostname` field.
6. Sintinel applies `BACKUP_RETENTION_POLICY` using `restic forget` and `restic prune`.
7. If configured, metrics are pushed to Prometheus Pushgateway.

---

## Monitoring & metrics

* `rest-server` can expose Prometheus metrics with `--prometheus` (enabled in the example compose with `OPTIONS`).
* Sintinel can push metrics to a Pushgateway when `PUSHGATEWAY_URL` is set in `.env`.

---

## Troubleshooting & FAQ

**Permission denied to the SSH key**

* Ensure the mounted key file on the host is mode `600` and readable by Docker.

**sintinel cannot reach restserver by name**

* With `docker-compose` the service name `restserver` resolves internally. If you run containers separately, set `RESTSERVER_HOST` to the IP/hostname reachable from the `sintinel` container.

**Backups succeed locally but not remotely**

* Check the remote host's SSH logs and verify the SSH user has the permissions to run `pg_dump` or read the directory.

**Repos not found / unauthorized**

* Verify restserver user credentials and repo passwords. `--private-repos` requires proper authentication.

**Retention policies not applied**

* Sintinel runs `restic forget` + `prune` based on `BACKUP_RETENTION_POLICY` — check Sintinel logs for forget/prune output.

**View logs**

```bash
# restserver logs
docker compose logs -f restserver

# sintinel logs (host path)
tail -f /sintinel/sintinel-data/logs/sintinel.logs
```

---

## Security recommendations

* Use `chmod 600` for private keys and `sintinel_clients.txt`.
* Limit network access to the `rest-server` (firewall, reverse-proxy with TLS) if exposing outside a local network.
* Periodically back up `./sintinel-data/rest-data` (the rest-server storage) — this is where snapshots are physically stored on the host.

