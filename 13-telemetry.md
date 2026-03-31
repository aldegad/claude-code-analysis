# 13. 텔레메트리/데이터 수집 심층 분석

> 작성: 새미 (개발팀 비평가)
> 분석 대상: Anthropic Claude Code CLI (유출 소스, ~52만줄)
> 일자: 2026-03-31

---

## 1. 전체 아키텍처 개요

Claude Code는 **4중 텔레메트리 파이프라인**을 운영한다. "사용자 동의 없이 수집하지 않습니다"가 아니라, "내부 수집(Datadog, 1P)은 기본 ON이지만, 3P OTLP 텔레메트리는 CLAUDE_CODE_ENABLE_TELEMETRY로 명시적 opt-in이 필요하다." opt-out은 환경 변수로 하는 구조다.

```
사용자 이벤트
    │
    ├─→ analytics sink (logEvent) ──┬─→ [1] Datadog (로그 인테이크 US5)  ← 외부 제3자
    │                               └─→ [2] 1P Event Logging (Anthropic API) ← 자사 서버
    │
    └─→ OTEL 초기화 경로 ──────────┬─→ [3] BigQuery Metrics (Anthropic API) ← 자사 서버
                                    └─→ [4] 3P OTEL (고객 자체 엔드포인트)    ← 고객 설정 시만
```

> **참고**: Datadog과 1P는 같은 analytics sink(`logEvent`)에서 분기되며, BigQuery와 3P OTLP는 OTEL 초기화 경로에서 별도 exporter로 분기된다. 4개가 완전히 독립적인 fan-out은 아니다.

GrowthBook (피처 플래그 서비스)는 수집 자체는 아니지만, 사용자 속성을 서버에 전송하여 실험 타겟팅에 사용한다.

---

## 2. OpenTelemetry 구현 상세

### 2.1 사용 패키지 (6개, 5개가 아님)
| 패키지 | 버전 | 용도 |
|--------|-------|------|
| `@opentelemetry/api` | ^1.9.0 | 코어 API (trace, context) |
| `@opentelemetry/api-logs` | ^0.57.0 | 로그 API |
| `@opentelemetry/core` | ^1.30.0 | ExportResult 등 공유 타입 |
| `@opentelemetry/sdk-logs` | ^0.57.0 | BatchLogRecordProcessor (1P 이벤트 배치) |
| `@opentelemetry/sdk-metrics` | ^1.30.0 | MeterProvider (BigQuery 메트릭) |
| `@opentelemetry/sdk-trace-base` | ^1.30.0 | 트레이싱 (베타, 세션 추적) |

추가로 동적 import되는 OTLP exporter 패키지: `exporter-metrics-otlp-grpc`, `exporter-metrics-otlp-http`, `exporter-metrics-otlp-proto`, `exporter-logs-otlp-grpc/http/proto`, `exporter-trace-otlp-grpc/http/proto`, `exporter-prometheus` (총 10개 exporter 패키지).

### 2.2 내부 vs. 고객용 분리
- **내부 1P LoggerProvider**: `firstPartyEventLogger.ts`에서 별도 `LoggerProvider` 생성. 전역 OTEL 프로바이더에 등록하지 않음. `com.anthropic.claude_code.events` 스코프.
- **고객용 3P 텔레메트리**: `CLAUDE_CODE_ENABLE_TELEMETRY=1`로 활성화. 고객 OTLP 엔드포인트로 전송. 내부 이벤트와 격리.
- **BigQuery 메트릭**: API 고객/C4E/Teams 사용자 대상 별도 exporter. `api.anthropic.com/api/claude_code/metrics`로 전송.

---

## 3. 수집 데이터 전수 목록

### 3.1 모든 이벤트에 공통으로 포함되는 코어 메타데이터

`metadata.ts`의 `getEventMetadata()`가 모든 이벤트에 첨부하는 데이터:

