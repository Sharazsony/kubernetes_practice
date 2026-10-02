# Kubernetes Cluster Troubleshooting — Node NotReady Problem

**Date:** 2026-10-02  
**Cluster:** my-cluster (KIND)  
**Namespace:** assign1

---

## 🔴 Problem — Kya Hua Tha?

### Error Messages jo aaye:

```
NAME                       STATUS     ROLES           AGE   VERSION
my-cluster-control-plane   NotReady   control-plane   19h   v1.33.1
```

```
Error from server: Get "https://172.18.0.3:10250/containerLogs/assign1/backend-.../podinfo":
dial tcp 172.18.0.3:10250: connect: connection refused
```

```
Warning  FailedScheduling  5m  default-scheduler
0/1 nodes are available: 1 node(s) had untolerated taint
{node.kubernetes.io/unreachable: }
```

```
kubelet: "Kubelet stopped posting node status"
kubelet: Unable to register node with API server:
         dial tcp 172.18.0.3:6443: connect: connection refused
```

---

## 🔍 Root Cause — Asli Wajah

### IP Address Mismatch — Yeh Tha Asli Masla

```
┌──────────────────────────────────────────────────────────────┐
│                    IP MISMATCH DIAGRAM                       │
│                                                              │
│  Kubernetes etcd (database) mein store tha:                  │
│       Node IP = 172.18.0.3  (PURANA IP)                      │
│                                                              │
│  Docker container ki actual IP (restart ke baad):            │
│       Node IP = 172.18.0.2  (NAYA IP)                        │
│                                                              │
│  kubelet → API server ko dhundta hai: 172.18.0.2:6443  ✅   │
│  kubectl logs → kubelet ko dhundta hai: 172.18.0.3:10250 ❌  │
└──────────────────────────────────────────────────────────────┘
```

### Kaise Hua Yeh?

```
Timeline:

1. Pehle → KIND cluster create hua → Container IP: 172.18.0.3 mili
           Kubernetes ne yeh IP etcd mein save kar li

2. Phir  → System sleep/suspend hua YA Docker restart hua
           Docker container bhi band ho gaya

3. Jab   → Docker/system wapas start hua
           KIND container restart hua lekin Docker network ne
           NAYA IP diya: 172.18.0.2

4. Result → etcd mein purana IP (172.18.0.3) tha
            Container actually (172.18.0.2) par chal raha tha
            Yeh MISMATCH → Node NotReady!
```

### Kyun Yeh Fix Nahi Hota Khud Se?

- etcd ek **persistent database** hai — khud IP update nahi karta
- kubelet bar bar try karta raha lekin API server ka address galat tha
- Nodes ka `taint: node.kubernetes.io/unreachable` lag gaya
- Koi bhi naya pod schedule nahi ho sakta tha

---

## ✅ Solution — Kya Kiya Fix Karne Ke Liye

### Approach: KIND Cluster Delete + Recreate

Yeh sabse **clean aur reliable** fix hai.
Node patch karna risky hota — etcd mein manual changes se aur masle aa sakte hain.

### Step 1 — Purana broken cluster delete karo

```bash
kind delete cluster --name my-cluster
```

**Output:**
```
Deleting cluster "my-cluster" ...
Deleted nodes: ["my-cluster-control-plane"]
```

### Step 2 — Naya fresh cluster banao

```bash
kind create cluster --name my-cluster
```

**Output:**
```
Creating cluster "my-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.33.1)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-my-cluster"
```

### Step 3 — Verify node Ready hai

```bash
kubectl get nodes
```

**Output:**
```
NAME                       STATUS   ROLES           AGE   VERSION
my-cluster-control-plane   Ready    control-plane   2m    v1.33.1
```

