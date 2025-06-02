# Docker Architecture, Components, and Versioning

Docker is an open-source platform designed to automate the deployment of applications inside lightweight, portable 
containers. It enables developers to package an application with all of its dependencies into a standardized unit, 
ensuring consistency across development, testing, and production environments. This article provides an in-depth look at
what Docker is, how it differs from virtual machines (VMs), and elaborates on Docker's architecture, components, and 
versioning system.

## What is Docker?

Docker is a containerization platform that allows applications to run in isolated environments called **containers**. 
A container includes everything needed to run the software: code, runtime, libraries, environment variables, and 
configuration files. Containers are built from **images**, which are templates that define the contents and configuration 
of the container.

Docker containers are fast to start, efficient in resource usage, and portable across different computing environments. 
This makes Docker especially useful in DevOps pipelines, microservices architectures, and cloud-native development.

## Docker vs Virtual Machines (VMs)

While both Docker containers and virtual machines offer isolated environments for running applications, they differ 
fundamentally in their architecture and performance.

| Feature                | Docker Containers                      | Virtual Machines                        |
|------------------------|----------------------------------------|------------------------------------------|
| **Isolation**          | Process-level (shares OS kernel)       | Full machine-level (separate OS per VM)  |
| **Performance**        | Lightweight, fast startup              | Heavier, slower startup and shutdown     |
| **Resource Usage**     | Low (no separate OS needed)            | High (each VM includes full OS)          |
| **Portability**        | High (runs on any host with Docker)    | Moderate (depends on hypervisor)         |
| **Boot Time**          | Seconds                                | Minutes                                  |

In essence, VMs emulate entire hardware stacks and run their own operating systems, which can consume significant memory 
and CPU resources. Docker containers, on the other hand, run on top of the host operating system and share its kernel, 
making them more lightweight and efficient.

## Docker Architecture

Docker follows a **client-server architecture**:

- **Docker Client**: The user interface that sends commands to the Docker daemon using the Docker API. It's typically the `docker` CLI tool.
- **Docker Daemon (`dockerd`)**: Runs in the background and is responsible for building, running, and managing Docker containers.
- **Docker Objects**: Include images, containers, volumes, and networks.
- **Docker Registry**: A repository for storing Docker images, such as Docker Hub or a private registry.

The client and daemon can run on the same system, or the client can connect to a daemon on a remote host, allowing for flexible deployment models.

## Core Components of Docker

### 1. Docker Engine

The Docker Engine is the core runtime that provides containerization capabilities. It includes the daemon, the REST API, and the CLI.

### 2. Docker Images

An image is a read-only template used to create containers. It includes the application and its environment. Images are 
versioned and can be shared using Docker registries.

### 3. Docker Containers

Containers are running instances of images. They are isolated, executable packages that include everything needed to run a 
piece of software. Containers can be created, started, stopped, moved, and deleted.

### 4. Dockerfile

A `Dockerfile` is a script with instructions to build a Docker image. It defines the base image, application code, environment 
variables, and how the container should behave at runtime.

### 5. Docker Compose

Docker Compose is a tool that allows users to define and manage multi-container applications. It uses a `docker-compose.yml` 
file to configure application services, making it easier to orchestrate multiple containers as part of a single application.

### 6. Docker Registries

Registries like Docker Hub allow users to store and distribute images. They support version tagging, which ensures 
consistent deployment across environments.

## Docker Versioning

Docker adheres to **semantic versioning** (`MAJOR.MINOR.PATCH`) to track changes and maintain compatibility:

- **MAJOR**: Significant changes or backward-incompatible updates.
- **MINOR**: New features added in a backward-compatible way.
- **PATCH**: Bug fixes or small improvements.

For example, `24.0.2` would be a patch update to the 24.0 release. Docker also maintains different release channels 
like `stable`, `test`, and `nightly`, which help teams choose the right balance between stability and innovation.

Additionally, Docker images are tagged with versions (e.g., `nginx:1.25`, `node:18-alpine`) to ensure that applications 
use consistent dependencies during deployment.

Docker offers a powerful, efficient, and portable solution for application deployment by leveraging containers. Unlike 
traditional virtual machines, Docker provides lightweight, OS-level virtualization that reduces overhead and improves speed. 
With a well-structured architecture, modular components like Docker Engine, Compose, and registries, and a clear versioning 
strategy, Docker has become a fundamental tool in modern software development and DevOps workflows.

## Working with Docker Images

Docker images are read-only templates that define what goes on in a container. Before running a container, you typically 
need to have an image locally (either pulled from a registry or built manually).

### Pulling an Image

To download an existing image from a registry (e.g., Docker Hub):

```bash
docker pull nginx:latest
# └─ Pulls the official `nginx` image with the "latest" tag from Docker Hub
```

* `docker pull`: Downloads the image layers that you don’t already have locally.
* `nginx:latest`: Specifies the repository (`nginx`) and tag (`latest`). If you omit the tag, Docker defaults to `:latest`.

### Listing Local Images

To view all images you have downloaded or built on your local machine:

```bash
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
# nginx        latest    a66f...        2 days ago      133MB
# ubuntu       20.04     f643...        3 weeks ago     72.9MB
```

* `docker images`: Shows a table of locally available images, including repository, tag, image ID, creation time, and size.

### Removing an Image

If you no longer need an image, remove it to free up space:

```bash
docker rmi nginx:latest
# └─ Deletes the `nginx:latest` image (only if no containers depend on it)
```

* `docker rmi`: Removes one or more images. If a container is using that image (even stopped), you’ll get an error unless you add `--force`.

```bash
docker rmi -f nginx:latest
# └─ Forces removal of the image, even if containers exist based on it
```

> **Caution:** Forcing image removal can break containers that rely on that image.

---

## Working with Docker Containers

A container is a running (or stopped) instance of an image. Containers are isolated processes with their own filesystem, network, and namespaces.

### Running a Container

The most common way to start a container:

```bash
docker run nginx:latest
# └─ Runs `nginx:latest` in the foreground, printing logs to your terminal
```

Since most images run a server or process that binds to a port, it’s typical to run containers in the background (detached mode) and map ports:

```bash
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  nginx:latest
# -d           : Run container in detached/background mode
# --name       : Assigns a custom name (`my-nginx`) to the container
# -p 8080:80   : Maps host port 8080 to container port 80
# nginx:latest : Image to run (if not present, Docker will pull it first)
```

* After this command, you can open `http://localhost:8080` in your browser and see the default Nginx welcome page.
* Always choose a descriptive name for long-running or production containers so you can refer to them easily.

### Listing Containers

To see running containers:

