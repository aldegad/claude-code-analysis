# 18. Claude Code 이스터에그와 만우절 기능 분석 보고서

> **분석팀 루미** | 2026-03-31  
> 대상: `/Users/soohongkim/Documents/workspace/personal/claude-code`  
> 범위: 이스터에그, 숨은 UX, 만우절성 기능, 내부 코드네임  
> 주의: 외부 빌드 스텁만 남은 기능은 "직접 확인됨"과 "직접 확인 불가"를 구분해 적는다.

---

## 1. 사용자 감정 감지 (`matchesNegativeKeyword`)

`matchesNegativeKeyword()`는 사용자 프롬프트를 소문자로 바꾼 뒤 정규식으로 부정적 표현을 감지한다. 확인된 패턴은 `wtf`, `wth`, `ffs`, `omfg`, `shit`, `shitty`, `shittiest`, `dumbass`, `horrible`, `awful`, `pissed off`, `pissing off`, `piece of shit`, `piece of crap`, `piece of junk`, `what the fuck`, `what the hell`, `fucking broken`, `fucking useless`, `fucking terrible`, `fucking awful`, `fucking horrible`, `fuck you`, `screw this`, `screw you`, `so frustrating`, `this sucks`, `damn it` 등이다. 증거는 `src/utils/userPromptKeywords.ts:4-10`이다.

이 감지 결과는 현재 노출된 코드 기준으로 텔레메트리에만 쓰인다. `processTextPrompt()`는 `matchesNegativeKeyword(userPromptText)` 결과를 `isNegative`에 담은 뒤 `logEvent('tengu_input_prompt', { is_negative: isNegative, is_keep_going: isKeepGoing })`만 호출하고, 이후에는 그대로 사용자 메시지를 만들어 반환한다. 별도의 모델 선택, 시스템 프롬프트 변경, 온도 조절, 권한 모드 조정 분기는 보이지 않는다. 증거는 `src/utils/processUserInput/processTextPrompt.ts:59-64`, `src/utils/processUserInput/processTextPrompt.ts:66-99`이다.

따라서 "욕하면 Claude가 더 잘한다"는 도시전설은 노출된 소스 기준으로는 사실이 아니다. 확인되는 동작은 "사용자가 부정적 표현을 썼는지 기록한다"뿐이며, 적어도 이 경로에서는 모델 행동을 바꾸지 않는다. 근거는 `src/utils/userPromptKeywords.ts:4-10`, `src/utils/processUserInput/processTextPrompt.ts:59-64`이다.

---

## 2. `/btw` -- 사이드 질문 기능

`/btw`는 메인 작업을 끊지 않고 "옆으로 새는 질문"을 던지는 기능이다. 파일 헤더 주석부터 "quick questions without interrupting the main agent context"라고 설명하고, 시스템 리마인더에도 "The main agent is NOT interrupted"라고 명시한다. 또한 별도 경량 에이전트를 띄우되 부모 컨텍스트의 프롬프트 캐시를 공유하고, 도구 사용은 전면 금지하며, `maxTurns: 1`로 한 번만 답하게 한다. 증거는 `src/utils/sideQuestion.ts:2-7`, `src/utils/sideQuestion.ts:48-60`, `src/utils/sideQuestion.ts:63-76`, `src/utils/sideQuestion.ts:80-96`이다.

REPL 쪽도 `/btw`를 "즉시형(immediate) 로컬 JSX 명령"으로 취급한다. 주석은 이 명령이 Claude가 처리 중일 때도 계속 떠 있도록 별도 ref로 관리한다고 적고, 하단 렌더링 코드 역시 즉시형 명령이 메인 대화가 스트리밍되는 동안 움직이지 않도록 `bottom` 슬롯에 유지한다고 설명한다. 증거는 `src/screens/REPL.tsx:1041-1058`, `src/screens/REPL.tsx:4594-4604`이다.

사용 횟수는 전역 설정의 `btwUseCount`로 추적한다. 설정 스키마에 `btwUseCount: number // Number of times user has used /btw`가 있고 기본값은 `0`이다. 실제 `/btw` 명령 실행 시 `btwUseCount: current.btwUseCount + 1`로 증가시킨다. 증거는 `src/utils/config.ts:355-356`, `src/utils/config.ts:608-610`, `src/commands/btw/btw.tsx:229-241`이다.

