# Kubernetes Assignment 1 — Three-Service App

A production-style Kubernetes deployment demonstrating core concepts:
ConfigMaps, Secrets, persistent storage, health probes, rolling updates,
Ingress routing, Jobs, CronJobs, and fault-injection debug drills.

---

## Architecture

```
                        ┌─────────────────────────────────┐
                        │   Ingress (app.local)            │
                        │   /api/*    → backend:9898       │
                        │   /whoami/* → whoami:80          │
                        │   /*        → nginx:80           │
                        └────┬──────────────┬─────────────┘
                             │              │
              ┌──────────────┘              └──────────────┐
              ▼                                            ▼
   ┌─────────────────────┐                    ┌──────────────────┐
   │   Nginx (×2)        │                    │  Whoami (×2)     │
   │   nginx:1.27-alpine │                    │  traefik/whoami  │
   │   Port 80           │                    │  Port 80         │
   │   - Reverse proxy   │                    │  Echoes headers  │
   │   - Static HTML     │                    └──────────────────┘
   └──────────┬──────────┘
              │ proxy_pass /api/
              ▼
   ┌─────────────────────┐
   │   Backend (×2)      │
   │   podinfo:6.7.1     │
   │   Port 9898         │
   │   - REST API        │
   │   - Web UI          │
   │   - Fault endpoints │
   └──────────┬──────────┘
              │ tcp://redis:6379
              ▼
   ┌─────────────────────┐
   │   Redis (×1)        │
   │   redis:7-alpine    │
   │   Port 6379         │
   │   - Cache store     │
   └─────────────────────┘
```

---

## Project Structure

```
kubernetes_assigment_1/
├── kind-config.yaml               # KIND cluster config (port mappings for Ingress)
│
├── manifests/
│   ├── base/                      # Core application resources
│   │   ├── namespace.yaml         # Namespace: assign1
│   │   ├── secret.yaml            # Secret: api-key, db-password
│   │   ├── configmap-backend.yaml # ConfigMap: podinfo env vars (immutable)
│   │   ├── configmap-nginx.yaml   # ConfigMap: nginx routing config
│   │   ├── configmap-static.yaml  # ConfigMap: static HTML page
│   │   ├── redis-deployment.yaml  # Redis Deployment (emptyDir by default)
│   │   ├── redis-service.yaml     # Redis ClusterIP Service
│   │   ├── redis-pvc.yaml         # PVC for Redis persistence (Task 9)
│   │   ├── backend-deployment.yaml# podinfo Deployment (probes, resources)
│   │   ├── backend-service.yaml   # podinfo ClusterIP Service
│   │   ├── whoami.yaml            # Whoami Deployment + Service
│   │   └── nginx.yaml             # Nginx Deployment + Service
│   │
│   ├── ingress/
│   │   └── ingress.yaml           # Ingress rules for app.local
│   │
│   └── jobs/
│       ├── job.yaml               # One-off curl Job
│       └── cronjob.yaml           # Per-minute curl CronJob
│
├── README.md                      # Original notes (learning journal)
├── ASSIGNMENT.md                  # This file — submission guide
├── TROUBLESHOOTING.md             # Known issues and fixes
└── KUBECTL_COMMANDS.md            # Command reference guide
```

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Docker | 24+ | https://docs.docker.com/get-docker |
| KIND | 0.23+ | `go install sigs.k8s.io/kind@latest` |
| kubectl | 1.29+ | https://kubernetes.io/docs/tasks/tools |

---

## Quick Start

### 1. Create the KIND Cluster

```bash
kind create cluster --config kind-config.yaml
```

Expected output:
```
✓ Ensuring node image (kindest/node:v1.33.1)
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
Set kubectl context to "kind-my-cluster"
```

### 2. Install Nginx Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait until the controller pod is ready (~60s)
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 3. Add Local DNS Entry

```bash
echo "127.0.0.1 app.local" | sudo tee -a /etc/hosts
```

Verify:
```bash
grep app.local /etc/hosts
# 127.0.0.1 app.local
```

### 4. Deploy All Resources (Order Matters)

```bash
# Namespace first — everything else goes inside it
kubectl apply -f manifests/base/namespace.yaml
kubectl config set-context --current --namespace=assign1

# Secrets and ConfigMaps — must exist before Deployments that reference them
kubectl apply -f manifests/base/secret.yaml
kubectl apply -f manifests/base/configmap-backend.yaml
kubectl apply -f manifests/base/configmap-nginx.yaml
kubectl apply -f manifests/base/configmap-static.yaml

# Workloads — Services before Deployments
kubectl apply -f manifests/base/redis-service.yaml
kubectl apply -f manifests/base/redis-deployment.yaml
kubectl apply -f manifests/base/backend-service.yaml
kubectl apply -f manifests/base/backend-deployment.yaml
kubectl apply -f manifests/base/whoami.yaml
kubectl apply -f manifests/base/nginx.yaml

# Ingress
kubectl apply -f manifests/ingress/ingress.yaml

# Batch jobs
kubectl apply -f manifests/jobs/job.yaml
kubectl apply -f manifests/jobs/cronjob.yaml
```