```bash
docker ps
# CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                  NAMES
# c3d2a7b8e0f1   nginx:latest  "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:8080->80/tcp   my-nginx
```

* `docker ps`: Shows only containers in the `Up` state by default.

To see all containers (including stopped/exited):

```bash
docker ps -a
# c3d2a7b8e0f1   nginx:latest   ...   Up 5 minutes   0.0.0.0:8080->80/tcp   my-nginx
# a1b2c3d4e5f6   ubuntu:20.04   "bash"   2 days ago   Exited (0) 1 day ago   hopeful_banach
```

* `-a` (or `--all`): Lists all containers (running, stopped, exited).

### Stopping and Starting Containers

To gracefully stop a running container (sends SIGTERM, then SIGKILL after timeout):

```bash
docker stop my-nginx
# └─ Stops the `my-nginx` container
```

* Docker’s default stop timeout is 10 seconds before it forcefully kills the process. You can adjust it with `-t <seconds>`.

```bash
docker stop -t 30 my-nginx
# └─ Gives NGINX 30 seconds to shut down gracefully before force-killing
```

To start a stopped container again:

```bash
docker start my-nginx
# └─ Starts the container named `my-nginx` in the background, reusing previous settings
```

* Note that ports, volumes, and other settings remain as they were when you first ran it.

### Removing a Container

Once you’ve finished with a container (and it’s stopped), remove it to free up resources:

```bash
docker rm my-nginx
# └─ Deletes the container named `my-nginx`
```

* If the container is still running, you will need to stop it first or use `-f` to force removal:

```bash
docker rm -f my-nginx
# └─ Stops (if needed) and forcefully removes `my-nginx`
```

### Inspecting Container Details

Docker allows you to inspect almost any detail about a container:

```bash
docker inspect my-nginx
# └─ Dumps a JSON document with detailed info: network settings, mounts, environment variables, etc.
```

You can filter that output with `--format` (Go templates) to extract specific fields. For example, to get the container’s IP address:

```bash
docker inspect \
  --format='{{ .NetworkSettings.IPAddress }}' \
  my-nginx
# └─ Shows something like: 172.17.0.2
```

### Viewing Logs

If your container writes logs to stdout/stderr, you can view them with:

```bash
docker logs my-nginx
# └─ Prints logs from container’s STDOUT/STDERR
```

To “follow” logs (like `tail -f`):

```bash
docker logs -f my-nginx
# └─ Streams new logs to your terminal
```

* `-f` (or `--follow`) keeps the connection open and shows new lines as they appear.
* `--tail <n>` shows only the last `n` lines:

  ```bash
  docker logs --tail 50 my-nginx
  # └─ Shows only the last 50 lines
  ```

### Executing Commands Inside a Running Container

Sometimes you need an interactive shell inside a container to troubleshoot or inspect files:

```bash
docker exec -it my-nginx /bin/bash
# -i    : Keep STDIN open
# -t    : Allocate a pseudo-TTY
# my-nginx : Name (or ID) of the container
# /bin/bash: Command to run
```

Once inside, you can run any commands the container’s image provides. When done, type `exit` to leave the shell.

If your container doesn’t have Bash (e.g., a minimal Alpine-based image), you might need:

```bash
docker exec -it alpine-container /bin/sh
# └─ Uses `/bin/sh` (common on Alpine or BusyBox-based images)
```

---

## Managing Data in Docker

By default, changes inside a container’s filesystem do not persist beyond that container’s lifecycle. To share or persist
data, use **volumes** or **bind mounts**.

### Volumes

Volumes are fully managed by Docker. They live on the host filesystem (in Docker’s volume directory) but are decoupled
from individual containers. Even if a container is removed, the volume can remain.

1. **Create a Named Volume**

   ```bash
   docker volume create my_data_vol
   # └─ Creates a volume named `my_data_vol`
   ```

2. **Run a Container with the Volume**

   ```bash
   docker run -d \
     --name redis-server \
     -v my_data_vol:/data \
     redis:latest
   # -v my_data_vol:/data : Mounts the named volume to `/data` inside container
   # In Redis’s case, `/data` is where it persists the RDB file
   ```

3. **Inspect Volumes**

   ```bash
   docker volume ls
   # DRIVER    VOLUME NAME
   # local     my_data_vol

   docker volume inspect my_data_vol
   # Shows mountpoint, driver, labels, etc.
   ```

4. **Remove a Volume**

   ```bash
   docker volume rm my_data_vol
   # └─ Deletes the volume (only if no containers are using it)
   ```

   To force removal (use with caution):

   ```bash
   docker volume rm -f my_data_vol
   # └─ Force-unmounts and deletes the volume (may lead to data loss)
   ```

### Bind Mounts

Bind mounts map an arbitrary location on your host filesystem into the container. Useful for local development when you
want to reflect host changes immediately inside a container.

1. **Run a Container with a Bind Mount**

   ```bash
   docker run -d \
     --name web-server \
     -p 8000:80 \
     -v /home/user/website:/usr/share/nginx/html:ro \
     nginx:latest
   # -v /home/user/website:/usr/share/nginx/html:ro: Binds host directory to `/usr/share/nginx/html`.
   #                                                         The `:ro` suffix makes it read-only inside container.
   ```

    * Host path `/home/user/website`: directory on your machine that contains HTML/CSS/JS.
    * Container path `/usr/share/nginx/html`: default document root for NGINX.
    * `:ro` ensures container cannot modify host files. Omit `:ro` (i.e., `:rw`) for read-write.

### Backing Up and Restoring Volumes

Because named volumes live under Docker’s control, you can back them up by temporarily mounting them to a helper container:

1. **Backup Volume to a Tarball**

   ```bash
   # Use an `alpine` container to tar up the contents of the volume
   docker run --rm \
     -v my_data_vol:/data \
     -v $(pwd):/backup \
     alpine \
     tar czf /backup/my_data_vol_backup.tar.gz -C /data .
   # --rm           : Remove the helper container after it finishes
   # -v my_data_vol  : Mount named volume at `/data` inside helper
   # -v $(pwd):/backup: Mount current host directory to `/backup`
   # alpine tar czf : Creates a compressed archive of `/data` into `/backup`
   ```

2. **Restore Volume from Tarball**

   ```bash
   docker run --rm \
     -v my_data_vol:/data \
     -v $(pwd):/backup \
     alpine \
     sh -c "cd /data && tar xzf /backup/my_data_vol_backup.tar.gz"
   # Extracts the backup contents back into `/data` (i.e., the volume)
   ```

---

## Networking in Docker

By default, Docker creates three networks on installation:

* **bridge**: The default network for standalone containers.
* **host**: Containers share the host’s network and all its interfaces.
* **none**: Containers have no network namespaces (fully isolated).

### Built-in Network Types

1. **bridge (default)**

    * Containers attached to the default `bridge` network can communicate via IP, but not by container name unless you
    * explicitly configure DNS.

2. **host**

    * Use `--network host` to allow the container to share the host’s network stack. No port mappings are needed (host’s ports are used directly).
    * **Linux only**—on macOS/Windows, it behaves like `bridge`.

3. **none**

    * The container has no network. Use for specialized security isolation.

### Creating a User-Defined Network

User-defined networks allow automatic DNS resolution by container name and more granular control:

```bash
docker network create app-net
# └─ Creates a new bridge network named `app-net`
```

Verify creation:

```bash
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# 5c7e8c1c760a   bridge    bridge    local
# 90f2b7f8f9c3   app-net   bridge    local
# ecbf8a7810e4   host      host      local
# 1b6a4b8e36f2   none      null      local
```

### Connecting Containers to Networks

When you run a container, specify the network:

```bash
docker run -d \
  --name db-server \
  --network app-net \
  postgres:13
# └─ Runs PostgreSQL attached to `app-net`
```

```bash
docker run -d \
  --name web-app \
  --network app-net \
  -p 5000:5000 \
  my-flask-image:latest
# └─ Runs your custom Flask app in the same `app-net`, mapping host port 5000 to container’s 5000
```

Within this network:

* **DNS resolution**: `web-app` can resolve `db-server` by simply using the hostname `db-server`.
* Example (inside `web-app` container):

  ```bash
  # e.g., a Python Flask app could connect to the DB at host `db-server`, port 5432
  import psycopg2
  conn = psycopg2.connect(
      host="db-server",
      port=5432,
      user="postgres",
      password="yourpassword",
      database="postgres"
  )
  ```

### Inspecting Network Details

To see which containers are connected to a network and the assigned IPs:

```bash
docker network inspect app-net
# └─ Outputs JSON with containers, their IPv4Address, IPv6Address, etc.
```

---

## Building Custom Images with Dockerfile

A **Dockerfile** is a plain text file that contains instructions (directives) to build a Docker image. It defines the
base image, sets up the filesystem, installs dependencies, and configures entrypoints.

### Dockerfile Basic Directives

1. **FROM**

    * Specifies the base image. Must be the first instruction in every Dockerfile.

2. **LABEL**

    * Adds metadata (key-value pairs) to the image (e.g., author, version).

3. **WORKDIR**

    * Sets the working directory for subsequent instructions (and for runtime if not overridden).

4. **COPY / ADD**

    * `COPY <src> <dest>`: Copies files/directories from host to image.
    * `ADD <src> <dest>`: Similar to `COPY`, but can also fetch remote URLs and auto-extract tar archives.

5. **RUN**

    * Executes a command in a new layer and commits the result to the image. Typically used to install packages or build software.

6. **ENV**

    * Defines environment variables that persist in the image.

7. **EXPOSE**

    * Documents that the container listens on the specified network ports at runtime. It does **not** actually publish
    * ports (you still need `docker run -p`).

8. **CMD**

    * Specifies the default command to run when a container is started from the image. Only one `CMD` is allowed; if
    * multiple are present, the last one takes effect.

9. **ENTRYPOINT**

    * Sets a fixed entrypoint, making the container behave like an executable. `CMD` can be used in conjunction to provide
    * default arguments to the `ENTRYPOINT`.

10. **USER**

    * Sets the user (and optionally group) for subsequent instructions and at container runtime.

11. **VOLUME**

    * Declares mount points for volumes (informational only; actual volumes are attached at runtime).

12. **ARG**

    * Defines a build-time variable that you can pass via `docker build --build-arg`.

### Writing a Simple Dockerfile

Let’s create a basic Node.js application Dockerfile as an example:

1. **Project Structure** (on host)

   ```
   my-node-app/
   ├── Dockerfile
   ├── package.json
   ├── package-lock.json
   └── index.js
   ```

    * `index.js` (example code):

      ```js
      // index.js
      const http = require('http');
      const PORT = process.env.PORT || 3000;
 
      const server = http.createServer((req, res) => {
        res.writeHead(200, { 'Content-Type': 'text/plain' });
        res.end('Hello from Dockerized Node.js!\n');
      });
 
      server.listen(PORT, () => {
        console.log(`Server listening on port ${PORT}`);
      });
      ```

2. **Dockerfile Contents**

   ```dockerfile
   # 1. Choose an official Node.js runtime as base image
   FROM node:18-alpine

   # 2. Label the image (optional but recommended for metadata)
   LABEL maintainer="Your Name <you@example.com>"

   # 3. Set working directory inside the image
   WORKDIR /usr/src/app

   # 4. Copy package.json and package-lock.json first (for dependency caching)
   COPY package*.json ./

   # 5. Install dependencies
   RUN npm install --production
   # If you need devDependencies during build, omit "--production"

   # 6. Copy application source code
   COPY . .

   # 7. Expose the port the app listens on
   EXPOSE 3000

   # 8. Define default environment variable (optional)
   ENV NODE_ENV=production

   # 9. Specify the default command to run the app
   CMD ["node", "index.js"]
   ```

   **Comments on each instruction:**

    1. `FROM node:18-alpine`

        * Uses the official Node.js 18 image built on Alpine Linux (small footprint).
    2. `LABEL maintainer="Your Name"`

        * Adds metadata about the image author.
    3. `WORKDIR /usr/src/app`

        * All subsequent actions occur inside `/usr/src/app`. If directory doesn’t exist, Docker creates it.
    4. `COPY package*.json ./`

        * Copies `package.json` and `package-lock.json` into the working directory. Using `package*.json` ensures you get
        * both files if they follow naming conventions.
    5. `RUN npm install --production`

        * Installs only dependencies listed under `"dependencies"` in `package.json`; omits dev dependencies to keep the image lean.
    6. `COPY . .`

        * Copies all other source files (including `index.js`) into the image.
    7. `EXPOSE 3000`

        * Documents that the container listens on port 3000. Doesn't actually publish the port—it’s informational and used by tools.
    8. `ENV NODE_ENV=production`

        * Sets the `NODE_ENV` environment variable to `production` to optimize package behavior at runtime.
    9. `CMD ["node", "index.js"]`

        * The default command to run when a container starts. Specified as a JSON array (exec form) to avoid shell processing.

### Building an Image from Dockerfile

From the `my-node-app` directory (where your `Dockerfile` lives), run:

