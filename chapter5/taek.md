# Stored Function

## 1. DETERMINISTIC vs NOT DETERMINISTIC
- DETERMINISTIC
  - 입력이 동일하면 호출 시 동일한 결과를 반환하는 것 (확정적)
    - 테이블의 레코드가 달라지는 것은 동일 입력으로 간주하지 않는다
    - 같은 문장에서 Stored Function이 여러 번 호출되더라도 해당 시점의 스냅샷을 본다
- NOT DETERMINISTIC
  - 입력이 동일하더라도 호출되는 시점에 따라서 결과가 달라질 수 있는 것 (비확정적)

## 2. NOT DETERMINISTIC 최적화 이슈
- NOT DETERMINISTIC 함수의 결과는 비확정적
- 즉, 매번 호출 시점마다 결과가 달라질 수 있음
  - 비교 기준 값이 상수가 아니라 변수임
  - 레코드를 읽은 후 WHERE절을 평가할 때마다 결과가 달라질 수 있음
  - 따라서 인덱스를 못 태운다

## 3. NOT DETERMINISTIC 효과
- NOT DETERMINISTIC Built-in Function
  - `RAND()`
  - `UUID()`
  - `SYSDATE()`
  - `NOW()`
  - ...

## 4. NOT DETERMINISTIC 예외
- `NOW()` vs `SYSDATE()`
  - 동일하게 현재 일자와 시간을 반환하는 함수
  - 둘 모두 NOT DETERMINISTIC 함수
  - `NOW()`는 하나의 문장에서는 DETERMINISTIC로 동작
  - `SYSDATE()`는 하나의 문장에서도 NOT DETERMINISTIC로 동작
- `created_at = NOW()`는 인덱스를 탈 수 있음, `created_at = SYSDATE()`는 풀스캔이 발생함
- MySQL 서버에 sysdate-is-now 설정을 하면 `SYSDATE()`도 `NOW()`처럼 동작함

## 5. Stored Function 주의사항
- 옵션을 명시하지 않으면 기본적으로 NOT DETERMINISTIC으로 인식한다
- 따라서 Stored Function 생성 시엔 기본 옵션을 꼭 명시해주는 게 좋다
- `DEFINER`와 `SQL SECURITY` 속성을 꼭 명시해주는 게 좋다
