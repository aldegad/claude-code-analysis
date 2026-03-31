# 06. 보안 및 테스트 분석

> 분석 대상: claude-code (Anthropic Claude Code CLI)
> 분석일: 2026-03-31
> 분석자: 새미 (개발팀 비평가)

---

## 1. 보안 아키텍처 개요

Claude Code의 보안은 4개 레이어로 구성된다:

```
Layer 4: 조직 정책 (policyLimits, remoteManagedSettings)
Layer 3: 도구 권한 시스템 (permissions/, sandbox/)
Layer 2: 인증/인가 (OAuth, API Key, mTLS)
Layer 1: 입력 검증 (Zod, bash 보안, path traversal 방지)
```

---

## 2. 인증 시스템

### 2.1 인증 방식 (src/utils/auth.ts, 2,003줄)

| 방식 | 환경변수/설정 | 용도 |
|------|-------------|------|
| OAuth 2.0 | 내부 토큰 관리 | Claude.ai 구독자 |
| API Key | `ANTHROPIC_API_KEY` | 직접 API 접근 |
| API Key Helper | `apiKeyHelper` 설정 | 외부 키 관리자 |
| AWS STS | AWS 자격 증명 | Bedrock |
| GCP OAuth | Google Auth Library | Vertex AI |
| Azure AD | DefaultAzureCredential | Foundry |
| File Descriptor | `--oauth-token-fd`, `--api-key-fd` | 프로세스 간 전달 |

### 2.2 OAuth 흐름 (src/services/oauth/)

```typescript
// PKCE 기반 OAuth 2.0
function buildAuthUrl({ codeChallenge, state, port, ... }) {
  authUrl.searchParams.append('code_challenge', codeChallenge)
  authUrl.searchParams.append('code_challenge_method', 'S256')
  // ...
}
```

**보안 평가**:
- PKCE(Proof Key for Code Exchange) 사용 -- Authorization Code Interception 공격 방지
- `state` 파라미터로 CSRF 방지
- 두 가지 인증 엔드포인트 지원: Claude.ai와 Console

### 2.3 토큰 관리

```typescript
// src/utils/auth.ts
export function isManagedOAuthContext(): boolean {
  return (
    isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
    process.env.CLAUDE_CODE_ENTRYPOINT === 'claude-desktop'
  )
}
```

**핵심 보안 로직**: 관리형 OAuth 컨텍스트(CCR, Claude Desktop)에서는 사용자의 로컬 API 키 설정을 무시한다. 이는 데스크톱 앱 세션이 사용자의 터미널 CLI 키를 오용하는 것을 방지한다.

### 2.4 토큰 갱신

```typescript
checkAndRefreshOAuthTokenIfNeeded()
refreshGcpCredentialsIfNeeded()
refreshAndGetAwsCredentials()
```

각 프로바이더별 자동 토큰 갱신을 지원한다.

---

## 3. 시크릿 저장소 (src/utils/secureStorage/)

### 3.1 플랫폼별 전략

```typescript
export function getSecureStorage(): SecureStorage {
  if (process.platform === 'darwin') {
    return createFallbackStorage(macOsKeychainStorage, plainTextStorage)
  }
  // TODO: add libsecret support for Linux
  return plainTextStorage
}
```

| 플랫폼 | 주 저장소 | 대체 저장소 |
|--------|----------|-----------|
| macOS | Keychain (security 명령) | ~/.claude/.credentials.json |
| Linux | 없음 (plaintext만) | ~/.claude/.credentials.json |
| Windows | 없음 (plaintext만) | ~/.claude/.credentials.json |

### 3.2 macOS Keychain 구현

```typescript
// 4096바이트 stdin 제한 처리
const SECURITY_STDIN_LINE_LIMIT = 4096 - 64

// stale-while-error 캐싱
if (prev.data !== null) {
  keychainCacheState.cache = { data: prev.data, cachedAt: Date.now() }
  return prev.data
}
```

**보안 비판 1**: Linux에서 `libsecret` 지원이 TODO로 남아있다. Linux 서버에서 Claude Code를 사용하는 경우 API 키가 평문 JSON 파일에 저장된다.

**보안 비판 2**: plainTextStorage에서 파일 권한 설정이 `chmodSync(storagePath, 0o600)` 정도는 있지만, 키 암호화는 없다. macOS 이외의 플랫폼에서는 디스크에 평문 시크릿이 존재한다.

### 3.3 Keychain 프리페치

```typescript
// src/utils/secureStorage/keychainPrefetch.ts
startKeychainPrefetch()  // main.tsx에서 모듈 로드와 병렬로 시작
```

