# n8n Webhook 데이터 처리 워크플로우 구축 가이드

## 목차
1. [개요](#개요)
2. [학습 목표](#학습-목표)
3. [사전 준비사항](#사전-준비사항)
4. [워크플로우 구조](#워크플로우-구조)
5. [단계별 구현](#단계별-구현)
6. [트러블슈팅](#트러블슈팅)
7. [테스트 방법](#테스트-방법)
8. [핵심 개념 정리](#핵심-개념-정리)

---

## 개요

이 가이드는 n8n을 사용하여 Webhook으로 데이터를 받아 처리하고 응답하는 워크플로우를 구축하는 방법을 단계별로 설명합니다.

### 구현할 기능
- POST 요청으로 데이터 수신
- 원본 데이터 보존
- 데이터 변환 (메시지, 타임스탬프, 대문자 변환)
- JSON 형식으로 응답 반환

---

## 학습 목표

이 튜토리얼을 완료하면 다음을 할 수 있습니다:

- ✅ n8n에서 Webhook 노드 설정하기
- ✅ Edit Fields 노드를 사용한 데이터 변환
- ✅ 중첩된 객체 구조 생성하기
- ✅ JavaScript 표현식 활용하기
- ✅ Webhook 응답 설정하기
- ✅ 워크플로우 테스트 및 디버깅하기

---

## 사전 준비사항

### 필수 도구
- n8n (설치 완료 및 실행 중)
- 브라우저 (Chrome, Firefox 등)
- curl 또는 Postman (테스트용, 선택사항)

### 필요한 지식
- 기본적인 JSON 이해
- HTTP 메서드 (GET, POST) 개념
- JavaScript 기초 문법 (선택사항)

---

## 워크플로우 구조

```
[Webhook] → [Edit Fields] → [Edit Fields1] → [Respond to Webhook]
    ↓              ↓                ↓                    ↓
  데이터 수신    원본 보존      데이터 변환          응답 반환
```

### 노드별 역할

| 노드 | 역할 | 주요 설정 |
|------|------|-----------|
| **Webhook** | POST 요청 수신 | HTTP Method: POST |
| **Edit Fields** | 데이터 통과 | Include Other Input Fields: 활성화 |
| **Edit Fields1** | 데이터 변환 | 3개 필드 생성 (message, timestamp, uppercase) |
| **Respond to Webhook** | 응답 반환 | First Incoming Item |

---

## 단계별 구현

### Step 1: 새 워크플로우 생성

1. n8n 대시보드에서 **새 워크플로우** 생성
2. 워크플로우 이름 설정: "Webhook Data Processing"

---

### Step 2: Webhook 노드 추가 및 설정

#### 2.1 노드 추가
1. 캔버스에서 **+** 버튼 클릭
2. "Webhook" 검색 후 선택

#### 2.2 기본 설정
```
HTTP Method: POST
Path: (자동 생성됨, 예: 65420b95-7bed-412b-9773-d4fc0c8ad16c)
Authentication: None
Response: Using 'Respond to Webhook' Node
```

#### 2.3 Mock 데이터 설정 (테스트용)
1. Webhook 노드 열기
2. **"set mock data"** 클릭
3. 다음 JSON 입력:
```json
[
  {
    "name": "First item",
    "code": 1,
    "body": {
      "message": "Hello World"
    }
  }
]
```

#### 2.4 URL 확인
- **Test URL**: `http://localhost:5678/webhook-test/[path]`
- **Production URL**: `http://localhost:5678/webhook/[path]`

> ⚠️ **중요**: 발행된 워크플로우는 Production URL을 사용해야 합니다!

---

### Step 3: Edit Fields 노드 추가 (데이터 통과)

#### 3.1 노드 추가
1. Webhook 노드 우측에 **Edit Fields** 노드 추가
2. 연결선으로 Webhook → Edit Fields 연결

#### 3.2 설정
```
Mode: Manual Mapping
Fields to Set: (비어있음)
Include Other Input Fields: All (활성화)
```

> 💡 **핵심 포인트**: "Include Other Input Fields"를 활성화하면 입력 데이터가 그대로 다음 노드로 전달됩니다.

---

### Step 4: Edit Fields1 노드 추가 (데이터 변환)

#### 4.1 노드 추가
1. Edit Fields 노드 우측에 **Edit Fields** 노드 추가
2. 노드 이름을 "Edit Fields1"로 변경 (선택사항)

#### 4.2 필드 설정

**필드 1: 메시지**
```
Field Name: processed.message
Type: String
Value: ={{ $json.body.message }}
```

**필드 2: 타임스탬프**
```
Field Name: processed.timestamp
Type: String
Value: ={{ new Date().toISOString() }}
```

**필드 3: 대문자 변환**
```
Field Name: processed.uppercase
Type: String
Value: ={{ $json.body.message?.toUpperCase() }}
```

#### 4.3 중첩 객체 생성 원리

필드명에 점(`.`)을 사용하면 자동으로 중첩 객체가 생성됩니다:

```javascript
// 필드 설정
processed.message = "Hello World"
processed.timestamp = "2026-01-25T10:00:00.000Z"
processed.uppercase = "HELLO WORLD"

// 결과 JSON
{
  "processed": {
    "message": "Hello World",
    "timestamp": "2026-01-25T10:00:00.000Z",
    "uppercase": "HELLO WORLD"
  }
}
```

---

### Step 5: Respond to Webhook 노드 추가

#### 5.1 노드 추가
1. Edit Fields1 노드 우측에 **Respond to Webhook** 노드 추가
2. 연결선으로 Edit Fields1 → Respond to Webhook 연결

#### 5.2 설정
```
Respond With: First Incoming Item
```

> 💡 이 설정으로 Edit Fields1에서 변환된 데이터가 응답으로 전송됩니다.

---

### Step 6: 워크플로우 테스트

#### 6.1 Mock 데이터로 테스트
1. **"Execute workflow"** 버튼 클릭
2. 각 노드의 결과 확인
3. 오류가 있다면 해당 노드 클릭하여 상세 내용 확인

#### 6.2 예상 결과 확인

**Webhook 노드 출력:**
```json
{
  "body": {
    "message": "Hello World"
  },
  "headers": {...},
  ...
}
```

**Edit Fields1 노드 출력:**
```json
{
  "processed": {
    "message": "Hello World",
    "timestamp": "2026-01-25T10:48:51.592Z",
    "uppercase": "HELLO WORLD"
  },
  "body": {...}
}
```

**Respond to Webhook 노드 출력:**
```json
{
  "processed": {
    "message": "Hello World",
    "timestamp": "2026-01-25T10:48:51.592Z",
    "uppercase": "HELLO WORLD"
  }
}
```

---

### Step 7: 워크플로우 발행

1. 우측 상단 **"Publish"** 버튼 클릭
2. 버전 이름 입력 (예: "Version 1")
3. **"Publish"** 확인

> ✅ 발행 후 상태가 "Published"로 변경되어야 합니다.

---

## 트러블슈팅

### 문제 1: 'original_data' expects a object but we got '[object Object]'

**원인**: Edit Fields 노드에서 Type: Object로 설정하고 `={{ $json.body }}`를 사용하면 객체가 문자열로 변환됨

**해결방법**:
```
방법 1: Include Other Input Fields 활성화
- Fields to Set을 비워두고
- Include Other Input Fields: All 활성화

방법 2: 개별 필드로 설정
- processed.message: ={{ $json.body.message }}
- processed.timestamp: ={{ new Date().toISOString() }}
```

---

### 문제 2: 404 Error on Test URL

**원인**: 워크플로우 발행 후 Test URL을 사용

**해결방법**:
```
발행 전: http://localhost:5678/webhook-test/[path]
발행 후: http://localhost:5678/webhook/[path]

Production URL을 사용하세요!
```

---

### 문제 3: Connection Refused (curl 실행 시)

**원인**: n8n 서버가 외부 연결을 차단하거나 네트워크 설정 문제

**해결방법**:
```bash
# 1. n8n이 실행 중인지 확인
ps aux | grep n8n

# 2. 브라우저에서 테스트
# n8n 워크플로우 화면에서 "Execute workflow" 사용

# 3. 또는 브라우저 개발자 도구에서 fetch 사용
fetch('http://localhost:5678/webhook/[path]', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: 'Hello World' })
})
```

---

### 문제 4: Expression Error

**원인**: JavaScript 표현식 문법 오류

**해결방법**:
```javascript
// ❌ 잘못된 예
{{ $json.body.message.toUpperCase() }}

// ✅ 올바른 예 (옵셔널 체이닝 사용)
{{ $json.body.message?.toUpperCase() }}

// ✅ 또는 기본값 설정
{{ ($json.body.message || '').toUpperCase() }}
```

---

## 테스트 방법

### 방법 1: curl 명령어

```bash
curl -X POST http://localhost:5678/webhook/65420b95-7bed-412b-9773-d4fc0c8ad16c \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello World"}'
```

**예상 응답:**
```json
{
  "processed": {
    "message": "Hello World",
    "timestamp": "2026-01-25T10:48:51.592Z",
    "uppercase": "HELLO WORLD"
  }
}
```

---

### 방법 2: HTML 테스트 페이지

```html
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <title>Webhook 테스트</title>
</head>
<body>
    <h1>n8n Webhook 테스트</h1>
    <button onclick="testWebhook()">테스트 실행</button>
    <pre id="result"></pre>

    <script>
        async function testWebhook() {
            const response = await fetch('http://localhost:5678/webhook/[YOUR-PATH]', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ message: 'Hello World' })
            });

            const data = await response.json();
            document.getElementById('result').textContent =
                JSON.stringify(data, null, 2);
        }
    </script>
</body>
</html>
```

---

### 방법 3: Postman

1. **New Request** 생성
2. 설정:
   ```
   Method: POST
   URL: http://localhost:5678/webhook/[YOUR-PATH]
   Headers: Content-Type: application/json
   Body (raw JSON):
   {
     "message": "Hello World"
   }
   ```
3. **Send** 클릭

---

### 방법 4: n8n 내장 테스트

1. Webhook 노드 열기
2. **"Listen for test event"** 클릭
3. 별도 터미널이나 Postman으로 POST 요청 전송
4. n8n에서 자동으로 데이터 캡처 및 실행

---

## 핵심 개념 정리

### 1. n8n 표현식 (Expression)

n8n에서 동적 데이터를 참조할 때 사용:

```javascript
// 기본 문법
={{ 표현식 }}

// 데이터 참조
$json          // 현재 아이템 데이터
$json.body     // body 필드
$node         // 특정 노드 데이터
$now          // 현재 시간

// JavaScript 함수 사용
={{ new Date().toISOString() }}
={{ $json.text.toUpperCase() }}
={{ $json.price * 1.1 }}
```

---

### 2. Include Other Input Fields

이 옵션의 동작 방식:

```javascript
// Include Other Input Fields: OFF
// 출력: 설정한 필드만
{
  "newField": "value"
}

// Include Other Input Fields: ON
// 출력: 기존 필드 + 새 필드
{
  "existingField": "original",
  "newField": "value"
}
```

---

### 3. 중첩 객체 생성

점(`.`) 표기법으로 자동 생성:

```javascript
// 필드 설정
user.name = "John"
user.email = "john@example.com"
user.age = 30

// 결과
{
  "user": {
    "name": "John",
    "email": "john@example.com",
    "age": 30
  }
}
```

---

### 4. Webhook URL 타입

| URL 타입 | 사용 시기 | 경로 |
|----------|-----------|------|
| **Test URL** | 개발/테스트 중 | `/webhook-test/[path]` |
| **Production URL** | 워크플로우 발행 후 | `/webhook/[path]` |

---

### 5. HTTP 응답 코드

n8n Webhook의 기본 동작:

| 상황 | 응답 코드 | 설명 |
|------|-----------|------|
| 성공 | 200 OK | 정상 처리 |
| 노드 오류 | 500 Internal Server Error | 워크플로우 실행 오류 |
| 잘못된 경로 | 404 Not Found | Webhook 경로가 존재하지 않음 |
| 잘못된 메서드 | 404 Not Found | GET 대신 POST 필요 |

---

## 고급 활용 예제

### 예제 1: 조건부 처리

```javascript
// IF 노드 추가하여 메시지 길이 확인
{{ $json.body.message.length > 10 }}

// 긴 메시지와 짧은 메시지 다르게 처리
```

---

### 예제 2: 여러 필드 변환

```javascript
// Edit Fields1 노드에서
processed.message = {{ $json.body.message }}
processed.timestamp = {{ new Date().toISOString() }}
processed.uppercase = {{ $json.body.message?.toUpperCase() }}
processed.lowercase = {{ $json.body.message?.toLowerCase() }}
processed.length = {{ $json.body.message?.length }}
processed.reversed = {{ $json.body.message?.split('').reverse().join('') }}
```

---

### 예제 3: 다중 입력 처리

```javascript
// body에 배열이 있는 경우
{
  "messages": ["Hello", "World", "Test"]
}

// 각 메시지 처리
{{ $json.body.messages.map(m => m.toUpperCase()) }}
```

---

### 예제 4: 에러 처리

```javascript
// 옵셔널 체이닝으로 안전하게 처리
{{ $json.body?.message?.toUpperCase() || 'DEFAULT' }}

// null 체크
{{ $json.body.message !== null ? $json.body.message : 'No message' }}
```

---

## 모범 사례 (Best Practices)

### ✅ DO (권장사항)

1. **명확한 노드 이름 사용**
   ```
   ❌ Edit Fields
   ✅ Transform User Data
   ```

2. **옵셔널 체이닝 사용**
   ```javascript
   ✅ {{ $json.body?.message?.toUpperCase() }}
   ❌ {{ $json.body.message.toUpperCase() }}
   ```

3. **워크플로우에 설명 추가**
   - 각 노드에 Notes 추가
   - 복잡한 표현식에 주석 달기

4. **테스트 데이터 설정**
   - Mock 데이터를 실제 사용 케이스와 유사하게 설정

5. **버전 관리**
   - 발행 시 의미있는 버전 이름 사용
   - 변경사항 설명 추가

---

### ❌ DON'T (피해야 할 것)

1. **복잡한 로직을 한 노드에 몰아넣기**
   ```
   ❌ 하나의 Edit Fields에서 모든 변환 시도
   ✅ 단계별로 노드 분리
   ```

2. **에러 처리 없이 표현식 사용**
   ```javascript
   ❌ {{ $json.data.user.name }}
   ✅ {{ $json.data?.user?.name || 'Unknown' }}
   ```

3. **Test URL로 프로덕션 사용**
   ```
   ❌ 발행 후에도 webhook-test 사용
   ✅ Production URL 사용
   ```

---

## 추가 학습 자료

### 공식 문서
- [n8n 공식 문서](https://docs.n8n.io/)
- [Webhook 노드 가이드](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
- [Edit Fields 노드 가이드](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set/)

### 커뮤니티
- [n8n 커뮤니티 포럼](https://community.n8n.io/)
- [n8n GitHub](https://github.com/n8n-io/n8n)

### 관련 주제
- RESTful API 설계
- JSON 데이터 구조
- JavaScript 기초
- HTTP 프로토콜

---

## 연습 과제

### 과제 1: 기본
다음 입력을 받아 처리하는 워크플로우 작성:
```json
{
  "name": "John Doe",
  "age": 30
}
```

**목표 출력:**
```json
{
  "processed": {
    "fullName": "John Doe",
    "ageNextYear": 31,
    "isAdult": true
  }
}
```

---

### 과제 2: 중급
배열 데이터를 처리하는 워크플로우 작성:
```json
{
  "items": [
    {"name": "Apple", "price": 100},
    {"name": "Banana", "price": 50}
  ]
}
```

**목표 출력:**
```json
{
  "processed": {
    "total": 150,
    "itemCount": 2,
    "items": [...]
  }
}
```

---

### 과제 3: 고급
조건부 처리를 포함한 워크플로우:
- 메시지 길이가 10자 이상이면 대문자로 변환
- 10자 미만이면 소문자로 변환
- 빈 메시지는 "No message" 반환

---

## 마무리

이 가이드를 통해 n8n으로 Webhook 데이터 처리 워크플로우를 구축하는 방법을 배웠습니다.

### 학습한 내용
✅ Webhook 노드 설정 및 테스트
✅ Edit Fields를 통한 데이터 변환
✅ JavaScript 표현식 활용
✅ 중첩 객체 구조 생성
✅ 워크플로우 디버깅 및 트러블슈팅

### 다음 단계
- 데이터베이스 연동 (MySQL, PostgreSQL)
- 외부 API 호출 (HTTP Request 노드)
- 조건부 분기 (IF 노드)
- 스케줄링 (Cron 노드)
- 에러 핸들링 (Error Trigger 노드)

---

**작성일**: 2026-01-25
**버전**: 1.0
**난이도**: 초급~중급
**예상 소요 시간**: 30-45분

---

© 2026 n8n Workflow Tutorial. 이 문서는 교육 목적으로 자유롭게 사용 및 공유할 수 있습니다.
