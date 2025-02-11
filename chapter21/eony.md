# JOIN UPDATE & JOIN DELETE
## USECASE
- 다른 테이블의 컬럼 값을 참조해서 UPDATE / DELETE 하고 싶은 경우
- 한 번에 여러 테이블에 대해 UPDATE / DELETE 하고 싶은 경우
- 조회 후 업데이트가 아닌 쿼리 하나로 처리하고 싶은 경우

## JOIN UPDATE
```sql
# SET절에서 다른 테이블 참조 가능
UPDATE product p
INNER JOIN fee_info f ON f.company_id=p.company_id
SET p.fee_amount=(p.price * f.fee_rate)
WHERE p.company_id=1;

# 참조하는 테이블은 모두 업데이트 가능
UPDATE product p
INNER JOIN order o ON o.product_id=p.id
SET p.name='new name',
    o.product_name='new_name'
WHERE p.id=1234;
```

- SET에 업데이트 대상 컬럼 명시
- 쿼리에서 참조하고 있는 테이블들 중 전체 혹은 일부에 대해 컬럼 값 업데이트 가능함
- LEFT JOIN 등 다른 유형의 JOIN도 사용 가능함
- `VALUES ROW()`를 사용해 여러 개의 단일 업데이트 쿼리를 하나의 쿼리로 처리 가능
    - `VALUES ROW()`로 데이터 배열을 만들어 업데이트 대상 테이블과 JOIN하는 방식

## JOIN DELETE
```sql
# WHERE 조건에 조인되는 테이블 참조
DELETE  ul
FROM user_log ul
INNER JOIN user u ON u.user_id=ul.user_id
WHERE u.last_active < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);

# 조인하여 여러 개의 테이블 삭제
DELETE p, c
FROM product p
INNER JOIN category c ON c.id=p.category_id
WHERE c.name='Fruits'
```
- DELETE ... FROM 절 사이에 데이터 삭제 대상 테이블 목록 명시
- 쿼리에서 참조하고 있는 테이블 전체 혹은 일부 삭제 가능
- LEFT JOIN 등 다른 유형의 JOIN 사용 가능함

## Using Optimizer Hint
- JOIN UPDATE/DELETE 시 여러 테이블에 접근하므로 SELECT절에서와 마찬가지로 옵티마이저가 동작함
    - 옵티마이저 힌트 설정 가능함
        - `DELETE /*+ JOIN_FIXED_ORDER() */ p, c ...`
        - `straight_join` 힌트와 동일하게 동작
        - 고정된 순서로 조인하도록 강제함

## 주의사항
- 참조하는 테이블 데이터에 대한 읽기 잠금(공유락) 발생하므로 잠금경합 발생 가능
- Join Update 시 테이블의 관계가 1:N일 때, N 테이블의 컬럼 값을 1 테이블에 업데이트하는 경우 예상과는 다르게 처리될 수 있음(N:M도 마찬가지)
    - N 관계의 데이터가 다 다르면, 그 중 어떤 값이 1 관계의 데이터에 업데이트 되어야하는지?
- 일반적인 UPDATE/DELETE 쿼리보다 복잡하므로 사전에 쿼리 실행계획을 반드시 확인 필요
    - 예상한 순서대로 조인이 실행되는지?
    - 인덱스가 없어 지나치게 많은 레코드를 스캔해 대량의 읽기 잠금을 유발하진 않는지?