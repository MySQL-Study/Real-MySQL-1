# Chapter8 - 인덱스(2) (8.4 ~ 8절 끝)

## 8.4 R-Tree 인덱스

-   공간에서 포함 관계를 비교할 때에 인덱스를 활용할 수 있다. R-Tree 인덱스로 저장된 경우 이 기능이 가능하다.
-   R-Tree 인덱스는 MBR(Minimum Boundary Rectangle)을 토대로 포함 관계가 B-Tree 형식처럼 구현되어 있다. 즉, 어떤 도형을 감싸는 최소 사각형을 만들고, 각 사각형 2개씩을 포함하는 중간 사각형을 만들고, 또 중간 사각형 n개를 포함하는 최고 사각형을 만들고... 하면서 사각형 간의 포함관계가 나타내질 수 있다. 가장 큰 사각형들이 루트가 되고, 그 사각형들 내에 포함되는 중간 사각형들이 브랜치 노드가 되며, 가장 최소 사각형이 리프 노드가 된다.
-   `ST_Contains()`나 `ST_Within()` 비교를 토대로 포함 여부를 인덱스로 확인할 수 있다 :
    -   `SELECT * FROM tb_location WHERE ST_Contains(사각 상자, px);`
    -   `SELECT * FROM tb_location WHERE ST_Within(px, 사각상자);`
    -   점 p의 위치를 기준으로 하는 반경 5km이내의 점들은 원 내에 찍힐 수 있다. 그리고 그 원을 포함하는 MBR, 즉 최소 사각형 내에 있는 점들을 구하는 쿼리이다.

## 8.5 전문 검색 인덱스

-   B-Tree 인덱스는 인덱스 걸린 컬럼 전체를 기반으로 찾아지는 게 아니라, InnoDB 기준 3072B까지만 잘라서 인덱스 키로 사용한다.
-   B-Tree 인덱스 특성상 전문/좌측 일치 판단에만 사용이 가능하다.
-   문서 전체 내용을 인덱스화해서 특정 키워드가 포함된 문서를 검색하는 Full Text(전문) 검색에는 일반적인 B-Tree를 사용하기 어렵다.
-   문서 전체 검색을 위한 인덱싱 알고리즘이 전문 검색 인덱스이다.

### 8.5.1 인덱스 알고리즘

-   검색할 키워드를 분석하여 이 키워드를 검색용으로 인덱스 구축함.

방식은 다음과 같다:

1. 어근 분석 알고리즘: word가 아닌 것들을 모두 필터링한 뒤, MeCab을 통해 형태소를 분석해서 명사와 조사를 구분한다. MeCab을 MySQL에 적용하는 건 쉬우나, 한글에 알맞는 완성도를 갖춰나가는 게 어려울 수 있다.
2. n-gram 알고리즘: 어근 분석보다 훨씬 간단. 그냥 언어 특성과 관계 없이 단순히 n개 길이만큼 단어를 쪼개어 인덱싱 하는 것. 그리고 이 쪼개어진 토큰들 중 불용어는 제거한 뒤, 최종 인덱스를 등록한다.

위 두 방식 모두 불용어 처리 과정이 있다. 이 때엔 MySQL 내장 불용어들이 사용된다. 하지만 불용어 처리 과정이 때론 사용자를 더 혼란스럽게 할 수 있다. 이를 위해 불용어 처리를 무시하거나, 사용자가 불용어를 정의하여 사용할 수 있다.

### 8.5.2 전문 검색 인덱스의 가용성

다음 두 조건이 지켜져야 전문 검색 인덱스를 사용할 수 있다.

1. 테이블 전문 검색 대상 칼럼에 대해서 전문 인덱스 보유
    ```sql
    CREATE TABLE tb_test (
    	doc_id INT,
    	doc_body TEXT,
    	PRIMARY KEY (doc_id),
    	FULLTEXT KEY fx_docbody (doc_body) WITH PARSER ngram
    ) ENGINE=InnoDB;
    ```
2. 쿼리 문장이 전문 검색을 위한 문법(MATCH ... AGAINST ...)을 사용
    - 이렇게 하면 풀테이블 스캔: `SELECT * FROM tb_test WHERE doc_body LIKE '%애플%'`;
    - 이렇게 해야 인덱스 스캔: `SELECT * FROM tb_test WHERE MATCH(doc_body) AGAINST ('애플' IN BOOLEAN MODE);`

## 8.6 함수 기반 인덱스

