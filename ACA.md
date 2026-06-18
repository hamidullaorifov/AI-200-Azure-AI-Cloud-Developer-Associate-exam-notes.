# Azure Container Apps – Exam & Work Notes

## 1. CLI Setup (Before Deploying)

```bash
az login
az upgrade
az extension add --name containerapp --upgrade

az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

- Add `--allow-preview true` if you need preview features.
- Keeping CLI/extension updated avoids confusing errors from changed command options.

---

## 2. Environments

### What an environment is
- A **logical boundary** where one or more container apps run.
- Group related services (e.g., AI API + background processor) to share **networking** and **logging**.
- Use separate environments for **dev / test / prod**.
- An environment is NOT just a folder — it affects internal traffic, logging integration, and governance.

### Create an environment
```bash
az group create --name rg-aca-demo --location centralus

az containerapp env create \
    --name aca-env-demo \
    --resource-group rg-aca-demo \
    --location centralus
```

### Inspect an environment
```bash
az containerapp env show \
    --name aca-env-demo \
    --resource-group rg-aca-demo
```

### Best practices
- **Separate by lifecycle** (dev/test/prod) → reduces blast radius.
- **Align with networking boundaries** → use internal ingress for private communication.
- **Standardize naming conventions** for resource groups/environments/apps.
- **Plan observability early** — decide log destinations before production traffic.

---

## 3. Deployment Methods

### Option A — Quick deploy: `az containerapp up`
- Fastest way; can auto-create environment + resources.
- Good for **prototypes / first deployment**.

```bash
az containerapp up \
    --name my-container-app \
    --resource-group rg-aca-demo \
    --location centralus \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --target-port 80 \
    --ingress external \
    --query properties.configuration.ingress.fqdn
```

⚠️ **Image names must be lowercase** — uppercase causes image pull failures that *look like* auth errors.

### Option B — Explicit: `az containerapp create`
- Use when aligning to an **existing environment/resource group**.
- Better for consistent defaults across environments.

```bash
az containerapp create \
    --name ai-api \
    --resource-group rg-aca-demo \
    --environment aca-env-demo \
    --image mcr.microsoft.com/k8se/quickstart:latest \
    --ingress external \
    --target-port 80
```

### Updating an app → creates a new Revision
```bash
az containerapp update \
    --name ai-api \
    --resource-group rg-aca-demo \
    --image myregistry.azurecr.io/ai-api:v2
```
- **Revisions** = versioned configuration changes (new image/env vars).
- Lets you **validate before shifting traffic**.

### Option C — YAML-based deployment
- Stores config as **source of truth**; reviewable via source control.
- ⚠️ When using `--yaml`, **all other flags are ignored**.

```bash
az containerapp create -n ai-api -g rg-aca-demo \
    --environment aca-env-demo --yaml ./containerapp.yml

az containerapp update -n ai-api -g rg-aca-demo \
    --yaml ./containerapp.yml
```

### Other deployment paths (mentioned, not core)
- Bicep (IaC), GitHub Actions (CI/CD), Azure Portal (manual changes).

---

## 4. Runtime Configuration: Env Vars & Secrets

### Environment Variables (non-sensitive)
- Use for: log levels, feature flags, URLs, timeouts, batch sizes.
- Keeps images **portable** across environments.

```bash
# At create time
az containerapp create -n ai-api -g rg-aca-demo \
    --environment aca-env-demo \
    --image myregistry.azurecr.io/ai-api:v1 \
    --ingress external --target-port 8000 \
    --env-vars LOG_LEVEL=info FEATURE_EMBEDDINGS=true

# Add/update later (doesn't remove existing)
az containerapp update -n ai-api -g rg-aca-demo \
    --set-env-vars LOG_LEVEL=debug
```

### Secrets (sensitive values)
- Keeps API keys, DB credentials, signing keys **out of images & source control**.
- Can rotate secrets **without redeploying**.

```bash
az containerapp secret set -n ai-api -g rg-aca-demo \
    --secrets embeddings-api-key="REPLACE_WITH_REAL_VALUE"
```

- Can also reference **Azure Key Vault** (Key Vault = system of record; fetched at runtime).

### Secret References (link env var → secret)
- Format: `secretref:<secret-name>`

```bash
az containerapp update -n ai-api -g rg-aca-demo \
    --set-env-vars EMBEDDINGS_API_KEY=secretref:embeddings-api-key
```

### YAML for runtime config
```yaml
properties:
  template:
    containers:
    - name: ai-api
      env:
      - name: LOG_LEVEL
        value: info
      - name: EMBEDDINGS_API_KEY
        secretRef: embeddings-api-key
