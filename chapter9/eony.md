# 에러 핸들링

## MySQL 에러구분
1. Gobal Error
    - Server, Client에서 공용으로 발생
2. Server Error
3. Client Error

일부 서버 에러는 클라이언트로 전달됨
    - 따라서 클라이언트에서 보여지는 에러는 클라이언트 혹은 서버 에러일 수 있음

에러 코드에 따라서 서버/클라이언트 에러를 구분하고 추적해나갈 수 있음
    
## MySQL 에러 포맷
3개의 파트로 구성됨
- Error No
- SQLState
- Error Message

### (1) Error No
- ~~4자리 정수~~ 최근에는 6자리 정수값을 이용한 문자열도 추가됨
- MySQL에서만 유효한 정보임
- 에러 번호의 구분
    - 1~999: 글로벌 에러
    - 1000~1999: 서버 에러
    - 2000~2999: 클라이언트 or 커넥터 에러
    - 3000~ :서버 에러
    - MY-010000~ :서버 에러
- 3500 이후 대역과 MY-010000 이후 대역
    - MySQL 8.0 이후 추가됨

### (2) SQL State
- 5글자 영문 숫자로 구성
- ANSI-SQL에서 제정한 Vendor **비의존적** 에러코드
- 두 파트로 구분됨
    - 앞 두글자: 상태값의 분류
        - 00: 정상
        - 01: 경고
        - 02: 레코드 없음
        - HY: ANSI-SQL에서 아직 표준 분류를 하지 않음 상태 (벤더사에 의존적인 상태값)
        - 나머지는 모두 에러
    - 뒷 세 글자: 주로 숫자값(가끔 영문), 각 분류별 상세 에러 코드 값

### (3) Error Message
- 사람이 인식할 수 있는 문자열
    - 변경 가능성이 비교적 높은 편이므로, 해당 값 자체로 애플리케이션 내부에서 예외 처리를 하면 안됨!
    - 버전/스토리지 엔진별로 다른 경우가 많으며, 문서에 명시되지 않는 케이스도 많음


#### 예시) `ERROR 1062 (23000): Duplicate entry 'abc...' for ...`
- `1062`: Error No
- `23000`: SQL STATE
- `Duplicate ...` : Error Message

## Error Handling

### (1) Error No를 비교해서 처리하는 경우
- MySQL의 스토리지 엔진에 종속적인 경우가 많음
- 에러 처리에 그렇게 적합한 값은 아닐 수 있음

Ex) 동일한 Duplicate Key에 대하여 스토리지 엔진에 따라 다른 번호를 가질 수 있음
- MySQL (NDB) - Error: **1022**
- MySQL (InnoDB, MyISAM) - Error: **1062**
- MySQL (NDB, Unique Constraint) - Error: **1169**

### (2) SQL State를 사용하는 경우
SQL State는 동일 종류의 에러에 대해 **동일한 값을 가짐**
- MySQL 서버의 스토리지 엔진간의 호환성 제공
- 다른 벤더사의 DBMS와의 호환성 제공 
    - ANSI-SQL에 의존적이기 때문에
        - 하지만 벤더사마다 ANSI-SQL을 해석하는 방식이 다를수도 있어서 꼭 100% 호환되는 것이 아닐수도 있음 (그러나 대부분에선 호환이 됨)
- But, SQL State가 미분류 상태(HY)인 경우가 존재함
    - 이럴 땐 SQL State보단 Error No를 사용하는 것이 권장됨
    - 버전 업그레이드되며 새로운 카테고리로 변경될 가능성도 있음

Ex) 동일 종류의 에러에 대하여 동일한 번호 가짐
- Error 1022 SQLSTATE: **23000** (ER_DUP_KEY)
- Error 1062 SQLSTATE: **23000** (ER_DUP_ENTRY)
- Error 1169 SQLSTATE: **23000** (ER_DUP_UNIQUE)

## Best Practice
- MySQL 서버에서 발생한 에러는 매우 다양하다.
- DBA와 개발자 간 가장 좋은 커뮤니케이션 방법은 SQL 문장과 Error 정보(Error No, SQL State, Message 포함)
- DBMS Server Error를 버리는 예외 핸들링은 비추천
    - Cause Exception을 그냥 버리지말자
- ORM 사용 시 DBMS 에러와 응용프로그램 에러 구분 필요
- DBMS 에러의 경우 반드시 함께 로깅

___

## 더 알아볼 것
- ANSI에 대해 알아보기
    - 그냥 SQL 표준 정도로만 알고 있음