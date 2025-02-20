# DBMS 활용 (배치 처리 주의사항)

## 1. 대용량 작업에 대한 생각의 차이
- 개발자
  - 최대한 굵고 짧게
  - 동시에 많은 쓰레드로 빠르게 처리 완료
- DBA
  - 가능한 가볍게 (+ 가능한 짧게)
  - 소수의 쓰레드로 최소의 DBMS 자원 소모

## 2. DBMS 서버
DBMS 서버의 CPU 사용률이 튀면 다른 쿼리에 영향을 미치게 된다.
- DBMS 서버는 공유 자원임을 명심하자
  - 배치에서 과도한 자원 점유 시, 실시간 서비스의 처리에 문제가 생길 수 있음
- DBMS 서버의 처리 용량에 맞게 동시성 제어가 필요하다
  - 코어가 4개(vCPU=4)인 DB 서버에서 쿼리 4개가 동시에 실행되면 다른 서비스 쿼리 처리는 지연될 가능성이 높다
- 인프라 환경별로 DBMS 서버 처리 사양이 다르다
  - 온-프레미스 환경
    - 비교적 높은 사양의 서버 사용
  - 클라우드 환경 (Private & Public)
    - 다양한 사양의 서버 사용 가능
    - 대부분 vCPU=2개 이하의 낮은 사양 DBMS 서버로 구축

## 3. Long Transaction & Query
- Idle Long Transaction
  - auto_commit=OFF 상태 또는 명시적 트랜잭션(BIGIN TRANSACTION) 사용 시에 BIGIN 이후 또는 쿼리 실행 이후 대기 상태로 남은 트랜잭션 (COMMIT 또는 ROLLBACK 실행 전)
- Active Long Transaction
  - auto_commit 모드 무관하게 오랜 시간동안 실행되는 트랜잭션
- MySQL은 고립수준(REPEATABLE READ)을 보장하기 위해 undo log를 활용함
  - undo log는 일정 시간이 지나면 제거됨
  - 그러나 오래된 트랜잭션은 오래된 undo log를 제거하지 못하게 만듦
  - undo log가 많이 쌓이면 많은 메모리 사용과 Disk I/O를 유발함
- DBMS 버전에 따라 영향도가 차이남
  - MySQL Community version
    - Primary/Replica가 물리적으로 연결되어 있지 않음
    - Replica의 Long Transaction은 Primary에 영향을 미치지 않음 
  - Aurora MySQL
    - 공유 스토리지를 사용 (= Primary/Replica가 물리적으로 연결되어 있음)
    - Primary/Replica가 서로간에 Long Transaction에 의한 영향을 받음

## 4. DBMS 부하 격리
- 격리된 배치 서버 운영
  - [OLTP & OLAP 용도](https://aws.amazon.com/ko/compare/the-difference-between-olap-and-oltp/)별로 DBMS 서버를 구축
  - 클라우드 환경의 Custom Endpoint를 활용하여 서비스 격리
    - 특정 서비스의 요청을 특정 DB 인스터로 전달하라는 의미!
    - API 서비스, 백그라운드 잡(배치 작업), 분석 쿼리 등을 분리하여 부하를 나눌 수 있음
      - `api-db.mycompany.com` → API 트래픽을 위한 DB 
      - `batch-db.mycompany.com` → 배치 작업을 위한 DB 
      - `analytics-db.mycompany.com` → 분석 쿼리를 위한 DB
- 트랜잭션 제어
  - 작은 트랜잭션 단위로 나누어서 작업 수행 (by 페이징/청크 처리)
