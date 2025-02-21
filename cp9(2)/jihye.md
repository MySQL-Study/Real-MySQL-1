## 9.3 고급 최적화
- MySQL 서버의 옵티마이저가 실행 계획을 수립할 때 통계 정보와 옵티마이저 옵션을 결합해서 최적의 실행 계획 수립

### 9.3.1 옵티마이저 스위치 옵션
- optimizer_switch 시스템 변수를 이용해서 제어
- 옵티마이저 스위치 옵션은 글로벌과 세션별 모두 설정할 수 있는 시스템 변수
```js
-- // MySQL 서버 전체적으로 옵티마이저 스위치 설정
mysql> SET GLOBAL optimizer_switch='index_merge=on, index_merge_union=on,...';

-- // 현재 커넥션의 옵티마이저 스위치만 설정
mysql> SET SESSION optimizer_switch='index_merge=on, index_merge_union=on,...';

-- // 현재 쿼리에만 옵티마이저 스위치 설정
mysql> SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=off') */
...
FROM ...;
```

#### 9.3.1.1 MRR과 배치 키 액세스(mrr & batched_key_access)
- MySQL 서버의 내부 구조상 조인 처리는 MySQL 엔진이 처리하지만, 실제 레코드를 검색하고 읽는 부분은 스토리지 엔진이 담당
- 네스티드 루프 조인: 드라이빙 테이블의 일치하는 레코드를 한 건 읽어서 드리븐 테이블의 일치하는 레코드를 찾아서 조인을 수행
- MRR(Multi-Range-Read): 드라이빙 테이블의 레코드를 읽어서 드리븐 테이블과의 조인을 즉시 실행하지 않고 조인 대상을 버퍼링
  - 조인 버퍼에 레코드가 가득 차면 비로소 MySQL 엔진은 버퍼링된 레코드를 스토리지 엔진으로 한 번에 요청
  - 스토리지 엔진은 읽어야 할 레코드들을 데이터 페이지에 정렬된 순서로 접근해서 디스크의 데이터 페이지 읽기를 최소화
- BKA(Batched Key Access) 조인: MRR을 응용해서 실행되는 조인 방식
  - 기본적으로 비활성화
  - <details>
    <summary>BKA 조인을 사용하게 되면 부가적인 정렬 작업이 필요해지면서 오히려 성능에 안 좋은 영향을 미치는 경우도 있음</summary>
    
    BKA 조인은 일반적인 NLJ보다 성능이 좋은 경우가 많지만, 다음과 같은 경우에는 부가적인 정렬 작업이 추가되어 오히려 성능 저하가 발생할 수 있음:

    - 드리븐 테이블에서 읽어온 데이터가 정렬되지 않은 경우 → ORDER BY 시 추가 정렬 비용 발생
    - 드리븐 테이블에서 다중 키 검색으로 인해 랜덤 액세스 발생 → 디스크 I/O 증가
    - 드라이빙 테이블이 정렬된 상태인데 조인 과정에서 정렬이 깨지는 경우 → 정렬 순서 유지가 어려워 추가적인 정렬 작업 필요
    </details>

#### 9.3.1.2 블록 네스티드 루프 조인(block_nested_loop)
- MySQL 서버에서 사용되는 대부분의 조인은 네스티드 루프 조인
  - 조인의 연결 조건이 되는 칼럼에 모두 인덱스가 있는 경우 사용되는 조인 방식
- 네스티드 루프 조인과 블록 네스티드 루프 조인의 가장 큰 차이는 조인 버퍼가 사용되는지 여부와 조인에서 드라이빙 테이블과 드리븐 테이블이 어떤 순서로 조인되느냐임
- 조인 알고리즘에서 `Block`이라는 단어가 사용되면 조인용으로 별도의 버퍼가 사용됐다는 것을 의미
- 조인 쿼리의 실행 계획에서 Extra 칼럼에 `Using Join Buffer`라는 문구가 표시되면 조인 버퍼를 사용한다는 것을 의미
- 어떤 방식으로도 드리븐 테이블의 풀 테이블 스캔이나 인덱스 풀 스캔을 피할 수 없다면 옵티마이저는 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 이 메모리 캐시를 조인하는 형태로 처리
- 이 때 사용되는 메모리의 캐시가 조인 버퍼
  - join_buffer_size라는 시스템 변수로 크기를 제한할 수 있으며, 조인이 완료되면 조인 버퍼는 바로 해제됨
