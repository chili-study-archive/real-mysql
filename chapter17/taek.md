# SELECT ... FOR UPDATE [NOWAIT | SKIP LOCKED]

## 1. SELECT ... FOR UPDATE NOWAIT
- 잠금 대상 레코드가 이미 다른 세션에 의해 잠겨있는 경우엔 대기하지 않고 바로 에러를 반환하는 명령이다
- `innodb_lock_wait_timeout` 옵션을 0으로 설정한 것과 유사한 효과다 (default 50)
- 트랜잭션 내에서 해당 명령에 의해 에러가 반환되더라도 트랜잭션은 그대로 유지되므로, 명시적으로 commit or rollback을 수행해야 한다

## 2. SELECT ... FOR UPDATE SKIP LOCKED
- 잠금 대상 레코드 중 다른 세션에 의해 잠겨있는 경우엔 스킵하고, 잠기지 않는 레코드를 잠그고 반환하는 명령이다
- 잠금 대상 레코드가 비결정적으로 정해진다 (=쿼리 실행 시 어떤 레코드가 잠길지 예측하기 어렵다)
- 잠금 대상 레코드들이 모두 잠금이 걸려있으면 빈 결과를 반환한다
    - 빈 결과를 반환하더라도 경우에 따라 Gap Lock을 걸 수 있다
- ORDER BY & LIMIT절과 많이 사용된다

## 3. NOWAIT & SKIP LOCKED with JOIN
- JOIN이 걸리면 락도 같이 걸리는 형태로 동작한다
- 즉, 1:N 관계 조인 시 락을 걸면 1과 N 모두가 잠긴다
- JOIN된 테이블에 대해 모두 잠금을 수행할 필요가 없다면 OF 구문을 통해 실제 잠금이 필요한 테이블의 레코드에 대해서만 잠금을 수행해야 한다 (`FOR UPDATE OF table`)

## 4. 정리
- [NOWAIT | SKIP LOCKED] 옵션을 사용하면 SELECT ... FOR UPDATE보다 동시 처리 성능이 높아진다
- NOWAIT은 잠금 대기가 발생하는 경우가 비즈니스 로직적으로 비정상인 케이스에 사용하기 적합하다 (바로 에러 리턴)
- SKIP LOCKED는 선착순 발급이나, 데이터 큐이이 후 배치잡 처리 등에 사용하기 적합하다 (데이터를 연속적으로 빠르게 읽기)
- SELECT FOR UPDATE 구문을 JOIN과 같이 사용하면 여러 테이블의 레코드에 대해 잠금이 걸리므로 OF 구문을 사용하여 필요한 테이블의 레코드에 대해서만 잠금을 수행해야 한다. 
