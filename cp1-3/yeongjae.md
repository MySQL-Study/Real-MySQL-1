# 03 사용자 및 권한
## **3.1 사용자 식별**

- MySQL은 계정과 접속지점을 계정의 일부로 간주함
    - ‘svc_id’@’127.0.0.1’
- 모든 접속 지점 허용은 ‘%’ 를 통해 가능
    - ‘svc_id’@’192.168.0.10’
    - ‘svc_id’@’%’
    - 이렇게 2가지가 있다면 192.168.0.10의 위치에서 접속 시 더 확실한 곳의 비밀번호를 받아야함

## **3.2 사용자 계정 관리**

### **3.2.1 시스템 계정과 일반 계정**

- SYSTEM_USER 권한 여부에 따라 시스템 계정과 일반 계정으로 구분

### **3.2.2 계정 생성**

- 5.7까지는 GRANT명령으로 권한 부여와 계정 생성이 가능했지만 8.0부터는 분리

```sql
CREATE USER 'user'@'%'
	IDENTIFIED WITH 'mysql_native_password' BY 'password'
	REQUIRE NONE
	PASSWORD EXPIRE INTERVAL 30 DAY
	ACCOUNT UNLOCK
	PASSWORD HISOTRY DEFAULT
	PASSWORD REUSE INTERVAL DEFAULT
	PSSWORD REQUIRE CURRENT DEFAULT;

```

**1.1 CREATE USER 'user'@'%'**

- MySQL에 'user'@'%'라는 계정을 생성.
- '%'는 **호스트 와일드카드**로, 모든 호스트(즉, 모든 IP 주소)에서 해당 계정으로 접근할 수 있음을 의미.

**1.2 IDENTIFIED WITH 'mysql_native_password' BY 'password'**

- IDENTIFIED WITH:
- 계정이 사용할 인증 플러그인을 지정.
- **mysql_native_password**는 MySQL에서 오래전부터 사용되던 기본 인증 플러그인으로, 비밀번호 기반 인증을 수행.
    - 최근 MySQL 8.0에서는 기본 플러그인이 **caching_sha2_password**로 변경.
- BY 'password':
    - 계정의 초기 비밀번호를 설정.
    - 위 예에서는 비밀번호를 password로 설정.

**1.3 REQUIRE NONE**

- 사용자 연결 시 보안 요구 사항을 설정.
- **NONE**은 사용자가 **SSL/TLS 암호화를 요구하지 않음을 의미**.
- 추가적으로 사용할 수 있는 값:
- REQUIRE SSL: SSL/TLS를 사용해야만 연결 가능.
- REQUIRE X509: 클라이언트가 인증서로 인증해야 함.
- REQUIRE ISSUER : 특정 인증서를 발급한 기관만 허용.

**1.4 PASSWORD EXPIRE INTERVAL 30 DAY**

- 계정의 **비밀번호 만료 정책**을 설정.
- 이 경우, 비밀번호는 **30일 후 만료**되며, 사용자는 이후 새로운 비밀번호로 변경해야 함.

**1.5 ACCOUNT UNLOCK**

- 계정을 활성화(언락) 상태로 설정.
- MySQL에서는 **LOCK**된 계정은 사용자가 로그인할 수 없음.
- ACCOUNT UNLOCK은 계정을 활성화하여 로그인 가능하게 만듬.

**1.6 PASSWORD HISTORY DEFAULT**

- 계정의 **비밀번호 기록 개수**를 설정.
- 사용자가 **이전에 사용한 비밀번호를 재사용하지 못하도록 제한**.
- **DEFAULT**는 MySQL 서버의 전역 설정(예:default_password_lifetime)을 따름.

**1.7 PASSWORD REUSE INTERVAL DEFAULT**

- **비밀번호 재사용 제한 주기**를 설정.
- 사용자가 **이전에 사용한 비밀번호를 다시 사용할 수 있는 기간**을 지정.
- **DEFAULT**는 MySQL 서버의 전역 설정을 따름.

**1.8 PASSWORD REQUIRE CURRENT DEFAULT**

- 비밀번호 변경 시, **현재 비밀번호 입력 여부**를 설정.
- **DEFAULT**는 MySQL 서버의 전역 설정을 따름.
- 만약 활성화되어 있으면, 사용자가 비밀번호를 변경할 때 **현재 비밀번호를 입력**해야 함.
- 보안 강화를 위해 이 옵션을 사용하는 것이 좋음.
- CURRENT 대신 OPTIONAL도 가능(입력하지 않아도 되게)
- 명시하지 않거나 DEFAULT로 명시하면 password_require_current 시스템 변수 값으로 설정

