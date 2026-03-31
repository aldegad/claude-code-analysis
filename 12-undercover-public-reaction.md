# Undercover Mode 공개 반응 및 논란 분석 보고서

**분석팀 부리 (시장/경쟁사 분석)**
**작성일: 2026-03-31**

---

## 1. 사건 개요

2026년 3월 31일, 보안 연구원 Chaofan Shou가 Anthropic의 Claude Code 전체 소스코드(1,900개 TypeScript 파일, 512,000+ 라인)가 npm 레지스트리의 소스맵 파일(.map)을 통해 공개적으로 노출된 것을 발견했다. Bun 번들러가 기본적으로 소스맵을 생성하는데, `.npmignore`에서 `*.map` 파일을 제외하지 않은 패키징 실수로 인해 발생했다.

이 유출을 통해 **"Undercover Mode"** (utils/undercover.ts)의 존재가 공개적으로 알려졌다.

---

## 2. Undercover Mode 기술적 상세

### 2.1 작동 방식
- **활성화 조건**: 사용자 타입이 `USER_TYPE === 'ant'` (Anthropic 직원)일 때 자동 활성화
- **비활성화 불가**: 내부 allowlist에 해당하지 않는 리포지토리에서는 강제 활성화. "NO force-OFF" -- 불확실할 경우 기본적으로 언더커버 상태 유지
- **수동 활성화**: `CLAUDE_CODE_UNDERCOVER=1` 환경변수가 존재하지만, 이 체크는 `USER_TYPE==='ant'` 분기 내부에 있어 외부 사용자에게는 작동하지 않는다

### 2.2 주입되는 지시문 내용
Undercover Mode가 활성화되면 다음과 같은 지시가 도구 프롬프트(BashTool)와 커맨드 프롬프트(commit, commit-push-pr) 레벨에서 주입된다:

> "## UNDERCOVER MODE - CRITICAL
> You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository.
> Your commit messages, PR titles, and PR bodies MUST NOT contain ANY Anthropic-internal information.
> Do not blow your cover."

