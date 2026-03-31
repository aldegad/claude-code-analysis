# 16. MCP (Model Context Protocol) 구현 분석

> 분석일: 2026-03-31  
> 분석 대상: Claude Code CLI 소스 (약 52만 줄)  
> 분석자: 뚝딱이 (개발팀 개발자)

---

## 목차

1. [아키텍처 개요](#1-아키텍처-개요)
2. [MCP 서버 구현 (mcp-server/)](#2-mcp-서버-구현)
3. [프로토콜 스펙: 지원 메서드](#3-프로토콜-스펙)
4. [도구 등록 메커니즘](#4-도구-등록-메커니즘)
5. [StreamingToolExecutor: 병렬/직렬 파티셔닝](#5-streamingtoolexecutor)
6. [MCP 클라이언트 구현](#6-mcp-클라이언트-구현)
7. [플러그인 시스템 연동](#7-플러그인-시스템-연동)
8. [보안: 인증/인가/샌드박싱](#8-보안)
9. [성능 최적화](#9-성능-최적화)
10. [에러 처리 및 복구 전략](#10-에러-처리)

---

## 1. 아키텍처 개요

### 전체 아키텍처 다이어그램

```
+--------------------------------------------------------------------+
|                        Claude Code CLI                              |
|                                                                     |
|  +------------------+     +-------------------+     +-----------+   |
|  |   QueryEngine    |---->| StreamingTool-    |---->| Tool.ts   |   |
|  |   (query.ts)     |     | Executor          |     | buildTool |   |
|  +------------------+     +-------------------+     +-----------+   |
|          |                        |                      |          |
|          v                        v                      v          |
|  +------------------+     +-------------------+  +-------------+   |
|  | AppState         |     | toolExecution.ts  |  | MCPTool.ts  |   |
|  | (mcp.clients,    |     | runToolUse()      |  | (template)  |   |
|  |  mcp.tools,      |     +-------------------+  +-------------+   |
|  |  mcp.resources)  |            |                      |          |
|  +------------------+            v                      v          |
|          |              +--------------------+  +-------------+    |
|          |              | Permission Check   |  | callMCPTool |    |
|          |              | (auto-mode/user)   |  | (client.ts) |    |
|          |              +--------------------+  +------+------+    |
|          |                                             |           |
+----------+---------------------------------------------+-----------+
           |                                             |
           v                                             v
+--------------------+                    +-----------------------------+
| MCPConnection-     |                    |   MCP Client SDK            |
| Manager.tsx        |                    |   @modelcontextprotocol/sdk |
| (React Context)    |                    +-----------------------------+
+--------------------+                           |
           |                                     |
           v                                     v
+--------------------+              +----------------------------+
| useManageMCP-      |              |    Transport Layer          |
| Connections.ts     |              |                            |
| (hook: connect,    |              |  +----------+  +---------+ |
|  reconnect, sync)  |              |  | Stdio    |  | SSE     | |
+--------------------+              |  | Transport|  | Client  | |
           |                        |  +----------+  +---------+ |
           v                        |  +----------+  +---------+ |
+--------------------+              |  | HTTP     |  | WebSocket||
| config.ts          |              |  | Streamable|  | Transport||
| (getAllMcpConfigs)  |              |  +----------+  +---------+ |
+--------------------+              |  +----------+  +---------+ |
           |                        |  | InProcess|  | SdkControl||
           v                        |  | Transport|  | Transport ||
+--------------------+              |  +----------+  +---------+ |
| Sources:           |              +----------------------------+
| - .mcp.json        |                          |
| - settings.json    |                          v
| - enterprise       |              +----------------------------+
| - claude.ai proxy  |              |  External MCP Servers       |
| - plugins          |              |  (stdio, sse, http, ws)     |
| - SDK (in-process) |              +----------------------------+
+--------------------+
```

### MCP 도구 호출 흐름도

```
사용자 입력
    |
    v
QueryEngine (LLM API 호출)
    |
    v
LLM 응답 (tool_use block: "mcp__serverName__toolName")
    |
    v
StreamingToolExecutor.addTool()
    |
    +-- isConcurrencySafe? ──> client.ts가 readOnlyHint를 isConcurrencySafe에 매핑
    |       |
    |   [Yes] ──> 다른 concurrent-safe 도구와 병렬 실행
    |   [No]  ──> 현재 실행 중인 도구 모두 완료 후 단독 실행
    |
    v
processQueue() ──> executeTool()
    |
    v
runToolUse() (toolExecution.ts)
    |
    +-- Permission Check (auto-mode / interactive)
    |       |
    |   [허용] ──> 도구 실행
    |   [거부] ──> REJECT_MESSAGE 반환
    |
    v
MCPTool.call() (client.ts에서 오버라이드된 call())
    |
    v
ensureConnectedClient() ──> 연결 확인/재연결
    |
    v
callMCPToolWithUrlElicitationRetry()
    |
    +-- URL Elicitation 필요? (-32042 에러)
    |       |
    |   [Yes] ──> 사용자에게 URL 표시 ──> 재시도 (최대 3회)
    |   [No]  ──> 직접 실행
    |
    v
callMCPTool()
    |
    v
client.callTool({name, arguments, _meta})
    |
    +-- Promise.race([실행, 타임아웃])
    |       |
    |   [성공] ──> 결과 변환 (텍스트/이미지/리소스)
    |   [isError] ──> McpToolCallError 발생
    |   [타임아웃] ──> 타임아웃 에러 발생
    |   [401] ──> McpAuthError (needs-auth 상태)
    |   [세션만료] ──> 캐시 클리어 + 재연결 재시도 (1회)
    |
    v
결과 반환 ──> StreamingToolExecutor.getCompletedResults()
    |
    v
QueryEngine ──> LLM에 tool_result 전달
```

---

## 2. MCP 서버 구현

### 2.1 서버 구조

**위치**: `mcp-server/` (독립 패키지)

| 파일 | 역할 |
|------|------|
| `src/server.ts` | 서버 팩토리 (`createServer()`) - 트랜스포트 비의존적 |
| `src/index.ts` | STDIO 진입점 |
| `package.json` | `@modelcontextprotocol/sdk ^1.12.1` 의존 |
| `tsconfig.json` | ESM 빌드 설정 |

### 2.2 서버 메타데이터

```typescript
const server = new Server(
  { name: "claude-code-explorer", version: "1.1.0" },
  {
    capabilities: {
      tools: {},       // 8개 도구 지원
      resources: {},   // 3개 정적 리소스 + 1개 템플릿
      prompts: {},     // 5개 프롬프트
    },
  }
);
```

### 2.3 서버가 노출하는 도구 (7개)

| 도구명 | 설명 | 필수 파라미터 |
|--------|------|---------------|
| `list_tools` | 에이전트 도구 전체 목록 | 없음 |
| `list_commands` | 슬래시 커맨드 전체 목록 | 없음 |
| `get_tool_source` | 특정 도구의 소스코드 읽기 | `toolName` |
| `get_command_source` | 특정 커맨드의 소스코드 읽기 | `commandName` |
| `read_source_file` | src/ 하위 파일 읽기 (라인 범위 지정 가능) | `path` |
| `search_source` | 소스코드 정규식 검색 | `pattern` |
| `list_directory` | 디렉토리 목록 | `path` |
| `get_architecture` | 아키텍처 개요 | 없음 |

### 2.4 리소스 (3 정적 + 1 템플릿)

| URI | MIME | 설명 |
|-----|------|------|
| `claude-code://architecture` | text/markdown | README.md 기반 개요 |
| `claude-code://tools` | application/json | 도구 레지스트리 |
| `claude-code://commands` | application/json | 커맨드 레지스트리 |
| `claude-code://source/{path}` | text/plain | 소스 파일 템플릿 |

### 2.5 프롬프트 (5개)

| 프롬프트명 | 인자 | 설명 |
|------------|------|------|
| `explain_tool` | `toolName` | 도구 구현 분석 |
| `explain_command` | `commandName` | 커맨드 구현 분석 |
| `architecture_overview` | 없음 | 전체 아키텍처 투어 |
| `how_does_it_work` | `feature` | 서브시스템 설명 (feature map 내장) |
| `compare_tools` | `tool1`, `tool2` | 도구 비교 분석 |

### 2.6 보안 메커니즘

- **경로 순회 방지**: `safePath()` 함수가 `path.resolve()` 후 `SRC_ROOT` 프리픽스 검증
- **읽기 전용**: 파일 시스템 쓰기 기능 없음, 오직 읽기만 허용

---

## 3. 프로토콜 스펙

### 3.1 서버 측 지원 메서드 (mcp-server/src/server.ts)

| 메서드 | 스키마 | 용도 |
|--------|--------|------|
| `tools/list` | `ListToolsRequestSchema` | 사용 가능한 도구 목록 반환 |
| `tools/call` | `CallToolRequestSchema` | 도구 실행 |
| `resources/list` | `ListResourcesRequestSchema` | 정적 리소스 목록 |
| `resources/read` | `ReadResourceRequestSchema` | 리소스 읽기 |
| `resources/templates/list` | `ListResourceTemplatesRequestSchema` | 리소스 템플릿 목록 |
| `prompts/list` | `ListPromptsRequestSchema` | 프롬프트 목록 |
| `prompts/get` | `GetPromptRequestSchema` | 프롬프트 실행 |

### 3.2 클라이언트 측 지원 메서드 (src/services/mcp/client.ts)

| 메서드 | 용도 |
|--------|------|
| `tools/list` | 외부 서버 도구 탐색 |
| `tools/call` | 외부 서버 도구 호출 (callTool) |
| `resources/list` | 리소스 탐색 |
| `prompts/list` | 프롬프트 탐색 |
| `roots/list` | 루트 디렉토리 요청 처리 (서버에서 클라이언트 호출) |
| `elicitation` | 사용자 입력 요청 (URL 인증 등) |
| `notifications` | 채널 알림, 파일 업데이트, 권한 요청 |

### 3.3 트랜스포트 타입

```typescript
type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk'
```

| 트랜스포트 | 대상 | 구현 |
|-----------|------|------|
| **stdio** | 로컬 CLI 프로세스 | `StdioClientTransport` (자식 프로세스 spawn) |
| **sse** | 원격 서버 (레거시) | `SSEClientTransport` + OAuth |
| **sse-ide** | IDE 확장 (VS Code/JetBrains) | SSE 기반, 인증 불필요 |
| **http** | 원격 서버 (Streamable HTTP) | `StreamableHTTPClientTransport` |
| **ws** | WebSocket 서버 | 커스텀 `WebSocketTransport` |
| **ws-ide** | IDE WebSocket 확장 | WebSocket + IDE 인증 토큰 |
| **sdk** | 인프로세스 (SDK 내장) | `SdkControlClientTransport` |
| **claudeai-proxy** | claude.ai 커넥터 | HTTP + OAuth Bearer 토큰 |

#### CLI 등록 vs 런타임 제약

**소스**: `src/services/mcp/utils.ts:313-319`

3.3 표의 8종 트랜스포트 중, CLI `mcp add` 명령이 허용하는 것은 **stdio, sse, http** 3종뿐이다. 런타임은 ws(WebSocket)도 이해하고 정상 동작하지만, CLI 등록 단계에서 유효성 검증에 의해 거부된다. ws 트랜스포트가 필요한 서버는 설정 파일을 직접 편집해야 한다.

---

## 4. 도구 등록 메커니즘

### 4.1 전체 도구 수

`buildTool()` 함수로 등록된 빌트인 도구: **40개 이상** (tools/ 디렉토리 각 하위 폴더 = 1 도구)

### 4.2 도구 등록 흐름

```
buildTool() ──> Tool 인터페이스 생성
    |
    +-- name: 도구 식별자
    +-- inputSchema: Zod 스키마 (입력 검증)
    +-- outputSchema: 출력 스키마
    +-- call(): 실행 함수
    +-- checkPermissions(): 권한 검증
    +-- isConcurrencySafe(): 병렬 실행 가능 여부
    +-- isReadOnly(): 읽기 전용 여부
    +-- isDestructive(): 파괴적 연산 여부
    +-- renderToolUseMessage(): UI 렌더링
```

### 4.3 MCP 도구 동적 등록

**핵심 함수**: `fetchToolsForClient()` (client.ts, 라인 1743)

```
MCP 서버 연결
    |
    v
client.request({method: 'tools/list'}, ListToolsResultSchema)
    |
    v
결과의 각 tool에 대해:
    |
    v
MCPTool 템플릿을 복제하고 다음을 오버라이드:
    +-- name: "mcp__${normalizedServerName}__${toolName}"
    +-- mcpInfo: {serverName, toolName}
    +-- description: tool.description (최대 2048자)
    +-- isConcurrencySafe: client.ts가 tool.annotations?.readOnlyHint를 isConcurrencySafe에 매핑
    +-- isReadOnly: tool.annotations?.readOnlyHint
    +-- isDestructive: tool.annotations?.destructiveHint
    +-- isOpenWorld: tool.annotations?.openWorldHint
    +-- inputJSONSchema: tool.inputSchema
    +-- call: callMCPToolWithUrlElicitationRetry 래핑
    +-- searchHint: tool._meta?.['anthropic/searchHint']
    +-- alwaysLoad: tool._meta?.['anthropic/alwaysLoad']
```

### 4.4 도구 네이밍 컨벤션

```
mcp__{normalizedServerName}__{toolName}

예: mcp__slack__send_message
    mcp__claude_ai_GitHub__search_code
```

정규화 규칙 (`normalizeNameForMCP`):
- `[^a-zA-Z0-9_-]` -> `_`
- claude.ai 서버: 연속 `_` 병합, 선행/후행 `_` 제거

### 4.5 도구 호출 상세

**소스**: `src/services/mcp/client.ts:1768, 3091`

4.3의 동적 등록 이후, 실제 도구 호출 시의 내부 동작을 정리한다.

**호출 흐름**:
1. 도구명 변환: `mcp__{서버명}__{도구명}` 형식으로 역파싱하여 대상 서버와 도구를 식별
2. `callTool` JSON-RPC 요청을 MCP 클라이언트를 통해 전송

**에러 처리 분기**:
- **401 응답**: 인증 오류로 분류되어 `McpAuthError` 발생. `needs-auth` 상태로 전환
- **HTTP 세션 만료**: 캐시를 클리어한 뒤 재연결을 시도하고, 1회 재시도

**재연결 전략**:
- 원격 transport(sse, http)만 백오프 재연결을 수행한다
- 최대 5회까지 지수 백오프(초기 1초, 최대 30초)로 재시도
- 로컬 transport(stdio, sdk)는 재연결 대상이 아니다

---

## 5. StreamingToolExecutor

### 5.1 위치 및 역할

**파일**: `src/services/tools/StreamingToolExecutor.ts`

LLM 응답에서 도구 호출 블록이 스트리밍으로 도착할 때, 실시간으로 실행을 스케줄링하는 핵심 컴포넌트.

### 5.2 동시성 모델

```
도구 분류:
  +-- isConcurrencySafe === true  ──> "Concurrent-safe" (읽기 전용 도구)
  +-- isConcurrencySafe === false ──> "Exclusive" (쓰기 도구)

실행 규칙:
  1. Concurrent-safe 도구끼리는 병렬 실행 가능
  2. Exclusive 도구는 단독 실행 (다른 도구 없어야 함)
  3. Exclusive 도구 뒤에 대기 중인 도구는 완료까지 차단
```

### 5.3 상태 머신

```
queued ──> executing ──> completed ──> yielded
                |
                +──> (abort) ──> completed (synthetic error)
```

### 5.4 에러 전파 메커니즘

```
BashTool 에러 발생
    |
    v
hasErrored = true
siblingAbortController.abort('sibling_error')
    |
    v
다른 실행 중인 Bash 도구 → 즉시 중단
다른 도구 (Read/WebFetch 등) → 영향 없음 (Bash만 전파)
    |
    v
아직 큐에 있는 도구 → synthetic error message 생성
    "Cancelled: parallel tool call {description} errored"
```

### 5.5 진행 상태 보고

- `pendingProgress` 배열로 각 도구의 진행 메시지 버퍼링
- `progressAvailableResolve`를 통해 비동기 웨이크업
- `getCompletedResults()` 호출 시 진행 메시지 즉시 yield (도구 완료 전에도)
- `getRemainingResults()`: 모든 도구 완료까지 대기하며 결과를 AsyncGenerator로 스트리밍

### 5.6 사용자 인터럽트 처리

```
사용자 ESC 또는 새 메시지 입력
    |
    v
abortController.signal에 'interrupt' reason 설정
    |
    v
각 도구의 interruptBehavior 확인:
    +-- 'cancel' ──> 즉시 중단, REJECT_MESSAGE 반환
    +-- 'block'  ──> 계속 실행 (기본값)
```

---

## 6. MCP 클라이언트 구현

### 6.1 연결 관리

**핵심 파일**: `src/services/mcp/client.ts` (3000줄+, 프로젝트에서 가장 큰 파일 중 하나)

#### 연결 생명주기

```
설정 로딩 (config.ts: getAllMcpConfigs())
    |
    v
서버 분류: local (stdio/sdk) vs remote (sse/http/ws)
    |
    v
getMcpToolsCommandsAndResources()
    |
    +-- 로컬: getMcpServerConnectionBatchSize() (기본 3) 병렬 연결
    +-- 원격: getRemoteMcpServerConnectionBatchSize() (기본 20) 병렬 연결
    |
    v
connectToServer() (memoized - 동일 서버 재연결 방지)
    |
    +-- 트랜스포트별 생성 로직:
    |   stdio: StdioClientTransport + 환경변수 확장
    |   sse: SSEClientTransport + OAuth AuthProvider
    |   http: StreamableHTTPClientTransport + 타임아웃 래핑
    |   ws: WebSocketTransport + TLS/프록시
    |   sdk: SdkControlClientTransport (IPC)
    |
    v
Client.connect(transport) + 타임아웃 (기본 30초)
    |
    v
fetchToolsForClient() + fetchCommandsForClient() + fetchResourcesForClient()
    |
    v
AppState 업데이트 (mcp.clients, mcp.tools, mcp.resources)
```

### 6.2 MCPConnectionManager (React Context)

```typescript
// src/services/mcp/MCPConnectionManager.tsx
export function MCPConnectionManager({children, dynamicMcpConfig, isStrictMcpConfig}) {
  const { reconnectMcpServer, toggleMcpServer } = useManageMCPConnections(
    dynamicMcpConfig, isStrictMcpConfig
  )
  // React Context로 하위 컴포넌트에 reconnect/toggle 함수 제공
}
```

### 6.3 설정 소스 우선순위

```
1. Enterprise (managed-mcp.json) ── 최고 우선
2. Dynamic (--mcp-config, --channels) ── CLI 인자
3. Plugin MCP 서버 (manifest.mcpServers / .mcpb) ── 플러그인 제공
4. Claude.ai 커넥터 (API 페치) ── claude.ai 웹 UI에서 토글
5. Project (.mcp.json) ── 프로젝트 로컬
6. User (~/.claude/settings.json) ── 사용자 글로벌
7. Local (.claude/settings.local.json) ── 로컬 오버라이드
```

중복 제거 로직:
- `dedupPluginMcpServers()`: 플러그인 vs 수동 서버 시그니처 비교
- `dedupClaudeAiMcpServers()`: claude.ai 커넥터 vs 수동 서버

#### 설정 병합 순서 상세

**소스**: `src/services/mcp/config.ts:1164-1218`

6.3에 기술된 우선순위를 실제 코드에서 병합하는 순서는 다음과 같다:

```
pluginServers → userServers → approvedProjectServers → localServers
```

- 프로젝트 `.mcp.json`은 가까운 디렉토리부터 우선 적용된다 (가장 가까운 `.mcp.json`이 먼 것보다 우선)
- **프로젝트 서버는 승인 기반**: `.mcp.json`에 정의되어 있더라도 사용자가 승인(approved)하지 않으면 연결을 시도하지 않는다
- 상위 소스(Enterprise, Dynamic)가 하위 소스(Plugin, User, Project)를 덮어쓴다

### 6.4 InProcessTransport

**파일**: `src/services/mcp/InProcessTransport.ts`

프로세스 간 통신 없이 같은 프로세스 내에서 MCP 서버/클라이언트를 연결.

```typescript
export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)  // a.send() → b.onmessage()
  b._setPeer(a)  // b.send() → a.onmessage()
  return [a, b]
}
```

`queueMicrotask()`를 사용해 동기 요청/응답 사이클의 스택 깊이 문제 방지.

### 6.5 SdkControlTransport

**파일**: `src/services/mcp/SdkControlTransport.ts`

SDK 인프로세스 MCP 서버용. CLI 프로세스와 SDK 프로세스 간 control message를 통해 JSONRPC 메시지를 라우팅.

```
CLI → SDK: SdkControlClientTransport.send()
  → control request {server_name, message}
  → stdout → SDK 프로세스
  → SdkControlServerTransport.onmessage()
  → MCP 서버 처리
  → SdkControlServerTransport.send() (응답)
  → control response → CLI
```

---

## 7. 플러그인 시스템 연동

### 7.1 플러그인-MCP 통합 구조

**파일**: `src/utils/plugins/mcpPluginIntegration.ts`

```
Plugin manifest
    |
    +-- manifest.mcpServers가 있으면:
        |
        +-- string인 경우:
        |   +-- .mcpb 확장자 → MCPB 로더 (DXT manifest 추출)
        |   +-- 기타 → JSON 파일 로드
        |
        +-- 배열인 경우:
        |   +-- 각 항목을 병렬 로드 (Promise.all)
        |   +-- string → 파일 로드
        |   +-- object → 인라인 설정
        |
        +-- object인 경우:
            +-- 직접 MCP 설정으로 사용
```

### 7.2 MCPB (MCP Bundle) 처리

- `.mcpb` 파일 또는 URL에서 DXT manifest 추출
- 사용자 설정 필요 시 `needs-config` 반환 (에러가 아님)
- 환경변수 치환: `CLAUDE_PLUGIN_ROOT` 등 자동 주입

### 7.3 플러그인 MCP 서버 스코프

```typescript
type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope  // 'local'|'user'|'project'|'dynamic'|'enterprise'|'claudeai'|'managed'
  pluginSource?: string  // 예: 'slack@anthropic'
}
```

### 7.4 플러그인 네이밍과 레지스트리

**소스**: `src/utils/plugins/mcpPluginIntegration.ts`, `src/plugins/bundled/index.ts`

- `plugin:` 프리픽스는 순수 네임스페이스일 뿐이다. Discord 등 특정 플러그인에 대한 하드코딩 특례는 없다
- built-in 플러그인 레지스트리는 비어있다 (`initBuiltinPlugins`에 등록된 것이 없음)
- 서버명은 `plugin:${pluginName}:${serverName}` 형태로 재작성된다
- 환경변수 치환: `CLAUDE_PLUGIN_ROOT` 등이 서버 설정 내에서 자동 치환된다

---

## 8. 보안

### 8.1 OAuth 인증 (원격 MCP 서버)

**파일**: `src/services/mcp/auth.ts`

```
ClaudeAuthProvider (OAuthClientProvider 구현)
    |
    +-- 토큰 저장: macOS Keychain / 보안 스토리지
    +-- 자동 갱신: refreshAuthorization (30초 타임아웃)
    +-- PKCE 지원: SHA-256 code_challenge
    +-- 브라우저 기반 인증 플로우
    +-- XAA (Cross-App Access) / SEP-990 지원
    +-- 비표준 에러 정규화 (Slack 등)
```

### 8.2 인증 캐시

```
mcp-needs-auth-cache.json
    |
    +-- 서버별 타임스탬프 (15분 TTL)
    +-- 401 응답 시 캐시 → 15분간 재연결 시도 건너뜀
    +-- 직렬화된 쓰기 체인 (writeChain)으로 race condition 방지
```

### 8.3 Claude.ai 프록시 보안

```typescript
createClaudeAiProxyFetch():
    +-- OAuth Bearer 토큰 자동 첨부
    +-- 401 시 1회 재시도 (토큰 갱신 후)
    +-- 토큰 변경 감지 (동시 401 대응)
```

### 8.4 도구 권한 모델

MCP 도구의 기본 권한:
```typescript
checkPermissions() → { behavior: 'passthrough' }
// 모든 MCP 도구는 passthrough — 상위 권한 시스템에 위임
```

자동 모드 분류기:
```typescript
mcpToolInputToAutoClassifierInput():
    // 입력을 "key=value key2=value2" 형태로 인코딩
    // 보안 분류기가 파괴적 명령 탐지
```

### 8.5 Path Traversal 방지 (MCP 서버)

```typescript
function safePath(relPath: string): string | null {
  const resolved = path.resolve(SRC_ROOT, relPath)
  if (!resolved.startsWith(SRC_ROOT)) return null  // 경로 순회 차단
  return resolved
}
```

### 8.6 엔터프라이즈 정책

```
allowedMcpServers ── 허용 목록 (이름/커맨드/URL 기반)
deniedMcpServers  ── 차단 목록 (이름/커맨드/URL 기반)
allowManagedMcpServersOnly ── 관리 서버만 허용 모드
```

URL 와일드카드 패턴 매칭 지원: `https://*.example.com/*`

### 8.7 채널 권한 (Telegram/Discord/iMessage)

```
channelPermissions.ts:
    +-- 5자 랜덤 ID 생성 (l 제외, 25^5 공간)
    +-- 비속어 블록리스트
    +-- 구조화된 이벤트 기반 (사용자 텍스트 직접 파싱 X)
    +-- GrowthBook 피처 게이트: tengu_harbor_permissions
    +-- 서버가 notifications/claude/channel/permission 이벤트 발행
```

### 8.8 입력 새니타이제이션

```typescript
recursivelySanitizeUnicode(result.tools)
// MCP 서버에서 받은 모든 도구 데이터에 유니코드 새니타이제이션 적용
```

---

## 9. 성능 최적화

### 9.1 연결 배치 처리

```typescript
// 로컬 서버 (stdio/sdk): 낮은 동시성 (기본 3)
// → 프로세스 스폰 리소스 경합 방지
getMcpServerConnectionBatchSize()  // MCP_SERVER_CONNECTION_BATCH_SIZE 환경변수

// 원격 서버: 높은 동시성 (기본 20)
// → 네트워크 연결은 가벼움
getRemoteMcpServerConnectionBatchSize()  // MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE 환경변수
```

pMap 기반 스케줄링 (2026-03 교체):
> 이전: 고정 크기 순차 배치 (배치 1 완료 → 배치 2 시작)  
> 현재: pMap으로 슬롯 해제 즉시 다음 서버 시작 (같은 동시성, 더 나은 스케줄링)

### 9.2 Memoization 전략

```
connectToServer()     ── lodash memoize (서버 이름+설정 기반)
fetchToolsForClient() ── LRU 캐시 (최대 20 항목)
fetchCommandsForClient() ── LRU 캐시
fetchResourcesForClient() ── LRU 캐시
getMcpAuthCache()     ── 싱글톤 프로미스 (쓰기 시 무효화)
```

### 9.3 타임아웃 관리

| 대상 | 타임아웃 | 설정 |
|------|---------|------|
| 서버 연결 | 30초 | `MCP_TIMEOUT` 환경변수 |
| 개별 요청 | 60초 | `MCP_REQUEST_TIMEOUT_MS` |
| 도구 호출 | ~27.8시간 (100M ms) | `MCP_TOOL_TIMEOUT` 환경변수 |
| OAuth 요청 | 30초 | `AUTH_REQUEST_TIMEOUT_MS` |
| 인증 캐시 TTL | 15분 | `MCP_AUTH_CACHE_TTL_MS` |

#### 타임아웃 래핑 전략

```typescript
wrapFetchWithTimeout(baseFetch):
    +-- GET 요청 제외 (SSE 스트림은 무기한)
    +-- POST에만 60초 타임아웃 적용
    +-- setTimeout + clearTimeout (AbortSignal.timeout 대신)
    +-- 이유: Bun에서 AbortSignal.timeout의 네이티브 메모리 2.4KB/요청 누수
    +-- Accept 헤더 보장: 'application/json, text/event-stream'
```

### 9.4 도구 설명 절삭

```typescript
const MAX_MCP_DESCRIPTION_LENGTH = 2048
// OpenAPI 생성 MCP 서버에서 15-60KB 설명 관찰
// p95 꼬리 절삭, 의도는 유지
```

### 9.5 출력 크기 관리

```typescript
// 기본 최대 MCP 출력: 25,000 토큰 (100,000자)
DEFAULT_MAX_MCP_OUTPUT_TOKENS = 25000

// 큰 출력 → 파일 저장 + 참조 반환
getLargeOutputInstructions():
    "Output saved to {path}. Use offset/limit to read specific portions."
    // Claude에게 순차적 청크 읽기 지시
```

### 9.6 이미지 처리 최적화

MCP 결과에 포함된 이미지:
```typescript
maybeResizeAndDownsampleImageBuffer()
// API 차원 제한에 맞춰 리사이즈 및 압축
```

### 9.7 비활성 서버 건너뛰기

```
disabled 서버 ──> 연결 시도 자체를 건너뜀 (HTTP 요청 없음)
needs-auth 캐시 ──> 15분간 재연결 건너뜀
hasMcpDiscoveryButNoToken ──> 토큰 없는 서버 프로빙 회피
```

---

## 10. 에러 처리

### 10.1 에러 클래스 계층

```
Error
  +-- McpAuthError ── 인증 실패 (401)
  |     └── serverName 포함
  +-- McpSessionExpiredError ── 세션 만료 (404 + -32001)
  +-- McpToolCallError ── 도구 실행 실패 (isError: true)
  |     └── mcpMeta (_meta 보존)
  +-- McpError (SDK) ── 프로토콜 에러
  |     └── code: ErrorCode.UrlElicitationRequired (-32042)
  +-- UnauthorizedError (SDK) ── OAuth 인증 필요
```

### 10.2 세션 만료 복구

```
callMCPTool() 호출
    |
    +-- McpSessionExpiredError 감지:
    |   HTTP 404 + JSON-RPC code -32001
    |   "Session not found" 메시지 확인
    |
    v
clearServerCache(name, config)
    |
    v
재연결 시도 (최대 1회)
    |
    +-- 성공 → 도구 재호출
    +-- 실패 → 에러 전파
```

### 10.3 자동 재연결 (SSE 서버)

```
useManageMCPConnections():
    +-- 최대 재연결 시도: 5회
    +-- 초기 백오프: 1초
    +-- 최대 백오프: 30초
    +-- 지수 백오프 적용
    +-- ResourceListChanged / ToolListChanged / PromptListChanged
        알림 수신 시 자동 도구 재조회
```

### 10.4 URL Elicitation 재시도

```
callMCPToolWithUrlElicitationRetry():
    +-- ErrorCode.UrlElicitationRequired (-32042) 감지
    +-- 최대 3회 재시도
    +-- 각 재시도마다:
        +-- Hook 체크 → 프로그래밍 방식 해결 가능
        +-- REPL 모드: 사용자에게 URL 표시 → 대기
        +-- SDK 모드: structuredIO 콜백
```

### 10.5 연결 실패 시 Graceful Degradation

```
서버 연결 실패
    |
    +-- FailedMCPServer 상태로 AppState에 등록
    +-- 도구/커맨드: 빈 배열
    +-- UI에 실패 상태 표시
    +-- 사용자가 /mcp에서 재연결 가능
```

### 10.6 플러그인 에러 분류

```typescript
PluginError types:
    'mcpb-download-failed'    // MCPB 다운로드 실패
    'mcpb-invalid-manifest'   // 매니페스트 유효성 검증 실패
    'mcpb-extract-failed'     // 번들 추출 실패
```

중복 에러 방지: `getErrorKey()`로 `type:source:plugin` 기반 deduplicate.

### 10.7 일반 MCP 서버 실패 원인 분석

실제 운영 시 MCP 서버 연결이 실패하는 주요 원인을 정리한다:

| 실패 원인 | 증상 | 대처 |
|-----------|------|------|
| **프로젝트 승인 누락** | `.mcp.json`에 서버를 정의했지만 연결되지 않음 | 프로젝트 서버는 승인(approved) 기반. 승인되지 않으면 연결 자체를 시도하지 않는다 |
| **Transport 불일치** | CLI `mcp add`에서 거부 | `mcp add`는 stdio/sse/http만 허용. 런타임은 ws도 이해하지만 CLI 등록 단계에서 거부된다 |
| **환경변수 미충족** | 서버 프로세스 시작 실패 | 서버 설정에 필요한 환경변수가 셸에 설정되어 있지 않으면 spawn 자체가 실패 |
| **인증/세션 문제** | 401 응답 후 `needs-auth` 상태 | 인증 캐시(15분 TTL)에 등록되어 재연결 시도를 건너뜀. `/mcp`에서 수동 재인증 필요 |
| **중복 제거** | 서버가 등록되었는데 연결되지 않음 | 같은 command/url이면 이름이 달라도 suppressed된다 (`dedupPluginMcpServers`, `dedupClaudeAiMcpServers`) |

---

## 핵심 발견 사항 요약

### 아키텍처 강점

1. **트랜스포트 추상화**: Transport 타입 6종(stdio, sse, sse-ide, http, ws, sdk) + 추가 분기 포함 8종(ws-ide, claudeai-proxy 추가)을 `Transport` 인터페이스로 통일. 새 트랜스포트 추가 시 코드 변경 최소화.
2. **동적 도구 등록**: MCP 서버의 도구가 런타임에 MCPTool 템플릿을 복제하여 네이티브 도구와 동일한 인터페이스로 작동.
3. **StreamingToolExecutor의 병렬화**: StreamingToolExecutor는 `isConcurrencySafe` 플래그로 파티셔닝하며, MCP 도구의 경우 client.ts가 readOnlyHint를 isConcurrencySafe에 매핑한다.
4. **다단계 인증**: OAuth PKCE + XAA + 프록시 + 채널 권한까지 계층적 보안 모델.
5. **Graceful Degradation**: 개별 서버 실패가 전체 시스템을 중단시키지 않음.

### 아키텍처 약점

1. **client.ts 파일 크기**: 3000줄 이상으로 god object 패턴에 가까움. 연결/도구 페치/도구 호출/인증을 분리해야 함.
2. **index.ts 중복 코드**: `mcp-server/src/index.ts`가 `server.ts`의 코드를 상당 부분 복제 (레거시 v1.0 코드가 그대로 존재).
3. **Memoization 복잡도**: `connectToServer`의 memoize 전략이 복잡도를 크게 높이지만 실질 성능 향상은 불명확 (코드 주석에서도 지적).
4. **타임아웃 기본값**: 도구 호출 타임아웃이 ~27.8시간으로 사실상 무제한. 장기 행 MCP 도구가 세션을 무한 차단할 위험.
5. **Bun 런타임 특수 분기**: WebSocket, AbortSignal 등에서 Bun/Node 분기가 여러 곳에 산재.

---

## 주요 파일 인덱스

| 파일 경로 | 줄 수 (추정) | 역할 |
|-----------|------------|------|
| `mcp-server/src/server.ts` | 957 | MCP 서버 팩토리 |
| `mcp-server/src/index.ts` | 959 | STDIO 진입점 (레거시 포함) |
| `src/services/mcp/client.ts` | 3200+ | MCP 클라이언트 핵심 (연결/도구/호출) |
| `src/services/mcp/types.ts` | 260 | 타입 정의 (8종 서버 설정, 5종 연결 상태) |
| `src/services/mcp/config.ts` | 800+ | 설정 로딩/검증/중복제거/정책 |
| `src/services/mcp/auth.ts` | 600+ | OAuth/XAA 인증 |
| `src/services/mcp/MCPConnectionManager.tsx` | 73 | React Context 제공 |
| `src/services/mcp/useManageMCPConnections.ts` | 600+ | 연결 라이프사이클 관리 |
| `src/services/mcp/InProcessTransport.ts` | 63 | 인프로세스 트랜스포트 |
| `src/services/mcp/SdkControlTransport.ts` | 100+ | SDK IPC 트랜스포트 |
| `src/services/mcp/normalization.ts` | 24 | 이름 정규화 |
| `src/services/mcp/mcpStringUtils.ts` | 50+ | 도구명 파싱/조립 |
| `src/services/mcp/channelPermissions.ts` | 100+ | 채널 권한 릴레이 |
| `src/services/mcp/elicitationHandler.ts` | 100+ | 사용자 입력 요청 처리 |
| `src/tools/MCPTool/MCPTool.ts` | 78 | MCP 도구 템플릿 |
| `src/services/tools/StreamingToolExecutor.ts` | 531 | 도구 병렬/직렬 실행 |
| `src/utils/mcpValidation.ts` | 100+ | 출력 크기 검증/절삭 |
| `src/utils/mcpOutputStorage.ts` | 60 | 대용량 출력 파일 저장 |
| `src/utils/plugins/mcpPluginIntegration.ts` | 200+ | 플러그인-MCP 연동 |
