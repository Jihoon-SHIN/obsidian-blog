## 1. prompt engineering 을 왜 해야하나

- LLM은 "다음 토큰 예측기"일 뿐이다. 같은 모델이라도 프롬프트 품질에 따라 결과가 수 배 차이난다.
- "모델을 바꾸는 것"보다 "프롬프트를 고치는 것"이 거의 항상 더 싸고 빠르다.
- 프롬프트는 코드다. 반드시 버전 관리, 실험 기록, 테스트가 필요하다.
    - 이번에 우리도 도입한 것으로 알고 있음 (Langfuse)

### **핵심 원칙 세 줄**

1. 구체적이고 예시를 함께 줘라.
2. 출력 포맷을 강제해라 (JSON schema 등).
3. 한 번 잘 된다고 끝이 아니다. 모델이 바뀌면 다시 튜닝.

---

## 2. 세 개의 레이어

현재 우리가 사용하는 범위 내에서 "프롬프트 엔지니어링"은 세 개의 서로 다른 레이어에서 동시에 일어나고 있다.

기본적으로 “프롬프트” 라는 것은 static 한 정적 파일임을 알고 있으면 쉽다.

### Layer 1 — Raw model API

- 대상: `claude-opus-4-7`, `gpt-5`, `gemini-3-flash` 같은 모델을 **SDK로 직접 호출**
- 우리 코드에서는 `server-common/services/ai` 가 해당
- 건드릴 수 있는 것
    - temperature, top_p, reasoning_effort, few-shot 예시, JSON schema, stop sequences, system prompt 전문 등

### Layer 2 — Agent / CLI 래퍼

- 대상: Claude Code, Cursor, Aider 같은 에이전트 프런트엔드
- 이미 거대한 system prompt (수천 토큰)와 tool use 루프가 내장돼 있음
- CoT / ReAct는 **프론트엔드 규약으로 내장됨** — Claude Code의 `think` / `ultrathink` 키워드가 thinking budget을 조절
- 사용자가 치는 건 "raw prompt"가 아니라 **agent에 대한 guidance**
- 여기서의 프롬프트 엔지니어링은 → `CLAUDE.md` 작성, task 설명 명료화, 필요 파일·맥락 지정

### Layer 3 — Plugin / Skill / MCP 레이어

- 대상: `oh-my-claudecode` 같은 plugin, skill, MCP 서버
- 특정 도메인의 **사전 작성된 프롬프트와 도구 묶음**을 agent에 주입
- 사용자는 trigger만 던지고 내부 prompt는 plugin 작성자가 책임
- 여기서의 프롬프트 엔지니어링은 → skill의 description / trigger 문구 / instruction 문서 작성

### 왜 이 구분이 본질적으로 중요한가

세 레이어는 **서로 다른 문제를 풀고 있음**

|레이어|주 사용자|프롬프트 엔지니어링의 형태|백서 기법의 유효성|
|---|---|---|---|
|Layer 1|서버 엔지니어 (우리)|SDK 파라미터 + 프롬프트 본문 설계|대부분 그대로 적용|
|Layer 2|IDE에서 코딩하는 개발자|`CLAUDE.md`, task 지시|원칙은 공통, 구체 기법은 제한적|
|Layer 3|plugin / skill 작성자|skill 명세·trigger 문구|원칙은 공통, 더 추상화됨|

혼동이 생길 수 있는 이유

- "Claude Code에 CoT 프롬프트 추가해야 하나?" 같은 질문은 사실 Layer 2에 Layer 1 기법을 잘못 적용하려 드는 것. Layer 2는 이미 자체 CoT 규약(ultrathink) 이 존재함.
- 같은 "Claude"라는 이름 아래 세 레이어가 다른 동작을 한다는 걸 의식해야 한다.

**공통 원칙 — 세 레이어 모두에서 유효**

- instructions over constraints (하지 마라 < X를 해라)
- 출력 포맷을 명시적으로 지시
- few-shot / 좋은 예시·나쁜 예시 제시
- 버전 관리, 반복 실험, 기록

### **이 발표는 이후 전부 Layer 1 기준**

---

## 3. 출력 제어 파라미터

### 3.1 Temperature

- 0에 가까울수록 결정론적 (가장 확률 높은 토큰 선택), 1에 가까울수록 다양성 증가.
- 사실·분류·파싱 작업은 `0` ~ `0.2`, 창의적 작업은 `0.7` ~ `0.9`.
- 주의: `temperature=0`도 완전 결정론적이지는 않다 (tie-breaking 존재).

### 3.2 Top-K / Top-P

