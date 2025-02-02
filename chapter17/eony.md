# SELECT ... FOR UPDATE (NOWAIT | SKIP LOCKED)

## SELECT ... FOR UPDATE NOWAIT
- 잠금 대상 레코드가 이미 다른 세션에 의해 잠겨있는 경우, 대기하지 않고 바로 에러를 반환함
- innodb_lock_wait_timeout 옵션을 0으로 설정한 것과 유사한 효과 (기본값 50초)
- 트랜잭션 내에서 NOWAIT 쿼리를 실행하여 에러가 반환되도, 열어둔 트랜잭션은 그대로 유지됨
    - 커밋 / 롤백을 명시해야함!
- 불필요하게 잠금을 오래 기다리지 않음!

## SELECT ... FOR UPDATE SKIP LOCKED
- 잠금 대상 레코드 중 다른 세션에 의해 이미 잠금이 걸려있는 레코드는 스킵, 잠금이 걸려있지 않은 레코드를 잠그고 반환
- 잠금 대상 레코드가 비결정적으 로 정해짐
    - 잠금 대상 레코드에 대한 예측이 어려움
- 레코드가 이미 모두 잠금되어 있으면 **빈 결과**를 반환함 (에러 반환 X)
    - 반환되는 결과 데이터가 없다고 해도, 경우에 따라 Gap-Lock 점유 가능함
- ORDER BY & LIMIT 절과 함께 많이 사용됨
    - ex) 잠금되지 않은 데이터 N개를 가져온다

## NOWAIT & SKIP LOCKED with JOIN
- 1 : N 관계에서 JOIN을 함께 사용 시 SKIP LOCKED가 예상대로 동작하지 않을 수 있음

```sql
SELECT c.coupon_no, e.name
FROM coupon c
INNER JOIN event e ON e.id = c.event_id
WHERE c.event_id = 1
    AND c.is_used = 'N'
ORDER BY c.id
LIMIT 1
FOR UPDATE SKIP LOCKED
```

- event : coupon = 1 : N
- JOIN하는 event 테이블에 대한 잠금이 발생
- 조건에 해당하는 coupon 레코드가 존재하더라도, 1 관계의 event 레코드가 잠금되면 INNER JOIN의 결과값이 없다고 간주됨
- 만약 특정 테이블에 대해서만 잠금하고 싶다면? - OF문 사용 가능
    - `FOR UPDATE OF {coupon} SKIP LOCKED`

# 정리
- NOWAIT / SKIP LOCKED를 통해 SELECT FOR UPDATE만 사용할때보다 더 효율적으로 DB 자원을 사용할 수 있음
- NOWAIT은 잠금 대기 상황이 비정상적인 것으로 간주되는 로직에서 빠른 실패처리를 위해 사용할 수 있음
- SKIP LOCKED는 선착순 쿠폰 발급 / 데이터 큐잉 후 배치잡 처리 등에서 유용하게 사용가능함
- SELECT FOR UPDATE에서 JOIN 사용 시 기본적으로 조인되는 테이블의 레코드까지 잠금이 되지만 OF {table}을 통해 특정 테이블만 잠금되도록하여 불필요한 잠금을 줄일 수 있음