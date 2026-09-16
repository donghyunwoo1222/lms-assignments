# Chapter 07 확장 실습 답안 템플릿

> **과제:** 실전 프로젝트 1 — 온라인 강의 수강신청 DB 완성하기  
> **사용 방법:** 이 파일을 내려받아 본인의 GitHub 저장소에 `chapter07_answer.md`라는 이름으로 저장한 뒤 실습하면서 바로 작성합니다.  
> **제출 방법:** LMS에는 파일을 직접 업로드하지 않고, **본인 GitHub 저장소의 `chapter07_answer.md` 파일 URL**을 제출합니다.

---

## 제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

```text
GitHub 계정 또는 별칭: donghyunwoo1222
과제 작성일: `26.09.15
사용한 AI 도구: claude
```

---

# 1. 시작 환경 확인

다음을 실행합니다.

```sql
SELECT current_database();
SELECT current_user;
SELECT current_schema();
SHOW search_path;
SHOW transaction_read_only;  
```

| 확인 항목 | 실제 결과 | 의미 |
| --- | --- | --- |
| `current_database()` |ai_database_book  |현재 데이터베이스  |
| `current_user` |postgres  |유저 이름  |
| `current_schema()` |public  |현재 스키마  |
| `search_path` |"$user", public  |현재 경로  |
| `transaction_read_only` |off  |쓰기 가능한 연결인가  |

- [x] 현재 DB가 `ai_database_book`이다.
- [x] 쓰기 가능한 연결인지 확인했다.
- [x] 실행할 SQL 범위를 확인했다.
- [x] Auto-commit 상태를 확인했다.

### 프로젝트 SQL을 실행하기 전에 시작 상태를 확인해야 하는 이유

```text

```

---

# 2. 프로젝트 범위와 요구사항 읽기

## 2-1. 포함 범위

본문을 그대로 복사하지 말고 자신의 말로 정리합니다.

```text
1. 학생
2. 강사
3. 강의
4. 수강신청
5. 신청 상태
6. 강의 기준 가격
7. 신청 당시 기록 금액
```

## 2-2. 제외 범위

```text
1. 실제 결제 승인·실패·환불
2. 강의 정원·대기열
3. 전체 상태 변경 이력
4. 진도·수료·콘텐츠
5.쿠폰·할인 이력
```

### 범위를 명확하게 정해야 하는 이유

```text
범위를 정해두지 않으면 설계 도중에 결제/정원 같은 기능을 즉흥적으로 끼워넣게 되어 테이블 구조가 계속 흔들린다. 
```

## 2-3. 요구사항 / 프로젝트 결정 / 미확정 질문 구분

아래 항목 중 대표 항목을 정리합니다.

| ID | 종류 | 내용 요약 | DB 구조/규칙에 미치는 영향 |
| --- | --- | --- | --- |
| P07-R01 | 요구사항 | 학생은 이름, 이메일, 가입일을 가진다 | students 테이블에 name, email, joined_at(또는 유사) 열 필요 |
| P07-R05 | 요구사항 | 수강신청은 학생, 강의, 신청일, 상태, 신청 시 기록 금액을 가진다 | enrollments가 student_id·course_id(FK), enrolled_at, status, recorded_amount를 가져야 함 |
| P07-R07 | 요구사항 | 학생·강사 이메일은 각 테이블 안에서 공백·중복 문자열 불허 | students.email, instructors.email에 NOT NULL/CHECK(공백 금지) + UNIQUE 적용 |
| P07-D02 | 프로젝트 결정 | 할인 기능 없음 → 신청 생성 시 courses.price를 recorded_amount에 복사 | INSERT ... SELECT로 값 복사, CHECK/FK가 자동 처리하지 않음(수동 보존) |
| P07-D03 | 프로젝트 결정 | 진행 중 중복 신청 금지 | (student_id, course_id)에 status IN ('신청','수강중') 조건의 부분 고유 인덱스 |
| P07-Q01 | 미확정 질문 | 학생·강사 이메일을 테이블 간에도 전역 고유하게 제한해야 하는가 | 아직 결정 안 됨 → 현재는 각 테이블 내부에서만 UNIQUE, 전역 UNIQUE는 적용 안 함 |

### 미확정 질문을 바로 제약조건으로 만들면 안 되는 이유

```text
미확정 질문은 아직 업무적으로 합의되지 않은 정책이다. 이걸 설계자가
임의로 판단해서 UNIQUE나 CHECK로 미리 확정해버리면
 
- 나중에 실제 정책이 다르게 결정됐을 때 제약조건을 다시 뜯어고쳐야 하고
- 그 사이에 이미 저장된 데이터가 새 규칙과 충돌해 마이그레이션이 어려워지며
- 애초에 "누가 이 규칙을 정했는가"라는 책임 소재가 불명확해진다.
 
