# week03_SQLite 기록장으로 Nana의 기억을 남기기

## 🗓 이번 주 개요
- 주차: Week 03 (7/13~7/19)
- 진행한 것:
  - Week 3 강의자료 반영
  - `student_parts/week03_build_nanas_logbook.py` 구현
  - SQLite 기반 저장, 조회, 수정, 삭제 tool 구현
  - PR 리뷰 피드백 반영
- 키워드: #AgenticAI #SQLite #Pydantic #ToolCalling #StructuredOutput #LangChain #Trace #PRReview

## 📚 이번 주 학습한 것

### 1. Week 2 구조화 결과를 SQLite에 저장하는 흐름
- **핵심 개념**: Week 2에서 만든 `StructuredRequest`를 Week 3에서는 Pydantic 입력 스키마로 검증한 뒤 SQLite에 저장한다.
- **내가 이해한 방식**:
  - Week 1은 현재 대화 안의 임시 메모리였고, Week 2는 자연어 요청을 구조화하는 단계였다.
  - Week 3부터는 구조화된 요청이 실제 앱 DB에 남는다. 그래서 "저장했다"는 답변보다 `extract_schedule_request → save_structured_request` 흐름이 실제 trace에 찍히고, SQLite row로 남는지가 더 중요해졌다.
  - `save_structured_request`는 SQL을 직접 작성하는 함수라기보다, 이미 검증된 tool 인자를 `AppSQLiteStore`에 넘기는 얇은 입구로 이해했다. DB 접근은 store 계층이 담당하고, 학생 파일의 tool은 입력 검증과 반환 payload 모양을 책임진다.

### 2. 저장된 요청과 저장된 일정은 다른 조회 대상이다
- **핵심 개념**: `list_saved_requests`는 원본 구조화 요청 이력을 조회하고, `personal_list_saved_schedules`는 실제 일정으로 정규화된 row를 조회한다.
- **내가 이해한 방식**:
  - 처음에는 둘 다 "조회 tool"처럼 보였지만, 멘토 리뷰를 통해 둘의 책임을 더 명확히 나누게 됐다.
  - `list_saved_requests`는 `structured_requests`에 저장된 원본 요청을 보는 tool이다. 일정뿐 아니라 todo, reminder, unknown 같은 요청 이력까지 확인할 때 어울린다.
  - `personal_list_saved_schedules`는 실제 일정 목록을 보는 tool이다. "내 일정 보여줘" 같은 질문에 답하거나, 수정·삭제 전에 `schedule_id` 후보를 확인할 때 사용한다.
  - 이 구분을 프롬프트에 명시하지 않으면 모델이 같은 조회 요청에서도 서로 다른 tool을 선택할 수 있다. 그래서 상황별 tool 선택 기준을 프롬프트에 넣고, 개인 일정, 그룹 일정, 할 일, 알림 조회 문장을 직접 돌려 의도한 tool이 선택되는지 확인했다.

### 3. 수정·삭제 tool에는 안전장치가 필요하다
- **핵심 개념**: 저장된 일정을 수정하거나 삭제할 때는 먼저 조회로 후보를 확인하고, 삭제 조건이 비어 있으면 실행하지 않아야 한다.
- **내가 이해한 방식**:
  - 저장 tool은 잘못 호출되어도 잘못된 row 하나가 생기는 정도지만, 삭제 tool은 조건을 잘못 잡으면 여러 row가 한 번에 사라질 수 있다.
  - 그래서 `personal_delete_saved_schedules`에서는 `schedule_ids`, `date`, `title`, `start_time`, `time_unspecified`, `delete_all` 중 명시적인 조건이 있을 때만 삭제하도록 guard를 두었다.
  - 조건 없는 삭제는 `ok=False`, `deleted_count=0`, `error`를 포함한 JSON으로 돌려주게 만들었다. 이 응답 구조 덕분에 trace만 봐도 "왜 삭제가 안 됐는지"를 확인할 수 있다.

### 4. 리뷰 피드백으로 응답 payload의 일관성을 배움
- **핵심 개념**: tool 반환 JSON은 LLM이 읽는 인터페이스이기 때문에, 같은 성격의 결과 정보는 같은 계층에 두는 것이 좋다.
- **내가 이해한 방식**:
  - 처음 구현에서는 저장 결과가 `saved` 한 겹 안에 들어가거나, Week 1 호환 생성의 `sqlite_save`에서 `ok`, `tool_name` 같은 정보가 일부 유실될 수 있었다.
  - 리뷰를 통해 `request_id`, `kind`, `saved_rows`, `shared_sync` 같은 저장 결과도 `ok`, `tool_name`과 같은 계층에 두는 편이 더 일관적이라는 점을 배웠다.
  - 또한 `original_text`에는 JSON 문자열이나 조합한 문장을 억지로 넣기보다, 실제 사용자 원문이 없으면 빈 문자열을 유지하는 것이 필드 의미에 맞다는 점도 짚고 넘어갔다.
  - 이번 리뷰는 단순히 "테스트가 통과하는가"를 넘어서, LLM이 읽을 payload를 얼마나 일관되고 설명 가능하게 설계했는지를 보게 해준 피드백이었다.

## 🧱 막혔던 지점 & 해결 과정

