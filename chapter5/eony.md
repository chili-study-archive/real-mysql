# Stored Function

### MySQL Function
- Built-in Function
    - MySQL에 내장된 함수
- User Defined Function (UDF)
    - 사용자가 직접 개발
- Stored Function

### 주 포인트
- DETERMINISTIC vs NOT DETERMINISTIC
- Function vs Index Optimization

(1) DETERMINISTIC
- 동일 상태와 동일 입력으로 호출 -> 동일한 결과 반환


(2) NOT DETERMINISTIC
- 매번 호출시점마다 결과가 달라질 수 있음
    - 비교 기준 값이 상수가 아니고 **변수**임
        - 옵티마이저가 기준 값을 CONST로 처리하지 못함을 의미
- 인덱스에서 특정 값 검색 불가
- 인덱스 최적화 사용 불가능
    - 풀테이블 스캔이 발생할 확률이 더 높아짐


#### NOT DETERMINISTIC의 Built-in Function 예시
- RAND()
- UUID()
- SYSDATE()
- NOW()

네 가지 모두 비교 컬럼의 인덱스를 효율적으로 사용하지 못함
- EX) `WHERE col1 = (RAND()*1000)`

_`NOW()`와 `SYSDATE()`는 약간은 예외인 함수!_
- 둘 다 현재일자와 시간 반환하는 함수
- 모두 NOT DETERMINISTIC
    - `NOW()`는 하나의 STATEMENT에서 DETERMINISTIC처럼 작동
        - 따라서 인덱스 사용가능함
    - `SYSDATE()`는 매번 함수 호출 시점에 반환
- `sysdate-is-now` 설정을 통해 `SYSDATE()`를 `NOW()`처럼 동작하게 만들 수 있음
    - 경험상 `SYSDATE()`의 작동방식이 필요했던 적은 거의 없었음

#### Stored Function 주의사항
- 옵션이 명시되지 않으면 NOT DETERMINISTIC이 기본 옵션임
- StoredFunction 생성 시 기본옵션을 반드시 명시

```sql
CREATE
    DEFINER=
    ...
    FUNCTION function_nam(...)
    RETURNS ...
    DETERMINISTIC /* 명시! */
    SQL SECURITY INVOKER 
BEGIN
    ...
```