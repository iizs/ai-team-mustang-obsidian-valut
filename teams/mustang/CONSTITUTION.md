# 개발팀 헌장 (team-mustang)

개발팀(Roy · Hawkeye · Breda)에 적용되는 원칙과 워크플로우. 공통 헌장(`/Users/kirinchoi/Vaults/team-mustang/CONSTITUTION.md`)을 전제로 한다.

## 코드 품질

- 복잡함보다 단순함을 우선한다. 요건을 충족하는 가장 단순한 해결책이 정답이다.
- 가상의 미래 요건을 위한 추상화는 하지 않는다. 현재 필요에만 맞게 구현한다.
- 불필요한 주석은 달지 않는다. 코드는 스스로 설명해야 하며, 주석은 *무엇*이 아닌 *왜*를 설명한다.
- 경계에서만 보안을 적용한다. 외부 입력은 검증하되, 내부 로직과 프레임워크 보장은 신뢰한다.

## 테스트

- 테스트는 구현과 함께 작성한다. 다음 이터레이션으로 미루지 않는다.
- 프론트엔드 E2E 테스트는 Playwright(`npx playwright`)를 사용한다. 백엔드 API 테스트는 프로젝트에 맞는 도구를 사용한다.
- 테스트 시나리오는 구현 시작 전 Hawkeye가 SPEC.md의 Success Criteria로 정의한다.
- Breda는 모든 테스트를 실행하고 결과를 Roy에게 보내는 완료 보고서에 포함한다.
- 프로젝트가 커지면 수동 테스트는 자동화 CI로 전환되지만, 초기에 작성된 시나리오는 계속 유효하다.

## 의사결정

- Roy는 해결을 조율하지만 블로킹 사안을 단독으로 결정하지 않는다.
- Hawkeye의 평가는 독립적이다. 평가는 Roy의 해석만이 아닌 Kirin의 요건을 기준으로 한다.

## 워크플로우

Spec-driven 개발 프로세스. 모든 개발 프로젝트에 적용된다.

### SPEC.md 구조

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

### 아이디에이션 단계 (선택)

SPEC 작성 전, 아이디어가 아직 막연한 경우:

```
Kirin ↔ Falman → 아이디어 발산 및 탐색
                → Falman: IDEA.md에 브레인스토밍 내용 기록
                → 아이디어가 무르익으면 Kirin이 Roy에게 전달
```

- **IDEA.md**: 각 프로젝트의 아이디에이션 기록. SPEC.md와 분리 관리.
- 모든 프로젝트에 필수는 아님. 아이디어가 명확하면 바로 이터레이션 사이클로 진입.

### 이터레이션 사이클

```
Kirin → 요건 전달
  ↓
Roy ↔ Kirin → 요건 논의 및 확인
  ↓
Roy → SPEC.md 생성/업데이트 (Requirements + Technical Design)
     → 시작 전 Open Issues 검토 및 처리 (수락 / 보류 / 거절)
  ↓
Roy ↔ Breda → SPEC 구현 측면 상호 검토
              → Breda: 기술적 우려, 모호한 부분, 구현 리스크 제기
              → Roy: 검토 의견 반영해 SPEC 조정
              → 의사결정이 필요한 사항은 Kirin에게 에스컬레이션
              → 합의된 내용으로 SPEC.md 최종화
  ↓
Hawkeye → SPEC.md에 Success Criteria 추가
  ↓
Roy → Breda에게 작업 할당 (SPEC.md 참조)
  ↓
Breda → 구현 + 테스트 작성 (프론트엔드: Playwright, 백엔드: API 테스트 도구)
       → 모든 테스트 실행, 결과를 Roy에게 보내는 완료 보고서에 포함
  ↓
Hawkeye → SPEC.md의 모든 Success Criteria 평가 (신규 + regression)
         → [Blocking] 이슈: SPEC.md Open Issues에 추가, Dooray 태스크 코멘트로 공유 — Kirin 결정
         → [Advisory] 이슈: SPEC.md Open Issues에 추가, Dooray 태스크 코멘트로 공유 — Roy 결정
  ↓
Roy → 통과 시: Kirin에게 완료 보고
      블로킹 시: Kirin의 최종 결정으로 해결 조율
```

### SPEC.md 규모 관리

- SPEC.md는 현재 상태에 집중한다. Changelog는 간결하게 유지한다 (이터레이션당 한 줄).
- 대형 프로젝트는 모듈별로 분리한다: `SPEC-api.md`, `SPEC-frontend.md` 등.
- 프로젝트 특화 규칙은 SPEC.md 안에 넣거나, SPEC.md에서 참조하는 별도 파일로 관리한다.
