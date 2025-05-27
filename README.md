# Complete Docker & Kubernetes Tutorial 🐳 ⚓

A comprehensive guide to containerization with Docker and orchestration with Kubernetes, designed for beginners who want to understand these technologies from the ground up.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1: Docker Fundamentals](#part-1-docker-fundamentals)
  - [Installation](#docker-installation)
  - [Core Concepts](#docker-core-concepts)
  - [Building Your First Application](#building-your-first-application)
  - [Understanding Key Files](#understanding-key-files)
  - [Docker Commands Reference](#docker-commands-reference)
- [Part 2: Kubernetes Orchestration](#part-2-kubernetes-orchestration)
  - [Installation](#kubernetes-installation)
  - [Core Concepts](#kubernetes-core-concepts)
  - [Deploying to Kubernetes](#deploying-to-kubernetes)
  - [Kubernetes Commands Reference](#kubernetes-commands-reference)
  - [Troubleshooting](#troubleshooting)
- [Advanced Topics](#advanced-topics)
- [Cleanup and Uninstallation](#cleanup-and-uninstallation)

## Overview

This tutorial teaches you how to containerize applications using Docker and manage those containers at scale using Kubernetes. You'll build a simple Node.js web application, containerize it with Docker, and then deploy it to a Kubernetes cluster for orchestration.

**What You'll Learn:**
- How to containerize applications with Docker
- Understanding Docker images, containers, and the build process
- Managing application dependencies and configurations
- Orchestrating containers with Kubernetes
- Scaling, load balancing, and managing containerized applications
- Best practices for container deployment and management

## Prerequisites

- Ubuntu/Debian-based Linux system
- Basic command line knowledge
- VS Code or any text editor
- Basic understanding of web applications

## Part 1: Docker Fundamentals

### Docker Installation

Docker provides a platform for developing, shipping, and running applications using containerization. Let's start by installing Docker on your Linux system.

```bash
# Update your package index
sudo apt update

# Install Docker Engine
sudo apt install docker.io

# Start and enable the Docker service to run automatically
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to the docker group to avoid using sudo with every command
sudo usermod -aG docker $USER

# Note: You'll need to log out and back in for this group change to take effect
```

After installation, verify Docker is working:

```bash
docker --version
docker run hello-world
```

### Docker Core Concepts

Before we build our application, let's understand the fundamental concepts that make Docker powerful:

**Images:** Think of images as blueprints or templates. They contain everything needed to run an application including the code, runtime environment, libraries, and system tools. Images are read-only and immutable.

**Containers:** Containers are running instances of images. When you start a container from an image, Docker creates a writable layer on top of the read-only image layers. Multiple containers can run from the same image simultaneously.

**Dockerfile:** This is a text file containing instructions that Docker uses to build an image automatically. Each instruction creates a new layer in the image, making builds efficient and cacheable.

**Build Context:** When you run `docker build`, Docker sends the current directory (called the build context) to the Docker daemon. The `.dockerignore` file helps exclude unnecessary files from this context.

### Building Your First Application

Let's create a simple Node.js web application that we'll containerize. This example demonstrates the complete workflow from application code to running container.

#### Step 1: Create the Web Application

First, create a new directory for your project and navigate into it:

```bash
mkdir docker-k8s-tutorial
cd docker-k8s-tutorial
```

Create the main application file `app.js`:

```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Define a simple route that responds to HTTP GET requests
app.get('/', (req, res) => {
    res.send('Hello from Docker! 🐳');
});

// Add a health check endpoint for Kubernetes probes
app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// Start the server and listen on the specified port
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

#### Step 2: Define Project Dependencies

Create `package.json` to define your project metadata and dependencies:

```json
{
  "name": "docker-k8s-tutorial",
  "version": "1.0.0",
  "description": "A comprehensive Docker and Kubernetes tutorial application",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "author": "Your Name",
  "license": "MIT"
}
```

### Understanding Key Files

Let's explore each file's role in the containerization process and why each component is essential.

#### The Application File (`app.js`)

The `app.js` file serves as the entry point of your web application. In the context of containerization, this file becomes the core process that runs inside your Docker container. Here's what makes it special:

**Route Handling:** The application defines HTTP endpoints that respond to incoming requests. The root endpoint (`/`) serves as the main page, while the `/health` endpoint provides a way for Kubernetes to check if the application is running correctly.

**Environment Configuration:** Notice how we use `process.env.PORT || 3000`. This allows the application to read the port number from environment variables, making it flexible for different deployment scenarios. In containers, this flexibility is crucial because port assignments might vary.

**Server Initialization:** The `app.listen()` function starts the HTTP server inside the container. When Docker runs this container, this server process becomes the main process that keeps the container alive.

#### The Package Configuration (`package.json`)

The `package.json` file acts as the blueprint for your Node.js application's dependencies and metadata. In Docker environments, this file plays several critical roles:

**Dependency Declaration:** The `dependencies` section lists all the external libraries your application needs. When Docker builds your image, it uses this file to install the exact packages required, ensuring consistency across different environments.

**Script Definition:** The `scripts` section defines commands that can be executed with npm. The `start` script is particularly important because Docker will use this to launch your application inside the container.

**Version Locking:** When combined with `package-lock.json` (generated when you run `npm install`), this file ensures that the same versions of dependencies are installed every time, preventing the "it works on my machine" problem.

#### Step 3: Create the Dockerfile

The Dockerfile contains the instructions for building your Docker image:

```dockerfile
# Use an official Node.js runtime as the base image
# Alpine Linux version is smaller and more secure
FROM node:18-alpine

# Set the working directory inside the container
# All subsequent commands will be executed from this directory
WORKDIR /app

# Copy package.json and package-lock.json first
# This allows Docker to cache the dependency installation step
COPY package*.json ./

# Install the application dependencies
# This step is cached unless package*.json files change
RUN npm install

# Copy the rest of the application source code
# This step runs after dependencies to optimize build caching
COPY . .

# Create a non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Change ownership of the app directory to the new user
RUN chown -R nextjs:nodejs /app
USER nextjs

# Expose the port the application will listen on
# This is primarily for documentation and inter-container communication
EXPOSE 3000

# Define the default command to run when the container starts
CMD ["npm", "start"]
```

#### Understanding the Dockerfile Instructions

Let's break down each instruction and understand why the order matters:

**FROM node:18-alpine:** This instruction sets the base image for your container. The Alpine variant is chosen because it's significantly smaller than the full Ubuntu-based Node.js images while still providing everything needed to run Node.js applications.

**WORKDIR /app:** This creates and sets the working directory inside the container. All subsequent commands will execute from this location, and it provides a clean, predictable environment for your application.

**COPY package*.json ./:** By copying the package files first, we take advantage of Docker's layer caching. If your application code changes but dependencies don't, Docker can reuse the cached layer with installed dependencies, making builds much faster.

**RUN npm install:** This installs the dependencies inside the container. Since this happens after copying package files but before copying source code, dependency installation is cached until package files change.

**COPY . .:** This copies your application source code into the container. It happens last to ensure that changes to your code don't invalidate the dependency installation cache layer.

**Security Considerations:** The Dockerfile creates a non-root user to run the application, following security best practices by not running applications as the root user inside containers.

#### Step 4: Create Docker Ignore File

Create `.dockerignore` to exclude unnecessary files from the build context:

```
# Dependencies that will be installed fresh in the container
node_modules

# Log files that aren't needed in the image
npm-debug.log
*.log

# Development files
.git
.gitignore

# Docker-related files
Dockerfile
.dockerignore

# IDE and editor files
.vscode
.idea

# OS-specific files
.DS_Store
Thumbs.db

# Environment files (should be handled via Kubernetes ConfigMaps/Secrets)
.env
.env.local
```

#### The Docker Ignore File (`.dockerignore`)

The `.dockerignore` file serves a similar purpose to `.gitignore` but for Docker builds. Understanding its importance helps you create efficient, secure container images:

**Build Performance:** By excluding the `node_modules` directory, Docker doesn't waste time copying potentially thousands of dependency files from your local machine. Instead, dependencies are installed fresh inside the container, ensuring consistency.

**Security:** Excluding `.env` files prevents accidentally including sensitive configuration or credentials in your Docker image. In production, these should be handled through Kubernetes ConfigMaps and Secrets.

**Image Size:** Excluding development tools, logs, and temporary files keeps your final image smaller, leading to faster deployments and reduced storage costs.

**Cache Efficiency:** By excluding files that change frequently but aren't needed in the container, you improve Docker's layer caching effectiveness.

### Building and Running Your Docker Container

Now let's build and test your containerized application:

```bash
# Build the Docker image with a descriptive tag
docker build -t my-node-app:latest .

# Verify the image was created successfully
docker images | grep my-node-app

# Run the container with port mapping
docker run -d -p 3000:3000 --name my-app-container my-node-app:latest

# Check if the container is running
docker ps

# Test the application
curl http://localhost:3000
# Should return: Hello from Docker! 🐳

# Check application logs
docker logs my-app-container

# Stop and remove the container when done testing
docker stop my-app-container
docker rm my-app-container
```

### Docker Commands Reference

Here are the essential Docker commands you'll use regularly, with explanations of when and why to use each:

#### Image Management Commands

```bash
# List all Docker images on your system
docker images

# Remove a specific image (useful for cleanup)
docker rmi <image_name_or_id>

# Remove all unused images (great for freeing up space)
docker image prune

# Build an image from a Dockerfile in the current directory
docker build -t <image_name>:<tag> .

# Pull an image from a registry (like Docker Hub)
docker pull <image_name>:<tag>
```

#### Container Management Commands

```bash
# Run a container from an image
docker run -d -p <host_port>:<container_port> --name <container_name> <image_name>

# List all running containers
docker ps

# List all containers (including stopped ones)
docker ps -a

# Stop a running container gracefully
docker stop <container_name_or_id>

# Force stop a container (use with caution)
docker kill <container_name_or_id>

# Remove a stopped container
docker rm <container_name_or_id>

# Remove all stopped containers
docker container prune
```

#### Debugging and Inspection Commands

```bash
# View logs from a container
docker logs <container_name_or_id>

# Follow logs in real-time (like tail -f)
docker logs -f <container_name_or_id>

# Execute a command inside a running container
docker exec -it <container_name_or_id> /bin/sh

# Inspect detailed information about a container
docker inspect <container_name_or_id>

# View resource usage statistics
docker stats <container_name_or_id>
```

## Part 2: Kubernetes Orchestration

Kubernetes takes containerization to the next level by providing orchestration capabilities. While Docker handles individual containers, Kubernetes manages clusters of containers across multiple machines, providing features like automatic scaling, load balancing, and self-healing.

### Kubernetes Installation

Let's install the tools needed to work with Kubernetes locally:

#### Install kubectl (Kubernetes CLI)

kubectl is the command-line tool for interacting with Kubernetes clusters:

```bash
# Download the latest stable version of kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make the binary executable
chmod +x kubectl

# Move it to a directory in your PATH
sudo mv kubectl /usr/local/bin/

# Verify the installation
kubectl version --client
```

#### Install Minikube (Local Kubernetes Cluster)

Minikube creates a single-node Kubernetes cluster on your local machine for development and learning:

```bash
# Download the latest Minikube binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install Minikube
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify the installation
minikube version

# Start your local Kubernetes cluster
minikube start

# Enable useful addons for enhanced functionality
minikube addons enable dashboard  # Web-based Kubernetes UI
minikube addons enable ingress    # Ingress controller for external access
```

### Kubernetes Core Concepts

Before deploying applications, let's understand the key Kubernetes concepts that make container orchestration possible:

**Pods:** The smallest deployable unit in Kubernetes. A pod typically contains one container (though it can contain multiple containers that need to work closely together). Pods are ephemeral and can be created, destroyed, and recreated as needed.

**Deployments:** Manage the lifecycle of pods. Deployments ensure that a specified number of pod replicas are running at all times. They handle rolling updates, rollbacks, and scaling operations.

**Services:** Provide stable network endpoints for accessing pods. Since pods can be created and destroyed frequently, services abstract the network layer and provide consistent access points.

**ConfigMaps:** Store configuration data separately from application code. This allows you to change application configuration without rebuilding container images.

**Namespaces:** Provide logical separation within a cluster. They help organize resources and provide scope for names and policies.

### Deploying to Kubernetes

Let's create Kubernetes manifests to deploy our containerized application:

#### Create Kubernetes Configuration

Create `k8s-deployment.yaml` for the complete Kubernetes configuration:

```yaml
---
# ConfigMap stores configuration data that can be consumed by pods
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-node-app-config
  labels:
    app: my-node-app
data:
  # Environment variables that will be injected into containers
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  MESSAGE: "Hello from Kubernetes! 🚀"

---
# Deployment manages a set of identical pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-node-app-deployment
  labels:
    app: my-node-app
    version: v1
spec:
  # Number of pod replicas to maintain
  replicas: 3
  
  # Label selector to identify which pods belong to this deployment
  selector:
    matchLabels:
      app: my-node-app
  
  # Template for creating pods
  template:
    metadata:
      labels:
        app: my-node-app
        version: v1
    spec:
      containers:
      - name: my-node-app
        image: my-node-app:latest
        imagePullPolicy: Never  # Use local image only (for Minikube)
        ports:
        - containerPort: 3000
          name: http
        
        # Environment variables
        env:
        - name: PORT
          value: "3000"
        
        # Load environment variables from ConfigMap
        envFrom:
        - configMapRef:
            name: my-node-app-config
        
        # Resource requests and limits for proper scheduling
        resources:
          requests:
            memory: "64Mi"   # Minimum memory required
            cpu: "50m"       # Minimum CPU required (50 millicores)
          limits:
            memory: "128Mi"  # Maximum memory allowed
            cpu: "100m"      # Maximum CPU allowed (100 millicores)
        
        # Health checks for container lifecycle management
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30  # Wait 30 seconds before first check
          periodSeconds: 10        # Check every 10 seconds
          failureThreshold: 3      # Restart after 3 consecutive failures
        
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5   # Wait 5 seconds before first check
          periodSeconds: 5         # Check every 5 seconds
          failureThreshold: 3      # Mark unready after 3 consecutive failures

---
# Service provides stable network access to pods
apiVersion: v1
kind: Service
metadata:
  name: my-node-app-service
  labels:
    app: my-node-app
spec:
  type: NodePort  # Exposes service on each node's IP at a static port
  ports:
  - port: 80           # Port exposed by the service
    targetPort: 3000   # Port on the container
    nodePort: 30080    # Port on each cluster node (30000-32767 range)
    protocol: TCP
    name: http
  selector:
    app: my-node-app   # Select pods with this label
```

#### Understanding Kubernetes Manifests

Let's examine each component of our Kubernetes configuration:

**ConfigMap Explanation:** ConfigMaps separate configuration from application code, making applications more portable and easier to manage. Instead of hardcoding configuration values or building them into container images, you store them in ConfigMaps and inject them into containers at runtime.

**Deployment Specification:** The deployment ensures that three replicas of your application are always running. If a pod fails or a node goes down, Kubernetes automatically creates replacement pods. The deployment also manages rolling updates, allowing you to update your application without downtime.

**Resource Management:** By specifying resource requests and limits, you help Kubernetes make intelligent scheduling decisions and prevent any single container from consuming all available resources on a node.

**Health Checks:** Liveness probes tell Kubernetes when to restart a container (if the application becomes unresponsive), while readiness probes tell Kubernetes when a container is ready to receive traffic.

**Service Configuration:** The service creates a stable endpoint for accessing your application. Even as pods are created and destroyed, the service maintains a consistent way to reach your application.

### Step-by-Step Deployment Process

#### Step 1: Prepare Your Docker Image for Minikube

Minikube runs in its own environment, so we need to make our Docker image available to it:

```bash
# Configure your shell to use Minikube's Docker daemon
eval $(minikube docker-env)

# Build your Docker image in Minikube's environment
docker build -t my-node-app:latest .

# Verify the image is available in Minikube
docker images | grep my-node-app
```

#### Step 2: Deploy to Kubernetes

```bash
# Apply the Kubernetes configuration
kubectl apply -f k8s-deployment.yaml

# Watch the deployment progress
kubectl get deployments -w

# Check pod status (press Ctrl+C to stop watching)
kubectl get pods -w
```

#### Step 3: Verify Your Deployment

```bash
# Check all resources created
kubectl get all

# Get detailed information about the deployment
kubectl describe deployment my-node-app-deployment

# Check service details
kubectl describe service my-node-app-service

# Verify pods are healthy and ready
kubectl get pods -o wide
```

#### Step 4: Access Your Application

```bash
# Get Minikube IP address
minikube ip

# Access your application using the NodePort
# Replace <minikube-ip> with the actual IP from the previous command
curl http://<minikube-ip>:30080

# Alternative: Use Minikube's service command (opens in browser)
minikube service my-node-app-service

# For testing, you can also use port forwarding
kubectl port-forward service/my-node-app-service 8080:80
# Then access http://localhost:8080
```

### Kubernetes Commands Reference

Here are the essential Kubernetes commands organized by function:

#### Cluster Management

```bash
# Get cluster information
kubectl cluster-info

# Check cluster status
kubectl get componentstatuses

# View cluster nodes
kubectl get nodes

# Get detailed node information
kubectl describe nodes
```

#### Application Management

```bash
# Deploy applications from YAML files
kubectl apply -f <filename>.yaml

# Update existing resources
kubectl apply -f <filename>.yaml

# Delete resources defined in a file
kubectl delete -f <filename>.yaml

# View all resources in the current namespace
kubectl get all

# Get resources with additional details
kubectl get all -o wide
```

#### Pod Management

```bash
# List all pods
kubectl get pods

# Get pods with additional information
kubectl get pods -o wide

# Watch pods in real-time
kubectl get pods -w

# Describe a specific pod
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>

# Follow logs in real-time
kubectl logs -f <pod-name>

# Execute commands in a pod
kubectl exec -it <pod-name> -- /bin/sh
```

#### Service and Network Management

```bash
# List all services
kubectl get services

# Describe a service
kubectl describe service <service-name>

# Forward a local port to a pod
kubectl port-forward pod/<pod-name> <local-port>:<pod-port>

# Forward a local port to a service
kubectl port-forward service/<service-name> <local-port>:<service-port>
```

#### Scaling and Updates

```bash
# Scale a deployment
kubectl scale deployment <deployment-name> --replicas=<number>

# Update a deployment's image
kubectl set image deployment/<deployment-name> <container-name>=<new-image>

# Check rollout status
kubectl rollout status deployment/<deployment-name>

# View rollout history
kubectl rollout history deployment/<deployment-name>

# Rollback to previous version
kubectl rollout undo deployment/<deployment-name>
```

### Testing Your Kubernetes Deployment

Let's verify that everything is working correctly:

#### Test Basic Functionality

```bash
# Check that all pods are running
kubectl get pods
# All pods should show STATUS as "Running"

# Test the health endpoint
kubectl port-forward service/my-node-app-service 8080:80 &
curl http://localhost:8080/health
# Should return JSON with health status

# Stop the port forward
kill %1
```

#### Test Scaling

```bash
# Scale up to 5 replicas
kubectl scale deployment my-node-app-deployment --replicas=5

# Watch the new pods being created
kubectl get pods -w

# Scale back down to 3 replicas
kubectl scale deployment my-node-app-deployment --replicas=3
```

#### Test Rolling Updates

```bash
# First, modify your app.js to change the response message
# Then rebuild the image with a new tag
eval $(minikube docker-env)
docker build -t my-node-app:v2 .

# Update the deployment to use the new image
kubectl set image deployment/my-node-app-deployment my-node-app=my-node-app:v2

# Watch the rolling update process
kubectl rollout status deployment/my-node-app-deployment

# Verify the update worked
minikube service my-node-app-service
```

### Troubleshooting

Common issues you might encounter and how to resolve them:

#### Minikube Issues

**Minikube won't start:**
```bash
# Check Minikube status
minikube status

# If services are stopped, try restarting
minikube stop
minikube start

# If that doesn't work, delete and recreate the cluster
minikube delete
minikube start --driver=docker --memory=4096 --cpus=2
```

**Docker daemon issues:**
```bash
# Verify you're using Minikube's Docker daemon
echo $DOCKER_HOST
# Should show something like tcp://192.168.49.2:2376

# If not set, run:
eval $(minikube docker-env)
```

#### Pod Issues

**ImagePullBackOff errors:**
```bash
# This usually means the image isn't available in Minikube
# Make sure you built the image in Minikube's environment:
eval $(minikube docker-env)
docker build -t my-node-app:latest .

# Or check if imagePullPolicy is set to Never in your YAML
```

**Pods not ready:**
```bash
# Check pod details
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>

# Common issues:
# - Application not listening on the expected port
# - Health check endpoints not working
# - Insufficient resources
```

#### Service Access Issues

**Can't access the service:**
```bash
# Verify service is running
kubectl get services

# Check if service selector matches pod labels
kubectl describe service my-node-app-service
kubectl get pods --show-labels

# Test service connectivity from within the cluster
kubectl run debug --image=busybox -it --rm -- /bin/sh
# Then from inside the pod:
wget -qO- http://my-node-app-service/
```

## Advanced Topics

### Working with Namespaces

Namespaces provide logical separation within a Kubernetes cluster:

```bash
# Create a new namespace
kubectl create namespace development

# Deploy to a specific namespace
kubectl apply -f k8s-deployment.yaml -n development

# List resources in a namespace
kubectl get all -n development

# Set default namespace
kubectl config set-context --current --namespace=development
```

### Using Secrets for Sensitive Data

For sensitive configuration like database passwords:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  # Base64 encoded values
  database-password: <base64-encoded-password>
---
# Reference in deployment
spec:
  containers:
  - name: my-node-app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: database-password
```

### Horizontal Pod Autoscaling

Automatically scale based on CPU usage:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-node-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-node-app-deployment
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Cleanup and Uninstallation

### Cleaning Up Kubernetes Resources

```bash
# Delete all resources created by your deployment
kubectl delete -f k8s-deployment.yaml

# Or delete specific resources
kubectl delete deployment my-node-app-deployment
kubectl delete service my-node-app-service
kubectl delete configmap my-node-app-config

# Clean up completed pods and other resources
kubectl delete pods --field-selector=status.phase=Succeeded
```

### Removing Docker Images and Containers

```bash
# Stop and remove all containers
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)

# Remove your application images
docker rmi my-node-app:latest my-node-app:v2

# Clean up unused resources
docker system prune -a
```

### Uninstalling Minikube and kubectl

```bash
# Stop and delete Minikube cluster
minikube stop
minikube delete

# Remove Minikube binary and configuration
sudo rm /usr/local/bin/minikube
rm -rf ~/.minikube

# Remove kubectl binary and configuration
sudo rm /usr/local/bin/kubectl
rm -rf ~/.kube

# Remove cached files
rm -rf ~/.kube/cache
```

### Uninstalling Docker (if needed)

```bash
# Remove Docker packages
sudo apt remove docker.io

# Remove Docker data
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker

# Remove user from docker group
sudo deluser $USER docker
```

## Best Practices and Next Steps

### Docker Best Practices

- Use specific image tags instead of `latest` in production
- Minimize image layers by combining RUN commands
- Use multi-stage builds for smaller production images
- Don't run containers as root
- Keep images up to date with security patches

### Kubernetes Best Practices

- Always define resource requests and limits
- Use namespaces to organize resources
- Implement proper health checks
- Use ConfigMaps and Secrets for configuration
- Set up monitoring and logging
- Plan for backup and disaster recovery

### Learning Path

1. **Master the Basics:** Practice creating different types of applications with Docker
2. **Explore Networking:** Learn about Docker networks and Kubernetes networking
3. **Study Storage:** Understand persistent volumes and storage classes
4. **Learn Security:** Implement RBAC, network policies, and security contexts
5. **Monitoring and Logging:** Set up Prometheus, Grafana, and centralized logging
6. **Production Readiness:** Learn about managed Kubernetes services (EKS, GKE, AKS)

## Conclusion

You've now learned the fundamentals of containerizing applications with Docker and orchestrating them with Kubernetes. This foundation will serve you well as you continue to explore the world of modern application deployment and management.

The combination of Docker and Kubernetes provides a powerful platform for building, deploying, and managing applications at scale. As you continue your journey, remember that these tools are constantly evolving, so stay curious and keep learning!

## Additional Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Docker Hub](https://hub.docker.com/) - Repository of container images
- [Kubernetes Interactive Tutorials](https://kubernetes.io/docs/tutorials/)
- [Play with Docker](https://labs.play-with-docker.com/) - Browser-based Docker playground
- [Play with Kubernetes](https://labs.play-with-k8s.com/) - Browser-based Kubernetes playground

---

**Happy containerizing and orchestrating! 🚀**
