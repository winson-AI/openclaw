# OpenClaw Architecture Analysis

## 1. What Is OpenClaw?

OpenClaw is a **self-hosted, single-user personal AI assistant** that runs on your own
devices and answers you across the messaging channels you already use — WhatsApp,
Telegram, Slack, Discord, Signal, iMessage, Google Chat, IRC, Matrix, Microsoft Teams,
LINE, and 10+ more. It can speak and listen on macOS/iOS/Android and renders a live
Canvas you control. The Gateway is the control plane; the product is the assistant.

**Core thesis:** a local, fast, always-on assistant that runs real tasks on a real
computer, with security as a deliberate tradeoff — strong defaults without killing
capability.

---

## 2. High-Level Architecture

```mermaid
graph TB
    subgraph NativeApps["Native Apps"]
        macOS["macOS Menu Bar + CLI<br/>(Swift 6.2)"]
        iOS["iOS App + Share Ext<br/>(Swift 6)"]
        Android["Android App<br/>(Kotlin / Compose)"]
        Watch["watchOS App"]
    end

    subgraph SharedKit["Shared Apple Layer"]
        Kit["OpenClawKit"]
        Proto["OpenClawProtocol"]
        ChatUI["OpenClawChatUI"]
    end

    macOS --> Kit
    iOS --> Kit
    Watch --> Kit
    Kit --> Proto
    Kit --> ChatUI

    NativeApps -- "HTTP / WebSocket<br/>(Gateway Protocol)" --> Gateway

    subgraph Gateway["Gateway Server (Node.js)"]
        direction TB
        Transport["Express HTTP + WebSocket<br/>AJV Schema Validation<br/>Control UI (Lit 3)"]

        Transport --> Routing
        subgraph Routing["Routing Layer"]
            RouteResolve["resolveAgentRoute()<br/>channel + account + peer → agentId + sessionKey"]
        end

        Routing --> Channels
        subgraph Channels["Channel Layer (22+ channels)"]
            direction LR
            Telegram["Telegram"]
            Discord["Discord"]
            Slack["Slack"]
            Signal["Signal"]
            WhatsApp["WhatsApp"]
            iMessage["iMessage"]
            LINE["LINE"]
            More["Matrix · MS Teams · IRC<br/>Feishu · Nostr · Zalo<br/>+ 10 more extensions"]
        end

        Channels --> AgentPipeline
        subgraph AgentPipeline["Auto-Reply / Agent Pipeline"]
            Dispatch["dispatchReplyFromConfig"]
            GetReply["getReplyFromConfig"]
            RunAgent["runAgentTurnWithFallback"]
            RouteReply["routeReply → channel send"]
            Dispatch --> GetReply --> RunAgent --> RouteReply
        end

        RunAgent --> Providers
        subgraph Providers["AI Providers"]
            direction LR
            APIKey["API-Key<br/>Anthropic · OpenAI · Google<br/>vLLM · OpenRouter · Bedrock"]
            OAuth["OAuth<br/>GitHub Copilot · Qwen<br/>Google Gemini CLI"]
            Local["Local<br/>node-llama-cpp"]
        end

        subgraph Plugins["Plugin System"]
            SDK["Plugin SDK"]
            Registry["Plugin Registry & Runtime"]
            Hooks["Hook Runner<br/>command · session · agent<br/>gateway · message"]
        end

        subgraph Services["Core Services"]
            direction LR
            Memory["Memory / Search<br/>sqlite-vec"]
            Media["Media Pipeline<br/>Sharp · PDF · TTS"]
            Browser["Browser Control<br/>Playwright"]
        end
    end

    subgraph CLI["CLI Layer"]
        Commander["Commander-based CLI<br/>onboard · configure · doctor · gateway<br/>message · agent · channels · tui<br/>Entry: openclaw.mjs → buildProgram()"]
    end

    CLI --> Gateway

    style NativeApps fill:#e8f4fd,stroke:#4a90d9
    style Gateway fill:#f0f7e8,stroke:#5a9e3a
    style Channels fill:#fff3e0,stroke:#e8a030
    style AgentPipeline fill:#fce4ec,stroke:#d32f2f
    style Providers fill:#f3e5f5,stroke:#7b1fa2
    style Plugins fill:#e0f2f1,stroke:#00897b
    style Services fill:#fff8e1,stroke:#f9a825
    style CLI fill:#e8eaf6,stroke:#3f51b5
    style SharedKit fill:#e8f4fd,stroke:#4a90d9
```

