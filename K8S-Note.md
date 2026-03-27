# ⚙️ Kubernetes Cheat Sheet — Mohammad Adnan Khan
> Based on my real hands-on learning | 90DaysOfDevOps
> Commands I actually used — not copy-paste from docs!

---

## 🏗️ Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                 CONTROL PLANE (Master)               │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │   API    │  │   ETCD   │  │   Scheduler +    │  │   │
│  │  │  Server  │  │(database)│  │Controller Manager│  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   WORKER 1   │  │   WORKER 2   │  │   WORKER 3   │     │
│  │              │  │              │  │              │     │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │     │
│  │  │  Pod   │  │  │  │  Pod   │  │  │  │  Pod   │  │     │
│  │  │[Flask] │  │  │  │[Nginx] │  │  │  │[MySQL] │  │     │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │     │
│  │  Kubelet     │  │  Kubelet     │  │  Kubelet     │     │
│  │  Kube-proxy  │  │  Kube-proxy  │  │  Kube-proxy  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**
| Component | What it does |
|---|---|
| API Server | Brain of cluster — all commands go through it |
| ETCD | Database — stores entire cluster state |
| Scheduler | Decides which node a Pod runs on |
| Controller Manager | Makes sure desired state = actual state |
| Kubelet | Agent on each node — runs pods |
| Kube-proxy | Handles networking between pods |

---

## 🐳 DevOps Flow: Docker → Kubernetes → CI/CD

```
Developer pushes code
        │
        ▼
┌───────────────┐
│  GitHub Repo  │
└───────┬───────┘
        │ git push triggers
        ▼
┌───────────────────┐
│  GitHub Actions   │
│  CI/CD Pipeline   │
└───────┬───────────┘
        │ builds & pushes
        ▼
┌───────────────┐
│  Docker Hub   │
│  (Image Reg.) │
└───────┬───────┘
        │ kubectl apply
        ▼
┌─────────────────────────┐
│   Kubernetes Cluster    │
│                         │
│  Deployment → ReplicaSet│
│       └──→ Pod(s)       │
│           └──→ Container│
│               └──→ App  │
└─────────────────────────┘
        │
        ▼
┌───────────────┐
│    Service    │
│  (LoadBalancer│
│   /NodePort)  │
└───────────────┘
        │
        ▼
     User 🌐
```

---

## 🛠️ Setup Commands I Used

```bash
# Check Docker is running
docker --version

# Install kind (Kubernetes IN Docker)
[ $(uname -m) = x86_64 ] && curl -Lo ./kind \
  https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
sudo mv ./kind /usr/local/bin/kind
kind --version

# Install kubectl (correct AMD64 version!)
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
kubectl version --client

# Verify all tools ready
docker --version && kind --version && kubectl version --client
```

> ⚠️ Common mistake: I downloaded arm64 version first — always check `uname -m` first!

---

## 🏠 Cluster Management

```bash
# Create cluster with config file
kind create cluster --config kind-config.yml

# Check cluster info
kubectl cluster-info --context kind-mak-cluster

# See all nodes
kubectl get nodes

# See cluster components
kubectl get all -n kube-system
```

**My kind-config.yml:**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: mak-cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

---

## 📦 Namespace Commands

```bash
# Create namespace from file
kubectl apply -f namespace.yml

# List all namespaces
kubectl get ns

# Work inside a specific namespace
kubectl get pods -n nginx-app

# Get everything in a namespace
kubectl get all -n nginx-app
```

**My namespace.yml:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-app
```

---

## 🫙 Pod Commands

```bash
# Apply pod from file
kubectl apply -f pod.yml
kubectl apply -f pod-flaskApp.yml

# List pods (default namespace)
kubectl get pods

# List pods in specific namespace
kubectl get pods -n nginx-app

# See pod logs (debugging!)
kubectl logs flask-app-pod

# Delete a pod
kubectl delete pod nginx-pod

# Port forward to access pod locally
kubectl port-forward pod/flask-app-pod 8080:80
kubectl port-forward pod/flask-app-pod 8082:80
kubectl port-forward pod/flask-app-pod 8080:5000
```

**My Flask App Pod (pod-flaskApp.yml):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: flask-app-pod
  labels:
    app: flask-app
spec:
  containers:
    - name: flask-container
      image: mohammadadnankhan/github-actions-app:latest
      ports:
        - containerPort: 80
```

