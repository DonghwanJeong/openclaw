# OpenClaw 아키텍처 리뷰: 상세 맵

이 파일은 `ARCHITECTURE_REVIEW_ABSTRACT.md` 의 기술적 심화판입니다.
framework extraction, subsystem review, 아키텍처 재설계를 염두에 두고 작성했습니다.

## 1. 규모 스냅샷

현재 저장소의 형태를 코드베이스 자체에서 읽어보면 대략 다음과 같습니다.

- `src/` 파일 수: 약 5,103개
- `src`, `test`, `extensions`, `ui` 전체 테스트 파일 수: 약 2,990개
- extension 디렉터리 수: 82개
- `extensions/*/openclaw.plugin.json` 기준 plugin manifest 수: 80개
- `ui/src` 브라우저 UI 소스 파일 수: 약 229개

`src/` 상위 영역별 파일 수는 다음과 같습니다.

| 영역             | 대략 파일 수 | 의미                                                |
| ---------------- | -----------: | --------------------------------------------------- |
| `src/agents`     |          974 | embedded agent runtime이 최상위 서브시스템이라는 뜻 |
| `src/infra`      |          534 | 공유 저수준 인프라 비중이 큼                        |
| `src/commands`   |          426 | CLI 동작 범위가 넓고 제품 표면적이 큼               |
| `src/gateway`    |          393 | control plane이 크고 기능이 많음                    |
| `src/auto-reply` |          348 | reply 생성과 orchestration 비중이 큼                |
| `src/cli`        |          326 | CLI wiring이 단순하지 않음                          |
| `src/config`     |          264 | config/state 관리가 핵심 관심사임                   |
| `src/plugins`    |          235 | plugin ownership과 loading이 중심축임               |
| `src/plugin-sdk` |          216 | public integration surface가 큼                     |
| `src/channels`   |          191 | channel abstraction이 독립적인 플랫폼 계층임        |

중요한 하위 영역 밀도는 다음과 같습니다.

- `src/commands/doctor`: 42개 파일
- `src/commands/models`: 31개 파일
- `src/gateway/server-methods`: 69개 파일
- `src/gateway/server`: 29개 파일
- `src/gateway/protocol`: 29개 파일
- `src/agents/pi-embedded-runner`: 115개 파일
- `src/agents/tools`: 83개 파일
- `src/agents/sandbox`: 62개 파일

해석:

- 이 저장소는 plugin이 달린 작은 CLI 앱이 아닙니다
- 여러 개의 반독립적 서브시스템을 품은 플랫폼 저장소입니다
- 여기서 새 framework를 추출하려면 초기에 kernel concern과 product concern을 분리해야 합니다

## 2. 워크스페이스 구조

`pnpm-workspace.yaml` 기준 워크스페이스 루트는 다음과 같습니다.

- root package
- `ui`
- `packages/*`
- `extensions/*`

상위 역할 맵은 다음과 같습니다.

| 경로           | 역할                                            |
| -------------- | ----------------------------------------------- |
| `openclaw.mjs` | 런처 엔트리포인트                               |
| `src/`         | 코어 런타임과 제품 로직                         |
| `extensions/`  | workspace plugins 및 공식 channel/provider 구현 |
| `ui/`          | 브라우저 Control UI                             |
| `apps/`        | macOS, iOS, Android 셸                          |
| `packages/`    | 작은 호환성 패키지                              |
| `test/`        | 횡단 관심사 테스트와 guardrail                  |
| `docs/`        | 공개 아키텍처 및 사용자 문서                    |

물리적으로는 모듈화돼 있지만, 제품 런타임의 중심은 `src/` 입니다.

## 3. 시스템의 중심: Gateway + Plugin Runtime

이 저장소의 구조적 중심은 CLI가 아닙니다.
실제 중심은 다음 세 가지입니다.

- Gateway 서버
- plugin registry와 runtime
- embedded agent runtime

CLI, web UI, 네이티브 앱은 이 중심 위에 올라간 셸입니다.

이 해석은 코드와 공개 문서가 모두 같은 방향을 가리킵니다.