이 기능을 한 번도 안 써본 사용자가 30초 이상 대기하면 스피너 팁으로 `/btw`가 노출된다. 조건은 `elapsedSnapshot > 30_000 && !getGlobalConfig().btwUseCount`이고, 실제 문구는 "Use /btw to ask a quick side question without interrupting Claude's current work"다. 증거는 `src/components/Spinner.tsx:255-259`이다.

---

## 3. `/stickers` -- 스티커 판매

`/stickers`는 숨은 디버그 명령이 아니라 진짜 굿즈 판매 링크를 여는 로컬 커맨드다. 커맨드 이름과 설명은 각각 `stickers`, `Order Claude Code stickers`로 등록돼 있다. 증거는 `src/commands/stickers/index.ts:3-9`이다.

실제 구현에는 URL이 `https://www.stickermule.com/claudecode`로 하드코딩돼 있고, `openBrowser(url)`을 호출해 열도록 되어 있다. 성공 시 "Opening sticker page in browser…"를, 실패 시 동일 URL을 직접 방문하라고 출력한다. 증거는 `src/commands/stickers/stickers.ts:4-15`이다.

`openBrowser()`는 내부적으로 macOS에서는 `open`, Windows에서는 `rundll32` 또는 `BROWSER`, Linux에서는 `xdg-open`을 써 시스템 기본 브라우저를 연다. 즉 `/stickers`는 텍스트만 보여주는 농담이 아니라 실제 판매 페이지 오픈 로직이다. 증거는 `src/utils/browser.ts:39-67`이다.

---

## 4. `/good-claude` -- 내부 전용 칭찬 기능

외부 빌드에서 `/good-claude`는 완전히 스텁 처리돼 있다. 노출된 파일은 `export default { isEnabled: () => false, isHidden: true, name: 'stub' };` 한 줄뿐이다. 증거는 `src/commands/good-claude/index.js:1`이다.

이 명령은 "외부 빌드에서 제거되는 내부 전용 명령" 목록에도 포함돼 있다. `INTERNAL_ONLY_COMMANDS` 바로 위 주석이 "Commands that get eliminated from the external build"라고 설명하고, 그 배열 안에 `goodClaude`가 들어간다. 즉 현재 덤프만 보면 외부판에는 비활성 스텁, 내부판에는 실구현이 있었던 구조로 해석된다. 증거는 `src/commands.ts:224-233`이다.

자동 실행 헬퍼에도 내부 전용 흔적이 남아 있다. `getAutoRunCommand()` 주석은 "ANT-ONLY: good-claude command only exists in ant builds"라고 적고, `reason === 'feedback_survey_good'`일 때 ant 빌드면 `/good-claude`, 아니면 `/issue`를 반환하도록 작성돼 있다. 다만 현재 외부 빌드로 노출된 파일에서는 이 분기가 `"external" === 'ant'`로 접혀 있어 실제로는 죽은 코드다. 증거는 `src/utils/autoRunIssue.tsx:97-106`이다.

중요한 점은, "피드백 서베이에서 Good 선택 시 자동 실행"은 노출된 REPL 코드만으로는 끝까지 확인되지 않는다는 것이다. 자동 실행 래퍼 주석에는 `/issue or /good-claude`가 언급되지만, 실제 `feedbackSurvey.handleSelect`에서 확인되는 자동 실행 트리거는 `selected === 'bad'` 한 경우뿐이다. `feedback_survey_good`를 `setAutoRunIssueReason()`로 넣는 호출은 이번 덤프에서 찾지 못했다. 따라서 "Good 선택 시 자동 실행"은 헬퍼 수준의 흔적은 있으나, 현재 공개된 코드만으로는 완전 입증되지 않는다. 증거는 `src/screens/REPL.tsx:1694-1706`, `src/screens/REPL.tsx:3580-3589`, `src/utils/autoRunIssue.tsx:77-117`이다.

---

## 5. 내부 코드네임 체계

### 5.1 `fennec` = Opus 4.6 이전 모델 별칭

