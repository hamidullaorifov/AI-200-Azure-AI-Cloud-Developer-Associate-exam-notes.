# Web App for Containers - Exam Notes

## Container Image Sources
- **Azure Container Registry (ACR)**: recommended for production. Integrates with Entra ID, supports managed identity, geo-replication, image scanning, private network access.
- **Other registries**: any HTTPS registry supporting Docker Registry HTTP API V2 (Docker Hub, GHCR, self-hosted). Private images need server URL + username + password; public images need only image name.

## Deploying via Azure Portal
1. Create a resource → Web App → Publish: **Container**, OS: **Linux**
2. Select/create App Service plan
3. On **Container** tab, configure image source:
   - **ACR**: pick registry → choose auth method → select image & tag
   - **Other registries**: provide server URL (e.g., `https://index.docker.io/v1/` for Docker Hub), username/password, full image name & tag (e.g., `nginx:latest`)

## ACR Authentication Methods
| Method | Notes |
|---|---|
| **Managed Identity** (recommended) | No stored credentials, better auditing. Requires **AcrPull** role on registry. |
| **Admin credentials** | Simpler for dev; requires enabling admin user on ACR (`az acr update --admin-enabled true`); manual rotation needed. |

### Managed Identity Types
- **System-assigned**: tied to web app lifecycle (created/deleted with the app); good when only one app needs access.
- **User-assigned**: independent resource; reusable across multiple apps; can configure permissions before app creation.

## Deploying via CLI
Create web app with container image:
```bash
az webapp create --resource-group <rg> --plan <plan> --name <app> \
  --container-image-name myregistry.azurecr.io/docprocessor:v1
```

**Docker Hub (public):**
```bash
az webapp create ... --container-image-name nginx \
  --docker-registry-server-url https://index.docker.io/v1/
```

**Private registry:**
```bash
az webapp create ... --container-image-name myuser/myapp:latest \
  --docker-registry-server-url https://index.docker.io/v1/ \
  --docker-registry-server-user <user> \
  --docker-registry-server-password <pwd>
```
- GitHub Container Registry server URL: `https://ghcr.io`

## Deploying via VS Code
- Use Docker + Azure App Service extensions
- Build image → push to registry → right-click tag in REGISTRIES view → **Deploy Image to Azure App Service**

## Updating Container Image
```bash
az webapp config container set --resource-group <rg> --name <app> \
  --container-image-name myregistry.azurecr.io/docprocessor:v2
```
⚠️ **Key exam point**: If you push a new image to the **same tag** (e.g., `latest`), App Service does **NOT** auto-detect the change. Must **restart manually** or **enable continuous deployment**.

## Continuous Deployment (CD)
```bash
az webapp deployment container config --resource-group <rg> --name <app> --enable-cd true
```
- Returns a **webhook URL**
- Configure registry (e.g., ACR webhook on push events) to call this URL → triggers restart/pull

## Image Pull Behavior (important for troubleshooting)
| Event | Behavior |
|---|---|
| Initial deployment | Pulls all layers |
| App restart | Checks for changes, pulls only modified layers (cached layers reused if unchanged) |
| Scale out | New instances may pull full image if layers not cached |
| Pricing tier change | May allocate new infrastructure → fresh pull, affects startup time |

## Verify Deployment
```bash
az webapp show --resource-group <rg> --name <app> --query defaultHostName --output tsv
```
- Open URL in browser/curl to confirm app responds

## 🎯 Quick Exam Tips
- ACR + Managed Identity = best practice combo
- `AcrPull` role = required for managed identity to pull from ACR
- Same-tag image updates need manual restart/CD — common trick question
- `--enable-cd true` → returns webhook for automation

# Container Runtime Settings - Exam Notes

## Startup Commands
- App Service runs container using Dockerfile's **ENTRYPOINT + CMD** by default.
- Custom startup command **overrides CMD only** — ENTRYPOINT stays unchanged.
- Use cases: pass runtime args, run DB migrations, start multiple processes, override framework config.

```bash
az webapp config set --resource-group <rg> --name <app> \
  --startup-file "gunicorn --bind=0.0.0.0:8000 --workers=4 app:application"
```

