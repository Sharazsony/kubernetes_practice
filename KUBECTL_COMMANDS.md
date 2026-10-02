# 📚 Kubectl Commands — Mera Personal Reference Guide
> Sirf woh commands jo maine actually use ki hain — Roman Urdu + English mein

---

## 🧠 Aapka Knowledge Level (Analysis)

README aur history dekh ke yeh samjha:

```
✅ Basic resources banana (apply, create)
✅ Resources dekhna (get, describe)
✅ Rollout history aur undo samajh liya (Git jaisa!)
✅ ConfigMap, Secret, Volume mount samjha
✅ Ingress setup kiya aur debug kiya
✅ Exec se pod ke andar gaye
✅ Job aur logs samjhe
✅ KIND cluster troubleshoot kiya
⏳ CronJob abhi seekh rahe ho
⏳ Advanced networking abhi seekh rahe ho
```

---

## 📂 CATEGORY 1 — Apply / Create (Resources Banana)

---

### `kubectl apply -f file.yaml`
```bash
kubectl apply -f namespace.yaml
kubectl apply -f backend_deploy.yaml
```
```
Kya karta hai:
  YAML file padh ke Kubernetes mein resource banata hai
  Agar pehle se hai → UPDATE karta hai
  Agar nahi hai    → CREATE karta hai

Yaad rakhne ka trick:
  "apply" = laga do (create ya update dono)
```

---

### `kubectl apply -f file.yaml --dry-run=client`
```bash
kubectl apply -f redis-deployment.yaml --dry-run=client
```
```
Kya karta hai:
  Sirf CHECK karta hai ke YAML sahi hai ya nahi
  Kubernetes mein KUCH NAHI bhejta
  Errors pehle pakad lo deployment se pehle

Yaad rakhne ka trick:
  "dry-run" = rehearsal (asli show nahi)
```

---

### `kubectl delete -f file.yaml`
```bash
kubectl delete -f namespace.yaml
```
```
Kya karta hai:
  YAML mein jo resource likha hai use DELETE karta hai
  apply ka ulta kaam
```

---

### `kubectl delete <resource> <naam>`
```bash
kubectl delete configmap podinfo-config -n assign1
kubectl delete pod backend-xxx -n assign1
kubectl delete job curl-be-once -n assign1
```
```
Kya karta hai:
  Specific resource delete karo
  pod delete → Deployment wapas naya pod banata hai
  job delete → hamesha ke liye gone
```

---

### `kubectl delete pod -l app=redis`
```bash
kubectl delete pod -l app=redis
```
```
Kya karta hai:
  -l = label selector
  "redis" label wale saare pods delete karo
  Deployment wapas naye pods banata hai (self-healing test)

Yaad rakhne ka trick:
  -l = label se dhundo phir delete
```

---

## 📂 CATEGORY 2 — Get (Resources Dekhna)

---

### `kubectl get all`
```bash
kubectl get all
kubectl get all -n assign1
```
```
Kya karta hai:
  Namespace ke saare resources ek saath dikhata hai:
  pods, deployments, replicasets, services

Output example:
  NAME                    READY   STATUS    RESTARTS
  pod/backend-xxx         1/1     Running   0
  pod/nginx-xxx           1/1     Running   0
  
  NAME              TYPE        CLUSTER-IP
  service/backend   ClusterIP   10.96.x.x
```

---

### `kubectl get pods`
```bash
kubectl get pods
kubectl get pods -n assign1
kubectl get pods -o wide        # zyada detail (node, IP)
kubectl get pods -w             # watch mode (live update)
kubectl get pods -A             # saare namespaces
```
```
Kya karta hai:
  Pods ki list dikhata hai

STATUS column:
  Running        = sab theek ✅
  Pending        = schedule nahi hua ⏳
  CrashLoopBack  = bar bar crash ho raha ❌
  ImagePullBackOff = image nahi mili ❌
  Completed      = Job finish ho gaya ✅

-w (watch) = terminal mein live update aata rehta hai
             Ctrl+C se band karo
```

---

