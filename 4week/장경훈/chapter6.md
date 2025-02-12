# 6장 키-값 저장소 설계

## 키-값 저장소 종류
- 단일 키-값 저장소 : 한대의 서버를 사용하여 모든 데이터를 메모리에 둔다. 빠른 속도를 보장 하지만 메모리의 양은 한정적임으로 모든 데이터를 두기 어렵다. <br>
<strong>개선 방법: 데이터 압축 , 자주 안쓰이는 데이터는 디스크에 저장</strong><br>
- 분산 키-값 저장소: 여러 서버에 분산해서 처리, 분산 서버 저장시 CAP정리를 인지하고 있어야함.

## CAP 정리
- consistency(일관성): 분산 시스템에 접속하는 모든 클라이언트는 어떤 노드에 접속했느냐와 관계없이 언제나 같은 데이터를 보게 되어야 한다.
- availability(가용성): 분산 시스템에 접속하는 클라이언트는 일부 노드에 장애가 발생하더라도 항상 응답을 받을 수 있어야 한다.
- partition tolerance(파티션 감내): 파티션은 두 노드 사이에 통신 장애가 발생하였음을 의미한다. 파티션 감내는 네트워크에 파티션이 생기더라도 시스템은 계속 동작하여야 한다는 것을 뜻한다.<br>

![img](https://azderica.github.io/assets/static/CapImage.07cc2b7.dd8b935c30da4454e015c7f8d2451c9c.png)
- CP 시스템: 일관성, 파티션 감내를 지원
- AP 시스템: 가용성과 파티션 감내를 지원
- CA 시스템: 일관성과 가용성을 지원하는 저장소. 실제로 존재하지 않음<br>

<strong>면접에 요구사항에 맞게 선택해야함</strong>

## 시스템 컴포넌트
### 데이터 파티션
- 데이터를 파티션 단위로 분할한 뒤 여러 대 서버에 저장하는 방식<br>
<strong>고려할점<br>
1.데이터를 여러 서버에 고르게 분산할 수 있는가<br>
2.노드가 추가되거나 삭제될 때 데이터의 이동을 최소화할 수 있는가<br>
안정해시 기법을 사용해서 이 문제를 해결할 수 있다.</strong><br>

![img](https://songkg7.github.io/assets/img/2023-06-04-Consistent-Hashing/Pasted-image-20230601133003.webp)
<strong>안정해시 기법을 사용하면 규모 확장 자동화과 다양성을 가질 수 있는 장점이 있다.</strong>

### 데이터 다중화
- N개의 서버에 데이터 사본을 보존하는 방식<br>

<img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcS5T9fkKdH6ELsJWJ1cVdgPHXjIBd_DAVib3KUpBOZRfw&s" 
width="800" height= "500"/>

<strong>물리 서버를 중복 선택하지 않도록 막아야함</strong>

### 데이터 일관성
여러 노드에 다중화된 데이터는 적절히 동기화가 되어야한다.
- N: 사본 개수
- W: 쓰기 연산에 대한 정족수, 적어도 W개의 서버로부터 성공했다는 응답을 받아야한다.
- R: 읽기 연산에 대한 정족수, 적어도 R개의 서버로부터 성공했다는 응답을 받아야한다.

<strong>면접 시 주로 설정하는 방법</strong>
- 빠른 읽기 연산에 최적화된 시스템: R=1, W=N
- 빠른 쓰기 연산에 최적화된 시스템: W=1, R=N
- 강한 일관성이 보장됨: W + R > N
- 강한 일관성이 보장되지 않음: W + R <= N

### 일관성 모델
일관성 모델은 데이터의 일관성의 수준을 결정
- 강한 일관성: 모든 읽기 연산은 가장 최근에 갱신된 결과를 반환한다. 다시 말해서 클라이언트는 절대로 낡은 데이터(out-of-date)를 보지 못한다.
- 약한 일관성(weak consistency): 읽기 연산은 가장 최근에 갱신된 결과를 반환하지 못할 수 있다
- 최종 일관성: 약한일관성에 속함. 갱신 결과가 모든 사본에 반영이 되는 모델

### 데이터 버저닝
- 데이터 다중화 시 가용성은 높아지지만 사본 간 일관성이 깨질 가능성이 높다. 버저닝과 벡터 시계로 이 문제를 해소
- 벡터 시계 : [서버, 버전]의 순서쌍을 데이터에 메단것이다. 어떤 버전이 선행 버전인지, 후행 버전인지, 아니면 다른 버전과 충돌이 있는지 판별.<br>
<strong>벡터 시계는 충돌만 감지한다. 충돌 해소를 위해서는 lww와 같은 옵션이 추가되어야 한다.</strong>

## 장애 처리
### 장애 감지
<strong>가십 프로토콜</strong>
- 각 노드는 멤버십 목록을 유지한다. 멤버십 목록은 각 멤버 ID, 박동 카운터쌍의 목록이다.
- 각 노드는 주기적으로 자신의 박동카운터를 증가시킨다.
- 각 노드는 무작위로 선정된 노드들에게 주기적으로 자기 박동 카운터 목록을 보낸다.
- 박동 카운터 목록을 받은 노드는 멤버십 목록을 최신값으로 갱신한다.
- 어떤 멤버의 박동 카운터 값이 지정된 시간동안 갱신되지 않으면 해당 멤버는 장애 상태인 것으로 간주된다. (아무 서버도 해당 서버에게서 리스트를 전달받지 못했다는 뜻)

### 일시적 장애 처리
- 임시 위탁 : 임시 처리 서버가 그 동안 발생한 변경 사항을 단서(hint)로 남겨두는 기법

### 영구 장애 처리
- 반-엔트로피(anti-entropy) 프로토콜: 사본들을 비교해서 최신 버전으로 갱신하는 과정을 포함 
- 머클 트리: 해시트리라고도 불리는 머클트리는 각 노드에 그 자식 노드들에 보관된 값의 해시, 또는 자식 노드들의 레이블로부터 계산된 해시값을 테이블로 붙여두는 트리다. 해시 트리를 사용하면 대규모 자료 구조의 내용을 효과적이면서도 보안상 안전한 방법으로 검증할 수 있다.<br>

<strong>사본 간의 일관성이 망가진 상태를 탐지하고 전송 데이터의 양을 줄이기 위해서 머클 트리를 사용한다.</strong>

## 시스템 아키텍처 다이어 그램
- 클라이언트는 키-값 저장소가 제공하는 API get(key)와 put(key, value)로 통신한다.
- 중재자는 클라이언트에게 키-값 저장소에 대한 프락시 역할을 하는 노드다.
- 노드는 안정해시의 해시링 위에 분포한다.
### 쓰기 경로 
- 쓰기 요청이 커밋 로그 파일에 기록된다.
- 데이터가 메모리 캐시에 기록된다.
- 메모리 캐시가 가득차거나 사전에 정의된 어떤 임계치에 도달하면 데이터는 디스크에 있는 SSTable에 기록된다.

### 읽기 경로
- 데이터가 메모리 있는지 검사한다. 없으면 2로 간다.
- 데이터가 메모리에 없으므로 블룸 필터를 검사한다.
- 블룸 필터를 통해 어떤 SSTable에 키가 보관되어 있는지 알아낸다.
- SSTable에서 데이터를 가져온다.
- 해당 데이터를 클라이언트에게 반환한다.


