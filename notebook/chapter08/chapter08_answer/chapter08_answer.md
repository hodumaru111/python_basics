Chapter 08 확장 실습 답안

과제: JOIN과 집계로 서비스 질문에 답하기
제출 방법: LMS에는 본인 GitHub 저장소의 chapter08_answer.md 파일 URL을 제출합니다.


제출 전 주의

이 파일과 캡처 화면에는 실제 비밀번호, 전체 DB 접속 URL, API Key, 개인정보를 기록하지 않습니다.

text
GitHub 계정 또는 별칭: hodumaru
과제 작성일: 2026-09-30
사용한 AI 도구: Claude
1. Chapter 07 기준 상태 확인
text
code/chapter08/00_check_course_project.sql
1-1. 사전 검사 결과
text
검증 메시지: Chapter 08 prerequisite check passed

students 행 수: 3
instructors 행 수: 2
courses 행 수: 3
enrollments 행 수: 5

전체 신청 건수: 5
전체 recorded_amount: 590000
활성 신청 건수: 3
활성 recorded_amount: 340000
취소 제외 신청 건수: 4
취소 제외 recorded_amount: 440000

기준값:

text
students = 3
instructors = 2
courses = 3
enrollments = 5

전체 = 5 / 590000
활성 = 3 / 340000
취소 제외 = 4 / 440000

→ 기준값과 모두 일치합니다.

기준값이 다르면 그대로 진행하면 안 되는 이유
text
Chapter 08의 모든 검산(5건, 440000, 강의 303의 0건, 과대 집계 440000 등)은
Chapter 07 최종 데이터를 전제로 설계되어 있다.
출발 데이터가 다르면 SQL이 틀려서 값이 다른 것인지, 데이터가 달라서 값이 다른 것인지
구분할 수 없다. 이때 실제 결과에 맞춰 기대값을 고치면 검산 자체가 의미를 잃는다.
따라서 먼저 원인을 확인하고, 필요하면 Chapter 07을 복원해 00 검사를 통과시킨 뒤 진행한다.
증거 화면

이미지 표시

2. 업무 질문을 SQL보다 먼저 정의하기
질문 A
text
업무 질문: 수강신청 한 건마다 어떤 학생이 어떤 강의를 신청했고, 강사는 누구인지 보여 주세요.
결과 한 행의 의미: 수강신청(enrollments) 한 건
포함 상태: 신청, 수강중, 완료, 취소 (전체 이력)
제외 상태: 없음
JOIN할 테이블: enrollments, students, courses, instructors
JOIN 경로: enrollments.student_id → students.id
           enrollments.course_id  → courses.id
           courses.instructor_id  → instructors.id
INNER JOIN / LEFT JOIN 선택: INNER JOIN
그 이유: 기준이 신청 행이고, FK NOT NULL + 고아 관계 0건이므로
         모든 신청은 반드시 학생·강의·강사를 가진다. 짝이 없는 행을 남길 이유가 없다.
예상 행 수: 5행 (enrollments 행 수와 같아야 함)
질문 B
text
업무 질문: 강의별로 취소되지 않은 신청 건수와 신청 시 기록 금액 합계를 보여 주세요.
결과 한 행의 의미: 강의(courses) 한 개
포함 상태: 신청, 수강중, 완료
제외 상태: 취소
JOIN할 테이블: courses, enrollments
JOIN 경로: courses.id = enrollments.course_id
집계 대상: COUNT(e.id) — 신청 건수 / SUM(e.recorded_amount) — 기록 금액
예상 결과: 301 = 2건 / 200000, 302 = 2건 / 240000, 303 = 0건 / 0
           강의별 금액 합 = 440000 (취소 제외 전체 금액과 같아야 함)
질문 C
text
업무 질문: 모든 학생을 보여 주고, 취소되지 않은 신청이 몇 건인지 표시해 주세요.
결과 한 행의 의미: 학생(students) 한 명
포함 상태: 신청, 수강중, 완료
제외 상태: 취소
0건인 부모도 보여야 하는가: 예 — 박서연(103)처럼 취소 제외 신청이 없는 학생도 0건으로 표시
NULL을 어떻게 해석할 것인가: LEFT JOIN 후 e.id가 NULL인 행은 "연결된 취소 제외 신청이 없음"을 뜻한다.
                            값이 모르는 상태가 아니라 "0건"으로 해석하고, COUNT(e.id)로 0이 되게 한다.
