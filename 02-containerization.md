# 02 - Containerization

## Virtual Machines vs Containers: A Fundamental Shift

Understanding containers requires recognizing how they differ from traditional virtual machines. Virtual machines create complete isolated environments by running full guest operating systems on top of a hypervisor like VMware or VirtualBox. Each VM consumes gigabytes of storage and memory, takes minutes to boot, and provides hardware-level isolation at the cost of significant resource overhead.

Containers take a radically different approach. Instead of virtualizing entire machines, they leverage OS-level virtualization to share the host kernel while maintaining process-level isolation. This architectural difference allows containers to measure in megabytes rather than gigabytes, launch in milliseconds instead of minutes, and consume minimal system resources while delivering comparable isolation benefits.

## Key Problems Containers Solve

### Eliminating Environment Inconsistencies

The classic "it works on my machine" dilemma has plagued developers for decades. Code that runs flawlessly on a developer's local setup suddenly breaks when deployed to staging or production environments. Containers fundamentally solve this by encapsulating applications with their entire runtime environment - dependencies, libraries, system tools, and configuration settings - into a portable package that behaves identically across any infrastructure.

### Resolving Library Version Conflicts

Modern applications rely on countless external libraries, often with specific version requirements. When multiple applications share the same server, they frequently demand incompatible versions of the same dependency, creating what developers call "dependency hell." Containers isolate each application in its own environment, allowing different applications to use whatever library versions they need without interference.

### Simplifying Infrastructure Management

Traditional deployment requires extensive server configuration through installation scripts, manual setup procedures, and documentation that quickly becomes obsolete. Containers replace this complexity with declarative Dockerfiles that define the entire environment as code. This approach ensures consistent, repeatable deployments while making infrastructure changes trackable and reversible.

### Maximizing Hardware Efficiency

Virtual machines solve isolation problems but at a steep cost - each VM requires its own complete operating system, consuming gigabytes of memory and CPU resources. Containers achieve the same isolation benefits while sharing the host operating system kernel, reducing resource overhead from gigabytes to mere megabytes per instance.

### Accelerating Development Velocity

Container orchestration enables rapid scaling and deployment that traditional infrastructure can't match. While virtual machines take minutes to boot and configure, containers launch in seconds. This speed advantage transforms not only production deployments but also development workflows, testing cycles, and the ability to respond quickly to changing demands.

## Core Concepts and Terminology

### Container

A running instance of an application with all its dependencies, isolated from other processes.

```bash
# Create and start a container from an image
docker run --name my-container nginx:alpine

# List running containers
docker ps

# Stop the container
docker stop my-container

# Remove the container
docker rm my-container
```

### Image

A read-only template used to create containers. Images are composed of layers that are stacked on top of each other.

```bash
# List local images
docker images
```

### Registry

A centralized service for storing and distributing images. Docker Hub is the default public registry.

```bash
# Pull from Docker Hub (default)
docker pull nginx

# Pull from a specific registry
docker pull gcr.io/google-containers/pause:3.2

# List images with their registry sources
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Dockerfile

A text file with instructions for building an image.

```bash
# Create a simple Dockerfile
cat > Dockerfile << 'EOF'
FROM alpine:latest
RUN apk add --no-cache curl
CMD ["curl", "--version"]
EOF

# Build image from Dockerfile
docker build -t my-curl:latest .

# Run container from built image
docker run --rm my-curl:latest
```

## How Containerization Works Under the Hood

### Linux Kernel Features

Containers leverage three main Linux kernel features:

#### Namespaces: Isolation Mechanism

Namespaces partition kernel resources so each container sees its own isolated instance. Linux implements eight namespace types:

- PID namespace: Isolates process IDs
- Network namespace: Isolates network interfaces, routing tables
- Mount namespace: Isolates filesystem mount points
- UTS namespace: Isolates hostname and domain name
- IPC namespace: Isolates inter-process communication
- User namespace: Isolates user and group IDs
- Cgroup namespace: Isolates cgroup hierarchy
- Time namespace: Isolates system clocks

When a container starts, the kernel creates new namespaces for it. The container's processes think they're the only ones on the system (PID 1), have their own network stack, and see only their mounted filesystems.

#### Control Groups (cgroups): Resource Management

Cgroups serve two main purposes: tracking resource usage and enforcing resource limits across CPU, memory, disk I/O, and network. These limits are organized hierarchically, meaning administrators can apply restrictions at multiple levels, from broad groups down to specific processes. When it comes to enforcement, the Linux kernel takes direct action. For memory, if a container exceeds its allocation, the kernel will kill processes inside that container. For CPU, the kernel enforces limits through its scheduler - a container limited to 0.5 CPUs will receive half the CPU time compared to an unrestricted container, effectively throttling its processing power.

#### Union File Systems: Efficient Storage

Union filesystems enable the layered architecture of images. When you run a container:

1. Read-only image layers are stacked using a union filesystem
2. A thin writable layer is added on top
3. Changes are written to this top layer (copy-on-write)
4. Multiple containers can share the same base layers

This means if you have 10 containers running from the same nginx image, the base nginx layers exist only once on disk.

### The Container Runtime Pipeline

When you run `docker run nginx`:

1. Image Resolution: Docker checks if the image exists locally, otherwise pulls from registry
2. Storage Preparation: Extracts and prepares filesystem layers
3. Namespace Creation: Creates isolated namespaces for the container
4. Cgroup Configuration: Sets up resource limits
5. Network Setup: Creates virtual network interfaces
6. Security Profiles: Applies AppArmor/SELinux profiles, capabilities, seccomp filters
7. Process Launch: Starts the container's main process in the prepared environment

## Docker Fundamentals

### Installation Verification

```bash
# Verify Docker installation
docker --version
docker info

