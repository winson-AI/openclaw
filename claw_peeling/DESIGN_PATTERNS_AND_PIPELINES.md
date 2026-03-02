# OpenClaw Design Patterns & Pipelines

## 1. Messaging Pipeline — Full Trace

### 1.1 Inbound Path

Every channel implements a **monitor** function that starts the channel connection
and registers event handlers:

```mermaid
graph LR
    subgraph Monitors["Channel Monitors"]
        TG["monitorTelegramProvider()<br/>Grammy long-poll / webhook"]
        DC["monitorDiscordProvider()<br/>Carbon WebSocket gateway"]
        SL["monitorSlackProvider()<br/>Bolt Events API / Socket Mode"]
        SIG["monitorSignalProvider()<br/>Signal CLI subprocess"]
        IM["monitorIMessageProvider()<br/>macOS AppleScript polling"]
        WA["monitorWebChannel()<br/>Baileys WhatsApp multi-device"]
        LN["monitorLineProvider()<br/>LINE webhook server"]
    end

    TG --> Normalize["Normalize to<br/>common format"]
    DC --> Normalize
    SL --> Normalize
    SIG --> Normalize
    IM --> Normalize
    WA --> Normalize
    LN --> Normalize

    Normalize --> Pipeline["Dispatch Pipeline"]

    style Monitors fill:#fff3e0,stroke:#e8a030
    style Normalize fill:#e8f4fd,stroke:#4a90d9
    style Pipeline fill:#fce4ec,stroke:#d32f2f
```

Each monitor normalizes the incoming message into a common format and invokes
the dispatch pipeline.

### 1.2 Routing

```mermaid
graph LR
    Input["(channel, accountId, peer)"] --> Resolve["resolveAgentRoute()"]
    Resolve --> Output["{ agentId, sessionKey }"]

    Resolve --> Binding
    subgraph Binding["Binding Types"]
        Account["Per-account<br/>(default)"]
        Guild["Per-guild<br/>(Discord guild + roles)"]
        Team["Per-team<br/>(Slack workspace)"]
        Peer["Per-peer<br/>(DM-level override)"]
    end

    style Input fill:#e8eaf6,stroke:#3f51b5
    style Resolve fill:#f0f7e8,stroke:#5a9e3a
    style Output fill:#e8eaf6,stroke:#3f51b5
    style Binding fill:#fff8e1,stroke:#f9a825
```

The routing layer resolves which agent and session should handle a message.
Session keys are scoped as `channel:account:peer`.

### 1.3 Dispatch

```mermaid
flowchart TD
    Dispatch["dispatchReplyFromConfig(ctx, opts)"]
    Dispatch --> Decision{Execution path?}

    Decision -->|Primary| AI["AI Agent<br/>getReplyFromConfig → runAgentTurnWithFallback"]
    Decision -->|External| ACP["ACP Runtime<br/>External agent via Agent Control Protocol"]
    Decision -->|Custom| Hooks["Hooks<br/>Plugin hook handlers"]

    style Dispatch fill:#fce4ec,stroke:#d32f2f
    style AI fill:#f3e5f5,stroke:#7b1fa2
    style ACP fill:#e0f2f1,stroke:#00897b
    style Hooks fill:#fff3e0,stroke:#e8a030
```

### 1.4 Agent Execution

```mermaid
flowchart TD
    Run["runAgentTurnWithFallback"]

    Run --> ModelRef["resolveModelRef<br/>default / per-session / per-agent"]
    Run --> ModelAuth["resolveModelAuth<br/>OAuth token / API key / rotation"]

    Run --> ExecPath{Execution path}

    ExecPath -->|Primary| Pi["runEmbeddedPiAgent"]
    ExecPath -->|CLI| CLI["runCliAgent"]
    ExecPath -->|External| ACP["ACP Runtime (acpx)"]

    Pi --> PiInternals
    subgraph PiInternals["Pi Framework"]
        Prompt["System prompt construction"]
        Tools["Tool resolution & execution"]
        Memory["Memory / context injection"]
        Stream["Streaming response generation"]
        Prompt --> Tools --> Memory --> Stream
    end

    subgraph FallbackChain["Fallback Chain"]
        P1["Provider 1 (Anthropic)"] -->|fail| P2["Provider 2 (OpenAI)"]
        P2 -->|fail| P3["Provider 3 (Google)"]
    end

    Run -.-> FallbackChain

    style Run fill:#fce4ec,stroke:#d32f2f
    style PiInternals fill:#f3e5f5,stroke:#7b1fa2
    style FallbackChain fill:#fff8e1,stroke:#f9a825
```

