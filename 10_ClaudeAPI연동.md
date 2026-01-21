# 10장: Claude API 연동

> n8n에서 Claude API를 사용하여 AI 기능을 추가합니다.
> 예상 학습 시간: 1.5시간

## Claude API란?

Claude는 Anthropic에서 만든 고급 AI 모델입니다.

### Claude의 특징

```
✅ 한국어 지원
✅ 창의적 작문
✅ 코드 분석 및 생성
✅ 데이터 분석
✅ 긴 문맥 이해 (200K 토큰)
✅ JSON 구조화된 출력
✅ 함수 호출(Function Calling)
```

### Claude API 모델

```
claude-3-5-sonnet (가장 균형잡힘 - 추천)
claude-3-5-haiku (빠름 - 간단한 작업)
claude-3-opus (강력함 - 복잡한 작업)
```

---

## Claude API 설정

### 1단계: API 키 확인

당신은 이미 Claude API 결제를 완료했습니다!

API 키는 다음에서 확인:
- https://console.anthropic.com

### 2단계: n8n에 Credential 등록

#### Credential 생성 방법

1. n8n 좌측 메뉴 → **Credentials**
2. **"+ Add new"** 버튼
3. **"Anthropic"** 검색 후 선택 (또는 HTTP로 수동 설정)

```
Credential 정보:
- Name: "My Claude API"
- API Key: sk-ant-... (당신의 키)
```

#### 또는 HTTP로 수동 설정

```
Method: POST
URL: https://api.anthropic.com/v1/messages
Headers:
  - Content-Type: application/json
  - x-api-key: {{ $env.CLAUDE_API_KEY }}
  - anthropic-version: 2023-06-01
```

---

## Claude API 호출 기본

### n8n에서 HTTP로 Claude 호출하기

#### 간단한 예시

```json
{
  "method": "POST",
  "url": "https://api.anthropic.com/v1/messages",
  "headers": {
    "Content-Type": "application/json",
    "x-api-key": "{{ $env.CLAUDE_API_KEY }}",
    "anthropic-version": "2023-06-01"
  },
  "body": {
    "model": "claude-3-5-sonnet-20241022",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "Hello Claude! What's your name?"
      }
    ]
  }
}
```

### n8n 워크플로우로 구현

#### 단계 1: Webhook 트리거
```
- HTTP Method: POST
```

#### 단계 2: HTTP 노드 설정
```
메서드: POST
URL: https://api.anthropic.com/v1/messages

Headers:
- Content-Type: application/json
- x-api-key: {{ $env.CLAUDE_API_KEY }}
- anthropic-version: 2023-06-01

Body (JSON):
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "{{ $json.body.message }}"
    }
  ]
}
```

#### 단계 3: 응답 처리 (Set)
```json
{
  "ai_response": "{{ $json.body.content[0].text }}",
  "model": "{{ $json.body.model }}",
  "usage": {
    "input_tokens": "{{ $json.body.usage.input_tokens }}",
    "output_tokens": "{{ $json.body.usage.output_tokens }}"
  }
}
```

---

## 실제 워크플로우 예시

### 예시 1: 간단한 질문 답변

```
워크플로우 이름: "Claude Q&A"

[Webhook POST] 
  ↓
[HTTP - Claude 호출]
  POST: https://api.anthropic.com/v1/messages
  Body: messages with user input
  ↓
[Set - 응답 처리]
  Extract: content[0].text
  ↓
[Slack 알림]
  Send: "Q: {question}\nA: {answer}"
```

**테스트:**
```bash
curl -X POST http://localhost:5678/webhook/[id] \
  -H "Content-Type: application/json" \
  -d '{
    "message": "파이썬에서 리스트를 정렬하는 방법은?"
  }'
```

### 예시 2: 텍스트 분석

```
워크플로우: "감정 분석"

[Webhook POST]
  ↓
[HTTP - Claude 분석]
  Body: {
    "messages": [{
      "role": "user",
      "content": "다음 텍스트의 감정을 분석해줘:\n\n{{ $json.body.text }}"
    }]
  }
  ↓
[Set - 결과 처리]
  {
    "text": "{{ $json.body.text }}",
    "sentiment": "{{ $json.body.content[0].text }}"
  }
  ↓
[Database 저장 또는 Slack 발송]
```

### 예시 3: 콘텐츠 생성

```
워크플로우: "블로그 제목 생성"

[Schedule - 매일 아침 9시]
  ↓
[HTTP - Claude 호출]
  Body: {
    "messages": [{
      "role": "user",
      "content": "다음 주제로 블로그 제목 5개를 만들어줘:\n{{ $json.body.topic }}"
    }]
  }
  ↓
[Set - 파싱]
  Extract titles
  ↓
[이메일 발송]
  To: your@email.com
  Body: Titles
```

---

## Claude API 심화 사용법

### 1. 시스템 프롬프트 (System Prompt)

Claude의 행동을 지정할 수 있습니다:

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "system": "You are a helpful Korean customer service assistant. Always respond in Korean.",
  "messages": [
    {
      "role": "user",
      "content": "안녕하세요"
    }
  ]
}
```

**n8n 구현:**
```
Body (JSON):
{
  "system": "{{ $json.body.system_prompt || 'You are a helpful assistant.' }}",
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "{{ $json.body.message }}"
    }
  ]
}
```

### 2. 멀티턴 대화 (Conversation)

대화 히스토리를 유지하는 방법:

```
[이전 메시지들 배열]
  ↓
[새 메시지 추가]
  ↓
[Claude 호출]
  ↓
[응답 저장]
  ↓
