# ADR-0004 — LLM 추상화: LiteLLM (v0.1부터)

- **상태**: Accepted
- **결정일**: 2026-05-05
- **결정자**: Kirin, Roy, Breda

## 컨텍스트

LLM 공급자(Anthropic, OpenAI, Ollama 등) 교체 가능성을 처음부터 열어둘지, MVP는 단일 공급자로 시작할지.

## 결정

**v0.1부터 LiteLLM 추상화 도입.** Anthropic API + Ollama 지원.

## 근거

- 오픈소스 self-hosted 도구로서 사용자가 Ollama로 운영하고 싶어할 가능성 높음 (비용·프라이버시)
- LiteLLM은 호출부 인터페이스를 통일해 추상화 비용 거의 0
- 나중에 추상화 추가는 비용이 더 큼 (모든 호출부 리팩터)

## 트레이드오프

- 모델별 instruction following 차이는 LiteLLM이 해결 못함 → 별도 호환성 처리 필요 (예: Ollama 계열은 system 메시지를 user 앞에 prepend)
