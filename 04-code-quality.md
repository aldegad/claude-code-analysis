# 04. 코드 품질 및 패턴 분석

> 분석 대상: claude-code (Anthropic Claude Code CLI)
> 규모: 1,892개 소스 파일, 515,894줄 TypeScript
> 분석일: 2026-03-31
> 분석자: 새미 (개발팀 비평가)

---

## 1. 기술 스택 평가

| 구성요소 | 기술 | 평가 |
|---------|------|------|
| 런타임 | Bun | JSX 네이티브 지원, `bun:bundle` feature flag으로 빌드타임 DCE(Dead Code Elimination) |
| 언어 | TypeScript strict mode | `strict: true`, `isolatedModules: true` 등 엄격한 설정 |
| 터미널 UI | React + Ink | CLI에 React를 쓴 독특한 선택 -- 컴포넌트 기반 TUI |
| CLI 파서 | Commander.js | `@commander-js/extra-typings`로 타입 안전한 CLI 파싱 |
| 린터/포매터 | Biome | ESLint 대비 빠름, 통합 린터+포매터 |
| 검증 | Zod v4 | 런타임 입력 검증에 일관 적용 |
| 분석/실험 | GrowthBook | 피처 플래그 + A/B 테스트 |

**핵심 평가**: 기술 스택 선정이 매우 의도적이다. Bun의 `feature()` 함수를 이용한 빌드타임 코드 제거는 52만줄 코드베이스에서 실제 배포 번들 크기를 크게 줄이는 핵심 전략이다.

---

## 2. 코딩 스타일 및 일관성

### 2.1 포매팅 규칙 (biome.json)

```json
{
  "indentStyle": "tab",
  "lineWidth": 100,
  "quoteStyle": "single",
  "semicolons": "asNeeded"
}
```

**일관성 평가**: 탭 인덴트, 100자 라인, 싱글쿼트, 세미콜론 선택적 -- 전반적으로 잘 지켜지고 있다. Biome의 자동 포매팅이 이를 보장한다.

### 2.2 네이밍 컨벤션

| 요소 | 컨벤션 | 준수도 |
|------|--------|--------|
| 파일명 | PascalCase (컴포넌트/도구) / kebab-case (명령어) | 높음 |
| 타입 | PascalCase + Props/State/Context 접미사 | 높음 |
| 훅 | `use` 접두사 | 높음 |
| 상수 | SCREAMING_SNAKE_CASE | 높음 |
| 분석 메타데이터 | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 특이하지만 의도적 |

**주목할 점**: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 같은 타입명은 코드/파일경로가 분석 메타데이터에 섞이는 것을 방지하려는 강제적 자기문서화(self-documenting) 패턴이다. 이 타입을 쓰려면 개발자가 "나는 이것이 코드나 파일경로가 아님을 확인했다"고 선언하는 셈이다. 센스 있다.

### 2.3 임포트 패턴

