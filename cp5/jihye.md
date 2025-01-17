## CHAPTER 5. 트랜잭션과 잠금
- 트랜잭션: 작업의 완전성을 보장해 주는 것, 데이터의 정합성을 보장하기 위한 기능
- 잠금: 동시성을 제어하기 위한 기능

### 트랜잭션
- 하나의 논리적인 작업 셋에 하나의 쿼리가 있든 두 개 이상의 쿼리가 있든 관계없이 논리적인 작업 셋 자체가 100% 적용되거나 아무것도 적용되지 않아야 함을 보장해 주는 것
- MyISAM이나 MEMORY 스토리지 엔진은 트랜잭션을 지원하지 않음
- 프로그램 코드에서 트랜잭션의 범위 최소화
  - 일반적으로 데이터베이스 커넥션은 개수가 제한적이어서 각 단위 프로그램이 커넥션을 소유하는 시간이 길어질수록 사용 가능한 여유 커넥션의 개수는 줄어듦
  - 메일 전송이나 FTP 파일 전송 작업 또는 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 어떻게 해서든 DBMS의 트랜잭션 내에서 제거
    - 프로그램이 실행되는 동안 메일 서버와 통신할 수 없는 상황이 발생한다면 웹 서버뿐 아니라 DBMS 서버까지 위험해지는 상황 발생
    - DBMS 서버가 높은 부하 상태로 빠지거나 위험한 상태에 빠질 경우 발생

### MySQL 엔진의 잠금
- MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지 않음
#### 글로벌 락
- `FLUSH TABLES WITH READ LOCK` 명령으로 획득
- MySQL에서 제공하는 잠금 가운데 가장 범위 큼 -> MySQL 서버 전체
- MySQL 서버에 존재하는 모든 테이블을 닫고 잠금
- 여러 데이터베이스에 존재하는 MyISAM이나 MEMORY 테이블에 대해 mysqldump로 일관된 백업을 받아야 할 떄 사용
- MySQL 8.0부터는 백업 툴들의 안정적인 실행을 위해 백업 락 도입
  - `LOCK INSTANCE FOR BACKUP`
  - 일반적인 데이터 변경은 허용
  - 정상적으로 복제는 실행되지만 백업의 실패를 막기 위해 DDL 명령이 실행되면 복제를 일시 중지하는 역할

#### 테이블 락
- 개별 테이블 단위로 설정되는 잠금
- 명시적으로는 `LOCK TABLES tableA_name [ READ | WRITE ]` 명령으로 특정 테이블의 락 획득
- 묵시적으로는 MyISAM이나 MEMORY 테이블에 데이터를 변경하는 쿼리 실행하면 발생
- InnoDB 테이블은 레코드 기반 잠금 제공하기 때문에 테이블 락이 설정되지만 대부분의 데이터 변경 쿼리(DML)에서는 무시되고 스키마를 변경하는 쿼리(DDL)의 경우에만 영향 미침

#### 네임드 락
- `GET_LOCK()` 함수를 이용해 임의의 문자열에 대해 잠금 설정 가능
- 단순히 사용자가 지정한 문자열에 대해 획득하고 반남(해제)하는 잠금
- 동일 데이터를 변경하거나 참조하는 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행하면 데드락 해결 가능
- MySQL 8.0 버전부터는 네임드 락을 중첩해서 사용과 현재 세션에서 획득한 네임드 락을 한 번에 모두 해제도 가능

#### 메타데이터 락
- 데이터베이스 객체(테이블이나 뷰 등)의 이름이나 구조를 변경하는 경우에 획득하는 잠금
- `RENAME TABLE a TO b`같이 테이블의 이름을 변경하는 경우 자동으로 획득하는 잠금

### InnoDB 스토리지 엔진 잠금
#### InnoDB 스토리지 엔진의 잠금
- 레코드 기반의 잠금 기능 제공
- 레코드와 레코드 사이의 간격을 잠그는 갭(GAP) 락 존재
###### 레코드 락
- 레코드 자체만을 잠그는 것
- 인덱스의 레코드를 잠금
- 대부분 보조 인덱스를 이용한 변경 작업은 넥스트 키 락 또는 갭 락을 사용하지만 프라이머리 키 또는 유니크 인덱스에 의한 변경 작업에서는 갭에 대해서는 잠그지 않고 레코드 자체에 대해서만 락을 검

###### 갭 락
- 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것
- 레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어

###### 넥스트 키 락
- 레코드 락과 갭 락을 합쳐 놓은 형태의 잠금

