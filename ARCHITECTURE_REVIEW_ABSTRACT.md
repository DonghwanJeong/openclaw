# OpenClaw 아키텍처 리뷰: 추상 맵

이 파일은 저장소를 가장 짧고 밀도 높게 파악하기 위한 상위 구조 지도입니다.
먼저 이 파일을 읽으세요.
그다음 `ARCHITECTURE_REVIEW_DETAILED.md` 를 읽으세요.

## OpenClaw은 실제로 무엇인가

OpenClaw은 하나의 앱이 아닙니다.
다음 요소를 함께 담고 있는 모노레포입니다.

- 로컬 우선 CLI
- 장시간 실행되는 Gateway 서버와 control plane
- 임베디드 agent runtime
- plugin 및 channel 플랫폼
- 브라우저 Control UI
- macOS, iOS, Android 네이티브 셸

실질적인 중심축은 CLI가 아니라 Gateway와 plugin runtime입니다.

## 최상위 정신 모델

이 시스템은 6개 계층으로 보면 이해가 빠릅니다.

1. 셸
   CLI, web UI, macOS 앱, iOS 앱, Android 앱
2. control plane
   Gateway WebSocket 및 HTTP 서버
3. 도메인 오케스트레이션
   commands, sessions, routing, outbound messaging, agent execution
4. capability 플랫폼
   plugins, channels, providers, tools, hooks, services
5. 특화 서브시스템
   embedded agent runtime, memory, context engine, ACP, media, browser
6. 영속성 및 정책
   config, sessions, credentials, state dirs, approvals, security rules

## 물리적 레이아웃

- `openclaw.mjs`
  Node 버전을 확인하고 `dist/entry.js` 를 로드하는 얇은 런처입니다.
- `src/entry.ts`
  실제 CLI 부트스트랩입니다.
- `src/cli/`
  Commander 프로그램 wiring, argv routing, lazy command registration을 담당합니다.
- `src/commands/`
  사용자에게 보이는 command 동작이 들어 있습니다.
- `src/gateway/`
  장시간 살아 있는 control plane 서버와 request handler가 들어 있습니다.
- `src/plugins/`
  plugin discovery, loading, registration, runtime ownership을 담당합니다.
- `src/channels/`
  공통 channel 추상화와 channel-plugin contract가 들어 있습니다.
- `src/agents/`
  embedded agent runtime, tools, auth-profile rotation, failover, sandbox, skills를 담당합니다.
- `src/memory/`
  search, indexing, embedding providers, hybrid retrieval을 담당합니다.
- `src/context-engine/`
  exclusive-slot context engine contract가 들어 있습니다.
- `extensions/`
  workspace plugins입니다. 부가 요소가 아니라 제품의 핵심 일부입니다.
- `ui/`
  브라우저 Control UI입니다.
- `apps/`
  네이티브 셸입니다.

## 세 가지 런타임 흐름

### 1. CLI 흐름

`openclaw.mjs`
-> `src/entry.ts`
-> `src/cli/run-main.ts`
-> `src/cli/program/*`
-> `src/commands/*`

CLI는 대체로 도메인 코드로 진입시키는 라우터 역할을 합니다.

### 2. Gateway 흐름

`src/gateway/server.ts`
-> `src/gateway/server.impl.ts`
-> `src/gateway/server-methods.ts`
-> `src/gateway/server-methods/*`
-> plugins, channels, agent runtime, memory, sessions

Gateway는 메인 런타임 커널입니다.

### 3. Plugin 흐름

`extensions/*/openclaw.plugin.json`
-> `src/plugins/discovery.ts`
-> `src/plugins/loader.ts`
-> `src/plugins/registry.ts`
-> active runtime registry
-> gateway, CLI, channels, providers, tools에서 소비

plugin 시스템이 실제 확장성 계층입니다.

## 코어 ownership 모델

이 코드베이스는 반복적으로 같은 경계를 사용합니다.

- core는 contract, orchestration, policy, fallback, lifecycle을 소유합니다
- plugins는 vendor별 또는 channel별 동작을 소유합니다
- Gateway는 장시간 실행되는 런타임 권한을 소유합니다
- 셸은 Gateway 위의 얇은 클라이언트입니다

이것이 이 저장소에서 가장 중요한 설계 선택입니다.

## 가장 중요한 contract

몇 개의 타입과 엔트리 포인트만 기억한다면 아래를 기억하면 됩니다.

- `src/cli/run-main.ts:82`
  CLI 엔트리
- `src/gateway/server.impl.ts:362`
  Gateway startup
- `src/gateway/server-methods.ts:68`
  Gateway method registry
- `src/plugins/types.ts:1314`
  `OpenClawPluginApi` registration surface
- `src/plugins/registry.ts:247`
  plugin registry construction
- `src/plugins/runtime/index.ts:184`
  `createPluginRuntime()`
- `src/channels/plugins/types.plugin.ts:55`
  `ChannelPlugin` contract
- `src/agents/pi-embedded-runner/run.ts`
  embedded agent execution loop
- `src/memory/manager.ts`
  memory index manager

## 리뷰 순서

목표가 framework extraction이라면 다음 순서로 보는 것이 좋습니다.

1. `ARCHITECTURE_REVIEW_DETAILED.md`
2. `src/plugins/types.ts`
3. `src/plugins/registry.ts`
4. `src/plugins/loader.ts`
5. `src/channels/plugins/types.plugin.ts`
6. `src/gateway/server-methods.ts`
7. `src/gateway/server.impl.ts`
8. `src/agents/pi-embedded-runner/run.ts`
9. `src/memory/manager.ts`
10. `src/commands/` 와 `src/cli/`

## Framework extraction 관점

이 저장소에서 새로운 framework를 뽑아낸다면 재사용 가능한 커널 후보는 대략 다음과 같습니다.

- control-plane 서버
- capability registry
- typed contract를 가진 plugin runtime
- 장시간 살아 있는 execution runtime
- shell adapters
- policy 및 state services

반대로 OpenClaw 특화 adapter일 가능성이 높은 부분은 다음과 같습니다.

- messaging channels
- OpenClaw session semantics
- OpenClaw command UX
- vendor/provider integrations
- 제품 전용 UI와 네이티브 클라이언트

## 한 문장 요약

OpenClaw은 Gateway 중심 control plane 위에 여러 얇은 셸이 얹힌, plugin-driven agent operating system에 가깝게 이해하는 편이 가장 정확합니다.
