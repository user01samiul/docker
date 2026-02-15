# Docker + Node.js Complete Reference

A Next.js 16 app (React 19, TypeScript, Tailwind CSS) used as a playground to learn Docker.

---

## Dockerfile vs Image vs Container

```
Dockerfile          →  Image              →  Container
(blueprint/recipe)     (template/snapshot)    (running instance)

- Text file with       - Built from a         - A running (or stopped)
  build instructions     Dockerfile             process created from
- YOU write this       - Read-only, layered     an image
- Lives in your        - Can be shared via    - Has its own filesystem,
  project repo           a registry             network, and processes
                       - Used to create       - You can run MANY containers
                         containers (and        from ONE image
                         other images)
```

**Analogy**: Dockerfile = recipe, Image = frozen meal made from the recipe, Container = the meal heated up and served. You cook once (build), freeze many copies (image), and serve individually (containers).

---

## Dockerfile - All Common Instructions

### Our Dockerfile (explained)

```dockerfile
FROM node:22           # Base image - every Dockerfile starts with FROM
WORKDIR /app           # Set working directory (created if it doesn't exist)
COPY package*.json ./  # Copy package.json + lock file first (layer caching trick)
RUN npm install        # Install dependencies (cached if package.json unchanged)
COPY . .               # Copy the rest of the source code
RUN npm run build      # Build the Next.js production bundle
ENV PORT=3000          # Set environment variable
EXPOSE 3000            # Document the port (does NOT actually publish it)
CMD ["npm", "start"]   # Default command when container starts
```

> **Layer caching trick**: Docker caches each instruction as a layer. By copying `package*.json` and running `npm install` BEFORE copying source code, Docker reuses the cached dependencies layer when only your code changes. This makes rebuilds much faster.

### All Common Dockerfile Instructions

| Instruction    | Purpose                                         | Example                                      |
| -------------- | ----------------------------------------------- | -------------------------------------------- |
| `FROM`         | Set the base image (required, must be first)    | `FROM node:22-alpine`                        |
| `WORKDIR`      | Set working directory for following instructions | `WORKDIR /app`                              |
| `COPY`         | Copy files from host into the image             | `COPY . .`                                   |
| `ADD`          | Like COPY but can extract .tar and fetch URLs   | `ADD archive.tar.gz /app`                    |
| `RUN`          | Execute a command during build (creates a layer) | `RUN npm install`                           |
| `CMD`          | Default command when container starts            | `CMD ["npm", "start"]`                      |
| `ENTRYPOINT`   | Like CMD but harder to override (main process)  | `ENTRYPOINT ["node", "server.js"]`           |
| `ENV`          | Set environment variables                        | `ENV NODE_ENV=production`                   |
| `ARG`          | Build-time variables (not available at runtime) | `ARG NODE_VERSION=22`                        |
| `EXPOSE`       | Document which port the app uses (metadata only) | `EXPOSE 3000`                               |
| `VOLUME`       | Create a mount point for persistent data         | `VOLUME /app/data`                          |
| `USER`         | Switch to a non-root user (security)            | `USER node`                                  |
| `LABEL`        | Add metadata to the image                        | `LABEL maintainer="sami"`                   |
| `HEALTHCHECK`  | Tell Docker how to check if the container is healthy | `HEALTHCHECK CMD curl -f http://localhost:3000/` |

### CMD vs ENTRYPOINT

```dockerfile
# CMD - easily overridden at runtime
CMD ["npm", "start"]
# docker run myapp npm run dev    ← replaces CMD entirely

# ENTRYPOINT - always runs, CMD becomes default arguments
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run myapp                ← runs: node server.js
# docker run myapp app.js         ← runs: node app.js (CMD overridden, ENTRYPOINT stays)
```

### ARG vs ENV

```dockerfile
ARG NODE_VERSION=22        # Only available during build
FROM node:${NODE_VERSION}

ENV NODE_ENV=production    # Available during build AND at runtime inside the container
```

---

## .dockerignore

Excludes files from the build context (smaller context = faster builds, smaller images).

```
node_modules
.next
.git
.gitignore
.env*
*.md
.vscode
.idea
```

