# Prepared Statement

## 1. Prepared Statement란? 
- Java의 JDBC 드라이버를 이용해서 프로그래밍할 때 Prepared Statement 객체를 이용해서 쿼리를 하는 것 (Binding Query라고도 불림)
```java
pstmt = connection.prepareStatement("SELCT * FROM matt id=?");
pstmt.setInt(1, 1234);
rs = pstmt.executeQuery();
```
- 값이 바인딩 되는 변수에는 물음표를 넣어 SQL 쿼리를 작성하고 (PREPARE 단계)
- 그 이후에 변수의 값을 바인딩해서 쿼리를 실행하는 형태 (EXECUTE 단계)
- 장점
  - SQL 인젝션 방지
  - 캐싱을 통한 쿼리 파싱 비용 감소
- 단점
  - 캐싱을 위한 메모리 사용량 증가
  - 첫 번째 실행되는 쿼리의 경우 PREPARE 단계에서 MySQL과의 서버통신이 필요하여 추가적인 Network I/O 발생
  - MySQL에서 실행계획은 캐시되지 않고, Parse Tree만 캐시하는 제약이 있음
  - MySQL에서 캐시된 Prepared Statement는 한 커넥션 내에서만 공유됨

## 2. Prepared Statement의 비밀
- MySQL의 PreparedStatement
  - Client Side Prepared Statement
    - MySQL이 Prepared Statement를 지원하지 않을 때 JDBC의 표준을 맞추기 위한 모방 기능 (실제 Prepared Statement처럼 동작 X)
  - Server Side Prepared Statement
    - 실제 MySQL 서버에서 제공하는 `Prepared Statement` 기능
  - Client/Server Side 모두 다 SQL 인젝션 방어 가능
- JDBC Server Side Prepared Statement는 `userServerPrepStmts=TRUE`인 경우에만 작동
  - 기본값은 `userServerPrepStmts=FALSE`
  - 직접 사용자가 활성화 하지 않는 이상 Server Side Prepared Statement 기능은 사용하지 않게 됨
- ORM에서 TRUE로 기본 설정되어 있는 경우가 많음


## 3. Prepared Statement 실익
**케이스1 - 반복문 안에서 Prepared Statement 함수 호출 + 변수 바인딩 및 쿼리 실행** 
- Prepared Statement 객체 참조를 유지하지 않아서 캐시 사용 불가
- 루프를 돌 때마다 서버 통신이 두 번씩 발생함
```java
for(int i = 0; i < 100; i ++) {
    PreparedStatement p = conn.prepareStatement("SELECT .. WHERE id=?");
    p.setInt(1, targetUserIds[i]);
    p.executeQuery();
}
```

**케이스2 - 반복문 바깥에서 Prepared Statement 함수 한 번 호출 + 반복만 안에서 변수 바인딩 및 쿼리 실행**
- Prepared Statement 객체 참조를 유지하지 않아서 캐시 사용 가능
- 최초 실행 시 서버 통신 두 번 발생 후 루프를 돌 때마다 서버 통신이 한 번씩만 발생함
```java
PreparedStatement p = conn.prepareStatement("SELECT .. WHERE id=?");
for(int i = 0; i < 100; i ++) {
    p.setInt(1, targetUserIds[i]);
    p.executeQuery();
}
```

## 4. PreparedStatement vs Connection Pool
- MySQL에서 Prepared Statement 사용 시 커넥션의 개수를 최소화해야 하며, 동시에 커넥션의 수명 주기를 최대한 길게 해야 한다
  - MySQL 서버의 Prepared Statement는 하나의 커넥션 내에서만 공유됨
  - 기본적으로 MySQL 서버는 대략 16,000개의 Prepared Statement를 캐시할 수 있고, 이 수치를 넘어가면 LRU 패턴에 의해 캐시가 제거됨 (max_prepared_stmt_count=16382)
  - 따라서 커넥션이 계속 재생성되면 MySQL 서버에 캐시되는 데이터의 양도 많아지고(서버의 최대치 내에서), 파싱 비용도 증가함(캐시에서 데이터가 밀려나면 다시 파싱되므로)
  - 적절한 Max Prepared Statement Count 변수를 설정해서 캐시의 양을 늘릴 수 있음
- 쿼리가 매우가 복잡한 경우엔 파싱 비용을 경감시켜주는 Prepared Statement가 도움이 되지만, 단순하면 장점이 경감됨

