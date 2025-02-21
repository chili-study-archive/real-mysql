# DBMS 활용 (배치 처리 주의사항)
## 대용량 작업
- 개발자와 DBA의 생각이 다름
    - 개발자
        - 최대한 굵고 짧게
        - 동시에 많은 스레드로 빠르게 처리
            -> DBMS 자원을 다 차지해버려서 다른 작업이 영향을 받음
    - DBA
        - 가능한 가볍고 짧게
        - 소수의 스레드로 최소한의 DBMS 자원 소모
            -> 배치 처리 이외의 다른 쿼리들이 영향받지 않고 정상적으로 실행되기를 원함

## DBMS 서버를 대하는 관점
- DBMS 서버는 공유 자원임
    - 특정 서비스에서 리소스 과점유시 다른 서비스의 처리 지연/실패/장애 유발함
- DBMS 서버의 처리 용량은 다양함
    - 처리 용량 및 스펙에 맞게 동시성 제어가 필요함
    - 처리가 밀린 경우 빠르게 처리하되 세심히 모니터링 및 제어하는 것이 필요
    - 사양이 낮은 DBMS에선 배치 작업 스레드가 늘어나면 매우 쉽게 응답 불능 상태가 유발될 수 있음
        - 문제 발생 시 해결이 어려움
            - CPU 등 자원 사용 급등 원인 분석 필요
            - 장애를 만들기는 쉽지만 장애를 분석하는 것이 매우 소모적인 일임

## DBMS 서버 처리 용량
- 온 프레미스 환경
    - 소수의 표준 사양의 하드웨어 사용하며 높은 사양 서버 사용
- 클라우드 환경
    - 다양한 사양의 서버 사용 가능
        - 응용 프로그램의 처리율 제어를 위한 참고사항으로 사용
    - 대부분 vCPU=2 이하의 낮은 성능을 사용
        - 자원 부족으로 인한 문제가 발생할 확률이 더 높음

## Long Transaction & Query
- Long Transcation
    - Idle long transaction
        - auto_commit = off 상태 또는 명시적 트랜잭션(BEGIN TX) 사용시 발생 가능
        - BEGIN 이후 쿼리 실행 이후 대기 상태로 남은 트랜잭션 (COMMIT 혹은 ROLLBACK 실행이 되지 않은 상태)
    - Active long transcation
        - auto_commit 상태와 무관하게 오랜 시간동안 실행되는 쿼리를 실행하는 경우

> MySQL 아키텍처
- Non-locking consistent read 기능은 MVCC 활용하여 구현
    - UNDO LOG 활용
        - 트랜잭션 격리 수준에 따라 세부적인 사항은 다를 수 있음
    - UNDO LOG는 오래된 것들에 대해 주기적으로 PURGE 되어야함
- UNDO LOG 부작용
    - 트랜잭션이 지나치게 길면 오래된 UNDO LOG에 대한 PURGE를 막을 수 있음
        - Active 트랜잭션들과 연관된 UNDO LOG는 제거할 수 없기 때문
    - 이는 곧 UNDO LOG가 지나치게 쌓이는 것을 유발하고, 많은 메모리 및 DISK IO 초래
    - 자원 사용률이 높아지고 쿼리 성능 저하됨

#### MySQL Community Version과 Aurora MySQL에서 UNDO LOG를 다루는 방식이 다르다!
- MySQL Community Version
    - Long TX는 실행되는 해당 서버에만 영향을 미침
    - 즉, Replica에선 트랜잭션 관련 영향 없이 UNDO LOG가 제때 삭제됨
        - Replica 서버에선 악영향도가 낮음
- Aurora MySQL
    - 공유 스토리지를 사용하는 구조
        - Write/Read Replica가 UNDO LOG 데이터 파일을 공유함
    - Write / Read Replica가 논리적으로 연결
        - 레플리카에서 발생한 Long TX가 Writer 서버에까지 영향을 미침
        - 따라서 Read Replica에서도 Long TX가 발생하지 않도록 주의해야함

## DBMS 부하 격리
- 격리된 배치 서버를 운영하기
    - 배치 작업이 빈번하다면 OLTP & OLAP 용도별 DBMS 서버 구축도 가능
        - Long TX의 경우 주의 필요
    - 클라우드 환경의 Custom Endpoint를 활용하여 서비스 격리도 가능함

- 트랜잭션 제어
    - Aurora MySQL 케이스에서처럼, 적절한 조치를 취하더라도 지나치게 긴 트랜잭션은 발생하지 않도록 제어 필요
    - 작은 트랜잭션 단위로 나누어서 작업 수행

**OLTP/OLAP**
(1) OLTP
    - 운영계 데이터 및 데이터 처리 방법 의미

(2) OLAP
    - 분석계 데이터 및 데이터 처리 방법 의미
    - 통계/집계 등