예상 결과: 3행 — 김민지 2, 이준호 2, 박서연 0
3. INNER JOIN과 다중 JOIN
3-1. 신청 한 건마다 학생 이름과 강의 제목 조회

실행 전 예상:

text
결과 한 행 = 수강신청 한 건
예상 행 수 = 5
JOIN 경로 = enrollments.student_id → students.id, enrollments.course_id → courses.id

내가 실행한 SQL:

sql
SELECT
    e.id AS enrollment_id,
    s.name AS student_name,
    c.title AS course_title,
    e.status
FROM course_project.enrollments AS e
INNER JOIN course_project.students AS s
    ON e.student_id = s.id
INNER JOIN course_project.courses AS c
    ON e.course_id = c.id
ORDER BY e.id;

실제 결과:

text
 enrollment_id | student_name | course_title       | status
 1001          | 김민지       | 데이터베이스 입문  | 완료
 1002          | 김민지       | 정규화 실습        | 신청
 1003          | 이준호       | 데이터베이스 입문  | 수강중
 1004          | 박서연       | 파이썬 데이터 분석 | 취소
 1005          | 이준호       | 정규화 실습        | 신청

실제 행 수: 5
예상과 일치 여부: 일치
학생 이름이 여러 번 보이는 것이 중복 오류가 아닐 수 있는 이유
text
학생과 신청은 1:N 관계다. 김민지는 1001, 1002 두 건을 신청했으므로
"신청 한 건 = 한 행" 기준에서는 이름이 두 번 나오는 것이 정상이다.
enrollment_id는 모두 다르므로 같은 행이 복제된 것이 아니다.
여기서 DISTINCT를 넣으면 오히려 신청 한 건을 잃어버리게 된다.
3-2. 학생·강의·강사까지 연결
text
결과 한 행 = 수강신청 한 건
강사까지 가는 JOIN 경로 = enrollments.course_id = courses.id → courses.instructor_id = instructors.id
sql
SELECT
    e.id AS enrollment_id,
    s.name AS student_name,
    c.title AS course_title,
    i.name AS instructor_name,
    e.status,
    e.recorded_amount,
    e.enrolled_at
FROM course_project.enrollments AS e
JOIN course_project.students AS s
    ON e.student_id = s.id
JOIN course_project.courses AS c
    ON e.course_id = c.id
JOIN course_project.instructors AS i
    ON c.instructor_id = i.id
ORDER BY e.id;

실제 행 수:

text
5행 (강사: 1001·1002·1003·1005 = 문길래, 1004 = 홍길동)
왜 실제 PK/FK 경로를 따라야 하는가
text
enrollments에는 강사 id가 없다. 강사는 "신청한 강의의 담당자"이므로
신청 → 강의 → 강사 순서로만 의미가 정확히 이어진다.
이름 같은 일반 열로 연결하면 동명이인이나 우연히 같은 값끼리 잘못 연결될 수 있고,
DB가 보장하는 관계(FK)가 아니므로 결과를 신뢰할 근거가 없다.
증거 화면

이미지 표시

4. LEFT JOIN과 0건 표현
4-1. 강의별 취소 제외 신청 수

실행 전:

text
결과 한 행 = 강의 한 개
강의 303의 예상 실제 신청 수 = 0
강의 303의 예상 고유 학생 수 = 0
강의 303의 예상 recorded_amount = 0

내 SQL:

sql
SELECT
    c.id AS course_id,
    c.title AS course_title,
    COUNT(e.id) AS non_cancelled_count,
    COUNT(DISTINCT e.student_id) AS student_count,
    COALESCE(SUM(e.recorded_amount), 0) AS non_cancelled_recorded_amount
FROM course_project.courses AS c
LEFT JOIN course_project.enrollments AS e
    ON c.id = e.course_id
   AND e.status <> '취소'
GROUP BY c.id, c.title
ORDER BY c.id;

실제 결과:

text
강의 301: 2건 / 2명 / 200000
강의 302: 2건 / 2명 / 240000
강의 303: 0건 / 0명 / 0

