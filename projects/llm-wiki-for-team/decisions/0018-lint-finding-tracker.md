# ADR-0018 — Lint Finding Tracker

- **상태**: Accepted (v0.5 킥오프 — 2026-05-18)
- **결정일**: 2026-05-18
- **결정자**: Kirin, Roy

## 컨텍스트

[ADR-0017 Lint Tier Model](0017-lint-tier-model.md)이 정의한 framework 위에서 **lint finding을 추적하는 시스템**을 v0.5에 구축한다.

검토 과정에서 GitHub Issues 같은 외부 issue tracker 통합도 논의했지만, **self-hosted 약속과의 충돌 + GitHub 계정 의존성**으로 폐기 (지금은 보류, 미래 재검토 가능). 자체 단순 tracker로 진행.

핵심 원칙 — **거대화 회피**. 풀스택 issue tracker 흉내 X. lint 흐름에 필수인 것만.

## 결정

### 데이터 모델

```sql
CREATE TABLE lint_findings (
    finding_id           uuid PRIMARY KEY,
    category             varchar NOT NULL,    -- enum
    source               varchar NOT NULL,    -- enum
    status               varchar NOT NULL,    -- enum (default 'open')
    page_path            varchar,             -- nullable (예: 위키 전체 통계 finding)
    description          text NOT NULL,
    details              jsonb,               -- 카테고리별 메타
    created_at           timestamp NOT NULL DEFAULT now(),
    reported_by          varchar NOT NULL,    -- user email or "agent:lint"
    decided_by           varchar,             -- nullable
    decided_at           timestamp,           -- nullable
    resolution_reason    text                 -- 한 줄
);

CREATE INDEX idx_findings_status ON lint_findings(status);
CREATE INDEX idx_findings_category ON lint_findings(category);
CREATE INDEX idx_findings_page ON lint_findings(page_path);
CREATE INDEX idx_findings_created ON lint_findings(created_at DESC);
```

### Enum 정의

**`category`** (확장 가능, v0.5에 실제 사용 = `user_reported`만):
- `dangling_link` — 존재 안 하는 페이지 가리키는 `[[X]]`
- `orphan_page` — 어디서도 참조 안 받는 페이지
- `frontmatter_missing` — 필수 frontmatter 필드 부재
- `stale` — 오래 갱신 안 됨 / source 변경 후 미반영
- `duplicate` — 의미적 중복 페이지
- `quality_low` — 빈/저품질 페이지
- `split_candidate` — 분할 후보
- `merge_candidate` — 통합 후보
- `tag_inconsistency` — 태그 일관성 문제
- `user_reported` — 사용자가 직접 신고
- `other`

**`source`** (확장 가능, v0.5에 실제 사용 = `user:web`만):
- `lint:tier1` (자동 검출 — v0.5.1+)
- `lint:tier2` (LLM judgment — v0.5.2+)
- `user:web` (UI 신고 — v0.5)
- `agent:explorer` (미래)

**`status`**:
- `open` — 신고/검출 직후
- `acknowledged` — 사람 또는 에이전트가 인지 + 정리 필요로 판단
- `wont_fix` — 사람 또는 에이전트가 정리 안 함 결정
- (~~`resolved`~~ — v0.5.1+에서 Tier 3 실행 후 도입)

### 의도적 제외 사항

Kirin 결정 (거대화 회피):

- **`severity` 필드 없음** — 판단 비용 vs 가치 적음
- **`dedup_key` / 중복 방지 로직 없음** — 매 신고/검출마다 신규 finding 생성. 중복 부담은 운영 관찰 후 결정 (v0.5.x에서 필요 시 추가)
- **`resolved` → `open` revive 없음** — status 한 방향(open → ack | wont_fix). 같은 issue 재발생 시 신규 finding 생성, 기존 status는 그대로
- **다중 코멘트/discussion 없음** — `resolution_reason` 단일 필드만
- **assignee 없음** — `decided_by`로 추적 정도
- **label/milestone 없음** — `category`로 갈음

### Status 흐름

```
open  ──┬─→ acknowledged
        └─→ wont_fix

(v0.5.1+에 추가)
acknowledged ──→ resolved (Tier 3 실행 후)
```

**v0.5에는 acknowledged 받아가는 주체 없음** — status change는 동작하지만 후속 처리 agent 미구현. v0.5.1+에서 Lint Agent가 acknowledged 이슈를 받아 Tier 3 실행.

