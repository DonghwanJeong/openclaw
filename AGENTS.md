# OpenClaw 에이전트 가이드

- 저장소: `https://github.com/openclaw/openclaw`
- 이 루트 파일이 이 저장소의 주요 에이전트 지침 파일이다. `.cursor/rules/**`, `.cursorrules`, `.github/copilot-instructions.md`는 존재하지 않는다.
- 채팅 답변에서는 항상 저장소 루트 기준 상대 경로만 사용한다. 예: `src/commands/onboard-search.test.ts:42`
- 제한적인 `CODEOWNERS` 규칙이 적용되는 보안 민감 파일은, 해당 소유자가 명시적으로 요청했거나 이미 참여 중인 경우가 아니면 수정하지 않는다.

## 프로젝트 구조

- 핵심 TypeScript 소스는 `src/`에 있다.
- CLI 연결 코드는 `src/cli`, 명령 구현은 `src/commands`에 있다.
- 공용 인프라는 `src/infra`, 미디어 파이프라인은 `src/media`에 있다.
- 워크스페이스 플러그인은 `extensions/*`에 있다.
- 테스트는 보통 소스와 같은 위치의 `*.test.ts`, 엔드투엔드 테스트는 `*.e2e.test.ts`를 사용한다.
- 문서는 `docs/`, 빌드 산출물은 `dist/`에 있다.
- UI 코드는 `ui/`, 모바일/데스크톱 앱은 `apps/`에 있다.

## 런타임과 패키지 매니저

- Node `>=22.16.0`을 사용한다. README 기준 권장 버전은 Node 24이다.
- 기본 패키지 매니저는 `pnpm@10.32.1`이다.
- `bun`으로 직접 TypeScript를 실행할 수는 있지만, 저장소 스크립트는 `pnpm` 기준으로 정의되어 있다.
- 의존성이 없으면 먼저 `pnpm install`을 실행한 뒤, 실패했던 동일 명령을 한 번 더 재시도한다.

## 자주 쓰는 명령

- 의존성 설치: `pnpm install`
- 개발용 CLI 실행: `pnpm openclaw ...`
- 메인 개발 진입점: `pnpm dev`
- Gateway watch 루프: `pnpm gateway:watch`
- 전체 빌드: `pnpm build`
- TypeScript 검사: `pnpm tsgo`
- 전체 저장소 검사: `pnpm check`
- 린트만 실행: `pnpm lint`
- 린트 자동 수정: `pnpm lint:fix`
- 포맷 검사: `pnpm format:check`
- 포맷 적용: `pnpm format` 또는 `pnpm format:fix`
- 전체 테스트 러너: `pnpm test`
- 커버리지: `pnpm test:coverage`
- 빠른 유닛 테스트: `pnpm test:fast`
- Gateway 집중 테스트: `pnpm test:gateway`
- E2E 테스트: `pnpm test:e2e`
- UI 테스트: `pnpm test:ui`
- 문서 검사: `pnpm check:docs`

## 단일 테스트 실행 방식

- 우선 사용하는 타깃 테스트 패턴: `pnpm test -- <path-or-filter> [vitest args...]`
- 파일 기준 예시: `pnpm test -- src/commands/onboard-search.test.ts`
- 파일 + 테스트 이름 예시: `pnpm test -- src/commands/onboard-search.test.ts -t "shows registered plugin providers"`
- 변경된 테스트만 실행: `pnpm test -- --changed origin/main`
- 기본값으로 raw `pnpm vitest run ...`를 사용하지 않는다. `scripts/test-parallel.mjs` 래퍼는 의도된 진입점이다.
- 유닛 테스트만 빠르게 돌릴 때는 `pnpm test:fast`를 사용하며, 내부적으로 `vitest.unit.config.ts`를 쓴다.
- Gateway 전용 테스트는 `pnpm test:gateway`를 사용하며, 내부적으로 `vitest.gateway.config.ts --pool=forks`를 쓴다.
- UI 전용 작업은 필요에 따라 `pnpm test:ui`와 `pnpm --dir ui test`를 사용한다.

## 검증 기준

