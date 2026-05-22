# openclaw — 라우팅 LLM 게이트웨이 사양

OpenClaw 클라이언트가 호출하는 LLM 라우터를 구축한다. 간단한 질의는 로컬 ollama로,
복잡한 질의는 Claude Code CLI로 보내는 구조다.

## 1. 서버

- FastAPI, `uvicorn`으로 기동.
- 바인딩: `http://127.0.0.1:18080`.
- 엔드포인트:
  - `POST /v1/chat/completions` — OpenAI 호환. **스트리밍(SSE) 필수**.
    `stream: true`일 때 OpenAI 표준 SSE 청크
    (`role` → `content` → `finish_reason` → `[DONE]`) 반환.
    `stream` 미지정/false면 일반 JSON.
  - `GET  /v1/models`
  - `GET  /healthz`
- 입력 스키마는 관대하게 — `messages[].content`는 문자열 또는 list 모두 허용,
  `tools`/`temperature` 등 OpenAI 추가 필드는 `extra: "allow"`로 받아준다.
- 요청 본문/헤더와 분류 결과를 로그로 남긴다 (`/tmp/openclaw.log`로 리다이렉트).

## 2. 라우팅 (LangGraph)

```
START → classify → (simple → ollama) | (complex → claude) → END
```

`classify_node`는 ollama로 SIMPLE/COMPLEX 한 토큰만 받게 한다. 분류 프롬프트는
**아래 케이스는 무조건 COMPLEX로 강제**:

- 실시간/최신/오늘/현재 정보
- 뉴스, 헤드라인, 기사
- 주식·환율·날씨·스포츠 등 라이브 데이터
- 특정 사이트/URL 요청 (investing.com 등)
- 웹 검색·브라우징이 필요한 모든 것

분류 결과는 로그로 찍는다: `classify verdict=... -> route=simple|complex`.

## 3. 백엔드

### Ollama (simple)
- 엔드포인트: `http://localhost:11434/api/chat`
- 모델: `gemma4:e2b-nvfp4`

### Claude (complex)
- **Anthropic API 직접 호출하지 않는다**. `claude` CLI를 서브프로세스로 호출.
  ```
  claude -p --model sonnet "<prompt>"
  ```
- 인증은 사용자의 Claude **구독(OAuth)** 사용.
  `run.sh`에서 `ANTHROPIC_API_KEY`, `CLAUDE_API_KEY` 환경변수를 **unset** 한 뒤 기동.
  (env 키가 있으면 CLI가 API 청구로 fallback하면서 잔액 부족 오류 발생.)
- 모델: `sonnet` (Claude Sonnet 4.6).
- 환경변수: `CLAUDE_BIN`, `CLAUDE_MODEL`, `CLAUDE_TIMEOUT` (기본 300s).
- 웹 검색/브라우징은 Claude Code CLI가 내장 도구로 알아서 처리.

## 4. 응답 포맷

OpenAI strict 클라이언트 (OpenAI/JS SDK 등) 호환을 위해:

- 응답 본문에 `usage`(prompt/completion/total tokens, 추정값) 포함.
- `choices[0]`에 `logprobs: null`, `finish_reason: "stop"` 포함.
- `system_fingerprint` 포함.
- 라우팅 메타데이터(`x-openclaw-route`, `x-openclaw-backend`)는 **응답 헤더**로만 보낸다
  (본문에 비표준 필드 넣지 않는다).
- `model` 필드에는 요청에서 받은 모델명을 그대로 echo (없으면 `openclaw-auto`).

## 5. 환경 및 실행

- Python: `pyenv` 사용 (`.python-version`).
- 의존성: `fastapi`, `uvicorn`, `httpx`, `langgraph`, `langchain-core`, `pydantic`.
  (Anthropic SDK는 사용하지 않음 — CLI 방식이므로.)
- 실행:
  ```bash
  ./run.sh   # ANTHROPIC_API_KEY 자동 unset 후 uvicorn 기동
  ```
- 로그: `/tmp/openclaw.log` (요청 본문, 분류 verdict, 백엔드 응답 길이 등).

## 6. 파일 구조

```
openclaw_lang_graph/
├── openclaw/
│   ├── __init__.py
│   ├── server.py    # FastAPI, SSE 스트리밍, 요청 로깅
│   └── graph.py     # LangGraph: classify → ollama|claude
├── run.sh           # env 정리 후 uvicorn 실행
├── .python-version
└── .venv/
```
