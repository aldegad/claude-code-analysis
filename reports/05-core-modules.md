# 05. 핵심 모듈 상세 분석

> 분석 대상: claude-code (Anthropic Claude Code CLI)
> 분석일: 2026-03-31
> 분석자: 새미 (개발팀 비평가)

---

## 1. 전체 아키텍처 개요

```
사용자 입력
    |
    v
[cli.tsx] -- 진입점, fast-path 분기
    |
    v
[main.tsx] -- Commander.js CLI 파서, 4,684줄
    |
    +---> [init.ts] -- 설정, 텔레메트리, OAuth, 프록시 초기화
    |
    v
[replLauncher.tsx] -- REPL 세션 시작
    |
    v
[REPL.tsx] -- 5,006줄, React+Ink 터미널 UI
    |
    v
[QueryEngine.ts] -- LLM API 호출 + 도구 실행 루프
    |
    +---> [query.ts] -- 스트리밍, 도구 실행, 자동 컴팩션
    |       |
    |       +---> [claude.ts] -- Anthropic SDK 클라이언트
    |       |
    |       +---> [StreamingToolExecutor.ts] -- 동시성 제어
    |       |
    |       +---> [toolOrchestration.ts] -- 직렬/병렬 도구 실행
    |
    +---> [Tool.ts] -- 도구 타입 정의, buildTool 팩토리
    |
    +---> [commands.ts] -- 명령어 레지스트리
```

---

## 2. 진입점 시스템 (src/entrypoints/)

### 2.1 cli.tsx -- 최상위 부트스트랩

**역할**: 프로세스 시작 후 가장 먼저 실행되는 파일. 빠른 경로(fast-path) 분기를 처리한다.

**핵심 설계 원칙**: "빠른 명령은 빠르게 끝내자"

