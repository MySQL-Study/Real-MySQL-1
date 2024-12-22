### 서버 설정

일반적으로 하나의 설정 파일을 사용

리눅스 : my.cnf

윈도우 : my.ini

—verbose —help 옵션을 통해 mysqld 프로그램을 실행하여 설치된 MySQL 서버가 어느 디렉토리에서 my.cnf 파일을 읽는지 확인할 수 있음

### MySQL 시스템 변수의 특징

아래의 명령어를 통해 시스템 변수 확인 가능

```sql
SHOW VARIABLES
SHOW GLOBAL VARIABLE
```

### 시스템변수 vs 글로벌 변수

시스템 변수가 가지는 속성

cmd-Line : 서버의 명령어 인자로 설정될 수 있는지 여부

Option file : MySQL의 설정 파일인 my.cnf로 제어할 수 있는지 여부

System Var : 시스템 변수인지 아닌지

Var Scope : 시스템 변수의 적용범위

Dynamic : 시스템 변수가 동적인지 정적인지 구분

### 글로벌 변수 vs 세션변수

**글로벌 변수**

글로벌 범위의 시스템 변수는 하나의 MySQL 서버 인스턴스에서 전체적으로 영향을 미치는 시스템 변수

SET GlOBAL 명령으로 변경 가능하며 서버를 재시작하면 초기화 됨

**세션 변수**

세션 범위의 시스템 변수는 MySQL 클라이언트가 MySQL 서버에 접속할 때 기본으로 부여하는 옵션의 기본값을 제어하는데 사용

특정 클라이언트 연결에만 영향을 미치는 변수로 SET SESSION 명령으로 변경가능하며 세션이 종료되면 변경된 값은 사라짐

### 동적 변수 vs 정적 변수

서버가 기동 중인 상태에서 변경 가능한지에 따라 동적 변수와 정적 변수로 구분

**동적 변수**

서버를 재시작하지 않고도 값을 변경할 수 있는 변수

서버를 재시작하면 초기화됨

**정적변수**

서버를 재시작해야 변경됨

my.cnf와 같은 MySQL 설정 파일을 통해서만 값을 설정 가능함

**동적 변수 활용**

- 운영 중 실시간 조정: 예를 들어, 서비스 사용량이 갑자기 증가하여 동시 접속 클라이언트 수를 늘려야 할 때:
    
    ```sql
    sql
    SET GLOBAL max_connections = 500;
    
    ```
    

**정적 변수 활용**

- 서버 초기 설정: InnoDB 버퍼 풀 크기(`innodb_buffer_pool_size`)를 서버 메모리 용량에 맞게 설정
    
    ```
    ini
    [mysqld]
    innodb_buffer_pool_size = 4G
    
    ```
    

### SET PERSIST

글로벌 변수를 영구적으로 사용하는데 사용됨

MySQL 서버를 재시작하더라도 유지
## MySQL의 role이란?

여러 권한을 묶어서 사용하는 것입니다

특정 계정에 역할을 부여함으로써 권한을 관리할 수 있습니다

다음과 같은 방식을 통해 사용합니다

**계정 생성**

```sql
CREATE USER ‘user’@’%’
```

**역할 정의**

```sql
CREATE ROLE
	role_emp_read,
	role_emp_write
```

**권한 부여**

```sql
GRANT SELECT PM employees.* TO role_emp_read;
GRANT INSERT, UPDATE, DELETE ON employees.* TO role_emp_write;
```

**역할 부여**

```sql
GRANT role_rmp_read TO reader@'127.0.0.1';
GRANT role_emp_read, role_emp_write TO writer@'127.0.01';
```

## 실제 프로젝트에서의 적용

### **현재 프로젝트**

<img width="522" alt="캡처" src="https://github.com/user-attachments/assets/0d6cf0c2-67bd-4017-a0a9-a52823cdf0e0" />


다음과 같이 다양한 종류의 사용자와 기능이 있고, 그에 따른 권한의 차이가 존재합니다.

