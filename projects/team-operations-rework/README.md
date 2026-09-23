# 팀 운영 재정비 (team-operations-rework)

_2026-09-03 착수 · 진행 중_

Mustang 팀의 협업 방식과 인프라를 재설계하는 initiative.

## 배경 / 문제

### 지난 몇 개월의 관찰

Mustang 팀과 그 밖 여러 시도에서 AI 에이전트를 운영해 온 경험을 요약하면:

**좋았던 점**

- **역할을 나눠 멀티 에이전트로 구성한 것**은 긍정적 효과가 컸다. Roy(설계) · Breda(개발) · Hawkeye(평가)로 나눈 지금의 구성이 최적인지는 미지수지만, 역할 분리라는 방향 자체는 유지할 가치가 있다.

**의심스러웠던 점**

- **모든 에이전트가 모든 대화를 관찰해야 했다.** Discord 그룹 채널은 언급 여부와 관계없이 모든 메시지를 전 에이전트에게 흘려보냈고, 각 에이전트는 자신과 무관한 내용까지 컨텍스트로 흡수해야 했다. 컨텍스트 · 토큰 낭비. Need-to-know 원칙 부재.
- **에이전트는 사람처럼 느껴질 만큼 실수한다.** 좋은 의미로 인격적이고 나쁜 의미로 불완전하다. 사람이 완벽하지 않아 조직과 절차가 필요하듯, 에이전트도 개별 실수를 조직 수준의 장치로 흡수해야 한다는 게 반복해 드러났다.
- **현실 사람 조직의 협업 방식과의 괴리.** 실제 조직에서 사람들은 메신저만 바라보고 일하지 않는다. 각자 자기 큐에 할 일을 쌓아두고 처리하다가, 다른 사람에게 넘길 것이 있을 때 메일 · 이슈 트래커 · 메신저 중 하나로 연락을 주고받는다. Mustang 팀에는 이 "각자 큐 + 필요할 때만 연락" 구조가 없었다.

### 문제 정의

위 관찰을 묶으면:

**에이전트를 사람처럼 인격체로 다루기 시작했으면서도, 사람이 조직에서 일하는 방식으로 협업 인프라를 짜지 못했다.**

역할 분리만으로는 반쪽이었고 — **need-to-know 소통 · 각자 큐 소유 · 절차 기반 실수 흡수** — 세 가지가 빠져 있었다.

## 목표

### 큰 그림

> **에이전트들과 현실 조직처럼 일하기.**

이 한 문장이 이번 재정비의 목표. 세부 원칙:

- **Need-to-know 기반 소통**: 각 에이전트는 자기 앞에 온 것만 처리한다. 관계없는 대화를 상시 관찰하지 않는다. 컨텍스트/토큰 낭비 제거.
- **각자 큐 소유**: 각 에이전트가 자기 큐를 갖고 자기 페이스로 처리. 실시간 동기화 강제 없음.
- **이슈 트래커를 핵심 플랫폼으로**: 사람↔에이전트 혼합 협업에 가장 자연스러운 매체는 실시간 메신저가 아니라 이슈 트래커. 개별 계정 부여가 자연스러운 Dooray가 유력 후보로 부상한 배경.
- **절차로 실수 흡수**: 개별 에이전트의 실수를 검토·리뷰·회고 같은 조직 수준의 장치로 흡수. 지금 있는 Hawkeye 검증 흐름이 이 방향의 첫 실증이고, 이후 확장 여지.

_(세부 원칙 추가 대기)_

## 방향 (핵심 결정)

### 협업 환경의 구성

재정비 후 팀은 다음 네 축의 조합으로 일한다.

**① 기록 공간 — Obsidian Vault (지속)**

- 지금까지 써 온 vault를 그대로 유지. 로컬이라 접근이 빠르고 위키 형식이라 참조가 쉽다.
- 부족했던 것은 "업무 할당" 개념뿐이며, 그 조각을 이슈 트래커로 채운다.
- 기록 내용 자체는 이후 정비 여지 있음 (별도 항목).

**② 업무 채널 — 이슈 트래커**

