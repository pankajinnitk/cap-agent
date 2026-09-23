# Develop Pro Code Agent and Joule Capabilities

In this exercise, you use the [Joule A2A Agent Toolkit](https://github.com/SAP-samples/joule-a2a-agent-toolkit) (Claude Code plugin) to scaffold the complete A2A agent project. The toolkit provides pre-built skills that guide Claude Code through generating a production-ready agent.

## Prerequisites

You have already completed the setup of your [local development environment](../ex0/Setup-Local-Environment.md) and [Joule and SAP Build Work Zone](../ex0/Setup-Joule-Workzone.md).

You need to enable SAP BTP, Cloud Foundry environment (if it is not already enabled) in the **HOW-PA Joule Agent nnn Subaccount**. 

1. Go to **BTP Cockpit → HOW-PA Joule Agent nnn → Overview**.
2. Choose **Enable Cloud Foundry**.
3. Add the following details:

    | Parameter     | Value                 |
    |---------------|-----------------------|
    | Environment   | Cloud Foundry Runtime |
    | Plan          | Standard              |
    | Landscape     | cf-eu10-005           |
    | Instance Name | prahowpa-jannn        |
    | Org Name      | prahowpa-jannn        |

4. Choose **Create**.
5. Choose **Create Space**.
6. As space name, enter **Dev** and choose **Create**.
7. Copy the **API Endpoint** url to login from cf cli.

To get the origin key of your identity provider,

1. Go to **BTP Cockpit → HOW-PA Joule Agent nnn → Security → Trust Configuration**.
2. Choose the Custom Identity Provider for Platform User **aywjhejac.accounts.ondemand.com (platform users)**.
3. Copy the **Origin Key**.

### Login to Cloud Foundry

```bash
# Target the Agent subaccount Cloud Foundry endpoint
cf login -a <API endpoint URL> --origin <Origin Key of your Identity Provider>
# Enter your BTP credentials and select the CF org and space
```
### Space member creation
1. Login into **BTP Cockpit**.
2. Navigate to **Cloud Foundry -> Space -> Dev -> Space Members** .
3. Add user `prahow-<nnn>@education.cloud.sap` with roles `Space Developer` and `Space Manager`.
4. Save it

#### Create an AI Core instance with the extended plan (if it is not already created).

1. Go to **BTP Cockpit → HOW-PA Joule Agent nnn → Services → Instances and Subscriptions**.
2. Choose **Create**.
3. For **Service**, select **SAP AI Core** (technical name: `aicore`).
4. For **Plan**, select **extended**.
5. For **Runtime Environment**, select **Cloud Foundry**.
6. For **Space**, select **dev**.
7. As the **Instance Name**, enter `psm-joule-agent-aicore`.
8. Choose **Create**.
9. Open the `psm-joule-agent-aicore` instance. Under **Service Keys**, choose **Create**.
10. As the binding name, enter `psm-joule-agent-aicore-key` and choose **Create**.

## Start Claude Code with the Plugin

The Joule A2A Agent Toolkit plugin enables Claude Code to build A2A agents and Joule capabilities. Follow the steps below to set it up:

### Step 1: Navigate to Your Workspace

Open a terminal and navigate to the directory where you have to build your project.

```bash
cd /path/to/your/workspace
```

### Step 2: Copy the Instructions File

Copy the required [Instruction.md](./Instructions.md) file into your working directory.

```bash
cp /path/to/instructions-file/Instruction.md .
```

### Step 3: Run Claude Code with the Plugin

Start Claude Code with the Joule A2A Agent Toolkit plugin loaded.

```bash
claude --plugin-dir /path/to/joule-a2a-agent-toolkit
```

Run **/skills** in claude to verify the joule a2a agent skills.

![SAP BTP Subaccount](../images/claude_skills.png)

#### Use VS Code

1. Open VS Code.
2. Open the Claude Code panel in VS Code.
3. Configure the plugin-dir setting in your workspace configuration to point to the Joule A2A Agent Toolkit directory.

---

## Build the Agent using the Plugin

Give Claude Code the following prompt. Claude in turn loads the joule-a2a-agent skill (create-agent) and starts building the agent.

> [!IMPORTANT]
> Verify that you are targeting the correct Cloud Foundry organization and space using `cf target`, and ensure the appropriate Joule instance is selected using `joule status` to prevent configuration issues.

```text
Build the joule A2A agent using @Instructions.md
```

> [!NOTE]
> **What is `Instructions.md`?**
> The [instructions file](./Instructions.md) is a structured specification that Claude Code reads as context for building the agent. It is needed because the agent has non-trivial requirements: specific architecture decisions, known edge cases, and strict constraints. These are too complex to express in a single chat prompt. By referencing it with `@Instructions.md`, Claude Code gets the full specification upfront and can generate consistent, production-ready code in one pass.
>
> The file covers:
>
> - **Architecture** — the required `Joule → A2A → LangGraph → MCP → CAP` flow
> - **MCP client** — how to resolve the endpoint and token together, and why tokens must never be cached
> - **LangGraph agent** — `StateGraph` node structure and system prompt rules
> - **Joule v0.2.x compatibility** — middleware to translate legacy `tasks/send` to `message/send`
> - **Multi-turn context** — how to strip `taskId` while preserving `contextId` for conversation continuity
> - **Deployment** — local `.env` setup and Cloud Foundry MTA configuration
> - **Definition of Done** — acceptance criteria the generated code must satisfy

**What is generated:**

```
psm-joule-agent/
├── srv/
│   ├── server.ts              # CAP bootstrap + A2A Express endpoints
│   ├── agent-executor.ts      # LangGraph StateGraph + MCP client
│   ├── service.cds            # CDS service definition
│   └── utils/
│       ├── prompts.ts         # System prompt with CQN format guidance
│       ├── a2aToLangchain.ts  # A2A → LangChain message converter
│       ├── a2a-operations.ts  # A2A event helpers
│       └── helpers.ts
├── joule-capability/
│   ├── capability.sapdas.yaml
│   ├── capability_context.yaml
│   ├── da.sapdas.yaml
│   ├── functions/call_agent.yaml
│   └── scenarios/invoke_agent.yaml
├── package.json
├── tsconfig.json
├── mta.yaml
├── .cdsrc.json
└── .env.example
```

### Overview of Generated Objects

**`srv/server.ts`** — the entry point. It bootstraps the CAP server, registers the A2A JSON-RPC endpoints (`message/send`, `tasks/send`, and `/.well-known/agent.json`), and wires up the agent executor. All incoming requests from Joule or curl arrive here first.

**`srv/agent-executor.ts`** — the brain of the agent. It connects to the CAP application's MCP server using the SAP BTP destination to discover available tools (query, describe, call_action), then builds a LangGraph state machine where the LLM decides which tools to call and in what order to answer the user's question.

**`srv/utils/prompts.ts`** — the system prompt that tells the LLM what it is, what tools are available, how the data model is structured (entities, field names, status codes), and the exact JSON format expected by the CAP MCP query tool. A well-written system prompt is critical for the agent to produce correct queries.

**`joule-capability/`** — the YAML files that register the agent as a capability inside Joule. They tell Joule what questions this agent can answer, which SAP BTP destination to use to reach the agent, and how to map user intents to the `call_agent` function. These are deployed separately using the `joule` CLI in exercise 4.

**`mta.yaml`** — the MTA deployment descriptor. It defines the Cloud Foundry app module, memory allocation, service bindings (destination service), and environment variables for the deployed agent.

---

## Test and Troubleshoot the Agent

The agent is generated entirely by Claude Code. Before deploying it, verify it compiles and runs correctly locally and use Claude Code to fix any issues that come up.

Claude generates the agent in the `psm-joule-agent` folder. For all subsequent steps, switch to and continue working from the `psm-joule-agent` directory.

### Step 1:  Open VSCode and Check TypeScript compiles

Open a terminal, navigate to the directory where you have built your project and open vscode.

```bash
cd /path/to/your/workspace
open .
cd psm-joule-agent
npm install
npx tsc --noEmit
```

If there are type errors, paste the output into Claude Code:

```text
Fix the TypeScript errors: <paste output>
```

### Step 2: Configure the local environment

Copy the sample environment file that is created.

```bash
cp .env.example .env
```

Get the service broker credentials for the Poetry Slam Manager application from the consumer subaccount.

**To get the Poetry Slam Manager application credentials, follow these steps:**

1. Go to **BTP Cockpit → HOW-PA Consumer nnn Subaccount → Services → Instances & Subscriptions**.
2. View the credentials of the service broker instance named `psm-sb-sub1-full`.
3. For the `ServiceBrokerEndpoint`, get the URL of psm-servicebroker in the endpoints section.
4. Get the `ServiceBrokerClientID` and `ServiceBrokerClientSecret` from the uaa section.
5. For the `ServiceBrokerTokenServiceURL`, get the URL of the uaa section and add /oauth/token.

**To get the AI Core service credentials, follow these steps:**

1. Go to **BTP Cockpit** → **HOW-PA Joule Agent nnn Subaccount** → **Services** → **Instances & Subscriptions**
2. Choose the `psm-joule-agent-aicore` service instance.
3. Click on `psm-joule-agent-aicore-key` service binding.
4. For the `AICORE_SERVICE_KEY-serviceurls`, get the `serviceurls.AI_API_URL`.
5. For the `AICORE_SERVICE_KEY-clientid` and `AICORE_SERVICE_KEY-clientsecret`, get the `uaa.clientid` and `uaa.clientsecret`.
6. For the `AICORE_SERVICE_KEY-url`, get the `uaa.url` and add /oauth/token.

Open `.env` and fill in:

| Variable                                | Value                           |
|-----------------------------------------|---------------------------------|
| `AICORE_SERVICE_KEY`                    | `AICORE_SERVICE_KEY`            |
| `PSM_CAP_URL`                           | `ServiceBrokerEndpoint`         |
| `PSM_TOKEN_URL/XSUAA_TOKEN_URL`         | `ServiceBrokerTokenServiceURL`  |
| `PSM_CLIENT_ID/XSUAA_CLIENT_ID`         | `ServiceBrokerClientID`         |
| `PSM_CLIENT_SECRET/XSUAA_CLIENT_SECRET` | `ServiceBrokerClientSecret`     |

### Step 3: Start the agent locally

```bash
npm run watch
```

The agent starts at `http://localhost:<PORT>`. Open a new terminal and verify the agent card is served:

```bash
curl http://localhost:<PORT>/.well-known/agent.json
```

You should get a JSON response with `name`, `description`, and `capabilities`.

### Step 4: Send a test A2A request

```bash
curl -X POST http://localhost:<PORT>/ -H "Content-Type: application/json" \
  -d '{
        "jsonrpc": "2.0",
        "id": "1",
        "method": "message/send",
        "params": {
                    "id": "t1",
                    "contextId": "c1",
                    "message": {
                                  "role": "user",
                                  "parts": [ {
                                                "type":"text",
                                                "text":"Show me the list of poetry slams"
                                              } ],
                                  "messageId":"m1"
                                }
                  }
      }' | jq
```

```bash
curl -X POST http://localhost:<PORT>/ -H "Content-Type: application/json" \
  -d '{
        "jsonrpc": "2.0",
        "id": "1",
        "method": "tasks/send",
        "params": {
                    "id": "t1",
                    "contextId": "c1",
                    "message": {
                                  "role": "user",
                                  "parts": [ {
                                                "type":"text",
                                                "text":"Show me the list of poetry slams"
                                              } ],
                                  "messageId":"m1"
                                }
                  }
      }' | jq
```

Both responses should contain a completed task with actual poetry slam data from the CAP application.

### Troubleshooting with Claude Code

If the agent doesn't behave as expected, describe the problem to Claude Code and let it fix the code. Common symptoms are listed below:

| Symptom | Prompt to give to Claude Code |
|---------|---------------------------|
| `401 Unauthorized` from CAP app | The agent gets 401 from the MCP endpoint. Check that tokens are fetched fresh on every tool call and not cached. |
| `method not found` for `tasks/send` | Joule sends tasks/send but the agent returns method not found. Add middleware to translate tasks/send to message/send. |
| `Task not found` on second turn | The agent fails with Task not found on follow-up messages. Strip taskId from message/send requests while keeping contextId. |
| Agent returns no data / calls wrong tool | The agent does not call the query tool for list requests. Update the system prompt to call query immediately without calling describe first. |
| TypeScript errors after changes | Fix the TypeScript errors: <paste tsc output> |

---

## Conclusion

You now have developed the agent and tested it locally. The next steps is to [deploy the agent and Joule capabilities](./1.2-Deploy-Agent-And-Joule-Capabilities.md).