```
- Apply with: `az containerapp update --yaml ./containerapp.yml`
- ⚠️ Store only **secret names** in YAML, never values.

### Best Practices — Configuration
| Practice | Why |
|---|---|
| Separate config from image | Same image runs everywhere |
| Prefer `secretref:` / `secretRef` | Values never appear in YAML/shell history |
| Rotate secrets independently | Faster credential response |
| Use YAML in source control | Reduces drift, enables review |

---

## 5. Private Registry Authentication

### Two options
| Method | Best for | Trade-off |
|---|---|---|
| Username & Password | Quick validation, non-Azure registries | More secrets to manage |
| Managed Identity (ACR only) | Production | Needs role assignment, but no long-lived creds |

### Username & Password
```bash
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io \
    --username MyRegistryUsername \
    --password MyRegistryPassword
```

### Managed Identity (for Azure Container Registry)
- Requires granting **`AcrPull`** role to the identity first.

```bash
az containerapp registry set -n ai-api -g rg-aca-demo \
    --server myregistry.azurecro.io \
    --identity system
```

### Verify registry config
```bash
az containerapp registry list -n ai-api -g rg-aca-demo

az containerapp registry show -n ai-api -g rg-aca-demo \
    --server myregistry.azurecr.io
```

⚠️ Registry issues often show as **image pull failures** that look like auth/permission errors — but the cause is registry config, not app config.

### Best Practices — Registry
- Prefer **managed identity** for ACR.
- Grant **least privilege** (`AcrPull` only).
- Use **fully qualified image names** (include registry hostname).
- **Validate after rollout** — confirm new revision starts successfully.

---

## 6. Verification & Troubleshooting

### Step 1 — Check app config/ingress
```bash
az containerapp show -n ai-api -g rg-aca-demo
```
- Confirms ingress type (internal vs external) and retrieves FQDN.

### Step 2 — Check container logs (fastest diagnosis)
```bash
# Recent logs (limited lines)
az containerapp logs show -n ai-api -g rg-aca-demo

# Live tail
az containerapp logs show -n ai-api -g rg-aca-demo \
    --follow --tail 30

# Platform/system logs
az containerapp logs show -n ai-api -g rg-aca-demo \
    --type system
```
- **Console logs** = app-level issues (missing env vars, crashes, dependency errors).
- **System logs** = platform-level issues.

### Step 3 — Check Revisions (confirm active version)
```bash
az containerapp revision list -n ai-api -g rg-aca-demo

# Include inactive revisions
az containerapp revision list -n ai-api -g rg-aca-demo --all
```

### Step 4 — Check Replicas (scaling/availability)
```bash
az containerapp replica list -n ai-api -g rg-aca-demo

az containerapp replica list -n ai-api -g rg-aca-demo \
    --revision MyRevision
```
- Replicas = running instances of a revision.
- Helps detect **scale-to-zero**, **crash loops**, or missing traffic.

### Best Practices — Verification
1. **Start with logs** (`logs show`) for startup/runtime errors.
2. **Check revisions** after updates → confirm new revision became active.
3. **Inspect replicas** during incidents.
4. Use **`--query`** to extract key fields (e.g., FQDN, revision status) for automation.

---

## 7. Quick Command Cheat Sheet

| Goal | Command |
|---|---|
| Quick deploy | `az containerapp up` |
| Explicit deploy | `az containerapp create` |
| Update image/config | `az containerapp update` |
| Deploy via YAML | `az containerapp create/update --yaml file.yml` |
| Set env vars | `--env-vars` (create) / `--set-env-vars` (update) |
| Set secret | `az containerapp secret set --secrets key=value` |
| Reference secret | `secretref:<secret-name>` |
| Configure registry (user/pass) | `az containerapp registry set --username --password` |
| Configure registry (identity) | `az containerapp registry set --identity system` |
| View app details | `az containerapp show` |
| View logs | `az containerapp logs show` |
| List revisions | `az containerapp revision list` |
| List replicas | `az containerapp replica list` |

---

## 8. Key Concepts to Remember for Exam

- **Environment** = isolation boundary (networking + logging shared within it).
- **Revision** = versioned snapshot of config; created on each update.
- **Replica** = running instance of a revision.
- `--yaml` flag **overrides/ignores all other CLI flags**.
- `secretref:` links env vars to stored secrets — never expose actual secret values.
- **Managed Identity + AcrPull** = best practice for ACR auth in production.
- Image pull failures often **mimic authentication errors** — check registry config & lowercase image names.
- Logs → Revisions → Replicas = standard troubleshooting order.


# Azure Container Apps — Exam & Reference Notes

---

## 1. Image Updates & Revision Management

### Tags vs. Digests
| | Tags | Digests |
|---|---|---|
| **Identity** | Mutable (can change) | Immutable (fixed) |
| **Use case** | Dev/staging iteration | Production releases |
| **Traceability** | Low | High — proves exact artifact |

> **Key rule:** Use **digests** in production so you can audit exactly what served requests during an incident.

### Update Image → Creates New Revision
```bash
az containerapp update \
  --name <app> --resource-group <rg> \
  --image <registry>/<repo>@sha256:<digest>
```

Verify the image was recorded:
```bash
az containerapp show \
  --name <app> --resource-group <rg> \
  --query properties.template.containers
