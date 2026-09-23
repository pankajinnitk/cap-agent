# Create Joule A2A Agent for Poetry Slam Manager

Develop a production-ready A2A agent named **`psm-joule-agent`** using **LangGraph**. This agent will enable SAP Joule to interact with the CAP Poetry Slam Manager MCP.

---

## Poetry Slam Manager Application Overview

- **Framework**: CAP Node.js
- **Authentication**: XSUAA (OAuth 2.0 client credentials)
- **MCP Endpoint**: `/mcp/poetry-slam-mcp`
- **MCP Adapter**: SAP CAP Model Context Protocol Adapter (`@cap-js/mcp ^1.1.0`)
- **MCP Tools**: Provide access to:
  - Poetry slams
  - Visitors
  - Visit bookings

---

## Architecture

The implementation must follow this architecture:

`Joule → A2A → LangGraph Agent → MCP → CAP Application`


### Key Requirements

1. **Tool Usage**: Use the tools provided by the CAP Model Context Protocol Adapter. Do not hardcode or invent MCP tool names or schemas.
2. **MCP Transport**: Use `StreamableHTTPClientTransport` from `@modelcontextprotocol/sdk` for communication with the MCP endpoint.
3. **Authentication**:
   - **Local Development**: Use environment variables for CAP URL and XSUAA client credentials.
   - **Cloud Foundry**: Resolve the `PSM_CAP_APP` destination using the BTP Destination service. Use the destination's OAuth configuration for authentication.

---

## Implementation Details

### MCP Client

- Use the official MCP SDK (`@modelcontextprotocol/sdk`) for all interactions.
- Support both JSON and SSE modes:
  - `Accept: application/json` → JSON response
  - `Accept: application/json, text/event-stream` → SSE stream
- Handle errors gracefully:
  - Destination errors
  - OAuth/401/403 errors
  - MCP initialization/tool discovery errors
  - Streaming/SSE errors
  - Request timeouts
- **URL resolution — MCP endpoint and auth token must be resolved together in a single function** that returns both `{ mcpUrl, accessToken }`:
  - **Cloud Foundry**: fetch the `PSM_CAP_APP` destination from the BTP Destination service → append `/mcp/poetry-slam-mcp` to the destination's `URL` field → fetch the OAuth token from the destination's `tokenServiceURL`. Use `destBinding.url` (XSUAA token endpoint) for the OAuth token call — **not** `destBinding.uri` (Destination Configuration API), which returns 404.
  - **Local development**: read `PSM_CAP_URL` from env (may be a base URL without the path) → always normalize by appending `/mcp/poetry-slam-mcp` if not already present → fetch token from `XSUAA_TOKEN_URL`. **Never use `PSM_CAP_URL` as-is as the MCP endpoint; always ensure the path `/mcp/poetry-slam-mcp` is appended.**
  - Do NOT have separate `getAccessToken()` and `getMcpEndpointUrl()` functions — the CF destination's base URL is only available when fetching the destination config, so both URL and token must be derived in the same code path.
- **Cache tokens with 401-retry fallback.** Fetching a fresh token on every tool call adds latency (2 extra OAuth round trips on CF). The correct pattern is:

  ```typescript
  interface TokenCache { accessToken: string; expiresAt: number; }

  // CF: destination service token cached separately from CAP app token + URL
  let _destServiceToken: TokenCache | null = null;
  let _capConnection: (TokenCache & { mcpUrl: string }) | null = null;

  // Local dev: CAP app token + normalized URL
  let _localConnection: (TokenCache & { mcpUrl: string }) | null = null;

  const isFresh = (c: TokenCache | null): c is TokenCache =>
      c !== null && Date.now() < c.expiresAt;

  function clearCaches(): void {
      _destServiceToken = null; _capConnection = null; _localConnection = null;
  }

  function isAuthError(error: unknown): boolean {
      const msg = error instanceof Error ? error.message : String(error);
      return /401|403|unauthorized|forbidden/i.test(msg);
  }
  ```

  - **Token expiry**: parse `expires_in` from the OAuth response and subtract a 5-minute safety buffer: `expiresAt = Date.now() + (expires_in - 300) * 1000`.
  - **`resolveUrlAndToken()`**: check the relevant cache (`_capConnection` on CF, `_localConnection` locally) with `isFresh()` before making any OAuth calls. On CF also cache the destination service token (`_destServiceToken`) independently so it can be reused across CAP token refreshes.
  - **`callTool()` with retry**: wrap the tool call in a two-attempt loop. On the first 401/403, call `clearCaches()` and retry once with a fresh token. Never retry more than once.

  ```typescript
  export async function callTool(name: string, args: Record<string, unknown>) {
      for (let attempt = 0; attempt <= 1; attempt++) {
          if (attempt > 0) clearCaches();
          const { mcpUrl, accessToken } = await resolveUrlAndToken();
          const client = await createMcpClient(mcpUrl, accessToken);
          try {
              return await client.callTool({ name, arguments: args });
          } catch (error) {
              if (attempt === 0 && isAuthError(error)) continue;
              throw error;
          } finally {
              await client.close();
          }
      }
      throw new Error("callTool: unreachable after retry loop");
  }
  ```

  - **`listTools()`**: also calls `resolveUrlAndToken()` — benefits from the cache automatically.
  - **MCP client instances**: still created per-call (do not reuse client instances across calls). Only tokens and the resolved URL are cached.

