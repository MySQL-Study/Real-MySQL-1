# Chapter 1

# 소개

- 엔터프라이즈 버전, 커뮤니티 버전이 있음
  커뮤니티 버전이 공개된 버전임
- 5.7 버전 까지는 안정성에 집중, 8.0부터는 상용 DBMS의 기능들을 장착

# 왜 MySQL인가?

- 가격과 비용
  오라클은 방대한 데이터를 저장하기에 너무 비싸다.
- 자신이 가장 잘 활용할 수 있는 걸 골라라
  그럼에도 기준을 제공한다면 다음과 같은 순서이다.
  - 안정성
  - 성능과 기능
  - 커뮤니티나 인지도

<br>
<br>
<br>

# Chapter 2

# 설치

다양한 형태로 설치가 가능하지만 운영체제 별 인스톨러를 활용하기를 권장한다.

버전을 선택할 때도 갓 출시된 버전은 버그가 있을 수 있기에 15~20번 릴리즈된 버전을 추천한다.

- mac: homebrew 로 간단설치
- linux(ubuntu) :
  - `apt-update`
  - `sudo apt-get install mysql-server`

### 시작과 종료

```tsx
//리눅스 환경
//상태 확인
systemctl status mysqld
//시작
systemctl start mysqld
//종료
systemctl stop mysqld

//원격 셧다운
mysql> SHUTDOWN;
```

원격 셧다운을 하기 위해서는 권한을 가지고 있어야 한다.

### 셧다운 시 주의할 사항

실제 트랜잭션이 정상적으로 커밋되어도 데이터 파일에 변경된 내용이 기록되지 않고 로그(리두 로그) 파일에만 기록되어 있을 수 있다. 심지어 서버가 종료된 후 다시 시작해도 계속 이상태로 남아있을 수 있다.

커밋된 내용을 모두 반영하고 종료하는 방법도 있는데 mysql 서버 옵션을 변경하고 종료하면 된다.

이렇게 모든 변경사항을 데이터 파일에 적용하고 종료하는 것을 클린 셧다운(clean shutdown) 이라고 한다.
클린 셧다운 이후 다시 시작하면 트랜잭션 복구 과정을 진행하지 않기에 빠르게 다시 시작할 수 있다.

```tsx
mysql> SET_GLOBAL innodb_fast_shutdown=0;
linux> systemctl stop mysqld
```

<aside>
❗

mysql 서버가 시작하고 종료될 때는 버퍼 풀의 내용을 백업하고 복구하는 과정이 내부적으로 실행된다. 다만 버퍼풀의 내용이 아니라 메타데이터를 백업하기에 용량은 작고 빠르게 백업된다.
하지만, 새로 시작될 때는 디스크에서 데이터 파일들을 모두 읽어서 적재해야 하므로 상당한 시간이 걸릴 수 있다.

</aside>

## 서버 연결 테스트

```tsx
// 1. 소켓 파일 이용
mysql -uroot -p --host=localhost --socket=/tmp/mysql.lock
// 2. tcp/ip로 로컬호스트 접속
mysql -uroot -p --host=127.0.0.1 --port=3306
// 3. 로컬 접속
mysql -uroot -p
```

1. mysql의 소켓 파일을 이용하여 접속한다.
2. tcp/ip 를 통해 로컬 호스트로 접속하는 경우 포트를 명시하는게 일반적이다.
   원격 호스트로 접속할때는 반드시 이 방법을 이용해야 한다.
   localhost 와 127.0.0.1 은 각각 의미가 다르다.
   - `localhost`
     이 옵션은 항상 소켓 파일을 통해 mysql 서버에 접속하게 된다. 이는 unix domain socket 을 이용하는 방식으로 tcp/ip 방식이 아니라 유닉스 프로세스간 통신(inter process communication)의 일종이다.
   - `127.0.0.1`
     이 방식은 자기 서버를 가리키는 루프백(loop back) ip 이기는 하지만 tcp/ip 방식을 이용한다.
3. 별도로 호스트와 ip를 입력하지 않는다.
   따라서 기본값인 localhost와 소켓 파일을 사용하게 된다.

- telnet 테스트
  글자가 깨졌지만 접속에 성공
  ![스크린샷 2024-12-19 오후 5.42.45.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/f8f04773-2f24-482c-8db8-1d2d03c099a1/b9e2033b-087e-4c23-b63e-2a3f8b5c2a74/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-12-19_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_5.42.45.png)

