# ADR-0006 — 백그라운드 워커: FastAPI BackgroundTasks

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Roy, Breda (검토)

## 컨텍스트

Job Queue의 워커 구현 방식. 후보:
- FastAPI 내 `asyncio` 백그라운드 태스크 (`BackgroundTasks` / `asyncio.create_task`)
- Celery + Redis (외부 큐 시스템)

## 결정

**FastAPI `BackgroundTasks` (asyncio 기반) 채택.**

## 근거

- MVP 워커 1개 (순차 FIFO) 요건 충분히 커버
- 외부 의존성(Celery/Redis) 없이 단일 프로세스로 처리
- Docker Compose 단순화 (컨테이너 1개 줄임)
- Self-hosted 사용자의 운영 부담 감소

## 트레이드오프

- 워커 N개 병렬 처리는 v0.2+에서 검토 — 그때 Celery 등으로 마이그레이션 가능
- backend 프로세스 죽으면 진행 중 Job 손실 → MVP에서는 수동 재시도 허용
