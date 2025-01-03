## 4.1 MySQL 엔진 아키텍처
MySQL 서버는 두 가지로 구분 가능
- `MySQL 엔진` : 사람의 머리 역할
- `스토리지 엔진` : 사람의 손발 역할, 핸들러 API를 만족하면, 눅든지 스토리지 엔진을 구현해서 MySQL 서버에 추가해서 설명 가능
  <br>

MySQL 서버에서는 기본적으로 `InnoDB 스토리지 엔진 ` 과 `MyISAM 스토리지 엔진` 제공됨.


### 4.1.1 MySQL의 전체 구조
MySQL 서버는 다른 DBMS에 비해 구조가 상당히 독특함.
- 독특한 구조 덕에 다른 DBMS에서 가질 수 없는 엄청난 혜택 가짐.
- 반대로, 다른 DBMS에서는 문제되지 않을 것들이 가끔 문제가 됨.

<img src="https://velog.velcdn.com/images/hyolim/post/b87bc59a-56f9-4272-97c4-cdc511c0eb22/image.png" width="500" height="auto">

- 일반 상용 RDBMS와 같이 대부분 프로그래밍 언어로부터 접근 방법을 모두 지원

    - MySQL 고유의 C API부터 시작해 JDBC, ODBC, .NET의 표준 드라이버 제공
    - 이러한 드라이버를 이용해 C/C++, PHP, 자바, 펄, 파이썬, 루비, .NET, 코볼까지 모든 언어로 MySQL 서버에서 쿼리를 사용할 . 수있게 지원함.

- MySQL 서버는 크게 `MySQL 엔진`과 `스토리지 엔진`로 구분


#### 4.1.1.1 MySQL 엔진
- MySQL 엔진

    - `커넥션 핸들러와 SQL 파서 및 전처리기`: 클라이언트로부터의 접속 및 쿼리 요청을 처리
    -  `옵티마이저`: 쿼리의 최적화된 실행
- MySQL은 표준 SQL(ANSI SQL) 문법을 지원하기 때문에 표준 문법에 따라 작성된 쿼리는 타 DBMS와 호환되어 실행될 수 있음.


#### 4.1.1.2 스토리지 엔진
- `MySQL 엔진`

    - 요청된 MySQL 문장을 분석, 최적화
    - DBMS 두뇌에해당하는 처리를 수행
    - MySQL 서버에서 MySQL 엔진은 1개

- `스토리지 엔진`

    - 실제 데이터를 디스크 스토리지에 저장
    - 디스크 스토리지로부터 데이터 읽어옴.
    - MySQL 서버에서에서 스토리지 엔진은 여러 개를 동시에 사용 가능
    - 테이블이 사용할 스토리지 엔진 지정 이후 해당 테이블의 모든 읽기 작업이나 변경 작업은 정의된 스토리지 엔진에서 처리
    ```
    mysql> CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=INNODB;
    ```
    - 각 스토리지 엔진은 성능 향상을 위해 키 캐시(MyISAM 스토리지 엔진)나 InnoDB 버퍼 풀(InnoDB 스토리지 엔진)과 같은 기능을 내장하고 있음.


#### 4.1.1.3 핸들러 API
- ` 핸들러(Handler) 요청` : MySQL 엔진의 쿼리 실행기에서 데이터 쓰거나 읽어야 할 때 각 스토리 엔진에 쓰기 또는 읽기 요청을 함.
- ` 핸들러 API` : ` 핸들러(Handler) 요청`에서 사용되는 API
- InnoDB 스토리지 엔진 또한 이 핸들러 API를 이용해 MySQL 엔진과 데이터를 주고 받음.
- `SHOW GLOBAL STATUS LIKE 'Handler%'` : 이 핸들러 API를 통해 얼마나 많은 데이터(레코드)작업이 있었는지 확인


### 4.1.2 MySQL 스레딩 구조

<img src="https://velog.velcdn.com/images/hyolim/post/551b2745-eef4-429d-a1d0-58fe2e88dedc/image.png" width="500" height="auto">

