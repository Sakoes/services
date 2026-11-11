# Homelab services

Independent Compose projects, run manually with rootful Podman. The sibling
[homelab flake](../homelab) owns Podman, the firewall, per-stack state
directories and native node-exporter/Alloy. Nothing here starts at boot.

## Rebuild status

The stacks are being rebuilt one at a time. Do **not** start a stack that is not
marked rebuilt: its mounts, image pins and docs are leftovers from experiments.

| Stack | Status |
|---|---|
| `traefik`, `homepage` | pending |
| `pihole` | pending (Pi still runs its existing setup) |
| `loki`, `prometheus`, `grafana` | pending |
| `home-assistant` | pending |
| `stremio` | pending |
| `nodeodm`, `webodm` | pending |

## Conventions

- **State:** `/srv/services/<stack>/` on homelab (ZFS `tank/services`). Nix creates
  the directories with their exact owners/modes; Compose only bind-mounts them.
  Never point a mount at the checkout or at a missing path.
- **Secrets:** `/srv/services/<stack>/secrets.env`, root:root 0600, created by hand,
  never committed. Pass it with `--env-file`; stacks without secrets use
  `--env-file /dev/null` so no checkout dotenv is loaded.
- **Network:** apps listen on `127.0.0.1` (or host network bound to loopback) and are
  reached through Traefik at `<name>.home`. Only Pi-hole (53) and Traefik (80) face
  the LAN. A new app needs a route in `traefik/routes.yml` and a record in
  `pihole/custom.list`.
- **Images:** fully qualified and pinned to a version tag.
- **Logs:** every container logs to journald with tag `compose-<project>.service`;
  Alloy turns that into Loki's `unit` label, no Alloy change needed:

  ```yaml
  logging:
    driver: journald
    options:
      tag: compose-<project>.service
  ```

## Running a stack

From the stack directory, always with `-p <stack>`:

```sh
findmnt -M /srv/services        # state dataset must be mounted
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage config -q
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage up -d
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage logs -f
sudo podman compose --env-file /dev/null -f compose.yaml -p homepage stop
```

`stop` keeps containers; `down` removes containers/network but not bind data.
Never use `down -v` or prune as routine recovery. Never print rendered `config`
for secret-bearing stacks.
