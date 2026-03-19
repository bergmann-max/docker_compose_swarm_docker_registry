# Docker Registry with Docker Swarm

![Registry](https://img.shields.io/badge/registry-3-blue)
![Platform](https://img.shields.io/badge/platform-Docker%20Swarm-blue)
![Storage](https://img.shields.io/badge/storage-NFS-orange)

This repository describes a private Docker registry stack based on `registry:3`.
The deployment is designed for Docker Swarm and uses:

- Basic Auth via Docker Secret
- Traefik as a reverse proxy
- an external NFS volume for persistent images

## Prerequisites

Before deployment, the following dependencies must be available:

- Docker Engine with Swarm enabled
- an external Docker network named `traefik`
- Traefik with TLS termination in the same Swarm
- a reachable NFS server for the registry volume

## Project Structure

```text
.
├── auth/htpasswd
├── docker-compose.yml
└── README.md
```

## Configuration

The main configuration is located in [`docker-compose.yml`].
Before using this setup in production, you should at least adjust the following values:

- `REGISTRY_HTTP_HOST`: public URL of the registry
- `traefik.http.routers.registry.rule`: registry hostname
- `volumes.registry-data.driver_opts.o`: address and options of the NFS server
- `volumes.registry-data.driver_opts.device`: export path on the NFS server

The file [`auth/htpasswd`] contains the credentials for Basic Auth and is mounted as a Docker Secret.

## Authentication

The `auth/htpasswd` file must use the format `user:hash`.
You can generate an entry with `htpasswd`:

```sh
htpasswd -nb admin password
```

Replace `admin` and `password` with the actual credentials.
If multiple users are required, extend the file accordingly.

The output will look roughly like this:

```text
admin:$2y$...
```

## Deployment

### 1. Create the Docker Secret

If the secret does not exist yet:

```sh
docker secret create registry_htpasswd ./auth/htpasswd
```

If the secret already exists and the credentials were changed, remove it first and then recreate it:

```sh
docker secret rm registry_htpasswd
docker secret create registry_htpasswd ./auth/htpasswd
```

### 2. Deploy the Stack

```sh
docker stack deploy -c docker-compose.yml registry
```

### 3. Check the Rollout

```sh
docker service ps registry_registry --no-trunc
```

Optional:

```sh
docker service logs -f registry_registry
```

## Usage

Log in to the registry:

```sh
docker login registry.example.com
```

Push an image:

```sh
docker tag alpine:latest registry.example.com/alpine:latest
docker push registry.example.com/alpine:latest
```

## Notes

- This repository is intended for `docker stack deploy`, not `docker compose up`.
- The `registry_htpasswd` secret is defined as `external: true` and is not created automatically.
- The `traefik` network is also external and must already exist.