### Tool Discovery

- The CAP MCP adapter exposes three tools: `describe`, `query`, `call`.
- Build Zod schemas from the `inputSchema` returned by `listTools()`:
  - Map JSON Schema types (`string`, `number`, `boolean`, etc.) to Zod types.
  - Mark fields as optional/required based on the `required` array in the schema.
- Do not cache fallback tools. Retry discovery on subsequent requests if the initial discovery fails.

### LangGraph Agent

- Implement the LangGraph agent using a `StateGraph` with the following nodes:
  - **Agent Node**: Handles user messages and invokes tools.
  - **Tool Node**: Executes MCP tool calls.
- Ensure the agent:
  - Calls `query` immediately for list/show/get requests.
  - Avoids unnecessary `describe` calls.
  - Returns results in a structured format (e.g., markdown tables).
- **System prompt must be directive, not conversational.** Explicitly instruct the LLM:
  - Never greet the user or ask clarifying questions before calling a tool.
  - For any list/show/retrieve request, call `query` immediately as the first action — no text response first.
  - Never call `describe` before `query`.
  - Include entity name hints in the prompt (e.g., `PoetrySlams`, `Visitors`, `Visits`) so the agent can call `query` without guessing schema.
- **The `query` tool requires a `cql` parameter — always a full CQL SELECT statement.** Never call `query({ entity: "..." })` — that is invalid and will fail silently. Always use:
  ```
  query({ cql: "SELECT from <Entity> { <fields> } WHERE <condition> ORDER BY <field>" })
  ```
- **Include the full entity schema in the system prompt** with field names and types so the LLM can build correct CQL for any user request without guessing:
  ```
  PoetrySlams: ID (UUID), title (String), description (String), dateTime (Timestamp),
               status_code (Integer), freeVisitorSeats (Integer), maxVisitorsNumber (Integer),
               visitorsFeeAmount (Decimal), visitorsFeeCurrency_code (String),
               createdAt (Timestamp), createdBy (String), modifiedAt (Timestamp)
  Visitors: ID (UUID), name (String), email (String), country (String), createdAt (Timestamp)
  Visits: ID (UUID), visitor_ID (UUID), poetrySlam_ID (UUID), artisticValue (Integer), worldWideAudience (Integer), createdAt (Timestamp)
  ```
- **Type constraints — must be explicit in the system prompt to prevent runtime errors:**
  - `dateTime` and `createdAt` are TIMESTAMP — always use full ISO 8601 with time component: `'2026-10-01T00:00:00Z'`. Bare date strings like `'2026-10-01'` throw "Wrong input for TIMESTAMP type".
  - `status_code` is INTEGER — always filter with a number, NEVER a string. `WHERE status_code = 4` ✓ — `WHERE status = 'cancelled'` ✗ throws "Wrong input for INT type".
  - Status code mapping: `1 = inPreparation`, `2 = published`, `3 = booked`, `4 = cancelled`.