### `kubectl get nodes`
```bash
kubectl get nodes
kubectl get nodes -o wide
```
```
Kya karta hai:
  Cluster ke nodes dikhata hai

STATUS:
  Ready    = sab theek ✅
  NotReady = node problem ❌ (humare saath hua tha — IP mismatch)
```

---

### `kubectl get svc` (services)
```bash
kubectl get svc
kubectl get svc -n assign1
```
```
Kya karta hai:
  Services ki list dikhata hai

TYPE column:
  ClusterIP    = sirf cluster ke andar accessible
  NodePort     = node IP:port se accessible
  LoadBalancer = cloud load balancer (production)
```

---

### `kubectl get deployment`
```bash
kubectl get deployment backend
kubectl get deployment backend -o yaml        # poori YAML dekho
kubectl get deployment backend -o yaml | grep image  # sirf image line
```
```
Kya karta hai:
  Deployment ka status dikhata hai

READY column:
  2/2 = 2 desired, 2 running ✅
  0/2 = koi pod ready nahi ❌
```

---

### `kubectl get configmap`
```bash
kubectl get configmap podinfo-config -n assign1
kubectl get configmap podinfo-config -n assign1 -o yaml
```
```
Kya karta hai:
  ConfigMap ka content dikhata hai
  -o yaml → poora YAML format mein
```

---

### `kubectl get ingress`
```bash
kubectl get ingress -n assign1
```
```
Kya karta hai:
  Ingress rules dikhata hai
  ADDRESS column mein IP/hostname hota hai jab ready ho
```

---

### `kubectl get job`
```bash
kubectl get job curl-be-once -n assign1
kubectl get job curl-be-once -w           # watch karo
```
```
STATUS:
  Complete = kaam ho gaya ✅
  Failed   = fail ho gaya ❌

COMPLETIONS:
  1/1 = 1 mein se 1 complete ✅
  0/1 = abhi koi complete nahi
```

---

### `kubectl get ns` (namespaces)
```bash
kubectl get ns
kubectl get namespace
```
```
Kya karta hai:
  Saare namespaces dikhata hai
  ns = namespace ka short form
```

---

## 📂 CATEGORY 3 — Describe (Detail Mein Dekhna)

---

### `kubectl describe pod <naam>`
```bash
kubectl describe pod nginx-bd97545f8-7gjc4
kubectl describe pod -n assign1 backend-xxx
```
```
Kya karta hai:
  Pod ki POORI detail dikhata hai:
  - Image kaunsi hai
  - Environment variables
  - Volumes mounted hain ya nahi
  - EVENTS (sabse important!) ← problems yahan dikhti hain

Events section example:
  Warning  FailedScheduling  → pod schedule nahi hua
  Warning  BackOff           → image pull fail
  Normal   Started           → pod start ho gaya
```

---

### `kubectl describe node <naam>`
```bash
kubectl describe node my-cluster-control-plane
```
```
Kya karta hai:
  Node ki detail dikhata hai:
  - CPU/Memory kitna hai
  - Conditions (Ready/NotReady)
  - Taints (kya pods aane se rok raha hai)
  - Kaunse pods chal rahe hain

Humne use kiya tha jab: Node NotReady tha
```

---

### `kubectl describe deployment <naam>`
```bash
kubectl describe deployment backend -n assign1
```
```
Kya karta hai:
  Deployment ki poori detail:
  - Replicas kitne hain
  - RollingUpdate strategy
  - Conditions
  - Events
```

---

## 📂 CATEGORY 4 — Logs (Output Dekhna)

---

### `kubectl logs <pod>`
```bash
kubectl logs backend-xxx -n assign1
kubectl logs deploy/backend -n assign1      # deployment se
kubectl logs job/curl-be-once -n assign1    # job se
kubectl logs -f deploy/backend              # live streaming (-f = follow)
kubectl logs backend-xxx --tail=50          # sirf aakhri 50 lines
kubectl logs backend-xxx --previous         # crash se pehle ki logs
```
```
Kya karta hai:
  Pod ka STDOUT output dikhata hai
  (jo bhi container ne print kiya)

-f = follow = live update aata rehta hai (Ctrl+C se band)
--tail=50 = sirf last 50 lines
--previous = pod crash ke baad pehle wali logs

Job logs:
  kubectl logs job/curl-be-once → automatically sahi pod choose karta hai
```