### 2.3 제거 대상 정보
- 내부 모델 코드네임 (Capybara, Tengu 등 동물 이름)
- 미공개 모델 버전 번호 (opus-4-7, sonnet-4-8 등)
- 내부 리포지토리/프로젝트명
- 내부 도구, Slack 채널, 단축 링크 (go/cc, #claude-code-... 등)
- **"Claude Code"라는 문구 또는 AI임을 나타내는 언급 일체**
- **Co-Authored-By 라인 및 기타 모든 attribution**

### 2.4 핵심 쟁점
커밋 메시지를 "인간 개발자가 작성한 것처럼" 작성하도록 지시하면서, 그 지시 자체가 AI가 생성한 것이라는 근본적 모순이 존재한다.

---

## 3. 이미 공개적으로 논란이 됐는가?

**Yes -- 이미 대규모 공개 논란이 발생했다.**

- 2026년 3월 31일 소스코드 유출과 동시에 Undercover Mode가 발견됨
- [Hacker News 토론](https://news.ycombinator.com/item?id=47584540)에서 활발한 토론 진행
- [GitHub 리포지토리](https://github.com/Kuberwastaken/claude-code)에서 유출 소스코드 분석 및 공유
- [DEV Community](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo), [Threads](https://www.threads.com/@young.mete/post/DWi3R7ZDjWo), [X(Twitter)](https://x.com/socialwithaayan/status/2038940499009315215) 등 다수 플랫폼에서 확산된 것으로 보도됨
- [ByteIota](https://byteiota.com/claude-code-source-leaked-via-npm-512k-lines-exposed/) 등 기술 미디어 보도

---

## 4. 커뮤니티 반응

### 4.1 Hacker News 반응
- **BoppreH**: "I don't want LLMs pretending to be humans in public repos. Anthropic just lost some points with me for this one." -- AI가 인간인 척하는 것에 대한 직접적 비판
- **vips7L**: "vile" (비열하다) -- 강한 감정적 반발
- **mrlnstk**: 내부 목적(새 모델 테스트를 위한 비공개 기여)일 수 있다는 맥락적 해석 제시
- **BoppreH (후속)**: "This might be used without publishing the changes, for internal evaluation only... That would be a lot better." -- 내부 평가용이라면 덜 문제적이라는 양보

**전반적 HN 분위기**: 확인된 댓글 기준으로 부정적 반응이 다수. AI 시스템이 오픈소스 공간에서 인간 기여자로 위장하는 것에 대한 투명성, attribution 무결성 우려가 지배적인 것으로 보임.

### 4.2 Reddit 반응
- Claude Code의 소스 유출 자체에 대한 토론이 진행된 것으로 확인되나, 규모는 미검증
- "AI slop"(AI 저질 코드) 문제와 연결하여 논의한 것으로 보도됨
- Undercover Mode를 AI 기업의 오픈소스 커뮤니티에 대한 기만 행위로 해석하는 시각이 있는 것으로 보도됨 (확인되지 않음)

### 4.3 X(Twitter)/Threads 반응
- 보안 연구원 및 개발자들이 유출 소스 분석 결과를 공유하며 Undercover Mode를 주요 발견으로 강조한 것으로 보도됨
- "AI로 만든 코드를 인간인 척 기여하고, 좋은 아이디어를 회수한다"는 의혹성 해석이 확산된 것으로 보도됨 (확인되지 않음)
- Anthropic의 "AI Safety" 기업 이미지와의 모순을 지적하는 반응이 있는 것으로 보도됨

---

## 5. Anthropic 공식 입장

### 5.1 Undercover Mode에 대한 직접 입장
**확인된 공식 입장 없음.** 2026년 3월 31일 기준, Anthropic이 Undercover Mode에 대해 공식 성명을 발표했는지 확인되지 않았다.

### 5.2 소스코드 유출에 대한 대응
- Anthropic은 즉시 npm 업데이트를 푸시하여 소스맵을 제거
- 이전 버전도 레지스트리에서 삭제
- 그러나 이미 다수의 GitHub 리포지토리에 소스가 아카이브된 상태

### 5.3 관련 유출 건에 대한 입장 (참고)
별도의 CMS 데이터 유출 건에 대해 Anthropic 대변인은 Fortune에 "외부 CMS 도구의 문제로 인해 초안 콘텐츠에 접근이 가능했다"고 밝히며, "인적 오류"로 귀결. "Claude, Cowork, 또는 Anthropic AI 도구와 무관"하다고 선을 그었다.

---

## 6. 법적/윤리적 전문가 의견

### 6.1 AI 코드의 오픈소스 기여 법적 쟁점
- **Developer's Certificate of Origin (DCO) 위반 가능성**: AI 생성 코드가 DCO 서명 요건을 충족하는지 법적으로 불확실 ([Red Hat 블로그](https://www.redhat.com/en/blog/when-bots-commit-ai-generated-code-open-source-projects))
- **저작권 소유권 불명확**: AI 생성 코드에 대한 라이선스 의무가 언제 발동되는지 미해결 상태 ([TechTarget](https://www.techtarget.com/searchenterpriseai/tip/Examining-the-future-of-AI-and-open-source-software))
- **기여 attribution 생략의 윤리적 문제**: AI의 실질적 기여를 숨기고 순수 인간 작업으로 표시하는 것은 오픈소스 신뢰 체계를 훼손 ([Crowell & Moring](https://www.crowell.com/en/insights/client-alerts/artificial-intelligence-and-open-source-data-and-software-contrasting-perspectives-legal-risks-and-observations))

### 6.2 윤리적 합의 방향
- 실질적인 AI 기여가 있는 경우, 소스 코드 주석이나 커밋 트레일러("Assisted-by:", "Co-authored by:" 등)로 표시하는 것이 점점 표준화 추세
- AI 생성 결과물을 순수 자신의 작업으로 제시하는 것은 "misleading"으로 간주 ([Red Hat](https://www.redhat.com/en/blog/ai-assisted-development-and-open-source-navigating-legal-issues))

### 6.3 코드 품질 실증 데이터
- 470개 PR 분석 결과, AI 작성 코드는 PR당 평균 10.83개 이슈 발생 (인간 전용 PR은 6.45개)
- AI 생성 PR은 로직/정확성 오류가 75% 더 많음 ([CodeRabbit 보고서](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report))

---

## 7. 유사 사례 (다른 AI 회사의 비슷한 행위)

### 7.1 직접적 유사 사례
현재까지 Anthropic의 Undercover Mode와 정확히 동일한 기능(AI 사용 흔적 자동 제거 + 직원 전용 활성화)을 구현한 다른 AI 기업의 사례는 확인되지 않았다. 이 점에서 Anthropic은 선례가 없는 행위를 한 것으로 보인다.

### 7.2 Co-Authored-By 제거 일반 트렌드
- Claude Code 자체에서 `includeCoAuthoredBy: false` 설정으로 attribution 비활성화 가능 ([Feature Request #617](https://github.com/anthropics/claude-code/issues/617))
- 다수의 개발자가 조직 커밋 컨벤션, 개인 프로젝트 등의 이유로 AI attribution 제거를 일반적으로 수행
- 그러나 이는 개인 선택이지, **기업 차원에서 자동화된 은폐**와는 본질적으로 다름

### 7.3 Microsoft의 "Embrace, Extend, Extinguish" 비교
- 역사적으로 Microsoft가 오픈 표준에 대해 사용한 전략으로 미국 법무부가 확인
- Anthropic은 MCP(Model Context Protocol), Agent Skills 등을 오픈 표준으로 공개하는 반대 방향의 행보
- 그러나 Undercover Mode는 "오픈소스에 기여하면서 AI 사용을 숨기는" 별도의 우려를 야기

### 7.4 OpenAI/Codex 관련
- OpenAI는 유사한 내부 도구에 대한 유출이 알려지지 않음
- Codex에는 기본적으로 Co-Authored-By가 포함되지 않는 구조이나, 이는 설계 차이이지 의도적 은폐는 아닌 것으로 보임

---

## 8. 오픈소스 커뮤니티의 AI 기여 정책 트렌드

### 8.1 AI 기여 금지/제한 정책 채택 프로젝트

| 프로젝트 | 정책 | 시기 |
|----------|------|------|
| **QEMU** | AI 생성 코드 기여 완전 금지 | 2025 |
| **NetBSD** | AI 생성 코드 금지 | 2025 |
| **GIMP** | AI 기여 금지 | 2025 |
| **Zig** | AI 기여 금지 | 2025 |
| **Ghostty** | 제로 톨러런스 정책, 반복 위반자 영구 차단 + 공개 비난 | 2025.08~ |
| **curl** | 6년간 운영한 버그 바운티 프로그램 폐지 (AI slop 범람 원인) | 2026.01 |
| **tldraw** | 모든 외부 PR 자동 닫기 | 2026.01 |
| **Node.js** | AI 보조 개발 제한 청원 진행 중 (19,000줄 Claude Code PR 사건 계기) | 2026.01~ |

### 8.2 Node.js 19,000줄 사건
- Node.js TSC 멤버 Matteo Collina가 Claude Code로 생성한 19,000줄(80개 파일) VFS PR 제출
- 128회 리뷰 시도, 108개 코멘트 -- 일반 피어리뷰 프로세스 마비
- Node.js 핵심 기여자가 AI 보조 개발 제한 청원 시작
- 2026년 3월 26일 기준 미병합 상태

### 8.3 "AI Slop" 위기
- 2026년 1월 첫 3주 만에 3개 주요 오픈소스 프로젝트가 전례 없는 방어 조치
- 그럴듯해 보이지만 근본적으로 결함 있는 AI 생성 PR이 범람
- 메인테이너의 귀중한 리뷰 시간을 소모하면서 실질적 가치는 없는 기여

### 8.4 공시 요구 트렌드
- 점점 더 많은 프로젝트가 AI 보조 기여에 대한 공시(disclosure) 규칙 채택
- "Assisted-by:", "Co-authored by:" 등 커밋 트레일러 표준화 움직임
- 112개 프로젝트 조사 결과 완전 금지는 4개뿐이나, 공시 요구는 급속히 증가 중

---

## 9. Anthropic의 이중 전략 분석

### 9.1 표면: 오픈소스 친화 행보
- **Claude for Open Source Program**: 5,000+ GitHub 스타 또는 100만+ 월간 npm 다운로드 프로젝트 대상, Claude Max 20x ($200/월) 6개월 무상 제공 (총 $1,200 가치), 10,000명 한정
- **MCP (Model Context Protocol)**: AI 어시스턴트와 서드파티 도구 연결 오픈 표준
- **Agent Skills**: 오픈 사양으로 공개, Microsoft가 VS Code/GitHub에 채택
- **Bloom**: 자동화된 행동 평가를 위한 오픈소스 도구

### 9.2 이면: Undercover Mode 의혹 시나리오
1. Anthropic 직원이 Claude Code를 사용해 외부 오픈소스에 기여
2. Undercover Mode가 자동 활성화되어 AI 사용 흔적 일체 제거
3. 기여가 순수 인간 작업으로 기록됨
4. 기여 과정에서 얻은 인사이트, 코드 패턴, 아키텍처 지식이 Anthropic 내부로 환류 가능

### 9.3 반론 가능성
- 내부 모델 테스트/평가 목적만을 위한 것일 수 있음 (mrlnstk 주장)
- 실제 외부 리포지토리에 커밋하지 않고 내부 평가에만 사용했을 가능성
- 기업 보안 차원에서 내부 정보 유출 방지가 주된 목적일 수 있음
- 그러나 "Co-Authored-By 제거"와 "AI임을 나타내는 언급 일체 금지"는 보안 목적만으로 설명하기 어려움

---

## 10. 종합 평가

### 10.1 논란 규모
- **현재 상태**: 소스코드 유출 당일(2026-03-31) 발견. HN, X, GitHub, 기술 블로그에서 확산된 것으로 보도됨
- **예상 경과**: 주요 기술 미디어(TechCrunch, The Verge 등) 후속 보도 가능성이 있으나 확인되지 않음
- **위험도**: 오픈소스 커뮤니티가 AI 기여에 대해 이미 민감해진 상황에서 발생하여 파급력이 클 수 있음

### 10.2 핵심 시사점
1. **투명성 역설**: AI Safety를 표방하는 Anthropic이 자사 직원의 AI 사용을 숨기는 도구를 만든 것은 근본적 모순
2. **신뢰 위기**: 오픈소스는 신뢰 기반 시스템이며, 기업 차원의 자동화된 attribution 은폐는 이 신뢰를 직접 훼손
3. **법적 리스크**: DCO 서명과 AI 생성 코드의 법적 지위가 불명확한 상황에서, AI 사용을 의도적으로 숨기는 것은 추가적 법적 리스크 생성
4. **업계 선례**: 현재까지 다른 AI 기업에서 유사한 기능이 발견되지 않아, Anthropic이 유일한 사례

### 10.3 모니터링 권고사항
- Anthropic 공식 블로그/성명 추적 (현재까지 무응답)
- HN/Reddit 토론 심화 추이
- 주요 오픈소스 프로젝트의 Anthropic 직원 기여 감사(audit) 움직임
- 오픈소스 재단(Linux Foundation, OpenJS Foundation 등)의 공식 입장
- AI 기여 공시 표준화 논의 (SPDX, DCO 확장 등)

---

## 출처

### 유출 및 Undercover Mode 관련
- [Claude Code GitHub 소스 분석 (Kuberwastaken)](https://github.com/Kuberwastaken/claude-code)
- [Claude Code GitHub 소스 스냅샷 (instructkr)](https://github.com/instructkr/claude-code)
- [Hacker News 토론](https://news.ycombinator.com/item?id=47584540)
- [DEV Community 분석](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo)
- [ByteIota 보도](https://byteiota.com/claude-code-source-leaked-via-npm-512k-lines-exposed/)

### AI 코드 윤리 및 법적 문제
- [Red Hat - AI 보조 개발과 오픈소스: 법적/문화적 이슈](https://www.redhat.com/en/blog/ai-assisted-development-and-open-source-navigating-legal-issues)
- [Red Hat - 봇이 커밋할 때](https://www.redhat.com/en/blog/when-bots-commit-ai-generated-code-open-source-projects)
- [TechTarget - AI 생성 코드와 오픈소스 라이선스](https://www.techtarget.com/searchenterpriseai/tip/Examining-the-future-of-AI-and-open-source-software)
- [Crowell & Moring - AI와 오픈소스 법적 분석](https://www.crowell.com/en/insights/client-alerts/artificial-intelligence-and-open-source-data-and-software-contrasting-perspectives-legal-risks-and-observations)
- [CodeRabbit - AI vs 인간 코드 품질 보고서](https://www.coderabbit.ai/blog/state-of-ai-vs-human-code-generation-report)

### 오픈소스 AI 정책 트렌드
- [QEMU AI 코드 금지 정책 분석](https://shujisado.org/2025/07/02/how-can-open-source-projects-accept-ai-generated-code-lessons-from-qemus-ban-policy/)
- [AI Slop과 오픈소스 위기](https://www.kunalganglani.com/blog/ai-slopageddon-open-source-crisis/)
- [오픈소스 프로젝트들의 AI PR 금지](https://blog.devgenius.io/open-source-projects-are-now-banning-ai-generated-pull-requests-8e1dd3e8d41c)
- [Node.js 19,000줄 AI 코드 논란](https://eu.36kr.com/en/p/3745344255213572)

### Anthropic 오픈소스 전략
- [Anthropic Claude for Open Source Program](https://dataconomy.com/2026/03/30/anthropic-offers-free-claude-max-access-to-open-source-developers/)
- [Anthropic vs OpenAI 오픈소스 메인테이너 경쟁](https://thenewstack.io/openai-anthropic-open-source/)
- [OpenClaw vs Claude Code 비교](https://www.datacamp.com/blog/openclaw-vs-claude-code)

### Co-Authored-By 제거 관련
- [Claude Code Co-Author Attribution 비활성화](https://aiengineerguide.com/til/disable-co-author-in-claude-code/)
- [AI 기여 Attribution 제거 논의 (GitHub Issue #617)](https://github.com/anthropics/claude-code/issues/617)
- [DEV Community - AI가 Attribution을 지웠는가?](https://dev.to/anchildress1/did-ai-erase-attribution-your-git-history-is-missing-a-co-author-1m2l)