<br>

## MySQL 서버 업그레이드

업그레이드 하는 방법은 다음과 같이 2가지 방법이 있다.

1. MySQL 서버의 데이터 파일을 그대로 두고 업그레이드 하는 방법
2. mysqldump 도구 등을 이용해 MySQL 서버의 데이터를 SQL문장이나 텍스트 파일로 덤프한 후, 새로운 MySQL 서버에 덤프된 데이터를 적재하는 방법

1번 방법은 `인플레이스 업그레이드(in-place upgrade)` 라고 하며 2번 방법은 `논리적 업그레이드(logical upgrade)` 라고 한다.

인플레이스 업그레이드는 여러가지 제약사항이 있지만, 시간을 크게 단축할 수 있다.
반대로 논리적 업그레이드는 제약사항이 거의 없지만, 시간이 매우 많이 소요될 수 있다.

<br>

### 인플레이스 업그레이드 제약사항

업그레이드는 `마이너(패치) 버전간 업그레이드`와 `메이저 버전 업그레이드`로 생각할수있다.

동일 메이저 버전끼리의 업그레이드는 대부분 데이터 파일의 변경 없이 진행된다. 예를들어 8.0.16 에서 8.0.21로 업그레이드 시 mysql 서버 프로그램만 재설치하면 된다.

하지만 메이저 버전 간 업그레이드는 대부분 크고 작은 데이터 파일의 변경이 필요하기에 반드시 직전 버전에서만 업그레이드가 가능하다. 예를들어 5.5 → 5.6은 가능하지만, 5.5 → 5.7로는 불가능하다.

따라서 `여러 버전을 건너뛴다면 논리적 업그레이드가 나을 수 있다.`

GA(general availability) 버전인 경우만 메이저 업그레이드가 지원된다. GA는 어느정도 안정성이 확인된 버전을 의미한다.

<br>

### MySQL 8.0 업그레이드시 고려사항

5.7 버전에서 8.0 버전으로 업그레이드 되면서 많은 변화가 일어났다.
8.0으로 업그레이드 하기 전 다음과 같은 사항을 검토해보자.

- 사용자 인증 방식 변경
  5.7 버전에서는 Native 인증 방식을 사용했지만 8.0부터는 Caching SHA-2 방식이 기본 인증 방식으로 변경되었다.
- MySQL 8.0과의 호환성 체크
  5.7버전에서 손상된 FRM 파일이나 호환되지 않는 데이터 타입 또는 함수가 있는지 mysqlcheck 유틸리티를 이용해 확인해볼 것을 권장한다.
- 외래키 이름 길이
  8.0에서는 64글자로 제한된다
- 인덱스 힌트
  8.0에서는 인덱스 힌트가 오히려 성능을 저하할 수 있다. 성능측정을 해보고 비교하자
- GROUP BY 정렬 옵션
  GROUP BY 절의 칼럼 뒤에 ASC 나 DESC 를 하고 있다면 제거하거나 다른 방식으로 변경하자
  (GROUP BY field_name [ASC | DESC])
- 파티션을 위한 공용 테이블 스페이스
  8.0이상에서는 파티션의 각 테이블 스페이스를 공용 테이블 스페이스에 저장할 수 없다.

### 서버 설정

mysql 서버는 단 하나의 설정파일을 사용하는데 유닉스 계열 에서는 `my.cnf`, 윈도우 계열에서는 `my.ini` 라는 이름을 사용한다. mysql은 서버가 시작될 때만 이 설정파일을 참고한다. 디렉터리를 순차적으로 탐색하며 처음 발견된 my.cnf 을 사용한다.

다만 단 하나의 설정파일만 사용하지만 설정파일이 위치한 디렉터리는 여러개일 수 있다는 점이다.
다음과 같은 명령어로 어떤 my.cnf 파일을 읽는지 파악할 수 있다.

```tsx
mysql --help
```

![스크린샷 2024-12-20 오후 4.59.39.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/f8f04773-2f24-482c-8db8-1d2d03c099a1/dafccccb-3a41-47d7-8c79-bd418eee4ddb/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-12-20_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_4.59.39.png)

`my.cnf` 파일에 여러개의 설정 그룹을 담을 수 있다.

