# ☸️ AKS Kubernetes — Exam & Work Quick Reference

---

## 1. Deployment Manifest

A YAML file that tells AKS **what to run, how many copies, and with what resources**.

### Minimal Structure

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-inference-api
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: inference-api        # ⚠️ Must match Pod label below
  template:
    metadata:
      labels:
        app: inference-api      # ⚠️ Must match selector above
    spec:
      containers:
      - name: api
        image: myregistry.azurecr.io/inference-api:v1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"        # 1000m = 1 CPU core
          limits:
            memory: "4Gi"
            cpu: "2000m"
        env:
        - name: MODEL_NAME
          value: "gpt-4"
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: api-key
```

### Key Fields Cheat Sheet

| Field | Purpose |
|---|---|
| `replicas` | Number of Pods to run |
| `selector.matchLabels` | Which Pods this Deployment manages |
| `template.metadata.labels` | Labels assigned to Pods (must match selector) |
| `image` | Full path: `registry.azurecr.io/name:tag` |
| `containerPort` | Port the app listens on inside container |
| `resources.requests` | Minimum needed to schedule the Pod |
| `resources.limits` | Maximum allowed — exceeded → Pod killed (OOMKill) |

---

## 2. Replicas — How Many to Run?

| Replicas | When to use |
|---|---|
| `2` | Dev / non-critical — basic crash resilience |
| `3` | Production — recommended for external APIs |
| `4+` | High traffic only — each replica costs resources |

> **Why it matters:** With 1 replica, a crash = downtime. With 2+, others keep serving traffic.

---

## 3. Resources — Requests vs Limits

```
requests → used for SCHEDULING (must fit on a node)
limits   → used for ENFORCEMENT (exceeded = terminated)
```

- **Too low requests** → Pod crashes (OOMKill)
- **Too high requests** → Wastes cluster resources & money
- **Rule of thumb:** model size + 20% overhead for memory

---

## 4. Environment Variables & Secrets

**Plain config** (non-sensitive):
```yaml
env:
- name: MODEL_NAME
  value: "gpt-4"
```

**Secrets** (API keys, passwords):
```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: api-secrets   # Secret object name
      key: api-key        # Key inside the Secret
```

**Create a Secret with kubectl:**
```bash
kubectl create secret generic api-secrets --from-literal=api-key=YOUR_KEY
```

> ⚠️ **Never hardcode secrets in manifests** — they end up in source control.

---

## 5. Service Types — Choosing the Right One

| Type | Access | Use When |
|---|---|---|
| `ClusterIP` *(default)* | Internal cluster only | Backend services, inter-pod communication |
| `NodePort` | External via `node-ip:port` | Dev/testing, simple external access |
| `LoadBalancer` | External public IP (Azure LB) | Production APIs, internet-facing apps |
| `ExternalName` | Maps to external DNS | Calling external services from within cluster |

---

## 6. Service Manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-api-service
spec:
  type: LoadBalancer
  selector:
    app: inference-api        # ⚠️ Must match Pod labels in Deployment
  ports:
  - protocol: TCP
    port: 80                  # Client connects here
    targetPort: 8080          # App listens here (inside container)
```

### Port Mapping

```
Client → port 80 (Service) → targetPort 8080 (container)
```

### Service Access URLs

| Type | URL format |
|---|---|
| ClusterIP | `http://inference-api-service.default.svc.cluster.local:8080` |
| NodePort | `http://<node-ip>:<nodeport>` (port 30000+) |
| LoadBalancer | `http://<EXTERNAL-IP>` (from `kubectl get svc`) |

---

## 7. Deploying — kubectl Commands

```bash
# Apply single file
kubectl apply -f deployment.yaml

# Apply multiple files
kubectl apply -f deployment.yaml -f service.yaml

# Apply all YAMLs in current directory
kubectl apply -f .
```

---

## 8. Verifying Deployment

```bash
# Check Pods
kubectl get pods

# Check Deployment (READY = running/desired)
kubectl get deployment

# Check Services (look for EXTERNAL-IP)
kubectl get svc

# View app logs
kubectl logs <pod-name>
kubectl logs <pod-name> --since=10m
kubectl logs -l app=inference-api    # all pods with label
```

### Pod Status — What They Mean

