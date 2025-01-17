# 풀스캔 쿼리 패턴 및 튜닝

## 1. 대표적인 풀스캔 쿼리 패턴 소개
### 1) 컬럼의 가공
- 연산
```mysql
SELECT * FROM tb1 WHERE count + 10 < 2000
```
- 함수
```mysql
SELECT * FROM tb1 WHERE MOD(id, 2) = 0
```
- 형변환
```mysql
SELECT * FROM tb1 WHERE str_column = 12345
```

-> 컬럼이 가공되면 컬럼에 인덱스가 존재하더라도 사용하지 못한다!

인덱스는 컬럼의 원본값을 기준으로 구성되어 있다.

형변환의 경우 한 가지 예외가 있다. 숫자 -> 문자열로 형변환하면 인덱스를 못 태우지만, 문자열 -> 숫자로 형변환하면 인덱스가 태워진다. 이는 MySQL 내부적으로 형변환 우선순위가 존재해서 문자열 -> 숫자를 우선 형변환하기 때문이다. 즉, 아래 케이스에선 비교값으로 주어진 '1234'가 숫자인 1234로 형변환된 것으로 해석할 수 있다.
```mysql
EXPLAIN SELECT * FROM users WHERE str_col = 3; -- 인덱스 X
EXPLAIN SELECT * FROM users WHERE num_col = '1234'; -- 인덱스 O
```

### 2) 인덱싱 되지 않은 컬럼을 조건절에 OR 연산과 함께 사용

```mysql
EXPLAIN SELECT * FROM users WHERE idx_col = '7' OR non_idx_col >= '2022-07-24 00:00:00'; -- 인덱스 X
```

### 3) 복합 인덱스의 컬럼들 중 선행 컬럼을 조건에서 누락
```mysql
EXPLAIN SELECT * FROM users WHERE second_idx_col >= '2022-07-24 00:00:00'; -- 인덱스 X
EXPLAIN SELECT * FROM users WHERE first_idx_col = '3' AND second_idx_col >= '2022-07-24 00:00:00'; -- 인덱스 O
```
인덱스를 구성하는 컬럼들의 순서가 인덱스 사용 여부를 결정하는 데 있어 중요하다. + 후행 컬럼만 조건에 명시하면 인덱스를 사용할 수 없다.

### 4) LIKE 연산에서 시작 문자열로 와일드 카드를 사용
```mysql
EXPLAIN SELECT * FROM users WHERE name LIKE '%Esther%'; -- 인덱스 X
EXPLAIN SELECT * FROM users WHERE name LIKE 'Esther%'; -- 인덱스 O
```

### 5) 정규식(REGEXP) 연산 사용
```mysql
EXPLAIN SELECT * FROM users WHERE name REGEXP '^Esther%'; -- 인덱스 X
```

### 6) 테이블 풀스캔이 인덱스 사용보다 더 효율적인 경우
```mysql
-- (group_name, count): ('A', 150012), ('B', 149982), ('C', 15), ('D', 15)
EXPLAIN SELECT * FROM users WHERE group_name IN ('A', 'B'); -- 인덱스 X
EXPLAIN SELECT * FROM users WHERE group_name IN ('C', 'D'); -- 인덱스 O
```
옵티마이저가 데이터 분포도에 따라 효율적인 처리를 위해 쿼리 실행 시 의도적으로  테이블 풀스캔을 사용할 수 있음 

+ Not Equal 조건과 IS NOT NULL 조건도 경우에 따라 인덱스를 탈 수 있음
```mysql
-- (dormant_at, count): (NULL, 299994), (NOT NULL, 30)
EXPLAIN SELECT * FROM users WHERE group_name NOT IN ('A', 'B'); -- 인덱스 O
EXPLAIN SELECT * FROM users WHERE group_name NOT IN ('C', 'D'); -- 인덱스 X
EXPLAIN SELECT * FROM users WHERE dormant_at IS NOT NULL; -- 인덱스 O
```
