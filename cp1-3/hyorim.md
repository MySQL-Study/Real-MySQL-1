# Chapter 1. 소개
## 1.1 MYSQL 소개
### 오픈소스 데이터베이스 MySQL
- 1979년 스웨덴 TcX 회사의 터미널 인터페이스 UNIREG로부터 시작
    - 1994년 웹 시스템의 데이터베이스로 사용하기 시작하면서, MySQL 버전 1.0 이 완성됨. -> 사내에서만 사용됨.
    - 1996년 일반인에게 공개됨.
      <br>
- 2000년 MySQL 개발한 중심인물(몬티 & 데이빗)이 MySQL AB라는 회사로 독립 -> FPL(Free Public Licence) 라이선스 정책으로 바뀜
  <br>
- 2006년 현재와 같은 두 가지 라이선스 정책 취하게 됨
    - 썬마이크로시스템즈에 인수되고, 다시 오라클로 인수됨 -> 특별한 라이선스 정책의 변화는 없음

### 현재 MySQL 라이선스 정책
`MySQL 엔터프라이즈 에디션`

`MySQL 커뮤니티 에디션` :  별도의 라이선스 계약 없이 일반 사용자가 내려받아 사용하는 버전
 
--------------------------

- MySQL은 100% 무료가 아님
- 하지만, 엔터프라이즈 에디션과 커뮤니티 에디션의 소스코드는 동일!!
- MySQL 5.5 이전 버전까지의 차이는 얼마나 자주 패치 버전이 릴리스 되느냐 정도였음.
- 커뮤니티 에디션은 당연히 소스코드가 공개돼 있었고, 엔터프라이즈 에디션도 라이선스 계약을 맺은 사용자에게는 소스코드를 공개했었음.
- 2011년 2월 MySQL 5.5 GA(General Available) 버전부터는 에넡프라이즈 에디션의 소스코드는 공개되지 않도록 바뀜.
  <br>
- 오라클에서 인수한 초기에는 큰 변화가 없는 듯 보였으나, 이때부터 MySQL 서버의 소스코드 레벨부터 리팩토링 되었음.
- 5.5부터 5.7버전까지는 안정성과 성능 개선에 집중
- 8.0 버전부터는 상용 DBMS가 가지고 있는 기능들이 장착됨.

--------------------------

## 1.2 왜 MySQL인가?
### MySQL의 경쟁력과 사용 이유
#### 가격이나 비용의 경쟁력
- 오라클과 비교했을 때, MySQL의 경쟁력은 가격이나 비용!
    - 이전에는 오라클과 MySQL의 주요 고객이 아예 달랐음.
    - DBMS는 보수적이기 때문에, 쉽게 기존의 DBMS를 다른 DBMS로 바꾸려고 하지는 않음.
    - 하지만, 이제는 금전적인 트랜잭션 처리라고 해서, MySQL 서버를 처음부터 배제하지는 않음.
    - 유명 포털 사이트도 빌링 시스템, 국내 대형 은행 시스템(코어 뱅킹 시스템은 아니더라도...)에서도 이제는 MySQL을 사용

 <br>
 -  최근 10년 간의 전자 제품 발전과 서버 컴퓨터 시장의 변화는 이전에 없던, 새롭고 엄청난 데이터를 만들어내기 시작
 - MySQL이 오라클의 RDBMS와 경쟁하지 않아도, 사용될 곳이 무한정 늘어나고 있음.
    - RDBMS는 방대한 양의 데이터를 저장하기에는 너무 비쌈.   
    
--------------------------

### 어떤 DBMS를 사용해야 하나? 어떤 DBMS가 좋은가?

**자기가 잘 사용하는 DBMS가 가장 좋은 DBMS이다!**
그럼에도 고민이 된다면, 아래 순서로 고려
- 안정성
- 성능과 기능
- 커뮤니티나 인지도
------------------------
<br>
## 2.1 MySQL 서버 설치
### MySQL 설치 방법
- Tar 또는 Zip으로 압축된 버전
- 리눅스 RPM 설치 버전(윈도우 인스톨러 및 macOS 설치 패키지)
- 소스코드 빌드

이 중에서, 리눅스의 RPM이나 운영체제 별 인스톨러 이용을 권장!

### 2.1.1 버전과 에디션 선택
####  버전 선택
- 다른 제약 사항(기존 솔루션이 특정 버전만 지원하는 경우)가 없다면, 가능한 최신 버전을 설치하는 것이 좋음.
- 기존 버전에서 새로운 메이저 버전으로 업그레이드 하는 경우라면, 최소 패치 버전이 15~20번 이상 릴리스 된 버전을 선택하는 것이 안정적인 서비스에 좋음.
    - 즉, MySQL 8.0버전이라면, MySQL 8.0.15부터 MySQL 8.0.20 사이의 버전부터 시작하는 것을 권장
- 새로운 서비스라면, 서비스 개발과 동시에 데이터베이스 서버를 함께 테스트해 나갈 수 있기 때문에 조금 더 빠른 패치 버전부터 시작해도 괜찮을 수 있음.
    - 하지만, 갓 출시된 메이저 버전을 선택하는 것은 위험할 수 있음.
    - 메이저 버전은 많은 변화를 거친 버전이므로, 갓 출시된 상태에서는 치명적이거나 보완하는 데 많은 시간이 걸릴 만한 버그가 발생할 수 있기 때문

#### 에디션 선택
- 초기에는 실제 MySQL 서버의 기능에 차이가 있었던 것이 아니라, 기술 지원의 차이만 있었음.
- MySQL 5.5 버전부터는 커뮤니티와 엔터프라이즈 에디션의 기능이 달라지면서, 소스코드도 달라졌고, MySQL 엔터프라이즈 에디션의 소스코드는 더이상 공개되지 않음.
- MySQL 서버의 상용화 전략은 핵심 내용은 두 에디션 모두 동일,
  특정 부가 기능들만 상용 버전인 엔터프라이즈 에디션에 포함되는 방식
  => 이런 상용화 방식을 `오픈 코어 모델(Open Core Model)`이라고 함.
- 엔터프라이즈 에디션에서만 지원되는 부가적 기능 & 서비스
    - Thread Pool
    - Enterprise Audit
    - Enterpirise TDE(Master Key 관리)
    - Enterpirise Authentication
    - Enterpirise Monitor
    - Enterpirise Backup
    - MySQL 기술 지원
