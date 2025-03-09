## 9.4 쿼리 힌트
- MySQL 서버는 우리의 서비스를 100% 이해하지 못하기 때문에 부족한 실행 계획을 수립할 때가 있을 수 있어서 옵티마이저에게 쿼리의 실행 계획을 어떻게 수립해야 할지 알려줄 수 있는 방법 필요

### 9.4.1 인덱스 힌트
- `STRAIGHT_JOIN`과 `USE INDEX` 등을 포함한 인덱스 힌트들은 모두 MySQL 서버에 옵티마이저 힌트가 도입되기 전에 사용되던 기능들
- SQL 문법에 맞게 사용해야 하기 때문에 사용하게 되면 ANSI-SQL 표준 문법을 준수하지 못하게 되는 단점이 있음 => 가능하다면 인덱스 힌트보다는 옵티마이저 힌틀르 사용할 것을 추천
- SELECT 명령과 UPDATE 명령에서만 사용 가능

#### 9.4.1.1 STRAIGHT_JOIN
- SELECT, UPDATE, DELETE 쿼리에서 여러 개의 테이블이 조인되는 경우 조인 순서를 고정하는 역할
```sql
EXPLAIN
SELECT * FROM employees e, dept_emp de, departments d
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;
```
- 옵티마이저가 그때그때 각 테이블의 통계 정보와 쿼리의 조건을 기반으로 가장 최적이라고 판단되는 순서로 조인
- 이 쿼리의 조인 순서를 변경하려는 경우에는 STRAIGHT_JOIN 힌트 사용 가능
```sql
SELECT STRAIGHT_JOIN e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;
```
- STRAIGHT_JOIN 힌트는 옵티마이저가 FROM 절에 명시된 테이블의 순서대로 조인을 수행하도록 유도 => 여기에서는 employees, dept_emp, departments 순서대로 조인 수행!
- 주로 다음 기준에 맞게 조인 순서가 결정되지 않는 경우에만 STRAIGHT_JOIN 힌트로 조인 순서 조정
  - 임시 테이블(인라인 뷰 또는 파생된 테이블)과 일반 테이블의 조인: 일반적으로 임시 테이블을 드라이빙 테이블로 선정하는 것이 좋음
  - 임시 테이블끼리 조인: 크기가 작은 테이블을 드라이빙으로 선택하는 것이 좋음
  - 일반 테이블끼리 조인: 양쪽 테이블 모두 조인 칼럼에 인덱스가 있거나 양쪽 테이블 모두 조인 칼럼에 인덱스가 없는 경우에는 레코드 건수가 적은 테이블을 드라이빙으로 선택해주는 것이 좋으며, 그 이외의 경우에는 조인 칼럼에 인덱스가 없는 테이블을 드라이빙으로 선택하는 것이 좋음
<details>
<summary>🔹 왜 조인 대상 테이블에 인덱스가 있는 것이 중요한가?</summary>

조인 과정에서 드라이빙 테이블의 각 행을 조인 대상 테이블에서 검색하는데,
이때 조인 대상 테이블에 인덱스가 없으면 매번 전체 테이블을 스캔(Full Table Scan)해야 함.
즉, 데이터 양이 많을수록 조인 속도가 기하급수적으로 느려짐.
</details>

- STRAIGHT_JOIN 힌트와 비슷한 역할을 하는 옵티마이저 힌트
  - JOIN_FIXED_ORDER
  - JOIN_ORDER
  - JOIN_PREFIX
  - JOIN_SUFFIX
- STRAIGHT_JOIN 힌트는 한 번 사용되면 FROM 절의 모든 테이블에 대해 조인 순서가 결정되는 효과를 냄
- JOIN_FIXED_ORDER 옵티마이저 힌트는 STRAIGHT_JOIN 힌트와 동일한 효과를 내고 나머지 3개는 일부 테이블의 조인 테이블의 조인 순서에 대해서만 제안하는 힌트

