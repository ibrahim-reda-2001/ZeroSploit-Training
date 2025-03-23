# 🌐 IRIS-Web Deployment Guide  
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)

## 📋 Table of Contents
- [Getting Started](#-getting-started)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Kubernetes Deployment](#-kubernetes-deployment)
  - [Persistent Storage](#1-persistent-storage)
  - [Database Setup](#2-database-setup)
  - [RabbitMQ Setup](#3-rabbitmq-setup)
  - [Application Deployment](#4-application-deployment)
  - [Worker Deployment](#5-worker-deployment)
  - [Ingress Configuration](#6-ingress-configuration)
- [Accessing Credentials](#-accessing-credentials)
- [Troubleshooting](#-troubleshooting)

---

## 🚀 Getting Started

This guide provides complete instructions for deploying IRIS-Web using both **Docker** and **Kubernetes**.

---

## 🛠️ Prerequisites

- **Docker** ([Install Guide](https://docs.docker.com/get-docker/))
- **Docker Compose** ([Install Guide](https://docs.docker.com/compose/install/))
- **Kubernetes Cluster**
- **kubectl** configured for your cluster
- **Persistent Storage** (Rook-CephFS in this example)

---

## 📦 Installation

### Clone Repository
```bash
git clone https://github.com/dfir-iris/iris-web.git
cd iris-web
```

### Build Docker Images
```bash
docker-compose build
```

### Verify Built Images
```bash
docker images
```

**Expected Output:**
```bash
REPOSITORY                           TAG                   IMAGE ID       CREATED        SIZE
ghcr.io/dfir-iris/iriswebapp_app     latest                7ff80d407da7   3 weeks ago    1.31GB
ghcr.io/dfir-iris/iriswebapp_db      latest                b9020d9f0ed8   3 weeks ago    266MB
ghcr.io/dfir-iris/iriswebapp_nginx   latest                bde1c8064593   3 weeks ago    212MB
rabbitmq                             3-management-alpine   7207fa51ea64   6 months ago   178MB
```

---

## ☸️ Kubernetes Deployment

### 1. Persistent Storage
```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iris-psql-claim
  namespace: ibrahimreda
spec:
  accessModes:
    - ReadWriteMany  
  resources:
    requests:
      storage: 20Gi
  storageClassName: rook-cephfs
```

---

### 2. Database Setup

#### Database Secrets
```yaml
# db-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: iris-psql-secrets
  namespace: ibrahimreda
  labels:
    site: iris
type: Opaque
data:
  POSTGRES_USER: cG9zdGdyZXM=        # postgres (base64)
  POSTGRES_PASSWORD: YWRtaW4=        # admin (base64)
  POSTGRES_ADMIN_USER: cmFwdG9y      # raptor (base64)
  POSTGRES_ADMIN_PASSWORD: YWRtaW4=  # admin (base64)
  POSTGRES_DB: aXJpc19kYg==          # iris_db (base64)
```

#### Database StatefulSet
```yaml
# db-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  namespace: ibrahimreda
  name: iris-psql-db-statefulset
  labels:
    app: iris-psql
    site: iris
spec:
  serviceName: "iris-psql-db"
  replicas: 1
  selector:
    matchLabels:
      app: iris-psql
  template:
    metadata:
      labels:
        app: iris-psql
    spec:
      containers:
      - name: iris-psql-db
        image: ghcr.io/dfir-iris/iriswebapp_db
        ports: [{ containerPort: 5432 }]
        env:
        - name: POSTGRES_USER
          valueFrom: { secretKeyRef: { name: iris-psql-secrets, key: POSTGRES_USER } }
        - name: POSTGRES_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-psql-secrets, key: POSTGRES_PASSWORD } }
        - name: POSTGRES_ADMIN_USER
          valueFrom: { secretKeyRef: { name: iris-psql-secrets, key: POSTGRES_ADMIN_USER } }
        - name: POSTGRES_ADMIN_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-psql-secrets, key: POSTGRES_ADMIN_PASSWORD } }
        - name: POSTGRES_DB
          valueFrom: { secretKeyRef: { name: iris-psql-secrets, key: POSTGRES_DB } }
        volumeMounts:
        - name: persistent-storage
          mountPath: /var/lib/postgresql/data
          subPath: psqldata
  volumeClaimTemplates:
  - metadata:
      name: persistent-storage
    spec:
      storageClassName: rook-cephfs
      accessModes: [ "ReadWriteOnce" ]
      resources: { requests: { storage: 1Gi } }
```

#### Database Service
```yaml
# db-service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: ibrahimreda
  name: iris-psql-service
  labels:
    site: iris
spec:
  clusterIP: None
  selector:
    app: iris-psql
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
```

---

### 3. RabbitMQ Setup

#### RabbitMQ Deployment
```yaml
# rabbitmq-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iris-rabbitmq-deployment
  namespace: ibrahimreda
  labels:
    app: iris-rabbitmq
    site: iris
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iris-rabbitmq
  template:
    metadata:
      labels:
        app: iris-rabbitmq
    spec:
      containers:
      - name: iris-rabbitmq
        image: rabbitmq:3-management-alpine
        ports:
        - containerPort: 5672
        - containerPort: 15672
```

#### RabbitMQ Service
```yaml
# rabbitmq-service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: ibrahimreda
  name: iris-rabbitmq-service
  labels:
    site: iris
spec:
  selector:
    app: iris-rabbitmq
  ports:
  - protocol: TCP
    port: 5672
    targetPort: 5672
  type: ClusterIP
```

---

### 4. Application Deployment

#### Application ConfigMap
```yaml
# app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-data
data:
  POSTGRES_SERVER: iris-psql-service
```

#### Application Secrets
```yaml
# app-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: iris-app-secrets
  labels:
    site: iris
type: Opaque
data:
  POSTGRES_USER: cmFwdG9y              # raptor (base64)
  POSTGRES_PASSWORD: YWRtaW4=          # admin (base64)
  POSTGRES_ADMIN_USER: cmFwdG9y        # raptor (base64)
  POSTGRES_ADMIN_PASSWORD: YWRtaW4=    # admin (base64)
  POSTGRES_PORT: NTQzMg==              # 5432 (base64)
  IRIS_SECRET_KEY: QVZlcnlTdXBlclNlY3JldEtleS1Tb05vdFRoaXNPbmU=
  IRIS_SECURITY_PASSWORD_SALT: QVJhbmRvbVNhbHQtTm90VGhpc09uZUVpdGhlcg==
```

#### Application Deployment
```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: ibrahimreda
  name: iris-app-deployment
  labels:
    site: iris
    app: iris-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iris-app
  template:
    metadata:
      labels:
        app: iris-app
    spec:
      containers:
      - name: iris-app
        image: ghcr.io/dfir-iris/iriswebapp_app
        ports: [{ containerPort: 8000 }]
        command: ['nohup', './iris-entrypoint.sh', 'iriswebapp']
        env:
        - name: POSTGRES_USER
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_USER } }
        - name: POSTGRES_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_PASSWORD } }
        - name: POSTGRES_ADMIN_USER
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_ADMIN_USER } }
        - name: POSTGRES_ADMIN_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_ADMIN_PASSWORD } }
        - name: POSTGRES_PORT
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_PORT } }
        - name: IRIS_SECRET_KEY
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: IRIS_SECRET_KEY } }
        - name: IRIS_SECURITY_PASSWORD_SALT
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: IRIS_SECURITY_PASSWORD_SALT } }
        - name: POSTGRES_SERVER
          valueFrom: { configMapKeyRef: { name: app-data, key: POSTGRES_SERVER } }
        volumeMounts:
        - name: iris-pcv
          mountPath: /home/iris/downloads
          subPath: downloads
        - name: iris-pcv
          mountPath: /home/iris/user_templates
          subPath: user_templates
        - name: iris-pcv
          mountPath: /home/iris/server_data
          subPath: server_data
        resources:
          requests: { memory: "512Mi", cpu: "500m" }
          limits: { memory: "1Gi", cpu: "1" }
      volumes:
      - name: iris-pcv
        persistentVolumeClaim: { claimName: iris-psql-claim }
```

#### Application Service
```yaml
# app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: iris-app-service
  labels:
    site: iris
spec:
  selector:
    app: iris-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: ClusterIP
```

#### Ingress Configuration
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: iris-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: iris.com
    http:
      paths: 
      - path: /
        pathType: Prefix
        backend:
          service:
            name: "iris-app-service"
            port: { number: 80 }
```

---

### 5. Worker Deployment

#### Worker Secrets
```yaml
# worker-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: iris-worker-secrets
  labels:
    site: iris
type: Opaque
data:
  POSTGRES_USER: cmFwdG9y              # raptor (base64)
  POSTGRES_PASSWORD: YWRtaW4=          # admin (base64)
  POSTGRES_ADMIN_USER: cmFwdG9y        # raptor (base64)
  POSTGRES_ADMIN_PASSWORD: YWRtaW4=    # admin (base64)
  POSTGRES_PORT: NTQzMg==              # 5432 (base64)
  IRIS_SECRET_KEY: QVZlcnlTdXBlclNlY3JldEtleS1Tb05vdFRoaXNPbmU=
  IRIS_SECURITY_PASSWORD_SALT: QVJhbmRvbVNhbHQtTm90VGhpc09uZUVpdGhlcg==
```

#### Worker ConfigMap
```yaml
# worker-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: worker-data
data:
  POSTGRES_SERVER: iris-psql-service
  CELERY_BROKER: amqp://iris-rabbitmq-service
  IRIS_WORKER: iris-worker-service
```

#### Worker Deployment
```yaml
# worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: iris-worker-deployment
  labels:
    app: iris-worker
    site: iris
spec:
  replicas: 1
  selector:
    matchLabels:
      app: iris-worker
  template:
    metadata:
      labels:
        app: iris-worker
    spec:
      containers:
      - name: iris-worker
        image: ghcr.io/dfir-iris/iriswebapp_app:latest
        command: ['./wait-for-iriswebapp.sh', 'iris-app-service', './iris-entrypoint.sh', 'iris-worker']
        env:
        - name: POSTGRES_USER
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_USER } }
        - name: POSTGRES_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_PASSWORD } }
        - name: POSTGRES_ADMIN_USER
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_ADMIN_USER } }
        - name: POSTGRES_ADMIN_PASSWORD
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_ADMIN_PASSWORD } }
        - name: POSTGRES_PORT
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: POSTGRES_PORT } }
        - name: IRIS_SECRET_KEY
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: IRIS_SECRET_KEY } }
        - name: IRIS_SECURITY_PASSWORD_SALT
          valueFrom: { secretKeyRef: { name: iris-app-secrets, key: IRIS_SECURITY_PASSWORD_SALT } }
        - name: POSTGRES_SERVER
          valueFrom: { configMapKeyRef: { name: worker-data, key: POSTGRES_SERVER } }
        - name: CELERY_BROKER
          valueFrom: { configMapKeyRef: { name: worker-data, key: CELERY_BROKER } }
        - name: IRIS_WORKER
          valueFrom: { configMapKeyRef: { name: worker-data, key: IRIS_WORKER } }
        volumeMounts:
        - name: iris-pcv
          mountPath: /home/iris/downloads
          subPath: downloads
        - name: iris-pcv
          mountPath: /home/iris/user_templates
          subPath: user_templates
        - name: iris-pcv
          mountPath: /home/iris/server_data
          subPath: server_data
      volumes:
      - name: iris-pcv
        persistentVolumeClaim: { claimName: iris-psql-claim }
```

#### Worker Service
```yaml
# worker-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: iris-worker-service
  labels:
    site: iris
spec:
  selector:
    app: iris-worker
  ports:
  - protocol: TCP
    port: 80
  type: ClusterIP
```

---

## 🔑 Accessing Credentials
```bash
kubectl logs <iris-app-pod-name> -n ibrahimreda
```

---

## 🚨 Troubleshooting
```bash
# Check pod status
kubectl get pods -n ibrahimreda

# View logs
kubectl logs -f <pod-name> -n ibrahimreda

# Describe resources
kubectl describe <resource-type>/<resource-name> -n ibrahimreda

# Verify services
kubectl get svc -n ibrahimreda
```

---

## 🎉 Deployment Complete!
Access your IRIS-Web application at:  
🌐 **http://iris.com**

**Next Steps:**
- Configure TLS for secure connections
- Scale deployments as needed
- Monitor resource usage
- Set up automated backups

> **Note**: Replace all base64-encoded values with your actual credentials using:  
> `echo -n "your_value" | base64`
