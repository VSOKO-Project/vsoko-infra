# VSOKO / Infra

> Part of the **VSOKO** education quality assessment system.
> 📖 [Full project description and architecture →](https://github.com/VSOKO-Project)

Reverse proxy for the VSOKO platform, deployed separately from the
application services (`vsoko-backend`, `vsoko-frontend`) and shared with
them through a common external Docker network.

## Components

| Component | What it does |
| --- | --- |
| `proxy/` | Traefik — TLS termination (Let's Encrypt, TLS challenge), routes to services on `web_network` based on their Docker labels |

## Running

```bash
docker network create web_network   # once, if the network doesn't exist yet

cd proxy
docker compose -f docker-compose.proxy.yml up -d
```

After that, `vsoko-backend` and `vsoko-frontend` come up via their own
`docker-compose.yml` files, joining the same `web_network` (Traefik
discovers them by their `traefik.*` labels).

## CI/CD (GitLab)

`.gitlab-ci.yml` — deploys over SSH: pushes `proxy/` to the target host at
`$BASE_DEPLOY_PATH/infrastructure`, pulls and restarts the Traefik
container via `docker compose`.