**Fallback chain:** If the primary model/provider fails (rate limit, auth error,
network), the system retries with the next provider in the fallback chain defined
in the models config.

### 1.5 Reply Routing

```mermaid
flowchart LR
    Reply["routeReply(ctx, reply)"]

    Reply --> Dispatcher
    subgraph Dispatcher["ReplyDispatcher"]
        Typing["Typing indicators"]
        Ack["Ack reactions"]
        Chunk["Chunking<br/>(per channel limits)"]
        MediaA["Media attachment"]
        Format["Formatting<br/>MD → Telegram HTML<br/>MD → Discord MD<br/>MD → Slack mrkdwn"]
        Confirm["Delivery confirmation"]
    end

    Dispatcher --> Send["Channel-specific send<br/>sendMessageTelegram()<br/>sendMessageDiscord()<br/>sendMessageSlack()<br/>..."]

    style Reply fill:#fce4ec,stroke:#d32f2f
    style Dispatcher fill:#e8f4fd,stroke:#4a90d9
    style Send fill:#fff3e0,stroke:#e8a030
```

---

## 2. Plugin System Architecture

### 2.1 Plugin Definition

Each plugin is defined by two files:

**`openclaw.plugin.json`** — Metadata and config schema:
```json
{
  "id": "my-plugin",
  "channels": ["my-channel"],
  "kind": "channel",
  "configSchema": { ... },
  "uiHints": { ... }
}
```

**`package.json`** — Entry points and dependencies:
```json
{
  "openclaw": {
    "extensions": { "./index.ts": "..." },
    "channel": {
      "id": "my-channel",
      "label": "My Channel",
      "aliases": ["mc"]
    }
  }
}
```

### 2.2 Plugin Types

```mermaid
graph TB
    subgraph PluginTypes["Plugin Kinds"]
        direction TB
        Channel["channel<br/>Messaging channel integration<br/><i>telegram, discord, slack, matrix, msteams</i>"]
        MemoryP["memory<br/>Memory/search backend (singleton)<br/><i>memory-core, memory-lancedb</i>"]
        Tool["tool<br/>Agent tool provider<br/><i>diffs, llm-task, phone-control</i>"]
        AuthP["auth<br/>OAuth/auth provider<br/><i>copilot-proxy, qwen-portal-auth</i>"]
        Diag["diagnostics<br/>Telemetry/monitoring<br/><i>diagnostics-otel</i>"]
        VoiceP["voice<br/>Voice call handling<br/><i>voice-call, talk-voice</i>"]
        General["general<br/>Other capabilities<br/><i>acpx, device-pair, lobster</i>"]
    end

    style Channel fill:#fff3e0,stroke:#e8a030
    style MemoryP fill:#e8f4fd,stroke:#4a90d9
    style Tool fill:#f0f7e8,stroke:#5a9e3a
    style AuthP fill:#fce4ec,stroke:#d32f2f
    style Diag fill:#f5f5f5,stroke:#9e9e9e
    style VoiceP fill:#f3e5f5,stroke:#7b1fa2
    style General fill:#e0f2f1,stroke:#00897b
```

### 2.3 Plugin Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Discovery: Gateway starts
    Discovery --> Registration: openclaw.plugin.json found
    Registration --> Initialization: loadGatewayPlugins()

    state Initialization {
        [*] --> CreateRuntime: createPluginRuntime()
        CreateRuntime --> InjectDeps
        state InjectDeps {
            config: config
            system: system info & paths
            media: media pipeline
            channel: channel adapters
            tools: tool registration API
            hooks: hook subscription API
        }
    }

    Initialization --> Runtime: Plugin ready
    Runtime --> Teardown: Gateway shutdown
    Teardown --> [*]: runGlobalGatewayStopSafely()

    state Runtime {
        [*] --> HandleHooks: Respond to hook events
        HandleHooks --> ProvideServices: Provide registered services
        ProvideServices --> HandleHooks
    }