### 5. Verify All Pods are Running

```bash
kubectl get all -n assign1
```

Expected:
```
NAME                           READY   STATUS    RESTARTS
pod/backend-xxx                1/1     Running   0
pod/backend-yyy                1/1     Running   0
pod/nginx-xxx                  1/1     Running   0
pod/nginx-yyy                  1/1     Running   0
pod/redis-xxx                  1/1     Running   0
pod/whoami-xxx                 1/1     Running   0
pod/whoami-yyy                 1/1     Running   0

NAME              TYPE        CLUSTER-IP     PORT(S)
service/backend   ClusterIP   10.96.x.x      9898/TCP
service/nginx     ClusterIP   10.96.x.x      80/TCP
service/redis     ClusterIP   10.96.x.x      6379/TCP
service/whoami    ClusterIP   10.96.x.x      80/TCP

NAME                      READY   UP-TO-DATE   AVAILABLE
deployment.apps/backend   2/2     2            2
deployment.apps/nginx     2/2     2            2
deployment.apps/redis     1/1     1            1
deployment.apps/whoami    2/2     2            2
```

---

## Verify Each Route

```bash
# Static page (served by Nginx)
curl http://app.local/

# Backend API — direct via Ingress (bypasses Nginx proxy)
curl http://app.local/api/version

# Whoami service — direct via Ingress
curl http://app.local/whoami/

# Redis cache: write then read
curl -X POST http://app.local/api/cache/mykey -d "myvalue"
curl http://app.local/api/cache/mykey
# Expected: myvalue
```

---

## Task Walkthroughs

### Task 7 — Config Change + Immutable ConfigMap

```bash
# configmap-backend.yaml has immutable: true
# Direct edit is blocked:
kubectl edit configmap podinfo-config -n assign1
# Error: field is immutable when `immutable` is set

# Correct workflow — delete, edit file, re-apply, restart
kubectl delete configmap podinfo-config -n assign1

# Edit PODINFO_UI_MESSAGE in manifests/base/configmap-backend.yaml
kubectl apply -f manifests/base/configmap-backend.yaml
kubectl rollout restart deployment backend -n assign1
kubectl rollout status deployment backend -n assign1

# Verify new message appears
curl http://app.local/api/
```

---

### Task 8 — Rolling Update and Rollback

```bash
# Current revision history
kubectl rollout history deployment backend -n assign1

# Update to a valid new tag
kubectl set image deployment/backend \
  podinfo=ghcr.io/stefanprodan/podinfo:6.6.0 -n assign1
kubectl rollout status deployment backend -n assign1

# Simulate bad image tag — rollout stalls with ImagePullBackOff
kubectl set image deployment/backend \
  podinfo=ghcr.io/stefanprodan/podinfo:99.99.99 -n assign1

# Watch stalled rollout
kubectl get pods -l app=backend -n assign1

# Rollback to previous working revision
kubectl rollout undo deployment backend -n assign1
kubectl rollout status deployment backend -n assign1

# Confirm revisionHistoryLimit: 3 — max 3 revisions kept
kubectl rollout history deployment backend -n assign1
```

---

### Task 9 — Redis Persistence (emptyDir vs PVC)

**Demonstrate data loss with emptyDir:**

```bash
# Write to cache
curl -X POST http://app.local/api/cache/testkey -d "will-this-survive?"

# Delete the Redis pod — Deployment recreates it automatically
kubectl delete pod -l app=redis -n assign1
kubectl get pods -l app=redis -n assign1 -w   # wait for Running

# Read — data is gone (emptyDir wiped on pod restart)
curl http://app.local/api/cache/testkey
# Expected: 404
```

**Switch to PVC for persistence:**

```bash
# Apply PVC
kubectl apply -f manifests/base/redis-pvc.yaml
kubectl get pvc redis-pvc -n assign1

# Edit manifests/base/redis-deployment.yaml
# Replace the volumes section:
#
#   volumes:
#   - name: redis-data
#     persistentVolumeClaim:
#       claimName: redis-pvc
#
kubectl apply -f manifests/base/redis-deployment.yaml

# Write, delete pod, read — data survives
curl -X POST http://app.local/api/cache/testkey -d "now-persistent!"
kubectl delete pod -l app=redis -n assign1
kubectl get pods -l app=redis -n assign1 -w
curl http://app.local/api/cache/testkey
# Expected: now-persistent!
```

---

### Task 10 — Job and CronJob

```bash
# One-off Job — runs once and exits
kubectl apply -f manifests/jobs/job.yaml
kubectl get job curl-backend-once -n assign1
kubectl logs job/curl-backend-once -n assign1
# Expected: {"version":"6.7.1",...}  HTTP Status: 200

# CronJob — runs every minute, keeps 2 successful Job histories
kubectl apply -f manifests/jobs/cronjob.yaml
kubectl get cronjob curl-backend-every-minute -n assign1

# Watch Jobs being created each minute
kubectl get jobs -n assign1 -w

# View a specific run's logs
kubectl logs job/<job-name> -n assign1

# Suspend CronJob when done testing
kubectl patch cronjob curl-backend-every-minute -n assign1 \
  -p '{"spec":{"suspend":true}}'
```

