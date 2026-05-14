# ADR-0011 — Ingest 파이프라인 다단계화 (Phase A)

- **상태**: Accepted (v0.3 킥오프 — 2026-05-08)
- **결정일**: 2026-05-08
- **결정자**: Kirin, Roy (+ Breda/Hawkeye 합동 검토)

## 컨텍스트

v0.1/v0.2 Ingest 파이프라인은 **원본 전체 → 단일 LLM 호출 → `=== FILE: ===` 마커 출력 파싱 → wiki 페이지 commit** 구조다. 다음 한계가 실측에서 드러남:

1. **기존 위키 미인지** — LLM이 위키에 이미 무엇이 있는지 모르는 상태로 새 페이지를 만든다 → 같은 주제 두 번 ingest 시 중복 페이지 생성
2. **다중 페이지 분리 안 함** — `prompts/ingest.txt` 예시가 단일 페이지뿐 → 1소스→1페이지 패턴 고착
3. **마커 기반 파싱의 취약성** — weak instruction-following 모델(Ollama gemma 등)이 마커 무시 → 다양한 fallback 필요해짐
4. **출력 구조의 의도 부재** — "이 페이지가 신규인지 통합 대상인지" LLM이 표현할 수단 없음

위 문제는 위키의 본질(축적 + 유기적 진화)에 직접 영향을 준다. 단순 프롬프트 강화로는 (2)/(4)가 해결 안 됨 — 구조적 변화 필요.

## 검토한 옵션

### 옵션 A: 위키 전체를 LLM에 던지기
- 정확도 높음
- 토큰 비용 폭발, 컨텍스트 한계 빠르게 도달
- → **기각**

### 옵션 B: 단일 호출 유지 + 프롬프트만 강화
- 변경 최소
- (1) 기존 위키 미인지 미해결, (4) 의도 표현 미해결
- → **기각**

### 옵션 C: 다단계 + 위키 목차(`index.md`)만 컨텍스트로
- 위키 전체 X, 목차(타이틀+한줄요약)만 LLM에게 제공
- 목차 크기는 페이지 수와 선형, 수백 페이지까지 컨텍스트 안에 무리 없음
- LLM이 기존 구조 알고 분해 plan을 짠다
- → **채택**

## 결정

**옵션 C 채택. v0.3a에서 Phase A부터 점진 도입.**

### Phase A (v0.3a, 본 ADR 범위)

1. **`index.md`를 Ingest LLM 컨텍스트에 추가** — 시스템 프롬프트 또는 user content에 포함
2. **LLM 출력을 JSON plan 구조로 변경** — `=== FILE: ===` 마커 폐기
3. **Plan 스키마**:
    ```json
    [
      {"action": "create",
       "page_path": "<kebab-case>.md",
       "content": "<full markdown including frontmatter>"},
      {"action": "merge_into",
       "target": "<existing-page>.md",
       "merged_content": "<full new content>"},
      {"action": "supersede",
       "target": "<existing-page>.md",
       "reason": "...",
       "new_content": "<full new content>"}
    ]
    ```
4. **Phase A 처리 범위**:
    - `create` 액션: 즉시 실행 (페이지 작성)
    - `merge_into`, `supersede`: **Phase A에서는 실행하지 않고 `log.md`에 표시만** (사용자 인지 + Phase B 도입 전 검증용)
5. **Pydantic 검증** — Plan JSON 파싱 실패 시 명시적 FAILED + 원본 출력 일부 `error_msg`에 보존
6. **Graceful degrade (다단계)**:
    - 1순위: JSON parse + Pydantic 검증
    - 2순위: 기존 4단계 fallback (`=== FILE: ===` 마커 / `---\nyaml\n---` / dangling `---` / `# 제목`만) — `log.md`/Jobs 탭에 "JSON parse failed, used legacy fallback" 명시
    - 3순위: 모두 실패 → Job FAILED + LLM 출력 snippet 보존
