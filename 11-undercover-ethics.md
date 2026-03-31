# 11. Undercover Mode 윤리적 심층 분석 보고서

> **새미 (개발팀 비평가)** | 2026-03-31
> 대상: Claude Code CLI 소스코드 (`src/utils/undercover.ts` 외 12개 파일)
> 의심 시나리오: "Anthropic 직원이 AI 흔적을 지우고 오픈소스 기여 -> 코드 흡수"

---

## 1. Undercover Mode 완전 해부

### 1.1 핵심 파일 구조

| 파일 | 역할 |
|------|------|
| `src/utils/undercover.ts` | 핵심 로직 (90줄). 활성화 판단, 지시문 생성, 알림 체크 |
| `src/utils/attribution.ts` | 커밋/PR attribution 텍스트 생성. undercover 시 빈 문자열 반환 |
| `src/utils/commitAttribution.ts` | 내부 리포 allowlist (INTERNAL_MODEL_REPOS), 리포 분류 캐시 |
| `src/constants/prompts.ts` | 시스템 프롬프트에서 모델 정보 억제 (5개 지점) |
| `src/tools/BashTool/prompt.ts` | Bash 도구 프롬프트에 undercover 지시문 주입 |
| `src/commands/commit.ts` | `/commit` 명령에 undercover 지시문 주입 |
| `src/commands/commit-push-pr.ts` | `/commit-push-pr` 명령에 undercover 지시문 + 내부 리뷰어 제거 |
| `src/components/PromptInput/PromptInputFooter.tsx` | UI 하단에 "undercover" 라벨 표시 코드가 존재하지만, 현재 빌드에서는 상수 폴딩으로 비활성화 |
| `src/screens/REPL.tsx` | 최초 활성화 시 알림 다이얼로그 표시 |

### 1.2 활성화 조건 - 정확한 로직

```typescript
// src/utils/undercover.ts:28-36
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {        // 빌드타임 상수. 외부 빌드에서는 false로 DCE됨
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true   // 강제 ON
    return getRepoClassCached() !== 'internal'   // 자동: 내부 리포가 아니면 ON
  }
  return false  // 외부 사용자는 항상 false
}
```

**핵심 발견:**
- `USER_TYPE === 'ant'`는 **빌드타임 상수** (`--define`). npm 배포 빌드에서는 `"external"`로 치환됨
- 외부 빌드에서는 번들러가 이 분기를 **물리적으로 제거** (Dead Code Elimination)
- 따라서 이 기능은 **Anthropic 직원 빌드에서만 존재**한다

**활성화 흐름:**
1. `setup.ts`에서 프로세스 시작 시 `isInternalModelRepo()`를 호출하여 리포 분류 캐시를 초기화
2. git remote URL을 `INTERNAL_MODEL_REPOS` allowlist와 대조
3. allowlist에 없으면 `repoClassCache = 'external'` -> undercover ON
4. allowlist에 있으면 `repoClassCache = 'internal'` -> undercover OFF, 시스템 프롬프트 캐시 무효화

### 1.3 비활성화 가능 여부 - 확정

**강제 OFF 옵션은 없지만, allowlist에 등록된 내부 repo에서는 자동으로 비활성화된다.**

코드 주석에 명시적으로 적혀 있다:
```
*   - There is NO force-OFF. This guards against model codename leaks -- if
*     we're not confident we're in an internal repo, we stay undercover.
```

유일한 OFF 조건: `getRepoClassCached() === 'internal'` -- 즉, git remote가 INTERNAL_MODEL_REPOS allowlist에 있을 때뿐이다.

allowlist는 하드코딩된 44개 엔트리 (SSH/HTTPS 쌍 = 22개 고유 리포):
- `anthropics/claude-cli-internal`
- `anthropics/anthropic`
- `anthropics/apps`
- `anthropics/casino`
- `anthropics/dbt`
- `anthropics/dotfiles`
- `anthropics/terraform-config`
- `anthropics/hex-export`
- `anthropics/feedback-v2`
- `anthropics/labs`
- `anthropics/argo-rollouts`
- `anthropics/starling-configs`
- `anthropics/ts-tools`
- `anthropics/ts-capsules`
- `anthropics/feldspar-testing`
- `anthropics/trellis`
- `anthropics/claude-for-hiring`
- `anthropics/forge-web`
- `anthropics/infra-manifests`
- `anthropics/mycro_manifests`/`mycro_configs`
- `anthropics/mobile-apps`

**주목:** `anthropics/claude-code` (공개 리포)는 allowlist에 **없다**. 즉, Anthropic 직원이 자사 공개 리포에 기여할 때도 undercover가 자동 활성화된다.

