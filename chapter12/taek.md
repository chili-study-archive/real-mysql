# SQL 문장의 가독성 향상

## 1. DISTINCT를 함수처럼 사용하는 형태 지양
**Bad Case**
```mysql
SELECT DISTINCT(fd1), fd2
FROM tab
WHERE ...

SELECT DISTINCT (fd1), (fd2)
FROM tab
WHERE ...
```
**Good Case**
```mysql
SELECT DISTINCT fd1, fd2
FROM tab
WHERE ...
```
- DISTINCT는 함수가 아니며, 괄호와 함께 사용하더라도 차이가 없음(위 쿼리는 모두 같은 결과임)
- 괄호 사용 시 오해의 여지가 있으므로 괄호 없이 사용을 권장

## 2. LEFT JOIN 사용 방법 준수
- LEFT JOIN 사용 시 Driven 테이블에 대한 조인 조건은 ON절에 명시해야 한다 ([10장 참고](../chapter10/taek.md))
- LEFT JOIN은 꼭 필요한 경우에만 사용

## 3. ORDER BY절 없이 LIMIT n,m 문법 사용 지양
- 쿼리에서 ORDER BY 없이 LIMIT이 사용되는 경우 어떤 의도로 작성된 건지 파악이 어려움
- 불필요한 LIMIT이라면 제거하거나, 만약 페이지네이션 처리를 위한 것이라면 반드시 ORDER BY절을 명시해서 사용 ([4장 참고](../chapter4/taek.md))

## 4. FULL GROUP BY 형태로 사용
- FULL GROUP BY? GROUP BY절을 사용한 쿼리에서 SELECT절에 GROUP BY절에 명시된 컬럼이나 집계 함수만을 사용하는 것 (MySQL 기본값)
- 정확한 데이터를 얻고, 분명한 의도를 전달하기 위해선 FULL GROUP BY를 써야 한다
- SELECT 절 내 불필요한 컬럼은 제거한다 

**Bad Case**
```mysql
SELECT fd1, fd2, COUNT(*)
FROM tab
GROUP BY fd1
```
**Good Case**
```mysql
SELECT fd1, SUM(fd2), COUNT(*)
FROM tab
GROUP BY fd1
```

## 5. AND/OR 조건 함께 사용 시엔 반드시 괄호 명시
- SQL에서 AND 연산자는 OR 연산자보다 우선 순위가 높아서, 괄호가 없는 경우 AND 연산자를 우선해서 처리
- 따라서 AND/OR 조건을 쿼리에서 함께 사용하는 경우 의도에 맞게 괄호를 명시해서 사용해야 한다

## 6. 데이터 건수 조회는 COUNT(*) 사용
- 아래와 같은 쿼리는 모두 동일한 데이터 건수를 반환함
  - `COUNT(*)`, `COUNT(1)`, `SUM(1)`, `COUNT(pk_col)`, `COUNT(not_null_col)`
- COUNT 함수의 인자로 특정 컬럼이나 상수를 사용하기보다는 asterisk(*)를 사용하는 게 가독성이 좋음