그래서 미확정 질문은 문서에 질문 형태로만 남기고, 실제 결정이 내려진
뒤에 제약조건으로 옮기는 것이 안전하다.
```

---

# 3. 네 테이블의 한 행 의미와 관계

## 3-1. 한 행 의미

```text
course_project.students 한 행 = 학생 한 명

course_project.instructors 한 행 = 강사 한 명 

course_project.courses 한 행 = 강의 한 개 

course_project.enrollments 한 행 = 신청한 사건 한 개
```

## 3-2. 키와 중요 규칙

| 테이블 | PK | FK | 중요 규칙 |
| --- | --- | --- | --- |
| students | id | 없음 | email NOT NULL + UNIQUE, 이름 공백 금지 |
| instructors | id | 없음 | email NOT NULL + UNIQUE, 이름 공백 금지 |
| courses | id | instructor_id → instructors.id | price NOT NULL + CHECK(price >= 0), 난이도 값 CHECK |
| enrollments | id | student_id → students.id, course_id → courses.id | status CHECK(허용값), recorded_amount NOT NULL + CHECK(>=0), 진행 중 중복 신청 금지(부분 고유 인덱스), 부모 삭제 시 ON DELETE RESTRICT |

## 3-3. 관계를 양방향 문장으로 작성

```text
instructors ↔ courses: 한 강사는 하나 이상의 강의를 맡을 수 있다. 
한 강의는 한 명의 강사를 참조한다. 

students ↔ enrollments: 한 학생은 하나 이상의 수강신청을 할 수 있다. 
한 수강신청은 한 명의 학생을 참조한다. 

courses ↔ enrollments: 한 강의는 하나 이상의 수강신청을 가질 수 있다. 
한 수강신청은 한 개의 강의를 참조한다. 
```

### 학생과 강의의 N:M 관계가 `enrollments`를 통해 어떻게 바뀌는지 설명

```text
학생 한 명은 여러 강의를 들을 수 있고, 강의 하나에는 여러 학생이 들어올 수
있으므로 원래 관계는 N:M이다. 관계형 DB는 N:M을 테이블 하나로 직접 표현할
수 없으므로, 중간에 enrollments라는 별도 테이블을 두고
 
- students(1) ─ enrollments(N): 학생 1명이 여러 신청을 가짐
- courses(1) ─ enrollments(N): 강의 1개가 여러 신청을 가짐
 
이렇게 두 개의 1:N 관계로 쪼갠다. 즉 "학생이 강의를 듣는다"는 사실 자체를
enrollments의 한 행(신청 사건)으로 표현해서 N:M을 해소한 것이다.
```

### `enrollments`가 단순 연결 테이블이 아니라 사건 테이블이라고 볼 수 있는 이유

```text
단순 연결 테이블이라면 student_id와 course_id 두 개의 FK만 있으면 된다.
하지만 enrollments는 그 외에도 enrolled_at(신청일), status(신청 상태),
recorded_amount(신청 당시 금액)라는, "이 신청 사건 자체"에만 속하는 속성을
갖고 있다. 이 값들은 학생 전체의 속성도 아니고 강의 전체의 속성도 아니라,
"이 학생이 이 강의를 이 시점에 신청한 사건" 하나에만 딸린 사실이다.
그래서 enrollments는 관계를 잇는 다리 역할을 넘어서, 그 자체로 독립적인
업무 의미(사건)를 가진 엔티티로 봐야 한다.
```

---

# 4. `recorded_amount`의 의미 이해

```text
courses.price = 해당 강의의 현재 기준 가격 (지금 이 순간의 가격, 계속 바뀔 수 있음)
 
