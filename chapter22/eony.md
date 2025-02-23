# 커넥션 관리
## Connection 메모리 사용량
- 타 DBMS(오라클, PostgreSql)와 다르게 MySQL에선 프로세스 기반이 아닌 스레드 기반으로 작동함
    - 커넥션 당 메모리 사용량이 상대적으로 낮은 편
- 일반적으로 커넥션 당 16KB ~ 3MB 메모리 사용
    - 낮은 용량이긴하지만 커넥션 개수가 많아지면 주의 필요
    - 공유 커넥션 풀을 사용하는 경우 특히 주의
- CPU 사용량 증가

## Max Connection 설정
- Server-side Max(max_connections)
    - MySQL: 최대 8천개 ~ 1만개
    - PostgreSQL: 최대 3천~5천개
        - 프로세스 별 점유 메모리가 크고 Linux page cache에 의존을 해 메모리 부족 시 성능 영향을 더 많이 받음
        - 미들웨어가 필수적인 편
    - MySQL 서버의 연결 처리 및 스레드는 가볍지만 많으면 부담됨

## Connection Pool 설정
- Client-side(앱 서버) Pool Max(connection)
    - 커넥션 풀 사용 필수
        - 앱 서버가 하나의 풀을 공유하면서 MySQL 서버와의 커넥션을 최소화하는 목적
    - 시작 설정은 Max 20 ~ 30
        - 처음에 높게 잡아두면 낮추는 튜닝이 어려움
    - Min = Max * (70% ~ 90%)
        - Min = Max 설정이 성능상 유리하지만 커넥션 부족 현상을 판단하기가 어려움
            - 트래픽이 많이 들어올 때 앱 서버가 맺고 있는 커넥션 개수의 차이를 보고 커넥션 설정이 적절한지 판단이 가능할 것임
        - MySQL 서버의 커넥션과 
    - 앱 서버가 100개이고, Max가 200이면 총 20K 커넥션이 필요함
        - 현대 쿠버네티스 환경에선 특히 커넥션이 과하게 잡히지 않도록 주의
    - 필요시 미들웨어를 이용한 서버 사이드 커넥션 재활용
- Connect Timeout 설정
    - Connection Open 또는 Connection Pool에서 커넥션을 가져오는 시간
    - 밀리초 이하의 값 설정은 권장하지 않음
        - MySQL 서버와의 연결을 새로 열어야하는 상황에선 1초 미만의 시간은 충분치 않을 수 있음
    - 인프라 여건에 따라 3~10초 정도로 설정
        - Connection 획득에 실패해도 일반적으로 가능한 Fallback 전략이 없음
        - 일시적인 DB 과부하 시 커넥션 요청 및 취소 반복으로 부가적인 자원 소모
        - 커넥션 획득 실패 시, 오류 화면을 보여주는 경우라면 짧은 Timeout 설정도 가능할 것
- Query Timeout 설정
    - 지나치게 짧으면 쿼리 수행 취소 및 동일 쿼리 재시도 시 DB 부담 가중
    - 재요청을 하지 않는 경우라면 짧은 Timeout도 가능
- Idle Timeout 설정
    - 일정 시간동안 아무런 쿼리를 실행하지 않는 유휴 상태의 커넥션을 정리하는 설정
    - Connection Pool 사용시 빈번하게 관련 오류 발생
        - 에러 회피 위해 Idle Connection Timeout을 매우 짧게 설정하기도 함
        - 그러나 에러 회피를 위해 커넥션을 짧게 사용하고 버리는 것인 좋은 방향이 아님
    - 최소 20~30분 이상 설정 권장
    - 에러 회피 목적이라면 Connection Validation 기능 최대한 활용
        - 커넥션 선택 알고리즘은 Round Robin이 적절하다고 생각됨
            - 그렇지 않으면 커넥션 하나가 지나치게 오래 유휴 상태로 머물 수 있음
    - 주요 기능의 경우 Retry 로직 구현 권장됨
        - 어떤 방법을 적용해도 커넥션이 끊기는 상황은 높은 확률로 발생할 수 있음

## Midleware (proxy)
- MySQL Router, ProxySQL, RDS Proxy
    - Server-side connection pool 역할
    - 앱 서버와 DBMS 간의 커넥션 매핑 & 쿼리 라우팅
    - 사용 단점?
        - 운영 비용 추가 발생
        - 이슈 발생 시 원인 분석 트러블슈팅 난이도 증가   
    - 가능하다면 필요한 경우에만 선택
    - AWS 사용한다면 Writer와 Reader 중 선별적으로 적용 검토
- ProxySQL (DNS Cache)
    - MySQL 서버의 도메인 네임을 아이피 주소로 변환한 결과를 일정 기간동안 ProxySQL이 캐싱하여 사용
        - DNS lookup 시간 단축
        - mysql_monitor_local_dns_cache_ttl 값을 통해 캐싱 시간 설정 가능
    - Read Replica(Cluster Endpoint) 적용 시, Read Replica 간 부하 불균형 주의
        - Read Replica에서 설정한 Round Robin 알고리즘이 적용되지 못하고 캐싱된 IP 값이 계속 반환되어 하나의 서버로만 요청이 몰림