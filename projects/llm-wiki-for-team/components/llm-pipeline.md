# LLM Pipeline

원본을 위키 페이지로 변환하고, 사용자 수정 요청을 위키에 반영하는 핵심 처리 흐름.

> LLM 추상화 결정: [ADR-0004 — LiteLLM](../decisions/0004-litellm-abstraction.md)
> Edit 단일 페이지 결정: [ADR-0009](../decisions/0009-edit-single-page.md)

## Ingest flow

원본 파일 → 위키 페이지(들).

1. 원본 파싱 (파일 → 텍스트)
2. LLM에 위키 작성/업데이트 요청 (축적 방식, 삭제 없음)
3. 결과를 Obsidian MD 형식으로 Wiki Store에 커밋
4. `index.md` 갱신
5. `log.md` 항목 append

## Edit flow

사용자 수정 요청 → 단일 페이지 수정.

1. 사용자가 특정 위키 페이지 조회 화면 하단 입력창에 수정 요청 텍스트 입력 (단일 페이지 컨텍스트)
2. 해당 페이지를 대상으로 LLM에 수정 요청 전달 (페이지 선택 단계 생략)
3. LLM이 해당 페이지를 수정 후 커밋
4. `index.md` 갱신
5. `log.md` 항목 append — `job_id` + 수정된 페이지만 기록 (요청 전문은 jobs 테이블에만 보관)

## 출력 견고화

LLM은 system prompt를 무시하고 평론형 응답을 줄 수 있다 (특히 weak instruction-following 모델). `parse_llm_pages`는 다음 4단계 fallback으로 흡수한다:

1. `=== FILE: <name>.md ===` 마커 (다중 페이지)
2. `---\n<yaml>\n---\n<body>` 정상 frontmatter (단일 페이지)
3. `---\n# 제목` (gemma 실측 패턴) → `---` 제거 + 자동 frontmatter
4. `---` 없이 `# 제목` 시작 → 자동 frontmatter 부여

평론형(첫 줄 "This is..." 류)만 명시적 FAILED.

## LLM 설정

### 연결 설정 (`.env`)

```env
LITELLM_PROVIDER=anthropic       # anthropic | ollama | openai 등
LITELLM_MODEL=claude-opus-4-7    # 사용할 모델명
LITELLM_API_KEY=sk-ant-...       # API 키 (Ollama는 생략)
LITELLM_BASE_URL=                # Ollama 등 로컬 엔드포인트 (선택)
```

- MVP: `.env` 파일로만 관리. Admin UI 설정 패널은 v0.2+
- 서비스 기동 시 설정값 로드. 변경 시 재기동 필요

### 프롬프트 관리

- 위치: `prompts/ingest.txt`, `prompts/edit.txt` (repo에 기본 템플릿 포함)
- MVP: 파일 직접 수정. UI 편집기는 v0.2+
- 기동 시 파일 로드, 변경 시 재기동

### Ollama 호환성

`provider=ollama` 계열일 때 `llm_client`는 system + user 메시지를 단일 user 메시지로 병합하고 끝에 'FINAL REMINDER' 블록을 추가한다. gemma 등 weak instruction following 모델 대응.

### 프롬프트 필수 제약

**Ingest 프롬프트:**
- 출력 형식: Obsidian MD (`[[페이지명]]` 내부 링크, YAML frontmatter)
- frontmatter 필수 필드: `type`, `created`, `last_updated`, `tags`, `sources`
- 축적 원칙: 기존 위키 내용을 삭제하지 말고 새 내용으로 보강
- 출력 단위: 페이지별 파일 단위로 응답

**Edit 프롬프트:**
- 대상 페이지 1개만 수정
- frontmatter 구조 유지 (`created` 변경 금지, `last_updated` 갱신)
- `[[링크]]` 형식 준수
- 지시받지 않은 내용 삭제 금지

## 관련 SC

SC-11-b, SC-12, SC-13, SC-15, SC-16, SC-16-b, SC-16-c, SC-17, SC-30
