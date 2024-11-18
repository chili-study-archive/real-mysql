# COUNT(*) & COUNT(DISTINCT)

## **COUNT(*)는 생각보다 무겁다!**

`SELECT *` 와 `SELECT COUNT(*)`는 대부분의 애플리케이션에서 사용 가능성이 매우 높은 쿼리임.

하지만 두 쿼리 모두 WHERE 조건에 부합하는 데이터를 모두 읽어야하기 때문에 데이터 개수에 따라 속도가 느려질 수 있고, 따라서 쿼리 사용시 퍼포먼스 및 부하에 대해 살펴봐야함.

### 일반적인 기대
- `SELECT COUNT(*)` > `SELECT *`

### 실제
- `SELECT COUNT(*)`와 `SELECT *` 의 퍼포먼스는 유사함!

비즈니스 요건 상 일반적으로 `SELECT COUNT(*)`에선 `LIMIT`을 걸지 않지만 `SELECT *` 에선 `LIMIT`을 검.

따라서 `LIMIT` 없이 사용하는 `SELECT COUNT(*)` 쿼리는 특히 더 신경써서 사용해야함!

<br>

## 개선방안
### 1. 커버링 인덱스 유도하기

카운트 쿼리를 인덱스 컬럼들로만 구성하여 실제 데이터 블록 접근없이 인덱스 페이지만 접근하여 카운트를 빠르게 조회할 수 있음.

But, 사실상 모든 카운트 쿼리에 커버링 인덱스를 적용하는 것은 불가하며 카운트 쿼리 성능 향상 목적으로 인덱스 컬럼들을 마구 추가하면 오히려 성능이 안좋아짐.

### 2. 서비스 로직을 변경하는 방향을 고민해보기

#### (1) 카운트 쿼리 제거하기
- 일반적으로 전체 카운트가 필요한 경우는 전체 페이지 번호를 사용자 화면에 뿌려주기 위한 용도
- 이를 "이전", "다음" 버튼만 노출하는 형태로 서비스를 변경하면 카운트 쿼리를 아예 없앨 수 있음
- 사용자들이 최신의 데이터 몇 건에만 접근하는 것이 일반적인 유스케이스이므로 서비스 특성을 고려하여 검토해볼 수 있는 옵션임

#### (2) 대략적인 건수를 사용하기
- 표시할 페이지 번호만큼의 레코드만 건수 확인
    - 페이지 번호는 임의로 표시하되, 실제 해당 페이지로 이동하면 페이지 번호 보정하기 (실제로 구글에서 사용하는 방식임)
    - EX) `(SELECT COUNT(*) FROM (SELECT 1 FROM table LIMIT 200) z)`
- 통계 정보 활용하기
    - 쿼리 조건이 없는 경우 테이블 통계 활용 가능
    ```sql 
    SELECT TABLE_ROWS as rows FROM INFORMATION_SCHEMA.tables WHERE schema_name = ? AND table_name = ?;
    ```
    
    - 쿼리 조건이 있는 경우, 실행 계획을 활용 (`EXPLAIN`)
        - 정확도 낮음
        - 조인이나 서브쿼리 사용 시 계산 난이도 높음
    - 성능은 빠르지만 예측치이므로 페이지 이동하면서 보정이 필요함

### 카운트 쿼리 개선 방향에 대한 판단 기준

(1) 제거하는 것이 좋은 경우
- `WHERE` 조건 없는 카운트
- `WHERE` 조건에 일치하는 레코드 건수가 많은 카운트

(2) 인덱스 활용하여 최적화할 수 있는 경우
- 정확한 카운트 수 필요한 경우
- 카운트 대상 건수가 소량인 경우
- `WHERE` 조건이 인덱스로 처리될 수 있는 경우

<br>

### **주의점 - (COUNT(DISTINCT)) 사용 유의하기**

ORM을 통해 자동으로 쿼리를 생성하는 경우 카운트 쿼리가 `COUNT(DISTINCT)`로 생성되는 케이스가 존재하는데, 이는 쿼리 성능을 크게 낮출 수 있으므로 반드시 실제로 사용되는 쿼리를 확인해봐야함

#### **COUNT(*)와 COUNT(DISTINCT)는 퍼포먼스 차이가 매우 크다!**

`COUNT(*)`과 `COUNT(DISTINCT)` 내부 동작 비교

(1) `COUNT(*)`
- 조건에 맞는 레코드 건수만 확인함.

(2) `COUNT(DISTINCT)`
- 임시 테이블을 사용한 중복 제거 작업이 포함됨.
    - 조건에 맞는 데이터를 임시 테이블에 모두 밀어넣음
    - 임시 테이블에 `INSERT`하기 전에 먼저 `SELECT`를 날려서 중복된 값이 있는지 먼저 확인하고 없는 경우 `INSERT` 실행
    - 해당 작업을 모두 마친 후의 결과값을 반환하는 구조
- 임시 테이블은 메모리에서 공간을 할당받아 동작하는데, 임시 테이블에 쌓는 데이터가 어느 이상으로 많아지면 데이터 일부를 디스크에 쓰는 작업이 동반됨
    - DISK I/O가 추가되므로 그만큼 성능에 악영향을 미친다.
