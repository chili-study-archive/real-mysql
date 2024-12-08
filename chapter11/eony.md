# PREPARED STATEMENT
- A.K.A Binding Query
- 애플리케이션에서 변수값을 동적으로 바인딩하여 사용하는 쿼리

EX)
```java
pstmt = connection.prepareStatement("SELECT * FROM matt WHERE id = ?");
pstmt.setInt(1, 1234);
rs = pstmt.executeQuery();
```

## 장단점

장점
- SQL Injection 방지
- 쿼리 파싱 비용 감소
  - 변수만 다른 동일한 Prepared Statement 쿼리를 캐싱해 사용함으로써 파싱 비용이 감소함

단점
- 메모리 사용량 증가
  - Prepared Statement 파싱을 위해 사용하는 파싱 트리가 메모리를 잡아먹을 수 있음
  - 첫 번째 실행 시, 2번의 Network round-trip 필요
    - 실제 쿼리 실행 이전에 Prepared Statement 단계가 필요함
    - Prepared Statement -> 실제 쿼리 실행의 총 2단계 소요됨
  - MySQL은 실행 계획까지 캐싱해서 재사용하진 않음 (파싱 트리만 재사용)
  - 캐시된 PreparedStatement는 하나의 **커넥션 내에서만 공유됨**

## PreparedStatement의 비밀 (MySQL에서만 해당)
- Client Side / Server Side로 구분됨
- 모두 SQL-Injection을 막을 수 있음
- JDBC Server Side PreparedStatement는
    - useSErverPrepStmts=TRUE인 경우에만 작동
    - FALSE가 기본값
    - ORM에서는 TRUE로 기본설정되는 경우 많음

## PreparedStatement 실익

```java
// 케이스 (1)
// for문 내에서 PreparedStatement를 계속 생성해 사용하는 경우
// MySQL 클라이언트에선 매번 PreparedStatement를 새로 생성하게 됨 
// - 파싱 트리 캐싱이 되지 않음
// - 매번 PreparedStatement 생성 -> 실제 쿼리 수행의 2단계를 거치게 됨
for (int idx=0; idx<100; idx++) {
    PreparedStatement pstmt = conn.prepareStatement("SELECT ... WHERE id=?");
    pstmt.setInt(1, targetUserIds[idx]);
    pstmt.executeQuery();
}


// 케이스 (2)
// PreparedStatement를 for문 밖에서 정의하고 재사용하는 경우
// for문 외부에서 생성한 PreparedStatement의 파싱트리를 캐싱하여 사용
PreparedStatement pstmt = conn.prepareStatement("SELECT ... WHERE id=?");

for (int idx=0; idx<100; idx++) {
    pstmt.setInt(1, targetUserIds[idx]);
    pstmt.executeQuery();
}
```

## PreparedStatement vs ConnectionPool
- MySQL의 PreparedStatement는
  - 하나의 Connection 내에서만 공유됨
    - Re-parsing 비용 최소화
  - But, 전체 커넥션이 5000개, 쿼리 패턴이 100개라면 5000 * 100 = 500000개의 파싱트리 객체가 MySQL 서버에 저장되어야함
    - max_prepared_stmt_count=16382(default) 값으로 최대 캐시 카운트 제한 가능
- 쿼리의 복잡도에 따라 매우 복잡하면 PreparedStatement가 도움이 되지만, 단순하면 그 장점이 경감됨.
- 메모리 사용량 vs CPU 사용량
  - AWS RDS는 매우 소규모 서버들을 사용
  - 일반적으로 메모리 적음
    - 오히려 CPU를 더 굴리는게 나을 수 있음

## 정리
- MySQL 서버에서는 Server-Side PreparedStatement가 부작용이 심한 경우가 많음
  - Client-Side PreparedStatement는 권장함. (동일하게 SQL-Injection 방지 효과가 있으나 Server-Side보다 부담이 없음)
- Server-Side PreparedStatement가
  - 예상만큼 성능을 크게 높여주진 못함
  - 메모리 소비를 꽤 많이 유발함 (OOM 유발)
  - max_prepared_stmt_count 부족 시, 쿼리 파싱 경감 효과 떨어짐

## 더 알아볼 것
- Client-Side PreparedStatement?