# Kubernetes and Docker Files - Table of Contents

## Table of Contents

1. [Introduction](#introduction)
2. [Docker Deployment](#docker-deployment)
   - [Docker Compose Configuration](#docker-compose-configuration)
   - [Dockerfile for Flask Application](#dockerfile-for-flask-application)
   - [Environment Variables](#environment-variables)
3. [Kubernetes Deployment](#kubernetes-deployment)
   - [Namespace and Resource Quotas](#namespace-and-resource-quotas)
   - [Application and Database Deployment](#application-and-database-deployment)
   - [ConfigMap and Secret for Database Credentials](#configmap-and-secret-for-database-credentials)

---

This document provides a detailed guide for deploying a Flask-based web application with a PostgreSQL database using Docker, Docker Compose, and Kubernetes. It includes necessary configuration files, Dockerfiles, and Kubernetes manifests required for the setup. The focus is on the DevOps aspect, covering both containerized and orchestrated environments.

## Docker Deployment

### Docker Compose Configuration

The `docker-compose.yml` file orchestrates the creation of two services: a PostgreSQL database (`db`) and a Flask application (`app`). Environment variables are managed through a `.env` file.

**docker-compose.yml**

```yaml
version: "3.8"

services:
  db:
    image: postgres:latest
    container_name: twoge-database-server
    environment:
      POSTGRES_DB: ${DB_DATABASE}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "${DB_PORT}:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init_sql_scripts:/docker-entrypoint-initdb.d
    networks:
      - twoge-app-network
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "admin", "-d", "twoge_db"]
      interval: 10s
      timeout: 5s
      retries: 3
    env_file: .env

  app:
    build: .
    restart: always
    container_name: twoge-app
    ports:
      - "${PORT_BACKEND}:8080"
    networks:
      - twoge-app-network
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}:${DB_DATABASE}
      PORT: ${PORT_BACKEND}
    depends_on:
      db:
        condition: service_healthy
    env_file: .env

networks:
  twoge-app-network:
    driver: bridge

volumes:
  db-data:
```

### Dockerfile for Flask Application

**Dockerfile**

```dockerfile
FROM python:alpine

RUN apk update && \
    apk add --no-cache build-base libffi-dev openssl-dev
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8080
CMD ["python", "app.py"]
```

### Environment Variables

**.env Example**

```
DB_DATABASE=twoge_db
DB_USER=admin
DB_PASSWORD=your_password
DB_PORT=5432
PORT_BACKEND=8080
DB_HOST=db
```

## Kubernetes Deployment

### Namespace and Resource Quotas

**Namespace**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vega-namespace
```

**Resource Quota**

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: vega-quota
  namespace: vega-namespace
spec:
  hard:
    pods: "10"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

### Application and Database Deployment

**Database Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
  namespace: vega-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:latest
          env:
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_DATABASE
          ports:
            - containerPort: 5432
```

**Application Deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twoge-kube-deployment
  namespace: vega-namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: twoge-kube
  template:
    metadata:
      labels:
        app: twoge-kube
    spec:
      containers:
        - name: twoge-kube
          image: tsmith4014/twoge-kube
          ports:
            - containerPort: 8080
          env:
            - name: DB_USER
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_HOST
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_PORT
            - name: DB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: DB_DATABASE
```

### ConfigMap and Secret for Database Credentials

**ConfigMap**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
  namespace: vega-namespace
data:
  DB_USER: admin
  DB_HOST: db
  DB_PORT: "5432"
  DB_DATABASE: twoge_db
```

**Secret**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: vega-namespace
type: Opaque
data:
  DB_PASSWORD: <base64-encoded-password>
```

---
