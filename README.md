# Raspberry Pi 5 Home Server

Ansible-deployed home server for a Raspberry Pi 5. A single playbook run turns a clean Raspberry Pi OS Lite install into a smart-home host: it installs Docker, renders all configuration, and brings up a stack of services (automation, MQTT, monitoring) as Docker Compose containers.

The project is declarative and reproducible: image tags are pinned, secrets live in Ansible Vault, and role correctness is checked by linters and Molecule in CI.

## Architecture

A control node (a laptop running Ansible) configures a single Raspberry Pi over SSH. The Pi runs one Docker Compose project under `/opt/home-server`; services talk to each other over the internal Compose bridge network, and only the required ports are published.

```
control node (Ansible)  ──ssh──▶  Raspberry Pi 5
                                     ├─ Docker Engine + Compose
                                     └─ /opt/home-server (docker-compose.yml + configs)
                                          ├─ Home Assistant ──┐
                                          ├─ Mosquitto (MQTT) ◀┘ (127.0.0.1:1883)
                                          ├─ Zigbee2MQTT ──▶ Mosquitto   (optional, needs dongle)
                                          ├─ Speedtest Tracker
                                          └─ Prometheus ──▶ scrape ──▶ Node Exporter / cAdvisor
                                                  ▲
                                                  └── Grafana (provisioned datasource + dashboard)
```

Monitoring flow: Node Exporter exposes host metrics, cAdvisor exposes per-container metrics, Prometheus scrapes and stores them, and Grafana visualizes them through an auto-provisioned datasource and the "Home Server" dashboard.

## Components

| Service | Image | Port | Purpose |
| --- | --- | --- | --- |
| Home Assistant | `ghcr.io/home-assistant/home-assistant:2026.6` | 8123 | Smart-home automation hub. Runs in `host` network mode; reaches MQTT at `127.0.0.1:1883`. |
| Mosquitto | `eclipse-mosquitto:2.0.22` | 1883 | MQTT broker. Anonymous access on the LAN by default. |
| Zigbee2MQTT | `ghcr.io/koenkk/zigbee2mqtt:2.12.0` | 8080 | Bridges a Zigbee coordinator to MQTT. Optional, needs a USB dongle; disabled by default. |
| Speedtest Tracker | `lscr.io/linuxserver/speedtest-tracker:1.14.4` | 8081 | Scheduled internet speed tests with history. |
| Prometheus | `prom/prometheus:v3.12.0` | 9090 | Metrics collection and storage (30-day retention). |
| Grafana | `grafana/grafana-oss:13.0.2` | 3000 | Dashboards. Prometheus datasource and the "Home Server" dashboard are provisioned automatically. |
| cAdvisor | `ghcr.io/google/cadvisor:v0.53.0` | 8088 | Per-container resource metrics. |
| Node Exporter | `quay.io/prometheus/node-exporter:v1.11.1` | 9100 | Host metrics (CPU, RAM, disk, network). |
| Uptime Kuma | `louislam/uptime-kuma:1` | 3001 | Uptime monitoring. Optional, disabled by default. |
| Tailscale | `tailscale/tailscale:stable` | — | Mesh VPN for remote access. Optional, disabled by default. |

Each service is toggled by a `*_enabled` flag in `group_vars/all.yml`; disabled services are excluded from the Compose file, the Prometheus config, and the firewall rules.

## Repository layout

```
.
├── ansible.cfg                  # Ansible settings (inventory, vault password file)
├── site.yml                     # playbook: base → docker → home_server roles
├── requirements.yml             # Galaxy collections (community.docker/general, ansible.posix)
├── inventory/
│   └── hosts.yml.example        # inventory template (real hosts.yml is gitignored)
├── group_vars/
│   └── all.yml                  # all configuration: services, ports, images, tuning
├── vault.yml                    # secrets (Ansible Vault, encrypted)
├── vault.example.yml            # secrets template
├── roles/
│   ├── base/                    # OS: packages, hostname, timezone, avahi, UFW
│   ├── docker/                  # Docker Engine, Compose plugin, daemon tuning
│   └── home_server/             # directories, config rendering, stack deployment
├── molecule/default/            # test scenario (converge/idempotence/verify)
└── .github/workflows/ci.yml     # CI: lint + molecule
```

## Roles

**base** — brings the OS to a working state: asserts a Debian-family host, updates the system, installs base packages, sets hostname and timezone, enables avahi-daemon, and optionally configures UFW (the allowed-port list is derived from the enabled services).

**docker** — installs Docker Engine and the Compose plugin from Docker's official repository, enables the service, adds the user to the `docker` group, and — if `docker_daemon_options` is set — writes `/etc/docker/daemon.json` (e.g. MTU) and restarts the daemon.

