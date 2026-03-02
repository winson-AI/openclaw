# OpenClaw Technology Stack Deep-Dive

## 1. Language & Runtime

```mermaid
graph LR
    subgraph Core["Core (TypeScript ESM)"]
        Node["Node.js ≥22<br/>Production runtime"]
        Bun["Bun<br/>Dev & scripts"]
    end

    subgraph Apple["Apple (Swift)"]
        Swift62["Swift 6.2<br/>macOS 15+"]
        Swift6["Swift 6<br/>iOS 18+ / watchOS 11+"]
    end

    subgraph Android["Android"]
        Kotlin["Kotlin<br/>Jetpack Compose"]
    end

    subgraph Shared["Shared"]
        Kit["OpenClawKit<br/>Swift Package<br/>macOS · iOS · watchOS"]
    end

    Core --- Shared
    Apple --- Shared

    style Core fill:#f0f7e8,stroke:#5a9e3a
    style Apple fill:#e8f4fd,stroke:#4a90d9
    style Android fill:#fff3e0,stroke:#e8a030
    style Shared fill:#f3e5f5,stroke:#7b1fa2
```

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Primary language** | TypeScript (ESM) | Strict typing, no `any` |
| **Runtime** | Node.js ≥22 | Production runtime; built output in `dist/` |
| **Alt runtime** | Bun | Supported for dev/scripts; `bun <file.ts>` / `bunx` |
| **Native apps** | Swift 6.2 (macOS/iOS/watchOS), Kotlin + Jetpack Compose (Android) | |
| **Shared Apple lib** | OpenClawKit (Swift Package) | Shared across macOS, iOS, watchOS |

---

## 2. Build System & Tooling

```mermaid
flowchart TD
    subgraph TSBuild["TypeScript Build"]
        pnpm["pnpm 10.23.0<br/>Package Manager"]
        tsdown["tsdown<br/>Bundler → dist/"]
        tsc["tsc<br/>Plugin SDK .d.ts"]
        tsgo["tsgo<br/>Fast type-checking (Go)"]
        tsx["tsx<br/>Script execution"]
        Vite["Vite 7<br/>UI dev & build"]
    end

    subgraph AppleBuild["Apple Build"]
        SPM["Swift Package Manager<br/>macOS deps"]
        XcodeGen["XcodeGen<br/>iOS project gen"]
    end

    subgraph AndroidBuild["Android Build"]
        Gradle["Gradle (Kotlin DSL)<br/>Java 17"]
    end

    pnpm --> tsdown
    pnpm --> tsc
    pnpm --> tsgo
    pnpm --> tsx
    pnpm --> Vite

    style TSBuild fill:#f0f7e8,stroke:#5a9e3a
    style AppleBuild fill:#e8f4fd,stroke:#4a90d9
    style AndroidBuild fill:#fff3e0,stroke:#e8a030
```

| Tool | Role | Config |
|------|------|--------|
| **pnpm** (10.23.0) | Package manager | `pnpm-workspace.yaml` — monorepo: root, `ui/`, `packages/*`, `extensions/*` |
| **tsdown** | TypeScript bundler → `dist/` | `tsdown.config.ts` — multi-entry (index, entry, daemon-cli, plugin-sdk, hooks) |
| **tsc** | Type declarations for plugin SDK | `tsconfig.plugin-sdk.dts.json` |
| **tsgo** (`@typescript/native-preview`) | Fast type-checking | Native TS checker (Go-based, experimental) |
| **tsx** | TS execution for scripts | `node --import tsx scripts/*.ts` |
| **Vite 7** | UI dev server & build | `ui/vite.config.ts` |
| **XcodeGen** | iOS project generation | `apps/ios/project.yml` |
| **Swift Package Manager** | macOS app dependencies | `apps/macos/Package.swift` |
| **Gradle** (Kotlin DSL) | Android build | `apps/android/build.gradle.kts` |

---

## 3. Linting & Formatting