---

## 📂 CATEGORY 5 — Exec (Pod Ke Andar Jana)

---

### `kubectl exec -it <pod> -- sh`
```bash
kubectl exec -it deploy/backend -n assign1 -- sh
kubectl exec -it backend-xxx -n assign1 -- sh
```
```
Kya karta hai:
  Pod ke andar interactive shell kholta hai
  
  -i = interactive (input dene do)
  -t = TTY (terminal mode)
  -- sh = sh shell kholo (ya bash, ya /bin/bash)

Andar jaake:
  env | grep PODINFO    ← env vars dekho
  curl http://localhost:9898  ← local request
  exit                  ← bahar aao
```

---

### `kubectl exec <pod> -- <command>` (bina andar gaye)
```bash
kubectl exec deploy/backend -n assign1 -- curl -s http://localhost:9898/version
kubectl exec deploy/backend -n assign1 -- env | grep PODINFO
```
```
Kya karta hai:
  Pod ke andar ek command chala ke result wapas lo
  Shell nahi khulti — sirf ek command

-- ke baad jo bhi likho woh pod ke andar chalega
```

---

## 📂 CATEGORY 6 — Rollout (Deployment Updates)

---

### `kubectl rollout status`
```bash
kubectl rollout status deployment backend -n assign1
```
```
Kya karta hai:
  Deployment update ka progress dikhata hai
  Jab tak complete na ho, wait karta hai

Output:
  Waiting for deployment "backend" rollout to finish...
  deployment "backend" successfully rolled out ✅
```

---

### `kubectl rollout history`
```bash
kubectl rollout history deployment backend
kubectl rollout history deployment backend --revision=4
```
```
Kya karta hai:
  Git log jaisa — purani versions ki list

REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>

--revision=4 → us specific revision ki detail
```

---

### `kubectl rollout undo`
```bash
kubectl rollout undo deployment backend
kubectl rollout undo deployment backend --to-revision=4
```
```
Kya karta hai:
  Git checkout jaisa — purani version par wapas jao

Important rule:
  Undo karne se purani revision NUMBER nahi aati
  Naya revision number banta hai (copy hoti hai)
  
  Revision 1,2,3,4 tha
  undo --to-revision=4 kiya
  Ab: 2,3,5 (4 gaya, naya 5 aaya jo 4 ki copy hai)
```

---

### `kubectl rollout restart`
```bash
kubectl rollout restart deployment backend -n assign1
```
```
Kya karta hai:
  Saare pods ko gracefully restart karta hai
  ConfigMap ya Secret change karne ke baad zaroori
  Zero downtime restart (RollingUpdate se)
```

---

## 📂 CATEGORY 7 — Set (Live Changes)

---

### `kubectl set image`
```bash
kubectl set image deployment/backend podinfo=ghcr.io/stefanprodan/podinfo:99.99.99
```
```
Kya karta hai:
  YAML file change kiye bina seedha cluster mein image update

IMPORTANT:
  Local YAML file NAHI badlti!
  Sirf cluster mein change hota hai
  Rollout history mein naya revision banta hai
```

---

## 📂 CATEGORY 8 — Config (Cluster Settings)

---

### `kubectl config set-context`
```bash
kubectl config set-context --current --namespace=assign1
```
```
Kya karta hai:
  Default namespace set karo
  Taake har command mein -n assign1 na likhna pade

Pehle:   kubectl get pods -n assign1
Baad:    kubectl get pods   ← same result
```

---

### `kubectl config current-context`
```bash
kubectl config current-context
```
```
Kya karta hai:
  Abhi kaunse cluster se connected ho yeh batata hai

Output:
  kind-my-cluster
```

---

## 📂 CATEGORY 9 — Port Forward (Local Testing)

---