코드 주석이 이를 명시적으로 설명:
```
* NOTE: This is intentionally a repo allowlist, not an org-wide check.
* The anthropics and anthropic-experimental orgs contain PUBLIC repos
* (e.g. anthropics/claude-code, anthropic-experimental/sandbox-runtime).
* Undercover mode must stay ON in those to prevent codename leaks.
* Only add repos here that are confirmed PRIVATE.
```

### 1.4 흔적 제거 범위 - 정확한 목록

Undercover Mode가 제거/억제하는 것들:

#### A. 커밋 메시지 (commit.ts, commit-push-pr.ts)
```
NEVER include in commit messages or PR descriptions:
- Internal model codenames (animal names like Capybara, Tengu, etc.)
- Unreleased model version numbers (e.g., opus-4-7, sonnet-4-8)
- Internal repo or project names (e.g., claude-cli-internal, anthropics/...)
- Internal tooling, Slack channels, or short links (e.g., go/cc, #claude-code-...)
- The phrase "Claude Code" or any mention that you are an AI
- Any hint of what model or version you are
- Co-Authored-By lines or any other attribution
```

#### B. Attribution 텍스트 (attribution.ts:53-54)
```typescript
if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    return { commit: '', pr: '' }  // 커밋과 PR 모두 빈 문자열
}
```
- `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` -> 제거
- `Generated with Claude Code` -> 제거

#### C. PR Attribution (attribution.ts:300-301)
```typescript
if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    return ''  // PR 본문에서 완전 제거
}
```

#### D. 시스템 프롬프트 (prompts.ts - 5개 지점)
| 억제 항목 | 정상 시 포함 내용 |
|-----------|-----------------|
| 모델 설명 | "You are powered by the model named Claude Opus 4.6. The exact model ID is claude-opus-4-6." |
| 모델 ID 목록 | "Opus 4.6: 'claude-opus-4-6-20250515', Sonnet 4.6: ..." |
| Claude Code 언급 | "Claude Code is available as a CLI..." |
| Fast 모드 설명 | "Fast mode for Claude Code uses the same model..." |
| 내부 모델 오버라이드 | GrowthBook 기반 모델 전환 관련 시스템 프롬프트 |

**모델명과 제품명은 숨기지만, 시스템 프롬프트는 여전히 'interactive agent'로 정체성을 부여한다.**

#### E. PR 생성 시 추가 제거 (commit-push-pr.ts:49-54)
```typescript
if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    prefix = getUndercoverInstructions() + '\n'
    reviewerArg = ''           // --reviewer anthropics/claude-code 제거
    addReviewerArg = ''        // --add-reviewer anthropics/claude-code 제거
    changelogSection = ''      // CHANGELOG 섹션 제거
    slackStep = ''             // Slack 알림 단계 제거
}
```

#### F. 제거하지 않는 것들
코드를 전수 분석한 결과, 다음은 undercover가 **하지 않는 것**이다:
- 코드 스타일 변환 (AI 스타일 -> 인간 스타일): **없음**
- 커밋 히스토리 조작 (rebase, squash 등): **없음**
- CLAUDE.md / .claude/ 디렉토리 자동 삭제/정리: **없음**
- .gitignore에 Claude 관련 항목 추가: **없음**
- git hook 자동 설치 (pre-push 등): **없음**
- git notes 조작: **없음**

---

## 2. 시나리오 타당성 검증

### 2.1 의심 시나리오 분해

> "Anthropic 직원이 Undercover Mode를 켜고 Claude Code로 오픈소스 프로젝트에 기여 -> Co-Authored-By 태그 등 AI 흔적 제거 -> 해당 오픈소스의 좋은 아이디어/코드를 Claude Code에 흡수"

이 시나리오를 세 단계로 분리하여 검증:

#### 단계 1: "Undercover Mode로 오픈소스 기여" -- 기술적으로 가능

**판정: 사실**

코드가 이 정확한 용도를 위해 설계되었다. `undercover.ts`의 첫 줄이 증거:
```
* Undercover mode -- safety utilities for contributing to public/open-source repos.
```

활성화 조건도 이를 뒷받침한다:
- 내부 리포가 아닌 모든 리포에서 자동 ON
- git remote가 없는 경우(예: `/tmp` 작업)에도 안전 기본값으로 ON
- `anthropics/claude-code` 같은 자사 공개 리포에서도 ON

#### 단계 2: "AI 흔적 제거" -- 사실이나 범위가 제한적

**판정: 부분적 사실**

제거되는 것:
- Co-Authored-By 태그
- "Generated with Claude Code" 문구
- 모델 코드네임/버전
- 내부 리포/도구 언급
- Claude Code 관련 리뷰어 설정

제거되지 않는 것:
- 코드 자체의 패턴 (AI가 생성한 코드의 특성은 남음)
- 파일 타임스탬프 (자연스러운 인간 작업 패턴과 다를 수 있음)
- git author 정보 (직원 이름은 유지)
- PR 토론, 코드 리뷰 내용
- CLAUDE.md나 .claude/ 관련 파일 (수동으로 .gitignore 해야 함)

