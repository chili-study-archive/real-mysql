# 페이징 쿼리 작성

## 1. 페이징 쿼리를 작성하는 방법
- DB 서버에서 제공하는 LIMIT & OFFSET 구문을 사용하여 페이징 쿼리를 작성하면 DB에 많은 부하를 일으킴
- 이는 DBMS에서 OFFSET을 지정한다 하더라도, 순차적으로 레코드를 읽기 때문임
- 따라서 LIMIT & OFFSET 구문을 사용하면 쿼리 실행 횟수가 늘어날수록 읽는 데이터도 많아지고 응답 시간도 길어짐

## 2. 범위 기반 방식
- 특정 날짜 기간이나 숫자 범위로 나눠서 데이터를 조회
- WHERE 절에서 조회 범위 지정, LIMIT절은 사용하지 않음
- 주로 배치 작업 등에서 테이블의 전체 데이터를 일정한 날짜/숫자 범위로 나눠서 조회할 때 사용
    - 여러 번 쿼리를 나눠 사용하더라도 쿼리 형태는 동일함
```SQL
-- id 기준 쿼리
SELECT *
FROM users
WHERE id > 0 AND id <= 5000
-- 날짜 기준 쿼리
SELECT *
FROM payments
WHERE finished_at >= '2022-03-01' AND finished_at < '2022-03-02'
```

## 3. 데이터 개수 기반 방식
- 지정된 데이터 건수만큼 결과 데이터를 반환
- 배치보다는 주로 애플리케이션에서 많이 사용되는 방식으로, ORDER BY & LIMIT절이 사용됨
    - 1회차 쿼리와 N회차 쿼리의 형태가 다름
    - WHERE절의 조건 타입에 따라 N회차 실행 시 쿼리의 형태도 제각기 달라질 수 있음

### 1) 동등 조건 사용 쿼리
```sql
-- 인덱스는 (user_id, id)
-- 1회차
SELECT *
FROM payments
WHERE user_id = ?
ORDER BY id
LIMIT 30
-- N회차
SELECT *
FROM payments
WHERE user_id = ? AND id > {이전 데이터의 마지막 id 값}
ORDER BY id
LIMIT 30
```
- ORDER BY절에는 각각의 데이터를 식별할 수 있는 식별자 컬럼(PK 등)이 반드시 포함되어야 함

### 2-1) 범위 조건 사용 쿼리 (1회차)
```sql
-- 인덱스는 (finished_at, id)
-- 1회차
SELECT *
FROM payments
WHERE finished_at >= '{시작 날짜}'
    AND finished_at < '{종료 날짜}'
ORDER BY finished_at, id
LIMIT 5
```
- ORDER BY절에 범위 조건 컬럼(finished_at)이 포함되어 있음
- id 컬럼만 명시하면 id 기준으로 다시 정렬하는 작업이 추가됨
- finished_at 컬럼을 선두에 명시하면 (finishee_at, id) 인덱스를 타서 정렬 작업 없이 순차적으로 데이터 읽기가 가능해짐

### 2-2) 범위 조건 사용 쿼리 (N회차)
```sql
-- N회차1 (id가 auto increment X)
SELECT *
FROM payments
WHERE ((finished_at = '{마지막 날짜}' AND id > '{마지막 id}'
    OR (finished_at > '{마지막 날짜}' AND finished_at < '{종료 날짜}'))
ORDER BY finished_at, id
LIMIT 5
-- N회차2 (id가 auto icrement O)
-- 형태 변화 없이 WHERE 절에 식별자 컬럼 조건만 추가해도 문제없는 케이스
SELECT 8
FROM payments
WHERE finished_at >= '{시작 날짜}'
  AND finished_at < '{종료 날짜}'
  AND id > '{마지막 id}'
ORDER BY finished_at, id
LIMIT 5
```
- 범위조건 컬럼과 식별자 컬럼의 순서가 다른 경우에는 1회차와 N회차의 쿼리 형태가 달라진다
- 범위조건 컬럼과 식별자 컬럼의 순서가 같은 경우에는 1회차와 N회차의 쿼리 형태가 같다 (auto increment)
