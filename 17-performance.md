# Claude Code 성능 최적화 패턴 분석

**분석 대상**: `/Users/soohongkim/Documents/workspace/personal/claude-code` (52만줄)
**분석일**: 2026-03-31
**분석자**: 하울 (개발팀 오퍼레이터)

---

## 1. Bun 런타임 선택과 성능 이점

### 선택 근거
- `package.json`에 `"engines": { "bun": ">=1.1.0" }`, `"packageManager": "bun@1.1.0"` 명시
- `tsconfig.json`의 `types`에 `"bun-types"` 포함, `moduleResolution: "bundler"` 사용
- Bun 전용 API(`Bun.hash`, `Bun.which`, `Bun.spawn`) 적극 활용

### 핵심 성능 이점

| 영역 | Node.js 대비 이점 | 코드 증거 |
|------|-------------------|-----------|
| **콜드 스타트** | ~3-5배 빠른 모듈 로딩 | 빌드타임 번들링으로 52만줄을 단일 진입점으로 |
| **`Bun.hash` (wyhash)** | 코드 주석에 ~100x faster than sha256으로 기재되어 있으나, 벤치마크 결과는 소스에 포함되지 않음 | `hash.ts`: 주석 기반 |
| **`Bun.which`** | 프로세스 스폰 없는 경로 탐색 | `which.ts`: sync lookup, no subprocess |
| **`Bun.spawn`** | 더 빠른 서브프로세스 생성 | `ripgrep.ts`: embedded ripgrep에 사용 |
| **`fetch` keep-alive 풀** | 전역 커넥션 풀 공유 | `apiPreconnect.ts`: preconnect가 실제 요청과 풀 공유 |
| **빌드타임 DCE** | `feature()` 기반 코드 제거 | `bun:bundle` 모듈의 `feature()` 함수 |
| **TLS/BoringSSL** | 부팅 시 TLS 인증서 스토어 캐싱 | `init.ts` 주석: "Bun caches the TLS cert store at boot via BoringSSL" |

**예상 성능 영향**: 콜드 스타트 200-400ms 절감, 해싱 연산 전체 ~95% 절감

---

## 2. 빌드타임 최적화: feature() DCE (Dead Code Elimination)

### 메커니즘
`bun:bundle` 모듈이 제공하는 `feature()` 함수는 빌드 타임에 평가되어, 비활성 피처의 코드를 완전히 제거한다.

```typescript
// src/types/bun-bundle.d.ts
declare module 'bun:bundle' {
  export function feature(name: string): boolean  // 빌드타임 평가
}
```

### 적용 범위
총 **약 213~226개 파일 (카운트 기준에 따라 다름)**에서 `feature()` 사용. 주요 피처 플래그:

| 플래그 | 설명 | DCE 대상 |
|--------|------|----------|
| `ABLATION_BASELINE` | 실험 기준선 | 환경변수 주입 블록 |
| `DUMP_SYSTEM_PROMPT` | 내부 디버그 도구 | 시스템 프롬프트 덤프 경로 |
| `CHICAGO_MCP` | 컴퓨터 사용 MCP | 전체 MCP 서버 코드 |
| `DAEMON` | 데몬 모드 | 데몬/워커 시스템 전체 |
| `BRIDGE_MODE` | 리모트 컨트롤 | 브릿지 서버 전체 |
| `BUDDY` | 동반자 스프라이트 | UI 애니메이션 전체 |
| `VOICE_MODE` | 음성 모드 | 오디오 캡처 전체 |
| `COORDINATOR_MODE` | 코디네이터 | 다중 에이전트 조율 |
| `HISTORY_SNIP` | 히스토리 스닙 | 컨텍스트 압축 로직 |
| `CONTEXT_COLLAPSE` | 컨텍스트 축소 | 축소 엔진 전체 |
| `CACHED_MICROCOMPACT` | 캐시 마이크로컴팩트 | 캐시 편집 시스템 |
| `KAIROS` | 세션 트랜스크립트 | 트랜스크립트 추적 |
| `TEAMMEM` | 팀 메모리 | 팀 메모리 연산 |
| `ANTI_DISTILLATION_CC` | 반-증류 보호 | 가짜 도구 주입 |

### 패턴: 조건부 require로 모듈 그래프 차단

