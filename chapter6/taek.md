# Lateral Derived Table

## 1. Lateral Derived Table이란?
- Derived Table(파생 테이블)의 한 종류
  - 파생 테이블은 쿼리의 FROM 절에서 서브쿼리를 통해 생성되는 임시 테이블을 의미
- 일반적으로 Derived Table은 선행테이블의 컬럼을 참조할 수 없으나, Lateral Derived Table은 참조 가능하다
- 정의된 Derived Table 앞에 LATERAL 키워드를 붙여서 사용 가능
- LATERAL 키워드를 사용해서 선행 테이블의 컬럼을 참조하는 경우엔 결과 데이터가 선행 테이블에 의존적이어서 처리 순서에 영향을 미친다


## 2. Lateral Derived Table의 사용 사례
### 활용 예제 1) 종속 서브 쿼리의 다중 값 반환 
- SELECT 절에서 사용되는 서브 쿼리는 하나의 컬럼 값만 반환이 가능하다 (=스칼라 서브쿼리)
- SELECT 절내에서 파생 테이블의 여러 컬럼의 값을 가져오고 싶은 경우 서브 쿼리의 FROM절에서 LATERAL 키워드를 사용해 하나의 서브쿼리로 원하는 값들을 모두 조회할 수 있다

### 활용 예제 2) SELECT 절 내 연산 결과 반복 참조
- SELECT 절에서 계산된 각각의 컬럼 값은 동일한 SELECT 절 내의 다른 컬럼에서 참조할 수 없다
- 이것을 회피하기 위해 동일한 연산을 중복 기재해서 사용하게 되면 가독성이 떨어짐
- FROM절에서 LATERAL 키워드를 사용해 연산 결과 값을 직접 참조할 수 있다

### 활용 예제 3) 선행 데이터를 기반으로 한 데이터 분석
- Lateral Derived Table에서 조건을 명시해서 필터링된 선행 데이터를 참조하면, 처리 대상이 되는 레코드의 수를 효율적으로 줄일 수 있다 

### 활용 예제 4) Top N 데이터 조회
- Lateral Derived Table 내에서 인덱스 컬럼으로 정렬을 수행하고 LIMIT으로 결과를 제한한다
- 그 후 WHERE 조건을 통해 선행테이블의 레코드와 매칭되는 레코드만 추출한다