```mermaid
graph LR
    subgraph RustTools["Rust-based (fast)"]
        Oxlint["Oxlint<br/>TS linting (type-aware)"]
        Oxfmt["Oxfmt<br/>TS/MD formatting"]
    end

    subgraph SwiftTools["Swift"]
        SwiftLint["SwiftLint"]
        SwiftFormat["SwiftFormat"]
    end

    subgraph KotlinTools["Kotlin"]
        ktlint["ktlint"]
    end

    subgraph DocsTools["Docs"]
        mdlint["markdownlint-cli2"]
    end

    style RustTools fill:#fce4ec,stroke:#d32f2f
    style SwiftTools fill:#e8f4fd,stroke:#4a90d9
    style KotlinTools fill:#fff3e0,stroke:#e8a030
    style DocsTools fill:#f5f5f5,stroke:#9e9e9e
```

| Tool | Scope | Command |
|------|-------|---------|
| **Oxlint** | TypeScript linting (type-aware) | `pnpm lint` → `oxlint --type-aware` |
| **Oxfmt** | TypeScript/Markdown formatting | `pnpm format` → `oxfmt --write` |
| **SwiftLint** | Swift linting | `pnpm lint:swift` |
| **SwiftFormat** | Swift formatting | `pnpm format:swift` |
| **ktlint** | Kotlin linting/formatting | `pnpm android:lint` / `pnpm android:format` |
| **markdownlint-cli2** | Docs markdown | `pnpm lint:docs` |

---

## 4. Testing Infrastructure

```mermaid
graph TB
    subgraph UnitLayer["Unit / Integration"]
        Unit["Unit Tests<br/>vitest.unit.config.ts<br/>V8 coverage 70%"]
        GW["Gateway Tests<br/>vitest.gateway.config.ts<br/>forked pool"]
        Ext["Extension Tests<br/>vitest.extensions.config.ts"]
        Chan["Channel Tests<br/>vitest.channels.config.ts"]
    end

    subgraph E2ELayer["E2E / Docker"]
        E2E["E2E Tests<br/>vitest.e2e.config.ts"]
        Docker["Docker E2E<br/>onboard · qr-import<br/>plugins · gateway"]
    end

    subgraph LiveLayer["Live Tests"]
        Live["Live Tests<br/>vitest.live.config.ts<br/>Real API keys"]
    end

    subgraph UILayer["UI"]
        UITest["UI Tests<br/>Vitest + Playwright browser"]
    end

    subgraph NativeLayer["Native"]
        macOSTest["macOS: SwiftTesting"]
        iOSTest["iOS: XCTest"]
        AndroidTest["Android: JUnit"]
    end

    Parallel["Custom Parallelization<br/>scripts/test-parallel.mjs"]

    UnitLayer --> Parallel
    E2ELayer --> Parallel

    style UnitLayer fill:#f0f7e8,stroke:#5a9e3a
    style E2ELayer fill:#e8eaf6,stroke:#3f51b5
    style LiveLayer fill:#fce4ec,stroke:#d32f2f
    style UILayer fill:#fff3e0,stroke:#e8a030
    style NativeLayer fill:#e8f4fd,stroke:#4a90d9
```

| Layer | Framework | Config |
|-------|-----------|--------|
| **Unit tests** | Vitest + V8 coverage (70% threshold) | `vitest.unit.config.ts` |
| **Gateway tests** | Vitest (forked pool) | `vitest.gateway.config.ts` |
| **Extension tests** | Vitest | `vitest.extensions.config.ts` |
| **Channel tests** | Vitest | `vitest.channels.config.ts` |
| **E2E tests** | Vitest + Docker scripts | `vitest.e2e.config.ts`, `scripts/e2e/*.sh` |
| **Live tests** | Vitest with real API keys | `vitest.live.config.ts` |
| **UI tests** | Vitest + Playwright browser | `ui/vitest.config.ts` |
| **Parallelization** | Custom `scripts/test-parallel.mjs` | Isolated file lists for heavy suites |
| **macOS** | SwiftTesting | `OpenClawIPCTests` |
| **iOS** | XCTest | `OpenClawTests` |
| **Android** | JUnit | `./gradlew :app:testDebugUnitTest` |

---

## 5. Messaging Channel SDKs