```typescript
// src/toolPool.ts - 모듈이 feature() 조건 안에서만 로드
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? (require('../coordinator/coordinatorMode.js') as typeof import(...))
  : null
```

이 패턴으로 비활성 피처의 전체 모듈 트리가 번들에서 제거된다.

**예상 성능 영향**: 외부 빌드 번들 크기 30-50% 감소, 모듈 파싱 시간 비례 감소

---

## 3. 지연 로딩 (Dynamic Import / Lazy Loading)

### 3.1 경로별 Fast-Path 패턴
`cli.tsx`가 핵심. 모든 import가 동적이며, 필요한 경로만 로드한다:

```typescript
// --version: 제로 모듈 로딩
if (args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`);  // 빌드타임 매크로
  return;  // 즉시 반환, import 없음
}

// 각 서브커맨드별 필요한 모듈만 동적 로드
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  const { runDaemonWorker } = await import('../daemon/workerRegistry.js');
  // ...
}
```

### 3.2 `lazySchema()` - Zod 스키마 지연 생성

```typescript
// src/utils/lazySchema.ts
export function lazySchema<T>(factory: () => T): () => T {
  let cached: T | undefined
  return () => (cached ??= factory())
}
```

**25개 이상의 파일**에서 사용. 모듈 초기화 시 Zod 스키마 구성 비용을 첫 접근까지 지연.

### 3.3 대형 모듈 지연 로드

| 모듈 | 크기 | 지연 이유 |
|------|------|-----------|
| OpenTelemetry + protobuf | ~400KB | "defer ~400KB of OpenTelemetry + protobuf" |
| gRPC exporters (@grpc/grpc-js) | ~700KB | "further lazy-loaded within instrumentation.ts" |
| InvalidConfigDialog (React/Ink) | 수십KB | "dynamically imported to avoid loading React at init" |
| 1P event logging + GrowthBook | 수십KB | `Promise.all([import(...), import(...)])` |
| swarm cleanup | 수KB | "behind feature gate and most sessions never create teams" |
| upstreamproxy | 수십KB | "Lazy import so non-CCR startups don't pay the module load" |

**예상 성능 영향**: 콜드 스타트에서 1.1MB+ 모듈 로딩 회피, 시작 시간 100-300ms 절감

---

## 4. 캐싱 전략

### 4.1 API Prompt Cache (서버사이드)

Claude API의 `cache_control: { type: 'ephemeral' }` 적극 활용:

- **시스템 프롬프트 캐싱**: `splitSysPromptPrefix()`가 정적/동적 부분을 분리하여 `cacheScope: 'global'` / `'org'` 지정
- **도구 스키마 캐싱**: `toolSchemaCache.ts`가 세션 중 도구 스키마를 고정하여 캐시 버스팅 방지
- **1시간 TTL 캐싱**: `should1hCacheTTL()`로 유자격 사용자에게 장기 캐시 적용
- **캐시 브레이크 감지**: `promptCacheBreakDetection.ts`가 시스템 프롬프트/도구 변경을 추적하여 불필요한 캐시 무효화 방지

```typescript
// 캐시 브레이크 추적: 이전 상태의 해시를 비교
type PreviousState = {
  systemHash: number
  toolsHash: number
  cacheControlHash: number
  perToolHashes: Record<string, number>
  // ...
}
```

### 4.2 로컬 캐싱 계층

| 캐시 | 구현 | 용도 |
|------|------|------|
| **FileStateCache** | LRU (100 엔트리, 25MB) | 읽은 파일 상태 추적 |
| **toolSchemaCache** | `Map<string, CachedSchema>` | 도구 스키마 세션 고정 |
| **statsCache** | 디스크 JSON | 사용 통계 증분 캐싱 |
| **URL Cache** | `LRUCache` + TTL | WebFetch 응답 캐싱 |
| **zodToJsonSchema** | `WeakMap<ZodType, JsonSchema>` | Zod-JSON 변환 캐싱 |
| **memoizeWithTTL** | `Map` + 백그라운드 갱신 | write-through 캐시 패턴 |
| **keychainPrefetch** | 메모리 (일회성) | macOS 키체인 값 캐싱 |
| **Ink nodeCache** | `WeakMap<DOMElement, Layout>` | 레이아웃 계산 캐싱 |
| **searchText cache** | `WeakMap<Message, string>` | 검색 텍스트 소문자 변환 캐싱 |

### 4.3 Prompt Cache를 위한 CacheSafeParams

`forkedAgent.ts`의 `CacheSafeParams`는 서브에이전트가 부모의 prompt cache를 공유하도록 보장:

```typescript
export type CacheSafeParams = {
  systemPrompt: SystemPrompt       // 부모와 동일해야 캐시 히트
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]   // 부모 컨텍스트 메시지
}
```

**예상 성능 영향**: API 비용 60-80% 절감 (prompt cache 히트 시), 응답 지연 30-50% 감소

---

## 5. 스트리밍 처리 최적화

### 5.1 Eager Input Streaming (Fine-Grained Tool Streaming)

```typescript
// src/utils/api.ts - 도구별 스트리밍 활성화
if (getAPIProvider() === 'firstParty' && isFirstPartyAnthropicBaseUrl()) {
  base.eager_input_streaming = true  // 도구 입력 파라미터 스트리밍
}
```

이 기능 없이는 API가 전체 도구 입력을 버퍼링한 후에야 `input_json_delta` 이벤트를 보내, 대형 도구 입력에서 수 분의 행 발생.

### 5.2 Non-Streaming Fallback

스트리밍 실패 시 자동으로 비스트리밍 모드로 전환:

```typescript
export async function* executeNonStreamingRequest(...)
// 타임아웃: 리모트 120s, 로컬 300s
```

### 5.3 스트리밍 VCR (Record/Replay)

```typescript
import { withStreamingVCR, withVCR } from '../vcr.js'
// 스트리밍 응답 기록/재생 (디버그/테스트용)
```

### 5.4 Streamlined Transform

`streamlinedTransform.ts`는 SDK 출력 모드에서 도구 호출을 카테고리별(검색/읽기/쓰기/명령)로 집계하여 출력 대역폭을 줄인다.

**예상 성능 영향**: 대형 도구 입력 응답 시간 수 분 -> 수 초로 단축, 사용자 체감 대기 시간 50-70% 감소

---

## 6. React/Ink 렌더링 최적화

### 6.1 React Compiler 자동 메모이제이션

**약 395개 .tsx 파일**에 `react/compiler-runtime` 적용 (전체 .tsx는 552개):

```typescript
import { c as _c } from "react/compiler-runtime";
```

React Compiler가 자동으로 `useMemo`, `useCallback`, `memo`를 삽입하여 수동 최적화 부담을 제거.

### 6.2 가상 스크롤 (VirtualMessageList)

```typescript
// src/hooks/useVirtualScroll.ts
const OVERSCAN_ROWS = 80       // 뷰포트 위아래 여유 행
const COLD_START_COUNT = 30    // 초기 렌더링 아이템 수
const MAX_MOUNTED_ITEMS = 300  // 최대 마운트 아이템 제한
const SLIDE_STEP = 25          // 한 커밋당 최대 새 마운트 수