- Percona에서 출시하는 Percona Server 백업 및 모니터링 도구 또는 Percona Server에서 지원하는 플러그인(Thread Pool과 Audit 플러그인 등)을 활용하면, MySQL 커뮤니티 에디션의 부족한 부분을 메꿀수 있음
  -> 엔터프라이즈 에디션에서 지원하는 것들이 꼭 필요한지 검토 필요!

### 2.1.2 MySQL 설치
#### 2.1.2.1 리눅스 서버의 Yum 인스톨러 설치
- Yum 인스톨러를 이용하기 위해서는 MySQL 소프트웨어 리포지토리(Repository) 등록 필요 <br> => `MySQL 다운로드 페이지`에서 RPM 설치 파일을 직접 받아서 설치 필요

- 각 운영체제의 버전에 맞는 RPM 파일을 다운로드해서 MySQL 서버를 설치하고자 하는 리눅스 서버에서 다음과 같이 Yum 리포지토리 정보 등록

  ```
  linux> sudo rpm -Uvh mysql180-community-release-e17-3.noarch.rpm
  ```
- Yum 리포지토리가 등록되면, MySQL 설치용 RPM 파일들이 저장된 경로를 가진 파일이 생성된 것을 확인 가능
  ```
  linux> ls -alh /etc/yum.repos.d/*mysql*
  ```
- Yum 인스톨러 명령을 이용해 버전별로 설치 가능한 MySQL 소프트웨어 목록 확인 가능해짐.
   ```
   
   ## 어떤 RPM 패키지가 있는지 확인 가능
  linux> sudo yum search mysql-community
  
  ## 설치 가능한 모든 버전 확인 가능
  linux> sudo yum ---showduplicates list mysql-community-server
  ```
cf) yum 명령어 앞에 사용된 sudo 명령어는 yum 명령어를 root 권한으로 실행하게 해줌. MySQL 서버를 설치하는 과정에서 리눅스 서버의 관리자만 접근할 수 있는 디렉터리에 파일들을 복사하기 때문에 반드시 root 권한이 필요함. 따라서, 만약 현재 사용자가 root가 아니라면, sudo 명령을 yum 명령과 함께 사용해야 함.

- 리눅스 서버에서 Yum 인스톨러나 RPM 설치를 하더라도, MySQL 서버를 바로 시작할 수 없음.

#### 2.1.2.2 리눅스 서버에서 Yum 인스톨러 없이 RPM 파일로 설치
- Yum 인스톨러를 사용하지 않고 RPM 패키지로 직접 설치하려면, 설치에 필요한 RPM 패키지 파일을 직접 다운로드 해야함.
- MySQL RPM 다운로드 페이지에서 운영체제의 버전과 CPU 아키텍처를 선택한 후, RPM 패키지를 다운로드 하면 됨.

- RMP 패키지 다운로드 페이지에서 RPM 패키지를 다운로드 후 의존 관계 순서대로 설치하면 됨.

#### 2.1.2.3 macOS용 DMG 패키지 설치
macOS에서 인스톨러로 설치하려면, 설치에 필요한 DMG 패키지 파일들을 직접 다운로드해야 함.

- MySQL 다운로드 페이지에서 운영체제의 버전을 선택한 후, DMG 패키지 파일을 다운로드
- **사용자 인증 방식 선택**
    - `Use Strong Password Encryption` : Caching SHA-2 Authentication 사용 -> 인터넷을 경유해서 MySQL 서버에 접속하는 경우
    - `Use Legacy Password Encryption` : Native Authentication 사용 -> 사설 네트워크에서만 사용하면 괜찮음.

- 기본설정으로 설치하면, 데이터 디렉터리와 로그 파일들을 `usr/local/mysql`디렉터리 하위에 설정하고, 관리자 모드로 MySQL 서버를 가동하기 때문에 관리자 계정에 비밀번호 설정이 필요

- **절대 삭제하면 안되는 디렉터리**
    - MySQL이 설치된 디렉터리 : `usr/local/mysql`
    - `bin` : MySQL 서버와 클라이언트 프로그램. 유티릴티를 위한 디렉터리
    - `data` : 로그 파일과 데이터 파일들이 저장되는 디렉터리
    - `include` : C/C++ 헤더 파일들이 저장된 디렉터리
    - ` lib` : 라이브러리 파일들이 저장된 디렉터리
    - `share` : 다양한 지원파일들이 저장되어 있으며, 에러 메시지나 샘플 설정 파일(my.cnf)이 있는 디렉터리

- macOS에 설치된 MySQL 서버의 설정 파일 등록 및 시작 종료는 `시스템 환경설정`의 최하단에 있는 `MySQL`을 클릭하면 실행되는 MySQL 관리 프로그램에서 수행 가능

- macOS에 설치된 MySQL 서버의 관리 프로그램에서 MySQL 서버를 시작하거나 종료 가능.
- macOS의 터미널에서 MySQL 서버를 시작하고 종료도 가능
  ```
  ## MySQL 서버 시작
	macOs> sudo /usr/local/mysql/support-files/mysql.server start
   ## MySQL 서버 종료
	macOs> sudo /usr/local/mysql/support-files/mysql.server stop
  ```
- MySQL 설정 파일 경로(Configuration File) : `/usr/local/mysql/my.cnf`로 설정

#### 2.1.2.4 윈도우 MSI 인스톨러 설치
- MySQL 다운로드 페이지에서 운영체제의 버전을 선택하여, MSI 인스톨 프로그램을 다운로드

----
## 2.2 MySQL 서버의 시작과 종료
### 2.2.1 설정 파일 및 데이터 파일 준비
- 리눅스 서버에서 Yum 인스톨러나 RPM을 이용해 MySQL 서버를 설치하면, 서버에 필요한 프로그램들과 디렉터리들은 일부 준비되지만, 트랜잭션 로그 파일과 시스템 테이블이 준비되지 않았기 때문에 MySQL 서버를 시작할 수 없음.

- MySQL 서버가 설치되면, `etc/my.cnf` 설정 파일이 준비됨. 이 파일은 MySQL 서버를 실행하는 데 꼭 필요한 3~4개의 아주 기본적인 설정만 기록됨.

    - 실제 서비스용으로 사용하기에는 많이 부족하지만, 테스트 용으로 실행한다면 괜찮음.

- 초기 데이터파일(시스템 테이블이 저장되는 데이터 파일)과 트랜잭션 로그(리두 로그) 파일을 생성

    - `--initialize-insecure` : 비밀번호가 없는 관리자 계정인 root 유저 생성
    - `--initialize` : 비밀번호를 가진 관리자 계정 생성
    - 에러 로그파일의 기본 경로 : `var/log/mysqld.log`


