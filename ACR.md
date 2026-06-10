# Azure Container Registry (ACR) — Study Notes

## What is ACR?

- **Managed private Docker registry** — stores container images, Helm charts, and OCI artifacts
- Hosted within Azure; unlike Docker Hub, gives you full control over access
- Login server URL format: `<registry-name>.azurecr.io`
- Integrates natively with **AKS**, **Azure Container Apps**, and **Azure App Service**

---

## Service Tiers

| Tier | Use Case | Key Features |
|------|----------|--------------|
| **Basic** | Dev / learning | Core push/pull, limited storage & throughput |
| **Standard** | Production | More storage & throughput than Basic |
| **Premium** | Enterprise | Geo-replication, private endpoints, content trust |

> ⚠️ **Content trust** and **private endpoints** require the **Premium** tier.

---

## Registry Hierarchy (3 levels)

### 1. Registry
- Top-level resource; hosts all content
- Provides authentication, access control, and management
- Unique login server: `myregistry.azurecr.io`

### 2. Repository
- Collection of images with the **same name but different tags**
- Supports **namespaces** using forward slashes `/` for organization:
  - `production/inference-api`
  - `staging/inference-api`
  - `ml-team/model-server`
- Namespaces enable granular access control (e.g. restrict a team to their namespace only)

### 3. Artifact
- The actual container image or content stored in a repository
- Each artifact has **tags**, **layers**, and a **manifest**

---

## Tags, Layers & Manifests

### Tags
- Human-readable version labels — format: `repository:tag`
  - Example: `inference-api:v1.2.0`
- If no tag is specified, Docker uses `latest` by default
- One artifact can have **multiple tags** (e.g. `v1.2.0` and `stable`)
- ⚠️ Tags are **mutable** — pushing a new image with the same tag moves the pointer

### Layers
- Each Dockerfile instruction that modifies the filesystem creates a new layer
- Layers are **content-addressable** (identified by a hash of their content)
- ACR **deduplicates** shared layers across images — reduces storage costs
- Especially valuable for AI workloads where many images share ML framework base layers

### Manifests
- Lists all layers and configuration for an artifact
- Identified by a **SHA-256 digest**: `sha256:abc123...`
- Digests are **immutable** — the digest never changes after an image is pushed
- ✅ Use digests in production for guaranteed reproducibility

---

## Addressing: Tag vs Digest

### By Tag
```
docker pull myregistry.azurecr.io/inference-api:v1.2.0
docker push myregistry.azurecr.io/inference-api:v1.2.0
```
- Human-readable and easy to use
- Tags can change — **not guaranteed reproducible**
- Good for development workflows

### By Digest
```
docker pull myregistry.azurecr.io/inference-api@sha256:0a2e01852872...
```
- References a specific, **immutable** image
- ✅ Use in **production deployments** where consistency across nodes is critical
- Find the digest in the Azure portal, via Azure CLI, or in the push output

---

## Best Practices

- **Use namespaces** — organize repos with `/` notation by team, environment, or project; simplifies access control
- **Plan repository structure** — group images by deployment pattern, team ownership, or lifecycle
- **Enable geo-replication** (Premium) — replicate images to deployment regions for lower pull latency and better reliability
- **Production = digest, not tag** — tags are mutable; digests guarantee the exact same image on every pull
- **Monitor storage** — implement retention policies to remove untagged manifests and prevent unnecessary costs
- **Maximize layer deduplication** — design images to share common base layers (e.g. the same Python or ML framework layer)

---

## Quick-Fire Exam Facts

- ACR = managed **private** registry (not public like Docker Hub)
- Login URL format: `<registry-name>.azurecr.io`
- **Geo-replication**, **private endpoints**, and **content trust** → Premium tier only
- Tags are **mutable**; digests (`sha256:...`) are **immutable**
- No tag specified → defaults to `latest`
- Layers are **content-addressable** (identified by content hash)
- ACR supports: Docker images, Helm charts, OCI artifacts
- Namespace separator = forward slash `/`
- Pull by **digest** for production; pull by **tag** for development

# ACR Tasks — Study Notes

## What are ACR Tasks?

- Suite of features that **offload container image builds to Azure** (no local Docker needed)
- Send source code to ACR → cloud handles the build and push
- Solves "works on my machine" problems with consistent, controlled cloud builds
- Integrates with **GitHub** and **Azure DevOps** for CI/CD pipelines
- Supports **Linux, Windows, and ARM64** (edge AI deployments)

---

## Three Main Scenarios

| Scenario | Use Case |
|----------|----------|
| **Quick tasks** | On-demand, one-off builds |
| **Automatically triggered tasks** | Continuous integration (commit, base image, schedule) |
| **Multi-step tasks** | Complex build-test-push workflows |

---

## Quick Tasks (On-Demand Builds)

- Single command builds and pushes an image — mirrors `docker build` + `docker push` but runs in Azure
- Command: `az acr build`

