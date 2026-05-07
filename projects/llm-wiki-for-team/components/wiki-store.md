# Wiki Store

LLM이 생성/수정한 위키 페이지의 영구 저장소.

> Git 저장 방식: [ADR-0001 — Normal repo](../decisions/0001-normal-vs-bare-repo.md)

## 저장 형태

- Normal Git repo (working tree 있음), gitpython으로 read/write/commit
- Obsidian MD 포맷: 페이지 간 내부 링크는 `[[페이지명]]` 형식
- YAML frontmatter: `type`, `created`, `last_updated`, `tags`, `sources`

## 페이지 구조

```yaml
---
type: [concept | process | policy | reference | glossary]
created: YYYY-MM-DD HH:MM:SS
last_updated: YYYY-MM-DD HH:MM:SS
tags: []
sources:
  - "원본파일.pdf"
---

# Page Title

본문. 다른 페이지 참조: [[다른-페이지]]
```

## 예약 파일 (repo root)

### `index.md`

위키 전체 지도. 모든 Ingest/Edit 완료 후 자동 갱신.

```markdown
# Wiki Index
*Last updated: YYYY-MM-DD HH:MM:SS*

| 페이지 | 타입 | 한줄 요약 | 최종 수정 |
|--------|------|-----------|-----------|
| [[페이지명]] | concept | ... | YYYY-MM-DD |
```

- 에이전트가 위키 구조 파악 시 첫 번째로 읽는 파일
- v0.1: LLM이 매번 전체 갱신 (비용 이슈 — Known Limitation)
- v0.2 후보: incremental 갱신

### `log.md`

모든 Job 이력. append-only (최신 항목이 상단). 시스템이 자동 기록.

> 기록 범위 결정: [ADR-0010 — log.md 요청 전문 미포함](../decisions/0010-log-md-no-request-text.md)

- **포함**: `job_id` + 결과(생성/수정된 페이지 목록)
- **미포함**: 요청 전문 (jobs 테이블에만 보관)
- git에 포함되어 clone 시 전체 공개됨 — 민감한 요청 내용 노출 방지

```markdown
# Sheska Operation Log

## 2026-05-06T19:00:00Z | INGEST | SUCCESS | job_id: abc123
- source: product_spec_v2.pdf
- created: [[제품개요]], [[기능정책]]
- updated: [[용어집]]

## 2026-05-06T18:30:00Z | EDIT | SUCCESS | job_id: def456
- page: [[기능정책]]
- modified: [[기능정책]]

## 2026-05-06T18:00:00Z | INGEST | FAILED | job_id: ghi789
- source: old_spec.pdf
- error: LLM context limit exceeded
```

### `_sheska.yaml`

Sheska 서버 설정. 원본 접근 base URL 보관. 자세한 내용: [Source Store](source-store.md).

```yaml
source_base_url: "https://sheska.yourcompany.com/api/sources"
```

## 관련 SC

SC-11-b, SC-12, SC-13, SC-14, SC-16-b, SC-16-c, SC-17, SC-26