`migrateFennecToOpus()`는 주석에서 "removed fennec model aliases"를 "new Opus 4.6 aliases"로 옮긴다고 직접 적는다. 매핑도 `fennec-latest -> opus`, `fennec-latest[1m] -> opus[1m]`, `fennec-fast-latest -> opus[1m] + fast mode`로 구체적이다. 증거는 `src/migrations/migrateFennecToOpus.ts:6-17`, `src/migrations/migrateFennecToOpus.ts:27-42`이다.

### 5.2 `bagel` = 웹 브라우저 도구

앱 상태 스토어는 `bagel`을 "WebBrowser tool (codename bagel)"이라고 주석으로 박아둔다. 이어서 `bagelActive`, `bagelUrl`, `bagelPanelVisible`이 각각 웹 브라우저 툴의 푸터 표시, 현재 URL, 패널 토글 상태를 뜻한다고 적는다. 프롬프트 입력 푸터에서도 실제 아이템 이름으로 `bagel`을 쓴다. 증거는 `src/state/AppStateStore.ts:248-253`, `src/components/PromptInput/PromptInput.tsx:460-478`, `src/components/PromptInput/PromptInput.tsx:1826-1827`이다.

### 5.3 `capybara` = 모델 버전 코드네임

`main.tsx`는 "Ant model aliases (capybara-fast etc.)"라고 직접 적고, 이것이 `tengu_ant_model_override`로 해석된다고 설명한다. 또 모델명 마스킹 함수 예시도 `capybara-v2-fast -> cap*****-v2-fast`를 쓴다. 따라서 `capybara`는 Buddy 캐릭터 이름이기도 하지만 동시에 내부 모델 코드네임이기도 하다. 증거는 `src/main.tsx:2001-2013`, `src/utils/model/model.ts:386-392`이다.

### 5.4 `tengu` = 상위 내부 코드네임

`tengu_*` 접두사는 이벤트, 설정, GrowthBook, 초기화 함수명 전반에 깔려 있다. 예를 들어 `BATCH_CONFIG_NAME = 'tengu_1p_event_batch_config'`, `logEvent(... 'tengu_api_query')` 예시, `logTenguInit()` 함수, `tengu_init` 이벤트가 확인된다. 다만 "Claude Code 프로젝트의 공식 코드네임이 Tengu"라고 한 줄로 선언한 주석은 이번 덤프에서 찾지 못했다. 따라서 이 결론은 단일 선언문이 아니라 네이밍 패턴에서 나오는 강한 추정이다. 증거는 `src/services/analytics/firstPartyEventLogger.ts:87-92`, `src/services/analytics/firstPartyEventLogger.ts:153-160`, `src/main.tsx:2496-2504`, `src/main.tsx:4514-4530`이다.

보조 증거로 Undercover 지침은 외부 저장소에서 새어나가면 안 되는 내부 코드네임 예시로 "Capybara, Tengu"를 직접 든다. 여기서는 Tengu가 적어도 Anthropic 내부 명명체계의 일부였음이 드러난다. 증거는 `src/utils/undercover.ts:4-6`, `src/utils/undercover.ts:47-50`이다.

---

## 6. Buddy 만우절 갓챠 시스템

### 6.1 전반 구조

Buddy는 빌드타임 피처 플래그 뒤에 숨어 있다. 커맨드 로딩은 `feature('BUDDY') ? require('./commands/buddy/index.js') : null` 구조이고, 알림과 스프라이트 렌더러도 시작 부분에서 `feature('BUDDY')`를 확인한다. 증거는 `src/commands.ts:118-122`, `src/buddy/useBuddyNotification.tsx:43-58`, `src/buddy/useBuddyNotification.tsx:79-97`, `src/buddy/CompanionSprite.tsx:167-175`, `src/buddy/CompanionSprite.tsx:215-217`이다.

Buddy의 뽑기 결과는 사실상 "사용자별 고정 가챠"다. `roll(userId)`는 `key = userId + SALT`를 만들고, 그 값을 해시해 `mulberry32` PRNG에 넣는다. 즉 실행할 때마다 랜덤을 다시 뽑는 구조가 아니라, 사용자 ID와 SALT에 의해 결정되는 결정론적 배정이다. 증거는 `src/buddy/companion.ts:15-24`, `src/buddy/companion.ts:27-37`, `src/buddy/companion.ts:84-85`, `src/buddy/companion.ts:104-117`이다.

