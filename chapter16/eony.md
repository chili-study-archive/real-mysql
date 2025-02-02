# COUNT(*) vs COUNT(fd)

```sql
# 아래와 같은 테이블이 있다고 가정
CREATE TABLE counter (
    id int NOT NULL AUTO_INCREMENT,
    ix1 varchar(200) NOT NULL,
    fd1 varchar(200) DEFAULT NULL,
    PRIMARY KEY (id),
    KEY idx1 (ix1)
)

# 아래 카운트 쿼리를 실행시켰을 때,
SELECT COUNT(*), COUNT(1), SUM(1), COUNT(id), COUNT(ix1), COUNT(fd1)
FROM counter;

    COUNT(*): 14108
    COUNT(1): 14108
    SUM(1): 14108
    COUNT(id): 14108
    COUNT(ix1): 14108
    COUNT(fd1): 14107 # 혼자 결과가 다름 -> 해당 컬럼값이 NULL이 아닌 것들만 카운트했기 때문
```

# COUNT(expr) 메뉴얼
- Returns a count of the number of non-NULL values of expr in the rows retrieved by a SELECT statement
    - NULL이 아닌 데이터만 카운트를 함!

EX) `SELECT COUNT(NULL)` -> 0

# COUNT(expr) 실행 계획
(1) WHERE 조건 가진 COUNT
    - Covering Index
    - Non-Covering Index
(2) WHERE 조건 없는 COUNT()
    - ha_records() 스토리지 API 사용
    - ha_index_next() 스토리지 API 사용
    - WHERE 조건이 없는 경우엔 실행계획보단 스토리지 엔진의 API에 의해 성능이 좌우됨

## 1. 조건없는 COUNT()
- 실행 계획이 동일하면 성능도 동일할까?
> COUNT(*) 및 COUNT(not_nullable_col)과 비교해 NULLABLE 컬럼으로 COUNT 쿼리를 실행시켰을 때 훨씬 오래걸렸음

이유
- Handler 상태 값 변화(`show status;`로 확인)
    - NOT NULL 컬럼의 경우 Handler_read_next, Handler_read_rnd_next 값에 변화가 없음(0으로 유지)
        - COUNT(*) 쿼리는 InnoDB 스토리지 엔진의 ha_records() API를 한 번만 호출해 전체 레코드 개수를 가져왔기때문
            - ha_records() 자체는 MySQL 핸들러 매트릭에 등록되어있지 않아 호출 횟수가 찍히지 않음
    - NULLABLE 컬럼의 경우 Handler_read_rnd_next가 증가, Handler_read_first가 1(Primary Index를 풀스캔했다는 의미)
    
-> MySQL 서버에서 호출하는 스토리지 엔진 API가 달라지며 성능 차이가 발생한다!

InnoDB 병렬 읽기 작업
- innodb_parallel_read_threads 설정
    - MySQL 8.0 버전은 조건없는 COUNT() 쿼리에 대해 병렬처리 지원
        - 모든 쿼리가 가능하진 않음
    - `SET innodb_parallel_read_threads`를 통해 스레드 개수를 설정할 수 있음
        - 기본값은 4임

만약 해당 스레드 개수를 1로 설정하고(병렬 처리 비활성화) 동일한 카운트 쿼리를 실행하면?

결과
- COUNT(*) -> 똑같이 빠름
- COUNT(non_nullable_column) -> 느려짐
- COUNT(nullable_column) -> 똑같이 느림

이유?
- 레코드 추출 작업!
    - COUNT(*)과 COUNT(non_nullable_column)은 레코드 추출이 일어나지 않음 (NULL 데이터를 필터링하지 않아도 되기 떄문)
    - COUNT(nullable_column)은 레코드 추출 작업이 발생해 시간이 더 오래걸림

- 병렬처리가 비활성화되어 COUNT(non_nullable_column)이 레코드 추출 작업을 하도록 변경되어 수행시간이 증가(내부적으로 왜 이렇게 동작한지는 정확히 파악이 안됨), COUNT(*)는 그대로 유지

> 결론은 COUNT(*)을 사용하는게 항상 좋은 결과를 가져올 가능성이 높음 (아마 스토리지 엔진 API 등에서 최적화를 따로 해놓지 않았을까하는 생각)

## 조건없는 COUNT() 정리
- COUNT(*)와 COUNT(not_null_column)
    - ha_records() 스토리지 엔진 API 사용
    - 레코드 건수와 관계없이 1회만 호출
    - COUNT(*) 쿼리는 레코드로부터 컬럼 추출 수행 X
    - innodb_parallel_read_threads >= 2일 때, COUNT(not_null_column) 쿼리도 컬럼 추출 수행 X
- COUNT(nullable_column)
    - ha_index_next() 스토리지 엔진 API 사용
    - 레코드 건수만큼 호출됨
    - 주어진 컬럼에 대해 Eval 작업 필요(레코드 컬럼 추출 필요함)

## 조건없는 COUNT()의 실행계획

MySQL 서버는 다른 DBMS와 다르게 인덱스가 NULL인 컬럼도 모두 포함됨.
    - 대상 컬럼이 NULL이든 아니든 인덱스를 스캔하면 테이블의 정확한 레코드 건수 가져올 수 있음

COUNT를 EXPLAIN시  type에 `index`, Extra에 `Using index` 사용
    - 카운트 쿼리가 커버링 인덱스로 처리되었다는 것을 의미함
    - 중요) 카운트 쿼리를 한다고 해서 무조건 PK를 타진 않음
        - PK에는 데이터의 모든 컬럼값이 있어서 읽어오는데 부담이 있음
        - 최소의 레코드만 테이블 데이터를 가져오도록 튜닝됨
            - InnoDB 버퍼풀에 캐싱된 페이지 개수를 이용해 최적의 인덱스를 선택 (8.0부터)
            - 크기가 작은 인덱스가 선택될 가능성이 높음

**실행계획 버그도 있음**
카운트 쿼리를 EXPLAIN 했을 때 key가 PK가 아닌 다른 인덱스로 표시되어있었으나, 실제 쿼리 수행 후 innodb 버퍼풀에 적재된 인덱스별 데이터 수를 확인해보면 PK의 인덱스 페이지가 적재되어있음
    - 조건없는 COUNT()는 실행계획과 무관하게 항상 PK를 이용하도록 코드가 작성되어있었음
    - 조건이 있다면 해당 버그가 발생하지 않을 것으로 보임

# 정리
- 아래 쿼리는 모두 동일한 결과
    - COUNT(*), COUNT(1), SUM(1), COUNT(pk_col), COUNT(not_null_col)
- 아래 쿼리는?
    - COUNT(nullable_col)
        - 쿼리의 가독성이 떨어진다 (실수인지 의도인지 식별 어려움)
        - 특정 컬럼의 카운트가 필요한 경우, 쿼리의 조건 또는 주석 표기
            - SELECT COUNT(*) FROM table WHERE nullable_col IS NOT NULL
- 전체 건수가 필요한 경우 COUNT(*)만 사용 권장