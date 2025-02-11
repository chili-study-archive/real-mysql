# JSON 타입 활용
## JSON 데이터 타입?
- JSON 형식의 데이터를 손쉽게 저장 및 조회, 관리
- 빌트인 함수를 사용해 JSON 데이터 조작 가능
- JSON 데이터의 일부만 업데이트 가능
- JSON 데이터의 특정 키에 대해 인덱스 생성 가능

## JSON 데이터 저장
### DML
```sql
CREATE TABLE tb (
    ...
    json_col1 json DEFAULT NULL,
    json_col2 json DEFAULT (json_object()),
    json_col3 json DEFAULT (json_array()),
    ...
)
```
- `json_object()`, `json_array()` 대신 `('{}')`, `('[]')` 문자열로도 지정 가능
- json 디폴트값은 8.0.13 버전 이상부터 가능
- 기존 테이블에 JSON 컬럼 추가 시 주의사항!
    - DEFAULT를 NULL로 지정하면 INSTANT ALGORITHM으로 바로 컬럼 추가가 가능하지만, 그 외에는 INSTANT 방식 적용 불가
        - COPY/INPLACE 방식이 강제되어 실행시간동안 DML이 제어됨

### 데이터 저장 방법
- 함수를 사용해 데이터 저장
    - 배열
        - JSON_ARRAY('abc', 12345)
            - `["abc", 12345]`
    - 객체
        - JSON_OBJECT('key1', 12345, 'key2', 'abc)
            - `{"key1": 12345, "key2": "abc"}`
    - 저장 시 빌트인 함수 사용 가능 (NOW() 등)
- 직접 값을 입력해 저장
    - `INSERT INTO tb1 (json_column) VALUES ('{"key1":123, "key2":"abc"}')`
    - 문자열로 구성하여 입력
    - 객체값 저장 시에는 키값을 반드시 쌍따옴표로 감싸줘야함
    - 저장 시 유효성 검사 수행됨. 유효하지 않으면 쿼리 실패

### 저장 구조
- 저장 요소
    - JSON 키 개수
    - JSON 데이터 사이즈
    - 키 주소
    - 값 주소
    - 키
    - 값

- 최적화된 바이너리 포맷으로 저장
    - JSON 요소들에 보다 빠르게 접근하기 위함
- 중복된 키-값은 마지막 순서의 데이터로 저장됨
    - `{"a": 123, "a": 456}` -> `{"a": 456}`
- JSON 데이터 내 키들을 정렬하여 저장함
- 키는 JSON 데이터마다 중복해서 저장되므로 적당한 길이로 사용하는 것을 권고

## JSON 데이터 조회
### JSON Path
- JSON 데이터의 요소를 쿼리하는 표준화된 방법
- 대표적인 연산자
    - `$`: JSON 데이터 계층의 루트를 의미
    - `.`: 객체의 하위 요소들을 참조할 때 사용
    - `[]`: 배열 내의 요소에 접근할 때 사용

### 조회 방법
- 함수
    - `JSON_EXTRACT(json_data, {JSON_Path})`
- 연산자
    - Column-path Operator
        - `json_column->path` = `JSON_EXTRACT(json_column, path)`
    - Inline-path Operator
        - `json_column->>path` = `JSON_UNQUOTE(json_column->path)` = `JSON_UNQUOTE(JSON_EXTRACT(json_column, path))`
        - 추출한 값에서 쌍따옴표와 이스케이핑 처리가 제거된 문자열이 반환됨
- 그 외 함수들
    - `JSON_CONTAINS(target_json, candidate_json[, path])`
        - target_json에 candidate_json이 포함되어있으면 True(1), 아니면 False(0) 반환
        - path가 주어진 경우에는 지정된 path에 위치한 값에 대해서만 확인
    - `JSON_OVERLAPS(json_data1, json_data2)`
        - 두 JSON가 하나라도 공통된 값을 지니면 True, 아니면 False
    - `value MEMBER OF(json_array)`
        - value에 주어진 값이 json_array에 포함되어있는지 반환

## JSON 데이터 변경
- `JSON_INSERT()`, `JSON_REPLACE()`, `JSON_SET()`, `JSON_REMOVE()` 등 MySQL에서 제공하는 함수들을 통해 JSON 데이터를 보다 세밀하게 조작 가능
- JSON 데이터의 특정 키 값만 변경 시 변경된 값에 대해서만 데이터를 업데이트하는 부분 업데이트 최적화 제공
- 불필요하게 전체 데이터를 다시 쓰지 않아서 쿼리 성능이 향상됨

### JSON 부분 업데이트
- 부분 업데이트가 실행되는 조건?
    - `JSON_REPLACE()`, `JSON_SET()`, `JSON_REMOVE()` 에서만 가능함
    - 함수의 인자로 주어진 컬럼과 변경 대상 컬럼이 일치해야함
    - 값 변경 시 기존 값을 새로운 값으로 "대체"하는 형태여야함
        - 새로운 키-값이 추가되는 변경은 부분 업데이트 처리불가함
    - 대체되는 새로운 값은 기존에 저장된 값보다 크기가 작거나 같아야함
- 부분 업데이트가 일반 업데이트보다 더 성능이 좋음

#### 바이너리 로그와 부분 업데이트
- MySQL 8.0에서의 바이너리 로그 기본 설정
    - `log_bin=ON`
        - 기본 활성화
    - `binlog_format=ROW`
        - 변경된 레코드들이 모두 기록됨
    - `binlog_row_image=full`
        - 레코드의 전체 데이터를 기록함
    - `binlog_row_value_options='' (empty string)`
        - JSON 데이터가 부분 업데이트됐을 때, 변경된 내용만 기록되게할지, 전체 데이터를 모두 기록되게 할지?
            - 빈 문자열 입력 시 전체 데이터를 기록
- 기본으로 설정된 값은 부분 업데이트 성능을 저하시킬 수 있음
    - 기본 설정값 변경 후 성능 변화
        - `SET binlog_row_image = MINIMAL;`
            - 전체가 아닌 최소한만 기록
        - `SET binlog_row_value_options = PARTIAL_JSON;`
            - JSON의 수정된 부분만 바이너리 로그에 기록됨
        - `SET binlog_format=STATEMENT`
            - 쿼리가 변경한 레코드가 아닌 실행된 쿼리만 기록함

## JSON 데이터 인덱싱
- 특정 키 값에 대해 인덱싱 가능
- 함수 기반 인덱스로 인덱스 생성
- 문자열 값을 인덱싱하는 경우 따옴표 및 콜레이션 주의

```sql
ALTER TABLE tb_json_index
    ADD KEY ix_string ( (CAST(fd->>'$.name' AS CHAR(30)))),
    ...
```
- Inline-Path Operator로 가져온 JSON 데이터는 LONGTEXT 타입임
- 따라서 JSON 값을 알맞은 형태의 데이터 타입으로 CAST 해주어야함
- 인덱스 사용 시에는 인덱스 조건 그대로 쿼리에 명시해야함
    - `SELECT * FROM tb_json_index WHERE CAST(fd->>'$.name' AS CHAR(30)) = 'Brynn';`
    - CAST 없이 사용해도 옵티마이저가 알아서 캐스팅해서 실행시켜주긴하지만, 암묵적 형변환을 비롯한 예상치못한 일이 발생가능함.
        - 가능하면 그대로 사용하기
    - 인덱스 타입에 맞는 값을 조건절에 넣어줘야함
        - `Brynn`은 CHAR(30)을 만족하는 값임
- 배열 인덱스 사용 시엔 특정 함수를 사용해야함
    - `MEMBER OF` 등..

### 인덱스 사용 시 주의사항
- 배열 인덱스 사용 시 주의사항
    - `MEMBER OF()`, `JSON_CONTAINS()`, `JSON_OVERLAPS()` 함수만 배열 인덱스 사용 가능
        - `SELECT ... FROM ... WHERE 1 MEMBER OF(fd->>...)`
    - 아직 성숙하지 못해 이슈가 존재함. 유의하여 사용
    - 기타 여러 제한 사항
        - 온라인으로 인덱스 생성 불가함
        - 커버링 인덱스 & 레인지 스캔 불가
        - 빈 배열 식별 불가
- 문자열 값 인덱싱 주의사항
    - 따옴표 포함 유무에 따라 같은 조건이라도 쿼리 결과가 달라짐
        - EX) 인덱스 생성 시엔 `fd->'$.name'`을 사용하면 따옴표를 포함한 문자열을 WHERE 절에 추가해야함
    - JSON 내 문자열 데이터 처리 시 utf8mb4_bin 콜레이션 사용(대소문자 구분 O)
        - CAST() 에서는 문자열을 utf8mb4_0900_ai_ci 콜레이션 사용(대소문자 구분 X)
        - 이로 인해 쿼리에서 인덱스 사용 여부에 따라 결과 데이터가 다를 수 있음
            - EX) 인덱스 추가 전엔 소문자 값만 조회되다가 추가 후엔 동일한 값의 대문자 값까지 조회될 수도 있음
