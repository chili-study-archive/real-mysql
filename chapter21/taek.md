# JOIN UPDATE & JOIN DELETE

## 1. UseCase
- 다른 테이블의 컬럼 값을 참조해서 Update / Delete 하고 싶은 경우
- 한 번에 여러 테이블에 대해 Update / Delete 하고 싶은 경우

## 2. Join Update
```mysql
UPDATE product p
INNER JOIN fee_info f ON f.company_id=p.company_id
SET p.fee_amount=(p.price * f.fee_rate)
WHERE p.company_id=1;
```
```mysql
UPDATE product p
INNER JOIN order o ON o.product_id=p.id
SET p.name='Carrot Juice',
    o.product_name='Carrot Juice'
WHERE p.id=1234;
```
- SET절에 업데이트 대상 컬럼을 명시
- LEFT JOIN 등 다른 유형의 JOIN도 사용 가능
---
```mysql
UPDATE user_coupon SET expired_at='2022-09-30'
WHERE coupon_id=1;
UPDATE user_coupon SET expired_at='2023-12-31'
WHERE coupon_id=2;
UPDATE user_coupon SET expired_at='2022-11-30'
WHERE coupon_id=3;
...
```
```mysql
UPDATE user_coupon uc
INNER JOIN (
    VALUES ROWS(1, '2022-09-30'),
    VALUES ROWS(2, '2023-12-31'),
    VALUES ROWS(3, '2022-11-30'),
    ...
) change_coupon (coupon_id, expired_at)
ON change_coupon.coupon_id=uc.coupon_id
SET uc.expired_at=change_coupon.expired_at;
```
- 단건 업데이트 쿼리를 가상 테이블을 활용한 다건 업데이트 쿼리로 바꾸면 성능을 향상시킬 수 있음

## 3. Join Delete
- DELETE ... FROM 절 사이에 삭제 대상 테이블을 명시
- LEFT JOIN 등 다른 유형의 JOIN도 사용 가능

## 4. Join Update & Join Delete 주의사항
- 참조하는 테이블의 데이터에는 읽기 잠금이 발생하여 잠금경합이 발생할 수 있음
- Join Update의 경우 조인된 테이블들의 관계가 1:N일 때 N테이블의 컬럼 값을 1테이블에 업데이트하는 경우 예상과 다르게 처리될 수 있음 (N:M 관계도 마찬가지)
  - 즉, N개의 로우 중 어떤 로우가 선택될지 모른다는 것
- Join Update & Join Delete 쿼리는 단일 Update/Delete 쿼리보다 쿼리 형태가 복잡하므로 반드시 사전에 쿼리 실행계획 학인 필요