For shell processing (e.g., chaining commands):
```bash
az webapp config set --resource-group <rg> --name <app> \
  --startup-file "/bin/bash -c 'python migrate.py && gunicorn app:application'"
```

## Port Configuration
- App Service auto-routes traffic if container listens on **port 80 or 8080**.
- For other ports, set **WEBSITES_PORT** app setting.
- ⚠️ Only **one port** can be exposed for HTTP requests on a custom container.
- TLS termination happens at the platform — container always receives **HTTP**, even for HTTPS clients.

```bash
az webapp config appsettings set --resource-group <rg> --name <app> \
  --settings WEBSITES_PORT=8000
```

### Common Port Defaults
| Framework | Default Port | Setting |
|---|---|---|
| Node.js (Express) | 3000 | `WEBSITES_PORT=3000` |
| Python (Gunicorn) | 8000 | `WEBSITES_PORT=8000` |
| Java (Spring Boot) | 8080 | `WEBSITES_PORT=8080` |
| ASP.NET Core | 80 | No change needed |

## Persistent Storage
- Default: container file system is **ephemeral** — lost on restart/move.
- App Service can mount persistent storage at **/home** for Linux custom containers.
- **Disabled by default** for Linux custom containers.

```bash
az webapp config appsettings set --resource-group <rg> --name <app> \
  --settings WEBSITES_ENABLE_APP_SERVICE_STORAGE=true
```

When enabled:
- `/home` persists across restarts
- Shared across all scaled-out instances
- `/home/LogFiles` stores logs
- Storage quota shared across all apps in the App Service plan
- For large/high-I/O needs → mount **Azure Storage** as extra volume

## Always-On
- Apps go idle after **~20 minutes** of inactivity → cold start on next request.
- Always-on keeps app loaded via periodic pings; eliminates cold starts.
- ⚠️ Requires **Basic tier or higher**.

```bash
az webapp config set --resource-group <rg> --name <app> --always-on true
```

Recommended for: production apps, long startup times, large images, background processes/connections.

## Health Checks
- App Service pings a configured path; expects **HTTP 200** for healthy.

```bash
az webapp config set --resource-group <rg> --name <app> \
  --generic-configurations '{"healthCheckPath": "/health"}'
```

Key behavior:
- Pings **every 1 minute**
- After **10 failed pings** (default) → instance removed from load balancer
- Prolonged unhealthy → instance may be **replaced**
- ⚠️ Changing health check config **restarts the app**

### Sample Health Endpoints (Python/Flask)
```python
@app.route('/health')
def health_check():
    return {'status': 'healthy'}, 200
```

```python
@app.route('/health')
def health_check():
    try:
        db.execute('SELECT 1')
        storage.list_containers()
        return {'status': 'healthy'}, 200
    except Exception as e:
        return {'status': 'unhealthy', 'error': str(e)}, 503
```

## 🎯 Quick Exam Tips
- Custom startup command overrides **CMD**, not ENTRYPOINT
- Only **one exposed port**; non-80/8080 → set `WEBSITES_PORT`
- `WEBSITES_ENABLE_APP_SERVICE_STORAGE=true` → enables persistent `/home`
- Always-on needs **Basic tier+**
- Health check failures threshold: **10 pings** (default), interval: **1 minute**

# App Settings & Configuration - Exam Notes

## App Settings (Environment Variables)
- App settings = name-value pairs injected as **environment variables** at container startup.
- Allows same image to run across environments with different config.
- ⚠️ **All app settings are encrypted at rest** (regardless of sensitivity).
- Names: only **letters, numbers, underscores** allowed.
- .NET nested keys: colon `:` → double underscore `__`
  - e.g., `ConnectionStrings:DefaultConnection` → `ConnectionStrings__DefaultConnection`

```bash
az webapp config appsettings set --resource-group <rg> --name <app> \
  --settings STORAGE_ACCOUNT_NAME=mystorageaccount LOG_LEVEL=INFO MAX_DOCUMENT_SIZE_MB=50
```

```python
import os
storage_account = os.environ.get('STORAGE_ACCOUNT_NAME')
log_level = os.environ.get('LOG_LEVEL', 'WARNING')
max_size = int(os.environ.get('MAX_DOCUMENT_SIZE_MB', 10))
```

