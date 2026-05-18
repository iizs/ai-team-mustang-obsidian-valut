# ADR-0017 — Lint Tier Model

- **상태**: Accepted (v0.5 킥오프 — 2026-05-18)
- **결정일**: 2026-05-18
- **결정자**: Kirin, Roy

## 컨텍스트

[ADR-0005](0005-lint-deferred.md)/[ADR-0008](0008-llm-accumulates-lint-cleans.md)/[ADR-0016](0016-index-trust-lint-repair.md)에서 Lint는 "LLM은 축적, 정리는 Lint", "Tools fast trust, Lint slow repair" 원칙을 명확히 했다. v0.5에서 Lint를 본격 도입하면서 **Lint가 수행하는 작업의 위험 수준별 분류 framework**가 필요하다.

Kirin 통찰: Lint 대상 작업에는 **기계적으로 처리 가능한 것**, **가치 판단이 필요한 것**, **사용자 승인이 필요한 것**이 섞여 있다. 이 셋은 자동화 정도, 책임 주체, 비용이 모두 다르므로 framework로 분리 정의한다.

## 결정

**Lint 작업을 3-tier로 분류한다.**

### Tier 1 — Mechanical (결정적, 자동)
- **주체**: 시스템 (LLM 미관여)
- **위험**: 매우 낮음 (결정적 결과)
- **자동화**: 백그라운드 실행 가능
- **예시**:
  - Backlinks 인덱스 재구축
  - Frontmatter 누락 기본값 채우기
  - Dangling `[[link]]` 검출
  - Orphan 페이지 검출
  - 위키 메트릭 산출 (페이지 수, 본문 크기, backlink 분포 등)

### Tier 2 — Judgment (LLM 판단, report only)
- **주체**: LLM (agentic loop)
- **위험**: 중간 (판단 비결정적, 단 결과는 리포트만 — 실행 안 함)
- **자동화**: 부분 (LLM이 판단해 finding 생성, 정리는 Tier 3)
- **예시**:
  - 스테일 페이지 식별
  - 모순/중복 페이지 식별
  - 빈/저품질 페이지 식별
  - 분할/통합 후보 제안
  - Hub 페이지 (지나친 참조 집중) 식별
  - 태그/카테고리 일관성 분석

### Tier 3 — Confirmation (사용자 승인 필수)
- **주체**: 사용자 (또는 신뢰 학습된 에이전트)
- **위험**: 높음 (되돌리기 어려움)
- **자동화**: 사용자가 plan 보고 명시 승인 후 시스템/에이전트가 실행
- **예시**:
  - 페이지 삭제
  - 페이지 통합 (합치고 한쪽 삭제)
  - 페이지 분할
  - 콘텐츠 재작성/요약 압축
  - Stale 페이지 재구성 (source 변경 반영)

## v0.5 적용 범위

**v0.5는 framework 정의 + tracker 인프라만 도입**, scanner/agent는 단계 도입:

| Tier | v0.5 | v0.5.1 | v0.5.2 |
|------|------|--------|--------|
| Tier 1 Scanner | ❌ | ✅ | |
| Tier 2 Scanner | ❌ | | ✅ |
| Tier 3 실행 | ❌ | ✅ | |
| Finding 모델 + UI | ✅ | | |
| 사용자 신고 (모든 tier로 push 가능) | ✅ | | |

→ **v0.5 동안 시스템에 들어오는 finding은 사실상 사용자 신고뿐.** 그러나 모델은 모든 source/category enum을 미리 정의해 후속 phase에서 자연스러운 확장.

## 카테고리별 Reviewer 정책 (v0.5)

코드 상수로 매핑 (정책 관리 UI는 v0.6+):

| source | 기본 reviewer | 동작 |
|--------|--------------|------|
| `lint:tier1` (자동 검출, 결정적) | `agent:lint` | 자동 ack (v0.5.1+ scanner 도입 시) |
| `lint:tier2` (LLM 판단) | 사람 (admin) | 사람 review 대기 (v0.5.2+) |
| `user:web` | 사람 (admin) | 사람 review 대기 (v0.5에 유일한 active source) |
| (미래 `agent:explorer`) | 사람 | 검토 |

## 근거

- **위험 수준 기반 분류는 자동화 정책 결정의 자연 축** — Kirin 통찰
- **점진 도입 (v0.5 → v0.5.1 → v0.5.2)** — 매 phase에서 가치 검증 후 다음 단계
- **Framework를 먼저 정의**해 후속 scanner/agent들이 일관된 모델 위에서 동작
- **사용자 신고 path가 모든 tier의 fallback**: scanner 도입 전이라도 사용자가 직접 issue 만들 수 있음 → v0.5 단독 가치 확보

## 트레이드오프

- **v0.5 단독으로는 자동 검출 없음** — finding은 사용자 신고만. 가치가 약해 보일 수 있지만 인프라/UI는 향후 phase 모두에 재사용
- **3-tier 경계가 항상 명확하진 않음** — 일부 작업은 tier 사이에 걸침 (예: stale page는 식별=Tier 2, 정리=Tier 3). 카테고리별로 tier가 다를 수 있으니 데이터 모델에 tier 필드를 가두지 않음 — `source` enum + `category` enum로 표현

## 영향 / 후속 작업

- **ADR-0018 Lint Finding Tracker** — 본 framework 위에 finding 데이터 모델/UI 구현
- **v0.5.1+ Scanner ADR** (예상): Tier 1 scanner 도입 시 검출 항목/주기/비용
- **v0.5.2+ LLM Scanner ADR**: Tier 2 LLM judgment 도입 시 비용 관리 (배치, 점진 검사 등)
- **v0.5.1+ Tier 3 실행 ADR**: 사용자 승인 UX (Plan Review)
