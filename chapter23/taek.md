# 테이블 파티셔닝

## 1. 테이블 파티셔닝이란?
- 하나의 테이블을 물리적으로 여러 테이블로 분할해서 데이터를 저장하는 기법
- 사용자는 기존처럼 하나의 테이블로 인식해서 사용함
- 테이블의 특정 컬럼이나 계산식을 기준으로 해서 날짜/숫자 범위나 리스트, 해시 형태 등으로 분할 가능

## 2. 테이블 파티셔닝이 필요한 이유
- 삭제 가능한 이력 데이터들을 효율적으로 관리할 수 있음
  - e.g. 로그성 데이터들이 저장되는 테이블
  - 보관 기간에 따라 일정 기간이 지난 데이터들을 삭제 대신 파티션 드랍으로 처리한다
    - 명령문 하나로 손쉽게 처리 가능
    - 사용된 디스크 공간을 온전히 반환 가능
- 자원 사용 효율 증가 및 쿼리 성능 향상
  - e.g. 게시판과 같이 최근에 저장된 데이터들 위주로 조회하는 케이스
  - 날짜 범위로 파티셔닝하여 각 파티션이 특정 기간의 데이터만을 보유하도록 설정
  - 쿼리가 특정 날짜 범위의 데이터를 요청할 때, MySQL은 조건 범위에 해당하지 않는 파티션은 쿼리 처리에서 자동적으로 제외
  - 이러한 과정을 파티션 프루닝(Partition Pruning)이라고 함
    - 필요한 데이터가 존재하는 테이블의 인덱스만을 메모리에 적재하고 액세스하는 기능
    - 자원 사용 효율 증가와 쿼리 성능 향상에 있어 핵심 기능

## 3. 파티셔닝 타입
```mysql
CREATE TABLE table_name(
    ...
) PARTITION BY [RANGE|RANGE COLUMNS] (...) (
    PARTITION partition_name VALUES LESS THAN (...),
    PARTITION partition_name VALUES LESS THAN (...),
    ...
)
```
- RANGE: 특정 값의 범위로 파티셔닝
- RANGE COLUMNS: +여러 컬럼 값의 조합 가능
```mysql
CREATE TABLE table_name(
    ...
) PARTITION BY [LIST|LIST COLUMNS] (...) (
    PARTITION partition_name VALUES LESS THAN (...),
    PARTITION partition_name VALUES LESS THAN (...),
    ...
)
```
- LIST: 특정 값의 목록으로 파티셔닝
- LIST_COLUMNS: +여러 컬럼 값의 조합 가능
```mysql
CREATE TABLE table_name(
    ...
) PARTITION BY [HASH|LINEAR HASH] (...)
PARTITIONS N;
```
- HASH: 해시함수를 통해 파티셔닝(지정한 표현식 사용 가능)
- LINEAR HASH: 파티션의 추가/삭제에 따른 부하 개선
```mysql
CREATE TABLE table_name(
...
) PARTITION BY [KEY|LINEAR KEY] (...)
PARTITIONS N;
```
- KEY: 해시함수를 통해 파티셔닝(컬럼만 사용 가능)
- LINEAR KEY: 파티션의 추가/삭제에 따른 부하 개선

실무에서는 일정한 범위로 파티션을 분할해서 사용하는 경우가 대다수이므로 RANGE와 RANGE COLUMNS가 주로 쓰인다.

## 4. 파티션 테이블 사용 제약 및 주의사항
- 외래키 및 공간 데이터 타입, 전문검색 인덱스 사용 불가
- 파티션 표현식에 MySQL 내장 함수들을 사용할 수 있으나, 모두 파티션 프루닝 기능을 지원하는 것은 아님
  - TO_DAYS(), TO_SECONDS(), YEAR(), UNIX_TIMESTAMP()만 지원
- 테이블의 모든 고유키(PK 및 UK)에 파티셔닝 기준 컬럼이 반드시 포함되어야 함
  - 파티셔닝의 경우 파티션을 독립적인 테이블처럼 취급한다 
  - 따라서 인덱스 또한 파티션 단위로 독립적으로 존재한다
  - 이를 로컬 인덱스라고 하며, 파티션 키를 고유키에 포함함으로써 파티션별 데이터 고유성을 보장한다
- WHERE절에 파티셔닝 기준 컬럼에 대한 조건이 포함되어야 필요한 파티션들에만 접근하는 최적화인 파티션 프루밍이 적용됨
- 값이 자주 변경되는 컬럼을 파티션 기준 컬럼으로 선정해서는 안됨
  - 파티션의 기준 컬럼값이 변경되는 경우 파티션간에 데이터가 이동한다

## 5. 파티션 테이블 사용 예시
| RANGE                  | RANGE COLUMNS                |
|------------------------|------------------------------|
| 파티션 표현식에 컬럼 또는 계산식 허용  | 파티션 표현식에 컬럼만 허용              |
| 하나의 컬럼만 사용 가능          | 하나 이상의 컬럼 사용 가능              |
| 계산식 또는 컬럼 모두 정수형 값만 허용 | 정수형 값뿐만 아니라 문자열, 날짜 타입 값도 허용 |

