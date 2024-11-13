# COUNT(*) & COUNT(DISTINCT)

## 1. SELECT VS COUNT
- SELECT는 주로 LIMIT과 같이 사용되는 반면, COUNT는 LIMIT 없이 사용된다
```sql
-- 20개씩 페이징하여 조회
SELECT * LIMIT 20;

-- 페이지네이션을 위해 전체건수 조회
COUNT(*);
```
- 따라서 COUNT 쿼리 수행 시에 성능이 문제가 될 확률이 높다

## 2. COUNT(*) 성능 개선
### 1) 커버링 인덱스 사용
- 커버링 인덱스를 사용하면 COUNT 쿼리의 성능이 개선되지만, 커버링 인덱스만을 위해 쿼리에 필요한 모든 컬럼들을 인덱스로 만드는 것은 비효율적이다
- 따라서 커버링 인덱스를 이용한 튜닝은 꼭 필요한 경우에만 사용해야 한다
```sql
-- 커버링 인덱스
SELECT COUNT(*) WHERE idx_fd1=? AND idx_fd2=?;
SELECT COUNT(idx_fd2) WHERE idx_fd1=?;

-- 논 커버링 인덱스
SELECT COUNT(*) WHERE idx_fd1=? AND non_idx_fd1=?;
SELECT COUNT(non_idx_fd1) WHERE idx_fd1=?;
```

### 2) COUNT(DISTINCT expr) 사용 지양하기
- COUNT(*)는 레코드 건수만 확인한다
- COUNT(DISTINCT expr)은 임시 테이블로 중복을 제거한 후에 건수를 확인한다
  - 이 과정에서 테이블의 레코드를 모두 임시 테이블로 복사하는 과정이 추가된다
  - 만약 테이블의 레코드가 너무 많다면 임시 테이블을 디스크로 다시 옮겨서 저장한다
  - 이는 메모리/CPU 외에도 디스크 I/O가 추가되어 쿼리 성능이 매우 저하된다
- **ORM에서 COUNT(DISTINCT expr)를 자동적으로 생성하는 경우**가 있어서 조심해야 한다 (꼭 쿼리를 확인하자!)

### 3) 쿼리 제거하기
- 애플리케이션의 로직 변경이 없는 COUNT(*) 튜닝은 커버링 인덱스를 사용하는 것이 유일하다
- 따라서 애플리케이션 로직 변경을 통해 **쿼리 자체를 제거하는 것**이 최고의 튜닝이다
  - 예를 들어 페이지 번호 없이 이전/이후 페이지로 이동하는 방식 등을 사용할 수 있다
  - 이를 통해 전체 건수 확인 쿼리를 제거하는 것을 목표로 삼아야 한다

### 4) 대략적인 건수 보여주기
- 전체 건수 확인 쿼리를 제거할 수 없다면 **대략적인 건수를 보여주는 방식**을 활용해볼 수 있다
#### 4-1) 표시할 페이지 번호만 큼의 레코드만 건수를 확인한다
  - 예를 들어 한 페이지당 20개씩 데이터를 보여주고, 10개의 페이지 네비게이션을 제공한다고 가정하면   200건까지의 레코드가 있는지만 확인해보면 된다
  - `SELECT COUNT(*) FROM (SELECT 1 FROM table LIMIT 200) z`
  - 200건까지는 정확한 레코드 건수를 조회할 수 있고, LIMIT을 초과하는 데이터는 조회하지 않는다
  - 뒤로 갈수록 성능이 느려지긴 하지만 대부분의 조회는 앞 페이지에서 중점적으로 발생하기 때문에 생각보다 개선효과가 크다
#### 4-2) COUNT 쿼리 없이 임의의 레코드 건수를 표현하는 방법이 있다
  - 우선 모든 페이지를 표시한 후, 실제 해당 페이지로 이동할 때 페이지 번호를 보정한다
  - 예를 들어 7번째 페이지를 클릭할 때 SELECT ALL 쿼리를 날리고, 그 결과에 따라 다른 동작을 수행한다
  - `SELECT * FROM table LIMIT 10 OFFSET 60`
  - 결과가 10개 미만이면 7페이지를 마지막 페이지로 표시해주고, 10개라면 페이지 번호를 그대로 유지한다
  - 만약 결과가 0개라면 COUNT ALL 쿼리를 실행해서 페이지 번호를 갱신해준다 (실제 레코드 건수가 많지 않기 때문에 성능적으로 큰 이슈가 되진 않는다)
  - 구현이 쉽지는 않지만, 제대로 구현된다면 매우 효율적인 방법일 수 있다
#### 4-3) 통계 정보를 이용하는 방법이 있다
- 쿼리 조건이 없는 경우엔 테이블 통계를 활용한다
  - MySQL에서는 Information Schema의 tables 뷰를 통해 특정 테이블의 전체 레코드 건수를 확인할 수 있다
  - COUNT ALL 쿼리가 WHERE 조건 없이 전체 레코드 건수를 조회해야 한다면 통계 정보에서 테이블의 레코드 예측치를 조회해서 사용할 수 있다
```sql
SELECT TABLE_ROWS as rows
FROM INFORMATION_SCHEMA.tables
WHERE schema_name=? AND table_name=?
```
- 쿼리 조건이 있는 경우엔 실행 계획을 활용한다
  - COUNT ALL의 WHERE 조건들이 인덱스를 타도록 튜닝하고, 쿼리의 실행 계획에서 rows 컬럼을 레코드 예측치로 사용할 수 있다
  - 정확도가 낮고, 조인이나 서브쿼리 사용 시에 계산 난이도가 높다는 단점이 있다
- 통계 정보를 이용하는 방법은 성능은 빠르지만, 비교적 덜 정확하므로 페이지를 이동하면서 보정해야 한다 (4-2 참고)
    
### 5) COUNT(*) 튜닝 대상 식별전략
#### 1) 제거하는 대상
  - WHERE 조건 없는 COUNT(*)
  - WHERE 조건에 일치하는 레코드 건수가 많은 COUNT(*)
#### 2_ 인덱스를 활용한 최적화 대상
  - 정확한 COUNT(*)가 필요한 경우
  - COUNT(*) 대상 건수가 소량인 경우
  - WHERE 조건이 인덱스로 처리될 수 있는 경우