**배울 점**: 키체인 읽기를 모듈 로드와 병렬로 실행하여 시작 시간을 단축한다. `ensureKeychainPrefetchCompleted()`로 필요 시점에 완료를 보장한다.

---

## 4. 도구 권한 시스템 (src/utils/permissions/)

### 4.1 권한 모드

```typescript
// 4가지 권한 모드
type PermissionMode = 'default' | 'plan' | 'bypassPermissions' | 'auto'
```

| 모드 | 동작 |
|------|------|
| `default` | 매 작업마다 사용자 확인 |
| `plan` | 계획 표시 후 한번에 승인 |
| `bypassPermissions` | 자동 승인 (위험) |
| `auto` | ML 분류기 기반 자동 판단 |

### 4.2 권한 규칙

```typescript
type PermissionRuleValue = {
  toolName: string
  ruleContent?: string  // 와일드카드 패턴
}

// 예시 규칙: Bash(git *), FileEdit(/src/*)
```

와일드카드 패턴으로 세밀한 권한 제어가 가능하다:
- `Bash(git *)` -- git 관련 명령만 허용
- `FileEdit(/src/*)` -- src/ 아래 파일만 편집 허용
- `Bash(npm run *)` -- npm 스크립트 실행만 허용

### 4.3 auto 모드 -- ML 분류기

```typescript
// permissions.ts에서 feature gate
const classifierDecisionModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./classifierDecision.js') : null
const autoModeStateModule = feature('TRANSCRIPT_CLASSIFIER')
  ? require('./autoModeState.js') : null
```

auto 모드에서는 ML 분류기가 도구 사용의 안전성을 판단한다:
- `bashClassifier.ts` -- Bash 명령 분류
- `yoloClassifier.ts` -- YOLO(bypass) 모드 분류
- `classifierDecision.ts` -- 분류기 결정 로직

**보안 비판**: auto 모드에서 분류기 실패 시 fallback이 "prompting"(사용자에게 묻기)이다. 분류기가 사용 불가능할 때 자동 거부하는 것이 더 안전할 것이다.

### 4.4 거부 추적 (denialTracking.ts)

```typescript
const DENIAL_LIMITS = { ... }
// 연속 거부 횟수 추적
// 임계값 초과 시 fallback to prompting
```

**배울 점**: 분류기가 연속으로 도구 사용을 거부하면, 임계값 이후 사용자에게 직접 물어보는 fallback 메커니즘이 있다. 이는 분류기 오탐(false positive)으로 인한 무한 거부 루프를 방지한다.

---

## 5. Bash 보안 (src/tools/BashTool/)

### 5.1 보안 체크 체인 (bashSecurity.ts, 2,593줄)

```
Bash 명령 입력
    |
    v
[커맨드 치환 패턴 검사]  -- $(), ``, <(), >()), ${}, ...
    |
    v
[Zsh 위험 명령 차단]  -- zmodload, emulate, sysopen, ztcp, ...
    |
    v
[Heredoc-in-substitution 차단]  -- $(... <<
    |
    v
[Tree-sitter 기반 AST 분석]  -- 구조적 명령 파싱
    |
    v
[jq 시스템 함수 차단]  -- jq의 system() 함수
    |
    v
[PowerShell 주석 구문 차단]  -- <# 방어적 차단
    |
    v
[권한 확인]  -- checkPermissions()
    |
    v
[샌드박스 실행]  -- @anthropic-ai/sandbox-runtime
```

### 5.2 위험 패턴 탐지 (bashSecurity.ts)

```typescript
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ...
]

const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload', 'emulate', 'sysopen', 'sysread', 'syswrite',
  'sysseek', 'zpty', 'ztcp', 'zsocket', 'mapfile',
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', ...
])
```

**보안 평가**: Zsh 특유의 보안 위협(모듈 로딩, 프로세스 치환, 소켓 생성)을 세밀하게 차단한다. 특히 `zmodload`는 게이트웨이 명령으로, 이것 하나가 여러 위험 모듈을 로드할 수 있으므로 최우선 차단 대상이다.

**방어 깊이**: PowerShell 주석 구문 `<#`도 차단하는데, "현재는 PowerShell에서 실행하지 않지만 미래 변경에 대비"한다는 주석이 있다. Defense in depth 원칙의 좋은 예시.

### 5.3 위험 허용 규칙 차단 (dangerousPatterns.ts)

```typescript
export const DANGEROUS_BASH_PATTERNS: readonly string[] = [
  // 인터프리터
  'python', 'python3', 'node', 'deno', 'ruby', 'perl', 'php', 'lua',
  // 패키지 러너
  'npx', 'bunx', 'npm run', 'yarn run',
  // 셸
  'bash', 'sh', 'zsh', 'fish',
  // eval/exec
  'eval', 'exec', 'env', 'xargs', 'sudo',
  // 원격
  'ssh',
]
```