---

## 3. Module Map

### Core source (`src/`) — 55+ subdirectories

```mermaid
graph LR
    subgraph EntryPoints["Entry Points"]
        entry["entry.ts"]
        index["index.ts"]
        cli["cli/"]
        commands["commands/"]
    end

    subgraph GatewayLayer["Gateway"]
        gateway["gateway/"]
        routing["routing/"]
        channels["channels/"]
    end

    subgraph AgentLayer["Agent System"]
        agents["agents/"]
        acp["acp/"]
        autoreply["auto-reply/"]
    end

    subgraph PluginLayer["Plugin System"]
        plugins["plugins/"]
        pluginSDK["plugin-sdk/"]
        hooks["hooks/"]
    end

    subgraph Infrastructure["Infrastructure"]
        config["config/"]
        infra["infra/"]
        daemon["daemon/"]
        security["security/"]
        pairing["pairing/"]
        logging["logging/"]
    end

    subgraph Processing["Processing"]
        media["media/"]
        mediaU["media-understanding/"]
        memory["memory/"]
        browser["browser/"]
        tts["tts/"]
    end

    subgraph ChannelImpls["Channel Implementations"]
        telegram["telegram/"]
        discord["discord/"]
        slack["slack/"]
        signal["signal/"]
        imessage["imessage/"]
        web["web/ (WhatsApp)"]
        line["line/"]
    end

    EntryPoints --> GatewayLayer
    GatewayLayer --> AgentLayer
    AgentLayer --> PluginLayer
    GatewayLayer --> ChannelImpls
    AgentLayer --> Processing
    GatewayLayer --> Infrastructure

    style EntryPoints fill:#e8eaf6,stroke:#3f51b5
    style GatewayLayer fill:#f0f7e8,stroke:#5a9e3a
    style AgentLayer fill:#fce4ec,stroke:#d32f2f
    style PluginLayer fill:#e0f2f1,stroke:#00897b
    style Infrastructure fill:#f5f5f5,stroke:#9e9e9e
    style Processing fill:#fff8e1,stroke:#f9a825
    style ChannelImpls fill:#fff3e0,stroke:#e8a030
```

| Module | Responsibility |
|--------|----------------|
| `cli/` | Commander program builder, command registry, deps injection |
| `commands/` | High-level commands: onboard, configure, doctor, auth-choice |
| `gateway/` | HTTP/WS server, protocol schema (AJV), control UI, server lanes |
| `routing/` | Route resolution: (channel, account, peer) → (agentId, sessionKey) |
| `channels/` | Channel registry, dock, allowlists, plugin channel interface |
| `auto-reply/` | Reply pipeline: templating, dispatch, routing, chunking |
| `agents/` | Agent runtime: model config, auth, selection, fallback, tools, Pi runner |
| `acp/` | Agent Control Protocol runtime, session, events |
| `plugins/` | Plugin registry, runtime, services, hook runner |
| `plugin-sdk/` | Public SDK for extension authors |
| `config/` | Config loading, schema, migrations, sessions |
| `infra/` | Binaries, ports, env detection, TLS, state, heartbeat |
| `media/` | MIME detection, media store, fetch, image operations |
| `media-understanding/` | Vision/audio understanding via providers |
| `memory/` | Memory/search, embeddings, sqlite-vec |
| `browser/` | Playwright-based browser control, routes, screenshots |
| `pairing/` | Device pairing and pairing store |
| `security/` | DM policy, ACP policy |
| `hooks/` | Internal hooks and bundled hook handlers |
| `tts/` | Text-to-speech (edge-tts) |
| `tui/` | Terminal UI components and theme |
| `terminal/` | Table rendering, CLI palette |
| `logging/` | Logging subsystem (tslog) |
| `daemon/` | Daemon/service management (launchd/systemd) |
| `cron/` | Cron job scheduling |
| `process/` | Process exec, supervisor, command queue |
| `wizard/` | Onboarding wizard flow |

### Channel modules (built-in)

