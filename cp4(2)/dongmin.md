### InnoDB 버퍼 풀

- InnoDB 스토리지 엔진에서 가장 핵심적인 부분
- 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해 두는 공간
    - 쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역할도 같이 함.
- 일반적인 애플리케이션에서는 데이터를 변경하는 쿼리는 데이터 파일의 이곳저곳에 위치한 레코드를 변경해서 랜덤한 디스크 작업을 발생시킴.
- 버퍼 풀이 변경된 데이터를 모아서 처리하면 랜덤한 디스크 작업의 횟수를 줄일 수 있음.

### 버퍼 풀의 크기 설정

- 버퍼 풀의 크기는 운영체제와 각 클라이언트 스레드가 사용할 메모리도 충분히 고려해서 설정해야 함.
- 레코드 버퍼는 클라이언트 세션에서 테이블의 레코드를 읽고 쓸 때 버퍼로 사용하는 공간을 말함.
    - 커넥션이 많고 사용하는 테이블도 많다면 레코드 버퍼 용도로 사용되는 메모리 공간이 꽤 많이 필요해질 수 있음.
- MySQL 5.7 버넞부터는 InnoDB 버퍼 풀의 크기를 동적으로 조절할 수 있게 개선되었음.

### 버퍼 풀의 구조

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dd9eb0a2-9c82-43da-92ea-939474e849bb/fba6e648-5dd8-4beb-a923-d7ee79bb95e8/7fc3981b-6c92-4a41-b78b-0d3b71a7527b.png)

- 버퍼 풀의 페이지 크기 조각을 관리하기 위해 InnoDB 스토리지 엔진은 크게 LRU 리스트와 플러시 리스트, 프리 리스트라는 3개의 자료 구조를 관리함.
- InnoDB 스토리지 엔진에서 데이터를 찾는 과정을 다음과 같음
    - 필요한 레코드가 저장된 데이터 페이지가 버퍼 풀에 있는지 검사.
        - InnoDB 어댑티브 해시 인덱스를 이용해 페이지를 검색
        - 해당 테이블의 인덱스(B-Tree)를 이용해 버퍼 풀에서 페이지를 검색
            - 버퍼 풀에 이미 데이터 페이지가 있었다면 해당 페이지의 포인터를 MRU 방향으로 승급
    - 디스크에서 필요한 데이터 페이지를 버퍼 풀에 적재하고, 적재된 페이지에 대한 포인터를 LRU 헤더 부분에 추가
    - 버퍼 풀의 LRU 헤더 부분에 적재된 데이터 페이지가 실제로 읽히면 MRU 헤더 부분으로 이동
    - 버퍼 풀에 상주하는 데이터 페이지는 사용자 쿼리가 얼마나 최근에 접근했었는지에 따라 나이가 부여되며, 버퍼 풀에 상주하는 동안 쿼리에서 오랫동안 사용되지 않으면 데이터 페이지에 부여된 나이가 오래되고 결국 해당 페이지는 버퍼 풀에서 제거 된다. 버퍼 풀의 데이터 페이지가 쿼리에 의해 사용되면 나이가 초기화되어 다시 젊어지고 MRU 헤덩 부분으로 옮겨진다.
    - 필요한 데이터가 자주 접근됐다면 해당 페이지의 인덱스 키를 어댑티브 해시 인덱스에 추가.
- 플러시 리스트는 디스크로 동기화되지 않은 데이터를 가진 데이터 페이지의 변경 시점 기준의 페이지 목록을 관리함.

### 버퍼 풀과 Redo 로그

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dd9eb0a2-9c82-43da-92ea-939474e849bb/5eb85731-8cd2-4237-a88f-1b20adc83e69/f1f667e6-82b3-442c-8204-67cd4b234c78.png)

- InnoDB의 버퍼 풀은 서버의 메모리가 허용하는 만큼 크게 설정하면 할수록 쿼리의 성능은 나빠짐.
- 하지만 버퍼 풀은 데이터베이스 서버의 성능 향상을 위해 데이터 캐시와 쓰기 버퍼링이라는 두 가지 용도가 있는데, 버퍼 풀의 메모리 공간만 단수히 늘리는 것은 데이터 캐시 기능만 향상시키는 것이다.
- InnoDB의 버퍼 풀은 디스크에서 읽은 상태로 전혀 변경되지 않은 클린 페이지와 함께 INSERT, UPDATE, DELETE 명령으로 변경된 데이터를 가진 더티 페이지도 가지고 있다.

### 버퍼 풀 플러시

- InnoDB 스토리지 엔진은 버퍼 풀에서 아직 디스크로 기록되지 않은 더티 페이지들을 성능상의 악영향 없이 디스크에 동기화하기 위해 아래와 같이 2개의 플러시 기능을 백그라운드로 실행함.

### 1. Flush List Flush (변경된 데이터 동기화)