```
  linux> mysqld --defaults-file=/etc/my.cnf --initialize-insecure
  ```


### 2.2.2 시작과 종료
- 유닉스 걔열 운영체제에서 RPM 패키지로 MySQL을 설치했다면 자동으로 `usr/lib/systemd/system/mysqld.service` 파일이 생성됨.

- **systemctl 이용**
  ```
  ## 서버 시작
  linux> systemctl start mysqld

  ## 서버 상태
  linux> systemctl status mysqld

  ## 서버 종료
  linux> systemctl stop mysqld
  ```

- **원격적으로 MySQL 서버 셧다운**
  ```
  mysql> SHUTDOWN;
  ```

- **MySQL 서버가 종료될 때 모든 커밋된 내용을 데이터 파일에 기록하고 종료**

    - 실제 트랜잭션이 정상적으로 커밋되어도 데이터 파일에 변경된 내용이 기록되지 않고, 로그 파일(리두 파일)에만 기록되어 있을 수 있음.
    - 심지어 MySQL 서버가 종료되고도 다시 시작된 이후에도 계속 이 상태로 남아있을 수도 있음.
  ```
  mysql> SET GLOBAL innodb_fast_shutdown=0;
  linuxx> systemctl stop mysqld.service
  
  ## 또는 원격으로 MySQL 서버 종료 시
  mysql> SET GLOBAL innodb_fast_shutdown=0;
  mysql> SHUTDOWN;
  ```

**위와 같이 모든 커밋된 데이터를 데이터 파일에 적용하고 종료하는 것을 클린 셧다운(Clean shutdown이라고 표현** -> 클린 셧다운으로 종료되면 다시 MySQL 서버가 기동될 때 별도의 트랜잭션 복구 과정을 진행하지 않기 때문에 빠르게 시작 가능

### 2.2.3 서버 연결 테스트
#### MySQK 서버 접속하기

1. MySQL 소켓 파일을 이용해 접속
```
## MySQL 소켓 파일을 이용해 접속
linux> mysql -uroot -p --host=localhost --socket=/tmp/mysql.sock
```
2. TCP/IP를 통해 접속

```
## TCP/IP를 통해 접속
≈
```

- 원격 호스트에 있는 MySQL 서버에 접속하는 경우에 반드시 사용
- `--host=localhosst`

    - 클라이언트 프로그램이 항상 소켓 파일을 이용해서 접속
    - `Unix domain socket`을 이용하는 방식으로, TCP/IP를 통한 통신이 아니라, 유닉스의 프로세스 간 통신(IPC: Inter Process Communication)의 일종


- `--host=127.0.0.1`

    - 자기 서버를 가리키는 루프백(loopback) IP이기는 하지만 TCP/IP 통신 방식을 사용하는 것


3. 별도로 호스트 주소와 포트 명시하지 않음.
```
linux> mysql -uroot -p
```
- 기본값으로 호스트는 localhost가 되며, 소켓 파일을 사용하게 되는데, 소켓 파일의 위치는 MySQL 서버의 설정 파일에서 읽어서 사용
- MySQL 서버가 기동될 때 만들어지는 유닉스 소켓 파일은 MySQL 서버를 재시작하지 않으면 다시 만들어 낼 수 없기 때문에 실수로 삭제하지 않도록 주의 필요
- 유닞ㄱ스나 리눅스에서 mysql 클라이언트 프로그램을 실행하는 경우에는 mysql 프로그램의 경로를 PATH 환경변수에 등록해 둔다.

4. 데이터베이스 목록 확인
```
linux> mysql -h127.0.0.1 -uroot -p
...
mysql> SHOW DATABASES;
```
- 때로는 MySQL 서버를 직접 로그인 하지 않고, 원격 서버에서 MySQL 서버의 접속 가능 여부만 확인해야 하는 경우 있음.
- 이처럼 커넥션이 가능한지만 확인하는 경우에는 MySQL 클라이언트를 설치하는 작업이 번거로울 수 있고, 때로는 보안상 이유로 MySQL 클라이언트 프로그램을 설치하지 못할 수도 있음.<br>
  **이 경우에는 `telnet` 명령이나 `nc(Netcat)` 명령을 이용해 원격지 MySQL 서버가 응답 가능한 상태인지 확인 가능**

---
## 2.3 MySQL 서버 업그레이드
#### 서버 업그레이드 하는 방법
**1. 인플레이스 업그레이드(In-place Upgrade)**
- MySQL 서버의 데이터 파일을 그대로 두고 업데이트 하는 방법
- 여러가지 제약 사항이 있음.
- 업그레이드 시간 크게 단축 가능

**2. 논리적 업그레이드(Logical Upgrade)**
- mysqldump 도구 등을 이용해 MySQL 서버의 데이터를 SQL 문장이나 텍스트로 덤프한 후, 새로 업데이트된 버전의 MySQL 서버에서 덤프된 데이터를 적재하는 방법
- 버전 간 제약 사항 거의 없음.
- 업그레이드 시간 많이 걸릴 수 있음.

### 2.3.1 인플레이스 업그레이드 제약 사항
#### [업그레이드 종류]

**1. 마이너(패치) 버전 간 업그레이드**
- 대부분 데이터 파일의 변경 없이 진행
- 많은 경우 여러 버전을 건너뛰어서 업그레이드 하는 것 허용
- ex) MySQL 8.0.16 버전 -> MySQL 8.0.21 버전 : MySQL 서버 프로그램만 재설치 하면 됨.

**2. 메이저 버전 간 업그레이드 **

- 대부분 크고 작은 데이터 파일의 변경이 필요
- 반드시 직전 버전에서만 업그레이드가 허용됨.

    - ex) MySQL 5.5 버전 -> MySQL 5.6 버전 : `허용`
    - ex) MySQL 5.5 버전 -> MySQL 5.7 버전 : `지원 안됨`
    - ex) MySQL 5.5 버전 -> MySQL 8.0 버전 : `지원 안됨`

        - 메이저 버전 업그레이드는 데이터 파일의 패치가 필요한데, MySQL 8.0 서버 프로그램은 직전 메이저 버전인 MySQL 5.7 버전에서 사용하던 데이터 파일과 로그 포맷만 인식하도록 구현되기 때문

 <br>
- 두 단계 이상을 한 번에 업그레이드해야 한다면, mysqldump 프로그램으로 MySQL 서버에서 데이터를 백업 받는 후 새로 구축된 MySQL 8.0 서버에 데이터를 적재하는 `논리적 업그레이드`가 더 나은 방법일 수 있음.
<br>