- **Include CQL examples covering all common query patterns** so the agent handles any user request:
  ```
  List all:          SELECT from PoetrySlams { ID, title, dateTime, status_code, freeVisitorSeats }
  By status:         SELECT from PoetrySlams { ID, title } WHERE status_code = 4
  By date range:     WHERE dateTime >= '2026-10-01T00:00:00Z' AND dateTime <= '2026-12-31T23:59:59Z'
  By title keyword:  WHERE title like '%Berlin%'
  By description:    WHERE description like '%festival%'
  Free seats:        WHERE freeVisitorSeats > 0
  Fully booked:      WHERE freeVisitorSeats = 0
  By fee:            WHERE visitorsFeeAmount < 100
  By currency:       WHERE visitorsFeeCurrency_code = 'EUR'
  Combined:          WHERE status_code = 2 AND freeVisitorSeats > 0 AND dateTime >= '2027-01-01T00:00:00Z'
  Count:             SELECT count(*) as total from PoetrySlams
  Sorted:            SELECT from PoetrySlams { ID, title, dateTime } ORDER BY dateTime ASC
  Visitor search:    SELECT from Visitors { ID, name, email } WHERE name like '%Smith%'
  Visits expand:     SELECT from Visits { ID, poetrySlam { title, dateTime }, visitor { name, email } }
  Visits for slam:   SELECT from Visits { ID, visitor { name } } WHERE poetrySlam_ID = '<uuid>'
  ```
- **Include an intent-to-CQL mapping** in the system prompt so natural language maps to correct CQL:
  ```
  "cancelled"              → WHERE status_code = 4
  "booked / sold out"      → WHERE status_code = 3
  "published / open"       → WHERE status_code = 2
  "in preparation / draft" → WHERE status_code = 1
  "in <month> <year>"      → WHERE dateTime >= '<year>-<MM>-01T00:00:00Z' AND dateTime <= '<year>-<MM>-<last>T23:59:59Z'
  "title contains"         → WHERE title like '%keyword%'
  "description mentions"   → WHERE description like '%keyword%'
  "seats available"        → WHERE freeVisitorSeats > 0
  "cheaper than X"         → WHERE visitorsFeeAmount < X
  "in EUR / USD"           → WHERE visitorsFeeCurrency_code = '<code>'
  "how many"               → SELECT count(*) as total from ...
  "cheapest / most expensive" → ORDER BY visitorsFeeAmount ASC/DESC
  "upcoming / latest"      → ORDER BY dateTime ASC
  "who visited / bookings" → SELECT from Visits with visitor { name } expand
  Any combination          → build the appropriate compound WHERE clause
  ```

### Joule v0.2.x Compatibility (`tasks/send`)

- Joule may send requests using the older `tasks/send` method (A2A v0.2.x). The `@a2a-js/sdk` v0.3.x only handles `message/send` — calling `tasks/send` returns `method not found`.
- Add an Express middleware that intercepts `tasks/send` before `jsonRpcHandler` and rewrites it as `message/send`:
  - `params.sessionId` / `params.contextId` → `message.contextId`
  - Part `{ type, text }` (v0.2) → `{ kind, text }` (v0.3)
  - Wrap in a valid v0.3 `message/send` envelope.
  - **Do NOT forward `params.id` as `message.taskId`** — the v0.2 task ID doesn't exist in the v0.3 `InMemoryTaskStore` yet, so setting it causes `DefaultRequestHandler` to fail with "Task not found".
- **Critical**: The SDK's `jsonRpcHandler` calls `express.json()` internally, so `req.body` is unparsed when the custom middleware runs. You must call `app.use(express.json())` **before** registering the translation middleware and `jsonRpcHandler`. The SDK's internal `express.json()` is harmless when the body is already parsed — body-parser skips re-parsing.
- **CAP bootstrap note**: In a CAP app, register `app.use(express.json())` inside `cds.on("bootstrap", (app) => { ... })` — this is the only place where the Express `app` instance is available. Register it as the very first middleware in that callback, before the `tasks/send` translation middleware and before `jsonRpcHandler`.

### Multi-Turn Context — `taskId` in `message/send`