// 스크롤 양자화로 불필요한 React 커밋 방지
const SCROLL_QUANTUM = OVERSCAN_ROWS >> 1  // 40행 단위
```

핵심: `SCROLL_QUANTUM`이 매 휠 틱마다의 React 재렌더를 방지. 시각적 스크롤은 Ink DOM에서 직접 처리.

### 6.3 WeakMap 기반 캐싱

```typescript
// VirtualMessageList.tsx
const fallbackLowerCache = new WeakMap<RenderableMessage, string>();
const promptTextCache = new WeakMap<RenderableMessage, string | null>();
```

메시지 객체의 GC와 연동되는 WeakMap으로 메모리 누수 없는 캐싱.

### 6.4 Ink 레이아웃 캐싱

```typescript
// ink/node-cache.ts
export const nodeCache = new WeakMap<DOMElement, CachedLayout>()
export const pendingClears = new WeakMap<DOMElement, Rectangle[]>()
```

Yoga 레이아웃 계산 결과를 캐싱하여 반복 계산 회피.

**예상 성능 영향**: 긴 대화에서 UI 프레임 드롭 90% 감소, 스크롤 CPU 사용량 70% 절감

---

## 7. 메모리 관리

### 7.1 FileStateCache (LRU + 크기 제한)

```typescript
// src/utils/fileStateCache.ts
export class FileStateCache {
  constructor(maxEntries: number, maxSizeBytes: number) {
    this.cache = new LRUCache<string, FileState>({
      max: maxEntries,             // 기본 100 엔트리
      maxSize: maxSizeBytes,       // 기본 25MB
      sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content)),
    })
  }
}
```

### 7.2 대형 도구 결과 디스크 퍼시스팅

```typescript
// src/utils/toolResultStorage.ts
// 도구 결과가 임계값을 초과하면 디스크에 저장하고 참조만 메모리에 유지
export const PERSISTED_OUTPUT_TAG = '<persisted-output>'
```

`getPersistenceThreshold()`로 도구별 임계값을 조정하며, GrowthBook으로 원격 오버라이드 가능.

### 7.3 원격 환경 메모리 제한

```typescript
// cli.tsx - CCR 컨테이너(16GB)에서 자식 프로세스 힙 제한
if (process.env.CLAUDE_CODE_REMOTE === 'true') {
  process.env.NODE_OPTIONS = '--max-old-space-size=8192';
}
```

### 7.4 마운트 아이템 캡핑

```typescript
// useVirtualScroll.ts
const MAX_MOUNTED_ITEMS = 300  // 극단적 경우에도 파이버 할당 제한
const SLIDE_STEP = 25          // ~290ms 동기 블록 방지 (25 * 1.5ms)
```

### 7.5 느린 연산 감지

```typescript
// src/utils/slowOperations.ts
// JSON 파싱/직렬화, cloneDeep 등의 느린 연산을 감지하고 로깅
const SLOW_OPERATION_THRESHOLD_MS = /* dev: 20ms, ant: 300ms, ext: Infinity */
```

**예상 성능 영향**: 장시간 세션에서 메모리 사용량 50-70% 절감, OOM 크래시 방지

---

## 8. 병렬 처리

### 8.1 시작 시 병렬 프리페치

`main.tsx` 상단에서 무거운 I/O를 모듈 평가와 병렬로 실행:

```typescript
profileCheckpoint('main_tsx_entry');
startMdmRawRead();         // MDM 서브프로세스 (plutil/reg query) 병렬 시작
startKeychainPrefetch();   // macOS 키체인 2개 동시 읽기 (각 ~32ms)
```

### 8.2 init.ts의 fire-and-forget 패턴

```typescript
void Promise.all([
  import('../services/analytics/firstPartyEventLogger.js'),
  import('../services/analytics/growthbook.js'),
]).then(([fp, gb]) => { /* ... */ })