- MySQL 서버는 프로세스 기반이 아니라, '스레드 기반'으로 동작
- 크게 포그라운드 스레드와 백그라운드 스레드로 구분 가능
- MySQL 서버에서 실행 중인 스레드 목록은 `perfomance_schema` 데이터베이스의 `threads` 테이블을 통해 확인 가능 <br>

- 포그라운드 스레드 : 사용자 요청을 처리
- 백그라운드 스레드 : MySQL 서버의 설정 내용에 따라 개수가 가변적
- 동일한 이름의 스레드가 2개 이상씩 보이는 것은 MySQL 서버의 설정 내용에 따라 여러 스레드가 동일 작업을 병렬로 처리하는 경우

#### 4.1.2.1 포그라운드 스레드(클라이언트 스레드)
- 포그라운드 스레드는 최소한 MySQL 서버에 접속된 클라이언트의 수만큼 존재
- 주로 각 클라이언트 사용자가 요청하는 쿼리 문장을 처리
- 클라이언트 사용자가 작업을 마치고 커넥션을 종료하면 해당 커넥션을 담당하던 스레드는 다시 스레드 캐시(Thread cache)로 되돌아감.

    - 이때 이미 스레드 캐시에 일정 개수 이상의 대기 중인 스레드가 있으면 스레드 캐시에 넣지 않고 스레드를 종료시켜 일정 개수의 스레드만 스레드 캐시에 존재하게 함.
    - 이때 스레드 캐시에 유지할 수 있는 최대 스레드 개수는 `thread_chache_size` 시스템 변수로 설정

<br>
- 포그라운드 스레드는 데이터를 MySQL의 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우에는 직접 디스크의 데이터나 인덱스 파일로부터 데이터를 읽어와서 작업을 처리함.

- MyISAM 테이블

    - 디스크 쓰기 작업까지 포그라운드 스레드가 처리
    - 지연된 쓰기가 있지만 일반적인 방법은 아님
- InnoDB 테이블

    - 데이터 버퍼나 캐시까지만 포그라운드 스레드가 처리
    - 나머지 버퍼로부터 디스크까지 기록하는 작업은 백그라운드 스레드가 처리


#### 4.1.2.2 백그라운드 스레드
- InnoDB에서는 다음과 같은 작업이 백그라운드로 처리됨

    - 인서트 버퍼(Insert Buffer)를 병합하는 스레드
    - 로그를 디스크로 기록하는 스레드
    - InnoDB 버퍼 풀의 데이터를 디스크에 기록하는 스레드
    - 데이터를 버퍼로 읽어 오는 스레드
    - 잠금이나 데드락을 모니터링하는 스레드


- 가장 중요한 역할은 로그 스레드(Log thread)와 버퍼의 데이터를 디스크로 내려쓰는 작업을 처리하는 쓰기 스레드(Write thread)

    - MySQL 5.5 버전부터는 데이터 쓰기 스레드와 데이터 읽기 스레드 개수를 2개 이상 지정할 수 있게 됨.
    - `innodb_write_io_threads`, `innodb_read_io_threads` 시스템 변수로 스레드 개수 설정
    - 읽는 작업은 주로 클라이언트 스레드에서 처리되기 때문에 읽기 스레드는 많이 설정할 필요 없음.
    - 쓰기 스레드는 아주 많은 작업을 백그라운드로 처리하기 때문에 일반적인 내장 디스크를 사용할 때는 2~4, DAS나 SAN 같은 디스크를 사용할 때는 디스크를 최적으로 사용할 수 있을 만큼 충분히 설정하는 것이 좋음.

- 사용자의 요청을 처리하는 도중 데이터 쓰기 작업은 지연(버퍼링)되어 처리될 수 있지만, 데이터의 읽기 작업은 절대 지연될 수 없다.

    - 일반적인 상용 DBMS에는 대부분 쓰기 작업을 버퍼링해서 일괄처리하는 기능 탑재 되어있음 -> `InnoDB`도 이러한 방식
    - `MyISAM`은 사용자 스레드가 쓰기 작업가지 함께 처리하도록 설계됨.
    - 따라서 InnoDB에서는 쿼리로 데이터 변경되는 경우 데이터가 디스크의 데이터 파일로 완전히 저장될 때까지 기다리지 않아도 되지만, MyISAM에서 일반적인 쿼리는 쓰기 버퍼링 기능 사용 불가