각각의 역할에 간단히 설명하자면

- 슈퍼호스트 : 질의응답 페이지를 생성한 사람
- 서브호스트 : 슈퍼호스트에 의해 호스트가 된 사람
- 참가자 : 슈퍼호스트가 만든 질의응답 페이지에 단순히 참여하는 사람

여기서 간단히 세션 종료기능에 대해서만 살펴보겠습니다.

세션 종료기능은 다음과 같은 기능 및 특징을 갖습니다

- 세션 종료시 질의응답 페이지에 접근이 불가합니다
- 세션 종료는 페이지를 생성한 **슈퍼호스트만이** 가능합니다

아래는 저희 프로젝트의 erd입니다

```
Table User {
  userId        
  sessions
  userSessionTokens
}

Table Session {
  sessionId
  title
  expiredAt
  createdAt
  createUserId //세션 페이지를 생성한 사용자 = 슈퍼호스트
}

```

각 테이블에 대해 설명해드리겠습니다.

- user : 회원가입한 사용자의 정보가 담겨있습니다.
- session : 페이지의 정보와 페이지를 생성한 사용자의 정보를 가지고 있습니다. 바로 이 사용자가 슈퍼호스트입니다.

session을 종료한다는 것은 **Session 테이블**에서 **expiredAt컬럼**을 현재보다 과거시간으로 **update**한다는 것을 의미합니다.

아래의 실제 코드를 살펴보겠습니다.

repository - session 테이블의 expiredAt 컬럼을 update하는 로직

```
async updateSessionExpiredAt(sessionId: string, expireTime: Date) {
    await this.prisma.session.update({
      where: {
        sessionId,
      },
      data: { expiredAt: expireTime },
    });
  }
```

service - session을 종료시킬 권한이 있는지를 확인하는 로직

```
 async terminateSession(sessionId: string, token: string) {
    const [{ createUserId }, { userId }] = await Promise.all([
      this.sessionRepository.findById(sessionId),
      this.sessionsAuthRepository.findByToken(token),
    ]);

    if (createUserId !== userId) {
      throw new ForbiddenException('세션 생성자만이 이 작업을 수행할 수 있습니다.');
    }
    const expireTime = new Date();
    await this.sessionRepository.updateSessionExpiredAt(sessionId, expireTime);
    return { expired: true };
  }
```

### MySQL의 role의 적용

**목표**

현재 프로젝트에서 service에 있는 복잡한 예외처리 없이 권한이 다른 role에 따라 작업을 처리하거나 block하는 것입니다.

**설계**

3개의 역할을 생성합니다.

- superhost
- subhost
- participant

각각의 역할의 권한을 부여합니다

- superhost : session table의 expiredAt 컬럼을 update할 수 있습니다.
- subhost : session table의 expiredAt 컬럼을 update할 수 없습니다.
- participant : session table의 exipredAt 컬럼을 update할 수 없습니다.

**한계**

위와 같은 설계를 한 뒤 다음과 같은 한계에 부딫혔습니다.

기본적으로 MySQL의 role은 db를 관리하는 개발자를 위해 만들어진 기능입니다. 하지만 현재 하고자 하는 작업은 서비스를 사용하는 사용자 사이의 역할과 권한을 나누려는 작업입니다. 즉, 서비스의 사용자가 db에 접근할 때 권한의 차이를 주겠다는 것이며, 이는 일반적이지 않은 상황임을 뜻합니다. 따라서 MySQL의 role을 실제 프로젝트에 적용하는 것은 무리라는 생각이 들었습니다.

**해결책**

다음과 같은 방법을 사용하면 제한적으로 MySQL의 role을 사용할 수 있을 것이라고 생각되었습니다.

먼저 superhost, subhost, participant에 해당하는 계정을 만들고 각각에 알맞는 권한과 역할을 부여합니다. 

사용자가 특정 작업을 수행하였다면, 해당 사용자가 어떤 사용자(superhost, subhost, participant)인지를 클라이언트로부터 넘겨받습니다. 