- `docs/concepts/architecture.md`
- `docs/plugins/architecture.md`

## 4. 부트와 엔트리 포인트

### 4.1 런처

`openclaw.mjs` 는 네 가지를 합니다.

- 최소 Node 버전 강제
- 가능할 때 compile cache 활성화
- warning filter 로드
- 빌드된 `dist/entry.js` import

즉, 이 파일은 배포용 wrapper이지 실제 아키텍처 자체는 아닙니다.

### 4.2 실제 CLI 부트스트랩

메인 경로는 다음과 같습니다.

`openclaw.mjs`
-> `src/entry.ts`
-> `src/cli/run-main.ts`

중요 파일:

- `src/entry.ts`
  runtime bootstrap, respawn logic, profile env 처리, 빠른 help/version 경로를 담당
- `src/cli/run-main.ts:82`
  실제 CLI 엔트리 함수 `runCli()`
- `src/cli/program/build-program.ts:8`
  Commander 프로그램 구성
- `src/cli/program/command-registry.ts:302`
  core command를 lazy 등록

핵심 관찰:

- CLI는 의도적으로 lazy-loaded 구조입니다
- command tree가 business logic 자체는 아닙니다
- command tree는 `src/commands/` 안의 도메인 모듈 위에 얹힌 셸입니다

### 4.3 라이브러리 export 경로

`src/index.ts` 는 compatibility entry이자 library shell입니다.
`src/library.ts` 는 config loading, session store access, process helper, default dependency creation 같은 선택된 기능만 재노출합니다.

이 점이 시사하는 바는 다음과 같습니다.

- OpenClaw은 일부 맥락에서 library surface로도 사용됩니다
- 하지만 library surface는 런타임 내부 구조보다 의도적으로 훨씬 작습니다

## 5. CLI 아키텍처

CLI는 크게 세 계층으로 나뉩니다.

1. bootstrap 및 argv normalization
2. command-program registration
3. command behavior modules

주요 파일:

- `src/cli/run-main.ts`
- `src/cli/program/*`
- `src/commands/*`

핵심 특성:

- startup 비용을 줄이기 위한 lazy command registration
- 루트 차원의 `--help`, `--version` fast path
- `src/plugins/cli.ts` 를 통한 runtime plugin CLI command 추가
- command 실행이 여전히 config, plugin registry, gateway, sessions, channels에 의존

즉, command는 고립된 micro-tool이 아닙니다.
같은 공유 런타임 그래프의 앞단 인터페이스입니다.

## 6. Gateway 아키텍처

### 6.1 공개 Gateway 엔트리

Gateway 엔트리는 다음과 같습니다.

- `src/gateway/server.ts`
  `startGatewayServer` 를 export
- `src/gateway/server.impl.ts:362`
  실제 startup 로직이 들어 있는 파일

### 6.2 Gateway startup이 실제로 하는 일

`src/gateway/server.impl.ts` 를 보면 startup 시퀀스는 대략 아래와 같습니다.

1. config snapshot 읽기
2. 필요 시 legacy config 마이그레이션
3. startup config 검증
4. config 및 environment 기준 plugin auto-enable
5. secrets runtime 활성화
6. plugin 로드 및 active registry 구성
7. plugin handler를 포함한 request handler 구성
8. discovery sidecar 시작
9. maintenance timer 시작
10. 필요 시 Tailscale 노출
11. browser 및 plugin sidecar 시작
12. config reloader 시작

관련 anchor:

- `src/gateway/server.impl.ts:380`
  초기 config 읽기
- `src/gateway/server.impl.ts:387`
  legacy migration
- `src/gateway/server.impl.ts:409`
  plugin auto-enable
- `src/gateway/server.impl.ts:760`
  discovery startup
- `src/gateway/server.impl.ts:806`
  maintenance timer
- `src/gateway/server.impl.ts:1037`
  exec approval handler
- `src/gateway/server.impl.ts:1040`
  secrets handler
- `src/gateway/server.impl.ts:1179`
  Tailscale 노출
- `src/gateway/server.impl.ts:1199`
  sidecar
