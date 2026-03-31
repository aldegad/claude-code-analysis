# 08. 구조적 결함 및 잠재적 버그 심층 분석

> 분석 대상: `claude-code/src/` (1,910개 파일, 약 515,894줄)
> 분석 일자: 2026-03-31
> 분석자: 새미 (개발팀 비평가)

---

## 요약

이 코드베이스는 **급속 성장의 대가를 치르고 있다.** 52만줄 규모에 순환 의존성 98개, `catch {}` 에러 삼킴 666곳, 타입 단언(as) 6,656곳, 비-null 단언(!) 310곳이 산재해 있다. 단일 `AppState` 객체가 100개 이상의 필드를 관리하면서 모든 UI 컴포넌트와 서비스가 이 거대한 상태 객체에 의존하는 구조다. 이 보고서는 실제 코드 위치와 함께 구체적 버그 시나리오를 제시한다.

---

## 1. 순환 의존성 [Critical]

### 1.1 탐지 결과

`madge --circular` 실행 결과 **98개의 순환 의존성 경로**가 발견되었다. 단순 변형을 제외해도 핵심 순환 고리는 다음과 같다.

### 1.2 핵심 순환 경로

#### [Critical] 경로 A: 유틸리티 삼각 순환
```
fsOperations.ts → slowOperations.ts → debug.ts → fsOperations.ts
```
- **파일 위치**:
  - `src/utils/fsOperations.ts:16` — `import { slowLogging } from './slowOperations.js'`
  - `src/utils/slowOperations.ts:12` — `import { logForDebugging } from './debug.js'`
  - `src/utils/debug.ts:14` — `import { getFsImplementation } from './fsOperations.js'`
- **위험**: 이 세 파일은 프로젝트 전역에서 사용되는 기초 유틸리티다. 모듈 로딩 순서에 따라 `getFsImplementation`이 `undefined`가 될 수 있다. Bun 번들러가 이를 처리하고 있지만, 핫 리로드나 테스트 환경에서 초기화 순서가 바뀌면 `TypeError: getFsImplementation is not a function` 발생 가능.

#### [Critical] 경로 B: 분석-인증-API 대순환
```
growthbook.ts → firstPartyEventLogger.ts → firstPartyEventLoggingExporter.ts
→ metadata.ts → auth.ts → mockRateLimits.ts → claudeAiLimits.ts
→ claude.ts → (client.ts → x402 → config.ts → modelOptions.ts → ...)
```
- **파일 위치**:
  - `src/services/analytics/growthbook.ts` → `firstPartyEventLogger.ts`
  - `src/services/analytics/metadata.ts` → `src/utils/auth.ts`
  - `src/utils/auth.ts` → `src/services/mockRateLimits.ts`
  - `src/services/claudeAiLimits.ts` → `src/services/api/claude.ts`
- **위험**: 분석 시스템 초기화가 API 클라이언트에 의존하고, API 클라이언트가 다시 분석 메타데이터에 의존한다. 이 순환은 최소 15개의 변형 경로를 만들어 낸다. 분석 시스템 초기화 실패 시 인증 모듈까지 연쇄 실패할 수 있다.

#### [Critical] 경로 C: 상태-훅-명령어 대순환
```
Tool.ts → commands.ts → commands/add-dir/ → validation.ts
→ filesystem.ts → growthbook.ts → ... → messages.ts
→ attachments.ts → Task.ts → AppState.tsx
→ useSettingsChange.ts → changeDetector.ts → hooks.ts
→ execAgentHook.ts → query.ts → useCanUseTool.tsx
→ PermissionRequest.tsx
```
- **파일 위치**: `src/Tool.ts:11` → `src/commands.ts:2-204` (55개 이상 명령어 import) → 깊은 체인
- **위험**: 이 경로는 무려 20단계 이상의 모듈을 거친다. `Tool.ts`가 `commands.ts`에 타입 의존하고, `commands.ts`가 55개 이상의 명령어 모듈을 직접 import한다. 명령어 모듈들은 다시 `Tool.ts`의 타입을 참조한다. 번들러가 해결해주는 상황이지만, 하나의 명령어 수정이 전체 빌드 캐시를 무효화한다.

