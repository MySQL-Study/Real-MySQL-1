# 9.3.2 조인 최적화 알고리즘
조인 최적회에는 Exhaustive, Greedy 두가지 방법 있다.  
Exhaustive는 모두 비교하므로 조인 양이 많이 지면 느려짐.  
Greedy는 아래처럼 이루어짐. 양이 많이져도 빠름.  
![Image](https://github.com/user-attachments/assets/561def88-5dcf-427a-ad04-01a0f1233d39)
optimizer_search_depth 시스템 변수로 한번에 얼만큼씩 가능한 조인 조합 만들지 결정할 수 있다.  

optimizer_search_depth
- 기본값 62, 1-62는 greedy 
- 0으로 하면 옵티마이저가 Exhaustive, Greedy 중 자동 선택.
- 이 값보다 조인에 사용될 테이블 수가 작으면 Exhaustive 사용됨.

요즘 거는 Heuristic 검색이 활성화 되어있다.  
optimizer_prune_level 를 1로 하면 된다. 기본이 1이라 그냥 안 건들면 된다.  
optimizer_prune_level 0으로 만들면 진짜 큰일난다. 성능 떨어짐.  
혹시 누군가 0으로 사용하고 있으면 1로 해주고 돈 받아도 될 듯.  

# 9.4 쿼리힌트
# 9.4.1 인덱스 힌트
인덱스 힌트 보다는 옵티마이저 힌트 사용하도록!  
힌트 안 사용하는 방향이 좋지만 쿼리 바꾸거나 건들기 힘들때는 힌트 사용  




