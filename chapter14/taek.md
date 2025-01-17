# UUID 사용 주의사항

## 1. UUID Version
- Version-1 & 2
  - Timestamp 기반의 UUID 생성
  - Timestamp의 값을 그대로 쓰지 않고, UUID 형식에 맞춰 값을 나누고 재배치해서 씀
  - 생성 시 Unique 값이 필요 O
- Version-3 & 5
  - name과 namespaec의 MD5 또는 SHA-1 해시 기반의 UUID 생성
  - 생성 시 Unique 값이 필요 X
- Version-4
  - 완전 랜덤한 UUID 생성
- 주로 Version 1과 Version 4가 주로 사용됨


## 2. UUID vs B-Tree
- UUID
  - 랜덤한 값 생성
  - 상대적으로 긴 문자열 (CHAR(32) or VARCHAR(32))
- B-Tree 인덱스의 성능 저해 요소
  - 정렬되지 않은 키 값 생성 & INESRT
  - 길이가 킨 키 값 -> MySQL은 모든 Secondary Index가 PK를 포함하므로 성능이 저하되고 용량이 낭비됨
  - 일반적으로 UUID 컬럼은 유니크 제약이 필요 -> 컬럼에 유니크 제약이 붙은 경우 [change buffer](https://dev.mysql.com/doc/refman/8.4/en/innodb-change-buffer.html)를 이용하지 못해 성능이 떨어짐
  - 

## 3. Index Working Set
- MySQL은 인덱스의 용량이 매우 크더라도 필요한 데이터만 메모리로 올려서 쿼리를 처리할 수 있음
- 여기서 쿼리 처리를 위해 메모리로 올린 인덱스 데이터를 Working Set이라고 함
- UUID는 완전한 랜덤값이기 때문에 모든 데이터가 Working Set임
- 최근 데이터를 주로 쿼리하는 경우, Timestamp 순서대로 정렬을 하면 Working Set을 적게 가져갈 수 있음
  - 최신 버전의 MySQL은 Timestamp 정렬 함수를 지원함
  - `UUID_TO_BIN(@uuid, 1)`, `BIN_TO_UUID(@uuid, 1)`
  - UUID의 timestamp값을 재배치 후, 테이블에 저장 또는 조회

## 4. UUID vs BIGINT
- UUID는 16바이트의 Binary값이지만, 실제로 저장될 땐 32바이트의 16진수 문자열(VARCHAR)로 저장하는 경우가 잦음 -> 인덱스로 사용하기에 용량이 상당히 큼
- 반면, BIGINT는 8바이트로 크기가 훨씬 작음 -> BIGINT로 uid를 사용하면 보다 효율적으로 데이터 공간 사용 가능
  - DBMS serial (Auto Increment, Sequence, ...)
  - Snowflake-uid, Sonyflake-uid와 같은 Timestamp 기반의 라이브러리
    - 이러한 값들은 별도의 PK 변경없이 레인지 파티셔닝을 적용할 수 있음
    - id가 시간 순서대로 생성되므로 월단위, 연단위 등으로 쉽게 나눌 수 있음

## 5. UUID vs Snowflake
- UUID와 Snowfloake 모두 동일 구성
  - Timestamp (epoch)
  - Node-id
  - Sequence
- UUID 대신 8바이트의 BIGINT 사용을 권장
  - DBMS Serial은 잔조 증가이므로 데이터의 크기가 예측 가능함 (정보 노출)
  - 반면 Snowflake-uid는 예측 불가 (단, 시점 노출 우려가 존재함)
- UUID를 사야 한다면 대체키를 활용할 것을 권장
  - 내부적으로는 Auto Increment 또는 Timestamp 기반의 PK 사용
  - 외부적으로는 UUID 기반의 Unique Secondary Index 사용
```mysql
CREATE TABLE table (
    id BIGINT NOT NULL AUTO_INCREMENT,
    external_uid CHAR(32) NOT NULL,
    ...
    PRIMARY_KEY (id),
    UNIQUE INDEDX ux_externaluid (external_uid)
);
```
