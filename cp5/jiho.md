# Chapter5 - 트랜잭션과 잠금

용어 간단 정리
(내가 알고 있는 대로 적어보기)

-   잠금: 마치 화장실 한 칸을 사용하는 것 처럼, 먼저 들어간 작업이 문을 잠그면 다른 작업이 먼저 들어간 작업에 영향을 주지 못하도록 하는 것.
-   트랜잭션: 원자성(all or nothing)을 보장하는 실행 단위. start tansaction으로 트랜잭션을 시작하고, commit/rollback으로 트랜잭션을 종료한다.
-   격리 수준: 트랜잭션이 수행 중에 있을 때, 동시에 수행되고 있는 다른 트랜잭션이 이미 수행 중인 트랜잭션에 대해 얼마만큼 공유/차단을 할 것인가.

(책에 나온 표현)

-   잠금: 동시성을 제어하기 위한 기능
-   트랜잭션: 작업의 완전성을 보장, 데이터의 정합성을 보장하기 위한 기능
-   격리 수준: 하나의 트랜잭션 내에서 또는 여러 트랜잭션 간의 작업 내용을 어떻게 공유하고 차단할 것인지를 결정하는 레벨

## 5.1 트랜잭션

### 5.1.1 MySQL에서의 트랜잭션

#### 1. MyISAM의 트랜잭션 미지원

```shell
mysql> CREATE TABLE tab_myisam (fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=MyISAM;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO tab_myisam (fdpk) VALUES (3);
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO tab_myisam (fdpk) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'tab_myisam.PRIMARY'
mysql> SELECT * FROM tab_myisam;
+------+
| fdpk |
+------+
|    1 |
|    2 |
|    3 |
+------+
3 rows in set (0.00 sec)
```

위와 같이 실패했음에도 불구하고, 1과 2는 들어간 걸 확인할 수 있다.

```shell
mysql> CREATE TABLE tab_innodb (fdpk INT NOT NULL, PRIMARY KEY (fdpk)) ENGINE=InnoDB;
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO tab_innodb (fdpk) VALUES (3);
Query OK, 1 row affected (0.00 sec)

mysql> INSERT INTO tab_innodb (fdpk) VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'tab_innodb.PRIMARY'
mysql> SELECT * FROM tab_innodb;
+------+
| fdpk |
+------+
|    3 |
+------+
1 row in set (0.00 sec)
```

innoDB에서는 실패한 경우 1과 2도 들어가지 않는다.

#### 2. 트랜잭션 범위 설정의 주의사항

