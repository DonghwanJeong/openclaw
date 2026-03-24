# OpenClaw Architecture Review: Detailed Map

This file is the technical deep-dive companion to `ARCHITECTURE_REVIEW_ABSTRACT.md`.
It is written for framework extraction, subsystem review, and architectural redesign.

## 1. Scale Snapshot

Current repository shape, from the codebase itself:

- `src/` files: about 5,103
- test files across `src`, `test`, `extensions`, `ui`: about 2,990
- extension directories: 82
- plugin manifests in `extensions/*/openclaw.plugin.json`: 80
- browser UI source files in `ui/src`: about 229

Top `src/` areas by file count:

| Area             | Approx. files | Meaning                                            |
| ---------------- | ------------: | -------------------------------------------------- |
| `src/agents`     |           974 | embedded agent runtime is a first-class subsystem  |
| `src/infra`      |           534 | shared low-level infrastructure is large           |
| `src/commands`   |           426 | CLI behavior is broad and product-facing           |
| `src/gateway`    |           393 | control plane is large and feature-rich            |
| `src/auto-reply` |           348 | reply generation and orchestration are substantial |
| `src/cli`        |           326 | CLI wiring is non-trivial                          |
| `src/config`     |           264 | config/state management is a major concern         |
| `src/plugins`    |           235 | plugin ownership and loading are central           |
| `src/plugin-sdk` |           216 | public integration surface is large                |
| `src/channels`   |           191 | channel abstraction is a real platform layer       |

Important sub-area density:

- `src/commands/doctor`: 42 files
- `src/commands/models`: 31 files
- `src/gateway/server-methods`: 69 files
- `src/gateway/server`: 29 files
- `src/gateway/protocol`: 29 files
- `src/agents/pi-embedded-runner`: 115 files
- `src/agents/tools`: 83 files
- `src/agents/sandbox`: 62 files

Interpretation:

- this is not a small CLI app with plugins
- this is a platform repo with multiple semi-independent subsystems
- any new framework extracted from it should separate kernel concerns from product concerns early

## 2. Workspace Structure

Workspace roots from `pnpm-workspace.yaml`:

- root package
- `ui`
- `packages/*`
- `extensions/*`

High-level role map:

| Path           | Role                                                            |
| -------------- | --------------------------------------------------------------- |
| `openclaw.mjs` | launcher entrypoint                                             |
| `src/`         | core runtime and product logic                                  |
| `extensions/`  | workspace plugins and official channel/provider implementations |
| `ui/`          | browser Control UI                                              |
| `apps/`        | macOS, iOS, Android shells                                      |
| `packages/`    | small compatibility packages                                    |
| `test/`        | cross-cutting tests and guardrails                              |
| `docs/`        | public architecture and user docs                               |

The repo is physically modular, but the product runtime is centered in `src/`.

## 3. The System Center: Gateway Plus Plugin Runtime

The architectural center is not the CLI.
The architectural center is:

- the Gateway server
- the plugin registry and runtime
- the embedded agent runtime

The CLI, web UI, and native apps are shells around that center.

This is consistent with both the code and the public architecture docs:

- `docs/concepts/architecture.md`
- `docs/plugins/architecture.md`

## 4. Boot and Entry Points

### 4.1 Launcher

`openclaw.mjs` does four things:

- enforces Node minimum version
- enables compile cache when available
- loads warning filter
- imports built `dist/entry.js`

It is a deployment wrapper, not the real architecture.

### 4.2 Real CLI Bootstrap

Main path:

`openclaw.mjs`
-> `src/entry.ts`
-> `src/cli/run-main.ts`

Important files:

- `src/entry.ts`
  runtime bootstrap, respawn logic, profile env handling, fast help/version path
- `src/cli/run-main.ts:82`
  real CLI entry function `runCli()`
- `src/cli/program/build-program.ts:8`
  constructs Commander program
- `src/cli/program/command-registry.ts:302`
  registers core commands lazily

Key observation:

- the CLI is deliberately lazy-loaded
- the command tree is not the business logic
- the command tree is a shell over domain modules in `src/commands/`