SALT 문자열은 `friend-2026-401`이다. 이 값의 의미를 설명하는 주석은 없으므로 "`401`이 2026-04-01을 뜻한다"는 해석은 정황 추정이다. 다만 Buddy의 티저/라이브 날짜 코드가 모두 2026년 4월을 가리키기 때문에, 만우절(2026-04-01)을 의식한 작명일 가능성은 높다. 증거는 `src/buddy/companion.ts:84-85`, `src/buddy/useBuddyNotification.tsx:9-20`이다.

### 6.2 만우절 타이밍

`isBuddyTeaserWindow()`는 로컬 날짜 기준으로 2026년 4월(`getMonth() === 3`) 1~7일(`getDate() <= 7`)만 티저 창으로 취급한다. 주석도 "Teaser window: April 1-7, 2026 only"라고 적는다. 증거는 `src/buddy/useBuddyNotification.tsx:9-16`이다.

`isBuddyLive()`는 2026년 4월 이후(`year > 2026 || (year === 2026 && month >= 3)`) 계속 true가 되게 작성돼 있다. 즉 티저는 1주일짜리지만, 라이브 함수 자체는 4월부터 영구 오픈 쪽으로 설계돼 있다. 다만 이번 덤프에서 `isBuddyLive()` 실제 호출 위치는 찾지 못했다. 증거는 `src/buddy/useBuddyNotification.tsx:17-20`이다.

### 6.3 가중치 기반 희귀도 뽑기

희귀도는 `rollRarity()`가 `RARITY_WEIGHTS` 총합을 기준으로 가중치 추첨하는 구조다. 구현은 전체 합을 구한 뒤 난수를 굴려 `common -> uncommon -> rare -> epic -> legendary` 순서로 차감해 첫 음수 지점의 희귀도를 반환한다. 증거는 `src/buddy/companion.ts:43-50`이다.

실제 확률 테이블은 `common: 60`, `uncommon: 25`, `rare: 10`, `epic: 4`, `legendary: 1`이다. 요청하신 5단계/가중치는 정확히 소스와 일치한다. 증거는 `src/buddy/types.ts:126-132`이다.

추가로 희귀도별 스탯 바닥값도 `common: 5`, `uncommon: 15`, `rare: 25`, `epic: 35`, `legendary: 50`으로 따로 존재한다. 높은 희귀도일수록 스탯 최저선이 올라간다. 증거는 `src/buddy/companion.ts:53-59`, `src/buddy/companion.ts:61-81`이다.

### 6.4 18종 캐릭터, 6종 눈, 8종 모자, 5종 스탯

종(species)은 총 18종이다. `duck`, `goose`, `blob`, `cat`, `dragon`, `octopus`, `owl`, `penguin`, `turtle`, `snail`, `ghost`, `axolotl`, `capybara`, `cactus`, `robot`, `rabbit`, `mushroom`, `chonk`가 `SPECIES` 배열에 들어 있다. 증거는 `src/buddy/types.ts:17-18`, `src/buddy/types.ts:19-29`, `src/buddy/types.ts:39-52`, `src/buddy/types.ts:54-73`이다.

눈 모양은 6종(`·`, `✦`, `×`, `◉`, `@`, `°`)이고, 모자는 8종(`none`, `crown`, `tophat`, `propeller`, `halo`, `wizard`, `beanie`, `tinyduck`)이며, 스탯은 5종(`DEBUGGING`, `PATIENCE`, `CHAOS`, `WISDOM`, `SNARK`)이다. 증거는 `src/buddy/types.ts:76-97`이다.

다만 요청하신 "legendary에만 모자"는 소스와 맞지 않는다. 실제 코드는 `common`일 때만 강제로 `hat: 'none'`이고, 그 외 희귀도는 `pick(rng, HATS)`로 뽑는다. `HATS` 배열 자체에 `none`까지 포함돼 있으므로 uncommon/rare/epic/legendary 모두 "모자가 없을 수도 있고, 있을 수도 있는" 구조다. 렌더러도 `bones.hat !== 'none'`일 때만 0번째 줄에 모자를 얹는다. 증거는 `src/buddy/companion.ts:91-100`, `src/buddy/types.ts:79-89`, `src/buddy/sprites.ts:443-467`이다.

### 6.5 ASCII 아트 렌더링