- 조인 버퍼가 사용되는 쿼리에서는 조인의 순서가 거꾸로인 것처럼 실행됨
- 일반적으로 조인이 수행된 후 가져오는 결과는 드라이빙 테이블의 순서에 의해 결정되지만 조인 버퍼가 사용되는 조인에서는 결과의 정렬 순서가 흐트러질 수 있음을 기억해야 함
- ※ MySQL 8.0.20 버전부터는 블록 네스티드 루프 조인은 더이상 사용되지 않고 해시 조인 알고리즘이 대체되어 사용됨

#### 9.3.1.3 인덱스 컨디션 푸시다운(index_condition_pushdown)
- MySQL 5.6 버전부터는 인덱스 컨디션 푸시다운이라는 기능 도입
- 인덱스를 비교하는 작업은 실제 InnoDB 스토리지 엔진이 수행하지만 테이블의 레코드에서 조건을 비교하는 작업은 MySQL 엔진이 수행하는 작업
- MySQL 5.5 버전까지는 인덱스를 범위 제한 조건으로 사용하지 못하는 조건은 MySQL 엔진이 스토리지 엔진으로 아예 전달해주지 않음
- MySQL 5.6 버전부터는 이렇게 인덱스를 범위 제한 조건으로 사용하지 못한다고 하더라도 인덱스에 포함된 칼럼의 조건이 있다면 모두 같이 모아서 스토리지 엔진으로 전달할 수 있게 핸들러 API 개선
- 인덱스 컨디션 푸시다운 기능은 고도의 기술력을 필요로 하는 기능은 아니지만 쿼리의 성능이 몇 배에서 몇십 배로 향상될 수도 있는 중요한 기능

#### 9.3.1.4 인덱스 확장(use_index_extension)
- use_index_extension 옵티마이저 옵션은 InnoDB 스토리지 엔진을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 프라이머리 키를 활용할 수 있게 할지를 결정하는 옵션
- 예를 들어, PRIMARY KEY (dept_no, emp_no), KEY ix_fromdate (from_date)로 인덱스를 생성하면 세컨더리 인덱스는 데이터 레코드를 찾아가기 위해 프라이머리 키에 명시된 순서대로 포함
- 최종적으로 ix_fromdate 인덱스는 (from_date, dept_no, emp_no) 조합으로 인덱스를 생성한 것과 흡사하게 작동 가능
- MySQL 서버가 업그레이드되면서 옵티마이저는 ix_fromdate 인덱스의 마지막에 (dept_no, emp_no) 칼럼이 숨어있다는 것을 인지하고 실행 계획을 수립하도록 개선됨

#### 9.3.1.5 인덱스 머지(index_merge)
- 인덱스 머지 실행 계획을 사용하면 하나의 테이블에 대해 2개 이상의 인덱스를 이용해 쿼리를 처리
- 하나의 인덱스만 사용해서 작업 범위를 충분히 줄일 수 있ㄴ느 경우라면 테이블별로 하나의 인덱스만 활용하는 것이 효율적
- 하지만 쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때 MySQL 서버는 인덱스 머지 실행 계획 선택

#### 9.3.1.6 인덱스 머지 - 교집합(index_merge_intersection)
```
SELECT * FROM employees WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```
- 실행 계획의 Extra 칼럼에 `Using intersect(ix_firstname, PRIMARY)`라고 표시된 것은 이 쿼리가 여러 개의 인덱스를 각각 검색해서 그 결과의 교집합만 반환했다는 것을 의미
- 그런데 ix_firstname 인덱스는 프라이머리 키인 emp_no 칼럼을 자동으로 포함하고 있기 때문에 그냥 ix_firstname 인덱스만 사용하는 것이 더 성능이 좋을 것으로 생각 가능
- 그렇다면 index_merge_intersection 최적화를 비활성화하면 됨
  - MySQL 서버 전체적으로 최적화를 제어하는 것이 불안하다면 현재 커넥션 또는 현재 쿼리에 대해서만 비활성화 가능

#### 9.3.1.7 인덱스 머지 - 합집합(index_merge_union)
```
SELECT * FROM employees WHERE first_name='Matt' OR hire_date='1987-03-31';
```
- 인덱스 머지의 `Using union`은 WHERE 절에 사용된 2개 이상의 조건이 각각의 인덱스를 사용하되 OR 연산자로 연결된 경우에 사용되는 최적화
- 위의 쿼리에서 first_name='Matt'이면서 hire_date='1987-03-31'인 사원이 있었다면 그 사원의 정보는 두 개의 인덱스 모두에 포함돼 있음
- 이 때 중복 제거하기 위해 정렬 작업이 필요했을텐데 실행 계획에는 정렬 표시가 없음
- MySQL 서버는 이 같은 중복 제거를 위해 내부적으로 어떤 작업을 수행??
  - 인덱스 검색을 통한 두 결과 집합 모두 프라이머리 키로 정렬돼 있음
  - 그래서 MySQL 서버는 두 집합에서 하나씩 가져와서 서로 비교하면서 프라이머리 키인 emp_no 칼럼의 값이 중복된 레코드들을 정렬 없이 걸러냄
  - 이렇게 정렬된 두 집합의 결과를 하나씩 가져와 중복 제거를 수행할 때 사용된 알고리즘을 우선순위 큐라고 함
