# 데드락

## 1. MySQL 데드락 감지
- DeadLock Detection Thread가 모든 트랜잭션이 획득 또는 대기하고 있는 잠금 Graph를 계속 감시
- 동시 트랜잭션이 많은 경우, DeadLock 체크 작업으로 인한 대기 발생
  - 이런 이유로 DeadLock Detection Thread를 비활성화하는 케이스도 있음
- 만약 비활성화 한다면 어떻게 될까? (innodb_deadlock_detect = OFF)
  - 기본적으로 타임아웃이 존재함 (innodb_lock_wait_timeout은 기본 50초)
  - 만약 DeadLock Detection Thread를 비활성화한다면 2~3초로 조정하는 것을 권장

## 2. MySQL 데드락 처리
- MySQL은 롤백이 쉬운 트랜잭션을 Victim trx로 선정
  - 롤백이 쉬운 트랜잭션 == Undo 레코드가 적은 트랜잭션
- Victim으로 선정된 트랜잭션은 강제 롤백 처리
- 남은 트랜잭션은 정상 처리
- 이런 이유로 배치 작업과 서비스 쿼리가 경합하면 항상 배치 프로세스가 살아남게 됨
  - 배치 작업이 대상 건수가 많으므로 Undo 레코드가 더 많기 때문에

## 3. MySQL 데드릭 정리
- DeadLock은 회피할 수 있는 경우도 있지만, 회피할 수 없는 경우가 더 많음
- DeadLock이 발생했다고 해서 UniqueKey나 PK 등의 인덱스를 삭제할 수는 없음
  - UniqueKey는 일반 인덱스보다 성능적인 이득은 거의 없으므로 데드락 빈도를 높이므로 모델링 시점에 최소화 하는 게 좋음
- DeadLock의 발생 빈도와 서비스 영향도에 따라서 로깅. 재처리 등의 적절한 처리가 필요함
