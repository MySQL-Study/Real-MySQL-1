## 9.3 고급 최적화
- MySQL 서버의 옵티마이저가 실행 계획을 수립할 때 통계 정보와 옵티마이저 옵션을 결합해서 최적의 실행 계획 세움.

- 옵티마이저 옵션
    - 조인 관련된 옵티마이저 옵션
        - MySQL 서버 초기 버전부터 제공되던 옵션
        - 조인이 많이 사용되는 서비스에서는 알아야 함.
    - 옵티마이저 스위치
        - MySQL 5.5 버전부터 지원됨.
        - MySQL 서버의 고급 최적화 기능을 활성화할지를 제어하는 용도로 사용

### 9.3.1 옵티마이저 스위치 옵션
- `optimizer_switch` 시스템 변수를 이용해서 제어
    - 여러개의 옵션을 세트로 묶어서 설정하는 방식으로 사용함.
    - 최적화 옵션
        - 각각의 옵티마이저 스위치 옵션은 `default`, `on`, `off` 중에서 하나 설정 가능
        - 스위치 옵션은 글로벌과 세션별 모두 설정할수 있는 시스템 변수이므로 MySQL 서버 전체적으로 또는 현재 커넥션에 대해서만 설정 가능
      ```
      // MySQL 서버 전체적으로 옵티마이저 스위치 설정
      mysql> SET GLOBAL optimizer_switch='index_merge=on, 		index_merge_union=on,...';
      
      // 현재 커넥션의 옵티마이저 스위치만 설정
      mysql> SET SESSION optimizer_switch='index_merge=on, 		index_merge_union=on,...';
      ```
        - "SET_VAR" 옵티마이저 힌트를 이용해 현재 쿼리에만 설정도 가능
      ```
      mysql> SELECT /*+ SET_VAR(optimizer_switch='condition_fanout_filter=pff') */ 
      ... FROM ...
      ```

#### 9.3.1.1 MRR과 배치 키 액세스(mrr & batched_key_access)
- MRR : Multi-Range Read
- 매뉴얼에서는 DS-MRR (Disk Sweep Multi-Range Read)라고도 함.
- MySQL 서버의 내부 구조상 조인 처리는 MySQL 엔진이 처리하지만, 실제 레코드를 검색하고 읽는 부분은 스토리지 엔진이 담당
    - 이 때 드라이빙 테이블의 레코드 건별로 드리븐 테이블의 레코드를 찾으면 레코드를 찾고 읽는 스토리지 엔진에서는 아무런 최적화 수행 불가
    - 단점을 보완하기 위해 MySQL 서버는 조인 대상 테이블 중 하나로부터 레코드를 읽어서 조인 버퍼에 버퍼링 함.
      -> 즉, 드라이빙 테이블의 레코드를 읽어서 드리븐 테이블과의 조인을 즉시 실행하지 않고 조인 대상을 버퍼링하는 것.
    - 조인 버퍼에 레코드가 가득 차면 MySQL 엔진은 버퍼링된 레코드를 스토리지 엔진으로 한 번에 요청
    - 스토리지 엔진은 읽어야 할 데이터 페이지에 정렬된 순서로 접근해서 디스크의 데이터 페이지 읽기를 최소화 할 수 있는 것 (InnoDB 버퍼 풀에 있다고 하더라도 버퍼 풀의 접근을 최소화 할 수 있음.)
      <br>
- `BKA(Batched Key Access) 조인` : MRR을 응용해서 실행되는 조인 방식
    - BKA 조인 최적화는 기본적으로 비활성화 되어 있음.
    - BKA 조인을 사용하게 되면 부가적인 정렬 작업이 필요해지면서 오히려 성능에 안 좋은 영향을 미치는 경우도 있음. -> 단점

#### 9.3.1.2 블록 네스티드 루프 조인(block_nested_loop)
- MySQL 서버에서 사용되는 대부분의 조인은 네스티드 루프 조인(Nested Loop Join)인데, 조인의 연결 조건이 되는 칼럼에 모두 인덱스가 있는 경우 사용되는 조인 방식
    - 레코드를 읽어서 다른 버퍼 공간에 저장하지 않고 즉시 드리븐 테이블의 레코드를 찾아서 반환한다는 것
      <br>

