# 팀 워크플로우

팀의 Spec-driven 개발 프로세스. 모든 프로젝트에 적용된다.

## SPEC.md 구조

각 프로젝트에는 다음 섹션으로 구성된 `SPEC.md`가 있다:

```
## Requirements
Kirin의 요건 원문.

## Technical Design
Roy의 기술 명세: 아키텍처, API 계약, 기술 스택, 데이터 모델.

## Success Criteria
Hawkeye가 Requirements에서 직접 도출한 검증 가능한 체크리스트.
각 기준은 "Given X, when Y, then Z" 형식의 테스트 시나리오를 포함한다.
이터레이션을 거쳐 누적된다 — 삭제하지 않고, 추가하거나 완료 표시만 한다.

## Open Issues
- [ ] [Blocking] 설명 — 이터레이션 N
- [x] [Advisory] 설명 — 이터레이션 N+1에서 해결

## Changelog
- v0.1: 초기 spec
- v0.2: 한 줄 요약 (상세 내용은 git commit)
```

이력은 git으로 관리한다(`git log SPEC.md`). 별도 이력 파일 없음.

## 이터레이션 사이클

```
Kirin → 요건 전달
  ↓
Roy → SPEC.md 생성/업데이트 (Requirements + Technical Design)
     → 시작 전 Open Issues 검토 및 처리 (수락 / 보류 / 거절)
  ↓
Hawkeye → SPEC.md에 Success Criteria 추가
  ↓
Roy → Breda에게 작업 할당 (SPEC.md 참조)
  ↓
Breda → 구현 + 테스트 작성 (프론트엔드: Playwright, 백엔드: API 테스트 도구)
       → 모든 테스트 실행, 결과를 Roy에게 보내는 완료 보고서에 포함
  ↓
Hawkeye → SPEC.md의 모든 Success Criteria 평가 (신규 + regression)
         → [Blocking] 이슈: SPEC.md Open Issues에 추가, 채널에 게시 — Kirin 결정
         → [Advisory] 이슈: SPEC.md Open Issues에 추가, 채널에 게시 — Roy 결정
  ↓
Roy → 통과 시: Kirin에게 완료 보고
      블로킹 시: Kirin의 최종 결정으로 해결 조율
```

## SPEC.md 규모 관리

- SPEC.md는 현재 상태에 집중한다. Changelog는 간결하게 유지한다 (이터레이션당 한 줄).
- 대형 프로젝트는 모듈별로 분리한다: `SPEC-api.md`, `SPEC-frontend.md` 등.
- 프로젝트 특화 규칙은 SPEC.md 안에 넣거나, SPEC.md에서 참조하는 별도 파일로 관리한다.