```

### 2.4 Channel Plugin Adapters

```mermaid
graph TB
    Plugin["ChannelPlugin"]
    Dock["ChannelDock<br/>(uniform interface)"]

    Plugin --> Auth["AuthAdapter<br/>Authentication<br/>Token management"]
    Plugin --> Outbound["OutboundAdapter<br/>Send messages<br/>Media & reactions"]
    Plugin --> Messaging["MessagingAdapter<br/>Message formatting<br/>Capabilities"]
    Plugin --> Group["GroupAdapter<br/>Group / channel<br/>management"]
    Plugin --> Mention["MentionAdapter<br/>@mention handling"]
    Plugin --> Threading["ThreadingAdapter<br/>Thread / reply<br/>support"]
    Plugin --> Monitor["MonitorAdapter<br/>Connection lifecycle<br/>Event handling"]

    Auth --> Dock
    Outbound --> Dock
    Messaging --> Dock
    Group --> Dock
    Mention --> Dock
    Threading --> Dock
    Monitor --> Dock

    Dock --> CoreLogic["Core Logic<br/>(channel-agnostic)"]

    style Plugin fill:#fff3e0,stroke:#e8a030
    style Dock fill:#e8f4fd,stroke:#4a90d9
    style CoreLogic fill:#f0f7e8,stroke:#5a9e3a
```

These adapters are wrapped by `ChannelDock`, which provides a uniform interface
for core logic to query channel capabilities without channel-specific code.

### 2.5 Hook System

```mermaid
flowchart TD
    Event["Event occurs"]
    Trigger["triggerInternalHook(event)"]
    Runner["Global Hook Runner"]

    Event --> Trigger --> Runner

    Runner --> CommandH["command hooks<br/>Before/after CLI<br/>command execution"]
    Runner --> SessionH["session hooks<br/>Session start, end<br/>context injection"]
    Runner --> AgentH["agent hooks<br/>Pre-turn, post-turn<br/>tool call interception"]
    Runner --> GatewayH["gateway hooks<br/>Start, stop<br/>config reload"]
    Runner --> MessageH["message hooks<br/>Pre-send, post-send<br/>delivery"]

    style Event fill:#fce4ec,stroke:#d32f2f
    style Runner fill:#f3e5f5,stroke:#7b1fa2
    style CommandH fill:#e8eaf6,stroke:#3f51b5
    style SessionH fill:#e8eaf6,stroke:#3f51b5
    style AgentH fill:#e8eaf6,stroke:#3f51b5
    style GatewayH fill:#e8eaf6,stroke:#3f51b5
    style MessageH fill:#e8eaf6,stroke:#3f51b5
