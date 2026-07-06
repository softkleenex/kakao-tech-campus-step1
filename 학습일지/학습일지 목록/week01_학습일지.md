# week01_Agentic AI와 Nana 개인 일정 도구의 시작

## 🗓 이번 주 개요
- 주차: Week 01 (6/29~7/5)
- 진행한 것:
  - 6/29(월) 실시간 강의
  - 7/2(목) 실시간 강의
  - `kakao_clone_coding_lecture` 강의 자료 학습
  - `kakao-tech-campus-step1-clone/kyungpook-clone` 별도 과제 진행
- 키워드: #AgenticAI #ToolCalling #LangChain #Trace #Python #CRUD #ScheduleAgent #GitHubPR

## 📚 이번 주 학습한 것

### 1. 일반 챗봇과 Agentic AI의 차이
- **핵심 개념**: 일반 챗봇은 자연어 답변 자체가 중심이고, Agentic AI는 답변을 만들기 전에 필요한 도구를 고르고 실행 결과를 다시 해석한다.
- **내가 이해한 방식**:
  - 이번 주 강의에서 가장 중요하게 느낀 부분은 "답변이 자연스러워 보이는가"보다 "어떤 tool을 어떤 arguments로 호출했는가"를 확인해야 한다는 점이었다.
  - Agentic AI에서는 모델의 최종 답변만 보면 실제로 일정이 만들어졌는지 알 수 없다. `tool_call`, `tool_result`, `payload`를 함께 봐야 모델이 요청을 제대로 해석했고, 코드가 기대한 결과를 반환했는지 검증할 수 있다.
  - 그래서 이번 과정의 실습은 단순히 채팅창에 답이 나오는 것을 확인하는 것이 아니라, 상세 trace를 읽으면서 모델과 도구 사이의 연결을 확인하는 연습이라고 이해했다.

### 2. Nana 개인 일정 tool의 기본 구조
- **핵심 개념**: Week 1의 Nana는 `personal_create_schedule`, `personal_list_schedules`, `personal_delete_schedule` 세 가지 tool을 통해 현재 대화 범위의 임시 개인 일정을 생성, 조회, 삭제한다.
- **내가 이해한 방식**:
  - 일정 정보는 모델이 직접 들고 있는 것이 아니라, tool이 받은 인자를 바탕으로 Python 코드의 임시 저장소에 기록된다.
  - `personal_create_schedule`은 제목, 날짜, 시작 시간, 종료 시간, 참석자 목록을 받아 schedule dict를 만들고, 결과 JSON의 `created_schedule`로 반환한다.
  - `personal_list_schedules`는 현재 대화 범위에 속한 일정만 조회하고, 필요하면 날짜 범위로 필터링한다.
  - `personal_delete_schedule`은 `schedule_id`가 일치하는 일정을 삭제하고, 실제 삭제 여부를 `deleted` 값으로 돌려준다.
  - 여기서 중요한 점은 Week 1의 저장소가 SQLite나 외부 캘린더가 아니라 현재 대화 안에서만 쓰는 임시 메모리라는 것이다. 저장 범위가 작기 때문에 오히려 tool call과 payload 구조를 집중해서 보기 좋았다.

### 3. 별도 과제에서 구현한 개인 일정 CRUD 흐름
- **핵심 개념**: 별도 과제에서는 `student_parts/week01_wake_up_nana.py`의 구현 대상 함수만 완성하고, 앱 실행 후 상세 trace로 결과를 확인하는 흐름을 따른다.
- **내가 이해한 방식**:
  - 과제의 핵심은 거창한 앱 전체 구조를 바꾸는 것이 아니라, Week 1에서 공개된 세 tool이 기대한 JSON 문자열을 안정적으로 반환하도록 만드는 것이었다.
  - `attendees`가 없으면 빈 리스트로 바꾸고, 새 일정에는 임시 ID와 생성 시각, 현재 session scope를 함께 넣는다.
  - 조회와 삭제에서는 전체 리스트를 그대로 다루는 것이 아니라 현재 대화 범위의 일정만 대상으로 삼아야 한다. 같은 앱을 여러 세션에서 쓸 수 있다고 생각하면 `session_id`를 기준으로 범위를 나누는 이유가 이해됐다.
  - 구현 후에는 "일정을 만들었다"는 답변만 보는 것이 아니라, 상세 trace에서 실제로 `personal_create_schedule`이 호출됐는지, 결과 JSON에 `created_schedule`, `schedules`, `deleted` 같은 top-level key가 들어왔는지 확인해야 한다.

### 4. PR과 리뷰 흐름도 학습 과정의 일부
- **핵심 개념**: 주차별 작업은 `<이름>/weekN` 브랜치에서 진행하고, PR base는 `main`이 아니라 개인 누적 브랜치인 `<이름>/final`로 둔다.
- **내가 이해한 방식**:
  - 이번 클론코딩 프로젝트는 코드 구현만큼이나 정해진 협업 흐름을 지키는 것도 중요하다.
  - PR을 규칙대로 열면 멘토 리뷰어 지정과 Discord 알림은 자동으로 처리된다. 대신 base branch를 잘못 잡거나 Draft로 열면 리뷰 흐름이 꼬일 수 있다.
  - 수정 요청을 받으면 새 PR을 만드는 것이 아니라 같은 week 브랜치에 이어서 push하고, 재리뷰 요청을 눌러야 한다는 점을 따로 기억해 두어야겠다.