- `src/gateway/server.impl.ts:1270`
  config reloader

해석:

- Gateway는 얇은 transport 계층이 아닙니다
- runtime kernel이자 lifecycle manager입니다
- initialization, policy, reload, background timer, sidecar를 모두 소유합니다

### 6.3 Gateway request 처리

method 집계는 다음 파일에서 이뤄집니다.

- `src/gateway/server-methods.ts:68`
  `coreGatewayHandlers`

이 파일은 다음 handler set을 합칩니다.

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

이 구조는 framework 설계 관점에서 매우 중요합니다.

- transport는 통합돼 있습니다
- business API는 method 기반입니다
- handler는 모듈식입니다
- request scoping과 auth는 중앙 집중적으로 처리됩니다

framework 관점으로 번역하면 다음과 같습니다.

- 하나의 control-plane protocol
- 하나의 method registry
- registry에 꽂히는 많은 feature module

## 7. Plugin 아키텍처

### 7.1 정적 manifest 계층

모든 native plugin은 `openclaw.plugin.json` 을 통해 정적 메타데이터를 노출할 수 있습니다.

예시 역할:

- channels: `extensions/telegram/openclaw.plugin.json`
- providers: `extensions/openai/openclaw.plugin.json`
- memory: `extensions/memory-core/openclaw.plugin.json`

정적 manifest의 책임:

- plugin identity
- capability 선언
- config schema
- auth choice 메타데이터
- UI hint

이 정적 계층 덕분에 plugin 코드를 실행하기 전에 discovery와 validation을 수행할 수 있습니다.

### 7.2 런타임 registration 계층

런타임 registration API는 다음에 정의돼 있습니다.

- `src/plugins/types.ts:1314`
  `OpenClawPluginApi`

주요 registration 메서드:

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

이것이 실제 확장성 contract입니다.

### 7.3 Loader 파이프라인

메인 loader:

- `src/plugins/loader.ts`

핵심 책임:

- plugin config normalize
- root 및 cache key 해석
- plugin 후보 discovery
- manifest registry 로드
- registry 구성
- plugin module import
- `register(api)` 또는 `activate` 호출
- 결과 registry를 글로벌하게 활성화

중요 anchor:

- `src/plugins/loader.ts:689`
  cache key 생성 및 activation mode 로직
- `src/plugins/loader.ts:821`
  `createPluginRegistry(...)`
- `src/plugins/loader.ts:828`
  discovery
- `src/plugins/loader.ts:834`
  manifest registry load
- `src/plugins/loader.ts:1276`
  active registry 활성화

### 7.4 Registry 구성

registry 구성은 다음 파일에 있습니다.

- `src/plugins/registry.ts:247`
  `createPluginRegistry(...)`

registry가 기록하는 항목:

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

이 객체가 이 저장소에서 framework kernel 객체에 가장 가까운 형태입니다.

### 7.5 Runtime surface

plugin runtime은 다음에서 생성됩니다.

- `src/plugins/runtime/index.ts:184`
  `createPluginRuntime()`

이 runtime이 노출하는 helper는 다음과 같습니다.

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
- lazy binding되는 TTS, STT, media-understanding, model-auth

중요한 설계 선택:

- plugin registration과 plugin runtime은 서로 다른 관심사입니다
- 정적 manifest discovery는 runtime import보다 먼저 일어납니다
- runtime 메서드는 late-bound이며 일부는 lazy입니다

### 7.6 Active runtime state

active plugin registry의 글로벌 상태는 다음 파일에 있습니다.

- `src/plugins/runtime.ts`

이 파일은 다음을 관리합니다.

- active plugin registry
- pinned HTTP route registry
- registry key와 version

framework extraction 관점 메모:

- 글로벌 singleton registry는 편리합니다
- 하지만 새 framework에서는 container나 runtime context 뒤로 추상화하고 싶어질 가능성이 큰 핵심 결합 지점이기도 합니다

## 8. Channel 아키텍처

### 8.1 Channel contract

channel plugin 타입:

- `src/channels/plugins/types.plugin.ts:55`
  `ChannelPlugin`

