
## 조인 최적화 알고리즘

MySQL은 여러 테이블을 조인할 때 최적의 실행 계획을 찾기 위해 다양한 알고리즘을 사용합니다. 테이블 수가 많아질수록 최적의 실행 계획을 찾는 것은 복잡해집니다.

### Exhaustive 검색 알고리즘
* 모든 가능한 테이블 조합을 검사하여 최적의 실행 계획을 찾는 방식
* 테이블이 n개일 때 n!개의 조합을 검사해야 하므로 테이블 수가 많을 경우 비효율적
* 10개 이상의 테이블을 조인할 경우 계획 수립에만 상당한 시간이 소요될 수 있음

### Greedy 검색 알고리즘
* 단계별로 최적의 조인 순서를 찾아가는 방식
* 모든 경우의 수를 검사하지 않고 각 단계에서 최적의 선택을 함
* 완벽한 계획을 보장하지는 않지만 대부분의 경우 효율적인 계획을 빠르게 수립
* MySQL 5.0부터 도입되어 대규모 조인 쿼리의 최적화 시간을 크게 단축

## 실행 계획의 중요성

실행 계획은 쿼리가 어떻게 처리될지를 보여주는 중요한 정보입니다. MySQL에서는 `EXPLAIN` 명령어를 통해 실행 계획을 확인할 수 있습니다.

```sql
EXPLAIN SELECT * FROM users JOIN orders ON users.id = orders.user_id WHERE users.status = 'active';
```

### 통계 정보의 활용

MySQL은 실행 계획을 수립할 때 다음과 같은 통계 정보를 활용합니다.

1. **테이블 및 인덱스 통계**
    * 테이블의 레코드 수, 인덱스의 카디널리티(유니크 값 개수) 등
    * MySQL 8.0에서는 통계 정보를 영구적으로 저장하여 서버 재시작 후에도 유지
    * `information_schema.statistics`, `mysql.innodb_table_stats` 테이블에서 확인 가능

2. **히스토그램**
    * 컬럼 값의 분포를 분석한 정보
    * 특정 값 범위에 얼마나 많은 데이터가 있는지 파악 가능
    * MySQL 8.0부터 지원되어 데이터 편향성을 고려한 더 정확한 실행 계획 수립
    * 다음 명령으로 생성 가능:
      ```sql
      ANALYZE TABLE users UPDATE HISTOGRAM ON email, status WITH 20 BUCKETS;
      ```

3. **코스트 모델**
    * 각 작업(디스크 읽기, 메모리 읽기, 인덱스 검색 등)의 비용을 수치화
    * 전체 쿼리 실행 비용을 계산하여 최적의 계획 선택
    * MySQL 8.0에서는 `mysql.server_cost`, `mysql.engine_cost` 테이블에서 조정 가능

## 실행 계획 분석 방법

### 주요 실행 계획 컬럼 이해하기

1. **id**
    * 쿼리 내 각 SELECT 문을 식별하는 번호
    * 같은 id는 하나의 SELECT 문에서 조인되는 테이블들

2. **select_type**
    * 쿼리의 유형을 나타냄
    * SIMPLE: 단순 SELECT
    * PRIMARY: 가장 바깥쪽 SELECT
    * SUBQUERY: 서브쿼리
    * DERIVED: 임시 테이블 생성이 필요한 서브쿼리
    * MATERIALIZED: 실체화된 서브쿼리

3. **table**
    * 참조하는 테이블 이름

4. **type**
    * 테이블 접근 방식을 나타내며 성능 지표로 중요
    * 성능 순으로: system > const > eq_ref > ref > range > index > ALL
    * const: 기본키나 유니크 키로 1건 조회
    * ref: 인덱스를 사용하여 동등 조건으로 조회
    * range: 인덱스를 사용한 범위 검색
    * ALL: 전체 테이블 스캔(가장 비효율적)

5. **key**
    * 실제 사용된 인덱스

6. **rows**
    * 검색해야 할 예상 레코드 수

7. **Extra**
    * 추가 정보
    * "Using index": 커버링 인덱스 사용
    * "Using where": WHERE 절로 필터링
    * "Using temporary": 임시 테이블 사용
    * "Using filesort": 정렬을 위한 추가 작업 필요

### 실행 계획 시각화

MySQL 8.0부터는 트리 형태로 실행 계획을 확인할 수 있어 이해하기 쉬워졌습니다.

```sql
EXPLAIN FORMAT=TREE SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

실행 시간 정보까지 확인하려면 아래와 같이 합니다

```sql
EXPLAIN ANALYZE SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

## 쿼리 힌트를 통한 최적화

옵티마이저가 항상 최적의 실행 계획을 선택하지는 않습니다. 이럴 때 개발자가 힌트를 통해 실행 계획에 영향을 줄 수 있습니다.

### 주요 쿼리 힌트

1. **인덱스 관련 힌트**
   ```sql
   SELECT * FROM users USE INDEX(idx_status) WHERE status = 'active';
   SELECT * FROM users FORCE INDEX(idx_email) WHERE email LIKE 'a%';
   SELECT * FROM users IGNORE INDEX(idx_created_at) ORDER BY created_at;
   ```

2. **조인 순서 제어**
   ```sql
   SELECT /*+ JOIN_ORDER(orders, users) */ * 
   FROM users JOIN orders ON users.id = orders.user_id;
   
   -- 레거시 문법
   SELECT STRAIGHT_JOIN * FROM orders, users WHERE orders.user_id = users.id;
   ```

3. **고급 옵티마이저 힌트**
   ```sql
   -- 인덱스 병합 비활성화
   SELECT /*+ NO_INDEX_MERGE(users) */ * FROM users 
   WHERE status = 'active' OR email LIKE 'test%';
   
   -- 해시 조인 강제
   SELECT /*+ HASH_JOIN(users, orders) */ * 
   FROM users JOIN orders ON users.id = orders.user_id;
   
   -- 특정 인덱스 사용 강제
   SELECT /*+ INDEX(users idx_email) */ * FROM users WHERE email LIKE 'a%';
   ```