enrollments.recorded_amount = 그 신청이 생성된 시점에 courses.price를 복사해 enrollments 행에 고정 저장한 금액 (사건 당시 값, 이후 변하지 않음)
```

### 두 값이 처음에는 같아도 같은 의미가 아닌 이유

```text
신청이 만들어지는 순간에는 courses.price를 그대로 복사하므로 두 값이
같다. 하지만 courses.price는 "지금 강의를 사면 얼마인가"를 나타내는
현재 사실이라 강의 가격이 바뀌면 함께 바뀌고, recorded_amount는
"그때 그 신청이 얼마로 기록됐는가"를 나타내는 과거 사건의 기록이라
가격이 바뀌어도 그대로 남아 있어야 한다. 즉 같은 숫자라도 하나는
"현재 상태"를, 하나는 "특정 시점의 스냅샷"을 의미하므로 서로 다른 사실이다.
```

### `recorded_amount`를 실제 결제 성공액이나 회계 매출로 해석하면 안 되는 이유

```text
recorded_amount는 신청을 생성할 때 강의 가격을 복사해 기록해 둔 값일 뿐,
실제로 결제가 승인됐는지, 환불됐는지, 부분 취소됐는지를 반영하지 않는다.
이번 프로젝트 범위에는 payments, payment_events, refunds 같은 실제 결제
처리 구조가 없으므로, recorded_amount의 합계를 그대로 매출이나 실제
수납액으로 취급하면 안 된다. 이 값은 어디까지나 "신청 시점에 기록된
금액"이라는 좁은 의미로만 해석해야 한다.
```

---

# 5. STEP 01 — 스키마와 테이블 생성

실행 파일:

```text
code/chapter07/01_course_project_schema.sql
```

## 5-1. 실행 전 예상

```text
course_project 스키마 존재 여부: 없음
예상 테이블 수: 4
예상 데이터 행 수: 0
예상되는 명명 제약조건 수: 15
예상되는 NOT NULL 열 수: 20
부분 고유 인덱스 존재 여부: 있음(uq_course_enrollments_active)
```

## 5-2. 실행 결과

```text
실제 테이블 수: 4
실제 명명 제약조건 수: 15
실제 NOT NULL 열 수: 20
부분 고유 인덱스: uq_course_enrollments_active
네 테이블의 실제 행 수: 0/0/0/0
통과 메시지: Chapter 07 course project schema creation passed
```

### 예상과 실제 비교

```text
일치함. 
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step05_schema.png
```

![alt text](image.png)

---

# 6. STEP 02 — Seed 데이터 입력

실행 파일:

```text
code/chapter07/02_course_project_seed.sql
```

## 6-1. 실행 전 예상

```text
students = 3
instructors = 2
courses = 3
enrollments = 4
recorded_amount 합계 = 470000
학생 101 신청 = 2
강의 301 신청 = 2
강사 201 강의 = 2
활성 중복 = 0
```

## 6-2. 실제 결과

```text
students:3
instructors:2
courses:3
enrollments:4
recorded_amount 합계: 470000
학생 101 신청 건수: 2
강의 301 신청 건수:2
강사 201 담당 강의 수: 2 
활성 중복 신청: 0 
1001 상태: 수강중
1004 상태: 신청 
1005 존재 여부: 없음
통과 메시지:Chapter 07 course project seed passed
```

### Seed 데이터를 단순 예제가 아니라 검증 데이터라고 볼 수 있는 이유

```text
Seed 데이터는 화면을 채우기 위한 임의의 샘플이 아니라, students=3, instructors=2, courses=3,
enrollments=4처럼 정확한 행 수와 관계가 미리 정해진 "기준값"입니다. 이후 실행하는 검증
쿼리들은 이 정해진 숫자와 실제 쿼리 결과가 일치하는지를 비교하는 방식으로 동작합니다.
즉 Seed는 예시 데이터가 아니라, 스키마·제약조건·관계가 의도대로 동작하는지를 확인할 수
있는 "정답이 있는 테스트 데이터"입니다.
```

---

# 7. STEP 03 — 변경 시나리오 실행

실행 파일:

```text
code/chapter07/03_course_project_changes.sql
```

## 7-1. 실행 전에 상태 변화를 예상

| 신청 ID | 변경 전 예상 상태 | 변경 후 예상 상태 | 예상 recorded_amount |
| ---: | --- | --- | ---: |
| 1001 |수강중  |완료  |100000  |
| 1004 |신청  |취소  |150000  |
| 1005 |없음  |신청  |120000  |

```text
변경 후 예상 enrollments 행 수: 5
변경 후 예상 전체 recorded_amount 합계: 590000
변경 후 예상 취소 제외 건수: 4건
변경 후 예상 취소 제외 recorded_amount 합계: 440000
```

## 7-2. 실제 결과

```text
1001 상태 / recorded_amount: 완료
1004 상태 / recorded_amount: 취소
1005 상태 / recorded_amount: 신청
최종 enrollments 행 수: 5개 
전체 recorded_amount 합계: 590000
취소 제외 건수:4
취소 제외 recorded_amount 합계:440000
활성 중복 신청: 0 
통과 메시지:
```

### 조건부 UPDATE에서 예상 이전 상태를 확인해야 하는 이유

```text
UPDATE는 실행 후 에러가 없다고 해서 의도한 행만 바뀌었다는 보장이 없다. WHERE 조건이
잘못되면 엉뚱한 행이 바뀌거나, 아무 행도 안 바뀌었는데도 성공한 것처럼 보일 수 있다.
그래서 변경 전 상태를 먼저 SELECT로 확인해두면, 변경 후 결과와 비교해서 "정확히 의도한
행이, 의도한 값으로만" 바뀌었는지 검증할 수 있다. 이전 상태 기록이 없으면 UPDATE가
실제로 맞게 동작했는지 판단할 기준 자체가 없어진다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step07_changes.png
```

![alt text](image-1.png)

---

# 8. STEP 04 — 최종 완료 게이트 실행

실행 파일:

```text
code/chapter07/04_course_project_validation.sql
```

## 8-1. 최종 검증 결과

```text
최종 행 수 students/instructors/courses/enrollments:
서비스 JOIN 결과 행 수: 5
학생 101 신청 수:2
강의 301 신청 수:2
강사 201 강의 수:2
고아 관계 수:0
활성 중복 신청 수:0
전체 recorded_amount:590000
취소 제외 recorded_amount:440000
통과 메시지:
```

### SQL 파일 4개가 모두 실행되었다는 사실과 프로젝트 검증 PASS가 다른 이유

```text
SQL 파일이 에러 없이 끝까지 실행되었다는 것은 "문법 오류나 런타임 오류 없이 명령이 수행됐다"는 뜻일 뿐이다. 반면 검증 PASS는 그 결과로 만들어진 데이터가 우리가 기대한 조건(행 수,합계, 제약조건 준수 여부 등)을 실제로 만족하는지까지 확인한 것이다. 즉 "실행됨"은 과정에 대한 확인이고, "PASS"는 결과의 정확성에 대한 확인이라서, 실행이 끝났다고 검증까지 통과했다고 단정할 수 없다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step08_validation.png
```

![alt text](image-2.png)

---

# 9. 무결성 테스트

실행 파일:

```text
code/chapter07/05_course_project_integrity_tests.sql
```

> 오류 테스트는 파일 전체를 무작정 실행하지 않고 **한 테스트 구간씩** 실행합니다.

## 9-1. 허용되어야 하는 경계값 1개

```text
테스트 내용: price = 0 
기대 결과:에러 없이 INSERT 성공, 이후 DELETE로 임시 행 정리도 정상 처리됨
실제 결과:에러 없이 INSERT 성공, 이후 DELETE로 임시 행 정리도 정상 처리됨
왜 허용되어야 하는가:price와 recorded_amount는 0 이상이면 되는 값이라 0원(무료 강의/무료신청)은 정상적인 비즈니스 케이스
```

## 9-2. 실패해야 하는 테스트 1 — 잘못된 참조 또는 값

```text
테스트 내용: 학생 이메일 중복
기대 결과: 에러발생
실제 오류 핵심: 키가 이미 있다. 고유 제약 조건 위반 
동작한 제약조건/규칙: unique 제약조건
왜 실패해야 하는가: 같은 이메일을 가진 학생 행이 두 개 이상 생기면, 구분할 수가 없다. 그래서 이메일 중복을 애초에 차단하는 것
```

## 9-3. 실패해야 하는 테스트 2 — 활성 중복 신청

```text
테스트 내용: 이미 학생 101이 강의 302에 활성 신청(1002)이 있는 상태에서, 같은
             학생·같은 강의로 상태 '수강중'인 두 번째 신청(1910)을 INSERT 시도함