```mermaid
graph TB
    subgraph BuiltIn["Built-in Channel SDKs"]
        TG["Telegram<br/>Grammy + @grammyjs/runner<br/>Polling / Webhook"]
        DC["Discord<br/>Carbon + discord-api-types<br/>WebSocket Gateway"]
        DCV["Discord Voice<br/>@discordjs/voice + opusscript"]
        SL["Slack<br/>Bolt + @slack/web-api<br/>Events API / Socket Mode"]
        WA["WhatsApp<br/>Baileys<br/>Web multi-device protocol"]
        SIG["Signal<br/>Signal CLI bridge<br/>Process exec"]
        IM["iMessage<br/>Native macOS APIs<br/>AppleScript / SPI"]
        LN["LINE<br/>@line/bot-sdk<br/>Webhook"]
    end

    subgraph Extensions["Extension Channel SDKs"]
        Teams["MS Teams<br/>Bot Framework<br/>HTTP Webhook"]
        Matrix["Matrix<br/>matrix-sdk-crypto-nodejs"]
        Feishu["Feishu/Lark<br/>@larksuiteoapi/node-sdk"]
        GChat["Google Chat<br/>google-auth-library + gaxios"]
        IRC["IRC<br/>Socket"]
    end

    style BuiltIn fill:#fff3e0,stroke:#e8a030
    style Extensions fill:#e0f2f1,stroke:#00897b
```

| Channel | SDK / Library | Integration Style |
|---------|---------------|-------------------|
| **Telegram** | Grammy (`grammy`) + `@grammyjs/runner` | Polling/webhook |
| **Discord** | Carbon (`@buape/carbon`) + `discord-api-types` | WebSocket gateway |
| **Discord Voice** | `@discordjs/voice` + `opusscript` | Voice channel audio |
| **Slack** | Bolt (`@slack/bolt`) + `@slack/web-api` | Events API / Socket Mode |
| **WhatsApp** | Baileys (`@whiskeysockets/baileys`) | Web multi-device protocol |
| **Signal** | Signal CLI bridge | Process exec |
| **iMessage** | Native macOS APIs | AppleScript / SPI |
| **LINE** | `@line/bot-sdk` | Webhook |
| **Microsoft Teams** | Bot Framework (extension) | HTTP webhook |
| **Matrix** | `@matrix-org/matrix-sdk-crypto-nodejs` (extension) | SDK |
| **Feishu/Lark** | `@larksuiteoapi/node-sdk` | SDK |
| **Google Chat** | `google-auth-library` + `gaxios` | Service account |
| **IRC** | Extension | Socket |

---

## 6. AI / LLM Integration

```mermaid
graph TB
    subgraph Abstraction["Provider Abstraction Layer"]
        Selection["Model Selection<br/>resolveModelRefFromString"]
        Auth["Auth Rotation<br/>OAuth / API key profiles"]
        Fallback["Fallback Chains<br/>provider → provider"]
    end

    subgraph AgentRuntimes["Agent Runtimes"]
        Pi["Pi Framework<br/>pi-agent-core · pi-ai<br/>pi-coding-agent · pi-tui"]
        ACP["ACP SDK<br/>@agentclientprotocol/sdk"]
        CLIAgent["CLI Agent Runner"]
    end

    subgraph Schema["Schema & Validation"]
        TypeBox["TypeBox<br/>Tool input schemas"]
        Zod["Zod<br/>Runtime validation"]
        AJV["AJV<br/>Protocol validation"]
    end

    subgraph CloudProviders["Cloud Providers"]
        direction LR
        Anthropic["Anthropic"]
        OpenAI["OpenAI"]
        Google["Google"]
        Bedrock["AWS Bedrock"]
        OpenRouter["OpenRouter"]
        vLLM["vLLM"]
        Together["Together"]
        HF["HuggingFace"]
        Groq["Groq"]
        Qwen["Qwen"]
    end

    subgraph LocalModels["Local Models"]
        Llama["node-llama-cpp<br/>(optional peer dep)"]
    end

    subgraph OAuthInteg["OAuth Integrations"]
        direction LR
        Copilot["GitHub Copilot"]
        QwenP["Qwen Portal"]
        GeminiCLI["Google Gemini CLI"]
        Chutes["Chutes"]
    end

    Abstraction --> AgentRuntimes
    AgentRuntimes --> CloudProviders
    AgentRuntimes --> LocalModels
    Abstraction --> OAuthInteg

    style Abstraction fill:#fce4ec,stroke:#d32f2f
    style AgentRuntimes fill:#f3e5f5,stroke:#7b1fa2
    style Schema fill:#e8eaf6,stroke:#3f51b5
    style CloudProviders fill:#f0f7e8,stroke:#5a9e3a
    style LocalModels fill:#fff8e1,stroke:#f9a825
    style OAuthInteg fill:#e0f2f1,stroke:#00897b
```