### API

| Method | Path | 권한 | 설명 |
|--------|------|------|------|
| POST | `/api/lint/findings` | 인증 사용자 | 사용자 신고 (page_path + description + 선택 category) |
| GET | `/api/lint/findings` | 인증 사용자 | 목록 조회. query: `status`, `category`, `source`, `page`, `size` (default 20) |
| GET | `/api/lint/findings/{id}` | 인증 사용자 | 상세 조회 |
| PATCH | `/api/lint/findings/{id}` | 인증 사용자 | status 변경 (ack/won't fix) + resolution_reason. `decided_by`/`decided_at`은 시스템 자동 |
| DELETE | (없음) | — | 삭제 미지원 (audit trail 보존) |

응답 형식 (목록): `{items, total, page, size}` — Jobs 탭과 일관.

### UI

**신규 `/lint` 페이지**:
- finding 목록 표 (created_at desc 정렬)
- 필터: status (default: `open`), category, source
- 페이지네이션 (Jobs 탭과 동일 패턴)
- 항목 클릭 → 상세 view + ack/won't fix 버튼 + reason 입력창

**위키 페이지 (`/wiki/[...slug]`)**:
- "이 페이지에 문제 신고" 버튼 (페이지 속성 영역 또는 별도 위치)
- 클릭 시 modal — 카테고리 선택 (default `user_reported`) + description text area + 제출
- 제출 시 `POST /api/lint/findings` 호출

**(보류 — v0.5.x)**:
- 페이지 조회 시 그 페이지 관련 finding 인라인 표시 — 페이지 UI 정돈 시 같이 고려 (Kirin 결정)

### Reviewer 정책 (코드 상수)

```python
AUTO_ACK_SOURCES = {"lint:tier1"}   # v0.5.1+ scanner 도입 시 활성화
MANUAL_REVIEW_SOURCES = {"lint:tier2", "user:web", "agent:explorer"}
```

v0.5에서 `lint:tier1` finding은 발생 안 함. 모든 finding은 `user:web` → 사람 review.

## 근거

- **단순성 강조** — Kirin 핵심 가이드. issue tracker 풀스택 흉내 회피
- **GitHub 통합 거절** — self-hosted 약속 + 계정 의존성. 자체 최소 시스템이 적합
- **Source/category enum 확장 가능 설계** — v0.5에선 user:web만 활성이지만 모델은 모든 tier 수용
- **dedup 의도적 미도입** — 거대화 회피. 운영 관찰 후 필요 시 추가
- **Audit trail 보존** — DELETE 미지원, 이력 영구. 결정자/시점/이유 추적

## 트레이드오프

- **dedup 부재 → 같은 issue 매번 새로 생기는 운영 부담 가능** — 사용자가 같은 페이지 여러 번 신고할 가능성. v0.5.x에서 dedup_key 도입 옵션 열어둠
- **v0.5에 acknowledged 받아가는 주체 없음** — 시스템이 ack된 finding을 "그저 보관". 사용자 입장에서 "고친다"는 행위가 안 일어남. v0.5.1까지 한 phase 기다림. 가치 작아 보일 수 있지만 framework 정착이 우선
- **GitHub 같은 외부 협업 도구의 풍부함 포기** — self-hosted 약속의 대가

## 영향 / 후속 작업

- `backend/app/models/lint_finding.py` 신규 (SQLAlchemy ORM)
- `backend/app/routes/lint.py` 신규 (API)
- `backend/scripts/init_env.py` — `lint_findings` 테이블 생성 추가
- `frontend/src/app/lint/page.tsx` 신규 (목록/필터)
- `frontend/src/app/lint/[id]/page.tsx` 또는 modal (상세)
- `frontend/src/app/wiki/[...slug]/page.tsx` — "문제 신고" 버튼 + modal
- 신규 SC (Hawkeye 작성, SC-81~): finding 생성/조회/처리, 사용자 신고 흐름, status 변경 권한, audit trail
- nav menu에 `/lint` 진입점 추가

## v0.5.1+ 예약

- Tier 1 Scanner 도입 (dangling_link 등 자동 검출 + 자동 ack)
- Tier 3 실행 (acknowledged → resolved, Lint Agent + Plan Review UI)
- dedup_key 도입 검토 (v0.5 운영 결과 따라)
- 외부 통합 재검토 (GitHub Issues / Gitea / 등)