- 메이저 버전 업그레이드가 특정 마이너 버전에서만 가능한 경우도 있음.

    - ex) MySQL 5.78 버전 -> MySQL 8.0 버전 : `업그레이드 불가`
    - GA(General Availability) 버전(안정성 확인된 버전)인 경우에만 가능

cf) MySQL 서버를 선택할 때에도 최소 GA 버전은 지나서 15~20번 이상의 마이너 버전 선택하는 것이 좋음.

### 2.3.2 MySQL 8.0 업그레이드 시 고려 사항
- MySQL 8.0에서 상당히 많은 기능들이 개선되거나 변경됨.
- MySQL 5.7 버전의 기본적인 차이점과 MySQL 8.0에서 사용할 수 없는 기능들이 있음.

#### MySQL 8.0으로 업그레이드 이전 검토해야 할 목록
- 사용자 인증 방식
- MySQL 8.0과 호환성 체크
- 외래키 이름의 길이
- 인덱스 힌트
- GroupBy에 사용된 정렬 옵션
- 파티션을 위한 공용 테이블스페이스


#### 실행 방법 및 체크하는 명령
```
## mysqlcheck 유틸리티 실행 방식
linux> mysqlcheck -u root -p -all-databases --check-upgrade
```

### 2.3.3 MySQL 8.0 업그레이드
MySQL 5.7에서 MySQL 8.0으로 업그레이드 하는 과정은 이전 버전처럼 단순하지 않음.<br>
- 대표적으로 MySQL 8.0 버전부터는 시스템 테이블의 정보와 데이터 딕셔너리 정보의 포맷이 완전히 바뀜.

#### MySQL 5.7에서 MySQL 8.0으로의 업그레이드는 크게 두가지로 나누어 이루어진다.
**1. 데이터 딕셔너리 업그레이드**
- `MySQL 5.7` : 데이터 딕셔너리 정보가 FRM 확장자를 가진 파일로 별도 보관됨.
- `MySQL 8.0` : 데이터 딕셔너리 정보가 트랜잭션이 지원되는 InnoDB 테이블로 저장되도록 개선됨.
- 데이터 딕셔너리 업그레이드는 기존의 FRM 파일의 내용을 InnoDB 시스템 테이블로 저장
- MySQL 8.0 버전부터는 딕셔너리 데이터의 버전 간 호환성 관리를 위해 테이블이 생성될 때 사용된 MySQL 서버의 버전 정보도 함께 기록

**2. 서버 업그레이드**
- MySQL 서버의 시스템 데이터베이스(perfomance_schema와 information_schema, 그리고 mysql 데이터베이스)의 테이블 구조를 MySQL 8.0에 맞게 변경

** `MySQL 8.0.15 이하 버전으로 업그레이드`**

- `'데이터 딕셔너리 업그레이드'` : MySQL 서버(mysqld) 프로그램이 실행
- `'서버 업그레이드'` : mysql_update 프로그램이 실행

** `MySQL 8.0.16부터`**
- mysql_upgrade 유틸리티가 없어지고, MySQL 서버(mysqld)이 시작되면서 `'데이터 딕셔너리 업그레이드'`와 `'서버 업그레이드'`를 순서대로 실행

----
## 2.4 서버 설정
- 일반적으로 MySQL 서버는 단 하나의 설정 파일을 사용

    - `유닉스 계열(리눅스 포함)` : my.cnf
    - `윈도우 계열` : my.ini
- 설정 파일의 경로가 고정돼 있는 것은 아님.
- MySQL 서버는 지정된 여러 개의 디렉터리 순차 탐색하며 처음 발견된 my.cnf 파일을 사용하게 됨.
- 어떤 파일을 사용하는지 궁금하다면, `--verbose --help` 옵션을 주어서 확인해보면됨.

    - mysqld 프로그램은 MySQL 서버의 실행프로그램으로 서비스용으로 사용되는 서버에서 이미 MySQL 서버가 실행 중인데 다시 mysqld 프로그램을 시작한다거나 하지 않도록 주의 필요


```
shell> mysqld --verbose --help

## (권장)
shell> mysql --help
```

- 실제 MySQL 서버는 단 하나의 설정 파일만 사용하지만, 설정 파일이 위치한 디렉터리는 여러 곳 일 수 있음. -> 사용자 혼란스럽게 만드는 부분이기도...


### 2.4.1 설정 파일의 구성
- 하나의 `my.cnf`나 `my.ini` 파일에 여러 개의 설정 그룹을 담을 수 있음.
- 대체로 실행 프로그램 이름을 그룹명으로 사용함.
- ex) mysqldump 프로그램은 [mysqldump] 설정 그룹을, mysqld 프로그램은 설정 그룹 이름이 [mysqld]인 영역을 참조, mysqld_safe vmfhrmfoadms [mysqld_safe]와 [mysqld] 섹션을 참조

### 2.4.2 MySQL 시스템 변수의 특징
- MySQL 서버는 기동하면서 설정 파일의 내용을 읽어 메모리나 작동 방식을 초기화하고, 접속된 사용자를 제어하기 위해 이러한 값을 별도로 저장해 둠. <br>
  이렇게 저장된 값 -> `시스템 변수(System Variables)`

#### 시스템 변수 확인
```
mysql> SHOW GLOBAL VARIABLES
```

#### 설명 페이지의 변수

| Name                  | Cmd-Line | Option File | System Var | Var Scope | Dynamic |
|-----------------------|----------|-------------|------------|-----------|---------|
| activate_all_roles_on_login | Yes      | Yes         | Yes        | Global    | Yes     |
| admin_address         | Yes      | Yes         | Yes        | Global    | No      |
| admin_port            | Yes      | Yes         | Yes        | Global    | No      |
| time_zone             | Yes      | Yes         | Yes        | Both      | Yes     |
| sql_10g_bin           | Yes      | Yes         | Yes        | Session   | Yes     |

- Cmd-Line

    - 서버의 명령행 인자로 설정될 . 수있는지 여부
- Option file

    - MySQL 설정파일인 my.cnf(또는 my.ini)f로 제어할 수 있는지 여부
- System Var

    - 시스템 변수인지 아닌지
- Var Scope

    - 시스템 변수의 적용 범위
- Dynamic

    - 시스템 변수가 동적인지 정적인지


### 2.4.3 글로벌 변수와 세션 변수
- 글로벌 변수와 세션 변수 : `시스템 변수의 적용 범위`에 따라 나눠짐.
- 일반적으로 세션별로 적용되는 시스템 변수의 경우 글로벌 변수뿐만 아니라 세션 변수에도 동시에 존재함.

    - 이런 경우, MySQL의 매뉴얼의 'Var Scope'에 'Both'라고 표시됨.


