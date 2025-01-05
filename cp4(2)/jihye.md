## InnoDB 스토리지 엔진 아키텍처
### InnoDB 버퍼 풀
- 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해 두는 공간
#### 버퍼 풀의 크기 설정
- 버퍼 풀의 크기를 줄이거나 늘릴 떄는 128MB단위로 처리
#### 버퍼 풀의 구조
- 버퍼 풀이라는 거대한 메모리 공간을 페이지 크기의 조각으로 쪼개어 엔진이 데이터를 필요로 할 때 해당 데이터 페이지를 읽어 각 조각에 저장
- 프리 리스트(Free)
  - 실제 사용자 데이터로 채워지지 않은 비어있는 페이지들의 목록
  - 사용자의 쿼리가 새롭게 디스크의 데이터 페이지를 읽어와야 하는 경우 사용
- LRU 리스트
  - LRU(Old 서브리스트)와 MRU(New 서브리스트)가 리스트가 결합된 형태
  - 처음 한 번 읽힌 데이터 페이지가 이후 자주 사용된다면 그 데이터 페이지는 MRU 영역에서 계속 살아남음
  - 거의 사용되지 않는다면 새롭게 디스크에서 읽히는 데이터 페이지들에 밀려서 LRU의 끝으로 밀려나 제거
- 플러시 리스트
  - 더티 페이지(디스크로 동기화하지 않은 데이터를 가진 데이터 페이지)의 변경 시점 기준의 페이지 목록을 관리
  - 데이터가 변경되면 리두 로그에 기록하고 버퍼 풀의 데이터 페이지에도 변경 내용을 반영
#### 버퍼 풀과 리두 로그
- 버퍼 풀은 서버의 메모리가 허용하는 만큼 크게 설정하면 할수록 쿼리의 성능이 빨라짐
- 버퍼 풀의 메모리 공간만 단순히 늘리는 것은 데이터 캐시 기능만 향상
- 활성 리두 로그: 재사용이 불가능한 공간
- LSN(Log Sequence Number): 리두 로그 파일의 공간은 계속 순환되어 재사용되지만 매번 기록될 때마다 로그 포지션은 계속 증가된 값을 가짐
- 체크포인트 에이지: 가장 최근 체크포인트의 LSN과 마지막 리두 로그 엔트리의 LSN의 차이, 활성 리두 로그 공간의 크기
#### 버퍼 풀 플러시
- 플러시 리스트 플러시
  - 오래된 리두 로그 공간이 지워지려면 반드시 버퍼 풀의 더티 페이지가 먼저 디스크로 동기화돼야 함
  - 주기적으로 플러시 리스트 플러시 함수를 호출해서 플러시 리스트에서 오래전에 변경된 데이터 페이지 순서대로 디스크에 동기화
  - 클리너 스레드: 더티 페이지를 디스크로 동기화하는 스레드
    - 가능하면 클리너 스레드 설정값과 버퍼 풀 인스턴스 설정값과 동일한 값으로 설정하기
- LRU 리스트 플러시
  - 사용 빈도가 낮은 데이터 페이지들을 제거해서 새로운 페이지들을 읽어올 공간을 만듦
  - 스캔하면서 더티 페이지는 디스크에 동기화, 클린 페이지는 즉시 프리 리스트로 페이지 이동
  - LRU 리스트의 스캔은 innodb_buffer_pool_instances * innodb_lru_scan_depth 수만큼 수행
#### 버퍼 풀 상태 백업 및 복구
- 워밍업: 디스크의 데이터가 버퍼 풀에 적재돼 있는 상태
- MySQL 5.6 버전부터는 버퍼 풀 덤프 및 적재 기능 도입 -> innodb_buffer_pool_dump_now 시스템 변수로 버퍼 풀 상태 백업 가능
- InnoDB 버퍼 풀의 백업은 데이터 디렉터리에 ib_buffer_pool 이름의 파일로 LRU 리스트에서 적재된 데이터 페이지의 메타 정보만 저장
- InnoDB 스토리지 엔진은 서버가 셧다운 되기 직전에 버퍼 풀의 백업을 실행하고 서버가 시작되면 자동으로 백업된 버퍼 풀의 상태를 복구할 수 있는 기능 제공
#### 버퍼 풀의 적재 내용 확인
- MySQL 8.0 버전부터 information_schema 데이터베이스에 innodb_cached_indexes 테이블 새로 추가
  - 테이블의 인덱스별로 데이터 페이지가 얼마나 InnoDB 버퍼 풀에 적재돼 있는지 확인 가능