#### 9.4.1.2 USE INDEX/ FORCE INDEX/ IGNORE INDEX
- 사용하려는 인덱스를 가지는 테이블 뒤에 힌트를 명시해야 함
- 보통 옵티마이저는 어떤 인덱스를 사용해야 할지를 잘 선택하는 편이지만 3~4개 이상의 칼럼을 포함하는 비슷한 인덱스가 여러 개 존재하는 경우 가끔 실수함
- 별도로 사용자가 부여한 이름이 없는 프라이머리 키는 `PRIMARY`라고 명시
- USE INDEX
  - MySQL 옵티마이저에게 특정 테이블의 인덱스를 사용하도록 권장하는 힌트
- FORCE INDEX
  - USE INDEX보다 옵티마이저에게 미치는 영향이 더 강한 힌트
  - 거의 사용할 필요 없음
- IGNORE INDEX
  - 특정 인덱스를 사용하지 목하게 하는 용도로 사용하는 힌트
  - 때로는 옵티마이저가 풀 테이블 스캔을 사용하도록 유도하기 위해 IGNORE INDEX 힌트를 사용할 수도 있음
- 인덱스 용도
  - USE INDEX FOR JOIN: 테이블 간의 조인뿐만 아니라 레코드를 검색하기 위한 용도까지 포함
  - USE INDEX FOR ORDER BY
  - USE INDEX FOR GROUP BY
```sql
SELECR * FROM USE INDEX(primary) WHERE emp_no=10001;
```
- 가장 훌륭한 최적화는 그 쿼리를 서비스에서 없애버리거나 튜닝할 필요가 없게 데이터를 최소화하는 것이며, 그것이 어렵다면 데이터 모델의 단순화를 통해 쿼리를 간결하게 만들고 힌트가 필요치 않게 하는 것
- 어떤 방법도 없다면 그다음으로는 힌트를 선택하는 것인데, 일반적으로 실무에서는 앞쪽의 작업들에 상당한 시간과 작업 능력이 필요하기 때문에 항상 이런 힌트에 의존하는 경우 많음

#### 9.4.1.3 SQL_CALC_FOUND_ROWS
- MySQL의 LIMIT을 사용하는 경우, 조건을 만족하는 레코드가 LIMIT에 명시된 수보다 더 많다고 하더라도 LIMIT에 명시된 수만큼 만족하는 레코드를 찾으면 즉시 검색 작업 멈춤. 하지만 SQL_CALC_FOUND_ROWS 힌트가 포함된 쿼리의 경우 LIMIT을 만족하는 수만큼의 레코드를 찾았다고 하더라도 끝까지 검색 수행
- SQL_CALC_FOUND_ROWS 힌트가 사용된 쿼리가 실행된 경우에는 FOUND_ROWS()라는 함수를 이용해 LIMIT을 제외한 조건을 만족하는 레코드가 전체 몇 건이었는지 알 수 있음
```sql
SELECT SQL_CALC_FOUND_ROWS * FROM employees LIMIT 5;
SELECT FOUND_ROWS() AS total_record_count;
```
- 위의 경우SQL_CALC_FOUND_ROWS 힌트 때문에 조건을 만족하는 레코드 전부를 읽어 봐야 함. 그래서 인덱스를 통해 실제 데이터 레코드를 찾아가는 작업과 디스크 헤드가 특정 위치로 움직일 때까지 기다려야 하는 랜덤 I/O가 레코드 수만큼 발생
- 전기적 처리인 메모리나 CPU의 연산 작업에 비해 기계적 처리인 디스크 작업이 얼마나 느린 작업인지를 고려하면 비교할 수도 없을 만큼 SQL_CALC_FOUND_ROWS를 사용하는 경우가 느리다는 것을 쉽게 알 수 있음
- SQL_CALC_FOUND_ROWS는 성능 향상을 위해 만들어진 힌트가 아니라 개발자의 편의를 위해 만들어진 힌트

### 9.4.2 옵티마이저 힌트
- MySQL 5.6 버전부터 새롭게 추가되기 시작한 힌트들

#### 9.4.2.1 옵티마이저 힌트 종류
- 인덱스: 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
- 테이블: 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
- 쿼리 블록: 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트로서, 특정 쿼리 블록의 이름을 명시하는 것이 아니라 힌트가 명시된 쿼리 블록에 대해서만 영향을 미치는 옵티마이저 힌트
- 글로벌(쿼리 전체): 전체 쿼리에 대해서 영향을 미치는 힌트