| fast-path | 로드하는 모듈 | 용도 |
|-----------|------------|------|
| `--version` | 없음 (MACRO.VERSION 인라인) | 버전 출력 |
| `--dump-system-prompt` | config, model, prompts | 시스템 프롬프트 덤프 |
| `--daemon-worker` | daemon/workerRegistry | 데몬 워커 |
| `remote-control` | bridge/* | 브릿지 모드 |
| `daemon` | daemon/main | 데몬 모드 |
| `ps/logs/attach/kill` | cli/bg | 백그라운드 세션 관리 |

```typescript
// --version은 모듈 로드 0개로 응답
if (args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`)
  return
}
```

**비평**: `cli.tsx`는 400줄이 넘고 분기가 10개 이상이다. 각 분기마다 `await import()`를 반복하는데, 라우터 패턴으로 분리하면 가독성이 좋아질 것이다. 다만 현재 구조가 성능 관점에서는 최적이다 -- 각 분기가 필요한 모듈만 정확히 로드한다.

### 2.2 init.ts -- 초기화 오케스트레이터

**역할**: 설정 활성화, 환경변수 적용, 텔레메트리, OAuth, mTLS, 프록시 등 초기화.

**초기화 순서**:
1. `enableConfigs()` -- 설정 시스템 활성화
2. `applySafeConfigEnvironmentVariables()` -- 신뢰 확인 전 안전한 환경변수만 적용
3. `applyExtraCACertsFromConfig()` -- TLS 인증서 (Bun이 부팅 시 캐시하므로 일찍 실행)
4. `setupGracefulShutdown()` -- 종료 핸들러
5. 비동기 병렬: 1P 이벤트 로깅, OAuth, JetBrains 감지, Git 리포 감지
6. `configureGlobalMTLS()` + `configureGlobalAgents()` -- 네트워크 설정
7. `preconnectAnthropicApi()` -- API 사전 연결 (TCP+TLS 워밍)

**배울 점**: `preconnectAnthropicApi()`는 네트워크 설정 완료 직후 API에 사전 연결한다. 이후 실제 API 요청 시 TCP+TLS 핸드셰이크(100-200ms)를 건너뛸 수 있다.

**주의할 점**: `memoize()`로 init을 감싸서 중복 호출을 방지하지만, `ConfigParseError` 발생 시 대화형 다이얼로그를 표시하는 fallback이 있다. 비대화형 모드에서는 stderr로 오류를 출력하고 종료한다.

### 2.3 mcp.ts -- MCP 서버 진입점

**역할**: Claude Code 자체를 MCP 서버로 노출. 도구를 MCP 프로토콜로 제공한다.

```typescript
const server = new Server(
  { name: 'claude/tengu', version: MACRO.VERSION },
  { capabilities: { tools: {} } },
)
```

**특이점**: `zodToJsonSchema()`로 Zod 스키마를 JSON Schema로 변환해서 MCP 도구 스키마로 제공한다. `outputSchema`가 `z.union`이나 `z.discriminatedUnion`인 경우 root 레벨에 `type: "object"`가 없으므로 스킵한다 (MCP SDK 제약).

---

## 3. QueryEngine -- 핵심 엔진

**위치**: `src/QueryEngine.ts`
**역할**: LLM API 호출, 도구 실행 루프, 세션 상태 관리의 핵심

### 3.1 설계

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]     // 대화 이력
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage   // 토큰 사용량
  private readFileState: FileStateCache  // 파일 상태 LRU 캐시
  private discoveredSkillNames = new Set<string>()  // 턴별 스킬 추적
}
```

**핵심 메서드**: `submitMessage()` -- AsyncGenerator로 SDK 메시지를 스트리밍 yield

```typescript
async *submitMessage(prompt, options): AsyncGenerator<SDKMessage, void, unknown> {
  // 1. 시스템 프롬프트 구성
  // 2. 유저 컨텍스트 수집 (git 상태, CLAUDE.md)
  // 3. processUserInput으로 슬래시 명령 처리
  // 4. query()로 API 호출 + 도구 실행 루프
  // 5. 세션 저장소에 트랜스크립트 기록
}
```

### 3.2 캐시 전략

```typescript
// SDK/headless 모드: 파일 상태 캐시에 크기 제한 (LRU)
const READ_FILE_STATE_CACHE_SIZE = 100  // MCP 모드
// REPL 모드: 크기 제한 없이 운영
```

**비평**: MCP 모드와 REPL 모드에서 캐시 전략이 다르다. MCP는 100개 파일 + 25MB 제한이지만 REPL은 무제한이다. 장시간 세션에서 메모리 누수 가능성이 있다.

---

## 4. 도구 시스템 (src/tools/)

### 4.1 buildTool 팩토리

```typescript
export const FileEditTool = buildTool({
  name: FILE_EDIT_TOOL_NAME,
  inputSchema: inputSchema(),
  outputSchema: outputSchema(),
  
  // 핵심 메서드들
  async call(args, context, canUseTool, parentMessage, onProgress) { ... },
  async checkPermissions(input, context) { ... },
  isConcurrencySafe(input) { ... },   // 병렬 실행 가능 여부
  isReadOnly(input) { ... },          // 읽기 전용 여부
  prompt(options) { ... },            // 시스템 프롬프트 주입
  
  // UI 렌더링
  renderToolUseMessage(input, options) { ... },
  renderToolResultMessage(content, progressMessages, options) { ... },
})
```

### 4.2 도구 동시성 제어

`StreamingToolExecutor`가 핵심 -- 스트리밍 중 도구를 실시간으로 실행:

```
도구 스트림 수신
    |
    +---> isConcurrencySafe? --YES--> 병렬 큐에 추가, 즉시 실행
    |
    +---> NO --> 배타적 실행 (다른 도구 완료 대기)
```

```typescript
// toolOrchestration.ts
function* partitionToolCalls(toolUseMessages, context) {
  // 연속된 동시성 안전 도구는 하나의 배치로 묶음
  // 비동시성 안전 도구는 개별 배치로 분리
}
```

**배울 점**: `partitionToolCalls`가 연속된 읽기 전용 도구를 배치로 묶어 병렬 실행하고, 쓰기 도구는 직렬로 실행한다. `MAX_TOOL_USE_CONCURRENCY`(기본 10)로 병렬도를 제한한다.

### 4.3 도구 레지스트리 (src/tools.ts)

40개 이상의 도구가 등록되어 있다:

| 범주 | 도구들 |
|------|-------|
| 파일 I/O | FileReadTool, FileEditTool, FileWriteTool, GlobTool, GrepTool |
| 실행 | BashTool, PowerShellTool, REPLTool |
| 에이전트 | AgentTool, SkillTool, TaskCreateTool/GetTool/UpdateTool/StopTool |
| 네트워크 | WebFetchTool, WebSearchTool |
| 컨텍스트 | NotebookEditTool, LSPTool, ToolSearchTool |
| 스웜 | TeamCreateTool, TeamDeleteTool, SendMessageTool |
| 기획 | EnterPlanModeTool, ExitPlanModeTool, TodoWriteTool |
| 환경 | EnterWorktreeTool, ExitWorktreeTool, ConfigTool |
| MCP | ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool |

**피처 게이트된 도구들**:
- `PROACTIVE`/`KAIROS`: SleepTool, SendUserFileTool, PushNotificationTool
- `COORDINATOR_MODE`: 코디네이터 모드 전용 도구
- `WORKFLOW_SCRIPTS`: WorkflowTool
- `HISTORY_SNIP`: SnipTool
- `OVERFLOW_TEST_TOOL`: OverflowTestTool (테스트 전용)
- `WEB_BROWSER_TOOL`: WebBrowserTool


### 4.4 NotebookEditTool 상세

**위치**: `src/tools/NotebookEditTool/NotebookEditTool.ts`

레지스트리(4.3)에서 "컨텍스트" 범주로 분류된 `NotebookEditTool`은 실제 구현이 존재하는 편집 전용 도구다.

**편집 모드** (3가지):
- `replace` -- 기존 셀 소스를 교체
- `insert` -- 지정 위치에 새 셀 삽입
- `delete` -- 셀 삭제

**입력 스키마**:
- `notebook_path`: 노트북 파일 경로
- `cell_id`: 대상 셀 식별자
- `new_source`: 새 셀 소스 코드
- `cell_type`: 셀 타입 (code/markdown)
- `edit_mode`: 편집 모드 (replace/insert/delete)

**핵심 동작**:
- **셀 실행 기능 없음** -- 순수 편집 전용. 실행은 외부에서 수행해야 한다
- **Read-before-Edit 강제** -- 편집 전 파일 상태를 반드시 읽고, stale read를 방지한다
- **execution_count/outputs 초기화** -- code cell 수정 시 `execution_count`와 `outputs`를 리셋하여 오래된 실행 결과가 남지 않게 한다
- **cell_id 우선 식별** -- `cell_id`로 셀을 식별하되, 없으면 `cell-N` 인덱스로 fallback
- **UNC 경로 보안** -- UNC 경로에 대한 NTLM credential leak 방지 처리가 포함되어 있다

**비평**: 프롬프트 문구에서는 `cell_number`로 셀을 설명하지만, 실제 구현은 `cell_id` 중심으로 동작한다. 약한 불일치가 존재하며, LLM이 혼동할 여지가 있다.

### 4.5 Claude in Chrome (브라우저 제어)

**소스**: `src/utils/claudeInChrome/`, `src/skills/bundled/claudeInChrome.ts`, `src/commands/chrome/`

레지스트리(4.3)에 `WEB_BROWSER_TOOL`이 피처 게이트된 도구로 등록되어 있지만, 실제 1차 브라우저 구현은 `claudeInChrome`이다. `WebBrowserTool`은 레지스트리에 흔적만 있고 구현 파일이 부재하다.

**아키텍처**:
- **MCP 서버 기반** -- CDP(Chrome DevTools Protocol)에 직접 연결하지 않고, MCP 서버를 통해 크롬을 조작한다
- **도구 네이밍**: `mcp__claude-in-chrome__*` 패턴으로 tabs_context, 클릭, 입력, 스크린샷, 콘솔 읽기 등의 도구를 제공

**연결 구조**:
- **이중 연결**: bridge URL(`wss://bridge.claudeusercontent.com`)과 native socket을 동시에 사용
- **native messaging host**: Chrome, Brave, Arc, Edge, Vivaldi, Opera에 자동 설치

**권한 모델** (2단):
1. **Claude Code permission mode** -- CLI 수준의 도구 권한
2. **Chrome 확장 사이트별 권한** -- 크롬 확장이 사이트별로 접근을 제한

**빌드 분기**:
- ant 전용 DEV/ANT 확장 ID 분기가 존재한다
- ant 환경에서는 `browser_task` lightning 루프가 주입되며, public 빌드에서는 트리셰이킹으로 제거된다
---

## 5. 명령어 시스템 (src/commands/)

### 5.1 명령어 타입

```typescript
type Command =
  | PromptCommand     // AI에게 프롬프트 전송 (대부분의 명령어)
  | LocalCommand      // 로컬에서 텍스트 반환
  | LocalJSXCommand   // 로컬에서 JSX 반환 (UI 컴포넌트)
```

### 5.2 등록된 명령어 (70개 이상)

주요 카테고리:
- **Git 관련**: commit, diff, review, pr_comments, branch
- **설정**: config, theme, permissions, privacy-settings, keybindings
- **세션**: session, resume, share, export, summary
- **인증**: login, logout, oauth-refresh
- **MCP/플러그인**: mcp, plugin, reload-plugins, skills
- **개발도구**: doctor, debug-tool-call, heapdump, perf-issue
- **UI**: help, color, vim, compact, files, status

**비평**: 70개 이상의 명령어가 `commands.ts` 한 파일에 등록된다. `biome-ignore-all assist/source/organizeImports` 주석으로 임포트 순서를 고정하는데, 이는 ANT-ONLY 마커 때문이다. feature flag으로 게이트된 명령어와 항상 활성화된 명령어가 섞여 있어 가독성이 떨어진다.

---

## 6. 쿼리 파이프라인 (src/query.ts)

### 6.1 핵심 흐름

```
submitMessage()
    |
    v
processUserInput()  -- 슬래시 명령 처리, 이미지/파일 첨부
    |
    v
fetchSystemPromptParts()  -- 시스템 프롬프트 + 유저/시스템 컨텍스트
    |
    v
query() -- 메인 루프
    |
    +---> normalizeMessagesForAPI()  -- API 호환 메시지 정규화
    |
    +---> claude.ts: createStream()  -- Anthropic SDK 스트리밍
    |
    +---> StreamingToolExecutor / runTools  -- 도구 실행
    |
    +---> autoCompact()  -- 컨텍스트 윈도우 초과 시 자동 압축
    |
    +---> query() 재귀  -- 도구 결과 기반 재쿼리
```

### 6.2 자동 컴팩션 (src/services/compact/)

```typescript
function getEffectiveContextWindowSize(model: string): number {
  const reservedTokensForSummary = Math.min(
    getMaxOutputTokensForModel(model),
    MAX_OUTPUT_TOKENS_FOR_SUMMARY,  // 20,000
  )
  return contextWindow - reservedTokensForSummary
}
```

- 컨텍스트 윈도우 크기에서 20,000 토큰을 예약 (요약 출력용)
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW` 환경변수로 윈도우 크기 오버라이드 가능
- 연속 실패 시 circuit breaker로 재시도 중단

### 6.3 max_output_tokens 복구

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

function isWithheldMaxOutputTokens(msg): msg is AssistantMessage {
  return msg?.type === 'assistant' && msg.apiError === 'max_output_tokens'
}
```

**배울 점**: 출력 토큰 초과 시 최대 3회까지 자동 복구를 시도한다. SDK 호출자에게 중간 에러를 노출하지 않도록 에러 메시지를 "보류"하는 패턴이 있다.

---

## 7. MCP 서버 (mcp-server/)

### 7.1 독립 패키지

`mcp-server/`는 별도 `package.json`과 `tsconfig.json`을 가진 독립 패키지다.

**제공 기능**:
- **Resources**: 아키텍처 개요, 도구 레지스트리, 명령어 레지스트리
- **Resource Templates**: `claude-code://source/{path}` -- 소스 파일 읽기
- **Tools**: list_tools, list_commands, get_tool_source, get_command_source, read_source_file, search_source
- **Prompts**: explain_tool, explain_command

### 7.2 보안 -- path traversal 방지

```typescript
function safePath(relPath: string): string | null {
  const resolved = path.resolve(SRC_ROOT, relPath)
  if (!resolved.startsWith(SRC_ROOT)) return null
  return resolved
}
```

**평가**: 단순하지만 효과적인 경로 순회 방지. `path.resolve()`로 정규화한 후 `SRC_ROOT` 접두사를 검증한다.

---

## 8. 상태 관리 (src/state/)

### 8.1 AppState

```typescript
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  toolPermissionContext: ToolPermissionContext
  messages: Message[]
  tasks: TaskState[]
  plugins: LoadedPlugin[]
  notifications: Notification[]
  // ... 100개 이상의 필드
}>
```

**비평**: `AppState`가 100개 이상의 필드를 가진 거대 타입이다. `DeepImmutable`로 불변성은 보장하지만, 상태 업데이트 시 전체 객체를 재생성해야 하므로 성능 비용이 있다. 상태 슬라이싱이나 도메인별 분리가 필요해 보인다.

### 8.2 상태 업데이트 패턴

```typescript
setAppState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    mode: newMode,
  },
}))
```

React 스타일 불변 업데이트를 사용하지만, 깊이 중첩된 객체에서는 스프레드가 복잡해진다.

---

## 9. 에이전트 시스템

### 9.1 agent.md -- 운영 가이드

```
Core Rules:
- 작고 타겟팅된 변경 유지
- 기존 명령어 동작 보존
- src/commands/, src/tools/ 패턴 따르기
- 지역적 문제 수정 시 광범위 리팩토링 금지
```

### 9.2 Skill.md -- 저장소 스킬

Skill.md는 에이전트가 이 코드베이스에서 작업할 때 참조하는 "저장소 지식"이다. 디렉토리 맵, 핵심 파일, 패턴, 네이밍 컨벤션을 문서화한다.

### 9.3 스킬 시스템 (src/skills/)

- `loadSkillsDir.ts`: 파일시스템에서 스킬을 동적 로드 (CLAUDE.md 파일 기반)
- `bundledSkills.ts`: 빌트인 스킬
- `mcpSkillBuilders.ts`: MCP 서버에서 스킬 빌드

**특이점**: 스킬은 `commands_DEPRECATED`, `skills`, `plugin`, `managed`, `bundled`, `mcp` 등 6개 출처에서 로드된다. `commands_DEPRECATED` 출처가 있는 것으로 보아 기존 명령어 시스템에서 스킬 시스템으로 마이그레이션 중인 것 같다.

---

## 10. 코디네이터 모드 (src/coordinator/)

```typescript
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

코디네이터 모드는 멀티에이전트 협업을 위한 기능이다. 메인 에이전트가 "코디네이터"로 동작하며, TeamCreateTool/SendMessageTool 등으로 하위 에이전트를 생성하고 조율한다.

**세션 모드 복원**: 세션 재개 시 코디네이터 모드 상태를 자동 복원한다:

```typescript
export function matchSessionMode(sessionMode): string | undefined {
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }
}
```

---

## 11. 서비스 레이어 (src/services/)

| 서비스 | 파일 수 | 핵심 역할 |
|--------|---------|----------|
| api/ | 20개 | Anthropic SDK 래핑, 스트리밍, 재시도, 캐시 |
| mcp/ | 23개 | MCP 클라이언트, OAuth, 도구 디스커버리 |
| oauth/ | 5개 | OAuth 2.0 플로우, 프로필 관리 |
| analytics/ | 다수 | GrowthBook, 텔레메트리, 이벤트 로깅 |
| compact/ | 다수 | 대화 압축, 자동 컴팩션, 세션 메모리 |
| plugins/ | 다수 | 플러그인 로더, 마켓플레이스 |
| lsp/ | 다수 | Language Server Protocol 매니저 |
| policyLimits/ | 다수 | 조직 정책, 레이트 리밋 |

### API 클라이언트 (src/services/api/client.ts)

다중 프로바이더 지원:
- **Direct API**: `ANTHROPIC_API_KEY`
- **AWS Bedrock**: AWS 자격 증명
- **Google Vertex AI**: GCP 프로젝트 + 리전
- **Azure Foundry**: Azure 리소스 + API 키 또는 Azure AD

```
인증 결정 흐름:
1. OAuth 토큰 확인 (Claude.ai 구독자)
2. 환경변수 API 키 (ANTHROPIC_API_KEY)
3. apiKeyHelper 설정 (외부 키 관리자)
4. Keychain/plaintext 저장소
```

---

## 12. 모듈 상호작용 핵심 흐름도

```
[사용자 터미널 입력]
        |
        v
  [PromptInput.tsx] -- Ink 컴포넌트, 키바인딩, vim 모드
        |
        v
  [processUserInput()] -- 슬래시 명령 감지, 이미지 첨부 처리
        |
   +----+----+
   |         |
슬래시      일반
명령어      프롬프트
   |         |
   v         v
[Command    [query()]
 Handler]      |
   |         +---> [claude.ts] -- API 스트리밍
   v         |
[LocalJSX/   +---> [StreamingToolExecutor]
 Text/           |
 Prompt]     +---> [permissions.ts] -- 도구 권한 확인
             |
             +---> [autoCompact] -- 컨텍스트 관리
             |
             v
          [Message 스트림] --> [REPL.tsx] 렌더링
```

---

## 13. 종합 평가

### 강점
1. **성능 최적화가 DNA에 녹아있다**: 시작 시간 최적화, 지연 로딩, 프리커넥트, 캐시 전략이 체계적
2. **확장 가능한 도구/명령어 시스템**: `buildTool()` 팩토리 패턴으로 새 도구 추가가 용이
3. **다중 프로바이더 지원**: AWS, GCP, Azure를 모두 지원하는 유연한 API 클라이언트
4. **스트리밍 도구 실행**: 응답 스트리밍 중 도구를 즉시 실행하는 혁신적 패턴

### 약점
1. **AppState 비대화**: 100개 이상 필드를 가진 거대 상태 객체
2. **순환 의존성**: 모듈 간 결합도가 높아 `require()` 우회가 다수
3. **코디네이터/스웜 복잡도**: 멀티에이전트 기능이 기존 코드에 feature flag으로 얽혀 있어 이해하기 어려움
4. **거대 파일들**: 핵심 파일 6개가 5,000줄 이상으로, 분리가 시급