#### 글로벌 범위의 시스템 변수
- 하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수를 의미함.
- 주로 MySQL 서버 자체에 관련된 설정일 때가 많음
- MySQL 서버에서 단 하나만 존재하는 InnoDB 버퍼 풀 크기(innodb_buffer_poop_size) 또는 MyISAM의 키 캐시 크기(key_buffer_size) 등이 가장 대표적

#### 세션 범위의 시스템 변수
- MySQL 클라이언트가 MySQL에 접속할 때기본적으로 부여하는 옵션의 기본값을 제어하는데 사용됨.
- 각 클라이언트가 처음에 접속하면 기본적으로 부여하는 기본값을 가짐
- 별도로 변경하지 않으면 그대로 값이 유지되지만, 클라이언트의 필요에 따라 개별 커넥션 단위로 다른 값으로 변경 가능
- 여기서 기본값은 글로벌 시스템 변수이며, 각 클라이언트가 가지는 값이 세션 시스템 변수
- autocommit 변수가 대표적인 예
  <br>

#### 범위가 Both인 시스템 변수
- 세션 범위의 시스템 변수 가운데 MySQL 서버의 설정 파일(my.cnf / my.ini)에 명시해 초기화 할 . 수있는 변수는 대부분 범위가 'Both'
- MySQL 서버가 기억만 하고 있다가 실제 클라이언트와의 커넥션이 생성되는 순간에 해당 커넥션의 기본값으로 사용되는 값
- 순수하게 세션으로 명시된 시스템 변수는 MySQL 서버의 설정 파일에 초기값을 명시할 수없으며, 커넥션이 만들어지는 순간부터 해당 커넥션에서 유효한 설정 변수를 의미함.

### 2.4.4 정적 변수와 동적 변수
- 정적변수와 동적 변수 : `MySQL 서버가 기동 중인 상태에서 변경 가능한지`에 따라 구분됨.

- MySQL 서버의 시스템 변수는 디스크에 저장되어 있는 설정 파일(my.cnf 또는 my.ini)을 변경하는 경우와 이미 가동 중인 MySQL 서버의 메모리에 있는 MySQL 서버의 시스템 변수를 변경하는 경우로 나눌 수 있음.

    - 디스크에 저장된 설정 파일의 내용은 변경하더라도, MySQL 서버를 재시작하기 전에는 적용되지 않음.
    - `SHOW` 명령으로 MySQL 서버에 적용된 변숫값을 확인하거나 `SET` 명령을 이용해 값 바꿀 수도 있음.
    -  `SET` 명령어을 통해 변경되는 시스템 변숫값이 MySQL 설정 파일인 my.cnf(또는 my.ini) 파일에 반영되는 것은 아니므로, 현재 구동중인 인스턴스에서만 유효함.
    - MySQL 서버가 재시작 하면, 다시 설정 파일의 내용으로 초기화 -> 설정을 영구히 저장하려면, my.cnf 파일도 바꾸어야함.
    - MySQL 8.0 버전부터는 `SET PERSIST` 명령을 이용하면 실행중인 MYSQL 서버의 시스템 변수를 변경함과 동시에 자동으로 설정 파일로도 기록됨.
    - `SHOW`와 `SET` 명령에서 `GLOBAL 키워드`를 사용하면, 글로벌 시스템 변수의 목록과 내용을 읽고 변경 가능, `GLOBAL 키워드`를 빼면 자동으로 세션 변수를 조회하고 변경

#### 글로벌 시스템 변수의 경우
MySQL 서버의 기동 중에는 변경할 수 없는 것이 많지만, 실시간으로 변경할 수있는 것도 있음.
- `SET PERSIST` 명령을 사용하는 경우 변경된 시스템 변수는 my.cnf 파일이 아닌 별도의 파일에 기록됨

#### 시스템 변수의 범위가 Both인 경우
글로벌 시스템 변수의 값을 변경해도 이미 존재하는 커넥션의 세션 변수 값은 변경되지 않고 그대로 유지됨.


### 2.4.5 SET PERSIST
**시스템 변수는 동적 변수와 정적 변수로 구분됨**

#### 동적 변수
- MySQL 서버에서 'SET GLOBAL' 명령으로 변경하면 즉시 MySQL 서버에 반영됨.
- 이 경우, 설정 파일에도 변경 내용을 적용해야하는 데, 이 문제점 보완하기 위해 `SET PERSIST` 명령을 도입
- `SET PERSIST` 명령을 사용하면, 변경된 값 즉시 적용 + 다시 시작할 때 기본 설정 파일(my.cnf) 뿐 아니라 별도의 설정 파일(mysql-auto.cnf)에 변경 내용을 추가로 기록
- 이 두 파일을 같이 참조해서 시스템 변수를 적용함.
- 따라서, 수동으로 변경하지 않아도, 자동으로 영구적으로 변경됨.
  <br>

**SET PERSIST ONLY**
- 현재 실행중인 서버에는 변경 내용을 적용하지 않고, 다음 재시작을 위해서 mysqld-auto.cnf 파일에만 변경 내용을 기록할 때 사용
- 정적인 변수의 값을 영구적으로 변경하고자 할 때에도 사용 가능

### 2.4.6 my.cnf 파일
- MySQL 8.0의 시스템 변수는 약 570개
- 사용하는 플러그인이나 컴포넌트에 따라 시스템 변수의 개수 더늘어날 수있음.

--------------------------

<br>

# Chapter 3. 사용자 및 권한

- MySQL에서 사용자 계정 설정하는 방법이나 각 계정의 권한 설정하는 방법은 다른 DBMS와 차이가 있음.

    - MySQL 사용자 계정 : 단순히 사용자 아이디 뿐 아니라, 해당 사용자가 어느 IP에서 접속하고 있는지도 확인
    - MySQK 8.0 버전 부터는 권한을 묶어서 관리하는 역할(Role, 롤)의 개념도 도입됨. -> 각 사용자의 권한으로 미리 준비된 권환 세트(Role) 부여 가능