#### 단계 3: "코드 흡수 메커니즘" -- 코드에 증거 없음

**판정: 코드베이스에 근거 없음**

철저히 검색한 결과:
- 외부 리포의 코드를 수집/저장하는 로직: **발견 안됨**
- 기여한 코드를 다시 가져오는(수확) 메커니즘: **발견 안됨**
- git push 전 자동 정리 hook: **발견 안됨**
- 오픈소스 코드를 Anthropic 서버로 전송하는 기능: **발견 안됨**
- "외부 프로젝트 기여" 관련 특수 로직: undercover 지시문 외 **없음**

`src/utils/privacyLevel.ts`는 텔레메트리 제어를 제공하며, `DISABLE_TELEMETRY` 환경변수로 분석 데이터 전송을 비활성화할 수 있다. 코드 자체를 수집하는 메커니즘은 없다.

### 2.2 시나리오 종합 판정

| 시나리오 요소 | 코드 근거 | 판정 |
|-------------|----------|------|
| Anthropic 직원이 Claude Code로 OSS 기여 가능 | `undercover.ts` 설계 의도 명시 | 사실 |
| AI 사용 흔적 자동 제거 | attribution, prompts 코드에서 확인 | 사실 |
| 비활성화 불가 | "There is NO force-OFF" 명시 | 사실 |
| 코드를 Claude에 흡수하는 메커니즘 | 관련 코드 전무 | **근거 없음** |
| 자동 git hook으로 push 전 정리 | 관련 코드 전무 | **근거 없음** |
| 코드 스타일 변환으로 AI 흔적 은폐 | 관련 코드 전무 | **근거 없음** |

---

## 3. 윤리적 분석

### 3.1 Anthropic의 명시적 의도 (코드에서 추론)

코드 주석과 구조에서 읽히는 의도:

1. **1차 목적**: 내부 정보 유출 방지 (코드네임 Capybara, Tengu 등)
2. **2차 목적**: 직원의 오픈소스 기여 시 AI 사용 사실을 공개하지 않는 것
3. **방어적 설계**: "안전 기본값은 ON" -- 확실하지 않으면 숨긴다

코드는 이것을 "safety utilities"라고 부르지만, 무엇의 안전인지가 핵심 논점이다:
- **Anthropic의 안전**: 영업 비밀(코드네임, 미출시 모델) 보호
- **직원의 안전**: AI 사용 공개에 따른 사회적/직업적 리스크 회피
- 오픈소스 커뮤니티의 안전은 고려 대상이 **아님**

### 3.2 오픈소스 라이선스 관점

| 측면 | 분석 |
|------|------|
| **저작권 귀속** | 대부분의 OSS 라이선스(MIT, Apache 2.0, GPL)는 저작자 표시를 요구하지만, 이는 라이선스 파일 내 원저작자 표시를 의미. 커밋의 Co-Authored-By는 관례일 뿐 법적 요구사항이 아님 |
| **CLA (Contributor License Agreement)** | 많은 프로젝트가 CLA에서 "본인이 작성했거나 기여 권한이 있음"을 선언하게 함. AI 생성 코드의 저작권은 법적 회색지대이나, CLA 위반 가능성 존재 |
| **DCO (Developer Certificate of Origin)** | `Signed-off-by` 라인으로 본인 작성을 증명하는 프로젝트에서, AI 생성 코드에 DCO를 붙이면 허위 진술에 해당할 수 있음 |
| **직접적 라이선스 위반?** | 대부분의 OSS 라이선스는 AI 사용 공개를 요구하지 않으므로, 엄밀히 라이선스 위반이라 보기 어려움 |

### 3.3 커뮤니티 기여 가이드라인 위반 여부

현재 트렌드:
- **Linux 커널**: 2024년부터 AI 생성 코드에 대한 공개 논의 진행 중
- **여러 OSS 프로젝트**: AI 사용 공개를 기여 가이드라인에 추가하는 추세
- **GitHub Copilot**: "AI-assisted" 표시가 점차 표준화되는 중

Undercover Mode는 이 트렌드와 **정면으로 충돌**한다:
- 커밋 메시지에서 "Claude Code" 문구를 **금지어로 지정**
- "Write commit messages as a human developer would"를 명시적으로 지시
- AI 사용 사실 자체를 숨기도록 설계

### 3.4 "AI Disclosure" 트렌드와의 충돌

| 트렌드 | Undercover Mode |
|--------|----------------|
| EU AI Act의 AI 생성물 표시 의무 | 직접적 위반 가능성 (단, 코드가 규제 대상인지는 논쟁 중) |
| ACM의 AI 사용 공개 정책 | 학술 논문에만 적용되나, 정신은 충돌 |
| GitHub의 Copilot 투명성 정책 | Copilot은 AI 사용을 공개하는 방향. Anthropic은 숨기는 방향 |
| 오픈소스 커뮤니티의 AI 공개 움직임 | 직접적 충돌 |