#### [High] 경로 D: 모델 설정 순환
```
config.ts → modelOptions.ts → context.ts → model.ts
→ modelAllowlist.ts → modelStrings.ts → configs.ts (→ config.ts)
```
- **파일 위치**:
  - `src/utils/config.ts` → `src/utils/model/modelOptions.ts`
  - `src/utils/model/model.ts` → `src/utils/modelCost.ts` → `src/utils/fastMode.ts` (→ config.ts)
- **위험**: 모델 설정/비용 계산 체인이 순환한다. 새 모델 추가 시 예기치 않은 초기화 순서 문제 발생 가능.

#### [High] 경로 E: ink 렌더링 내부 순환
```
ink/dom.ts ↔ ink/focus.ts
ink/dom.ts ↔ ink/node-cache.ts
ink/dom.ts ↔ ink/squash-text-nodes.ts
```
- **파일 위치**: `src/ink/dom.ts` (3곳 순환)
- **위험**: 터미널 UI 렌더링 엔진 내부의 순환. 렌더링 중 DOM 노드 캐시 접근 시점에 따라 stale 노드 참조 가능.

#### [Medium] 경로 F: Git 유틸리티 순환
```
git.ts → detectRepository.ts (→ git.ts)
git.ts → git/gitFilesystem.ts (→ git.ts)
file.ts → fileReadCache.ts (→ file.ts)
```

### 1.3 `require()` 순환 회피 — 설계 부채 증거

```typescript
// src/state/AppStateStore.ts:459-461
const teammateUtils =
    require('../utils/teammate.js') as typeof import('../utils/teammate.js')
```

`getDefaultAppState()`에서 런타임 `require()`를 사용해 순환 의존성을 회피하고 있다. 이 패턴은 순환 문제를 해결하지 않고 은폐한다. 향후 `teammate.ts`의 export 구조가 변경되면 `as typeof import(...)` 타입 단언이 실제 런타임 타입과 불일치할 수 있다.

---

## 2. 타입 안전성 구멍 [Critical]

### 2.1 린트 설정 자체의 구멍

```json
// biome.json
"noExplicitAny": "off",
"noNonNullAssertion": "off"
```

**52만줄 코드베이스에서 이 두 규칙을 모두 끈 것은 타입 시스템의 가치를 상당 부분 훼손한다.**

### 2.2 수치 요약

| 패턴 | 발견 수 | 파일 수 |
|------|---------|---------|
| `as` 타입 단언 | 6,656 | 1,332 |
| `as AnalyticsMetadata_I_VERIFIED_...` | 976 | 166 |
| `as any` | 86+ | 48 |
| 비-null 단언 (`!`) | 310 | 145 |
| `catch {}` (bare catch, 에러 변수 없음) | 666 | 270 |
| `JSON.parse() as T` (검증 없이) | 11+ | 8 |

### 2.3 `as AnalyticsMetadata` 남용 [High]

- **위치**: `src/services/tools/toolExecution.ts` (50곳), `src/services/api/logging.ts` (45곳), `src/services/api/claude.ts` (68곳)
- `toolExecution.ts`에서만 50곳에서 `as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 타입 단언을 사용한다. 이 패턴의 의도는 "개인정보가 아님을 개발자가 확인했다"는 표시인데, 실질적으로는 **어떤 문자열이든 분석 이벤트에 넣을 수 있는 escape hatch**가 되었다. 976번 사용되었다는 것은 이 검증이 형해화(形骸化)되었음을 의미한다.

### 2.4 `JSON.parse()` 후 타입 검증 없이 사용 [High]

```typescript
// src/utils/sessionStorage.ts:298
return JSON.parse(raw) as AgentMetadata

// src/utils/sessionStorage.ts:352
return JSON.parse(raw) as RemoteAgentMetadata

// src/utils/sessionStorage.ts:389
results.push(JSON.parse(raw) as RemoteAgentMetadata)