###### 자동 증가 락
- 테이블 수준의 잠금
- 트랜잭션과 관계없이 INSERT나 REPLACE 문장에서 AUTO_INCREMENT 값을 가져오는 순간만 락이 걸렸다가 즉시 해제됨
- 자동 증가 락을 최소화하기 위해서 자동 증가 값이 한 번 증가하면 절대 줄어들지 않음
- MySQK 5.1 이상부터는 innodb_autonic_lock_mode라는 시스템 변수를 이용해 자동 증가 락의 작동 방식을 변경 가능
  - `innodb_autonic_lock_mode = 0`: MySQL 5.0과 동일한 잠금 방식으로 자동 증가 락 사용
  - `innodb_autonic_lock_mode = 1`: 연속 모드, 레코드의 건수를 정확하게 예측할 수 있을 때는 자동 증가 락을 사용하지 않고 훨씬 가볍고 빠른 래치(뮤텍스)를 이용해 처리
  - `innodb_autonic_lock_mode = 2`: 인터리빙 모드, 절대 자동 증가 락을 걸지 않고 경량화된 래치(뮤텍스) 사용, MySQL 8.0부터의 기본값

#### 인덱스와 잠금
- 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 함
- 테이블에 인덱스가 하나도 없다면 풀 스캔하면서 모든 레코드를 잠그면서 UPDATE함

#### 레코드 수준의 잠금 확인 및 해제
- MySQL 5.1부터 레코드 잠금과 잠금 대기에 대한 조회 가능 -> information_schema라는 DB에 INNODB_TRX, INNODB_LOCKS, INNODB_LOCK_WAITS 테이블을 통해 확인 가능
- MySQL 8.0부터는 performance_schema의 data_locks, data_lock_waits 테이블로 대체
- 강제로 잠금을 해제하려면 KILL 명려을 이용해 서버의 프로세스 강제로 종료

### MySQL의 격리 수준
- 여러 트랜잭션이 동시에 처리될 떄 특정 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼 수 있게 허용할지 말지를 결정하는 것

#### READ UNCOMMITTED
- 각 트랜잭션에서의 변경 내용이 COMMIT이나 ROLLBACK 여부에 상관없이 다른 트랜잭션에서 보임
- 더티 리드: 어떤 트랜잭션에서 처리한 작업이 완료되지 않았는데도 다른 트랜잭션에서 볼 수 있는 현상
- DIRTY READ, NON-REPEATABLE READ, PHANTOM READ 발생

#### READ COMMITTED
- 오라클 DBMS에서 기본으로 사용되는 격리 수준
- 어떤 트랜잭션에서 데이터를 변경했더라도 COMMIT이 완료된 데이터만 다른 트랜잭션에서 조회 가능
- 새로운 값은 테이블에 즉시 기록되고 이전 값은 언두 영역으로 백업됨
- REPEATABLE READ: 사용자가 하나의 트랜잭션 내에서 똑같은 SELECT 쿼리를 실행했을 때 항상 같은 결과를 가져와야 함 
- NON-REPEATABLE READ, PHANTOM READ 발생
- READ COMMITTED 격리 수준에서는 트랜잭션 내에서 실행되는 SELECT 문장과 트랜잭션 외부에서 실행되는 SELECT 문장의 차이가 별로 없음
- REPEATABLE READ 격리 수준에서는 기본적으로 SELECT 쿼리 문장도 트랜잭션 범위 내에서만 작동

#### REPEATABLE READ
- 바이너리 로그를 가진 MySQL 서버에서는 최소 REPEATABLE READ 격리 수준 이상을 사용해야 함
- InnoDB 스토리지 엔진은 트랜잭션이 ROLLBACK될 가능성에 대비해 변경되기 전 레코드를 언두 공간에 백업해두고 실제 레코드 값을 변경: MVCC
- 모든 InnoDB의 트랜잭션은 고유한 트랜잭션 번호(순차적으로 증가하는 값)를 가지며, 언두 영역에 백업된 모든 레코드에는 변경을 발생시킨 트랜잭션의 번호가 포함됨
- 언두 영역의 백업된 데이터는 InnoDB 스토리지 엔진이 불필요하다고 판단하는 시점에 주기적으로 삭제
- PHANTOM READ: 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상
- PHANTOM READ 발생(InnoDB는 없음)
  - InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 PHANTOM READ 발생 X

#### SERIALIZABLE
- 가장 엄격한 격리 수준
- 동시 처리 성능도 다른 트랜잭션 격리 수준보다 떨어짐
- 한 트랜잭션에서 읽고 쓰는 레코드를 다른 트랜잭션에서는 절대 접근 불가