- `네스티드 루프 조인` VS `블록 네스티드 루프 조인(Block Nested Loop Join`
    - 조인 버퍼(join_buffer_size 시스템 설정으로 조정되는 조인을 위한 퍼퍼)가 사용되는지 여부와 조인에서 드라이빙 테이블과 드리븐 테이블이 어떤 순서로 조인되느냐의 차이점.
    - 조인 알고리즘에서 "Block"이라는 단어가 사용되면 조인용으로 별도의 버퍼가 사용됐다는 것을 의미함.
    - 조인 쿼리의 실행 계획에서 Extra 칼럼에서 "Using Join buffer"라는 문구가 표시되면 그 실행 계획은 조인 버퍼를 사용한다는 것을 의미함.
      <br>
- `조인 버퍼(Join buffer)`
  : 어떤 방식으로도 드리븐 테이블의 풀 테이블 스캔이나 인덱스 풀 스캔을 피할 수 없다면 옵티마이저는 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 메모리 캐시를 조인하는 형태로 처리하는데, 이 때 사용되는 메모리의 캐시
  - `join_buffer_size`라는 시스템 변수로 크기를 제한 가능
  - 조인이 완료되면 조인 버퍼는 바로 해제됨.
  - 일반적으로 조인이 수행된 후 가져오는 결과는 드라이빙 테이블의 순서에 의해 결정되지만, 조인 버퍼가 사용되는 조인에서는 결과의 정렬 순서가 흐트러질 수 있음.

#### 9.3.1.3 인덱스 컨디션 푸시다운(index_condition_pushdown)
- MySQL 5.6 버전부터는 인덱스 컨디션 푸시다운(index_condition_pushdown)이라는 기능이 도입됨.
- 인덱스 컨디션 푸시다운 기능은 고도의 기술력을 요하는 기능은 아니지만, 쿼리의 성능이 몇 배에서 몇십 배로 향상될 수 있는 중요한 기능.

#### 9.3.1.4 인덱스 확장(use_index_extensions)
- `use_index_extensions 옵티마이저 옵션`
  : InnoDB 스토리지 엔진을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 프라이머리 키를 활용하게 할지를 결정하는 옵션
  - 옵티마이저가 `ix_fromdate` 인덱스의 마지막에 칼럼을 인지하고, 쿼리가 세컨더리 인덱스의 마지막에 자동 추가되는 프라이머리 키를 제대로 활용할 수 있게 실행 계획을 수립하도록 개선
  - 정렬 작업도 인덱스를 활용해서 처리되는 장점도 있음.

#### 9.3.1.5 인덱스 머지(index_merge)
- 인덱스를 이용해 쿼리를 실해앟는 경우, 대부분 옵티마이저는 테이블별로 하나의 인덱스만 사용하도록 계획을 수립
- 인덱스 머지 실행 계획을 사용하면 하나의 테이블에 대해 2개 이상의 인덱스를 이용해 쿼리를 처리
    - 하나의 인덱스만 사용해서 작업 범위를 충분히 줄일 수 있는 경우라면 테이블별로 하나의 인덱스만 활용하는 것이 효율적
    - 쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만적하는 레코드 건수가 많은 것으로 예상될 때 인덱스 머지 실행 계획을 선택함.
      <br>
      -세부 실행 계획
    - `index_merge_intersection`
    - `index_merge_sort_union`
    - `index_merge_union`

#### 9.3.1.6 인덱스 머지-교집합(index_merge_intersection)
- 실행 계획의 Extra 칼럼에 "Using intersection"라고 표시됨
  -> 여러 개의 인덱스를 각각 검색해서 그 결과의 교집합만 반환했다는 의미

#### 9.3.1.7 인덱스 머지-합집합(index_merge_union)
- 인덱스 머지의 "Using union'을 WHERE 절에 사용된 2개 이상의 조건이 각각의 인덱스를 사용하되 OR 연산자로 연결된 경우에 사용하는 최적화
- 쿼리 실행 계획에서 Extra 칼럼에 "Using union"dms 'Union' 알고리즘으로 병합했다는 의미
- 두 집합의 합집합을 가져왔다는 것을 의미함.
- 중복을 제거하기 위한 별도의 정렬 작업을 진행하지 않은 이유
  : 쿼리를 실행하면, 두 결과 집합이 모두 프라이머리 키로 정렬돼 있고, 두 집합에서 하나씩 가져와서 서로 비교하면서 프라이머리 키가 값이 중복된 레코드들을 정렬 없이 걸러낼 수 있음.
  -> 우선순위 큐(Priority Queue) 알고리즘