## Connection Strings
- Specialized app settings for DB connectivity — App Service adds a **type-prefixed** env variable name.

| DB Type | Env Variable Prefix |
|---|---|
| SQL Server | `SQLCONNSTR_` |
| SQL Azure | `SQLAZURECONNSTR_` |
| MySQL | `MYSQLCONNSTR_` |
| PostgreSQL | `POSTGRESQLCONNSTR_` |
| Custom | `CUSTOMCONNSTR_` |

```bash
az webapp config connection-string set --resource-group <rg> --name <app> \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="Server=myserver.database.windows.net;Database=mydb;..."
```
→ Results in env var: `SQLAZURECONNSTR_DefaultConnection`

⚠️ **Note**: For non-.NET apps (Python, Node.js), prefer plain **app settings** — the prefix adds complexity without benefit.

## Bulk Editing
Export settings to JSON:
```bash
az webapp config appsettings list --resource-group <rg> --name <app> --output json > settings.json
```

JSON format:
```json
[
  { "name": "STORAGE_ACCOUNT_NAME", "value": "mystorageaccount", "slotSetting": false },
  { "name": "LOG_LEVEL", "value": "INFO", "slotSetting": false }
]
```

Import (using `@` to read from file):
```bash
az webapp config appsettings set --resource-group <rg> --name <app> --settings @settings.json
```
- Portal equivalent: **Environment variables → Advanced edit**