### 4.3 Library Export Path

`src/index.ts` is a compatibility entry and library shell.
`src/library.ts` re-exports selected functions like config loading, session store access, process helpers, and default dependency creation.

This suggests a secondary identity:

- OpenClaw is also used as a library surface in some contexts
- but the library surface is intentionally much smaller than the runtime internals

## 5. CLI Architecture

The CLI has three layers:

1. bootstrap and argv normalization
2. command-program registration
3. command behavior modules

Main files:

- `src/cli/run-main.ts`
- `src/cli/program/*`
- `src/commands/*`

Key characteristics:

- lazy command registration to reduce startup overhead
- some root-level fast paths for `--help` and `--version`
- plugin CLI commands can be added at runtime through `src/plugins/cli.ts`
- command execution still relies on config, plugin registry, gateway, sessions, and channels

Commands are not isolated micro-tools.
They are frontends into the same shared runtime graph.

## 6. Gateway Architecture

### 6.1 Public Gateway Entry

Gateway entry:

- `src/gateway/server.ts`
  exports `startGatewayServer`
- `src/gateway/server.impl.ts:362`
  contains the real startup logic

### 6.2 What Gateway Startup Actually Does

Reading `src/gateway/server.impl.ts` shows a startup sequence roughly like this:

1. read config snapshot
2. migrate legacy config if needed
3. validate startup config
4. auto-enable plugins from config and environment
5. activate secrets runtime
6. load plugins and build active registry
7. build request handlers, including dynamic plugin handlers
8. start discovery sidecars
9. start maintenance timers
10. expose Tailscale if configured
11. start browser and plugin sidecars
12. start config reloader

Relevant anchors:

- `src/gateway/server.impl.ts:380`
  initial config read
- `src/gateway/server.impl.ts:387`
  legacy migration
- `src/gateway/server.impl.ts:409`
  plugin auto-enable
- `src/gateway/server.impl.ts:760`
  discovery startup
- `src/gateway/server.impl.ts:806`
  maintenance timers
- `src/gateway/server.impl.ts:1037`
  exec approval handlers
- `src/gateway/server.impl.ts:1040`
  secrets handlers
- `src/gateway/server.impl.ts:1179`
  Tailscale exposure
- `src/gateway/server.impl.ts:1199`
  sidecars
- `src/gateway/server.impl.ts:1270`
  config reloader

Interpretation:

- the Gateway is not a thin transport
- it is a runtime kernel and lifecycle manager
- it owns initialization, policy, reload, background timers, and sidecars

### 6.3 Gateway Request Handling

Method aggregation:

- `src/gateway/server-methods.ts:68`
  `coreGatewayHandlers`

This file composes many handler sets:

- `agent`
- `agents`
- `browser`
- `channels`
- `chat`
- `config`
- `connect`
- `cron`
- `devices`
- `doctor`
- `exec-approvals`
- `health`
- `logs`
- `models`
- `nodes`
- `push`
- `send`
- `sessions`
- `skills`
- `system`
- `talk`
- `tools-catalog`
- `tts`
- `update`
- `usage`
- `voicewake`
- `web`
- `wizard`

This is a major clue for framework design:

- transport is unified
- business API is method-based
- handlers are modular
- request scoping and auth are centralized

In framework terms, this looks like:

- one control-plane protocol
- one method registry
- many feature modules plugged into the registry

## 7. Plugin Architecture

### 7.1 Static Manifest Layer

Every native plugin can expose static metadata via `openclaw.plugin.json`.

Example roles:

- channels: `extensions/telegram/openclaw.plugin.json`
- providers: `extensions/openai/openclaw.plugin.json`
- memory: `extensions/memory-core/openclaw.plugin.json`

Static manifest responsibilities:

- plugin identity
- capability declarations
- config schema
- auth choice metadata
- UI hints

The static layer exists so discovery and validation can happen before executing plugin code.

### 7.2 Runtime Registration Layer

The runtime registration API is defined in:

- `src/plugins/types.ts:1314`
  `OpenClawPluginApi`