```bash
az acr build --registry myregistry --image inference-api:v1.0.0 .
```

- `.` at the end = current directory as build context

### Build Context Locations

| Source | Example |
|--------|---------|
| Local directory | `.` (uploads and compresses local files) |
| Git repository | `https://github.com/myorg/repo.git` |
| Remote tarball | URL to a `.tar.gz` on a web server |

```bash
# Build directly from a Git repo (no local clone needed)
az acr build --registry myregistry \
  --image inference-api:v1.0.0 \
  https://github.com/myorg/inference-api.git
```

### When to Use Quick Tasks
- Validating Dockerfile changes before committing
- Building without Docker installed locally
- Simple one-off build steps in CI/CD pipelines
- Testing new base images or dependency updates

---

## Automatically Triggered Tasks

### Source Code Triggers
- Rebuild on **commits or pull requests** to GitHub / Azure DevOps
- ACR creates a webhook in the repo that fires on commits

```bash
az acr task create \
  --registry myregistry \
  --name build-inference-api \
  --image inference-api:{{.Run.ID}} \
  --context https://github.com/myorg/inference-api.git#main \
  --file Dockerfile \
  --git-access-token $PAT
```

> ⚠️ `{{.Run.ID}}` creates a unique tag per build — use this to ensure every triggered build is distinctly identifiable.  
> ⚠️ Store PATs in **Azure Key Vault** — never hardcode in scripts.

### Base Image Triggers
- Automatically rebuilds app images when their **base image updates**
- Detects changes to the `FROM` statement in your Dockerfile
- Critical for AI apps depending on ML frameworks, CUDA drivers, PyTorch images
- Works for base images in the **same ACR registry** or public registries (Docker Hub)
- For private base images → both base and app image should be in the **same registry**

### Scheduled Triggers
- Run tasks on a **cron schedule**

```bash
az acr task create \
  --registry myregistry \
  --name nightly-build \
  --image inference-api:nightly \
  --context https://github.com/myorg/inference-api.git \
  --schedule "0 0 * * *" \
  --file Dockerfile \
  --git-access-token $PAT
```

**Use cases for scheduled triggers:**
- Nightly builds with latest dependencies
- Periodic rebuilds to pick up base image patches
- Regular security scans
- Cleanup of old untagged images

---

## Multi-Step Tasks

- Defined in **YAML** — supports sequential or parallel steps
- Each step can: build images, run containers, or execute commands
- Steps can depend on each other; conditional logic supported

```yaml
version: v1.1.0
steps:
  - build: -t {{.Run.Registry}}/inference-api:{{.Run.ID}} .
  - push:
    - {{.Run.Registry}}/inference-api:{{.Run.ID}}
  - cmd: {{.Run.Registry}}/inference-api:{{.Run.ID}} python -m pytest tests/
```

> This builds → pushes → runs tests. If tests fail, the image is flagged before deployment.

```bash
az acr task create \
  --registry myregistry \
  --name build-test-push \
  --context https://github.com/myorg/inference-api.git \
  --file acr-task.yaml \
  --git-access-token $PAT
```

**Multi-step tasks support:**
- Build-test-push workflows (validate before deploying)
- Sequential steps with dependencies
- Parallel execution of independent steps
- Conditional logic based on previous step results

---

## Running a Container for Testing

- `az acr run` executes a command inside a container from your registry
- Use `/dev/null` as context when no source files are needed

```bash
az acr run --registry myregistry \
  --cmd 'inference-api:v1.0.0 python --version' \
  /dev/null
```

**Use cases:**
- Verify an image starts correctly
- Check required tools/frameworks are present
- Smoke tests and health checks

---

## Best Practices

- **Use run variables** — `{{.Run.ID}}` or `{{.Run.Date}}` in tags for unique, traceable builds
- **Secure PATs** — store personal access tokens in **Azure Key Vault**; never commit to repos
- **Monitor logs** — `az acr task logs` shows full build output for diagnosing failures
- **Optimize build context** — use `.dockerignore` to exclude unnecessary files (speeds up upload)
- **Layer caching** — ACR caches layers between builds; put frequently changing instructions **late** in the Dockerfile to maximize cache hits

---

## Quick-Fire Exam Facts

- ACR Tasks = cloud-based builds, **no local Docker required**
- `az acr build` = quick task (on-demand)
- `az acr task create` = persistent task (triggered or scheduled)
- `az acr run` = run a container for testing
- `{{.Run.ID}}` = unique tag per build run
- Base image triggers detect changes to the `FROM` statement
- Private base images need to be in the **same registry** for auto-trigger
- Multi-step tasks defined in **YAML**
- Context `/dev/null` = no source files needed
- Store PATs in **Azure Key Vault**
- Cron schedule format used for scheduled triggers: `"0 0 * * *"` = midnight UTC daily
