# SQL 문장의 가독성 향상
## 가독성?
  - 단순히 읽기 좋다는 것뿐 아니라, 쿼리가 **어떠한 목적**을 가지고 작성되었는지 보기 좋게 표현하는 것

## 가독성의 중요성
- 작성 의도를 쉽게 이해 가능
- 커뮤니케이션 비용이 줄고 업무효율이 상승
- 문제 원인을 빠르게 찾을 수 있고 실수가 감소함
- 유지보수에도 용이함

## 가독성 향상하기
(1) DISTINCT를 함수처럼 사용하는 형태를 지양
- Distinct는 함수가 아니며 괄호와 함께 사용해도 그렇지 않은 경우와 결과가 동일함
- 괄호 사용 시 오해의 여지가 있으므로 괄호없는 형태로 사용하는 것을 권고

```sql
# BAD CASE
SELECT DISTINCT(fd1), fd2
...;

SELECT DISTINCT (fd1), (fd2)
...;

# BETTER CASE
SELECT DISTINCT fd1, fd2
...;

```

(2) LEFT JOIN 사용 방법 준수
- LEFT JOIN 사용 시 드리븐 테이블에 대한 조건을 WHERE절에 명시하는 경우 INNER JOIN을 사용한 것과 동일한 결과가 출력
- LEFT JOIN 사용 시엔 드리븐 테이블 조건을 ON절에 명시
- LEFT JOIN은 필요한 경우에마 ㄴ사용
  - 1:1 관계로 LEFT JOIN하면서 드라이빙 테이블에 속한 컬럼만 SELECT하거나, COUNT 쿼리를 실행하는 경우엔 LEFT JOIN 제거

(3) ORDER BY절 없이 LIMIT n, m 문법 사용 지양
- 쿼리에서 ORDER BY 없이 LIMIT이 사용되는 경우 어떤 의도로 작성된 것인지 파악 어려움
- 불필요한 LIMIT이면 제거하거나, 페이지네이션을 위한 것이라면 반드시 ORDER BY절을 명시

(4) 그룹핑시엔 FULL GROUP BY 형태로 사용
FULL GROUP BY란?
- GROUP BY절을 사용한 쿼리에서 SELECT절에 GROUP BY 절에 명시된 컬럼 or 집계함수 컬럼만 명시하는 것

```sql
# BAD CASE
# GROUP BY절에 명시되지 않은, 집계함수도 아닌 fd2 컬럼을 사용함
SELECT fd1, fd2, COUNT(*)
FROM tab
GROUP BY fd1;

# BETTER CASE
SELECT fd1, SUM(fd2), COUNT(*)
FROM tab
GROUP BY fd1
```

-> 예상치 못한 결과값이 나올 수 있고, 의도가 불분명하게 보일 수 있음
- SQL MODE에 FULL GROUP BY MODE를 지정하면 FULL GROUP BY 형태의 쿼리를 강제할 수 있음(기본값으론 지정되어있음)

(5) AND/OR 조건 함께 사용 시 반드시 괄호 명시
- SQL에서 AND 연산자는 OR 연산자보다 우선순위 높음
- 괄호가 없는 경우엔 AND 연산자를 우선해서 처리함
- 가독성을 위해 AND/OR을 함께 사용하는 경우 의도에 맞게 괄호를 반드시 명시해서 사용해야함

(6) 데이터 건수 조회는 COUNT(*) 사용
- 카운트 쿼리 수행 시 아래와 같이 다양하게 실행하기도 함
  - COUNT(*), COUNT(1), SUM(1), COUNT(pk_col), COUNT(not_null_col)
- COUNT 함수의 인자로 특정 컬럼이나 1과 같은 상수값을 사용하는 경우 쿼리의 가독성이 떨어지고 작성자의 의도를 명확히 알기 어려움
- 전체 건수가 필요한 경우엔 COUNT(*) 사용 권장

## 더 알아볼 것
- FULL GROUP BY?
  - 여태껏 이렇게 사용하지 않은 케이스가 더러 있었던 것 같음.
  - GROUP BY를 특정 컬럼을 기준으로 중복된 값을 없애고자 할 때도 사용했는데, 그런 케이스에는 어떻게 쿼리를 수정할 수 있을지?