`Bash(python:*)` 같은 허용 규칙은 임의 코드 실행을 허용하므로, auto 모드 진입 시 이런 위험 규칙을 자동으로 제거한다.

### 5.4 샌드박스 시스템 (src/utils/sandbox/)

```typescript
// @anthropic-ai/sandbox-runtime 외부 패키지 래핑
import {
  SandboxManager as BaseSandboxManager,
  SandboxRuntimeConfigSchema,
  SandboxViolationStore,
} from '@anthropic-ai/sandbox-runtime'
```

**구성**:
- 파일시스템 읽기/쓰기 제한
- 네트워크 접근 제한 (호스트 패턴 기반)
- 위반 이벤트 추적
- 사용자 설정의 `excludedCommands`로 추가 제한

**주의**: `excludedCommands`는 "사용자 편의 기능이지 보안 경계가 아니다"라고 명시적으로 주석처리되어 있다.

```typescript
// NOTE: excludedCommands is a user-facing convenience feature, not a security boundary.
// It is not a security bug to be able to bypass excludedCommands
```

---

## 6. MCP 프로토콜 보안

### 6.1 MCP 서버 인증 (src/services/mcp/auth.ts, 2,466줄)

- OAuth 2.0 기반 MCP 서버 인증
- XAA(Cross-App Access) 지원 -- IdP 연동
- `authServerMetadataUrl`은 HTTPS만 허용

```typescript
authServerMetadataUrl: z.string().url()
  .startsWith('https://', {
    message: 'authServerMetadataUrl must use https://',
  })
```

### 6.2 경로 순회 방지 (mcp-server/src/server.ts)

```typescript
function safePath(relPath: string): string | null {
  const resolved = path.resolve(SRC_ROOT, relPath)
  if (!resolved.startsWith(SRC_ROOT)) return null
  return resolved
}
```

### 6.3 LRU 캐시 제한

```typescript
// MCP 서버 모드에서 메모리 폭주 방지
const READ_FILE_STATE_CACHE_SIZE = 100  // 최대 100개 파일
// + 25MB 크기 제한
```

---

## 7. 민감정보 처리

### 7.1 팀 메모리 시크릿 가드

```typescript
// FileEditTool에서 팀 메모리 파일 편집 시
const secretError = checkTeamMemSecrets(fullFilePath, new_string)
if (secretError) {
  return { result: false, message: secretError, errorCode: 0 }
}
```

**배울 점**: 팀 공유 메모리 파일에 시크릿이 포함되는 것을 도구 레벨에서 차단한다.

### 7.2 분석 메타데이터 타입 가드

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = { ... }

logEvent('event_name', metadata as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS)
```

코드나 파일경로가 분석 이벤트에 전송되는 것을 타입 레벨에서 방지한다.

### 7.3 SSRF 가드 (src/utils/hooks/ssrfGuard.ts)

훅 실행 시 SSRF(Server-Side Request Forgery) 공격을 방지하는 가드가 있다.

### 7.4 진단 로깅 PII 방지

```typescript
logForDiagnosticsNoPII('info', 'init_completed', {
  duration_ms: Date.now() - initStartTime,
})
```

`NoPII` 접미사가 있는 로깅 함수는 개인식별정보를 포함하지 않는 메트릭만 기록한다.

---

## 8. 정책 관리

### 8.1 조직 정책 (src/services/policyLimits/)

```typescript
isPolicyAllowed('allow_remote_control')
isPolicyAllowed('allow_bridge_mode')
```

엔터프라이즈 조직이 기능별 접근 제어를 설정할 수 있다.

### 8.2 원격 관리 설정 (src/services/remoteManagedSettings/)

```typescript
if (isEligibleForRemoteManagedSettings()) {
  initializeRemoteManagedSettingsLoadingPromise()
}
```

엔터프라이즈 사용자의 설정을 원격에서 관리/동기화한다.

### 8.3 MDM 지원 (src/utils/settings/mdm/)

macOS plutil, Windows Registry를 통한 MDM(Mobile Device Management) 설정 지원.

---

## 9. 테스트 분석

### 9.1 테스트 파일 부재

```bash
$ find src -name "*.test.*" -o -name "*.spec.*"
# (결과 없음)
```

**충격적 사실**: src/ 디렉토리에 테스트 파일이 하나도 없다.

### 9.2 테스트가 없는 이유 추정

1. **빌드 시스템**: `package.json`에 `test` 스크립트가 없다
2. **별도 저장소**: 테스트가 별도 저장소나 CI 파이프라인에 존재할 가능성
3. **유출된 소스**: 이 프로젝트는 description에 "leaked source"로 표기되어 있어, 테스트 디렉토리가 포함되지 않았을 수 있다
4. **타입 체크 의존**: `typecheck` 스크립트와 `strict: true` 설정으로 컴파일 타임 검증에 의존

### 9.3 테스트 관련 코드 흔적

```typescript
// src/tools/testing/TestingPermissionTool.ts
export const TestingPermissionTool = ...