// src/services/x402/client.ts:40
const parsed = JSON.parse(headerValue) as PaymentRequirement

// src/services/analytics/growthbook.ts:177
envOverrides = JSON.parse(raw) as Record<string, unknown>
```

- **위험**: 파일이 손상되거나 외부 입력이 예상 스키마와 다를 경우 런타임에서 `undefined` 필드 접근 에러 발생. 특히 `sessionStorage.ts`는 세션 복구 경로에 있으므로, 손상된 세션 파일이 전체 앱 크래시를 유발할 수 있다.
- **양호 사례**: `src/utils/hooks.ts:4416`에서는 `hookJSONOutputSchema().parse(JSON.parse(trimmed))`로 Zod 스키마 검증을 수행한다. 이 패턴이 전체적으로 적용되어야 한다.

### 2.5 `as any` 사용 [High]

```typescript
// src/main.tsx:256
const inspector = (global as any).require('inspector');
```

```typescript
// src/types/generated/ (자동 생성 파일들)
return PublicApiAuth.fromPartial(base ?? ({} as any))  // 5곳 이상
```

자동 생성 코드의 `as any`는 생성기 문제이지만, `main.tsx`에서의 `global as any`는 Bun 런타임 특정 API 접근을 위한 것으로, 런타임 환경이 달라지면 즉시 깨진다.

---

## 3. God Object / 거대 파일 [High]

### 3.1 5,000줄 이상 파일 (상위 5개)

| 파일 | 줄 수 | catch 블록 | 책임 |
|------|-------|-----------|------|
| `src/cli/print.ts` | 5,596 | 32 | 모든 CLI 출력 포맷팅 + SDK 통신 + 세션 타이틀 생성 + 제어 요청 처리 |
| `src/utils/messages.ts` | 5,513 | 4 | 모든 메시지 타입의 생성/변환/필터링/직렬화 |
| `src/utils/sessionStorage.ts` | 5,106 | 33 | 세션 저장/로드/마이그레이션/메타데이터/잠금 |
| `src/utils/hooks.ts` | 5,023 | 20 | 모든 사용자 정의 훅(pre/post-query, tool, permission, stop) |
| `src/screens/REPL.tsx` | 5,006 | 9 | 전체 REPL UI 로직 (입력/출력/상태/키바인딩/세션관리) |

### 3.2 `REPL.tsx` 단일 책임 원칙 위반 [Critical]

`src/screens/REPL.tsx`는 단일 React 컴포넌트가 5,006줄이다. 상위 80줄의 import만 보아도:

- 80개 이상의 import 문
- 다수의 `setInterval`/`setTimeout` 호출
- 상당수는 cleanup이 존재하지만, 일부 타이머 패턴은 cleanup 유무를 추가로 점검할 필요가 있음
- 세션 관리, 키바인딩, 권한 처리, 메시지 렌더링, 비용 추적, 알림, 테마, AI 쿼리 트리거링 등을 단일 컴포넌트에서 처리

**줄 3136**: `setTimeout(ref => { ref.current = false; }, 100, initialMessageRef)` -- 100ms 딜레이로 ref를 리셋하는 패턴은 경쟁 조건의 전형적 징후.

### 3.3 `print.ts` 역할 과부하 [High]

`src/cli/print.ts`는 5,596줄로 다음을 모두 담당한다:
- CLI 대화형 출력 포맷팅
- SDK/구조화 IO 프로토콜 메시지 처리
- 원격 세션 제어 요청(generate_session_title, side_question 등)
- 프롬프트 제안(suggestion) 상태 머신
- 에러 결과 직렬화

줄 2424-2444에서는 에러 핸들러 안에서 또 다른 try-catch로 구조화된 에러 결과를 전송하려 시도한다. 이중 catch 패턴은 에러 전파 체인이 복잡하게 꼬여 있음을 보여준다.

### 3.4 `commands.ts` 중앙집중 등록 [Medium]

`src/commands.ts`에서 55개 이상의 명령어를 개별 import로 등록한다(줄 2-204). 이 파일이 변경되면 모든 명령어 모듈이 영향을 받으며, 순환 의존성의 허브가 된다.

---

## 4. 상태 관리 문제 [Critical]

### 4.1 AppState 비대 [Critical]

`src/state/AppStateStore.ts`에서 정의된 `AppState` 타입은 **120개 이상의 필드**를 가진다 (줄 89-452, 363줄에 걸친 단일 타입 정의).

주요 필드 카테고리:
- **기본 설정** (15개): settings, verbose, mainLoopModel, isBriefOnly 등
- **원격 브릿지** (15개): replBridgeEnabled, replBridgeConnected, replBridgeSessionActive, replBridgeReconnecting, replBridgeConnectUrl, replBridgeSessionUrl, replBridgeEnvironmentId, replBridgeSessionId, replBridgeError 등
- **MCP** (5개 + 중첩): mcp.clients, mcp.tools, mcp.commands, mcp.resources, mcp.pluginReconnectKey
- **플러그인** (5개 + 중첩): plugins.enabled, plugins.disabled, plugins.commands, plugins.errors, plugins.installationStatus, plugins.needsRefresh
- **팀/스웜** (15개+): teamContext, agentNameRegistry, foregroundedTaskId, viewingAgentTaskId, inbox, workerSandboxPermissions, pendingWorkerRequest, pendingSandboxRequest
- **추측 실행** (10개+): speculation, speculationSessionTimeSavedMs, promptSuggestion.*
- **컴퓨터 사용** (10개+): computerUseMcpState.*
- **기타** (30개+): tasks, todos, notifications, elicitation, sessionHooks, tungstenActiveSession, bagelActive, ultraplanLaunching 등

### 4.2 상태 경쟁 조건 (Race Condition) [High]

```typescript
// src/state/store.ts
setState: (updater: (prev: T) => T) => {
    const prev = state
    const next = updater(prev)
    if (Object.is(next, prev)) return
    state = next
    onChange?.({ newState: next, oldState: prev })
    for (const listener of listeners) listener()
},
```

이 store 구현은 **동기적(synchronous) updater 함수**를 사용하지만, 여러 비동기 작업(MCP 연결, 브릿지 폴링, 원격 세션, 프롬프트 제안 등)이 동시에 `setState`를 호출한다. 각 `setState` 호출은 이전 상태의 스냅샷을 기반으로 업데이트하므로, 두 비동기 작업이 거의 동시에 `setState`를 호출하면 **먼저 적용된 업데이트가 유실**될 수 있다.

특히 위험한 시나리오:
- `src/services/mcp/useManageMCPConnections.ts`의 다수의 `setTimeout`과 `src/screens/REPL.tsx`의 다수의 `setTimeout`이 동시에 상태를 업데이트할 때
- `src/state/AppStateStore.ts:449` 주석에서 직접 인정: "Races against local UI + bridge + hooks + classifier via claim()"

### 4.3 메모리 누수 패턴 [High]

#### 타이머 불일치
`src/screens/REPL.tsx`에는 `setTimeout`/`setInterval`이 여러 곳에서 사용되며, 일부는 cleanup이 명시적이지만 일부 패턴은 cleanup이 코드상 분명하지 않다. 따라서 타이머 관련 정리 누수 가능성은 검토 대상이지만, 정확한 개수는 추가 확인이 필요하다.

특히:
- 줄 430: `setTimeout(() => alive && setIndexStatus(null), 2000)` — alive 플래그 의존이지만, 컴포넌트 언마운트 시 타이머 취소 불명확
- 줄 3136: `setTimeout(ref => { ref.current = false; }, 100, initialMessageRef)` — ref 기반 타이머, cleanup 없음

#### agentNameRegistry (Map) 무한 성장
`AppStateStore.ts:471`: `agentNameRegistry: new Map()`. 에이전트가 생성될 때마다 이 Map에 추가되지만, 에이전트 종료 시 제거 로직이 AppState 레벨에서 보이지 않는다. 장시간 세션에서 메모리 누수 가능.

#### 이벤트 리스너 누적
`src/bridge/bridgeMain.ts`에서 `addEventListener` 사용 후 `process.off()`로 정리하는 패턴(줄 2659-2660, 2755-2757)은 양호하지만, `src/services/analytics/growthbook.ts`도 `resetGrowthBook()`에서 `process.off('beforeExit', ...)`와 `process.off('exit', ...)`로 핸들러를 명시적으로 제거한다.

---

## 5. 에러 처리 [High]

### 5.1 Bare catch (에러 삼킴) [Critical]

전체 코드베이스에서 **666곳의 `catch {`** (에러 변수 없이 catch)가 발견되었다. 이는 전체 catch 블록(1,426개)의 **46.7%**에 해당한다.

주요 문제 지점:

#### 분석/텔레메트리 에러 삼킴
```typescript
// src/services/analytics/growthbook.ts:446
} catch {
    return undefined    // URL 파싱 실패를 무시
}

