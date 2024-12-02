# Generated 컬럼 & 함수 기반 인덱스

## Generated Column이란?
- "표현식"으로 정의된 컬럼
    - "표현식"?
        - 고정된 값, 함수 또는 다른 컬럼들에 대한 연산 조합 등이 해당
- 정의된 "표현식"에 따라 컬럼의 값이 자동으로 생성
- 사용자가 직접 값을 입력하거나 변경할 수 없음
- 두 가지 종류 존재
    - Virtual Generated Column (가상 컬럼)
    - Stored Generated COlumn (스토어드 컬럼)

## GENERATED Column 생성

예시)
```sql
ALTER TABLE test ADD COLUMN generated_column AS (col1 + col2) VIRTUAL;
```
- 기본적으로 VIRTUAL 타입으로 생성 & NULLABLE
- PRIMARY KEY로는 STORED 타입만 허용
- 하나의 테이블에서 가상 컬럼과 스토어드 컬럼 혼합해서 사용 가능

## GENERATED Column 종류
### (1) 가상 컬럼 (Virtual Generated Column)
- GENERATED_COLUMN 디폴트 타입
- **컬럼의 값을 디스크에 저장하지 않음**
    - **컬럼의 값은 레코드가 읽히기 전 또는 BEFORE 트리거 실행 직후 계산됨**
- 인덱스 생성 가능함
    - 인덱스 데이터는 **실제 값**이 디스크에 저장됨
    - 값 생성 / 수정 시 인덱스에 업데이트됨

### (2) 스토어드 컬럼 (Stored Generated Column)
- 컬럼의 값을 디스크에 저장함
- 컬럼의 값은 레코드가 INSERT / UPDATE 될 때 계산되어 저장
- 인덱스 생성 가능함

## Generated Column DDL 작업
- ALTER 명령으로 ADD / MODIFY / CHANGE / DROP / RENAME 가능
- 일반 컬럼을 스토어드 컬럼으로, 스토어드 컬럼을 일반 컬럼으로 변경 가능
    - 가상 컬럼은 일반 컬럼으로 전환 불가
        - 어찌보면 당연..? - 전환 시 가상 컬럼 값을 모두 디스크에 써줘야하기 때문일 듯
- 스토어드 컬럼 <-> 가상 컬럼 간 변경 불가
    - 새로 컬럼을 추가하고 삭제하는 식으로만 전환 가능
- Online DDL 지원여부
    - [공식문서 참고!](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-generated-column-operations)
    - DDL 명령 실행 시 실행 알고리즘 직접 명시하는 것을 권장 (실행 불가한 알고리즘 설정 시 에러가 발생하므로 예기치 못한 DDL 내부 프로세스를 방지할 수 있음)
    ```sql
    ALTER TABLE ...
    ALGORITHM=INSTANT; # 가장 빠른 알고리즘

    ALTER TABLE ...
    ALGORITHM=INPLACE, LOCK=NONE; # 그 다음
    ```
- 가상 컬럼 추가 또는 변경 시 사용 가능한 유효성 검사 옵션
    - WITHOUT VALIDATION
        - 기본 설정
        - 기존 데이터 무결성을 확인하지 않으며, 가능한 경우 in-place 방식으로 작업
        - 계산된 값이 컬럼 값 범위를 벗어날 수 있음 (경고 또는 에러 발생)
        - 처리시간 빠름
    - WITH VALIDATION
        - 테이블 데이터 복사 수행
        - 작업 중 DML 유입 시 잠금 대기 (메타데이터 락 대기) 발생
        - 작업 시 계산된 값이 컬럼 값 범위를 벗어나는 경우 명령문 실패
        - 처리시간 느림

## 인덱스 사용
- 일반 컬럼과 동일하게 쿼리에서 인덱스 사용 가능함
- Generated 컬럼명 대신 동일한 표현식을 사용해도 인덱스 사용 가능함
    - 대신 Generated 컬럼에 정의된 표현식과 **완전**하게 일치해야함!
        - 컬럼의 표현식이 (col1 + 1)인데 쿼리에서 (1 + col1) 사용하면 인덱스 사용 불가