#### 9.3.1.8 인덱스 머지-정렬 후 합집합(index_merge_sort_union)
- 만약 인덱스 머지 작업을 하는 도중에 결과의 정렬이 필요한 경우 MySQL 서버는 인덱스 머지 최적화의 'Sort union' 알고리즘 사용
- 실행 계획의 Extra 칼럼에 "Using sort_union" 문구가 표시됨.

#### 9.3.1.9 세미 조인(semijoin)
- 세미 조인(semi-Join)
  : 다른 테이블과 실제 조인을 수행하지는 않고, 단지 다른 테이블에서 조건에 일치하는 레코드가 있는지 없는지만 체크하는 형태의 쿼리
  - 세미 조인 쿼리에 대한 최적화 방법
  - 세미 조인 최적화
  - IN-to-EXISTS 최적화
  - MATERIALIZATION 최적화
  - 안티 세미 조인 쿼리 최적화 방법
  - IN-to-EXISTS 최적화
  - MATERIALIZATION 최적화

- 세미 조인 최적화
    - Table Pull-out
    - Duplicate Weed-out
    - First Match
    - Loose Scan
    - Materialization

- Table pull-out 최적화 전략은 사용 가능하면 항상 세미 조인보다는 좋은 성능을 내기 때문에 별도로 제어하는 옵티마이저 옵션을 제공하지 않음.
- First Match와 Loose Scan 최적화 전략은 각각 firstmatch와 loosescan 옵티마이저 옵션으로 사용 여부 결정 가능
- Duplicate Weed-out과 Materialization 최적화 전략은 materialization 옵티마이저 스위치로 사용 여부를 선택 가능
- optimizer_switch 시스템 변수의 semijoin 옵이마이저 옵션은 firstmatch와 loosescan, materialization 옵티마이저 옵션을 한 번에 활성화하거나 비활성화할 때 사용

#### 9.3.1.10 테이블 풀-아웃(Table Pull-out)
- 세미 조인의 서브쿼리에 사용된 테이블을 아우터 쿼리로 끄집어낸 후에 쿼리를 조인 쿼리로 재작성하는 형태의 최적화
- 서브쿼리 최적화가 도입되기 이전에 수동으로 쿼리를 튜닝하던 대표적인 방법이었음.
- 별도로 실행 계획에 Extra 칼럼에 "Using table pullout"과 같은 문구가 출력되지 않음.
- Table pullout 최적화가 사용됐는지 더 정확하게 확인하는 방법은 EXPLAIN 명령을 실행한 직후 다음과 같이 SHOW WARNINGS 명령으로 MySQL 옵티마이저가 재작성(Re-Write)한 쿼리를 살펴보는 것
  <br>
- Table pullout 최적화 제한 사항과 특성
    - Table pullout 최적화는 세미 조인 서브쿼리에서만 사용 가능
    - Table pullout 최적화는 서브 쿼리 부분이 UNIQUE 인덱스나 프라이머리 키 룩업으로 결과가 1건인 경우에만 사용 가능
    - Table pullout이 적용된다고 하더라도 기존 쿼리에서 가능했던 최적화 방법이 사용 불가능한 것은 아니므로 MySQL에서는 가능하다면 Table pullout 최적화를 최대한 적용
    - Table pullout 최적화는 서브 쿼리의 테이블을 아우터 쿼리로 가져와서 조인으로 풀어쓰는 최적화를 수행하는데, 만약 서브쿼리의 모든 테이블이 아우터 쿼리로 끄집어 낼 수 있다면 서브쿼리 자체는 없어짐.
    - MySQL에서는 "최대한 서브쿼리를 조인으로 풀어서 사용해라"라는 튜닝 가이드가 많은데, Table pullout 최적화는 사실 이 가이드를 그대로 실행하는 것. 이제부터는 서브쿼리를 조인으로 풀어서 사용할 필요가 없음.