---

### Task 11 — Debug Drills

#### Drill 1: Readiness Probe Failure

```bash
POD=$(kubectl get pod -l app=backend -n assign1 \
  -o jsonpath='{.items[0].metadata.name}')

# Disable readiness on one pod
kubectl exec -it $POD -n assign1 -- \
  curl -X POST http://localhost:9898/readyz/disable

# Pod shows 0/1 Running — alive but removed from Service endpoints
kubectl get pods -l app=backend -n assign1

# Confirm the disabled pod's IP is absent from endpoints
kubectl get endpoints backend -n assign1
kubectl describe pod $POD -n assign1 | grep -i "warning\|readiness"

# Re-enable readiness
kubectl exec -it $POD -n assign1 -- \
  curl -X POST http://localhost:9898/readyz/enable
```

#### Drill 2: Panic and Restart Count

```bash
POD=$(kubectl get pod -l app=backend -n assign1 \
  -o jsonpath='{.items[0].metadata.name}')

# Trigger panic — container crashes immediately
kubectl exec -it $POD -n assign1 -- curl http://localhost:9898/panic

# RESTARTS counter increments
kubectl get pods -l app=backend -n assign1

# Read crashed container's logs (previous container instance)
kubectl logs $POD -n assign1 --previous

# Full event timeline
kubectl describe pod $POD -n assign1
kubectl get events -n assign1 --sort-by='.lastTimestamp' | tail -10
```

#### Drill 3: Ephemeral Debug Container

```bash
POD=$(kubectl get pod -l app=backend -n assign1 \
  -o jsonpath='{.items[0].metadata.name}')

# Inject a busybox container into the running pod
kubectl debug -it $POD \
  --image=busybox:1.36 \
  --target=podinfo \
  -n assign1

# Inside the debug container:
wget -qO- http://localhost:9898/version   # verify app responds
nslookup redis                            # test Redis DNS resolution
nslookup backend                          # test backend DNS
exitharaz-sony@sharazlab:~/3_assigment_kubernetes$ kubectl debug -it $POD \
  --image=busybox:1.36 \
  --target=podinfo \
  -n assign1
Targeting container "podinfo". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
Defaulting debug container name to debugger-c5twp.
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
/ # 
/ # 
/ # exit
Session ended, the ephemeral container will not be restarted but may be reattached using 'kubectl attach backend-5998bb6fbf-6ph9h -c debugger-c5twp -n assign1 -i -t' if it is still running
```

#### Drill 4: No Failed Requests Under Pod Deletion

```bash
# Terminal 1 — continuous request loop
while true; do
  curl -s -o /dev/null -w "%{http_code} " http://app.local/api/version
  sleep 0.3
done

# Terminal 2 — kill a backend pod
kubectl delete pod -l app=backend -n assign1

# Terminal 1 output should remain: 200 200 200 200 200 (no failures)
# Guaranteed by: maxUnavailable: 0 + readinessProbe
```

---

## Key Concepts Reference

| Concept | Resource | File |
|---------|----------|------|
| Namespace isolation | Namespace | `base/namespace.yaml` |
| Non-sensitive config via env | ConfigMap | `base/configmap-backend.yaml` |
| Immutable ConfigMap | ConfigMap | `base/configmap-backend.yaml` |
| Config as mounted file | ConfigMap (volume + subPath) | `base/configmap-nginx.yaml` |
| Static HTML content | ConfigMap (volume) | `base/configmap-static.yaml` |
| Sensitive data — env + file | Secret | `base/secret.yaml` |
| Ephemeral storage | emptyDir | `base/redis-deployment.yaml` |
| Persistent storage | PVC | `base/redis-pvc.yaml` |
| Startup / Liveness / Readiness | Probes | `base/backend-deployment.yaml` |
| CPU and memory limits | resources | All Deployments |
| Zero-downtime deploy | RollingUpdate maxUnavailable:0 | `base/backend-deployment.yaml` |
| Rollback history | revisionHistoryLimit: 3 | `base/backend-deployment.yaml` |
| Internal DNS routing | ClusterIP Service | All Service files |
| External path routing | Ingress | `ingress/ingress.yaml` |
| One-off task | Job | `jobs/job.yaml` |
| Scheduled task | CronJob | `jobs/cronjob.yaml` |
| Live debug without restart | Ephemeral container | Task 11 drills |

---

## Teardown

```bash
# Remove all application resources
kubectl delete namespace assign1

# Remove the cluster
kind delete cluster --name my-cluster

# Remove the hosts entry (optional)
sudo sed -i '/app.local/d' /etc/hosts
```

---

## Environment

| Item | Value |
|------|-------|
| Kubernetes | v1.33.1 (KIND) |
| Cluster | my-cluster |
| Namespace | assign1 |
| Ingress host | app.local |
| Backend | ghcr.io/stefanprodan/podinfo:6.7.1 |
| Cache | redis:7-alpine |
| Proxy | nginx:1.27-alpine |
| Whoami | traefik/whoami:latest |