**My Nginx Pod (pod.yml):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: nginx-app
  labels:
    app: nginx
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

> ⚠️ Lesson learned: Port forward failed when Flask was running on port 5000 
> but I was forwarding to port 80. Always check which port your app runs on!

---

## 🚀 Deployment Commands

```bash
# Create deployment
kubectl apply -f deployment.yml

# List deployments
kubectl get deployments -n nginx-app

# Scale deployment UP (tested with 22 replicas!)
kubectl scale deployment nginx-app-deployment --replicas=22 -n nginx-app

# Scale deployment DOWN
kubectl scale deployment nginx-app-deployment --replicas=1 -n nginx-app

# Watch pods scale in real time
kubectl get pods -n nginx-app
```

**My Deployment (deployment.yml):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-deployment
  namespace: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx:latest
          ports:
            - containerPort: 80
```

---

## 🌐 Service Commands

```bash
# Create service
kubectl apply -f service.yml

# List services
kubectl get svc -n nginx-app

# Get everything (pods + deployments + services)
kubectl get all -n nginx-app

# Port forward to service (not pod directly)
kubectl port-forward svc/nginx-service 8085:80 -n nginx-app
kubectl port-forward svc/nginx-service 8086:80 -n nginx-app
```

**My Service (service.yml):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx-app
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

## 🐛 Debugging Commands

```bash
# Check pod logs
kubectl logs flask-app-pod
kubectl logs flask-app-pod -n nginx-app

# Describe pod (shows events, errors)
kubectl describe pod flask-app-pod

# Check pod status
kubectl get pods -n nginx-app

# Check Docker containers (kind uses Docker)
docker ps
```

---

## 🧹 Cleanup Commands

```bash
# Delete specific pod
kubectl delete pod nginx-pod

# Delete everything from a file
kubectl delete -f deployment.yml
kubectl delete -f service.yml

# Stop Docker containers (kind cluster nodes)
docker stop CONTAINER_ID && docker rm CONTAINER_ID
```

---

## 📊 Quick Reference — kubectl Commands

| Command | What it does |
|---|---|
| `kubectl get pods` | List pods in default namespace |
| `kubectl get pods -n NAME` | List pods in specific namespace |
| `kubectl get all -n NAME` | List everything in namespace |
| `kubectl get ns` | List all namespaces |
| `kubectl get nodes` | List cluster nodes |
| `kubectl apply -f file.yml` | Create/update resource from file |
| `kubectl delete -f file.yml` | Delete resource from file |
| `kubectl logs POD_NAME` | See pod logs |
| `kubectl describe pod NAME` | Detailed pod info |
| `kubectl port-forward pod/NAME PORT:PORT` | Forward port to pod |
| `kubectl port-forward svc/NAME PORT:PORT -n NS` | Forward port to service |
| `kubectl scale deployment NAME --replicas=N` | Scale deployment |
| `kubectl cluster-info` | Show cluster details |

---

## 🧠 K8s Objects I Learned

| Object | Purpose | My example |
|---|---|---|
| Pod | Smallest unit — runs containers | flask-app-pod, nginx-pod |
| Namespace | Logical isolation of resources | nginx-app |
| Deployment | Manages Pods + ReplicaSets | nginx-app-deployment |
| Service | Exposes Pods to network | nginx-service |
| ReplicaSet | Ensures N pods always running | Auto-created by Deployment |

---

## 💡 Real Lessons from My Hands-on

1. **Wrong architecture** — Downloaded arm64 kubectl on x86_64 machine → Exec format error. Always run `uname -m` first!
2. **Wrong port** — Port forwarded to 80 but Flask runs on 5000 → connection refused. Check your app port!
3. **Namespace mismatch** — Pod created in nginx-app namespace but `kubectl get pods` showed nothing. Always add `-n NAMESPACE`!
4. **kind cluster naming** — Use `--context kind-mak-cluster` to target your specific cluster
5. **Scaling is EASY** — `--replicas=22` spun up 22 pods instantly. K8s handles it automatically!

---

## 🔗 My Links
- GitHub: [AddyKhan257/90DaysOfDevOps](https://github.com/AddyKhan257/90DaysOfDevOps)
- Docker Hub: [mohammadadnankhan](https://hub.docker.com/u/mohammadadnankhan)

---

*#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham #Kubernetes #K8s*
