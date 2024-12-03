# Generated 컬럼 및 함수 기반 인덱스

## 1. Generated 컬럼
- 표현식으로 정의되고, 정의된 표현식에 따라 컬럼의 값이 자동으로 생성되는 컬럼
  - 표현식이란? 고정된 값, 함수 또는 다른 컬럼들에 대한 연산 조합
- 사용자가 직접 값을 입력하거나 변경할 수 없음
- Virtual 타입, Stored 타입 두 가지 종류가 존재함
  - 기본적으로는 Virtual 타입으로 생성되고 NULL 값을 허용함
  - PK로는 Stored 타입만 허용
  - 하나의 테이블에서 Virtual 타입 컬럼과 Stored 타입 컬럼 혼용 가능

```sql
col_name data_type [GENERATED ALWAYS] AS (expr)
  [VIRTUAL | STORED] [NOT NULL | NULL]
  [UNIQUE [KEY]] [[PRIMARY] KEY]
  [COMMENT 'string']
```
```mysql
ALTER TABLE test ADD COLUMN generated_column AS (col1 + col2) VIRTUAL;
```

### 1) 가상 컬럼 (Virtual Generated Column)
- 타입을 지정하지 않거나 Virtual로 지정한 경우 가상 컬럼으로 생성
- 컬럼의 값을 디스크에 저장하지 않고, 레코드가 읽히기 전 or Before 트리거 실행 직후에 계산됨
- 인덱스 생성 가능 (인덱스 데이터는 디스크에 저장됨)

### 2) 스토어드 컬럼 (Stored Generated Column)
- 컬럼의 값을 디스크에 저장하고, 레코드가 INSERT 되거나 UPDATE 될 때 계산되어 저장
- 인덱스 생성 가능

## 2. Generated 컬럼 DDL
- 일반 컬럼을 스토어드 컬럼으로, 스토어드 컬럼을 일반 컬럼으로 변경 가능
- 가상 컬럼은 변경이 불가능하다
  - 가상 컬럼 <-> 일반 컬럼간 변경 불가
  - 가상 컬럼 <-> 토어드 컬럼간 변경 불가
- 알고리즘을 명시하지 않고 DDL을 실행하면 MySQL에서 잠금을 최소화하는 발생시키는 방향으로 동작하므로 예상과 다르게 동작할 수 있다 
- 따라서 알고리즘을 직접 명시해서 DDL을 실행시키는 것을 권장한다
  - 명시한 알고리즘을 실행시킬 수 없을 땐 에러가 발생하므로 예측 가능하게 동작한다
```mysql
ALTER TABLE test
    ADD COLUMN virtual_column AS (col1 + col2) VIRTUAL, 
    ALGORITHM=INPLACE, LOCK=NONE;
```
- 스토어드 컬럼은 값을 디스크에 저장하므로 자동적으로 유효성 검사를 수행하는 반면, 가상 컬럼은 키워드를 명시하여 옵셔널하게 유효성 검사를 수행할 수 있다 (WITH VALIDATION or WITHOUT VALIDATION)
  - WITH VALIDATION은 테이블 데이터 전체 복사가 발생하고, 그 동안 DML문이 잠금대기로 인해 실행되지 못하는 문제가 있으므로 검증은 별도로 수행하고 WITHOUT VALIDATION을 사용하는 것을 권장한다 

## 3. Generated 컬럼 Index
- 일반 컬과 동일하게 쿼리에서 인덱스 사용 가능
- 쿼리에 표현식을 사용해도 인덱스 사용이 가능하다
  - 단, 표현식은 컬럼에 정의된 표현식과 완전히 일치해야 하며, 표현식 결과값(=조건 우변의 조건값)의 타입도 완전히 일치해야 한다  
    - 컬럼은 (col + 1)인데, 쿼리는 (1 + co1)이라면 인덱스 사용이 불가능하다
  - 또한 조건의 연산자는 =, <, <= , >, >=, BETWEEN, IN 연산자로 제한된다