#### 9.4.2.2 MAX_EXECUTION_TIME
```sql
SELECT /*+ MAX_EXECUTION_TIME(100) */ * FROM employees
ORDER BY last_name LIMIT 1;
```
- 쿼리의 최대 실행 시간을 설정하는 힌트
- 옵티마이저 힌트 중에서 유일하게 쿼리의 실행 계획에 영향을 미치지 않는 힌트
- 밀리초 단위의 시간을 설정
- 영향 범위: 글로벌

#### 9.4.2.3 SET_VAR
```sql
EXPLAIN SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off' */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```
- 쿼리 실행을 위한 시스템 변수 제어
- 영향 범위: 글로벌
- join_buffer_size 시스템 변수는 쿼리에 아무런 영향을 미치지 않을 것 같지만 옵티마이저는 조인 버퍼의 공간이 충분하면 조인 버퍼를 활용하는 형태의 실행 계획을 선택할 수 있음
- 옵티마이저 힌트로 부족한 경우 optimizer_switch 시스템 변수를 제어해야 할 수도 있음
- SET_VAR 힌트는 실행 계획을 바꾸는 용도뿐만 아니라 조인 버퍼나 정렬용 버퍼(소트 버퍼)의 크기를 일시적으로 증가시켜 대용량 처리 쿼리의 성능을 향상시키는 용도로도 사용 가능
- 모든 시스템 변수를 SET_VAR 힌트로 조정할 수는 없음

#### 9.4.2.4 SEMIJOIN & NO_SEMIJOIN
- 서브쿼리의 세미 조인 최적화 전략 제어
- 영향 범위: 쿼리 블록
- Table Pull-out 최적화 전략은 그 전략을 사용할 수 있다면 항상 더 나은 성능을 보장하기 때문에 별도로 힌트 사용 불가

|최적화 전략|힌트|
|---|---|
|Duplicate Weed-out|SEMIJOIN(DUPSWEEDOUT)|
|First Match|SEMIJOIN(FIRSTMATCH)|
|Loose Scan|SEMIJOIN(LOOSESCAN)|
|Materialization|SEMIJOIN(MATERIALIZATION)|
|Table Pull-out|없음|

```sql
EXPLAIN
SELECT /*+ SEMIJOIN(@subq1 MATERIALIZATION) */ * FROM departments d
WHERE d.dept_no IN (SELECT /*+ QB_NAME(subq1) */ de.dept_no FROM dept_emp de);
SELECT * FROM departments d WHERE d.dept_no IN (SELECT /*+ NO_SEMIJOIN(DUPSWEEDOUT, FIRSTMATCH) */ de.dept_no FROM dept_emp de);
```

#### 9.4.2.5 SUBQUERY
- 서브쿼리의 세미 조인 최적화 전략 제어
- 세미 조인 최적화가 사용되지 못할 때 사용하는 최적화 방법
- 영향 범위: 쿼리 블록
- 세미 조인 최적화는 주로 IN(subquery) 형태의 쿼리에 사용될 수 있지만 안티 세미 조인의 최적화에는 사용될 수 없음

|최적화 전략|힌트|
|---|---|
|IN-to-EXISTS|SUBQUERY(INTOEXISTS)|
|Materialization|SUBQUERY(MATERIALIZATION)|

#### 9.4.2.6 BNL & NO_BNL & HASHJOIN & NO_HASHJOIN
- MySQL 8.0.20 버전부터는 해시 조인 알고리즘이 블록 네스티드 루프 조인까지 대체하도록 개선
- MySQL 8.0.20과 그 이후 버전에서 BNL 힌트와 NO_BNL 힌트는 사용 가능하지만 HASHJOIN과 NO_HASHJOIN 힌트는 MySQL 8.0.18 버전에서만 유효하며, 그 이후 버전에서는 효력 없음
- 영향 범위: 쿼리 블록, 테이블

#### 9.4.2.7 JOIN_FIXED_ORDER & JOIN_ORDER & JOIN_PREFIX & JOIN_SUFFIX
- STRAIGHT_JOIN의 단점을 보안하기 위해 4개의 힌트 제공
- JOIN_FIXED_ORDER: STRAIGHT_JOIN 힌트와 동일하게 FROM 절의 테이블 순서대로 조인 실행
- JOIN_ORDER: 힌트에 명시된 테이블 순서대로 조인 실행
- JOIN_PREFIX: 힌트에 명시된 테이블을 조인의 드라이빙 테이블로 조인 실행
- JOIN_SUFFIX: 힌트에 명시된 테이블을 조인의 드리븐 테이블(가장 마지막에 조인돼야 할 테이블들)로 조인 실행
- 영향 범위: 쿼리 블록

