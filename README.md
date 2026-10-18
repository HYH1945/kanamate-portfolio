**Live** → https://HYH1945.github.io/kanamate-portfolio/

---

# KanaMate · 멀티에이전트 일정 조율

개인 일정 관리와 팀 회의 시간 조율을 하는 LangChain 멀티에이전트

|  |  |
| --- | --- |
| **기간** | 2026.06 ~ 08 · 카카오테크 캠퍼스 6주 과정 |
| **역할** | 과정에서 제공한 런타임 · MCP 서버 · UI 위에서 에이전트 로직 구현 |
| **스택** | LangChain · Pydantic · SQLite · ChromaDB · MCP |
| **모델** | gpt-4.1-mini |

## 00 실행 예시

> 철수, 하린이랑 7월 7일부터 17일 사이에 1시간짜리 회의 시간 잡아줘

| 7/7 | 09 | 10 | 11 | 12 | 13 |
| --- | --- | --- | --- | --- | --- |
| 철수 | 회의 | API 실습 | | **확정** | |
| 하린 | 회의 | | | **확정** | |
| 나 | 회의 | | 회의 | **확정** | |

**Kana** · 철수, 하린 님과 2026년 7월 7일부터 7월 17일 사이 1시간 회의는 2026년 7월 7일 12시부터 13시까지로 확정했습니다. 이 시간에 모두 일정이 비어 있습니다.

2026-10-18 재실행, 실습용 고정 데이터 기준 · [단계별 trace 보기](https://HYH1945.github.io/kanamate-portfolio/trace.html)

## 01 6주간의 구현

매주 이전 코드 위에 기능을 쌓는 방식

<details><summary><b>W1 · Tool calling</b></summary>

- 개인 일정 추가 · 조회 · 삭제 tool 3개 구현
- LLM은 어떤 tool을 어떤 인자로 부를지만 고르고, 실행은 Python이 담당
- 대화마다 session_id로 일정 범위를 분리
</details>
<details><summary><b>W2 · Structured output</b></summary>

- 자연어 요청을 Pydantic 스키마로 변환 (개인 일정 · 그룹 일정 · 할 일 · 알림 · 미분류)
- 종류 필드의 기본값을 없애 LLM이 반드시 분류하도록 수정 (리뷰 반영)
- 형식이 맞지 않는 LLM 응답은 그냥 통과시키지 않고 에러로 차단
</details>
<details><summary><b>W3 · SQLite 영속화</b></summary>

- tool 입력 스키마 그대로 SQLite에 저장 · 조회 · 수정 · 삭제
- LLM이 만든 원본과 일정 · 할 일 · 알림 테이블을 함께 저장해 나중에 추적 가능
- 조건 없는 전체 삭제 차단, 실패 응답도 성공과 같은 형식 유지 (리뷰 반영)
</details>
<details><summary><b>W4 · 출처별 RAG</b></summary>

- 메모(ChromaDB), 저장 기록(SQLite), 지난 대화(ChromaDB에 lazy sync) 검색 tool 분리
- 관련 · 무관 질문의 거리를 실측해 1.2 기준으로 무관한 결과 필터링
- 어떤 질문에 어떤 tool을 쓰는지 예시 중심으로 prompt를 다시 써 라우팅 버그 수정
</details>
<details><summary><b>W5 · MCP 연동</b></summary>

- 팀원 대화 · 공유 일정 MCP tool을 LangChain tool로 감싸기
- 내 일정과 팀원 일정을 병합할 때 같은 날짜 필터 적용, 내 일정 중복 제거
- 외부 서버가 실패해도 내 일정은 유지하고 실패 여부만 함께 반환
</details>
<details><summary><b>W6 · Multi-agent</b></summary>

- Supervisor + Nana / Kana로 분리, 하위 agent는 한 번만 만들어 재사용
- 후보 선택과 확정은 Agent, 겹침 검증과 기록은 Tool이 맡는 구조
- 하위 agent trace에서 최종 결정 payload를 꺼내 Supervisor 결과에 포함
</details>

## 02 구조

Week 5에서 한 agent가 **tool 21개**를 모두 판단하게 되어, Week 6에서 역할별로 분리

**Supervisor** · 요청을 보고 Nana 또는 Kana에게 넘기기만 함, 직접 일하는 tool 없음

| Nana · 개인 비서 (tool 14개) | Kana · 그룹 조율 (tool 8개) |
| --- | --- |
| 내 일정 추가 · 조회 · 수정 · 삭제 | 팀원과 나눈 지난 대화에서 일정 찾기 |
| 자연어 요청을 정리해 일정 · 할 일 · 알림으로 저장 | 나와 팀원의 바쁜 시간 모으기 |
| 메모, 저장한 기록, 지난 대화를 출처별로 검색 | 모두 비는 시간을 골라 최종 확정 (저장 권한 없음) |
| SQLite · ChromaDB | MCP Server |

**나눈 기준은 데이터와 권한** · Nana는 내 데이터 읽기·쓰기, Kana는 팀원 데이터 읽기만

## 03 핵심 설계 · 고르는 건 Agent, 검증은 Tool

`수집(Tool) → 후보 선택(Agent) → 검증(Tool) → 확정(Agent) → 기록(Tool)`

Python만 쓰면 선택 이유가, LLM만 쓰면 검증이 빠져서 둘을 나누고 같은 규칙을 tool 설명과 prompt에 명시

## 04 문제 해결

| 증상 | 해결 |
| --- | --- |
| 취향을 말해도 저장이 안 되고, 대화 검색과 저장 기록 검색이 뒤섞임 (W4) | 어떤 질문에 어떤 tool을 쓰는지 예시 중심으로 prompt 재작성 |
| 철수와의 대화 검색 요청이 Nana로 위임됨 (W6) | 다른 사람이 언급된 대화·일정 요청은 Kana로 보내는 기준을 Supervisor prompt에 추가 |
| 공통 시간이 없으면 기록 없이 대화가 끝남 (W6) | 후보가 없어도 결정 tool을 호출해 보류 상태와 이유를 남기도록 변경 |

tool 설명과 prompt에 적히지 않은 규칙은 agent가 지키지 않는다는 것, 버그는 대부분 예외 경로에서 나온다는 것을 배움

## 05 한계와 다음 단계

- **한계** · 확정 tool이 agent가 넘긴 시간을 후보와 다시 대조하지 않아, 실행 기록 중 겹치는 시간이 확정된 사례 1건. 확정 후 Supervisor가 Nana에 불필요하게 재위임하는 경우도 있음
- **다음 단계** · 확정 시간을 후보 목록과 대조, 확정 후 위임 종료 규칙 추가, tool 호출 순서 회귀 테스트
