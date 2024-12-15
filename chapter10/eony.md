# LEFT JOIN 주의사항 & 튜닝

대표적으로 사용되는 JOIN 종류

(1) INNER JOIN
- 두 테이블 간의 교집합

(2) LEFT JOIN
- `SELECT FROM A LEFT JOIN B`시, A는 전체조회, B는 A와 교집합을 가진 데이터만 조회

## LEFT JOIN시 주의사항
(1) LEFT JOIN이 INNER JOIN처럼 작동하는 경우

**아래 두 쿼리는 결과가 다름!**

```sql
SELECT u.id, u.name, uc.coupon_id, uc.ues_yn
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id = u.id AND uc.coupon_id = 3 # 쿠폰 아이디 조건을 ON절에 명시
```
-> 만약 user 테이블 데이터가 30000건이라면, 결과값의 개수는 30000건이 반환됨

```sql
SELECT u.id, u.name, uc.coupon_id, uc.ues_yn
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id = u.id
WHERE uc.coupon_id = 3 # 쿠폰 아이디 조건을 WHERE절에 명시
```
-> coupon_id가 3인건이 1000건이라면 1000건이 반환됨

__이유?__

조건의 위치에 따라 해당 조건이 수행하는 위치가 다름

ON절에 주어지면?
  - user 테이블에 대해 user_coupon 테이블을 연결하는데 사용됨
  - 즉, user 테이블은 데이터개수가 모두 반환되고, 그 중 user_id와 coupon_id 조건에 맞는 user_coupon 데이터만 JOIN됨. (나머지는 모두 NULL로 채워짐)
WHERE절에 주어지면?
  - 실제 결과로 반환된 값을 필터링하는데 사용되었기 때문
  - 해당 쿼리는 MySQL 옵티마이저에서 내부적으로 INNER JOIN으로 바꾸어 실행함

## LEFT JOIN과 INNER JOIN의 내부적 처리 과정 차이
(1) LEFT JOIN
- LEFT JOIN 왼쪽의 테이블(Driving Table or Outer Table)을 먼저 모두 읽어옴
  - 해당 테이블이 기준이 됨

(2) INNER JOIN
- LEFT JOIN 오른쪽 테이블(Driven Table or Inner Table)을 먼저 읽고, 해당 결과값을 바탕으로 JOIN 조건에 해당하는 Driving Table을 읽어옴
  - 항상 그렇다는 것은 아님. 옵티마이저가 판단한 효율적인 방법으로 데이터를 읽어오는 순서가 바뀜


## COUNT(*) with LEFT JOIN

일반적으로 LEFT JOIN 쿼리의 결과값 개수를 가져오고자할 때, 단순히 SELECT 컬럼 값을 COUNT(*)로 변경하는 것이 대다수
- 실제 쿼리를 보면 JOIN이 불필요한 경우가 더러 있다.

EX)
```sql
SELECT count(*)
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id = u.id AND uc.coupon_id = 3;

SELECT count(*)
FROM user u;

# 두 쿼리의 결과값은 사실상 같음.
```

JOIN 쿼리 사용 시 일반적으로 소요시간이 더 길기 때문에, JOIN 사용여부와 별개로 카운트값이 동일한 경우에는 JOIN 없이 사용하는 것이 훨씬 더 효율적임

## 정리
- LEFT JOIN을 사용하고자 한다면 Driven Table 컬럼의 조건은 반드시 ON절에 명시해 사용 (IS NULL 조건은 예외)
- LEFT JOIN과 INNER JOIN은 결과 데이터 및 쿼리 처리 방식이 달라 필요에 맞게 올바르게 사용하는 것이 중요함
  - 조인 쿼리에 INNER / LEFT 등 반드시 명시
- LEFT JOIN 쿼리에서 COUNT 사용 시 LEFT JOIN이 불필요하지 않은지 확인이 필요