# ADR-0005 — Lint: v0.2+로 보류

- **상태**: Accepted
- **결정일**: 2026-05-06
- **결정자**: Kirin, Roy, Breda

## 컨텍스트

Karpathy 컨셉의 핵심 중 하나가 **Lint** — 위키 health-check, 스테일/모순 페이지 정리. 초기 SPEC은 v0.1에 포함했음.

## 결정

**Lint는 v0.2+로 보류.** MVP에서 제외.

## 근거

- 위키 규모가 커지면 Lint 비용이 폭발 (전체 페이지 LLM 통과)
- MVP 단계에서는 Lint 동작 검증보다 "원본 → 위키" 흐름 자체의 안정화가 우선
- "구성되는 것까지 보는 게 더 중요" (Kirin)

## 영향

- 원본 교체 시 제거된 내용이 위키에 잔존할 수 있음 → Known Limitation으로 명시
- v0.2에서 Lint 도입 시 scope 좁히는 방향 검토 (전체 vs 변경분 vs 선택)