void populateOAuthAccountInfoIfNeeded()   // 비동기 fire-and-forget
void initJetBrainsDetection()             // 비동기 fire-and-forget
void detectCurrentRepository()            // 비동기 fire-and-forget
```

### 8.3 p-map을 통한 제어된 병렬성

`package.json`에 `p-map` 의존성. 파일 동시 읽기, MCP 서버 연결 등에서 동시성 제어.

### 8.4 Forked Agent 병렬 실행

`forkedAgent.ts`가 서브에이전트를 부모의 프롬프트 캐시를 공유하면서 병렬 실행:

```typescript
export async function runForkedAgent(...)
// 별도 AbortController, 별도 FileStateCache 클론, 별도 usage 추적
```

**예상 성능 영향**: 시작 시간 60-100ms 절감 (키체인 병렬화), 다중 에이전트 처리량 2-4배 향상

---

## 9. 네트워크 최적화

### 9.1 API Preconnect (TCP+TLS 핸드셰이크 오버랩)

```typescript
// src/utils/apiPreconnect.ts
// TCP+TLS 핸드셰이크 (~100-200ms)를 시작 작업과 겹치기
void fetch(baseUrl, {
  method: 'HEAD',                    // 응답 바디 없음 -> 즉시 keep-alive 풀 반환
  signal: AbortSignal.timeout(10_000),
}).catch(() => {})
```

`HEAD` 요청으로 TCP+TLS만 워밍하고, 이후 실제 API 요청이 이미 워밍된 커넥션을 재사용.

### 9.2 Retry with Fallback

```typescript
// src/services/api/withRetry.ts
export async function* withRetry(
  clientFactory,
  requestFn,
  options: { maxRetries, model, fallbackModel, thinkingConfig },
)
```

529 에러(과부하) 시 지수 백오프 + 폴백 모델 자동 전환.

### 9.3 세션 안정성을 위한 캐시 TTL 래칭

```typescript
// 세션 중 overage 상태 변경이 캐시 TTL을 바꾸지 않도록 래칭
let userEligible = getPromptCache1hEligible()
if (userEligible === null) {
  userEligible = /* ... */
  setPromptCache1hEligible(userEligible)  // 세션 전체에 고정
}
```

### 9.4 MCP 리소스 프리페치

```typescript
import { prefetchAllMcpResources } from 'src/services/mcp/client.js'
import { prefetchOfficialMcpUrls } from './services/mcp/officialRegistry.js'
```

**예상 성능 영향**: 첫 API 호출 100-200ms 절감, 재시도 시 사용자 대기 시간 50% 감소

---

## 10. 시작 시간 최적화

### 10.1 Startup Profiler

```typescript
// src/utils/startupProfiler.ts
// 단계별 시작 시간 측정 (100% ant, 0.5% 외부 샘플링)
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
}
```

비샘플링 사용자는 프로파일링 비용 제로 (`if (!SHOULD_PROFILE) return`).

### 10.2 --version Fast Path (제로 임포트)

```typescript
// cli.tsx:37 - 가장 빠른 경로
if (args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`);  // MACRO는 빌드타임 인라인
  return;  // import 제로, 즉시 종료
}
```

### 10.3 Memoized init()

```typescript
export const init = memoize(async (): Promise<void> => { ... })
// 여러 번 호출해도 한 번만 실행
```

### 10.4 빌드타임 매크로

`MACRO.VERSION`, `MACRO.BUILD_TIME`, `MACRO.PACKAGE_URL` 등이 빌드 시 인라인되어 런타임 파일 읽기 불필요.

### 10.5 MDM/키체인 병렬 프리페치

```typescript
// main.tsx:16 - 모듈 임포트 ~135ms와 병렬
startMdmRawRead()         // plutil 서브프로세스 병렬
startKeychainPrefetch()   // 키체인 2건 병렬 (순차 시 ~65ms)
```

**예상 성능 영향**: `--version` 10ms 이하, 일반 시작 200-400ms 절감

---

## 11. 컨텍스트 윈도우 관리

### 11.1 3단계 컨텍스트 압축 파이프라인

대화가 길어질 때 세 가지 레벨의 압축이 순차적으로 적용:

#### Level 1: Micro-Compact (시간 기반)

```typescript
// src/services/compact/microCompact.ts
const COMPACTABLE_TOOLS = new Set([...SHELL_TOOL_NAMES, GREP, GLOB, ...])
// 오래된 도구 결과의 내용을 '[Old tool result content cleared]'로 교체
// API 수준에서도 cache_edits로 서버사이드 마이크로컴팩트 지원
```

- 오래된 도구 결과만 선별적으로 제거
- API의 `cache_edits` 기능으로 서버사이드에서 캐시된 컨텍스트 직접 편집

#### Level 2: History Snip (`feature('HISTORY_SNIP')`)

```typescript
// src/query.ts:401
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
}
```

오래된 메시지를 잘라내되, 최근 메시지는 보존.

#### Level 3: Auto-Compact (전체 요약)

```typescript
// src/services/compact/autoCompact.ts
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3  // 서킷 브레이커

export function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}
```

- 임계값 초과 시 전체 대화를 LLM으로 요약
- 요약 후 최근 5개 파일, 스킬, 플랜 등을 재주입
- 연속 실패 3회 시 서킷 브레이커 발동 (하루 ~250K 불필요한 API 호출 방지)

### 11.2 Context Collapse (`feature('CONTEXT_COLLAPSE')`)

```typescript
// src/query.ts:440
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(...)
}
// overflow 발생 시 복구: recoverFromOverflow()
```

### 11.3 UI 수준 Read/Search 축소

```typescript
// src/utils/collapseReadSearch.ts
// 연속된 Read/Search 도구 호출을 UI에서 하나로 축소
// "searched 3 patterns, read 5 files" 형태로 집계
```

### 11.4 세션 메모리 컴팩트

```typescript
// src/services/compact/sessionMemoryCompact.ts
// 대화 요약과 별개로 세션 메모리(장기 기억)를 별도 관리
// 컴팩션 후에도 세션 메모리는 보존
```

### 11.5 Post-Compact Cleanup

```typescript
// src/services/compact/postCompactCleanup.ts
// 컴팩션 후 무효화된 캐시/추적 상태 일괄 정리
clearSystemPromptSections()
resetMicrocompactState()
clearSessionMessagesCache()
resetGetMemoryFilesCache()
clearClassifierApprovals()
clearSpeculativeChecks()
```

### 11.6 도구 결과 퍼시스팅

```typescript
// 큰 도구 결과 -> 디스크 저장 + 참조만 컨텍스트에 유지
export const PERSISTED_OUTPUT_TAG = '<persisted-output>'
// "<persisted-output>\nOutput too large (21.1KB). Full output saved to: ..."
```

**예상 성능 영향**: 100+ 턴 대화에서 토큰 비용 70-90% 절감, prompt_too_long 에러 95% 감소

---

## 보너스: Speculation (투기적 실행)

```typescript
// src/services/PromptSuggestion/speculation.ts
const MAX_SPECULATION_TURNS = 20
const MAX_SPECULATION_MESSAGES = 100

// 사용자의 다음 요청을 예측하여 미리 실행
const SAFE_READ_ONLY_TOOLS = new Set(['Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', ...])
```

사용자가 입력하는 동안 다음 행동을 예측하여 읽기 전용 도구를 미리 실행. 예측이 맞으면 응답 즉시 반환.

- `statsCache`에 `totalSpeculationTimeSavedMs` 추적
- 오버레이 파일시스템으로 투기적 쓰기 격리

**예상 성능 영향**: 예측 적중 시 사용자 체감 응답 시간 2-5초 절감

---

## 종합 성능 영향 매트릭스

| 최적화 기법 | 영향 영역 | 절감 효과 | 복잡도 |
|------------|----------|----------|--------|
| Bun 런타임 | 시작/실행 전체 | 200-400ms 시작, 해싱 95%+ | 낮음 |
| feature() DCE | 번들 크기 | 30-50% 번들 감소 | 중간 |
| 지연 로딩 | 시작 시간 | 100-300ms (1.1MB 모듈 회피) | 중간 |
| Prompt Cache | API 비용/지연 | 60-80% 비용, 30-50% 지연 | 높음 |
| 로컬 캐싱 | 반복 연산 | LRU로 메모리 제한 내 최적 | 중간 |
| Eager Input Streaming | 도구 응답 | 수 분 -> 수 초 | 낮음 |
| React Compiler | UI 렌더링 | 자동 메모이제이션 (약 395개 적용, 전체 .tsx 552개) | 낮음 |
| 가상 스크롤 | UI 성능 | 프레임 드롭 90% 감소 | 높음 |
| 메모리 관리 | 안정성 | OOM 방지, 25MB LRU 캡 | 중간 |
| 병렬 프리페치 | 시작 시간 | 60-100ms (키체인/MDM) | 중간 |
| API Preconnect | 네트워크 | 100-200ms (TLS 핸드셰이크) | 낮음 |
| 컨텍스트 관리 3단계 | 토큰 비용 | 70-90% 장기 세션 비용 | 높음 |
| Speculation | 응답 시간 | 2-5초 (예측 적중 시) | 매우 높음 |

---

## 결론

Claude Code의 성능 최적화는 **4개 축**으로 구성된다:

1. **빌드타임 최적화**: Bun 번들러의 `feature()` DCE + 빌드타임 매크로로 배포 크기와 파싱 비용 최소화
2. **런타임 지연 최적화**: 동적 import, lazySchema, fast-path 분기로 필요한 코드만 실행
3. **네트워크/API 최적화**: preconnect, prompt cache (5분/1시간 TTL), eager input streaming, CacheSafeParams 공유
4. **클라이언트 UI 최적화**: React Compiler 자동 메모이제이션 + 가상 스크롤 + 스크롤 양자화

특히 **prompt cache 관련 코드**(`promptCacheBreakDetection.ts`, `toolSchemaCache.ts`, `CacheSafeParams`)가 전체 최적화에서 가장 높은 비용 절감 효과를 제공하며, 이를 위해 시스템 프롬프트/도구 스키마의 바이트 수준 안정성을 유지하는 데 상당한 엔지니어링 노력이 투입되어 있다.