COALESCE(SUM(...), 0)은 "연결된 신청이 없어 SUM이 NULL"인 경우를 업무 의미상 0원으로 표시하려는 것입니다. 행 단위와 JOIN이 올바른 것을 먼저 확인한 뒤 적용했습니다.

4-2. COUNT(*)와 COUNT(e.id) 비교

강의 303 기준:

text
COUNT(*) 결과: 1
COUNT(e.id) 결과: 0
COUNT(DISTINCT e.student_id) 결과: 0
왜 COUNT(*) = 1인데 실제 신청 수는 0일 수 있나요?
text
LEFT JOIN은 오른쪽에 짝이 없어도 왼쪽(강의 303) 행을 하나 남기고,
오른쪽 열은 모두 NULL로 채운다. COUNT(*)는 이 "JOIN 결과 행"을 세므로 1이 된다.
하지만 그 행에는 실제 신청이 없다(e.id = NULL). 즉 1은 강의 행이 남았다는 뜻이지
신청이 1건이라는 뜻이 아니다.
자식 사건 수를 셀 때 COUNT(child.id)가 더 적절한 이유
text
COUNT(열)은 NULL을 세지 않는다. child.id는 PK라서 실제 자식 행이 있으면 절대 NULL이 아니고,
없을 때만 NULL이다. 따라서 COUNT(e.id)는 "실제로 연결된 신청 건수"를 정확히 센다.
0건 부모를 남기려고 LEFT JOIN을 썼다면 COUNT(*)는 0건을 1건으로 부풀린다.
5. LEFT JOIN에서 ON과 WHERE 조건 비교
5-1. 조건을 ON에 둔 경우
sql
SELECT
    s.id,
    s.name,
    COUNT(e.id) AS non_cancelled_count
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
   AND e.status <> '취소'
GROUP BY s.id, s.name
ORDER BY s.id;
text
결과 학생 수: 3 (김민지 2, 이준호 2, 박서연 0)
박서연 포함 여부: 포함 (0건)
5-2. 조건을 WHERE에 둔 경우
sql
SELECT
    s.id,
    s.name,
    COUNT(e.id) AS non_cancelled_count
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
WHERE e.status <> '취소'
GROUP BY s.id, s.name
ORDER BY s.id;
text
결과 학생 수: 2 (김민지 2, 이준호 2)
박서연 포함 여부: 제외됨
5-3. 차이 설명
text
ON 조건이 LEFT JOIN의 오른쪽 연결 대상을 제한하는 방식:
  ON은 "어떤 신청을 짝으로 붙일 것인가"를 정한다. 취소 신청은 짝 후보에서 빠지지만,
  짝이 하나도 없는 학생도 LEFT JOIN 규칙에 따라 NULL 확장 행으로 남는다.

WHERE 조건이 JOIN 이후 결과 행을 제거하는 방식:
  WHERE는 JOIN이 끝난 결과에서 행을 걸러낸다. NULL <> '취소'는 TRUE가 아니라 NULL(알 수 없음)이므로
  NULL 확장 행도 함께 제거되어 사실상 INNER JOIN처럼 동작한다.

이번 사례에서 ON = 3명, WHERE = 2명이 되는 이유:
  박서연의 신청은 1004(취소) 하나뿐이다.
  ON: 1004가 짝에서 빠짐 → 박서연 + NULL 행이 남음 → 3명
  WHERE: 1004 행은 '취소'라서 제거 → 박서연 행이 하나도 남지 않음 → 2명
6. 신청이 없는 학생 찾기 — 두 방법 비교
방법 1. LEFT JOIN ... IS NULL
sql
SELECT
    s.id,
    s.name,
    s.email
FROM course_project.students AS s
LEFT JOIN course_project.enrollments AS e
    ON s.id = e.student_id
   AND e.status <> '취소'
WHERE e.id IS NULL;
방법 2. NOT EXISTS
sql
SELECT
    s.id,
    s.name,
    s.email