하나의 channel이 소유할 수 있는 것:

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

이 contract는 매우 넓고 풍부합니다.

### 8.2 Bundled channels

공식 bundled channel은 다음 파일에서 정적으로 import됩니다.

- `src/channels/plugins/bundled.ts`

이 파일이 노출하는 것:

- `bundledChannelPlugins`
- `bundledChannelSetupPlugins`
- `getBundledChannelPlugin()`
- 일부 channel에 대한 runtime setter

manifest 기준 관찰:

- 약 21개의 plugin이 channel capability를 선언하고 있습니다

중요한 아키텍처적 함의:

- 공식 channel은 workspace plugin으로 구현됩니다
- 하지만 core는 여전히 공식 bundled channel entrypoint를 직접 알고 있습니다
- 즉, 순수 plugin architecture와 curated first-party platform 사이의 하이브리드 구조입니다

### 8.3 공유 message tool 경계

공개 plugin architecture 문서와 현재 코드 배치를 보면:

- core가 공유 message-tool orchestration을 소유합니다
- channel plugin은 channel별 discovery와 execution을 소유합니다

이건 framework 관점에서 꽤 강한 패턴입니다.

- 중앙 action host
- capability별 adapter
- domain plugin은 바깥 orchestration 전체를 소유하지 않고도 자신의 동작을 기여

## 9. Agent Runtime 아키텍처

이 영역은 저장소에서 가장 큰 서브시스템입니다.

### 9.1 Agent execution의 중심

메인 embedded agent loop:

- `src/agents/pi-embedded-runner/run.ts`

이 파일이 의존하는 것:

- context engine 초기화
- plugin hook runner
- provider runtime auth
- auth-profile 선택 및 cooldown
- model resolution
- failover policy
- usage accounting
- lane 기반 command queueing
- prompt payload 빌드
- compaction 및 transcript maintenance

즉, 이건 단순한 LLM SDK wrapper가 아닙니다.
정책이 매우 많이 들어간 execution runtime입니다.

### 9.2 Agent 계층이 소유하는 것

`src/agents/` 트리가 소유하는 관심사는 다음과 같습니다.

- model auth와 profile rotation
- tool surface
- sandbox와 process tool
- session 및 workspace semantics
- skill loading
- embedded Pi integration
- failover와 retry policy
- usage tracking
- transcript와 compaction policy

핫스팟 하위 영역:

- `src/agents/pi-embedded-runner`
- `src/agents/tools`
- `src/agents/sandbox`
- `src/agents/auth-profiles`

framework extraction 관점 메모:

- agent 계층은 범용 agent-runtime concern과 OpenClaw 전용 session/messaging semantics를 섞고 있습니다
- 새 설계에서 이 둘을 분리하는 것이 가장 유력한 재설계 대상입니다

## 10. Memory 및 Context 아키텍처

### 10.1 Memory index manager

메인 memory orchestrator:

- `src/memory/manager.ts`

memory 계층이 소유하는 것:

- embedding provider 생성
- vector 및 FTS search
- hybrid rank merge
- watch 및 sync 동작
- agent별 memory index manager cache
- read-only recovery
- session file ingestion

memory는 사소한 기능이 아닙니다.
장시간 살아 있는 indexed retrieval 서브시스템입니다.

### 10.2 Context engine

context engine contract는 다음에 있습니다.

- `src/context-engine/types.ts`
- `src/context-engine/registry.ts`

context engine은 exclusive slot으로 취급됩니다.
plugin은 `registerContextEngine(...)` 을 통해 context engine을 등록할 수 있습니다.

framework extraction 관점 메모:

- 이것은 many-provider capability가 아니라 exclusive capability의 좋은 예시입니다
- 이 플랫폼은 이미 공유 contract와 exclusive slot을 구분하고 있습니다

## 11. ACP와 Control Plane 확장

ACP는 다음 경로에 있습니다.

- `src/acp/`

포함되는 내용:

- runtime registry
- control-plane manager
- session mapping
- translation 및 policy
- persistent binding