-   원래 인덱스는 컬럼 전체 또는 왼쪽 일부만 생성 가능
-   컬럼 값을 조금 변경해서 인덱스를 구축하고 싶을 경우라면 함수 기반 인덱스를 활용
-   함수 기반 인덱스 또한 인덱스의 내부 구조 및 유지 관리 방법은 B-Tree와 동일하다. 단지 인덱싱할 값을 계산하는 과정에 차이만 있을 뿐이다.

### 8.6.1 가상 칼럼을 이용한 인덱스

-   first_name 컬럼과 last_name 컬럼이 각각 있는 상황에서, 이름을 합쳐 full_name 기반 검색이 필요한 상황을 가정하자.
-   이 경우 다음과 같이 가상 컬럼을 추가하고, 그 가상 컬럼에 인덱스를 생성할 수 있다.
    ```sql
    ALTER TABLE user
    	ADD full_name VARCHAR(30) AS (CONCAT(first_name, ' ', last_name)) VIRTUAL,
    	ADD INDEX ix_fullname (full_name);
    ```
-   이후 full_name 컬럼 검색도 새로 만들어진 인덱스를 이용해 실행 계획이 만들어진다.

### 8.6.2 함수를 이용한 인덱스

다음과 같이 함수를 직접 사용하여 인덱스를 걸 수도 있다:

```sql
CREATE TABLE user (
	user_id BIGINT,
	first_name VARCHAR(10),
	last_name VARCHAR(10),
	PRIMARY KEY (user_id),
	INDEX ix_fullname ((CONCAT(first_name, ' ', last_name)))
);
```

이 경우에는 꼭 인덱스 생성시 명시한 함수대로 쿼리를 날려야 옵티마이저가 인덱스를 탄다. 그렇지 않으면 같은 결과일지라도 이 인덱스를 사용하지 못한다.

```sql
EXPLAIN SELECT * FROM user WHERE CONCAT(first_name, ' ', last_name)='Matt Lee';
```

## 8.7 멀티 밸류 인덱스

-   MySQL에서 JSON 타입으로 저장하는 경우에 있어서, 배열 value에 대한 인덱스를 걸어줄 수 있어졌다.

```sql
CREATE TABLE user (
	user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
	first_name VARCHAR(10),
	last_name VARCHAR(10),
	credit_info JSON,
	INDEX mx_creditscores ( (CAST(credit_info->'$.credit_scores' AS UNSIGNED ARRAY)) )
);

INSERT INTO user VALUES (1, 'Matt', 'Lee', '{"credit_scores":[360, 353, 351]}');
```

위 경우 인덱스를 타려면 다음 함수들을 이용해야만 한다.

-   MEMBER OF()
-   JSON_CONTAINS()
-   JSON_OVERLAPS()

```sql
SELECT * FROM user WHERE 360 MEMBER OF(credit_info->'$.credit_scores');
```

## 8.8 클러스터링 인덱스

### 8.8.1 클러스터링 인덱스

-   클러스터링 인덱스, 논클러스터링 인덱스는 어떠한 인덱스 종류가 아니라, **레코드의 저장 방식**에 대한 용어임.
-   레코드가 PK 값으로 정렬되어 저장되는 경우를 '클러스터링 인덱스'라고 부른다.
-   InnoDB에서만 이 클러스터링 인덱스 방식을 사용한다.
-   물리 구조를 봤을 때, 세컨더리 인덱스를 위한 B-Tree는 리프 노드에 레코드의 PK가 기록된다. 반면 PK의 B-Tree를 봤을 때, 리프 노드에는 모든 컬럼의 값들이 다 저장돼있다.
-   때문에 PK 값을 변경하게 되면, 실제 데이터 레코드의 위치가 변경된다. 물론 PK값이 변경되는 경우는 잘 없겠지만.
-   프라이머리 키가 없는 테이블의 경우, InnoDB 스토리지 엔진이 프라이머리 키를 대체할 컬럼을 선택한다. 적절한 클러스터링 키 후보를 찾지 못하면 InnoDB 스토리지 엔진이 내부적으로 레코드의 일련번호 컬럼을 생성한다.

### 8.8.2 세컨더리 인덱스에 미치는 영향

-   MyISAM, MEMORY 스토리지 엔진의 경우 레코드가 한 번 저장되면 그 위치는 절대 바뀌지 않는다. 이러한 물리적 저장 주소를 ROWID라고 부른다. 프라이머리 키나 세컨더리 키가 그 레코드를 찾아올 때에는 해당 ROWID를 기반으로 접근하게 된다.
-   하지만 InnoDB의 세컨더리 인덱스에서도 물리적 저장 주소를 토대로 찾아오는 건 곤란하다. PK 기준으로 정렬되어 저장되기 때문에, 클러스터링 키 값 변경으로 인하여 재정렬된다면 세컨더리 인덱스에서도 주소를 다 바꾸어주어야 한다.
-   때문에 InnoDB 세컨더리 인덱스에서는 PK(논리주소)를 기록하며, PK 인덱스를 타서 데이터에 접근하게 된다.

