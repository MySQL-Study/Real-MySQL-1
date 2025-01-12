잠금: 동시성 제어

트랜잭션: 데이터 정합성 보장

잠금: 여러 커넥션에서 동시에 동일한 자원을 요청할 경우 순서대로 한 시점에는 한나의 커넥션만 변경할 수 있게 해주는 역할

격리수준: 하나의 트랜잭션 내에서 또는 여러 트랜잭션 간의 작업 내용을 어떻게 공유하고 차단할 것인지 결정하는 레벨

InnoDB만 집중해서 살펴보자

주의사항

DBMS의 커넥션과 동일하게 최소의 코드에만 적용하는 것이 좋다.

커넥션 개수가 제한적이어서 각 단위가 커넥션 소유 시간이 길어지면 어느 순간 기다리는 상황 발생 가능

즉 커넥션, 트랜잭션 범위 최소화하라는 의미

메일 전송이나 FTP파일 전송 작업 또는 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 어떻게 해서든 트랜잭션 내에서 제거하는 것이 좋다.

프로그램이 실행되는 동안 메일 서버와 통신할 수 없는 상황이 발생하면 웹 서버 뿐 아니라 DBMS서버까지 위험해짐

단순조회는 트랜잭션을 사용하지 않는 것도 고려해야함

네트워크 작업이 있는 부분은 반드시 제거해야함 한두줄이라도. → 그럼 디폴트 트랜잭션 꺼야함?

1. 트랜잭션(Transaction)의 이해
- 예시: 은행 계좌 이체
    - A가 B에게 100만원을 이체하는 상황
    - 과정 1: A의 계좌에서 100만원 출금
    - 과정 2: B의 계좌에 100만원 입금
    - 만약 과정 1만 실행되고 과정 2가 실패하면? → A의 돈만 사라지는 심각한 문제 발생
    - 트랜잭션은 이 두 과정을 하나로 묶어서 "모두 성공" 또는 "모두 실패"가 되도록 보장
1. 동시성 문제 예시
   a) 콘서트 티켓팅 상황

```

상황: 마지막 1장의 티켓이 남았을 때 동시에 100명이 예매 시도
- 동시성 제어가 없다면:
  1. 100명 모두 "티켓 남아있음" 확인
  2. 100명 모두 예매 진행
  3. 실제로는 1장인데 100장이 판매되는 오류 발생

- 동시성 제어 적용 시:
  1. 첫 번째 구매자가 티켓 구매 시도할 때 해당 좌석 "잠금"
  2. 다른 사용자들은 잠금이 풀릴 때까지 대기
  3. 첫 번째 구매자의 거래가 완료되면 티켓은 매진 처리
  4. 다른 사용자들은 "매진" 메시지 수신

```

b) 쇼핑몰 재고 관리

```

상황: 인기 상품의 마지막 3개 재고
- 동시성 제어가 없다면:
  1. 동시에 10명이 구매 시도
  2. 각자의 시스템에서는 재고 3개 확인
  3. 10명 모두 주문 성공
  4. 실제 배송 불가능한 주문 7개 발생

- 동시성 제어 적용 시:
  1. 주문 시도마다 재고 확인 및 잠금
  2. 첫 3명의 주문만 성공
  3. 나머지 7명은 "품절" 메시지 수신

```

1. 락(Lock)의 종류와 상황별 적용

a) 비관적 락 (Pessimistic Lock)

```

상황: 예약 기차표 구매
- 작동 방식:
  1. 손님 A가 특정 좌석 선택 시 해당 좌석 즉시 잠금
  2. 다른 손님들은 해당 좌석 선택 불가
  3. A의 결제 완료 또는 시간 초과 시 잠금 해제

- 적합한 경우:
  * 인기 노선의 명절 기차표 예매
  * 선착순 이벤트 상품 구매
  * 좌석이 지정된 공연 예매

```

b) 낙관적 락 (Optimistic Lock)

```

상황: 온라인 장바구니
- 작동 방식:
  1. 고객이 상품을 장바구니에 담을 때 버전 정보 기록
  2. 결제 시도 시 버전 확인
  3. 버전이 변경되었다면 재고 다시 확인 후 진행

- 적합한 경우:
  * 일반적인 쇼핑몰 상품 구매
  * 동시 구매 가능성이 낮은 상품
  * 재고가 여유있는 상품

```

1. 격리 수준(Isolation Level)의 실제 적용

```

상황: 실시간 재고 현황 페이지
- READ UNCOMMITTED (가장 낮은 수준)
  * 장점: 매우 빠른 조회 가능
  * 위험: 잘못된 재고 수량이 표시될 수 있음
  * 적합: 정확도보다 속도가 중요한 대략적인 재고 표시

- SERIALIZABLE (가장 높은 수준)
  * 장점: 완벽한 데이터 정확성
  * 단점: 처리 속도 매우 느림
  * 적합: 금융 거래, 재고 실사 등 정확성이 필수인 상황

```

1. 실제 시스템 설계 시 고려사항

```

1. 부하 분산
   - 대형 콘서트 티켓팅
   - 대기열 시스템 도입
   - 순차적 접근 허용

2. 시간 제한
   - 장바구니 상품 임시 확보 (예: 15분)
   - 결제 진행 시간 제한 (예: 5분)
   - 시간 초과 시 자동 취소

3. 보상 트랜잭션
   - 결제는 성공했으나 재고 할당 실패 시
   - 자동 환불 프로세스
   - 대체 상품 제안 시스템

```

이러한 상황들을 잘 이해하면, 왜 트랜잭션과 동시성 제어가 필요한지, 그리고 각각의 방식들이 어떤 상황에 적합한지 더 쉽게 파악할 수 있습니다. 특히 처음 시스템을 설계할 때는 예상되는 동시 접속자 수와 데이터의 중요도를 고려하여 적절한 방식을 선택하는 것이 중요합니다.

## 락

동시성 제어를 위해 필요

레코드락

인덱스 레코드를 잠금

없으면 자동 생성된 클러스터 인덱스 이용

보조 인덱스는 키락 갭락 사용 ↔ pk나 유니크 인덱스에 의한 변경 작업에서는 레코드락 사용

갭락

index record의 갭에 걸리는 락

갭 : index record가 없는 경우

레코드와 레코드 사이에 새로운 래코드 생성되는 것을 제어하는 것 넥스트키락의 일부로 사용됨.

인덱스가 중요!!!!!!!!