- 사람↔에이전트 및 에이전트↔에이전트 사이 태스크를 주고받는 통로. 예: Roy가 Breda에게 태스크를 할당. 때로는 assignee가 Kirin이 되기도.
- 태스크 본문은 **vault 문서를 참조하도록 지시**하는 것을 기본으로 한다. 예: "vault의 기획서 v1.0을 읽고 구현".
- 다만 지시 시점의 **스냅샷 보존**이 중요한 문서라면 vault 원문을 태스크에 통째로 첨부하는 것도 선택지. 툴 사용의 유연성 영역.

**③ 소스 이력 — GitHub (지속)**

- 지금까지처럼 GitHub을 소스 관리 백본으로 유지. 신규 도입 없음.

**④ 도구 확장 — 필요할 때마다 붙임**

- 에이전트가 직접 접근 가능한 도구를 점점 확장한다. 예: 얼마 전 GCP Cloud Run 배포 환경을 붙여둔 것.
- 미리 다 만들지 않고, 필요한 순간에 붙인다.

**요약**: 이슈 트래커 + 위키의 조합에 GitHub과 확장 도구가 얹힌다.

### 이벤트 채널 · 진리의 원천 (Dooray as source of truth, Monitor as trigger)

이슈 트래커와 GitHub 모두 webhook을 제공하지만, 에이전트들은 가정용 맥미니에 산다. 외부 webhook을 직접 못 받으니 **Cloudflare Tunnel + 로컬 receiver** 로 이 사이를 이어준다.

여기서 핵심 원칙: **상태의 진리는 Dooray가 이미 갖고 있다.** 우리가 별도 큐에 status(pending/in_progress/done)를 만들면 Dooray의 workflow와 이중 관리가 되어 필연적으로 어긋난다. 그러므로:

- **Dooray = source of truth** (태스크 상태 · 담당자 · 코멘트 · workflow 다 이미 있음).
- **우리 layer = 이벤트 배달 계층에 국한**. 실제 처리 필요 여부는 항상 Dooray 재조회로 판단.
- **Monitor(Claude Code built-in) = latency 최적화 채널** — durability 보증하지 않음. 놓쳐도 다음 sync가 흡수.

**Receiver의 최소 역할**

1. Webhook 수신 (Cloudflare Tunnel 뒤에 FastAPI 등).
2. 담당 agent 판단 (assignee organizationMemberId → agent name 매핑).
3. 해당 세션의 Monitor 채널로 "이벤트 있음" 알림 push (WebSocket 또는 FIFO).
4. (선택) 짧은 dedup 캐시 · 감사 로그.

**"agent가 큐를 갖는다" 의 재정의**

목표에서 얘기한 "각자 큐 소유" 는 우리 layer의 별도 큐가 아니라 **Dooray 안의 내 담당 태스크 집합** 자체를 큐로 본다. Receiver는 그 큐에 접근하는 알림 채널.

**Claude Code plugin 으로 auto-start**

Monitor는 세션이 살아있는 동안만 유효. 세션 재시작 시 다시 시작해야 함. Claude Code plugin이 monitor를 declaration으로 auto-start 해주므로, `mustang-hub-agent` 플러그인을 만들어 각 agent 세션에 활성화하면 리스너가 결정론적으로 붙는다 (모델 준수에 의존 X). 플러그인 초기 스코프는 **Monitor auto-start 만** (얇게) — 세션-측 로직은 skill(`hub`)로 캡슐화. Queue 조작 도구는 기각 (Dooray가 truth).

**폐기된 대안 (기록용)**

- 각 에이전트에 별도 agentic runner 만들기 → Claude Code 발전 혜택 못 누리므로 폐기.
- 로컬 durable 큐에 status 3단계 관리 → Dooray와 중복 관리라 폐기.
- 세션이 큐를 매 초 poll → Monitor push가 더 자연스러우니 primary 아님 (안전망으로만 유지 가능).

### 에이전트 세션 라이프사이클