### Double Write Buffer
- InnoDB 스토리지 엔진의 리두 로그는 공간의 낭비를 막기 위해 페이지의 변경된 내용만 기록
- 페이지가 일부만 기록되는 현상: 파결 페이지 또는 톤 페이지
- 위 문제를 막기 위해 Double-Write 기법 이용
  - 변경된 데이터 페이지를 모아서 한 번의 디스크 쓰기로 시스템 테이블스페이스의 DoubleWrite 버퍼에 기록
  - InnoDB 스토리지 엔진은 각 더티 페이지를 파일의 적당한 위치에 하나씩 랜덤으로 쓰기 실행
- DoubleWrite 버퍼의 내용은 실제 데이터 파일의 쓰기가 중간에 실패할 때만 원래의 목적으로 사용됨
### 언두 로그
- InnoDB 스토리지 엔진은 트랜잭션과 격리 수준을 보장하기 위해 DML로 변경되기 이전 버전의 데이터를 별도로 백업
- 언두 로그 모니터링
  - MySQL 5.5 버전까지는 언두 로그의 사용 공간이 한 번 늘어나면 서버를 새로 구축하지 않는 한 줄일 수 없었음
  - 8.0에서는 언두 로그를 돌아가면서 순차적으로 사용해 디스크 공간을 줄이는 것도 가능하며 서버가 필요한 시점에 사용 공간을 자동으로 감소도 가능
  - 서비스 중인 서버에서 활성 상태의 트랜잭션이 장시간 유지되는 것을 성능상 좋지 않기 때문에 언두 로그가 얼마나 증가했는지 항상 모니터링하는 것이 좋음
  - *급증했을 때 처리법 찾아보기*
- 언두 테이블스페이스 관리
  - 언두 테이블스페이스: 언두 로그가 저장되는 공간
  - MySQL 5.6 이전 버전에서는 언두 로그가 모두 시스템 테이블스페이스(ibdata.idb)에 저장
  - 8.0.14 버전부터 언두 로그는 항상 시스템 테이블스페이스 외부의 별도 로그 파일에 기록되도록 개선
  - 하나의 언두 테이블스페이스는 1개 이상 128개 이하의 롤백 세그먼트를 가지며 하나의 롤백 세그먼트는 InnoDB의 페이지 크기를 16바이트로 나눈 값의 언두 슬롯을 가짐
  - `최대 동시 트랜잭션 수 = (InnoDB 페이지 크기) / 16 * (롤백 세그먼트 개수) * (언두 테이즣스페이스 개수)`
  - MySQL 8.0 이전까지는 한 번 생성된 언두 로그는 변경이 허용되지 않고 정적으로 사용됨
  - 8.0 버전부터는 `CREATE UNDO TABLESPACE`나 `DROP TABLESPACE` 같은 명령으로 새로운 언두 테이블스페이스를 동적으로 추가하고 삭제하도록 개선
  - `Undo tablespace truncate`: 언두 테이블스페이스 공간을 필요한 만큼만 남기고 불필요하거나 과도하게 할당된 공간을 운영체제로 반납
### 체인지 버퍼
- 변경해야 할 인덱스 페이지가 버퍼 풀에 있으면 바로 업데이트를 수행하지만 그렇지 않고 디스크로부터 읽어와서 업데이트해야 한다면 이를 즉시 실행하지 않고 임시 공간에 저장해 두고 바로 사용자에게 결과를 반환하는 형태로 향상
- 이때 사용하는 임시 메모리 공간 = 체인지 버퍼
- 반드시 중복 여부를 체크해야 하는 유니크 인덱스는 체인지 버퍼 사용 불가
### 리두 로그 및 로그 버퍼
- 리두 로그는 하드웨어나 소프트웨어 등 여러 가지 문제점으로 인해 서버가 비정상적으로 종료됐을 때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전 장치
- 커밋됐지만 데이터 파일에 기록되지 않은 데이터: 리두 로그에 저장된 데이털르 데이터 파일에 다시 복사
- 롤백됐지만 데이터 파일에 이미 기록된 데이터: 변경되기 전 데이터를 가진 언두 로그의 내용을 가져와 데이터 파일에 복사ㅁ
- 리두 로그를 어느 주기로 디스크에 동기화할지를 결정하는 `innodb_flush_log_at_trx_commit` 시스템 변수 제공
- 로그 버퍼: 리두 로그 버퍼링에 사용되는 공간
#### 리두 로그 아카이빙
- 8.0 버전부터 리두 로그를 아카이빙할 수 있는 기능 추가
- 데이터 변경이 많아서 리두 로그가 덮어쓰인다고 하더라도 백업이 실패하지 않게 함
- `innodb_redo_log_archive_dirs` 시스템 변수 설정
- 만약 리두 로그 아카이빙을 시작한 세션이 `innodb_redo_log_archive_stop UDF`를 실행하기 전에 연결이 끊어진다면 엔진은 리두 로그 아카이빙을 멈추고 아카이빙 파일도 자동으로 삭제
#### 리두 로그 활성화 및 비활성화
- 서버에서 트랜잭션이 커밋돼도 데이터 파일을 즉시 디스크로 동기화되지 않는 반면 리두 로그(트랜잭션 로그)는 항상 디스크로 기록됨
- 8.0 버전부터는 수동으로 리두 로그를 활성화하거나 비활성화할 수 있게 됨
- `ALTER [ENABLE | DISABLE] INNODB REDO_LOG`
### 어댑티브 해시 인덱스
- 사용자가 수동으로 생성하는 인덱스가 아니라 InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스
- B-Tree 검색 시간을 줄여주기 위해 도입된 기능
- 어댑티브 해시 인덱스가 성능 향상에 크게 도움이 되지 않는 경우
  - 디스크 읽기가 많은 경우
  - 특정 패턴의 쿼리가 많은 경우(조인이나 LIKE 패턴 검색)
  - 매우 큰 데이터를 가진 테이블의 레코드를 폭넓게 읽는 경우