```typescript
// ES 모듈 + .js 확장자 (Bun 컨벤션)
import { foo } from '../utils/bar.js'

// 순환 의존성 해결을 위한 lazy require
const getModule = () => require('./heavy.js')

// feature flag 기반 조건부 임포트 -- 빌드타임 DCE
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

**배울 점**: feature flag을 빌드타임 최적화 도구로 쓰는 패턴이 매우 체계적이다. `feature('COORDINATOR_MODE')`, `feature('VOICE_MODE')`, `feature('BRIDGE_MODE')` 등 20개 이상의 플래그가 코드 전반에 걸쳐 사용된다. 이걸로 내부 전용 기능이 외부 빌드에서 완전히 제거된다.

**안티패턴**: `biome-ignore` 주석이 과도하게 사용된다. 특히 `cli.tsx`에서는 한 파일에 10개 이상의 `eslint-disable` + `biome-ignore` 주석이 있다. 이는 top-level side effect 규칙과의 충돌이 원인인데, 진입점 파일에서는 불가피하더라도 문서화가 부족하다.

---

## 3. 타입 안전성

### 3.1 설정

```json
// tsconfig.json
{
  "strict": true,
  "forceConsistentCasingInFileNames": true,
  "isolatedModules": true
}
```

```json
// biome.json
{
  "noExplicitAny": "off",    // any 허용
  "noNonNullAssertion": "off" // ! 연산자 허용
}
```

**비판**: `noExplicitAny: "off"`는 52만줄 코드베이스에서 상당한 타입 안전성 구멍을 만든다. 물론 빠른 프로토타이핑에는 도움이 되지만, MCP 프로토콜 핸들러나 보안 관련 코드에서 `any`가 사용되면 런타임 에러의 원인이 된다.

### 3.2 DeepImmutable 패턴

```typescript
// src/types/utils.ts에서 정의된 DeepImmutable을 광범위하게 적용
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  // ...
}>
```

**배울 점**: 상태 객체에 `DeepImmutable`을 적용하여 불변성을 타입 레벨에서 강제한다. React 상태 관리에서 실수로 직접 mutation하는 것을 컴파일 타임에 잡아낸다.

### 3.3 Zod를 이용한 런타임 검증

```typescript
// MCP 서버 설정 스키마 (src/services/mcp/types.ts)
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(),
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)
```

**배울 점**: `lazySchema()` 패턴으로 Zod 스키마를 지연 초기화한다. 52만줄에서 모든 스키마를 즉시 초기화하면 시작 시간이 늘어나므로, 필요할 때만 스키마를 생성하는 최적화다.

---

## 4. 에러 처리 패턴

### 4.1 graceful shutdown

```typescript
// src/entrypoints/init.ts
setupGracefulShutdown()
registerCleanup(shutdownLspServerManager)
registerCleanup(async () => {
  const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js')
  await cleanupSessionTeams()
})
```

**평가**: 클린업 레지스트리 패턴이 잘 구현되어 있다. `registerCleanup()`으로 프로세스 종료 시 정리할 리소스를 등록하고, `gracefulShutdownSync()`로 동기적 종료도 지원한다.

### 4.2 stale-while-error (macOS Keychain)

```typescript
// src/utils/secureStorage/macOsKeychainStorage.ts
if (prev.data !== null) {
  logForDebugging('[keychain] read failed; serving stale cache', {
    level: 'warn',
  })
  keychainCacheState.cache = { data: prev.data, cachedAt: Date.now() }
  return prev.data
}
```

**배울 점**: 키체인 읽기 실패 시 이전 캐시값을 반환하는 stale-while-error 전략. 일시적인 `security` 커맨드 실패가 "Not logged in"으로 나타나는 것을 방지한다. HTTP 캐싱의 `stale-while-revalidate`를 로컬 키체인에 적용한 영리한 패턴.

### 4.3 에러 로깅 분리

```
logForDebugging()     -- 디버그 전용 (--debug 플래그)
logForDiagnosticsNoPII()  -- 진단용 (PII 없음)
logError()            -- 일반 에러
logAntError()         -- Anthropic 내부 에러
```

**평가**: 에러 로깅이 4개 레벨로 분리되어 있다. 특히 `NoPII`(개인식별정보 없음) 접미사는 GDPR 등 프라이버시 규정 준수를 코드 레벨에서 강제하는 좋은 패턴이다.

---

## 5. 성능 최적화 패턴

### 5.1 시작 시간 최적화

```typescript
// src/main.tsx -- 첫 3줄이 핵심
import { profileCheckpoint } from './utils/startupProfiler.js'
profileCheckpoint('main_tsx_entry')
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js'
startMdmRawRead()  // MDM 서브프로세스 병렬 시작
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js'
startKeychainPrefetch()  // 키체인 읽기 병렬 시작
```

**배울 점**: 임포트 순서 자체를 최적화 도구로 사용한다. 무거운 모듈 로드(~135ms) 동안 MDM 읽기와 키체인 프리페치를 병렬로 시작한다. 이 패턴은 `biome-ignore-all assist/source/organizeImports` 주석으로 자동 정렬을 방지한다.

### 5.2 지연 로딩

```typescript
// OpenTelemetry 지연 로드 (~400KB + protobuf)
const { initializeTelemetry } = await import('../utils/telemetry/instrumentation.js')