| Status | Meaning |
|---|---|
| `Running` | ✅ All good |
| `Pending` | ⏳ Waiting to be scheduled (resource shortage?) |
| `CrashLoopBackOff` | ❌ App keeps crashing — check logs |
| `ImagePullBackOff` | ❌ Can't pull the image — check image path/tag |

---

## 9. Troubleshooting Quick Reference

### ImagePullBackOff
- Wrong image path or tag in manifest
- Image not pushed to registry
- AKS lacks credentials to pull from ACR

```bash
kubectl describe pod <pod-name>   # See events section for error
az acr repository list --name myregistry  # Verify image exists
```

### CrashLoopBackOff
- App crashes on startup (missing env vars, bad config)

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # Logs from last crash
kubectl describe pod <pod-name>
```

### Pending Pods
- Not enough CPU/memory on cluster nodes

```bash
kubectl describe pod <pod-name>    # Look for "Insufficient memory/cpu"
kubectl get nodes
# Scale cluster if needed:
az aks scale --resource-group mygroup --name myaks --node-count 3
```

### Service Has No Endpoints (traffic goes nowhere)
- Pod labels ≠ Service selector → no match

```bash
kubectl get pods --show-labels
kubectl describe svc inference-api-service   # Check "Endpoints:" field
```

> **Fix:** Make sure `template.metadata.labels` in Deployment matches `selector` in Service.

---

## 10. The Golden Rule — Label Matching

```yaml
# Deployment Pod template
labels:
  app: inference-api     ← must equal ↓

# Service selector
selector:
  app: inference-api     ← must equal ↑
```

If these don't match → Service has no endpoints → no traffic reaches your app.

---

## Quick Commands Summary

```bash
kubectl apply -f <file>                  # Deploy/update resources
kubectl get pods                         # List pods & status
kubectl get svc                          # List services & IPs
kubectl get deployment                   # Deployment ready status
kubectl logs <pod>                       # App logs
kubectl describe pod <pod>               # Events & config details
kubectl describe svc <svc>              # Service details & endpoints
kubectl create secret generic <name> \
  --from-literal=<key>=<value>          # Create a Secret
```

# Kubernetes Configuration & Storage — Exam Notes

---

## 1. ConfigMaps

### What & Why
- Stores **non-sensitive** configuration as key-value pairs
- Keeps config out of container images and source code
- Max size: **1 MiB** per ConfigMap
- Keys: alphanumeric characters, dashes, underscores, or dots only

### Define a ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
data:
  FEATURE_X_ENABLED: "true"
  SERVICE_ENDPOINT: "https://api.example.com"
  app.config: |
    log_level=info
    timeout_seconds=30
```

### Inject as Environment Variables
Two methods:

| Method | YAML field | Use when |
|---|---|---|
| Single key | `valueFrom.configMapKeyRef` | You need specific keys |
| All keys | `envFrom.configMapRef` | Many values, avoid repetitive YAML |

```yaml
env:
- name: FEATURE_X_ENABLED
  valueFrom:
    configMapKeyRef:
      name: app-settings
      key: FEATURE_X_ENABLED
# OR load all at once:
envFrom:
- configMapRef:
    name: app-settings
```

### Mount as Files
- Each ConfigMap key becomes a **file** in the mount directory
- Use when the app expects config files on disk

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-settings
containers:
- name: api
  volumeMounts:
  - name: config-volume
    mountPath: /app/config
    readOnly: true
```

### ⚠️ Auto-Update Behavior
| Injection method | Auto-updates? |
|---|---|
| Volume mount | ✅ Yes (kubelet sync period) |
| Environment variable | ❌ No — requires Pod restart |
| `subPath` mount | ❌ No |

### Immutable ConfigMaps
- Set `immutable: true` to prevent accidental changes
- Performance benefit: Kubernetes closes watches → less API server load
- **Cannot** change data or revert immutable setting once set
- Must **delete and recreate** to make changes

```yaml
immutable: true
```

### Azure App Configuration Integration
- **Azure App Configuration Kubernetes Provider** runs in the cluster
- Generates standard Kubernetes ConfigMaps from App Configuration data
- Syncs feature flags, endpoints, Key Vault references automatically
- Useful for centralized config across multiple AKS clusters

### Verify ConfigMaps
```bash
kubectl apply -f configmap.yaml
kubectl describe configmap app-settings
kubectl exec <pod-name> -- printenv | grep FEATURE
```

---

## 2. Secrets

### What & Why
- Stores **sensitive** values: API keys, connection strings, passwords
- Keeps credentials out of source control and images

### Secret Types
| Type | Use case |
|---|---|
| `Opaque` (default) | API keys, passwords, generic strings |
| `kubernetes.io/dockerconfigjson` | Container registry credentials |
| `kubernetes.io/tls` | TLS certificates and keys |

### Create a Secret
```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_CONNECTION="Host=db;User=app;Password=secure" \
  --from-literal=API_KEY="your-api-key"