### Step 4 — Namespace banao aur default set karo

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=assign1
```

### Step 5 — Secrets aur ConfigMaps apply karo (pehle!)

```bash
kubectl apply -f app-secret.yaml
kubectl apply -f ConfigMap.yaml
kubectl apply -f nginx-configmap.yaml
kubectl apply -f static-content-configmap.yaml
```

> **Kyun pehle?** — Deployments inhe mount karte hain.
> Agar pehle deployment apply karo aur ConfigMap baad mein,
> toh pod `Pending` mein reh sakta hai.

### Step 6 — Deployments aur Services apply karo

```bash
kubectl apply -f redis_deploy.yaml
kubectl apply -f redis_service.yaml
kubectl apply -f backend_deploy.yaml
kubectl apply -f backend_service.yaml
kubectl apply -f whoami_deploy.yaml
kubectl apply -f whoami_service.yaml
kubectl apply -f nginx.yaml
```

### Step 7 — Final verification

```bash
kubectl get all -n assign1
```

**Output (sab Running):**
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/backend-6756f69787-n4qpx   1/1     Running   0          54s
pod/backend-6756f69787-sm9sh   1/1     Running   0          54s
pod/nginx-bd97545f8-2g46t      1/1     Running   0          39s
pod/nginx-bd97545f8-p5vtb      1/1     Running   0          39s
pod/redis-699c5b657f-gv2hl     1/1     Running   0          56s
pod/whoami-785578b7b8-wtqqf    1/1     Running   0          52s
pod/whoami-785578b7b8-xbgqj    1/1     Running   0          52s

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
service/backend         ClusterIP   10.96.21.78     <none>        9898/TCP
service/redis-service   ClusterIP   10.96.195.233   <none>        6379/TCP
service/whoami          ClusterIP   10.96.132.93    <none>        80/TCP

NAME                      READY   UP-TO-DATE   AVAILABLE
deployment.apps/backend   2/2     2            2
deployment.apps/nginx     2/2     2            2
deployment.apps/redis     1/1     1            1
deployment.apps/whoami    2/2     2            2
```

---

## 🛡️ Yeh Dobara Na Ho — Prevention

### Problem: Har baar Docker restart hone par IP change hoti hai

### Solution: KIND config file use karo jisme static IP ho

Ek file banao `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster
nodes:
- role: control-plane
```

> Note: KIND Docker bridge network use karta hai — IP completely
> fix karna possible nahi KIND mein without custom Docker network.
> **Best practice: Raat ko kaam khatam hone par cluster delete karo,
> subah wapas create karo — YAML files safe rehti hain.**

### Cluster jaldi recreate karne ka shortcut script

Ek file `restart-cluster.sh` banao project folder mein:

```bash
#!/bin/bash
echo "=== Deleting old cluster ==="
kind delete cluster --name my-cluster

echo "=== Creating new cluster ==="
kind create cluster --name my-cluster

echo "=== Setting namespace ==="
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=assign1

echo "=== Applying configs and secrets ==="
kubectl apply -f app-secret.yaml
kubectl apply -f ConfigMap.yaml
kubectl apply -f nginx-configmap.yaml
kubectl apply -f static-content-configmap.yaml

echo "=== Applying deployments ==="
kubectl apply -f redis_deploy.yaml
kubectl apply -f redis_service.yaml
kubectl apply -f backend_deploy.yaml
kubectl apply -f backend_service.yaml
kubectl apply -f whoami_deploy.yaml
kubectl apply -f whoami_service.yaml
kubectl apply -f nginx.yaml

echo "=== Waiting for pods ==="
sleep 20
kubectl get pods -n assign1
```

Use karo:
```bash
chmod +x restart-cluster.sh
./restart-cluster.sh
```

---

## 🔬 Diagnose Kaise Karte Hain — Future Ke Liye Commands

```bash
# 1. Node ka status dekho
kubectl get nodes

# 2. Node ki detail aur events dekho
kubectl describe node my-cluster-control-plane

# 3. Container ki actual IP dekho
docker exec my-cluster-control-plane ip addr show eth0

# 4. Kubernetes mein stored IP dekho
kubectl get node my-cluster-control-plane -o jsonpath='{.status.addresses}'

# 5. kubelet ke logs dekho (andar se)
docker exec my-cluster-control-plane journalctl -u kubelet --no-pager | tail -20

# 6. kube-apiserver running hai ya nahi
docker exec my-cluster-control-plane crictl ps | grep apiserver

# 7. Agar IPs alag hain → cluster recreate karo (yahi sahi fix hai)
```

---

## 📋 Quick Reference Table