```
[start]
  ↓
[drain phase]  Dooray 조회 → 액션 필요 태스크 → 처리 → 다시 조회
  ↓  (액션 필요 태스크 없을 때까지 loop)
[monitor phase]  idle + Monitor 리스닝
  ↓  (event 도착)
[triggered re-check]  Dooray 재조회 (특정 태스크 or 전체) → 액션 필요 시 처리
  ↓
back to [monitor phase]
```

**"액션 필요" 판단 룰 (초안, 확장 예정)**

- workflow=registered + assignee=me → 신규 할당, 초기 반응 필요
- workflow=working + 마지막 코멘트가 상대방 → 답변/진행 필요
- workflow=working + 마지막 액션이 나 → 액션 없음 (대기)
- workflow=closed → 액션 없음

**3중 안전망**

| 레벨 | 매커니즘 | 담당 |
|---|---|---|
| 재기동 | drain phase가 backlog 자동 흡수 | plugin skill |
| 실시간 | Monitor가 이벤트 즉시 delivery | plugin declaration |
| Idle 안전망 | 매 N분(예: 30-60분) `/loop` dynamic 로 Dooray dry-sync | skill |

Monitor가 한 채널만 죽어도 idle 안전망이 눈치채고, 안전망도 안 돌면 다음 재기동이 흡수. 손실 시나리오 없음.

### 태스크 처리 흐름 안의 도달 semantics

- Monitor notification은 **다음 turn boundary** 에 컨텍스트 삽입 (선점 없음). 세션이 다른 태스크 처리 중이면 그 이터레이션 종료 후 자연 큐잉.
- 200ms 이내 stdout 라인은 하나의 알림으로 batch.
- Idempotency는 Dooray state check로 자동 확보 — "이미 코멘트 달았나?" 를 Dooray 조회로 판단하고 필요 시만 액션.
- 처리 완료 = Dooray state 갱신 (workflow 이동 · 코멘트 · done 처리). 우리 layer엔 done 마킹 불필요.

### Dooray 사용 규범 (assignee vs mention)

- **Assignee = 현재 반응 책임자.** 태스크의 assignee 는 "지금 이 태스크를 다음으로 처리해야 할 사람" 을 명시. Receiver 는 오직 assignee 기준으로 라우팅 · 배달.
- **Mention (@Roy 등) = 문맥상 부르는 표기.** 대화 흔적 · 참조 · CC 용도. 반응 의무는 만들지 않는다.
- **반응 필요한 코멘트는 assignee 도 함께 변경.** 코멘트만 남기고 assignee 를 그대로 두면 상대는 알림을 받지 않는다. 예:
  - Roy 가 Kirin 에게 질문 코멘트 남길 때 → **동시에 assignee 를 Kirin 으로 변경**.
  - Kirin 이 Roy 에게 지시 · 재작업 요청 코멘트 남길 때 → **동시에 assignee 를 Roy 로 변경**.
- 이 규범은 사용자(사람 · 에이전트 모두) 가 지켜야 한다. Receiver 는 mention 기반 라우팅을 하지 않는다 (스코프 밖).

### 실패 처리 방침 (조직 규범)

에이전트가 태스크 처리 중 실패하면:

1. **자동 재시도 없음** — 같은 원인으로 반복 실패 확률 높음. 재시도는 사람 판단.
2. **원 지시자에게 반환** — 태스크 assignee를 원 생성자로 되돌리고 실패 코멘트 남김. 반환 없이 코멘트만 남기면 에이전트 큐에 계속 남아 무한 실패 loop → 반드시 반환.
3. **지시자가 확인 후 판단** — 원인 파악 · 재실행 · 폐기 · 다른 에이전트 재할당 중 선택. 지시자가 사람이면 Dooray 자체 알림으로 인지, 지시자가 에이전트면 그 에이전트의 큐에 자동 진입.
4. **반복 실패 관찰** — 같은 task 가 여러 번 반환-재할당-실패 반복하면 프로세스 재검토 (task 정의 부적절 · 필요한 도구 부재 · 판단 룰 부적절 등).

구현 상세 (edge case 방어 · 코멘트 포맷): [[projects/mustang-hub-agent/README]] "실패 처리 (원 지시자에게 반환)" 소절.

## Track A — 협업 인프라