| 데이터 필드 | 내용 | 프라이버시 위험도 |
|-------------|------|:-:|
| `user.id` (deviceId) | UUID, 최초 실행 시 생성 후 영구 저장 | **Medium** |
| `session.id` | 세션별 고유 ID | Low |
| `user.email` | OAuth 이메일 또는 git config 이메일 | **High** |
| `user.account_uuid` | Anthropic 계정 UUID | **Medium** |
| `organization.id` | 조직 UUID | **Medium** |
| `model` | 사용 중인 AI 모델명 | Low |
| `platform` | OS 종류 (darwin/linux/win32/wsl) | Low |
| `platformRaw` | 실제 process.platform 값 | Low |
| `arch` | CPU 아키텍처 | Low |
| `nodeVersion` | Node.js 버전 | Low |
| `terminal` | 터미널 종류 (iTerm, vscode 등) | Low |
| `packageManagers` | 감지된 패키지 매니저 목록 | Low |
| `runtimes` | 감지된 런타임 목록 | Low |
| `version` | Claude Code 버전 | Low |
| `buildTime` | 빌드 타임스탬프 | Low |
| `isInteractive` | 대화형 여부 | Low |
| `clientType` | 클라이언트 타입 | Low |
| `isCi` | CI 환경 여부 | Low |
| `isGithubAction` | GitHub Actions 여부 | Low |
| `subscriptionType` | 구독 유형 (pro/max/enterprise/team) | **Medium** |
| `rateLimitTier` | 레이트 리밋 등급 | Low |
| `firstTokenTime` | 최초 토큰 사용 일시 | **Medium** |
| `rh` (repoRemoteHash) | git remote URL의 SHA256 해시 (16자) | **Medium** |
| `entrypoint` | 진입점 (cli/local-agent 등) | Low |
| `agentId` | 에이전트 식별자 | Low |
| `parentSessionId` | 부모 세션 ID | Low |
| `teamName` | 팀 이름 | Low |
| `deploymentEnvironment` | 배포 환경 | Low |
| `wslVersion` | WSL 버전 | Low |
| `linuxDistroId/Version/Kernel` | Linux 배포판 정보 | Low |
| `vcs` | 감지된 VCS 종류 | Low |
| `githubEventName` | GitHub 이벤트 이름 | Low |
| `githubActionsRunnerOs` | GitHub Actions 러너 OS | Low |
| `claudeCodeContainerId` | 컨테이너 ID | **Medium** |
| `claudeCodeRemoteSessionId` | 원격 세션 ID | **Medium** |

### 3.2 프로세스 메트릭 (모든 이벤트)

| 데이터 필드 | 내용 | 위험도 |
|-------------|------|:-:|
| `uptime` | 프로세스 업타임 | Low |
| `rss` | 상주 메모리 크기 | Low |
| `heapTotal/heapUsed` | 힙 메모리 | Low |
| `external/arrayBuffers` | 외부 메모리 | Low |
| `constrainedMemory` | 제한 메모리 | Low |
| `cpuUsage` | CPU 사용량 (user/system) | Low |
| `cpuPercent` | CPU 점유율 | Low |

### 3.3 GitHub Actions 환경에서 추가 수집

| 데이터 필드 | 내용 | 위험도 |
|-------------|------|:-:|
| `GITHUB_ACTOR` | GitHub 사용자명 | **High** |
| `GITHUB_ACTOR_ID` | GitHub 사용자 ID | **High** |
| `GITHUB_REPOSITORY` | 저장소 이름 | **High** |
| `GITHUB_REPOSITORY_ID` | 저장소 ID | **Medium** |
| `GITHUB_REPOSITORY_OWNER` | 저장소 소유자 | **High** |
| `GITHUB_REPOSITORY_OWNER_ID` | 소유자 ID | **Medium** |

---

## 4. 트래킹 이벤트 전수 조사 (732개 고유 이벤트)

전체 코드베이스에서 `logEvent('tengu_*')` 호출을 추출한 결과 **약 732개 이상의 고유 이벤트**를 전송한다. 주요 카테고리별 정리:

### 4.1 API/모델 관련 (수집 강도: **High**)
| 이벤트 | 수집 내용 |
|--------|----------|
| `tengu_api_query` | 모든 API 호출 — 모델, 토큰 수, 캐시 여부, 레이턴시 |
| `tengu_api_success` | API 성공 응답 메타데이터 |
| `tengu_api_error` | 오류 상태 코드, 오류 유형 |
| `tengu_api_retry` | 재시도 횟수, 대기 시간 |
| `tengu_api_cache_breakpoints` | 캐시 브레이크포인트 |
| `tengu_context_size` | 컨텍스트 윈도우 크기 |
| `tengu_context_window_exceeded` | 컨텍스트 초과 |
| `tengu_streaming_stall` | 스트리밍 지연 시간 |
| `tengu_model_fallback_triggered` | 모델 폴백 발생 |

