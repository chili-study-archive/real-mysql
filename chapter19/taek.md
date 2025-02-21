# JSON 타입 활용

## 1. JSON 데이터 타입
- JSON 형식의 데이터 저장/조회/관리 기능
- JSON 데이터 조작을 위한 빌트인 함수 지원
- JSON 데이터의 특정 키에 대한 인덱스 생성 가능

DEFAULT 값으로 `NULL`과 `json_object()`, `json_array()` 표현식 등을 지정할 수 있다. 기존 테이블에 JSON 컬럼 추가 시 DEFAULT 값이 null인 경우엔 INSTANT 방식으로 컬럼 추가가 가능하지만, DEFAULT 값이 표현식인 경우엔 COPY/INPLACE 방식만 사용이 가능하다.

## 2. 데이터 저장
표현식
- 배열: `JSON_ARRAY(value1, value2, value3, ...)`
- 객체: `JSON_OBJECT(key1, value1, key2, value2, ...)`

직접 값을 입력하기
- `INSERT INT tb1 (json_colmn) VALUES ('[1, "abc", "2023-12-01"]');`
- `INSERT INT tb1 (json_colmn) VALUES ('{"key1":123, "key2":"abc"}');`
  - key 지정 시 항상 쌍따옴표(")를 사용해야 한다 

저장구조
- JSON 타입을 최적화된 바이너리 포맷으로 변환되어 저장된다 (이때 유효성 검사를 수행한다)
- 중복된 key-value은 마지막에 등장한 키-값이 저장된다
- key는 정렬되어 저장된다
- 여러 JSON 데이터가 같은 key를 갖고 있다 하더라도 따로 저장되기 때문에 key는 적당한 길이를 사용해야 한다

## 3. 데이터 조회
- 대표적으로 JSON Path가 있다
  - JSON Path는 JSON 데이터의 요소를 쿼리하는 표준화된 방법이다
  - https://www.ietf.org/archive/id/draft-ietf-jsonpath-base-21.html#jsonpath-abnf
- 조회를 할 땐 JSON Path와 함수/연산자를 함께 사용한다
  - `JSON_EXTRACT(json_data, path ...)`
  - `json_column -> path`
  - `json_column ->> path`
- 그 외 `JSON_CONTATINS`, `JSON_OVERLAPS` 등의 함수가 있는데 이건 필요 시 찾아봐도 될 것 같다

## 4. 데이터 변경
- MySQL은 `JSON-INSERT()`, `JSON_REPLACE()`, `JSON_SET()`, `JSON_REMOVE()` 등의 조작 함수를 제공한다
- JSON 데이터의 특정 키 값만 변경 시 변경된 키 값에 대해서만 데이터를 업데이트하는 "부분 업데이트" 최적화를 제공한다
  - 이로써 불필요하게 전체 데이터를 다시 쓰지 않으므로 쿼리 성능이 향상된다
  - 단, 대체되는 새로운 값이 기존에 저장된 값보다 저장되는 크기가 작거나 같아야 한다

## 5. 데이터 인덱싱
- 특정 키 값에 대해 인덱싱 가능
- 함수 기반 인덱스(Function Based Index)로 인덱스 생성
- 문자열 값을 인덱싱하는 경우 따옴표 및 콜레이션 주의
- 주의사항
  - `MEMBER_OF()`, `JSON_CONTAINS()`, `JSON_OVERLAPS()` 함수만 배열 인덱스 사용 가능
  - 아직 기능이 성숙하지 못해 버그가 존재하는 상황이므로 유의해서 사용
  - 기타 여러 제한사항이 존재함
  - JSON 데이터 사이즈가 클 수록 조회 성능이 저하됨