-   커넥션을 가지고 있는 범위와 트랜잭션이 활성화돼 있는 프로그램의 범위를 최소화해야 한다.
    ![Pasted image 20250112002212](https://github.com/user-attachments/assets/ebcd44c9-7ad9-4188-a0af-dabfa340ad1f)

## 5.2 MySQL 엔진의 잠금

-   MySQL 엔진 레벨의 락은 모든 스토리지 엔진에게 영향을 줌. (스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향 X)
-   종류
    -   테이블 데이터 동기화를 위한 테이블 락
    -   테이블의 구조를 잠그는 메타데이터 락
    -   사용자의 필요에 맞게 사용할 수 있는 네임드 락

### 5.2.1 글로벌 락

`FLUSH TABLES WITH READ LOCK`

-   `FLUSH TABLES`: 메모리(버퍼)에 있는 테이블 데이터를 디스크에 쓰고
-   `WITH READ LOCK`: 전역적인 읽기 잠금을 설정 (SELECT 가능, INSERT UPDATE, DELETE 불가능)
-   MySQL 서버 전체에 영향을 미침. 즉, 해당 MySQL 서버의 모든 데이터베이스와 테이블에 영향을 미침.
-   여러 데이터베이스에 존재하는 테이블들에 대해 백업이 필요할 때 사용함.
-   그러나 웬만해서는 사용하지 않는 것이 좋음.

`LOCK INSTANCE FOR BACKUP;` & `UNLOCK INSTANCE;`

-   백업 락
-   InnoDB 사용이 일반화 되면서 도입됨. InnoDB가 트랜잭션을 지원하기 때문에 모든 데이터 변경을 멈출 필요가 없어짐.
-   백업 락을 획득하면 모든 세션에서 테이블 스키마 변경, 사용자 인증 관리 정보 변경이 불가능함
-   그러나 데이터 변경은 허용됨

### 5.2.2 테이블 락

-   명시적 테이블 락
    -   `LOCK TABLES table_name` 명령으로 테이블 락을 획득할 수 있으나
    -   거의 쓸 일이 없음.
-   묵시적 테이블 락
    -   MyISAM, MEMORY 스토리지 엔진을 쓸 경우
        -   데이터 변경 쿼리 실행 시 잠금 시작, 변경 후 즉시 잠금 해제
    -   InnoDB는 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공
        -   단순 쿼리(DML)로 인한 묵시적 테이블 락X (DML 쿼리에선 무시)
        -   DDL의 경우 O

### 5.2.3 네임드 락

-   **문자열**에 대해 잠금을 설정함
-   언제 사용?
    1.  WAS 5대와 MySQL server 1대, 5대의 WAS가 어떤 정보를 동기화해야하는 경우
    2.  배치 프로그램처럼 한 번에 많은 레코드를 변경하는 쿼리는 데드락을 유발함. 이 때에 해결 방은으로써 네임드 락을 사용할 수 있음. 동일 데이터를 변경하거나 참조하는 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행!

### 5.2.4 메타데이터 락

-   데이터베이스의 테이블아니 뷰의 구조를 변경할 때 자동으로 설정되는 잠금
-   사용자가 직접 설정 및 해제 X
-   특정 작업 시 자동으로 적용

-   테이블 이름 바꾸기

    -   방법1 (권장)
        `RENAME TABLE rank TO rank_backup, rank_new TO rank;`
        -   한 번에 두 작업을 실행하여 중간에 `Table not found 'rank'`오류 없이 적용 가능
    -   방법2 (비권장)
        `RENAME TABLE rank TO rank_backup;`
        `RENAME TABLE rank_new TO rank;`
        -   따로 실행하면 잠깐이라도 rank 테이블이 없는 순간이 생겨서 오류가 발생할 수 있음

-   대용량 테이블 구조 변경하기 (잠금과 트랜잭션 동시 사용)
    1.  새로운 구조의 테이블 생성
    2.  데이터를 여러 스레드로 나누어 복사 (병렬 처리)
    3.  남은 최신 데이터(2번 과정에서 발생한 데이터들) 복사 (이때 잠시 락이 걸려 INSERT 불가능)
    4.  복사가 완료되면 RENAME 명령으로 새로운 테이블을 서비스로 투입
    5.  기존 테이블 삭제
        (구체적인 내용은 P165-P166)

## 5.3 InnoDB 스토리지 엔진 잠금

-   레코드 기반의 잠금 방식 (MyISAM보다 훨씬 뛰어난 동시성 처리)

    -   테이블 전체가 아닌, 실제 접근 레코드만 잠그는 방식

        ```sql
        -- MyISAM의 경우 (테이블 전체 잠금)
        UPDATE users SET name = '김철수' WHERE id = 1;
        -- 이 쿼리 실행 시 users 테이블 전체가 잠김

        -- InnoDB의 경우 (레코드 단위 잠금)
        UPDATE users SET name = '김철수' WHERE id = 1;
        -- id가 1인 레코드만 잠김, 다른 레코드는 계속 수정 가능
        ```

-   이원화된 잠금 처리
    -   MySQL 엔진과 InnoDB 스토리지 엔진 각각에서 잠금을 따로 관리함
-   잠금 모니터링
    -   information_schema 데이터베이스에 존재하는 INNODB_TRX, INNODB_LOCKS, INNODB_LOCK_WAITS 테이블을 조인해서 조회
    -   Performance Schema를 이용한 모니터링 방법

### 5.3.1 InnoDB 스토리지 엔진의 잠금

-   레코드 락
    -   레코드 자체만을 잠그는 것
    -   인덱스의 레코드를 잠근다. (인덱스가 하나도 없더라도 내부적으로 자동 생성된 인덱스를 이용해 잠금을 설정한다.)
    -   PK 또는 유니크 인덱스에 의한 변경 작업 시 레코드 락이 걸린다.
-   갭 락
    -   레코드와 가장 인접한 다른 레코드 사이의 '간격'을 잠그는 것. 즉, 레코드 사이에 새로운 레코드가 생성되는 것을 제어하는 것.
-   넥스트 키 락
    -   레코드 락 + 해당 레코드의 뒷쪽 갭 락
    -   왜 필요한가? 레플리카 서버에서도 동일한 결과를 만들어내는 것을 보장하기 위해 사용 (중간에 다른 데이터가 삽입되면 결과가 달라질 수 있기 때문)
    -   그러나 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 트랜잭션이 기다리는 경우가 많으므로 가능하면 사용 줄이기
-   자동 증가 락
    -   `AUTO_INCREMENT`<- 증가하는 일련번호 추출을 위해 사용
    -   테이블 수준의 잠금
    -   AUTO_INCREMENT 값이 한 번 증가하면 절대 줄어들지 않는 이유?
        -   AUTO_INCREMENT 잠금을 최소화하기 위해서!
        -   설령 INSERT가 실패했더라도 한 번 증가된 값은 그대로 유지됨

### 5.3.2 인덱스와 잠금

![Pasted image 20250113010038](https://github.com/user-attachments/assets/9046b753-55f6-4506-9df3-2fbc2e97f525)

-   인덱스를 어디에 걸었느냐에 따라 잠금의 범위가 달라진다.
-   위와 같은 사진처럼, first_name에만 인덱스를 건 상황을 가정해보자. first_name이 Georgy인 사람은 253명이고, 그 중 Klassen이라는 last_name을 가진 사람이 단 한 명이다. 이 경우 `UPDATE employees SET hire_date=NOW() WHERE first_name='Georgi' AND last_name='Klassen'`문을 날리게 되면 253 개의 인덱스에 모두 다 락이 걸린다. 오로지 한 개의 레코드 변경을 위해 모두 락이 걸리는 셈이다.

### 5.3.3 레코드 수준의 잠금 확인 및 해제

-   확인 방법: `SHOW PROCESSLIST;`
    -   performance_schema의 data_locks 테이블과 data_lock_waits 테이블을 조인
    -   자세한 것은 p174~p175
-   해제 방법: `KILL 17` (17은 스레드 번호)

## 5.4 MySQL의 격리 수준

-   READ UNCOMMITTED는 DIRTY READ라고 불릴 정도로, 일반적인 DBMS에선 거의 사용 안 됨
-   SERIALIZABLE 또한 동시성이 중요한 DBMS에선 거의 사용 안 됨
    <br><br>
-   DIRTY READ: 커밋되지 않은 변경 사항을 다른 트랜잭션에서 읽을 수 있는 현상
-   NON-REPEATABLE READ: 한 트랜잭션 내에서 같은 행을 두 번 읽었을 때 결과가 다른 현상
-   PHANTOM READ: 데이터 삽입으로 인한 결과 집합 불일치
    <br><br>
-   READ UNCOMMITTED -> READ COMMITTED: 더티 리드 해결
-   READ COMMITTED -> REPEATABLE READ: 논 리피터블 리드 해결
-   REPEATABLE READ -> SERIALIZABLE: 펜텀 리드 해결 (MySQL InnoDB는 REPEATABLE READ에서도 넥스트 키 락으로 해결)

### 5.4.1 READ UNCOMMITTED

-   DIRTY READ가 발생할 수 있으므로 쓰지 말자. READ COMMITTED 이상을 쓰자.
-   DIRTY READ: 커밋되지 않은 변경 사항을 다른 트랜잭션에서 읽을 수 있는 현상

    ```SQL
    -- 트랜잭션 1
    BEGIN;
    UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
    -- 아직 COMMIT하지 않은 상태

    -- 트랜잭션 2
    BEGIN;
    SELECT balance FROM accounts WHERE id = 1;
    -- 트랜잭션 1이 커밋되기 전의 변경된 값을 읽음
    -- 이후 트랜잭션 1이 롤백되면 이 값은 잘못된 값이었던 것
    ```

### 5.4.2 READ COMMITTED

-   오라클 DBMS에서 주로 사용되는 격리 수준
-   InnoDB에서는 이 READ COMMITTED를 위해 SELECT를 할 적에 언두 로그에 있는 것을 가져다가 보여준다.
-   그러나 READ COMMITTED는 NON-REPEATABLE READ가 발생할 수 있다.

    -   REPEATABLE READ 정합성: 한 트랜잭션 내에서 같은 SELECT 쿼리를 실행했을 땐 언제나 같은 결과를 가져와야 한다

-   NON-REPEATABLE READ: 한 트랜잭션 내에서 같은 행을 두 번 읽었을 때 결과가 다른 현상

    ```SQL
    -- 트랜잭션 1
    BEGIN;
    SELECT balance FROM accounts WHERE id = 1;  -- 결과: 50000원

    -- 트랜잭션 2에서 업데이트 실행 & 커밋
    UPDATE accounts SET balance = 40000 WHERE id = 1;
    COMMIT; -- 커밋

    -- 트랜잭션 1
    SELECT balance FROM accounts WHERE id = 1;  -- 결과: 40000원
    -- 같은 트랜잭션 내에서 같은 쿼리의 결과가 다름
    ```

### 5.4.3 REPEATABLE READ

-   SELECT할 적에 언두로그를 활용한다는 것은 동일함. 다만, 커밋된 것이더라도 해당 트랜잭션 안에서 실행되는 모든 SELECT 쿼리는 트랜잭션 번호가 자신의 트랜잭션 번호보다 낮은 트랜잭션에서 변경한 것만 보인다. (10번 트랜잭션은 10보다 작은 트랜잭션에서의 커밋 결과만 볼 수 있음)
    ![Pasted image 20250113022429](https://github.com/user-attachments/assets/fce56655-eb51-4c73-8a5b-e726173ff76e)
-   다만 트랜잭션이 장시간 종료되지 않아 언두 로그에 많은 내용이 쌓인 경우, 처리 성능이 떨어질 수 있다.

-   PHANTOM READ: 데이터 삽입으로 인한 결과 집합 불일치

    ```SQL
    -- 트랜잭션 1
    BEGIN;
    SELECT COUNT(*) FROM accounts WHERE balance >= 10000;  -- 결과: 5개

    -- 트랜잭션 2에서 INSERT 실행 & 커밋
    INSERT INTO accounts VALUES (6, 20000);
    COMMIT;

    -- 트랜잭션 1
    SELECT COUNT(*) FROM accounts WHERE balance >= 10000;  -- 결과: 6개
    -- 같은 조건으로 조회했는데 레코드가 추가됨
    ```

### 5.4.4 SERIALIZABLE

-   가장 엄격
-   읽기조차 잠금을 획득해야 함 이로 인해 레코드 변경 불가