# Test Docker installation
docker run --rm hello-world
```

### Getting Help

```bash
# Get help on any command
docker --help
docker container --help
docker run --help
```

### Understanding Docker Architecture

Docker uses a client-server architecture:

- Docker Client: The `docker` command you use
- Docker Daemon: Background service managing containers
- Docker Registry: Stores images

## Working with Images

### Pulling and Managing Images

```bash
# Pull an image (latest tag by default)
docker pull ubuntu

# Pull specific version
docker pull ubuntu:20.04

# List local images
docker images

# Search Docker Hub
docker search redis --limit 5

# Remove an image
docker rmi ubuntu:20.04

# Remove all unused images
docker image prune -a
```

### Image Layers and Storage

```bash
# Inspect image details
docker inspect nginx:alpine

# View image layers
docker history nginx:alpine

# View storage usage
docker system df
```

### Tagging Images

```bash
# Tag an image
docker tag nginx:alpine my-nginx:v1

# Multiple tags can point to same image ID
docker tag nginx:alpine my-nginx:latest

# View tags
docker images | grep my-nginx
```

## Container Management

### Running Containers

```bash
# Basic run (foreground)
docker run --rm alpine echo "Hello World"

# Interactive mode
docker run -it --rm ubuntu:20.04 /bin/bash
# Type 'exit' to leave

# Detached mode (background)
docker run -d --name webserver nginx:alpine

# With environment variables
docker run --rm -e MY_VAR="Hello" alpine env

# With port mapping
docker run -d -p 8080:80 --name web nginx:alpine
# Access at http://localhost:8080

# Clean up
docker rm -f web webserver
```

### Container Lifecycle Management

```bash
# Create a test container
docker run -d --name test-nginx nginx:alpine

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop, start, restart
docker stop test-nginx
docker start test-nginx
docker restart test-nginx

# Pause/unpause (freeze processes)
docker pause test-nginx
docker unpause test-nginx

# View logs
docker logs test-nginx
docker logs -f test-nginx  # Follow logs (Ctrl+C to exit)

# Execute commands in running container
docker exec test-nginx ls -la /
docker exec -it test-nginx /bin/sh
# Type 'exit' to leave

# Copy files
echo "Hello from host" > test.txt
docker cp test.txt test-nginx:/tmp/
docker exec test-nginx cat /tmp/test.txt

# Inspect container
docker inspect test-nginx | head -20

# View resource usage
docker stats --no-stream test-nginx

# Clean up
docker rm -f test-nginx
rm -f test.txt
```

### Resource Limits

```bash
# Memory limit
docker run -d --memory="512m" --name mem-limited nginx:alpine

# CPU limit (0.5 = half a CPU)
docker run -d --cpus="0.5" --name cpu-limited nginx:alpine

# Verify limits
docker stats --no-stream mem-limited cpu-limited

# Clean up
docker rm -f mem-limited cpu-limited
```

## Data Persistence with Volumes

Docker containers are ephemeral by default - when a container is removed, its data is lost. Volumes provide ways to persist data beyond the container lifecycle, share data between containers, and manage application state.

### Understanding Volume Types

Docker provides three ways to persist data:

1. Volumes: Managed by Docker (recommended for production)
2. Bind mounts: Map host directories (great for development)
3. tmpfs mounts: Store in memory only (for sensitive/temporary data)

### Named Volumes

```bash
# Create a named volume
docker volume create mydata

# List volumes
docker volume ls

# Use volume in container
docker run -d \
  --name data-test \
  -v mydata:/data \
  alpine sh -c "echo 'Persistent data' > /data/test.txt && sleep 3600"

# Verify data exists
docker exec data-test cat /data/test.txt

# Remove container (volume persists)
docker rm -f data-test

