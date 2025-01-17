# 6. 데이터 압축

## 6.1 페이지 압축

- 디스크 저장 시점에 압축
- 펀치홀을 통해 16KB안에서 남는 부분을 운영체제에 반납
- 하지만 운영체제와 하드웨어 호환이 잘 되는지를 따져야해서 잘 사용안됨

## 6.2 테이블 압축

- 하드웨어 상관없이 가능
- 단점:
    - 버퍼 풀 공간 활용률 낮음
    - 쿼리 처리 성능 낮음
    - 빈번한 데이터 변경 시 압축률 감소

### 6.2.1 압축 테이블 생성

- 압축을 위한 별도의 테이블 스페이스 필요
- innodb_file_per_table 변수 ON
- ROW_FORMAT=COMPRESSED 옵션 명시
- KEY_BLOCK_SIZE
- I/O레이어가는 일하지않음
- 목표크기에 맞을때까지 페이지 스플릿

### 6.2.2 KEY_BLOCK_SIZE 결정

- 압축 실패율 보면서 결정

### 6.2.3 압축된 페이지의 버퍼 풀 적재 및 사용

- 압축 해제버전과 그대로 읽은 버전 둘 다 적재
- 메모리 낭비 효과가 있음
- 그래서 어댑티브 알고리즘 사용
    - CPU 사용량 높으면 압축과 압축해제 피함 → Unzip_LRU 비율 높임
    - Disk IO 사용량 높으면 → Unzip_LRU 리스트 비율 낮춤

### 6.2.4 테이블 압축 관련 설정

- 193p~194p 참고

# 7. 데이터 암호화

- 응용 프로그램에서 암호화 vs MySQL 암호화
- 둘 다 하는 회사는 데이터를 굉장히 중요시하는 곳

## 7.1 MySQL 서버의 데이터 암호화

- 단방향 암호화가 아님 → 복호화가 가능
- I/O 레이어에서만 암/복호화가 일어남 → 사용자는 그대로 잘 볼 수 있음
- 이러한 방식을 TDE(Transparent Data Encryption)

### 7.1.1 2단계 키 관리

- 마스터 키 → 테이블 스페이스 키를 암호화
- 테이블 스페이스 키로 데이터를 암호화
- 마스터키는 플러그인으로 관리
- 마스터 키 로테이션 시 테이블 스페이스 키 복호화해서 다시 새로운 마스터키로 암호화
- 테이블 암호화는 진행되지않기 때문에 편함 → 만약 1단계 암호화였다면 키 바꿀때마다 매번 대용량 암호화를 해야해서 안좋아짐

### 7.1.2 암호화와 성능

- 버퍼풀은 복호화돼서 올라옴 → 디스크에서 버퍼 풀로 적재할때는 복호화가 필요함

### 7.1.3 암호화와 복제

- 단순하게 생각하면 복제서버도 같은 마스터키를 공유하면 좋겠지만, 테이블 스페이스 키도 복제가 안됨
- 레플리카는 전부 각자의 마스터키와 테이블 스페이스키들을 가짐

## 7.2 keyring_file 플러그인 설치

## 7.3 테이블 암호화

### 7.3.1 테이블 생성

- 테이블 생성 쿼리 뒤에 ENCRYPTION=’Y’; 붙여주면됨
- MySQL 서버에서 볼때는 전혀 다르지 않음

### 7.3.2 응용 프로그램 암호화와의 비교

- 정렬용 컬럼 암호화를 응용프로그램에서 해버리면 제대로 DB를 사용할 수 없으니 주의
- 선택해야 하는 상황이면 MySQL의 암호화를 추천 그러나 but MySQL 서버가 털리면 끝장
- 비밀번호는 단방향 응용프로그램 암호화가 좋음
    - 왜 why MySQL 암호화는 접속하면 평문으로 볼 수 있기 때문

### 7.3.3 테이블스페이스 이동

- export 할때 암호화된 테이블은 데이터파일과 임시 마스터키가 저장된 *.cfp 파일을 무조건 함께 복사해야함

## 7.4 언두 로그 및 리두 로그 암호화

- 시스템 변수 키면 그 순간부터 암호화돼서 들어감, 이전꺼 암호화 안해줌, 해제도 마찬가지

## 7.5 바이너리 로그 암호화

- 긴 시간동안 보관하기때문에 암호화가 중요함

### 7.5.1 바이너리 로그 암호화 키 관리

- 바이너리 로그 암호화 키= 마스터 키와 역할 동일
- 바이너리 로그 파일 키= 테이블 스페이스 키와 역할 동일

### 7.5.2 바이너리 로그 암호화 키 변경

- 많은 과정을 거쳐야 함

### 7.5.3 mysqlbinlog 도구 활용

- 터미널 도구기 때문에 mysql의 서버만가지는 바이너리 로그 암호화 키가 없음 → 불가능
- mysqlbinglog —read-from-remote-server -uroot -p -vvv mysql-bin.000011 → MySQL 서버에서 가져오는 방식의 명령이라 이거 하면 가능함