// cost-tracker.ts
export { resetStateForTests }

// context.ts
if (process.env.NODE_ENV === 'test') {
  return null  // 테스트 시 순환 방지
}
```

**평가**: `TestingPermissionTool`, `resetStateForTests()`, `NODE_ENV === 'test'` 가드 등의 흔적으로 볼 때, 테스트 코드가 존재하지만 이 소스에 포함되지 않은 것으로 보인다.

### 9.4 코드 품질 검증 수단

테스트 대신 사용하는 검증 수단:
1. **TypeScript strict mode** -- 컴파일 타임 타입 체크
2. **Biome linter** -- 코드 품질 규칙, cognitive complexity 경고
3. **Zod 스키마** -- 런타임 입력 검증
4. **`DeepImmutable`** -- 상태 불변성 타입 강제
5. **React Compiler** -- 렌더링 최적화 자동 검증

---

## 10. 보안 취약점 종합 평가

### 높은 위험도

| 항목 | 설명 | 영향 |
|------|------|------|
| Linux/Windows 평문 시크릿 | API 키가 `~/.claude/.credentials.json`에 평문 저장 | 디스크 접근 가능한 공격자가 API 키 탈취 가능 |
| `noExplicitAny: "off"` | 타입 안전성 약화 | 보안 관련 코드에서 런타임 에러 가능성 |

### 중간 위험도

| 항목 | 설명 | 영향 |
|------|------|------|
| auto 모드 분류기 fallback | 분류기 실패 시 사용자에게 물어봄 (거부 아님) | 비대화형 모드에서 예기치 않은 승인 가능 |
| `excludedCommands` 비보안 경계 | 우회 가능한 편의 기능 | 사용자가 보안 경계로 오인 가능 |
| 환경변수 기반 보안 설정 | `CLAUDE_CODE_COORDINATOR_MODE` 등 | 환경변수 조작으로 보안 모드 변경 가능 |

### 낮은 위험도

| 항목 | 설명 | 영향 |
|------|------|------|
| Keychain 4096바이트 제한 | 매우 긴 자격 증명 저장 불가 | 특수 경우에만 해당 |
| MCP 서버 100개 파일 캐시 | 캐시 미스 시 디스크 I/O 증가 | 성능 문제 |

---

## 11. 보안 강점 종합

1. **다층 방어 (Defense in Depth)**: Bash 보안만 해도 5단계 이상의 검증 레이어
2. **Zsh 공격 벡터 대응**: 업계에서 드물게 Zsh 특유의 보안 위협을 체계적으로 대응
3. **PKCE OAuth**: 최신 OAuth 보안 표준 준수
4. **PII 방지 타입**: 분석 메타데이터에 코드/파일경로 유출 방지
5. **팀 메모리 시크릿 가드**: 공유 메모리에 시크릿 유입 차단
6. **SSRF 가드**: 훅 실행 시 SSRF 방지
7. **MDM/정책 지원**: 엔터프라이즈 보안 정책 원격 관리
8. **path traversal 방지**: `safePath()` + `SRC_ROOT` 검증
9. **stale-while-error**: 키체인 실패 시에도 인증 상태 유지

---

## 12. 종합 권고사항

### 보안
1. **Linux libsecret 지원 추가**: 평문 시크릿 저장 해소
2. **`noExplicitAny` 점진적 활성화**: 최소한 보안 관련 코드에서는 `any` 금지
3. **auto 모드 fail-closed**: 분류기 실패 시 기본 거부로 변경
4. **환경변수 보안 모드**: 보안 관련 환경변수에 서명/검증 추가

### 테스트
1. **테스트 프레임워크 설정**: Bun 내장 테스트 러너 또는 Vitest 도입
2. **보안 관련 유닛 테스트 우선**: bashSecurity, permissions, auth 모듈
3. **통합 테스트**: 도구 실행 흐름, 권한 체크 파이프라인
4. **프로퍼티 기반 테스트**: Bash 보안 패턴 매칭에 fuzzing 적용

### 코드 품질
1. **거대 파일 분리**: 5,000줄 이상 파일 6개를 모듈별로 분리
2. **순환 의존성 해소**: 의존성 역전(DI) 패턴으로 구조 개선
3. **통합 피처 게이팅**: `USER_TYPE` 체크와 `feature()` 플래그 통합