## 🧱 막혔던 지점 & 해결 과정

### 1. 최종 답변만 보면 실제 동작을 확인했다고 착각하기 쉬움
- **문제 상황**: 채팅 답변이 자연스럽게 나오면 일정 생성이나 조회가 성공한 것처럼 느껴진다. 하지만 Agentic AI에서는 답변이 그럴듯해도 tool을 호출하지 않았거나, 잘못된 arguments로 호출했을 수 있다.
- **시도한 방법**: 강의 자료의 trace 예시를 기준으로 `tool_call`, `arguments`, `tool_result`를 순서대로 보는 방식으로 정리했다.
- **최종 해결 / 배운 점**: 최종 답변은 마지막 확인 대상이고, 먼저 봐야 하는 것은 tool 이름, 입력 인자, 결과 payload다. 특히 일정 생성에서는 `title`, `date`, `start_time`, `attendees`가 사용자 요청과 맞는지 확인해야 한다.

### 2. 임시 메모리와 영구 저장소의 경계가 처음에는 헷갈림
- **문제 상황**: 이전 프리코스에서는 DB나 배포 환경도 다뤘기 때문에, 이번 Week 1 일정도 어딘가에 영구 저장되는 것처럼 생각할 수 있었다.
- **시도한 방법**: Week 1 문서와 과제 파일의 구현 가이드를 다시 읽으면서 `PERSONAL_SCHEDULES`와 `session_id` 기준으로 저장 범위를 구분했다.
- **최종 해결 / 배운 점**: Week 1의 목적은 영구 저장이 아니라 tool calling의 가장 작은 단위를 이해하는 것이다. 그래서 현재 대화 범위의 임시 리스트만 사용하고, SQLite/App store 같은 저장소는 뒤 주차에서 다루는 것으로 분리해서 생각하면 된다.

### 3. 과제 범위를 어디까지 봐야 하는지 정리할 필요가 있었음
- **문제 상황**: 별도 과제 저장소에는 `fixed` 코드, MCP 관련 파일, 앱 실행 파일 등 여러 파일이 있어서 처음 보면 어디를 수정해야 하는지 범위가 커 보였다.
- **시도한 방법**: `README.md`, `CURRICULUM.md`, `student_parts/week01_wake_up_nana.py`의 수강생 구현 가이드를 먼저 읽었다.
- **최종 해결 / 배운 점**: Week 1에서는 구현 대상 함수가 명확히 `student_parts/week01_wake_up_nana.py` 안의 세 tool로 좁혀져 있다. 주변 코드는 agent가 tool 목록을 어떻게 받고 trace를 어떻게 보여주는지 이해하기 위한 참고 코드로 보면 된다.

## 🔁 이번 주 회고 (KPT)
- **Keep** 유지하고 싶은 습관: 강의 자료를 볼 때 최종 결과 화면만 보지 않고, trace와 payload를 함께 확인하려고 한 점. 특히 tool 이름과 JSON key를 기준으로 동작을 검증하는 습관은 앞으로의 주차에서도 계속 필요할 것 같다.
- **Problem** 아쉬웠던 점: 첫 주라 강의 repo, 학교별 clone repo, PR 규칙, 별도 과제 흐름이 한꺼번에 들어와서 전체 구조를 잡는 데 시간이 걸렸다. Agentic AI 개념 자체보다 "어느 저장소에서 무엇을 해야 하는지"를 파악하는 데 에너지를 많이 쓴 느낌이 있다.
- **Try** 다음 주에 시도할 것: 강의를 들은 당일에 바로 "프롬프트 1개 + 호출된 tool + 결과 payload"를 짧게 기록해 두기. 과제도 구현만 끝내지 말고, 내가 확인한 trace를 같이 남겨서 나중에 같은 흐름을 다시 설명할 수 있게 만들기.

## 🎯 다음 주 목표
- [ ] Week 2의 structured output 흐름을 보면서 자연어 요청이 어떤 schema로 정리되는지 설명하기
- [ ] 개인 일정 생성, 조회, 삭제 각각에 대해 직접 실행한 프롬프트와 trace 결과를 하나씩 기록하기
- [ ] 주차별 브랜치와 PR base 규칙을 다시 확인하고, 리뷰 요청 흐름까지 실수 없이 진행하기

---

## 🔗 관련 자료
- [Agentic AI 오리엔테이션](../../kakao_clone_coding_lecture/docs/orientation.md)
- [1주차 강의 문서](../../kakao_clone_coding_lecture/docs/week01.md)
- [1주차 강의 노트북](../../kakao_clone_coding_lecture/notebook/1주차_나나를_깨우다.ipynb)
- 별도 과제 저장소: `/Volumes/samsd/workspace_v2/kakao-tech-campus-step1-clone/kyungpook-clone`
- 별도 과제 구현 파일: `student_parts/week01_wake_up_nana.py`
