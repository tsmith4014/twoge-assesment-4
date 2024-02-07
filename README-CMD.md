## Table of Contents

1. [Introduction](#introduction)
2. [Docker and Docker Compose Deployment](#docker-and-docker-compose-deployment)
   - [Building and Running with Docker Compose](#building-and-running-with-docker-compose)
   - [Building for Multiple Architectures and Pushing to Docker Hub](#building-for-multiple-architectures-and-pushing-to-docker-hub)
3. [Kubernetes Deployment Using Minikube](#kubernetes-deployment-using-minikube)
   - [Initial Setup](#initial-setup)
   - [Configuring Database and Application](#configuring-database-and-application)
   - [Verifying and Accessing the Deployment](#verifying-and-accessing-the-deployment)
   - [Updating Service Type](#updating-service-type)

---

# Comprehensive Deployment Guide for Docker and Kubernetes

This guide encompasses all necessary steps to deploy a Flask web application with a PostgreSQL database using Docker, Docker Compose, and Kubernetes, including building images for multiple architectures and pushing them to Docker Hub for use in Kubernetes deployments.

## Docker and Docker Compose Deployment

### Building and Running with Docker Compose

1. **Build Docker images** for both the application and database:

   ```bash
   docker-compose build
   ```

   This command constructs Docker images as specified in the `Dockerfile` for the application and settings in the `docker-compose.yml`.

2. **Start the services** defined in `docker-compose.yml`:

   ```bash
   docker-compose up
   ```

   Launches the application and database containers, applying all defined settings such as network and volume configurations.

3. **Teardown the setup** when needed, to stop and remove containers, networks, and volumes created by `up`:
   ```bash
   docker-compose down
   ```

### Building for Multiple Architectures and Pushing to Docker Hub

1. **Log in to Docker Hub** (if not already logged in):

   ```bash
   docker login
   ```

   This step is necessary to push images to your Docker Hub repository.

2. **Build and push the Docker image** for multiple architectures using Docker's buildx tool:
   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 -t <your-dockerhub-username>/twoge-kube --push .
   ```
   This command builds the Docker image for both `amd64` and `arm64` architectures and pushes the built images to Docker Hub. Replace `<your-dockerhub-username>` with your actual Docker Hub username.

## Kubernetes Deployment Using Minikube

### Initial Setup

1. **Create a namespace** to isolate the deployment within the cluster:

   ```bash
   kubectl create namespace vega-namespace
   ```

2. **Create a resource quota** within the namespace to manage resource allocation:
   ```bash
   kubectl create quota my-quota --hard=pods=10,services=5 --namespace=vega-namespace
   ```

### Configuring Database and Application

1. **Apply the database secret** for secure storage of sensitive information:

   ```bash
   kubectl apply -f db-secret.yml
   ```

2. **Apply the database ConfigMap** to configure the database:

   ```bash
   kubectl apply -f db-configmap.yml
   ```

3. **Deploy the database** with its deployment and service configurations:

   ```bash
   kubectl apply -f db-deployment.yml
   kubectl apply -f db-service.yml
   ```

4. **Deploy the application** by applying its deployment and service manifests:
   ```bash
   kubectl apply -f app-deployment.yml
   kubectl apply -f app-service.yml
   ```

### Verifying and Accessing the Deployment

1. **Check the status of services** to ensure they are running and accessible:

   ```bash
   kubectl get services -n vega-namespace
   ```

2. **Check the status of deployments** to confirm successful deployment:

   ```bash
   kubectl get deployments -n vega-namespace
   ```

3. **Forward a local port to the application service** for local testing, making the application accessible at `http://localhost:8080`:
   ```bash
   kubectl port-forward svc/twoge-kube-service 8080:80 -n vega-namespace
   ```

### Updating Service Type

In case you need to change the service type (for example, from LoadBalancer to ClusterIP for internal access):

1. **Update the `app-service.yml` file** to reflect the new service type.

2. **Apply the updated service configuration**:

   ```bash
   kubectl apply -f app-service.yml
   ```

   This command updates the service in the cluster to the new configuration.

3. **Confirm the change** by checking the service status again:
   ```bash
   kubectl get services -n vega-namespace
   ```