```

### Reference in a Deployment
```yaml
env:
- name: DB_CONNECTION
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: DB_CONNECTION
```

### Azure Secret Management Options

| Option | How it works | Best for |
|---|---|---|
| **Key Vault + CSI Driver** | Mounts secrets from Key Vault directly as files; polls for changes | Strict compliance, fine-grained audit trails |
| **App Configuration + Key Vault refs** | App Config stores references; provider resolves + generates K8s Secrets | Centralized secret mapping, standard K8s resources |

### Best Practices ✅
- **RBAC**: Restrict Secret access to only service accounts that need it
- **No source control**: Never commit Secret manifests with literal values
- **Rotate regularly**: Trigger Deployment rollouts after updates so Pods get new values
- **Key Vault for production**: Enables automated rotation, audit trails
- **Encrypt at rest**: Configure etcd encryption at cluster creation time

### Verify Secrets
```bash
kubectl get secrets
kubectl describe secret app-secrets   # shows keys, NOT values
kubectl describe deployment web-api
```

---

## 3. Persistent Storage

### Why Persistent Storage?
- Container filesystems are **ephemeral** — data lost on Pod restart/reschedule
- **PersistentVolume (PV)** + **PersistentVolumeClaim (PVC)** provide durable storage

### Two AKS Storage Approaches
| Approach | Description | Best for |
|---|---|---|
| **CSI Drivers** | Standard StorageClasses; provisions Azure Disk, Files, Blob | General-purpose workloads |
| **Azure Container Storage** | Container-native, NVMe-oF protocols, fast attach/detach | I/O intensive, rapid scaling of stateful workloads |

### Default CSI StorageClasses

| StorageClass | Backing | Access Mode | Performance | Use case |
|---|---|---|---|---|
| `managed-csi` | Azure Disk | ReadWriteOnce | Standard HDD/SSD | Single Pod, databases |
| `managed-csi-premium` | Azure Disk Premium | ReadWriteOnce | Premium SSD | Low latency, high throughput |
| `azurefile-csi` | Azure Files | ReadWriteMany | Standard HDD/SSD | Shared multi-Pod access |
| `azurefile-csi-premium` | Azure Files Premium | ReadWriteMany | Premium SSD | High-perf shared access |

### Access Modes
- **ReadWriteOnce (RWO)**: Single node only → Azure Disk
- **ReadWriteMany (RWX)**: Multiple nodes/Pods simultaneously → Azure Files

### Define a PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: default
```

### Mount PVC in a Deployment
```yaml
spec:
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-pvc
  containers:
  - name: api
    volumeMounts:
    - name: data-volume
      mountPath: /app/data
```

### Verify Persistence
```bash
kubectl apply -f pvc.yaml
kubectl apply -f deployment.yaml
kubectl describe pvc data-pvc          # should show: Status: Bound
kubectl describe pod -l app=web-api
```
> **Test tip**: Write a file inside the Pod → delete Pod → check the file still exists in the new Pod.

---

## Quick Reference: Decision Cheatsheet

```
Need to store config?
├── Non-sensitive → ConfigMap
│   ├── App reads env vars? → envFrom / valueFrom
│   └── App reads files? → Volume mount
└── Sensitive → Secret
    ├── Dev/test → Opaque Secret
    └── Production → Azure Key Vault + CSI Driver

Need storage?
├── Single Pod access → Azure Disk (managed-csi / managed-csi-premium)
├── Multi-Pod shared access → Azure Files (azurefile-csi / azurefile-csi-premium)
└── High-perf stateful workloads → Azure Container Storage
```

---

## Key kubectl Commands Summary

