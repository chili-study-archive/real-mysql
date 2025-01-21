# COUNT(*) vs COUNT(col)

## 1. COUNT(*) vs COUNT(col)
```text
COUNT(*) == COUNT(non_null_col)
COUNT(*) != COUNT(null_col)
```
COUNT 함수는 non-NULL 값의 개수를 리턴한다.

## 2. COUNT(expr) 실행 계획
WHERE절의 조건 여부에 따라 COUNT(expr)의 실행 계획이 달라진다.

### 1) 조건없는 COUNT()
COUNT(null_col)은 테이블 풀스캔이 발생한다. 기본적으로 COUNT(non_null_col)은 COUNT(*)와 동일한 성능을 내지만, 병렬처리를 비활성화 하면 실행 계획은 동일한 반면, 쿼리 성능은 달라질 수 있다.
- MySQL 8.0은 조건없는 COUNT 쿼리에 대해 병렬 처리 지원 (default 쓰레드 4개)
- But, 쓰레드가 1개면 Inno DB 엔진이 내부적으로 레코드를 읽어서 컬럼을 추출하여 성능에 영향을 줄 수 있음
- 값을 추출해야 하는 COUNT(non_null_col)이 성능이 떨어지나, 값을 추출하지 않아도 되는 COUNT(*)은 성능이 우수함

COUNT(col)보다는 COUNT(*)를 사용하자!

MySQL은 내부적으로 최소의 크기를 지닌 인덱스를 기반으로 COUNT 쿼리를 수행한다.

강의 기준 버전의 MySQL에선 조건없는 COUNT 쿼리 사용 시 실제 실행계획과 다르게 PK 인덱스를 타는 버그가 있었음. (인지만 해두자)

### 2) 조건있는 COUNT()

COUNT 쿼리의 성능을 향상시키기 위해서는 2가지에 집중해야 한다.
1) 쿼리가 인덱스를 사용할 수 있도록 해야 한다.
2) 최소의 레코드만으로 테이블 데이터를 가져오도록 튜닝한다.

이 중 2번은 Disk I/O 때문인데, 커버링 인덱스를 적용하면 이 과정이 생략되므로 가능한 한 커버링 인덱스를 사용하는 것이 좋다.

```mysql
-- 커버링 인덱스 O
SELECT COUNT(1) FROM counter WHERE ix1='comment';
SELECT COUNT(*) FROM counter WHERE ix1='comment';
SELECT COUNT(ix1) FROM counter WHERE ix1='comment';

-- 커버링 인덱스 X
SELECT COUNT(fd1) FROM counter WHERE ix1='comment';
```


## 3. 정리
- COUNT(col)보다는 COUNT(*)를 사용하는 것이 실행계획상 더 유리하다
- COUNT(col)보다는 COUNT(*)가 가독성이 더 좋다
- COUNT(non_null_col)은 의도가 불분명하므로 필요한 경우엔 `SELECT COUNT(*) FROM tab WHERE nullable_col IS NOT NULL`처럼 명확하게 작성한다
- 전체 건수가 필요한 경우엔 COUNT(*)로 작성한다 