**A1. Task management 도구 선정 — 완료 (2026-09-04)**
Dooray 확정 (개인 free tier 워크스페이스). 근거: 개별 계정 부여 자연, 국내 결제 마찰 낮음, Kirin 회사에서 이미 사용 중이라 익숙, REST API + unofficial CLI 존재.

**A2. `dooray-skill` 라이브러리 — 완료 (2026-09-09/10)**
Python 라이브러리 + Claude Code skill. v0.1 인증 · me · projects · tasks. v0.2 CRUD · workflow · tag · log · actions. v0.3 인증 재설계 — env `$DOORAY_API_KEY` 우선. 유닛 62 + Sandbox e2e 12 통과. 정본: [[projects/dooray-skill/README]].

**A3. Task Hub receiver — PoC 완료 (2026-09-14), 정본 진화 중**
`mustang-hub` 프로젝트로 파생 ([[projects/mustang-hub/README]]). FastAPI + Docker Compose. v0.2 실증 완료 — 라우팅 · self-loop 필터 · human 필터 · WS fan-out · payload archival · 구조화 로그. 2026-09-15 `mustang-task-hub` → `mustang-hub` 리네임.

**A4. Claude Code plugin `mustang-hub-agent` — 착수 (2026-09-15)**
설계 정본: [[projects/mustang-hub-agent/README]]. 스코프 얇게 확정 — Monitor auto-start만. Queue 조작 도구는 기각 (Dooray가 truth). Skill(`hub`)은 플러그인 내부(`skills/hub/`)로 함께 배포. Path Y 확정(command + `ws-consumer.py`, `ws:` source는 스키마 미지원 실증).

**A5. 세션 라이프사이클 skill `hub` — 설계 초안 (2026-09-15)**
Plugin에 포함. Lifecycle 4단계 (drain → monitor → triggered re-check → drain 재개) + 액션 필요 판단 룰 + 우선순위 규칙 (overdue > priority > 생성시각) + 실패 시 원 지시자 재할당 정형. 상세: [[projects/mustang-hub-agent/README]].

**A6. Idle 안전망 — skill `hub`에 포함 (2026-09-15)**
`/loop` dynamic 매 60분 wake-up 으로 drain phase 재실행. Monitor 채널 실패 · Cloudflare tunnel 순간 단절 · 세션 조용한 상태 등에서 backup. Skill 실행 정책이라 별도 인프라 없음.

**A7. Discord 실제 폐기 — 완료 (2026-09-22)**
Interactive 세션에서 Discord 접점 폐기. `scripts/start.sh` 에서 `plugin:discord@inline` 및 `--dangerously-load-development-channels` 제거. Discord 를 계속 쓸 에이전트는 별도 브릿지로 이관: [[projects/discord-agent-runner/README]] (agent-envy·lust 로 실증). 공유 memory 의 Discord 관련 엔트리 (feedback_discord_reply_tool.md 등) 는 실 세션에서 마주칠 때 삭제 예정.

## Track B — 정책 확산

**B1. 나머지 에이전트 CLAUDE.md 통일** — 미이행. Falman / envy / lina / lust. 각자 세션에서 순차. Dooray identity(별 봇 계정) 는 Kirin이 나중에 챙길 사항.

**B2. 저널 자동 트리거 기준 정의** — 유예. 매뉴얼 사용 몇 주 겪은 뒤 패턴 보고 결정.

**B3. Journal skill 실제 사용 흔적 축적** — 진행 중 (매 세션 결정 후 사용).

## 결정 기록 (연대순)

