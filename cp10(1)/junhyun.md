# 9.3.2 조인 최적화 알고리즘
조인 최적회에는 Exhaustive, Greedy 두가지 방법 있다.  
Exhaustive는 모두 비교하므로 조인 양이 많이 지면 느려짐.  
Greedy는 아래처럼 이루어짐. 양이 많이져도 빠름.  
![Image](https://github.com/user-attachments/assets/561def88-5dcf-427a-ad04-01a0f1233d39)
optimizer_search_depth 시스템 변수로 한번에 얼만큼씩 가능한 조인 조합 만들지 결정할 수 있다.  

optimizer_search_depth
- 기본값 62, 1-62는 greedy 
- 0으로 하면 옵티마이저가 Exhaustive, Greedy 중 자동 선택.
- 이 값보다 조인에 사용될 테이블 수가 작으면 Exhaustive 사용됨.

요즘 거는 Heuristic 검색이 활성화 되어있다.  
optimizer_prune_level 를 1로 하면 된다. 기본이 1이라 그냥 안 건들면 된다.  
optimizer_prune_level 0으로 만들면 진짜 큰일난다. 성능 떨어짐.  
혹시 누군가 0으로 사용하고 있으면 1로 해주고 돈 받아도 될 듯.  

# 9.4 쿼리힌트
# 9.4.1 인덱스 힌트
인덱스 힌트 보다는 옵티마이저 힌트 사용하도록!   
힌트 안 사용하는 방향이 좋지만 쿼리 바꾸거나 건들기 힘들때는 힌트 사용  

# 9.4.2 옵티마이저 힌트
- SET_VAR
  - 실행 계획을 바꿈  
  - 조인버퍼, 정렬용 버퍼 크기를 일시적으로 증가시켜 대용량 처리 쿼리의 성능 향상 용도로 사용 가능.
  - 모든 시스템 변수를 이 힌트로 조정할 수 없음.
---
# 10. 실행 계획

# 10.2 실행 계획 확인
EXPLAIN FORMAT=JSON, EXPLAIN FORMAT=TREE 이나 단순 테이블(기본)로 선택 가능  

## 10.2.2 쿼리의 실행 시간 확인
`EXPLAIN ANALYZE`로 실행 계획의 단계별로 소요된 시간 확인 가능.  
- 들여쓰기가 같은 레벨은 상단에 위치한 라인이 먼저 실행
- 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행  
  (책의 예시 참고)  

여기에는 actual time(실제 소요 시간, 두개로 되어있으면 첫번째는 평균, 두번째는 마지막 레코드 가져오는데 걸린 시간), rows(처리한 레코드 건수), loops(반복 횟수)가 표시된다.  

# 10.3 실행 계획 분석
표의 각 라인(레코드) = 쿼리 문장에서 사용된 테이블(서브쿼리로 임시 테이블을 생성한 경우 그 임시 테이블까지 포함)  
실행순서 = 위에서 아래 순서로(UNION 이나 상관 서브쿼리와 같은 경우 순서대로 표시되지 않을 수도 있다.)  

## 10.3.1 id 칼럼
- 단위 select쿼리 별로 부여
- select는 하나인데 여러 개 테이블이 조인되는 경우에는 id값이 같음
- id값이 테이블 접근 순서를 의미하는 건 아님

## 10.3.2 select_type 칼럼
각 단위 select 쿼리의 타입 표시  
### 10.3.2.1 SIMPLE
- union이나 서브쿼리가 없는 단순한 select 쿼리
- 쿼리가 복잡해도 실행 계획에서는 제일 바깥 select 하나만 SIMPLE로 표시됨

### 10.3.2.2 PRIMARY
- union이나 서브쿼리가 있는 select 쿼리
- 가장 바깥쪽에 있는 단위 쿼리가 PRIMARY로 표시됨
- SIMPLE처럼 하나만 존재 가능

### 10.3.2.3 UNION
- UNION으로 연결된 select 쿼리들 중 첫 번째 제외하고 두 번째 select부터
- 첫 번째는 derived로 표시됨

### 10.3.2.4 DEPENDENT UNION
- 외부 쿼리에 의해 영향 받는 것

### 10.3.2.5 UNION RESULT
- 8.0 부터는 union all 의 경우 임시 테이블 사용 안 함. (union, union distinct는 임시 테이블을 사용)
- 저 임시 테이블을 가리키는 라인이 UNION RESULT
- 실제 쿼리에서 단위 쿼리가 아니라 id 부여되지 않음.

### 10.3.2.6 SUBQUERY
- FROM절 이외에서 사용되는 서브쿼리
- FROM절에서 사용되는 서브쿼리는 DERIVED로 표시됨

### 10.3.2.7 DEPENDENT SUBQUERY
- 바깥쪽 select 쿼리에서 정의된 칼럼을 사용하는 경우

### 10.3.2.8 DERIVED
- DERIVED의 단위는 select 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것
- 파생 테이블 이라고도 함.
- 8.0 부터는 from절 서브쿼리 최적회도 개선되어 가능하다면 불필요한 서브쿼리는 조인으로 쿼리를 재작성해 처리.

### 10.3.2.9 DEPENDENT DERIVED
- 해당 테이블이 LATERAL JOIN으로 사용된 경우

### 10.3.2.10 UNCACHEABLE SUBQUERY
- 서브쿼리가 한 번만 사용되지 않을 수도 있어 서브쿼리의 결과를 내부적인 캐시 공간에 담아둠.
- 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능한 경우
  - 사용자 변수가 서브쿼리에 사용된 경우
  - not-deterministic 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
  - uuid()나 rand()와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우

### 10.3.2.11 UNCACHEABLE UNION

### 10.3.2.12 MATERIALIZED
- 주로 from절이나 in(subquery) 형태이ㅡ 쿼리에 사용된 서브쿼리의 최적화 위해서 사용됨.
- 서브쿼리의 내용을 임시 테이블로 구체화(materialization)해서 바깥과 조인하는 형태로 최적화