넘겨받은 정보에 따라 계정을 선택하고 작업을 진행합니다.

**구현**

**DB 설정**

계정생성

```
-- Superhost 계정 생성
CREATE USER 'superhost_user'@'%' IDENTIFIED BY 'superhost_password';

-- Subhost 계정 생성
CREATE USER 'subhost_user'@'%' IDENTIFIED BY 'subhost_password';

-- Participant 계정 생성
CREATE USER 'participant_user'@'%' IDENTIFIED BY 'participant_password';

```

역할생성

```
-- Superhost 역할 생성
CREATE ROLE 'superhost_role';

-- Subhost 역할 생성
CREATE ROLE 'subhost_role';

-- Participant 역할 생성
CREATE ROLE 'participant_role';

```

권한부여

session_table에 대해 superhost에게는 update권한을 부여합니다.

```
-- Superhost 역할에 SELECT 및 UPDATE 권한 부여
GRANT SELECT, UPDATE ON your_database.session_table TO 'superhost_role';
```

역할부여

```
-- Superhost 계정에 Superhost 역할 할당
GRANT 'superhost_role' TO 'superhost_user'@'%';

-- Subhost 계정에 Subhost 역할 할당
GRANT 'subhost_role' TO 'subhost_user'@'%';

-- Participant 계정에 Participant 역할 할당
GRANT 'participant_role' TO 'participant_user'@'%';

```

역할 활성화

```
-- Superhost 계정이 로그인 시 superhost_role 활성화
SET DEFAULT ROLE 'superhost_role' FOR 'superhost_user'@'%';

-- Subhost 계정이 로그인 시 subhost_role 활성화
SET DEFAULT ROLE 'subhost_role' FOR 'subhost_user'@'%';

-- Participant 계정이 로그인 시 participant_role 활성화
SET DEFAULT ROLE 'participant_role' FOR 'participant_user'@'%';

```

여기까지가 기본 db에서의 설정입니다.

**application 설정**

3개의 계정을 연결해두고, client로부터 받은 사용자의 정보에 따라 알맞은 prisma client를 반환합니다. 이루 쿼리를 실행합니다.

```
import { Injectable } from '@nestjs/common';
import { PrismaClient as SuperHostPrismaClient } from '@prisma/client'; // superhost 용 Prisma client
import { PrismaClient as SubHostPrismaClient } from '@prisma/client';   // subhost 용 Prisma client
import { PrismaClient as ParticipantPrismaClient } from '@prisma/client'; // participant 용 Prisma client

@Injectable()
export class DatabaseService {
  private superHostPrisma: SuperHostPrismaClient;
  private subHostPrisma: SubHostPrismaClient;
  private participantPrisma: ParticipantPrismaClient;

  constructor() {
    this.superHostPrisma = new SuperHostPrismaClient({
      datasources: {
        db: {
          url: 'mysql://superhost_user:superhost_password@localhost:3306/your_database',
        },
      },
    });

    this.subHostPrisma = new SubHostPrismaClient({
      datasources: {
        db: {
          url: 'mysql://subhost_user:subhost_password@localhost:3306/your_database',
        },
      },
    });

    this.participantPrisma = new ParticipantPrismaClient({
      datasources: {
        db: {
          url: 'mysql://participant_user:participant_password@localhost:3306/your_database',
        },
      },
    });
  }

  // 역할에 맞는 Prisma Client를 반환
  getPrismaClient(role: string) {
    switch (role) {
      case 'superhost':
        return this.superHostPrisma;
      case 'subhost':
        return this.subHostPrisma;
      case 'participant':
        return this.participantPrisma;
      default:
        throw new Error('Invalid role');
    }
  }

  // 예시: 쿼리 실행
  async query(role: string, model: string, data: any) {
    const prisma = this.getPrismaClient(role);
    return prisma[model].create({ data });
  }
}

```