기대 결과: 에러 발생 
실제 오류 핵심: 학생 101, 강의 302 조합이 이미 활성 상태로 존재한다. 
동작한 인덱스/규칙: unique 제약 
왜 실패해야 하는가: 같은 학생이 같은 강의를 동시에 두 번 "진행 중" 상태로 신청하면 어떤
                 신청이 유효한 건지 알 수 없는 모순이 생김. 부분 고유 인덱스가 학생+강의
                 조합의 활성 중복을 원천 차단해서 이런 모순을 막아줌
```

## 9-4. 실패 후 기준 상태 재검증

```text
04 validation 재실행 결과:
기준 데이터가 유지되었는가:
```

### 실패 테스트가 프로젝트 품질 검증에 필요한 이유

```text
정상적인 입력만 테스트하면 "올바르게 쓸 때 잘 동작한다"는 것만 증명된다. 하지만 실무
데이터베이스는 잘못된 입력, 중복 신청, 존재하지 않는 참조 같은 비정상적인 상황도 반드시
막아내야 한다. 실패해야 하는 테스트가 실제로 실패(거부)하는지 확인해야만, 제약조건과
규칙이 "느슨하게 통과시키는 게 아니라 실제로 방어 역할을 하고 있다"는 것을 증명할 수
있다. 실패 테스트가 없으면 시스템이 우연히 잘 동작한 것인지, 실제로 안전한지 구분할
수 없다.
```

### 증거 화면

권장 경로:

```text
assignments/chapter07/images/step09_integrity.png
```

![alt text](image-3.png)

---

# 10. 재현성 실험

> 이 단계는 본인의 실습 환경이며 보존할 데이터가 없을 때만 수행합니다.

실행 순서:

```text
reset_course_project.sql
→ 01_course_project_schema.sql
→ 02_course_project_seed.sql
→ 03_course_project_changes.sql
→ 04_course_project_validation.sql
```

```text
처음 실행의 최종 결과: Chapter 07 course project validation passed
재실행의 최종 결과: Chapter 07 course project validation passed
두 결과가 일치했는가: O. reset 후 같은 순서로 다시 실행했을 때도 최종 validation이 통과했다.
중간에 수동 수정이 필요했는가: X
```

### 다른 사람이 같은 순서로 실행해 같은 결과를 얻는 것이 중요한 이유

```text
같은 스크립트를 같은 순서로 실행했을 때 항상 같은 결과가 나와야, 내 컴퓨터에서만 되는 것이 아니라 협업자·리뷰어·채점자의 환경에서도 동일하게 동작한다는 것을 보장할 수 있다. 재현이 안 되면 "내 로컬에서는 됐다"는 주장을 검증할 방법이 없고, 배포나 팀 작업 시 데이터 불일치·숨은 수동 조작이 버그의 원인이 되기 쉽다. 재현 가능한 스크립트는 곧 "문서화된 사실"이며, 리뷰어가 결과를 신뢰할 수 있는 유일한 근거가 된다.
```

---

# 11. Chapter 01~06 개인 프로젝트를 중간 프로젝트 초안으로 확장

온라인 강의 예제를 이름만 바꾸지 않고 본인의 아이디어를 사용합니다.

## 11-1. 프로젝트 기본 정보

```text
프로젝트 이름: 영화 예매 시스템

