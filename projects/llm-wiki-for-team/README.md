# Sheska — llm-wiki for team

> Karpathy의 [llm-wiki 컨셉](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)을 팀 협업 도구로 확장한 오픈소스 솔루션.
> 사람이 원본 자료를 투입하면 LLM이 위키를 생성·관리하며, 팀 에이전트가 로컬에서 연결해 사용한다.

## 문서 인덱스

### 핵심 명세
- [REQUIREMENTS](REQUIREMENTS.md) — 사용자/비기능 요건
- [ARCHITECTURE](ARCHITECTURE.md) — 시스템 큰 그림, 기술 스택, 데이터 모델
- [Success Criteria](success-criteria.md) — 검증 기준 (영구 ID)

### 컴포넌트 상세
- [Auth](components/auth.md)
- [Job Queue](components/job-queue.md)
- [Source Store](components/source-store.md)
- [LLM Pipeline](components/llm-pipeline.md)
- [Wiki Store](components/wiki-store.md)
- [Wiki UI](components/wiki-ui.md)
- [Local Sync](components/local-sync.md)

### 결정 (ADR)
- [decisions/](decisions/) — 주요 기술/설계 결정과 그 배경

### 진행
- [iterations/](iterations/) — 이터레이션별 목표·태스크·회고
  - [v0.1](iterations/v0.1.md) — MVP (완료)
  - [v0.2](iterations/v0.2.md) — 사용성·접근성 (완료)
  - [v0.3](iterations/v0.3.md) — LLM 파이프라인 다단계화 Phase A + 환경 초기화 (완료)
  - [v0.3.1](iterations/v0.3.1.md) — Phase B: Plan 액션 실행 + Wiki Command (완료)
  - v0.4 — Agentic 재설계 (Planning, Kirin 화두 제기)
- [Backlog](backlog.md) — 정렬되지 않은 아이디어 풀
- [reviews/](reviews/) — Breda/Hawkeye 리뷰 영구 보관

## 빠른 시작

구현체 코드: https://github.com/iizs/sheska

```bash
git clone https://github.com/iizs/sheska
cd sheska
cp .env.example .env  # API 키 입력
docker compose up --build
```

## 문서 작성/관리 원칙

- **변경 빈도 분리**: 안정적 문서(REQUIREMENTS)와 활발히 변하는 문서(BACKLOG)는 한 파일에 두지 않는다
- **결정의 영속성**: 주요 결정은 ADR로 시점·근거·맥락을 보존
- **ID 안정성**: SC ID는 영구. 추가는 끝에 붙이고, 번호는 재사용하지 않는다
- **리뷰 영속화**: Discord에서 휘발될 리뷰 결과는 `reviews/`에 일자·주체·범위 명기로 저장

## Archive

이전의 단일 파일 SPEC: [archive/SPEC_v0.1.md](archive/SPEC_v0.1.md) (분리 전 마지막 버전)
