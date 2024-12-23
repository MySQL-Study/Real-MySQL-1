# 목차

### [Chapter 1 - 소개](#chapter1---소개)

-   [1. MySQL의 역사적 발전과정](#1-mysql의-역사적-발전과정)
-   [2. 라이선스 정책의 변화](#2-라이선스-정책의-변화)
-   [3. MySQL의 경쟁력](#3-mysql의-경쟁력)
-   [4. DBMS 선택 기준](#4-dbms-선택-기준)

### [Chapter 2 - 설치와 설정](#chapter2---설치와-설정)

-   [1. MySQL 서버 설치](#1-mysql-서버-설치)
-   [2. MySQL 서버의 시작과 종료](#2-mysql-서버의-시작과-종료)

### [Chapter 3 - 사용자 및 권한](#chapter3---사용자-및-권한)

-   [1. 사용자 식별](#1-사용자-식별)
-   [2. 사용자 계정 관리](#2-사용자-계정-관리)
-   [3. 비밀번호 관리](#3-비밀번호-관리)
-   [4. 권한](#4-권한-privilege)
-   [5. 역할](#5-역할-role)

---

# Chapter1 - 소개

## 1. MySQL의 역사적 발전과정

MySQL은 1979년 스웨덴 TcX사의 UNIREG로 시작되었다. 1994년 MySQL 1.0이 완성되었으나 초기에는 사내에서만 사용되다가 1996년에 일반 공개되었다. 2000년 MySQL AB 회사 설립과 함께 FPL 라이선스를 도입했으며, 이후 썬마이크로시스템즈를 거쳐 오라클에 인수되었다.

오라클 인수 이후 주목할 만한 변화:

-   소스코드 레벨의 리팩토링 진행
-   MySQL 5.5~5.7: 안정성과 성능 개선에 집중
-   MySQL 8.0: 상용 DBMS 수준의 기능 도입

## 2. 라이선스 정책의 변화

현재 MySQL은 두 가지 에디션으로 운영된다:

-   MySQL 엔터프라이즈 에디션
-   MySQL 커뮤니티 에디션

주요 변화점:

-   MySQL 5.5 이전: 두 에디션 간 소스코드 동일
-   MySQL 5.5 GA 버전 이후: 엔터프라이즈 에디션 소스코드 비공개 전환
-   커뮤니티 에디션: 지속적인 오픈소스 정책 유지

## 3. MySQL의 경쟁력

비용 효율성

대규모 데이터 처리에서 상용 DBMS 대비 현저한 비용 우위를 보인다. 페이스북의 한 DBA는 "페이스북이 가진 데이터를 모두 오라클 RDBMS에 저장하면 페이스북은 망할 것이다"라고 말했다. 이는 대규모 데이터 처리에서 MySQL의 비용 효율성이 얼마나 중요한 경쟁력인지를 단적으로 보여준다.

## 4. DBMS 선택 기준

우선순위:

1. 안정성: 비용으로 해결할 수 없는 핵심 요소
2. 성능과 기능: 실무 적용성
3. 커뮤니티/인지도: 기술 지원 및 인력 수급

# Chapter2 - 설치와 설정

## 1. MySQL 서버 설치

### 1.1 버전 선택

-   최신 버전 설치 권장 (다른 제약사항이 없다면)
-   메이저 버전 업그레이드 시: 패치 버전 15~20번 이상 릴리스된 버전 선택
    -   예: MySQL 8.0의 경우 8.0.15 ~ 8.0.20

### 1.2 에디션 : 엔터프라이즈 vs 커뮤니티

-   MySQL 5.5 이전: 기술 지원 차이만 존재
-   MySQL 5.5 이후: 기능 및 소스코드 차이 발생
-   엔터프라이즈 에디션 추가 기능:
    -   Thread Pool
    -   Enterprise Audit
    -   Enterprise TDE
    -   Enterprise Authentication
    -   Enterprise Firewall
    -   Enterprise Monitor
    -   Enterprise Backup
    -   기술 지원

저자는 Percona의 백업 및 모니터링 도구, 플러그인을 활용하면 MySQL 커뮤니티 에디션의 부족한 부분을 메꿀 수 있었기에 엔터프라이즈 에디션의 필요성을 느끼지 못했다고 한다.

MySQL 엔터프라이즈 에디션과 커뮤니티 에디션의 기본 성능이 다르거나 한 것은 아니므로 엔터프라이즈 에디션에서 지원하는 것들이 꼭 필요한지 검토하는 게 좋다.

### 1.3 MySQL 설치하기

#### 1.3.1 MySQL Server vs Client

-   Server(mysqld): 데이터베이스 엔진으로, 실제 데이터를 저장하고 처리하는 서버 프로그램
-   Client(mysql): 서버에 접속해서 쿼리를 실행하고 결과를 조회할 수 있는 프로그램

각각 독립적으로 실행되며, 네트워크를 통해 통신한다!

#### 1.3.2 윈도우 설치 시 주요 구성 요소

-   MySQL Server: 데이터를 저장하고 처리하는 핵심 데이터베이스 엔진이다. 사용자 관리, 쿼리 처리, 백업 등 기본적인 데이터베이스 기능을 모두 담당한다.
-   MySQL Shell: JavaScript, Python, SQL을 지원하는 고급 명령행 도구이다. SQL 실행, 서버 관리, 스크립트 작성 등이 가능하다.
-   MySQL Router: 여러 MySQL 서버를 관리하기 위한 중간 관리자 프로그램이다. 서버 간 요청 분배, 장애 대응, 부하 분산 등을 처리한다.

#### 1.3.3 MySQL 서버 디렉토리 구조

##### 1.3.3.1 MySQL 설정 파일(my.ini)

MySQL 서버의 설정 파일은 `my.ini`로, MySQL 서버의 핵심 설정 정보를 담고 있다. Windows 환경에서 `ls` 명령어로 확인해보면 다음과 같이 위치해 있다:

```powershell
PS C:\ProgramData\MySQL\MySQL Server 8.0> ls

    Directory: C:\ProgramData\MySQL\MySQL Server 8.0

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d----        2024-12-17 오후 12:56                Data
d----        2023-09-18 오전 11:19                Uploads
-a---        2023-09-18 오전 11:19            487 installer_config.xml
-a---        2023-09-18 오전 11:19          15641 my.ini
```

##### 1.3.3.2 MySQL 서버 디렉터리 구조

MySQL 서버는 다음과 같은 기본 디렉터리 구조를 가진다:

-   bin: MySQL 서버/클라이언트 프로그램, 유틸리티가 저장된 디렉터리
-   include: C/C++ 헤더 파일들이 저장된 디렉터리
-   lib: 라이브러리 파일들이 저장된 디렉터리
-   share: 다양한 지원 파일들이 저장되어 있으며, 에러 메시지나 샘플 설정 파일(my.ini)이 있는 디렉터리

이러한 디렉터리들이 삭제되면 MySQL 서버가 정상적으로 실행되지 않을 수 있으니 주의가 필요하다.

## 2. MySQL 서버의 시작과 종료

### 2.1 시작/종료/상태 명령어

-   `systemctl start mysqld`
-   `systemctl stop mysqld`
-   `systemctl status mysqld`

### 2.2 서버 셧다운

#### 2.2.1 기본 셧다운 방법

-   원격으로 MySQL 서버에 로그인한 후 `SHUTDOWN` 명령어 실행
-   `SHUTDOWN` 권한이 필요하다.
-   `mysql> SHUTDOWN;`

#### 2.2.2 클린 셧다운 (Clean shutdown)

-   모든 커밋된 데이터를 데이터 파일에 적용 후 종료하는 방식이다.
-   다음 서버 시작 시 트랜잭션 복구 과정이 불필요해 빠르게 시작할 수 있다.

```shell
mysql> SET GLOBAL innodb_fast_shutdown=0;
linux> systemctl stop mysqld.service

## 또는 원격으로 MySQL 서버 종료 시
mysql> SET GLOBAL innodb_fast_shutdown=0;
mysql> SHUTDOWN;
```

#### 2.2.3 시작/종료 시 버퍼 풀 처리

-   종료 시: 버퍼 풀의 메타 정보만 백업 (빠름)
-   시작 시: 데이터 파일을 다시 읽어 버퍼 풀 복구 (느림)
-   서버 시작이 느린 경우 버퍼 풀 복구 상태를 확인해볼 것

# Chapter3 - 사용자 및 권한

## 1. 사용자 식별

MySQL의 사용자 식별은 두 가지 요소를 조합하여 이루어진다 :

-   사용자 계정명 (아이디)
-   접속 지점 (호스트명, 도메인, IP주소)
    예시: `'svc_id'@'127.0.0.1'`

특징:

-   모든 외부 접속을 허용하려면 호스트 부분을 `'%'`로 설정 (`'svc_id'@'%'`)
-   MySQL은 동일한 계정명이 있을 경우 항상 범위가 더 작은(구체적인) 계정을 선택함

## 2. 사용자 계정 관리

MySQL 8.0부터 계정은 두 가지로 구분된다:

### 2.1 계정 종류

#### 2.1.1 시스템 계정

-   SYSTEM_USER 권한 보유
-   데이터베이스 관리자용 계정
-   주요 권한:
    -   계정 관리(생성, 삭제, 권한 부여)
    -   다른 세션 강제 종료
    -   스토어드 프로그램의 DEFINER 설정

#### 2.1.2 일반 계정

-   응용 프로그램이나 개발자용 계정
-   시스템 계정 관리 불가

### 2.2 내장 계정

MySQL 서버의 주요 내장 계정:

-   'mysql.sys'@'localhost': sys 스키마 객체용
-   'mysql.session'@'localhost': MySQL 플러그인용
-   'mysql.infoschema'@'localhost': information_schema용

```SQL
mysql> SELECT user,host,account_locked FROM mysql.user WHERE user LIKE 'mysql.%';
+------------------+-----------+----------------+
| user             | host      | account_locked |
+------------------+-----------+----------------+
| mysql.infoschema | localhost | Y              |
| mysql.session    | localhost | Y              |
| mysql.sys        | localhost | Y              |
+------------------+-----------+----------------+
3 rows in set (0.01 sec)
```

(삭제되지 않도록 주의해야한다!)

### 2.3 계정 생성과 설정

MySQL 8.0부터 계정 생성과 권한 부여가 분리됐다:

-   계정 생성: CREATE USER
-   권한 부여: GRANT

주요 인증 방식

-   Native Authentication: SHA-1 해시 알고리즘 사용
-   Caching SHA-2 Authentication: MySQL 8.0 기본값, SSL/TLS 필요
-   PAM Authentication: 외부 인증 (Enterprise Edition)
-   LDAP Authentication: LDAP 인증 (Enterprise Edition)

주요 설정 옵션:

-   IDENTIFIED WITH: 인증 방식 설정
-   REQUIRE: SSL/TLS 사용 여부
-   PASSWORD EXPIRE: 비밀번호 만료 기간
-   PASSWORD HISTORY: 이전 비밀번호 재사용 제한
-   PASSWORD REUSE: 재사용 금지 기간
-   PASSWORD REQUIRE: 변경 시 현재 비밀번호 필요 여부
-   ACCOUNT LOCK/UNLOCK: 계정 잠금 관리

## 3. 비밀번호 관리

### 3.1 MySQL의 컴포넌트 개념?

-   MySQL 8.0에서 도입된 새로운 플러그인 아키텍처
-   서버의 기능을 확장하기 위한 모듈화된 구조
-   기존 플러그인의 한계를 극복하기 위해 설계됨

### 3.2 validate_password 컴포넌트

-   비밀번호 정책 관리용 컴포넌트
-   설치: `INSTALL COMPONENT 'file://component_validate_password'`
-   정책 수준: LOW(길이만), MEDIUM(문자조합), STRONG(금칙어)

### 3.3 이중 비밀번호(Dual Password) 시스템

#### 3.3.1 일반적인 비밀번호 변경의 문제

-   여러 애플리케이션이 하나의 DB 계정을 공유
-   비밀번호 변경 시 모든 애플리케이션의 설정도 변경 필요
-   모든 애플리케이션을 동시에 재시작해야 함 -> 서비스 중단 발생

#### 3.3.2 이중 비밀번호(Dual Password)란?

-   두 개의 비밀번호 동시 사용 가능
-   무중단 비밀번호 변경 지원
-   PRIMARY(신규), SECONDARY(이전) 구조

```SQL
-- 새 비밀번호 설정하면서 기존 비밀번호 유지
ALTER USER 'user'@'host' IDENTIFIED BY 'new_password'
RETAIN CURRENT PASSWORD;

-- 이전 비밀번호 삭제
ALTER USER 'user'@'host' DISCARD OLD PASSWORD;
```

#### 3.3.3 무중단으로 비밀번호 변경하는 구체적인 방법?

1. 새 비밀번호 추가 (이전 비밀번호는 그대로 유지)

    ```SQL
    ALTER USER 'user'@'host'
    IDENTIFIED BY 'new_password'
    RETAIN CURRENT PASSWORD;
    ```

2. 이 시점부터:
    - 이전 비밀번호로도 로그인 가능
    - 새 비밀번호로도 로그인 가능
3. 순차적으로 애플리케이션 설정 변경 및 재시작
    - 각 애플리케이션을 하나씩 새 비밀번호로 변경
    - 서비스 중단없이 점진적 변경 가능
4. 모든 변경이 완료되면 이전 비밀번호 제거

    ```SQL
    ALTER USER 'user'@'host'
    DISCARD OLD PASSWORD;
    ```

## 4. 권한 (Privilege)

### 4.1 권한 종류

#### 4.1.1 정적 권한

정적 권한은 MySQL이 처음부터 가지고 있던 전통적인 권한 체계다. 예를 들어, 데이터베이스 관리자가 새로운 개발자에게 특정 데이터베이스만 접근할 수 있게 하려면 다음과 같이 설정할 수 있다:

```SQL
GRANT SELECT, INSERT ON employees.* TO 'developer'@'localhost';
```

이렇게 하면 개발자는 'employees' 데이터베이스의 모든 테이블에 대해 조회와 입력만 가능하다.

정적 권한은 MySQL 서버의 소스코드에 고정적으로 명시돼 있는 권한을 의미한다.

정적 권한의 종류는 다음과 같다:

-   글로벌 권한: 서버 전체 대상 (FILE, CREATE USER 등)
-   객체 권한: DB/테이블 대상 (SELECT, INSERT 등)

#### 4.1.2 동적 권한(MySQL 8.0부터)

MySQL 8.0부터는 더 세분화된 권한 관리가 필요하다는 요구사항을 반영해 동적 권한을 도입했다. 예를 들어, 백업 담당자에게는 백업 관련 권한만 주고 싶다면:

```SQL
GRANT BACKUP_ADMIN ON *.* TO 'backup_user'@'localhost';
```

동적 권한은 MySQL 서버가 시작되면서 동적으로 생성하는 권한이다.

### 4.2 권한 부여 방법

사용자를 먼저 생성하고, GRANT 명령으로 다음과 같이 권한을 부여해야 한다.

```SQL
mysql> GRANT privilege_list ON db.table TO 'user'@'host'
```

#### 4.2.1 글로벌 권한의 적용

서버 전체(테이블 및 스토어드 프로시저, 함수 등을 모두 포함)에 대한 권한은 반드시 `*.*` 형식으로만 부여할 수 있다.

```SQL
GRANT SUPER ON *.* TO 'user'@'localhost';
```

#### 4.2.2 DB 권한의 적용

DB 단위 권한은 `*.*` 또는 `db.*` 형식으로 부여 가능하다.

```sql
-- 특정 DB에만 이벤트 권한
GRANT EVENT ON employees.* TO 'user'@'localhost';

-- 모든 DB에 이벤트 권한
GRANT EVENT ON *.* TO 'user'@'localhost';
```

#### 4.2.3 테이블 권한의 적용

```sql
-- 모든 DB, 모든 테이블에 권한
GRANT SELECT, INSERT ON *.* TO 'user'@'localhost';

-- 특정 DB의 모든 테이블에 권한
GRANT SELECT, INSERT ON employees.* TO 'user'@'localhost';

-- 특정 테이블에만 권한
GRANT SELECT, INSERT ON employees.department TO 'user'@'localhost';
```

#### 4.2.4 칼럼 단위 권한의 적용과 주의사항

```sql
-- dept_name 칼럼만 업데이트 가능하도록
GRANT SELECT, INSERT, UPDATE(dept_name)
ON employees.department TO 'user'@'localhost';
```

주의사항:

-   칼럼 단위 권한은 전체 성능에 영향을 미침
    -   칼럼 단위의 권한이 하나라도 설정되면 나머지 모든 테이블의 모든 칼럼에 대해서도 권한 체크를 하기 때문
-   대안으로 뷰(VIEW) 사용을 권장:

    ```sql
    -- 특정 칼럼만 포함한 뷰 생성
    CREATE VIEW emp_names AS
    SELECT emp_no, first_name, last_name FROM employees;

    -- 뷰에 권한 부여
    GRANT SELECT ON emp_names TO 'user'@'localhost';
    ```

## 5. 역할 (Role)

MySQL 8.0부터 도입된 역할은 권한을 그룹화하여 관리할 수 있게 해준다.

### 5.1 실습

#### 5.1.1 STEP1: 역할&권한 생성 및 부여

```sql
-- 역할 생성
CREATE ROLE 'role_emp_read', 'role_emp_write';

-- 역할에 권한 부여
GRANT SELECT ON employees.* TO 'role_emp_read';
GRANT INSERT, UPDATE, DELETE ON employees.* TO 'role_emp_write';

-- 사용자 생성
CREATE USER 'reader'@'127.0.0.1' IDENTIFIED BY 'qwerty';
CREATE USER 'writer'@'127.0.0.1' IDENTIFIED BY 'qwerty';

-- 사용자에게 역할 부여
GRANT 'role_emp_read' TO 'reader'@'127.0.0.1';
GRANT 'role_emp_read', 'role_emp_write' TO 'writer'@'127.0.0.1';
```

이처럼 역할이 부여되어도 바로 사용할 수 없다는 오류를 만나게 될거다. MySQL 서버에서는 '역할'이 자동으로 활성화되지 않도록 설정돼 있기 때문이다.

활성화 여부 확인법:

```SQL
-- 현재 활성화된 역할 확인
SELECT current_role();
-- 결과: NONE (활성화 전)
```

#### 5.1.2 STEP2: 역할 활성화

-   왜 역할 활성화가 필요한가?
    -   보안을 위해 명시적인 활성화 과정이 필요하다.
    -   사용자가 실수로 권한을 사용하는 것을 방지한다.

역할 활성화 하기:

```SQL
-- 수동 활성화
SET ROLE 'role_emp_read';
```

```SQL
-- 사용자가 MySQL 서버에 로그인할 때 역할을 자동으로 활성화
SET GLOBAL activate_all_roles_on_login = ON;
```

활성화 후 권한 사용:

```sql
-- 활성화 전: 권한 없음
SELECT * FROM employees.employees;
-- ERROR 1142: SELECT command denied

-- 역할 활성화
SET ROLE 'role_emp_read';

-- 활성화 후: 조회 가능
SELECT * FROM employees.employees;
-- 정상 실행
```

### 5.2 '역할'과 '계정'의 관계

흥미로운 점은 MySQL 서버 내부적으로 역할과 계정이 사실상 동일한 객체라는 것이다. MySQL DB의 user 테이블을 살펴보면 이를 확인할 수 있다:

```sql
SELECT user, host, account_locked FROM mysql.user;
+----------------+-------------+----------------+
| user           | host        | account_locked |
+----------------+-------------+----------------+
| role_emp_read  | %           | Y              |
| role_emp_write | %           | Y              |
| reader         | 127.0.0.1   | N              |
| writer         | 127.0.0.1   | N              |
| root           | localhost   | N              |
+----------------+-------------+----------------+
```

유일한 차이점은 `account_locked` 칼럼의 값뿐이다. 역할은 직접 로그인에 사용할 수 없도록 'Y'로 잠겨있고, 계정은 'N'으로 설정되어 있다.

이렇게 설계된 이유는 보안 때문이다. 데이터베이스 관리자는 CREATE ROLE 권한만 가지고 있다면 역할을 생성할 수 있지만, 이 역할로 직접 로그인은 할 수 없다. 반면 CREATE USER 권한이 있어야만 실제 로그인 가능한 계정을 만들 수 있다. 이를 통해 데이터베이스의 권한 관리와 계정 관리를 분리할 수 있다.
