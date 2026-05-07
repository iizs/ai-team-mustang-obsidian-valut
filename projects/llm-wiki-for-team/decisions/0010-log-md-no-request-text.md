# ADR-0010 — log.md 기록 범위: 요청 전문 미포함

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Kirin, Roy

## 컨텍스트

`log.md`는 git에 포함되어 위키와 함께 clone/zip으로 외부에 배포된다. Edit 요청 텍스트에 사내 민감 정보(예: 사람 이름, 내부 시스템 명, 미공개 결정)가 들어갈 수 있음.

## 결정

**`log.md`에는 `job_id` + 결과(생성/수정된 페이지 목록)만 기록. 요청 전문은 jobs 테이블(DB)에만 보관.**

## 근거

- log.md는 wiki repo에 포함 → clone하면 누구나 전체 공개 가능
- 요청 텍스트는 작성자가 사내 컨텍스트로 적은 것이지 위키 본문처럼 노출용이 아님
- jobs 테이블은 role-based 접근 (Admin: 전체 / Member: 본인) → 요청 전문 조회는 권한 체크된 UI(Jobs 탭)에서만 가능

## 영향

- log.md는 "무엇이 언제 어떻게 변경됐는지"를 보여주는 변경 이력 역할
- 디버그/포렌식 목적의 상세 정보(요청 전문, 에러 stack 등)는 jobs 테이블에서 조회

## 관련

- [Wiki Store: log.md](../components/wiki-store.md#logmd)
- [Job Queue](../components/job-queue.md)