#### 9.4.2.8 MERGE & NO_MERGE
- FROM 절의 서브쿼리나 뷰를 외부 쿼리 블록으로 병합하는 최적화를 수행할지 여부 제어
- 영향 범위: 테이블
- 예전 버전의 MySQL 서버에서는 FROM 절에 사용된 서브쿼리를 항상 내부 임시 테이블로 생성 -> 불필요한 자원 소모 유발
- MySQL 5.7과 8.0 버전에서는 가능하면 임시 테이블을 사용하지 않게 FROM 절의 서브쿼리를 외부 쿼리와 병합하는 최적화 도입
- 하지만 MySQL 옵티마이저가 최적의 방법을 선택하지 못할 때 MERGE & NO_MERGE 힌트 사용

#### 9.4.2.9 INDEX_MERGE & NO_INDEX_MERGE
- 인덱스 병합 실행 계획 사용 여부 제어
- 영향 범위: 테이블, 인덱스
- MySQL 서버는 가능하다면 테이블당 하나의 인덱스만을 이용해 쿼리 처리
- 인덱스 머지: 하나의 테이블에 대해 여러 개의 인덱스를 동시에 사용하는 것

#### 9.4.2.10 NO_ICP
- ICP(인덱스 컨디션 푸시다운) 최적화 전략 사용 여부 제어
- 영향 범위: 테이블, 인덱스
- ICP 최적화는 사용 가능하다면 항상 성능 향상에 도움이 되므로 옵티마이저는 최대한 ICP 기능을 사용하는 방향으로 실행 계획 수립
- 그래서 MySQL 옵티마이저에서는 ICP 힌트는 제공되지 않음

#### 9.4.2.11 SKIP_SCAN & NO_SKIP_SCAN
- 인덱스 스킵 스캔 사용 여부 제어
- 영향 범위: 테이블, 인덱스
- 인덱스의 선행 칼럼에 대한 조건이 없어도 옵티마이저가 해당 인덱스를 사용할 수 있게 해주는 매우 훌륭한 최적화 기능
- 조건이 누락된 선행 칼럼이 가지는 유니크한 값의 개수가 많아진다면 인덱스 스킵 스캔의 성능은 오히려 더 떨어짐

#### 9.4.2.12 INDEX & NO_INDEX
- GROUP BY, ORDER BY, WHERE 절의 처리를 위한 인덱스 사용 여부 제어
- 영향 범위: 인덱스
- 예전 MySQL 서버에서 사용되던 인덱스 힌트를 대체하는 용도로 제공

|인덱스 힌트|옵티마이저 힌트|
|---|---|
|USE INDEX|INDEX|
|USE INDEX FOR GROUP BY|GROUP_INDEX|
|USE INDEX FOR ORDER BY|ORDER_INDEX|
|IGNORE INDEX|NO_INDEX|
|IGNORE INDEX FOR GROUP BY|NO_GROUP_INDEX|
|IGNORE INDEX FOR ORDER BY|NO_ORDER_INDEX|

```
EXPLAIN SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE first_name='Matt';
```

# CHAPTER 10. 실행 계획
## 10.1 통계 정보
- 5.7 버전까지 테이블과 인덱스에 대한 개괄적인 정보를 가지고 실행 계획 수립
- 8.0 버전부터는 인덱스되지 않은 칼럼들에 대해서도 데이터 분포도를 수집해서 저장하는 히스토그램 정보 도입

### 10.1.1 테이블 및 인덱스 통계 정보
- 비용 기반 최적화에서 가장 중요한 것은 통계 정보

#### 10.1.1.1 MySQL 서버의 통계 정보
- 5.6 버전부터 InnoDB 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적으로 관리할 수 있게 개선됨
  - 각 테이블의 통계 정보를 mysql 데이터베이스의 innodb_index_stats 테이블과 innodb_table_stats 테이블로 관리 가능
