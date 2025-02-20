# 데드락
## 데드락 예시(1)
- A -> B에게 100원 송금
- B -> A에게 500원 송금

아래 순서로 트랜잭션이 처리됨
- `TX-1> UPDATE ... SET amount=amount-100 WHERE user_id = 'A';`
- `TX-2> UPDATE ... SET amount=amount-500 WHERE user_id = 'B';`
- `TX-1> UPDATE ... SET amount=amount+100 WHERE user_id = 'B';`
- `TX-2> UPDATE ... SET amount=amount+500 WHERE user_id = 'A';`

-> 1, 2번 UPDATE 실행 후 TX-1이 user_id = 'A'에 대한 배타락을, TX-2이 user_id = 'B'에 대한 배타락을 가지게 됨
    - 이후 3, 4번 UPDATE에서 서로가 가진 락을 기다리게되어 데드락 발생함

#### 회피하는 법?
- `-` 연산 후 `+` 연산을 처리하던 방식 대신 `user_id` 순서대로 처리
- `Index(user_id)` 순서대로 잠금 실행 시, Lock Wait은 발생하지만 데드락은 발생하지 않음

## 데드락 예시(2)
- 3개 트랜잭션이 순서대로 실행
    - `TX-1> BEGIN; DELETE FROM tab WHERE pk=2;`
    - `TX-2> BEGIN; INSERT INTO tab(pk) VALUES(2);`
    - `TX-3> BEGIN; INSERT INTO tab(pk) VALUES(2);`
    - `TX-1> COMMIT;`

- 데드락 발생 원인?
    - TX-1이 pk=2 레코드에 배타락 획득
    - TX-2와 TX-3은 INSERT 시 중복된 레코드에 대해 공유락이 필요해 대기
    - TX-1의 COMMIT과 동시에 TX-2와 TX-3은 공유락 획득
    - TX-2와 TX-3은 PK=2 레코드에 대해 동시에 배타락 획득 대기
        - TX-2와 TX-3이 모두 공유락을 획득하고 있으므로 배타락 획득 불가

- 의문 사항
    - 왜 공유락을 먼저 걸고 배타락을 걸어야하는가?
    - 이미 삭제된 레코드에 대해 어떻게 락을 걸 수 있는가?
- MySQL 서버의 잠금 구현 방법
    - PK는 1개의 유니크한 값만 허용
        - UNIQUE 제약 보장을 위해 DML은 레코드 존재 시 공유락 필요
    - 레코드가 삭제되더라도 필요시까지 Deletion-Mark만 설정
        - Deletion-Mark가 적용되어도 스토리지 엔진에선 여전히 유효한 레코드로 인식되므로 락이 걸릴 수 있음

## MySQL 데드락 감지
- 트랜잭션이 공유/배타락을 걸때마다 메모리에 Graph 구조체를 만듦
- DeadLock detection thread가 모든 트랜잭션이 획득/대기 중인 잠금 Graph를 계속 감시하며 데드락 발생여부를 확인함
- 동시 트랜잭션이 많은 경우, 데드락 체크 작업으로 인한 대기가 발생
    - DeadLock detection thread를 비활성화하기도 함
        - 구글 일부 서비스에서 차용 (비즈니스 로직과 잠금이 매우 간단하여 데드락 발생률이 매우 낮고, DeadLock detection thread으로 인한 성능 저하가 더욱 치명적)
        - `innodb_deadlock_detect=OFF`
            - `innodb_lock_wait_timeout`
                - 기본 값은 50초
                - 2~3초 정도로 조정 검토 필요

## MySQL의 데드락 처리
- MySQL은 롤백이 쉬운 트랜잭션을 Victim으로 선정
    - 롤백이 쉽다? = Undo 레코드가 적은 트랜잭션
- Victim의 트랜잭션은 강제 롤백처리
- 남은 트랜잭션은 정상 처리
- 일반적으로 배치 작업과 서비스 쿼리가 경합하면 배치 작업이 살아남을 가능성이 큼

## MySQL 데드락 해석의 어려움
- 동일 SQL 문장이라도, 항상 동일한 잠금을 사용하진 않음
- 레코드 뿐 아니라, 갭락, 넥스트 키락 등 또한 잠금의 대상
- 데드락 시점의 모든 트랜잭션을 로깅하진 않음
- 사용하는 잠금이나 상황에 따라 중간중간 잠금이 해제되기도 함
    - 각 잠금의 라이프사이클이 다름 (기본은 트랜잭션 단위)
    - 찰나의 시간차이로 데드락이 발생하거나 하지 않을 수 있음
        - 데드락을 재현하는 것이 매우 어려움
- 잠금의 대상은 모든 인덱스 키 (Cluster & Secondary & Foreign 모두 해당)

-> MySQL의 데드락을 100% 파악하는 것은 불가능에 가까움

> 모든 데드락 발생 가능성을 없애기 위해 지나치게 많은 노력과 시간을 투자하는 것이 불필요할 수 있음

## 데드락에 대한 생각
- 많은 오해
    - 프로그램 코드 오류로 데드락이 발생
    - 개발 능력이나 DB 문제로 데드락이 발생
    - 데드락이 발생하면 프로그램 코드 개선 필요

- 그러나,
    - 데드락 회피가 가능할 수 있지만 그렇지 않는 경우가 더 많음
    - 데드락이 발생했다고 해서 UK, PK를 삭제할 수 없음
        - 가능하다면 모델링 시점에서 최소화하는 노력이 필요
            - UK는 성능상의 이점은 거의 없는 반면에 데드락 발생 가능성을 증가시키는 경향이 있음
    - 데드락의 발생 빈도와 서비스 영향도에 따라 무시도 가능 (로깅 및 별도 재처리)
    - 프로그램 코드에서의 트랜잭션 재처리
        - `if SqlState == "40001" -> retry`
    - Retry 코드를 넣었다고 해서 코드 품질이 낮아지는 것은 아님