- ※ 2개의 WHERE 조건이 OR 연산자로 연결된 경우에는 둘 중 하나라도 제대로 인덱스를 사용하지 못하면 항상 풀 테이블 스캔으로밖에 처리하지 못함

#### 9.3.1.8 인덱스 머지 - 정렬 후 합집합(index_merge_sort_union)
```
SELECT * FROM employees
WHERE first_name='Matt'
OR hire_date BETWEEN '1987-03-01' AND '1987-03-31';
```
- 만약 인덱스 머지 작업을 하는 도중에 결과의 정렬이 필요한 경우 MySQL 서버는 인덱스 머지 최적화의 `Sort union` 알고리즘을 사용
- 두 번째 쿼리의 결과는 emp_no 칼럼으로 정렬돼 있지 않음 -> 중복 제거를 위해 우선순위 큐 사용 불가능
- 그래서 MySQL 서버는 두 집합의 결과에서 중복을 제거하기 위해 각 집합을 emp_no 칼럼으로 정렬한 다음 중복 제거 수행
- 인덱스 머지 최적화에서 중복 제거를 위해 강제로 정렬을 수행해야 하는 경우에는 실행 계획의 Extra 칼럼에 `Using sort_union` 문구 표시

#### 9.3.1.9 세미 조인(semijoin)
```
SELECT * FROM employees e
WHERE e.emp_no IN (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```
- 세미 조인: 다른 테이블과 실제 조인을 수행하지는 않고, 단지 다른 테이블에서 조건에 일치하는 레코드가 있는지 없는지만 체크하는 형태의 쿼리
- MySQL 5.7 서버는 전톧적으로 세미 조인 형태의 쿼리를 최적화하는 부분이 상당히 취약
- MySQL 서버에서 세미 조인 최적화가 도입되기 전에는 employees 테이블을 풀 스캔하면서 한 건 한 건 서브쿼리의 조건에 일치하는지 비교
- `= (subquery)` 형태와 `IN (subquery)` 형태의 세미 조인 쿼리에 대한 최적화 방법
  - 세미 조인 최적화
  - IN-to-EXISTS 최적화
  - MATERIALIZATION 최적화
- `<> (subquery)` 형태와 `NOT IN (subquery)` 형태의 안티 세미 조인 쿼리에 대한 최적화 방법
  - IN-to-EXISTS 최적화
  - MATERIALIZATION 최적화
- MySQL 서버 8.0 버전부터 세미 조인 쿼리의 성능을 개선하기 위한 최적화 전략
  - Table Pull-out
  - Duplicate Weed-out
  - First Match
  - Loose Scan
  - Materialization
- Table Pull-out 최적화 전략은 사용 가능하면 항상 세미 조인보다는 좋은 성능을 내기 때문에 별도로 제어하는 옵티마이저 옵션 제공 X
- optimizer_switch 시스템 변수의 semijoin 옵션은 firstmatch, loosescan, materialization 옵티마이저 옵션을 한 번에 활성화하거나 비활성화할 때 사용

#### 9.3.1.10 테이블 풀-아웃(Table Pull-out)
- 세미 조인의 서브쿼리에 사용된 테이블을 아우터 쿼리로 끄집어낸 후에 쿼리를 조인 쿼리로 재작성하는 형태의 최적화
- Table Pull-out 최적화가 사용됐는지 확인 방법
  - 실행 계획에서 해당 테이블들의 id 칼럼 값이 같은지 다른지 비교(그러면서 Extra 칼럼에 아무것도 출력되지 않는 경우) -> 동일한 값을 가지면 조인으로 처리됐음을 의미
  - EXPLAIN 명령을 실행한 직후 SHOW WARNINGS 명령으로 MySQL 옵티마이저가 재작성한 쿼리 살피기 -> JOIN으로 재작성된 쿼리를 확인
