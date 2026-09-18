**Live** → https://HYH1945.github.io/kanamate-portfolio/

---

# KanaMate — LangChain 기반 멀티에이전트 일정관리 시스템

> "다음 주에 A/B/C랑 회의 잡아줘" 한 문장을, 검증 가능한 tool call과 DB row로 끝까지 추적되는 일정 결정으로 바꾸는 Agentic AI 시스템

|  |  |
| --- | --- |
| **기간** | 2026.05 ~ 2026.06 (7주, 28 commits) |
| **역할** | 단독 개발 — 에이전트 아키텍처 설계 · 구현 · 검증 체계 구축 |
| **기술 스택** | Python 3.10 · LangChain 1.2 / LangGraph · Pydantic v2 · SQLite · ChromaDB · MCP · uv |
| **규모** | 12개 tool · 2개 sub-agent · 4개 정규화 table · 6개 실행 노트북 |

---

## 1. 문제 정의

일반 챗봇은 "다음 주 화요일 3시 어떠세요?"라는 **그럴듯한 문장**을 만든다. 그런데 그 문장은

- 내 기존 일정과 충돌하는지 확인한 적이 없고,
- 팀원들이 실제로 그 시간에 가능한지에 대한 근거가 없고,
- 앱이 읽어서 저장할 수 있는 데이터도 아니다.

그래서 **"모델이 무슨 말을 했는가"가 아니라 "어떤 tool을 어떤 인자로 호출했고 어떤 row가 남았는가"를 1급 관심사로 두는** 일정관리 에이전트를 설계했다.

## 2. 아키텍처

```text
   "팀원 A/B/C와              ┌──────────────────┐
    다음 주 회의   ─────────▶ │    Supervisor    │  delegate tool만 호출
    시간을 잡아줘"            └───┬──────────┬───┘
                    nana_agent │          │ kana_agent
                      ┌────────▼───┐  ┌───▼────────┐
                      │    Nana    │  │    Kana    │
                      │  개인 일정  │  │  그룹 일정  │
                      └──────┬─────┘  └─────┬──────┘
  personal_create/list/delete_schedule │    │ search_previous_conversations
  save_structured_request              │    │ load_conversation_messages
  search_rag_memory                    │    │ extract_schedules_from_history
  search_sqlite_requests               │    │ decide_final_slot
                      ┌────────────────▼┐  ┌▼──────────────────┐
                      │ SQLite · ChromaDB│  │    MCP Server     │
                      └──────────────────┘  └───────────────────┘
```

설계 원칙은 하나다. **Supervisor는 업무 tool을 직접 호출하지 않는다.** delegate tool(`nana_agent`, `kana_agent`)만 호출하고 각 sub-agent의 내부 trace를 근거로 최종 답을 만든다. 책임이 섞이면 "누가 무엇을 잘못 판단했는지" 추적이 불가능해지기 때문이다.

## 3. 핵심 구현

| 레이어 | 구현 | 결과물 |
| --- | --- | --- |
| Tool calling | 개인 일정 생성/조회/삭제를 tool로 분리, 모델은 인자만 생성 | `personal_*_schedule` trace |
| Structured output | Pydantic으로 요청을 5종(`personal_schedule`, `group_schedule`, `todo`, `reminder`, `unknown`)으로 분류·검증 | 검증된 payload |
| 영속화 | tool 입력 스키마(`BaseModel`)를 그대로 SQLite 정규화 저장 | 4개 table + `request_id` 연결 |
| Agentic RAG | ChromaDB(자유 메모) / SQLite(구조화 row) 검색 tool을 분리하고 모델이 선택 | `hits` / `rows` 근거 |
| MCP | 과거 대화 DB 접근을 MCP tool 경계 뒤로 이동 | 에이전트–데이터 소스 분리 |
| Multi-agent | Supervisor + Nana/Kana sub-agent 위임 구조 | `final_slot` + `reason` |

## 4. 기술적 의사결정

**병렬 tool 호출을 차단해 실행 순서를 보장했다.**
LangChain agent는 기본적으로 tool을 병렬 호출한다. 하지만 `search_previous_conversations → extract_schedules_from_history → decide_final_slot`은 앞 결과가 뒤의 입력이 되는 파이프라인이다. 미들웨어로 `parallel_tool_calls=False`를 주입해 순서를 강제했고, 그 결과 trace가 매번 같은 모양으로 재현된다.

```python
class DisableParallelToolCalls(AgentMiddleware):
    def wrap_model_call(self, request, handler):
        request.model_settings["parallel_tool_calls"] = False
        return handler(request)
```

**원본 payload와 정규화 row를 이중으로 저장했다.**
`save_structured_request`는 모델이 만든 tool arguments 전체를 `structured_requests.payload_json`에 원본 그대로 남기고, 동시에 `kind`에 따라 `schedules` / `todos` / `reminders`로 정규화한다. 둘은 `request_id`로 연결된다. 정규화 row만 남기면 "모델이 무엇을 어떻게 잘못 구조화했는지" 사후 분석이 불가능하다.

**검색 tool을 대상별로 쪼갰다.**
자유 서술형 메모는 ChromaDB embedding 검색(`search_rag_memory`), 구조화 저장된 일정/할 일은 SQLite row 검색(`search_sqlite_requests`)으로 나눴다. 하나의 "검색" tool로 합치지 않은 이유는, 질문의 성격에 따라 검색 대상을 고르는 것 자체가 agentic 판단이고 그 판단이 trace에 남아야 하기 때문이다.

**역할 경계를 프롬프트가 아니라 도구 구성으로 강제했다.**
Kana에게는 개인 일정 tool을 아예 주지 않아, Kana가 내 일정 충돌 여부를 임의로 판단할 수 없게 만들었다. 개인 일정 판단은 Nana의 trace에서만 나온다.

## 5. 검증

"답변이 그럴듯한가"로 동작을 판정하지 않기 위해 두 겹의 검증을 넣었다.

1. **Trace assertion** — tool 호출 순서와 저장 row count를 `assert`로 고정한다.
   `assert called_tools[:2] == ["search_previous_conversations", "extract_schedules_from_history"]`
2. **LLM-as-a-judge** — supervisor trace와 delegate payload JSON을 judge 모델에 넘겨 6개 체크리스트(위임 순서, 역할 침범 여부, 최종 답변의 근거 반영 등)로 PASS/FAIL을 받는다. 첫 줄이 `PASS`가 아니면 실행이 실패한다.

재현성은 `uv.lock`과 `langgraph-prebuilt==1.0.8` 핀으로 고정했다. LangChain 1.x의 `create_agent` import가 이 전이 의존성 버전에 민감해 실제로 환경이 깨진 적이 있어, lockfile과 별개로 직접 핀을 박았다.

## 6. 결과와 다음 단계

**결과** — 골든 케이스("팀원 A/B/C와 다음 주 회의 시간을 잡아줘")에서 supervisor가 `nana_agent → kana_agent` 순서로 위임하고, 내 일정 충돌 확인과 팀원 가능 시간 후보를 모두 반영한 `final_slot`과 선택 `reason`을 산출한다. judge 검증 PASS.

**한계** — 노트북 기반 PoC이며, 외부 멤버의 일정은 고정 데이터로 대체했다.

**다음 단계** — 실제 캘린더 API 연동, Gradio UI 연결, 후보 시간이 충돌할 때의 재협상 루프 추가.