### 3.5 이 기능의 존재 자체가 증명하는 것

Undercover Mode의 존재는 다음을 객관적으로 증명한다:

1. **Anthropic은 직원들이 Claude Code로 외부 오픈소스에 기여한다는 것을 알고 있다** -- 기능을 만들었으므로
2. **AI 사용 사실을 숨기는 것이 의도적이다** -- 코드 주석에 "don't blow your cover"라고 적혀 있음
3. **비활성화를 불가능하게 만든 것은 의도적이다** -- "There is NO force-OFF"를 명시적으로 설계
4. **자사 공개 리포(claude-code)에서도 활성화된다** -- 외부 커뮤니티에 대한 일관된 은폐 의도
5. **"safety"라는 프레이밍은 자사 보호**이지, 커뮤니티 보호가 아니다

---

## 4. 비평가로서의 종합 평가

### 4.1 정당한 우려

코드네임 유출 방지는 합리적인 보안 조치다. `Capybara`, `Tengu` 같은 내부 코드네임이 커밋 메시지에 노출되면 경쟁사에 제품 로드맵을 알려주는 셈이다.

### 4.2 과잉 대응

그러나 Undercover Mode는 코드네임 보호를 **훨씬 넘어선다**:
- Co-Authored-By 태그 제거 -- 이것은 코드네임이 아니라 AI 기여 투명성의 문제
- "Claude Code"라는 이름 자체를 금지어로 지정 -- 이미 공개된 제품명
- "any mention that you are an AI" -- AI 사용 자체를 부정하라는 지시
- "Write commit messages as a human developer would" -- 적극적인 위장 지시

코드네임 보호만 원했다면, `Co-Authored-By: Claude <noreply@anthropic.com>`처럼 코드네임을 빼되 AI 사용은 공개하는 방식이 가능했다. 그러나 Anthropic은 **AI 사용 자체를 숨기는 것**을 선택했다.

### 4.3 "코드 흡수" 시나리오에 대한 판정

코드베이스에 자동 코드 수확 메커니즘은 **없다**. 그러나 이것이 시나리오를 완전히 부정하지는 않는다:

- 직원이 외부 프로젝트에서 얻은 아이디어를 수동으로 내부에 적용하는 것은 코드와 무관한 인적 행위
- 대화 로그가 Anthropic 서버에 전송되면 이론적으로 학습 데이터로 활용 가능 (그러나 이는 Undercover와 무관한 별도 문제)
- Undercover Mode의 목적은 "코드 흡수"가 아니라 "AI 사용 은폐"가 더 정확한 설명

### 4.4 건강한 회의와 확인된 사실 분리

| 구분 | 내용 |
|------|------|
| **확인된 사실** | Anthropic 직원이 Claude Code로 OSS 기여 시 AI 흔적이 자동/의무적으로 제거된다 |
| **확인된 사실** | 비활성화가 불가능하도록 의도적으로 설계되었다 |
| **확인된 사실** | "인간처럼 작성하라"는 적극적 위장 지시가 포함되어 있다 |
| **확인된 사실** | 자사 공개 리포에서도 활성화된다 |
| **미확인/근거없음** | 코드를 수확하는 자동 메커니즘은 코드에 없다 |
| **미확인/근거없음** | 코드 스타일을 변환하는 기능은 없다 |
| **미확인/근거없음** | git 히스토리를 조작하는 기능은 없다 |

---

## 5. 결론

Undercover Mode는 기술적으로는 "내부 정보 유출 방지"라는 정당한 보안 기능이다. 그러나 그 범위가 코드네임 보호를 넘어 **AI 사용 자체를 조직적으로 은폐**하는 수준까지 확장되어 있다.

"코드 흡수"라는 의심된 시나리오는 코드베이스에서 근거를 찾을 수 없었다. 그러나 Undercover Mode의 존재 자체가 제기하는 윤리적 질문 -- **AI 사용 투명성을 조직적으로 거부하는 것이 오픈소스 정신과 양립할 수 있는가** -- 은 별도의 심각한 문제로 남는다.

특히 Anthropic이 "Responsible AI"를 기업 정체성의 핵심으로 내세우는 회사라는 점에서, 자사 직원의 AI 사용을 적극적으로 숨기는 기능을 만든 것은 아이러니라 할 수 있다.

---

**관련 코드 파일:**
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/utils/undercover.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/utils/attribution.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/utils/commitAttribution.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/constants/prompts.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/tools/BashTool/prompt.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/commands/commit.ts`
- `/Users/soohongkim/Documents/workspace/personal/claude-code/src/commands/commit-push-pr.ts`
