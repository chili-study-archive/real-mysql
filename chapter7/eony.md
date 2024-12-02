# SELECT ... FOR UPDATE

## SELECT (REPEATABLE-READ)
- Non-Locking consistence read (MVCC): 잠금 없는 일관된 읽기
    - SELECT시 레코드 단위로 공유락 / 배타락을 걸면 동시처리 성능이 매우 떨어질 수 있음
    - 따라서 레코드 조회 시 대상 레코드 잠금 없이도 일관된 읽기를 제공함
    - HOW?
        - 데이터 변경 시 항상 변경 전 레코드를 UNDO공간에 백업
            - 변경되는 도중 다른 세션에서 해당 레코드를 읽으려하면, 실제 데이터값이 아닌 UNDO 공간의 데이터를 읽도록 함
            - 순수 SELECT 쿼리는 잠금을 걸지도 않고, 다른 쿼리의 잠금에 의한 영향을 받지도 않음
    - But, 격리수준에 따라서 반환 값이 달라질 수도 있음
        - READ COMMITTED - 가장 최근에 커밋된 데이터를 반환
        - REPEATABLE READ - 트랜잭션이 시작된 시점 이전의 데이터를 반환

## SELECT ... FOR UPDATE
- 격리 수준과 무관하게 항상 SELECT가 실행되는 시점의 최신 커밋된 데이터를 조회
    - SELECT ... FOR UPDATE / SHARE 모두 해당
- 레코드에 대한 배타락(Exclusive Lock) 획득
    - 레코드 변경없이 락을 거는 것이기 때문에 당연히 TRANSACTION 내에서 처리해야만 효과 있음 (오토커밋이면 락 걸었다가 그냥 풀어버림)
- 남용하지 않고 굳이 필요치 않은 경우라면 제거하는 것이 좋음
    - 레코드 락을 지나치게 오래 점유할 가능성이 있기 때문

_SELECT ... FOR UPDATE 튜닝?_
- SELECT ... FOR UPDATE시 WHERE 조건에 가능한 세부적인 필터링 조건 명시하기
    - 락을 거는 레코드의 범위를 줄이기 위함

## SELECT ... FOR SHARE
- SELECT ... FOR SHARE = S-Lock
    - USECASE
        - 부모 테이블의 레코드 삭제 방지
        - 부모 테이블의 SELECT와 자식 테이블의 INSERT 시점 사이에 부모 삭제 방지
- But, SELECT ... FOR SHARE 이후 UPDATE 혹은 DELETE가 필요한 경우 FOR SHARE보단 FOR UPDATE로 거는 것이 더 유리함
    - 내부적으로 S-Lock에서 X-Lock으로 업그레이드하는 작업이 일어남
    - 이 과정에서 데드락 발생 가능성이 매우 높아짐

## JPA Optimistic VS Pessimistic
MySQL의 SELECT ... FOR SHARE / UPDATE와 JPA의 Optimistic, Pessimistic
- 용어를 혼동하지 말기!
- JPA의 Optimistic과 Pessimistic은 트랜잭션 내에서 쿼리를 어떻게 날릴 것인지와 더 연관되어 있음 (잠금 자체와는 연관 X)
    - 그저 Lost Update Anomaly를 방지하기 위한 방법 중 하나임

#### (1) Optimistic Lock (변경 시점에 잠금)
```sql
BEGIN;
SELECT * FROM user WHERE id = 1; # 아무런 잠금 걸지 않음
...
UPDATE user SET address = ? WHERE id = 1;
COMMIT;
```
- MySQL에서는 Optimisitc Lock이란 것은 있을 수 없음
    - 사실상 모든 업데이트 / 삭제 작업은 락이 걸리므로 Pessimistic으로 처리된다고 말할 수는 있으나, 구분하는 것이 의미가 있진 않음

_구현 방법_
- 레코드의 버전을 할당하여 원하지 않는 UPDATE 실행을 방지
```sql
UPDATE account
SET balance = balance - 150, version = 2 # version 조건 추가
WHERE id = 1 and version = 1; # 다른 트랜잭션이 이미 실행되었다면 version은 1이 아닐 것이므로 쿼리는 아무런 영향도 주지 못함
```
- 해당 방식은 MySQL 서버가 제공하는 잠금 기법은 아니며, affected row가 없더라도 아무런 에러도 뱉지 않음 (JPA에선 관련 예외 뱉음)
- 충돌 가능성이 적다고 가정하는 방식.

#### (2) Pessimistic Lock (읽는 시점에 잠금 - 미리 충돌 예상)
```sql
BEGIN;
SELECT * FROM user WHERE id = 1 FOR UPDATE; # X-Lock 잠금 획득
...
UPDATE user SET address = ? WHERE id = 1;
COMMIT;
```

_구현 방법_
- MySQL의 SELECT ... FOR UPDATE를 사용하는 방식
- 에러 발생 확률 낮음
- 동시성 처리는 약간 떨어질수도 있음.

___

## 정리해보면 좋을 것
_트랜잭션 격리수준 관련해 참고하면 좋을 자료 (이미 유명하긴하지만 공유)_
- https://mangkyu.tistory.com/299

_트랜잭션 격리수준에 따른 Lock Release 조건_
- https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html
        - binlog_format=MIXED | ROW