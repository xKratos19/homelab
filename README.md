# Homelab Infrastructure
 
A self-hosted homelab running on Proxmox VE, built to learn and demonstrate DevOps practices — infrastructure as code, container orchestration, monitoring, and CI/CD automation.
 
> Coming from a background in IT and programming, this project documents my transition into DevOps through hands-on infrastructure work.
 
---
 
## What's Running
 
A Proxmox VE host managing **10 LXC containers** across multiple workloads:
 
| Container | Role | Stack |
|-----------|------|-------|
| `apps` | Primary Docker host | Immich, Arr stack, Insurance Tracker, WordPress, Portfolio |
| `n8n` | Workflow automation | n8n |
| `adguard` | Network DNS blocker | AdGuard Home |
| `jellyfin` | Media server | Jellyfin |
| `wizarr` | Jellyfin invite management | Wizarr |
| `lyrion` | Music streaming | Lyrion Music Server |
| `myspeed` | Internet speed tracking | MySpeed |
| `monitoring` | Observability stack | Prometheus + Grafana |
| `homeassistant` | Home automation | Home Assistant |
 
---
 
## Infrastructure Overview
 
```
Proxmox VE Host (bare metal)
│
├── LXC 101 — apps          → 25+ Docker containers
├── LXC 102 — n8n           → Automation workflows
├── LXC 103 — adguard       → DNS for entire LAN
├── LXC 105 — jellyfin      → Media streaming
├── LXC 106 — wizarr        → Jellyfin user management
├── LXC 108 — myspeed       → Speed test monitoring
├── LXC 109 — homeassistant → Home automation (QEMU VM)
├── LXC 110 — lyrion        → Music server
└── LXC 111 — monitoring    → Prometheus + Grafana
```
 
---
 
## DevOps Practices
 
### Infrastructure as Code — Ansible
 
All LXC containers are managed via Ansible from the Proxmox host.
 
```bash
# Audit all containers — generates Markdown report per LXC
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/audit.yml
 
# Deploy node_exporter to all containers
ansible-playbook -i ansible/inventory/hosts.yml ansible/playbooks/monitoring.yml
```
 
Roles built:
- **audit** — collects OS, memory, disk, running services, Docker containers per host
- **node_exporter** — installs and configures Prometheus exporter on all LXCs
- **monitoring** — deploys full Prometheus + Grafana stack
 
### GitOps — GitHub Actions
 
Docker stacks on LXC 101 are managed via Git. Pushing a change to any `docker-compose.yml` automatically redeploys that stack.
 
```
Push to main (lxc/101-apps/*)
    └─► Self-hosted runner on LXC 101 detects changed stack
            └─► docker compose up -d --remove-orphans
                    └─► Verify containers running
```
 
**Self-hosted runner** runs directly on LXC 101 — no public IP or port forwarding required.
 
Stacks managed via GitOps:
- `immich` — photo management
- `arr` — media automation (Radarr, Sonarr, Prowlarr, Bazarr, qBittorrent)
- `insurance-tracker` — custom app (Python + MongoDB)
- `portfolio` — personal website
- `wordpress` — WordPress instance
- `jellystat` — Jellyfin statistics
 
### Monitoring — Prometheus + Grafana
 
Full observability across all containers and the Proxmox host.
 
**Metrics sources:**
- `node_exporter` on every LXC → system metrics (CPU, RAM, disk, network)
- `pve-exporter` on Proxmox host → per-VM/LXC resource allocation via Proxmox API
- `prometheus-node-exporter` built into Proxmox → host-level metrics
 
**Custom Grafana dashboard** showing:
- Host CPU, RAM, disk usage (gauges + time series)
- Per-container CPU and memory history
- LXC status table (running/stopped)
- Network in/out per container
- Disk usage per container
 
---
 
## Repository Structure
 
```
homelab/
├── README.md
│
├── ansible/                        # Infrastructure automation
│   ├── inventory/hosts.yml         # All LXCs grouped by role
│   ├── group_vars/all.yml          # Shared variables
│   ├── playbooks/
│   │   ├── audit.yml               # Generate per-LXC reports
│   │   ├── monitoring.yml          # Deploy node_exporter everywhere
│   │   └── setup-monitoring.yml    # Deploy Prometheus + Grafana
│   └── roles/
│       ├── audit/                  # Fact collection + Markdown report
│       ├── node_exporter/          # Prometheus metrics agent
│       └── monitoring/             # Prometheus + Grafana stack
│
├── lxc/
│   └── 101-apps/                   # Docker stacks (GitOps managed)
│       ├── immich/
│       ├── arr/
│       ├── insurance-tracker/
│       ├── portfolio/
│       ├── wordpress/
│       └── jellystat/
│
├── docs/
│   └── lxc/                        # Per-container documentation
│       ├── 101-apps.md
│       ├── 102-n8n.md
│       └── ...
│
└── .github/
    └── workflows/
        └── deploy-apps.yml         # GitOps pipeline for LXC 101
```
 
---
 
## Tech Stack
 
| Category | Technology |
|----------|------------|
| Hypervisor | Proxmox VE |
| Containers | LXC (Linux Containers) |
| Container runtime | Docker + Docker Compose |
| IaC / Config management | Ansible |
| CI/CD | GitHub Actions (self-hosted runner) |
| Monitoring | Prometheus + Grafana |
| Metrics agents | node_exporter, pve-exporter |
| DNS | AdGuard Home |
| OS | Debian 12 (all LXCs) |
 
---
 
## Key Decisions
 
**Why LXC instead of VMs for most workloads?**
LXC containers share the host kernel, making them significantly lighter than full VMs. With only 2 cores and 15GB RAM on the host, this allows running 10 isolated environments simultaneously.
 
**Why a self-hosted GitHub Actions runner?**
The homelab runs on a private LAN with no public IP. A self-hosted runner eliminates the need for port forwarding or VPN — the runner initiates an outbound connection to GitHub, not the other way around.
 
**Why Ansible over manual setup?**
Every service configuration is reproducible. If a container is destroyed, the entire setup can be recreated with a single playbook run. It also documents exactly what's installed and how.
 
**Why pve-exporter alongside node_exporter?**
`node_exporter` gives per-process system metrics inside each LXC. `pve-exporter` queries the Proxmox API directly, providing a bird's-eye view of resource allocation across all guests — two complementary data sources.
 
---
 
## Security Practices
 
- All secrets stored in `.env` files — never committed to Git
- `.env.example` files document required variables without exposing values
- SSH key authentication only — no password auth on any LXC
- AdGuard Home as network DNS — blocks malicious domains at the resolver level
- Private LAN IPs only — no services directly exposed to the internet
 
---
 
## What's Next
 
- [ ] Ansible Vault for encrypted secret management
- [ ] Extend GitOps to remaining LXCs
- [ ] Alerting rules in Grafana (disk > 85%, container down)
- [ ] Automated backups with documented restore procedure
- [ ] Gitea self-hosted for internal pipelines