- 로직을 바꿨다면 먼저 가장 직접적인 범위의 테스트를 실행하고, 영향 범위가 넓으면 점진적으로 검증 범위를 넓힌다.
- 빌드 산출물, 패키징, lazy-loading, 모듈 경계, 배포 표면을 건드렸다면 `pnpm build`가 반드시 통과해야 한다.
- `main`으로 들어갈 가능성이 있는 변경은 기본적으로 `pnpm check`와 `pnpm test`까지 통과하는 수준을 목표로 한다.
- 좁은 범위 테스트가 통과했다고 해서 관련 있어 보이는 다른 실패를 무시하지 않는다.
- 현재 `main`에 이미 관련 없는 실패가 있다면 숨기지 말고 명확하게 언급한다.
- 로컬 Vitest가 메모리 압박을 받으면 `OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test`를 사용한다.

## 언어와 포맷팅

- 기본 언어는 ESM 모듈 기반 TypeScript다.
- 기존 포맷을 따른다. 저장소는 `oxfmt`와 `oxlint`를 사용한다.
- 엄격한 타입을 유지한다. `any`, `@ts-ignore`, `@ts-nocheck`는 피한다.
- 억제 주석보다 명시적 타입과 좁은 union 타입을 선호한다.
- 주석은 꼭 필요한 비자명한 로직에만 짧게 쓴다.
- 코드, 문서, 주석, UI 문자열은 미국식 영어를 사용한다.
- 제품/앱/문서 이름은 `OpenClaw`, CLI/패키지/경로/설정 키는 `openclaw`를 사용한다.

## import 와 모듈 경계

- 수정하기 전에 주변 파일의 import 스타일을 먼저 맞춘다.
- 프로덕션 경로에서 같은 모듈에 대해 static import와 `await import()`를 섞지 않는다.
- lazy loading이 필요하면 전용 `*.runtime.ts` 경계를 만들고, 그 경계를 동적 import 한다.
- lazy-loading 리팩터링 후에는 `pnpm build`를 실행하고 `[INEFFECTIVE_DYNAMIC_IMPORT]` 경고가 없는지 확인한다.
- `extensions/<id>/**` 내부에서는 해당 패키지 루트 바깥으로 나가는 상대 import를 사용하지 않는다.
- 확장 production 코드는 `openclaw/plugin-sdk/*`와 로컬 `api.ts` 또는 `runtime-api.ts` 배럴만 공개 표면으로 취급해야 한다.
- 확장 production 코드에서 core `src/**`, `src/plugin-sdk-internal/**`, 다른 확장의 `src/**`를 import 하지 않는다.
- 확장에 새 경계면이 필요하면 먼저 `openclaw/plugin-sdk/<subpath>` 공개 export를 추가한다.

## 플러그인과 채널 규칙

- 디렉터리가 여전히 `extensions/*`여도 문서/UI/체인지로그에서는 `plugin`이라는 용어를 사용한다.
- 별도 예외가 없으면 `openclaw.plugin.json:id`, `extensions/<id>`, 패키지 이름 사이의 번들 플러그인 ID를 일치시킨다.
- 런타임 플러그인 의존성은 core도 필요하지 않는 한 루트가 아니라 확장 패키지에 둔다.
- 확장 런타임 `dependencies`에 `workspace:*`를 쓰지 말고, `openclaw`에는 `devDependencies` 또는 `peerDependencies`를 선호한다.
- 공용 채널 로직을 리팩터링할 때는 내장 채널과 확장 채널을 함께 고려한다.
- 채널, 확장, 앱, 문서를 추가하면 `.github/labeler.yml`과 대응하는 GitHub 라벨도 함께 업데이트한다.

## 코드 구조 선호사항

- `V2` 복사본이나 큰 중복 분기보다 작은 헬퍼를 선호한다.
- CLI 명령 연결을 건드릴 때는 기존 `createDefaultDeps` 기반 의존성 주입 패턴을 사용한다.
- 파일은 적당히 작게 유지하고, 가독성이나 테스트성이 좋아지면 분리한다.
- 공용 동작을 위해 prototype을 변형하지 않는다. 명시적 상속이나 조합을 선호한다.
- 테스트에서는 `SomeClass.prototype` 패치보다 인스턴스 단위 stub을 선호한다.
- 까다로운 제어 흐름, 프로토콜 세부사항, 경계 조건에는 짧은 설명 주석을 추가해도 된다.