```

---

## 3. Gateway Protocol

### 3.1 Transport & Frame Types

```mermaid
sequenceDiagram
    participant Client as Client (App / UI)
    participant GW as Gateway Server

    Note over Client,GW: HTTP REST (/api/*)
    Client->>GW: GET /api/status
    GW-->>Client: JSON response

    Note over Client,GW: WebSocket (bidirectional)
    Client->>GW: Request frame (invoke method)
    GW-->>Client: Response frame (result)
    GW-->>Client: Event frame (push notification)
    GW-->>Client: Event frame (status change)
    Client->>GW: Request frame (send message)
    GW-->>Client: Response frame (ack)
```

All WebSocket communication uses JSON frames validated by AJV schemas.

### 3.2 Protocol Codegen

```mermaid
flowchart TD
    Source["TypeScript Definitions<br/>src/gateway/protocol/schema.js"]

    Source -->|scripts/protocol-gen.ts| JSON["JSON Schema<br/>dist/protocol.schema.json"]
    Source -->|scripts/protocol-gen-swift.ts| Swift["Swift Models<br/>GatewayModels.swift"]

    JSON --> GWValidation["Gateway AJV Validation"]
    Swift --> macOSApp["macOS App"]
    Swift --> iOSApp["iOS App"]
    Swift --> watchApp["watchOS App"]
    JSON -.->|manual match| KotlinModels["Kotlin Models<br/>Android App"]

    style Source fill:#f0f7e8,stroke:#5a9e3a
    style JSON fill:#e8eaf6,stroke:#3f51b5
    style Swift fill:#e8f4fd,stroke:#4a90d9
    style KotlinModels fill:#fff3e0,stroke:#e8a030
```

This ensures native apps and the gateway always speak the same protocol.

---

## 4. Configuration System

### 4.1 Config Loading

```mermaid
flowchart LR
    File["~/.openclaw/config.yaml"] --> Load["Config Loader"]
    EnvVars["Environment Variables"] --> Load
    Load --> Validate["Schema Validation"]
    Validate --> Migrate["Migration<br/>(format changes)"]
    Migrate --> Defaults["Default Injection"]
    Defaults --> Config["Final Config Object"]

    style File fill:#e8eaf6,stroke:#3f51b5
    style Config fill:#f0f7e8,stroke:#5a9e3a
```

### 4.2 Config Scopes

| Scope | Controls |
|-------|----------|
| **gateway** | Port, bind address, mode (local/remote), TLS |
| **channels** | Per-channel enable/disable, credentials, allowlists |
| **models** | Default model, provider configs, auth profiles, fallback chains |
| **agents** | Agent definitions, session routing bindings |
| **security** | DM policy, ACP policy, pairing |
| **plugins** | Per-plugin config matching their declared schema |

---

## 5. Concurrency Model

```mermaid
graph TB
    subgraph SessionQueues["Per-Session Command Queues"]
        direction TB
        S1["Session A<br/>whatsapp:+123:+456"]
        S1Q["Turn 1 ✅ → Turn 2 ⏳ → Turn 3 ⏸"]
        S1 --> S1Q

        S2["Session B<br/>telegram:bot:user42"]
        S2Q["Turn 1 ⏳ → Turn 2 ⏸"]
        S2 --> S2Q

        S3["Session C<br/>discord:guild:chan"]
        S3Q["Turn 1 ⏳"]
        S3 --> S3Q
    end

    subgraph LanePool["Gateway Lanes (configurable pool)"]
        L1["Lane 1: Session A, Turn 2"]
        L2["Lane 2: Session B, Turn 1"]
        L3["Lane 3: Session C, Turn 1"]
        L4["Lane 4: (idle)"]
    end

    subgraph AgentLanes["Agent Lanes"]
        AL["Controls Pi agent<br/>execution concurrency<br/>Prevents resource exhaustion"]
    end

    SessionQueues --> LanePool
    LanePool --> AgentLanes

    style SessionQueues fill:#fff3e0,stroke:#e8a030
    style LanePool fill:#e8f4fd,stroke:#4a90d9
    style AgentLanes fill:#fce4ec,stroke:#d32f2f
```

Per-session command queues serialize turns within a session, gateway lanes
control overall concurrency, and agent lanes prevent resource exhaustion.

---

## 6. Memory & Search

```mermaid
flowchart TD
    subgraph Ingest["Ingestion"]
        Msg["Incoming Message"]
        Embed["Embedding Generation<br/>(configured model provider)"]
        Store["Vector Storage"]
        Msg --> Embed --> Store
    end

    subgraph Retrieve["Retrieval"]
        Turn["Agent Turn Starts"]
        Query["Context Retrieval Query"]
        Search["Similarity Search<br/>(top-K)"]
        Inject["Context Injection<br/>into agent prompt"]
        Turn --> Query --> Search --> Inject
    end

    Store -.-> Search

    subgraph Backends["Memory Backends (singleton slot)"]
        Core["memory-core<br/>SQLite + sqlite-vec"]
        Lance["memory-lancedb<br/>LanceDB columnar"]
    end

    Store --> Backends
    Search --> Backends

    style Ingest fill:#e8f4fd,stroke:#4a90d9
    style Retrieve fill:#f0f7e8,stroke:#5a9e3a
    style Backends fill:#f3e5f5,stroke:#7b1fa2
```

---

## 7. Media Pipeline

```mermaid
flowchart TD
    Input["Inbound Media<br/>image · audio · video · PDF · link"]

    Input --> MIME["MIME Detection<br/>(file-type)"]
    MIME --> MediaStore["Media Store<br/>(local FS, keyed by hash)"]

    MediaStore --> Processing
    subgraph Processing["Processing"]
        direction TB
        Images["Images<br/>Sharp: resize, convert, optimize"]
        PDF["PDF<br/>pdfjs-dist: text extraction"]
        Links["Links<br/>Readability + linkedom"]
        Audio["Audio<br/>opusscript (Discord voice)"]
        QR["QR<br/>qrcode-terminal (pairing)"]
    end

    Processing --> Understanding
    subgraph Understanding["Media Understanding"]
        direction LR
        OpenAIV["OpenAI Vision"]
        AnthropicV["Anthropic Vision"]
        GoogleV["Google Vision"]
        GroqA["Groq Audio"]
    end

    Understanding --> Context["Context Injection<br/>into agent turn"]

    style Input fill:#fff3e0,stroke:#e8a030
    style Processing fill:#e8f4fd,stroke:#4a90d9
    style Understanding fill:#f3e5f5,stroke:#7b1fa2
    style Context fill:#f0f7e8,stroke:#5a9e3a
```

---

## 8. Cross-Platform Protocol Sharing

```mermaid
graph TB
    TS["TypeScript<br/>(single source of truth)"]

    TS --> JSONSchema["JSON Schema<br/>dist/protocol.schema.json"]
    TS --> SwiftModels["Swift Models<br/>GatewayModels.swift"]

    JSONSchema --> GWServer["Gateway Server<br/>AJV Validation"]
    JSONSchema --> WebUI["Control UI (Lit 3)"]

    SwiftModels --> SharedKit
    subgraph SharedKit["OpenClawKit (Swift Package)"]
        Protocol["OpenClawProtocol<br/>typed models"]
        Kit["OpenClawKit<br/>device & camera commands"]
        Chat["OpenClawChatUI<br/>chat composer & sessions"]
    end

    SharedKit --> macOS["macOS App"]
    SharedKit --> iOS["iOS App"]
    SharedKit --> watchOS["watchOS App"]

    TS -.->|manual| Kotlin["Kotlin Models<br/>Android App"]

    style TS fill:#f0f7e8,stroke:#5a9e3a
    style SharedKit fill:#e8f4fd,stroke:#4a90d9
    style JSONSchema fill:#e8eaf6,stroke:#3f51b5
    style SwiftModels fill:#e8f4fd,stroke:#4a90d9
```

---

## 9. Extension Catalog (35+ plugins)

```mermaid
graph TB
    subgraph ChannelPlugins["Channel Plugins (22)"]
        direction LR
        CP1["bluebubbles · discord · feishu"]
        CP2["googlechat · imessage · irc"]
        CP3["line · matrix · mattermost"]
        CP4["msteams · nextcloud-talk · nostr"]
        CP5["signal · slack · synology-chat"]
        CP6["telegram · tlon · twitch"]
        CP7["whatsapp · zalo · zalouser · webchat"]
    end

    subgraph UtilityPlugins["Utility Plugins (13+)"]
        direction LR
        UP1["acpx · copilot-proxy · device-pair"]
        UP2["diagnostics-otel · diffs · llm-task"]
        UP3["lobster · memory-core · memory-lancedb"]
        UP4["minimax-portal-auth · open-prose"]
        UP5["phone-control · qwen-portal-auth"]
        UP6["google-gemini-cli-auth · talk-voice"]
        UP7["thread-ownership · voice-call"]
    end

    style ChannelPlugins fill:#fff3e0,stroke:#e8a030
    style UtilityPlugins fill:#e0f2f1,stroke:#00897b
```

---

## 10. Key Architectural Decisions

| Decision | Rationale |
|----------|-----------|
| **TypeScript over Python/Go** | Orchestration system; hackability, ecosystem breadth, fast iteration |
| **Single-user, self-hosted** | Privacy-first; operator controls all data and permissions |
| **Channel-agnostic core** | Dock/adapter pattern lets core logic work without channel-specific code |
| **Plugin-first for optionality** | Core stays lean; capabilities ship as plugins |
| **CalVer versioning** | Date-based versions communicate freshness over semver compat promises |
| **Pi framework for agents** | Embedded agent runtime with tool use, memory, streaming |
| **Protocol codegen** | Single source of truth prevents native app / gateway protocol drift |
| **Lit over React** | Lightweight web components for the control UI; no heavy SPA framework |
| **Express 5 over Fastify** | Mature, well-known, sufficient for single-user gateway workload |
| **pnpm monorepo** | Workspace support, strict dependency resolution, disk efficiency |
| **Oxlint + Oxfmt over ESLint + Prettier** | Faster Rust-based tooling; unified lint + format pipeline |