```bash
docker build -t my-node-app:1.0 .
# -t my-node-app:1.0 : Tags the image as `my-node-app` with version `1.0`
# .                  : Builds using the Dockerfile in the current directory
```

* Docker reads the `Dockerfile`, executes each step, and layers the result.
* The `-t` (or `--tag`) flag assigns a name and optional tag (format: `name:tag`). If no tag is given, Docker uses `:latest` by default.

Verify image creation:

```bash
docker images | grep my-node-app
# my-node-app     1.0      7d8f9ab435cd   3 minutes ago   125MB
```

### Tagging Images

If you built an image without a specific tag or want to create another tag (for pushing to a registry):

```bash
docker tag my-node-app:1.0 mydockerhubusername/my-node-app:1.0
# └─ Prepares the image with a new repository path (e.g., for pushing to Docker Hub)
```

* `docker tag <local-image>:<tag> <registry-username>/<repo-name>:<tag>`
* After tagging, `docker images` will show both references.

### Multi-Stage Builds

Multi-stage builds allow you to separate “build” dependencies from “runtime” dependencies, minimizing final image size.
For example, building a Go binary and copying only the binary into a minimal image:

```dockerfile
# Stage 1: Build the Go binary
FROM golang:1.20-alpine AS builder

WORKDIR /app
COPY . .
RUN go mod download
RUN go build -o myapp .

# Stage 2: Copy binary into a minimal base image
FROM alpine:latest

# (optional) Install ca-certificates if your app needs HTTPS
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /app/myapp .

# Expose port and set entrypoint
EXPOSE 8080
ENTRYPOINT ["./myapp"]
```

* **Stage 1** (`builder`): Uses the `golang:1.20-alpine` image, compiles the code, produces a static `myapp` binary.
* **Stage 2**: Uses minimal `alpine:latest`, installs only what’s necessary (e.g., `ca-certificates`), and copies the
* compiled binary from the `builder` stage.
* Result: Final image is small (only the compiled binary and minimal runtime libs).

Build it:

```bash
docker build -t my-go-app:1.0 .
# Docker automatically keeps only the final stage in the tagged image
```

---

## Using a Docker Registry

A **registry** is a place to store and distribute Docker images. Docker Hub is the default public registry, but you
can run a private registry or use other services (e.g., AWS ECR, Google Container Registry, GitLab, etc.).

### Logging In to Docker Hub (or another Registry)

```bash
docker login
# └─ Prompts for Docker Hub username and password (or personal access token)
```

* If you need to login to a different registry (e.g., a private one), specify its URL:

  ```bash
  docker login my-registry.example.com
  # └─ Prompts for credentials for that registry
  ```

Credentials are stored in `~/.docker/config.json` by default.

### Pushing an Image

Assuming you have a local image tagged with your Docker Hub repo, e.g., `mydockerhubusername/my-node-app:1.0`:

```bash
docker push mydockerhubusername/my-node-app:1.0
# └─ Uploads image layers to Docker Hub (or the specified registry)
```

* If the image name matches `<username>/<repo>:<tag>`, Docker pushes to Docker Hub.
* For other registries, use the full registry domain prefix: `my-registry.example.com/my-node-app:1.0`.

### Pulling an Image

You can pull the same image on another machine:

```bash
docker pull mydockerhubusername/my-node-app:1.0
# └─ Downloads the image layers from Docker Hub
```

After pulling, you can run it the same way you do a local image:

```bash
docker run -d -p 3000:3000 mydockerhubusername/my-node-app:1.0
```

---

## Docker Compose (Introduction)

Docker Compose is a tool for defining and running multi-container Docker applications. With a single YAML
file (`docker-compose.yml`), you declare all your services, networks, and volumes, then bring them up with one command.

### Installing Docker Compose