- On subsequent turns, Joule sends native `message/send` with both `contextId` and `taskId` set on the message (obtained from the prior turn's response). However, `InMemoryTaskStore` is **ephemeral** — after a task completes or the process restarts, the `taskId` no longer exists in the store, causing `DefaultRequestHandler` to fail with "Task not found".
- **Fix**: Add a middleware before `jsonRpcHandler` that strips `taskId` from `message/send` requests while preserving `contextId`. This lets the SDK create a fresh task while the agent executor uses `contextId` to look up and continue the conversation history from the in-memory `contexts` Map.
- **Why `contextId` is sufficient for continuity**: Conversation history is maintained in a `Map<contextId, Message[]>` in the agent executor. As long as `contextId` is forwarded, the agent has the full prior conversation and can continue coherently without needing `taskId`.
- **Destination service token URL**: The destination service binding exposes two URL fields — `url` (XSUAA token endpoint) and `uri` (Destination Configuration API). Always use `destBinding.url` for OAuth token calls. Using `destBinding.uri` returns 404.

---

## Deployment

### Local Development

1. Use `.env` for CAP URL and XSUAA credentials.
2. Set `PSM_CAP_URL` to the **base URL** of the CAP app (e.g. `https://your-app.cfapps.eu10.hana.ondemand.com`). Do **not** include the MCP path — the code appends `/mcp/poetry-slam-mcp` automatically.
3. Ensure the application runs without requiring a Destination service binding.

### Cloud Foundry

1. Deploy to **SAP BTP Cloud Foundry (eu10)**.
2. Provide the following:
    - TypeScript source code
    - `package.json` with dependencies
    - MTA configuration (`mta.yaml`)
    - Required BTP service bindings
    - Deployment instructions

---

## Definition of Done

The implementation is complete when:

1. The agent can:
    - Authenticate with the MCP endpoint.
    - Discover and invoke MCP tools successfully.
2. Do not cache the tokens. A fresh token is fetched from the BTP Destination Service on every request.
3. The full flow is verified:
    `Joule → A2A → LangGraph Agent → MCP → CAP Application`
4. Both local execution and Cloud Foundry deployment are functional.
5. TypeScript compiles without errors:

    ```bash
    cd psm-joule-agent && npx tsc --noEmit
    ```

6. All acceptance criteria are met:
    - `tasks/send` (Joule v0.2.x) returns a completed result.
    - `message/send` (A2A v0.3.x) works as expected.
    - The agent can list poetry slams and return actual data.
    - The agent card is accessible at `/.well-known/agent.json`.

---

## BTP Service & Destination Setup

### Required BTP Services

#### 1. AI Core (for SAP GenAI Hub — both local and CF)

Create an AI Core service instance in your BTP subaccount:

1. In BTP Cockpit → Services → Service Marketplace → search **AI Core**
2. Create instance with plan **extended**
3. Create a Service Key (name it e.g. `psm-joule-agent-aicore-key`)
4. Download the service key JSON — you will need it for local development as `AICORE_SERVICE_KEY`
5. In AI Core, deploy a model (e.g. `gpt-4.1`) via the AI Launchpad or API

#### 2. XSUAA (for CAP application authentication)

The CAP Poetry Slam Manager application has an XSUAA instance bound to it. Obtain its service key:

1. In BTP Cockpit → Cloud Foundry space → Services → find the XSUAA instance bound to the CAP app
2. Create a Service Key (name it e.g. `psm-cap-xsuaa-key`)
3. From the service key JSON, extract:
   - `url` → use as base for `XSUAA_TOKEN_URL` (append `/oauth/token`)
   - `clientid` → `XSUAA_CLIENT_ID`
   - `clientsecret` → `XSUAA_CLIENT_SECRET`

---

### BTP Destinations to Create

Two destinations are required — create both in BTP Cockpit → Connectivity → Destinations.

#### Destination 1: `PSM_CAP_APP` (agent → CAP app, CF deployment only)

This destination is used by the agent running on Cloud Foundry to reach the CAP Poetry Slam Manager MCP endpoint. The agent resolves both the URL and OAuth token from this destination automatically.

| Property | Value |
|---|---|
| Name | `PSM_CAP_APP` |
| Type | HTTP |
| URL | `https://<psm-cap-app>.cfapps.<landscape>.hana.ondemand.com` |
| Proxy Type | Internet |
| Authentication | `OAuth2ClientCredentials` |
| Client ID | `<clientid from CAP app XSUAA service key>` |
| Client Secret | `<clientsecret from CAP app XSUAA service key>` |
| Token Service URL | `https://<tenant>.authentication.<landscape>.hana.ondemand.com/oauth/token` |

Additional properties:
| Property | Value |
|---|---|
| `HTML5.DynamicDestination` | `true` |

> **Note:** This destination is only needed for Cloud Foundry deployment. For local development, environment variables (`PSM_CAP_URL`, `XSUAA_*`) are used directly instead.

#### Destination 2: `PSM_JOULE_AGENT_A2A` (Joule → agent)

This destination points Joule to the deployed A2A agent. It must be in the same BTP subaccount where Joule is provisioned.

| Property | Value |
|---|---|
| Name | `PSM_JOULE_AGENT_A2A` |
| Type | HTTP |
| URL | `https://<psm-joule-agent-srv>.cfapps.<landscape>.hana.ondemand.com` |
| Proxy Type | Internet |
| Authentication | `NoAuthentication` |

Additional properties:
| Property | Value |
|---|---|
| `HTML5.DynamicDestination` | `true` |

> The URL must match the deployed CF app route — no trailing path, no `/mcp/...`.

---

## Environment Variables — Where to Get Each Value

For local development, copy `.env.example` to `.env` and fill in the following values:

| Variable | Where to get it |
|---|---|
| `PSM_CAP_URL` | Base URL of the CAP Poetry Slam Manager app (e.g. `https://psm-app.cfapps.eu10.hana.ondemand.com`). **Do not include `/mcp/poetry-slam-mcp`** — the agent appends it automatically. For local CAP, use `http://localhost:4004`. |
| `XSUAA_TOKEN_URL` | From the XSUAA service key bound to the CAP app: `"url"` field + `/oauth/token`. Example: `https://your-tenant.authentication.eu10.hana.ondemand.com/oauth/token` |
| `XSUAA_CLIENT_ID` | From the XSUAA service key: `"clientid"` field |
| `XSUAA_CLIENT_SECRET` | From the XSUAA service key: `"clientsecret"` field |
| `MODEL_NAME` | Name of the model deployed on your AI Core instance (e.g. `gpt-4.1`). Optional — defaults to `gpt-4.1`. |
| `AICORE_SERVICE_KEY` | Full JSON content of the AI Core service key (single line). The SAP AI SDK reads this automatically for local GenAI Hub access. |

### How to get the AI Core service key

```bash
# In BTP Cockpit → your CF space → Service Instances → AI Core instance → Service Keys
# Create a key and copy the full JSON, then set it as:
AICORE_SERVICE_KEY='{"clientid":"...","clientsecret":"...","url":"...","serviceurls":{"AI_API_URL":"..."}}'
```

### How to get the XSUAA service key

```bash
# Option A: BTP Cockpit → CF space → Service Instances → XSUAA instance → Service Keys
# Option B: CF CLI
cf service-key <xsuaa-instance-name> <key-name>
```

---

## Local Testing

### Start the agent locally

```bash
cd psm-joule-agent

# 1. Copy and fill in the env file
cp .env.example .env
# Edit .env with PSM_CAP_URL, XSUAA_TOKEN_URL, XSUAA_CLIENT_ID, XSUAA_CLIENT_SECRET, AICORE_SERVICE_KEY

# 2. Copy local CDS config
cp .cdsrc.sample.json .cdsrc.json

# 3. Install dependencies
npm install

# 4. Start the agent (hybrid mode — local server, remote AI Core)
npm run watch
# → http://localhost:4004
```

### Verify agent card

```bash
curl -s http://localhost:4004/.well-known/agent.json | jq .
```

### Test with curl — `message/send`

#### List all poetry slams

```bash
curl -s -X POST http://localhost:4004/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "message/send",
    "params": {
      "message": {
        "kind": "message",
        "messageId": "msg-001",
        "role": "user",
        "parts": [{ "kind": "text", "text": "Show me all poetry slams" }]
      }
    }
  }' | jq '.result.status.message.parts[0].text'
```

#### List cancelled poetry slams

```bash
curl -s -X POST http://localhost:4004/ \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": "2",
    "method": "message/send",
    "params": {
      "message": {
        "kind": "message",
        "messageId": "msg-002",
        "role": "user",
        "parts": [{ "kind": "text", "text": "Show me all cancelled poetry slams" }]
      }
    }
  }' | jq '.result.status.message.parts[0].text'
```