# Mount volume in new container
docker run --rm \
  -v mydata:/data \
  alpine cat /data/test.txt

# Clean up
docker volume rm mydata
```

### Bind Mounts

```bash
# Create a test directory
mkdir -p ~/docker-data
echo "Hello from host" > ~/docker-data/test.txt

# Mount host directory
docker run --rm \
  -v ~/docker-data:/data \
  alpine cat /data/test.txt

# Development use case: Live code updates
mkdir -p ~/my-website
echo "<h1>Hello World</h1>" > ~/my-website/index.html

docker run -d \
  --name web-dev \
  -p 8080:80 \
  -v ~/my-website:/usr/share/nginx/html:ro \
  nginx:alpine

echo "Visit http://localhost:8080"
echo "Edit ~/my-website/index.html and refresh browser"

# Clean up
docker rm -f web-dev
rm -rf ~/docker-data ~/my-website
```

### tmpfs Mounts

```bash
# Create tmpfs mount (memory-only storage)
docker run -d \
  --name tmp-test \
  --tmpfs /app/cache \
  alpine sh -c "echo 'temporary data' > /app/cache/test.txt && sleep 60"

# Verify it's using tmpfs
docker exec tmp-test df -h /app/cache
docker exec tmp-test cat /app/cache/test.txt

# tmpfs with size limit
docker run -d \
  --name tmp-sized \
  --tmpfs /app/cache:size=100m \
  alpine sh -c "echo 'cache data' > /app/cache/data.txt && sleep 60"

# Check the mount
docker exec tmp-sized df -h /app/cache

# Practical use case: Temporary cache for web server
docker run -d \
  --name web-cache \
  -p 8081:80 \
  --tmpfs /var/cache/nginx:size=100m \
  nginx:alpine

echo "Nginx running with memory-only cache at http://localhost:8081"
echo "Cache data is never written to disk"

# Clean up
docker rm -f tmp-test tmp-sized web-cache
```

## Container Networking

Docker networking allows containers to communicate with each other and the outside world. By default, containers are isolated, but Docker provides several networking options to connect them securely.

### Network Types

Docker provides several network drivers:

- bridge: Default network for containers (isolated network segment)
- host: Container uses host's network directly (no isolation)
- none: No network access (complete isolation)
- overlay: Multi-host networking (used in Swarm mode)

### Bridge Networking (Default)

```bash
# Run containers on default bridge
docker run -d --name web1 nginx:alpine
docker run -d --name web2 nginx:alpine

# Containers can communicate via IP (not name on default bridge)
WEB1_IP=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1)
docker exec web2 ping -c 1 $WEB1_IP

# Clean up
docker rm -f web1 web2
```

### Custom Bridge Networks

```bash
# Create custom network
docker network create myapp-network

# Run containers on custom network
docker run -d --name app1 --network myapp-network alpine sleep 1000
docker run -d --name app2 --network myapp-network alpine sleep 1000

# Containers can resolve each other by name!
docker exec app1 ping -c 1 app2

# Clean up
docker rm -f app1 app2
docker network rm myapp-network
```

### Port Mapping

```bash
# Map container port to host
docker run -d -p 8080:80 --name web nginx:alpine
echo "Access at http://localhost:8080"

# Random host port
docker run -d -P --name web2 nginx:alpine
docker port web2

# Clean up
docker rm -f web web2
```

## Building Custom Images

A Dockerfile is a text file containing instructions that Docker uses to build custom images. Think of it as a recipe that defines the environment your application needs to run - from the base operating system to your application code and runtime configuration.

### Creating a Dockerfile

```bash
# Create project directory
mkdir -p ~/my-app && cd ~/my-app

# Create application file
cat > app.js << 'EOF'
const http = require('http');
const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from Docker!\n');
});
server.listen(3000, () => console.log('Server on port 3000'));
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:16-alpine
WORKDIR /app
COPY app.js .
EXPOSE 3000
CMD ["node", "app.js"]
EOF

# Build the image
docker build -t my-app:v1 .

# Run container
docker run -d -p 3000:3000 --name myapp my-app:v1

# Wait for container to start
sleep 2

# Test it
curl http://localhost:3000

# Clean up
docker rm -f myapp
docker rmi my-app:v1
cd ~ && rm -rf ~/my-app
```

### Multi-stage Builds

```bash
# Create Go example
mkdir -p ~/go-app && cd ~/go-app

cat > main.go << 'EOF'
package main
import "fmt"
func main() {
    fmt.Println("Hello from multi-stage build!")
}
EOF

cat > Dockerfile << 'EOF'
# Build stage
FROM golang:1.19-alpine AS builder
WORKDIR /build
COPY main.go .
RUN go build -o myapp main.go

# Runtime stage
FROM alpine:latest
WORKDIR /app
COPY --from=builder /build/myapp .
CMD ["./myapp"]
EOF

