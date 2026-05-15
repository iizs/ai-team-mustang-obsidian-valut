# ADR-0016 — Index-Trust + Lint-Repair Pattern

- **상태**: Accepted (v0.4 운영 회고 — 2026-05-15)
- **결정일**: 2026-05-15
- **결정자**: Kirin, Roy

## 컨텍스트

v0.4 agentic 흐름에서 `rename_page` tool이 backlinks 인덱스를 신뢰해 영향 범위를 결정한다. Kirin 실측에서 일부 페이지의 backlinks 인덱스가 누락되어 rename 시 referrer 일부가 갱신 안 되는 상황 발견.

해결 옵션:
- (A) `rename_page`가 wiki 전체 본문 grep으로 referrer를 다시 찾도록 fallback 추가 — O(N) 작업, 위키 성장 시 비용 증가
- (B) 인덱스 신뢰 + 부정합은 별도 정리 단계(Lint)에서 처리

## 결정

**(B) 채택 — Index-Trust + Lint-Repair 분리 원칙.**

### 운영 원칙

> **Tools는 메타(인덱스)를 신뢰해서 빠르게 동작한다 (O(1) ~ O(referrers))**
>
> **Lint는 O(N) 비싼 작업으로 인덱스 부정합 / dangling 정리를 담당한다**
>
> **두 단계 분리로 일상 운영은 빠르고, 정합성은 주기적으로 회복된다**

### 구체화

- **Tools (rename_page, get_backlinks 등)**: frontmatter `backlinks` 인덱스만 조회. 누락 시 silently miss 허용.
- **Lint (B-6, v0.5+)**: 위키 전체 스캔으로 인덱스 재구축, dangling link 발견/리포트, frontmatter 누락 보강
- **`init_backlinks.py`**: 마이그레이션/응급 재구축 도구. 사용자가 명시 트리거.

### 부정합이 발생하는 경로 (Lint가 정리)

1. v0.3.1 이전 만들어진 페이지 (backlinks 필드 부재)
2. 시스템 자동 갱신 hook의 edge case 누락
3. LLM이 표준 외 형식으로 link 작성한 경우 (예: `[[X|alias]]` 처리 차이)
4. 외부 도구로 위키 직접 편집한 경우 (Obsidian 등)
5. 비동기 race (B-8 Job 병렬 처리 도입 시)

## 근거

- **위키는 성장한다** — Tools가 매번 O(N) 스캔하면 위키 규모에 따라 응답 지연 누적
- **부정합은 드물고 사용자 직접 영향 작음** — `[[삭제된페이지]]` 클릭은 SC-39 안내로 무해 처리
- **Lint를 비싸게 둘 수 있음** — 백그라운드 Job 또는 명시 트리거로 동작, 사용자 인터랙션 차단 X
- **Claude Code 등 Agent 도구의 일반 패턴** — fast read + scheduled repair 분리

## 트레이드오프

- 인덱스 누락 시 사용자가 즉시 인지하기 어려움 — Lint 실행 전까지 dangling 상태 잔존
- 완전 정합성을 즉시 원하면 사용자가 `init_backlinks.py --rebuild-backlinks` 수동 실행 필요
- Lint 정착 전까지(=B-6 도입 전까지)는 manual repair에 의존

## 영향 / 후속 작업

- **B-6 (Lint 도입)** 본문 보강: backlinks 재구축, dangling link, frontmatter 누락 보강 명시
- v0.5+ Lint Job 도입 시 본 ADR 원칙 따라 구현
- 다른 미래 인덱스(예: tag 인덱스, 검색 인덱스 등)도 같은 원칙 적용 가능
