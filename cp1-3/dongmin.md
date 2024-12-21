# Real-MySQL을 읽고 난 후..

## 목차
1. [데이터베이스 선택의 기준](#데이터베이스-선택의-기준)
2. [MySQL 서버 설정](#MySQL-서버-설정)
3. [MySQL 8.0 업그레이드](#MySQL-8.0-업그레이드)
4. [사용자 계정 관리](#사용자-계정-관리)
5. [권한과 역할](#권한과-역할)

## 데이터베이스 선택의 기준

### NoSQL vs RDBMS
- NoSQL은 특정 유스케이스에 적합한 DBMS
- RDBMS는 범용 DBMS로 대부분의 서비스에서 기본으로 선택
- 대부분의 회사는 처음에 범용 RDBMS를 선택하고, 필요에 따라 일부 도메인을 NoSQL로 이전

### ORM 사용 시 주의점
- ORM은 DB와의 상호작용을 블랙박스로 만들 수 있음
- ORM 생성 쿼리가 항상 최적의 성능을 보장하지는 않음
- 유스케이스별로 ORM 생성 쿼리 검토 필요

### MySQL 선택 이유
1. 안정성
2. 성능과 기능
3. 커뮤니티 지원
- MySQL과 오라클을 비교해봤을 때 MySQL을 사용하는 이유는 가격일 것임.
- 어떤 DBMS가 좋은지는 자기가 가장 잘 활용할 수 있는 DBMS가 가장 좋은 DBMS이다.
- 안정성 → 성능과 기능 → 커뮤니티나 인지도 순서로 고려


## MySQL 서버 설정

### 서버 접속 방법
```bash
# 소켓 파일 사용
mysql -u root -p --host=localhost --socket=/tmp/mysql.sock

# TCP/IP 사용
mysql -u root -p --host=127.0.0.1 --port=3306

# 기본 설정 사용
mysql -u root -p
```

- 첫 번째는 MySQL의 소켓 파일을 이용해 접속하는 방법
- 두 번째는 TCP/IP를 통해 127.0.0.1(로컬 호스트)로 접속하는 방법
- 원격 호스트에 있는 MySQL에 접속할 때는 반드시 두 번째 방법 사용
- 세 번째 방법은 호스트가 localhost가 되고 MySQL 서버의 설정 파일의 소켓 파일을 사용


### 설정 파일
- 유닉스 계열: my.cnf
- 윈도우 계열: my.ini
- MySQL 8.0 서버는 약 570개의 시스템 변수 보유

## MySQL 8.0 업그레이드

### 업그레이드 방식
1. **인플레이스 업그레이드**
    - 데이터 파일 유지
    - 시간 단축
    - 제약사항 존재

2. **논리적 업그레이드**
    - mysqldump 도구 사용
    - 시간 소요 큼
    - 제약사항 적음

- 마이너 버전의 업그레이드는 대부분 데이터 파일의 변경 없이 진행되며, 많은 경우 여러 버전을 건너뛰어서 업그레이드하는 것이 허용.
ex) MySQL 8.0.16 → MySQL 8.0.21 : MySQL 서버 프로그램만 재설치하면됨.

- 메이저 버전 간 업그레이드는 대부분 크고 작은 데이터 파일의 변경이 필요하기 때문에 반드시 직전 버전에서만 업그레이드가 허용.
ex) MySQL 5.5 → MySQL 5.6 가능, 
MySQL 5.5 → MySQL 5.7, MySQL 8.0 은 불가능

- 메이저 버전 업그레이드는 데이터 파일의 패치가 필요해서 MySQL 8.0 서버 프로그램은 직전 메이저 버전인 MySQL 5.7 버전에서 사용하던 데이터 파일과 로그 포맷만 인식하도록 구현되기 때문.

### 업그레이드 시 고려사항
1. **인증 방식 변경**
    - Caching SHA-2 Authentication이 기본
    - 이전 버전과 호환성 확인 필요

2. **호환성 체크**
    - mysqlcheck 유틸리티 사용
    - FRM 파일 손상 여부 확인
    - 데이터 타입 호환성 검증

3. **제약사항**
    - 외래키 이름 64글자 제한
    - GROUP BY 절 정렬 옵션 변경
    - 파티션 테이블스페이스 제약

### 업그레이드 프로세스
1. **데이터 딕셔너리 업그레이드**
    - FRM 파일에서 InnoDB 테이블로 이전
    - 트랜잭션 지원 가능

2. **서버 업그레이드**
    - 시스템 테이블 구조 변경
    - MySQL 8.0 버전에 맞게 조정

## 사용자 계정 관리

### 계정의 구성
- 사용자 계정과 접속 지점으로 구성
```sql
'svc_id'@'127.0.0.1'
'svc_id'@'192.168.0.10'
'svc_id'@'%'
```

### 시스템 계정과 일반 계정
1. **시스템 계정**
    - SYSTEM_USER 권한 보유
    - 서버 관리자용 계정
    - 계정 관리 가능
    - 다른 세션 제어 가능

2. **일반 계정**
    - 응용 프로그램이나 개발자용
    - 제한된 권한

### 내장 계정
```sql
'mysql.sys'@'localhost'      -- sys 스키마용
'mysql.session'@'localhost'  -- 플러그인용
'mysql.infoschema'@'localhost' -- information_schema용
```

### 계정 생성
```sql
CREATE USER 'user'@'%'
IDENTIFIED WITH 'mysql_native_password' BY 'password'
REQUIRE NONE
PASSWORD EXPIRE INTERVAL 30 DAY
ACCOUNT UNLOCK
PASSWORD HISTORY DEFAULT
PASSWORD REUSE INTERVAL DEFAULT
PASSWORD REQUIRE CURRENT DEFAULT;
```

### 비밀번호 관리
1. **정책 레벨**
    - LOW: 길이만 검증
    - MEDIUM: 길이, 문자 조합 검증
    - STRONG: MEDIUM + 금칙어 검증

2. **이중 비밀번호**
    - Primary Password
    - Secondary Password
    - 무중단 비밀번호 변경 가능

## 3. 사용자 및 권한

사용자의 계정 뿐만 아니라 접속 지점(클라이언트가 실행된 호스트명이나 도메인 또는 IP 주소)도 계정의 일부가 됨.

```tsx
'svc_id'@'127.0.0.1'
'svc_id'@'192.168.0.10'
'svc_id'@'%'
```

`192.168.0.10` 으로 접속 시 MySQL은 가장 범위가 좁은 것을 선택. 따라서 두 번째 선택

### 시스템 계정과 일반 계정

SYSTE_USER 권한을 가지고 있느냐에 따라 시스템 계정과 일반 계정으로 구분.

시스템 계정: 데이터베이스 서버 관리자를 위한 계정

일반 계정: 응용 프로그램이나 개발자를 위한 계정

다음은 시스템 계정으로만 수행 가능

- 계정 관리(계정 생성 및 삭제, 계정 권한 부여 및 제거)
- 다른 세션(Connection) 또는 그 세션에서 실행 중인 쿼리를 강제 종료
- 스토어드 프로그램 생성 시 DEFINER를 타 사용자로 설정

MySQL 서버에는 다음과 같이 내장된 계정들이 있는데, 각기 다른 목적이므로 삭제되지 않게 주의

- ‘mysql.sys’@’localhost’: MySQL 8.0부터 기본으로 내장된 sys 스키마의 객체들의 DEFINER로 사용되는 계정
- ‘mysql.session’ @ ‘localhost’: MySQL 플러그인이 서버로 접근할 때 사용되는 계정
- ‘mysql.infoschema’ @ ‘localhost’: information_schema에 정의된 뷰의 DEFINER로 사용되는 계정


### 계정 생성

```tsx
CREATE USER 'user'@'%'
							IDENTIFIED WITH 'mysql_native_password' BY 'password'
							REQUIRE NONE
							PASSWORD EXPIRE INTERVAL 30 DAY
							ACCOUNT UNLOCK
							PASSWORD HISTORY DEFAULT
							PASSWORD REUSE INTERVAL DEFAULT
							PASSWORD REQUIRE CURRENT DEAFULT;
```

- 계정의 인증 방식과 비밀번호
- 비밀번호 관련 옵션(유효 기간, 이력 개수, 재사용 불가 시간)
- 기본 역할
- SSL 옵션
- 계정 잠금 여부

- IDENTIFIED WITH
    - 사용자의 인증 방식과 비밀번호를 설정
    - Caching SHA-2 Authentication
- REQUIRE
    - MySQL 서버에 접속할 때 암호화된 SSL/TLS 채널을 사용할지 여부 설정.
- PASSWORD EXPIRE
    - 비밀번호의 유효 기간을 설정
    - 명시하지 않으면 default_password_lifetime 시스템 변수에 저장된 기간으로 설정.
- PASSWORD HISTORY
    - 한 번 사용했던 비밀번호를 재사용하지 못하게 설정하는 옵션
    - password_history테이블 사용해 이전 비밀번호 기억
- PASSWORD REUSE INTERVAL
    - 한 번 사용했던 비밀번호의 재사용 금지 기간 설정
- PASSWORD REQUIRE
    - 비밀번호가 만료되어 새로운 비밀번호로 변경할 때 현재 비밀번호를 필요로 할지 말지를 결정.
- ACCOUNT LOCK/UNLOCK
    - 계정 생성 시 또는 ALTER USER 명령을 사용해 계정 정보를 변경할 때 계정을 사용핮 못하게 잠글지 여부 결정

### 비밀번호 관리

글자의 조합을 강제하거나 금칙어 설정 기능도 있음.


비밀 번호 정책은 다음 3가지 중 하나로 default는 MEDIUM

- LOW: 비밀번호의 길이만 검증
- MEDIUM: 비밀번호의 길이를 검증하며, 숫자와 대소문자, 그리고 특수문자의 배합을 검증
- STRONG: MEDIUM 레벨의 검증을 모두 수행하며, 금칙어가 포함됐는지 여부까지 검증

### 이중 비밀번호

MySQL 8.0 버전부터는 계정의 비밀번호로 2개의 값을 동시에 사용할 수 있는 기능 추가

최근에 설정한 비밀번호가 프라이머리 비밀번호, 이전 비밀번호가 세컨더리 비밀번호


첫 번째 명령 실행 시 프라이머리 비밀번호는 ‘koh0714!’로 설정, 세컨더리 비밀번호는 빈 상태

두 번째 명령 실행 시 세컨더리가 ‘koh0714!’, 프라이머리가 ‘new0714!’로 설정

두 비밀번호 중에 아무거나 입력 시 로그인 됨.

세컨더리 비밀번호는 삭제하는게 좋음.

### 권한

데이터베이스나 테이블 이외의 객체제 적용되는 권한을 글로벌 권한

데이터베이스나 테이블을 제어하는 데 필요한 권한을 객체 권한.

- 객체 권한은 GRANT 명령으로 권한을 부여할 때 반드시 특정 객체 명시
- 글로벌 권한은 명시하지 말아야함.

MySQL 8.0부터 동적권한이 추가됨.

서버가 생성되면서 동적으로 생성되는 권한.

- 사용자에게 권한 부여시 GRANT 명령 사용

```bash
GRANT privilege_list ON db.table TO 'user'@'host';
```

- 글로벌 권한은 on절에 항상 **.** 을 사용

```bash
GRANT SUPER ON *.* TO 'user'@'localhost';
```

- DB권한
- employees.dept 와 같이 테이블까지 명시할 수는 없음.

```bash
GRANT EVENT ON employees.* TO 'user'@'localhost';
GRANT EVENT ON *.* TO 'user'@'localhost';
```


- 테이블 권한은 모두 가능

```bash
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.* TO 'user'@'localhost';
GRANT SELECT, INSERT, UPDATE, DELETE ON employees.dept TO 'user'@'localhost';
```

- 칼럼 단위의 권한은 잘 사용하지 않음.
- 칼럼 단위의 권한인 하나라도 설정되면 나머지 모든 테이블의 모든 칼럼에 대해서도 권한 체크를 해서 전체적인 성능에 영향 미칠 수 있음.

### 역할

- MySQL 8.0 버전부터 권한을 묶어서 역할을 사용할 수 있음.


- 역할 생성 후 특정 권한을 부여가능

- 사용자를 생성하여 역할을 부여 가능


- 실제 역할이 적용될려면 역할을 활성화시켜줘야함.

- 다시 로그인하면 활성화되지 않은 상태로 초기화되버림.


- user테이블에는 계정과 권한이 모두 저장.

- 하나의 계정에 다른 계정의 권한을 병합하기만 하면 되므로 MySQL 서버는 역할과 계정을 구분할 필요가 없다.

- MySQL 서버 내부적으로 계정과 역할은 아무런 차이가 없으며, 실제 관리자나 사용자가 볼 때도 역할인지 계정인지 구분하기가 어렵다. 역할과 계정을 정확히 구분하고자 한다면 데이터베이스 관리자가 식별할 수 있는 프리픽스나 키워드를 추가해 이름 선택