쿼리를 실행하면 mysql의 권한에 따라 성공하거나 실패합니다.

```
import { Injectable } from '@nestjs/common';
import { DatabaseService } from '../database/database.service';

@Injectable()
export class SessionsService {
  constructor(private readonly dbService: DatabaseService) {}

  // 세션 종료 예시
  async closeSession(sessionId: string, userRole: string) {
  try {
    const prisma = this.dbService.getPrismaClient(userRole);
    const session = await prisma.session.update({
      where: { session_id: sessionId },
      data: { expiredAt: new Date() },
    });
    } catch(error) {
	    console.log("권한 없음");
	    }
    return session;
  }
}
```

예상 에러 메시지

Prisma에서 발생하는 오류

```
PrismaClientKnownRequestError: 
Error Code: P2002
Message: Permission denied for table 'session_table'
```

MySQL에서 반환하는 오류 메시지

```
ER_TABLEACCESS_DENIED_ERROR: SELECT command denied to user 'subhost_user'@'localhost' for table 'session_table'
```

결론

위와 같이 구현하게 된다면 다음과 같은 장단점이 존재할 것 같습니다.

장점 

- 별도의 권한 관리를 구현할 필요가 없고, 역할을 기반으로 할당만 하면 되기 때문에 관리가 간단합니다.
- 데이터베이스 수준에서 권한을 제한하기 때문에 보안이 강화됩니다.
- 새 역할을 추가하거나 기존 역할의 권한을 변경할 경우 서비스 로직을 수정할 필요가 없습니다.

단점

- MySQL의 권한 체계는 테이블, 열(column), 행(row) 수준에서의 세부적인 제어는 어려울 수 있습니다. 예를 들어, "자신의 데이터만 조회 가능"과 같은 조건은 뷰(View)나 저장 프로시저로 구현해야 하며, 구현과 유지보수가 어렵습니다.
    - 뷰(View)란?
        
        ### **기본 개념**
        
        - **뷰(View)**: 뷰는 데이터베이스 테이블의 일부 데이터를 보여주는 가상 테이블입니다. 실제 데이터를 저장하지 않고, 정의된 쿼리 결과를 통해 데이터를 제공합니다.
        - **`CURRENT_USER()` 함수**: 현재 데이터베이스에 접속한 사용자의 이름을 반환합니다. 이를 활용하면 접속한 사용자가 본인의 데이터만 조회하도록 제한할 수 있습니다.
        
        ---
        
        ### **뷰 생성의 상세 내용**
        
        ```sql
        sql
        코드 복사
        CREATE VIEW employee_view AS
        SELECT *
        FROM database_name.employee_data
        WHERE user_id = CURRENT_USER();
        
        ```
        
        ### **코드의 의미**
        
        - `CREATE VIEW employee_view`: `employee_view`라는 이름의 뷰를 생성합니다.
        - `SELECT *`: `employee_data` 테이블의 모든 열을 선택합니다.
        - `FROM database_name.employee_data`: `employee_data` 테이블을 데이터 소스로 사용합니다.
        - `WHERE user_id = CURRENT_USER()`: `employee_data` 테이블의 `user_id` 값이 현재 접속한 사용자의 ID(`CURRENT_USER()`)와 일치하는 데이터만 반환합니다.
        
       
- 사용자마다 다른 MySQL 연결(pool)을 사용해야 하므로 성능에 문제가 생길 수 있습니다. 또한 db의 리소스 소모가 커집니다.
- MySQL 계정, 권한, 역할을 관리하는 작업이 서비스 배포 파이프라인에 포함되어야 하므로 배포가 복잡해질 수 있습니다.
- MySQL의 역할 관리가 db와 강한 의존성을 가지게 됩니다.

따라서 MySQL의 role과 권한 관리는 일반적으로 서비스에서 사용하는 것이 어렵다고 할 수 있습니다. 하지만 DB 수준에서의 철저한 권한 관리가 필요하다면 유용하게 사용될 수도 있을 것 같습니다.