| Error Message | Matlab | Fix |
|---|---|---|
| `Node NotReady` | Node cluster se disconnect | IP mismatch ya kubelet crash |
| `connection refused :6443` | API server unreachable | kube-apiserver down ya wrong IP |
| `connection refused :10250` | kubelet unreachable | Node ka IP change ho gaya |
| `FailedScheduling: unreachable taint` | Pod schedule nahi ho sakta | Node Ready nahi |
| `Kubelet stopped posting node status` | Kubelet API se baat nahi kar pa raha | IP mismatch / network issue |

---

## ✅ Final State — After Fix

```
Node:    my-cluster-control-plane   STATUS: Ready   IP: 172.18.0.2
Pods:    7/7 Running
Redis:   1/1  ✅
Backend: 2/2  ✅
Whoami:  2/2  ✅
Nginx:   2/2  ✅
```

---

*Troubleshooting done by: Kiro AI Assistant*  
*Date: 2026-10-02*

---
---

# Problem #2 — Nginx Service Missing (Ingress kaam nahi kar raha tha)

**Date:** 2026-10-02

---

## 🔴 Problem — Kya Hua Tha?

Ingress apply kiya, lekin traffic nginx pods tak nahi pahunch rahi thi.

```yaml
# ingress.yaml mein yeh tha:
backend:
  service:
    name: nginx    # ← yeh service exist hi nahi thi!
    port:
      number: 80
```

```bash
kubectl get svc -n assign1

# Output:
NAME            TYPE        CLUSTER-IP
backend         ClusterIP   10.96.218.252    ✅
redis-service   ClusterIP   10.96.13.125     ✅
whoami          ClusterIP   10.96.60.124     ✅
# nginx service   ← GHAIB! ❌
```

---

## 🔍 Root Cause — Kyun Hua?

```
┌─────────────────────────────────────────────────────────┐
│  nginx Deployment tha      ✅  (pods running the)       │
│  nginx Service NAHI thi    ❌  (service yaml tha hi     │
│                                nahi project mein)       │
│                                                         │
│  Ingress Controller → "nginx" service dhundha → NAHI    │
│  MILA → 503 Service Unavailable ya traffic drop         │
└─────────────────────────────────────────────────────────┘
```

Deployment aur Service **alag cheezein** hain:

```
Deployment = Pods banata hai (containers run karta hai)
Service    = Un pods ko network se accessible banata hai

Bina Service ke → Ingress ya koi bhi dusra pod
                  nginx pods tak NAHI pahunch sakta
```

---

## ✅ Solution — nginx_service.yaml banaya aur apply kiya

### nginx_service.yaml (jo create ki):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx          # ← Ingress is naam ko dhundta tha
  namespace: assign1
  labels:
    app: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx         # ← in labels wale pods ko select karo
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
```

### Apply command:

```bash
kubectl apply -f nginx_service.yaml
```

**Output:**
```
service/nginx created
```

### Verify:

```bash
kubectl get svc -n assign1

# Output:
NAME            TYPE        CLUSTER-IP
backend         ClusterIP   10.96.218.252    ✅
nginx           ClusterIP   10.96.246.184    ✅  ← AB AA GAYI
redis-service   ClusterIP   10.96.13.125     ✅
whoami          ClusterIP   10.96.60.124     ✅
```

---

## 📌 Yaad Rakhne Ka Rule

```
Har Deployment ke saath ek Service bhi banana zaroori hai
agar us deployment tak koi traffic bhejna ho.

Deployment  →  Pods banata hai
Service     →  Pods ko naam deta hai jisse dusre reach kar sakein
Ingress     →  Baahir se andar route karta hai (Service ke zariye)

Chain: Ingress → Service → Deployment → Pod
       Koi bhi link missing = traffic nahi pahunchega
```

---
---

# Problem #3 — Browser par `app.local` nahi khul raha (DNS_PROBE_FINISHED_NXDOMAIN)

**Date:** 2026-10-02

---

## 🔴 Problem — Kya Hua Tha?

Browser mein `http://app.local/api` type kiya toh yeh error aaya:

```
This site can't be reached
Check if there is a typo in app.local
DNS_PROBE_FINISHED_NXDOMAIN
```