| Component | Technology |
|-----------|-----------|
| **Provider abstraction** | Custom multi-provider layer with model selection, auth rotation, fallback chains |
| **Embedded agent** | Pi framework (`@mariozechner/pi-agent-core`, `pi-ai`, `pi-coding-agent`, `pi-tui`) |
| **Agent Control Protocol** | ACP SDK (`@agentclientprotocol/sdk`) for external agent runtimes |
| **Tool schema** | TypeBox (`@sinclair/typebox`) — no Union types in tool schemas |
| **Schema validation** | Zod (`zod`) + AJV (`ajv`) |
| **Local models** | `node-llama-cpp` (optional peer dep) |
| **Cloud providers** | Anthropic, OpenAI, Google, AWS Bedrock, OpenRouter, vLLM, Together, HuggingFace, Groq, Qwen, etc. |
| **OAuth integrations** | GitHub Copilot, Qwen Portal, Google Gemini CLI, Chutes |

---

## 7. Web UI

```mermaid
graph LR
    subgraph WebUI["Control UI"]
        Lit["Lit 3<br/>Web Components"]
        Vite["Vite 7<br/>Build"]
        Signals["@lit-labs/signals<br/>+ @lit/context<br/>+ signal-utils"]
        I18n["I18n<br/>en · zh-CN · zh-TW · pt-BR"]
        Tests["Vitest +<br/>Playwright browser"]
    end

    Lit --> Output["dist/control-ui/<br/>Served by gateway"]

    style WebUI fill:#e8f4fd,stroke:#4a90d9
    style Output fill:#f0f7e8,stroke:#5a9e3a
```

| Component | Technology |
|-----------|-----------|
| **Framework** | Lit 3 (Web Components) |
| **Build** | Vite 7 |
| **State** | `@lit-labs/signals` + `@lit/context` + `signal-utils` |
| **I18n** | Custom i18n (en, zh-CN, zh-TW, pt-BR) |
| **Output** | `dist/control-ui/` — served by gateway at runtime |
| **Tests** | Vitest + `@vitest/browser-playwright` |

---

## 8. Native App Stack

```mermaid
graph TB
    subgraph macOSApp["macOS App (Swift 6.2)"]
        MenuBar["Menu Bar App<br/>MenuBarExtraAccess"]
        IPC["OpenClawIPC"]
        Discovery["OpenClawDiscovery<br/>Bonjour"]
        MacCLI["openclaw-mac CLI"]
        Sparkle["Sparkle<br/>Auto-update"]
        Peekaboo["Peekaboo<br/>Camera"]
    end

    subgraph iOSApp["iOS App (Swift 6)"]
        SwiftUI["SwiftUI<br/>Observation framework"]
        XcodeGen["XcodeGen"]
        ShareExt["Share Extension"]
        WatchApp["Watch App<br/>watchOS 11+"]
        SharedCode["OpenClawKit<br/>OpenClawChatUI<br/>OpenClawProtocol"]
    end

    subgraph AndroidApp["Android App (Kotlin)"]
        Compose["Jetpack Compose"]
        GradleB["Gradle (Kotlin DSL)<br/>Java 17"]
        SDK["compileSdk 36<br/>minSdk 31"]
        KotlinS["Kotlin Serialization"]
        Benchmark["Benchmark module"]
    end

    style macOSApp fill:#e8f4fd,stroke:#4a90d9
    style iOSApp fill:#e8f4fd,stroke:#4a90d9
    style AndroidApp fill:#fff3e0,stroke:#e8a030
```

### macOS (Swift 6.2)