- **2026-09-03** 프로젝트 착수. Discord 폐기 방향. 각 에이전트 저널링 도입.
- **2026-09-04** Task mgmt 도구 = **Dooray** 확정. Skill 스코프 좁게 (DOORAY_API_KEY env), agent 인자 폐기 원칙.
- **2026-09-09** `dooray-skill` 프로젝트 착수 · v0.1 실증 완료 (인증 + me + projects + tasks).
- **2026-09-09** `dooray-skill` v0.2 완료 (CRUD + workflow + tag + log + actions). 유닛 59 + 통합 12 테스트.
- **2026-09-10** `dooray-skill` v0.3 완료 (인증 env-based, agent 인자 제거). launcher가 env 주입.
- **2026-09-13** **이벤트 채널 아키텍처 확정**: Dooray as source of truth · Monitor as trigger · 3중 안전망 (drain / monitor / idle dry-sync). 별도 durable 큐 불필요. Claude Code plugin으로 Monitor auto-start. 폐기 대안: 별도 agentic runner, 로컬 status 3단계 큐.
- **2026-09-13** 에이전트 세션 라이프사이클 정의: drain → monitor → triggered re-check. "액션 필요" 판단 룰 초안 4개 케이스.
- **2026-09-14** `mustang-task-hub` PoC 착수 및 검증 완료. Cloudflare Quick Tunnel → Dooray Sandbox webhook 4/4 도달. Payload 관찰: `requestOrigin.type=open-api` 를 self-loop 방지 필터로 활용 가능, `hookVersion` 문서와 실제(`version`) discrepancy 발견.
- **2026-09-14** `dooray-skill` v0.4 — `hooks.create` 추가. Hook 등록엔 프로젝트 admin 권한 필요 (Roy를 Sandbox admin으로 승격 후 등록 성공).
- **2026-09-15** `mustang-task-hub` v0.2 구현 · 실증 완료 ([[projects/mustang-hub/README]]). Self-loop / 라우팅 / WS fan-out / archival / 구조화 로그 정상. Cloudflare Quick Tunnel 뒤에서 Roy WS session에 시뮬 payload 배달 성공.
- **2026-09-15** Identity 규약: `kirin` userCode = Kirin의 사람 계정, `iizs` = Dooray/GitHub 관리자 역할. 팀 워크플로우 문서 정리 시 반영 예정 (receiver 로직엔 영향 없음).
- **2026-09-15** `mustang-agent-plugin` 설계 초안 ([[projects/mustang-hub-agent/README]]). Plugin scope 얇게, queue 도구 기각, lifecycle 4단계, 우선순위 3축. Plugin monitor의 `ws:` source 지원 여부는 실증 대기 (Path X→Y fallback 준비).
- **2026-09-15** Path Y 확정 — plugin monitor 매니페스트는 `command`만 인식(`ws:` unrecognized), 실증 로그로 확인. WS consumer 스크립트 프로토타입 실증 성공.
- **2026-09-15** 사람 계정 배달 정책: receiver `TASK_HUB_HUMAN_AGENTS` 로 지정된 agent 이름은 WS 배달 skip (Dooray 자체 알림에 위임). mustang-task-hub c9e587f 반영. Kirin(kirin) 이 첫 대상.

## 미결 / 다음 단계

**당장 착수 후보 (Track A 순서)**
- `mustang-task-hub` receiver 프로젝트 착수 (FastAPI + Cloudflare Tunnel + 라우팅).
- Claude Code plugin `mustang-task-hub` 스켈레톤 (Monitor auto-start declaration).
- 세션 라이프사이클 skill (drain + triggered re-check + dry-sync).

**결정 유예**
- Plugin scope 최종 (얇게 A vs 중간 C — receiver 만든 뒤 판단).
- "액션 필요" 판단 룰셋 세부 (담당자 여러 명 / cc-only / 파일 첨부 반응 등).
- 실패 처리 정책 (자동 재시도 X, 사람 escalate — 세부 채널 미정).
- Receiver dedup 캐시 정책 (TTL · 저장소).
- 향후 GitHub webhook 같은 소스 흡수 시 receiver 엔드포인트 분리 vs 통일.
- 저널 자동 트리거 기준 (B2).

**폐기된 대안** (재검토 필요 시 참조)
- 각 에이전트에 커스텀 agentic runner 구축 → Claude Code 발전 혜택 못 누림.
- 로컬 durable 큐에 status 3단계 (pending/in_progress/done) → Dooray와 중복 관리.
- Poll-only 웨이크업 (Monitor 없이 세션 주기 조회) → Monitor push가 자연스러움, poll은 idle 안전망 용도로 격하.
