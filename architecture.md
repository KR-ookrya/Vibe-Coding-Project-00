# Architecture — AI 자기소개서 첨삭 앱

---

## 전체 구조

```
┌─────────────────────────────────────────────────────┐
│                    Browser (Client)                  │
│           public/index.html  +  style.css            │
│                  Vanilla JS (EventSource)            │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP / SSE
┌──────────────────────▼──────────────────────────────┐
│                 Node.js + Express                    │
│                    server.js                         │
│                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ Rate Limiter│  │Input Sanitise│  │Share Store │  │
│  │ 10req/60s   │  │ XSS 방지     │  │ 24h TTL    │  │
│  └─────────────┘  └──────────────┘  └────────────┘  │
│                                                      │
│  POST /api/analyze   →  SSE 스트리밍                  │
│  POST /api/rewrite   →  SSE 스트리밍                  │
│  POST /api/share     →  JSON                         │
│  GET  /api/share/:id →  JSON                         │
└──────────────────────┬──────────────────────────────┘
                       │ HTTPS (Anthropic SDK)
┌──────────────────────▼──────────────────────────────┐
│               Anthropic Claude API                   │
│         Model: claude-opus-4-7                       │
│         Prompt Caching (ephemeral)                   │
│         Max Tokens: 4096 (analyze) / 1024 (rewrite)  │
└─────────────────────────────────────────────────────┘
```

---

## 디렉터리 구조

```
ai-cover-letter-editor/
├── server.js                  # 서버 진입점 (Express + API 라우트 전체)
├── package.json
├── .env                       # 환경변수 (gitignore 대상)
├── .env.example               # 환경변수 템플릿
├── public/
│   ├── index.html             # 단일 페이지 UI
│   └── style.css              # 스타일
├── docs/
│   ├── setup.md               # 개발 환경 셋업 가이드
│   └── architecture.md        # 이 문서
└── .planning/
    └── decisions/
        ├── ADR-0001-mobile-framework.md
        ├── ADR-0002-state-management.md
        └── ADR-0003-backend-choice.md
```

---

## API 명세

### `POST /api/analyze`
자기소개서 전체 첨삭 — SSE 스트리밍 응답

**Request Body**
```json
{
  "text":     "자기소개서 본문 (최소 50자)",
  "jobTitle": "지원 직무 (선택)",
  "company":  "지원 회사 (선택)"
}
```

**Response** — `text/event-stream`
```
data: {"text": "## 📊 종합 평가\n..."}
data: {"text": "계속 스트리밍..."}
data: [DONE]
```

---

### `POST /api/rewrite`
특정 문단 재첨삭 — SSE 스트리밍 응답

**Request Body**
```json
{
  "paragraph": "재첨삭할 문단 (최소 10자)",
  "jobTitle":  "지원 직무 (선택)",
  "company":   "지원 회사 (선택)"
}
```

---

### `POST /api/share`
첨삭 결과 공유 링크 생성

**Request Body**
```json
{
  "result":   "첨삭 결과 전문",
  "jobTitle": "직무",
  "company":  "회사",
  "score":    85
}
```

**Response**
```json
{ "id": "a1b2c3d4e5f6" }
```

> 생성된 링크: `http://localhost:3000/share/a1b2c3d4e5f6`  
> TTL: **24시간** (인메모리 저장, 서버 재시작 시 초기화)

---

### `GET /api/share/:id`
공유된 첨삭 결과 조회

**Response**
```json
{
  "result":   "첨삭 결과 전문",
  "jobTitle": "직무",
  "company":  "회사",
  "score":    85
}
```

---

## 핵심 모듈

### Rate Limiter
- IP당 분당 **10회** 요청 제한
- `Map<ip, { count, resetAt }>` 인메모리 구조
- 2분마다 만료된 항목 자동 정리

### Input Sanitiser
- `<`, `>` 제거로 XSS 방지
- 자기소개서 본문: 최대 **5,000자**
- 직무·회사명: 최대 **200자**

### SSE 스트리밍
```
Client                        Server                   Claude API
  │                              │                          │
  │── POST /api/analyze ────────▶│                          │
  │                              │── messages.stream ──────▶│
  │◀── text/event-stream ────────│                          │
  │◀── data: {"text": "..."} ────│◀── content_block_delta ──│
  │◀── data: [DONE] ─────────────│◀── stream end ───────────│
```

### Prompt Caching
- System Prompt에 `cache_control: { type: "ephemeral" }` 적용
- 동일 System Prompt 반복 호출 시 캐시 히트 → 입력 토큰 비용 절감
- `/api/analyze`와 `/api/rewrite` 각각 별도 System Prompt 캐싱

---

## 데이터 흐름

```
사용자 입력
    │
    ▼
Input Sanitise (XSS 제거, 길이 제한)
    │
    ▼
Rate Limit 체크 (IP당 10req/min)
    │
    ▼
Claude API 호출 (claude-opus-4-7, SSE stream)
    │
    ▼
content_block_delta 이벤트 수신
    │
    ▼
res.write(`data: {...}\n\n`) → 클라이언트로 실시간 전송
    │
    ▼
res.write(`data: [DONE]\n\n`) → 스트림 종료
```

---

## 기술 스택 요약

| 영역 | 기술 | 버전 |
|---|---|---|
| 런타임 | Node.js | 20.x |
| 웹 서버 | Express | 4.x |
| AI 모델 | Claude Opus | claude-opus-4-7 |
| AI SDK | @anthropic-ai/sdk | 0.52.x |
| 환경변수 | dotenv | 16.x |
| 프론트엔드 | Vanilla HTML/CSS/JS | — |
| 상태 저장 | In-memory (Map) | — |
