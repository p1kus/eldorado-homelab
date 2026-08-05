# Eldorado

Homelab designed as a small, production-like environment. this helped me dive deep into Windows Server administration, virtualization, containerization, monitoring, networking, automation and identity management.

## Goals

- Learn technologies used in enterprise environments.
- Build reproducible and well-documented infrastructure.
- Centralize authentication and implement SSO.
- Monitor services, hosts and network devices.
- Automate deployments and maintenance.
- Practice backup and disaster recovery.
- Store infrastructure configuration in Git.

## Hardware

### HP Mini, Proxmox

| ------------- | --------------------- |
| CPU           | Intel Core i5-9400T   |
| RAM           | 16 GB DDR4            |
| System disk   | 512 GB NVMe           |
| Media storage | None yet              |
| Hypervisor    | Proxmox VE            |

Planned role:

- Virtual machines and test environments.
- Windows Server and Active Directory.
- Proxmox LXC containers.
- Heavier services
- Media storage

### Raspberry Pi 5

| --------- | ----------- |
| Role      | Docker host |
| Storage   | 256 GB NVMe |
| Network   | Ethernet    |
| Power     | PoE         |

The Raspberry Pi 5 is the heart of the homelab used for the Docker services

### Network

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

The Crontab Guru stack requires the existing `Dockerfile` and application files under `stacks/crontab-guru/`. They were not included in the original configuration and therefore cannot be recreated by this repository.


## Daily Operations

### Entire environment

```bash
docker compose config --quiet
```

`config` resolves and validates the complete configuration


```bash
docker compose up -d
```

`up` creates or updates the services. The `-d` detaches from the active terminal window.

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

`docker compose down -v` removes data stored in named volumes!!


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

- Dockhand has access to `/var/run/docker.sock`, HAS to be behind auth
- Set `N8N_SECURE_COOKIE=true` after exposing n8n to the internet
- Pi-hole listens on all interfaces. Restrict access to its DNS port on it's firewall.
- The images currently use the `latest` tag. Pinning specific versions will make future updates more predictable.

## Roadmap
- [ ] Deploy Authentik and configure Authentik
- [ ] Deploy Termix for managing SSH connections
- [ ] Deploy and configure Jellyfin
- [ ] Implement automated backups and test the restore process.
- [ ] Create separate VLANs for servers, IoT devices and guests.
- [ ] Replace `latest` tags with pinned image versions.