[히스토리 업데이트]
```

**구현 예시:**
```json
{
  "messages": [
    {
      "role": "user",
      "content": "안녕하세요"
    },
    {
      "role": "assistant",
      "content": "안녕하세요! 무엇을 도와드릴까요?"
    },
    {
      "role": "user",
      "content": "파이썬에 대해 알려줄래?"
    }
  ]
}
```

### 3. 구조화된 출력 (JSON Mode)

Claude에게 JSON 형식으로 응답하도록 요청:

```json
{
  "model": "claude-3-5-sonnet-20241022",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "다음 텍스트를 JSON 형식으로 분석해줘:\nText: '오늘 날씨가 좋아'\n\nJSON 형식:\n{\"sentiment\": \"\", \"emotion\": \"\", \"keywords\": []}"
    }
  ]
}
```

**n8n 파싱:**
```javascript
// Set 노드에서:
{{ JSON.parse($json.body.content[0].text) }}
```

### 4. 이미지 분석 (Vision)

이미지를 분석하려면:

```json
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "image",
          "source": {
            "type": "base64",
            "media_type": "image/jpeg",
            "data": "{{ $json.imageBase64 }}"
          }
        },
        {
          "type": "text",
          "text": "이 이미지를 설명해줘"
        }
      ]
    }
  ]
}
```

---

## 토큰과 비용 이해하기

### 토큰이란?

토큰 = 글자 단위의 기본 단위 (대략 4글자 = 1토큰)

```
예시:
"Hello" = 1토큰
"안녕하세요" = 약 2-3토큰
100단어 = 약 75토큰
```

### 비용 계산

```
Claude 3.5 Sonnet:
- Input: $3 / 1M tokens
- Output: $15 / 1M tokens

예시:
- 입력 1000 tokens + 출력 500 tokens
- 비용 = (1000 * $3/1M) + (500 * $15/1M) = $0.0105
```

### n8n에서 비용 모니터링

```json
{
  "input_tokens": "{{ $json.body.usage.input_tokens }}",
  "output_tokens": "{{ $json.body.usage.output_tokens }}",
  "cost": "{{ ($json.body.usage.input_tokens * 3 + $json.body.usage.output_tokens * 15) / 1000000 }}"
}
```

---

## 에러 처리

### 일반적인 에러

#### 1. 401 Unauthorized
```
원인: API 키 오류
해결:
- API 키 확인
- x-api-key 헤더 확인
- 환경 변수 설정 확인
```

#### 2. 400 Bad Request
```
원인: 요청 형식 오류
해결:
- JSON 형식 확인
- 필수 필드 확인 (model, messages)
- max_tokens 범위 확인 (1-4096)
```

#### 3. 429 Rate Limit
```
원인: 요청 속도 제한
해결:
- 요청 간격 증가
- 배치 처리
- 캐싱 활용
```

#### 4. 500 Internal Server Error
```
원인: Claude 서버 오류
해결:
- 재시도 (Retry)
- 나중에 다시 시도
```

### n8n 에러 처리 워크플로우

```
[HTTP - Claude 호출]
  ↓
[If - 에러 확인]
  - 성공: 다음 노드로
  - 실패: 에러 처리 노드로
  ↓
[에러 처리]
  - 로깅
  - 알림 (Slack, Email)
  - 재시도
```

---

## 성능 최적화

### 1. max_tokens 조절

```
적절한 max_tokens 설정:
- 짧은 응답: 256
- 중간 응답: 512-1024
- 긴 응답: 2048-4096
```

### 2. 캐싱

같은 요청을 자주 하면:

```
[캐시 확인]
  ↓ 없음
[Claude 호출]
  ↓
[결과 저장]
```

### 3. 배치 처리

여러 요청을 묶어서 처리:

```
[Loop 노드]
  ↓ 각 항목마다
[Claude 호출]
  ↓
[결과 수집]
```

---

## 실전 팁

### 프롬프트 작성 팁

1. **명확한 지시사항**
```
❌ 안 좋음: "이 텍스트를 분석해"
✅ 좋음: "다음 텍스트의 감정을 분석하고 JSON 형식으로 반환해. {sentiment: positive/negative/neutral, confidence: 0-1}"
```

2. **컨텍스트 제공**
```
"당신은 전문 블로그 작가입니다. 기술 블로그 독자를 위해 이 주제를 설명해주세요: ..."
```

3. **예시 제공 (Few-shot)**
```
"다음과 같이 분류해줘:
감정1: '좋아요!' → positive
감정2: '싫어요' → negative
텍스트: '훌륭합니다!' → ?"
```

### 디버깅 팁

1. **각 노드의 입출력 확인**
```
HTTP 노드 선택 → "Inspect" → Input/Output 탭에서 확인
```

2. **응답 구조 파악**
```
- response 전체 구조 확인
- content 배열의 첫 번째 요소 (content[0])
- text 필드에 실제 응답 저장
```

3. **토큰 사용량 확인**
```
Set 노드에서: {{ $json.body.usage }}
```

---

## 체크리스트

이 장을 완료했는지 확인하세요:

- [ ] Claude API 키가 준비되었다
- [ ] n8n에서 Claude API를 호출할 수 있다
- [ ] 기본 Q&A 워크플로우를 만들었다
- [ ] 응답을 올바르게 파싱할 수 있다
- [ ] 에러 처리를 이해했다
- [ ] 토큰과 비용을 이해했다
- [ ] 시스템 프롬프트를 사용할 수 있다

모두 완료했다면, 다음 장 [11_AI워크플로우.md](11_AI워크플로우.md)로 이동하세요!

---

## 다음 단계

**11_AI워크플로우.md**에서 배우는 것:
- 고급 AI 활용 패턴
- 다단계 AI 처리
- AI 결과를 다른 서비스와 연동
- 실제 비즈니스 유스케이스