### 4.1.3 메모리 할당 및 사용 구조
<img src="https://velog.velcdn.com/images/hyolim/post/16a63d21-869f-4062-9587-d757f94e3270/image.png" width="500" height="auto">

- MySQL에서 사용되는 메모리 공간

    - `글로벌 메모리 영역`

        - MySQL 서버가시작되면서 운영체제로부터 할당됨
        - 요청된 메모리 공간을 100% 할당해줄 수도, 그 공간만큼 예약해두고 필요할 때 조금씩 할당해주는 경우도 있음.
        - 단순히 시스템 변수로 설정해 둔 운영체제로부터 메모리를 할당받는다고 생각해도 됨.
    - `로컬 메모리 영역`

    - 글로벌 메모리 영역과 로컬 메모리 영역은 MySQL 서버 내에 존재하는 많은 스레드가 공유해서 사용하는 공간인지 여부에 따라 구분됨.

#### 4.1.3.1 글로벌 메모리 영역
- 일반적으로 클라이언트 스레드의 수와 무관하게 하나의 메모리 공간만 할당됨.
- 필요에 따라 2개 이상의 메모리 공간을 할당받을 수도 있지만 클라이언트의 스레드 수와는 무관하며, 생성된 글로벌 영역이 N개라고 하더라도 모든 스레드에 의해 공유됨
  <br>
- 대표적인 글로벌 메모리 영역

    - 테이블 캐시
    - InnoDB 버퍼풀
    - InnoDB 어댑티브 해시 인덱스
    - InnoDB 리두 로그 버퍼

#### 4.1.3.2 로컬 메모리 영역
- 세션 메모리 영역이라고도 함.
- MySQL 서버 상에 존재하는 클라이언트 스레드가 쿼리를 처리하는 데 사용하는 메모리 영역
- 대표적으로 커넥션 버퍼, 정렬(소트)버퍼가 있음.
- 클라이언트가 MySQL 서버에 접속하면 MySQL 서버에서 클라이언트 커넥션으로부터의 요청을 처리하기 위해 스레드를 하나씩 할당하게 되는데, 클라이언트 스레드가 사용하는 메모리 공간이라고 해서 `클라이언트 메모리 영역`이라고 하기도 함.
- 클라이언트와 MySQL 서버와의 커넥션을 세션이라고 하기 때문에 `세션 메모리 영역`이라고도 표현함.

<br>

- 로컬 메모리는 각 클라이언트 스레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않음.
- 일반적으로 글로벌 영역의 크기는 주의해서 설정하지만 소트 버퍼와 같은 로컬 메모리 영역은 크게 신경 쓰지 않고 설정하는데, 최악의 경우에 MySQL 서버가 메모리 부족으로 멈춰버릴 수도 있으므로 적절한 메모리 공간 설정 중요함.
- 로컬 메모리 공간은 각 쿼리의 용도별로 필요할 때만 공간이 할당되고 필요하지 않은 경우에는 MySQL이 메모리 공간을 할당조차도 안 할 수도 있음. (ex. 소트 버퍼, 조인 버퍼)
- 로컬 메모리 공간은 커넥션이 열려 있는 동안 계속 할당된 상태로 남아있는 공간(커넥션 버퍼나 결과 버퍼)도 있고, 그렇지 않고 쿼리를 실행하는 순간에만 할당했다가 다시 해제하는 공간(소트 버퍼, 조인 버퍼)도 있음.

<br>

- 대표적인 로컬 메모리 영역

    - 정렬 버퍼(Sort Buffer)
    - 조인 버퍼
    - 바이너리 로그 캐시
    - 네트워크 버퍼

### 4.1.4 플러그인 스토리지 엔진 모델

<img src="https://velog.velcdn.com/images/hyolim/post/ccfce870-a6bd-4500-818b-c4d02bdcf8bf/image.png" width="500" height="auto">

- MySQL의 독특한 구조 중 대표적인 것이 `플러그인 모델`
- 플러그인해서 사용할 수 있는 것이 스토리지 엔진만 있는 것은 아님

    - 전문 검색 엔진을 위한 검색어 파서(인덱싱할 키워드를 분리해내는 작업)
    - 사용자 인증을 위한 Native Authentication
    - Caching SHA-2 Authentication
