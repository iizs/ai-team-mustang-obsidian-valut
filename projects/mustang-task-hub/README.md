# mustang-task-hub

_2026-09-14 착수 · PoC 단계_

Task Hub receiver + agent glue. Cloudflare Tunnel 뒤에서 외부 webhook(Dooray · 추후 GitHub 등)을 받아 담당 agent에게 배달. 정본 아키텍처 방향: [[projects/team-operations-rework/README]] Track A3/A4.

소스: [github.com/iizs/mustang-task-hub](https://github.com/iizs/mustang-task-hub) (private)

## 현재 스코프 — PoC

- FastAPI catch-all endpoint. 모든 요청의 headers + body를 파일로 dump.
- docker-compose 로 로컬 8080 노출.
- 라우팅 · 큐 · agent 매핑 · dedup 없음.

## 검증 결과 (2026-09-14)

- Cloudflare Quick Tunnel(`https://logan-administration-flying-domains.trycloudflare.com`) 뒤에서 Dooray Sandbox의 4개 이벤트 (postCreated · postCommentCreated · postWorkflowChanged × 2) 모두 정상 수신.
- 페이로드 예시 저장 위치: `~/Projects/mustang-task-hub/logs/2026-09-14T13-58-*.json`

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
| `DOORAY_API_KEY` | ✓ | 멤버 캐시 fetch용 (프로젝트 admin 계정 토큰). |
| `DOORAY_BASE_URL` | | default `https://api.dooray.com`. 클라우드별 override. |
| `TASK_HUB_WEBHOOK_TOKEN` | ✓ | URL secret path token. |
| `TASK_HUB_PROJECT_IDS` | ✓ | 멤버 캐시 대상 프로젝트 id, 쉼표 구분. |
| `TASK_HUB_MAPPING_FILE` | | override yaml 경로. default `/data/mapping.yaml`. |
| `TASK_HUB_LOG_DIR` | | payload archive dir. default `/data/logs`. |
| `TASK_HUB_LOG_KEEP_DAYS` | | rotation retention (default 30). |
| `TASK_HUB_PUBLIC_URL` | | informational (Cloudflare tunnel URL), `/health`에 echo. |
| `TASK_HUB_LOG_LEVEL` | | default `info`. |

**환경변수 (agent session)**:

| 변수 | 설명 |
|---|---|
| `TASK_HUB_WS_URL` | 세션이 Monitor로 붙을 WebSocket URL. 예: `wss://<tunnel>/events/roy`. Launcher가 주입. |

Cloudflare Tunnel은 receiver 컨테이너 밖에서 별도로 실행 (docker compose에 포함 안 함) — 재시작 시 URL이 바뀌는 Quick Tunnel 특성상 tunnel URL 변경 시 agent 세션 env 갱신하고 재시작 필요.

**v0.3 (예정)** 관측성 · dedup
- 이벤트 id 기반 dedup 캐시 (짧은 TTL).
- 감사 로그 DB or 파일.

**v0.4 (예정)** Named tunnel 도입 (Quick Tunnel URL 휘발 문제 해소).

## 결정 기록

- **2026-09-14** 프로젝트 착수. 이름 `mustang-task-hub` · 리포 `iizs/mustang-task-hub` (private) · 언어 Python + FastAPI · Docker 실행.
- **2026-09-14** PoC receiver 검증 완료. Cloudflare Quick Tunnel + Dooray Sandbox webhook 4/4 도달.
- **2026-09-14** 관찰: `requestOrigin.type=open-api` 를 self-loop 방지 필터로 사용 가능. `hookVersion` vs `version` 문서 discrepancy 문서화.
- **2026-09-14** v0.2 스펙 확정 (매핑 규약, WS 채널, 인증, self-loop, archival, health, 로깅, fan-out 등 8개 결정).
- **2026-09-15** v0.2 구현 완료 · 실증. 모듈 재편 (`config`, `dooray_client`, `member_cache`, `router`, `sources/dooray`, `ws_manager`, `archiver`, `log_setup`, `main`). 멤버 캐시는 프로젝트 members(id only) + `/common/v1/members/{id}` 개별 조회 조합으로 userCode 획득. Sandbox에서 self-loop skip · non-Roy source → Roy WS 배달 · 잘못된 token 404 모두 검증 통과.

## 미결

- 도메인 기반 named tunnel 도입 시점 (PoC 넘어 안정 운영 시).
- Receiver dedup 정책 (event_id 저장 방식 · TTL) — v0.3에서.
- Roy 봇이 Sandbox admin이어야 hook 등록 가능 — 봇 유저 권한 표준화 (다른 프로젝트도 admin 승격 필요).
- Sandbox의 옛 hook (PoC 단계 등록, `/dooray-webhook` 지향, id `4421553773067818743`) 는 새 URL과 무관해 여전히 활성 → 404 응답. Dooray 웹UI에서 삭제해야 완전 정리 (API에 hook delete 없음). 방치해도 receiver는 정확히 404로 응답하니 blocker 아님.
- Agent session 쪽 launcher가 `TASK_HUB_WS_URL` 을 자동으로 세팅하는 매커니즘 — 지금은 수동 입력.
