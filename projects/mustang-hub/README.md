# mustang-hub

_2026-09-14 착수 · v0.2 실증 완료 · 2026-09-15 `mustang-task-hub` → `mustang-hub` 리네임 (플러그인·skill 이름 정합화) · 2026-09-20 폴 드레인 추가 (에이전트-side 드레인 제거) · 2026-09-20 Cloudflare Quick Tunnel → Tailscale Funnel 이관 (안정 URL 확보)._

Task Hub receiver. Cloudflare Tunnel 뒤에서 외부 webhook(Dooray · 추후 GitHub 등)을 받아 담당 agent WebSocket 채널로 fan-out. 사람 계정은 Dooray 자체 알림 위임(skip). 정본 아키텍처: [[projects/team-operations-rework/README]] Track A3. Agent 쪽: [[projects/mustang-hub-agent/README]].

소스: [github.com/iizs/mustang-hub](https://github.com/iizs/mustang-hub) (private. 구 URL `mustang-task-hub` 는 GitHub redirect 유지.)

## 현재 스코프 — PoC

- FastAPI catch-all endpoint. 모든 요청의 headers + body를 파일로 dump.
- docker-compose 로 로컬 8080 노출.
- 라우팅 · 큐 · agent 매핑 · dedup 없음.

## 검증 결과 (2026-09-14)

- Cloudflare Quick Tunnel(`https://logan-administration-flying-domains.trycloudflare.com`) 뒤에서 Dooray Sandbox의 4개 이벤트 (postCreated · postCommentCreated · postWorkflowChanged × 2) 모두 정상 수신.
- 페이로드 예시 저장 위치: `~/Projects/mustang-task-hub/logs/2026-09-14T13-58-*.json`

## Public URL — Tailscale Funnel (2026-09-20 이관)

- 현재 `https://kirins-mac-mini.tail465f29.ts.net/` (Mac mini 재부팅에도 유지).
- `tailscale funnel --bg 8080` 으로 port 443 공개 → localhost:8080 프록시. Tailscale 관리 콘솔에서 (1) DNS "HTTPS Certificates" 활성 (2) `nodeAttrs: [{target:["*"], attr:["funnel"]}]` 사전 세팅 필요.
- 이관 배경: Quick Tunnel 은 실행마다 URL 이 바뀌어 `.env` 갱신·훅 재등록 필요. Named domain 대안 검토 후 개인 도메인 없는 조건에서 Tailscale Funnel 채택 (개인 tailnet 무료 · HTTPS 자동 · 관리 편함).

## Dooray webhook payload 관찰 사항

문서와 실제 응답 차이:

- 문서상 `hookVersion` → 실제 `version` 필드로 옴 (`version: 2`).
- 실제 payload는 `webhookType` + `hookEventType` 두 필드가 함께 옴 (같은 값).
- `titleLink` — 웹UI 링크 (`https://mustang.dooray.com/project/tasks/{id}`).
- `requestOrigin.type` — `"open-api"` 값이 이벤트가 API 호출로 유발됐음을 표시. **이벤트 루프 방지 필터로 활용 가능** (agent가 API로 취한 액션이 다시 webhook으로 돌아올 때 스킵).
- `source.member` — 이벤트를 유발한 사용자 정보 (id, name, userCode, email).
- `post.users.to[].workflowId` — 담당자별 workflow id (개별 상태 트랙).

## 진화 로드맵

**v0.1 (완료)** PoC — payload dump.

**v0.2 (착수 · 2026-09-14)** Fully functional receiver

**결정 매트릭스**:

- **매핑**: `userCode == agent name` 규약. 새 멤버 추가 시 receiver 재시작. `mapping.yaml` 로 특정 `organizationMemberId → agent name` 오버라이드 지원.
- **채널**: Monitor용 **WebSocket per agent** (`wss://<tunnel>/events/<agent>`). Fan-out — 같은 agent 이름으로 여러 세션 연결 시 모두에게 배달 (Dooray truth 원칙이 안전 보장).
- **엔드포인트**: `/webhook/<source>/<token>` 소스 무관 구조. Source별 payload parser 모듈.
- **인증**: URL secret path token (env로 주입). 유출 시 rotation = env token 갱신 + receiver 재시작 + Dooray 웹UI에서 옛 hook 삭제 + `dooray hook-create` 재등록.
- **Self-loop 방지**: `requestOrigin.type == "open-api"` **AND** `source.member.userCode` == 라우팅 대상 agent → 배달 skip.
- **Payload archival**: 매 요청 파일 dump 유지. rotation (일자별 dir, N일 보관).
- **Health/status**: `/health` 에 ws 연결 상태, 이벤트 카운트, 버퍼 크기 등 노출.
- **로깅**: json line stdout (received / routed / delivered / dropped / error).
- **In-memory 버퍼 (skip)**: receiver-agent가 localhost라 실제 disconnection 창이 좁아 marginal — Dooray truth의 session drain 이 그 gap을 커버.

**환경변수 (receiver 컨테이너)**:

| 변수 | 필수 | 설명 |
|---|---|---|
| `DOORAY_API_KEY` | ✓ | 멤버 캐시 fetch용 (프로젝트 admin 계정 토큰). 폴 드레인의 posts 조회도 이 토큰. |
| `DOORAY_BASE_URL` | | default `https://api.dooray.com`. 클라우드별 override. |
| `TASK_HUB_WEBHOOK_TOKEN` | ✓ | URL secret path token. |
| `TASK_HUB_PROJECT_IDS` | ✓ | 멤버 캐시 · 폴 드레인 대상 프로젝트 id, 쉼표 구분 (public 프로젝트 위주). |
| `TASK_HUB_MAPPING_FILE` | | override yaml 경로. default `/data/mapping.yaml`. |
| `TASK_HUB_LOG_DIR` | | payload archive dir. default `/data/logs`. |
| `TASK_HUB_LOG_KEEP_DAYS` | | rotation retention (default 30). |
| `TASK_HUB_POLL_INTERVAL_SEC` | | 폴 드레인 주기 (default 600 = 10분). `0` 이면 폴 비활성. |
| `TASK_HUB_PUBLIC_URL` | | informational (Cloudflare tunnel URL), `/health`에 echo. |
| `TASK_HUB_LOG_LEVEL` | | default `info`. |

**환경변수 (agent session)**:

| 변수 | 설명 |
|---|---|
| `TASK_HUB_WS_URL` | 세션이 Monitor로 붙을 WebSocket URL. 예: `wss://<tunnel>/events/roy`. Launcher가 주입. |

Tunnel(현재 Tailscale Funnel)은 receiver 컨테이너 밖 host 에서 별도로 실행 (docker compose 에 포함 안 함).

**v0.2.1 (2026-09-20)** 폴 드레인 추가.

- 배경: agent-side drain(세션 시작 시 · 60분 안전망)은 세션 형태(/loop vs --continue)에 종속 — `ScheduleWakeup` 은 `/loop dynamic` 모드 전용이라 우리 `--continue` 세션에선 안전망이 안 걸림. `--continue` 를 유지하면서 드레인을 살리려면 서버 쪽으로 옮기는 게 맞음.
- 설계: `Poller` (app/poller.py) 가 백그라운드 asyncio task 로 `TASK_HUB_POLL_INTERVAL_SEC` 마다 tick.
  - WS 연결된 각 agent (`human_agents` 제외) 별로 `member_id_for_agent` 역조회.
  - `TASK_HUB_PROJECT_IDS` 각 프로젝트에 `list_pending_posts(toMemberIds=<mid>, postWorkflowClasses=registered,working)` 호출.
  - 각 pending post 를 `hook_event=pollDrain`, `request_origin=poll` envelope 으로 WS push.
- **간단 dedup**: `(agent, post_id) → last postUpdatedAt` 을 in-memory 로 추적. 값 변화 없으면 같은 tick 에서 skip → 매 폴마다 skill 재호출 폭풍 방지. 재시작 시 상태 리셋 (Dooray truth idempotency 로 회복 가능).
- 완료·재할당으로 pending 에서 빠지면 상태 prune → 재등장 시 즉시 push.
- 사람 계정 (`human_agents`) 은 폴 대상에서 자동 제외.
- 개인(private) 프로젝트는 스코프 밖 — 현 단계 public 프로젝트 (`TASK_HUB_PROJECT_IDS` 명시분) 만 훑음.
- `poll_delivered` 카운터, `/health` 응답에 `poll_interval_sec` 노출.

**v0.3 (예정)** 관측성 · dedup 고도화
- Webhook 이벤트 id 기반 dedup 캐시 (짧은 TTL).
- 감사 로그 DB or 파일.

**v0.4 (완료 · 2026-09-20)** 안정 URL 확보 — Cloudflare Named Tunnel 대신 **Tailscale Funnel** 채택. 개인 도메인 취득/등록 회피, 개인 tailnet 무료. (Cloudflare Tunnel 은 zone 소유 요구 · 기존 GoDaddy 도메인은 이관 부담. Tailscale 은 이미 사용 중이라 마찰 최소.)

## 결정 기록

- **2026-09-14** 프로젝트 착수. 이름 `mustang-task-hub` · 리포 `iizs/mustang-task-hub` (private) · 언어 Python + FastAPI · Docker 실행.
- **2026-09-14** PoC receiver 검증 완료. Cloudflare Quick Tunnel + Dooray Sandbox webhook 4/4 도달.
- **2026-09-14** 관찰: `requestOrigin.type=open-api` 를 self-loop 방지 필터로 사용 가능. `hookVersion` vs `version` 문서 discrepancy 문서화.
- **2026-09-14** v0.2 스펙 확정 (매핑 규약, WS 채널, 인증, self-loop, archival, health, 로깅, fan-out 등 8개 결정).
- **2026-09-15** v0.2 구현 완료 · 실증. 모듈 재편 (`config`, `dooray_client`, `member_cache`, `router`, `sources/dooray`, `ws_manager`, `archiver`, `log_setup`, `main`). 멤버 캐시는 프로젝트 members(id only) + `/common/v1/members/{id}` 개별 조회 조합으로 userCode 획득. Sandbox에서 self-loop skip · non-Roy source → Roy WS 배달 · 잘못된 token 404 모두 검증 통과.
- **2026-09-16** postCommentCreated / postWorkflowChanged payload 에 assignee 정보 부재 발견 → 파서에 top-level `users.to` fallback 추가 + `post_assignees` API fallback (Dooray truth 원칙).
- **2026-09-20** 폴 드레인 추가 (v0.2.1). 에이전트-side drain 을 receiver 로 이관. 드레인 전략을 서버에 두어 세션 형태(/loop vs --continue)에 종속되지 않게 함.
- **2026-09-20** Cloudflare Quick Tunnel → Tailscale Funnel (v0.4). Public URL `https://kirins-mac-mini.tail465f29.ts.net/`. Dooray Sandbox hook 재등록 (id `4426061117507948068`, 옛 hook 은 웹UI 삭제 대기).

## 미결

- Receiver dedup 정책 (event_id 저장 방식 · TTL) — v0.3에서.
- Roy 봇이 Sandbox admin이어야 hook 등록 가능 — 봇 유저 권한 표준화 (다른 프로젝트도 admin 승격 필요).
- Sandbox 의 옛 hook 들 (PoC `4421553773067818743` + Quick Tunnel `logan-administration-flying-domains.trycloudflare.com` 지향 훅) 은 API delete 미지원 → Dooray 웹UI에서 수동 삭제 필요. 방치해도 receiver 는 정확히 404 또는 미도달로 응답하니 blocker 아님.
- Agent session 쪽 launcher가 `TASK_HUB_WS_URL` 을 자동으로 세팅하는 매커니즘 — 지금은 수동 입력.
- 개인(private) 프로젝트 폴 대상 포함 여부 — 현 단계 스코프 밖. 필요해지면 poller 가 별도로 `--type private` 프로젝트 리스트도 훑고 각 agent 별 개인 프로젝트를 커버하는 방식으로 확장 가능.
- 폴 주기(600s) 적정성 — 관찰 후 fine-tune.
- Tailscale Funnel 은 host 부팅 시 자동 재기동 필요 — 지금은 수동 `tailscale funnel --bg 8080`. Mac 재부팅 대비 launchd plist 로 자동화 검토.
- `/events/<agent>` WS 는 인증 없음 — 이전에도 Quick Tunnel 동일 조건이었지만 안정 URL 이 되어 우연 접근 가능성 소폭 증가. 단기 blocker 아님, 별도 hardening iteration 필요 시 검토.