FROM course_project.students AS s
WHERE NOT EXISTS (
    SELECT 1
    FROM course_project.enrollments AS e
    WHERE e.student_id = s.id
      AND e.status <> '취소'
);
text
방법 1 결과: 1명
방법 2 결과: 1명
두 결과가 같은가: 같다
찾아진 학생: 103 박서연
두 방식의 공통 의미를 자신의 말로 설명
text
둘 다 "취소 제외 신청이 하나도 연결되지 않은 학생"을 찾는다.
방법 1은 붙여 본 뒤 짝이 없어서 NULL로 남은 학생을 고르고,
방법 2는 조건에 맞는 신청이 존재하는지 물어보고 "없다"인 학생만 고른다.
표현은 다르지만 같은 업무 질문(anti-join)이다.
주의: 방법 1에서 status 조건을 WHERE로 옮기면 5장과 같은 이유로 결과가 달라진다.
7. 기본 집계 검산
분석 범위	예상 건수	실제 건수	예상 금액	실제 금액	일치?
전체 신청	5	5	590000	590000	O
활성 신청	3	3	340000	340000	O
취소 제외	4	4	440000	440000	O
취소	1	1	150000	150000	O

범위 정의: 활성 = status IN ('신청', '수강중'), 취소 제외 = status <> '취소'

교차 검산: 취소 제외 4 + 취소 1 = 전체 5, 440000 + 150000 = 590000

7-1. 전체 평균 recorded_amount
text
예상 평균: 118000.00
실제 평균: 118000.00   (590000 / 5)
7-2. 취소 제외 평균
text
예상 평균: 110000.00
실제 평균: 110000.00   (440000 / 4)
recorded_amount를 실제 회계 매출이라고 부르면 안 되는 이유
text
recorded_amount는 "신청 당시 신청 행에 기록한 금액"이다.
결제가 실제로 성공했는지, 환불·할인이 반영됐는지는 이 테이블이 알려 주지 않는다.
예를 들어 취소된 1004의 150000도 전체 합계에 들어 있다.
숫자 계산이 맞아도 "매출"이라는 이름을 붙이면 업무 의미가 틀린 보고가 된다.
8. GROUP BY, HAVING, FILTER
8-1. 상태별 신청 건수
sql
SELECT
    status,
    COUNT(*) AS enrollment_count,
    SUM(recorded_amount) AS total_recorded_amount
FROM course_project.enrollments
GROUP BY status
ORDER BY CASE status
    WHEN '신청' THEN 1
    WHEN '수강중' THEN 2
    WHEN '완료' THEN 3
    WHEN '취소' THEN 4
    ELSE 99
END;

결과:

text
신청: 2 (240000)
수강중: 1 (100000)
완료: 1 (100000)
취소: 1 (150000)
상태별 합계: 5 (590000)
상태별 건수 합이 전체 신청 5건과 맞는지 검산
text
2 + 1 + 1 + 1 = 5 → 전체 신청 5건과 일치
240000 + 100000 + 100000 + 150000 = 590000 → 전체 금액과 일치
FILTER로 한 번에 구한 값도 같다: 신청 2, 수강중 1, 활성 3, 완료 1, 취소 1

FILTER 확인용 SQL:

sql
SELECT
    COUNT(*) AS total_count,
    COUNT(*) FILTER (WHERE status IN ('신청', '수강중')) AS active_enrollment_count,
    COUNT(*) FILTER (WHERE status = '완료') AS completed_count,
    COUNT(*) FILTER (WHERE status = '취소') AS cancelled_count,
    SUM(recorded_amount) FILTER (WHERE status <> '취소') AS non_cancelled_recorded_amount
FROM course_project.enrollments;
-- 결과: 5 / 3 / 1 / 1 / 440000
8-2. 강의별 취소 제외 신청 수와 금액
sql
SELECT
    c.id AS course_id,
    c.title AS course_title,
    COUNT(e.id) AS non_cancelled_count,
    COUNT(DISTINCT e.student_id) AS student_count,
    COALESCE(SUM(e.recorded_amount), 0) AS non_cancelled_recorded_amount
FROM course_project.courses AS c
LEFT JOIN course_project.enrollments AS e
    ON c.id = e.course_id
   AND e.status <> '취소'