Main registration methods:

- `registerTool`
- `registerHook`
- `registerHttpRoute`
- `registerChannel`
- `registerGatewayMethod`
- `registerCli`
- `registerService`
- `registerProvider`
- `registerSpeechProvider`
- `registerMediaUnderstandingProvider`
- `registerImageGenerationProvider`
- `registerWebSearchProvider`
- `registerCommand`
- `registerContextEngine`
- `registerMemoryPromptSection`

This is the real extensibility contract.

### 7.3 Loader Pipeline

Main loader:

- `src/plugins/loader.ts`

Important responsibilities:

- normalize plugin config
- resolve roots and cache key
- discover candidate plugins
- load manifest registry
- build registry
- import plugin modules
- call `register(api)` or `activate`
- activate the resulting registry globally

Useful anchors:

- `src/plugins/loader.ts:689`
  cache key build and activation mode logic
- `src/plugins/loader.ts:821`
  `createPluginRegistry(...)`
- `src/plugins/loader.ts:828`
  discovery
- `src/plugins/loader.ts:834`
  manifest registry load
- `src/plugins/loader.ts:1276`
  activate active registry

### 7.4 Registry Construction

Registry construction lives in:

- `src/plugins/registry.ts:247`
  `createPluginRegistry(...)`

The registry records:

- plugins
- tools
- hooks
- typed hooks
- channels
- channel setup entries
- providers
- speech providers
- media-understanding providers
- image-generation providers
- web-search providers
- gateway handlers
- HTTP routes
- CLI registrars
- services
- commands
- conversation-binding handlers
- diagnostics

This is the closest thing to a framework kernel object in the repo.

### 7.5 Runtime Surface

Plugin runtime is created by:

- `src/plugins/runtime/index.ts:184`
  `createPluginRuntime()`

It exposes runtime helpers for:

- config
- agent
- subagent
- system
- media
- image generation
- web search
- tools
- channel
- events
- logging
- state
- lazily bound TTS, STT, media-understanding, model-auth

Important design choice:

- plugin registration and plugin runtime are separate concerns
- static manifest discovery happens before runtime import
- runtime methods are late-bound and partially lazy

### 7.6 Active Runtime State

Active plugin registry global state is in:

- `src/plugins/runtime.ts`

This file manages:

- active plugin registry
- pinned HTTP route registry
- registry key and version

Framework extraction note:

- the global singleton registry is convenient
- but it is also one of the key coupling points you would likely abstract behind a container or runtime context in a new framework

## 8. Channel Architecture

### 8.1 Channel Contract

Channel plugin type:

- `src/channels/plugins/types.plugin.ts:55`
  `ChannelPlugin`

A channel can own:

- config
- setup
- pairing
- security
- groups
- mentions
- outbound
- status
- gateway methods
- auth
- elevated operations
- commands
- lifecycle hooks
- exec approvals
- allowlist logic
- bindings
- streaming
- threading
- messaging
- agent prompt behavior
- directory lookup
- resolver
- actions
- heartbeat
- agent tools

This is a very rich channel contract.

### 8.2 Bundled Channels

Official bundled channels are statically imported in:

- `src/channels/plugins/bundled.ts`

This file exposes:

- `bundledChannelPlugins`
- `bundledChannelSetupPlugins`
- `getBundledChannelPlugin()`
- runtime setters for selected channels

Observed from manifests:

- about 21 plugins declare channel capabilities

Important architectural implication:

- official channels are implemented as workspace plugins
- but core still knows about official bundled channel entrypoints
- this is a hybrid between pure plugin architecture and curated first-party platform

### 8.3 Shared Message Tool Boundary

From the public plugin architecture docs and current code layout:

- core owns shared message-tool orchestration
- channel plugins own channel-specific discovery and execution

This is a strong framework pattern:

- central action host
- capability-specific adapters
- domain plugins contribute behavior without owning the entire outer orchestration

## 9. Agent Runtime Architecture

This is the largest subsystem in the repo.

### 9.1 Center of Agent Execution

Main embedded agent loop:

