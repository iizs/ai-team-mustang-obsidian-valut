# Prompt 로드 버그 + LiteLLM Ollama 호환성 Postmortem

- **일자**: 2026-05-08
- **주체**: Roy, Breda, Hawkeye, Kirin (실측)
- **대상**: v0.3 (Phase A) 실사용 단계에서 드러난 일련의 핫픽스
- **상태**: 모든 핫픽스 적용 완료, 회고용 기록

> 본 회고는 v0.2 회고 메모에 짚어둔 "LLM이 instruction을 잘 안 따른다"는 평가를 **재해석**하게 만든 사건이다. 진짜 원인은 모델이 아니라 **prompt가 LLM에 도달하지 않았던 것**이었다.

## 사건 경위

### 1. 정상 동작처럼 보였던 v0.1~v0.2 시기
- mock 기반 테스트는 모두 통과
- Anthropic / Ollama 모두 wall-clock으로 "동작"하는 것으로 보고됨
- v0.2 회고에서 "Ollama gemma는 instruction following이 약함" 평가 — 후속 핫픽스로 prompt 강화, system→user merge, FINAL REMINDER 블록 도입

### 2. v0.3 Phase A 완료 후 Kirin 실사용
- Anthropic (Sonnet): `system: text content blocks must be non-empty` 에러
- Ollama (gemma): `litellm.APIConnectionError: 'arguments'` (`ollama.py:566`)

### 3. 원인 발견 (Kirin 가설 → Roy/Breda 확인)
- Kirin: "그동안 Gemma가 말을 안 들은 게 아니었을 수도 있겠네?"
- 코드 추적 결과:
  - `pipeline._load_prompt("ingest")`가 `Path(settings.prompts_path) / "ingest.txt"`을 단순 처리
  - `prompts_path = "./prompts"` (default, relative)
  - uvicorn이 `backend/`에서 실행 → `backend/prompts/` 못 찾음 → 빈 문자열 반환
  - 빈 system prompt → Anthropic API 거부, Ollama provider는 응답 파싱 단계 별도 버그

### 4. 핫픽스 시리즈 (commits)

| Commit | 내용 |
|--------|------|
| `7ec4f9f` | prompts path 해상도 (PROJECT_ROOT → BACKEND_ROOT → CWD) + silent empty 폐기 (FileNotFoundError / ValueError raise) |
| `7a94a52` | Ollama JSON 강제 분기 — `response_format` → `format="json"` (1차 우회) |
| `86ea69e` | Ollama JSON 강제 완전 포기 — `format`/`response_format` 둘 다 미사용 (LiteLLM Ollama provider 응답 파싱 경로 자체가 버그) |

### 5. Kirin 재측정 결과
- Anthropic Sonnet: 두 페이지 생성 + `[[wikilink]]` 자동 연결 확인
- Ollama Gemma: 단일 위키 문서 생성 확인 (링크 생성은 제한적)

## 영향 분석

이 사건은 다음 평가들을 **무효화**했다:

1. **v0.2 "weak 모델 instruction following 약함" 결론** — 빈 prompt 환경에서 측정한 거라 진짜 모델 능력은 미측정 상태였음
2. **v0.2 핫픽스 일부의 정당성**:
   - `system → user merge` 분기: gemma 호환을 위해 도입했지만, 빈 prompt + FINAL REMINDER만 보내고 있던 상태가 원인. 진짜 system prompt가 도달했다면 표준 분리로도 동작했을 가능성
   - `prompts/ingest.txt` 강화 (마커 강제, 금지 문구, 예시): 모두 LLM에 도달 안 함. v0.3 도달 후에야 비로소 효과 측정 가능
3. **v0.3 "merge 행동 결정성 부족"** 같은 평가도 모두 무효 — 강화된 JSON Plan prompt가 처음으로 도달했기 때문

## 교훈

### 아키텍처 / 설계
- **silent empty/default return은 디자인 안티패턴.** v0.3 핫픽스에서 `_load_prompt`의 silent empty 폐기로 명시 에러로 전환. 다른 곳도 점검 필요.
- **외부 라이브러리 (LiteLLM) 추상화 신뢰는 provider/모델별 실측 검증 후.** Anthropic은 동작해도 Ollama provider가 다르게 동작할 수 있음.
- **임시 우회는 흔적 남기기.** ADR-0011에 운영 변경 흐름 명시 + 백로그 B-21로 제거 경로 추적.

### 검증 / 관찰성
- **mock 기반 단위 테스트의 한계 재확인.** Hawkeye가 v0.2 후 자체 메모에 "SC 정적 검증의 한계 — 엔드투엔드 통합 흐름은 별도"라고 적었던 것과 일맥상통.
- **실사용 환경(uvicorn 디렉토리, .env 경로 등)이 mock과 다를 때 발생하는 path/config 이슈는 통합 테스트로만 잡힘.**
- **빈 system prompt → API 에러**라는 명백한 신호가 있었지만, Ollama 분기에서는 API 에러 없이 평론형 응답으로 "동작하는 것처럼" 보였음. 이게 진단을 늦췄음.

### 운영
- **Kirin의 한 줄 가설("말 안 들은 게 아닐 수도")이 디버깅의 turning point.** 사용자 직관이 코드 추적 방향을 제시.
- **모델 평가는 환경 설정 검증 이후로 미루는 게 안전.** 환경이 깨진 상태의 측정은 모델 평가가 아니라 환경 평가.

## 백로그 영향

- **B-5 (프롬프트 다듬기)** 우선순위 재평가 — 강화된 prompt가 처음으로 정상 도달했으니, 추가 작업 가치는 실측 후 결정. 일부는 이미 충분할 수 있음.
- **B-21 (Ollama 우회 제거)** 신규 추가 — LiteLLM 업그레이드 / Ollama API 직접 / OpenAI-compatible 호스팅 중 한 경로로 우회 제거.
- **SC-46 (중복 ingest merge 등장) Advisory** — 이전 실측 데이터 무효. v0.3.1 Phase B 도입 후 다시 측정.

## 후속 작업

- ADR-0011 운영 메모 보강 ✅ (운영 변경 흐름 명시, 우회 임시 조치 명시)
- B-21 백로그 추가 (예정)
- SC-46 첫 의미 있는 실측은 v0.3.1 Phase B 후 진행