```bash
# ConfigMaps
kubectl apply -f configmap.yaml
kubectl describe configmap <name>
kubectl exec <pod> -- printenv | grep <KEY>

# Secrets
kubectl create secret generic <name> --from-literal=KEY=value
kubectl get secrets
kubectl describe secret <name>

# Storage
kubectl describe pvc <name>
kubectl get pods
kubectl describe pod -l app=<label>
```


# AKS Monitoring & Troubleshooting — Exam/Reference Notes

---

## 1. Monitor Application Logs & Metrics

### Why It Matters
- AI workloads (inference APIs, background processors) need predictable performance
- Monitoring helps distinguish **normal variation** from **incidents**
- Helps decide: scale, investigate, or roll back

---

### Key Signals to Watch

| Signal | Why It Matters |
|---|---|
| Response latency & throughput | Core health of AI endpoints |
| Error rates (HTTP 5xx, timeouts) | Reliability indicator |
| Pod restart counts & exit codes | Container stability |
| CPU & memory vs. requests/limits | Resource saturation |

---

### Viewing Logs — Azure Portal
**Path:** AKS cluster → Kubernetes resources → Workloads → select pod → **Live Logs**

- Streams `stdout` / `stderr` in real time
- Can pause, search text, switch containers in multi-container pods
- **Container Insights** (Monitoring → Insights): cross-pod log filtering by namespace/pod/container

---

### Viewing Logs — kubectl

```bash
kubectl get pods -n ai-workloads                          # list pods
kubectl logs <pod-name> -n ai-workloads                   # view logs
kubectl logs -f <pod-name> -n ai-workloads                # stream (follow)
kubectl logs <pod-name> -c inference-api -n ai-workloads  # specific container
```

> `-n ai-workloads` targets a specific **namespace** (logical boundary grouping workloads)

---

### Viewing Metrics — Azure Portal
**Path:** AKS cluster → Monitoring tab → Insights → Nodes / Controllers / Containers tabs

- CPU & memory graphs per node pool
- **Live Metrics** for individual pods (real-time CPU, memory, network)
- Container insights: heat maps, performance charts

---

### Viewing Metrics — kubectl

```bash
kubectl top nodes               # per-node CPU & memory
kubectl top pods -n ai-workloads  # per-pod CPU & memory
```

> Requires **metrics-server** installed on the cluster

**High CPU** → model needs more resources or scale-out → can cause **inference latency**  
**High memory** → container restarts or degraded performance

---

### Combined Workflow (Best Practice)

```
Portal Monitoring tab  →  quick health overview
Live Logs (portal)     →  real-time investigation
kubectl logs           →  filter/pipe output
kubectl top            →  current resource snapshot
Container Insights     →  historical trends, cross-pod analysis
```

---

### Best Practices ✅
- Use **structured logging** (JSON) — easier to query
- Add **request IDs, model name, version** to log entries for tracing
- Define **latency & error budget targets** and alert on thresholds
- Start with portal for overview → kubectl for targeted queries

---

## 2. Troubleshoot Pods & Services

### Common Pod Problem States

| State | Cause |
|---|---|
| `ImagePullBackOff` | Wrong image reference or missing registry credentials |
| `CrashLoopBackOff` | Container starts and exits repeatedly (app error) |
| `Pending` | Can't schedule — insufficient resources |
| Frequent restarts | Memory leaks or unhandled exceptions |

---

### Inspecting Pods — Azure Portal
**Path:** AKS cluster → Workloads → select pod → Live Logs  
**Diagnose and solve problems** (AKS menu) → automated diagnostics + remediation suggestions

---

### Inspecting Pods — kubectl

```bash
kubectl get pods -n ai-workloads
kubectl describe pod <pod-name> -n ai-workloads
kubectl get events -n ai-workloads   # recent events across all pods
```

**In `describe` output, check:**
- Readiness & liveness probe results
- Environment variables & mounted Secrets
- Volume mounts (model files, config)
- Events section (image pull errors, scheduling issues, probe failures)

---

### Debugging Inside a Container

**Portal:** AKS cluster → Workloads → select pod → **Console** tab → choose shell

**kubectl:**
```bash
kubectl exec -it <pod-name> -n ai-workloads -- /bin/sh
```

**Inside the container, check:**
- Model files / config directories exist
- Environment variables are correct
- `curl http://localhost:8080/health` responds

