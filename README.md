# Kubernetes 3-Tier Application

A simple 3-tier application deployed on Kubernetes as a learning project to understand Kubernetes workloads, networking, service discovery, and basic application architecture.

The application consists of three tiers:

- Frontend
- Backend / API
- Database

All components are deployed inside a single Kubernetes namespace.

## Architecture

```text
                         User
                           |
                           v
                  +----------------+
                  |    Frontend    |
                  |    NodePort    |
                  +-------+--------+
                          |
                          v
                  +----------------+
                  |    Backend     |
                  |    ClusterIP   |
                  +-------+--------+
                          |
                          v
                  +----------------+
                  |    Database    |
                  |    ClusterIP   |
                  +----------------+
```

Each tier has its own ReplicaSet and Service.

```
Namespace: simple-3tier
|
+-- Frontend
|   +-- ReplicaSet
|   +-- Service (NodePort)
|
+-- Backend
|   +-- ReplicaSet
|   +-- Service (ClusterIP)
|
+-- Database
    +-- ReplicaSet
    +-- Service (ClusterIP)
    +-- Secret
```

## Tech Stack

- Kubernetes
- Docker
- Nginx
- Node.js
- PostgreSQL

## Kubernetes Resources

| Tier | Workload | Replicas | Service |
|------|----------|----------|---------|
| Frontend | ReplicaSet | 2 | NodePort |
| Backend | ReplicaSet | 2 | ClusterIP |
| Database | ReplicaSet | 1 | ClusterIP |

### Frontend

The frontend uses Nginx and runs with two replicas.

The Service uses `NodePort` so the frontend can be accessed from outside the Kubernetes cluster.

```
frontend-service
       |
       +-- frontend-pod
       |
       +-- frontend-pod
```

### Backend

The backend runs with two replicas and is exposed internally through a `ClusterIP` Service.

```
backend-service
       |
       +-- backend-pod
       |
       +-- backend-pod
```

The frontend does not need to know the IP address of each backend Pod.

It can communicate with the backend using the Kubernetes Service DNS name:

```
http://backend-service:3000
```

### Database

The database uses PostgreSQL and runs with one replica for this project.

The backend connects to PostgreSQL through:

```text
database-service:5432
```

Database credentials are stored using a Kubernetes Secret.

---

# Project Structure

```
kubernetes-3tier-app/
|
+-- README.md
|
+-- architecture/
|   +-- architecture.png
|
+-- k8s/
|   |
|   +-- namespace.yaml
|   |
|   +-- frontend/
|   |   +-- replicaset.yaml
|   |   +-- service.yaml
|   |
|   +-- backend/
|   |   +-- replicaset.yaml
|   |   +-- service.yaml
|   |
|   +-- database/
|       +-- replicaset.yaml
|       +-- service.yaml
|       +-- secret.example.yaml
|
+-- app/
|   |
|   +-- frontend/
|   |
|   +-- backend/
|
+-- scripts/
|   +-- deploy.sh
|   +-- cleanup.sh
|
+-- .gitignore
```

---

# Prerequisites

Before running the project, make sure you have:

- Docker
- Kubernetes cluster
- `kubectl`

For local development, you can use either Minikube or kind.

Check your Kubernetes setup:

```
kubectl version --client
kubectl get nodes
```

---

# Deployment

## 1. Clone the repository

```bash
git clone https://github.com/<username>/kubernetes-3tier-app.git

cd kubernetes-3tier-app
```

## 2. Create the namespace

```bash
kubectl apply -f k8s/namespace.yaml
```

Check the namespace:

```bash
kubectl get namespace
```

Expected:

```
simple-3tier
```

## 3. Deploy the frontend

```bash
kubectl apply -f k8s/frontend/
```

## 4. Deploy the backend

```bash
kubectl apply -f k8s/backend/
```

## 5. Configure the database Secret

Create the Secret file from the example:

```bash
cp k8s/database/secret.example.yaml k8s/database/secret.yaml
```

Update the values in `secret.yaml` if needed.

Then deploy the database:

```bash
kubectl apply -f k8s/database/
```

The actual `secret.yaml` should not be committed to the repository.

## 6. Check the resources

```bash
kubectl get all -n simple-3tier
```

You can also check each resource individually:

```bash
kubectl get pods -n simple-3tier
kubectl get rs -n simple-3tier
kubectl get svc -n simple-3tier
```

Example:

```text
NAME                     READY   STATUS
pod/frontend-xxxxx       1/1     Running
pod/frontend-yyyyy       1/1     Running
pod/backend-xxxxx        1/1     Running
pod/backend-yyyyy        1/1     Running
pod/database-xxxxx       1/1     Running
```

ReplicaSets:

```text
NAME        DESIRED   CURRENT   READY
frontend    2         2         2
backend     2         2         2
database    1         1         1
```

Services:

```text
NAME               TYPE        PORT
frontend-service   NodePort    80
backend-service    ClusterIP   3000
database-service   ClusterIP   5432
```

---

# How It Works

The request flow is:

```text
User
 |
 | HTTP
 v
frontend-service
 |
 +---- frontend-pod
 |
 +---- frontend-pod
 |
 | API request
 v
backend-service
 |
 +---- backend-pod
 |
 +---- backend-pod
 |
 | PostgreSQL connection
 v
database-service
 |
 +---- database-pod
```

The Services provide stable endpoints for each tier.

Pod IP addresses can change when Pods are recreated, so the application should communicate through Services instead of connecting directly to Pod IPs.

For example:

```text
Frontend
   |
   +--> backend-service:3000
                    |
                    +--> Backend Pod
                    +--> Backend Pod
```

The backend connects to the database through:

```text
database-service:5432
```

---

# Service Discovery

Kubernetes provides DNS-based service discovery.

The frontend can access the backend using:

```text
backend-service
```

The backend can access the database using:

```text
database-service
```

There is no need to hardcode the IP address of individual Pods.

This makes the application more resilient when Pods are recreated or scaled.

---

# Scaling

The frontend can be scaled from two replicas to three:

```bash
kubectl scale rs frontend --replicas=3 -n simple-3tier
```

The backend can also be scaled:

```bash
kubectl scale rs backend --replicas=3 -n simple-3tier
```

Check the ReplicaSets:

```bash
kubectl get rs -n simple-3tier
```

Example:

```text
NAME        DESIRED   CURRENT   READY
frontend    3         3         3
backend     3         3         3
database    1         1         1
```

ReplicaSet continuously tries to maintain the desired number of Pods.

For example, if one frontend Pod is deleted:

```text
Before:

frontend
|
+-- Pod 1  Running
+-- Pod 2  Running
+-- Pod 3  Running

Pod 2 is deleted
```

The ReplicaSet detects that the current number of Pods is below the desired state and creates a replacement Pod.

---

# Why ReplicaSet?

This project intentionally uses ReplicaSets directly to understand how ReplicaSets work and how Kubernetes maintains the desired number of Pods.

For example:

```yaml
spec:
  replicas: 2
```

This tells Kubernetes that two Pods should be running for the selected workload.

In a typical production environment, frontend and backend workloads would usually be managed using a `Deployment` instead of creating ReplicaSets directly.

A Deployment provides additional capabilities such as:

- Rolling updates
- Rollbacks
- Revision history
- ReplicaSet management

This project focuses on understanding the fundamentals before moving to Deployments.

---

# Kubernetes Concepts Covered

### Namespace

All application resources are isolated inside:

```
simple-3tier
```

### ReplicaSet

Maintains the desired number of Pods.

### Pod

The smallest deployable unit in Kubernetes and the place where the application containers run.

### Service

Provides a stable network endpoint for accessing Pods.

This project uses:

- `NodePort` for the frontend
- `ClusterIP` for the backend
- `ClusterIP` for the database

### Secret

Stores database credentials separately from the application configuration.

### Service Discovery

Services allow different tiers to communicate using DNS names instead of Pod IP addresses.

---

# Troubleshooting

Check all resources:

```bash
kubectl get all -n simple-3tier
```

Check Pod status:

```bash
kubectl get pods -n simple-3tier
```

Check Pod logs:

```bash
kubectl logs <pod-name> -n simple-3tier
```

Describe a Pod:

```bash
kubectl describe pod <pod-name> -n simple-3tier
```

Check ReplicaSets:

```bash
kubectl get rs -n simple-3tier
```

Check Services:

```bash
kubectl get svc -n simple-3tier
```

Check Service endpoints:

```bash
kubectl get endpoints -n simple-3tier
```

---

# Cleanup

To remove the entire application:

```bash
kubectl delete namespace simple-3tier
```

Since all resources are inside the namespace, deleting the namespace will also remove the application resources.

---

# What I Learned

This project helped me understand the relationship between Kubernetes resources:

```
Namespace
   |
   +-- ReplicaSet
   |      |
   |      +-- Pod
   |
   +-- Service
   |
   +-- Secret
```

Some of the things I practiced:

- Kubernetes namespace management
- ReplicaSet and desired state
- Pod lifecycle
- Kubernetes Services
- ClusterIP vs NodePort
- Kubernetes service discovery
- Basic Secret management
- Scaling Pods
- Basic 3-tier application architecture
- Communication between application tiers

---

# Next Steps

There are several things I want to improve after this project:

- Replace ReplicaSets with Deployments for frontend and backend
- Use StatefulSet for the database
- Add PersistentVolume and PersistentVolumeClaim
- Add ConfigMap
- Add liveness and readiness probes
- Add CPU and memory requests/limits
- Add Ingress
- Add Horizontal Pod Autoscaler
- Implement rolling updates
- Explore Helm
- Add Prometheus and Grafana monitoring

---

# Note

This is a learning project focused on understanding Kubernetes fundamentals and deploying a simple 3-tier application.

For a production environment, additional considerations would be required, especially around database persistence, secrets management, health checks, resource management, ingress, observability, backups, and workload deployment strategies.
