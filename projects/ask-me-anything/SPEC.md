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
POST /approve/{discord_id}      # 승인 (note 파라미터 포함) → 봇이 DM 발송
POST /deny/{discord_id}         # 거절 → 봇이 DM 발송
GET  /users                     # 승인된 사용자 목록 (username, Discord ID, note, 승인 시간)
GET  /sessions                  # 세션 목록 + 토큰 사용량 + claude_session_id
DELETE /sessions/{session_id}   # 세션 강제 종료
GET  /settings                  # 허용 쉘 도구 목록 설정
POST /settings                  # 허용 도구 목록 저장
```

**DB 추가 컬럼**
- `users.note TEXT` — 승인 시 어드민이 입력하는 메모 (실명, 소속 등)

**환경 변수 (`.env`)**
```
DISCORD_BOT_TOKEN=...
SESSION_IDLE_TIMEOUT_HOURS=1
ADMIN_PORT=8900
PROJECT_DIR=/absolute/path/to/your/project
CLAUDE_MODEL=claude-haiku-4-5-20251001
```
(`ANTHROPIC_API_KEY` 불필요 — Claude Code 기존 인증 사용)

---

## 성공 기준 (Success Criteria)

### 승인 플로우

**SC-1: 신규 사용자 DM — 대기 등록**
- Given: DB에 없는 Discord 사용자가 봇에게 DM을 보낸다
- When: 봇이 메시지를 수신한다
- Then: 사용자에게 "승인 대기 중" 메시지를 응답하고, DB에 `status='pending'` 레코드(discord_id, username, first_message, requested_at)가 생성된다

**SC-2: 대기 중 사용자 추가 DM**
- Given: `status='pending'` 사용자가 DM을 보낸다
- When: 봇이 메시지를 수신한다
- Then: "아직 승인 대기 중"임을 안내하는 메시지를 응답하고, 새 레코드가 생성되지 않는다

**SC-3: 어드민 승인 → 사용자 알림**
- Given: 어드민 패널에서 pending 사용자의 승인 버튼을 클릭한다
- When: 어드민 API가 `POST /approve/{discord_id}`를 처리한다
- Then: DB에서 해당 사용자 `status='approved'`로 변경되고, 봇이 해당 사용자에게 승인 완료 DM을 발송한다

**SC-4: 어드민 거절 → 사용자 알림**
- Given: 어드민 패널에서 pending 사용자의 거절 버튼을 클릭한다
- When: 어드민 API가 `POST /deny/{discord_id}`를 처리한다
- Then: DB에서 해당 사용자 `status='denied'`로 변경되고, 봇이 해당 사용자에게 거절 DM을 발송한다

**SC-5: 거절된 사용자 DM**
- Given: `status='denied'` 사용자가 DM을 보낸다
- When: 봇이 메시지를 수신한다
- Then: "접근이 거절됐어요"에 해당하는 메시지를 응답하고, 세션이 생성되지 않는다

### 세션 동작

**SC-6: 승인된 사용자 DM — Claude 응답**
- Given: `status='approved'` 사용자가 DM을 보낸다
- When: 봇이 메시지를 수신한다
- Then: Claude 세션을 통해 응답이 반환되고, 사용자는 Discord DM으로 답변을 받는다

**SC-7: 사용자별 세션 격리**
- Given: 사용자 A와 사용자 B가 각각 독립된 세션에서 대화 중이다
- When: A의 세션에서 특정 파일/컨텍스트를 참조한다
- Then: B의 세션 응답에는 A의 컨텍스트가 유입되지 않는다 (세션 ID가 분리됨)

**SC-8: 세션 내 컨텍스트 유지**
- Given: 승인된 사용자가 Claude와 여러 차례 대화한다
- When: 이전 대화 내용을 참조하는 질문을 보낸다
- Then: Claude가 동일 세션 내 이전 메시지를 인지한 응답을 반환한다

**SC-9: 비활성 타임아웃 후 세션 재시작**
- Given: 승인된 사용자의 세션이 `SESSION_IDLE_TIMEOUT_HOURS` 동안 비활성 상태다
- When: 해당 사용자가 새 DM을 보낸다
- Then: 새 세션이 생성되고(이전 컨텍스트 초기화), DB에 새 session 레코드가 기록된다

### 읽기 전용 제어

**SC-10: 허용 도구 범위 내 동작**
- Given: Claude 세션이 `allowedTools: ["Read", "Glob", "Grep"] + 어드민 승인 목록`으로 설정되어 있다
- When: 사용자가 소스 코드 분석 질문을 보낸다
- Then: Claude는 허용된 도구만 사용하여 응답하고, 허용 목록 외 도구 호출 시도가 발생하지 않는다

**SC-11: 파일 수정/쓰기 도구 사용 불가**
- Given: Claude 세션이 읽기 전용 도구만 허용하도록 설정되어 있다
- When: 사용자가 파일 수정·생성을 요청한다
- Then: Claude가 해당 동작을 수행하지 않고 "권한 없음"에 준하는 응답을 반환한다

### 어드민 패널

**SC-12: 팬딩 목록 표시**
- Given: pending 사용자가 한 명 이상 존재한다
- When: 어드민이 `/pending` 페이지에 접근한다
- Then: Discord username, Discord ID, 첫 메시지, 요청 시간이 표시되고, 각 행에 승인/거절 버튼이 있다

**SC-13: 세션 목록 및 토큰 사용량 표시**
- Given: approved 사용자의 세션 이력이 존재한다
- When: 어드민이 `/sessions` 페이지에 접근한다
- Then: 각 세션의 사용자, 생성 시간, 마지막 활성 시간, 누적 입출력 토큰 수가 표시된다

**SC-14: 세션 강제 종료**
- Given: 어드민이 `/sessions` 페이지에서 특정 세션의 종료 버튼을 클릭한다
- When: `DELETE /sessions/{session_id}`가 처리된다
- Then: 해당 Claude 세션이 종료되고 DB에서 세션 상태가 종료로 기록된다; 이후 해당 사용자 DM에는 새 세션이 생성된다

**SC-15: 허용 쉘 도구 목록 설정**
- Given: 어드민이 `/settings` 페이지에서 허용 쉘 도구 목록을 수정하고 저장한다
- When: `POST /settings`가 처리된다
- Then: 이후 생성되는 신규 Claude 세션에 수정된 도구 목록이 반영된다

### 보안 및 환경

**SC-16: 어드민 패널 localhost 전용**
- Given: 어드민 서버가 실행 중이다
- When: 외부 IP 또는 0.0.0.0에서 어드민 엔드포인트에 접근을 시도한다
- Then: 연결이 거부되거나 응답이 반환되지 않는다

**SC-17: 시크릿 하드코딩 없음**
- Given: 소스 코드 전체
- When: 코드 검토 시
- Then: DISCORD_BOT_TOKEN, ANTHROPIC_API_KEY 등 모든 자격증명이 .env 파일에서만 로드되고 소스 코드 내 하드코딩이 없다

### 토큰 추적

**SC-18: 메시지별 토큰 사용량 기록**
- Given: 승인된 사용자의 DM에 Claude가 응답한다
- When: 응답이 완료된다
- Then: 해당 교환의 input_tokens, output_tokens가 `token_usage` 테이블에 session_id와 함께 기록된다

---

## 미결 이슈 (Open Issues)

- [x] [Advisory] 허용 쉘 도구 목록의 초기 기본값 — **빈 목록으로 결정** (보안 우선, 어드민이 명시적으로 추가해야 활성화) — v0.1, v0.1.1에서 해소

---

## 변경 이력 (Changelog)

- v0.1: 초기 스펙 — Discord DM + Claude Code 세션 연동, 로컬 어드민 패널
- v0.1.1: Success Criteria SC-1~SC-18 추가 (Hawkeye); 쉘 도구 기본값 빈 목록으로 결정
- v0.1.2: 구현 중 반영 사항 — PROJECT_DIR/CLAUDE_MODEL 환경변수 추가; 승인 시 note 필드; /users 페이지; 세션 목록에 claude_session_id 노출; ANTHROPIC_API_KEY 불필요 확인