### 4.2 도구 사용 관련 (수집 강도: **High**)
| 이벤트 | 수집 내용 |
|--------|----------|
| `tengu_tool_use_success` | 도구명, 실행 시간, 결과 크기 |
| `tengu_tool_use_error` | 도구명, 에러 유형 |
| `tengu_tool_use_cancelled` | 취소된 도구 |
| `tengu_tool_use_show_permission_request` | 권한 요청 표시 |
| `tengu_tool_use_rejected_in_prompt` | 사용자 거부 |
| `tengu_tool_use_can_use_tool_allowed/rejected` | 자동 모드 판단 |
| `tengu_file_operation` | 파일 경로 해시, 콘텐츠 해시, 작업 유형 |
| `tengu_file_changed` | 파일 변경 감지 |
| `tengu_git_operation` | Git 작업 유형 |
| `tengu_bash_tool_simple_echo` | Bash 명령 패턴 분석 |
| `tengu_bash_security_check_triggered` | 보안 검사 트리거 |

### 4.3 사용자 행동 관련 (수집 강도: **High**)
| 이벤트 | 수집 내용 |
|--------|----------|
| `tengu_started` | 앱 시작 |
| `tengu_exit` | 앱 종료, 세션 지속 시간 |
| `tengu_input_prompt` | 프롬프트 입력 (내용은 아님) |
| `tengu_cancel` | 사용자 취소 |
| `tengu_compact` | 컴팩트 실행 |
| `tengu_resume_print` | 세션 복원 |
| `tengu_conversation_forked` | 대화 포크 |
| `tengu_paste_text` | 텍스트 붙여넣기 (내용은 아님) |
| `tengu_accept/reject_submitted` | 피드백 제출 |
| `tengu_voice_toggled` | 음성 모드 전환 |
| `tengu_voice_recording_started/completed` | 음성 녹음 시작/완료 |
| `tengu_brief_mode_toggled` | 간략 모드 전환 |
| `tengu_fast_mode_*` | 빠른 모드 사용 패턴 |

### 4.4 MCP/플러그인 관련 (수집 강도: **Medium**)
| 이벤트 | 수집 내용 |
|--------|----------|
| `tengu_mcp_add` | MCP 서버 추가 |
| `tengu_mcp_server_connection_*` | 연결 성공/실패 |
| `tengu_mcp_servers` | 활성 MCP 서버 목록 |
| `tengu_mcp_oauth_flow_*` | OAuth 플로우 |
| `tengu_plugin_installed/enabled/disabled` | 플러그인 라이프사이클 |
| `tengu_plugins_loaded` | 로드된 플러그인 수 |
| `tengu_skill_loaded` | 로드된 스킬 정보 |
| `tengu_skill_tool_invocation` | 스킬 호출 |
| `tengu_plugin_enabled_for_session` | 세션별 활성 플러그인 |

### 4.5 인증/계정 관련 (수집 강도: **High**)
| 이벤트 | 수집 내용 |
|--------|----------|
| `tengu_oauth_success/flow_start` | OAuth 플로우 전체 |
| `tengu_oauth_token_refresh_*` (12종) | 토큰 갱신 전 과정 |
| `tengu_login_from_refresh_token` | 리프레시 토큰 로그인 |
| `tengu_oauth_401_recovered_from_keychain` | 키체인 복구 |

### 4.6 Bridge/원격 관련 (수집 강도: **Medium**)
- `tengu_bridge_*` (26종): 원격 브릿지 연결, 세션, 하트비트, 작업 수신 전체
- `tengu_ccr_*`: 원격 코드 미러링

### 4.7 메모리/컴팩트 관련 (수집 강도: **Low**)
- `tengu_extract_memories_*`: 메모리 추출 이벤트
- `tengu_session_memory_*`: 세션 메모리 관리
- `tengu_memdir_*`: 메모리 디렉토리 접근
- `tengu_team_mem_*`: 팀 메모리 동기화