Each channel follows a consistent pattern: monitor, inbound handler, send, API helpers.

| Channel | Key files |
|---------|-----------|
| `telegram/` | Grammy-based; monitor, process inbound, send |
| `discord/` | Carbon-based; monitor, preprocess, send, voice |
| `slack/` | Bolt-based; monitor, prepare message |
| `signal/` | Signal CLI bridge; monitor, event handler |
| `imessage/` | Native macOS; monitor, send, probe |
| `web/` | WhatsApp via Baileys; monitor inbox, message handler |
| `line/` | LINE SDK; monitor, send, flex templates |

---

## 4. Data Flow — Message Lifecycle

```mermaid
sequenceDiagram
    participant User as User (on channel)
    participant Channel as Channel Monitor
    participant Router as Routing Layer
    participant Dispatch as Dispatcher
    participant Agent as Agent Runtime
    participant Provider as AI Provider
    participant Reply as Reply Router
    participant Outbound as Channel Send

    User->>Channel: 1. Send message
    Note over Channel: Normalize to<br/>common format

    Channel->>Router: 2. resolveAgentRoute()
    Note over Router: channel + account + peer<br/>→ agentId + sessionKey

    Router->>Dispatch: 3. dispatchReplyFromConfig()
    Note over Dispatch: Choose path:<br/>AI / ACP / Hooks

    Dispatch->>Agent: 4. runAgentTurnWithFallback()

    Agent->>Provider: 5. Model call
    Note over Agent: resolveModelRef<br/>resolveModelAuth<br/>fallback chain

    Provider-->>Agent: Response + tool calls

    loop Tool execution
        Agent->>Agent: Execute tools
        Agent->>Provider: Continue with results
        Provider-->>Agent: Final response
    end

    Agent->>Reply: 6. routeReply()
    Note over Reply: Typing indicators<br/>Chunking · Formatting<br/>Media attachment

    Reply->>Outbound: 7. Channel-specific send
    Outbound->>User: Deliver response
```

---

## 5. Key Design Patterns

```mermaid
graph TB
    subgraph Adapter["Adapter / Dock Pattern"]
        direction LR
        Core["Core Logic"] --> Dock["ChannelDock"]
        Dock --> Auth["AuthAdapter"]
        Dock --> Out["OutboundAdapter"]
        Dock --> Msg["MessagingAdapter"]
        Dock --> Grp["GroupAdapter"]
        Dock --> Mention["MentionAdapter"]
        Dock --> Thread["ThreadingAdapter"]
    end

    subgraph DI["Dependency Injection"]
        direction LR
        Deps["createDefaultDeps()"] --> CLIDeps["CLI deps bag"]
        Deps --> GWDeps["Gateway deps bag"]
        Runtime["PluginRuntime"] --> Config2["config"]
        Runtime --> System["system"]
        Runtime --> Media2["media"]
        Runtime --> Channel2["channel"]
        Runtime --> Tools["tools"]
    end

    subgraph Fallback["Agent Fallback Chains"]
        direction LR
        Primary["Primary Provider"] -->|fail| Secondary["Secondary Provider"]
        Secondary -->|fail| Tertiary["Tertiary Provider"]
    end

    subgraph Concurrency["Queue & Lane Concurrency"]
        direction TB
        Session1["Session A Queue<br/>Turn 1 → Turn 2 → Turn 3"]
        Session2["Session B Queue<br/>Turn 1 → Turn 2"]
        Lanes["Gateway Lanes (pool)"]
        Session1 --> Lanes
        Session2 --> Lanes
    end

    style Adapter fill:#e8f4fd,stroke:#4a90d9
    style DI fill:#f0f7e8,stroke:#5a9e3a
    style Fallback fill:#fce4ec,stroke:#d32f2f
    style Concurrency fill:#fff3e0,stroke:#e8a030
```

### 5.1 Adapter / Dock Pattern
Each channel exposes capabilities through a `ChannelDock` that wraps typed adapters
(Auth, Outbound, Messaging, Group, Mentions, Threading). Core logic queries the dock
without knowing which channel it is.

### 5.2 Dependency Injection
`createDefaultDeps()` builds a dependency bag passed through the CLI and gateway.
Plugins receive a `PluginRuntime` with config, system, media, channel, and tool access.