- 5.6 버전에서 테이블을 생성할 때는 STATS_PERSISTENT 옵션 설정 가능 -> 이 설정값에 따라 테이블 단위로 영구적인 통계 정보를 보관할지 말지 결정 가능
```sql
CREATE TABLE tab_test (fd1 INT, fd2 VARCHAR(20), PRIMARY KEY(fd1)) ENGINE=InnoDB
STATS_PERSISTENT={ DEFAULT | 0 | 1 }
```
- STATS_PERSISTENT=0: 테이블의 통계 정보를 5.5 버전 이전의 방식대로 관리하며, msyql 데이터베이스의 innodb_index_stats와 innodb_table_stats 테이블에 저장하지 않음
- STATS_PERSISTENT=1: 테이블의 통계 정보를 mysql 데이터베이스의 innodb_index_stats와 innodb_table_stats 테이블에 저장
- STATS_PERSISTENT=DEFAULT: 테이블을 생성할 때 별도로 STATS_PERSISTENT 옵션을 설정하지 않은 것과 동일하며, 테이블의 통계를 영구적으로 관리할지 말지를 innodb_stats_persistent 시스템 변수의 값으로 결정
- innodb_stats_persistent 시스템 설정 변수는 기본적으로 ON(1)으로 설정되어 있음

### 10.1.2 히스토그램
- 8.0 버전으로 업그레이드되면서 MySQL 서버도 드디어 칼럼의 데이터 분포도를 참조할 수 있는 히스토그램 정보를 활용 가능

#### 10.1.2.1 히스토그램 정보 수집 및 삭제
- 히스토그램 정보는 칼럼 단위로 관리됨
- 이는 자동으로 수집되지 않고 `ANALYZE TABLE ... UPDATE HISTOGRAM` 명령을 실행해 수동으로 수집 및 관리
- 수집된 히스토그램 정보는 딕셔너리에 함께 저장되고 MySQL 서버가 시작될 때 딕셔너리의 히스토그램 정보를 information_schema 데이터베이스의 column_statistics 테이블로 로드
- 실제 히스토그램 정보를 조회하려면 column_statistics 테이블을 SELECT해서 참조 가능
- MySQL 8.0 버전에서는 2종류의 히스토그램 타입 지원
  - Singleton(싱글톤 히스토그램)
    - 칼럼값 개별로 레코드 건수를 관리하는 히스토그램으로, Value-Based 히스토그램 또는 도수 분포라고도 불림
    - 칼럼이 가지는 값별로 버킷이 할당
    - 각 버킷이 칼럼의 값과 발생 빈도의 비율의 2개 값을 가짐
    - 주로 코드 값과 같이 유니크한 값의 개수가 상대적으로 적은(히스토그램의 버킷 수보다 적은) 경우 사용됨
  - Equi-Height(높이 균형 히스토그램)
    - 칼럼값의 범위를 균등한 개수로 구분해서 관리하는 히스토그램으로, Height-Balanced 히스토그램이라고도 불림
    - 개수가 균등한 칼럼값의 범위별로 하나의 버킷이 할당됨
    - 각 버킷이 범위 시작 값과 마지막 값, 그리고 발생 빈도율과 각 버킷에 포함된 유니크한 값의 개수 등 4개의 값을 가짐
- 히스토그램의 모든 레코드 건수 비율은 누적으로 표시됨
- information_schema.column_statistics 테이블의 HISTOGRAM 칼럼이 가진 나머지 필드들은 다음과 같은 의미를 가짐
  - sampling-rate: 히스토그램 정보를 수집하기 위해 스캔한 페이지의 비율 저장
  - histogram-type: 히스토그램의 종류 저장
  - number-of-buckets-specified: 히스토그램을 생성할 때 설정했던 버킷의 개수 저장. 기본 100개의 버킷 사용됨
  - 8.0.19버전부터 InnoDB 스토리지 엔진 자체적으로 샘플링 알고리즘을 구현했으며, 더이상 히스토그램 수집 시 풀 테이블 스캔이 필요치 않게 됨