## 3.1 사용자 식별
- MySQL의 사용자는 사용자 계정 뿐 아니라, 사용자의 접속 지점(클라이언트가 실행된 호스트명이나 도메인 또는 IP 주소)도 계정의 일부가 됨

    - 따라서, MySQL에서 계정을 언급할 때, 항상 아이디와 호스트를 함게 명시
    - 아이디와 IP주소를 감싸는 역따옴표(`)는 식별자를 감싸는 따옴표 역할을 함.
    - 종종 홀따옴표(')로 바뀌어서 사용되기도 함.

- `'svc_id'@'127.0.0.1'`

    - 항상 MySQL 서버가 기동 중인 로컬 호스트에서 svc_id로 접속할 때만 사용될 수 있는 계정이라는 의미
    - 다른 컴퓨터에서는 svc_id라는 아이디로 접속할 수 없음.
- `'svc_id'@'192.168.0.10' (이 계정의 비밀번호는 123)` <br>`'svc_id'@'%' (이 계정의 비밀번호는 abc)`

    - 모든 외부 컴퓨터에서 접속 가능한 사용자 계정 생성하고 싶다면 호스트 부분을 `%`로 대체하면 됨.
    - 권한이나 계정 정보에 대해 MySQL의 범위가 가장 작은 것을 항상 먼저 선택 (192.168.0.10)의 IP에서 접속할 경우 'abc'라는 비밀번호는 접속이 거절됨.



## 3.2 사용자 계정 관리
### 3.2.1 시스템 계정과 일반 계정
MySQL 8.0부터 계정은 SYSTEM_USER 권한을 가지고 있느냐에 따라서 시스템 계정과 일반 계정으로 나누어짐.

#### 시스템 계정(System Account)
- SYSTEM_USER 권한을 가짐.
- MySQL 서버 내부적으로 실행되는 백그라운드 스레드와 무관
- 일반계정과 같이 사용자를 위한 계정
- 데이터베이스 서버 관리자를 위한 계정
- 시스템 계정과 일반 계정을 관리(생성, 삭제 및 변경) 가능
- 데이터베이스 서버 관리와 관련된 중요 작업은 시스템 계정으로만 수행 가능

    - 계정 관리(계정 생성 및 삭제, 그리고 계정의 권한 부여 및 제거)
    - 다른 세션 또는 그 세션에서 실행 중인 쿼리를 강제 종료
    - 스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정

#### 일반 계정(Regular Account)
- SYSTEM_USER 권한을 가지지 않음.
- 일반 응용 프로그램이나 개발자를 위한 계정
- 시스템 계정을 관리 불가

#### 시스템 계정과 일반 계정 개념 도입 이유
- DBA(데이터베이스 관리자) 계정에는 `SYSTEM_USER` 권한을 할당하고, 일반 사용자를 위한 계정에는 `SYSTEM_USER` 권한을 부여하지 않기 위해서


#### MySQL 내장된 계정
- `'mysql.sys'@'localhost'`  : MySQL 8.0부터 기본으로 내장된 sys 스키마의 객체(뷰나 함수, 그리고 프로시저)들의 DEFINER로 사용되는 계정
- `'mysql.session'@'localhost'` : MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- `'mysql.infoschema'@'localhost'` : information_schema에 정의된 뷰의 DEFINER로 사용되는 계정
  <br>**위 계정들은 처음부터 잠겨(account_locked 컬럼)있는 상태이므로 의도적으로 잠긴 계정을 풀지 않는 한 악의적인 용도로 사용할 수 없으므로 보안 걱정할 필요 X**

### 3.2.2 계정 생성
- `MySQL 5.7버전까지`

    - `GRANT` 명령으로 권한의 부여와 동시에 계정 생성 가능

- `MySQL 8.0버전부터`

    - **계정 생성** : `CREATE USER` 명령
    - **권한 부여** : `GRANT` 명령
    - 계정 생성할 때 다양한 옵션 설정 가능

        - 계정의 인식 방식과 비밀번호
        - 비밀번호 관련 옵션(비밀번호 유효 기간, 비밀번호 이력 개수, 비밀번호 재사용 불가 기간)
        - 기본 역할(Role)
        - SSL 옵션
        - 계정 잠금 여부

#### 3.2.2.1 IDENTIFIED WITH
- 사용자의 인증방식과 비밀번호 설정
- `IDENTIFIED WITH` 뒤에는 반드시 **'인증방식(인증 플러그인의 이름)'** 명시해야 함.

    - 기본 인증 방식 사용 : `IDENTIFIED WITH 'password'`
- MySQL에서는 다양한 인증 방식을 플러그인으로 제공

    - `Native Pluggable Authentication` : MySQL 5.7버전까지 기본 인증 방식
    - `Caching SHA-2 Pluggable Authentication` : MySQL 8.0 버전부터 기본 인증 방식
    - `PAM Pluggable Authentication`
    - `LDAP Pluggable Authentication`

- `CREATE USER`과 `ALTER USER` 명령을 이용해 MySQL 서버의 계정을 생성 또는 변경할 때 연결 방식과 비밀번호 옵션, 자원 사용과 관련된 여러 옵션을 설정할 수 있음.

#### 3.2.2.2 REQUIRE
- MySQL 서버에 접속할 때 암호화된 SSL/TLS 채널을 사용할지 여부를 설정함.
- 별도로 설정하지 않으면 비암호화 채널로 연결하게 됨
- `REQUIRE`	 옵션을 SSL로 설정하지 않아도 `Caching SHA_2 Authentication` 인증 방식을 사용하면 암호화된 채널만으로 MySQL 서버에 접속 가능

#### 3.2.2.3 PASSWORD EXPIRE
- 비밀번호의 유효기간을 설정하는 옵션
- 별도로 명시하지 않으면 `default_password_lifetime` 시스템 변수에 저장된 기간을 유효기간으로 설정
- 개발자나 데이터베이스 관리자의 비밀번호는 유효기간을 설정하는 것이 보안상 안전
- 응용 프로그램 접속용 계정에는 유효기간을 설정하는 것은 위험할 수 있음.
- `PASSWIRD EXPIRE` 절에 설정 가능한 옵션들

    - `PASSWORD EXPIRE` : 계정 생성과 동시에 비밀번호의 만료 처리
    - `PASSWORD EXPIRE NEVER` : 계정 비밀번호의 만료기간 없음.
    - `PASSWORD EXPIRE DEFAULT` : `default_password_lifetime` 시스템 변수에 저장된 기간으로 비밀번호 유효기간 설정
    - `PASSWORD EXPIRE INTERVAL n DAY` : 비밀번호 유효기간을 오늘부터 n일자로 설정

#### 3.2.2.4 PASSWORD HISTORY
- 한 번 사용했던 비밀번호를 재사용하지 못하게 설정하는 옵션
- `PASSWORD HISTORY` 절에 설정 가능한 옵션

    - `PASSWORD HISTORY DEFAULT` : `password_history` 시스템 변수에 저장된 개수만큼 비밀번호 이력을 저장하며, 저장된 이력에 남아있는 비밀번호는 사용 불가
    - `PASSWORD HISTORY n` : 비밀번호의 이력을 최근 n개까지만 저장하며, 저장된 이력에 남아있는 비밀번호는 재사용할 수 없음.
- 한 번 사용했던 비밀번호를 사용하지 못하게 하려면, 이전에 사용했던 비밀번호를 MySQL 서버가 저장해야 함. -> `password_history` 테이블 사용

#### 3.2.2.5 PASSWORD REUSE INTERVAL
- 한 번 사용했던 비밀번호의 재사용 금지 기간 설정하는 옵션
- 별도로 명시하지 않으면 `password_reuse_interval` 시스템 변수에 저장된 기간으로 설정됨
- `PASSWORD REUSE INTERVAL` 절에 설정 가능한 옵션

    - `PASSWORD REUSE INTERVAL DEFAULT` : `password_reuse_interval` 변수에 저장된 기간으로 설정
    - `PASSWORD REUSE INTERVAL n DAY` : n일자 이후에 비밀번호를 재사용할 수 있게 설정

#### 3.2.2.6 PASSWORD REQUIRE
- 비밀번호가 만료되어 새로운 비밀번호로 변경할 때 현재 비밀번호(변경하기 전 만료된 비밀번호)를 필요로 할 지 말지를 결정하는 옵션
- 별도로 명시하지 않으면 `password_require_current` 시스템 변수의 값으로 설정됨
-  `PASSWORD REQUIRE` 절에 설정 가능한 옵션

- `PASSWORD REQUIRE CURRENT` : 비밀번호를 변경할 때 현재 비밀번호를 먼저 입력하도록 설정
- `PASSWORD REQUIRE OPTIONAL` : 비밀번호를 변경할 때 현재 비밀번호를 입력하지 않아도 되도록 설정
- `PASSWORD REQUIRE DEFAULT` : 시스템 변수의 값으로 설정

#### 3.2.2.7 ACCOUNT LOCK / UNLOCK
- 계정 생성 시, `ALTER USER` 명령을 사용해 계정 정보를 변경할 때 계정을 사용하지 못하게 잠글지 여부 결정
- `ACCOUNT LOCK` : 계정을 사용하지 못하게 잠금
- `ACCOUNT UNLOCK` : 잠긴 계정을 다시 사용 가능 상태로 잠금 해제

## 3.3 비밀번호 관리
### 3.3.1 고수준 비밀번호
- MySQL 서버의 비밀번호는 유효기간이나 이력 관리를 통한 재사용 금지 기능 뿐만 아니라 비밀번호를 쉽게 유추할 수 있는 단어들이 사용되지 않게 글자의 조합을 강제하거나 금칙어를 설정하는 기능 존재
- 비밀번호 유효성 체크 규칙 적용: `validate_password` 컴포넌트 사용

    - 우선 `validate_password` 컴포넌트 설치
    - 서버 프로그램에 내장되어있기 때문에 `INSTALL COMPONENT` 명령의 `file//` 부분에 별도의 파일 경로 지정 필요 X
```
## validate_password 컴포넌ㅌ 설치
mysql> INSTALL COMPONENT 'file://component_validate_password';

## 설치된 컴포넌트 확인
mysql> SELECT * FROM mysql.component;

## 컴포넌트에서 제공하는 시스템 변수 확인
mysql> SHOW GLOBAL VARIABLES LIKE 'validate_password%';
```

- 비밀번호 정책

    - `LOW` : 비밀번호 길이만 검증
    - `MEDIUM` : 비밀번호의 길이를 검증, 숫자와 대소문자, 특수문자 배합 검증(기본값)
    - `STRONG` : MEDIUM 레벨의 검증을 모두 수행, 금칙어 포함 여부 검증

- 금칙어 파일

    - `validate_password.dictionary_file` 시스템 변수에 금칙어들이 저장된 사전 파일 등록
    - 금칙어를 한 줄에 하나씩 기록해서 텍스트 파일로 작성
    - `validate_password.policy`가 `STRONG`인 경우에만 작동
      -> `validate_password.policy` 시스템 변수도 함께 변경 필요

      ```
      mysql> SET GLOBAL validate_password.dictionary_file='prohibitive_word.data';
      mysql> SET GLOBAL validate_password.policy='STRONG';
      ```

### 3.3.2 이중 비밀번호
많은 응용 프로그램 서버들이 공용으로 데이터베이스 서버를 사용하기 때문에 데이터베이스 서버의 계정 정보는 응용 프로그램 서버로부터 공용으로 사용되는 경우 많음.<br>
=> 데이터 베이스 서섭의 계정 정보 쉽게 변경 어려움, 특히 데이터베이스 계정의 비밀번호는 서비스가 실행 중인 상태에서 변경 불가능 했음.<br>
=> `MySQL 8.0` 버전부터는 계정의 비밀번호로 2개이 값을 동시에 사용 할 수있는 기능 추가 => `이중 비밀번호(Dual Password)`

<br>

- 하나의 계정에 대해 2개의 비밀번호를 동시에 설정할 수 있음.

    - `프라이머리 (Primary)` : 최근에 설정된 비밀번호
    - `세컨더리 (Secondary)` : 이전 비밀번호
- 이중 비밀번호를 사용하려면 기존 비밀번호 변경 구문에 `RETAIN CURRENT PASSWORD` 옵션만 추가하면 됨.

```
## root 계정의 비밀번호가 'old_passord'로 변경됨
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'old_password';

## 이전 비밀번호인 'old_password'는 세컨더리
## 새로운 비밀번호인 'new_password'는 프라이머리로 설정됨
## 둘 중아무거나 입력해도 로그인 가능
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;
```
- MySQL 서버에 접속하는 모든 응용 프로그램의 재시작이 완료되려면 이제 다음 명령으로 세컨더리 비밀 번호 삭제 (보안을 위해 삭제하는 것이 좋음)


## 3.4 권한(Privilege)
### MySQL 5.7 버전 이전

MySQL 5.7 버전까지 권한은 `글로벌(Global) 권한`과 `객체 단위의 권한`으로 구분됨.
#### 글로벌 권한
- 데이터베이스나 테이블 이외의 객체에 적용되는 권한
- `GRANT` 명령으로 권한을 부여할 때 반드시 특정 객체 명시하면 안됨.

#### 객체 권한
- 데이터베이스나 테이블을 제어하는데 필요한 권한
- `GRANT` 명령으로 권한을 부여할 때 반드시 특정 객체 명시 필요

<br>

**예외적으로 `ALL(또는 ALL PRIVILEGES)`은 글로벌과 객체 권한 두가지 용도로 사용 가능<br> 특히, 객체에 `ALL` 권한이 부여되면 해당 객체에 적용될 . 수있는 모든 객체 권한을 부여하며, 글로벌로 `ALL`이 사용되면 글로벌 수준에서 가능한 모든 권한 부여하게 됨.**



### MySQL 8.0버전 이후
- `동적 권한`이 추가됨.
- MySQL 5.7버전부터 제공된던 권한은 `정적 권한`이라고 함.

#### 동적 권한
- MySQL 서버가 시작되면서 동적으로 생성하는 권한
- ex) MySQL 서버의 컴포넌트나 플러그인이 설치되면 그때 등록되는 권한
- MySQL 5.7 버전까지는 `SUPER`라는 권한이 데이터베이스 관리를 위해 꼭 필요한 권한이었지만, MySQL 8.0버전부터는`SUPER`라는 권한이 잘게 쪼개져 `동적 권한`으로 분리됨. <br> => 백업 관리자와 복제 관리자 개별로 꼭 필요한 권한만 부여할 . 수있게 됨.
  <br>