> ⚠️ Never make permanent fixes inside containers — update manifests/source code instead

---

### Inspecting Services — Azure Portal
**Path:** AKS cluster → Services and ingresses → select Service

Check: type, cluster IP, external IP, port mappings, **Endpoints** (empty = no backing pods)

---

### Inspecting Services — kubectl

```bash
kubectl get service -n ai-workloads
kubectl describe service <service-name> -n ai-workloads
kubectl get endpointslices -l kubernetes.io/service-name=<service-name> -n ai-workloads
```

**In Service definition, verify:**
- **Selector labels** match pod labels
- **Ports & targetPorts** align with container ports
- **EndpointSlices** list actual pod IPs

> Empty EndpointSlices = no traffic reaches pods, even if pods are healthy

---

### Best Practices ✅
- Start with portal Workloads for quick restart-count overview
- Use **Diagnose and solve problems** for guided auto-detection
- Fix label mismatches first when a Service has no endpoints
- Keep environment reproducible — changes via manifests only

---

## 3. Verify Service Connectivity & Endpoints

### Kubernetes Service Types

| Type | Scope | Use Case |
|---|---|---|
| `ClusterIP` | Internal only | Internal AI microservices |
| `LoadBalancer` | External (Azure LB) | Public-facing AI APIs |
| `NodePort` | Node-level port mapping | Less common |

---

### Reviewing Services — Azure Portal
**Path:** AKS cluster → Services and ingresses

- Filter by namespace
- Check **External IP** column (LoadBalancer services)
- Verify **selector labels** and **port configs**
- Empty endpoints list = Service can't route traffic
- Ingress resources: host rules, paths, backend services, external address

---

### Validating Service-to-Pod Connectivity — kubectl

```bash
kubectl get service -n ai-workloads
kubectl describe service inference-api -n ai-workloads
kubectl get endpointslices -l kubernetes.io/service-name=inference-api -n ai-workloads
```

**Verify:**
1. Selector matches pod labels
2. Ports/targetPorts align
3. EndpointSlices contain pod IPs

---

### Testing Endpoints with Port-Forward

```bash
# Forward local port 8080 → Service port 80
kubectl port-forward service/inference-api 8080:80 -n ai-workloads

# In another terminal, test the endpoint
curl http://localhost:8080/api/inference
```

> Useful before ingress is configured or for debugging new model endpoints

---

### Verifying External Connectivity

**Portal:** Services and ingresses → External IP column (LoadBalancer) or Address column (Ingress)

**kubectl:**
```bash
kubectl get service inference-api -n ai-workloads   # check EXTERNAL-IP
kubectl get ingress -n ai-workloads                  # check Address column
```

---

### Best Practices ✅
- Use **consistent labels** across pods, Services, and ingress
- Test with **port-forward** before exposing externally
- Start portal → visual overview; kubectl → detailed endpoint inspection
- Pair connectivity tests with **logs & metrics** to see config change impact

---

## Quick Reference — kubectl Cheat Sheet

```bash
# Pods
kubectl get pods -n <ns>
kubectl describe pod <name> -n <ns>
kubectl logs <name> -n <ns>
kubectl logs -f <name> -n <ns>                   # follow/stream
kubectl logs <name> -c <container> -n <ns>       # specific container
kubectl exec -it <name> -n <ns> -- /bin/sh       # shell into container

# Resources
kubectl top nodes
kubectl top pods -n <ns>

# Services & Networking
kubectl get service -n <ns>
kubectl describe service <name> -n <ns>
kubectl get endpointslices -l kubernetes.io/service-name=<name> -n <ns>
kubectl port-forward service/<name> <local>:<remote> -n <ns>
kubectl get ingress -n <ns>

# Events
kubectl get events -n <ns>
```

---

## Key Concepts to Remember

- **Namespace** — logical boundary grouping related workloads; use `-n <namespace>` in kubectl
- **Labels** — key/value pairs on pods; Services use **selectors** to match them
- **EndpointSlices** — list of pod IPs a Service routes to; empty = broken connectivity
- **Container Insights** — Azure Monitor integration for historical trends & cross-pod analysis
- **Metrics Server** — must be installed for `kubectl top` to work
- **CrashLoopBackOff** — container keeps crashing; check logs for app errors
- **ImagePullBackOff** — image can't be pulled; check image name & registry credentials