### 8.8.3 클러스터링 인덱스의 장점과 단점

#### 장점

-   PK 기준으로 검색할 때 처리 성능이 빠르다. 특히 범위검색에 강하다.
-   테이블의 모든 세컨더리 인덱스가 PK를 가지고 있기 때문에 커버링 인덱스로 처리될 경우가 많다. (인덱스만으로 처리 가능!)

#### 단점

-   모든 세컨더리 인덱스가 클러스터링 키를 가지기 때문에, 클러스터링 키 값의 크기가 클 경우 인덱스 크기가 커진다.
-   세컨더리 인덱스를 통해 검색할 때, PK를 한 번 더 타고 들어가야하기 때문에 처리 성능이 MyISAM의 세컨더리 인덱스 방식 대비 느리다.
-   INSERT할 때 PK에 의해 레코드 위치가 결정되므로 그냥 순서 상관없이 INSERT하는 MyISAM 대비 처리 성능이 느리다.
-   PK를 변경하는 경우 레코드를 DELETE하고 다시 INSERT해주어야 하기 때문에 처리 성능이 느리다.

#### 결론

-   장점은 빠른 읽기, 단점은 느린 쓰기

### 8.8.4 클러스터링 테이블 사용 시 주의사항

1. PK 크기가 너무 커지지 않도록 유의하기
    - 일반적으로 테이블에 세컨더리 인덱스가 4~5개 정도 생성된다.
    - 이것을 고려할 때 PK 크기가 커지면 세컨더리 인덱스도 배로 커진다.
    - 그만큼 메모리가 많이 필요해지므로 InnoDB 테이블의 PK는 신중해야한다.
2. PK는 AUTO-INCREMENT보다는 업무적인 컬럼으로 생성하기
    - PK로 검색하는 경우 논클러스터링 테이블보다 훠얼씬 빠르게 처리할 수 있다. 그러니 이 장점을 활용하기 위해 단순 AUTO-INCREMENT를 사용하기보단, 업무적으로 대표되는 컬럼을 사용함으로써 검색 속도의 이점을 챙겨라.
    - 참고로 MyISAM, postgres같이 클러스터링이 아닌 경우엔 사실 PK로 뭘 선택하든 상관 없다. PK든 세컨더리 인덱스든 동일한 처리 속도를 가질 것이므로.
3. 프라이머리 키는 반드시 명시할 것
    - PK를 정의하지 않으면 InnoDB 내부에서 일련 번호 컬럼을 추가한다. 하지만 이 자동 추가 컬럼은 사용자가 볼 수 없고, 때문에 이를 토대로 조회할 수 없다. 이러면 클러스터링의 장점을 하나도 못 살린다.
    - 무의미한 AUTO_INCREMENT 컬럼을 PK로 설정하는 것과 다를 바 없다. 그러니 차라리 사용자가 접근할 수라도 있는 AUTO_INCREMENT로라도 명시하여 사용하라.
4. AUTO_INCREMENT를 인조 식별자로 활용하기
    - PK의 키가 길더라도 세컨더리 인덱스를 안 사용할 수 있으면 길게 하라.
    - 하지만 PK도 길고 세컨더리도 필요하다면 AUTO_INCREMENT 컬럼을 PK로 두고, 세컨더리를 만들어라. 이 경우의 AUTO_INCREMENT를 인조 식별자라고 한다.
    - 로그 파일처럼 INSERT 위주의 테이블은 AUTO_INCREMENT를 이용하여 PK를 인조 식별자로 두면 도움이 된다.

## 8.9 유니크 인덱스

-   인덱스가 있어야 유니크 제약 설정이 가능하다.
-   유니크 인덱스에선 NULL이 2개 이상 가능하다. NULL은 특정 값이 아니기 때문에!
-   물론 PK는 NULL을 허용하지 않는 유니크 속성이 자동으로 부여된다.

### 8.9.1 유니크 인덱스와 일반 세컨더리 인덱스 비교