mysqldump 그룹은 mysqldump 설정을, mysqld 그룹은 mysqld 설정 영역을 사용하면 된다.

```bash
[client]
# client 옵션
default-character-set = utf8
port = 3306
socket = /home/centos/mysql/mysql.sock

[mysqld]
# MySQL 서버 기본 옵션
basedir = /home/centos/mysql
datadir = /home/centos/mysql/data
socket = /home/centos/mysql/mysql.sock
log-error = /home/centos/mysql/mysql-log/mysqldb.log
pid-file = /home/centos/mysql/mysql-log/mysqldb.pid
user = centos
innodb_buffer_pool_size = 10G

[mysql]
# MySQL 설정그룹
soket = /home/centos/mysql/mysql.sock
port = 3304

[mysqld_safe]
# mysqld_safe는 mysqld를 감시하는 데몬. mysqld_safe가 내부에서 mysqld를 실행
log-error = /home/centos/mysql/mysql-log/mysqldb.log
pid-file = /home/centos/mysql/mysql-log/mysqldb.pid
```

<br>
<br>

### MySQL 시스템 변수

MySQL 서버는 기동하면서 설정 파일의 내용을 읽어 메모리나 작동 방식을 초기화 하고, 접속된 사용자를 제어하기 위해 이러한 값을 별도로 저장한다. 이 값들을 `시스템 변수`라고 한다.
아래와 같은 명령으로 확인할 수 있다.

```bash
SHOW GLOBAL VARIABLES;
```

시스템 변수값이 어떻게 mysql 서버와 클라이언트에 영향을 미치는지 확인하려면 각 변수가 `글로벌 변수`인지 `세션 변수` 인지 구분해야한다.

- 글로벌 변수
  하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수
  주로 MySQL 서버 자체 관련 설정이 많다. 예를들어 innoDB 버퍼 풀의 크기, MyISAM의 키 캐시크기 등이 대표적이다.
- 세션 변수
  클라이언트가 MySQL 서버에 접속할 때 기본적으로 부여하는 옵션의 기본값을 제어하는데 사용한다.
  세션 변수는 커넥션 별로 설정값을 다르게 지정할 수 있으며, 한번 연결된 커넥션의 변수는 서버에서 변경할 수 없다. auto commit 을 대표적으로 볼 수 있다.

<br>

### 정적 변수와 동적 변수

MySQL 서버가 기동 중인 상태에서 변경 가능한지에 따라 동적 변수와 정적 변수로 구분된다.

디스크에 저장된 변수(my.cnf파일)와 기동 중인 MySQL 서버의 메모리에 있는 MySQL 서버의 시스템 변수를 변경하는 경우로 구분할 수 있다.

동적 변수를 set 명령을 통해 변경한 경우 인스턴스에만 적용되기 때문에 다음 실행에서는 적용되어 있지 않다.
영구적으로 변경하기 위해서는 `my.cnf` 파일에서 변경해야 한다.

또는 `SET PERSIST` 를 이용해서 동적 시스템 변수를 설정파일까지 변경할 수 있다. 그러나 `my.cnf` 파일에 적용되지 않고 별도의 파일에 기록된다.

<br>

### SET PERSIST

서버를 재시작하지 않고, `SET` 명령을 이용하여 동적 변수를 변경하여 문제를 해결한 뒤 이를 잊고 서버를 재시작 하는 경우 똑같은 문제가 발생한다. 이를 보완하기 위해 `SET PERSIST` 명령이 추가되었다.

`SET PERSIST` 명령으로 시스템 변수를 변경하면 즉시 적용함과 동시에 별도의 설정파일 `mysqld-auto.cnf` 에 변경 내용을 추가로 기록한다. 이를 통해 서버를 재시작할 때, my.cnf 와 mysqld-auto.cnf 를 함께 참조하여 서버를 재실행 한다.

`SET PERSIST_ONLY` 명령은 인스턴스에 변경을 적용하지 않고 `mysqld-auto.cnf` 파일만 변경할 수 있다.
또한, 정적 변수를 영구적으로 변경할 때도 이용할 수 있다. 이렇게 동적 변수뿐만 아니라 정적 변수도 변경할 수있다. (누가 언제 어떤 값이 변경 되었는지도 기록된다 json 형태로)