### 1) timestamp
```mysql
CREATE TABLE user_log (
    id int NOT NULL AUTO_INCREMENT,
    user_id int NOT NULL,
    action_type VARCHAR(10) NOT NULL,
    ...,
    created_at timestamp NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (UNIX_TIMESTAMP(created_at))
(
    PARTITION p202401 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-02-01') ),
    PARTITION p202402 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-03-01') ),
    PARTITION p202403 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-04-01') ),
    PARTITION pMAX VALUES LESS THAN ( MAXVALUE )
)
```
- 고유키에는 파티션 키가 포함되어야 함
- RANGE와 timestamp 타입을 같이 쓰기 위해선 `UNIX_TIMESTAMP()`를 사용해야 함
  - `UNIX_TIMESTAMP()`는 timestamp를 정수값으로 바꿔줌
- 만약 MAX 파티션이 없으면 4월 이후의 데이터를 저장하려할 때 에러가 발생하고 데이터가 저장되지 않음 
  - 이러한 문제를 방지하기 위해 MAX 파티션을 사용하는 것이 좋음
```mysql
SELECT * FROM user_log WHERE created_at >= '2024-03-01'
```

### 2) timestamp + 추가적인 정밀도
```mysql
CREATE TABLE user_log (
    id int NOT NULL AUTO_INCREMENT,
    user_id int NOT NULL,
    action_type VARCHAR(10) NOT NULL,
    ...,
    created_at timestamp(6) NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (FLOOR(UNIX_TIMESTAMP(created_at)))
(
    PARTITION p202401 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-02-01') ),
    PARTITION p202402 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-03-01') ),
    PARTITION p202403 VALUES LESS THAN ( UNIX_TIMESTAMP('2024-04-01') ),
    PARTITION pMAX VALUES LESS THAN ( MAXVALUE )
)
```
- timestamp로 마이크로초까지 저장할 수 있도록 정밀도를 조정한 케이스 (`timestamp(6)`)
- 정수형이 아니기 때문에 파티션 프루닝이 불가능하므로 권장하는 형태는 아님
- `FLOOR()` 함수를 사용해서 정수형을 반환하도록 해야 함
- 밀리초나 마이크로초까지 시간값을 저장하고, 그 값을 기준으로 파티셔닝을 하려면 RANGE 대신 RANGE COLUMNS 사용을 권장

## 6. 파티션 관리
### 1) 파티션 추가
- 마지막 파티션 이후 범위의 신규 파티션을 추가
  - MAXVALUE 파티션이 존재하지 않는 경우
    - `ALTER TABLE user_log ADD PARTITION (...)`
  - MAXVALUE 파티션이 존재하는 경우
    - `ALTER TABLE user_log REORGANIZE PARTITION pMAX INTO (...)`
- 기존 파티션들 사이에 새로운 파티션 추가
  - `ADD PARTITION`은 사용 불가, `REORGANIZE PARTITION`를 사용해야 함
- 파티션 제거
  - `ALTER TABLE user_log DROP_PARTIION p202401`
  - 필요 시 별도 스토리지(콜드 데이터 저장용)백업 후 제거
- 파티션 비우기
  - `ALTER TABLE user_log TRUNCATE PARTITION p202401`

## 7. 파티션 사용 시 팁
- 주로 접근하는 범위 또는 삭제 대상 범위를 바탕으로 일/주/월/년 등으로 필요에 맞게 파티셔닝 
  - 즉, 비즈니스 로직에 맞춰 적절히 파티셔닝하는 것이 필요함
  - 이때 파티션 범위를 너무 작거나 크게 잡으면 자원 사용이나 관리 차원에서 비효율적
- 예상치 못한 상황을 대비해 MAXVALUE 파티션 사용
- 파티션 추가/삭제 시 잠금(메타데이터 락)이 발생하므로 가능하다면 트래픽이 적은 시점에 수행하는 것이 안전함
  - 추가적으로 DB에 오래 열려있는 트랜잭션이 존재하는지도 같이 체크해야 함
- 파티션 추가/삭제가 잦다면 수동으로 처리하기가 힘드므로 필요 시엔 파티션 관리 프로그램(자동화 스크립트)을 개발하는 것이 좋음

## 8. 파티션 테이블과 인덱스 사용
- 고유키가 아닌 일반 보조 인덱스의 경우 자유롭게 구성해서 생성 가능
- 파티션 별로 동일한 구조의 인덱스가 생성됨
- 파티션 프루닝을 통해 접근 대상 파티션 선정 후 인덱스 스캔
- 파티션 프루닝은 쿼리의 WHERE절에 파티셔닝 기준 컬럼에 대한 조건이 있어야 가능
  - 꼭 표현식과 동일한 형태로 WHERE절에 명시해야 하는 것은 아님 (`UNIX_TIMESTAMP()`, `YEAR()`, ...)
```text
파티션 프루닝 동작 가능?
- YES
--- 인덱스 사용가능?
----- YES -> 조회 대상 파티션에만 접근해 인덱스 스캔
----- NO -> 조회 대상 파티션 풀스캔
- NO
--- 인덱스 사용가능?
----- YES -> 전체 파티션 접근해 인덱스 스캔
----- NO -> 전체 파티션 풀스캔
```
- 실행계획의 "partitions" 항목에서 액세스하는 파티션의 목록을 확인할 수 있다
- 쿼리의 FROM절에서 특정 파티션만 조회하도록 직접 지정이 가능하다
  - `SELECT * FROM user_log PARTITION(p202304)`