- MySQL은 이미 기본적으로 많은 스토리 엔진 가지지만, 다른 전문 개발 회사 또는 사용자가 직접 스토리지 엔진을 개발하는 것 가능

<img src="https://velog.velcdn.com/images/hyolim/post/d43bfa60-31c5-40e2-81b2-ef1694979c8b/image.png" width="500" height="auto">

- 위 그림과 같이 나눈다면 대부분 작업이 MySQL 엔진에서 처리되고, 마지막 '데이터 읽기/쓰기'만 스토리지 엔진에서 처리됨. (사용자가 새로운 용도의 스토리지 엔진을 만든다 하더라도 DBMS 전체 기능이 아닌 일부 기능만 수행하는 엔진을 작성하게 된다는 의미)

- MySQL 서버에서 MySQL 엔진은 스토리지 엔진을 조정하기 위해 핸들러를 사용하게 됨.

    - MySQL 엔진이 . 각스토리지 엔진에게 데이터를 읽어오거나 저장하도록 명령하려면 반드시 핸들러를 통해야 함.
    - MySQL 서버의 상태 변수 가은데 `'Handler_'`로 시작하는 것은 MySQL 엔진이 각 스토리지 엔진에게 보낸 명령의 횟수를 의미하는 변수

- MySQL에서 MyISAM이나 InnoDB와 같이 다른 스토리지 엔진을 사용하는 테이블에 대해 쿼리를 실행하더라도 MySQL의 처리 내용은 동일하며, 데이터 읽기/쓰기 영역의 차이만 있음.
- 실질적인 `GROUP BY`나 `ORDER BY`등 복잡한 처리는 스토리지 엔진 영역이 아니라 MySQL 엔진의 처리 영역인 '쿼리 실행기'에서 처리됨.

<br>

- 플러그인 형태로 빌드된 스토리지 엔진 라이브러리를 다운로드해서 끼워넣기만 하면, MySQL 서버를 다시 빌드(컴파일)하지 않고 사용 가능
- 플러그인 형태의 스토리지 엔진은 손쉽게 업그레이드 가능
- `SHOW PLUGINS` : 스토리지 엔진 뿐 아니라 인증 및 전문 검색용 파서와 같은 플러그인도 확인 가능
- MySQL 서버에서는 다양한 기능을 플러그인 형태로 지원

    - 인증이나 전문 검색 파서
    - 쿼리 재작성
    - 비밀번호 검증과 커넥션 제어
    - MySQL 서버의 기능을 커스텀하게 확장할 수 있게 플러그인 API 매뉴얼이 공개되어 있음.


### 4.1.5 컴포넌트
- MySQL 8.0부터는 기존의 플러그인 아키텍처를 대체하기 위해 컴포넌트 아키텍처가 지원됨.

- MySQL 서버의 플러그인 단점

    - 플러그인은 오직 MySQL 서버와 인터페이스 할 수 있고, 플러그인끼리는 통신할 수 없음
    - 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않음(캡슐화 안 됨)
    - 플러그인은 상호 의존 관계를 설정할 수 없어서 초기화가 어려움

 <br>
 - MySQL 5.7 버전까지는 비밀번호 검증 기능이 플러그인 형태로 제공됐지만, MySQL 8.0부터는 컴포넌트로 개선됨.


### 4.1.6 쿼리 실행 구조
<img
src="https://velog.velcdn.com/images/hyolim/post/b1e9121c-301f-491e-b8c2-170374f20639/image.png" width="500" height="auto">

#### 4.1.6.1 쿼리 파서
- 사용자 요청으로 들어온 쿼리 문장을 토큰으로 분리해 트리 형태의 구조로 만들어 내는 작업
- 쿼리 문장의 기본 문법 오류는 이 과정에서 발견되고, 사용자에게 오류 메시지를 전달하게 됨.

#### 4.1.6.2 전처리기
- 파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인
- 각 토큰을 테이블 이름이나 칼럼 이름, 또는 내장 함수와 같은 개체를 매핑해 해당 객체의 존재 여부와 객체의 접근 권한 등을 확인
- 실제 존재하지 않거나 권한 상 사용할 수 없는 개체의 토큰이 걸러짐.

