# Sheska v0.2 Backlog

> v0.1 MVP 실사용 후 도출된 개선/추가 항목 모음.
> 우선순위 미확정 — 다음 이터레이션 킥오프 시 정렬한다.

---

## 1. Web UI `[[wikilink]]` 처리 (SC-20 완성)

**현 상태**: SPEC §SC-20 [Advisory] — Obsidian에서는 동작하지만 Web UI에서 `[[페이지명]]`은 plain text로 보임.

**필요 작업**:
- Markdown 렌더러(`ReactMarkdown`)에 커스텀 plugin 추가
- `[[페이지명]]` → `<a href="/wiki/페이지명">페이지명</a>` 변환
- 내부 페이지 존재 여부 확인 → 없는 링크는 visual cue (회색 등)

**관련 파일**: `frontend/src/app/wiki/[...slug]/page.tsx`

---

## 2. 신규 계정 생성 UI

**현 상태**:
- 첫 Admin은 `backend/scripts/seed_admin.py` CLI로 생성
- 이후 사용자는 `POST /api/users/` (Admin 전용 API)로만 생성 가능
- UI에서 직접 사용자 관리 화면이 없음

**필요 작업**:
- `/users` 또는 `/admin/users` 페이지 신설 (Admin only)
  - 사용자 목록 조회
  - 신규 사용자 생성 폼 (email, password, role)
  - 역할 변경 (Member ↔ Admin)
  - 비활성화
- 첫 Admin 부트스트랩: `seed_admin.py`는 유지하되, 환경변수 `INITIAL_ADMIN_EMAIL/PASSWORD` 지원으로 자동 시딩 옵션 추가 검토

**관련 파일**: `backend/app/routes/users.py` (백엔드는 이미 완비), `frontend/src/app/` (신규 화면 필요)

---

## 3. Prompt 다듬기 (weak instruction-following 모델 호환)

**배경**: Ollama gemma 등 instruction following이 약한 모델이 평론형 응답 / 부분 frontmatter / 마커 누락 등 비표준 출력을 자주 냄. v0.1에서 파서를 견고화해 흡수했으나, 출력 품질 자체를 끌어올릴 여지 있음.

**필요 작업**:
- `prompts/ingest.txt`, `prompts/edit.txt` 추가 강화
  - few-shot 예시 1개 → 2~3개로 확장 (긴 문서, 한국어, 다중 페이지 케이스 포함)
  - 부정 예시 ("이렇게 답하지 마세요") 추가
- LiteLLM `response_format` / Ollama `format: json` 활용 가능성 검토 — 구조화 출력으로 받고 시스템이 마크다운 변환
- 모델별 프롬프트 분기(?) — 운영 복잡도 증가하므로 신중

---

## 4. 단일 소스 → 다중 위키 페이지 분리 ⭐

**배경**: 큰 원본(예: 30페이지 PDF)을 1개 위키 페이지로 합치면 가독성/검색성 저하. 주제 단위로 분리하는 게 위키의 본질.

**현 상태 (설계상)**:
- `=== FILE: name.md ===` 마커 포맷이 다중 페이지 출력 지원
- `parse_llm_pages`가 마커 단위 split

**현 상태 (실측)**:
- `prompts/ingest.txt` 예시가 단일 페이지뿐이라 LLM이 분리 시도 안 함
- 결과적으로 1소스 → 1페이지 패턴 고착

**필요 작업**:
- 프롬프트에 "분리 기준" 명시:
  - 주제가 명확히 구분되는 섹션 2개 이상이면 별도 파일
  - 한 페이지 권장 길이 상한 (예: 2000자) 가이드
- 다중 페이지 예시 추가
- 파일명 충돌 처리 강화 (현재 같은 stem이면 덮어쓰기)
- 페이지 간 자동 `[[링크]]` 생성 가이드 (분리된 페이지가 서로 참조)

**열린 질문**:
- 분리 결정을 LLM에 맡길지, 시스템이 후처리로 분리할지
- "축적 원칙" 하에서 기존 페이지가 있을 때의 분리 전략

---

## 5. UI 정돈

**필요 작업** (우선순위 실사용 후 결정):
- 페이지 정렬, 타이포그래피, 여백 일관성
- Jobs 탭 가독성 (긴 요청 텍스트 truncate, 시간 포맷 ko-KR 등)
- 에러 표시 개선 (현재 `alert()` 기반 다수)
- 로딩 상태 표시
- 모바일 반응형

---

## 메타: SPEC 문서 관리 체계

**Kirin 메모 (2026-05-07)**: SPEC v0.1~v0.7로 진행하면서 변경 추적/검색이 점차 어려워짐. 회사 업무에서도 같은 고민이 있어 다음 이터레이션 전에 한 번 정리할 가치 있음.

**검토 후보**:
- 변경 단위(시맨틱 vs incremental) 가이드라인
- v 번호와 git tag 연동
- Open Issues / Backlog / Done의 라이프사이클
- 별도 리뷰 프로세스 vs 인라인 토론

---

## 다음 이터레이션 시작 전 체크리스트

- [ ] 이 백로그 우선순위 정렬 (Kirin)
- [ ] 각 항목 별도 GitHub Issue로 분리할지 결정
- [ ] SPEC v0.7 vs 새 버전(v0.8) 또는 SPEC_v0.2.md 분기 결정
- [ ] Hawkeye에게 SC 보강 사전 협의
