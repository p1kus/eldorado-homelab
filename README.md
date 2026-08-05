# Eldorado

> Home infrastructure for learning System Administration, DevOps, Networking and Automation.

Homelab designed as a small, production-like environment. My main goal while creating the system was learning Windows Server administration, virtualization, containerization, monitoring, networking, automation and identity management.

## Goals

- Learn technologies used in enterprise environments.
- Build reproducible and well-documented infrastructure.
- Centralize authentication and implement SSO.
- Monitor services, hosts and network devices.
- Automate deployments and maintenance.
- Practice backup and disaster recovery.
- Store infrastructure configuration in Git.

## Hardware

### HP Mini — Proxmox

| Component     | Value                 |
| ------------- | --------------------- |
| CPU           | Intel Core i5-9400T   |
| RAM           | 16 GB DDR4            |
| System disk   | 256 GB NVMe           |
| Media storage | Samsung T5 1 TB USB-C |
| Hypervisor    | Proxmox VE            |

Planned role:

- Virtual machines and test environments.
- Windows Server and Active Directory.
- A dedicated Docker VM.
- Services requiring more resources than the Raspberry Pi can provide.
- Media storage on the Samsung T5.

### Raspberry Pi 5

| Component | Value       |
| --------- | ----------- |
| Role      | Docker host |
| Storage   | 256 GB NVMe |
| Network   | Ethernet    |
| Power     | PoE         |

The Raspberry Pi 5 is the current host for the Docker services defined in this repository.

### Raspberry Pi Zero 2 W

Planned roles:

- Secondary Pi-hole.
- DNS redundancy.
- Small status display.
- Emergency network services.

### Network

| Function | Solution                              |
| -------- | ------------------------------------- |
| Router   | MikroTik hAP ax²                      |
| Switch   | TP-LINK SG108P PoE switch             |
| DNS      | Primary Pi-hole and secondary Pi-hole |
| VPN      | WireGuard                             |

## Architecture

```text
Internet
    │
WireGuard
    │
MikroTik hAP ax²
    │
    ├── HP Mini
    │   └── Proxmox VE
    │       ├── Docker VM
    │       ├── Windows Server
    |       ├── Pi-Hole DNS (Secondary to RPI5 Pi-Hole)
    │       └── Test Linux VMs
    |
    │
    └── Raspberry Pi 5
        └── Docker Engine

```

## Virtual Machines

### Docker VM

Planned purpose:

- Docker Engine.
- Reverse proxy.
- SSO.
- Services requiring more resources than the Raspberry Pi can provide.

### Windows Server

Planned purpose:

- Active Directory.
- DNS and lab DHCP.
- Group Policy.
- Windows environment administration.

### Future VMs

- Ubuntu Server.
- Temporary test environments.

## Docker Services

### Currently defined

| Area          | Stack               | Containers                               |
| ------------- | ------------------- | ---------------------------------------- |
| Management    | Dockhand            | `dockhand`                               |
| Reverse proxy | Nginx Proxy Manager | `npm`                                    |
| Monitoring    | Monitoring          | `prometheus`, `grafana`, `node-exporter` |
| DNS           | Pi-hole             | `pihole`                                 |
| Remote access | RustDesk Server     | `hbbs`, `hbbr`                           |
| Automation    | n8n                 | `n8n`                                    |
| Tools         | Omni Tools          | `omni-tools`                             |
| Scheduling    | Crontab Guru        | `crontab-guru-dashboard`                 |

### Planned

- Authentik — SSO, OIDC and MFA.
- Jellyfin — media server.
- Node-RED — automation.
- cAdvisor — container metrics.
- Blackbox Exporter — service availability monitoring.

## Repository Layout

```text
eldorado/
├── compose.yml
└── stacks/
    ├── dockhand/
    ├── rustdesk/
    ├── pihole/
    ├── monitoring/
    ├── omni-tools/
    ├── nginx-proxy-manager/
    ├── n8n/
    └── crontab-guru/
```

The root `compose.yml` uses `include` to combine all projects into one environment named `eldorado`. Each directory under `stacks/` also remains an independent Compose project that can be started and updated separately.

Each stack has its own:

- `compose.yml` containing the service definitions.
- `.env.example` providing a safe configuration template.
- Local `.env` containing settings and secrets that must not be committed to Git.
- Directories or named volumes containing persistent data.

## Requirements

- Docker Engine.
- Docker Compose 2.20.0 or newer because the root file uses `include`.
- Linux for services using host networking or mounting `/proc`, `/sys` and `/`.
- The configured host ports must be available.

Check the installed Docker Compose version:

```bash
docker compose version
```

This command prints the installed Docker Compose version and does not modify the system.

## Initial Setup

### 1. Prepare the configuration

Create a local `.env` file for every stack by copying its example:

