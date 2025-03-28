### 4.2.7 InnoDB 버퍼 풀 
InnoDB 스토리지 엔진에서 가장 핵심적인 부분으로, 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해두는 공간이다. 
Insert, Update, Delete 와 같이 데이터를 변경하는 쿼리는 랜덤 디스크 I/O를 발생시킨다. 
이러한 작업을 모아서 한번에 처리하면 랜덤 디스크 I/O 횟수를 줄일 수 있다. 

> 디스크에 접근하고 데이터를 변경하는 일은 굉장히 느린 일이다. (HDD의 접근이 느리다.)
랜덤 I/O 횟수를 줄이는 것은 이 디스크 접근 횟수를 줄여 데이터 처리 속도를 증가시키는 것이다. 


#### 4.2.7.1 버퍼 풀의 크기 설정
운영체제와 각 클라이언트 스레드가 사용할 메모리를 고려해서 설정해야 한다. 
클라이언트 세션에서 레코드를 읽고 쓸때 사용하는 `레코드 버퍼`라는 공간이 존재하는데 커넥션 수와 테이블 수가 많아진다면 이 공간이 많이 필요할 수 있다. 하지만 이는 별도로 설정할 수 없고 동적으로 변경되기에 정확한 크기를 측정하기 어렵다.

5.7버전부터 InnoDB 버퍼 풀의 크기를 동적으로 조절할 수 있게 되었다. 
`innodb_buffer_pool_size`라는 시스템 변수로 크기를 설정할 수 있다. 
가능하면 작은 값부터 천천히 늘려가면서 적절히 설정해야한다. 우선 전체 운영체제 메모리의 50%만 InnoDB 버퍼 풀로 설정하고 조금씩 변경하며 최적점을 찾는다.
하지만 매우 크리티컬한 변경이므로 사용자가 적은 시간대에 꼭 필요한 경우만 하도록 하자

기존에는 버퍼 풀 전체를 관리하는 잠금(세마포어)로 인하여 내부 잠금 경합을 많이 유발했다. 이런 경합을 줄이기 위해 버퍼 풀을 여러개로 쪼개 관리할 수 있게 개선되었다.


#### 4.2.7.2 버퍼 풀의 구조

innodb 스토리지 엔진은 버퍼 풀이라는 거대한 메모리 공간을 페이지 크기(innodb_page_size 시스템 변수)의 조각으로 쪼개 데이터를 필요로 할때마다 조각에 저장하고 사용한다. 
이 페이지 조각을 관리하기 위해 LRU(Least Recently Used) 리스트와 플러시(Flush) 리스트, 프리(Free) 리스트 3가지 자료구조로 관리한다.

아래 그림은 LRU 리스트 구조이다. 
LRU 리스트를 관리하는 목적은 가장 최근에 읽은 데이터를 리스트의 앞으로 가져오고, 오래된 데이터를 리스트의 끝으로 밀어내고 삭제하며 `디스크 읽기를 최소화` 하는 것이다. 