Modern versions of Docker Desktop on macOS/Windows come with Compose pre-installed as `docker-compose` or integrated
as `docker compose`. On Linux, you might need to install separately:

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
```

Verify:

```bash
docker-compose --version
# docker-compose version 2.20.2, build e55c7b7
```

> **Note:** The “v2” series of Compose can also be invoked as `docker compose` (without hyphen), integrated into the Docker CLI.

### Writing a `docker-compose.yml`

Below is a simple example for a web application with two services: a web front-end (Node.js) and a Redis cache.

1. **Project Structure**

   ```
   my-app/
   ├── docker-compose.yml
   ├── web/
   │   ├── Dockerfile
   │   └── index.js
   └── redis_data/
   ```

2. **`docker-compose.yml` Contents**

   ```yaml
   version: '3.9'  # Use the latest v3.x Compose syntax

   services:
     web:
       build:
         context: ./web
         dockerfile: Dockerfile
       image: mydockerhubusername/my-web-app:latest
       container_name: web-app
       ports:
         - "8080:3000"
       environment:
         - REDIS_HOST=redis
         - REDIS_PORT=6379
       depends_on:
         - redis

     redis:
       image: redis:6-alpine
       container_name: redis
       volumes:
         - redis-data:/data

   volumes:
     redis-data:
       driver: local
   ```

   **Comments on Key Sections:**

    * `version: '3.9'`: Specifies Compose file format version.
    * **Services**: Defines two services, `web` and `redis`.

        * `web`:

            * `build.context`: Path to the directory containing the Dockerfile (`./web`).
            * `image`: Final image name/tag to build and (optionally) push later.
            * `container_name`: Assigns a custom name to the container (instead of random).
            * `ports`: Maps host port `8080` → container port `3000`.
            * `environment`: Sets environment variables inside the container.
            * `depends_on`: Ensures `redis` service starts before `web`.
        * `redis`:

            * `image`: Uses the official `redis:6-alpine` image.
            * `container_name`: Names the container `redis`.
            * `volumes`: Mounts a named volume `redis-data` at `/data` inside the Redis container.
    * **Volumes**: Defines `redis-data` as a named volume using the default `local` driver.

### Starting and Stopping with Compose

1. **Start All Services (Build if Needed)**

   From the directory with `docker-compose.yml`:

   ```bash
   docker-compose up -d
   # -d : Run all services in detached mode
   ```

    * Docker Compose will:

        1. Build `web` image (if there are changes in `web/Dockerfile` or source).
        2. Pull `redis:6-alpine` (if not already local).
        3. Create a network for the project (by default, named `my-app_default`).
        4. Create and start `redis` container, then `web` container (as per `depends_on`).

2. **View Running Services**

   ```bash
   docker-compose ps
   # NAME      COMMAND                  SERVICE   STATUS    PORTS
   # redis     "docker-entrypoint.s…"   redis     Up        6379/tcp
   # web-app   "node index.js"          web       Up        0.0.0.0:8080->3000/tcp
   ```

3. **View Logs for All Services**

   ```bash
   docker-compose logs -f
   # └─ Streams logs from both `web` and `redis` services
   ```

4. **Stop Services**

   ```bash
   docker-compose down
   # └─ Stops and removes containers, networks, and default volumes (unless you add `-v`)
   ```

    * If you want to remove volumes as well, use:

      ```bash
      docker-compose down -v
      # └─ Deletes named volumes (e.g., `redis-data`) and networks
      ```

### Scaling Services

If a service can run multiple replicas (typically stateless web servers), you can scale it:

```bash
docker-compose up -d --scale web=3
# └─ Runs 3 replicas of the `web` service, each in its own container (e.g., web_1, web_2, web_3)
```

* Note: Depending on how your application is architected, you might need a load balancer or NGINX in front to distribute traffic across replicas.
* You can also add `deploy: replicas: 3` under the `web` service in your Compose file if using Docker Swarm mode.

---

## Cleaning Up and Best Practices

Over time, Docker can accumulate unused images, containers, volumes, and networks. Cleaning up periodically keeps disk usage under control.

### Pruning Unused Resources

1. **Remove Exited Containers**

   ```bash
   docker container prune
   # └─ Prompts for confirmation, then removes all stopped containers
   ```

2. **Remove Unused Images**

   ```bash
   docker image prune
   # └─ Removes dangling images (images not tagged and not referenced by any container)
   ```

   To remove all images not used by any container (use with caution):

   ```bash
   docker image prune -a
   # └─ Removes all unused images, even if they’re tagged (as long as no container is using them)
   ```

3. **Remove Unused Volumes**

   ```bash
   docker volume prune
   # └─ Removes volumes not referenced by any container
   ```

4. **Remove Unused Networks**

   ```bash
   docker network prune
   # └─ Removes user-defined networks not used by any container
   ```

5. **System-Wide Prune**

   ```bash
   docker system prune
   # └─ Interactively removes all stopped containers, dangling images, unused networks
   #     To also remove all unused (dangling and unreferenced) volumes, add `--volumes`
   ```

   ```bash
   docker system prune -a --volumes
   # └─ WARNING: This is very aggressive! It removes EVERYTHING not in use:
   #   • All stopped containers
   #   • All networks not used by at least one container
   #   • All images (both dangling and unreferenced)
   #   • All build cache
   #   • All volumes not used by containers
   ```

### Tagging and Versioning Strategy

* Always tag your images with a meaningful version (e.g., `1.0.0`, `2025.05.31`, `v2-beta`) rather than relying solely on `:latest`.
* Use semantic versioning (SemVer) when possible (`MAJOR.MINOR.PATCH`).
* For CI/CD pipelines, consider tagging images with the Git SHA or pipeline build number to ensure reproducibility.

Example:

```bash
# Build and tag with version from package.json or Git SHA
VERSION=1.2.3
GIT_SHA=$(git rev-parse --short HEAD)

docker build -t myrepo/myapp:$VERSION -t myrepo/myapp:$GIT_SHA .
```

### Security Considerations

1. **Least Privilege**

    * Avoid running containers as `root` unless absolutely necessary. Use the `USER` directive in Dockerfile to switch to a non-root user.

      ```dockerfile
      # Example: After building, switch to `node` user inside the node:alpine image
      USER node
      ```
2. **Keep Images Updated**

    * Regularly rebuild images based on updated base images (e.g., `node:18-alpine`) to pull in security patches.
    * Use image scanning tools (e.g., Docker Bench for Security, Clair, Trivy) to detect vulnerabilities in your images.
3. **Limit Resource Usage**

    * Constrain CPU and memory to prevent a runaway container from consuming all host resources:

      ```bash
      docker run -d \
        --name resource_limited_container \
        --memory="512m" \
        --cpus="1.0" \
        my_image:latest
      ```
4. **Network Security**

    * Use user-defined bridge networks to isolate containers.
    * If exposing ports to the public internet, ensure proper firewall rules or reverse proxy with TLS termination.
    * Consider using Docker’s new “rootless mode” for extra security on Linux hosts.
5. **Secrets Management**

    * Never store secrets (passwords, API keys) directly in Dockerfiles or environment variables in plaintext.
    * Use Docker Secrets (in Swarm mode), or mount external secrets from a secure store (e.g., HashiCorp Vault) into the container at runtime.
    * Example:

      ```bash
      echo "mysecretpassword" | docker secret create db_password -
      ```

      Then in `docker-compose.yml` under `deploy:` section (Swarm mode), reference `db_password` as a secret.

---

## Daily Container Administration

Even in a simple single-host or small-cluster environment, you’ll want to have a routine to monitor containers, collect 
logs, gather performance metrics, and ensure critical services automatically recover from failures. This section outlines 
common daily administration tasks and demonstrates commands you can integrate into scripts or monitoring systems.

### 1. Log Collection

**Why it matters:**

* Logs let you troubleshoot application errors, track usage patterns, and investigate security incidents.
* Container logs (STDOUT/STDERR) are ephemeral: if you don’t collect or store them externally, you risk losing valuable 
* information when a container restarts or is removed.

#### a. Viewing and Tail-Following Logs

```bash
# View all logs from a container (stdout + stderr)
docker logs my-app-container
# └─ Prints everything the container has logged since start

# Follow logs in real-time (like `tail -f`)
docker logs -f my-app-container
# └─ Streams new output to your terminal until you hit CTRL+C

# Show only the last N lines (to avoid huge output)
docker logs --tail 100 my-app-container
# └─ Shows only the most recent 100 lines
```

> **Comment:**
>
> * `docker logs` works only for containers run with Docker’s default “json-file” log driver (the default). If you configured a different driver, you’ll need to fetch logs via that logging backend.
> * Combining `--tail` with `-f` lets you both see recent context and then follow new entries:
    >
    >   ```bash