// src/services/analytics/firstPartyEventLogger.ts:125, 430
} catch { ... }         // 이벤트 로깅 실패를 무시

// src/services/analytics/firstPartyEventLoggingExporter.ts:160, 179, 213
} catch { ... }         // 이벤트 내보내기 실패를 무시
```

분석 시스템의 에러를 삼키는 것은 의도적일 수 있으나, **분석 자체의 장애 진단이 불가능**해진다.

#### 보안 관련 에러 삼킴
```typescript
// src/services/policyLimits/index.ts (6곳: 줄 186, 243, 402, 487, 605, 627)
} catch { ... }
```

정책 제한(policy limits) 확인 중 에러를 삼키면, **보안 정책 우회가 조용히 발생**할 수 있다.

#### 인증/OAuth 에러 삼킴
```typescript
// src/services/mcp/auth.ts (5곳: 줄 121, 167, 925, 1093, 2170)
} catch { ... }
```

OAuth 인증 흐름에서 에러를 삼키면 사용자가 "왜 인증이 안 되는지" 진단할 수 없다.

### 5.2 `.catch(() => {})` Promise 에러 삼킴 [High]

```typescript
// src/main.tsx (7곳)
commandsPromise?.catch(() => {});   // 줄 1932
agentDefsPromise?.catch(() => {});  // 줄 1933
mcpPromise.catch(() => {});         // 줄 2452
sessionStartHooksPromise?.catch(() => {}); // 줄 2611
void clearServerCache(c.name, c.config).catch(() => {}); // 줄 2761