- 주어진 조건값과 컬럼 타입도 동일해야함
- 동등연산, 비교연산, `BETWEEN`, `IN` 사용 시 최적화 적용됨

## 제한사항
- 표현식에 `비결정적 함수`, `스토어드 프로그램`, `변수`, `서브쿼리`는 사용 불가
- INSERT / UPDATE 시 Generated 컬럼에 직접 값을 지정할 수 없으며, 지정할 수 있는 값은 `DEFAULT`만 가능함

___

## Function Based Index란? (함수 기반 인덱스)
- 일반 인덱스는 컬럼 or 컬럼의 Prefix만 인덱싱이 가능함
- 함수 기반 인덱스는 "표현식"을 인덱싱 값으로 사용 가능
- 쿼리의 조건절에서 컬럼을 가공하는 경우에 유용하게 사용 가능함
    ```sql
    # 아래 쿼리 실행이 필요할 때,
    SELECT * FROM tab WHERE (col1 + col2) > 10; 
    
    # 아래의 인덱스를 추가할 수 있음
    CREATE INDEX f_index on tab ((col1 + col2));
    ```

### 동작 방식
- 정의된 인덱스와 동일한 가상 컬럼을 자동 생성 후, 해당 값으로 인덱싱
    - 자동생성된 가상컬럼은 일반적인 환경에선 확인이 불가함
    - 가상 컬럼의 이름은 `!hidden!{index_name}!{key_part}!{counter}` 형태로 지정, 타입도 자동 지정됨

### 사용방법
- 각각의 표현식은 반드시 괄호로 묶어서 명시
    - EX) `CREATE INDEX f_index ON tab ((col1 + col2), (col1 * col2))`
- 일반 컬럼과 함께 복합 인덱스로도 구성 가능
    - EX) `CREATE INDEX f_index ON tab (col1, (LOWER(col2)), col3(20))`
- 표현식 값에 ASC, DESC 지정 가능
- UNIQUE 설정 가능

### 활용 예시
(1) 문자열 값의 특정 부분에 대해서 조회
```sql
CREATE INDEX ix_email_domain ON users ((SUBSTRING_INDEX(email, '@', -1)));
```

(2) 일/월/연도 별 조회

```sql
CREATE INDEX ix_createdat_day ON events ((DAY(created_at)));
CREATE INDEX ix_createdat_month ON events ((MONTH(created_at)));
CREATE INDEX ix_createdat_year ON events ((YEAR(created_at)));
```

(3) 대소문자 구분없이 문자열 검색
```sql
CREATE INDEX ix_title ON books ((LOWER(title)));
```

(4) 계산된 값 조회
```sql
CREATE INDEX ix_discounted_price ON products ((price * (1 - discount_rate)));
```

(5) 해싱된 값 조회
```sql
CREATE INDEX ix_config_md5 ON metadata ((MD5(config)));
```

### 주의사항
- 인덱스 생성 후 실행 계획 반드시 확인 필요함!
    - 표현식을 **정확하게** 명시해야 인덱스가 사용 가능하므로, 쿼리 수행 전 인덱스 타는지 반드시 확인
- 표현식 결과의 데이터 타입을 명확하게 확인해서 조건값 지정
    - double, long, int, varchar 등.. 형 변환이 일어나는 컬럼들끼리 섞어서 사용하고 있지 않은지 확인이 필요함
- 기본적으로 일반 인덱스보다 추가적인 계산 비용이 발생함
    - 변경 잦은 컬럼 & 복잡한 표현식 사용 시 오버헤드 커질 수 있음

### 제한 사항
- 표현식이 Non-Deterministic 함수 사용 불가함
- 일반 컬럼을 단독으로 사용할 수 없고, Prefix 지정 컬럼은 키 값으로 지정 불가
- 공간 인덱스나 전문검색 인덱스 지원하지 않음
- Primary Key에 표현식 포함 불가

___

## 더 알아볼 것
- Online DDL
    - 실행 알고리즘 및 잠금
    - 예전부터 어떻게 돌아가는지 궁금했었음