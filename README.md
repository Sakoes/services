# Homelab services

Ten independent Compose projects, run manually with rootful Podman. The sibling
[homelab flake](../homelab) installs Podman and pins its Compose provider, manages
firewall/directory prerequisites, and manages native node-exporter and Alloy with
its journal collection pipeline.
Pi-hole uses the same stack on homelab and Pi, configured per host below.
There are no container boot units, deployment scripts or combined top-level stack.

Review this checkout before running it with sudo: Compose definitions can mount
host files and execute with rootful deployment authority. Keep state and secrets
outside Git. Private env files remain under `/var/lib/<stack>`. No rootless
setup is needed.

**Pending rebuild:** these Compose files still bind `/srv/services/<stack>`, but
homelab no longer provisions service datasets or directories there. Its ZFS mirror
now mounts `tank/files` at `/srv/data`, independently of Compose. Do not start these
stacks until their storage paths, required directories and ownership are reconciled.
The old service-dataset migration plan is obsolete; see the
[host overview](../homelab/README.md).

## Services

Run from the directory below and always use its name with `-p`.

| Directory / project | Listeners | State |
|---|---|---|
| `homepage` | `127.0.0.1:3001` | `/srv/services/homepage` |
| `stremio` | TCP 11470, 12470 | `/srv/services/stremio`, `/srv/downloads/stremio` (SSD cache) |
| `nodeodm` | TCP 3000 | `/srv/services/nodeodm` |
| `webodm` | TCP/UDP 8000 | `/srv/services/webodm/{db,media}` |
| `home-assistant` | host network, TCP 8123 | `/srv/services/home-assistant` |
| `loki` | loopback HTTP 3100 / gRPC 9096 | `/srv/services/loki/data`, UID:GID 10001:10001 |
| `prometheus` | loopback 9090 | `/srv/services/prometheus`, UID:GID 65534:65534 |
| `grafana` | loopback Grafana 3002 / MCP 8001 | `/srv/services/grafana/data`, UID:GID 472:0 |
| `pihole` | configurable DNS/admin binds; homelab defaults `192.168.129.31:53` TCP/UDP and `127.0.0.1:8081` | `/srv/services/pihole/{etc-pihole,etc-dnsmasq.d}` |
| `traefik` | host network, `192.168.129.31:80` | no persistent state |

Homelab opens TCP 53, 80, 3000, 8000, 8123, 11470, 12470 and UDP 53 only.
Pi opens TCP 53/80 and UDP 53.
WebODM's UDP publish does not add a host firewall opening. Keep admin/monitoring
endpoints loopback-only. Routes and Pi-hole records provide `.home` names;
`homelab.home` serves Homepage. HTTP routes are for a trusted LAN, not the internet.

## Configuration and secrets

Config mounts are relative and read-only; keep this checkout at a stable path.
Persistent application state belongs under `/srv/services`, not in the checkout. Stremio's disposable
torrent cache uses `/srv/downloads/stremio`, mounted over the image's default
`/root/.stremio-server/stremio-cache`; leave its cache location at the default.
Before starting any stack, reconcile its bind mounts with the new storage layout
and verify the intended backing filesystem. For Stremio also verify
`findmnt -M /srv/downloads`. Do not start containers with missing storage.
Homelab does not currently create application directories; their exact IDs/modes
must be set as part of the rebuild. Preserve application/database ownership,
never recursively chown state. WebODM shares media between webapp/worker and keeps `:z`/`:Z` mounts;
check SELinux requirements on the target host. Register NodeODM at host:3000 in
WebODM's UI. Do not start parallel projects against the same writable directories
or WebODM's fixed container names.

Private interpolation files must be root:root 0600:

| `--env-file` | Required nonempty keys |
|---|---|
| `/var/lib/pihole/secrets.env` | `WEBPASSWORD` |
| `/var/lib/grafana/secrets.env` | `GF_SECURITY_ADMIN_USER`, `GF_SECURITY_ADMIN_PASSWORD`, `GF_SECURITY_SECRET_KEY`, `GRAFANA_SERVICE_ACCOUNT_TOKEN`, `MCP_GRAFANA_SERVER_TOKEN` |
| `/var/lib/webodm/secrets.env` | `WO_SECRET_KEY` |

These are Compose dotenv files, not shell scripts or wholesale container `env_file`
inputs. Never source them or print rendered `config`/inspect environments. Use
`config -q`; unset inherited overrides when validating. Privately check quoting
for `$`, `${...}`, quotes, spaces, `#` or multiline values: systemd and dotenv
parsing are not equivalent. Tests cover fabricated values, not your actual secrets.
Missing/empty required values fail closed. Use `--env-file /dev/null` for other
stacks so checkout dotenv files cannot be loaded implicitly.

Defaults are in Compose: Brussels timezone, Pi-hole's loopback web UI and WebODM's
port 8000/settings. Change listeners together with the host firewall and routes.
Grafana admin credentials bootstrap a new database, not password rotation. Keep
`GF_SECURITY_SECRET_KEY` stable with any database you retain, or stored credentials
may become unreadable. Back up the key privately with recovery material.

### Pi-hole on either host

Use `pihole/compose.yaml`, the shared `pihole/custom.list`, and `-p pihole` on each
host. The DNS list points `.home` names at homelab, including `grafana.home`, on
both resolvers; Pi serves these records but does not run those applications.
The data and `/var/lib/pihole/secrets.env` are local to each host, never shared.
On Pi, NixOS provides `/srv/services/pihole` as a symlink to the existing
`/var/lib/pihole`; its data stays on Pi's local disk. Activate that prerequisite
before using this updated Compose file on Pi. No Pi deployment is implied by a
homelab storage change. The pending storage reconciliation above applies only to homelab.

Homelab defaults need no overrides. On Pi, add these nonsecret settings to its
private env file alongside `WEBPASSWORD`:

```dotenv
PIHOLE_DNS_BIND=<verified-Pi-LAN-IPv4-address>
PIHOLE_WEB_BIND=0.0.0.0:80
```

Replace the placeholder before validation. Do not use homelab's DNS address on Pi
unless the live inventory confirms that address belongs to it. The Pi admin UI
is LAN-accessible on port 80; homelab keeps it loopback-only behind Traefik.
Both hosts run the same manual commands:

```sh
cd /home/isaac/repos/services/pihole
sudo podman compose --env-file /var/lib/pihole/secrets.env -f compose.yaml -p pihole config -q
sudo podman compose --env-file /var/lib/pihole/secrets.env -f compose.yaml -p pihole up -d --pull never --no-build
```

Back up the active resolver and arrange direct-IP SSH/DNS fallback before cutover.
Stop its Nix-managed controller and retire it through the host configuration
switch before manual startup; do not run both controllers. Change one resolver
at a time. No data or live secret files are moved by this repository change.

### Home Assistant

`/srv/services/home-assistant` is writable config/state. Install the approved
`home-assistant/configuration.yaml` and `automations_manual.yaml` there, merging
any local changes you want to keep. There is no automatic config copier.
Preserve UI-owned `automations.yaml`; if absent, create it as an empty YAML list
`[]` (root:root 0644), without overwriting an existing file. Both includes must
exist before startup. Check HA configuration, discovery and any device access.

## Manual operation

After completing host setup and the pending storage reconciliation,
run from a stack directory. The host's system containers.conf selects the installed
`podman-compose` provider; confirm it is effective for root.

```sh
cd /home/isaac/repos/services/homepage
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage config -q
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage up -d --pull never --no-build
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage logs -f
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage stop
```

For secret-bearing stacks use their private file, for example:

```sh
cd /home/isaac/repos/services/grafana
sudo podman compose --env-file /var/lib/grafana/secrets.env -f compose.yaml -p grafana config -q
sudo podman compose --env-file /var/lib/grafana/secrets.env -f compose.yaml -p grafana up -d --pull never --no-build
```

`stop` keeps containers; `down` removes project containers/network, not bind data.
Do not use `down -v` or prune as routine recovery. Explicitly pull images when
choosing to install/update them; `--pull never` makes a missing local image fail
rather than silently fetch one. `latest`, `stable` and untagged images are mutable;
record image IDs before updates if you need reproducibility.

Start Loki/Prometheus before Grafana, and Pi-hole before Traefik. WebODM manages
its internal database/broker/worker dependencies. Validate health and real workloads
rather than treating detached startup as readiness.

**Nothing starts automatically after reboot.** DNS, routing and applications stay
offline until manually started. Crash restart policies apply once running; they
do not fix unhealthy-but-running apps or order independent projects. Audit other
boot launchers and verify this policy with an approved reboot on each host.
Pi's DNS also stays offline after reboot until its Pi-hole project is started.

## Monitoring

Native node-exporter listens on `127.0.0.1:9100`; Alloy on `127.0.0.1:12345`.
Prometheus scrapes itself, node-exporter, Alloy and Loki. Grafana supplies dashboards
and is the only alert evaluator; notification delivery is not configured.

Alloy's pipeline lives in the homelab repo under `modules/nixos/alloy/`.
NixOS installs its config and reloads changes on activation; there is
no manual config installation here. Alloy needs journal access, not checkout/home
traversal, private env files or a container socket.

Containers use journald tags `compose-<project>.service` (MCP shares Grafana's tag).
Alloy accepts project names starting with a lowercase letter or digit, followed by
lowercase letters, digits, `_` or `-`; adding a project needs no Alloy config edit.
Set this on each new container, replacing `my-project` with its stable project name:

```yaml
logging:
  driver: journald
  options:
    tag: compose-my-project.service
```

Matching tags and selected native units become indexed `unit` labels. Use a stable
project name, not a container ID or timestamp, to avoid excessive Loki streams.
Container identities/raw units are packed into the log message, not indexed.
Check `journalctl CONTAINER_TAG=compose-homepage.service`, then find a harmless
application-generated marker in Loki/Grafana:

```logql
{host="homelab",job="journal",unit="compose-homepage.service"} | unpack |= "<marker>"
```

Source tests do not prove journal fields, permissions or delivery. Verify the actual
marker, native host logs and dashboard filters on the host. Alloy positions are not
a durable send queue: Loki downtime beyond retries can lose logs. Never delete
checkpoints/chunks as routine recovery. Loki keeps 72h of logs, Prometheus 7d/20GB;
retention is not a filesystem quota. Watch `/srv/services` disk space and inodes.

MCP requires a Grafana Viewer service-account token and a separate client bearer
token. It is read-only and loopback-only. Tunnel with
`ssh -N -L 8001:127.0.0.1:8001 <user>@<host>` and connect at
`http://127.0.0.1:8001/mcp` with `Authorization: Bearer <MCP_GRAFANA_SERVER_TOKEN>`.
Never give MCP clients the Grafana service-account token. Verify unauthenticated
requests return 401 and authenticated clients can read tools. Logs/metrics may
contain secrets even for read-only users.

## Checks

Use installed Python with PyYAML and podman-compose; no new framework is needed.
Tests parse Compose with isolated fabricated credentials and never start containers.

```sh
python3 -m unittest discover -s tests -v
# Optional, matching Prometheus 3.15.1:
HOMELAB_PROMTOOL=/path/to/promtool python3 -m unittest discover -s tests -v
```

Live checks remain separate: DNS TCP/UDP and client addresses, routes, loopback
boundaries, secret parsing, application state/jobs, logs, crash restarts and reboot.
Back up Pi-hole before changing the running host. Other services are not yet in use;
keep or reset their data only by explicit choice, never as an automatic setup step.