### `kubectl port-forward`
```bash
kubectl port-forward svc/backend 9898:9898 -n assign1
kubectl port-forward deploy/backend 9898:9898 -n assign1
```
```
Kya karta hai:
  Cluster ke andar ki service ko aapki local machine par
  temporarily accessible banata hai

Format:
  LOCAL_PORT:CLUSTER_PORT

  9898:9898 matlab:
  localhost:9898 → cluster mein backend service:9898

Ctrl+C se band hota hai
Browser mein: http://localhost:9898
```

---

## 📂 CATEGORY 10 — Scale

---

### `kubectl scale`
```bash
kubectl scale deployment backend -n assign1 --replicas=0
kubectl scale deployment backend -n assign1 --replicas=3
```
```
Kya karta hai:
  Deployment ke pods ki tadaad change karo

--replicas=0  → saare pods band (maintenance mode)
--replicas=3  → 3 pods chalao
```

---

## 📂 CATEGORY 11 — Patch (Quick Edit)

---

### `kubectl patch`
```bash
kubectl patch configmap podinfo-config -n assign1 \
  --type='json' \
  -p='[{"op":"replace","path":"/data/PODINFO_UI_MESSAGE","value":"New msg"}]'
```
```
Kya karta hai:
  YAML file khole bina specific field change karo

immutable: true ConfigMap par → ERROR aata hai (humne test kiya)
```

---

## 📂 CATEGORY 12 — KIND (Cluster Management)

---

### `kind create cluster`
```bash
kind create cluster --name my-cluster
kind create cluster --config kind-config.yaml
```
```
Kya karta hai:
  Docker ke andar naya Kubernetes cluster banata hai
  
  --config → custom settings (port mapping, labels)
  
  Humne use kiya:
  - Ingress ke liye port 80/443 expose karne ko
  - node-labels: ingress-ready=true set karne ko
```

---

### `kind delete cluster`
```bash
kind delete cluster --name my-cluster
```
```
Kya karta hai:
  Poora cluster delete karta hai (Docker container bhi)
  
  Humne use kiya jab:
  Node NotReady tha → IP mismatch → clean fix
  Ingress ke liye naya cluster banana tha
```

---

## 🔑 Quick Cheat Sheet

```
BANANA:
  kubectl apply -f file.yaml          resource banao/update karo
  kubectl create namespace naam       namespace banao

DEKHNA:
  kubectl get pods                    pods dekho
  kubectl get all -n assign1          sab kuch dekho
  kubectl describe pod naam           detail dekho
  kubectl logs deploy/backend         logs dekho
  kubectl logs -f deploy/backend      live logs

ANDAR JANA:
  kubectl exec -it deploy/backend -- sh    shell kholo
  kubectl exec deploy/backend -- command   sirf ek command

UPDATE:
  kubectl rollout restart deployment naam  restart
  kubectl rollout history deployment naam  history
  kubectl rollout undo deployment naam     wapas jao
  kubectl set image deployment/naam ...    image change

DELETE:
  kubectl delete -f file.yaml         resource delete
  kubectl delete pod naam             pod delete
  kubectl delete configmap naam       configmap delete

CLUSTER:
  kind create cluster --name naam     naya cluster
  kind delete cluster --name naam     cluster delete
  kubectl config set-context \        default namespace
    --current --namespace=assign1
```

---

## 💡 Important Lessons Jo Seekhe

```
1. Deployment ≠ Service
   Deployment → pods banata hai
   Service    → pods ko accessible banata hai
   Dono zaroori hain!

2. immutable: true ConfigMap
   Edit nahi ho sakta
   Delete → Recreate hi ek tarika hai

3. kubectl set image → YAML nahi badlti
   Sirf cluster update hota hai

4. rollout undo → naya revision number banta hai
   Purana number wapas nahi aata (Git se farq)

5. Job logs → pod ka STDOUT hai
   Pod delete → logs gone

6. KIND cluster restart → IP change
   Fix: kind delete + kind create

7. rewrite-target: /$2 zaroori hai Ingress mein
   agar path capture groups use kar rahe ho

8. /etc/hosts → browser ke liye fake domain
   echo "127.0.0.1 app.local" | sudo tee -a /etc/hosts
```

---

*Reference guide by: Sharaz Sony*
*Date: 2026-10-02*
*Cluster: KIND | Namespace: assign1*