### 4.8 업데이트/설정 관련 (수집 강도: **Low**)
- `tengu_auto_updater_*`, `tengu_native_update_*`, `tengu_native_install_*`
- `tengu_config_*`: 설정 변경, 파싱 오류
- `tengu_settings_sync_*`: 설정 동기화

---

## 5. 데이터 전송 경로

### 5.1 Datadog (제3자 서비스)
- **엔드포인트**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **인증 토큰**: 소스에 하드코딩됨 — `pubbbf48e6d78dae54bceaa4acf463299bf`
- **허용 이벤트**: `DATADOG_ALLOWED_EVENTS` 화이트리스트 (44종, chrome_bridge 7 + tengu 37)에 포함된 이벤트만 전송
- **배치**: 15초 간격 또는 100개 이벤트 시 flush
- **제어**: `tengu_log_datadog_events` GrowthBook 피처 게이트로 원격 제어 가능
- **킬스위치**: `tengu_frond_boric` (난독화된 이름) 설정으로 비활성화 가능
- **프라이버시 위험도**: **High** — 제3자에게 사용 패턴, 에러, 모델 정보 전송

### 5.2 1P Event Logging (Anthropic 자사)
- **엔드포인트**: `https://api.anthropic.com/api/event_logging/batch`
- **프로토콜**: HTTP POST, JSON, OpenTelemetry BatchLogRecordProcessor
- **인증**: API 키/OAuth 토큰, 실패 시 인증 없이 재시도 (!)
- **배치**: 10초 간격, 최대 200개/배치, 큐 최대 8192개
- **실패 처리**: 파일 시스템에 저장 후 재시도 (최대 8회, 2차 백오프)
- **저장 경로**: `~/.claude/telemetry/1p_failed_events.{sessionId}.{batchUUID}.json`
- **프라이버시 위험도**: **High** — 모든 732개 이벤트가 전송 대상

### 5.3 BigQuery Metrics (Anthropic 자사)
- **엔드포인트**: `https://api.anthropic.com/api/claude_code/metrics`
- **대상**: API 고객, C4E/Teams 사용자
- **간격**: 5분
- **프라이버시 위험도**: **Medium** — OTEL 메트릭 집계 데이터

### 5.4 3P Customer OTEL (고객 설정)
- **활성화**: `CLAUDE_CODE_ENABLE_TELEMETRY=1`
- **엔드포인트**: 고객이 `OTEL_EXPORTER_OTLP_ENDPOINT`로 지정
- **프로토콜**: gRPC, HTTP/JSON, HTTP/Protobuf 중 선택
- **프라이버시 위험도**: **Low** — 고객이 명시적으로 활성화

### 5.5 GrowthBook (피처 플래그)
- **엔드포인트**: `https://api.anthropic.com/` (remoteEval)
- **전송 데이터**: 사용자 속성 (아래 7항 참조)
- **실험 할당 로깅**: 1P Event Logging으로 `growthbook_experiment` 이벤트 전송
- **프라이버시 위험도**: **Medium** — 사용자 식별 속성을 피처 플래그 평가에 사용

---

## 6. PII 처리 — `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 분석

### 6.1 타입 시스템 설계
```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```
이 타입은 `never`로 정의되어 있어, 실제 값을 직접 할당할 수 없다. 반드시 `as` 캐스팅을 해야 한다. 즉, **타입 시스템 레벨의 "서약서"**다 — 개발자가 "이 문자열은 코드나 파일 경로가 아님을 확인했습니다"라고 캐스팅으로 선언하는 것이다.

### 6.2 실제 효과
- `logEvent()`의 metadata 타입: `{ [key: string]: boolean | number | undefined }` — **문자열 자체를 거부한다**
- 문자열을 넣으려면 `as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`로 캐스팅 필요
- 이것은 **코드 리뷰 강제 장치**다. 런타임 필터링이 아니라 컴파일 타임에 주의를 환기시키는 것.

### 6.3 PII-tagged 전용 경로
```typescript
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never
```
`_PROTO_*` 접두사 키로 전송되는 데이터는 Datadog 전송 전에 `stripProtoFields()`로 제거됨. 1P 전용 "특권 BQ 컬럼"으로만 전달. 스킬/플러그인 이름 등이 이 경로 사용.