>   docker logs --tail 50 -f my-app-container
>   ```

#### b. Configuring a Centralized JSON/File Log Rotation

By default, Docker stores logs in JSON files under `/var/lib/docker/containers/<container-id>/<container-id>-json.log`. These files can grow indefinitely. To avoid disk exhaustion:

1. **Set a Log Rotation Policy in `daemon.json`** (on the Docker host):

   Edit (or create) `/etc/docker/daemon.json` and add a `log-opts` section. For example:

   ```jsonc
   {
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "10m",
       "max-file": "5"
     }
   }
   ```

    * `"max-size": "10m"`: Each container’s log file is capped at 10 MB.
    * `"max-file": "5"`: Keep up to 5 rotated log files per container (e.g., `container-id-json.log.1`, `.2`, etc.).

2. **Restart the Docker Daemon** so changes take effect:

   ```bash
   sudo systemctl restart docker
   #—or—
   sudo service docker restart
   ```

3. **Verify Log Driver and Options** on a running container:

   ```bash
   docker inspect --format='{{ .HostConfig.LogConfig.Type }}' my-app-container
   # └─ Should print "json-file"

   docker inspect \
     --format='{{ range $index, $value := .HostConfig.LogConfig.Config }}{{ printf "%s=%s\n" $index $value }}{{ end }}' \
     my-app-container
   # └─ Should list max-size=10m, max-file=5
   ```

> **Comment:**
>
> * After configuring `daemon.json`, any *new* containers inherit these settings. Existing containers (already running) keep their previous log-driver until restarted or recreated.
> * If you need to centralize logs externally (e.g., to an ELK stack, Fluentd, or Splunk), you can switch the `log-driver` to something like `"gelf"` (for Graylog), `"fluentd"`, or `"syslog"`, and provide the appropriate options.

#### c. Shipping Logs to a Central Aggregator

If you run multiple containers across hosts, you’ll want a centralized log aggregator—Elasticsearch/Logstash/Kibana (ELK) or similar. \
You can configure each container to use the Fluentd or GELF driver:

```bash
docker run -d \
  --name my-app-container \
  --log-driver=gelf \
  --log-opt gelf-address=udp://log-collector.example.com:12201 \
  my-app-image:latest
# └─ Docker sends container logs over UDP to your GELF-compatible log collector
```

* **`--log-driver=gelf`**: Instructs Docker to use the Graylog Extended Log Format.
* **`--log-opt gelf-address=udp://HOST:PORT`**: Where to send logs.

Similarly, for Fluentd:

```bash
docker run -d \
  --name my-app-container \
  --log-driver=fluentd \
  --log-opt fluentd-address=fluentd-host:24224 \
  --log-opt tag="docker.myapp" \
  my-app-image:latest
# └─ Routes logs to Fluentd on TCP port 24224, tagging them “docker.myapp”
```

> **Comment:**
>
> * When using a non-default log driver, `docker logs` will not work. You must query your external log system.
> * Ensure your aggregator is highly available, especially for production workloads—don’t rely on a single Fluentd instance if you can spread the load or use a reliable queue.

---

### 2. Gathering Performance Metrics

Monitoring CPU, memory, network, and I/O usage helps you detect performance regressions, resource saturation, or outright 
failures. You can collect metrics in several ways:

#### a. `docker stats` for Live Metrics

```bash
docker stats
# └─ Shows live CPU %, memory usage, network I/O, block I/O for all running containers

# To monitor a specific container:
docker stats my-app-container
# └─ Streams resource usage for that one container

# Sample output columns:
# CONTAINER ID   NAME              CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
# f3f1d2c7c1b2   my-app-container  3.18%     150MiB / 1GiB       14.64%    1.2MB / 0.5MB  100kB / 20kB  12
```

> **Comment:**
>
> * `docker stats` samples in real-time (refreshing every second by default). Press `CTRL+C` to exit.
> * You can customize the output format with `--format` (Go templates) to integrate with scripts:

```bash
docker stats --no-stream --format "{{.Name}}: {{.CPUPerc}} CPU, {{.MemPerc}} Mem"
# └─ One-shot output for quick inclusion in logs or alerts
```

#### b. Integrating with cAdvisor or Prometheus