## Slot Settings
- Deployment slots = run different app versions side-by-side; share App Service plan compute.
- **Slot settings stay with the slot** during swap (don't move with the code).

```bash
az webapp config appsettings set --resource-group <rg> --name <app> --slot staging \
  --slot-settings ENVIRONMENT=staging API_ENDPOINT=https://api-staging.example.com
```

Use slot settings for:
- Environment identifiers (`ENVIRONMENT=production` shouldn't swap to staging)
- Environment-specific endpoints/connections
- Feature flags
- Diagnostic/logging levels

View slot settings:
```bash
az webapp config appsettings list --resource-group <rg> --name <app> \
  --query "[?slotSetting==\`true\`].name"
```

## Key Vault References
- Reference secrets stored in Azure Key Vault via app setting syntax — app reads as normal env var, no code changes.

```bash
az webapp config appsettings set --resource-group <rg> --name <app> \
  --settings API_KEY="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/api-key)"
```

### Requirements
- **Managed identity** enabled on the web app
- Managed identity granted **access to read secrets** from Key Vault
- Correct Key Vault reference syntax in setting value

### Behavior
- No version specified → resolves to **latest** secret version
- Secret rotation → App Service refreshes within **24 hours**
- App restart → forces **immediate refetch** of referenced secrets

## Verify Configuration
- SCM (Kudu) site: `https://<app-name>.scm.azurewebsites.net`
- Environment variables view: `https://<app-name>.scm.azurewebsites.net/Env`
- Shows both app settings and system-provided variables
- Alternative: diagnostic endpoint or startup logs

## 🎯 Quick Exam Tips
- All app settings **encrypted at rest** by default
- `:` → `__` for .NET nested config keys on Linux
- Connection string prefixes are **.NET-specific concern** — non-.NET use plain app settings
- **Slot settings** = "sticky" to slot, don't swap with code
- Key Vault reference needs: managed identity + secret read access + `@Microsoft.KeyVault(SecretUri=...)` syntax
- Check env vars via `/Env` on the **Kudu/SCM** site

# Container Diagnostics & Troubleshooting - Exam Notes

## Container Logs
- App Service captures **stdout/stderr** from the container.
- Enable filesystem logging:

```bash
az webapp log config --resource-group <rg> --name <app> --docker-container-logging filesystem
```

- Logs stored at `/home/LogFiles/`.

### Types of log content
- **Application output** – stdout messages
- **Error output** – stderr / exception traces
- **Framework logs** – web server startup, request logs
- **Platform messages** – container lifecycle events

## Log Stream
- Real-time container output, useful for startup debugging and live monitoring.

```bash
az webapp log tail --resource-group <rg> --name <app>
```
- Press `Ctrl+C` to stop.
- Portal: **Monitoring → Log stream**.
- Shows logs from **all instances** in scaled-out apps, each tagged with an instance identifier.

## Diagnostic Console (Kudu / SCM)
- URL: `https://<app-name>.scm.azurewebsites.net`
- Requires authentication with web app management credentials.

### Key Kudu features
- **Environment page**: view all env variables (verify app settings injection)
- **Debug console (file browser)**: access mounted storage paths (e.g., `/home`, `/home/LogFiles/`)
- **Diagnostic dump**: downloadable ZIP of logs, config, diagnostics — for offline/support analysis

⚠️ **Limitation**: SCM site is a **separate environment** — cannot browse full container filesystem or inspect running processes inside app container. Use **SSH** for in-container inspection.

## Platform Diagnostics (Azure Monitor / Log Analytics)
For long-term retention, send logs to **Log Analytics**, **Event Hubs**, or **Storage**.

```bash
resourceId=$(az webapp show -g <rg> -n <app> --query id -o tsv)
workspaceId=$(az monitor log-analytics workspace show -g <rg> -n <workspace> --query id -o tsv)

az monitor diagnostic-settings create \
    --resource "$resourceId" \
    --name myDiagnosticSettings \
    --workspace "$workspaceId" \
    --logs '[{"category":"AppServiceConsoleLogs","enabled":true},{"category":"AppServiceHTTPLogs","enabled":true}]'
```

### Log Categories
| Category | Description |
|---|---|
| `AppServiceConsoleLogs` | Container stdout/stderr |
| `AppServiceHTTPLogs` | HTTP request/response info |
| `AppServicePlatformLogs` | Container lifecycle/platform events |
| `AppServiceAppLogs` | Application-level logs (if configured) |

### Sample KQL Query
```kusto
AppServiceConsoleLogs
| where Level == "Error"
| where TimeGenerated > ago(1h)
| project TimeGenerated, ResultDescription
| order by TimeGenerated desc
```

## SSH Access
- Requires SSH **enabled within the container image** itself.

### Container requirements
- Install `openssh-server`
- SSH must listen on **port 2222**
- Root password must be **`Docker!`** (App Service requirement)
- Start SSH daemon alongside the app

### Example Dockerfile additions
```dockerfile
RUN apt-get update && apt-get install -y openssh-server \
    && echo "root:Docker!" | chpasswd

COPY sshd_config /etc/ssh/

EXPOSE 8000 2222

CMD ["/bin/bash", "-c", "service ssh start && gunicorn app:application"]
```
- Access via Portal → **Development Tools → SSH**

## Common Issues & Solutions

### 1. Container Fails to Start
**Symptoms**: app URL errors, logs show startup failures
**Diagnosis**: check log stream, verify image/credentials in registry, test container locally
**Common causes**:
- Missing required env variables
- Port mismatch (`WEBSITES_PORT` vs container's actual listening port)
- Crash during init due to missing dependencies

### 2. 404 Responses After Deployment
**Symptoms**: container starts but requests return 404
**Diagnosis**: verify `WEBSITES_PORT`, check app binds to `0.0.0.0` (not `localhost`), confirm routing
**Common causes**:
- App listening on `localhost` instead of all interfaces
- Incorrect port config
- Routing not set up for expected paths

### 3. Missing Environment Variables
**Symptoms**: logs show missing config/undefined values
**Diagnosis**: verify settings saved (portal/CLI), check Kudu Environment page, use SCM `/Env` view
**Common causes**:
- Settings not saved
- Typos in setting names
- App reads variables before App Service injects them

### 4. Slow Cold Starts
**Symptoms**: first request after idle takes much longer
**Diagnosis**: check image size (`docker images`), review startup logs, verify always-on
**Solutions**:
- Enable **always-on**
- Reduce image size (smaller base images, multi-stage builds)
- Defer heavy initialization at startup

## 🎯 Quick Exam Tips
- Container logs (stdout/stderr) → `/home/LogFiles/` via filesystem logging
- `az webapp log tail` = real-time streaming, shows instance IDs in scaled-out apps
- Kudu/SCM = separate from app container, **cannot** browse app container FS or processes
- SSH requires port **2222** + root password **`Docker!`** baked into image
- 404 troubleshooting checklist: `WEBSITES_PORT` match + bind to `0.0.0.0`
- Diagnostic settings → Log Analytics categories: Console, HTTP, Platform, App logs
