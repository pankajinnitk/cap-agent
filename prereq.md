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