- `src/agents/pi-embedded-runner/run.ts`

That file depends on:

- context engine initialization
- plugin hook runner
- provider runtime auth
- auth-profile selection and cooldown
- model resolution
- failover policy
- usage accounting
- lane-based command queueing
- prompt payload building
- compaction and transcript maintenance

This is not a thin wrapper around an LLM SDK.
It is a policy-heavy execution runtime.

### 9.2 What the Agent Layer Owns

The `src/agents/` tree owns:

- model auth and profile rotation
- tool surfaces
- sandbox and process tools
- session and workspace semantics
- skill loading
- embedded Pi integration
- failover and retry policy
- usage tracking
- transcript and compaction policy

Sub-area hotspots:

- `src/agents/pi-embedded-runner`
- `src/agents/tools`
- `src/agents/sandbox`
- `src/agents/auth-profiles`

Framework extraction note:

- the agent layer mixes generic agent-runtime concerns with OpenClaw-specific session and messaging semantics
- separating those would be a prime redesign target

## 10. Memory and Context Architecture

### 10.1 Memory Index Manager

Main memory orchestrator:

- `src/memory/manager.ts`

The memory layer owns:

- embedding provider creation
- vector and FTS search
- hybrid rank merge
- watch and sync behavior
- per-agent memory index manager caching
- read-only recovery
- session file ingestion

Memory is not a trivial feature.
It is a long-lived indexed retrieval subsystem.

### 10.2 Context Engine

Context engine contract lives in:

- `src/context-engine/types.ts`
- `src/context-engine/registry.ts`

The context engine is treated as an exclusive slot.
Plugins can register a context engine via `registerContextEngine(...)`.

Framework extraction note:

- this is a good example of an exclusive capability, not a many-provider capability
- the platform already distinguishes shared contracts from exclusive slots

## 11. ACP and Control Plane Extensions

ACP lives under:

- `src/acp/`

It includes:

- runtime registry
- control-plane manager
- session mapping
- translation and policy
- persistent bindings

This is another sign that OpenClaw is more of an agent platform than a simple chat gateway.

## 12. UI and Native Shells

### 12.1 Browser Control UI

Browser UI lives in:

- `ui/src/ui/controllers`
- `ui/src/ui/views`
- `ui/src/ui/components`

The browser UI is a separate Vite project with its own Vitest configuration.
This is an outer shell, not the source of business logic.

### 12.2 Native Apps

Native shells:

- `apps/macos`
- `apps/ios`
- `apps/android`

Observed structure:

- macOS contains app, IPC, discovery, CLI bridge, and protocol modules
- iOS contains gateway, onboarding, chat, media, status, voice, settings, and device areas
- Android follows the expected app shell layout

Architecturally, these appear to be shells over the Gateway and control plane rather than separate backends.

## 13. Testing and Architectural Guardrails

Key files:

- `vitest.config.ts`
- `vitest.unit.config.ts`
- `ui/vitest.config.ts`
- `test/architecture-smells.test.ts`
- `test/plugin-extension-import-boundary.test.ts`
- `test/extension-plugin-sdk-boundary.test.ts`
- `test/official-channel-catalog.test.ts`

Important observation from `vitest.config.ts`:

- many integration-heavy areas are intentionally excluded from strict core coverage accounting
- large surfaces such as `src/gateway/**`, `src/agents/**`, `src/channels/**`, `src/plugins/**`, `src/cli/**`, and `src/commands/**` are validated more by targeted tests, contract tests, and integration coverage than by raw percentage

Important observation from guardrail tests:

- the repo explicitly tests architectural smells and import boundaries
- this means architecture is not only documented, it is partially enforced

Framework extraction note:

- keep these guardrail tests
- they are valuable design constraints, not test noise

## 14. Extension Landscape

Observed capability counts from manifests:

- channel plugins: about 21
- provider plugins: about 36
- memory plugins: 2

There are also hybrid and feature plugins covering:

- web search
- diagnostics
- device pairing
- thread ownership
- voice/call behavior
- tool bundles

Examples:

- `extensions/openai`
  provider ownership, auth choices