```

---

## 2. Revision Modes

| Mode | Behavior |
|---|---|
| **Single** | Only one active revision at a time; simpler ops |
| **Multiple** | Multiple active revisions; enables canary/traffic-split rollouts |

**"Active"** = revision can receive traffic. Inactive revisions still exist and are useful for rollback/investigation.

### Revision Commands
```bash
# List all revisions
az containerapp revision list --name <app> --resource-group <rg> -o table

# Inspect a specific revision
az containerapp revision show --name <app> --resource-group <rg> --revision <rev-name>

# Deactivate (safe first step — preserves evidence)
az containerapp revision deactivate --name <app> --resource-group <rg> --revision <rev-name>
```

> **Deactivate before delete** — stops traffic, keeps rollback options, preserves evidence.

**Revision retention:** Auto-purge kicks in at **100 inactive revisions**. Adjust with `--max-inactive-revisions`.

---

## 3. App Lifecycle: Stop / Start / Restart

| Action | Scope | When to use |
|---|---|---|
| **Stop** | Whole app | Guarantee no replicas start during incident mitigation |
| **Start** | Whole app | Resume after stop |
| **Restart** | Whole app | Clear transient runtime state (deadlock, stuck process) |
| **Revision deactivate** | Single revision | Isolate one bad release without affecting others |

```bash
az containerapp stop    --name <app> --resource-group <rg>
az containerapp start   --name <app> --resource-group <rg>
az containerapp restart --name <app> --resource-group <rg>
```

> ⚠️ For AI services: restarting too often increases **cold starts** (model loading is slow). Use restart intentionally.

---

## 4. Failing Revision Checklist

When a revision fails, check these in order:

1. **Image pull failure** — registry credentials missing/invalid, or wrong image reference
2. **Port mismatch** — container listens on a different port than ingress expects
3. **Missing config** — required env vars or secrets not set/misnamed
4. **Probe failure** — wrong path, or too short a timeout for model loading
5. **Resource pressure** — OOM kill (memory) or throttling (CPU)

Quick status query:
```bash
az containerapp revision list \
  --name <app> --resource-group <rg> \
  --query "[].{name:name,active:properties.active,health:properties.healthState}" \
  -o table
```

---

## 5. Logs & Monitoring

### Stream logs in real time
```bash
az containerapp logs show \
  --name <app> --resource-group <rg> \
  --follow
```

### Recommended log fields for AI services
| Field | Purpose |
|---|---|
| Request/Correlation ID | Trace requests across services |
| Revision / Build ID | Matches image tag or digest |
| Model version | Know which model handled the request |
| Latency breakdown | Total + per-dependency timings |

> Log **identifiers and metadata**, not raw prompts or documents — balance debuggability with data privacy.

### Incident workflow
1. Identify which revision is active and which is failing
2. Stream logs while reproducing the issue
3. Compare config between working and failing revision
4. Apply fix → verify new revision becomes ready

---

## 6. Health Probes

### Readiness vs. Liveness
| Probe | Question | Failure action |
|---|---|---|
| **Readiness** | Can this replica take traffic now? | Stop routing to it |
| **Liveness** | Is the process healthy enough to run? | Restart the replica |

> For AI: Readiness = guard for model loading. Liveness = safety valve for deadlocks/runaway memory.

### Sample YAML config
```yaml
probes:
- type: Readiness
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 20   # time before first check
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3

- type: Liveness
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 60   # give model time to load
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

### Common probe failure causes
- Wrong port or path
- Timeout too short for model warmup
- Startup dependency (DB, cache) is slow or unavailable

> If you see liveness failures shortly after startup → liveness is **too aggressive** or the process fails fast due to misconfiguration.

---

## 7. Resource Sizing & Scaling

### Update CPU / Memory
```bash
az containerapp update \
  --name <app> --resource-group <rg> \
  --cpu <cores> --memory <size>
```

### Sizing signals
| Symptom | Fix |
|---|---|
| High latency, throttling | Increase **CPU** |
| OOM restarts | Increase **memory** |
| Cold starts too frequent | Set **minimum replicas > 0** |

### API vs. Background worker
| Workload | Strategy |
|---|---|
| Synchronous API | Set min replicas; predictable latency |
| Background / event-driven | Scale to zero is OK if drain is safe |

### Cost formula
> **Total cost = per-replica cost × number of replicas**

Minimum replicas reduce cold starts but **raise baseline cost** — use intentionally.

---

## 8. Safe Rollout — Best Practices Summary

1. **Use image digests** for production deployments
2. **Verify revision health** (status + probes + logs) before shifting traffic
3. **Deactivate first**, investigate second
4. **Define a retention strategy** (`--max-inactive-revisions`)
5. **Tune probes** to match real startup time (especially for model loading)
6. **Change one variable at a time** when optimizing resources
7. **Reassess after every model version change** — startup time, memory, and latency can all shift

---