#### 4.1.6.3 옵티마이저
- 사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할 지를 결정
- DBMS의 두뇌라고 할 수 있음

#### 4.1.6.4 실행 엔진
- 만들어진 계획대로 각 핸들러에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할 수행
  <br>

- 옵티마이저가 GROP BY를 처리하기 위해 임시 테이블 사용하기로 결정한 예

    1. 실행엔진이 핸들러에게 임시 테이블을 만들라고 요청
    2. 다시 실행 엔진은 WHERE 절에 일치하는 레코드를 읽어오라고 핸들러에게 요청
    3. 읽어 온 레코드들을 1번에서 준비한 임시 테이블로 저장하라고 다시 핸들러에게 요청
    4. 데이터가 준비된 임시 테이블에서 필요한 방식으로 데이터를 읽어 오라고 핸들러에게 다시 요청
    5. 최종적으로 실행 엔진은 결과를 사용자나 다른 모듈로 넘김

#### 4.1.6.5 핸들러(스토리지 엔진)
- MySQL 서버의 가장 밑단에서 MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 디스크로부터 읽어 오는 역할을 담당
- 스토리지 엔진을 의미함.

    - MyISAM 테이블을 조작하는 경우에는 핸들러가 MyISAM 스토리지 엔진이 됨.
    - InnoDB 테이블을 조작하는 경우에는 핸들러가 InnoDB 스토리지 엔진이 됨.

### 4.1.7 복제
- MySQL 서버에서 매우 중요한 역할을 담당

### 4.1.8 쿼리 캐시
- 빠른 응답을 필요로 하는 웹 기반의 응용 프로그램에서 매우 중요한 역할을 담당함.
- 쿼리 캐시는 SQL 실행 결과를 메모리에 캐시하고, 동일 SQL 쿼리가 실행되면 테이블을 읽지 않고 즉시 결과를 반환하기 때문에 매우 빠른 성능
- 테이블 데이터가 변경되면 캐시에 저장된 결과 중에서 변경된 테이블과 관련된 것들은 모두 삭제(Invalidate)해야 하여 심각한 동시 처리 성능 저하 유발
- MySQL 서버가 발전되면서 성능이 개선되는 과정에서 쿼리 캐시는 계속된 동시 처리 성능 저하와 많은 버그의 원인이 되기도 함.
- MySQL 8.0으로 올라오면서 쿼리 캐시가 MySQL 서버 기능에서 완저닣 제거되고, 관련 시스템 변수도 모두 제거됨.

### 4.1.9 스레드 풀
- MySQL 서버 엔터프라이즈 에디션 : 스레드 풀(Thread Pool) 제공

    - MySQL 서버 프로그램에 내장되어 있음.

- MySQL 커뮤니티 에디션 : 스레드 풀 제공 X

    - 스레드 풀을 사용하고 싶다면 동일 버전의 Percona Server에서 스레드 풀 플러그인 라이브러리(thread_pool.so 파일)를 MySQL 커뮤니티 에디션 서버에 설치(INSTALL PLUGIN 명령)해서 사용하면 됨.

- Percona Server의 스레드 풀

    - 플러그인 형태로 작동하게 구현되어 있음.

- 스레드 풀은 내부적으로 사용자의 요청을 처리하는 스레드 개수를 줄여서 동시 처리되는 요청이 많더라도 MySQL 서버의 CPU가 제한된 개수의 스레드 처리에만 집중할 수 있게 해서 서버의 자원 소모를 줄이는 것이 목적

- 스케줄링 과정에서 CPU 시간을 제대로 확보하지 못하는 경우에는 쿼리 처리가 더 느려질 수도 있음.

    - 제한된 수의 스레드만으로 CPU가 처리하도록 적절히 유도한다면 CPU 프로세서 친화도(Processor affinity)도 높이고 운영체제 입장에서는 불필요한 컨텍스트 스위치(Context switch)를 줄여서 오버헤드 낮출 수 있음.