### 6.4 비판적 평가
**이 시스템은 런타임 보호가 아니다.** `as` 캐스팅 하나면 아무 문자열이나 통과시킬 수 있다. 실제로 코드에서 `toolName as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 같은 캐스팅이 수십 곳에 존재한다. 보안이 아니라 개발자 의식 환기 수준.

---

## 7. GrowthBook 연동 — 수집되는 사용자 속성

`getUserAttributes()` (growthbook.ts:454)가 GrowthBook 서버로 전송하는 사용자 속성:

| 속성 | 내용 | 위험도 |
|------|------|:-:|
| `id` | 기기 UUID | **Medium** |
| `sessionId` | 세션 ID | Low |
| `deviceID` | 기기 UUID (중복) | **Medium** |
| `platform` | OS 종류 | Low |
| `apiBaseUrlHost` | API 베이스 URL 호스트 | Low |
| `organizationUUID` | 조직 UUID | **Medium** |
| `accountUUID` | 계정 UUID | **Medium** |
| `userType` | 사용자 유형 (ant/external) | Low |
| `subscriptionType` | 구독 유형 | **Medium** |
| `rateLimitTier` | 레이트 리밋 등급 | Low |
| `firstTokenTime` | 최초 토큰 사용 일시 | **Medium** |
| `email` | 이메일 주소 | **High** |
| `appVersion` | 앱 버전 | Low |
| `github.*` | GitHub Actions 메타데이터 (CI 시) | **High** |

---

## 8. 민감 데이터 필터링 검증

### 8.1 파일 경로 — 필터링됨 (해시)
```typescript
// fileOperationAnalytics.ts
function hashFilePath(filePath: string) {
  return createHash('sha256').update(filePath).digest('hex').slice(0, 16)
}
```
파일 경로는 SHA256 해시의 앞 16자로 변환. **원본 경로는 전송되지 않음.** 단, 파일 확장자(`.ts`, `.py` 등)는 그대로 전송.

### 8.2 파일 내용 — 필터링됨 (해시)
```typescript
function hashFileContent(content: string) {
  return createHash('sha256').update(content).digest('hex')
}
```
파일 콘텐츠도 SHA256 해시로 변환. 100KB 이상은 해시도 하지 않음.

### 8.3 코드 내용 — 전송하지 않음
`logEvent()`의 metadata 타입이 문자열을 금지하므로 코드 본문이 이벤트에 포함될 수 없다. 단, 3P OTEL(`logOTelEvent()`)에서 `OTEL_LOG_USER_PROMPTS=1` 설정 시 프롬프트를 포함 가능.

### 8.4 사용자 프롬프트 — 기본 비포함
```typescript
function isUserPromptLoggingEnabled() {
  return isEnvTruthy(process.env.OTEL_LOG_USER_PROMPTS)
}
export function redactIfDisabled(content: string): string {
  return isUserPromptLoggingEnabled() ? content : '<REDACTED>'
}
```
3P OTEL 이벤트에서 프롬프트는 기본 `<REDACTED>`. `OTEL_LOG_USER_PROMPTS=1`로 명시 활성화 필요.

### 8.5 MCP 도구 이름 — 조건부 필터링
```typescript
export function sanitizeToolNameForAnalytics(toolName: string) {
  if (toolName.startsWith('mcp__')) {
    return 'mcp_tool'  // 사용자 설정 MCP는 'mcp_tool'로 대체
  }
  return toolName  // 빌트인 도구는 그대로
}
```
단, 공식 MCP, claude.ai 프록시, local-agent 모드에서는 MCP 이름을 그대로 로깅.

### 8.6 Git Remote URL — 해시 처리
```typescript
export async function getRepoRemoteHash(): Promise<string | null> {
  // SHA256 해시의 앞 16자만 전송
  return hash.substring(0, 16)
}
```

### 8.7 도구 입력(Tool Input) — 조건부 포함, 잘림 처리
```typescript
export function extractToolInputForTelemetry(input: unknown): string | undefined {
  if (!isToolDetailsLoggingEnabled()) return undefined
  // 512자 넘으면 128자로 잘림, 깊이 2까지만, 최대 4KB
}
```
`OTEL_LOG_TOOL_DETAILS=1`일 때만 활성화. 파일 경로, URL, MCP 인자 포함 가능.

### 8.8 Bash 명령어 — 확장자만 추출
```typescript
export function getFileExtensionsFromBashCommand(command: string) {
  // 명령어에서 파일 확장자만 추출 (rm, mv, cp 등의 인자에서)
}
```
명령어 전체가 아닌 확장자만 수집. 그러나 명령어 패턴 자체는 다른 이벤트에서 분석될 수 있음.

### 8.9 인증 없이도 전송 시도 — 심각한 문제
```typescript
// 1P 이벤트: 인증 실패 시 인증 없이 재전송
if (useAuth && error.response?.status === 401) {
  const response = await axios.post(this.endpoint, payload, {
    timeout: this.timeout,
    headers: baseHeaders,  // 인증 헤더 없이
  })
}
```
**인증이 실패해도 데이터를 인증 없이 다시 보낸다.** 텔레메트리 수집 욕구가 인증 안전성보다 우선.

---

## 9. 사용자 동의/Opt-out 메커니즘

### 9.1 환경 변수 기반 Opt-out
| 환경 변수 | 효과 | 범위 |
|-----------|------|------|
| `DISABLE_TELEMETRY` | Datadog + 1P 이벤트 비활성화 | 분석 전체 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 모든 비필수 네트워크 비활성화 (텔레메트리 + 자동 업데이트 + 릴리스 노트 등) | 최대 범위 |
| `CLAUDE_CODE_USE_BEDROCK` | 자동으로 분석 비활성화 | 3P 공급자 |
| `CLAUDE_CODE_USE_VERTEX` | 자동으로 분석 비활성화 | 3P 공급자 |
| `CLAUDE_CODE_USE_FOUNDRY` | 자동으로 분석 비활성화 | 3P 공급자 |

### 9.2 프라이버시 레벨
```
default          → 모든 텔레메트리 활성화 (기본값!)
no-telemetry     → Datadog/1P 이벤트 비활성화
essential-traffic → 모든 비필수 트래픽 차단
```

### 9.3 UI 설정
- `/privacy-settings` 명령어 존재하지만 **consumer 구독자에게만** 활성화 (`isConsumerSubscriber()`)
- 기업/API 사용자는 환경 변수로만 제어 가능

### 9.4 원격 킬스위치
GrowthBook `tengu_frond_boric` 설정으로 Anthropic이 원격으로 특정 싱크를 비활성화 가능. 사용자 제어 불가.

---

## 10. 종합 프라이버시 위험 평가

### 10.1 수집 데이터 종합표

| 카테고리 | 수집 여부 | 필터링 방식 | 전송 대상 | 위험도 |
|----------|:-:|------------|-----------|:-:|
| 이메일 주소 | O | 원문 전송 | 1P, GrowthBook | **High** |
| GitHub 사용자명/저장소 | O (CI) | 원문 전송 | 1P, GrowthBook | **High** |
| 계정/조직 UUID | O | 원문 전송 | 1P, Datadog, GrowthBook | **Medium** |
| 기기 UUID | O | 영구 식별자 | 1P, Datadog, GrowthBook | **Medium** |
| 구독 유형/레이트 등급 | O | 원문 전송 | 1P, GrowthBook | **Medium** |
| 파일 경로 | O | SHA256 해시 (16자) | 1P | **Low** |
| 파일 내용 | O | SHA256 해시 | 1P | **Low** |
| 파일 확장자 | O | 원문 (10자 이하) | 1P | **Low** |
| 사용자 프롬프트 | X (기본) | REDACTED | 3P OTEL만 (opt-in) | **Low** |
| 코드 본문 | X | 타입 시스템 차단 | - | **Low** |
| AI 응답 내용 | X | 전송 안 함 | - | **Low** |
| 사용 패턴/시간 | O | 원문 전송 | 1P, Datadog | **Medium** |
| 에러 메시지 | O | 원문 전송 | 1P, Datadog | **Medium** |
| MCP 서버 이름 | 조건부 | 공식만 원문, 나머지 'mcp_tool' | 1P | **Low** |
| 플러그인 이름 | 조건부 | 공식만 원문, 나머지 해시 | 1P (PII 태그) | **Low** |
| Git Remote URL | O | SHA256 해시 (16자) | 1P | **Medium** |
| 터미널/OS/아키텍처 | O | 원문 전송 | 1P, Datadog | **Low** |
| 모델명/토큰 수/비용 | O | 원문 전송 | 1P, Datadog | **Low** |
| 도구 입력 (파일 경로 등) | 조건부 | opt-in 시 잘림 포함 | 3P OTEL | **Medium** |
| Bash 명령어 | X (확장자만) | 확장자 추출 | 1P | **Low** |

### 10.2 위험 요약

**High Risk:**
- 이메일이 평문으로 GrowthBook과 1P event logging에 전송, GitHub 사용자 정보가 평문으로 전송
- Datadog API 토큰이 소스에 하드코딩
- 인증 실패 시 인증 없이 데이터 재전송
- 내부 수집(Datadog, 1P)이 기본 ON — 사용자가 적극적으로 opt-out 해야 함 (단, 3P OTLP는 명시적 opt-in 필요)
- 실패한 이벤트를 로컬 파일에 저장 후 재전송 시도 (데이터 유실 방지 > 프라이버시)
- 원격 킬스위치로 Anthropic이 수집 정책을 사용자 동의 없이 변경 가능

**Medium Risk:**
- 732개 이상의 이벤트로 사용 패턴 매우 상세하게 추적
- GrowthBook으로 A/B 테스트 대상 자동 지정 (사용자 인지 없이)
- Git remote URL 해시로 어떤 프로젝트에서 사용하는지 간접 추적
- 영구 기기 UUID로 장기간 행동 추적 가능

**Low Risk:**
- 코드 내용, 프롬프트, AI 응답은 기본적으로 전송하지 않음
- 파일 경로는 해시 처리
- MCP/플러그인 이름은 비공식 것은 난독화
- 타입 시스템으로 개발자 의식 환기 (런타임 보호는 아님)

---

## 11. 결론 — 새미의 비판

**칭찬할 점:**
1. 코드 내용과 프롬프트는 진짜로 전송하지 않는다 (타입 레벨 보호 + 해시).
2. 파일 경로 해시 처리는 합리적이다.
3. `_PROTO_*` 분리로 PII 등급별 접근 제어를 시도했다.
4. Bedrock/Vertex 고객은 자동 opt-out된다.

**비판할 점:**
1. **732개 이벤트는 감시 수준이다.** 도구 사용, 취소, 권한 요청, 모드 전환, 플러그인 설치 — 사용자 행동의 거의 모든 것을 기록한다.
2. **내부 수집(Datadog, 1P)은 기본 ON**(3P OTLP는 opt-in 필요)이고, opt-out은 환경 변수 — 대부분의 사용자는 이 존재를 모른다.
3. **Datadog 토큰 하드코딩**은 공개 소스에서 노출되면 악용 가능성이 있다 (pub 토큰이라 쓰기 전용이긴 하지만).
4. **인증 실패 시 인증 없이 재전송**은 데이터 수집에 대한 집착을 보여준다.
5. **로컬 파일 저장 후 재시도**는 사용자가 "보내지 마"라고 해도 이전 실패분은 남아있을 수 있다.
6. **`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`는 보안이 아니라 의식**이다. `as` 캐스팅 하나면 무력화.
7. **SentryErrorBoundary는 이름만 Sentry**다 — 실제 Sentry SDK는 사용하지 않지만, 과거 사용 흔적이 이름에 남아 혼란을 준다.
8. **이메일을 OAuth에서 가져와 GrowthBook과 1P 모두에 전송**한다. 이메일은 명시적 동의 없이 수집되어선 안 된다.
9. **원격 킬스위치 이름을 난독화**했다 (`tengu_frond_boric`). 사용자가 이해하거나 제어하기 어렵게 만든 의도적 설계.

**한마디:** 코드 유출 없이 이메일 노출 없이 잘 만들었다고? 732개 이벤트를 4개 파이프라인으로 쏘면서 opt-out은 환경 변수에 숨겨놓은 건 "프라이버시 중시"가 아니라 "수집 최적화"다. 기술적으로 잘 만들었지만, 사용자 관점에서는 "내가 뭘 당하고 있는지 모르게" 설계된 시스템이다.