- 사용자에게 권한을 부여할 때는 `GRANT` 명령을 사용

  ```
  mysql> GRANT privilege_list ON db.table TO 'user'@'host';
  ```
- MySQL 8.0부터는 존재하지 않는 사용자에 대해 `GRANT` 명령을 실해앟면 에러 발생 -> 반드시 사용자 먼저 생성하고 권한 부여
- `GRANT OPTION` : `GRANT` 명령의 마지막에 `WITH GRANT OPTION` 명시해서 부여

    - `privilege_list`에는 구분자(,)를 써서 앞으 표에 명시된 권한 여러 개를 동시에 명시 가능
    - `TO` 키워드 뒤에는 권한을 부여할 대상 사용자 명시
    - `ON` 키워드 뒤에는 어떤 DB의 어떤 오브젝트에 권한 부여할지 결정 -> 권한 범위에 따라 사용 방법 달라짐
      <br>

- 글로벌 권한
    - 글로벌 권한은 특정 DB나 테이블에 부여될 수 없음 -> 항상 `*.*` 사용
```
mysql> GRANT SUPER ON *.* TO 'user'@'localhost';
```
- DB 권한

    - DB 권한은 특정 DB에 대해서만 권한을 부여하거나 서버에 존재하는 모든 DB에 대해 권한을 부여할 수 있음. (여기서 DB는 DB 내부에 존재하는 테이블 뿐 아니라 스토어드 프로그램들도 모두 포함하는 말)
    - DB 권한만 부여하는 경우에는 테이블까지 명시하여 권한 부여하는 것이 불가