<img width="405" alt="스크린샷 2024-12-23 00 37 01" src="https://github.com/user-attachments/assets/dbead371-62c5-4d33-b651-d2133063eca9" />


## 3.3 비밀번호 관리

### 3.3.1 고수준 비밀번호

- INSTALL COMPONENT ‘file://component_validate_password’;

<img width="521" alt="스크린샷 2024-12-23 00 20 33" src="https://github.com/user-attachments/assets/a82efa2a-8c2b-4d87-a50f-6f50aed9a6ac" />


- policy 값에 따른 차이
    - LOW: 길이만 검증 (length)
    - MEDIUM: 길이, 숫자와 대소문자, 특수주문자 배합 검증 (mixed_case_count, number_count, special_char_count)
    - STRONG: MEDIUM 레벨 검증 후 금칙어 포함여부 검증(dictionary_file = 금칙어가 행단위로 적힌 파일의 위치)

### 3.3.2 이중 비밀번호(Dual)

- 접속중이어도 비밀번호 변경을 가능하게 하기 위함
- Primary, Secondary로 구분
- 둘 중 아무거나 입력해도 로그인이 됨
    
    <img width="741" alt="스크린샷 2024-12-23 00 48 15" src="https://github.com/user-attachments/assets/7dbc4754-d0bf-41e8-944a-039bb8ce33ea" />

    
- 고수준 비밀번호 컴포넌트를 다운받았기 때문에 조건이 걸려서 당황 → 조건에 맞게 해줌
- 기존 비밀번호와 신규 비밀번호 둘 다 로그인이 가능함, 기존 비밀번호 삭제 → 안되는거까지 확인

## 3.4 권한(Privilege)

- 정적 권한
    - 글로벌 권한
        
        ```sql
        GRANT SUPER ON *.* TO 'user'@'localhost';
        ```
        
        - mysql.user에 권한 컬럼들이 나타냄
            
            <img width="959" alt="스크린샷 2024-12-23 01 18 56" src="https://github.com/user-attachments/assets/e915bd39-942f-44d3-b970-ac77169a0a72" />

            
    - 객체 권한
    
    ```sql
    GRANT EVENT ON *.* TO 'user'@'localhost';
    GRANT EVENT ON employees.* 'user'@'localhost';
    ```
    
    <img width="1861" alt="스크린샷 2024-12-23 01 24 52" src="https://github.com/user-attachments/assets/3b9bea60-89bb-4510-a3fe-d78badc2905e" />

    
- 동적 권한
    
    <img width="611" alt="스크린샷 2024-12-23 01 26 20" src="https://github.com/user-attachments/assets/ecda8458-9614-4f6e-8706-504a595c8e6b" />

    
- 권한은 글로벌 > 데이터베이스 > 테이블 > 컬럼 > 스토어드 프로시저 순으로 계층화가 돼있음
- 글로벌에서 Select_priv가 Y로 돼있다면 모든 곳을 조회할 수 있음, 점차 좁혀나가는 형식임(그래서 내려가면서 점점 허용해줄 권한들이 없어짐)
- select를 글로벌 수준에서 권한을 부여해도 하위계층에 레코드로 생기지는 않음
- 권한 확인은 상위에서 하위로 이루어짐 → 적합한 권한 발견 시 탐색을 종료

## 3.5 역할(Role)

- 실제로는 계정과 똑같음
- 계정에 부여해야하고 활성화도 해야함
- 자동으로 ROCK이 걸려있어서 로그인은 불가능

```sql
CREATE ROLE
	role_emp_read;
	
GRANT SELECT ON `real-mysql`.* TO role_emp_read;
	
SELECT user, host from mysql.user;

CREATE USER reader@'localhost' IDENTIFIED BY 'Db1004@@'

GRANT role_emp_read TO reader@'localhost';

SHOW GRANTS FOR reader;

exit
```

```bash
mysql -u reader -p
Db1004@@
```

```sql
SELECT current_role();

SET ROLE 'role_emp_read'; // 수동 활성화

SET GLOBAL activate_all_roles_on_login=ON; // role 자동 활성화 root로 해야 가능
```

- 역할과 호스트부분은 연관이 전혀 관련이 없어서 상관없음(지정은 가능함)

```sql
ALTER USER 'role_emp_read'@'%' ACCOUNT UNLOCK;
```

- ROLE의 락을 풀면 mysql -u role_emp_read 명령어로 바로 접속이 됨
- 계정과 동일하게 db계층 권한을 얻은 부분을 볼 수 있음

<img width="1920" alt="스크린샷 2024-12-23 02 06 34" src="https://github.com/user-attachments/assets/8b857df0-791f-42af-833b-9736816d4292" />