해결하려는 문제:사용자가 상영 중인 영화를 검색하고, 원하는 상영관/회차의 좌석을 선택해
예매 및 결제하며, 예매 내역을 확인·취소할 수 있는 온라인 영화 예매 서비스가 필요하다.
동시에 같은 좌석을 여러 명이 중복 예매하지 못하도록 막고, 상영 취소/변경 시 예매자에게
영향이 전파되도록 데이터 구조가 뒷받침되어야 한다.

주요 사용자: 일반 회원 / 관리자
```

## 11-2. 포함 범위 / 제외 범위

```text
[포함]
1. 영화 정보 등록 및 조회 (제목, 장르, 상영시간, 등급)
2. 상영관/좌석 배치 관리
3. 상영 스케줄(날짜·시간·상영관) 관리
4. 좌석 선택 및 예매 생성, 예매 취소
5. 결제 상태 관리(결제완료/미결제/환불)

[제외]
1. 실시간 PG사 결제 연동(실제 카드 승인 로직) — 결제 "상태"만 DB에서 관리
2. 회원 소셜 로그인/OAuth 연동
3. 영화 추천 알고리즘, 리뷰/평점 기능
```

## 11-3. 요구사항

최소 8개를 작성합니다.

| ID | 요구사항 | 관련 테이블/관계 | 검증 방법 후보 |
| --- | --- | --- | --- |
| P07-MR01 | 모든 예매는 반드시 하나의 상영 스케줄(showtime)에 속해야 한다 | reservations → showtimes (FK) | 존재하지 않는 showtime_id를 가진 예매가 0건인지 확인 |
| P07-MR02 | 같은 상영 스케줄에서 동일 좌석은 동시에 하나의 유효한(취소되지 않은) 예매만 가질 수 있다 | reservations, seats, showtimes 복합 유니크 제약 | 같은 (showtime_id, seat_id) 조합으로 status='CONFIRMED'인 행이 2건 이상인지 검사 |
| P07-MR03 | 결제가 완료되지 않은 예매는 일정 시간 후 자동으로 만료(EXPIRED) 상태가 되어야 한다 | reservations.status, reservations.expires_at | 만료 시각이 지난 PENDING 예매가 남아있지 않은지 검사하는 배치/쿼리 |
| P07-MR04 | 회원은 자신의 예매 내역만 조회·취소할 수 있어야 한다 | reservations.user_id ↔ users.user_id | 다른 회원의 예매를 취소하는 쿼리가 0건 영향인지 확인 |
| P07-MR05 | 상영 스케줄은 영화, 상영관, 시작시간, 종료시간을 반드시 가져야 한다 | showtimes 필수 컬럼 NOT NULL | NULL 컬럼 존재 여부 검증 쿼리 |
| P07-MR06 | 같은 상영관에서 상영 시간은 겹칠 수 없다 | showtimes.theater_id + 시간 구간 | 같은 theater_id에서 시간 구간이 겹치는 두 행이 있는지 검사 |
| P07-MR07 | 결제 금액은 좌석 등급별 가격의 합과 일치해야 한다 | payments.amount, seats.price_grade | 예매 좌석 가격 합계 vs payments.amount 비교 쿼리 |
| P07-MR08 | 취소된 예매의 좌석은 다시 예매 가능한 상태로 즉시 반영되어야 한다 | reservations.status='CANCELLED' → 좌석 재판매 가능 | 취소 후 동일 좌석으로 새 예매 INSERT가 성공하는지 테스트 |

## 11-4. 프로젝트 결정

최소 3개를 작성합니다.

| ID | 이번 프로젝트에서 내린 결정 | 이유 | 구현 후보 |
| --- | --- | --- | --- |
| P07-MD01 | 좌석은 상영관(theater)에 고정된 좌표(row, col)로 관리하고, showtime마다 별도의 좌석 점유 테이블을 둔다 | 좌석 배치는 상영관 고유 속성이고, 예매 가능 여부는 상영 회차마다 달라지기 때문에 분리해야 중복 없이 관리 가능 | seats(theater_id 소속) + reservation_seats(showtime_id, seat_id) |
| P07-MD02 | 결제는 예매(reservation)와 1:1 관계로 별도 테이블로 분리한다 | 결제 실패/환불 등 결제 자체의 상태 변화를 예매 상태와 독립적으로 추적하기 위함 | payments 테이블, reservation_id UNIQUE FK |
| P07-MD03 | 예매 취소는 행을 삭제하지 않고 status를 'CANCELLED'로 변경하는 소프트 삭제 방식을 사용한다 | 취소 이력을 남겨야 환불/통계/CS 대응이 가능하고, 삭제 시 FK로 연결된 결제 이력도 같이 사라지는 문제를 피하기 위함 | reservations.status ENUM, cancelled_at 컬럼 |

## 11-5. 미확정 질문

최소 3개를 작성합니다.

```text
P07-MQ01. 좌석 등급(price_grade)은 상영관 단위로 고정인지, 상영 스케줄(예: 조조/심야 할인)에 따라
          달라지는지 아직 결정하지 못했다.