### 1. Week 1 호환 tool과 Week 3 저장 흐름이 함께 노출되는 문제
- **문제 상황**: Week 1의 `personal_create_schedule` 이름을 유지하면서도 Week 3에서는 SQLite 저장까지 해야 했다. 모델이 임시 일정만 만들고 저장 단계를 생략할 가능성이 있었다.
- **시도한 방법**: Week 1 tool 호출 결과를 `structured_request_from_week01_schedule()`로 변환하고, 같은 내용을 `save_structured_request_payload()`로 SQLite에 저장하는 호환 wrapper를 만들었다.
- **최종 해결 / 배운 점**: 과거 주차와 호환되는 이름을 유지하더라도, 뒤 주차의 책임을 wrapper에서 보강할 수 있다. 다만 trace에서 임시 생성 결과와 SQLite 저장 결과가 모두 보이도록 `structured_request`, `sqlite_save`를 함께 반환하는 것이 중요했다.

### 2. 조회 tool 선택 기준이 처음에는 느슨했음
- **문제 상황**: 프롬프트에서 저장 요청 조회에 `list_saved_requests` 또는 `personal_list_saved_schedules`를 사용한다고만 적으면 모델 입장에서 두 tool의 차이가 모호했다.
- **시도한 방법**: 리뷰 코멘트를 바탕으로 두 tool이 보는 테이블과 사용 상황을 나눠 정리했다.
- **최종 해결 / 배운 점**: 일정 목록과 캘린더 조회는 `personal_list_saved_schedules`, 할 일·알림·unknown 및 저장 요청 이력은 `list_saved_requests`, 정확한 `request_id` 단건 원본 조회는 `get_saved_request`로 프롬프트에 명시했다. 이후 조회 문장 4개를 직접 실행해 의도한 tool 라우팅을 확인했다.

### 3. 브랜치 흐름이 지침과 달라진 점
- **문제 상황**: 처음에는 Week 3 강의자료를 주차 브랜치에 직접 반영해, 최신 `main`을 `final`에 먼저 반영하고 그 `final`에서 주차 브랜치를 따는 흐름과 달라졌다.
- **시도한 방법**: PR 본문에 이 문제를 명확히 적고, 다음 주부터는 작업 전에 최신 `main → final → weekN` 흐름을 먼저 확인하기로 정리했다.
- **최종 해결 / 배운 점**: 기능 구현만큼 브랜치 흐름도 과제의 일부다. 특히 공통 강의자료와 내 구현 코드가 함께 바뀌는 프로젝트에서는, 어떤 변경이 "내 작업"이고 어떤 변경이 "반영해야 할 베이스 코드"인지 구분해야 PR diff가 깔끔해진다.

## 🔍 리뷰 피드백에서 배운 것
- 저장 결과 payload는 `saved` 같은 중첩 key 안에 숨기기보다, `request_id`, `kind`, `saved_rows`, `shared_sync`를 `ok`, `tool_name`과 같은 계층에 두는 편이 LLM과 사람이 모두 읽기 좋다.
- `sqlite_save`처럼 다른 helper 결과를 포함하는 경우에도 `ok`, `tool_name`이 유실되지 않게 전체 결과를 보존해야 한다.
- `original_text`는 이름 그대로 원본 자연어 요청이어야 한다. 실제 원문이 없는 Week 1 호환 경로에서는 빈 문자열을 유지하는 편이 억지로 만든 문장을 저장하는 것보다 낫다.
- `list_saved_requests`와 `personal_list_saved_schedules`의 차이를 프롬프트에 명시해야 모델의 tool 선택이 안정된다.
- 리뷰 반영 후에는 "수정했습니다"에서 끝내지 말고, 실제 조회 문장을 돌려 의도한 tool이 선택되는지까지 확인해야 한다.

## 🔁 이번 주 회고 (KPT)
- **Keep** 유지하고 싶은 습관: 구현 가이드를 입력 스키마, DB 호출, 반환 JSON, trace 검증으로 나눠 보는 방식. 이번 주에는 `py_compile`, `git diff --check`, row count 확인, DB 재오픈 후 영속성 확인, 실제 LLM 반복 실행까지 이어서 검증한 점이 좋았다.
- **Problem** 아쉬웠던 점: 처음부터 조회 tool의 책임을 명확하게 나누지 못했고, 브랜치 흐름도 지침과 조금 다르게 시작했다. 구현은 진행됐지만 PR 리뷰에서 설명해야 할 지점이 늘어났다.
- **Try** 다음 주에 시도할 것: 구현 전에 tool별 책임 표를 먼저 만들고, 프롬프트에는 "언제 어떤 tool을 쓰는지"를 기준 중심으로 적기. 또한 주차 시작 전에 최신 `main` 반영 방식과 브랜치 시작점을 먼저 확인하기.

## 🎯 다음 주 목표
- [ ] Week 4에서 RAG나 검색 흐름이 추가될 때 SQLite 저장 결과와 검색 근거를 구분해서 설명하기
- [ ] tool 반환 JSON의 계층 구조를 먼저 설계하고 구현하기
- [ ] PR 올리기 전에 "내 작업 변경"과 "베이스 코드 반영 변경"이 섞이지 않았는지 diff로 확인하기

---

## 🔗 관련 자료
- 3주차 PR: https://github.com/kakaotechcampus-4/kyungpook-clone/pull/122
- 별도 과제 저장소: `/Volumes/samsd/workspace_v2/kakao-tech-campus-step1-clone/kyungpook-clone`
- 주요 구현 파일: `student_parts/week03_build_nanas_logbook.py`
- 연결 구현 파일: `student_parts/week02_structure_natural_language_requests.py`
