# LEFT JOIN 주의사항 & 튜닝

## 1. LEFT JOIN 사용법 및 실행순서
- LEFT JOIN을 잘못 사용하면 INNER JOIN가 완전히 동일하게 동작할 수 있음
- ON절에 조건을 넣으면 user 테이블의 데이터에 대해서 user_coupon 테이블의 데이터를 연결하는 역할을 함
```mysql
-- ON절에 조건이 붙은 케이스
SELECT u.id, u.name, uc.coupon_id, uc.use_yn
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id=u.id AND uc.coupon_id=3
```
- WHERE절에 조건을 넣으면 실제 결과로 반환되는 데이터를 필터링하는 역할을 수행함
- 아래 쿼리의 경우 MySQL에서는 옵티마이저가 자동으로 INNER JOIN으로 바꿔줌
```mysql
-- WHERE절에 조건이 붙은 케이스
SELECT u.id, u.name, uc.coupon_id, uc.use_yn
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id=u.id
WHERE uc.coupon_id=3
```
---

- LEFT JOIN 왼쪽에 있는 테이블을 DRIVING 테이블 또는 OUTER 테이블이라고 함
- LEFT JOIN 오른쪽에 있는 테이블을 DRIVEN 테이블 또는 INNER 테이블이라고 함
```mysql
SELECT u.id, u.name, uc.coupon_id, uc.use_yn
FROM user u -- DRVING TABLE (=OUTER TABLE)
LEFT JOIN user_coupon -- DRIVEN TABLE (=INNER TABLE)
... 
```
- LEFT JOIN을 실제 기준이 되는 DRIVING 테이블을 먼저 읽음
- 반면 INNER JOIN은 조인에 참여하는 테이블들의 교집합 데이터를 반환하므로 LEFT JOIN과 달리 테이블 접근 순서가 고정되지 않고, 옵티마이저의 최적화 방식에 따라 바뀔 수 있음

## 2. COUNT(*) with LEFT JOIN
- LEFT JOIN을 하지 않아도 결과 건수가 동일한 경우엔 불필요한 LEFT JOIN은 제거해서 사용한다
```mysql
-- AS-IS) 결과: 30000건
SELECT COUNT(*)
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id=u.id AND uc.coupon_id=3

-- TO-BE) 결과: 30000건
SELECT COUNT(*)
FROM user u
```
- LEFT JOIN에 의해 결과 건수가 달라지는 경우엔 반드시 LEFT JOIN을 붙여서 사용해야 한다
```mysql
-- 결과: 27000건
SELECT COUNT(*)
FROM user u
LEFT JOIN user_coupon uc ON uc.user_id=u.id
WHERE uc.user_id IS NULL
```