// gRPC exporters 추가 지연 로드 (~700KB)
```

**평가**: 텔레메트리 모듈만 해도 1.1MB를 지연 로드한다. `--version` 같은 빠른 경로에서는 이 모듈이 아예 로드되지 않는다.

### 5.3 React Compiler

```typescript
// src/hooks/useCanUseTool.tsx
import { c as _c } from "react/compiler-runtime"
// ...
const $ = _c(3)
```

**배울 점**: React Compiler를 사용하여 메모이제이션을 자동화한다. `useMemo`/`useCallback` 수동 관리 없이 컴파일 타임에 최적화된 코드를 생성한다.

---

## 6. 코드 구조적 문제점

### 6.1 거대 파일 문제

| 파일 | 줄 수 | 문제 |
|------|-------|------|
| `cli/print.ts` | 5,596줄 | 헤드리스 모드 전체 로직 |
| `utils/messages.ts` | 5,513줄 | 메시지 관련 유틸리티 전부 |
| `utils/sessionStorage.ts` | 5,106줄 | 세션 저장소 전체 |
| `utils/hooks.ts` | 5,023줄 | 훅 시스템 전체 |
| `screens/REPL.tsx` | 5,006줄 | REPL 화면 전체 |
| `main.tsx` | 4,684줄 | CLI 진입점 |

**비판**: 5,000줄 이상 파일이 6개나 된다. `utils/messages.ts`는 메시지 생성, 정규화, 필터링, 변환을 모두 담당하고 있어 단일 책임 원칙(SRP)을 위반한다. 분리가 필요하다.

### 6.2 순환 의존성

```typescript
// src/tools.ts
// Lazy require to break circular dependency: tools.ts -> TeamCreateTool -> ... -> tools.ts
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

```typescript
// src/coordinator/coordinatorMode.ts
// Duplicated here because importing filesystem.ts creates a circular dependency
function isScratchpadGateEnabled(): boolean {
  return checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_scratch')
}
```

**비판**: `require()`를 이용한 순환 의존성 해결이 최소 20곳 이상에서 발견된다. 이는 모듈 간 결합도가 높다는 신호다. 코디네이터 모듈에서는 아예 함수를 복제해서 순환을 피하고 있는데, 이는 코드 동기화 문제를 일으킬 수 있다.

### 6.3 `process.env.USER_TYPE === 'ant'` 분기

```typescript
// src/tools.ts
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool : null

// src/commands.ts
const agentsPlatform = process.env.USER_TYPE === 'ant'
  ? require('./commands/agents-platform/index.js').default : null
```

**비판**: `USER_TYPE === 'ant'` 분기가 수십 곳에 흩어져 있다. 이는 Anthropic 내부용 기능과 외부 기능이 같은 코드베이스에 있다는 뜻인데, `feature()` 플래그와 `USER_TYPE` 체크가 혼재되어 있어 관리가 복잡해진다. 통합된 피처 게이팅 시스템이 필요하다.

---

## 7. 배울만한 패턴 종합

1. **빌드타임 DCE**: `bun:bundle`의 `feature()` 함수로 조건부 코드를 빌드 시 완전 제거
2. **자기문서화 타입명**: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`로 개발자 검증 강제
3. **stale-while-error**: 키체인 캐시에서 실패 시 이전값 반환
4. **임포트 순서 최적화**: 진입점 파일에서 임포트 순서를 성능 최적화 수단으로 활용
5. **`DeepImmutable`**: 상태 불변성을 타입 레벨에서 강제
6. **`lazySchema()`**: Zod 스키마 지연 초기화로 시작 시간 최적화
7. **`NoPII` 접미사**: 함수명으로 PII 비포함을 강제

## 8. 안티패턴 종합

1. **5,000줄 이상 파일 6개**: SRP 위반, 분리 필요
2. **순환 의존성 20곳 이상**: `require()` 우회가 아닌 구조적 리팩토링 필요
3. **`noExplicitAny: "off"`**: 타입 안전성 약화
4. **`USER_TYPE` + `feature()` 이중 게이팅**: 통합 필요
5. **`biome-ignore` 과다 사용**: 진입점 파일에서 억제 규칙이 코드 가독성 저하
