# SELECT ... FOR UPDATE

## 1. SELECT
- MySQL에서의 SELECT은 기본적으로 잠금 없는 일관된 읽기로 동작한다
  - MySQL은 데이터 변경 시 변경 전 버전의 레코드를 Undo라는 공간에 백업한 뒤, 데이터를 변경한다
  - 변경되는 도중에 다른 세션에서 변경중인 데이터를 읽고자 하면 MySQL은 변경중인 데이터 파일의 레코드가 아닌, 언두 영역에 백업된 레코드를 읽어서 반환한다
  - 이 과정에서 SELECT 쿼리가 어떤 잠금을 걸지도 않고, 다른 세션의 잠금에 제약을 가하지도 않기 때문에 이를 잠금 없는 일관된 읽기, 즉 Non-Locking Consistence Read(MVCC)로 부른다
- 또한, 격리 수준에 따라 반환되는 레코드의 버전이 달라진다
  - Read Commited에서는 가장 최근에 커밋된 데이터를 반환한다
  - Repeatable Read에서는 트랜잭션이 시작된 시점 직전에 커밋된 데이터를 반환한다
    - 같은 트랜잭션 내에서 동일한 읽기를 보장한다!


## 2. SELECT ... FOR UPDATE
- SELECT FOR ... [UPDATE | SHARE] 는 잠금을 걸어야 하므로 격리 수준과 무관하게 항상 최근에 커밋된 데이터를 반환한다
  - Exclusive 락(이하 X락)을 먼저 걸고 레코드를 읽는다
  - 따라서 다른 트랜잭션의 SELECT ... FOR UPDATE는 동일 트랜잭션에 대해 X락을 걸지 못하므로 대기한다 
  - SELECT ... FOR UPDATE는 레코드를 변경시키진 않으므로 AUTO COMMIT=false로 두고 트랜잭션을 시작(BEGIN)하는 방식으로 사용해야 효력이 있다
- UPDATE의 조건에 넣어서 X락을 획득할 수 있으므로 꼭 필요한 상황이 아니면 사용을 지양해야 한다
  - 트랜잭션 내에서 afftected_rows 변수를 통해 대상 건이 존재하는지 여부를 확인할 수 있다


## 3. SELECT ... FOR UPDATE 튜닝
- 업데이트 대상이 존재할 확률이 낮은 경우, WHERE 절에 추가 조건을 명시하여 필터링하면 레코드 잠금을 적게 걸 수 있다
- 이떄, 레코드에 잠금이 걸리지 않도록 하기 위해서는 두 가지 조건을 충족해야 한다
  - transactional_isolation=READ-COMMITTED
  - binlog_format=MIXED | ROW
  - https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html


## 4. SELECT ... FOR SHARE
- 부모-자식 관계를 갖는 테이블에서 부모 테이블의 레코드를 확인한 이후에 자식 테이블에 인서트하는 로직 처리에서 사용이 가능하다
- 부모 테이블의 레코드에 대해 Shared 락(이하 S락)을 걸어서 삭제하거나 변경하지 못하도록 방지한다
- 그러나 SELECT ... FOR SHARE 이후에 읽기 잠금을 건 레코드를 업데이트하게 된다면 FOR UPDATE를 쓰는 게 낫다 
  - FOR SHARE를 쓰면 S락을 획득한 상태에서 x락으로 업그레이드 해야 한다
  - 이를 락 업그레이드라고 하는데, Dead Lock을 유발하기 쉬운 패턴이므로 지양해야 한다
- SELECT 이후 해당 레코드에 대해 업데이트가 실행될 확률이 매우 낮다면 SELECT FOR SHARE를, 그렇지 않다면 SELECT FOR UPDATE를 사용하는 것이 좋다


## 5. JPA Optimistic Lock과 Pessimistic Lock
- MySQL에서 잠금 없이 레코드를 업데이트하는 것이 불가능하기 때문에 Optimistic Lock은 없다
- JPA의 Optimistic Lock과 Pessimistic Lock은 MySQL의 기능이라기보다는 트랜잭션 내에서 어떤 SQL 문장을 사용하느냐에 따라 달라지는 것이다
### 1) Optimistic Lock (변경 시점에 잠금)
```sql
BEGIN;
SELECT * FROM account WHERE id=1;
...
UPDATE account SET balance=?, version=2
WHERE id=1 AND version=1;
COMMIT;
```
- SELECT 실행 시점에는 잠금을 걸지 않고 업데이트 시점에 잠금을 건다
- 다른 트랜잭션과 동일 레코드에 대해 동시에 변경할 가능성이 낮을 거라는 가정 하에서 사용한다
- 레코드의 버전을 할당하여, 원하는 않는 UPDATE 실행을 방지한다
- 충돌 발생 시 AfftectedLowCounter가 0인 것을 확인하고 JPA가 OptimisiticLockingFailureException을 던진다

### 2) Pessimistic Lock (읽는 시점에 잠금 - 미리 충돌 예상)
```sql
BEGIN;
SELECT * FROM account WHERE id=1 FOR UPDATE;
...
UPDATE account SET balance=? WHERE id=1;
COMMIT;
```
- 먼저 잠금을 걸고 업데이트를 한다
- 다른 트랜잭션과 동일 레코드에 대해 동시에 변경할 가능성이 높을 거라는 가정 하에서 사용한다