GROUP BY c.id, c.title
ORDER BY c.id;
text
강의 301: 2건 / 2명 / 200000
강의 302: 2건 / 2명 / 240000
강의 303: 0건 / 0명 / 0
강의별 합계를 다시 더한 값: 200000 + 240000 + 0 = 440000
전체 취소 제외 기준 440000과 일치 여부: 일치
8-3. HAVING 사용
sql
SELECT
    c.id,
    c.title,
    COUNT(e.id) AS enrollment_count
FROM course_project.courses AS c
JOIN course_project.enrollments AS e
    ON c.id = e.course_id
WHERE e.status <> '취소'
GROUP BY c.id, c.title
HAVING COUNT(e.id) >= 2
ORDER BY c.id;
text
예상 강의 수: 2
실제 강의 수: 2 (301 데이터베이스 입문, 302 정규화 실습)
text
WHERE  → GROUP BY 전에 개별 신청 행을 거른다 (취소 신청 제외)
HAVING → GROUP BY 후 강의 그룹 결과를 거른다 (2건 이상인 강의만)
여기서는 "2건 이상인 강의"만 보면 되므로 0건 강의를 남길 필요가 없어 INNER JOIN을 사용했다.
9. 과대 집계 오류 직접 관찰

강사 201(문길래)의 강의 가격 합계를 구한다고 가정합니다.

9-1. 신청까지 JOIN해서 잘못 집계한 결과
sql
SELECT
    i.id AS instructor_id,
    i.name AS instructor_name,
    SUM(c.price) AS wrong_course_price_sum
FROM course_project.instructors AS i
JOIN course_project.courses AS c
    ON i.id = c.instructor_id
JOIN course_project.enrollments AS e
    ON c.id = e.course_id
GROUP BY i.id, i.name
ORDER BY i.id;
text
강사 201 잘못된 가격 합계: 440000
(참고: 강사 202는 신청이 1건뿐이라 우연히 150000으로 맞아 보이지만, 쿼리 기준은 똑같이 잘못됨)
9-2. 강의 수준에서 올바르게 집계
sql
SELECT
    i.id AS instructor_id,
    i.name AS instructor_name,
    COALESCE(SUM(c.price), 0) AS course_price_sum
FROM course_project.instructors AS i
LEFT JOIN course_project.courses AS c
    ON i.id = c.instructor_id
GROUP BY i.id, i.name
ORDER BY i.id;
text
강사 201 올바른 가격 합계: 220000 (301 = 100000 + 302 = 120000)
9-3. 왜 두 결과가 달라졌나요?
text
JOIN 전 강의 행 수: 강사 201의 강의 = 2행 (301, 302)

JOIN 후 강의가 반복된 이유:
  강의와 신청은 1:N이다. 301에는 신청 1001·1003, 302에는 1002·1005가 있어
  JOIN 후 강의 행이 신청 수만큼 늘어나 4행이 된다.

  course | enrollment | price
  301    | 1001       | 100000
  301    | 1003       | 100000
  302    | 1002       | 120000
  302    | 1005       | 120000

SUM이 무엇을 반복해서 더했는가:
  강의 가격(c.price)을 신청 수만큼 더했다.
  100000×2 + 120000×2 = 440000 → 정답 220000의 두 배.
  결과 행 단위가 "신청"으로 바뀌었는데 "강의" 단위 값을 합산한 것이 원인이다.
SUM(DISTINCT c.price)를 일반적인 해결책으로 사용하면 안 되는 이유
text
DISTINCT는 "같은 강의"가 아니라 "같은 가격 값"을 하나로 합친다.
서로 다른 두 강의가 우연히 둘 다 100000원이면 하나가 사라져 합계가 줄어든다.
지금 데이터에서 우연히 맞더라도 원인(불필요한 JOIN으로 행 단위가 바뀜)을 숨길 뿐이다.
질문이 강의 가격 합계라면 신청 테이블을 JOIN하지 않고 강의 수준에서 계산하는 것이 올바르다.
증거 화면

이미지 표시

10. 상세 결과 ↔ 집계 결과 교차 검산
text
선택한 course_id: 301
강의 제목: 데이터베이스 입문
검산 범위: 취소 제외 (status <> '취소')
10-1. 상세 신청 행 조회
sql
SELECT
    e.id,
    e.student_id,
    e.status,
    e.recorded_amount
FROM course_project.enrollments AS e
WHERE e.course_id = 301
  AND e.status <> '취소'