// src/services/mcp/client.ts (4곳)
transport.close().catch(() => {})   // 줄 1059
inProcessServer.close().catch(() => {})  // 줄 1057, 1148
```

이 패턴들은 "fire-and-forget"을 의도했지만, MCP 서버 연결/해제 실패, 명령어 로딩 실패, 세션 시작 훅 실패 등 **중요한 초기화 에러를 침묵시킨다.**

### 5.3 에러 전파 체인 끊김 [Medium]

```typescript
// src/cli/print.ts:2424-2444
} catch (error) {
    try {
        await structuredIO.write({
            type: 'result',
            subtype: 'error_during_execution',
            ...
            errors: [errorMessage(error), ...getInMemoryErrors().map(_ => _.error)],
        })
    } catch (_writeError) {
        // structuredIO 쓰기 자체가 실패하면?
        // → 여기서 에러가 삼켜진다
    }
}
```

에러 보고 메커니즘 자체의 에러가 처리되지 않는 구조.

### 5.4 catch-return 패턴 (에러를 정상 값으로 변환) [Medium]

```typescript
// src/tools/BashTool/sedValidation.ts:376-377
} catch (_error) {
    return true // "파싱 실패하면 위험하다고 가정" — 의도적이지만 false positive 유발
}

// src/utils/git.ts:458-459
} catch (_) {
    return false  // git 명령어 실패를 "아니요"로 변환
}