Buddy의 비주얼은 실제 ASCII 스프라이트다. `sprites.ts`는 종별 몸체 프레임을 문자열 배열로 들고 있고, "Each sprite is 5 lines tall, 12 wide"라고 설명한다. `renderSprite()`는 눈(`{E}`)을 치환하고, 조건이 맞으면 모자 줄을 덮어쓴 뒤 최종 문자열 배열을 반환한다. 증거는 `src/buddy/sprites.ts:23-25`, `src/buddy/sprites.ts:454-467`이다.

`CompanionSprite.tsx`는 그 결과를 `renderSprite(companion, spriteFrame)`로 받아 줄 단위 `<Text>`로 찍는다. 즉 UI 라이브러리 위에서 문자열 스프라이트를 그대로 렌더링하는 구조다. 증거는 `src/buddy/CompanionSprite.tsx:258-289`이다.

### 6.6 말풍선, 유휴 애니메이션, 하트 이펙트

말풍선은 `SpeechBubble` 컴포넌트가 담당한다. 텍스트를 30자 기준으로 감싸고, 둥근 테두리와 꼬리(`tail`)를 붙여 렌더링한다. `CompanionSprite`는 반응 문자열(`reaction`)이 있을 때 말풍선을 붙이고, `BUBBLE_SHOW = 20`, `FADE_WINDOW = 6`으로 약 10초 노출 후 마지막 약 3초 동안 희미해지게 만든다. 증거는 `src/buddy/CompanionSprite.tsx:17-19`, `src/buddy/CompanionSprite.tsx:43-61`, `src/buddy/CompanionSprite.tsx:112-150`, `src/buddy/CompanionSprite.tsx:205-214`, `src/buddy/CompanionSprite.tsx:220-221`, `src/buddy/CompanionSprite.tsx:286-289`이다.

유휴 애니메이션도 명시적이다. `IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]`로 대부분 정지 프레임을 유지하다 가끔 몸짓 프레임과 눈깜빡임(`-1`)을 끼워 넣는다. 반응 중이거나 쓰다듬는 중이면 전체 프레임을 빠르게 순환한다. 증거는 `src/buddy/CompanionSprite.tsx:21-27`, `src/buddy/CompanionSprite.tsx:242-257`이다.

`/buddy pet` 하트 이펙트의 시각 효과는 분명히 존재한다. 상태 스토어는 `companionPetAt`을 "Timestamp of last /buddy pet"이라고 설명하고, `CompanionSprite.tsx`는 `PET_BURST_MS = 2500`, `PET_HEARTS` 프레임, `petting` 판정으로 하트를 2.5초 띄운다. 다만 이번 덤프에서는 `companionPetAt`을 실제로 설정하는 `/buddy pet` 명령 구현 파일을 찾지 못했다. 즉 "하트 이펙트"는 직접 확인되지만, "어디서 상태를 세팅하는가"는 노출된 코드만으로는 미확인이다. 증거는 `src/state/AppStateStore.ts:168-171`, `src/buddy/CompanionSprite.tsx:19`, `src/buddy/CompanionSprite.tsx:25-27`, `src/buddy/CompanionSprite.tsx:178-199`, `src/buddy/CompanionSprite.tsx:222-223`, `src/buddy/CompanionSprite.tsx:232-243`, `src/buddy/CompanionSprite.tsx:259-289`이다.

---

## 결론

Claude Code의 이스터에그는 단순 장난이 아니라 세 부류로 나뉜다.

1. 실제 UX 기능: `/btw`, `/stickers`처럼 지금도 동작 구조가 완성된 것.
2. 내부 전용 흔적: `/good-claude`처럼 외부 빌드에선 스텁만 남고, 내부판 정황만 남은 것.
3. 만우절성 실험 기능: Buddy처럼 날짜 연출, 가챠, ASCII 캐릭터, 반응형 UI를 한데 묶은 것.

특히 Buddy는 "만우절 티저"로 포장돼 있지만, 코드 구조만 보면 일회성 농담을 넘어서 장기 유지도 염두에 둔 설계다. 반면 몇몇 인터넷 밈성 주장, 예를 들면 "욕하면 더 잘한다", "legendary만 모자를 쓴다", "Good 선택 시 외부 빌드에서도 /good-claude가 자동 실행된다" 같은 말은 이번 덤프의 실제 코드와는 다르거나, 끝까지 확인되지 않는다.