```sql
ANALYZE TABLE employees.employees DROP HISTOGRAM ON gender, hire_date;
```
- 히스토그램의 삭제 작업은 테이블의 데이터를 참조하는 것이 아니라 딕셔너리의 내용만 삭제하기 때문에 다른 쿼리 처리의 성능에 영향을 주지 않고 즉시 완료되지만 쿼리의 실행 계획이 달라질 수 있음에 주의
``` sql
SET GLOBAL optimizer_switch='condition_fanout_filter=off';
```
- 옵티마이저가 히스토그램을 사용하지 않게 하려면 optimizer_switch 시스템 변수의 값 변경하면 됨

#### 10.1.2.2 히스토그램의 용도
- 기존 MySQL 서버가 가지고 있던 통계 정보는 테이블의 전체 레코드 건수와 인덱스된 칼럼이 가지는 유니크한 값의 개수 정도
- 이러한 단점을 보완하기 위해 히스토그램이 도입
- 히스토그램은 특정 칼럼이 가지는 모든 값에 대한 분포도 정보를 가지지는 않지만 각 범위(버킷)별로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정확한 예측 가능
- 히스토그램 정보가 없으면 옵티마이저는 데이터가 균등하게 분포돼 있을 것으로 예측하지만 히스토그램이 있으면 특정 범위의 데이터가 많고 적음을 식별 가능

#### 10.1.2.3 히스토그램과 인덱스
- MySQL 서버에서는 쿼리의 실행 계획을 수립할 때 사용 가능한 인덱스들로부터 조건절에 일치하는 레코드 건수를 대략 파악하고 최종적으로 가장 나은 실행 계획을 선택
- 인덱스 다이브: 조건절에 일치하는 레코드 건수를 예측하기 위해 옵티마이저는 실제 인덱스의 B-Tree를 샘플링해서 살펴봄
- MySQL 8.0 서버에서는 인덱스된 칼럼을 검색 조건으로 사용하는 경우 그 칼럼의 히스토그램은 사용하지 않고 실제 인덱스 다이브를 통해 직접 수집한 정보를 활용함
- 이는 실제 검색 조건의 대상 값에 대한 샘플링을 실행하는 것이므로 항상 히스토그램보다 정확한 결과를 기대할 수 있기 때문
- 그래서 MySQL 8.0 버전에서 히스토그램은 주로 인덱스되지 않은 칼럼에 대한 데이터 분포도를 참조하는 용도로 사용됨
- 하지만 인덱스 다이브 작업은 어느 정도의 비용이 필요하며, 때로는 (IN 절에 값이 많이 명시된 경우) 실행 계획 수립만으로도 상당한 인덱스 다이브를 실행하고 비용도 그만큼 커짐

### 10.1.3 코스트 모델
- MySQL 서버가 쿼리를 처리하기 위해 필요로 하는 작업들
  - 디스크로부터 데이터 페이지 읽기
  - 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
  - 인덱스 키 비교
  - 레코드 평가
  - 메모리 임시 테이블 작업
  - 디스크 임시 테이블 작업
- 코스트 모델: 전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용
- MySQL 5.7 버전부터 MySQL 서버의 소스 코드에 상수화돼 있던 각 단위 작업의 비용을 DBMS 관리자가 조정할 수 있게 개선됨
- MySQL 8.0 버전으로 업그레이드되면서 비로소 칼럼의 데이터 분포를 위한 히스토그램과 각 인덱스별 메모리에 적재된 페이지의 비율이 관리되고 옵티마이저의 실행 계획 수립에 사용되기 시작함
- MySQL 8.0 서버의 코스트 모델은 다음 2개 테이블에 저장돼 있는 설정값을 사용
  - server_cost: 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
  - engine_cost: 레코드를 가진 데이터 페이지를 가져오는 데 필요한 비용 관리
- 코스트 모델에서 중요한 것은 각 단위 작업에 설정되는 비용 값이 커지면 어떤 실행 계획들이 고비용으로 바뀌고 어떤 실행 계획들이 저비용으로 바뀌는지를 파악하는 것

## 10.2 실행 계획 확인
- MySQL 서버의 실행 계획은 DESC 또는 EXPLAIN 명령으로 확인 가능

### 10.2.1 실행 계획 출력 포맷
```
EXPLAIN FORMAT=TREE
SELECT * FROM employees e
```
- MySQL 8.0 버전부터는 FORMAT 옵션을 사용해 실행 계획의 표시 방법을 JSON이나 TREE, 단순 테이블 형태로 선택 가능