```
mysql> GRANT EVENT ON *.* TO 'user'@'localhost';
mysql> GRANT EVENT ON employees.* TO 'user'@'localhost';
```
- 테이블 권한

    - 모든 DB에 대해 권한 부여 가능
    - 특정 DB의 오브젝트에 대해서만 권한 부여 가능
    - 특정 DB의 특정 테이블에 대해서만 권한 부여 가능
    - 특정 칼럼에 대해서만 권한을 부여하는 경우에는 `DELETE` 권한 제외하고 부여 가능
    - 컬럼 단위의 권한이 하나라도 설정되면, 나머지 모든 테이블의 모든 칼럼에 대해서도 권한 체크를 하기 때문에 전체적인 성능에 영향을 미침
    - 컬럼 단위의 접근 권한이 꼭 필요하다면, 테이블에서 권한을 허용하고자 하는 칼럼만으로 VIEW를 만들어 사용하는 방법 고료 필요
```
mysql> GRANT SELECT,INSERT,UPDATE,DELETE ON *.* TO 'user'@'localhost';
mysql> GRANT SELECT,INSERT,UPDATE,DELETE ON employees.* TO 'user'@'localhost';
mysql> GRANT SELECT,INSERT,UPDATE,DELETE ON employees.department TO 'user'@'localhost';
```


## 3.5 역할(Role)
- MySQL 8.0 버전 부터는 권한을 묶어서 역할(Role)을 사용할 수있게 됨.

- 빈 껍데기만 있는 권한 부여
  ```
  mysql> CREATE ROLE
          role_emp_read,
          role_emp_write;
  ```

- 실질적인 권한 부여
  ```
  mysql> GRANT SELECT ON emplyees.* TO role_emp_read;
  mysql> GRANT INSERT, UPDATE, DELETE on employees.* TO role_emp_write;
  ```

- 계정에 역할 부여
  ```
  mysql> CREATE USER reader@'127.0.0.1' IDENTIFIED BY 'qwerty';
   mysql> CREATE USER writer@'127.0.0.1' IDENTIFIED BY 'qwerty';
   mysql> GRANT role_emp_read TO reader@'127.0.0.1';
   mysql> GRANT role_emp_read, role_emp_write TO writer@'127.0.0.1';
  ```

- 역할 활성화
  ```
  mysql> SET ROLE 'role_emp_read'
  ```

- MySQL 서버에 로그인할 . 때역할을 자동으로 활성화
  ```
  mysql> SET GLOBAL active_all_roles_on_login=ON;
  ```

- 계정과 역할은 내부적으로 똑같은 객체
    - `CREATE USER`과 `CREATE ROLE`을 구분해서 지원하는 이유는 데이터베이스 관리의 직무를 분리할 수있게 하여 보안을 강화하는 용도로 사용할 . 수있게 하기 위함
    - `CREATE USER`에 대해서는 권한이 없지만 `CREATE ROLE` 명령만 실행 가능한 사용자는 역할 생성 가능, 하지만 accout_lock 칼럼 값이 'Y'로 설정되어있어 로그인 용도로는 사용 불가

------------------------

### 참고
[Chapter1 정리](https://velog.io/@hyolim/Real-MySQL-01.-%EC%86%8C%EA%B0%9C)

[Chapter2 정리](https://velog.io/@hyolim/Real-MySQL-02.-%EC%84%A4%EC%B9%98%EC%99%80-%EC%84%A4%EC%A0%95)

[Chapter3 정리](https://velog.io/@hyolim/Real-MySQL-03.-%EC%82%AC%EC%9A%A9%EC%9E%90-%EB%B0%8F-%EA%B6%8C%ED%95%9C)
