# UUID 사용 주의사항
## UUID Version
IETF의 표준 UUID는 5개 버전. Version 1, 4가 주로 사용됨

Version 2는 표준에 명시는 되어있으나 RFC 명세가 부실해서 잘 구현되진 않음

- Version-1, 2
    - Timestamp 기반의 UUID
    - 별도의 Unique한 값 입력 없이 생성 가능함
- Version-3, 5
    - name과 namespace의 MD5 또는 SHA-1 해시 기반의 UUID 생성함
    - 생성시 Unique한 입력이 필요함
- Version-4
    - 완전 랜덤한 UUID 생성
    - 별도의 Unique한 값 입력 없이 생성 가능함

## UUID 포맷 (Version - 1)

**01234567-89ab-11ee-8234-56789abcdef0**
- Timestamp = 1ee89ab01234567
    - 타임스탬프 값을 몇 부분으로 잘라서 순서를 바꾸어 배치함
- Version - 1 (11ee 부분 중 맨 앞 1)
- Sequence - 8234
- MacAddress - 56789abcdef0

- Timestamp의 비트 순서가 바뀌어서 UUID에 배치됨
    - 생성 시점이 동일해도 정렬 순서가 일치하지 않음
- UUID version-1의 타임스탬프는 100 나노초 단위로 1씩 증가
- 첫 번째 파트는 계속 증가하다가 7분 10여초 단위로 리셋됨

## UUID는 B-Tree 기반 인덱스의 성능을 저해시킬 수 있다.
- UUID
    - 시점과는 별개로 랜덤한 값 생성
    - 상대적으로 긴 문자열 (CHAR(32) or VARCHAR(32))
- B-Tree 인덱스의 성능 저해 요소
    - 정렬되지 않은 키 값 생성 & INSERT
    - 길이가 긴 키 값 (PK로 사용 시 모든 Secondary Index에 해당 값이 포함되므로 성능에 영향을 미침)
    - 일반적으로 UUID 컬럼은 유니크 제약 필요

## Index WorkingSet
- 특정 prefix에 해당하는 인덱스만 읽어서 쿼리 작업이 가능하여 메모리 효율이 증가함
    - 이 때, prefix를 통해 불러오는 일부의 인덱스 값을 WorkingSet이라 부름
- UUID 컬럼 인덱스는? -> 전체 인덱스 크기 만큼의 메모리 필요! (전체가 WorkingSet임)
    - UUID는 정렬되지 않고 무작위로 생성되는 값이기 때문에 전체가 WorkingSet이 될 수밖에 없음

## Timestamp-ordered UUID
- UUID version-6
    - DRAFT로 IETF 검토결과 DROP 됨
    - Timestamp를 분리하지 않고 UUID 최앞단에 배치
- UUID_TO_BIN() & BIN_TO_UUID()
    - MySQL Builtin-function
    - UUID의 timestamp 값을 재배치 후, 테이블에 저장 또는 조회
        - UUID가 정렬되는 효과
        - 생성 순서를 기준으로 데이터를 조회하는 경우 WorkingSet이 작아져서 메모리 효율이 높아짐

## UUID vs BIGINT
- UUID: 32 chars(VARCHAR), 16 bytes (BINARY)
    - 길이가 꽤 김!
- BIGINT: 8 bytes
- 1억건 테이블 (10개 인덱스를 가진 테이블의 PK인 경우)
    - 단일 인덱스 크기 = 24GB vs 6GB
    - 전체 인덱스 크기 = 264GB vs 66GB
- 비용차이 발생 심함

UUID를 대체할 수 있는 방법
- BIGINT uid
    - DBMS serial - Auto increment
        - 값을 예측할 수 있다는 단점이 있으나, 그것만 잘 극복하면 가장 쉬운 대안임
    - snowflake-uid, sonyflake-uid
        - timestamp 기반이므로 정렬 가능함
- timestamp prefix
    - 가벼운 단조 증가 PK
    - 레인지 파티션을 위한 PK로 활용 가능
        - PRIMARY KEY (id, created_at) -- 일반적인 방법
        - PRIMARY KEY (uid) -- PK 변경없이 파티션 적용 가능 (snowflake, sonyflake로 생성된 값을 사용할 수 있음)

## UUID vs Snowflake-uid
- UUID와 Snowflake 모두 동일 구성
    - Timestamp
    - Node-id
    - Sequence
- 꼭 UUID여야 할 필요가 있는가?
    - 일반적으로 대체제가 더 나은 선택임
- 정보 노출 우려
    - Auto Increment & Serial은 단조 증가이므로 데이터 크기 예측이 가능함
    - Snowflake-uid는 예측이 불가 (생성 시점은 노출될 수 있음)

## 대체키 활용 (Hybrid)
만약 꼭 UUID를 사용해야한다면?
- 내부적으로는 Auto Increment 또는 Timestamp 기반의 프라이머리 키 생성
- 외부적으로는 UUID 기반의 유니크 세컨더리 인덱스를 사용