- Percona Server의 스레드 풀은 기본적으로 CPU 코어 수만큼 스레드 그룹을 생성하는데, 스레드 그룹의 개수는 thread_pool_size 시스템 변수를 변경해서 조정 가능

    - 일반적으로는 CPU 코어 개수와 맞추는 것이 CPU 프로세서 친화도 높이는 것에 좋음
    - MySQL 서버가 처리해야 할 요청이 생기면 스레드 풀로 처리를 이관하는데, 이미 스레드 풀이 처리 중인 작업이 있는 경우에는 thread_pool_oversubscribe 시스템에 설정된 개수만큼 추가로 더 받아들여져 처리하는 데, 이 값이 너무 크면 스케줄링해야 할 스레드가 많아져 비효율적

<br>  

- 스레드 그룹의 모든 스레드가 일을 처리하고 있다면, 스레드 풀은 해당 스레드 그룹에 새로운 작업 스레드(Worker thread)를 추가할지, 아니면 기존 작업 스레드가 처리를 완료할 때까지 기다릴지 여부 판단 필요

    - 스레드 풀의 타이머 스레드 : 주기적으로 스레드 그룹의 상태를 체크해서 `thread_pool_stall_limit` 시스템 변수에 정의된 밀리초만큼 작업 스레드가 지금 처리 중인 작업을 끝내지 못하면 새로운 스레드를 생성해서 스레드 그룹에 추가
    - 이때 전체 스레드 풀에 있는 스레드 개수는 `thread_pool_max_threads` 시스템 변수에 설정된 개수 넘어설 수 없음.
      -> 즉, 모든 스레드 그룹의 스레드가 각자 작업을 처리하고 있는 상태에서 새로운 쿼리 요청이 들어오더라도 스레드 풀은 `thread_pool_stall_limit` 시간 동안 기다려야만 새로 들어온 요청을 처리할 수 있다는 뜻
    - `thread_pool_stall_limit`을 0에 가까운 값으로 설정해야 한다면 스레드 풀을 사용하지 않는 편이 나음.

- Percona Server의 스레드 풀 플러그인은 선순위 큐와 후순위 큐를 이용해 특정 트랜잭션이나 쿼리를 우선적으로 처리할 수 있는 기능 제공

    - 먼저 시작된 트랜잭션 내에 속한 SQL을 빨리 처리해주면 해당 트랜잭션이 가지고 있던 잠금이 빨리 해제되고, 잠금 경합을 낮춰서 전체적인 처리 성능을 향상 가능

    <img src="https://velog.velcdn.com/images/hyolim/post/1975e399-e640-4823-95b8-ad2dabc942df/image.png" width="500" height="auto">


### 4.1.10 트랜잭션 지원 메타데이터
- 데이터베이스 서버에서 테이블의 구조 정보와 스토어드 프로그램 등의 정보를`데이터 딕셔너리` 또는`메타데이터`라고 함.

----
## 4.2 InnoDB 스토리지 엔진 아키텍처

<img src="https://velog.velcdn.com/images/hyolim/post/7380e002-a044-4f5e-80fc-91f7f901b420/image.png" width="500" height="auto">

- InnoDB 스토리지 엔진

    - InnoDB는 MySQL에서 사용할 수 있는 스토리지 엔진 중 거의 유일하게 레코드 기반 잠금 제공
    - 동시성 처리가 가능하고 안정적이고 성능 뛰어남.

### 4.2.1 프라이머리 키에 의한 클러스터링
- InnoDB의 모든 테이블은 기본적으로 프라이머리 키를 기준으로 클러스터링되어 저장됨.

    -  프라이머리 키 값으 순서대로 디스크에 저장됨
    - 세컨더리 인덱스는 레코드의 주소 대신 프라이머리 키의 값을 논리적인 주소로 사용
    - 프라이머리 키를 이용한 레인지 스캔 빨리 처리 가능
    - 프라이머리 키는 기본적으로 다른 보조 인덱스에 비해 비중이 높게 설정됨.
    - 오라클 DBMS의 IOT(Index organized table)와 동일한 구조가  InnoDB에서는 기본적인 일반적 테이블 구조

<br>

