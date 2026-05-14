# ADR-0015 — Backlink Frontmatter Index

- **상태**: Proposed (v0.4 킥오프 검토 중)
- **결정일**: 2026-05-14 (예정)
- **결정자**: Kirin, Roy

## 컨텍스트

agentic 흐름에서 표제어(페이지명) 변경 또는 페이지 병합/분리 시 해당 페이지를 참조하는 다른 페이지의 `[[링크]]`를 같이 갱신해야 한다. 매번 전체 위키를 검색하는 것은 비효율적이며, 위키가 커지면 비용 폭발.

Kirin 통찰: "문서 속성에 '백링크들'을 추가하고 그걸 믿고 추적하는 게 효과적".

## 결정

### Frontmatter 확장

각 위키 페이지의 frontmatter에 `backlinks` 필드 추가:

```yaml
---
type: concept
created: 2026-05-14 10:00:00
last_updated: 2026-05-14 12:30:00
tags: [auth, jwt]
sources:
  - "auth_design.pdf"
backlinks:
  - 결제정책
  - 환불정책
---
```

**의미**: 이 페이지를 `[[이페이지명]]` 형태로 참조하는 다른 페이지들의 stem 목록.

### 갱신 주체 — 시스템 자동

**LLM에게 맡기지 않음.** 매 `write_page`/`patch_page`/`delete_page` tool 실행 후 시스템이 자동으로 갱신:

```
1. 변경된 페이지의 body에서 [[X]] 패턴 추출 (출입 링크 = outgoing links)
2. 이전 outgoing 링크와 비교해 추가/삭제 식별
3. 추가된 링크 X: X 페이지의 frontmatter.backlinks에 현재 페이지 stem union
4. 삭제된 링크 X: X 페이지의 frontmatter.backlinks에서 현재 페이지 stem 제거
5. 삭제된 페이지 X: X를 참조하던 모든 페이지의 backlinks에서 X 제거는 자동, 그러나 본문의 [[X]] 자체는 그대로 유지 (SC-39 안내 페이지로 자연 노출, Lint(v0.5+)가 사후 정리)
```

### 마이그레이션

기존 위키에는 `backlinks` 필드 없음 → 한 번 전체 스캔으로 초기 구축 필요:
- `backend/scripts/init_backlinks.py` 신규 또는 `init_env.py`의 `--reset` 후 자동 실행
- 모든 페이지의 body에서 `[[X]]` 추출 → 역색인 구축 → 각 페이지의 frontmatter에 `backlinks` 작성 → commit
- v0.4 첫 부팅 시점에 자동 1회 실행 + 이후 매 write/patch/delete 후 점진 갱신

### LLM 활용 흐름

- LLM은 `get_backlinks(page)` tool로 영향 범위 조회
- 페이지명 변경(rename) 같은 작업 시 LLM이 backlinks 모두 read_page → 본문의 [[old]] 패턴을 [[new]]로 patch_page

## 근거

- **결정성 / 비용**: 시스템 자동 = 결정적, LLM 호출 0회. tool로 갱신 위임하면 토큰 폭발.
- **데이터 모델 위치**: frontmatter 임베드 → 페이지와 같이 git tracked, 별도 인덱스 파일 불필요, Obsidian에서도 호환.
- **양방향성**: outgoing 링크는 본문에 있고 incoming 링크는 frontmatter에 있음 — 양쪽 모두 git에 보존됨.

## 트레이드오프

- **frontmatter 크기 증가**: 인기 페이지(많은 곳에서 참조)는 backlinks 리스트가 길어짐. 페이지 본문과 frontmatter 비율 영향 — UI에서 페이지 속성 영역(SC-33)이 길어질 수 있음. v0.2 결정대로 기본 접힘이라 본문 가독성엔 영향 작음.
- **체크포인트 부정합 위험**: write 중 실패하면 backlinks가 일부만 갱신될 수 있음 — agentic loop 매 step commit과 함께 묶어 atomic하게 처리 (write_page + backlink 갱신 = 한 commit).
- **마이그레이션 비용**: 1회성. 위키가 커진 후엔 backlinks 일관성을 유지하기 어렵지만, 매 변경 시 점진 갱신이므로 정상 운영 중엔 일관 유지.

## 영향 / 후속 작업

- `services/wiki_store.py` — `update_backlinks(changed_page, old_links, new_links)` 헬퍼 추가
- `services/tools.py` (ADR-0014) — `get_backlinks(page)` tool 구현
- `services/pipeline.py` — 매 write/patch/delete 후 backlinks 자동 갱신 호출
- `services/plan.py` (또는 frontmatter helper) — frontmatter에 `backlinks: []` 기본값 + 시스템 갱신 시 set union 로직
- `backend/scripts/init_backlinks.py` 신규 (마이그레이션)
- Frontend `/wiki/[...slug]` Properties 표 — `backlinks`를 클�카능 링크 리스트로 렌더
- 신규 SC — backlink 자동 갱신, get_backlinks 정확성, 마이그레이션 동작

## 보류

- **링크 형식 표제어 변경** (rename) — LLM이 backlinks 따라가며 `[[old]]` → `[[new]]` patch_page. v0.4 agentic 흐름에 자연 포함. 별도 자동화 X.
- **본문 dangling 링크 정리** — 삭제된 페이지로의 `[[X]]`는 SC-39 안내로 자연 처리. 정리는 Lint(v0.5+).
