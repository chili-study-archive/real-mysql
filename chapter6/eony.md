# Lateral Derived Table

Lateral Derived Table이란?
- 파생 테이블(Derived Table)은 쿼리의 FROM 절에서 서브쿼리를 통해 생성되는 임시 테이블임
- 일반적으로 선행 테이블의 컬럼을 참조할 수 없음
    - Lateral Derived Table에선 참조 가능!
    - Derived Table 앞에 Lateral 키워드 추가하여 사용
    - 참조한 값을 바탕으로 동적으로 결과 생성

## 동작 방식
예시쿼리)
```sql
SELECT e.emp_no, s.sales_count, s.total_sales
FROM employees e
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS sales_count, IFNULL(SUM(total_price), 0) AS total_sales
    FROM sales
    WHERE emp_no = e.emp_no /* 파생 테이블에서 선행 테이블은 employees e를 참조 */
) s ON TRUE /* 문법 준수를 위한 ON TRUE 임의 추가*/;
```

Lateral Derived Table로 사용된 테이블(sales)은 실행계획 확인 시 `select_type`에 `DEPENDENT DERIVED`가 명시됨

## 활용 예제

### (1) 종속 서브 쿼리의 다중 값 반환

_USECASE: 부서별 가장 먼저 입사한 직원의 입사일과 직원 이름을 조회_

> LATERAL DERIVED TABLE을 사용하지 않는 경우
```sql
SELECT d.dept_name,
    (SELECT e.hire_date AS earliest_hire_date, CONCAT(e.first_name, ' ', e.last_name) AS full_name
    FROM dept_emp de
    INNER JOIN employees e ON e.emp_no = de.emp_no
    WHERE de.dept_no = d.dept_no
    ORDER BY e.hire_date LIMIT 1)
FROM departments d;
```

Error 발생!
    - SELECT 절에서 서브쿼리 사용 시 하나의 컬럼 값만 반환 가능한데 두 개의 값을 반환함 (`earliest_hire_date`, `full_name`)

#### 해결 1

필요한 두 개의 컬럼을 각각의 서브쿼리로 분리해 가져오기

```sql
SELECT d.dept_name,
    (SELECT e.hire_date
    FROM dept_emp de
    INNER JOIN employees e ON e.emp_no = de.emp_no
    WHERE de.dept_no = d.dept_no
    ORDER BY e.hire_date LIMIT 1) AS earliest_hire_date,

    (SELECT CONCAT(e.first_name, ' ', e.last_name)
    FROM dept_emp de
    INNER JOIN employees e ON e.emp_no = de.emp_no
    WHERE de.dept_no = d.dept_no
    ORDER BY e.hire_date LIMIT 1) AS full_name

FROM departments d
```

But, 동일한 데이터를 가져오는 서브쿼리가 중복해서 실행되므로 비효율적

#### 해결 2

**FROM 절에서 LATERAL 키워드를 사용해 하나의 서브쿼리로 원하는 값들을 모두 조회**

```sql
SELECT d.dept_name, x.earliset_hire_date, x.full_name
FROM departments d
INNER JOIN LATERAL (
    SELECT e.hire_date AS earliset_hire_date, CONCAT(e.first_name, ' ', e.last_name) AS full_name
    FROM dept_emp de
    INNER JOIN employees e ON e.emp_no = de.emp_no
    WHERE de.dept_no = d.dept_no
    ORDER BY e.hire_Date LIMIT 1) x
```

___

### (2) SELECT절 내 연산 결과 반복 참조

_USECASE: 일별 매출 데이터를 조회하는 쿼리_
```sql
SELECT (total_sales * margin_rate) AS profit, ((total_sales * margin_rate) / total_sales_number) AS avg_profit, (expected_sales * margin_rate) AS expected_profit,
((total_sales * margin_rate) / (expected_sales * margin_rate) * 100) AS sales_achievement_rate
FROM daily_revenue
WHERE sales_date = '2023-12-01';
```

But, SELECT문 내 연산 결과를 참조하기 위해 동일한 연산(`total_sales * margin_rate`, `expected_sales * margin_rate`) 중복기재하여 사용

#### 해결 1

```sql
SELECT profit, avg_profit, expected_profit, sales_achievement_rate
FROM daily_revenue,
    LATERAL (SELECT (total_sales * margin_rate) AS profit) p,
    LATERAL (SELECT (profit / total_sales_number) AS avg_profit) ap,
    (SELECT (expected_sales * margin_rate) AS expected_profit) ep,
    (SELECT (profit / expected_profit * 100) AS sales_achievement_rate) sar
WHERE sales_date = '2023-12-01';
```

**FROM 절에서 LATERAL 키워드를 사용해 연산 결과 값 직접 참조**

___

### (3) 선행 데이터를 기반으로 한 데이터 분석