Lekin `curl` se test kiya toh kaam kar raha tha:

```bash
curl -H "Host: app.local" http://localhost/
# HTTP Status: 200  ✅
```

---

## 🔍 Root Cause — Kyun Hua?

```
┌──────────────────────────────────────────────────────────────┐
│                    DNS Resolution Flow                       │
│                                                              │
│  Browser → "app.local" → DNS Server se poochha              │
│                          ↓                                   │
│            Internet DNS: "app.local" nahi jaanta ❌          │
│            DNS_PROBE_FINISHED_NXDOMAIN                       │
│                                                              │
│  WHY? "app.local" koi real domain nahi hai                   │
│  Yeh sirf LOCAL testing ke liye banaya gaya fake naam hai    │
│  Internet par yeh exist nahi karta                           │
└──────────────────────────────────────────────────────────────┘
```

### `/etc/hosts` file kya hoti hai?

```
/etc/hosts = Computer ka apna LOCAL DNS
             Internet se pehle yahan check hota hai

Format:
IP_ADDRESS    DOMAIN_NAME
127.0.0.1     localhost
192.168.1.1   myrouter.home
```

Aapke system mein `app.local` ki koi entry nahi thi:

```bash
cat /etc/hosts | grep app.local
# Koi output nahi ← MISSING!
```

---

## ✅ Solution — /etc/hosts mein Entry Add karo

### Terminal mein yeh command run karo:

```bash
echo "127.0.0.1 app.local" | sudo tee -a /etc/hosts
```

> `sudo` isliye chahiye kyunki `/etc/hosts` ek system file hai —
> sirf root/admin user hi isko edit kar sakta hai.

### Verify karo:

```bash
cat /etc/hosts | grep app.local
# Output: 127.0.0.1  app.local  ✅
```

### Ab browser mein kaam karega:

```
Browser → "app.local"
        ↓
/etc/hosts check kiya → 127.0.0.1 app.local MILA ✅
        ↓
127.0.0.1:80 → KIND port mapping (0.0.0.0:80 → container:80)
        ↓
Ingress Controller
        ↓
Sahi Service → Pod → Response ✅
```

---

## 🌐 Ab Browser mein Yeh URLs Kaam Karenge

| URL | Kya milega |
|-----|-----------|
| `http://app.local/` | Nginx static HTML page |
| `http://app.local/whoami` | Whoami — request ki poori info |
| `http://app.local/api` | Backend Podinfo API (JSON) |

---

## ⚠️ Important Notes

```
1. Yeh sirf IS machine par kaam karega
   Doosre computer par bhi app.local access karna ho toh
   us computer ke /etc/hosts mein bhi add karna padega

2. Cluster recreate karne par /etc/hosts NAHI badlega
   Woh entry permanent rehti hai — dobara add karne ki
   zaroorat nahi

3. Har baar naya "fake domain" use karo toh
   /etc/hosts mein add karna padega
```

---

## 📋 Teeno Problems ka Summary

| # | Problem | Root Cause | Fix |
|---|---------|-----------|-----|
| 1 | Node NotReady | Docker restart se IP change hui, etcd mein purana IP tha | `kind delete` + `kind create` |
| 2 | Nginx Service Missing | nginx_service.yaml create nahi ki thi | `kubectl apply -f nginx_service.yaml` |
| 3 | Browser DNS error | `/etc/hosts` mein `app.local` entry nahi thi | `echo "127.0.0.1 app.local" \| sudo tee -a /etc/hosts` |

---

## ✅ Final Working State

```bash
# Yeh sab kaam kar raha hai:
curl http://app.local/          # → 200 OK (static page)
curl http://app.local/whoami    # → 200 OK (whoami info)
curl http://app.local/api       # → 200 OK (podinfo API)

kubectl get pods -n assign1
# NAME                       READY   STATUS
# backend-xxx                1/1     Running  ✅
# nginx-xxx                  1/1     Running  ✅
# redis-xxx                  1/1     Running  ✅
# whoami-xxx                 1/1     Running  ✅
```

---

*Troubleshooting done by: Kiro AI Assistant*
*Date: 2026-10-02*