P07-MQ02. 예매 만료(PENDING → EXPIRED) 처리를 DB 트리거/스케줄러로 자동화할지, 애플리케이션 배치
          작업으로 처리할지 미정이다.
P07-MQ03. 한 번의 예매(reservation)로 여러 좌석을 묶어서 예매하는 구조(예매 1건 : 좌석 N개)로 갈지,
          좌석 1개당 예매 1건으로 갈지 아직 확정하지 못했다.
```

---

# 12. 개인 프로젝트 ERD와 한 행 의미

## 12-1. 테이블 후보

최소 4개를 권장합니다.

| 테이블 | 한 행의 의미 | PK 후보 | FK 후보 | 주요 규칙 |
| --- | --- | --- | --- | --- |
| users | 회원 1명 | user_id | - | email UNIQUE |
| movies | 영화 1편 | movie_id | - | title, genre, rating_grade NOT NULL |
| theaters | 상영관 1개 | theater_id | - | 좌석 수(total_seats) |
| showtimes | 특정 영화의 특정 상영관·시간대 회차 1개 | showtime_id | movie_id → movies, theater_id → theaters | 같은 theater_id는 시간 겹침 불가 |
| seats | 특정 상영관의 좌석 1개 | seat_id | theater_id → theaters | (theater_id, row, col) UNIQUE |
| reservations | 예매 1건 | reservation_id | user_id → users, showtime_id → showtimes | status ENUM('PENDING','CONFIRMED','CANCELLED','EXPIRED') |
| reservation_seats | 특정 예매에 포함된 좌석 1개 | reservation_seat_id | reservation_id → reservations, seat_id → seats | (showtime_id, seat_id) 중 유효 예매는 유일 |
| payments | 결제 1건 | payment_id | reservation_id → reservations (UNIQUE) | amount, status ENUM('UNPAID','PAID','REFUNDED') |

## 12-2. 관계 문장

```text

1. 한 명의 회원(users)은 여러 건의 예매(reservations)를 가질 수 있지만, 하나의 예매는 한 명의
   회원에게만 속한다. (1:N)
2. 하나의 상영 스케줄(showtimes)에는 여러 예매(reservations)가 걸릴 수 있지만, 하나의 예매는
   하나의 상영 스케줄에만 속한다. (1:N)
3. 하나의 예매(reservations)는 여러 좌석(reservation_seats를 통해 seats)을 포함할 수 있고,
   하나의 좌석은 같은 상영 스케줄 내에서 유효한 예매 하나에만 속할 수 있다. (N:M을 중개 테이블로 해소)
```

## 12-3. ERD

권장 이미지 경로:

```text
assignments/chapter07/images/personal_project_erd.png
```

![alt text](image-4.png)

### Chapter 05~06 ERD에서 이번에 바꾼 점

```text
- 예매(reservations)와 좌석을 단순 FK 한 줄로 연결하던 구조에서, 예매 1건이 여러 좌석을
  가질 수 있다는 점을 반영해 reservation_seats 중개 테이블을 새로 분리했다.
- 결제 정보를 reservations 테이블 안의 컬럼(예: payment_status)으로 두었다면, 이번에는
  결제 상태 변화를 독립적으로 추적하기 위해 payments 테이블로 완전히 분리했다.