- `extensions/google`
  multi-capability vendor plugin
- `extensions/telegram`
  channel plugin
- `extensions/memory-core`
  memory slot
- `extensions/duckduckgo`
  web-search plugin
- `extensions/voice-call`
  feature plugin that also registers gateway methods

This is a healthy clue for framework design:

- vendor plugins
- channel plugins
- feature plugins
- exclusive-slot plugins

These should probably be explicit framework concepts, not just emergent conventions.

## 15. Likely Kernel vs Product Split

If extracting a framework, a useful split would be:

### Framework Kernel Candidates

- control-plane request bus
- method registry
- plugin discovery and registration
- capability contracts
- runtime container and scoped context
- long-lived execution runtime
- state and lifecycle services
- event bus

### Product-Specific Layers

- channel semantics
- OpenClaw session-key conventions
- OpenClaw command UX
- vendor/provider implementations
- app-specific UI and client shells
- docs and release process

## 16. Architectural Strengths

Strong patterns already present in the repo:

- manifest-first discovery before runtime import
- explicit plugin ownership boundaries
- typed capability registration
- request handler modularity
- late-bound runtime helpers
- clear distinction between shared contracts and vendor-specific behavior
- strong test guardrails for import boundaries

These are good ingredients for a reusable framework.

## 17. Architectural Tensions

Likely complexity hotspots:

- global singleton runtime state in several places
- Gateway startup owns too many responsibilities
- agent runtime mixes generic execution with product-specific session behavior
- channel contract is powerful but broad
- command layer, gateway layer, and plugin layer all expose adjacent control surfaces
- first-party bundled plugins blur the line between core and external extension

These are not bugs.
They are the normal result of product growth.
But they are the seams you would revisit when extracting a framework.

## 18. Recommended Reading Order for Framework Design

Read these in order.
Each file gives a specific architectural answer.

1. `ARCHITECTURE_REVIEW_ABSTRACT.md`
   What the repo is.
2. `src/plugins/types.ts`
   What the platform lets extensions do.
3. `src/plugins/registry.ts`
   How extensions become runtime-owned capabilities.
4. `src/plugins/loader.ts`
   How static manifest, discovery, cache, and activation fit together.
5. `src/plugins/runtime/index.ts`
   What trusted runtime helpers are exposed.
6. `src/channels/plugins/types.plugin.ts`
   What a channel owns.
7. `src/gateway/server-methods.ts`
   What the control plane exposes.
8. `src/gateway/server.impl.ts`
   What the runtime kernel starts and owns.
9. `src/agents/pi-embedded-runner/run.ts`
   How the execution runtime behaves under real policy pressure.
10. `src/memory/manager.ts`
    How long-lived indexed state is managed.
11. `src/context-engine/registry.ts`
    How exclusive slots differ from shared capability registration.
12. `src/cli/run-main.ts`
    How shells enter the system.
13. `src/commands/*`
    How product UX is layered over the runtime.

## 19. Suggested Framework Design Extraction

A plausible extracted architecture would look like:

1. `kernel`
   runtime container, scoped context, lifecycle, event bus
2. `control-plane`
   transport adapters plus method registry
3. `capabilities`
   shared contracts for provider, channel, memory, search, media, execution
4. `extensions`
   manifest, discovery, loader, registry, activation
5. `execution`
   agent runtime, retries, failover, tool system, compaction
6. `state`
   config, session storage, approvals, persistence
7. `surfaces`
   CLI, web UI, mobile, desktop
8. `products`
   channel and vendor implementations

OpenClaw already contains most of these pieces.
The current challenge is not invention.
The challenge is separation and naming.

## 20. Final Takeaway

OpenClaw is structurally closest to a plugin-centered agent platform with a Gateway kernel.

If you review it as:

- "a CLI app"
- or "a chat bot"
- or "a thin wrapper around model providers"

you will underestimate the real architecture.

If you review it as:

- a control-plane kernel
- plus a capability registry
- plus an embedded execution runtime
- plus many product adapters

the repo starts to make sense very quickly.