## 에러 처리와 안전성

- 증상만 가리지 말고 근본 원인을 수정한다. cast나 광범위 억제로 에러를 숨기지 않는다.
- 작업 트리의 기존 사용자 변경은 보존하고, 관련 없는 수정은 되돌리지 않는다.
- `node_modules`는 절대 수정하지 않는다.
- Carbon 의존성은 절대 업데이트하지 않는다.
- 명시적 승인 없이 의존성, vendored 코드, `pnpm.patchedDependencies` 항목을 패치하지 않는다.
- `pnpm.patchedDependencies`로 고정된 의존성은 반드시 exact version을 유지한다.

## 테스트 메모

- 기본 테스트 프레임워크는 Vitest이며, lines/branches/functions/statements 모두 70% 커버리지 기준을 사용한다.
- 테스트는 `--isolate=false`에서도 안정적으로 동작하도록 타이머, env var, 글로벌, mock, 소켓, 임시 디렉터리, 모듈 상태를 정리해야 한다.
- 실패를 잠재우기 위해 baseline, inventory, snapshot, ignore list, expected-failure 파일을 수정하지 않는다.
- 테스트 워커 수를 16보다 높이지 않는다.
- 현재 `main` 기준의 새 근거 없이 Vitest VM pool을 다시 도입하지 않는다.
- 테스트에서 샘플 모델 상수를 건드릴 때는 `sonnet-4.6`과 `gpt-5.4`를 선호한다.

## 문서 규칙

- 문서는 Mintlify를 사용하며 `docs/` 아래에 있다.
- 문서 내부 링크는 루트 기준 상대 경로를 사용하고 `.md` / `.mdx` 확장자는 생략한다. 예: `[Config](/configuration)`
- `README.md`에서는 `https://docs.openclaw.ai/...` 형태의 절대 링크를 사용한다.
- 문서는 일반적인 내용으로 유지하고, 개인 기기 이름, 실제 전화번호, 실사용 자격 증명은 포함하지 않는다.
- `docs/zh-CN/**`는 생성 산출물이므로, 해당 파이프라인을 명시적으로 요청받은 경우가 아니면 수정하지 않는다.
- 영문 문서를 수정한 뒤 i18n을 다시 돌릴 때는 먼저 `docs/.i18n/glossary.zh-CN.json`을 업데이트한다.

## 플랫폼별 메모

- SwiftUI 작업에서는 새로운 `ObservableObject`보다 Observation 프레임워크(`@Observable`, `@Bindable`)를 선호한다.
- 모바일 작업에서 simulator를 쓰기 전에 먼저 연결된 실제 디바이스가 있는지 확인한다.
- SSH 환경에서 macOS 앱을 다시 빌드하지 않는다.
- macOS에서 Gateway 디버깅은 ad-hoc tmux 세션이 아니라 앱 또는 `scripts/restart-mac.sh`를 통해 진행한다.
- CLI 진행 UI는 `src/cli/progress.ts`, 공용 터미널 색상은 `src/terminal/palette.ts`를 사용한다.

## 커밋과 PR

- 커밋 시 staging 범위를 제한하려면 `scripts/committer "<msg>" <file...>`를 사용한다.
- 커밋 메시지는 `CLI: add verbose flag to send`처럼 짧고 동작 중심으로 쓴다.
- `main`에서 merge commit을 만들지 말고 `origin/main` 기준 rebase를 사용한다.
- 명시적으로 요청받지 않았다면 `git stash`, `git worktree`, 브랜치 전환을 사용하지 않는다.

## 에이전트 작업 요약

- 수정 전에는 항상 주변 코드를 읽고, 로컬 패턴에 맞춰 구현한다.
- 검증은 가능한 가장 좁은 명령부터 시작하되, 영향 범위가 넓으면 `pnpm build`, `pnpm check`, 더 넓은 테스트로 단계적으로 올린다.
- 추측하지 말고 코드로 확인한 높은 확신의 결론만 보고한다.