# Build and run
docker build -t go-app:slim .
docker run --rm go-app:slim

# Compare sizes (golang image may not be present locally)
echo "Final image size:"
docker images go-app:slim

# Clean up
docker rmi go-app:slim
cd ~ && rm -rf ~/go-app
```

## Docker Compose for Multi-Container Applications

Docker Compose is a tool for defining and running multi-container applications. Instead of running multiple `docker run` commands, you define your entire application stack in a YAML file and manage it with simple commands. Perfect for development environments and small-scale deployments.

### Basic Docker Compose

```bash
# Create project
mkdir -p ~/compose-demo && cd ~/compose-demo

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro

  api:
    image: node:16-alpine
    command: >
      sh -c "echo \"const http = require('http');
      http.createServer((req, res) => {
        res.writeHead(200, {'Content-Type': 'application/json'});
        res.end(JSON.stringify({message: 'API Response'}));
      }).listen(3000, () => console.log('API on port 3000'));\" | node"
    expose:
      - "3000"

volumes:
  app-data:
EOF

# Create HTML file
mkdir html
echo "<h1>Docker Compose Demo</h1>" > html/index.html

# Start services
docker compose up -d

# View status
docker compose ps

# View logs
docker compose logs

# Stop and remove
docker compose down -v

# Clean up
cd ~ && rm -rf ~/compose-demo
```

## Best Practices

Run as non-root user

```dockerfile
FROM node:16-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs
```

Use minimal base images

```dockerfile
FROM alpine:latest  # Small and secure
```

Don't include secrets in images

```bash
# Use environment variables or Docker secrets
docker run -e API_KEY=$API_KEY myapp
```

Scan images for vulnerabilities

```bash
docker scout cves nginx:alpine  # If Docker Scout available
```

### Image Optimization

```bash
# Create optimization example
mkdir -p ~/optimization-example && cd ~/optimization-example

# Create .dockerignore to exclude unnecessary files
cat > .dockerignore << 'EOF'
node_modules
.git
.env
*.log
*.md
.DS_Store
EOF

# Example: Inefficient Dockerfile (multiple layers)
cat > Dockerfile.inefficient << 'EOF'
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean
EOF

# Example: Optimized Dockerfile (single layer)
cat > Dockerfile.optimized << 'EOF'
FROM ubuntu:20.04
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
EOF

# Build both and compare layer count
docker build -f Dockerfile.inefficient -t app:inefficient .
docker build -f Dockerfile.optimized -t app:optimized .

# Compare number of layers
echo "Inefficient image layers:"
docker history app:inefficient | wc -l

echo "Optimized image layers:"
docker history app:optimized | wc -l

# Clean up
docker rmi app:inefficient app:optimized
cd ~ && rm -rf ~/optimization-example
```

### Container Health Checks

```bash
# Add health check
docker run -d \
  --name healthy-app \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=3s \
  --health-retries=3 \
  nginx:alpine

# Check health status
sleep 35
docker inspect --format='{{.State.Health.Status}}' healthy-app

# Clean up
docker rm -f healthy-app
```

## Docker Swarm: Container Orchestration

Docker Swarm is Docker's built-in orchestration tool that turns a group of Docker hosts into a single virtual Docker host. It provides native clustering capabilities to Docker, enabling high availability, load balancing, and easy scaling of containerized applications across multiple machines.

### Initialize Swarm

```bash
# Initialize single-node swarm
docker swarm init

# View swarm info
docker info | grep Swarm

# List nodes
docker node ls
```

### Create Services

```bash
# Create a replicated service
docker service create \
  --name web \
  --replicas 3 \
  --publish 8080:80 \
  nginx:alpine

# List services
docker service ls

# Scale service
docker service scale web=5

# Update service
docker service update --image nginx:latest web

# Remove service
docker service rm web
```

### Leave Swarm

```bash
# Leave swarm mode
docker swarm leave --force
```

## Docker Command Quick Reference

### Essential Commands

```bash
# Images
docker pull <image>              # Download image
docker images                    # List images
docker rmi <image>              # Remove image

# Containers
docker run <image>              # Run container
docker ps                       # List running containers
docker ps -a                    # List all containers
docker stop <container>         # Stop container
docker rm <container>           # Remove container
docker logs <container>         # View logs
docker exec -it <container> sh  # Shell access

# Volumes
docker volume create <name>     # Create volume
docker volume ls                # List volumes
docker volume rm <name>         # Remove volume

# Networks
docker network create <name>    # Create network
docker network ls               # List networks
docker network rm <name>        # Remove network

# Compose
docker compose up -d            # Start services
docker compose down             # Stop services
docker compose logs             # View logs

# Cleanup
docker system prune -a          # Remove all unused resources
```