- 더티 페이지(변경된 페이지)들을 변경 시점 순서대로 디스크에 기록
- 클리너 스레드(Cleaner Thread)가 이 작업을 담당
- Adaptive Flush 기능으로 자동 최적화 지원

- `innodb_page_cleaners`: 클리너 스레드 수 조정
- `innodb_max_dirty_pages_pct`: 버퍼 풀의 더티 페이지 최대 비율 설정 (기본값 90%)
- `innodb_max_dirty_pages_pct_lwm`: 더티 페이지 동기화 시작 임계값
- `innodb_adaptive_flushing`: Adaptive Flush 기능 활성화 여부

1. 더티 페이지 관리
    - 과도한 더티 페이지는 디스크 I/O 버스트 유발 가능
    - 적절한 임계값 설정으로 점진적 동기화 구현
2. Adaptive Flush
    - Redo 로그 증가 속도 모니터링
    - 버퍼 풀 상태에 따른 자동 조절
    - 시스템 부하 균형 유지

### 2. LRU List Flush (메모리 공간 확보)

- LRU(Least Recently Used) 알고리즘 기반 페이지 관리
- 사용 빈도가 낮은 페이지를 대상으로 수행
- 새로운 페이지를 위한 공간 확보가 주 목적

### 프로세스

1. LRU 리스트 스캔
2. 더티 페이지 발견 시 디스크 동기화
3. 클린 페이지는 Free 리스트로 이동
4. 결과적으로 새로운 페이지를 위한 공간 확보

### MySQL 버퍼 풀 상태 관리

버퍼 풀의 효율적인 관리는 MySQL 성능에 직접적인 영향을 미치는 핵심 요소입니다. 특히 서버 재시작 시 발생하는 콜드 스타트 현상을 방지하기 위한 다양한 기능들이 버전별로 발전해왔습니다.

### 버퍼 풀 Warming Up의 중요성

- 디스크 I/O는 메모리 접근보다 수백~수천 배 느림
- 적절한 Warming Up으로 성능이 수십 배 향상 가능
- 서비스 초기 응답 시간 안정화에 필수적

`버전별 Warming Up 방식`

- MySQL 5.5 이전
    - 수동적인 Warming Up 방식 사용
    - 서비스 시작 전 주요 테이블 Full Scan 수행
    - 인덱스 강제 로드 필요

```sql

SELECT COUNT(*) FROM main_table FORCE INDEX(PRIMARY);
SELECT COUNT(*) FROM main_table FORCE INDEX(idx_column);

```

- MySQL 5.6 이후
    - 버퍼 풀 덤프/로드 기능 도입
    - 시스템 변수를 통한 제어 가능

```sql

SET GLOBAL innodb_buffer_pool_dump_now = ON
SET GLOBAL innodb_buffer_pool_load_now = ON;

```

### 버퍼 풀 모니터링

### MySQL 5.6 - 8.0

- `information_schema.INNODB_BUFFER_POOL_STATS`
    - 버퍼 풀 전반적인 상태 확인
    - 페이지 수, 히트율 등 확인 가능

```sql
SELECT * FROM information_schema.INNODB_BUFFER_POOL_STATS;

```

- `information_schema.INNODB_BUFFER_PAGE`
    - 페이지별 상세 정보 제공
    - 주의: 대용량 버퍼 풀에서 조회 시 성능 저하 발생

```sql
SELECT
    TABLE_NAME,
    COUNT(*) AS PAGE_COUNT,
    SUM(IF(IS_HASHED = 'YES', 1, 0)) AS HASHED_PAGES
FROM information_schema.INNODB_BUFFER_PAGE
GROUP BY TABLE_NAME;

```

### Double Write Buffer

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dd9eb0a2-9c82-43da-92ea-939474e849bb/6daef69e-a4ed-454a-a917-883c44b8ac4d/1954e830-4b42-4922-8d52-81f6826442a5.png)

redo 로그는..

- 공간 효율성을 위해 페이지의 변경된 부분만 기록
- 전체 페이지가 아닌 변경 사항만 저장

Partial-Page는..

- Dirty Page 플러싱 중 일부만 기록되는 현상
- 페이지 일부만 기록된 상태에서 장애 발생 시 복구 불가능
- 데이터 무결성 위협 요소

### Double Write 동작 방식

1. Dirty Page 묶음('A'~'E')을 Double Write 버퍼에 일괄 기록
2. Double Write 버퍼 기록 완료 후 실제 데이터 파일에 쓰기 수행

- 정상 동작 시: Double Write 버퍼 내용은 사용되지 않음
- 장애 발생 시:
    - InnoDB 재시작 시점에 검증 수행
    - Double Write 버퍼와 데이터 파일 페이지 비교
    - 불일치 발견 시 Double Write 버퍼의 내용으로 복구

### Undo 로그

