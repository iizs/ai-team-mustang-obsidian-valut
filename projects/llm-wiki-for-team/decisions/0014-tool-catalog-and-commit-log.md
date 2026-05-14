# ADR-0014 — Agentic Tool 카탈로그 + Commit/Log 정책

- **상태**: Accepted (v0.4 킥오프 — 2026-05-14)
- **결정일**: 2026-05-14
- **결정자**: Kirin, Roy (+ Breda/Hawkeye 합동 검토)

## 컨텍스트

[ADR-0013](0013-agentic-loop-provider-adapter.md)으로 agentic loop가 도입되면, LLM이 호출할 도메인 tool 카탈로그가 필요하다. 또한 매 tool 호출(특히 write 계열)을 git commit / log.md에 어떻게 기록할지도 결정해야 한다.

## 결정

### Tool 카탈로그 (v0.4 기준)

| Tool | 입력 | 출력 | 호출 조건 / 안전장치 |
|------|------|------|--------------------|
| `get_index()` | — | 표제어 + 한줄요약 표 (현 `index.md` 내용 + 파싱) | 자유 호출 |
| `read_page(path)` | `path: str` | frontmatter dict + body str | 자유 호출. 페이지 부재 시 `null` 반환 |
| `read_source(filename)` | `filename: str` | 원본 텍스트 (PDF 추출 포함) | INGEST 흐름에서만 의미. 다른 흐름에서도 호출 가능하지만 보통 미사용 |
| `search_pages(query)` | `query: str` | 매칭 페이지 목록 + 매칭 라인 snippet | 자유 호출. 기본 구현은 ripgrep 또는 단순 grep |
| `write_page(path, content)` | `path: str`, `content: str` (full markdown + frontmatter) | 성공 여부 | 신규 작성 또는 전체 덮어쓰기용. 예약 파일 거부 |
| `patch_page(path, diff)` | `path: str`, `diff: str` (unified diff format) | `{applied, hunks_applied, hunks_failed, error}` | **기존 페이지 부분 수정용** (full content 재생성 회피 → 출력 토큰 절약). LLM이 unified diff 생성 → 시스템이 context 기반으로 적용. **Kirin 결정 2026-05-14** |
| `delete_page(path, reason)` | `path: str`, `reason: str` | 성공 여부 | WIKI_COMMAND/Lint 흐름만 허용. INGEST 흐름에선 거부 (축적 원칙). 예약 파일 거부 |
| `get_backlinks(page)` | `page: str` | 백링크 페이지 리스트 | [ADR-0015](0015-backlink-frontmatter-index.md)의 frontmatter 인덱스 조회 |

#### 안전장치 (시스템이 강제)
- `write_page`/`patch_page`/`delete_page`는 예약 파일(`index.md`/`log.md`/`_sheska.yaml`/`_`로 시작) 거부
- `delete_page`는 INGEST job_type에서 호출 시 거부 (SC-63 정신 유지)
- `patch_page` context mismatch (hunk fail) 시 — 결과에 `hunks_failed` 명시 + `error` 메시지에 mismatch된 line 정보. LLM이 read_page 재실행 후 재시도 가능 (max 3 retry). 모든 hunk 실패 시 no commit.
- `path`는 항상 kebab-case slug + `.md`. path traversal 거부

### `search_pages` v0.4 도입 결정 (Kirin 보강)

- v0.4에 포함. tool 자체 호출은 LLM 비용 아님 (시스템 grep 실행)
- LLM이 `get_index` + `read_page`만 쓰는 것보다 검색 가능하면 의도 적중도 ↑
- 단순 구현: `ripgrep` 또는 Python `re` 기반 grep, frontmatter+body 모두 매칭

### `write_page` vs `patch_page` 분리 + diff 형식 (Kirin 결정 2026-05-14)

- 신규 페이지 작성: `write_page` (full content 필요)
- 기존 페이지 수정: `patch_page` — **unified diff format**
- LLM이 두 tool 중 적절히 선택. `agent_system.txt`에 "신규는 write, 수정은 patch" 가이드 명시.

**patch_page 형식 — POSIX/git 표준 unified diff:**
- `difflib.unified_diff` 호환
- 시스템은 **context 기반 fuzzy match로 적용** (LLM이 line number 부정확해도 안전)
- Context line 길이: **5줄** 권장 (위키 짧은 줄/자연어에서 3줄로 충돌 가능성)
- 라이브러리: Python `unidiff` 또는 `patch-ng` 검토 (Breda 선택)

**왜 unified diff (search-replace 대비):**
- 출력 토큰 절약 (변경 hunk + 짧은 context만)
- POSIX/git 표준이라 디버깅/도구 호환
- 위키 markdown 본문(자연어 줄 단위)에 자연스러움
- Claude가 unified diff 생성 능력 충분 (Anthropic-only 결정 하에 더 안전)

**`last_updated` frontmatter 갱신:**
- LLM이 diff에 포함 안 해도 됨. **시스템이 매 write/patch 후 자동 갱신** (LLM 부담 ↓)

### Commit 단위

- **현재 결정 (v0.4): 매 write/patch/delete tool 호출 = 1 commit** — 디버깅 용이.
- 이상적 결정 (Kirin 메모): 1 액션 = 1 commit (= 사용자 명령 단위로 묶기). v0.5+에서 검토.
- Commit message 패턴: `[job:<id> step:<n>] <tool>: <path>` (예: `[job:abc123 step:3] patch_page: payment.md`)

### Log 정책

- `log.md` (audit 로그) — **기존 구조 유지**. job 단위 요약만 기록 (시작/종료/요약).
- 페이지별 history — **별도 파일 생성 안 함**. `git log <pagename>.md` 또는 GitHub UI로 조회 가능.
- commit message에 `job:<id>`를 포함해 audit 로그와 git log 연관성 확보.

## 근거

- **Tool 분리 원칙**: 호출 비용/위험 수준이 다르면 tool 분리. `read_*`는 자유, `write_page`(full)/`patch_page`(edit)/`delete_page`는 위험도 다름.
- **`patch_page` 도입**: Claude Code 패턴 차용. 부분 수정의 정확성 + 토큰 절약 + 의도 명확성 모두 개선.
- **commit/log 분리**: git log가 페이지별 narrative 역할 → 별도 history 파일 불필요. `log.md`는 job 추적용 audit 로그로 단순화.

## 트레이드오프

- **Tool 수가 많아짐**: LLM이 잘못된 tool 선택할 가능성 — prompt에 가이드 명시 필요
- **`patch_page` 정확성 의존**: `old_string`이 본문에 정확히 존재해야 함. LLM이 부정확히 인용하면 edit skip. 안전한 실패지만 LLM이 적응 필요.
- **매 step commit**: git history 폭발 가능 (한 명령에 수십 commit). v0.5에서 1 명령 = 1 commit로 진화 검토.

## 영향 / 후속 작업

- `services/tools.py` 신규 — tool 정의 (Anthropic tool schema) + 실행 함수
- `services/wiki_store.py` — `patch_page` 함수 신규 (search-replace 알고리즘)
- `services/llm_client.py` — agentic loop가 tool 호출 디스패치
- `services/pipeline.py` — INGEST/WIKI_COMMAND 양쪽에서 같은 tool 사용
- 신규 SC — 각 tool 동작 검증 + 안전장치
