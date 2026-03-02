# 🐳 Docker Cheatsheet

## Basic Commands

| Command                               | Description                             |
| ------------------------------------- | --------------------------------------- |
| `docker ps`                           | List running containers                 |
| `docker ps -a`                        | List all containers (running + stopped) |
| `docker images`                       | List all local images                   |
| `docker rename <old_name> <new_name>` | Rename a container                      |

---

## Running Containers

**First-time run (pulls & starts image in a new container):**

```bash
docker run -d --name <container_name> -p <host_port>:<container_port> <image_name>:<tag>
```

**Restart an existing (stopped) container:**

```bash
docker start <container_name>
```

**Stop a running container:**

```bash
docker stop <container_name>
```

**Run and auto-remove container when it stops:**

```bash
docker run --rm <image_name>
```

---

## Entering a Container's Terminal

```bash
# For most images (bash)
docker exec -it <container_id_or_name> bash

# For Alpine-based images (e.g., node:alpine)
docker exec -it <container_id_or_name> sh
```

> **Note:** Inside the container, use the relevant CLI tool — `redis-cli`, `mysql`, `mongosh`, etc.

---

## Common Flags

| Flag                    | Description                            |
| ----------------------- | -------------------------------------- |
| `-d`                    | Detached mode (runs in background)     |
| `-it`                   | Interactive mode (attach terminal)     |
| `-p <host>:<container>` | Port mapping                           |
| `-e KEY=VALUE`          | Set environment variable               |
| `--name`                | Name the container                     |
| `--rm`                  | Auto-remove container on stop          |
| `--env-file .env`       | Load environment variables from a file |
| `--network=<name>`      | Set container network                  |

---

## Port Mapping (`-p`)

Maps `host_port:container_port`:

```bash
docker run -d -p 8000:8000 my-image          # same port
docker run -d -p 2026:8000 my-image          # different ports
docker run -it -p 8080:8080 -p 3001:3000 my-node-server bash  # multiple ports
```

---

## Environment Variables

```bash
# Inline
docker run -d -p 8000:8000 -e API_KEY=abc123 my-image

# From .env file
docker run --env-file .env my-image
```

---

## Images

```bash
# Build a custom image (. = Dockerfile in current directory)
docker build -t <image_name> .

# Remove an image
docker rmi <image_name>:<tag>

# Remove a container
docker rm <container_name>
```

> Docker uses **cached layers** — unchanged lines in the Dockerfile are not rebuilt. Changes trigger a rebuild from that line onward.

---

## Dockerfile Reference

```dockerfile
FROM ubuntu                          # Base image (always first)
RUN apt-get update && apt-get install -y curl   # Run shell commands
COPY ./src /app/src                  # Copy files: source → destination
WORKDIR /app                         # Set working directory
ENTRYPOINT ["node", "index.js"]      # Command to run on container start
```

**Build & run example:**

```bash
docker build -t my-backend .
docker run -d -p 3001:3001 --env-file .env my-backend
```

---

## Docker Compose

Defines and runs multi-container apps from a single `docker-compose.yml` file.

```bash
docker compose up          # Start all services
docker compose up -d       # Start all services in detached mode
docker compose down        # Stop and remove all services
```

---

## Networking

```bash
docker network ls                        # List all networks
docker network inspect bridge            # Inspect bridge network
docker network inspect host              # Inspect host network
docker network inspect none              # Inspect none network
```

**Run with a specific network:**

```bash
docker run -it --network=host --name my-container ubuntu
docker run -it --network=none --name my-container ubuntu   # No internet access
```

**Bridge vs Host:**

- **Bridge** (default): Requires explicit port mapping (`-p`)
- **Host**: Container shares the host's network directly — no port mapping needed

**Create a custom network (so containers can talk to each other):**

```bash
docker network create -d bridge my-network

docker run --name=container1 -it --network=my-network ubuntu bash
docker run --name=container2 -it --network=my-network ubuntu bash

docker network inspect my-network
```

---

## Volumes (Persistent Storage)

Without volumes, data is lost when a container is removed.

### Method 1 — Bind Mount (host folder directly)

```bash
docker run -it --name my-container -v <host_path>:<container_path> ubuntu bash
```

### Method 2 — Named Volume

```bash
docker volume create my-volume
docker volume ls
docker volume inspect my-volume
docker volume rm my-volume

# Use volume in a container
docker run -it --name my-container --mount type=volume,src=my-volume,dst=/data ubuntu bash
```

---

## Redis Stack Example

```bash
# Install & run
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest

# Restart later
docker start redis-stack

# Enter Redis CLI
docker exec -it redis-stack bash
# then inside: redis-cli
```

## Build & RUN Production Grade docker image of express app

```
1. docker build -t <name>:<version> --platform linux/arm64 .
2. go to /backend folder make sure there should a .env file presents & env variables must not contains the whitespaces
2. docker run --rm -p  1001:8001 --env-file .env mern-express:v1
```

### Remarks

1. Use --platform linux/arm64 only when, host machine is linux (vps) not windows(local pc)

---

## Tips

- `CTRL + L` — Clear the terminal
- Use `sh` instead of `bash` for Alpine-based images (e.g., `node:alpine`)