### 10.2.2 쿼리의 실행 시간 확인
- MySQL 8.0.18 버전부터는 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있는 EXPLAIN ANALYZE 기능 추가
- EXPLAIN ANALYZE 명령은 항상 결과를 TREE 포맷으로 보여주기 때문에 FORMAT 옵션 사용 불가
- 실제 실행 순서는 다음 기준으로 읽음
  - 들여쓰기가 같은 레벨에서는 상단에 위치한 라인이 먼저 실행
  - 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행

## 10.3 실행 계획 분석
- 아무런 옵션 없이 EXPLAIN 명령을 실행하면 쿼리 문장의 특성에 따라 표 형태로 된 1줄 이상의 결과가 표시됨
- 표의 각 라인(레코드)은 쿼리 문장에서 사용된 테이블(서브쿼리로 임시 테이블을 생성한 경우 그 임시 테이블까지 포함)의 개수만큼 출력됨
- 실행 순서는 위에서 아래로 순서대로 표시(UNION이나 상관 서브쿼리와 같은 경우 순서대로 표시되지 않을 수도 있음)
- 출력된 실행 계획에서 위쪽에 출력된 결과일수록(id 칼럼의 값이 작을수록) 쿼리의 바깥 부분이거나 먼저 접근한 테이블이고, 아래쪽에 출력된 결과일수록(id 칼럼의 값이 클수록) 쿼리의 안쪽 부분 또는 나중에 접근한 테이블에 해당

### 10.3.1 id 칼럼
- 단위(SELECT) 쿼리: SELECT 키워드 단위로 구분한 것
- 실행 계획에서 가장 왼쪽에 표시되는 id 칼럼은 단위 SELECT 쿼리별로 부여되는 식별자 값
- 하나의 SELECT 문장 안에서 여러 개의 테이블을 조인하면 조인되는 테이블의 개수만큼 실행 계획 레코드가 출력되지만 같은 id 값이 부여됨
- 한 가지 주의해야 할 것은 실행 계획의 id 칼럼이 테이블의 접근 순서를 의미하지 않는다는 것
- TREE FORMAT으로 확인해보면 순서를 더 정확히 알 수 있음

### 10.3.2 select_type 칼럼
- 각 단위 SELECT 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼

#### 10.3.2.1 SIMPLE
- UNION이나 서브쿼리를 사용하지 않는 단순한 SELECT 쿼리인 경우 해당 쿼리 문장의 select_type은 SIMPLE로 표시됨
- 일반적으로 제일 바깥 SELECT 쿼리의 select_type이 SIMPLE로 표시됨

#### 10.3.2.2 PRIMARY
- UNION이나 서브쿼리를 가지는 SELECT 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리는 select_type이 PRIMARY로 표시됨

#### 10.3.2.3 UNION
- UNION으로 결합하는 단위 SELECT 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 SELECT 쿼리의 select_type은 UNION으로 표시됨
- UNION의 첫 번째 단위 SELECT는 select_type이 UNION이 아니라 UNION되는 쿼리 결과들을 모아서 저장하는 임시 테이블(DERIVED)이 select_type으로 표시됨

#### 10.3.2.4 DEPENDENT UNION
- DEPENDENT UNION 또한 UNION select_type과 같이 UNION이나 UNION ALL로 집합을 결합하는 쿼리에서 표시됨
- DEPENDENT는 UNION이나 UNION ALL로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는 것을 의미
- 내부 쿼리가 외부의 값을 참조해서 처리될 때 select_type에 DEPENDENT 키워드가 표시됨

#### 10.3.2.5 UNION RESULT
- UNION 결과를 담아두는 테이블
- MySQL 8.0 버전부터는 UNION ALL의 경우 임시 테이블을 사용하지 않도록 개선됨 -> UNION(또는 UNION DISTINCT)은 여전히 임시 테이블에 결과를 버퍼링함
- 실행 계획상에서 임시 테이블을 가리키는 라인의 select_type이 UNION RESULT
- UNION ALL을 사용하면 UNION RESULT 라인이 없어짐 -> 임시 테이블에 버퍼링하지 않기 때문에