// src/services/tips/tipRegistry.ts:153-154
} catch (_) {
    return false  // 팁 조건 확인 실패를 "표시하지 않음"으로 변환
}
```

---

## 6. 실제 버그 추론 [Critical]

위 구조적 결함들을 종합하면, 다음과 같은 버그가 실제로 발생했거나 발생 중일 가능성이 높다:

### 버그 시나리오 1: 세션 복구 실패 크래시 [Critical]

**경로**: `sessionStorage.ts:298` → `JSON.parse(raw) as AgentMetadata`

- 사용자가 비정상 종료 후 세션을 복구(`--resume`)하려 할 때
- 디스크의 세션 메타데이터 파일이 부분적으로 기록되어 유효하지 않은 JSON이 되었다면
- `JSON.parse()`가 throw하고, `catch` 블록이 `isFsInaccessible(e)` 체크만 하므로 `SyntaxError`는 그대로 상위로 전파
- 결과: 세션 복구가 전혀 불가능해지고, 사용자는 해당 세션의 모든 컨텍스트를 잃는다
- **영향 범위**: 모든 사용자, 특히 장시간 세션 사용자

### 버그 시나리오 2: MCP 서버 인증 무한 루프 [High]

**경로**: `mcp/auth.ts` 내 5개의 bare catch + `mcp/client.ts`의 세션 만료 재시도 로직

- MCP 서버의 OAuth 토큰이 만료되면 `handleOAuth401Error()` 호출 (client.ts:402)
- 이 함수가 `.catch(() => false)`로 에러를 삼키고 false 반환
- 재시도 루프가 MAX_SESSION_RETRIES까지 반복하지만, 실제 토큰 갱신은 실패 중
- auth.ts의 bare catch(줄 121, 167)가 토큰 갱신 에러를 삼키면, 사용자에게 "연결이 끊겼다"는 모호한 에러만 표시
- 결과: 사용자가 MCP 도구를 사용하려 할 때마다 타임아웃 대기 후 실패
- **영향 범위**: OAuth 기반 MCP 서버 사용자

### 버그 시나리오 3: 상태 업데이트 유실로 인한 UI 불일치 [High]

**경로**: `store.ts` 동기 setState + 다중 비동기 업데이트

- 프롬프트 제안(speculation)이 활성화된 상태에서 MCP 서버 재연결이 발생
- `useManageMCPConnections.ts`가 `setState`로 `mcp.clients` 업데이트 시도
- 거의 동시에 `usePromptSuggestion.ts`가 `setState`로 `promptSuggestion` 업데이트
- 두 setState 호출이 같은 `prev` 스냅샷을 기반으로 하면, 먼저 적용된 MCP 업데이트가 유실
- 결과: UI에서 MCP 서버가 "연결됨"으로 표시되지만 실제로는 도구가 비활성화, 또는 그 반대
- **영향 범위**: MCP 서버와 프롬프트 제안을 동시 사용하는 모든 사용자

### 버그 시나리오 4: 정책 제한 우회 [Critical]

**경로**: `policyLimits/index.ts`의 6개 bare catch 블록

- 원격 관리 설정(managed settings)에서 정책 제한을 가져오는 중 네트워크 에러 발생
- `catch {}` 블록이 에러를 삼기고 기본값(제한 없음)으로 폴백
- 결과: 관리자가 설정한 토큰 사용량 제한, 도구 접근 제한 등이 적용되지 않음
- 네트워크 불안정한 환경에서 지속적으로 재현 가능
- **영향 범위**: 기업 환경의 관리형 배포

### 버그 시나리오 5: 분석 초기화 실패로 인한 Feature Flag 기본값 사용 [High]

**경로**: GrowthBook → firstPartyEventLogger → metadata → auth 순환 의존성

- 앱 시작 시 GrowthBook 초기화가 auth 모듈에 의존
- auth 모듈 초기화가 API 클라이언트에 의존
- API 클라이언트 설정이 다시 GrowthBook feature flag에 의존
- 이 순환이 한 지점에서 깨지면, 모든 feature flag가 기본값(대부분 false)으로 폴백
- 결과: 프롬프트 제안, 자동 메모리, 팁 표시 등 GrowthBook으로 제어되는 모든 기능이 비활성화
- 앱은 정상적으로 보이지만, 고급 기능이 조용히 꺼져 있는 상태
- **영향 범위**: 네트워크 초기화 순서가 불안정한 환경

### 버그 시나리오 6: 장시간 세션 메모리 누수 [High]

**경로**: REPL.tsx 일부 타이머 패턴 + agentNameRegistry 무한 성장

- 사용자가 수 시간 동안 세션을 유지하면서 여러 에이전트(Agent 도구)를 반복 실행
- 각 에이전트 실행마다 `agentNameRegistry` Map에 항목 추가, 제거 없음
- REPL.tsx에는 cleanup이 명시적이지 않은 타이머 패턴이 있어 클로저를 통해 이전 상태 객체를 붙잡을 가능성을 점검할 필요가 있음
- 결과: 점진적 메모리 사용량 증가, 수 시간 후 성능 저하 가능성
- **영향 범위**: 장시간 세션 사용자 (headless 모드, CI/CD 환경)

### 버그 시나리오 7: x402 결제 헤더 파싱 실패로 인한 예기치 않은 동작 [Medium]

**경로**: `x402/client.ts:40` — `JSON.parse(headerValue) as PaymentRequirement`

- x402 서버가 비표준 형식의 결제 헤더를 반환하면
- `JSON.parse()`는 성공하지만 필수 필드(`scheme`, `network` 등)가 없을 수 있음
- 줄 41-43에서 필드 존재 여부를 확인하지만, 필드가 존재하되 **잘못된 타입**(예: `maxAmountRequired`가 문자열이 아닌 숫자)일 경우 `as PaymentRequirement` 단언으로 인해 타입 체커가 잡지 못함
- 결과: 의도하지 않은 결제 금액 계산 또는 런타임 에러

---

## 7. 리팩토링 우선순위 제안

### 즉시 조치 (Critical)

1. **`policyLimits/index.ts`의 bare catch 수정** — 보안 정책 우회 위험 제거
2. **`sessionStorage.ts`의 JSON.parse에 스키마 검증 추가** — 세션 복구 크래시 방지
3. **`REPL.tsx` 분할** — 5,006줄 컴포넌트를 최소 5개 이상 하위 컴포넌트로 분리
4. **AppState 분리** — 120+ 필드를 도메인별 슬라이스로 분할 (bridge, mcp, speculation, team 등)

### 단기 조치 (High)

5. **유틸리티 삼각 순환 해소** — `fsOperations ↔ slowOperations ↔ debug` 의존성 방향 정리
6. **`commands.ts` 동적 등록으로 전환** — 55개 정적 import를 레지스트리 패턴으로 변경
7. **`biome.json`에서 `noExplicitAny: "warn"` 활성화** — 점진적 타입 안전성 강화
8. **store.ts에 배치 업데이트 메커니즘 추가** — 동시 setState 호출의 경쟁 조건 해소

### 중기 조치 (Medium)

9. **`print.ts`, `messages.ts` 분할** — 책임별 모듈 분리
10. **`as AnalyticsMetadata` 패턴 대체** — 브랜드 타입 + 빌더 함수로 변환
11. **`.catch(() => {})` 패턴 전수 조사** — 최소 로깅이라도 추가

---

## 8. 부록: 도구 및 방법론

- **순환 의존성**: `npx madge --circular --extensions ts,tsx src/` (1,910 파일 처리, 726 경고)
- **타입 단언**: `ripgrep` 패턴 `\bas\s+\w+` (*.ts, *.tsx)
- **에러 처리**: `catch\s*\{` (bare catch) 및 `catch\s*\(\w*\)\s*\{\s*//` (주석만 있는 catch) 패턴
- **파일 크기**: `wc -l` 전수 검사
- **타이머 균형**: `setInterval`/`setTimeout` vs `clearInterval`/`clearTimeout` 쌍 대조

---

*"코드가 동작한다"는 것은 "코드가 올바르다"를 의미하지 않는다. 이 코드베이스는 동작하지만, 구조적 부채가 매 릴리스마다 복리로 쌓이고 있다.*
