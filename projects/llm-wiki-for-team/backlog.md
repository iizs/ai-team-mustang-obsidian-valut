# Backlog

정렬되지 않은 아이디어/작업 풀. 이터레이션 킥오프 시 여기서 끌어와 우선순위·담당·DoD 부여한다.

> 이터레이션에서 작업 중인 항목은 [iterations/](iterations/) 참조.

---

## 사용자 경험

### B-1. Web UI `[[wikilink]]` 클릭 이동 (SC-20 완성)

- **현 상태**: SPEC §SC-20 [Advisory] — Obsidian에서는 동작하지만 Web UI에서 `[[페이지명]]`은 plain text로 보임
- **필요 작업**:
  - Markdown 렌더러(`ReactMarkdown`)에 커스텀 plugin 추가
  - `[[페이지명]]` → `<a href="/wiki/페이지명">페이지명</a>` 변환
  - 내부 페이지 존재 여부 확인 → 없는 링크는 visual cue (회색 등)
- **관련 파일**: `frontend/src/app/wiki/[...slug]/page.tsx`

### B-2. 신규 계정 생성 UI

- **현 상태**:
  - 첫 Admin은 `backend/scripts/seed_admin.py` CLI로 생성
  - 이후 사용자는 `POST /api/users/` (Admin 전용 API)로만 생성 가능
  - UI에서 직접 사용자 관리 화면이 없음
- **필요 작업**:
  - `/users` 또는 `/admin/users` 페이지 신설 (Admin only)
    - 사용자 목록 조회
    - 신규 사용자 생성 폼 (email, password, role)
    - 역할 변경 (Member ↔ Admin)
    - 비활성화
  - 첫 Admin 부트스트랩: `seed_admin.py` 유지하되, 환경변수 `INITIAL_ADMIN_EMAIL/PASSWORD` 자동 시딩 옵션 검토

### B-3. UI 정돈

- 페이지 정렬, 타이포그래피, 여백 일관성
- Jobs 탭 가독성 (긴 요청 텍스트 truncate, 시간 포맷 ko-KR 등)
- 에러 표시 개선 (현재 `alert()` 기반 다수)
- 로딩 상태 표시
- 모바일 반응형

---

## LLM Pipeline

### B-4. 단일 소스 → 다중 위키 페이지 분리 ⭐

- **배경**: 큰 원본(예: 30페이지 PDF)을 1개 위키 페이지로 합치면 가독성/검색성 저하. 주제 단위 분리가 위키 본질
- **현 상태 (설계상)**:
  - `=== FILE: name.md ===` 마커가 다중 페이지 출력 포맷
  - `parse_llm_pages`가 마커 단위 split
- **현 상태 (실측)**:
  - `prompts/ingest.txt` 예시가 단일 페이지뿐이라 LLM이 분리 시도 안 함
  - 결과적으로 1소스 → 1페이지 패턴 고착
- **필요 작업**:
  - 프롬프트에 "분리 기준" 명시 (주제 2개 이상 / 길이 상한)
  - 다중 페이지 예시 추가
  - 파일명 충돌 처리 강화
  - 페이지 간 자동 `[[링크]]` 가이드
- **열린 질문**:
  - 분리 결정을 LLM에 맡길지, 시스템 후처리로 분리할지
  - "축적 원칙" 하에서 기존 페이지가 있을 때의 분리 전략

### B-5. 프롬프트 다듬기 (weak 모델 호환)

- Ollama gemma 등 instruction following이 약한 모델 대응 강화
- few-shot 예시 1개 → 2~3개로 확장 (긴 문서, 한국어, 다중 페이지 케이스)
- 부정 예시 ("이렇게 답하지 마세요") 추가
- LiteLLM `response_format` / Ollama `format: json` 활용 검토
- 모델별 프롬프트 분기 가능성 검토 (운영 복잡도 트레이드오프)

---

## 컴포넌트

### B-6. Lint 도입 ([ADR-0005](decisions/0005-lint-deferred.md) 해소)

- 위키 health-check, 스테일/모순 페이지 정리
- Job Queue에 LINT 타입 추가 가능성
- 범위: 전체 vs 변경분 vs 선택 페이지 — 비용 대비 가치 검토 필수

### B-7. 원본 soft-delete (SC-9)

- `_trash/` 디렉토리로 이동
- 활성 목록에서 제거
- Admin 권한
- Lint 도입 시 `_trash/` 원본을 참조하는 위키 페이지 정리 트리거 가능

### B-8. Job 병렬 처리

- v0.1: 워커 1개 (FIFO)
- v0.2+: 워커 N개 검토
- 동일 페이지 동시 수정 충돌 처리 정책 필요
- 실현 시 [ADR-0006](decisions/0006-fastapi-backgroundtasks.md) Superseded 가능성

---

## 인프라

### B-9. SSO

- v0.1: 이메일/패스워드 + JWT
- v0.2+: SAML/OIDC 지원
- Keycloak 도입 검토

### B-10. 멀티 테넌시

- 한 Sheska 인스턴스에서 여러 팀(테넌트) 운영
- 위키 repo, 사용자 DB, Job Queue 격리

### B-11. 원본 출처 추적 가시화 (source graph)

- 위키 페이지 ↔ 원본 파일 의존 그래프 시각화
- 원본 변경 시 영향받는 위키 페이지 식별

### B-12. URL 소화 (크롤링)

- v0.1: 파일 업로드만
- v0.2+: URL 입력 → 크롤링 → INGEST
- robots.txt, 인증, 동적 콘텐츠 등 별도 영역

### B-13. DOCX 지원

- v0.1: PDF/TXT/MD만
- v0.2+: `python-docx` 도입 검토

---

## 메타

### M-1. SPEC 문서 관리 체계 ✅ (v0.1 회고에서 진행)

- v0.1 마무리 시점에 단일 SPEC.md → 분산 구조로 리팩터
- README/REQUIREMENTS/ARCHITECTURE/components/decisions/iterations/reviews 분리
- **현 상태**: 이 백로그 작성 시점에 진행 중. 완료되면 항목 제거.

### M-2. v0.2 킥오프 체크리스트

- [ ] 이 백로그 우선순위 정렬 (Kirin)
- [ ] 각 항목 별도 GitHub Issue로 분리할지 결정 (현재는 vault 내 관리)
- [ ] v0.2 iteration 파일 스코프 확정
- [ ] Hawkeye에게 SC 보강 사전 협의 (신규 SC = SC-31~로 시작)