#### 9.3.1.11 퍼스트 매치(firstmatch)
- First Match 최적화 전략은 IN(subquery) 형태의 세미 조인을 EXISTS(subquery) 형태로 튜닝한 것과 비슷한 방법으로 실행됨.
- FirstMatch는 서브쿼리가 아니라 조인으로 풀어서 실행하면서 일치하는 첫번째 레코드만 검색하는 최적화를 실행한 것
- FirstMatch 최적화는 MySQL 5.5에서 수행했던 최적화 방법인 In-to-EXISTS 변환과 거의 비슷한 로직 수행
- MySQL 5.5의 In-to-EXISTS 변환에 비해 장점
    - 기존의 In-to-EXISTS 최적화에서는 이러한 동등 조건 전파(Equality propagation) 서브 쿼리 내에서만 가능했지만 FirstMatch에서는 조인 형태로 처리되기 때문에 서브쿼리뿐만 아니라 아우터 쿼리 테이블까지 전파될 수 있음. 최종적으로는 FirstMatch 최적화로 실행되면 더 많은 조건이 주어지는 것이므로 더 나은 실행계획 수립 가능
    - In-to-EXISTS 변환 최적화 전략에서는 아무런 조건 없이 변환이 가능한 경우에는 무조건 그 최적화를 수행했지만, FirstMatch 최적화에서는 서브쿼리의 모든 테이블에 대해 FirstMatch 최적화를 수행할지 아니면 일부 테이블에 대해서만 수행할지 취사선택할 수 있다는 것이 장점

- FirstMatch 최적화의 제한 사항과 특성
    - FirstMatch는 서브쿼리에서 하나의 레코드만 검색되면 더이상의 검색을 멈추는 단축 실행 경로(Short-cut-path)이기 때문에 FirstMatch 최적화에서 서브쿼리는 그 서브쿼리가 참조하는 모든 아우터 테이블이 먼저 조회된 이후에 실행됨.
    - FirstMatch 최적화가 사용되면 실행 계획의 Extra 칼럼에는 "FirstMatch(table-N)" 문구가 표시됨.
    - FirstMatch 최적화는 상관 서브쿼리(Correlated subquert)에서도 사용 가능
    - FirstMatch 최적화는 GROUP BY나 집합 함수가 사용된 서브쿼리의 최적화에는 사용될 수 없음.

#### 9.3.1.12 루스 스캔(loosescan)
- 세미 조인 서브쿼리 최적화의 LooseScan은 인덱스를 사용하는 GROUP BY 최적화 방법에서 살펴본 "Using index for group-by"의 루스 인덱스 스캔(Loose Index Scan)과 비슷한 읽기 방식을 사용
- LooseScan 최적화의 특성
  : 서브쿼리 부분이 루스 인덱스 스캔을 사용할 수 있는 조건이 갖춰져야 사용할 수 있는 최적화
  - 옵티마이저가 LooseScan 최적화를 사용하지 못하게 비활성화하려면 `optimizer_switch` 시스템 변수에서 `loosescan` 최적화 옵션을 off로 설정하면 됨.
  ```
  mysql> SET optimizer_switch='loosescan=off'
  ```

#### 9.3.1.13 구체화(Materialization)
- Materialization 최적화는 세미 조인에 사용된 서브쿼리를 통째로 구체화해서 쿼리를 최적화한다는 의미
- 구체화(Materialization)는 쉽게 표현하면 내부 임시 테이블을 생성한다는 것을 의미함.
- Materialization 최적화는 다른 서브쿼리 최적화와는 달리, 다음 쿼리와 같이 서브쿼리 내에 GROUP BY절이 있어도 이 최적화 전략을 사용 가능

- Materialization 최적화가 사용될 수 있는 형태의 쿼리의 제한 사항과 특성
    - IN(subquery)에서는 서브쿼리는 상관 서브쿼리(Correlated subquery)가 아니어야 함.
    - 서브쿼리는 GROUP BY나 집합 함수들이 사용돼도 구체화를 사용할 수 있음.
    - 구체화가 사용된 경우에는 내부 임시 테이블이 사용됨.

- Materialization 최적화는 `optimizer_switch` 시스템 변수에서 `semijoin` 옵션과 `Materialization` 옵션이 모두 ON으로 활성화된 경우에만 사용됨.
    - MySQL 8.0버전에서는 기본적으로 이 두 옵션은 ON으로 활성화돼 있음.
    - Materialization 최적화만 비할성화하고자 한다면 semijoin 옵티마이저 옵션은 ON으로 활성화하되, materialization 옵티마이저 옵션만 OFF로 비활성화하면 됨.

