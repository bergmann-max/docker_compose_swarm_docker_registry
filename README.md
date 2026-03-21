# Docker Registry with Docker Swarm

![Registry](https://img.shields.io/badge/registry-3-blue)
![Platform](https://img.shields.io/badge/platform-Docker%20Swarm-blue)
![Storage](https://img.shields.io/badge/storage-NFS-orange)

This repository describes a private Docker registry stack based on `registry:3`.
The deployment is designed for Docker Swarm and uses:

- Basic Auth via Docker Secret
- Traefik as a reverse proxy
- An external NFS volume for persistent images
- A pull-through cache for Docker Hub (`registry-proxy`)

## Prerequisites

Before deployment, the following dependencies must be available:

- Docker Engine with Swarm enabled
- An external Docker network named `traefik`
- Traefik with TLS termination in the same Swarm
- A reachable NFS server for the registry volumes

## Project Structure

```text
.
├── docker-compose.yml
└── README.md
```

## Services

### `registry`

A private Docker registry for storing and distributing your own images.

### `registry-proxy`

A pull-through cache that proxies requests to Docker Hub (`registry-1.docker.io`).
Images are fetched from Docker Hub on first pull and cached locally on NFS.
Subsequent pulls are served directly from the cache.

## Configuration

The main configuration is located in [`docker-compose.yml`].
Before using this setup in production, adjust the following values:

| Variable / Label | Service | Description |
|---|---|---|
| `REGISTRY_HTTP_HOST` | both | Public URL of the registry |
| `traefik.http.routers.*.rule` | both | Hostname for Traefik routing |
| `volumes.*.driver_opts.o` | both | NFS server address and options |
| `volumes.*.driver_opts.device` | both | Export path on the NFS server |
| `REGISTRY_PROXY_USERNAME` | registry-proxy | Docker Hub username (optional) |
| `REGISTRY_PROXY_PASSWORD` | registry-proxy | Docker Hub access token (optional) |

## Authentication

Credentials are stored as Docker Secrets in `user:hash` format.
No credentials are stored in the repository.

`htpasswd` from the `apache2-utils` package is required:

```sh
apt-get install -y apache2-utils
```

> **Note:** `registry:3` requires bcrypt password hashes (`-B` flag). MD5-based hashes are not supported and will cause 401 Unauthorized errors.

### Create Secrets

**Private registry:**
```sh
htpasswd -nbB admin 'yourpassword' | docker secret create registry_htpasswd -
```

**Pull-through proxy:**
```sh
htpasswd -nbB proxyuser 'yourpassword' | docker secret create registry_proxy_htpasswd -
```

### Recreate a Secret

If credentials change, remove the stack first, then recreate the secret:

```sh
docker stack rm registry
docker secret rm registry_htpasswd
htpasswd -nbB admin 'yourpassword' | docker secret create registry_htpasswd -
docker stack deploy -c docker-compose.yml registry
```

## Deployment

### 1. Create required secrets (once)

```sh
htpasswd -nbB admin 'yourpassword' | docker secret create registry_htpasswd -
htpasswd -nbB proxyuser 'yourpassword' | docker secret create registry_proxy_htpasswd -
```

### 2. Deploy the stack

```sh
DOCKERHUB_USERNAME=myuser \
DOCKERHUB_TOKEN=mytoken \
docker stack deploy -c docker-compose.yml registry
```

`DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` are optional but recommended to avoid
Docker Hub rate limits (100 pulls/6h anonymous, 200/6h free account).

### 3. Check the rollout

```sh
docker stack services registry
docker stack ps registry
```

## Usage

### Private registry

```sh
docker login registry.example.com
docker tag alpine:latest registry.example.com/alpine:latest
docker push registry.example.com/alpine:latest
docker pull registry.example.com/alpine:latest
```

### Pull-through cache (Docker Hub mirror)

Login once:
```sh
docker login registry-proxy.example.com
```

Pull images through the cache:
```sh
# official images require the "library/" prefix
docker pull registry-proxy.example.com/library/nginx:latest

# non-official images
docker pull registry-proxy.example.com/bitnami/postgresql:15
```

Or configure it as a daemon-level mirror so plain `docker pull nginx` is transparently cached.
Edit `/etc/docker/daemon.json` on each Docker host:

```json
{
  "registry-mirrors": ["https://registry-proxy.example.com"]
}
```

Then restart Docker:
```sh
sudo systemctl restart docker
```

## CI/CD (Gitea Actions)

The workflow in `.gitea/workflows/deploy.yml` handles automated deployments.
The following secrets must be configured in Gitea:

| Secret | Required | Description |
|---|---|---|
| `SWARM_SSH_KEY` | yes | SSH private key for the Swarm manager |
| `SWARM_MANAGER_HOST` | yes | Hostname or IP of the Swarm manager |
| `SWARM_MANAGER_USER` | yes | SSH user on the Swarm manager |
| `REGISTRY_BASIC_AUTH_USER` | yes | Username for the private registry |
| `REGISTRY_BASIC_AUTH_PASSWORD` | yes | Password for the private registry |
| `REGISTRY_PROXY_BASIC_AUTH_USER` | yes | Username for the proxy registry |
| `REGISTRY_PROXY_BASIC_AUTH_PASSWORD` | yes | Password for the proxy registry |
| `DOCKERHUB_USERNAME` | no | Docker Hub username (avoids rate limits) |
| `DOCKERHUB_TOKEN` | no | Docker Hub access token |

## Notes

- This repository is intended for `docker stack deploy`, not `docker compose up`.
- Both secrets (`registry_htpasswd`, `registry_proxy_htpasswd`) are `external: true` and must be created manually before the first deploy.
- The `traefik` network is external and must already exist.