ORDER BY e.id;
text
 id   | student_id | status | recorded_amount
 1001 | 101        | 완료   | 100000
 1003 | 102        | 수강중 | 100000

상세 행 수: 2
상세 recorded_amount를 직접 더한 값: 100000 + 100000 = 200000
10-2. 집계 SQL
sql
SELECT
    COUNT(e.id) AS enrollment_count,
    SUM(e.recorded_amount) AS recorded_amount_sum
FROM course_project.enrollments AS e
WHERE e.course_id = 301
  AND e.status <> '취소';
text
집계 건수: 2
집계 금액: 200000
10-3. 비교
text
상세 행 수와 COUNT 결과 일치 여부: 일치 (2 = 2)
상세 금액 합과 SUM 결과 일치 여부: 일치 (200000 = 200000)
8-2 강의별 집계의 301 값(2건 / 200000)과도 일치
다르다면 원인: 해당 없음
  (다를 경우 확인 순서: 결과 한 행 기준 → JOIN 경로 → 상태 범위 → ON/WHERE 위치
   → COUNT 대상 → 중복 행 → GROUP BY 수준)
11. 자동 완료 게이트
text
code/chapter08/03_join_aggregation_validation.sql
text
최종 검증 메시지: Chapter 08 join and aggregation validation passed

기대 메시지:

text
Chapter 08 join and aggregation validation passed
증거 화면

이미지 표시

자동 검증이 통과했어도 사람이 SQL 의미를 설명해야 하는 이유
text
자동 검증은 "지금 데이터에서 숫자가 기대값과 같은가"만 확인한다.
업무 질문을 올바르게 정의했는지, recorded_amount를 매출로 잘못 부르지 않았는지,
앞으로 데이터가 바뀌어도 같은 의미가 유지되는지는 증명하지 못한다.
예를 들어 강사 202의 잘못된 가격 합계는 현재 데이터에서 우연히 정답과 같다.
숫자가 맞아도 쿼리 의미가 틀릴 수 있으므로 사람이 행 단위와 범위를 설명할 수 있어야 한다.
12. 개인 프로젝트 업무 질문 3개 만들기

【직접】 아래는 "스터디 모임 관리" 예시입니다. Chapter 07 개인 프로젝트의 테이블·상태값으로 바꿔 작성하세요.
예시 구조: members(id, name), studies(id, leader_id → members.id, title, capacity), study_members(id, study_id → studies.id, member_id → members.id, status, joined_at)
status 값: 참여중, 대기, 탈퇴

질문 ID	업무 질문	결과 한 행	포함/제외 범위	JOIN 경로	집계 대상	검산 방법
P08-Q01	스터디별 현재 참여 인원은? 참여자가 0명인 스터디도 보여야 한다.	스터디 1개	포함: 참여중 / 제외: 대기, 탈퇴	studies LEFT JOIN study_members (ON에 상태 조건)	COUNT(sm.id)	스터디 1개를 골라 상세 참여 행 수와 비교, 스터디별 합 = 전체 참여중 건수
P08-Q02	회원별로 참여 중인 스터디 수는? 참여 0건 회원도 보여야 한다.	회원 1명	포함: 참여중 / 제외: 대기, 탈퇴	members LEFT JOIN study_members (ON에 상태 조건)	COUNT(sm.id)	회원별 합 = 전체 참여중 건수, 0건 회원은 NOT EXISTS 결과와 비교
P08-Q03	정원(capacity)보다 참여 인원이 적어 모집이 필요한 스터디는?	스터디 1개	포함: 참여중 / 제외: 대기, 탈퇴	studies LEFT JOIN study_members	COUNT(sm.id) 후 HAVING으로 capacity와 비교	Q01 결과에서 capacity − 인원 > 0인 스터디를 직접 골라 비교
12-1. 질문 1 SQL
sql
-- 결과 한 행 = 스터디 1개 / 0명 스터디 유지 → LEFT JOIN, 상태 조건은 ON에
SELECT
    st.id AS study_id,
    st.title,
    COUNT(sm.id) AS active_member_count
FROM studies AS st
LEFT JOIN study_members AS sm
    ON st.id = sm.study_id
   AND sm.status = '참여중'
GROUP BY st.id, st.title
ORDER BY st.id;

