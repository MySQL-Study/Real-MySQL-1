**table 칼럼**

MySQL 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아니라 테이블 기준으로 표시됩니다. 테이블의 이름에 별칭이 부여된 경우에는 별칭이 표시됩니다. 만약 별도의 테이블을 사용하지 않는 SELECT 쿼리인 경우에는 table 컬럼에 NULL이 표시됩니다. table 컬럼에 "<>"로 둘러싸인 이름이 명시되는 경우 이는 임시 테이블을 의미합니다.

**partitions 칼럼**

MySQL 8.0 버전부터 EXPLAIN 명령으로 파티션 관련 실행 계획까지 모두 확인할 수 있게 변경되었습니다. 실행 계획에서 조회 조건에 맞는 파티션만 접근하고 불필요한 나머지 파티션에 대해서는 분석을 실행하지 않습니다. 이를 파티션 프루닝이라고 합니다.

실행 계획을 보면 특정 파티션만 접근했다는 것을 알 수 있습니다. 이때 주의해야 될 점은 type에 ALL로 표기된다는 점입니다. partition은 물리적으로 개별 테이블처럼 별도의 저장 공간을 가지기 때문에 type에 파티션 일부만 읽는 쿼리라도 테이블 풀 스캔처럼 ALL로 표기됩니다.

**type 칼럼**

type 이후 컬럼은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타냅니다. MySQL의 메뉴얼에서는 type 컬럼을 조인 타입으로 소개하고 있습니다. 또한 MySQL에서는 하나의 테이블로부터 레코드를 읽는 작업도 조인처럼 처리합니다. 그래서 SELECT 쿼리의 테이블 개수에 관계없이 실행 계획의 type 컬럼을 조인 타입이라고 명시하고 있습니다.

ALL을 제외한 나머지 모두 인덱스를 사용하는 접근 방법입니다. 하나의 단위 SELECT 쿼리는 위의 접근 방법 중에서 단 하나만 사용할 수 있습니다. 또한 index_merge를 제외한 나머지 방법은 하나의 인덱스만 사용합니다.

- **system**: 레코드가 1건만 존재하는 테이블 또는 한 건도 존재하지 않는 테이블을 참조하는 형태의 접근 방법을 system이라고 합니다. 이 접근 방법은 InnoDB 스토리지 엔진을 사용하는 테이블에서는 나타나지 않습니다.
- **const**: 테이블의 레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 조건절을 가지고 있으며 반드시 1건을 반환하는 쿼리의 처리 방식을 const라고 합니다. 다른 DBMS에서는 이를 유니크 인덱스 스캔이라고 표현합니다.
- **eq_ref**: 조인에서 첫 번째 읽은 테이블의 컬럼 값을 이용해 두 번째 테이블을 프라이머리 키나 유니크 키로 동등 조건 검색(두 번째 테이블은 반드시 1건의 레코드만 반환)합니다.
- **ref**: 조인의 순서와 인덱스 종류에 관계없이 동등 조건으로 검색합니다(1건의 레코드만 반환된다는 보장이 없어도 됨). eq_ref와는 달리 조인의 순서와 관계없이 사용되며 또한 프라이머리키나 유니크키 등의 제약조건도 없습니다.
- **fulltext**: 전문 검색(Full-text Search) 인덱스를 사용해 레코드를 읽는 접근 방법입니다. 전문 검색용 인덱스가 테이블에 정의돼 있어야 오류가 발생하지 않고 사용이 가능합니다.
- **ref_or_null**: ref 접근 방법과 같은데 NULL 비교가 추가된 형태입니다.
- **unique_subquery**: WHERE 조건절에 사용될 수 있는 IN 형태의 쿼리를 위한 접근 방법입니다. 서브쿼리에서 중복되지 않는 유니크한 값만 반환할 때 이 접근 방법을 사용합니다.
- **index_subquery**: IN(subquery) 형태의 조건에서 subquery의 반환 값에 중복된 값이 있을 수 있지만 인덱스를 이용해 중복된 값을 제거할 수 있을 때 사용됩니다.
- **range**: 레인지 스캔 형태의 접근 방법입니다. 주로 <, >, IS NULL, BETWEEN, IN, LIKE 등의 연산자를 이용해 인덱스를 검색할 때 사용됩니다.
- **index_merge**: 2개 이상의 인덱스를 이용해 각각의 검색 결과를 만들어낸 후 그 결과를 병합해서 처리하는 방식입니다.
- **index**: 인덱스를 처음부터 끝까지 읽는 인덱스 풀 스캔을 의미합니다.
- **ALL**: 풀 테이블 스캔을 의미하는 접근 방법입니다.

**possible_keys 컬럼**

MySQL 옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정했던 접근 방법에서 사용되는 인덱스의 목록을 보여줍니다. 이 컬럼은 무시해도 되는데, 절대 possible_keys 컬럼에 인덱스 이름이 나열됐다고 해서 그 인덱스를 사용한다고 판단하지 말아야 합니다.

**key 컬럼**

최종 선택된 실행 계획에서 사용하는 인덱스를 의미합니다. PRIMARY인 경우에는 프라이머리 키를 사용한다는 의미이며 그 이외의 값은 모두 테이블이나 인덱스를 생성할 때 부여했던 고유 이름입니다. 실행 게획의 type이 ALL일 때와 같이 인덱스를 전혀 사용하지 못하면 key 컬럼은 NULL로 표시됩니다.

**key_len 컬럼**

쿼리를 처리하기 위해 다중 컬럼으로 구성된 인덱스에서 몇 개의 컬럼까지 사용했는지 알 수 있습니다. 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값입니다.

**ref 컬럼**

접근 방법이 ref면 참조 조건(Equal 비교 조건)으로 어떤 값이 제공됐는지 보여줍니다. 사용자가 명시적으로 값을 변환하거나 내부적으로 값을 변환할 때 ref 칼럼에 func라고 출력됩니다. 가급적 조인 칼럼의 타입은 일치시키는 것이 좋습니다.

**rows 컬럼**

MySQL 실행 계획의 rows 컬럼 값은 실행 계획의 효율성 판단을 위해 예측했던 레코드 건수를 보여줍니다. 이 값은 각 스토리지 엔진별로 가지고 있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이라서 정확하지는 않습니다.

**filtered 컬럼**

filtered 칼럼의 값은 필터링되어 버려지는 레코드의 비율이 아니라 필터링되고 남은 레코드의 비율을 의미합니다. rows * (filtered / 100) = 수행한 레코드의 건수로 계산할 수 있습니다. 일치하는 레코드 건수가 적은 테이블이 드라이빙 테이블이 되는 것이 조인의 횟수를 줄일 수 있어서 좋습니다.