#### 10.3.2.6 SUBQUERY
- select_type의 SUBQUERY는 FROM 절 이외에서 사용되는 서브쿼리만을 의미
- MySQL 서버의 실행 계획에서 FROM 절에 사용된 서브쿼리는 select_type이 DERIVED로 표시되고, 그 밖의 위치에서 사용된 서브쿼리는 전부 SUBQUERY라고 표시됨
- 파생테이블이라는 단어는 DERIVED와 같은 의미로 이해하면 됨

#### 10.3.2.7 DEPENDENT SUBQUERY
- 서브쿼리가 바깥쪽 SELECT 쿼리에서 정의된 칼럼을 사용하는 경우, select_type에 DEPENDENT SUBQUERY라고 표시됨
- DEPENDENT UNION과 같이 DEPENDENT SUBQUERY 또한 외부 쿼리가 먼저 수행된 후 내부 쿼리(서브쿼리)가 실행돼야 하므로 일반 서브쿼리보다 처리 속도가 느릴 때가 많음

#### 10.3.2.8 DERIVED
- MySQL 5.5 버전까지는 서브쿼리가 FROM 절에 사용된 경우 항상 select_type이 DERIVED인 실행 계획을 만듦
- 하지만 MySQL 5.6 버전부터는 옵티마이저 옵션에 따라 FROM 절의 서브쿼리를 외부 쿼리와 통합하는 형태의 최적화가 수행되기도 함
- DERIVED: 단위 SELECT 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블을 생성하는 것
- MySQL 5.6 버전부터는 옵티마이저 옵션에 따라 쿼리의 특성에 맞게 임시 테이블에도 인덱스를 추가해서 만들 수 있게 최적화됨
- 쿼리를 튜닝하기 위해 실행 계획을 확인할 때 가장 먼저 select_type 칼럼의 값이 DERIVED인 것이 있는지 확인해야 함
- 서브쿼리를 조인으로 해결할 수 있는 경우라면 서브쿼리보다 조인을 사용할 것을 강력히 권장

#### 10.3.2.9 DEPENDENT DERIVED
- MySQL 8.0 이전 버전에서는 FROM 절의 서브쿼리는 외부 칼럼을 사용할 수가 없었는데, MySQL 8.0 버전부터는 래터럴 조인(LATERAL JOIN) 기능이 추가되면서 FROM 절의 서브쿼리에서도 외부 칼럼 참조 가능
- LATERAL 키워드가 없는 서브쿼리에서 외부 칼럼을 참조하면 오류 발생

#### 10.3.2.10 UNCACHEABLE SUBQUERY
- 하나의 쿼리 문장에 서브쿼리가 하나만 있더라도 실제 그 서브쿼리가 한 번만 실행되는 것은 아님
- 그런데 조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 담아둠
- DEPENDENT SUBQUERY는 서브쿼리 결과가 캐시는 되지만, 딱 한 번만 캐시되는 것이 아니라 외부 쿼리의 값 단위로 캐시가 만들어지는 방식으로 처리됨
- select_type이 SUBQUERY인 경우와 UNCACHEABLE SUBQUERY는 이 캐시를 사용할 수 있느냐 없느냐의 차이가 있음
- 서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능할 수가 있는데, 그럴 경우 select_type이 UNCACHEABLE SUBQUERY로 표시됨
- 캐시를 사용하지 못하게 하는 요소
  - 사용자 변수가 서브쿼리에 사용된 경우
  - NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
  - UUID()나 RAND()와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우

#### 10.3.2.11 UNCACHEABLE UNION
- UNCACHEABLE과 UNION 키워드의 속성이 혼합된 select_type

#### 10.3.2.12 MATERIALIZED
- MySQL 5.6 버전부터 도입된 select_type으로, 주로 FROM 절이나 IN(subquery) 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용됨
```
EXPLAIN
SELECT * FROM employees e
WHERE e.emp_no IN (SELECT emp_no FROM salaries WHERE salary BETWEEN 100 AND 1000);
```
- MySQL 5.6 버전까지는 employees 테이블을 읽어서 employees 테이블의 레코드마다 salaries 테이블을 읽는 서브쿼리가 실행되는 형태로 처리
- 5.7 버전부터는 서브쿼리의 내용을 임시 테이블로 구체화(Materialization)한 후, 임시 테이블과 employees 테이블을 조인하는 형태로 최적화되어 처리됨
