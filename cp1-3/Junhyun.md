# 02. 설치와 설정


## 2.4 서버 설정
일반적으로 MySQL 서버는 단 하나의 설정 파일 사용.(my.cnf , my.ini)
시작될 때 이 설정 파일을 참조.
```shell
$ mysqld --verbose --help
```
또는
```shell
$ mysql --help
```
위 명령어를 통해 설정 파일 참조 순서를 알 수 있다.

서버용 설정 파일은 주로 다음 경로에 위치한 것을 사용.
- /etc/my.cnf
- /etc/mysql/my.cnf


### MySQL 시스템 변수의 특징
```mysql
mysql> SHOW GLOBAL VARIABLES;
```

- **글로벌 변수** : 하나의 서버 전체에 영향을 미치는 변수, 서버 자체 관련 설정이 많다. 
- **세션 변수** : 클라이언트가 MySQL서버에 접속할 때 기본으로 부여하는 옵선의 기본값 제어.
---
# 03. 사용자 및 권한
## 3.1 사용자 식별
아이디와 IP가 계정의 일부
'svc_id'@'%' -> svc_id라는 사용자가 어떤 IP에서든 접속 가능

## 3.2 사용자 계정 관리
- 시스템 계정: SYSTEM_USER 권한, 서버관리자를 위한 계정, 계정 관리 가능
- 일반 계정

MySQL 에서 내장된 계정들 삭제하지 않도록 주의

### 계정 생성
8.0 부터 생성: CREATE USER
권한 부여: GRANT

일반적으로 많이 사용되는 옵션을 가진 CREATE USER 문
```mysql
mysql> CREATE USER 'user'@'%' 
    IDENTIFIED WITH 'mysql_native_password' BY 'password'
    REQUIRE NONE
    PASSWORD EXPIRE INTERVAL 30 DAY
    ACCOUNT UNLOCK
    PASSWORD HISTORY DEFAULT
    PASSWORD REUSE INTERVAL DEFAULT
    PASSWORD REQUIRE CURRENT DEFAULT;
```

### IDENTIFIED WITH
뒤에 by '~'를 붙여 인증 방식을 명시한다.
### REQUIRE
서버에 접속할 때 암호화된 SSL/TLS 사용할지 여부를 설정한다.
### PASSWORD EXPIRE INTERVAL
default -> default_password_lifetime 시스템 변수에 설정된 값
interval 30 day -> 30일 후 암호 만료
never -> 암호 만료 없음
### PASSWORD HISTORY
mysql DB의 password_history 테이블에 비밀번호 기억해 재사용을 금지함.
### PASSWORD REUSE INTERVAL
재사용 금지 기간 설정
default -> password_reuse_interval 시스템 변수에 설정된 값
### PASSWORD REQUIRE
비밀번호 변경할때 현재 비밀번호 필요로 할지 말지
current -> 현재 비밀번호 필요
optional -> 현재 비밀번호 필요 없음
default -> password_require_current 시스템 변수에 설정된 값
### ACCOUNT LOCK/UNLOCK
LOCK -> 계정을 사용하지 못하게 잠금
UNLOCK -> 계정 잠금 해제
## 3.3 비밀번호 관리
### 고수준 비밀번호
```mysql
mysql> INSTALL COMPONENT 'file://component_validate_password';
```
내장된 validate_password 컴포넌트 설치
```mysql
mysql> SHOW GLOBAL VARIABLES LIKE 'validate_password%';
```
컴포넌트에서 제공하는 시스템변수 확인

비밀번호 정책
- LOW: 길이만 검증
- MEDIUM: 길이, 숫자, 대소문자, 특수문자 포함
- STRONG: MEDIUM + 금칙어가 포함됐는지 여부
금칙어는 validate_password.dictionary_file 시스템 변수에 설정된 사전 파일에 명시된 단어를 포함하고 있는지 검증
```mysql
mysql> SET GLOBAL validate_password.dictionary_file = 'prohibited_words.data';
mysql> SET GLOBAL validate_password.policy = 'STRONG';
```

### 이중 비밀번호
2개중 하나만 일치하면 로그인이 통과됨.
primary, secondary 비밀번호를 설정
```mysql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password' RETAIN CURRENT PASSWORD;
```
이렇게 새로운 비밀번호를 primary로 설정하고 과거를 secondary 설정
데이터베이스에 연결하는 응용 프로그램의 소스코드나 설정파일의 비밀번호를 새로운 비밀번호로 변경하고 배포 및 재시작
모든 응용프로그램이 재시작 완료되면 보안을 위해 secondary 비밀번호를 삭제
```mysql
mysql> ALTER USER 'root'@'localhost' DISCARD OLD PASSWORD;
```

## 3.4 권한
- 글로벌 권한
  - 데이터베이스나 테이블 이외의 객체에 적용되는 권한
  - GRANT 명시하면 안 됨.
    - ALL(또는 PRIVILEGES) : 글로벌 수준에서 가능한 모든 권한 부여
- 객체 단위 권한
  - 데이터베이스나 테이블을 제어하는 데 필요한 권한
  - GRANT 명령으로 권한을 부여할 때 반드시 특정 객체를 명시
    - ALL(또는 PRIVILEGES) : 해당 객체에 적용될 수 있는 모든 객체 권한을 부여
- 동적 권한 (8.0에서 추가, 위에는 정적 권한)
  - 서버가 시작되며 동적으로 생성하는 권한(컴포넌트나 플로그인이 설치되면 그때 등록되는 권한)

GRANT 명령어를 통해 권한 부여
사용자를 먼저 생성하고 권한을 부여해야 한다.
ON 뒤에는 어떤 오브젝트(DB, TABLE)
TO 뒤에는 어떤 사용자

글로벌 권한
```mysql
mysql> GRANT SUPER ON *.* TO 'user'@'localhost';
```
특정해서 부여할 수 없기 때문에 *.*으로 모든 오브젝트에 대해 부여

DB권한


테이블 권한


## 3.5 역할
빈껍데기 역할 정의
```mysql
mysql> CREATE ROLE
       role_emp_read,
       role_emp_write;
```
권한 부여
```mysql
mysql> GRANT SELECT ON employees.* TO role_emp_read;
mysql> GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
```