7. **모델 1 채택 (단일 호출에 풀 콘텐츠 포함)**
8. **JSON 강제 메커니즘**: ⚠️ 운영 변경 흐름 (2026-05-08 실측 결과로 정착):
    - **(원안)** LiteLLM `response_format={"type":"json_object"}` 통일 — *(Kirin 결정 2026-05-08)*
    - **(1차 변경)** Anthropic 정상, Ollama provider에서 `KeyError: 'arguments'` 발생 → Ollama만 `format="json"` 분기 시도
    - **(2차 변경, 현 운영)** Ollama provider가 `format="json"`에서도 동일한 응답 파싱 경로(line 566)에서 KeyError → **Ollama는 JSON 강제 자체 미사용**으로 결정. prompt 강제 + Pydantic + graceful degrade에 의존.
    - Anthropic/OpenAI 등: `response_format={"type":"json_object"}` 그대로 사용 (현 운영)
    - 이 우회는 **임시 조치**. LiteLLM 업그레이드 또는 Ollama API 직접 호출로 우회 제거 필요 — [백로그 B-21](../backlog.md#b-21-ollama-llm-호환성-우회-제거-경로) 참조.
9. **`create` 충돌 정책**: `create` 액션의 `page_path`가 기존 페이지명과 충돌 시 **자동으로 `merge_into` 액션으로 강등**. Phase A에서는 어차피 skip + log 명시되므로 데이터 손실 없음. — *(Kirin 결정 2026-05-08)*
10. **Plan 안전장치**:
    - Path traversal 거부 (page_path는 kebab-case 슬러그만 허용)
    - `merge_into`/`supersede`의 `target` 존재 검증 (없으면 skip + log "target invalid")
    - `sources` frontmatter는 시스템이 `[원본 파일명]`을 강제 주입 (LLM 신뢰 X, SC-14 보장)
    - Plan frontmatter `type` enum 위반 시 `reference` 기본값으로 보정 — *(Kirin 결정 2026-05-08)*
11. **가시성**: Plan 전체 JSON을 `Job.payload`에 보존 + Jobs 탭에서 plan 표시 (어떤 액션이 실행/skip 됐는지 사용자가 인지)

### Phase B (v0.3b 또는 v0.4)

- `merge_into`, `supersede` 실제 실행 (기존 페이지 read → 통합 → 덮어쓰기)
- Wiki Command UI (다중 페이지 작업 진입점) — 별도 ADR

### Phase C (조건부)

- Step 2 정제(원본 노이즈 제거) — PDF 추출 품질이 진짜 블로커로 드러나면 추가

## 근거

- **목차 크기 제어**: `index.md`는 페이지명 + 한줄 요약 + 최종 수정. 수백 페이지 규모까지 LLM 컨텍스트에 안전
- **Plan output의 가치**: 액션 라벨이 있어 사후 의도 추적 가능 (log.md, jobs 테이블에 plan 저장 시)
- **점진 도입**: Phase A만으로도 중복 페이지 감소 효과 확인 가능. 검증 후 Phase B로 진화
- **Graceful degrade**: 실패해도 v0.2 동작과 동등하거나 명시 실패. 회귀 위험 낮음
- **현재 코드와의 호환**: `parse_llm_pages`의 4단계 fallback은 그대로 유지하되, JSON parse를 가장 우선 case로 추가

## 트레이드오프

- **출력 토큰 ↑** — 단일 호출에 풀 콘텐츠 포함이라 JSON wrapping/escape 비용 발생. 중소 문서엔 영향 작음
- **JSON 출력 신뢰성** — weak 모델(Ollama gemma)에서 깨질 가능성 → graceful degrade로 완화
- **`merge_into`/`supersede`가 Phase A에서 무시됨** — 사용자가 "merge가 잡혔는데 왜 적용 안 됐나" 헷갈릴 가능성 → log.md에 명시 + UI 표시 (Jobs 탭) 검토 필요

## 영향 / 후속 작업

- **`prompts/ingest.txt` 재작성** — JSON 스키마 강제, 예시 1~2개 (단일 페이지 + 다중 페이지 + merge_into 케이스 포함)
- **`pipeline.py: run_ingest`** — `index.md` 로드 + 컨텍스트 주입, 출력 JSON parse, Pydantic 검증, action 디스패치
- **`wiki_store.py`** — `parse_llm_pages` 앞단에 JSON case 추가
- **`log.md`** — Phase A에서 `merge_into`/`supersede` 발생 시 "skipped (phase A)" 명시
- **신규 SC** — Hawkeye 작성 예정 (예: 동일 주제 두 번 ingest 시 plan에 `merge_into`로 잡혀야 하고, 실제 두 번째 페이지 생성은 안 됨)
- **신규 ADR-0012** (필요 시) — Plan 모델 1 vs 2 vs 3 트레이드오프 별도 ADR화