![](https://velog.velcdn.com/images/pi1199/post/7759346a-4868-4e21-8331-6d8afaade41d/image.png)


1. 필요한 레코드가 저장된 데이터 페이지가 버퍼 풀에 있는지 검사
	- `InnoDB Adaptive Hash Index`를 사용
	- 해당 테이블의 인덱스를 이용해 버퍼 풀에서 페이지 검색
	- 버퍼 풀에 데이터 페이지가 있다면 해당 페이지의 포인터를 MRU 방향으로 승급
2. 디스크에서 필요한 데이터 페이지를 버퍼 풀에 적재하고, 적재된 페이지에 대한 포인터를 LRU 헤더에 추가
3. LRU에 적재된 데이터 페이지가 실제로 읽히면 MRU 헤더로 이동
4. 버퍼 풀에 상주하는 데이터 페이지는 사용자 쿼리가 얼마나 최근에 접근했냐에 따라 Age가 부여되고, 쿼리에서 오랫동안 사용되지 않으면 데이터 페이지에 부여된 나이가 오래되고 버퍼 풀에서 페이지가 제거 됨
5. 필요한 데이터가 자주 접근되면 해당 페이지의 인덱스 키를 Adaptive Hash Index에 추가

> Adaptive Hash Index란?
InnoDB에서 빈번히 액세스되는 데이터 페이지를 효율적으로 검색하기 위해, B-Tree 인덱스 위에 추가적으로 생성되는 해시 인덱스입니다.
```sql 
SELECT * FROM users WHERE id = 123;
```
- 일반 B-Tree 탐색
루트 노드 → 중간 노드 → 리프 노드 → 데이터 페이지로 이동.
- Adaptive Hash Index 탐색
해시 테이블에서 id = 123에 해당하는 데이터 페이지를 즉시 찾아 접근.


**플러시 리스트**는 디스크로 동기화되지 않은 데이터를 가진 페이지(더티 페이지라고 함)의 변경 시점 기준의 페이지 목록을 관리한다.

데이터의 변경이 1번 이라도 일어났다면 플러시 리스트에서 관리되고, 특정 시점에 디스크에 기록되어야 한다. 
데이터가 변경되면 InnoDB는 변경 내용을 리두로그에 기록하고 버퍼 풀의 데이터 페이지에도 변경 내용을 반영한다.
또한, 체크 포인트를 발생시켜 리두로그와 데이터 페이지의 상태를 동기화 한다. 체크 포인트는 mysql서버가 시작될 때 리두 로그의 어느 부분 부터 복구를 실행할지 판단하는 기준점 역할을 한다. 




#### 4.2.7.3 버퍼 풀과 리두 로그

InnoDB의 버퍼 풀과 리두 로그는 매우 밀접한 관계에 있다. 버퍼 풀의 기능은 데이터 캐시 기능과 버퍼링 이라는 두가지 용도가 있다. 
버퍼 풀의 크기를 늘리는 것은 데이터 캐시 기능을 향상 시키는 것이고, 버퍼링을 향상 시키려면 리두 로그와 버퍼 풀의 관계를 이해해야 한다.
![](https://velog.velcdn.com/images/pi1199/post/9911065a-db18-4ea2-8e3c-6e239b4757fc/image.png)

버퍼 풀은 읽은 상태 그대로 변경되지 않은 `클린 페이지` 와 데이터 변경이 발생한 `더티 페이지`를 가지고 있다. 
더티 페이지는 버퍼 풀에 무한정 머무를 수 없다.

리두 로그는 1개이상의 고정 크기 파일을 연결하여 순환 고리처럼 사용한다. 따라서 재사용 가능한 공간과 불가능한 공간을 따로 관리하는데 이때, 재사용 불가능한 공간을 `활성 리두 로그`라고 한다. 그림에서 화살표를 가진 곳이 활성 리두 로그이다. 

리두 로그 파일은 계속 순환되어 사용하지만, 기록 될때마다 로그 포지션은 증가된 값을 갖는다. LSN (Log Sequence Number) 

InnoDB는 주기적으로 체크포인트 이벤트를 발생시켜 리두 로그와 더티 페이지를 디스크로 동기화 시킨다. 이때 LSN이 활성 리두 로그의 시작점이 된다. 
또한, 체크 포인트간 LSN 의 차이가 체크포인트 age라고 한다. (활성 리두 로그 공간의 크기가 된다.)


결론적으로 버퍼 풀이 아무리 커도 리두 로그파일의 크기가 작다면 버퍼링 기능을 제대로 사용하지 못한다. 
예를들어 100GB의 버퍼 풀과 100MB의 리두 로그를 갖고 있다고 생각해 보자. 
100GB의 데이터를 버퍼 풀에 임시로 저장할 수는 있지만, 데이터 동기화를 위해 리두 로그가 한 번에 처리할 수 있는 용량은 100MB에 불과합니다.



#### 4.2.7.4 버퍼 풀 플러시 

InnoDB 스토리지 엔진은 버퍼 풀에서 아직 디스크에 기록되지 않은 더티 페이지를 동기화 하기위해 다음과 같은 2개의 플러시 기능을 백그라운드로 실행한다.

- 플러시 리스트 플러시
- LRU 리스트 플러시

</br>

**플러시 리스트 플러시**

InnoDB 스토리지 엔진은 리두 로그의 공간 재활용을 위해 오래된 리두 로그를 지운다. 이때 리두 로그를 지우려면 InnoDB 버퍼 풀의 더티 페이지가 동기화 되어야 한다. 

이 동기화를 위해 플러시 리스트 플러시 함수를 호출하여 오래전에 변경된 데이터 페이지부터 순서대로 동기화 한다. 이때, 얼마나 많은 더티 페이지를 한번에 디스크에 기록하느냐에 따라 성능에 영향을 미칠 수 있다. 

더티 페이지를 디스크로 동기화 하는 스레드를 `클리너 스레드` 라고 한다. 이 클리너 스레드는 버퍼 풀 인스턴스 하나당 하나로 설정하는 것이 좋다. (버퍼 풀 인스턴스보다 클리너 스레드가 많으면 줄여주지만, 적으면 늘려주진 않는다.) 

기본적으로 더티 페이지를 많이 가지면 한번에 디스크에 쓰는양이 많아지므로 디스크 쓰기 횟수를 줄여 성능을 향상시킬 수 있다. (기본으로 버퍼풀 페이지의 90%까지 더티 페이지를 가질 수 있다.) 

그러나, 버퍼 풀에 더티 페이지가 많을수록 디스크 쓰기 폭발이 발생할 가능성이 높다. 디스크 쓰기 폭발은 더티 페이지 비율이 90%가 넘어가면 급작스럽게 디스크 쓰기가 폭증하는 현상이다. 
이 문제를 완화하기 위해 시스템 변수(innodb_max_dirty_pages_lwm)를 설정하여 조금씩 디스크 쓰기를 실행하게 하여 방지할 수 있다. 

`Adaptive flush`라는 기능을 통해 새로운 알고리즘을 통해 플러시가 동작한다. 
더티 페이지가 어느 속도로 증가하는지 즉, 리두 로그가 어느 속도로 증가하는지 분석하여 적절한 수준의 디스크 쓰기를 진행하도록 한다. 


**LRU 리스트 플러시**

사용 빈도가 낮은 페이지를 제거하여 새로운 페이지를 위한 공간을 만드는 동작이다. 
LRU 리스트 끝부터 설정된 개수만큼 스캔하며 더티 페이지는 디스크에 동기화하고, 클린 페이지는 프리(Free) 리스트로 옮긴다. 
LRU 스캔은 실질적으로 `인스턴스 수 * 변수에 설정된 LRU스캔 수`이다. 


#### 4.2.7.5 버퍼 풀 상태 백업 및 복구

앞서와 마찬가지로 버퍼 풀은 성능어 큰 영향이 있다. 
서버를 죽였다가 다시 시작하는 경우 쿼리 성능이 평소의 1/10도 안나오는 경우가 대부분이다. 이는 버퍼 풀에 이미 데이터가 있기 때문이다. 

이렇게 버퍼 풀에 데이터를 적재하는 것을 워밍업이라고 표현한다. 5.6부터는 `innodb_buffer_pool_dump_now=ON` 시스템 변수를 이용하여 버퍼 풀의 상태를 백업하고 `innodb_buffer_pool_load_now=ON`으로 복구할 수 있다. 

이 백업 데이터는 메타 데이터만 백업하기에 매우 빠르게 저장된다. 하지만 불러오는 과정은 버퍼 풀의 크기에 따라 상당한 시간이 걸릴 수 있다. 이는 메타 데이터를 이용하여 디스크에서 데이터를 다시 읽어와야 하기 때문이다. 


#### 4.2.7.6 버퍼 풀의 적재 내용 확인

5.6부터 information_schema 데이터베이스의 inno_db_buffer 테이블을 이용해 어떤 테이블의 페이지들이 적재되어 있는지 확인할 수 있다. 하지만, 버퍼 풀이 크면 성능상의 이유로 서비스용 서버에서는 활용이 불가능 했다.

8.0부터는 information_schema 데이터 베이스에 innodb_cached_indexed 가 추가되어 테이블의 인덱스별로 얼마나 버퍼풀에 적재되어 있는지 확인할 수 있다. 



### 4.2.8 Double Write Buffer

InnoDB의 리두 로그는 공간의 낭비를 막기위해 페이지의 변경된 내용만 기록한다. 
이때 더티 페이지를 디스크로 플러시할 때 일부만 기록되는 문제가 발생하는데 이를 `파셜 페이지 (partial-page)`또는 `톤 페이지(Torn-page)`라고 한다. 
이는 하드웨어 오작동이나 시스템 비정상 종료 등에서 발생할 수 있다. 

이런 문제를 막기 위해 `Double-Write` 기법을 사용한다. 

![](https://velog.velcdn.com/images/pi1199/post/b465e5bf-97ae-41c9-ba7c-2c9223c20b30/image.png)

그림에서는 A부터 E까지의 더티 페이지를 플러시 한다
우선 테이블 스페이스의 DoubleWriteBuffer에 A~E를 묶어서 기록한다. 그리고 InnoDB는 디스크에 각 더티 페이지를 하나씩 랜덤 쓰기를 진행한다. 

이 DoubleWriteBuffer의 내용은 정상적으로 쓰기가 완료된다면 필요 없다. 비 정상적으로 종료된 경우에만 재시작 시 디스크의 내용과 비교하여 반영한다. 



### 4.2.9 언두 로그


InnoDB 스토리지 엔진은 트랜잭션과 격리 수준을 보장하기 위해 DML로 변경되기 이전 버전의 데이터를 별도로 백업한다. 이 백업된 데이터를 `언두 로그`라고 한다 

- 트랜잭션 보장
트랜잭션이 롤백되면 트랜잭션 도중 변경된 데이터를 이전 데이터로 복구해야 하는데 이때 언두로그에 백업해둔 이전 버전의 데이터를 이용해 복구한다. 

- 격리 수준 보장
특정 커넥션에서 데이터를 변경하는 도중 다른 커넥션에서 데이터를 조회하면 격리 수준에 맞게 언두 로그에 있는 데이터를 읽기도 한다. 


#### 4.2.9.1 언두 로그 모니터링

언두 영역은 위에서 본 것처럼 트랜잭션의 롤백과 격리수준을 유지하며 높은 동시성을 제공하기 위함이다. 

5.5 버전 이전에는 한번 증가한 언두 로그의 영역 크기를 다시 줄일 방법이 없었다. 
트랜잭션 기간이 길거나 대용량의 데이터를 한번에 처리하는 경우 언두 영역의 크기가 커졌다. 
언두 영역의 크기가 커지면 조회시 언두 로그의 이력만큼 스캔이 필요하기에 성능이 저하되었다. 또한, 백업해야할 크기도 증가하는 문제가 발생했다. 

8.0에서는 언두로그를 순차적으로 사용해 디스크 공간을 줄이고, 서버가 필요한 시점에 자동으로 줄여주기도 한다. 하지만, 여전히 활성 상태의 트랜잭션이 오래 지속되는 것은 성능상 좋지 않다. 따라서 언두 로그를 모니터링 하는 것이 좋다. 
```//모든 버전에서 사용가능한 명령
SHOW ENGINE INNODB STATUS \G
```


#### 4.2.9.2 언두 테이블 스페이스 관리

언두 로그가 저장되는 공간을 언두 테이블 스페이스라고 한다. 

5.6버전 이전에서는 언두 로그가 모두 시스템 테이블스페이스(ibdata.ibd)에 저장되었다. 하지만 이는 서버가 초기화 될 때 생성되므로 확장에 한계가 있었다. 

8.0 이후 버전에서는 별도의 언두 로그 파일에 기록되도록 변경되었다. 

![](https://velog.velcdn.com/images/pi1199/post/b5f9674d-11e4-4f39-907b-753c62ecc31a/image.png)

하나의 언두 테이블 스페이스는 1개이상 128개 이하의 롤백 세그먼트를 가지며, 롤백 세그먼트는 1개이상의 언두 슬롯을 가진다. 
하나의 롤백 세그먼트는 InnoDB의 페이지 크기를 16바이트로 나눈 값의 개수만큼 언두 슬롯을 가진다. 
예를 들어 InnoDB의 페이지 크기가 16KB 라면 롤백 세그먼트는 1024개의 언두 슬롯을 가지게 된다. 그리고 임시 테이블을 갖지 않는 트랜잭션이라고 하면 약 2개정도의 언두 슬롯을 사용하므로 다음과 같은 수식이 나온다. 

> 최대 동시 트랜잭션 수 = InnoDB 페이지 크기 / 16 * 롤백 세그먼트 개수 * 언두 테이블스페이스 개수 / 2 (1개의 트랜잭션이 2개의 언두 슬롯을 사용)

가장 일반적인 설정인 16KB InnoDB는 언두 테이블 스페이스 2개, 롤백 세그먼트 128개를 사용한다고 가정한다면 다음과 같다. 
131072 = 16*1024 / 16 * 128 * 2 / 2


언두 슬롯이 부족하다면 트랜잭션을 시작할 수 없는 심각한 문제가 발생하므로 주의 해야한다.



#### 4.2.10 체인지 버퍼

레코드가 Insert 되거나 Update 될때 데이터 파일을 변경하는 것 뿐만 아니라 해당 테이블에 포함된 인덱스를 변경하는 작업도 필요하다.
이때 인덱스를 읽기 위해 랜덤하게 디스크를 읽는 작업이 필요하므로 테이블에 인덱스가 많다면 자원이 소모된다. 
InnoDB는 변경해야할 인덱스 페이지가 버퍼 풀에 있다면 바로 업데이트를 진행하지만, 그렇지 않다면 이를 `체인지 버퍼`에 담아두고 바로 사용자에게 결과를 반환하도록 성능을 향상 시켰다. 

유니크 인덱스의 경우에는 체인지 버퍼를 사용할 수 없다. (중복검사가 필요하기 때문)

이렇게 체인지 버퍼에 임시 저장된 인덱스 레코드 조각은 백그라운드 스레드에 의해 병합된다. 이를 체인지 버퍼 머지 스레드 라고 한다. 

`innodb_change_buffering` 변수를 이용하여 작업의 종류별(insert, delete등)로 체인지 버퍼를 적용하도록 설정할 수 있다. 

기본적으로 InnoDB 버퍼 풀의 25%를 사용하도록 되어 있지만, 필요에 따라 시스템 변수를 통해 변경할 수 있다.

### 4.2.11 리두 로그 및 로그 버퍼

리두 로그는 트랜잭션의 4가지 요소인 ACID 중에서 D에 해당하는 영속성과 가장 밀접하게 연관되어 있다.
서버가 비 정상적으로 종료 되었을때 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치이다.

대부분의 DBMS는 데이터 변경 내용을 로그에 먼저 기록한다. 
대부분의 DBMS는 쓰기보다 읽기 성능을 고려한 자료구조를 가지고 있기 때문에 쓰기에는 랜덤 액세스가 필요하다. 그렇기에 쓰기 비용이 낮은 자료구조를 가진 리두 로그를 사용한다. 마찬가지로 리두 로그를 버퍼링 할 수 있는 로그 버퍼와 같은 자료구조도 있다. 


서버가 비정상 종료되는경우 일관되지 않은 데이터를 갖는 경우가 2가지 있다. 
1. 커밋 됐지만 데이터파일에 기록되지 않은 데이터
2. 롤백 됐지만 데이터파일에 이미 기록된 데이터

1번의 경우 리두 로그에 저장된 데이터를 다시 기록하면 된다. 그러나 2번은 리두로그로 해결할 수 없다. 
2번은 변경되기 전 데이터를 가진 언두 로그의 내용을 가져와 데이터 파일을 복사한다. 리두 로그가 전혀 필요하지 않은 것은 아니고 커밋여부, 롤백여부 등등 정보를 위해 리두 로그가 필요하다.

> **데이터베이스 서버에서 리두 로그는 트랜잭션이 커밋되면 즉시 디스크로 기록되도록 시스템 변수를 설정하는 것을 권장한다.**

그래야만 서버가 비정상적으로 종료되었을 때 직전까지의 트랜잭션 커밋내용이 리두 로그에 기록될 수 있고, 리두 로그를 이용해 복구가 가능하다. 

리두 로그의 전체 크기는 InnoDB 버퍼 풀의 효율성을 결정한다. 기본은 16MB이나 blob이나 Text타입을 사용한다면 더 크게 설정하는게 좋다. 


#### 4.2.11.1 리두 로그 아카이빙
8.0버전 부터 리두 로그를 아카이빙 할 수 있다. 
엔터프라이즈 에디션이나 Xtrabackup 툴은 데이터 파일을 복사하는 동안 리두 로그에 쌓인 엔트리도 함께 복사한다. 그러나 너무 빠른 속도로 리두 로그가 쌓인다면 복사하기 전에 덮어써져 버리기 때문에 백업이 실패할 수도 있다. 
리두 로그 아카이빙 기능은 리두 로그가 덮어써지더라도 백업이 실패하지 않게 해주는 기능이다. 

아카이빙은 리두 로그가 저장될 디렉터리를 설정해야한다. 또한, 아카이빙을 시작한 세션이 끊어진다면 아카이빙을 함께 지워버리기 때문에 아키이빙시 세션을 유지해야한다. 


#### 4.2.11.2 리두 로그 활성화 및 비활성화

트랜잭션을 복구하기 위해 리두 로그는 항상 활성화 되어 있다. 
그러나 대용량 데이터를 적재하거나, 데이터를 복구하는 경우 성능을 위해 비활성화 할 수 있다. 


### 4.2.12 어댑티브 해시 인덱스

일반적으로 '인덱스'는 테이블에 사용자가 생성한 B-Tree 인덱스를 의미한다. 
하지만 어댑티브 해시 인덱스는 InnoDB 스토리지 엔진에서 사용자가 자주 요청하는 데이터에 대해 자동으로 생성하는 인덱스이다. 

B-Tree 인덱스에서 특정 값을 찾는 과정이 빠르다고 생각하지만 결국 상대적인 것이다. 
B-Tree 인덱스는 루트 -> 브랜치 -> 리프노드 까지 찾아가야 원하는 레코드를 읽을 수 있다. 이는 평소에는 괜찮지만 몇천개의 스레드가 동시에 진행한다면 cpu가 엄청난 프로세스 스케줄링을 하게되고 자연히 쿼리 성능이 저하된다. 

**어댑티브 해시 인덱스는 이러한 B-Tree 검색시간을 줄이기 위해 도입된 기능이다.**
자주 읽히는 데이터 페이지의 키 값을 이용하여 인덱스를 만들고, 필요할 때마다 어댑티브 해시 인덱스를 검색하여 레코드가 저장된 페이지를 즉시 찾아간다. 
즉, B-Tree 에서 루트-> 브랜치 -> 리프 까지의 과정을 없애 더 빠르고 많은 쿼리를 사용할 수 있게 도와준다. 

해시 인덱스는 '인덱스 키 값'과 해당 데이터가 저장된 '데이터 페이지 주소(InnoDB 버퍼 풀에 로딩된 페이지 주소)' 쌍으로 관리된다. 
'인덱스 키 값'은 B-Tree 인덱스의 고유번호(id) 와 B-Tree 인덱스의 실제 키값 으로 생성된다.
id가 포함되는 이유는 InnoDB 스토리지 엔진에서 어댑티브 해시 인덱스는 하나만 존재하기 때문에 다른 인덱스들과 구분되어야 하기 때문이다.

![](https://velog.velcdn.com/images/pi1199/post/71f91d30-994c-4881-b55e-16fa4ab5081f/image.png)
![](https://velog.velcdn.com/images/pi1199/post/1e61e0dc-27db-4d78-807c-479f77e6a6df/image.png)

예를들어 `name='Bob'` 인 데이터를 검색시 다음과 같이 동작합니다. 
1. B-Tree 인덱스의 고유번호와 키 값을 사용하여 해시 키를 생성
hash(2, 'Bob') → 0xB3C4D5.
2. 해시 인덱스를 사용해 0xB3C4D5 해시 키를 조회하고, 버퍼 풀에 저장된 데이터 페이지 주소(0x2000)를 얻음.
3. 데이터 페이지에서 실제 데이터를 검색하여 반환


위의 데이터 페이지 메모리 주소는 InnoDB버퍼 풀에 로딩된 페이지의 주소를 의미한다. 
그래서 어댑티브 해시 인덱스는 버퍼 풀에 올려진 데이터 페이지에 대해서만 관리된다. 

다만 하나의 메모리에서 관리하다 보니 세마포어(내부잠금)으로 인한 경합이 심했다. 8.0버전에서는 어댑티브 해시 인덱스를 파티션으로 나누어 이를 해결했다. 기본은 8개이다. 


이렇게 보면 너무나 좋은 기능이지만 비활성화 하는 경우도 많다. 

**어댑티브 해시 인덱스가 도움이 되지 않는 경우**

- 디스크 읽기가 많은 경우
- 특정 패턴의 쿼리가 많은 경우 (조인이나 like)
- 매우 큰 데이터를 가진 테이블의 레코드를 폭넓게 읽는 경우

**도움이 되는 경우**

- 디스크의 데이터가 InnoDB 버퍼 풀과 크기가 비슷한 경우 (디스크 읽기가 적음)
- 동등 조건 검색이 많은 경우 (동등비교, In 연산자 등)
- 쿼리가 일부 데이터에만 집중되는 경우 


결국 무조건적인 장점은 없다. 메모리를 사용하는 것이기 때문에 오히려 비효율적인 상황이 있을 수 있다. 

어댑티브 해시 인덱스가 도움이 되는지 확인해야 하는데, 가장 쉬운 방법은 MySQL 서버의 상태 값들을 살펴보는 것이다. 
> SHOW ENGINE INNODB STATUS\G


### 4.3.13 InnoDB 와 MyISAM, Memory 스토리지 엔진 비교

5.5 이전에는 MyISAM이 기본이었지만, 8.0부터는 전부 InnoDB로 교체되었다. 
호환성을 위해서만 남아있지 더이상 쓰지는 않을 것 같다.

</br>

## 4.3 MyISAM 스토리지 엔진 아키텍처

![](https://velog.velcdn.com/images/pi1199/post/04eadcf1-ab2f-4b06-a9c8-ac1ce8095559/image.png)


### 4.3.1 키 캐시

InnoDB의 버퍼 풀과 비슷한 역할을 하는 것이 키 캐시이다.
하지만 이름 그대로 키 캐시는 인덱스만을 대상으로 작동하며, 인덱스의 디스크 쓰기 작업에 대해서만 부분적으로 버퍼링 역할을 한다. 

매뉴얼에서는 일반적으로 키 캐시를 이용한 쿼리의 비율(히트율)을 99% 이상으로 유지하라고 권장한다.
히트율이 99% 미만이라면 키 캐시를 조금 더 크게 설정하는 것이 좋다.


### 4.3.2 운영체제의 캐시 및 버퍼

MyISAM 테이블의 인덱스는 키 캐시를 이용해 디스크를 검색하지 않고도 충분히 빠르게 검색할 수 있다.
하지만 데이터에 대해서는 디스크 I/O를 해결해줄 어떠한 캐시나 버퍼링 기능이 없다. 
따라서 데이터 읽기나 쓰기 작업은 항상 운영체제의 디스크 쓰기 또는 읽기 작업으로 요청될 수 밖에 없다. 

운영체제의 캐시 기능은 InnoDB처럼 데이터의 특성을 알고 전문적으로 캐시나 버퍼링을 하지 못한다. 하지만 없는 것 보다는 낫다. 운영체제의 캐시 공간은 남는 메모리를 사용하는 것이 기본 원칙이다. 
따라서 데이터베이스에서 MyISAM 테이블을 주로 사용한다면 운영체제가 사용할 수 있는 캐시 공간을 위해 충분한 메모리를 비워둬야 한다.


### 4.3.3 데이터 파일과 프라이머리 키(인덱스) 구조

InnoDB는 프라이머리 키에 의해 클러스터링 되어 저장되는 반면, MyISAM은 데이터 파일이 힙 공간 처럼 사용된다.
insert 되는 순서대로 데이터 파일에 저장된다. 이때 레코드는 모두 ROWID 라는 물리적 주소 값을 갖는데, 프라이머리 키와 세컨더리 인덱스는 모두 데이터 파일에 저장된 레코드의 ROWID값을 포인터로 가진다. 

ROWID는 가변길이와 고정길이 두가지 방법으로 저장될 수 있다. 

- 고정길이 ROWID
- 가변길이 ROWID



## 4.4 MySQL 로그 파일 
로그 파일을 이용하면 원인을 쉽게 찾아 해결할 수 있다. 

### 4.4.1 에러 로그 파일
MySQL이 실행되는 도중에 발생하는 에러나 경고 메시지가 출력되는 로그 파일이다.
my.cnf 에 정의된 위치에 저장된다 정의하지 않았다면 데이터 디렉터리에 .err 확장자로 저장된다.

- 시작 과정과 관련된 정보성 메세지
시스템 변수 적용등 시작과정에서의 정보를 나타내는 메세지이다. 

- 비정상 종료시 트랜잭션 복구 메세지
- 쿼리 처리도중 발생하는 메세지 
- 비정상적으로 종료된 커넥션 메세지
- 모니터링 또는 상태 조회 메세지
- mysql 종료 메세지 


### 4.4.2 제너럴 쿼리 로그 파일
MySQL 서버에서 실행되는 쿼리로 어떤 것들이 있는지 전체 목록을 뽑아서 검토해 볼 때가 있는데, 이때는 쿼리 로그를 활성화해서 쿼리를 쿼리 로그 파일로 기록하게 한 다음, 그 파일을 검토하면 된다.

쿼리 로그 파일에는 시간 단위로 실행됐던 쿼리의 내용이 모두 기록된다.


### 4.4.3 슬로우 쿼리 로그
서비스 운영중에 MySQL 서버의 전체적인 성능 저하를 검사하거나 정기적인 점검을 위한 튜닝을 하는 경우, 어떤 쿼리가 문제의 쿼리인지 판단하기 어렵다.
이 경우 서비스에서 사용되는 쿼리중 어떤 것이 문제인지 판단하는데 슬로우 쿼리로그가 많은 도움이 된다. 

슬로우 쿼리 로그 파일에는 `long_query_time` 시스템 변수에 설정한 시간 이상의 시간이 소요된 쿼리가 모두 기록된다.
슬로우 쿼리 로그는 실제 소요된 시간을 기준으로 슬로우 쿼리 로그에 기록할지 여부를 판단하기 때문에 반드시 쿼리가 정상적으로 실행이 완료돼야 슬로우 쿼리 로그에 기록될 수 있다.



일반적으로 슬로우 쿼리 또는 제너럴 로그 파일의 내용이 상당히 많아서 직접 쿼리를 하나씩 검토하기에는 시간이 너무 많이 걸리거나 어느 쿼리를 집중적으로 튜닝해야 할지 식별하기 어려울 수 있다.

이런 경우 Percona Toolkit의 pt-query-digest 스크립트를 이용하면 쉽게 빈도나 처리 성능별로 쿼리를 정렬해서 살펴볼 수 있다.

로그 파일의 분석이 완료되면 결과가 3개의 그룹으로 나뉘어 저장된다.
- 슬로우 쿼리 통계
분석 결과의 최상단에 표시되며, 모든 쿼리를 대상으로 슬로우 쿼리 로그의 실행 시간, 잠금 대기 시간 등에 대해 평균 및 최소/최대 값을 표시한다.

- 실행 빈도 및 누적 실행 시간순 랭킹
각 쿼리별로 응답 시간과 실행 횟수를 보여준다.
--order-by 옵션으로 정렬 순서를 변경할 수 있다.

- 쿼리별 실행 횟수 및 누적 실행 시간 상세 정보
Query ID 별 쿼리를 쿼리 랭킹에 표시된 순서대로 자세한 내용을 보여준다.
랭킹별 쿼리에서는 대상 테이블에 대해 어떤 쿼리인지만을 표시하는데, 실제 상세한 쿼리 내용은 개별 쿼리의 정보를 확인해보면 된다.