| Component | Technology |
|-----------|-----------|
| **App type** | Menu bar app (MenuBarExtraAccess) |
| **IPC** | Custom OpenClawIPC library |
| **Discovery** | Bonjour via `@homebridge/ciao` (Node) + OpenClawDiscovery (Swift) |
| **CLI** | `openclaw-mac` companion CLI (OpenClawMacCLI) |
| **Auto-update** | Sparkle framework |
| **Camera** | Peekaboo framework |
| **Platform** | macOS 15+ |

### iOS (Swift 6)

| Component | Technology |
|-----------|-----------|
| **UI** | SwiftUI (Observation framework preferred) |
| **Project gen** | XcodeGen (`project.yml`) |
| **Targets** | Main app, Share Extension, Watch App, Watch Extension |
| **Shared code** | OpenClawKit, OpenClawChatUI, OpenClawProtocol |
| **Platform** | iOS 18+, watchOS 11+ |

### Android (Kotlin)

| Component | Technology |
|-----------|-----------|
| **UI** | Jetpack Compose |
| **Build** | Gradle (Kotlin DSL), Java 17 |
| **SDK** | compileSdk 36, minSdk 31, targetSdk 36 |
| **Serialization** | Kotlin Serialization |
| **Benchmarking** | `:benchmark` module |

---

## 9. Media & Content Processing

```mermaid
graph LR
    subgraph ImageAudio["Image & Audio"]
        Sharp["Sharp<br/>Image processing"]
        Opus["opusscript<br/>Audio codec"]
        TTS["node-edge-tts<br/>Text-to-speech"]
    end

    subgraph Documents["Documents"]
        PDF["pdfjs-dist<br/>PDF parsing"]
        MD["markdown-it<br/>Markdown rendering"]
        Read["@mozilla/readability<br/>+ linkedom"]
    end

    subgraph Archives["Archives"]
        Zip["jszip"]
        Tar["tar"]
    end

    subgraph Automation["Automation"]
        PW["playwright-core<br/>Browser automation"]
        PTY["@lydell/node-pty<br/>Terminal PTY"]
    end

    subgraph Detection["Detection"]
        FT["file-type<br/>MIME detection"]
        QR["qrcode-terminal<br/>QR codes"]
    end

    style ImageAudio fill:#fce4ec,stroke:#d32f2f
    style Documents fill:#e8eaf6,stroke:#3f51b5
    style Archives fill:#f5f5f5,stroke:#9e9e9e
    style Automation fill:#f3e5f5,stroke:#7b1fa2
    style Detection fill:#fff8e1,stroke:#f9a825
```

| Capability | Library |
|------------|---------|
| **Image processing** | Sharp (`sharp`) |
| **PDF parsing** | `pdfjs-dist` |
| **HTML readability** | `@mozilla/readability` + `linkedom` |
| **QR codes** | `qrcode-terminal` |
| **MIME detection** | `file-type` |
| **Zip handling** | `jszip` |
| **Archive extraction** | `tar` |
| **Text-to-speech** | `node-edge-tts` |
| **Browser automation** | `playwright-core` |
| **Markdown rendering** | `markdown-it` |
| **Terminal PTY** | `@lydell/node-pty` |

---

## 10. Infrastructure & Ops

```mermaid
graph TB
    subgraph Server["Server"]
        Express["Express 5<br/>HTTP server"]
        WS["ws<br/>WebSocket"]
        Undici["undici + gaxios<br/>HTTP client"]
        Proxy["https-proxy-agent"]
    end

    subgraph Daemon["Daemon"]
        launchd["launchd (macOS)"]
        systemd["systemd (Linux)"]
        Bonjour["@homebridge/ciao<br/>mDNS / Bonjour"]
        Croner["croner<br/>Cron"]
    end

    subgraph Config["Config & CLI"]
        YAML["yaml + json5<br/>Config formats"]
        Dotenv["dotenv"]
        Commander["Commander<br/>CLI framework"]
        Clack["@clack/prompts<br/>+ osc-progress"]
    end

    subgraph Output["Output"]
        tslog["tslog<br/>Logging"]
        Chalk["chalk + cli-highlight<br/>Terminal colors"]
        Tables["Custom tables<br/>src/terminal/table.ts"]
    end

    subgraph Data["Data"]
        SQLiteVec["sqlite-vec<br/>Vector search"]
        IPAddr["ipaddr.js"]
        Jiti["jiti<br/>Plugin loading"]
    end

    style Server fill:#f0f7e8,stroke:#5a9e3a
    style Daemon fill:#e8eaf6,stroke:#3f51b5
    style Config fill:#fff3e0,stroke:#e8a030
    style Output fill:#f5f5f5,stroke:#9e9e9e
    style Data fill:#f3e5f5,stroke:#7b1fa2
```

