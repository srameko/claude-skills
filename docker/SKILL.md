---
name: docker
description: >
  Docker Swarm stack management for docker.home.ondrejsramek.cz. Use this skill
  whenever the user works with Docker Compose stacks, adds or updates services,
  manages secrets, updates image versions, works with Traefik, Graylog, or any
  service in the home stack. Also trigger for Docker networking, volumes, logging,
  or CI lint/scan workflows. Always use this skill for anything touching the
  docker repo.
---

# Docker Skill

## Repository
`https://github.com/srameko/docker.home.ondrejsramek.cz`

---

## Structure

```
stack.env                    ← image version pins (single source of truth for upgrades)
stacks/
  home/dckr-cmps.yml         ← main stack: Traefik, AdGuard, certs-dumper, Homarr
  logging/dckr-cmps.yml      ← logging stack: Graylog, OpenSearch, MongoDB
.github/
  workflows/lint.yml         ← validate compose + Trivy secret/misconfig scan
  dependabot.yml             ← weekly GitHub Actions updates (grouped)
```

---

## Stacks Overview

### home stack
| Service | Image | Purpose |
|---|---|---|
| traefik | `dhi.io/traefik` | Reverse proxy, TLS via ACME (Active24 DNS challenge) |
| adguardhome | `adguard/adguardhome` | DNS server |
| certs-dumper | `ldez/traefik-certs-dumper` | Dumps Traefik certs to files for other services |
| homarr | `ghcr.io/homarr-labs/homarr` | Dashboard |

### logging stack
| Service | Image | Purpose |
|---|---|---|
| graylog | `graylog/graylog` | Log aggregation (GELF UDP 12201) |
| opensearch | `opensearchproject/opensearch` | Search backend for Graylog |
| mongo | `mongo` | Graylog metadata store |

---

## Key Patterns

### Image Version Management
All versions are pinned in `stack.env` — **never hardcode versions in compose files**:
```env
TRAEFIK=3-dev
ADGUARD=latest
GRAYLOG=7.0.4
```
To upgrade a service: update `stack.env` → redeploy stack.

### Secrets
Secrets are Docker Swarm external secrets — **never put secrets in compose files or stack.env**:
```yaml
secrets:
  MY_SECRET:
    external: true
```
Access in container via `_FILE` convention:
```yaml
environment:
  - MY_VAR_FILE=/run/secrets/MY_SECRET
```

### Networking
- `proxy` network: external overlay, shared between stacks for Traefik routing
- `graylog_net`: internal overlay, logging stack only
- Services not needing external access should NOT be on `proxy`

### Logging
All services log to Graylog via GELF:
```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://192.168.1.4:12201"
    tag: <service-name>
```

### Traefik Labels Pattern
```yaml
deploy:
  labels:
    - traefik.enable=true
    - traefik.http.routers.<n>.rule=Host(`<subdomain>.home.sramkovi.family`)
    - traefik.http.routers.<n>.entrypoints=websecure
    - traefik.http.routers.<n>.tls=true
    - traefik.http.routers.<n>.tls.certresolver=le
    - traefik.http.services.<n>.loadbalancer.server.port=<port>
    - traefik.swarm.network=proxy
```

### Volumes
Volumes use local bind mounts to `/volume1/docker/<service>/`:
```yaml
volumes:
  my_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /volume1/docker/<service>/data
```

---

## Adding a New Service

1. Add image version to `stack.env`
2. Add service to appropriate stack compose file
3. Use external secret if credentials needed
4. Add to `proxy` network + Traefik labels if web-accessible
5. Add GELF logging
6. Use bind mount volume under `/volume1/docker/<service>/`
7. Test locally: `docker compose -f stacks/<stack>/dckr-cmps.yml config --quiet --no-interpolate`

---

## CI Workflow

**lint.yml** runs on every push/PR:
1. `docker compose config --quiet --no-interpolate` — validates both stack files
2. Trivy scan — secrets + misconfigs, CRITICAL/HIGH, exit-code 1

Actions are **SHA-pinned** — maintain when updating.

**Dependabot** updates GitHub Actions weekly (grouped as `actions`).

---

## Common Operations

```bash
# Validate compose locally
docker compose -f stacks/home/dckr-cmps.yml config --quiet --no-interpolate

# Deploy/update stack (Docker Swarm)
docker stack deploy -c stacks/home/dckr-cmps.yml home
docker stack deploy -c stacks/logging/dckr-cmps.yml logging

# Check stack status
docker stack services home
docker stack services logging

# View service logs
docker service logs home_traefik -f
```