#### 9.3.1.14 중복 제거(Duplicated Weed-out)
- Duplicated Weed-out은 세미 조인 서브쿼리를 일반적인 INNER JOIN 쿼리로 바꿔서 실행하고 마지막에 중복된 레코드를 제거하는 방법으로 처리되는 최적화 알고리즘

- Duplicated Weed-out 최적화 알고리즘은 원본 쿼리를 위와 같이 INNER JOIN + GROUP BY 절로 바꿔서 실행하는 것과 동일한 작업으로 쿼리를 처리

- Duplicated Weed-out 최적화의 장점과 제약 사항
    - 서브쿼리가 상관 서브쿼리라고 하더라도 사용할 수 있는 최적화임.
    - 서브쿼리가 GROUP BY나 집합 함수가 사용된 경우에는 사용될 수 없음.
    - Duplicated Weed-out은 서브쿼리의 테이블을 조인으로 처리하기 때문에 최적화할 수 있는 방법이 많음.

#### 9.3.1.15 컨디션 팬아웃(condition_fanout_filter)
- 조인을 실행할 때 테이블 순서는 쿼리 성능에 매우 큰 영향 미침.
- MySQL 옵티마이저는 여러 테이블이 조인되는 경우 가능하다면 일치하는 레코드 건수가 작은 순서대로 조인을 실행
- MySQL 8.0 버전에서 `condition_fanout_filter` 최적화가 활성화되면 다음과 같은 조건을 만족하는 컬럼의 조건들에 대해 조건을 만족하는 레코드 비율 계산 가능
    - WHERE 조건절에 사용된 컬럼에 대해 인덱스가 있는 경우
    - WHERE 조건절에 사용된 칼럼에 대해 히스토그램이 존재하는 경우

- `condition_fanout_filter` 최적화 기능을 활성화하면 MySQL 옵티마이저는 더 정교한 계산을 거쳐서 실행 계획을 수립
  -> 쿼리의 실행 계획 수립에 더 많은 시간과 컴퓨팅 자원을 사용하게 됨.
    - 쿼리가 간단하고 MySQL 8.0 이전 버전에서도 쿼리 실행 계획이 잘못된 선택을 한 적이 별로 없다면,`condition_fanout_filter` 최적화는 성능 향상에 크게 도움이 되지 않을 수 있음.
    - MySQL 서버가 처리하는 쿼리의 빈도가 매우 높다면 실행 계획 수립에 추가되는 오버헤드가 더 크게 보일 수 있으므로 가능하면 업그레이드를 실행하기 전에 성능 테스트를 진행하는 것이 좋음.

#### 9.3.1.16 파생 테이블 머지(derived_merge)
- 예전 버전의 MySQL 서버에서는 다음과 같이 FROM 절에 사용된 서브쿼리는 먼저 실행해서 그 결과를 임시 테이블로 만든 다음 외부 쿼리 부분을 처리함.
- `파생 테이블(Derived Table)`
  : FROM 절에 사용된 서브쿼리
  - 임시 테이블이 메모리에 상주할 만큼 크기가 작다면 성능에 큰 영향을 미치지 않겠지만, 레코드가 많아진다면 임시 테이블로 레코드를 복사하고 읽는 오버헤드로 인한 쿼리의 성능은 많이 느려짐.
  <br>
  - 옵티마이저가 자동으로 서브쿼리를 외부 쿼리로 병합할 수 있는 경우
  - SUM() 또는 MIN(), MAX() 같은 집계 함수와 윈도우 함수(Window Function)가 사용된 서브 쿼리
  - DISTNICT가 사용된 서브쿼리
  - GROUP BY나 HAVING이 사용된 서브쿼리
  - LIMIT가 사용된 서브쿼리
  - UNION 또는 UNION ALL을 포함하는 서브쿼리
  - SELECT 절에 사용된 서브쿼리
  - 값이 변경되는 사용자 변수가 사용된 서브쿼리

#### 9.3.1.17 인비저블 인덱스(use_invisible_indexes)
- MySQL 8.0 버전부터는 인덱스 가용 상태를 제어할 수 있는 기능이 추가됨.
    - `MySQL 8.0 이전 버전`
      : 인덱스가 존재하면 항상 옵티마이저가 실행 계획을 수립할 때 해당 인덱스를 검토하고 사용
      - `MySQL 8.0 버전 부터`
      : 인덱스를 삭제하지 않고, 해당 인덱스를 사용하지 못하게 제어하는 기능을 제공

