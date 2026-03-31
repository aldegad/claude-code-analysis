# 아키텍처 및 디렉토리 구조 분석 보고서

**분석 대상**: nirholas/claude-code -- src/ 디렉토리
**분석일**: 2026-03-31
**분석자**: 루미 (분석팀장)

---

## 1. 전체 아키텍처 개요

Claude Code는 **React/Ink 기반의 터미널 UI** 위에 **에이전트 루프(tool-call loop)**를 구동하는 구조이다. 사용자 입력이 들어오면 LLM API를 호출하고, LLM이 도구 호출을 요청하면 해당 도구를 실행하고 결과를 다시 LLM에 전달하는 반복 루프를 형성한다.

```
사용자 입력
    |
    v
[REPL / CLI Parser (Commander.js)]
    |
    v
[QueryEngine] <---> [Anthropic API (Streaming)]
    |                       |
    v                       v
[Tool System]         [Response Rendering]
    |                       |
    v                       v
[Permission Check]    [Ink/React Components]
    |
    v
[Tool Execution]
    |
    v
[Result -> QueryEngine 재귀]
```

---

## 2. 디렉토리 구조 및 코드량 분포

### 2.1 코드량 순위 (라인 수 기준)

| 순위 | 디렉토리 | 라인 수 | 파일 수 | 역할 |
|------|----------|---------|---------|------|
| 1 | **utils/** | 181,036 | 564 | 유틸리티 (배시 파서, 플러그인, 권한, 설정 등) |
| 2 | **components/** | 81,963 | 389 | Ink/React UI 컴포넌트 |
| 3 | **services/** | 54,843 | 136 | 외부 서비스 통합 (API, MCP, OAuth, LSP 등) |
| 4 | **tools/** | 51,012 | 184 | 에이전트 도구 구현체 |
| 5 | **commands/** | 26,818 | 190 | 슬래시 커맨드 구현체 |
| 6 | **ink/** | 19,938 | 96 | Ink 렌더러 래퍼 및 확장 |
| 7 | **hooks/** | 19,308 | 104 | React 훅 (권한, 키바인딩, 설정 등) |
| 8 | **bridge/** | 12,675 | 31 | IDE 통합 브릿지 (VS Code, JetBrains) |
| 9 | **cli/** | 12,391 | 19 | CLI 출력, 헤드리스 모드 |
| 10 | **screens/** | 5,980 | 3 | 전체 화면 UI (REPL, Doctor, Resume) |
| 11 | **skills/** | 4,086 | 20 | 스킬 시스템 |
| 12 | **native-ts/** | 4,085 | 4 | 네이티브 TS 유틸 (yoga-layout 등) |
| 13 | **entrypoints/** | 4,059 | 8 | 초기화 로직, Agent SDK, MCP 엔트리 |
| 14 | **types/** | 3,467 | 12 | TypeScript 타입 정의 |
| 15 | **tasks/** | 3,298 | 12 | 태스크 관리 |
| **src 루트 파일** | | ~15,700 | 18 | 핵심 엔트리 + 레지스트리 |

**utils가 전체 코드의 약 35%를 차지**하며, 이는 Bash 파서, 플러그인 시스템, 권한 시스템, 설정 관리 등 핵심 인프라가 유틸리티로 분류되어 있기 때문이다.

### 2.2 가장 큰 파일 Top 15

| 파일 | 라인 수 | 역할 |
|------|---------|------|
| `cli/print.ts` | 5,596 | CLI 출력 포매팅 |
| `utils/messages.ts` | 5,513 | 메시지 처리 유틸리티 |
| `utils/sessionStorage.ts` | 5,106 | 세션 스토리지 관리 |
| `utils/hooks.ts` | 5,023 | 훅 시스템 (Git hooks 등) |
| `screens/REPL.tsx` | 5,006 | REPL 전체 화면 UI |
| `main.tsx` | 4,684 | CLI 엔트리포인트 + Commander 파서 |
| `utils/bash/bashParser.ts` | 4,437 | Bash 명령어 파서 |
| `utils/attachments.ts` | 3,998 | 첨부 파일 처리 |
| `services/api/claude.ts` | 3,420 | Anthropic API 클라이언트 |
| `services/mcp/client.ts` | 3,349 | MCP 클라이언트 |
| `utils/plugins/pluginLoader.ts` | 3,303 | 플러그인 로더 |
| `commands/insights.ts` | 3,202 | 인사이트 커맨드 |
| `bridge/bridgeMain.ts` | 3,001 | IDE 브릿지 메인 루프 |
| `utils/bash/ast.ts` | 2,680 | Bash AST 처리 |
| `utils/plugins/marketplaceManager.ts` | 2,644 | 플러그인 마켓플레이스 |

---

## 3. 핵심 모듈 분석

### 3.1 엔트리포인트 및 초기화 시퀀스

```
src/entrypoints/cli.tsx          -- 최상위 엔트리포인트
    |
    +--> --version 패스트패스   -- 제로 모듈 로딩
    +--> --dump-system-prompt   -- 시스템 프롬프트 덤프 (내부용)
    +--> --chrome-native-host   -- Chrome 네이티브 호스트
    |
    v
src/main.tsx                     -- Commander.js CLI 파서
    |
    +--> startMdmRawRead()       -- MDM 설정 프리페치 (병렬)
    +--> startKeychainPrefetch() -- 키체인 프리페치 (병렬)
    |
    v
src/entrypoints/init.ts          -- Config, 텔레메트리, OAuth, MDM
    |
    v
src/replLauncher.tsx              -- REPL 세션 런처
    |
    v
src/screens/REPL.tsx              -- REPL 전체 화면 UI
```

특이점: **패스트 패스(fast-path)** 패턴으로 `--version` 같은 단순 명령은 모듈을 전혀 로딩하지 않고 즉시 응답한다.

### 3.2 쿼리 엔진 (QueryEngine.ts)

LLM API 호출의 핵심 엔진. 주요 책임:

- **스트리밍 응답 처리**: Anthropic SDK를 통한 스트리밍 API 호출
- **도구 호출 루프**: LLM이 tool_use를 반환하면 도구 실행 -> 결과 전달 -> 재귀 호출
- **씽킹 모드**: 확장된 사고 과정 지원
- **재시도 로직**: API 에러 분류 및 자동 재시도
- **토큰 카운팅**: 사용량 추적
- **컨텍스트 관리**: 시스템 프롬프트, 메모리, 파일 상태 캐시

import 분석을 보면 이 파일이 거의 모든 주요 모듈과 연결되어 있어, 사실상 애플리케이션의 "심장"이다.

### 3.3 도구 시스템 (tools/)

각 도구는 독립적인 디렉토리 구조로 구성:

```
src/tools/BashTool/
    ├── BashTool.tsx           -- 메인 도구 로직
    ├── bashPermissions.ts     -- 권한 검증 (2,622줄)
    ├── bashSecurity.ts        -- 보안 검증 (2,593줄)
    ├── UI.tsx                 -- 렌더링
    ├── prompt.ts              -- 시스템 프롬프트
    └── toolName.ts            -- 상수
```

`buildTool()` 팩토리 함수를 통해 일관된 인터페이스:
- `call()` -- 실행 로직
- `checkPermissions()` -- 권한 검사
- `isConcurrencySafe()` -- 병렬 실행 가능 여부
- `isReadOnly()` -- 읽기 전용 여부
- `prompt()` -- 시스템 프롬프트 주입
- `renderToolUseMessage()` / `renderToolResultMessage()` -- UI 렌더링

**BashTool이 가장 복잡**: 권한(2,622줄) + 보안(2,593줄) = 5,215줄의 보안 로직만으로도 독립 모듈급 규모.

### 3.4 커맨드 시스템 (commands/)

세 가지 타입:

| 타입 | 설명 | 예시 |
|------|------|------|
| **PromptCommand** | LLM에 프롬프트를 보내고 도구 사용 허용 | `/commit`, `/review` |
| **LocalCommand** | 로컬에서 실행, 텍스트 반환 | `/cost`, `/clear` |
| **LocalJSXCommand** | 로컬에서 실행, React JSX 반환 | `/doctor`, `/config` |

`src/commands.ts`에서 중앙 레지스트리로 관리.

### 3.5 권한 시스템

`src/utils/permissions/`에 23개 파일, `src/hooks/toolPermission/`에 추가 훅.

권한 모드:
- `default` -- 매 작업마다 사용자 승인 요청
- `plan` -- 계획을 먼저 제시하고 구현 단계 진입 시 승인 요청
- `bypassPermissions` -- 권한 검사를 우회해 자동 승인 (위험)
- `auto` -- ML 분류기 기반 자동 결정
- `acceptEdits` -- 작업 디렉터리 내 파일 수정은 자동 허용하되, 그 외 작업은 별도 판단
- `dontAsk` -- 추가 질문이 필요한 작업은 승인 프롬프트 대신 거부

와일드카드 패턴 매칭: `Bash(git *)`, `FileEdit(/src/*)` 형태.

### 3.6 브릿지 시스템 (bridge/)

IDE(VS Code, JetBrains)와의 양방향 통신 레이어. 31개 파일, 12,675줄.

핵심 구성:
- `bridgeMain.ts` (3,001줄) -- 브릿지 메인 루프
- `replBridge.ts` (2,408줄) -- REPL 세션 브릿지
- `bridgeMessaging.ts` -- 메시지 프로토콜
- `jwtUtils.ts` -- JWT 기반 인증
- `sessionRunner.ts` -- 세션 실행 관리

### 3.7 서비스 레이어 (services/)

| 서비스 | 역할 |
|--------|------|
| `api/` | Anthropic SDK 클라이언트, 파일 API, 부트스트랩 |
| `mcp/` | MCP 클라이언트, 도구/리소스 디스커버리, 인증 |
| `oauth/` | OAuth 2.0 인증 플로우 |
| `lsp/` | Language Server Protocol 매니저 |
| `analytics/` | GrowthBook 기반 피처 플래그, A/B 테스팅 |
| `plugins/` | 플러그인 로더, 마켓플레이스 |
| `compact/` | 대화 컨텍스트 압축 |
| `policyLimits/` | 조직 정책 제한 |
| `remoteManagedSettings/` | Enterprise용 원격 관리 설정 |
| `extractMemories/` | 자동 메모리 추출 |
| `teamMemorySync/` | 팀 메모리 동기화 |
| `x402/` | 결제 관련 |
| `MagicDocs/` | 자동 문서 생성 |

### 3.8 상태 관리 (state/)

React Context + 커스텀 스토어 패턴:

- `AppState.tsx` -- React Context Provider
- `AppStateStore.ts` -- 커스텀 스토어
- `store.ts` -- 스토어 기본 구현
- `selectors.ts` -- 파생 상태 셀렉터
- `onChangeAppState.ts` -- 상태 변경 옵저버

뮤터블 상태 객체가 도구 컨텍스트로 전달되는 구조.

---

## 4. 핵심 아키텍처 패턴

### 4.1 병렬 프리페치 (Parallel Prefetch)

시작 시간 최적화를 위해 MDM 설정, 키체인 읽기, API 프리커넥트를 병렬로 실행. 무거운 모듈 평가가 시작되기 전에 사이드이펙트로 발동.

### 4.2 지연 로딩 (Lazy Loading)

OpenTelemetry (~400KB), gRPC (~700KB) 같은 무거운 모듈은 `dynamic import()`로 실제 필요한 시점까지 지연.

```typescript
const getModule = () => require('./heavy.js')  // 순환 의존성 방지
```

### 4.3 피처 플래그를 통한 죽은 코드 제거

`bun:bundle`의 `feature()` 함수로 빌드 타임에 비활성 코드를 완전히 제거:

```typescript
import { feature } from 'bun:bundle'
if (feature('VOICE_MODE')) { /* ... */ }
```

주요 플래그: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `COORDINATOR_MODE`, `WORKFLOW_SCRIPTS`, `ABLATION_BASELINE`, `DUMP_SYSTEM_PROMPT`

### 4.4 에이전트 스웜 (Agent Swarms)

`AgentTool`로 서브에이전트를 생성하고, `coordinator/`가 다중 에이전트 오케스트레이션을 처리. `TeamCreateTool`로 팀 레벨의 병렬 작업 가능.

### 4.5 플러그인 + 스킬 아키텍처

두 가지 확장 메커니즘:
- **플러그인**: 외부 패키지 로딩, 마켓플레이스 기반 검색/설치
- **스킬**: 번들된 워크플로, YAML/마크다운 정의

---

## 5. 모듈 의존성 핵심 흐름

```
entrypoints/cli.tsx
    -> main.tsx (Commander 파싱)
        -> replLauncher.tsx (REPL 시작)
            -> screens/REPL.tsx (전체 화면 UI)
                -> QueryEngine.ts (LLM 호출)
                    -> query.ts (쿼리 파이프라인)
                        -> services/api/claude.ts (API 클라이언트)
                    -> tools.ts (도구 레지스트리)
                        -> tools/* (개별 도구)
                    -> commands.ts (커맨드 레지스트리)
                        -> commands/* (개별 커맨드)
                    -> context.ts (컨텍스트 수집)
                        -> constants/prompts.ts (시스템 프롬프트)
                        -> memdir/memdir.ts (메모리)
                    -> cost-tracker.ts (비용 추적)
                    -> history.ts (세션 히스토리)
                -> components/* (UI 렌더링)
                -> hooks/* (상태 + 권한 + 키바인딩)
                -> state/* (앱 상태)
```

---

## 6. 설정 체계

다층 설정 구조:

| 레벨 | 위치 | 용도 |
|------|------|------|
| **글로벌** | `~/.claude/config.json`, `~/.claude/settings.json` | 사용자 전역 설정 |
| **프로젝트** | `.claude/config.json`, `.claude/settings.json` | 프로젝트별 설정 |
| **시스템** | macOS 키체인 + MDM, Windows 레지스트리 + MDM | OS 레벨 관리 |
| **원격** | Enterprise 원격 동기화 | 관리형 설정 |

Zod 스키마(`src/schemas/`)로 설정 검증, `src/migrations/`로 버전 간 마이그레이션 처리.