-   읽기: 유니크와 일반 세컨더리 인덱스 간에 같은 1건을 읽게 된다고 했을 때 성능 차이는 거의 없다. 그냥 세컨더리 인덱스가 유니크 인덱스보다 통상 많은 레코드를 비교하게 될 뿐이지, 인덱스 자체의 읽기 성능이 느린 건 아니다.
-   쓰기: 유니크 인덱스는 일반 세컨더리 인덱스에 비해 쓰기 성능이 느리다. 키 값을 쓸 때 중복된 값이 있는지 체크하는 과정에서 읽기 잠금을 사용하고, 쓰기를 할 땐 쓰기 잠금을 사용하는데, 이 때에 데드락이 빈번하게 발생한다. 또한 단순 세컨더리 인덱스였다면 PK 저장을 버퍼링하기 위해 체인지 버퍼가 사용되어 빠르나, 유니크 인덱스의 경우 중복 체크를 해야하므로 버퍼링이 불가능하다.

### 8.9.2 유니크 인덱스 사용 시 주의사항

1. 유니크 인덱스가 성능 개선을 할거라 생각하고 불필요하게 만드는 경우를 주의하자. 오히려 락걸린다.
2. 같은 컬럼에 대해 유니크 인덱스와 일반 인덱스를 다 걸 필요 없다. 유니크 인덱스 자체가 일반 인덱스 역할을 하기 때문이다. 중복 인덱스를 걸지 말자.
3. 같은 컬럼에 대해 PK와 유니크 인덱스를 동일하게 생성할 필요도 없다.

#### 결론

유일성이 꼭 보장되어야 하는 컬럼에만 인덱스 걸기. 필요하지 않다면 유니크 인덱스는 락이 걸리므로 쓰기에 느리니까 유니크하지 않은 세컨더리 인덱스를 활용해볼 것을 추천.

## 8.10 외래키

-   외래키 제약이 설정되면 자동으로 연관되는 테이블의 컬럼에 인덱스가 생성된다.
-   외래키에서는 다음 두 가지 특징을 기억해야 한다.
    -   테이블의 변경(쓰기 잠금)이 발생하는 경우에만 잠금 경합(잠금 대기)이 발생한다.
    -   외래키와 연관되지 않은 컬럼의 변경은 최대한 잠금 경합(잠금 대기)를 발생시키지 않는다.

### 8.10.1 자식 테이블의 변경이 대기하는 경우

| 작업번호 | 커넥션-1                                          | 커넥션-2                                   |
| -------- | ------------------------------------------------- | ------------------------------------------ |
| 1        | BEGIN;                                            |                                            |
| 2        | UPDATE tb_parent<br>SET fd='change-2' WHERE id=2; |                                            |
| 3        |                                                   | BEGIN;                                     |
| 4        |                                                   | UPDATE tb_child<br>SET pid=2 WHERE id=100; |
| 5        | ROLLBACK;                                         |                                            |
| 6        |                                                   | Query OK, 1 row affected (3.04 sec)        |

부모 테이블이 id=2 레코드의 쓰기 락을 걸고 들어갔다. 이로 인해 자식 테이블 측에서 부모 테이블 참조를 id=2로 바꿔보려고 하고 있으나, 레코드에 락이 걸려 대기하게 되었다.

### 8.10.2 부모 테이블의 변경 작업이 대기하는 경우

| 작업번호 | 커넥션-1                                                 | 커넥션-2                             |
| -------- | -------------------------------------------------------- | ------------------------------------ |
| 1        | BEGIN;                                                   |                                      |
| 2        | UPDATE tb_child<br>SET fd='changed-100'<br>WHERE id=100; |                                      |
| 3        |                                                          | BEGIN;                               |
| 4        |                                                          | DELETE FROM tb_parent<br>WHERE id=1; |
| 5        | ROLLBACK;                                                |                                      |
| 6        |                                                          | Query OK, 1 row affected (6.09 sec)  |

자식 테이블이 부모 테이블 레코드 중 id=1인 것을 참조 중에 있다. 이 경우 자식 테이블이 해당 레코드를 변경하는 데에 레코드의 쓰기 락을 걸고 들어갔다. 이로 인해 부모 테이블이 해당 테이블을 삭제하려고 하더라도 ON DELETE CASCADE가 걸려 있기 때문에 자식이 가져간 레코드 락이 풀려야 삭제가 동작할 수 있다. 이로 인해 부모 테이블이 대기된다.

### 외래키로 인해 느려지니까, 외래키를 쓰지 않는다?

-   이전에 호눅스님 마클에서 한 번 언급해주셨던 것 같음.
-   외래키를 안 쓴다는 게 아니고, 외래 키 제약조건을 걸지 않음으로써 락으로 인한 느려짐을 없애는 전략이 있다고 함.
-   대신 이 경우 제약조건을 BE 서비스 로직에서 잘 처리해주어야 함.
-   하지만 많이 느려지는 걸 확인하기 전이라면 굳이 외래 키를 안 쓸 이유가 있을까?