- 같은 상영관에서 시간이 겹치는 상영을 막는 규칙(P07-MR06)을 이번 단계에서 새로 추가했다.]
```

---

# 13. 개인 프로젝트 완료 기준 만들기

“잘 동작한다”처럼 모호하게 쓰지 말고 검증 가능한 기준을 최소 6개 작성합니다.

| 번호 | 완료 기준 | 자동 SQL 검증 가능? | 검증 방법 |
| ---: | --- | --- | --- |
| 1 | Seed 실행 후 users/movies/theaters/showtimes 테이블의 행 수가 각각 5/4/2/10이다 | O | `SELECT COUNT(*) FROM ...` 각 테이블별 확인 |
| 2 | 존재하지 않는 movie_id/theater_id/user_id를 참조하는 행은 0건이다 | O | LEFT JOIN 후 부모가 NULL인 자식 행 COUNT |
| 3 | 같은 (showtime_id, seat_id) 조합으로 상태가 CONFIRMED인 행은 항상 1건 이하다 | O | GROUP BY (showtime_id, seat_id) HAVING COUNT(*) > 1 인 결과가 0건인지 확인 |
| 4 | 허용되지 않은 reservations.status 값(정의된 ENUM 외)은 DB가 거부한다 | O | 잘못된 status 값 INSERT 시 오류 발생 여부 테스트 |
| 5 | 같은 상영관에서 시간이 겹치는 두 개의 showtime은 존재하지 않는다 | O | 같은 theater_id 내 시간 구간 겹침 여부를 검사하는 self-join 쿼리 |
| 6 | 예매 취소(status='CANCELLED') 시 해당 좌석은 같은 showtime에서 즉시 재예매 가능하다 | O | 취소 후 동일 (showtime_id, seat_id) 신규 예매 INSERT가 성공하는지 테스트 |

예시 형식:

```text
Seed 실행 후 A/B/C/D 테이블의 행 수가 각각 5/3/8/12다.
존재하지 않는 부모를 참조하는 행은 0건이다.
허용되지 않은 상태 입력은 DB가 거부한다.
검증 SQL이 예상 결과를 반환한다.
```

---

# 14. AI를 프로젝트 리뷰어로 사용

AI에게 프로젝트를 대신 완성시키지 않고 누락과 위험을 찾게 합니다.

## 14-1. AI에게 전달한 핵심 자료

```text
요구사항: 11-3의 요구사항 표 (P07-MR01~08) — 예매/좌석/결제/시간겹침 관련 8개 규칙

테이블/ERD 설명: 12-1, 12-2 — users/movies/theaters/showtimes/seats/reservations/
reservation_seats/payments 8개 테이블과 그 사이 관계 3문장

프로젝트 결정: 11-4의 프로젝트 결정 표 (P07-MD01~03) — 좌석/showtime 분리, 결제 테이블
분리, 소프트 삭제 방식

미확정 질문: 11-5 — 좌석 등급 정책(P07-MQ01), 예매 만료 처리 방식(P07-MQ02), 예매-좌석
카디널리티(P07-MQ03)

완료 기준: 13번 표 — Seed 행 수, 고아 FK, 좌석 중복 예매, 잘못된 status 거부, 상영 시간
겹침, 취소 후 재예매 가능 여부 등 6개
```

## 14-2. 내가 사용한 프롬프트

```text
"영화 예매 데이터베이스 프로젝트로 Chapter07 과제 템플릿을 채워줘. 요구사항, 테이블 설계, 결정, 미확정 질문, 완료 기준을 검토하고 누락이나 위험을 찾아줘."
```

## 14-3. AI 제안 검토

| AI 제안 | 수용 / 수정 / 보류 / 거절 | 실제 근거 | 반영 내용 |
| --- | --- | --- | --- |
| 좌석-예매를 reservation_seats 중개 테이블로 분리 | 수용 | N:M 관계(예매 1건에 여러 좌석)를 정규화하려면 중개 테이블이 필요함 | 12-1 테이블 목록에 반영 |
| 결제(payments)를 예매와 별도 테이블로 분리 | 수용 | 결제 상태와 예매 상태를 독립적으로 추적해야 환불/CS 처리가 쉬움 | P07-MD02로 반영 |
| 같은 상영관 시간 겹침 방지 규칙 추가 | 수용 | 물리적으로 한 상영관에서 두 상영이 동시에 불가능하므로 데이터 정합성 규칙 필요 | P07-MR06, 완료 기준 5번으로 반영 |
| 좌석 등급을 showtime마다 다르게 둘지 여부 확정 제안 | 보류 | 아직 요금 정책(조조/심야 할인 등)을 결정하지 않았으므로 지금 확정하지 않음 | 미확정 질문 P07-MQ01로 남김 |

### AI가 미확정 정책을 임의로 확정하려 한 부분이 있었나요?

```text
있었다. 처음에 AI는 "좌석 등급은 상영관에 고정으로 두는 것이 편하다"는 식으로 제안했는데,
이는 조조/심야 할인처럼 아직 우리가 결정하지 않은 요금 정책을 암묵적으로 확정해버리는
제안이었다. 그래서 이 부분은 수용하지 않고 P07-MQ01 미확정 질문으로 남겨두었다.
```

### AI가 제안한 규칙 중 아직 배우지 않은 기능이라 보류한 것이 있나요?

```text
예매 만료(PENDING → EXPIRED) 자동 처리를 DB 트리거나 이벤트 스케줄러로 구현하자는 제안이
있었지만, 트리거·스케줄러는 수업에서 아직 다루지 않은 기능이라 지금 단계에서는 도입하지
않고 P07-MQ02 미확정 질문으로만 남겨두었다.
```

### AI 활용 후 실제로 좋아진 부분

```text
요구사항과 프로젝트 결정을 섞어서 쓰던 부분을 분리해서 정리할 수 있었고, 좌석 중복 예매를
막는 제약 조건을 처음부터 요구사항에 명시할 수 있었다.
```

---

# 15. 최종 성찰

아래 문장은 반드시 본인의 말로 작성합니다.

```text
1. 데이터베이스 프로젝트가 완료되었다고 판단하려면
   SQL 파일의 존재보다 __그 SQL이 실제로 실행되어 검증 쿼리를 통과하는지__ 이 중요하다.
 