- Top-K: 확률 상위 K개 토큰만 후보. `K=1`은 greedy와 동일.
- Top-P (nucleus): 누적 확률 P까지의 토큰만 후보. 보통 `0.9` ~ `0.99`.
- Top-K, Top-P는 temperature와 함께 쓸 때 서로의 효과를 일부 무효화할 수 있다. 하나를 주력으로 튜닝하는 게 낫다.

백서 권장 기본값

|용도|temperature|top-P|top-K|
|---|---|---|---|
|균형잡힌 일반 작업|0.2|0.95|30|
|창의적인 출력|0.9|0.99|40|
|덜 창의적인 출력|0.1|0.9|20|
|정답이 하나인 작업 (math 등)|0|-|-|

### 3.3 Max tokens / Output length

- 비용 = 토큰 수. 너무 크게 잡으면 지갑이 녹고, 너무 작게 잡으면 잘린 JSON으로 파싱 실패.
- "간결하게 답해라"는 프롬프트 문구 + `max_tokens` 상한 둘 다 쓰는 게 안전.

### 3.4 Reasoning effort / Thinking budget (추론 전용 신규 파라미터)

- GPT-5 계열: `reasoning_effort = minimal | low | medium | high` (GPT-5.1/5.2는 `none` 가능, `minimal` 불가).
- GPT-5 계열: `verbosity = low | medium | high` 로 "답변 길이"만 따로 제어.
- Claude: `thinking.budget_tokens` 로 "추론에 쓸 토큰 예산"을 직접 지정. 활성화하면 temperature는 자동으로 1로 고정된다.

정리: "깊게 생각해라"와 "답은 짧게 해라"는 별개의 축이다. 최신 모델은 둘을 분리해서 제공한다.

---

## 4. 프롬프트 기법 총정리

### 4.1 기본 3종

- **Zero-shot**: 예시 없이 지시만. "다음 리뷰를 긍정/부정/중립으로 분류하라."
- **One-shot**: 예시 1개를 함께.
- **Few-shot**: 예시 3~5개. 분류·구조화 출력에서 정확도가 가장 크게 뛴다.

Few-shot 팁

- 예시는 **다양하게**. 한쪽 클래스로 치우치면 모델이 그쪽으로 편향된다.
- 엣지 케이스를 하나쯤 넣으면 robust 해진다.
- 예시 순서를 섞어라 (특히 분류).

### 4.2 System / Role / Contextual prompting

- **System**: 모델의 역할·규칙·출력 포맷을 상단에 고정. "너는 X만 답한다. 출력은 반드시 JSON."
- **Role**: 페르소나. "시니어 백엔드 엔지니어로서..."
- **Contextual**: 배경 정보 주입. "이 회사는 헬스케어 앱이고 주 사용자는 40대..."

### 4.3 Step-back prompting

한 단계 추상화된 질문을 먼저 던지고 그 답을 context로 다시 넣는다. 예: "이 버그를 고쳐줘" → 먼저 "일반적으로 이런 종류 버그의 원인은 무엇인가?" 를 묻고, 답을 prepend 한 뒤 다시 원본 질문.

### 4.4 Chain of Thought (CoT)

"Let's think step by step." 한 줄만 넣어도 reasoning 정확도가 오른다. 단점: 토큰 많이 먹는다.

어디에 쓰고 어디에 안 쓰는지 구분이 중요하다.

- **추론 모델 (내부에서 이미 CoT를 돌림 → CoT 불필요)**
    - OpenAI: `o1`, `o3`, GPT-5 계열 (`GPT_5`, `GPT_5_MINI`, `GPT_5_NANO`, `GPT_5_1`, `GPT_5_2`)
    - Claude: `thinking_budget_tokens > 0` 으로 켠 상태
- **비추론 모델 (CoT가 여전히 효과 있음)**
    - OpenAI: `GPT_4O`, `GPT_4O_MINI`, `GPT_4_1`, `GPT_4_1_NANO`
    - Claude: Opus / Sonnet / Haiku 를 `thinking_budget_tokens` 없이 호출 (우리 코드의 현재 기본 상태)

참고: 우리 코드에는 `thinking_budget_tokens`를 세팅하는 호출부가 아직 없다. 즉 Claude 모델들도 현재는 전부 비추론 모드로 돌고 있고, CoT를 얹을 여지가 남아 있다.

### 4.5 Self-consistency

같은 CoT 프롬프트를 N번 돌리고 **다수결**로 답 선택. 비싸지만 정답률 상승이 크다. 분류·수학 문제에서 특히 효과적.

### 4.6 Tree of Thoughts (ToT)

여러 reasoning 경로를 동시에 탐색하고 중간에 가지치기. 복잡한 계획·탐색 문제에 적합. 구현 복잡도는 높음.

### 4.7 ReAct

**Reason + Act**. "생각 → 도구 호출 → 관찰 → 다시 생각..."의 루프. 우리가 MCP / tool calling 으로 하고 있는 게 이것의 한 형태.