- `use_invisible_indexes` 옵티마이저 옵션을 이용하면 INVISIBLE로 설정된 인덱스라 하더라도 옵티마이저가 사용하게 제어 가능
    - `use_invisible_indexes` 옵션의 기본값은 off
      -> INVISIBLE 상태의 인덱스는 옵티마이저가 볼 수 없는 상태
    - 옵티마이저 옵션을 변경하면 옵티마이저가 INVISIBLE 상태의 인덱스도 볼 수 있게 설정 가능

#### 9.3.1.18 스킵 스캔(skip_scan)
- 인덱스의 핵심은 값이 정렬돼 있다는 것이며, 이로 인해 인덱스를 구성하는 칼럼의 순서가 매우 중요함.
- 인덱스 스킵 스캔은 제한적이긴 하지만 인덱스의 이런 제약 사항을 뛰어넘을 수 있는 최적화 기법
- MySQL 8.0 버전부터는 인덱스 스킵 스캔 최적화가 도입됐으면, 이 기능은 인덱스의 선행 칼럼이 조건절에 사용되지 않더라도 후행 칼럼의 조건만으로 인덱스를 이용한 쿼리 성능 개선이 가능

#### 9.3.1.19 해시 조인(hash_join)
- 네스티드 루프 조인보다 해시 조인이 항상 빠른 것은 아님.
    - 해시 조인은 첫 번째 레코드를 찾는데 시간이 많이 걸리지만, 최종 레코드를 찾는 데까지 시간이 많이 걸리지 않음을 알 수 있음.
    - 네스티드 루프 조인은 마지막 레코드를 찾는 데까지는 시간이 많이 걸리지만 첫 번째 레코드를 찾는 것은 상대적으로 훨씬 빠름.

- 해시 조인 쿼리는 최고 스루풋(Best Throughput) 전략에 적합하며, 네스티드 루프 조인은 최고 응답 속도(Best-Response-time) 전략에 적합함.
- 일반적인 웹 서비스는 온라인 트랜잭션(OLTP) 서비스이기 때문에 스루풋도 중요하지만 응답 속도가 더 중요함.
  -> 분석과 같은 서비스는 사용자의 응답 시간보다는 전체적으로 처리 소요 시간이 중요하기 때문에 응답 속도보다는 전체 스루풋이 중요함.

<br>

- 해시 조인의 단계
    - 빌드 단계(Build-phase)
      : 조인 대상 테이블 중에서 레코드 건수가 적어서 해시 테이블로 만들기에 용이한 테이블을 골라서 메모리에 해시 테이블을 생성(빌드)하는 작업을 수행
      - `빌드 테이블` : 빌드 단계에서 해시 테이블을 만들 때 사용되는 원본 테이블
      - 프로브 단계(Pgrobe-phase)
      : 나머지 테이블의 레코드를 읽어서 해시 테이블의 일치 레코드를 찾는 과정을 의미
      - `프로브 테이블` : 이 때 읽는 나머지 테이블
      - 해시 테이블을 메모리에 저장할 때 MySQL 서버는 `join_buffer_size` 시스템 변수로 크기를 제어할 수 있는 조인 버퍼를 사용
      - 해시 테이블의 레코드 건수가 많아서 조인 버퍼의 공간이 부족한 경우
      : MySQL 서버는 빌드 테이블과 프로브 테이블을 적당한 크기(하나의 청크가 조인 버퍼보다 작도록)의 청크로 분리
      -> 청크별로 해시조인(메모리에서 모두 처리 가능한 경우)와 동일 방식으로 해시 조인을 처리함.

#### 9.3.1.20 인덱스 정렬 선호(prefer_ordering_index)
- MySQL 옵티마이저는 ORDER BY 또는 GROUP BY를 인덱스를 사용해 처리 가능한 경우 쿼리의 실행 계획에서 인덱스의 가중치를 높이 설정해서 실행됨.
- MySQL 8.0.21 버전부터는 MySQL 서버 옵티마이저가 ORDER BY를 위한 인덱스에 너무 가중치를 부여하지 않도록 `prefer_ordering_index` 옵티마이저 옵션이 추가됨.
    - `prefer_ordering_index` 옵티마이저 옵션의 기본값은 ON