- DML 연산 실행 전 데이터의 이전 상태를 보관하는 로그
- INSERT, UPDATE, DELETE 작업에 대한 변경 이전 데이터 백업

### Undo 로그의 진화

- MySQL 5.5 이전의 한계
    - Undo 로그 공간의 비탄력적 운영
        - 한번 증가한 공간 축소 불가
        - 대용량 데이터/장기 트랜잭션 시 공간 낭비
        - 과도한 백업 오버헤드 발생

- MySQL 5.7 이후 개선사항
    - 공간 관리의 유연성 확보
    - MySQL 8.0의 주요 개선
        - 순차적 Undo 로그 사용으로 디스크 공간 최적화
        - 동적 Undo 테이블스페이스 관리 가능

### Undo 로그 모니터링

모니터링 방법

```sql
sql
Copy
SHOW ENGINE INNODB STATUS \G

```

- 활성 트랜잭션 상태 확인
- 장기 실행 트랜잭션 식별
- 리소스 사용량 분석

### Undo 테이블스페이스 구조와 관리

- 구조적 특징
    - 1~128개의 롤백 세그먼트 포함
    - 각 롤백 세그먼트는 복수의 Undo Slot 보유
    - 트랜잭션당 최대 4개 Undo Slot 사용

용량 관리

- 기본 설정(16KB)으로 약 131,072 트랜잭션 동시 처리 가능
- MySQL 8.0의 동적 관리 기능

```sql

CREATE UNDO TABLESPACE tablespace_name;

DROP TABLESPACE tablespace_name;

```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/dd9eb0a2-9c82-43da-92ea-939474e849bb/e397aa5b-d94a-43a1-9658-b44448e2e5f2/a96cf966-0640-4025-b811-19ae05149ee8.png)

### 체인지 버퍼

Change Buffer는 인덱스 페이지 업데이트의 효율성을 높이기 위한 InnoDB의 메모리 영역입니다.

- 인덱스 업데이트 발생 시..
    - 대상 페이지가 버퍼 풀에 존재 → 즉시 업데이트
    - 대상 페이지가 버퍼 풀에 부재 → Change Buffer에 임시 저장
- Unique Index는 Change Buffer 사용 불가
    - 중복 검사 필요성으로 인한 제약
- MySQL 8.0 이전 버전: INSERT 작업만 지원

### 리두 로그 및 로그 버퍼

- 리두 로그
    - 데이터베이스의 Durability(지속성) 보장
    - 비정상 종료 시 데이터 복구 메커니즘 제공
    - 트랜잭션의 영속성 보장
    - 데이터 변경 사항을 순차적으로 기록
    - Write-Ahead Logging(WAL) 방식 사용
        - 데이터 파일 변경 전 로그 먼저 기록
        - 순차적 I/O로 성능 최적화

- 커밋된 트랜잭션 복구
    - 상황: 커밋은 됐으나 데이터 파일 미반영
    - 해결: Redo 로그에서 변경사항 재적용



- 롤백된 트랜잭션 처리
    - 상황: 롤백됐으나 데이터 파일에 기록된 경우
    - 용도: 트랜잭션 상태 추적 및 관리

### 어댑티브 해시 인덱스

Adaptive Hash Index(AHI)는 InnoDB에서 제공하는 자동 최적화 인덱스 메커니즘으로, 자주 접근하는 데이터에 대한 해시 기반 빠른 접근을 제공합니다.

- B-Tree 인덱스를 기반으로 동작
- 자주 접근하는 데이터에 대해 자동으로 해시 인덱스 생성
- 버퍼 풀 내의 데이터 페이지만 대상으로 관리
- B-Tree 인덱스 ID + 실제 키 값의 조합
- 전체 엔진에 단일 AHI 존재
    - 인덱스 고유번호로 구분 필요

- MySQL 8.0의 개선사항

  파티션 기능 도입으로 경합 감소


```sql
SET GLOBAL innodb_adaptive_hash_index_parts = value;
```

### 모니터링

```sql
SHOW ENGINE INNODB STATUS\G

```

- 해시 인덱스 사용 통계 확인
- 효율성 분석 가능

### InnoDB와 MyISAM, MEMORY 스토리지 엔진 비교

## 시대별 기본 스토리지 엔진

### MySQL 5.5 이전

- MyISAM이 기본 스토리지 엔진
- 시스템 테이블도 대부분 MyISAM 사용

### MySQL 5.5

- InnoDB가 기본 스토리지 엔진으로 전환
- 일부 시스템 테이블은 여전히 MyISAM 사용

### MySQL 8.0

- 모든 시스템 테이블이 InnoDB로 마이그레이션
- 완전한 InnoDB 중심 아키텍처로 전환

## 현재 상황

- InnoDB가 표준 스토리지 엔진으로 자리잡음
- MyISAM과 MEMORY 엔진은 레거시 상태
    - 지원 중단 가능성 존재
    - 새로운 프로젝트에서 사용 비권장