_USECASE: 처음 서비스에 가입하고나서 일주일내로 결제 완료한 사용자의 비율_
- 2024년 1월에 가입한 유저들 대상
- 사용자 관련 이벤트 저장하는 user_events 테이블 활용 (50만건이라고 가정)
    ```sql
    CREATE TABLE user_events (
        id int NOT NULL AUTO_INCREMENT,
        user_id int NOT NULL,
        event_type varchar(50) NOT NULL,
        ... ,
        created_at datetime NOT NULL,
        PRIMARY KEY(id),
        KEY ix_eventtype_user_id_createdat (event_type, user_id, created_at)
    )
    ```

_추출 플로우_
1. 2024 1월에 새로 가입한 유저목록 추출
2. 유저 아이디 & 가입일시 정보를 참조해 일주일 내 결제 유무 확인

#### LATERAL DERIVED TABLE 사용하지 않는 경우

```sql
SELECT SUM(sign_up) AS signed_up, SUM(complete_purchase) AS completed_purchase, (SUM(completed_purchase) / SUM(sign_up) * 100) AS conversion_rate
FROM (
    SELECT user_id, 1 AS sign_up, MIN(created_at) AS sign_up_time
    FROM user_events
    WHERE event_type = 'SIGN_UP'
    AND created_at >= '2024-01-01' AND created_at < '2024-02-01'
    GROUP BY user_id
) e1 LEFT JOIN (
    SELECT 1 AS completed_purchase, MIN(created_at) AS complete_purchase_time
    FROM user_events
    WHERE event_type = 'COMPLETE_PURCHASE'
    GROUP BY user_id
) e2 ON e2.user_id = e1.user_id
    AND e2.complete_purchase_time >= e1.sign_up_time
    AND e2.complete_purchase_time < DATE_ADD(e1.sign_up_time, INTERVAL 7 DAY);
```

두 개의 일반 DERIVED TABLE 사용

두 번째 DERIVED TABLE(`e2`)에서 `created_at`이 2024년 1월이 아닌 데이터까지 조회가 되어 비효율적임

#### LATERAL DERIVED TABLE 사용하는 경우
```sql
SELECT SUM(sign_up) AS signed_up, SUM(complete_purchase) AS completed_purchase, (SUM(complete_purchase) / SUM(sign_up) * 100) AS conversion_rate
FROM (
    SELECT user_id, 1 AS sign_up, MIN(created_at) AS sign_up_time
    FROM user_events
    WHERE event_type = 'SIGN_UP'
    AND created_at >= '2024-01-01' AND created_at < '2024-02-01'
    GROUP BY user_id
) e1 LEFT JOIN LATERAL (
    SELECT 1 AS complete_purchase
    FROM user_events
    WHERE user_id = e1.user_id
    AND event_type = 'COMPLETE_PURCHASE'
    AND created_at >= e1.sign_up_time
    AND created_At < DATE_ADD(e1.sign_up_time, INTERVAL 7 DAY)
    ORDER BY event_type, user_id, created_at
    LIMIT 1
) e2 ON TRUE;
```

선행 테이블인 `e1`에서 먼저 대상 유저들을 추출한 후, 추출된 유저들만을 대상으로 `e2` 값을 뽑아올 수 있음

두 쿼리 실행 시, 유의미한 퍼포먼스 차이 존재함

___

### (4) Top N 데이터 조회
_USECASE: 카테고리별 조회수가 가장 높은 3개 기사 추출

아래 테이블이 있다고 가정

```sql
CREATE TABLE categories (
    id int NOT NULL AUTO_INCREMENT,
    name varchar(50) NOT NULL,
    ... ,
    PRIMARY KEY (id)
);

CREATE TABLE articles (
    id int NOT NULL AUTO_INCREMENT,
    category_id int NOT NULL,
    title varchar(255) NOT NULL,
    views int NOT NULL,
    ... ,
    PRIMARY KEY (id),
    KEY ix_categoryid_views (category_id, views)
);
```

#### LATERAL DERIVED TABLE 사용하지 않는 경우
```sql
SELECT x.name, x.title, x.views
FROM (
    SELECT c.name, a.title, a.views,
        ROW_NUMBER() OVER
            (PARTITION BY a.category_id ORDER BY a.views DESC) AS article_rank
    FROM categories c
    INNER JOIN articles a ON a.category_id = c.id
) x
WHERE x.article_rank <= 3
```

실행 시 내부 로직
1. articles 데이터 우선 조회 및 정렬
2. categories와 JOIN

-> JOIN을 위해 articles 전체 데이터를 불필요하게 다 읽기 때문에 비효율적인 쿼리가 발생함

#### LATERAL DERIVED TABLE을 사용하는 경우
```sql
SELECT c.name, a.title, a.views
FROM categories c
INNER JOIN LATERAL (
    SELECT category_id, title, views
    FROM articles
    WHERE category_id = c.id
    ORDER BY category_id DESC, views DESC
    LIMIT 3
) a
```

실행 시 내부 로직
1. categories 데이터 우선 조회
2. `WHERE category_id = c.id`에 따라 articles 데이터를 조회
3. `ix_categoryid_views` 인덱스를 활용해 빠르게 데이터 조회
    - 인덱스 정렬을 통해 별도 정렬 작업도 불필요함