-- 검산용 상세 SQL
SELECT sm.id, sm.member_id, sm.status
FROM study_members AS sm
WHERE sm.study_id = :선택한_study_id
  AND sm.status = '참여중';
text
예상 결과: 스터디 수만큼 행, 참여자 없는 스터디는 0
실제 결과: 미실행 【직접】
검산 결과: 미실행 【직접】
12-2. 질문 2 SQL
sql
-- 결과 한 행 = 회원 1명 / 0건 회원 유지 → LEFT JOIN
SELECT
    m.id AS member_id,
    m.name,
    COUNT(sm.id) AS active_study_count
FROM members AS m
LEFT JOIN study_members AS sm
    ON m.id = sm.member_id
   AND sm.status = '참여중'
GROUP BY m.id, m.name
ORDER BY m.id;

-- 검산: 회원별 합계 = 전체 참여중 건수
SELECT COUNT(*) FROM study_members WHERE status = '참여중';
text
예상 결과: 회원 수만큼 행, 회원별 합 = 전체 참여중 건수
실제 결과: 미실행 【직접】
검산 결과: 미실행 【직접】
12-3. 질문 3 SQL
sql
-- 결과 한 행 = 스터디 1개 / 참여자 0명 스터디도 모집 대상이므로 LEFT JOIN
SELECT
    st.id AS study_id,
    st.title,
    st.capacity,
    COUNT(sm.id) AS active_member_count,
    st.capacity - COUNT(sm.id) AS open_seats
FROM studies AS st
LEFT JOIN study_members AS sm
    ON st.id = sm.study_id
   AND sm.status = '참여중'
GROUP BY st.id, st.title, st.capacity
HAVING COUNT(sm.id) < st.capacity
ORDER BY open_seats DESC;
text
예상 결과: 참여 인원 < 정원인 스터디만 표시 (0명 스터디 포함)
실제 결과: 미실행 【직접】
검산 결과: 미실행 【직접】 — Q01 결과와 capacity를 비교해 직접 골라 본 목록과 같은지 확인

개인 프로젝트 테이블을 아직 PostgreSQL로 구현하지 않아 SQL 초안과 예상 검산 방법까지만 작성했습니다. (미실행)

13. AI를 JOIN·집계 리뷰어로 활용
13-1. 내가 AI에게 전달한 질문
text
다음 업무 질문과 SQL을 검토해 주세요.
SQL이 실행된다는 이유로 정답이라고 판단하지 말아 주세요.
먼저 다음을 확인해 주세요.
1. 결과 한 행의 의미
2. 포함/제외 상태
3. PK/FK JOIN 경로
4. INNER JOIN과 LEFT JOIN 선택
5. COUNT 대상
6. NULL과 0건 해석
7. 여러 1:N JOIN으로 인한 과대 집계 위험
8. 상세 결과로 검산할 방법
근거 없는 DISTINCT나 COALESCE를 먼저 제안하지 말고,
질문의 의미와 행 단위가 잘못되었는지 먼저 확인해 주세요.

[업무 질문] 스터디별 현재 참여 인원 (0명 스터디 포함)
[테이블 구조] studies, study_members(study_id FK, member_id FK, status)
[내 SQL] 12-1 질문 1 SQL
13-2. 내 SQL과 AI SQL 비교

【직접】 AI 제안 칸은 본인이 실제로 받은 AI 답변에 맞게 수정하세요.

