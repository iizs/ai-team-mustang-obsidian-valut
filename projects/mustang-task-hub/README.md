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

**v0.2 (예정)** 라우팅
- assignee organizationMemberId → agent name 매핑.
- 이벤트 타입별 처리 정책.
- `requestOrigin.type=open-api` + source가 자기 자신 = self-loop skip.

**v0.3 (예정)** 배달 채널
- Monitor용 WebSocket endpoint per agent (`/events/<agent>`).
- 또는 FIFO 대체 매커니즘.

**v0.4 (예정)** 관측성
- dedup 캐시 (event_id 기반, 짧은 TTL).
- 감사 로그 (수신 · 배달 이력).

## 결정 기록

- **2026-09-14** 프로젝트 착수. 이름 `mustang-task-hub` · 리포 `iizs/mustang-task-hub` (private) · 언어 Python + FastAPI · Docker 실행.
- **2026-09-14** PoC receiver 검증 완료. Cloudflare Quick Tunnel + Dooray Sandbox webhook 4/4 도달.
- **2026-09-14** 관찰: `requestOrigin.type=open-api` 를 self-loop 방지 필터로 사용 가능. `hookVersion` vs `version` 문서 discrepancy 문서화.

## 미결

- 도메인 기반 named tunnel 도입 시점 (PoC 넘어 안정 운영 시).
- Receiver dedup 정책 (event_id 저장 방식 · TTL).
- Agent name 매핑 소스 (config yaml vs Dooray tag vs 별도 매핑 DB).
- Roy 봇이 Sandbox admin이어야 hook 등록 가능 — 봇 유저 권한 표준화 (다른 프로젝트도 admin 승격 필요).