### 4.8 Automatic Prompt Engineering (APE)

"이 태스크를 풀 프롬프트 10개를 써줘" 를 LLM 한테 시키고, 가장 성능 좋은 걸 선택. 프롬프트를 직접 쓰는 것보다 나은 경우가 꽤 있다.

---

## 5. 머니워크 코드로 배워보자

위치: `packages/server-common/src/services/ai/`

```
ai/
  clients.ts                # OpenAI / Anthropic 클라이언트 생성 + timeout 상수
  types.ts                  # AiModelType, AskAiParams(union), AiUseCase 등
  method/ask.ts             # askAi, askAiPatiently, askAiWithNoCircuitBreaker
  provider/
    index.ts                # model → provider 매핑 (registry)
    openAi.ts               # OpenAI 구현
    claude.ts               # Claude 구현
    types.ts                # AiProvider 인터페이스
```

### 5.1 공개 API

- `askAi` — 10초 timeout. 사용자 응답 기다리는 동기 요청에 적합.
- `askAiPatiently` — 60초 timeout. 백그라운드·비동기 파이프라인용.
- `askAiWithNoCircuitBreaker` — 논프로덕션 전용. 디버깅·스크립트용.

### 5.2 타입으로 파라미터를 강제한다

`AskAiParams` 는 discriminated union이다. 모델 족에 따라 허용 파라미터가 다르다.

|모델 족|temperature|reasoning_effort|verbosity|thinking_budget_tokens|
|---|---|---|---|---|
|GPT-4o / 4.1 계열|yes|no|no|no|
|GPT-5 / 5-mini / 5-nano|no|yes (`minimal` 포함)|yes|no|
|GPT-5.1 / 5.2|no|yes (`none` 포함, `minimal` 제외)|yes|no|
|Claude Opus/Sonnet/Haiku|yes|no|no|yes|

잘못된 모델-파라미터 조합은 **컴파일 타임에 막힌다**. 이 구조는 그대로 유지하고 새 파라미터를 union에 추가하는 방향으로 확장하면 된다.

### 5.3 현재 사용 예시

Notion 티켓 요약 — 가장 단순한 형태.

```tsx
const aiResponse = await askAi({
  aiUseCase: AiUseCase.NOTION_TICKET_SUMMARY,
  model: AiModelType.GPT_4O_MINI,
  text: prompt,
});
```

걸음수 인증 이미지 검증 — 이미지 포함.

```tsx
const reply = await askAi({
  aiUseCase: AiUseCase.STEP_IMAGE_VERIFY,
  model: AiModelType.GPT_4O,
  text: prompt,
  imageFileInfo: { file: buffer, detail: AiImageDetail.AUTO },
});
```

식단 비교 — 이미지 여러 장.

```tsx
await askAiPatiently({
  aiUseCase: AiUseCase.AI_DIETS,
  model: modelType,
  text: prompt,
  imageFileInfo: [
    { file: beforeImageBuffer, detail: AiImageDetail.HIGH },
    { file: afterImageBuffer,  detail: AiImageDetail.HIGH },
  ],
});
```

번역 — GPT-5.1에 reasoning_effort / verbosity 분기.

```tsx
await askAiPatiently({
  aiUseCase: AiUseCase.AUTO_TRANSLATION,
  text: prompt,
  model,
  ...(model === AiModelType.GPT_5_1 && {
    reasoning_effort: "low",
    verbosity: "low",
  }),
});
```

CS 챗봇 라우터 — reasoning/verbosity 튜닝된 케이스.

```tsx
askAiPatiently({
  aiUseCase: AiUseCase.CUSTOMER_SUPPORT_ROUTER,
  model,
  text: prompt,
  reasoning_effort: routerReasoningEffort ?? "low",
  verbosity:        routerVerbosity ?? "low",
});
```

---

## 6. 지금 우리가 쓰는 / 안 쓰는 파라미터

### 6.1 쓰고 있음

- `model`, `text`, `aiUseCase` — 기본.
- `imageFileInfo` (+ `detail`) — 비전 태스크.
- `temperature` — GPT-4o 계열과 Claude에서 선택적.
- `reasoning_effort`, `verbosity` — GPT-5 계열 일부 (CS 라우터, 쿠폰 번역).
- `thinking_budget_tokens` — **코드상 지원하지만 현재 호출부에서 아무도 안 씀.**

### 6.2 우리가 안 쓰는 파라미터 (OpenAI SDK 기준)

