# UNION vs UNION ALL

## 1. UNION
- JOIN은 결과셋의 컬럼을 확장하는 기능
- UNION은 결과셋의 로우를 확장하는 기능
- UNION는 인덱스를 사용하더라도 임시 테이블을 통한 가공작업이 필요하지만, 특정 경우엔 임시 테이블을 사용하지 않을 수 있다

## 2. UNION ALL vs UNION DISTINCT
- UNION ALL은 교집합에 대해 중복제거를 수행하지 않는다
    - 결과셋을 정렬하거나 임시 테이블에 저장할 필요가 없어서 성능이 빠르다
- UNION DISTINCT는 교집합에 대해 중복제거를 수행한다
    - 결과셋의 중복을 제거해야 하므로 성능이 느리다

## 3. UNION DISTINCT
중복 제거 방법
- 임시 테이블을 만들고, 결과셋에서 한건씩 조회하며 중복 레코드의 존재 여부에 따라 INSERT or DROP 여부를 결정한다
- 이때, 중복 여부는 PK나 UK가 아닌 레코드의 모든 컬럼 값 조합으로 확인한다

## 4. 정리
- 꼭 필요한 게 아니라면 성능 차이가 많이 나므로 UNION ALL 사용하자
- 중복 제거 필요 시엔 UNION ALL을 UNION DISTINCT로 바꾸고 애플리케이션에서 중복을 제거하는 것을 고려해보자
- MySQL에선 UNION 키워드만 단독 사용 시엔 UNION DISTINCT로 동작한다 

