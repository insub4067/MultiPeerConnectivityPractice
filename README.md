# 뮤지션을 위한 박자 공유 메트로놈

>### WHAT WE WANTED
뮤지션들이 합주 혹은 공연 때 인이어를 구비해서 박자를 공유하는게 비용적으로 많이 소모된다고 판단했습니다.
앱을 통해 즉각적인 박자를 공유하고 이어폰으로 들으면 어떨까에서 출발한 아이디어입니다.

>### WHAT WE TRIED
인터뷰를 통해 아이디어 검증 단계에서 매우 좋은 반응을 확인하였습니다.
그래서 애플워치의 햅틱 진동을 통해 박자를 전달하는 방법과
아이폰을 통해 박자를 공유하는 방법을 구상했습니다.
하지만 애플워치는 기기의 리소스 관리의 한계로 원하는 퍼포먼스를 기대하지 못했것이란 판단으로
아이폰을 통한 박자 싱크로 기능을 정의 하였습니다.

## 시도1
### 기능 및 프로세스
1. MultiPeer Connectivity 를 이용해 Host 가 되는 아이폰이 연결된 아이폰들에게 신호를 보낸다.
2. 신호를 받은 아이폰들은 약속된 타겟 시간까지 기다렸다 메트로놈을 재생한다.

### 문제점
Host 아이폰은 연결된 아이폰들에게 현재 시간으로 부터 0.5초 이후를 전달하도록 설계하였습니다
그러면 타켓시간을 전달받은 아이폰은 현재 시간을 1ms 마다 받아서 타겟 시간과 맞는지 계산하고
타켓 시간과 일치할때 메트로놈을 재생합니다.

하지만 예상하지 못한 문제점을 아이폰마다 Date().timeIntervalSince1970 이 반환하는 값이 다르다는 것이였습니다.
완벽히 일치하지 않고 100ms ~ 400ms 정도의 오차를 발견하였고 그렇기 때문에 정확한 동시성을 보장해야할 메트로놈이 요구하는
퍼포먼스에 적합하지 않았습니다.

### 해결시도
Date().timeIntervalSince1970 를 통해 현재 시간을 가지고 오는걸 시도했을때 기기별 오차를 발견하여
CFAbsoluteTimeGetCurrent() 를 호출하여 시간을 가지고 오고자 했으나 이전 방식과 동일하게 기기별 오차를 발견하였습니다.
그렇기 때문에 API 혹은 GPS 등을 활용한 절대시간을 가지고오는 방법을 알아보았으나
외부에서 통신을 통해 가지고오는 값은 Response가 전달 되기 직전에 계산된 시간이 전달되기 때문에
reponse 를 client 에서 받을때 네트워크 환경에 따라 오차가 발생할수 있음을 발견했습니다.


## 시도2
### 기능 및 프로세스
1.  MultiPeer Connectivity 를 이용해 Host 가 되는 아이폰이 연결된 아이폰들에게 4번의 신호를 일정한 간격으로 보낸다.
2.  각 아이폰들은 2번 3번 신호의 간격을 계산하고 ( 테스트 결과 2번 3번이 비교적 가장 잘 맞았습니다. ) 5번째 신호 타이밍을 예측해서 메트로놈을 시작한다.
- 예) 2번 신호 시점 : 1100ms, 3번 신호 : 1200ms -> 5번 신호 : 1400ms 에 메트로놈 시작

### 문제점
비교적 가장 좋은 방법이였다고 판단이 됩니다.
네트워크 환경에 따라 거의 차이가 느껴지지 않을 만큼 레이턴시가 적었고 
따라서 2번째 신호와 3번째 신호가 정확하고 인터벌이 동일하다면 정확한 메트로놈 시작 타이밍을 시작할수 있었습니다.
하지만 환경에 따라 경우에 따라 오차에 대한 편차가 심했습니다.
어떤 경우는 레이턴시가 느껴지지 않을 만큼 굉장히 정확했지만
어떤 경우에는 전혀 동시성이 느껴지지 않을 만큼 오차가 컸습니다.
따라서 QC를 보장할수 없기에 문제점을 발견하였습니다

### 해결시도
신호를 보내줄때는 DistpatchQue.global().async { } 를 멀티 쓰레드에서 서로 각자의 타이밍에 신호를 보낼수 있도록 분리하였습니다
또한 신호를 받고 메트로놈을 재생할때에는 DispatchQue.main.async { } 를 통해 메인 쓰레드에서 즉각적으로 메트로놈을 재생하게끔 하여 퍼포먼스 개선을 하였습니다
하지만 iOS system 의 판단에 따라 thread 관리가 달라져 오차가 발생한다고 예측하였고 system 을 디버깅 할수 없어 해결할수 없었습니다