|파라미터|지원 모델|역할|도입 시 기대 효과|
|---|---|---|---|
|`top_p`|대부분|확률 누적 컷오프|같은 temperature에서 **변동성 미세 조정**. 지금은 temperature 하나에만 의존 중.|
|`top_k`|일부|상위 K 토큰만 고려|백서 권장 기본값(30/40)을 못 쓰고 있음. 창의 vs 정확 스펙트럼 세밀화.|
|`max_tokens`|전부|출력 상한|**비용 상한 강제**. 지금 Claude는 4096으로 하드코딩, OpenAI는 미지정.|
|`stop` / `stop_sequences`|전부|중단 토큰 지정|JSON 파싱 시 뒷꼬리 텍스트 방지, streaming 구조화에 유리.|
|`frequency_penalty`, `presence_penalty`|OpenAI only|반복/신규 토큰 페널티|번역·요약 반복 억제. 지금은 반복 나와도 그대로 통과.|
|`seed`|OpenAI|결정론 재현|**재현 가능한 테스트**. 동일 입력 → 동일 출력으로 회귀 테스트 가능.|
|`n`|OpenAI|N개 샘플 동시 생성|Self-consistency 구현 시 한 번 호출로 N개 확보.|
|`logprobs` / `top_logprobs`|OpenAI|토큰별 확률|"모델이 얼마나 확신하는가"로 **confidence threshold** 설정 가능. 애매한 답만 사람에게 넘기기.|
|`response_format: { type: "json_schema", ... }`|OpenAI|**Structured Output** (스키마 강제)|지금 `json_object`만 씀. 스키마 강제로 바꾸면 파싱 실패·repair 로직이 사실상 사라진다.|
|`tools` / `tool_choice`|전부|Function calling|우리는 MCP 밖에서는 안 쓰고 있음. 단순 라우팅·검증을 tool call 한 방으로 바꾸면 JSON 파싱 자체가 불필요.|

### 6.3 특히 지금 당장 만져도 좋은 세 가지

1. **`response_format: { type: "json_schema" }` 도입 (OpenAI)**
    - 현재 `json_object` 모드는 "JSON이긴 하다"까지만 보장. 스키마 어긋나면 런타임에서 터진다.
    - JSON schema 모드는 모델 디코딩 단계에서 스키마를 강제하므로 파싱 실패가 원천 차단.
    - 적용 대상: 요약·분류·라우팅 파이프라인 전반.
2. **`seed` + 고정 temperature로 회귀 테스트**
    - 프롬프트 바꿨을 때 "정말 좋아졌나?"를 수치로 비교 가능해진다.
    - CS 라우터처럼 결과가 분기에 영향을 주는 곳에 특히 유용.
3. **`max_tokens` 명시 + `stop` 사용**
    - Claude쪽은 지금 `4096` 하드코딩. thinking 켤 때만 조정됨. use case별로 실제 필요량 기반 설정 필요.
    - OpenAI쪽은 상한 미지정 상태. 악성 긴 응답 → 비용 튐 리스크.

### 요약 표

|항목|OpenAI|Claude|
|---|---|---|
|JSON 강제|`response_format: json_object` / `json_schema`|system prompt 로 강제 (우리 구현)|
|추론 제어|`reasoning_effort`, `verbosity` (enum)|`thinking.budget_tokens` (숫자)|
|이미지 detail|low / high / auto 지원|미지원 (우리 코드에서 무시)|
|max_tokens 기본|모델 기본|우리 코드에서 `4096` 하드코딩|
|thinking 시 온도|자유|1로 고정 (우리 코드가 자동 제외)|
|인증|env var|Secrets Manager lazy load|
|Structured Output|json_schema 지원|미지원 (tool use로 우회)|

---

## 폴 Interview

- 프롬프트 엔지니어링에서 가장 중요하다고 생각하는 지점은 무엇인지?
    - 모호한 지점을 없애는 것
        - 구체적인 예시와 지시를 내리는 것. 명확한 아웃풋을 내도록 지시하는 것
    - 어떤 input 을 넣었을 때, 어떤 output 이 나오는 지 정하고 측정하는 것이 제일 중요
        - 평가지표를 어떻게 설정할지가 제일 중요함
        - 예시로, 어떤 기업들은 input 과 output 만 정해두고 prompt 가 좋아질 때까지 ai 로 프롬프트를 수정해가며 평가한다고함
- Q. prompt engineering 을 잘하면 추론 모델을 사용하지 않고, 비추론 모델을 사용하는 것이 비용적이나 응답 속도 측면에서 유리한 경우도 있을 것 같은데 어떻게 선택하고 판단을 내리는지?
    - 충분히 그렇게 생각할 수 있다. 하지만 ai 는 대개로 최신 모델이 제일 잘한다. 응답 속도를 고려해도 최신 모델의 정확도와 답변의 퀄리티 자체가 다르기 때문에 최신 모델을 사용하는 것이 좋고, 그래서 추론 모델을 보통 사용한다.