## 4. Generated 컬럼 제한사항
- 표현식에는 비결정적 함수, 스토어드 프로그램, 변수, 서브쿼리 사용이 불가능
- INSERT/UPDATE 시 Generated 컬럼에 직접 값 지정 불가 (DEFAULT 값만 지정 가능)

## 5. 함수 기반 인덱스
- 일반 인덱스는 컬럼 또는 컬럼의 Prefix만 인덱싱 가능
  - `CREATE INDEX ix_col ON tab (col);`
  - `CREATE INDEX ix_col20 ON tab (col(20))`
- 함수 기반 인덱스는 표현식을 인덱싱 값으로 사용 가능
  - `CREATE INDEX f_index ON tab ((col1 + col2), (col1 * col2));`
  - `CREATE INDEX f_index ON tab (DATE(col1));`
- 쿼리의 조건절에서 컬럼을 가공하는 경우에 유용하게 사용 가능
  - 사용하는 쿼리) `SELECT * FROM tab WHERE (col1 + col2) > 10;`
  - 쿼리를 위한 인덱스) `CREATE INDEX f_index ON tab ((col1 + col2));`
- **수식이나 함수를 사용한 쿼리가 빈번하게 호출될 수 있는 경우엔 함수 기반 인덱스를 사용하자**


## 6. 함수 기반 인덱스 동작 방식
- 함수 기반 인덱스를 생성하면 내부적으로 테이블의 인덱스에 지정된 표현식으로 정으된 가상 컬럼을 자동 생성 후 해당 가상 컬럼에 대해 인덱싱한다
  - 이 가상 컬럼은 스키마 조회 시 노출되지 않도록 숨겨져 있고, 별도의 디버그 환경을 설정한 경우에만 노출된다

## 7. 함수 기반 인덱스 사용 방법
- 각각의 표현식은 반드시 괄호로 묶어서 명시
  - `CREATE INDEX f_index ON tab ((col1 + col2), (col1 * col2));`
- 일반 컬럼과 함께 복합 인덱스로도 구성 가능
  - `CREATE INDEX f_index ON tab (col1, (LOWER(col2)), col3(20));`
- 표현식 값에 대해 ASC & DESC 지정하여 정렬 가능
- UNIQUE 설정 가능

## 8. 함수 기반 인덱스 활용 예시
- 문자열 값의 특정 부분에 대해서 조회
- 일/월/연도별 조회
- 대소문자 구분 없이 문자열 검색
- 계산된 값 조회
- 해싱된 값 조회
- 컬럼에 JSON 데이터를 담고, 이 JSON 안의 특정 필드에 대해 조회하는 경우 (19장에서 소개 예정)

## 9. 함수 기반 인덱스 주의사항
- 인덱스 생성 후 실행 계획을 반드시 확인
  - 표현식과 완전히 동일한지 체크
  - 표현식 결과의 데이터 타입을 명확하게 확인해서 조다값 지정
    - `mysql --column-type-info`를 사용하면 반환되는 필드들에 대한 메타 정보를 반환하므로 타입을 확인하기가 쉽다
  - 정확하게 명시하더라도 인덱스를 안 타는 경우가 있으므로 반드시 확인해야 함(BUG)
- 기본적으로 일반 인덱스보다 오버헤드가 크므로 변경이 잦은 컬럼에 대해 사용하거나 복잡한 표현식 사용은 지양해야 한다

## 10. 함수 기반 인덱스 제한사항
- 표현식에 비결정적(Non-Deterministic) 함수 사용 불가
  - 입력 값이 같더라도 호출할 떄마다 다른 결과를 반환하므로 값이 일관되지 않아 인덱싱 불가
- 일반 컬럼 및 Prefix 길이가 지정된 컬럼은 키 값으로 지정 불가
  - 괄호 없이 사용하거나, SUBSTRING 또는 CAST 함수를 사용하는 방식으로 우회 가능
- 공간 인덱스나 전문검색 인덱스는 지원하지 않음
- PK에 표현식은 포함 불가
  - 함수 기반 인덱스가 스토어드 컬럼이 아닌 가상 컬럼을 기반으로 하므로
  - 함수 기반 인덱스는 Generated 컬럼이나 가상 컬럼에 대한 제약사항을 모두 동일하게 가진다