- Table Pull-out 최적화의 제한 사항과 특성
  - Table Pull-out 최적화는 세미 조인 서브쿼리에서만 사용 가능
  - Table Pull-out 최적화는 서브쿼리 부분이 UNIQUE 인덱스나 프라이머리 키 룩업으로 결과가 1건인 경우에만 사용 가능
  - Table Pull-out이 적용된다고 하더라도 기존 쿼리에서 가능했던 최적화 방법이 사용 불가능한 것은 아니므로 MySQL에서는 가능하다면 Table Pull-out 최적화 최대한 적용
  - Table Pull-out 최적화는 서브쿼리의 테이블을 아우터 쿼리로 가져와서 조인으로 풀어쓰는 최적화를 수행하는데, 만약 서브쿼리의 모든 테이블이 아우터 쿼리로 끄집어 낼 수 있다면 서브쿼리 자체는 없어짐
  - MySQL에서는 `최대한 서브쿼리를 조인으로 풀어서 사용해라`라는 튜닝 가이드가 많은데, Table Pull-out 최적화는 사실 이 가이드를 그대로 실행하는 것 -> 이제부터 서브쿼리를 조인으로 풀어서 사용할 필요 없음

#### 9.3.1.11 퍼스트 매치(firstmatch)
```
EXPLAIN SELECT * FROM employees e
WHERE e.first_name='Matt'
AND e.emp_no IN (SELECT t.emp_no FROM titles t WHERE t.from_date BETWEEN '1995-01-01' AND '1995-01-30');
```
- First Match 최적화 전략은 IN(subquery) 형태의 세미 조인을 EXISTS(subquery) 형태로 튜닝한 것과 비슷한 방법으로 실행됨
- 실행 계획에서 id 칼럼의 값이 모두 같고 Extra 칼럼에 `FirstMatch(e)` 문구 출력
- `FirstMatch(e)` 문구는 employees 테이블의 레코드에 대해 titles 테이블에 일치하는 레코드 1건만 찾으면 더이상의 titles 테이블 검색을 하지 않는다는 것 의미
- FirstMatch는 서브쿼리가 아니라 조인으로 풀어서 실행하면서 일치하는 첫번째 레코드만 검색하는 최적화 실행
- FirstMatch 최적화는 MySQL 5.5에서 수행했던 최적화 방법인 IN-to-EXISTS 변환과 거의 비슷한 처리 로직 수행
- FirstMatch의 장점
  - 가끔은 여러 테이블이 조인되는 경우 원래 쿼리에는 없던 동등 조건을 옵티마이저가 자동으로 추가하는 형태의 최적화가 실행되기도 함. 기존의 IN-to-EXISTS 최적화에서는 이러한 동등 조건 전파가 서브쿼리 내에서만 가능했지만 FirstMatch에서는 조인 형태로 처리되기 때문에 서브쿼리뿐만 아니라 아우터 쿼리의 테이블까지 전파 가능. 최종적으로는 FirstMatch 최적화로 실행되면 더 많은 조건이 주어지는 것이므로 더 나은 실행 계획 수립 가능
  - IN-to-EXISTS 변환 최적화 전략에서는 아무런 조건 없이 변환이 가능한 경우에는 무조건 그 최적화 수행. 하지만 FirstMatch 최적화에서는 서브쿼리의 모든 테이블에 대해 FirstMatch 최적화를 수행할지 아니면 일부 테이블에 대해서만 수행할지 취사선택 가능
- FirstMatch 최적화의 제한 사항과 특성
  - FirstMatch는 서브쿼리에서 하나의 레코드만 검색되면 더이상의 검색을 멈추는 단축 실행 경로이기 떄문에 FirstMatch 최적화에서 서브쿼리는 그 서브쿼리가 참조하는 모든 아무터 테이블이 먼저 조회된 이후에 실행됨
  - FirstMatch 최적화가 사용되면 실행 계획의 Extra 칼럼에는 `FirstMatch(table-N)` 문구가 표시됨
  - FirstMatch 최적화는 상관 서브쿼리에서도 사용 가능
  - FirstMatch 최적화는 GROUP BY나 집합 함수가 사용된 서브쿼리의 최적화에는 사용 불가

#### 9.3.1.12 루스 스캔(loosescan)
- 세미 조인 서브쿼리 최적화의 LooseScan은 인덱스를 사용하는 GROUP BY 최적화 방법에서 살펴본 `Using index for group-by`의 루스 인덱스 스캔과 비슷한 읽기 방식을 사용

#### 9.3.1.13 구체화(Materialization)

#### 9.3.1.14 중복 제거(Duplicated Weed-out)

#### 9.3.1.15 컨디션 팬아웃(condition_fanout_filter)

#### 9.3.1.16 파생 테이블 머지(derived_merge)

#### 9.3.1.17 인비저블 인덱스(use_invisible_indexes)

#### 9.3.1.18 스킵 스캔(skip_scan)

#### 9.3.1.19 해시 조인(hash_join)

#### 9.3.1.20 인덱스 정렬 선호(prefer_ordering_index)
