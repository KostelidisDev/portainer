# Portainer

Docker Compose setup for running [Portainer CE](https://www.portainer.io/) as a single, hardened container.

Two variants are provided:

| File                           | Use for                                                                                                                                       |
|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `docker-compose.yml`           | Standard Docker hosts. Includes CPU, memory and PID limits.                                                                                   |
| `docker-compose.synology.yml`  | Synology NAS (Container Manager). Mounts Synology's Docker volumes directory and omits resource limits and `start_interval`, which Synology's Docker version doesn't support. |

## Quick start

```sh
# Optional: copy the example settings and adjust them
cp .env.example .env

# Standard host
docker compose up -d

# Synology
docker compose -f docker-compose.synology.yml up -d
```

Then open the UI at <https://127.0.0.1:9443> and create the admin user. Portainer's initial setup window times out after a few minutes; if it does, restart the container with `docker compose restart`.

## Accessing the UI

The web UI (port `9443`) is bound to `127.0.0.1` only, so it isn't reachable from the network. To reach it from another machine, either put it behind a reverse proxy on the host or use an SSH tunnel:

```sh
ssh -L 9443:127.0.0.1:9443 user@docker-host
# then browse to https://127.0.0.1:9443
```

Portainer uses a self-signed certificate by default, so expect a browser warning.

Port `8000` is the Edge Agent tunnel. It is bound to `0.0.0.0` by default; set `PORTAINER_TUNNEL_BIND=127.0.0.1` if you don't use Edge Agents.

## Configuration

All settings are optional. Put overrides in `.env` next to the compose files; start from `.env.example`. `.env` is git-ignored, so local values stay out of the repo.

The **Default** column is what the compose files use when a variable is unset. `.env.example` sets tighter resource limits than those defaults, so copying it as-is gives you the values in the **`.env.example`** column.

| Variable                       | Default                    | `.env.example`             | Description                                             |
|--------------------------------|----------------------------|----------------------------|---------------------------------------------------------|
| `PORTAINER_TAG`                | `2.45.1-alpine`            | `2.45.1-alpine`            | `portainer/portainer-ce` image tag.                     |
| `TZ`                           | `Europe/Athens`            | `Europe/Athens`            | Container timezone.                                     |
| `PORTAINER_TUNNEL_BIND`        | `0.0.0.0`                  | `0.0.0.0`                  | Host address for the Edge Agent port `8000`.            |
| `PORTAINER_CPU_LIMIT`          | `1.0`                      | `0.5`                      | CPU limit (standard variant only).                      |
| `PORTAINER_MEMORY_LIMIT`       | `512m`                     | `128m`                     | Memory limit (standard variant only).                   |
| `PORTAINER_MEMORY_RESERVATION` | `128m`                     | `32m`                      | Memory reservation (standard variant only).             |
| `PORTAINER_PIDS_LIMIT`         | `200`                      | `200`                      | Max processes (standard variant only).                  |
| `SYNOLOGY_DOCKER_VOLUMES`      | `/volume1/@docker/volumes` | `/volume1/@docker/volumes` | Synology's Docker volumes path (Synology variant only). |

## What's included

- **Hardening:** `no-new-privileges`, `init` for signal handling and zombie reaping, UI bound to localhost.
- **Health check:** polls `/api/system/status` every 30s.
- **Log rotation:** JSON logs capped at 5 × 10 MB, compressed.
- **Persistence:** Portainer data lives in the named volume `portainer`; the network is also named `portainer`.

## Common tasks

```sh
# Upgrade: bump PORTAINER_TAG in .env (or pull the same tag), then
docker compose pull && docker compose up -d

# Logs
docker compose logs -f portainer

# Health status
docker inspect --format '{{.State.Health.Status}}' portainer

# Back up the data volume
docker run --rm -v portainer:/data -v "$PWD":/backup alpine \
  tar czf /backup/portainer-data-$(date +%F).tar.gz -C /data .
```

Add `-f docker-compose.synology.yml` to the `docker compose` commands when using the Synology variant.

## Security note

Portainer mounts `/var/run/docker.sock`, which gives it root-equivalent control over the host. Keep the UI off the public internet and use a strong admin password.