검토 항목	내 판단/SQL	AI 제안	최종 선택	이유
결과 한 행	스터디 1개	스터디 1개가 맞음	수용	질문이 "스터디별"이므로
상태 범위	참여중만	대기를 "현재 인원"에 넣을지 업무 정의 확인 필요	수정(정의 명시)	대기는 확정 인원이 아니므로 제외하고 문서에 기록
JOIN 경로	studies.id = study_members.study_id	동일	수용	FK 경로와 일치
INNER/LEFT 선택	LEFT JOIN	LEFT JOIN, 상태 조건은 ON에	수용	WHERE에 두면 0명 스터디가 사라짐
COUNT 대상	COUNT(sm.id)	COUNT(DISTINCT sm.member_id)도 고려	수정	같은 회원 중복 참여를 UNIQUE 제약으로 막고 있다면 COUNT(sm.id)로 충분, 없다면 제약 추가가 먼저
과대 집계 위험	1:N JOIN 하나라 낮음	출석 등 다른 1:N 테이블을 추가하면 위험	수용	추가 JOIN 시 먼저 서브쿼리로 집계 후 JOIN
상세 검산 방법	스터디 1개 상세 행 수 비교	스터디별 합 = 전체 참여중 건수도 확인	수용	두 방향 검산이 더 강함
AI가 만든 SQL에서 발견한 위험 또는 확인한 점
text
【직접】 예시:
- AI가 처음에 COUNT(*)를 쓰면 0명 스터디가 1명으로 표시되는 문제가 있다.
- 상태 조건을 WHERE에 둔 버전은 실행은 되지만 0명 스터디가 결과에서 사라진다.
- DISTINCT를 제안받았지만, 중복 행의 원인이 데이터 제약 부재인지 JOIN 때문인지 먼저 확인해야 한다.
AI SQL이 실행 성공했다고 바로 정답이라고 할 수 없는 이유
text
문법 오류가 없다는 것은 PostgreSQL이 문장을 해석했다는 뜻일 뿐이다.
9장의 440000처럼 행 단위가 잘못된 SQL도 오류 없이 실행된다.
정답 여부는 결과 한 행의 의미, 상태 범위, JOIN 경로가 질문과 맞는지,
그리고 상세 결과와의 교차 검산이 일치하는지로 판단해야 한다.
14. 최종 성찰

【직접】 본인의 말로 다듬으세요.

text
1. JOIN SQL을 작성하기 전에 가장 먼저 정해야 하는 것은
   결과 한 행이 무엇을 뜻하는지와 어떤 상태를 포함·제외할지 이다.

2. LEFT JOIN에서 COUNT(*) 대신 COUNT(child.id)를 검토해야 하는 이유는
   자식이 없어도 부모 행이 NULL과 함께 남아 COUNT(*)가 0건을 1건으로 세기 때문 이다.

3. ON과 WHERE 조건 위치가 중요한 이유는
   ON은 연결할 대상을 고르고 WHERE는 JOIN 결과 행을 지워서, LEFT JOIN의 0건 부모가 남는지 사라지는지가 달라지기 때문 이다.

4. 여러 1:N 관계를 JOIN한 뒤 바로 SUM하면 위험한 이유는
   부모 값이 자식 수만큼 반복된 행에서 합산되어 440000처럼 과대 집계되기 때문 이다.

5. 집계 결과를 신뢰하기 전에 가장 좋은 검산 방법 중 하나는
   같은 조건의 상세 행을 직접 조회해 행 수·금액 합을 COUNT·SUM 결과와 비교하는 것 이다.
15. 제출 체크리스트
 chapter08_answer.md를 본인 저장소에 만들었다.
 00_check_course_project.sql이 통과했다.
 업무 질문마다 결과 한 행을 먼저 정의했다.
 INNER JOIN과 다중 JOIN을 실행했다.
 LEFT JOIN에서 0건 부모를 확인했다.
 COUNT(*)와 COUNT(child.id) 차이를 설명했다.
 ON과 WHERE 조건 위치 차이를 직접 비교했다.
 LEFT JOIN ... IS NULL과 NOT EXISTS를 비교했다.
 전체/활성/취소 제외 기준값을 직접 검산했다.
 GROUP BY, HAVING을 사용했다.
 과대 집계 오류와 수정 결과를 비교했다.
 상세 결과와 집계 결과를 교차 검산했다.
 03_join_aggregation_validation.sql이 통과했다.
 개인 프로젝트 업무 질문 3개를 작성했다.
 AI SQL을 실행 성공 여부가 아니라 의미와 검산 결과로 평가했다.
 핵심 캡처는 3~4장 정도만 사용했다. 【직접】
 비밀번호·개인정보·비밀정보가 없다. 【직접】
 GitHub 웹에서 Markdown과 이미지가 정상적으로 보인다. 【직접】
 최종 답안을 commit/push했다. 【직접】
16. LMS 제출 URL
text
https://github.com/<본인-GitHub-ID>/<본인-저장소>/blob/main/assignments/chapte