```bash
cp stacks/dockhand/.env.example stacks/dockhand/.env
cp stacks/rustdesk/.env.example stacks/rustdesk/.env
cp stacks/pihole/.env.example stacks/pihole/.env
cp stacks/monitoring/.env.example stacks/monitoring/.env
cp stacks/omni-tools/.env.example stacks/omni-tools/.env
cp stacks/nginx-proxy-manager/.env.example stacks/nginx-proxy-manager/.env
cp stacks/n8n/.env.example stacks/n8n/.env
cp stacks/crontab-guru/.env.example stacks/crontab-guru/.env
```

`cp` copies each source file to the specified destination. These commands do not use any additional flags.

Next:

- Replace every value beginning with `ZMIEN_`.
- Set the correct `HOMELAB_PATH` for Dockhand.
- Generate a long, random `N8N_ENCRYPTION_KEY`.
- Keep a backup of the `.env` files outside the repository.

The Crontab Guru stack requires the existing `Dockerfile` and application files under `stacks/crontab-guru/`. They were not included in the original configuration and therefore cannot be recreated by this repository.

### 2. Validate the configuration

```bash
docker compose config --quiet
```

`config` resolves and validates the complete configuration. The `--quiet` flag suppresses valid output and only reports errors.

### 3. Start Eldorado

```bash
docker compose up -d
```

`up` creates or updates the services. The `-d` flag leaves them running in the background.

## Daily Operations

### Entire environment

```bash
docker compose ps
docker compose logs -f
docker compose pull
docker compose up -d
docker compose down
```

- `ps` shows the current container status.
- `logs` prints container logs, while `-f` follows new entries.
- `pull` downloads newer versions of the configured images.
- `up -d` recreates changed containers and leaves them running in the background.
- `down` stops and removes the project containers and networks while preserving named volumes.

Do not use `docker compose down -v` unless you intend to remove data stored in named volumes. The `-v` flag instructs Compose to remove those volumes.

### Individual stack

The simplest method is to enter the stack directory:

```bash
cd stacks/monitoring
docker compose up -d
```

`cd` changes the current directory. The following command automatically uses the `compose.yml` and `.env` files located there.

You can also manage a stack from the repository root:

```bash
docker compose --env-file stacks/monitoring/.env -f stacks/monitoring/compose.yml up -d
```

The `--env-file` flag selects the environment file, while `-f` selects a specific Compose file. `up -d` starts the selected stack in the background.

## Default Ports

| Service             |         Host port |
| ------------------- | ----------------: |
| Dockhand            |              3000 |
| RustDesk            | Host network mode |
| Pi-hole DNS         |        53 TCP/UDP |
| Pi-hole HTTP/HTTPS  |       8080 / 8443 |
| Prometheus          |              9090 |
| Grafana             |              3001 |
| Node Exporter       |              9100 |
| Omni Tools          |              8081 |
| Nginx Proxy Manager |     80 / 443 / 81 |
| n8n                 |              5678 |
| Crontab Guru        |              9000 |

Ports for services that do not use host networking can be changed in their respective `.env` files.

## Data and Backups

Backups should include at least:

- Named volumes `dockhand_data`, `prometheus_data`, `grafana_data`, `n8n_storage` and `cronitor_data`.
- `stacks/rustdesk/data/`.
- `stacks/pihole/etc-pihole/`.
- `stacks/nginx-proxy-manager/data/`.
- `stacks/nginx-proxy-manager/letsencrypt/`.
- `stacks/crontab-guru/crontabs/`.
- All local `.env` files stored in a secure secret store.

Persistent data directories, local databases, certificates, private keys and `.env` files are ignored by Git.

## Security

- Never commit `.env` files or private keys. Only `.env.example` files should be tracked by Git.
- Dockhand has access to `/var/run/docker.sock`, which effectively gives it extensive control over the Docker host. Restrict the panel to trusted networks and protect it with authentication.
- Set `N8N_SECURE_COOKIE=true` after placing n8n behind HTTPS.
- Pi-hole listens on all interfaces. Restrict access to its DNS port using the firewall.
- Nginx Proxy Manager stores private TLS keys in the `letsencrypt/` directory.
- The images currently use the `latest` tag. Pinning specific versions will make future updates more predictable.

## Roadmap

- [ ] Move resource-intensive services to the Docker VM running on Proxmox.
- [ ] Deploy Authentik and OIDC authentication.
- [ ] Deploy Jellyfin with its media library stored on the Samsung T5.
- [ ] Add Node-RED.
- [ ] Extend monitoring with cAdvisor and Blackbox Exporter.
- [ ] Deploy Alertmanager and notifications.
- [ ] Run a secondary Pi-hole on the Raspberry Pi Zero 2 W.
- [ ] Implement automated backups and test the restore process.
- [ ] Create separate VLANs for servers, IoT devices and guests.
- [ ] Replace `latest` tags with pinned image versions.

## Development Principles

- Use one Compose file per logical stack.
- Keep secrets outside Git.
- Every important service should have monitoring and backups.
- Infrastructure changes should be documented and committed to the repository.