- MyISAM 스토리지 엔진에서는 클러스터링 키를 지원하지 않음.

    - 프라이머리 키와 세컨더리 인덱스는 구조적으로 차이가 없음.(프라이머리 키는 유니크 제약을 가진 세컨더리 인덱스일 뿐)
    - 프라이머리 키를 포함한 모든 인덱스는 물리적인 레코드의 주소 값(ROWID)을 가짐.

### 4.2.2 외래키 지원
- 외래 키에 대한 지원은 InoDB 스토리지 엔진 레벨에서 지원하는 기능으로 MyISAM이나 MEMORY 테이블에서는 사용할 수 없음.

    - 외래 키는 데이터베이스 서버 운영 불편함 때문에 서비스용 데이터베이스에서는 생성하지 않는 경우도 자주 있음.
    - 개발 환경의 데이터베이스에서는 좋은 가이드 역할

- InnoDB에서 외래 키는 부모 테이블과 자식 테이블 모두 해당 칼럼에 인덱스 생성 필요

    - 변경 시에는 반드시 부모 테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업 필요하므로 잠금이 여러 테이블로 전파됨. -> 데드락 발생할 때 많음

- 수동으로 데이터를 적재하거나 스키마 변경 등의 관리 작업 실패할 수 있음.

    - 부모 테이블과 자식 테이블의 관계를 명확히 파악해서 순서대로 작업한다면 문제 없이 실행 가능하지만, 외래키가 복잡하게 얽히면 간단하지 않음
    - `foreign_key_checks` 시스템 변수를 `OFF`로 설정하면 외래 키 관계에 대한 체크 작업을 일시적으로 멈출 수 있음.
    - 외래 키 체크를 일시적으로 멈추면 레코드 적재나 삭제 등의 작업도 부가적 체크가 필요 없기 때문에 훨씬 빠르게 체크 가능
    - 외래 키 체크를 해제해도 부모와 자식 테이블 간의 관계가 깨진 상태로 유지해도 되는 것은 아님
    - `foregin_key_checks` 시스템 변수는 적용 범위를 GLOBAL과 SESSION 모두로 설정 가능한 변수 -> 명시하지 않으면 현재 세션만 적용됨.

    ```
    mysql> SET foreign_key_checks=OFF;
    
    // 작업 실행
    mysql> SET foreign_key_checks=ON;
    
    ```


### 4.2.3 MVCC(Multi Version Concurrency Control)
- 일반적으로 레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능
- MVCC의 가장 큰 목적은 잠금을 사용하지 않는 일관된 읽기를 제공하는 데 있음.

    - InnoDB는 언두 로그(Undo log)를 이용해 이 기능 구현
    - 멀티 버전은 하나의 레코드에 대해 여러 개의 버전이 동시에 관리됨.


### 4.2.4 잠금 없는 일관된 읽기(Non-Locking Consisted Read)
- InnoDB 스토리지 엔진은 MVCC 기술을 이용해 기술을 이용해 잠금을 걸지 않고 읽기 작업 수행

    - 잠금을 걸지 않기 때문에 InnoDB에서 읽기 작업은 다른 트랜잭션이 가지고 있는 잠금을 기다리지 않고, 읽기 작업 가능
    - 격리 수준이 `SERIALIZABE`이 아닌 `READ_UNCOMMITTED`나 `READ_COMITTED`, `REPEATABLE` 수준의 경우 INSERT와 과녈되지 않은 순수한 읽기 작업은 항상 잠금을 대기하지 않고 바로 실행됨. -> '잠금 없는 일관된 읽기'
    - InnoDB에서는 변경되기 전의 데이터를 읽기 위해 언두 로그 사용
    - 오랜 시간 동안 할성 상태인 트랜잭션으로 인해 MySQL 서버가 느려지거나 문제가 발생할 때가 가끔 있는데, 일관된 읽기를 위해 언두 로그를 삭제하지 못하고 예속 유지해야 하기 때문
    - 트랜잭션이 시작됐다면, 가능한 빨리 롤백이나 커밋을 통해 트랜잭션을 완료하는 것이 좋음.