2. Seed 데이터의 목적은 단순히 화면을 채우는 것이 아니라
   __경계값과 예외 상황(중복 예매, 만료된 예매 등)을 재현해서 제약 조건이 실제로 동작하는지 확인하는 것__ 이다.
 
3. 실패 테스트가 필요한 이유는
   __DB가 잘못된 데이터를 실제로 거부하는지 확인해야, 제약 조건이 있는 척만 하고 있지 않다는 것을 알 수 있기__ 때문이다.
 
4. 요구사항과 프로젝트 결정을 구분해야 하는 이유는
   __요구사항은 반드시 지켜야 하는 목표이고, 프로젝트 결정은 그 목표를 달성하기 위해 내가 선택한 방법이므로,
   나중에 방법을 바꿔야 할 때 목표까지 흔들리지 않게 하기__ 위해서다.
 
5. 내가 만든 개인 프로젝트에서 가장 먼저 추가 확인해야 할 정책은
   __동시에 여러 사용자가 같은 좌석을 예매하려고 할 때(동시성 문제) 어떻게 막을 것인가__ 이다.
```

---

# 16. 제출 체크리스트

- [X] `chapter07_answer.md`를 본인 저장소에 만들었다.
- [X] 시작 환경과 현재 DB를 확인했다.
- [X] 프로젝트 포함/제외 범위를 설명했다.
- [X] 요구사항/결정/미확정 질문을 구분했다.
- [X] 네 테이블의 한 행 의미와 관계를 설명했다.
- [X] `01_course_project_schema.sql`을 실행하고 결과를 확인했다.
- [X] `02_course_project_seed.sql`의 기준 상태를 확인했다.
- [X] `03_course_project_changes.sql` 전후 상태를 비교했다.
- [X] `04_course_project_validation.sql` PASS를 확인했다.
- [X] 허용 경계값 1개 이상을 확인했다.
- [X] 실패 테스트 2개 이상을 한 구간씩 실행했다.
- [X] 실패 후 validation을 다시 실행했다.
- [X] 개인 프로젝트 요구사항 8개 이상을 작성했다.
- [X] 프로젝트 결정 3개 이상과 미확정 질문 3개 이상을 작성했다.
- [X] 개인 프로젝트 ERD를 작성했다.
- [X] 검증 가능한 완료 기준 6개 이상을 작성했다.
- [X] AI 제안을 수용/수정/보류/거절로 구분했다.
- [X] 핵심 캡처는 3~4장 정도로 정리했다.
- [X] 캡처에 비밀번호나 개인정보가 없다.
- [X] GitHub 웹에서 Markdown과 이미지가 정상적으로 보인다.
- [X] 최종 파일을 commit/push했다.

---

# 17. LMS 제출 URL

아래 형식의 **본인 GitHub 파일 URL**을 LMS에 제출합니다.

```text
https://github.com/<본인-GitHub-ID>/<본인-저장소>/blob/main/assignments/chapter07/chapter07_answer.md
```

내 제출 URL:

```text

```

> 저장소 메인 URL, 교수자 템플릿 URL, Raw URL이 아니라 **작성 완료된 본인 `chapter07_answer.md` 파일 화면 URL**을 제출합니다.