- `SHOW ENGINE INNODB STATUS\G`
  - `1.03 hash searches/s, 2.64 non-hash serches/s`
  - 초당 3.67번의 검색 중 1.03번을 어댑티브 해시 인덱스를 사용
- 어댑티브 해시 인덱스의 효울은 검색 횟수가 아니라 두 값의 비율(해시 인덱스 히트율)과 어댑티브 해시 인덱스가 사용 중인 메모리 공간, 그리고 서버의 CPU 시용량을 종합해서 판단
### InnoDB와 MyISAM, MEMORY 스토리지 엔진 비교
- MySQL 8.0버전부터는 서버의 모든 기능을 InnoDB 스토리지 엔진만으로 구현할 수 있게됨
- MEMORY 스토리지 엔진은 모든 처리를 메모리에서만 수행하니 하나의 스레드에서만 데이터를 읽고 쓴다면 InnoDB보다 빠를 수 있음
- 하지만 서버는 일반적으로 온라인 트랜잭션 처리를 위한 목적으로 사용되며 동시 처리 성능이 매우 중요
- MEMORY 스토리지 엔진은 임시 테이블의 용도로 사용됐지만 가변 길이 타입의 칼럼을 지원하지 않는다는 문제점 때문에 MySQL 8.0부터는 TempTable 스토리지 엔진이 대체해 사용됨
## MyISAM 스토리지 엔진 아키텍처
### 키 캐시
- 인덱스만을 대상으로 작동
- 인덱스의 디스크 쓰기 작업에 대해서만 부분적으로 버퍼링 역할
- `키 캐시 히트율 = 100 - (Key_reads / Key_read_requests * 100)`
- 메뉴얼에서는 일반적으로 키 캐시를 이용한 쿼리의 비율을 99% 이상으로 유지하라고 권장
- 미만이라면 키 캐시를 조금 더 크게 설정
### 운영체제의 캐시 및 버퍼
- MyISAM 테이블의 데이터에 대해서는 디스크로부터의 I/O를 해결해 줄 만한 어떠한 캐시나 버퍼링 기능도 가지고 있지 않음
- 그래서 MyISAM 테이블의 데이터 읽기나 씀기 작업은 항상 운영체제의 디스크 읽기 또는 쓰기 작업으로 요청
- MyISAM이 주로 사용되는 MySQL에서 일반적으로 키 캐시는 최대 물리 메모리의 40% 이상을 넘지 ㅇ낳게 설정하고 나머지 메모리 공간은 운영체제가 자체적인 파일 시스템을 위한 캐시 공간 마련하면 좋음
### 데이터 파일과 프라이머리 키(인덱스) 구조
- MyISAM 테이블에 레코드는 프라이머리 키 값과 무관하게 삽입되는 순서대로 데이터 파일에 저장
- MyISAM 테이블에 저장되는 레코드는 모두 ROWID라는 물리적인 주솟값을 가짐
## MySQL 로그 파일
### 에러 로그 파일
- MySQL이 실행되는 도중에 발생하는 에러나 경고 메시지가 출력되는 로그 파일
- MySQL이 시작하는 과정과 관련된 정보성 및 에러 메시지
- 마지막으로 종료할 때 비정상적으로 종료된 경우 나타나는 InnoDB의 트랜잭션 복구 메시지
- 쿼리 처리 도중에 발생하는 문제에 대한 에러 메시지
- 비정상적으로 종료된 커넥션 메시지
- InnoDB 모니터링 또는 상태 조회 명령의 결과 메시지
- MySQL의 종료 메시지
### 제너럴 쿼리 로그 파일(제너럴 로그 파일, General log)
- MySQL 서버에서 실행되는 쿼리로 어떤 것들이 있는지 확인할 때 쿼리 로그를 활성화해서 쿼리를 쿼리 로그 파일로 기록한 후 확인
### 슬로우 쿼리 로그
- 슬로우 쿼리 로그 파일에는 long_query_time 시스템 변수에 설정한 시간 이상의 시간이 소요된 쿼리가 모두 기록