### 4.2.5 자동 데드락 감지
- InnoDB 스토리지 엔진은 내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프(Wait-for-List) 형태로 관리
    - InnoDB 스토리지 엔진은 데드락 감지 스레드를 가지고 있어 데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션을 찾아 그 중 하나를 강제 종료
    - 어느 트랜잭션을 강제 종료할 지의 기준은 트랜잭션의 언두 로그 양 -> 언두 로그 레코드를 더 적게 가진 트랜잭션이 롤백의 대상이 됨
      -> 트랜잭션 강제 롤백으로 인한 MySQL 서버의 부하도 덜 유발하기 때문

- InnoDB 스토리지 엔진은 상위 레이어인 MySQL 엔진에서 관리되는 테이블 잠금은 볼 수 없어 데드락 감지가 불확실할 수 있음.

    - `innodb_table_locks` 시스템 변수를 활성화하면 InnoDB 스토리지 엔진 내부의 레코드 잠금뿐 아니라 테이블 레벨의 잠금까지 감지 가능

- 동시 처리 스레드가 매우 많아지거나 각 트랜잭션이 가진 잠금의 개수가 많아지면 데드락 감지 스레드가 느려짐.

    - 데드락 감지 스레드는 잠금 목록을 검사해야 하기 때문에 잠금 상태가 변경되지 않도록 잠금 목록이 저장된 리스트에 새로운 잠금을 걸고 데드락 스레드를 찾게됨.
    - 데드락 감지 스레드가 느려지면 서비스 쿼리 처리 중인 스레드는 작업 진행하지 못하고 대기하면서 서비스에 악영향 -> CPU 자원 소모량 증가할 수 있음.

- `innodb_deadlock_detect` 시스템 변수를 OFF로 설정하면 데드락 감지 스레드 작동하지 않게 됨.

    - 데드락 감지 스레드가 작동하지 않으면 InnoDB 스토리지 엔진 내부에서 2개 이상의 트랜잭션이 상대방이 가진 잠금을 요구하는 상황이 발생해도 중재하지 않기 때문에 무한정 대기하게 됨.
    - `innodb_lock_wait_timeout` 시스템 변수를 활성화하면 일정 시간이 지나면 자동으로 요청이 실패하게 되고 에러 메시지를 반환 (50s 이하를 추천)


### 4.2.6 자동화된 장애 복구
- InnoDB에는 손실이나 장애로부터 데이터 보호하기 위한 여러 메커니즘 탑재되어 있음.

    - 이러한 매커니즘을 이용해 MySQL 서버가 시작될 때 완료되지 못한 트랜잭션이나 디스크에 일부만 기록된(Partial write) 데이터 페이지 등에 대한 일련의 복구 작업이 자동으로 진행됨.

- InnoDB 스토리지 엔진은 매우 견고해 데이터 파일이 손상되거나 MySQL 서버가 시작되지 못하는 경우는 거의 없음.

    - MySQL 서버와 무관하게 디스크나 서버 하드웨어 이슈로 InnoDB 스토리지 엔진이 자동으로 복구 못하는 경우는 복구하기 어려움.
    - InnoDB 데이터 파일은 기본적으로 MySQL 서버가 시작될 때 항상 자동 복구를 수행하는 데, 이 때 자동 복구가 불가하면 MySQL 서버가 종료됨.
    - 자동 복구가 불가해 MySQL 서버가 종료된 경우에는 `innodb_force_recovery` 시스템 변수를 설정해 MySQL 서버를 시작해야 함. -> InnoDB 스토리지 엔진이 데이터 파일이나 로그 파일 손상 여부 검사 과정을 선별적으로 진행하게 함.

- MySQL 서버가 기동되고 InnoDB 테이블이 인식된다면 `mysqldump`를 이용해 데이터를 가능한 만큼 백업하고, 그 데이터로 MySQL 서버의 DB와 테이블을 다시 생성하는 것이 좋음.

    - `innodb_force_recovery` 옵션은 1부터 6까지 설정 가능
    - 0이 아닌 복구 모드에서는 SELECT 이외는 불가

- MySQL 서버가 시작되지 않으면 백업을 이용해 다시 구축해야 함.

    - 백업이 있다면 마지막 백업으로 데이터베이스를 새로 구축하고, 바이너리 로그를 사용해 최대한 장애 시점까지 데이터 복구 가능