| Component | Technology |
|-----------|-----------|
| **HTTP server** | Express 5 |
| **WebSocket** | `ws` |
| **HTTP client** | `undici` + `gaxios` |
| **Proxy support** | `https-proxy-agent` |
| **Daemon** | launchd (macOS) / systemd (Linux) |
| **Service discovery** | mDNS/Bonjour (`@homebridge/ciao`) |
| **Cron** | `croner` |
| **Config format** | YAML (`yaml`) + JSON5 (`json5`) |
| **Env management** | `dotenv` |
| **CLI framework** | Commander (`commander`) |
| **CLI prompts** | `@clack/prompts` + `osc-progress` |
| **Logging** | `tslog` |
| **Syntax highlighting** | `cli-highlight` + `chalk` |
| **Terminal tables** | Custom (`src/terminal/table.ts`) |
| **Vector search** | `sqlite-vec` |
| **IP parsing** | `ipaddr.js` |
| **Plugin loading** | `jiti` (runtime TypeScript resolution) |

---

## 11. Documentation

| Component | Technology |
|-----------|-----------|
| **Platform** | Mintlify (`docs.openclaw.ai`) |
| **I18n pipeline** | Go-based (`scripts/docs-i18n/main.go`) |
| **Languages** | English (primary), zh-CN, ja-JP |
| **Glossary** | `docs/.i18n/glossary.zh-CN.json` |
| **Translation memory** | `docs/.i18n/zh-CN.tm.jsonl` |
| **Link auditing** | `scripts/docs-link-audit.mjs` |
| **Spellcheck** | `scripts/docs-spellcheck.sh` |

---

## 12. CI / Release

```mermaid
flowchart LR
    subgraph CI["CI Pipeline"]
        GHA["GitHub Actions"]
        PreCommit["Pre-commit hooks<br/>(git-hooks/ + prek)"]
        Lint["Oxlint + Oxfmt"]
        TypeCheck["tsgo type-check"]
        Tests["Vitest suite"]
        Build["tsdown build"]
    end

    subgraph Release["Release"]
        ReleaseCheck["release-check.ts"]
        Smoke["Install smoke (Docker)"]
        NPM["npm publish<br/>(1Password OTP)"]
        MacSign["macOS signing<br/>Xcode + notarytool"]
        MacDist[".app · .dmg<br/>Sparkle appcast"]
    end

    subgraph Codegen["Protocol Codegen"]
        ProtoGen["protocol-gen.ts<br/>→ JSON Schema"]
        SwiftGen["protocol-gen-swift.ts<br/>→ Swift models"]
    end

    CI --> Release
    CI --> Codegen

    style CI fill:#f0f7e8,stroke:#5a9e3a
    style Release fill:#fce4ec,stroke:#d32f2f
    style Codegen fill:#e8eaf6,stroke:#3f51b5
```

| Component | Technology |
|-----------|-----------|
| **CI** | GitHub Actions (`ci.yml`) |
| **Pre-commit** | Custom git hooks (`git-hooks/`) + `prek install` |
| **Release checks** | `scripts/release-check.ts`, `pnpm release:check` |
| **npm publishing** | Manual with 1Password OTP |
| **macOS signing** | Xcode + notarytool (`APP_STORE_CONNECT_*` env vars) |
| **macOS distribution** | `.app` bundle, `.dmg`, Sparkle appcast |
| **Version scheme** | CalVer: `YYYY.M.D` (stable), `YYYY.M.D-beta.N` (beta) |
| **Changelog** | `CHANGELOG.md` — Changes + Fixes, user-facing only |
| **Install smoke** | `pnpm test:install:smoke` (Docker) |
| **Protocol codegen** | `scripts/protocol-gen.ts` (JSON schema) + `protocol-gen-swift.ts` (Swift models) |