> **Important**: Always ignore `node_modules`. The Dockerfile runs `npm install` fresh inside the container, so copying host `node_modules` wastes time and can cause platform-specific bugs (e.g., native binaries compiled for Windows won't work in a Linux container).

---

## Common Commands

### Build

| Command                                       | Description                                  |
| --------------------------------------------- | -------------------------------------------- |
| `docker build -t sami/demoapp:1.0 .`         | Build image with tag (name:version)          |
| `docker build -t myapp .`                     | Build with just a name (defaults to :latest) |
| `docker build --no-cache -t myapp .`          | Build without using layer cache              |
| `docker build --build-arg NODE_VERSION=20 .`  | Pass build arguments (for ARG)               |

### Images

| Command                                | Description                              |
| -------------------------------------- | ---------------------------------------- |
| `docker images`                        | List all local images                    |
| `docker image ls`                      | Same as above                            |
| `docker rmi <image>`                   | Remove an image                          |
| `docker image prune`                   | Remove dangling images (untagged)        |
| `docker image prune -a`               | Remove ALL unused images                 |
| `docker tag myapp sami/myapp:2.0`     | Tag an existing image with a new name    |
| `docker history <image>`              | Show layers/build history of an image    |
| `docker inspect <image>`              | Show detailed image metadata (JSON)      |

### Run Containers

| Command                                                  | Description                                  |
| -------------------------------------------------------- | -------------------------------------------- |
| `docker run <image>`                                     | Run a container (foreground)                 |
| `docker run -d <image>`                                  | Run in detached mode (background)            |
| `docker run -p 3000:3000 <image>`                        | Map host port to container port              |
| `docker run --name myapp <image>`                        | Give the container a name                    |
| `docker run --rm <image>`                                | Auto-remove container when it stops          |
| `docker run -e NODE_ENV=development <image>`             | Pass environment variable                    |
| `docker run --env-file .env <image>`                     | Load env vars from file                      |
| `docker run -v $(pwd)/data:/app/data <image>`            | Mount a host directory into container        |
| `docker run -v myvolume:/app/data <image>`               | Mount a named volume                         |
| `docker run --restart unless-stopped <image>`            | Auto-restart policy                          |
| `docker run -d -p 3000:3000 --name myapp --rm <image>`  | Common combo: background + port + name + cleanup |

### Manage Containers

| Command                                  | Description                                    |
| ---------------------------------------- | ---------------------------------------------- |
| `docker ps`                              | List running containers                        |
| `docker ps -a`                           | List all containers (including stopped)        |
| `docker stop <container>`                | Gracefully stop a container                    |
| `docker kill <container>`                | Force stop a container                         |
| `docker start <container>`               | Start a stopped container                      |
| `docker restart <container>`             | Restart a container                            |
| `docker rm <container>`                  | Remove a stopped container                     |
| `docker rm -f <container>`               | Force remove (even if running)                 |
| `docker logs <container>`                | View container logs                            |
| `docker logs -f <container>`             | Follow logs in real-time (like tail -f)        |
| `docker logs --tail 100 <container>`     | Show last 100 lines of logs                    |
| `docker exec -it <container> sh`         | Open a shell inside a running container        |
| `docker exec -it <container> node`       | Open a Node.js REPL inside the container       |
| `docker cp <container>:/app/file.txt .`  | Copy a file from container to host             |
| `docker inspect <container>`             | Show detailed container metadata               |
| `docker stats`                           | Live resource usage (CPU, memory, network)     |

### Registry (Docker Hub)

| Command                               | Description                              |
| -------------------------------------- | ---------------------------------------- |
| `docker login`                        | Log in to Docker Hub                     |
| `docker push sami/myapp:1.0`         | Push image to Docker Hub                 |
| `docker pull sami/myapp:1.0`         | Pull image from registry                 |
| `docker pull node:22-alpine`          | Pull a public image                      |

### Cleanup

| Command                                | Description                                    |
| -------------------------------------- | ---------------------------------------------- |
| `docker container prune`               | Remove all stopped containers                  |
| `docker image prune -a`               | Remove all unused images                       |
| `docker volume prune`                  | Remove all unused volumes                      |
| `docker system prune`                  | Remove all unused data (containers+images+networks) |
| `docker system prune -a --volumes`    | Nuclear cleanup (everything unused)            |
| `docker system df`                     | Show Docker disk usage                         |

---

## Volumes (Persistent Data)

Containers are ephemeral - when removed, their data is gone. Volumes persist data.

```bash
# Named volume (Docker manages the location)
docker volume create mydata
docker run -v mydata:/app/data <image>

# Bind mount (you choose the host path)
docker run -v $(pwd)/data:/app/data <image>

# List volumes
docker volume ls

# Remove a volume
docker volume rm mydata
```

---

## Docker Compose (Multi-Container Apps)

For apps with multiple services (e.g., Node.js + database). Create a `docker-compose.yml`:

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  pgdata:
```

| Command                        | Description                                  |
| ------------------------------ | -------------------------------------------- |
| `docker compose up`            | Start all services (foreground)               |
| `docker compose up -d`         | Start all services (background)               |
| `docker compose up --build`    | Rebuild images and start                      |
| `docker compose down`          | Stop and remove all containers                |
| `docker compose down -v`       | Stop, remove containers AND volumes           |
| `docker compose logs -f`       | Follow logs for all services                  |
| `docker compose ps`            | List running services                         |
| `docker compose exec app sh`   | Shell into a specific service                 |

---

## Networking

Containers can talk to each other by service name in Docker Compose. For standalone containers:

```bash
# Create a network
docker network create mynet

# Run containers on the same network
docker run -d --name api --network mynet myapi
docker run -d --name web --network mynet myweb

# Now "web" can reach "api" using http://api:3000
```

---

## Node.js Base Image Tags

| Tag                  | Description                              | Size     | Use Case                    |
| -------------------- | ---------------------------------------- | -------- | --------------------------- |
| `node:22`            | Full Debian-based image                  | ~1 GB    | Development, full tooling   |
| `node:22-slim`       | Debian with minimal packages             | ~200 MB  | Smaller production image    |
| `node:22-alpine`     | Alpine Linux (minimal)                   | ~130 MB  | Smallest production image   |

> **Tip**: Use `node:22-alpine` for smaller images in production. Use `node:22` if you need build tools like `gcc` or `python` for native modules.

---

## Multi-Stage Build (Production Optimization)

Use multiple `FROM` statements to keep the final image small:

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2: Production (only what's needed to run)
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
EXPOSE 3000
USER node
CMD ["npm", "start"]
```

> This copies only the build output into a clean image, leaving behind devDependencies, source code, and build tools. The result is a much smaller and more secure image.

---

## Port Forwarding Explained

```
docker run -p 3000:3000 <image>
              ↑     ↑
         host:container

- Host port (left):      the port on YOUR machine (browser)
- Container port (right): the port the app listens on INSIDE the container
```

`docker run -p 8080:3000` = open `http://localhost:8080`, Docker forwards to port 3000 inside the container.

---

## Workflow

```
1. Write Dockerfile    →  the blueprint
2. Build image         →  docker build -t sami/demoapp:1.0 .
3. Test locally        →  docker run -p 3000:3000 sami/demoapp:1.0
4. Open browser        →  http://localhost:3000
5. Push to registry    →  docker push sami/demoapp:1.0
6. Deploy anywhere     →  docker pull sami/demoapp:1.0 && docker run ...
```

---

## Quick Tips

- **EXPOSE doesn't publish ports** - it's just documentation. You still need `-p` at runtime.
- **Use `.dockerignore`** - without it, `node_modules` (hundreds of MBs) gets sent to Docker on every build.
- **Use `--rm` during development** - auto-removes the container when you stop it, avoids clutter.
- **Use `docker compose` for multi-container setups** - don't manually link containers.
- **Tag images with versions** - don't rely on `:latest`, it makes rollbacks hard.
- **Run as non-root** - add `USER node` in your Dockerfile for security (Node images include a `node` user).
- **Use Alpine images in production** - `node:22-alpine` is ~130 MB vs ~1 GB for `node:22`.
- **Don't store data in containers** - use volumes for anything that needs to persist (databases, uploads).
- **One process per container** - run your app in one container, your database in another.
