# OpenClaw Architecture Review: Abstract Map

This file is the shortest high-signal map of the repository.
Read this first.
Read `ARCHITECTURE_REVIEW_DETAILED.md` second.

## What OpenClaw Actually Is

OpenClaw is not one app.
It is a monorepo that combines:

- a local-first CLI
- a long-lived Gateway server and control plane
- an embedded agent runtime
- a plugin and channel platform
- a browser Control UI
- native macOS, iOS, and Android shells

The practical center of gravity is the Gateway plus the plugin runtime, not the CLI.

## Top-Level Mental Model

Think of the system as six layers:

1. Shells
   CLI, web UI, macOS app, iOS app, Android app
2. Control plane
   Gateway WebSocket and HTTP server
3. Domain orchestration
   commands, sessions, routing, outbound messaging, agent execution
4. Capability platform
   plugins, channels, providers, tools, hooks, services
5. Specialized subsystems
   embedded agent runtime, memory, context engine, ACP, media, browser
6. Persistence and policy
   config, sessions, credentials, state dirs, approvals, security rules

## Physical Layout

- `openclaw.mjs`
  Thin launcher that verifies Node and loads `dist/entry.js`.
- `src/entry.ts`
  Actual CLI bootstrap.
- `src/cli/`
  Commander program wiring, argv routing, lazy command registration.
- `src/commands/`
  User-facing command behavior.
- `src/gateway/`
  Long-lived control plane server and request handlers.
- `src/plugins/`
  Plugin discovery, loading, registration, runtime ownership.
- `src/channels/`
  Shared channel abstractions and channel-plugin contracts.
- `src/agents/`
  Embedded agent runtime, tools, auth-profile rotation, failover, sandbox, skills.
- `src/memory/`
  Search, indexing, embedding providers, hybrid retrieval.
- `src/context-engine/`
  Exclusive-slot context engine contract.
- `extensions/`
  Workspace plugins. This is part of the product, not an afterthought.
- `ui/`
  Browser Control UI.
- `apps/`
  Native shells.

## Three Runtime Flows

### 1. CLI Flow

`openclaw.mjs`
-> `src/entry.ts`
-> `src/cli/run-main.ts`
-> `src/cli/program/*`
-> `src/commands/*`

The CLI is mostly a router into domain code.

### 2. Gateway Flow

`src/gateway/server.ts`
-> `src/gateway/server.impl.ts`
-> `src/gateway/server-methods.ts`
-> `src/gateway/server-methods/*`
-> plugins, channels, agent runtime, memory, sessions

The Gateway is the main runtime kernel.

### 3. Plugin Flow

`extensions/*/openclaw.plugin.json`
-> `src/plugins/discovery.ts`
-> `src/plugins/loader.ts`
-> `src/plugins/registry.ts`
-> active runtime registry
-> consumed by gateway, CLI, channels, providers, tools

The plugin system is the real extensibility layer.

## Core Ownership Model

The codebase repeatedly uses the same boundary:

- core owns contracts, orchestration, policy, fallback, and lifecycle
- plugins own vendor-specific or channel-specific behavior
- the Gateway owns long-lived runtime authority
- shells are thin clients over the Gateway

This is the single most important design choice in the repo.

## Primary Contracts

If you only remember a few types and entry points, remember these:

- `src/cli/run-main.ts:82`
  CLI entry.
- `src/gateway/server.impl.ts:362`
  Gateway startup.
- `src/gateway/server-methods.ts:68`
  Gateway method registry.
- `src/plugins/types.ts:1314`
  `OpenClawPluginApi` registration surface.
- `src/plugins/registry.ts:247`
  Plugin registry construction.
- `src/plugins/runtime/index.ts:184`
  `createPluginRuntime()`.
- `src/channels/plugins/types.plugin.ts:55`
  `ChannelPlugin` contract.
- `src/agents/pi-embedded-runner/run.ts`
  Embedded agent execution loop.
- `src/memory/manager.ts`
  Memory index manager.

## Review Order

If your goal is framework extraction, review in this order:

1. `ARCHITECTURE_REVIEW_DETAILED.md`
2. `src/plugins/types.ts`
3. `src/plugins/registry.ts`
4. `src/plugins/loader.ts`
5. `src/channels/plugins/types.plugin.ts`
6. `src/gateway/server-methods.ts`
7. `src/gateway/server.impl.ts`
8. `src/agents/pi-embedded-runner/run.ts`
9. `src/memory/manager.ts`
10. `src/commands/` and `src/cli/`

## Framework Extraction View

If you were designing a new framework out of this repo, the likely reusable kernel would be:

- a control-plane server
- a capability registry
- a plugin runtime with typed contracts
- a long-lived execution runtime
- shell adapters
- policy and state services

The likely OpenClaw-specific adapters would be:

- messaging channels
- OpenClaw session semantics
- OpenClaw command UX
- vendor/provider integrations
- product-specific UI and native clients

## One-Sentence Summary

OpenClaw is best understood as a plugin-driven agent operating system with a Gateway-centered control plane and multiple thin shells around it.