## TEXT 타입 vs JSON 타입
- TEXT
    - 입력된 문자열 그대로 저장
    - 데이터 조회 시 저장된 데이터를 변환하지 않고 전송
    - 항상 전체 데이터 업데이트
- JSON
    - 최적화된 바이너리 포맷 저장 & 유효성 검사
    - 데이터 조회 시 바이너리 JSON 데이터를 문자열 형식으로 변환 후 전송
    - 부분 업데이트 가능

#### TEXT가 유리한 경우
- 데이터를 저장 후 전체 데이터를 조회하는 패턴으로 주로 사용하는 경우
- JSON 형식이 아닌 데이터도 저장될 수 있음
- TEXT 타입 컬럼도 JSON 함수 사용 및 특정 키 값에 대한 인덱스가 생성 가능함!
    - 전체 조회하는 패턴이면 TEXT로도 충분히 대응 가능

#### JSON이 유리한 경우
- JSON 데이터의 특정 키 값만 주로 조회하고 변경하는 경우

## 정규화된 컬럼 vs JSON 컬럼
- 정규화
    - 정적 스키마
        - 스키마 변경 시 수정이 번거롭고 시간이 많이 소요될 수 있음
    - 데이터 일관성 및 유지보수 용이
    - 쿼리 최적화와 인덱싱 편리함
- JSON 컬럼
    - 유연한 스키마
    - 개발 편의성
    - 쿼리와 인덱싱 방식이 다소 복잡해질 수 있음
    - 크기가 큰 데이터를 반복적으로 조회 시 부하 발생 가능함
        - 레코드 별로 JSON 컬럼이 약 1MB 정도만 되더라도 성능 차이가 매우 큼
        - JSON 데이터가 필요한 경우에만 SELECT 절에 명시하여 조회하는 것을 권고
        - 데이터가 지나치게 큰 경우 RDB에 저장하는 것이 부적절한 것일 수 있음. 별도 저장 서버를 구축하는 것도 좋은 방법임