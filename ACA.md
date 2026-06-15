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