### 5.3 Routing by Session Key
Sessions are scoped to (channel + account + peer). `resolveAgentRoute` resolves this
triple to an agentId and sessionKey, supporting per-guild, per-team, and per-account
bindings.

### 5.4 Agent Fallback Chains
`runAgentTurnWithFallback` supports model/provider fallback: if the primary provider
fails, it retries with the next in the chain. Auth rotation (OAuth vs API key profiles)
is handled transparently.

### 5.5 Plugin Lifecycle
Plugins declare `openclaw.plugin.json` with id, channels, kind, and config schema.
The gateway loads plugins at startup via `loadGatewayPlugins()`. Plugins hook into
the lifecycle via the hook system (command, session, agent, gateway, message events).

### 5.6 Protocol Schema Validation
The Gateway protocol uses JSON frames validated by AJV schemas. Request/response/event
frames are strongly typed and generated from TypeScript (with Swift codegen for native
apps via `protocol-gen-swift.ts`).

### 5.7 Queue and Lane Concurrency
Per-session command queues and gateway lanes control concurrency, preventing message
interleaving and ensuring ordered processing per conversation.

---

## 6. Deployment Topology

```mermaid
graph TB
    subgraph UserMachine["User's Machine"]
        subgraph GW["Gateway (Node.js)"]
            HTTP["Express HTTP Server"]
            WS["WebSocket Server"]
            Monitors["Channel Monitors (always-on)"]
            PluginRT["Plugin Runtime"]
            ControlUI["Control UI (Lit 3)"]
        end

        Daemon["Daemon<br/>launchd (macOS) / systemd (Linux)"]
        Daemon -- "keeps running" --> GW

        subgraph MacApp["macOS Menu Bar App (Swift)"]
            Lifecycle["Gateway Lifecycle"]
            IPC["IPC with Gateway"]
            Sparkle["Sparkle Auto-Update"]
        end
        MacApp <--> GW

        subgraph MobileApps["iOS / Android Apps"]
            Voice["Voice & Camera"]
            Share["Share Extension"]
            AppProto["Gateway Protocol Client"]
        end
        MobileApps <-- "HTTP / WS" --> GW
    end

    subgraph CloudProviders["AI Provider APIs (HTTPS)"]
        direction LR
        Anthropic["Anthropic"]
        OpenAI["OpenAI"]
        Google["Google"]
        Others["OpenRouter · vLLM<br/>Together · HuggingFace<br/>Bedrock · Groq"]
    end

    GW -- "outbound HTTPS" --> CloudProviders

    style UserMachine fill:#f5f5f5,stroke:#616161
    style GW fill:#f0f7e8,stroke:#5a9e3a
    style MacApp fill:#e8f4fd,stroke:#4a90d9
    style MobileApps fill:#e8f4fd,stroke:#4a90d9
    style CloudProviders fill:#f3e5f5,stroke:#7b1fa2
```

---

## 7. Security Model

```mermaid
graph TB
    subgraph SecurityLayers["Security Layers"]
        direction TB
        Operator["Single-User Operator<br/>Gateway owner = sole user"]
        DMPolicy["DM Policy<br/>Per-channel allowlists"]
        Pairing["Device Pairing<br/>Pairing store for multi-device"]
        ACPPolicy["ACP Policy<br/>External agent runtime permissions"]
        Secrets["Secrets<br/>~/.openclaw/credentials/"]
        TLS["TLS + Tailscale<br/>Secure remote access"]
        HostEnv["Host Env Policy<br/>Generated Swift security policy"]
    end

    Operator --> DMPolicy --> Pairing --> ACPPolicy --> Secrets --> TLS --> HostEnv

    style SecurityLayers fill:#fce4ec,stroke:#d32f2f
```

- **Single-user, operator-controlled:** The gateway owner is the sole user and operator.
- **DM policy:** Configurable per-channel allowlists control who can message the assistant.
- **Pairing:** Device pairing with pairing store for multi-device access.
- **ACP policy:** Controls external agent runtime permissions.
- **Secrets:** Runtime secrets snapshot; credentials stored at `~/.openclaw/credentials/`.
- **TLS:** Optional TLS for gateway; Tailscale integration for secure remote access.
- **Host env policy:** Generated Swift security policy for native apps.
