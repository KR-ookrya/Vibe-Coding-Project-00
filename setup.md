# Setup Guide — AI 자기소개서 첨삭 앱

---

## 1. 필요한 도구 버전

| 도구 | 버전 | 확인 명령 |
|---|---|---|
| Node.js | 20.x 이상 | `node -v` |
| npm | 10.x 이상 | `npm -v` |
| Flutter | 3.x 이상 | `flutter --version` |
| JDK | 17 이상 | `java -version` |
| Git | 최신 권장 | `git --version` |

> Flutter / JDK는 모바일 버전 빌드 시 필요합니다. 웹 서버만 실행할 경우 Node.js만 있으면 됩니다.

---

## 2. 클론 명령

```bash
git clone https://github.com/<your-username>/ai-cover-letter-editor.git
cd ai-cover-letter-editor
```

---

## 3. 의존성 설치

```bash
npm install
```

설치되는 주요 패키지:

| 패키지 | 역할 |
|---|---|
| `express` | 웹 서버 |
| `@anthropic-ai/sdk` | Claude API 연동 |
| `dotenv` | 환경변수 로드 |

---

## 4. 환경변수 설정

프로젝트 루트에 `.env` 파일을 생성하고 아래 내용을 작성하세요.

```bash
cp .env.example .env
```

`.env` 파일 내용:

```env
ANTHROPIC_API_KEY=your_api_key_here
PORT=3000
```

| 변수 | 설명 | 필수 |
|---|---|---|
| `ANTHROPIC_API_KEY` | Anthropic 콘솔에서 발급한 API 키 | ✅ |
| `PORT` | 서버 포트 (기본값: 3000) | 선택 |

> API 키 발급: [https://console.anthropic.com](https://console.anthropic.com) → API Keys 메뉴

---

## 5. 첫 실행

```bash
npm start
```

실행 후 브라우저에서 접속:

```
http://localhost:3000
```

터미널에 아래 메시지가 뜨면 정상 실행된 것입니다:

```
✅ AI 자기소개서 첨삭 앱 실행 중
👉 http://localhost:3000
```

---

## 6. 문제 해결 (FAQ)

### Q1. `Error: ANTHROPIC_API_KEY is missing` 오류가 납니다.

`.env` 파일이 프로젝트 루트에 있는지 확인하세요.  
파일명이 `.env.example` 그대로이거나 `.env.txt`로 저장된 경우 인식되지 않습니다.

```bash
# 파일 존재 여부 확인 (Mac/Linux)
ls -a | grep .env

# Windows
dir /a | findstr .env
```

---

### Q2. `npm install` 후 `node_modules`가 생기지 않습니다.

Node.js 버전을 확인하세요. 20.x 미만이면 업그레이드가 필요합니다.

```bash
node -v          # v20.x.x 이상이어야 함
npm install      # 재시도
```

---

### Q3. 포트 3000이 이미 사용 중이라는 오류가 납니다.

`.env` 파일에서 포트를 변경하세요.

```env
PORT=3001
```

또는 실행 중인 프로세스를 종료하세요.

```bash
# Mac/Linux
lsof -ti:3000 | xargs kill

# Windows PowerShell
Stop-Process -Id (Get-NetTCPConnection -LocalPort 3000).OwningProcess
```

---

### Q4. AI 첨삭 결과가 중간에 끊깁니다.

SSE 스트리밍 응답이 끊기는 경우입니다. 아래를 확인하세요.

- 네트워크 프록시나 VPN이 SSE 연결을 끊고 있을 수 있습니다 → VPN 해제 후 재시도
- `max_tokens` 한도 초과 → 자기소개서 길이를 줄여 재시도
- API 키 크레딧 부족 → [Anthropic 콘솔](https://console.anthropic.com)에서 잔액 확인

---

### Q5. `429 요청이 너무 많습니다` 오류가 납니다.

서버의 Rate Limiter가 적용된 것입니다. **IP당 분당 10회** 요청이 제한됩니다.  
1분 후 다시 시도하거나, `server.js`의 `rateLimit` 함수에서 한도를 조정할 수 있습니다.

```js
// server.js:19 — 한도 수정 예시 (10 → 20)
if (e.count > 20) return res.status(429)...
```
