# 🚀 Kubernetes Pod-to-Pod Communication

A production-ready example demonstrating how to connect Kubernetes pods to each other using **Services**, **ConfigMaps**, **Deployments**, and **NetworkPolicies**.

---

## 📁 Project Structure

```
k8s-pod-communication/
├── namespace.yaml                   # Namespace: app-ns
├── networkpolicy.yaml               # Restricts backend access to frontend only
├── backend/
│   ├── backend-configmap.yaml       # Backend environment config
│   ├── backend-deployment.yaml      # Backend app (2 replicas)
│   └── backend-service.yaml         # ClusterIP Service (stable DNS)
└── frontend/
    ├── frontend-configmap.yaml      # Frontend environment config (BACKEND_URL)
    └── frontend-deployment.yaml     # Frontend app (calls backend every 5s)
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      namespace: app-ns                      │
│                                                             │
│   ┌─────────────────┐           ┌───────────────────────┐  │
│   │  frontend pod   │           │   backend-service     │  │
│   │  app: frontend  │──────────►│   (ClusterIP :80)     │  │
│   └─────────────────┘           └──────────┬────────────┘  │
│                                            │               │
│                               ┌────────────▼────────────┐  │
│                               │     backend pods (x2)   │  │
│                               │     app: backend :8080  │  │
│                               └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

- The **frontend** pod calls the **backend** via the Service DNS name `http://backend-service:80`.
- The **Service** load-balances traffic across all backend pod replicas.
- The **NetworkPolicy** ensures only `frontend` pods can reach the `backend` pods.

---

## ⚙️ Prerequisites

- Kubernetes cluster (local: [minikube](https://minikube.sigs.k8s.io/) / [kind](https://kind.sigs.k8s.io/), or cloud: EKS / GKE / AKS)
- `kubectl` configured and connected to your cluster
- A CNI plugin that supports NetworkPolicy (e.g. **Calico**, **Cilium**, **Weave Net**)

---

## 🚀 Quick Start

### 1. Clone the Repository
```bash
git clone https://github.com/your-username/k8s-pod-communication.git
cd k8s-pod-communication
```

### 2. Deploy All Resources
```bash
kubectl apply -f namespace.yaml
kubectl apply -f backend/
kubectl apply -f frontend/
kubectl apply -f networkpolicy.yaml
```

### 3. Verify Pods Are Running
```bash
kubectl get pods -n app-ns
```

Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
backend-xxxxxxxxxx-xxxxx    1/1     Running   0          30s
backend-xxxxxxxxxx-xxxxx    1/1     Running   0          30s
frontend-xxxxxxxxxx-xxxxx   1/1     Running   0          25s
```

### 4. Watch Frontend Calling Backend
```bash
kubectl logs -f deploy/frontend -n app-ns
```

Expected output:
```
Starting frontend...
Calling backend at http://backend-service:80 ...
Hello from backend!
Calling backend at http://backend-service:80 ...
Hello from backend!
```

---

## 🔍 Manual Testing

```bash
# Exec into the frontend pod and curl the backend directly
kubectl exec -it deploy/frontend -n app-ns -- curl http://backend-service:80

# Check the backend Service endpoints (lists backend pod IPs)
kubectl get endpoints backend-service -n app-ns

# Describe the NetworkPolicy
kubectl describe networkpolicy backend-allow-frontend -n app-ns
```

---

## 📄 File Details

### `namespace.yaml`
Creates a dedicated namespace `app-ns` to isolate all resources.

### `backend/backend-configmap.yaml`
Stores backend environment variables (`PORT`, `APP_ENV`) as a ConfigMap.

### `backend/backend-deployment.yaml`
Deploys the backend app with:
- **2 replicas** for availability
- **Readiness & Liveness probes** for health checking
- **Resource requests & limits** for stable scheduling

### `backend/backend-service.yaml`
Exposes the backend pods via a **ClusterIP Service** named `backend-service`.
This gives a stable internal DNS name: `http://backend-service.app-ns.svc.cluster.local`

### `frontend/frontend-configmap.yaml`
Stores `BACKEND_URL=http://backend-service:80` — injected into the frontend container as an env var.

### `frontend/frontend-deployment.yaml`
Deploys the frontend app that reads `$BACKEND_URL` and calls the backend every 5 seconds.

### `networkpolicy.yaml`
Enforces that **only** pods with label `app: frontend` can reach the backend pods on port `8080`. All other pods are blocked.

---

## 🌐 DNS Resolution

| Form | Address |
|---|---|
| Short (same namespace) | `http://backend-service` |
| With namespace | `http://backend-service.app-ns` |
| Full FQDN | `http://backend-service.app-ns.svc.cluster.local` |

> For **cross-namespace** communication, always use the Full FQDN.

---

## 🔌 Using the Connection in Your App Code

**Node.js**
```js
const axios = require('axios');
const res = await axios.get(process.env.BACKEND_URL);
```

**Python**
```python
import os, requests
res = requests.get(os.environ["BACKEND_URL"])
```

**Go**
```go
resp, err := http.Get(os.Getenv("BACKEND_URL"))
```

---

## 🧹 Cleanup

### Delete everything at once (recommended)
```bash
kubectl delete namespace app-ns
```

### Or delete using YAML files
```bash
kubectl delete -f networkpolicy.yaml
kubectl delete -f frontend/
kubectl delete -f backend/
kubectl delete -f namespace.yaml
```

---

## 📌 Key Rules

- ✅ Always use a **Service** — never hardcode pod IPs (they change on restart)
- ✅ Use **short DNS names** (`backend-service`) within the same namespace
- ✅ Use **full FQDN** for cross-namespace communication
- ✅ Use **NetworkPolicy** to restrict which pods can talk to which
- ✅ Pass URLs via **env vars / ConfigMap** — not hardcoded in the image
- ⚠️ NetworkPolicy requires a compatible **CNI plugin** (Calico, Cilium, Weave Net)

---

## 📚 References

- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes DNS for Services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

---

## 📝 License

MIT License — feel free to use, modify, and distribute.
