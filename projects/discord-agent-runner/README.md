# discord-agent-runner

_2026-09-22 착수 · v0.1 초기 구현_

Discord 메시지를 받아 Claude Code CLI 를 headless(`--print --output-format json`) 로 호출하고 응답을 다시 Discord 로 보내는 per-agent 브릿지. `plugin:discord@inline` 을 대체하기 위한 것 (완전 대체는 아니고, "이 에이전트는 Discord 로만 접근한다" 라는 선택지를 여는 컴포넌트).

소스: `/Users/kirinchoi/Projects/ai-teams/discord-agent-runner/` (ai-teams 리포 안 · 별도 repo 없음). 관련 큰그림: [[projects/team-operations-rework/README]] Track A (agent runtime).

## 배경

- 기존 `plugin:discord@inline` 은 interactive Claude Code 세션 안에서 실행돼, Kirin 이 세션을 살려둔 상태로 Discord 메시지를 컨텍스트에 삽입 받는 방식. 세션 상시 유지 필요.
- 일부 에이전트는 Discord 로만 접근하면 되고 (다른 UI 필요 없음) · 세션 상시 유지도 낭비. Headless CLI 로 요청마다 새 프로세스 spawn 하는 편이 자연스러움.
- 결정: Discord 만 쓸 에이전트용으로 별도 브릿지 프로세스. Interactive 세션은 두는 에이전트는 그대로 유지.

## 아키텍처

**프로세스 모델**: **에이전트 1개당 브릿지 프로세스 1개** (`python -m discord_agent_runner <agent-name>`). 각 프로세스가 자기 Discord bot token 으로 로그인 · 자기 대화만 listen.

**메시지 흐름**:

```
Discord message (DM 또는 mention)
  → AgentBot.on_message (discord.py)
  → per-channel FIFO Lock (동시성: 큐 대기)
  → typing indicator ON
  → ClaudeRunner.run(channel_id, prompt)
      → asyncio subprocess: claude -p "<prompt>" --output-format json [-r <session-id>]
      → cwd = agent-<name>/  (CLAUDE.md 자동 로드)
      → JSON stdout parse → text + session_id
  → typing indicator OFF
  → 응답 2000자 초과면 줄바꿈 우선 chunk 분할 → 순서대로 채널로 send
```

**세션 정책**:

- 채널(또는 DM 채널) 별로 `(session_id, last_msg_ts)` in-memory 저장.
- 유휴 시간 (`DAR_SESSION_IDLE_MINUTES`, default 30분) 이내면 `-r <session-id>` 로 재개. 초과면 새 세션 시작.
- `!reset` 채널 메시지로 즉시 세션 상태 삭제 → 다음 메시지는 새 세션.
- 프로세스 재시작 시 상태 리셋 (자연스러운 초기화).

**동시성**: 채널당 `asyncio.Lock` FIFO — 한 번에 하나의 요청만 Claude 로 넘어감. 사용자가 처리 중에 후속 메시지 보내면 typing 표시 유지되며 순차 처리.

**반응 범위**: DM 이거나 봇이 mention 된 메시지만 처리 (스팸 방지). `DAR_ALLOWED_CHANNEL_IDS` 로 특정 채널만 허용 가능.

## 파일 구성

```
discord-agent-runner/
├── .venv/                       # (gitignored)
├── discord_agent_runner/
│   ├── __init__.py
│   ├── __main__.py              # entry: python -m discord_agent_runner <agent>
│   ├── config.py                # .env → Config dataclass
│   ├── logging_setup.py         # file + stdout logger
│   ├── claude_runner.py         # subprocess wrapper + session mgmt
│   └── bot.py                   # discord.py Client + on_message
├── logs/                        # <agent>.log (rotated 안 함, 필요시 확장)
├── run/                         # <agent>.pid, <agent>.out
├── requirements.txt             # discord.py, python-dotenv
├── .gitignore
└── README.md                    # 개발자용 (본 문서는 vault 정본)
```

스크립트:

```
ai-teams/scripts/
├── discord-agent-runner-start.sh   <agent>   # 백그라운드 기동 (PID 파일)
└── discord-agent-runner-stop.sh    <agent>   # SIGTERM + 10s 대기 후 SIGKILL fallback
```

## 환경변수 (`agent-<name>/.env`)

| 변수 | 필수 | 기본값 | 설명 |
|---|---|---|---|
| `DISCORD_BOT_TOKEN` | ✓ | — | 그 에이전트가 로그인할 Discord bot 토큰 |
| `DAR_LOG_LEVEL` | | `INFO` | `DEBUG` / `INFO` / `WARNING` / `ERROR` |
| `DAR_SESSION_IDLE_MINUTES` | | `30` | 이 시간 이상 유휴면 다음 메시지는 새 세션 |
| `DAR_CLAUDE_BIN` | | `claude` | 커스텀 경로 필요 시 override |
| `DAR_ALLOWED_CHANNEL_IDS` | | (없음) | CSV. 지정 시 그 채널만 응답 (DM 은 항상 응답) |

## 결정 사항 확정 (2026-09-22)

- **이름**: `discord-agent-runner` (Kirin 결정 · 목적 서술)
- **위치**: ai-teams 리포 top-level (별도 repo 없음)
- **venv**: 프로젝트 안 `.venv` (dot prefix)
- **테스트 에이전트**: agent-envy
- **동시성**: per-channel FIFO 큐 대기
- **응답 포맷**: 2000자 초과 시 줄바꿈 우선 chunk 분할, 순차 전송
- **로그**: `discord-agent-runner/logs/<agent>.log`
- **부팅 자동화**: 이 iteration scope 밖 (별도 논의)

## 미결 · 확인 대기

- **부팅 자동화 (launchd LaunchAgent)** — 별도 iteration.
- **로그 rotation** — 지금 append-only. 규모 커지면 logrotate 나 handler 교체.
- **응답이 매우 오래 걸리는 경우** (Claude tool use 여러 turn) — Discord typing indicator 는 10초마다 재전송이 필요할 수 있음. 지금 코드는 `async with channel.typing()` 컨텍스트 매니저에 위임. 필요하면 자체 heartbeat 로 대체.
- **에러 응답 UX** — 현재는 `⚠️ ...` prefix 로 사용자에게 노출. 로그로만 남기고 사용자엔 간결한 메시지로 갈지 검토.
- **`plugin:discord@inline` 완전 제거 시점** — 이 브릿지가 안정화되면 (2~3 iteration 관찰 후) `scripts/start.sh` 에서 관련 라인 제거 예정.

## 사용 예

```
# 처음: venv 준비
cd ~/Projects/ai-teams/discord-agent-runner
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 기동 (백그라운드)
~/Projects/ai-teams/scripts/discord-agent-runner-start.sh envy

# 로그 관찰
tail -f ~/Projects/ai-teams/discord-agent-runner/logs/envy.log

# 종료
~/Projects/ai-teams/scripts/discord-agent-runner-stop.sh envy
```
