# Orchestrator Analyze 프롬프트

> `D:/project/orchestrator/skills/analyze.js` 에서 실제 사용 중.
> 세션 종료 시 JSONL을 분석해 지식 파이프라인 처리 방향을 결정한다.

---

## 시스템 프롬프트

```
당신은 지식 파이프라인의 분석 엔진입니다.
입력 내용의 의도·중요도·처리방향을 JSON으로만 반환하세요.
```

---

## 유저 프롬프트 (템플릿)

```
다음 내용을 분석하세요:

## 입력
{content — 최대 1000자}

JSON 형식으로만 응답:
{
  "intent": "question|statement|command|reference|conversation",
  "importance": "critical|high|medium|low",
  "action": "store|respond|ignore|escalate",
  "tags": ["태그1", "태그2"],
  "summary": "핵심 요약 (100자 이내)",
  "language": "ko|en",
  "reasoning": "판단 근거"
}
```

---

## 분류 기준

| intent | 설명 |
|--------|------|
| `question` | 질문·참조 요청 |
| `statement` | 일반 진술·설명 |
| `command` | 실행·구현 명령 |
| `reference` | 외부 자료 참조 |
| `conversation` | 일상 대화 |

| action | Vault 저장 위치 |
|--------|----------------|
| `store` | `LLM-Wiki/raw/success/` |
| `respond` | `LLM-Wiki/raw/learning/` |
| `ignore` | `LLM-Wiki/raw/failure/` |
| `escalate` | `LLM-Wiki/raw/failure/` + 알림 |

---

## 설계 포인트

- **JSON only** 강제 → 파싱 실패율 최소화
- 중요도 텍스트를 숫자로 정규화: `critical=1.0 / high=0.8 / medium=0.5 / low=0.2`
- 1000자 입력 제한 → Haiku 토큰 비용 절감
- 결과로 structure_skill → memory_skill 체인 트리거