**home_server** — resolves the Zigbee dongle path (when Zigbee2MQTT is enabled), creates the directory tree under `/opt/home-server`, renders every config (Mosquitto, Zigbee2MQTT, Prometheus, Grafana provisioning, `docker-compose.yml`), deploys the stack via `community.docker.docker_compose_v2`, and waits for services to become ready on their ports (Home Assistant additionally gets an HTTP readiness check).

## Quickstart

**Prerequisites**

- A Raspberry Pi running Raspberry Pi OS Lite 64-bit, with SSH key access and passwordless or `-K`-prompted sudo.
- Ansible on the control machine (`pip install ansible-core` or `brew install ansible`).
- For Zigbee2MQTT: a USB Zigbee coordinator plugged into the Pi.

**1. Install collections**

```bash
ansible-galaxy collection install -r requirements.yml
```

**2. Point Ansible at your Pi**

```bash
cp inventory/hosts.yml.example inventory/hosts.yml
# edit inventory/hosts.yml — set ansible_host and ansible_user
```

```yaml
# inventory/hosts.yml
all:
  hosts:
    pi:
      ansible_host: 192.168.1.10
      ansible_user: youruser
      ansible_python_interpreter: /usr/bin/python3
```

**3. Create the vault with your secrets**

```bash
cp vault.example.yml vault.yml
# fill in real values, e.g.:
#   vault_grafana_admin_password: "$(openssl rand -base64 24)"
#   vault_speedtest_app_key:      "base64:$(openssl rand -base64 32)"
echo 'your-vault-password' > .vault_pass && chmod 600 .vault_pass
ansible-vault encrypt vault.yml
```

**4. Review configuration**

Open `group_vars/all.yml` and check at least `pi_hostname`, `server_timezone`, the `*_enabled` flags, and `zigbee_adapter` if you use Zigbee.

**5. Deploy**

```bash
ansible all -m ping                 # connectivity check
ansible-playbook site.yml -K        # -K prompts for the sudo password
```

The first run pulls multi-gigabyte images and can take a while. Re-running is idempotent.

**6. Access the services** (replace `<pi>` with your Pi's address)

```
Home Assistant     http://<pi>:8123
Grafana            http://<pi>:3000      (admin + vault_grafana_admin_password)
Prometheus         http://<pi>:9090/targets
Speedtest Tracker  http://<pi>:8081      (default admin@example.com / password)
cAdvisor           http://<pi>:8088
```

**Day-2 operations** (on the Pi, in `/opt/home-server`)

```bash
docker compose ps           # status
docker compose logs -f      # logs
docker compose up -d        # apply changes / restart
docker compose down         # stop the stack
```

## Configuration

All user-facing configuration lives in `group_vars/all.yml`:

- **Host:** `pi_hostname`, `server_timezone`, `server_user`, `base_dir`.
- **Services:** `*_enabled` flags and `*_port` ports.
- **Images:** tags are pinned (no `:latest`) for reproducibility.
- **Docker:** `docker_daemon_options` — contents of `daemon.json` (defaults to `mtu: 1280` and `max-concurrent-downloads: 1`, see Networking).
- **Service settings:** Prometheus retention, Speedtest schedule, Zigbee adapter type, etc.

The inventory (`inventory/hosts.yml`) holds the Pi's real address and user and is intentionally not committed — only `hosts.yml.example` is in the repo.

## Testing & CI

- **yamllint** and **ansible-lint** (`basic` profile) — static analysis.
- **Molecule** (`molecule/default`) — `converge → idempotence → verify` of the `base` and `docker` roles in a systemd-enabled Debian 12 container. The full service stack is not tested in CI (it needs a Zigbee dongle and multi-GB images).
- **GitHub Actions** (`.github/workflows/ci.yml`) — `lint` and `molecule` jobs on push to `main` and on PRs. A repository secret `ANSIBLE_VAULT_PASSWORD` is required so CI can decrypt the vault.

Tasks that can't run inside a container (editing `/etc/hostname`, `/etc/hosts`) are tagged `molecule-notest`; they are skipped only under Molecule and run normally on a real host.

## Security model

The defaults assume a trusted LAN:

- Mosquitto allows anonymous connections (`mosquitto_allow_anonymous: true`).
- UFW is disabled; when enabled, the open TCP port list is built from the active services.
- External access (Tailscale) is disabled.

To harden: enable UFW, configure Mosquitto authentication/ACLs, use Tailscale instead of exposing ports, and protect the Speedtest Prometheus endpoint with a token (`speedtest_metrics_enabled` + `vault_speedtest_metrics_bearer_token`).

## Networking

Image pulls from some CDNs can stall on double-NAT/PPPoE links because of a Path-MTU-Discovery black hole: the TLS handshake succeeds but large layers hang. The fix is to lower Docker's MTU (`docker_daemon_options.mtu`, default 1280) — the `docker` role writes this to `daemon.json` before deploying. A network-wide fix is MSS clamping on the router's WAN interface. If the IPv6 route to a registry is broken, restrict Docker to IPv4.
