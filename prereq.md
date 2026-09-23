# Local Environment Setup

Install and configure development tools required for building the Joule A2A Agent.

---

## Navigate to Your Workspace

Create a new folder as your workspace, then open a Terminal/Command Prompt and navigate to it.

```bash
cd /path/to/your/workspace
```

## 1. Install Node.js 24.x

This is the required runtime for the CAP application and build tools. Download it from [nodejs.org](https://nodejs.org), if it's not installed.

```bash
node --version
# Expected: v24.x.x
```

> [!NOTE]
> If `npm` is not recognized in the VS Code terminal, grant the required permissions to the current user:
>
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```

---

## 2. Install VS Code with Claude Code Extension

This is the primary IDE for development.

1. Download it from [code.visualstudio.com](https://code.visualstudio.com).
2. Install the **Claude Code** extension from the VS Code Marketplace.

---

## 3. Install Claude Code CLI (Optional)

This is an alternative to the VS Code extension for terminal-based workflows.

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

> [!NOTE]
> Use either the VS Code extension or the CLI. Both provide the same agent scaffolding capabilities.

---

## 4. Install Cloud Foundry CLI v8

This is required for deploying applications to SAP BTP, Cloud Foundry environment. Download it from [https://github.com/cloudfoundry/cli/releases](https://github.com/cloudfoundry/cli/releases).

```bash
cf --version
# Expected: cf version 8.x.x
```

---

## 5. Install MTA Build Tool (mbt)

This builds multi-target application archives.

```bash
npm install -g mbt
mbt --version
```

---

## 6. Install MultiApps CF CLI Plugin

This is required for deploying `.mtar` archives with `cf deploy`.

```bash
cf install-plugin multiapps
cf plugins
# Verify: multiapps plugin listed
```

---

## 7. Install SAP CDS Development Kit

CAP framework command-line tools.

```bash
npm install -g @sap/cds-dk
cds --version
```

---

## 8. Install Joule CLI

This is required for deploying Joule capabilities.

```bash
npm install -g @sap/joule-studio-cli
joule --version
```

---

## 9. Clone the Joule A2A Agent Toolkit

The [Joule A2A Agent Toolkit](https://github.com/SAP-samples/joule-a2a-agent-toolkit) provides the Claude Code plugin to build, deploy, and connect AI agents to SAP Joule using the A2A (Agent-to-Agent) protocol on SAP BTP, Cloud Foundry environment. Open Visual Studio Code and in terminal execute below command:

> [!NOTE]
> Install [GIT CLI ](https://git-scm.com/install/) to clone repository.

```bash
git clone https://github.com/SAP-samples/joule-a2a-agent-toolkit.git
# Note the local path for later use
```

---

## 10. Connect Claude Code to SAP AI Core Instance

To use Claude Code with SAP Generative AI Hub, you need to set up a LiteLLM proxy that bridges Claude Code with SAP AI Core's foundation models. This setup enables Claude Code to leverage enterprise-grade AI capabilities through your SAP BTP environment.

### Prerequisites

Before following the setup guide, ensure you have:

- An SAP BTP subaccount with **SAP AI Core** service instance.
- A **service key** for your SAP AI Core instance

### Setup Instructions

Follow the comprehensive step-by-step guide in this SAP Community blog post:

📖 **[Running Claude Code on SAP Generative AI Hub (SAP AI Core) via LiteLLM](https://community.sap.com/t5/technology-blog-posts-by-sap/running-claude-code-on-sap-generative-ai-hub-sap-ai-core-via-litellm/ba-p/14409432)**

> [!IMPORTANT]
> The blog mentions Docker Desktop, but we recommend using **Podman** as it is open-source and free for enterprise use. Download Podman Desktop from [podman-desktop.io/downloads](https://podman-desktop.io/downloads).
> Run the Podman Desktop installer with administrator privileges to ensure proper installation.

The blog covers:

- Installing and configuring the LiteLLM proxy
- Setting up authentication with SAP AI Core
- Configuring Claude Code to use the proxy endpoint
- Testing the connection

### Verification

After completing the setup, verify the connection by opening Claude Code and running a simple prompt. If configured correctly, Claude Code will route requests through LiteLLM to SAP AI Core.

---

## Troubleshoot

For any issues encountered during local environment setup, refer to the [Troubleshooting Guide](./Troubleshooting%20guide.md).

---

## Checklist

- [ ] Node.js 24.x installed
- [ ] VS Code installed with Claude Code extension
- [ ] Claude Code CLI installed (optional)
- [ ] CF CLI v8 installed
- [ ] MBT installed globally (`mbt --version`)
- [ ] MultiApps CF plugin installed (`cf plugins`)
- [ ] CDS DK installed (`cds --version`)
- [ ] Joule CLI installed (`joule --version`)
- [ ] Joule A2A Agent Toolkit cloned
- [ ] Claude Code is connected to SAP AI Core

## Conclusion

You have successfully completed the local setup required to develop the Pro Code Agent. Next, proceed to [set up Joule and the SAP Build Work Zone](./Setup-Joule-Workzone.md) in the `HOW-PA Joule Agent nnn` subaccount.


# install NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash
source ~/.zshrc

# check verion of nvm
nvm --version

# install local node v24 into ~/.nvm/*
nvm install 24

# check path for global package installation
npm root -g

# install and check MTA build tool
npm install -g mbt
mbt --version


# SAP Generative AI Hub + LiteLLM + Claude Code — Windows PowerShell Runbook

> **Reference:** [SAP Blog — Joule A2A: Connect Code-Based Agents into Joule](https://community.sap.com/t5/technology-blog-posts-by-sap/joule-a2a-connect-code-based-agents-into-joule/ba-p/14329279)
>
> **Environment:** Windows 10 · Docker Desktop · PowerShell · WSL 2
> **Project directory:** `C:\Users\hp\Documents\litellm-sapaicore`

---

## Architecture Overview

```
Claude Code
    │
    │  ANTHROPIC_BASE_URL = http://localhost:4000
    ▼
LiteLLM Proxy (:4000)
    │
    ▼
SAP Generative AI Hub
    │
    ▼
Claude Model (Anthropic)
```

Supporting services started via Docker Compose: **LiteLLM**, **PostgreSQL**, **Prometheus**

---

## Issue 1 — Bash Syntax Does Not Work in PowerShell

**Problem:** The SAP tutorial uses Linux/macOS Bash syntax. Running it verbatim in PowerShell fails.

| Bash (tutorial) | PowerShell (use this) |
|---|---|
| `export VAR=value` | `$env:VAR="value"` |
| `curl` | `curl.exe` or `Invoke-RestMethod` |
| `\` line continuation | `` ` `` backtick |
| `0.0.0.0` (client URL) | `localhost` |

> `export` is a Linux shell built-in. PowerShell uses `$env:` prefix for environment variables.

---

## Issue 2 — WSL 2 Not Enabled / Docker Requires Virtualisation

**Problem:** Docker Desktop requires WSL 2 and Windows Hypervisor. Run the following **once** from an **Administrator Command Prompt**, restarting as indicated.

```cmd
:: Step 1 — Enable WSL (restart after)
wsl.exe --install --no-distribution
shutdown /r /t 0

:: Step 2 — After reboot: Enable required Windows features
dism /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
dism /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart

:: Step 3 — Set WSL 2 as default
wsl --set-default-version 2

:: Step 4 — Enable hypervisor auto-launch
bcdedit /set hypervisorlaunchtype auto

:: Step 5 — Restart Windows
shutdown /r /t 0

:: Step 6 — Install Ubuntu (required for Docker WSL 2 engine)
wsl --install -d Ubuntu
:: Restart again if prompted: shutdown /r /t 0
```

**Verify (Administrator CMD after all reboots):**
```cmd
wsl --status
wsl -l -v
dism /online /get-featureinfo /featurename:VirtualMachinePlatform
systeminfo | findstr /i "Hyper-V"
bcdedit /enum {current}
```

Expected outputs:
- `Default Version: 2`
- `Ubuntu ... 2`
- `State : Enabled`
- `A hypervisor has been detected`
- `hypervisorlaunchtype    Auto`

---

## Issue 3 — Docker Desktop Not Using WSL 2 Engine

**Problem:** Docker Desktop may default to the Hyper-V backend and fail on this hardware.

**Fix (GUI):**
1. Open **Docker Desktop → Settings → General**
   - Enable: ☑ `Use the WSL 2 based engine`
2. Go to **Settings → Resources → WSL Integration**
   - Enable: ☑ `Enable integration with my default WSL distro`
   - Enable: ☑ `Ubuntu`
3. Click **Apply & Restart**

**Verify:**
```cmd
docker version
docker info
docker run hello-world
```

---

## Issue 4 — LiteLLM Docker Image Pull Fails (`main-latest` / `latest` tags)

**Problem:** Pulling `main-latest` or `latest` tags fails with a TLS decode error.

```
# DO NOT USE:
docker pull ghcr.io/berriai/litellm-database:main-latest   # ❌ tls: error decoding message
docker pull ghcr.io/berriai/litellm-database:latest         # ❌
```

**Fix — use the specific versioned tag:**
```cmd
docker pull ghcr.io/berriai/litellm-database:v1.98.0
```

**Verify:**
```cmd
docker images | findstr /i litellm
```
Expected: `ghcr.io/berriai/litellm-database    v1.98.0`

> In `docker-compose.yml`, always reference `v1.98.0` instead of `main-latest`.

---

## Issue 5 — Project Files Not Created

**Problem:** LiteLLM needs `.env`, `config.yaml`, and `docker-compose.yml` in the project directory.

```powershell
cd C:\Users\hp\Documents\litellm-sapaicore
```

**Generate random keys for LiteLLM (run twice — one value each):**
```powershell
python -c "import secrets; print(secrets.token_urlsafe(32))"
```
Or:
```cmd
openssl rand -hex 32
```

**`.env` file structure:**
```env
# You generate these:
LITELLM_MASTER_KEY="YOUR_GENERATED_KEY_1"
LITELLM_SALT_KEY="YOUR_GENERATED_KEY_2"

# SAP AI Core provides these (from BTP service key):
AICORE_SERVICE_KEY='YOUR_SAP_AI_CORE_SERVICE_KEY_JSON'
AICORE_RESOURCE_GROUP="YOUR_RESOURCE_GROUP"
```
> Never commit `.env` to Git.

**`docker-compose.yml` — use the pinned image tag:**
```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm-database:v1.98.0
    ports:
      - "4000:4000"
    volumes:
      - ./config.yaml:/app/config.yaml
    env_file:
      - .env
    command: ["--config", "/app/config.yaml", "--port", "4000"]
```
> If the SAP tutorial provides a fuller Compose file (with PostgreSQL/Prometheus), use that structure but replace the image tag with `v1.98.0`.

---

## Issue 6 — LiteLLM API Test Fails in PowerShell (Bash `curl` syntax)

**Problem:** Copying the tutorial's `curl` command into PowerShell fails due to quoting and line-continuation differences.

**Fix — use PowerShell `Invoke-RestMethod`:**
```powershell
$body = @{
    model    = "claude-opus-4-7"
    max_tokens = 1000
    messages = @(@{ role = "user"; content = "What is the capital of France?" })
} | ConvertTo-Json -Depth 10

$headers = @{ Authorization = "Bearer YOUR_LITELLM_MASTER_KEY" }

$response = Invoke-RestMethod `
    -Uri         "http://localhost:4000/v1/messages" `
    -Method      POST `
    -Headers     $headers `
    -ContentType "application/json" `
    -Body        $body

$response | ConvertTo-Json -Depth 20
```

**Alternative — `curl.exe` with a JSON file:**
```powershell
# 1. Create request.json
@'
{
  "model": "claude-opus-4-7",
  "max_tokens": 1000,
  "messages": [{ "role": "user", "content": "What is the capital of France?" }]
}
'@ | Set-Content request.json

# 2. Send request
curl.exe -X POST "http://localhost:4000/v1/messages" `
  -H "Authorization: Bearer YOUR_LITELLM_MASTER_KEY" `
  -H "Content-Type: application/json" `
  --data-binary "@request.json"
```

---

## Issue 7 — Claude Code Environment Variables Not Set (PowerShell)

**Problem:** The tutorial uses `export VAR=value` which is not valid in PowerShell.

**Fix:**
```powershell
$env:ANTHROPIC_AUTH_TOKEN = "YOUR_LITELLM_MASTER_KEY"
$env:ANTHROPIC_BASE_URL   = "http://localhost:4000"
```

**Verify:**
```powershell
echo $env:ANTHROPIC_AUTH_TOKEN
echo $env:ANTHROPIC_BASE_URL
```

> These variables are session-scoped. Re-set them each time you open a new PowerShell window, or add them to your PowerShell profile.

---

## Standard Startup Sequence (Every Restart)

```powershell
# 1. Navigate to project
cd C:\Users\hp\Documents\litellm-sapaicore

# 2. Start containers
docker compose up -d

# 3. Verify containers are running
docker compose ps

# 4. Check port is open
Test-NetConnection localhost -Port 4000   # Expect: TcpTestSucceeded : True

# 5. Set Claude Code environment variables
$env:ANTHROPIC_AUTH_TOKEN = "YOUR_LITELLM_MASTER_KEY"
$env:ANTHROPIC_BASE_URL   = "http://localhost:4000"

# 6. Launch Claude Code
claude
```

---

## Useful Diagnostic Commands

```powershell
docker compose ps               # Container status
docker compose logs -f litellm  # LiteLLM live logs
docker compose logs -f          # All service logs
docker ps                       # Running containers
docker ps -a                    # All containers (including stopped)
docker images                   # Downloaded images

docker compose restart          # Restart all services
docker compose down             # Stop and remove containers
docker compose up -d            # Start containers (detached)

curl.exe http://localhost:4000/health         # LiteLLM health check
curl.exe http://localhost:4000/v1/models      # List configured models
Test-NetConnection localhost -Port 4000       # TCP connectivity check
```

---

## Setup Status Checklist

| Component | Status |
|---|---|
| Windows / PowerShell identified | ✅ |
| WSL 2 configured | ✅ |
| Docker Desktop working | ✅ |
| Docker networking & GHCR access | ✅ |
| LiteLLM image (`v1.98.0`) pulled | ✅ |
| Docker Compose configuration created | ✅ |
| `.env` concept established | ✅ |
| LiteLLM master/salt keys generated | ✅ |
| SAP AI Core credentials identified | ✅ |
| Docker Compose started successfully | ✅ |
| LiteLLM reachable on `localhost:4000` | ✅ |
| PowerShell API test approach identified | ✅ |
| Claude Code env variable syntax identified | ✅ |
| **Full chain verified (LiteLLM → SAP → Claude)** | ⚠️ Next step |