이 역시 OpenClaw이 단순한 chat gateway보다 agent platform에 더 가깝다는 강한 신호입니다.

## 12. UI와 네이티브 셸

### 12.1 브라우저 Control UI

브라우저 UI는 다음 경로에 있습니다.

- `ui/src/ui/controllers`
- `ui/src/ui/views`
- `ui/src/ui/components`

브라우저 UI는 자체 Vite 프로젝트와 별도 Vitest 설정을 가진 독립 셸입니다.
비즈니스 로직의 근원이라기보다 외곽 surface에 가깝습니다.

### 12.2 네이티브 앱

네이티브 셸:

- `apps/macos`
- `apps/ios`
- `apps/android`

관찰되는 구조:

- macOS는 app, IPC, discovery, CLI bridge, protocol 모듈을 포함
- iOS는 gateway, onboarding, chat, media, status, voice, settings, device 영역을 포함
- Android는 예상 가능한 app shell 레이아웃을 따름

아키텍처 관점에서 보면 이들은 별도 백엔드라기보다 Gateway 및 control plane 위의 셸로 보입니다.

## 13. 테스트와 아키텍처 Guardrail

핵심 파일:

- `vitest.config.ts`
- `vitest.unit.config.ts`
- `ui/vitest.config.ts`
- `test/architecture-smells.test.ts`
- `test/plugin-extension-import-boundary.test.ts`
- `test/extension-plugin-sdk-boundary.test.ts`
- `test/official-channel-catalog.test.ts`

`vitest.config.ts` 에서 중요한 관찰:

- integration 비중이 높은 영역이 엄격한 core coverage 집계에서 의도적으로 제외돼 있습니다
- `src/gateway/**`, `src/agents/**`, `src/channels/**`, `src/plugins/**`, `src/cli/**`, `src/commands/**` 같은 큰 surface는 raw percentage보다 targeted test, contract test, integration coverage로 더 많이 검증됩니다

guardrail 테스트에서 중요한 관찰:

- 이 저장소는 architectural smell과 import boundary를 명시적으로 테스트합니다
- 즉, 아키텍처가 문서화만 돼 있는 것이 아니라 일부는 강제되고 있습니다

framework extraction 관점 메모:

- 이런 guardrail 테스트는 유지하는 편이 좋습니다
- 테스트 잡음이 아니라 설계 제약으로서 가치가 큽니다

## 14. Extension 지형도

manifest 기준 capability 수:

- channel plugins: 약 21개
- provider plugins: 약 36개
- memory plugins: 2개

그 외에도 다음을 다루는 hybrid 및 feature plugin이 있습니다.

- web search
- diagnostics
- device pairing
- thread ownership
- voice/call behavior
- tool bundle

예시:

- `extensions/openai`
  provider ownership, auth choice
- `extensions/google`
  multi-capability vendor plugin
- `extensions/telegram`
  channel plugin
- `extensions/memory-core`
  memory slot
- `extensions/duckduckgo`
  web-search plugin
- `extensions/voice-call`
  gateway method까지 등록하는 feature plugin

framework 설계 관점에서 이건 꽤 좋은 신호입니다.

- vendor plugin
- channel plugin
- feature plugin
- exclusive-slot plugin

이 구분은 단순한 관습이 아니라 framework 개념으로 명시하는 편이 좋습니다.

## 15. Kernel과 Product의 분리 후보

framework를 추출한다면 다음과 같은 분리가 유용할 가능성이 높습니다.

### Framework kernel 후보

- control-plane request bus
- method registry
- plugin discovery 및 registration
- capability contract
- runtime container 및 scoped context
- 장시간 살아 있는 execution runtime
- state 및 lifecycle service
- event bus

### Product 전용 계층

- channel semantics
- OpenClaw session-key convention
- OpenClaw command UX
- vendor/provider implementation
- 앱 전용 UI 및 client shell
- 문서와 release process

## 16. 아키텍처 강점

이미 저장소 안에 존재하는 강한 패턴:

- runtime import 이전의 manifest-first discovery
- 명시적인 plugin ownership 경계
- typed capability registration
- request handler의 모듈성
- late-bound runtime helper
- 공유 contract와 vendor-specific behavior의 명확한 구분
- import boundary를 지키기 위한 강한 테스트 guardrail

이것들은 재사용 가능한 framework를 만드는 데 좋은 재료입니다.

## 17. 아키텍처 긴장 지점

복잡도가 몰려 있을 가능성이 높은 지점:

- 여러 곳의 global singleton runtime state
- 너무 많은 책임을 가진 Gateway startup
- 범용 execution과 제품 전용 session behavior가 섞인 agent runtime
- 강력하지만 범위가 넓은 channel contract
- command 계층, gateway 계층, plugin 계층이 서로 인접한 control surface를 동시에 노출
- core와 외부 extension의 경계를 흐리게 하는 first-party bundled plugin

이것들은 버그가 아닙니다.
제품이 성장하면서 자연스럽게 생기는 결과입니다.
다만 framework를 추출한다면 가장 먼저 다시 볼 seam이기도 합니다.

## 18. Framework 설계를 위한 권장 읽기 순서

아래 순서대로 읽는 것을 권장합니다.
각 파일은 하나의 구체적인 아키텍처 질문에 답합니다.

1. `ARCHITECTURE_REVIEW_ABSTRACT.md`
   저장소가 무엇인지
2. `src/plugins/types.ts`
   플랫폼이 extension에게 무엇을 허용하는지
3. `src/plugins/registry.ts`
   extension가 어떻게 runtime-owned capability가 되는지
4. `src/plugins/loader.ts`
   static manifest, discovery, cache, activation이 어떻게 맞물리는지
5. `src/plugins/runtime/index.ts`
   trusted runtime helper가 무엇인지
6. `src/channels/plugins/types.plugin.ts`
   하나의 channel이 무엇을 소유하는지
7. `src/gateway/server-methods.ts`
   control plane이 무엇을 노출하는지
8. `src/gateway/server.impl.ts`
   runtime kernel이 무엇을 시작하고 소유하는지
9. `src/agents/pi-embedded-runner/run.ts`
   execution runtime이 실제 정책 압력 아래에서 어떻게 동작하는지
10. `src/memory/manager.ts`
    장시간 유지되는 indexed state를 어떻게 관리하는지
11. `src/context-engine/registry.ts`
    exclusive slot이 공유 capability registration과 어떻게 다른지
12. `src/cli/run-main.ts`
    셸이 시스템으로 어떻게 진입하는지
13. `src/commands/*`
    제품 UX가 런타임 위에 어떻게 얹히는지

## 19. 제안하는 Framework 추출 방향

추출된 아키텍처는 대략 다음처럼 생길 가능성이 높습니다.

1. `kernel`
   runtime container, scoped context, lifecycle, event bus
2. `control-plane`
   transport adapter + method registry
3. `capabilities`
   provider, channel, memory, search, media, execution에 대한 공유 contract
4. `extensions`
   manifest, discovery, loader, registry, activation
5. `execution`
   agent runtime, retry, failover, tool system, compaction
6. `state`
   config, session storage, approvals, persistence
7. `surfaces`
   CLI, web UI, mobile, desktop
8. `products`
   channel 및 vendor implementation

OpenClaw 안에는 이미 이 구성 요소 대부분이 들어 있습니다.
지금의 문제는 발명이 아닙니다.
분리와 이름 짓기입니다.

## 20. 최종 요약

OpenClaw은 구조적으로 볼 때 Gateway kernel을 중심으로 한 plugin 중심 agent platform에 가장 가깝습니다.

이 저장소를 다음처럼 리뷰하면:

- "CLI 앱"
- 또는 "챗봇"
- 또는 "모델 provider를 감싼 얇은 wrapper"

실제 아키텍처를 크게 과소평가하게 됩니다.

반대로 다음처럼 보면:

- control-plane kernel
- 여기에 capability registry가 붙고
- 여기에 embedded execution runtime이 붙고
- 그 위에 많은 product adapter가 올라간 구조

저장소 구조가 훨씬 빠르게 이해되기 시작합니다.
