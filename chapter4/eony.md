# 페이징 쿼리 작성

## 페이징 쿼리란?
- 원하는 데이터를 전체 데이터가 아닌 부분적으로 나눠서 조회 및 처리
- DB 및 애플리케이션 서버의 리소스 사용 효율 증가 및 처리 시간 단축

### 페이징 쿼리 작성하기
- LIMIT, OFFSET 사용하기
    - DBMS 서버에 더 많은 부하 발생시킬 가능성도 있음
        - 정해진 OFFSET의 레코드로 바로 갈 수는 없음 (그 전까지 순차적으로 읽는 작업이 필요함)
        - 따라서 OFFSET이 늘어날수록 읽어야하는 데이터가 늘어나 응답 시간이 길어짐
        - 총합을 따지면 오히려 모든 데이터를 한 번에 읽어서 가져가는 것보다 더 많은 리소스가 투입될 수 있음

> LIMIT, OFFSET을 사용하지 않으면서 페이징을 할 수 있도록 쿼리 작성할 필요가 있음!

### LIMIT, OFFSET 없이 페이징하기

(1) 범위기반 방식
- 날짜/숫자 범위로 데이터 나눠서 조회하는 방식
- WHERE 절에서 조회 범위 지정함, LIMIT 사용하지 않음 
- 주로 배치 작업 등에서 날짜/숫자(auto increment 값 등) 범위로 나눠 조회할 때 사용
- 여러 번 쿼리를 나누더라도 범위 값만 바꾸면 되므로 조회가 단순해짐

___

(2) 데이터 개수 기반 방식
- 지정된 데이터 건수만큼 결과 데이터 반환하는 방식
- 배치보다는 일반 서비스에서 주로 사용됨
- ORDER BY, LIMIT 절을 사용
- 처음 쿼리 실행할 때와 이후 쿼리를 실행할 때 쿼리 형태가 달라짐
    - 조회 전 적절한 인덱스 생성이 필요함    
- **WHERE 절에 따라서도 쿼리가 달라질 수 있음!**


#### WHERE절에 따라 쿼리가 달리지는 케이스 예시

아래 테이블이 있다고 가정

```sql
CREATE TABLE payments (
    id int NOT NULL AUTO_INCREMENT,
    user_id int NOT NULL,
    finished_at datetime NOT NULL,
    ...,
    PRIMARY KEY (id),
    KEY ix_finished_at_id (finished_at, id)
)
```
### WHERE에서 동등 조건 사용 시
_1회차 쿼리_
```sql
SELECT *
FROM payments
WHERE user_id = ?
ORDER BY id
LIMIT 30;
```

_N회차 쿼리_
```sql
SELECT *
FROM payments
WHERE user_id = ?
    ANS id > {이전 데이터의 마지막 id값}
ORDER BY id
LIMIT 30;
```

### WHERE에서 범위 조건 사용 시

_1회차 쿼리_
```sql
SELECT *
FROM payments
WHERE finished_at >= {시작날짜}
    AND finished_at < {종료날짜}
ORDER BY finished_at, id
LIMIT 30;
```
**동등 조건을 사용한 경우와 비교했을 때, ORDER BY에 finished_at이 포함되어있음**

이유?
- id 컬럼만 명시되는 경우
    - 조건을 만족하는 데이터(WHERE에 부합하는)를 모두 읽은 후 id로 정렬한 후 지정된 건수만큼을 반환함.
- finished_at을 사용하면 인덱스를 사용해서 정렬 작업 없이 원하는 건수만큼 순차적으로 데이터를 읽을 수 있음

_N회차 쿼리_
```sql
SELECT *
FROM payments
WHERE (finished_at = {N-1 쿼리의 마지막 finished_at 값} AND id > {N-1 쿼리의 마지막 id 값} 
OR (finished_at > {N-1 쿼리의 마지막 finished_at 값} AND finisehd_at < {조회할 범위}))
ORDER BY finished_at, id
LIMIT 30;
```

- 범위를 지정할 때 id로만 하면 누락되는 값이 생길 수 있음.
- 동일한 범위 값으로 저장된 케이스를 고려하여 OR 조건으로 포함할 수 있도록 쿼리 작성

-> but, id값만 추가해도 문제없는 케이스도 존재함
- finished_at과 id 순서가 동일한 케이스!
    - 하지만 id값과 finished_at의 최대 범위만 WHERE 조건에 넣으면, 이전에 조회된 데이터들에 해당하는 범위도 인덱스가 추가되므로 `{N-1}에 조회된 마지막 finished_at 값 < finished_at < 조회하고자 하는 범위` 는 유지할 수 있도록 해야함

___

## 정리
- LIMIT & OFFSET은 DB서버에 부하를 발생시키므로 지양
- 페이징 쿼리는 대표적으로 두 가지
    - 범위 기반
    - 데이터 개수 기반

- 범위 기반은 날짜/숫자 값을 기준으로 특정 범위에 대한 쿼리를 실행
    - 1회차 쿼리와 N회차 쿼리가 같음
- 데이터 개수 기반은 LIMIT & WHERE을 함께 사용하는 형태
    - 1회차 쿼리와 N회차 쿼리가 다름
    - 쿼리에 사용되는 조건에 따라 쿼리 형태가 달라지고 복잡해질 수 있으므로 신경써야함