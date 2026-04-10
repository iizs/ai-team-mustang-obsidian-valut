# Ask Me Anything — SPEC

## 요건 (Requirements)

**서비스 개요**
Discord DM으로 봇에게 말을 걸면 Claude Code 세션이 응답해주는 서비스. 각 사용자는 독립된 세션을 가지며, 소스 코드 분석에 특화된 읽기 전용 Claude 에이전트와 대화한다.

**메신저**
- Discord DM (1:1). 그룹 채널 미지원. 향후 확장 가능성 열어둠.

**사용자 세션**
- 사용자별 독립 Claude Code 세션
- 세션은 Claude 프로세스가 살아있는 한 지속 (컨텍스트 유지)
- 세션 비활성 타임아웃: 설정 가능

**허용 동작 (읽기 전용)**
- 파일 시스템 소스 코드 분석: `Read`, `Glob`, `Grep`
- 어드민이 사전 승인한 쉘 도구 목록 (예: `find` 등 읽기 전용 명령어)
- 소스 코드 실행, 파일 수정, 쓰기 동작 불허

**승인 플로우**
1. 새 사용자가 봇에게 DM → 봇이 "대기 중" 응답, DB에 팬딩 레코드 생성
2. 어드민 웹 패널에서 팬딩 목록 확인 (Discord username, Discord ID, 첫 메시지, 요청 시간)
3. 어드민이 승인/거절 → 봇이 사용자에게 DM으로 결과 통보
4. 팬딩 상태에서 추가 DM 시: "아직 대기 중" 응답
5. 거절된 사용자 DM 시: "접근이 거절됐어요" 응답

**어드민 웹 페이지**
- 로컬 접속 전용 (localhost only), 인증 불필요
- 팬딩 승인 목록 및 승인/거절 동작
- 세션 목록 및 세션별 토큰 사용량
- 세션 강제 종료 기능
- 허용 쉘 도구 목록 설정

**데이터 저장**
- SQLite (또는 파일 기반)으로 충분

---

## 기술 설계 (Technical Design)

**디렉토리 구조**
```
ask-me-anything/
├── bot/
│   ├── main.py               # Discord 봇 엔트리포인트
│   ├── session_manager.py    # Claude Code 세션 생성·관리
│   └── db.py                 # SQLite 접근 레이어
├── admin/
│   ├── server.py             # FastAPI 어드민 서버
│   └── templates/            # HTML 템플릿
├── claude_sessions/          # 사용자별 작업 디렉토리 (gitignored)
├── requirements.txt
├── .env                      # 봇 토큰, API 키 (gitignored)
└── ask-me-anything.db        # SQLite DB (gitignored)
```

**기술 스택**
- Discord 봇: Python, discord.py
- Claude 세션: Claude Code Agent SDK (`claude-agent-sdk`)
- 어드민 서버: FastAPI (localhost only)
- 어드민 UI: Jinja2 HTML 템플릿
- DB: SQLite (SQLAlchemy)
- 비동기: asyncio (봇 + 세션 병렬 처리)

**DB 스키마**
```sql
-- 사용자 접근 관리
users:
  discord_id   TEXT PRIMARY KEY
  username     TEXT NOT NULL
  status       TEXT NOT NULL  -- 'pending' | 'approved' | 'denied'
  first_message TEXT
  requested_at DATETIME
  resolved_at  DATETIME

-- Claude 세션 추적
sessions:
  id                INTEGER PRIMARY KEY
  discord_id        TEXT REFERENCES users
  claude_session_id TEXT UNIQUE        -- Agent SDK에서 반환하는 세션 ID
  created_at        DATETIME
  last_active       DATETIME

-- 토큰 사용량 (메시지 단위 누적)
token_usage:
  id          INTEGER PRIMARY KEY
  session_id  INTEGER REFERENCES sessions
  input_tokens  INTEGER
  output_tokens INTEGER
  recorded_at DATETIME
```

**Claude 세션 관리**
- 사용자 DM 수신 → `discord_id`로 DB 조회
  - `approved`: 기존 세션 재개 (`claude_session_id` 있으면 resume, 없으면 신규 시작)
  - `pending`: "대기 중" 응답
  - `denied`: "거절됨" 응답
  - 신규: `pending` 레코드 생성, "대기 중" 응답
- Claude 세션: `allowedTools: ["Read", "Glob", "Grep"] + 어드민 승인 목록`
- 각 응답의 `usage` 필드를 파싱해 `token_usage`에 저장
- 세션 비활성 타임아웃: 환경변수로 설정 (기본 1시간)

**어드민 API**
```
GET  /                          # 대시보드
GET  /pending                   # 팬딩 목록
POST /approve/{discord_id}      # 승인 → 봇이 DM 발송
POST /deny/{discord_id}         # 거절 → 봇이 DM 발송
GET  /sessions                  # 세션 목록 + 토큰 사용량
DELETE /sessions/{session_id}   # 세션 강제 종료
GET  /settings                  # 허용 쉘 도구 목록 설정
POST /settings                  # 허용 도구 목록 저장
```

**환경 변수 (`.env`)**
```
DISCORD_BOT_TOKEN=...
ANTHROPIC_API_KEY=...
SESSION_IDLE_TIMEOUT_HOURS=1
ADMIN_PORT=8900
```

---

## 성공 기준 (Success Criteria)

*Hawkeye가 작성 예정*

---

## 미결 이슈 (Open Issues)

- [ ] [Advisory] 허용 쉘 도구 목록의 초기 기본값 — 어드민 설정 전 기본값을 빈 목록으로 할지, 안전한 읽기 전용 셋으로 할지 결정 필요 — v0.1

---

## 변경 이력 (Changelog)

- v0.1: 초기 스펙 — Discord DM + Claude Code 세션 연동, 로컬 어드민 패널