For continuous monitoring, consider running [cAdvisor](https://github.com/google/cadvisor) or using the Prometheus Docker plugin. Example with cAdvisor:

```bash
docker run -d \
  --name cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  gcr.io/cadvisor/cadvisor:latest
# └─ cAdvisor exposes metrics at http://localhost:8080/metrics (Prometheus format)
```

> **Comment:**
>
> * cAdvisor collects container-level metrics and exposes them in Prometheus format. You can point a Prometheus server at it, scrape metrics regularly, and visualize them in Grafana.
> * Ensure you mount the `/var/lib/docker` and `/sys` directories as `ro` so cAdvisor can read container stats without risk of modification.

#### c. Built-in Events and `docker events`

Docker emits events on container lifecycle changes (start, stop, die, kill, etc.). You can listen for them:

```bash
docker events
# └─ Streams a live feed of events (creation, deletion, health status changes, etc.)

# Filter for only “die” events (container stopped)
docker events --filter 'event=die'
# └─ Shows when any container exits
```

> **Comment:**
>
> * Integrate `docker events` into automation or alert systems. For instance, if a critical container “dies,” you can trigger a notification or remediation script.

---

### 3. Configuring Application Restart Policies

Ensuring that critical containers restart automatically if they crash or the daemon restarts is vital for high availability. Docker provides several restart policies:

* **`no`** (default): Do not automatically restart.
* **`on-failure[:max-retries]`**: Restart only if the container exits with a non-zero exit code. Optionally limit retries.
* **`always`**: Always restart the container if it stops (regardless of exit code). Also restarts on Docker daemon restarts.
* **`unless-stopped`**: Behaves like `always`, but does not restart if the user manually stopped the container.

#### a. Specifying Restart Policy at `docker run`

```bash
docker run -d \
  --name my-critical-service \
  --restart unless-stopped \
  my-critical-image:latest
# └─ If my-critical-service crashes (exit ≠ 0) or the Docker daemon restarts, Docker will automatically bring it back up.
```

* **`--restart on-failure:5`**: Try restarting up to 5 times if it exits with a non-zero status, then give up:

  ```bash
  docker run -d \
    --name flaky-worker \
    --restart on-failure:5 \
    my-worker-image
  ```

#### b. Updating an Existing Container’s Restart Policy

If you have a running container and want to change its restart behavior without recreating it:

```bash
docker update \
  --restart always \
  my-app-container
# └─ Changes restart policy in-place for `my-app-container`
```

> **Comment:**
>
> * `docker update` can change restart policies, resource limits (`--cpus`, `--memory`), and other settings on a live container.
> * If you switch from `no` to `always` (or `unless-stopped`), the next time the container stops (or Docker restarts), it will spin back up automatically.

#### c. Best Practices for Restart Policies

1. **Use `unless-stopped` for Long-Running Services**

    * E.g., databases, web servers, message queues—anything you want to keep alive unless you explicitly “docker stop” it.

2. **Use `on-failure` for Batch or One-Off Jobs**

    * E.g., data-processing jobs that should retry a few times on error but not loop indefinitely.

3. **Monitor Restart Loops**

    * If a container is continuously crashing (restarting over and over), it may overload the host.
    * Consider using health checks (see below) to avoid restart loops for containers stuck in a bad state.

#### d. Health Checks (Optional Enhancement)

You can define a `HEALTHCHECK` in your Dockerfile so Docker knows if your containerized application is “healthy”:

```dockerfile
# In Dockerfile
FROM nginx:latest

# Healthcheck: attempt to curl localhost:80 every 30 seconds, with a 5s timeout
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Rest of Dockerfile...
```

* Docker sets the container’s status to “unhealthy” if the check fails (after 3 retries).
* You can then combine this with `--restart on-failure` or put logic in your monitoring to alert and/or remove unhealthy containers.

Check health status:

```bash
docker ps
# └─ Look under “STATUS” for “Up X seconds (healthy)” or “unhealthy”
```

> **Comment:**
>
> * Health checks help ensure that Docker only routes traffic to containers that are truly up and running (especially when using orchestrators like Swarm or Kubernetes).
> * For standalone Docker, you can poll `docker inspect --format='{{ .State.Health.Status }}' my-container` in a script to detect unhealthy containers and take action (e.g., `docker restart my-container`).

---

## Hiding Sensitive Data in Image Building

Including secrets—API keys, TLS certificates, database credentials, or proprietary code—directly in a Dockerfile or built image is a common mistake that leads to information leakage. Because each `RUN` and `COPY` instruction creates a new layer, anything you write into those layers can be retrieved later (even if you later delete the file). Below are best practices for keeping secrets out of your final image.

### 1. Never Hard-Code Secrets in Dockerfile

❌ **Bad Example:** Embedding a private key or password directly in the Dockerfile:

```dockerfile
# BAD: Don’t do this!
FROM ubuntu:20.04

# Hard-coded API key stored in image layers
ENV AWS_SECRET_KEY=ABCD1234SECRET

RUN apt-get update && apt-get install -y python3
COPY . /app
CMD ["python3", "/app/app.py"]
```

* **Problem:** Even if you later overwrite or unset `AWS_SECRET_KEY`, that layer still contains it (visible via `docker history` or by inspecting the tarball).

### 2. Use Build-Time Arguments (ARG) Carefully

`ARG` allows you to pass a variable during `docker build`, but the value only exists during build-time and is not persisted into the image’s final metadata—unless you explicitly use it in an `ENV` or `RUN` that writes it. For example:

```dockerfile
# Dockerfile
FROM python:3.11-slim

# Declare build-time variable, no default
ARG MY_SECRET

# Use the secret in a build step (e.g., to download a private repo)
RUN git clone https://username:${MY_SECRET}@github.com/yourorg/private-repo.git /opt/app

# Remove any credentials after use (optional, but the password was still in the layer)
RUN rm -rf /opt/app/.git

WORKDIR /opt/app
RUN pip install -r requirements.txt

# NOTE: We do NOT set ENV MY_SECRET anywhere, so it will not persist into container
CMD ["python3", "main.py"]
```

* **How to build:**

  ```bash
  docker build --build-arg MY_SECRET=$(cat ~/secret.txt) -t my-private-app:latest .
  ```
* **Caveats:**

    * Even if you avoid `ENV`, the secret still appears in the intermediate layer created by that `RUN`. Anyone with access to the image’s layer tarballs can extract it.
    * To mitigate:

        1. Perform the secret-dependent operation (e.g., `git clone`) in a single `RUN` statement.
        2. Immediately delete the credential or credentials file before the end of that same `RUN`.
        3. Use multi-stage builds so that the layer containing the secret never appears in the final stage.

#### a. Multi-Stage Build to Keep Secrets Out

```dockerfile
# Stage 1: Download private repo using secret
FROM alpine/git AS downloader

ARG MY_SECRET
# Clone private repository with the secret
RUN git clone https://username:${MY_SECRET}@github.com/yourorg/private-repo.git /tmp/app-source

# Stage 2: Build final image without any secret
FROM python:3.11-slim

# Copy only the source code that’s needed, not the .git folder with embedded secret
COPY --from=downloader /tmp/app-source /opt/app

WORKDIR /opt/app
RUN pip install -r requirements.txt

CMD ["python3", "main.py"]
```

* **Comments on this multi-stage approach:**

    * In the first stage (`downloader`), the secret (`MY_SECRET`) is used to clone the private repo. All layers in this stage (including the one with the embedded URL+secret) remain confined to the “builder” image and are **not** carried forward.
    * In the second stage (the final image), you only copy the resulting files (`/tmp/app-source`) into a fresh, secret-free image.
    * Even though the first stage contains the secret, you generally tag only the final stage as `my-private-app:latest`. Because intermediate stages are not tagged unless you explicitly do so, casual users do not pull them.

#### b. Using Docker Secrets with BuildKit

Docker BuildKit (enabled by default in newer Docker versions) supports mounting secrets at build time without baking them into layers. Example:

1. **Enable BuildKit** (if not already enabled):

   ```bash
   export DOCKER_BUILDKIT=1
   ```

2. **Create a Secret File** locally (e.g., `db_password.txt`):

   ```bash
   echo "SuperSecretPassword" > db_password.txt
   ```

3. **Dockerfile Using `--mount=type=secret`**

   ```dockerfile
   # syntax=docker/dockerfile:1.4
   FROM ubuntu:20.04

   # Install client that needs secret (e.g., MySQL client)
   RUN apt-get update && apt-get install -y mysql-client

   # During build, mount secret at /run/secrets/db_password (in-memory only)
   RUN --mount=type=secret,id=db_pass \
       export DB_PASS=$(cat /run/secrets/db_pass) && \
       mysql --user=admin --password="$DB_PASS" \
             --host=mysql.example.com \
             --execute="USE mydatabase; SELECT VERSION();"

   # Rest of build (no secret remains in layers)
   COPY app /opt/app
   WORKDIR /opt/app
   RUN make build

   CMD ["./start-app"]
   ```

4. **Building with the Secret**

   ```bash
   docker build \
     --secret id=db_pass,src=db_password.txt \
     -t my-db-client-image:latest .
   ```

    * `--secret id=db_pass,src=db_password.txt`: Mounts the local file at build time under `/run/secrets/db_pass`.
    * **Why this is safe:** The secret is available only during that `RUN` command. It never persists to an image layer or intermediate filesystem.

> **Comment:**
>
> * BuildKit secrets are ephemeral; once the `RUN` finishes, the secret is discarded.
> * Ensure your Dockerfile’s first line (`syntax=...`) matches the required BuildKit version—older Docker releases may need a different syntax or explicit BuildKit enablement.

### 3. Environment Files and Avoiding `ENV` Leaks

If your application needs secrets at runtime (e.g., database credentials), pass them at container start, not at build time:

1. **Use an `.env` File (for `docker run`)**—Store secrets in a file that’s never added to Git:

   ```bash
   # .env (never commit this file)
   DB_HOST=prod-db.example.com
   DB_USER=admin
   DB_PASS=SuperSecretPassword
   ```

   Run container with:

   ```bash
   docker run -d \
     --env-file .env \
     --name my-app \
     my-app-image:latest
   # └─ Docker automatically injects every KEY=VALUE from .env as an environment variable
   ```

2. **Use Docker Secrets (Swarm Mode)**—For production, especially in swarm/kubernetes:

   ```bash
   # Create a secret in Docker Swarm
   echo "SuperSecretPassword" | docker secret create db_pass -
   ```

   In `docker-compose.yml` (v3.1+), reference the secret:

   ```yaml
   version: "3.8"

   services:
     web:
       image: my-app-image:latest
       secrets:
         - db_pass
       environment:
         - DB_USER=admin
         # DB_PASS is sourced from the secret
       configs:
         - source: app_config
           target: /etc/app/config.yaml

   secrets:
     db_pass:
       external: true
       name: db_pass
   ```

   Inside the container, the secret is mounted at `/run/secrets/db_pass` by default. Your application should read the file rather than expect an environment variable.

> **Comment:**
>
> * Docker Secrets (Swarm) encrypts secrets at rest and in transit; only services granted access can read them.
> * Never put secrets into a public Git repository—always keep them out of version control.

---

### 4. Summary of Hiding Secrets

1. **Never write secrets as `ENV` or `COPY` in a Dockerfile.** That bakes them into layers.
2. **Use multi-stage builds** to perform secret-requiring steps in a throwaway stage.
3. **Leverage BuildKit’s `--mount=type=secret`** for truly ephemeral build-time secrets.
4. **At runtime, inject secrets via `--env-file` or Docker Secrets** (Swarm). Do not bake runtime secrets into images.
5. **Keep `.env` files or secret material outside of Git.** Use `.gitignore` to prevent accidental commits.

By following these patterns, you can build images for private, sensitive workloads without fear of leaking credentials or proprietary code.

## Summary of Key Commands

Below is a quick-reference table of commonly used Docker commands. Remember to replace placeholders (e.g., `my-container`, `my-image:tag`) with actual names/tags.

| Operation                        | Command Example                                             | Comment                                                                                    |
| -------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Pull an image**                | `docker pull ubuntu:20.04`                                  | Downloads the `ubuntu:20.04` image from Docker Hub                                         |
| **List local images**            | `docker images`                                             | Shows all local images                                                                     |
| **Remove an image**              | `docker rmi my-image:1.0`                                   | Deletes the specified image (if no container depends on it)                                |
| **Run a container (detached)**   | `docker run -d --name my-container -p 8080:80 nginx:latest` | Runs Nginx in the background, mapping host port 8080 → container port 80                   |
| **List running containers**      | `docker ps`                                                 | Shows only containers that are currently running                                           |
| **List all containers**          | `docker ps -a`                                              | Shows running, stopped, and exited containers                                              |
| **Stop a container**             | `docker stop my-container`                                  | Gracefully stops the container (SIGTERM → SIGKILL after 10 seconds)                        |
| **Start a container**            | `docker start my-container`                                 | Starts a stopped container                                                                 |
| **Remove a container**           | `docker rm my-container`                                    | Deletes a stopped container                                                                |
| **Inspect container**            | `docker inspect my-container`                               | Dumps JSON details (network, mounts, environment, etc.)                                    |
| **View logs**                    | `docker logs -f my-container`                               | Streams logs (STDOUT/STDERR) of the container                                              |
| **Execute inside container**     | `docker exec -it my-container /bin/bash`                    | Opens an interactive Bash shell inside a running container                                 |
| **Create a named volume**        | `docker volume create my-vol`                               | Creates a Docker-managed volume named `my-vol`                                             |
| **Run container with volume**    | `docker run -d -v my-vol:/data redis:latest`                | Mounts named volume `my-vol` at `/data` inside Redis container                             |
| **Remove a volume**              | `docker volume rm my-vol`                                   | Deletes the volume if unused by containers                                                 |
| **Create a network**             | `docker network create app-net`                             | Creates a user-defined bridge network `app-net`                                            |
| **Run container on network**     | `docker run -d --name db --network app-net postgres:13`     | Connects PostgreSQL container to `app-net`                                                 |
| **Inspect a network**            | `docker network inspect app-net`                            | Shows containers connected, IP addresses, driver info                                      |
| **Build image from Dockerfile**  | `docker build -t my-app:1.0 .`                              | Builds an image from `Dockerfile` in current directory, tags it `my-app:1.0`               |
| **Tag an existing image**        | `docker tag my-app:1.0 myrepo/my-app:1.0`                   | Assigns repository prefix (for registry push)                                              |
| **Push image to registry**       | `docker push myrepo/my-app:1.0`                             | Uploads image to Docker Hub or specified registry                                          |
| **Docker Compose up (detached)** | `docker-compose up -d`                                      | Builds (if needed) and starts all services defined in `docker-compose.yml`                 |
| **Docker Compose down**          | `docker-compose down`                                       | Stops and removes containers, networks, and default volumes (unless `-v` used for volumes) |
| **Prune system (aggressive)**    | `docker system prune -a --volumes`                          | Removes all unused images, containers, networks, build cache, and volumes                  |

## Final Notes

1. **Experiment and Iterate**

    * Docker has many more features (e.g., Swarm mode, Kubernetes integration, custom storage drivers). Start with these core operations until you’re comfortable.
    * Consider reading official docs ([https://docs.docker.com](https://docs.docker.com)) for deeper dives on security, Swarm, Kubernetes, BuildKit, etc.

2. **Leverage Layers and Caching**

    * Order your Dockerfile instructions so that less frequently changing steps (e.g., dependency installs) happen before copying source code. Docker caches layers to speed up subsequent builds.
    * Example: copy `package.json` → run `npm install` → copy the rest of your source. This way, if you only change application code (not dependencies), Docker reuses the cached `npm install` layer.

3. **Automate Builds and Deployments**

    * Use CI/CD tools (GitHub Actions, GitLab CI, Jenkins, etc.) to automatically build, test, tag, and push images on every commit or release.
    * Store credentials securely via environment variables or vault services.
    * Integrate security scanning (Trivy, Snyk) as part of your pipeline.

