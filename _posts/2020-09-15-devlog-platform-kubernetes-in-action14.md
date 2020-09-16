---
layout: post
title: "[kubernetes-in-action] 14. 파드의 컴퓨팅 리소스 관리"
subtitle: "[kubernetes-in-action] 14. 파드의 컴퓨팅 리소스 관리"
categories: devlog
tags: platform
---

# 14. 파드의 컴퓨팅 리소스 관리
> 파드의 예상 소비량과 최대 소비량을 설정하는 것은 파드 정의에서 매우 중요하다.
> 이는 리소스를 공평하게 공유하고, 클러스터 전체에서 파드가 스케줄링되는 방식에도 영향을 미친다.

## 14.1 파드 컨테이너의 리소스 요청
 - 파드를 생성할 때 컨테이너가 필요로 하는 CPU와 메모리 양(requests)과 사용할 수 있는(limits)  엄격한 제한을 지정할 수 있다.
 - 컨테이너에 개별적으로 지정되며 파드 전체에 지정되지 않는다.
 - 파드는 모든 컨테이너의 요청과 제한의 합으로 계산된다.

### 14.1.1 리소스 요청을 갖는 파드 생성
 - 아래와 같이 파드 매니페스트에 요청을 지정할 수 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: requests-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      requests:        # 컨테이너 리소스 요청
        cpu: 200m      # cpu 200밀리코어 (1 core : 1000m 또는 1, 200밀리코어는 1/5 core 시간을 의미)
        memory: 10Mi   # 10Mi 메모리를 요청
```

 - CPU 요청을 지정하지 않으면 컨테이너에서 실행중인 프로세스에 할당되는 CPU시간에 신경쓰지 않는다는 것과 같다. 하지만 최악의 경우 CPU 시간을 전혀 할당받지 못할수도 있다.(다른 프로세스가 CPU 요청을 많이 한 상황에서 발생할 수 있음)
    - 이러한 이유로 배치같은 경우에 CPU를 지정하지 않을수도 있으나, 웹애플리케이션같은곳에서는 필수적으로 지정하는것이 좋다.


``` sh
# 리소스 요청을 명시한 파드 생성
kubectl create -f requests-pod.yaml 

# 컨테이너 내부 CPI와 메모리 사용량 확인
kubectl exec -it requests-pod top
```

<img width="556" alt="스크린샷 2020-09-15 오후 2 21 25" src="https://user-images.githubusercontent.com/6982740/93168838-c3ab6e80-f75e-11ea-8706-ba47b940aff6.png">

 - minikube 가상머신이 CPU 코어 2개를 사용하고 있고, dd 명령을 통해 최대 CPU를 소비하지만 스레드 하나로 실행되어 코어 하나를 모두 점유하므로 cpu use가 50%인것을 볼수 있다.
 - 그러나 이는 요청했던 200밀리코어보다 더 많은 양의 CPU타임을 점유하는 것이므로 이를 막으려면 CPU 제한(limit)을 지정해야 한다.

### 14.1.2 리소스 요청이 스케줄링에 미치는 영향
 - 파드에 필요한 리소스의 최소량을 지정할 수 있다.
 - 스케줄러는 파드를 스케줄링할 때 먼저 파드의 리소스 요청 사항을 만족하는 충분한 리소스를 가진 노드만를 고려한다. 

#### 파드가 특정 노드에 실행할 수 있는지 스케줄러가 결정하는 방법
 - 스케줄러는 스케줄링하는 시점에 각 개별 리소스가 얼마나 사용되는지 보지 않고, 노드에 배포된 파드들의 리소스 요청량의 전체 합만을 기준으로 처리한다.
 - 실제 리소스 사용량에 기반해 다른 파드를 스케줄링한다는 것은 이미 배포된 파드에 대한 리소스 요청 보장을 깨뜨릴 수 있기 때문이다.
 - 아래 그림처럼 파드D를 배포하려고 하지만 노드에 미할당된 CPU양으로는 커버할수 없기 때문에 해당 노드에 스케줄링되지 않는다.

<img width="570" alt="스크린샷 2020-09-15 오후 2 25 11" src="https://user-images.githubusercontent.com/6982740/93169078-47fdf180-f75f-11ea-8c07-67904dd93040.png">

#### 스케줄러가 파드를 위해 최적의 노드를 선택할 때 파드의 요청을 사용하는 방법
 - 스케줄러는 노드의 목록을 필터링한 다음 설정된 우선순위 함수에 따라 남은 노드의 우선순위를 지정할 수 있다.
 - 스케줄러는 우선순위 함수 중 한가지를 선택할 수 있다.

##### 스케줄러의 우선순위 함수(요청된 리소스 양에 기반해 노드의 순위를 정한다.)
 - LeastRequestedPriority :  할당되지 않은 리소스의 양이 많은 노드 우선(요청 리소스가 적은)
 - MostRequestedPriority : 할당된 리소스의 양이 가장 많은 노드 우선(요청 리소스가 많은)

##### MostRequestedPriority의 필요성
 - 클라우드 기반 환경에서는 노드수가 비용이기 떄문에 MostRequestedPriority와 같은 스케줄러 우선순위 함수를 차용하는것이 비용절감에 효과적일 수 있다.

#### 노드의 용량 검사
 - Capacity는 노드 전체 용량, Allocatable은 할당 가능한 리소스를 의미한다.
 - 특정 리소스는 쿠버네티스 시스템 구성 요소로 스케줄링되어 실제 노드의 리소스 100%만큼 파드를 할당할 수는 없다.


``` sh
# 노드 정보 조회
kubectl describe nodes

# 파드 생성
kubectl run requests-pod-2 --image=busybox --restart Never --requests='cpu=800m,memory=20Mi' -- dd if=//dev/zero of=/dev/null
```

<img width="406" alt="스크린샷 2020-09-15 오후 2 30 12" src="https://user-images.githubusercontent.com/6982740/93169395-fa35b900-f75f-11ea-9cbb-573ebfd6efbb.png">

#### 어느 노드에도 실행할 수 없는 파드 생성
 - 노드의 전체의 리소스가 부족하기 때문에 Pending되고 노드에 스케줄링 되지 못한다.

``` sh
# cpu 1core를 사용하는 파드 생성
kubectl run requests-pod-3 --image=busybox --restart Never --requests='cpu=1,memory=20Mi' -- dd if=//dev/zero of=/dev/null
```

<img width="725" alt="스크린샷 2020-09-15 오후 3 03 33" src="https://user-images.githubusercontent.com/6982740/93171751-a2e61780-f764-11ea-9323-efcb7da09341.png">

<img width="789" alt="스크린샷 2020-09-15 오후 3 04 30" src="https://user-images.githubusercontent.com/6982740/93171827-c5783080-f764-11ea-85fc-6c247f6b6012.png">

 - Events를 보면 Insufficient cpu로 인해 FailedScheduling된 것을 알 수 있다.

#### 파드가 스케줄링되지 않은 이유 확인
 - 요청한 파드보다 더 많은 CPU를 사용하고 있다. 이는 파드 목록에 있는 kube-system 파드에서 사용하고 있는것을 알 수 있다.
 - 새로운 파드를 요청할 때 노드를 오버커밋(overcommitted)하게 만들기 때문에 스케줄러는 노드에 파드를 스케줄링 할 수 없다.

``` sh
# 노드 정보 확인
kubectl describe node
```

<img width="923" alt="스크린샷 2020-09-15 오후 3 06 38" src="https://user-images.githubusercontent.com/6982740/93172062-20aa2300-f765-11ea-9eb3-a0b8f205cda9.png">

#### 파드가 스케줄링될 수 있도록 리소스 해제
 - 두번째 파드를 삭제하면 스케줄러는 삭제를 통지받고, 두번째 파드가 종료되자마자 세번째 파드를 스케줄링할것 이다.

``` sh
# 두번쨰 파드 삭제
kubectl delete po requests-pod-2

# 파드 확인
kubectl get po
```

<img width="427" alt="스크린샷 2020-09-15 오후 3 10 15" src="https://user-images.githubusercontent.com/6982740/93172300-91e9d600-f765-11ea-929d-b9dd536a9a8c.png">


### 14.1.3 CPU 요청이 CPU 시간 공유에 미치는 영향
 - CPU 요청은 단지 스케줄링에만 영향을 미칠 뿐만 아니라 남은 CPU 시간을 파드에 분배하는 방식도 결정한다.
 - 미사용된 CPU는 두 파드 사이에 요청 비율에 맞게 나뉘어진다.(첫번쨰 파드가 200m, 두번쨰 파드가 1000m이므로 1:5 비율로 나뉜다)
 - 이 외에도 한 컨테이너가 CPU를 최최대로 사용하려는 순간 나머지 파드가 유휴 상태에 있다면 전체 CPU 시간을 사용할 수도 있다.(아무도 사용하지 않는다면 사용 가능한 모든 CPU를 사용 가능)

<img width="580" alt="스크린샷 2020-09-15 오후 3 11 40" src="https://user-images.githubusercontent.com/6982740/93172404-c493ce80-f765-11ea-9395-a7ffd11d1e25.png">

### 14.1.4 사용자 정의 리소스의 정의와 요청
 - 쿠버네티스는 사용자 정의 리소스를 노드에 추가하고 파드의 리소스 요청에 이를 사용할 수 있다.
 - 18 버전 이후에 확장 리소스(Extended resources)를 이용할 수 있다.
 - GPU 단위 수와 같은 부분들을 사용자 정의 리소스로 만들어 처리할 수 있음.

## 14.2 컨테이너에 사용 가능한 리소스 제한
### 14.2.1 컨테이너가 사용 가능한 리소스 양을 엄격한 제한으로 설정
 - 특정 컨테이너가 지정한 CPU 양보다 많은 CPu를 사용하는것을 막고 싶을 수 있다. 
 - CPU는 압축 가능한 리소스이므로 프로세스에 악영향을 주지 않으면서 컨테이너가 사용하는 CPU양을 조절할 수 있다.
 - 반면 메모리는 프로세스에 주어지면 프로세스가 메모리르 해제하지 않은 한 다른 프로세스에서 사용할 수 없다.(그래서 컨테이너에 할당되는 메모리의 최대량을 제한해야 한다.)
    - 메모리를 제한하지 않으면 워커노드에 실행 중인 컨테이너는 사용 가능한 모든 메모리를 사용해서 노드에 있는 다른 모든 파드와 노드에 스케줄링되는 새 파드에 영향을 미칠 수 있다.

#### 리소스 제한은 갖는 파드 생성
 - 쿠버네티스는 모든 컨테이너의 리소스에 제한을 지정할 수 있다.
 - 리소스 요청을 지정하지 않고 제한만 지정한 경우, 자동적으로 요청과 제한이 동일한 값으로 설정된다.
 
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: limited-pod
spec:
  containers:
  - image: busybox
    command: ["dd", "if=/dev/zero", "of=/dev/null"]
    name: main
    resources:
      limits:           # 컨테이너의 리소스 제한
        cpu: 1          # 최대 CPU 1코어를 사용할 수 있다.
        memory: 20Mi    # 최대 20Mi 메모리를 사용할 수 있다.
```

#### 리소스 제한 오버커밋
 - 리소스 제한은 노드의 할당 가능한 리소스 양으로 제한되지 않는다.(리소스 요청과 다른 부분)
     - 노드 용량의 100%를 초과할 수 있다는 의미(오버커밋될 수 있다.)
 - 여기서 노드 리소스의 100%가 다 소진되면 특정 컨테이너는 제거가 되어야 한다.(쿠버네티스에서 리소스 제한을 초과해 사용하려고 하거나 초과하는 경우에 컨테이너를 제거한다.)

<img width="580" alt="스크린샷 2020-09-15 오후 4 07 32" src="https://user-images.githubusercontent.com/6982740/93177300-91edd400-f76d-11ea-922e-72997462ecc9.png">

### 14.2.2 리소스 제한 초과
 - CPU 제한이 설정돼 있으면 프로세스는 설정된 제한보다 많은 CPU 시간을 할당받을 수 없다.
 - 메모리의 경우는 CPU와 달리 제한보다 많은 메모리를 할당받으려 시도하면 프로세스는 강제 종료된다.(컨테이너 OOMKilled)
 - 파드의 재시작 정책(restart policy)이 Always 또는 OnFailure로 설정된 경우 프로세스는 즉시 다시 시작하므로 이를 알아치리지 못할수도 있음.
 - 메모리 제한 초과와 종료가 지속적으로 반복하면 쿠버네티스는 재시작 사이의 지연 시간을 증가시키면서 재시작한다.(이런 재시작의 경우 CrashLoopBackOff STATUS를 갖는다.)

#### CrashLoopBackOff
 - 이 상태는 크래시 후 kubelet이 컨테이너를 다시 시작하기 전에 간격을 늘리는 상태를 의미한다.
 - 첫 번째 크래시 후에 kubelet은 컨테이널르 즉시 다시 시작하고, 다시 크래시가 발생하면 10초를 기다리고, 그 이후에는 20초, 40초, 80초 지수 단위로 증가하다가 마지막 300초(5분)로 제한된다.
 - 컨테이너의 크래시된 이유를 보면 ```Reason : OOMKilled ```을 확인할 수 있다.
 - 컨테이너가 종료되는 것을 원하지 않는다면 메모리 제한을 너무 낮게 설정하지 않는 것이 중요하다.(하지만 컨테이너가 제한을 넘어서지 않아도 OOMKilled되는 경우는 있음)
 
<img width="579" alt="스크린샷 2020-09-15 오후 4 59 38" src="https://user-images.githubusercontent.com/6982740/93182604-d9c42980-f774-11ea-9fa6-e5cf80e9f1d9.png">


### 14.2.3 컨테이너의 애플리케이션이 제한을 바라보는 방법
#### 컨테이너는 항상 컨테이너 메모리가 아닌 노드의 메모리를 본다.
 - 컨테이너 내부에서 top 명령을 수행해보면 컨테이너가 실행중인 전체 노드의 메모리 양을 표시한다.(컨테이너 레벨에서 이 제한을 인식할 수 없다.)
 - JVM은 컨테이너에 사용 가능한 메모리 대신 호스트의 총 메모리를 기준으로 힙 크기를 설정하여 OOMKilled 위험이 있음.(최대 힙 크기를 지정하지 않은 경우 JVM GC가 돌기전에 메모리 제한을 넘어서 OOMKilled 될 수 있음.)
    - 1.8.192 이상의 JVM에서는 -XX:+UseCGroupMemoryLimitForHeap 을 통해 cgroup으로 할당된 컨테이너 메모리를 사용할 수 있음.
    - JVM 10 이상부터는 -XX+UseContainerSupport 를 사용할 수 있다.

#### 컨테이너는 노드의 모든 CPU 코어를 본다.
 - 메모리와 마찬가지로 컨테이너에 설정된 CPU 제한과 관계없이 노드의 모든 CPU를 본다.
 - CPU 제한이 하는 일은 컨테이너가 사용할 수 있는 CPU 시간의 양을 제한하는것 뿐이다.
 - CPU 제한이 1코어로 설정되더라도 컨테이너의 프로세스는 한 개 코어에서만 실행되는 것이 아니다. 
 - 쿠버네티스 클러스터에서 애플리케이션을 개발할때 주의할 점은 시스템의 CPU 코어수를 기준으로 작업 스레드를 결정하는 경우에 문제가 될 수 있다.
     - 많은 수의 코어를 갖는 노드에 이 앱이 배포되면 너무 많은 스레드가 기동돼 CPU 시간을 두고 경합이 발생할 수 있다.
     - 이런 방식 대 Downward API를 사용해 컨테이너의 CPU 제한을 전달하고 이를 사용하는 것이 좋다.

## 14.3 파드 Qos 클래스 이해
> 리소스 제한은 오버커밋될 수 있으므로 노드가 모든 파드의 리소스 제한에 지정된 양의 리소스를 반드시 제공할 수는 없다.
> 그러면 파드들 중에서 더 많은 메모리 사용을 요구하게 되면서 노드가 필요한 만큼의 메모리르 제공할 수 없을때는 결국 어떤 컨테이너는 종료시키고 메모리를 확보해야 한다. 이 경우 어떤 기준으로 종료될 파드를 결정할까?
> 이는 상황에 따라 다르며, 쿠버네티스 스스로 결정할 수 없기 때문에 쿠버네티스에서는 이 우선순위를 지정하는 방법을 제공한다.

### 파드를 분류하는 세가지 서비스 품질(QoS, Quality of Service) 클래스
 - BestEffort(최하위 우선순위)
 - Burstable 
 - Guaranteed(최상위 우선순위)

### 14.3.1 파드의 QoS 클래스 정의
 - QoS 클래스는 파드 컨테이너의 리소스 요청과 제한의 조합으로 파생된다.

#### 1) BestEffort 클래스
 - 우선순위가 가장 낮은 클래스
 - 아무런 리소스 요청과 제한이 없는 경우 부여된다.
 - 이런 파드에 실행중인 컨테이너는 리소스를 보장 받지 못한다.(최악의 경우 CPU 시간을 전혀 받지 못할 수도 있다.)
 - 메모리가 필요한 겨우 가장 먼저 종료된다.
 - 그러나 설정된 메모리 제한이 없기 때문에 메모리가 충분하다면 컨테이너는 원하는 만큼 메모리를 사용할 수 있다.

#### 2) Guaranteed 클래스
 - 이 클래스는 리소스 요청과 리소스 제한이 동일한 파드에 부여된다.
 - 이런 파드의 컨테이너는 요청된 리소스의 양을 얻는것은 보장되지만 추가 리소스를 사용할 수 없다.

##### Guaranteed의 조건
 - CPU와 메모리 리소스 요청과 제한이 모두 설정되어야 한다.
 - 각 컨테이너에 설정되어 있어야 함.
 - 리소스 요청과 제한이 동일해야 한다.

#### 3) Burstable 클래스
 - BestEffort와 Guaranteed 사이의 클래스(BestEffort와 Guaranteed가 아닌 경우 해당된다.)
 - 컨테이너의 리소스 제한이 리소스 요청과 일치하지 않은 단일 컨테이너 파드
 - 적어도 한 개의 컨테이너가 리소스 요청을 지정했지만 리소스 제한을 설정하지 않은 모든 파드가 여기에 속한다.
 - 이 클래스 특징은 요청한 양 만큼의 리소스를 얻지만 필요하면 추가 리소스(리소스 제한까지)를 사용할 수 있다.


#### 리소스 요청과 제한 사이의 관계가 QoS 클래스를 정의하는 방법

<img width="508" alt="스크린샷 2020-09-15 오후 5 58 26" src="https://user-images.githubusercontent.com/6982740/93189406-1005a700-f77d-11ea-8523-a50ec94fa824.png">

> 참고 : 컨테이너에 제한만 설정된 경우, 요청은 기본으로 제한과 동일한 값으로 설정되므로 Guaranteed QoS 클래스를 갖게 된다.

#### 다중 컨테이너 파드의 QoS 클래스 파악
 - 다중 컨테이너 파드의 경우 모든 컨테이너가 동일한 QoS 클래스를 가지면 그것이 파드의 QoS가 된다.
 - 파드의 QoS 클래스는 kubectl describe pod 또는 status.qosClass 필드에서 확인할 수 있다.

| 컨테이너1 QoS 클래스 | 컨테이너2 QoS 클래스 | 컨테이너3 QoS 클래스 | 파드 QoS 클래스 |
|----------------------|----------------------|----------------------|-----------------|
| BestEffort           | BestEffort           | BestEffort           | BestEffort      |
| BestEffort           | Burstable            | Guaranteed           | Burstable       |
| ...                  | ...                  | ...                  | Burstable            |
| Guaranteed           | Guaranteed           | Guaranteed           | Guaranteed      |



### 14.3.2 메모리가 부족할 때 어떤 프로세스가 종료되는지 이해
 - BestEffort 클래스가 가장 먼저 종료되고, 다음은 Burstable 파드가 종료되며, 마지막으로 Gurantted 파드는 시스템 프로세그가 메모리를 필요로 하는 경우에만 종료된다.

#### QoS 클래스가 우선순위를 정하는 방법
 - BestEffort 파드에 실행 중인 프로세스가 Burstable 파드의 프로세스 이전에 항상 종료된다.

<img width="595" alt="스크린샷 2020-09-15 오후 6 04 05" src="https://user-images.githubusercontent.com/6982740/93190058-daad8900-f77d-11ea-808e-6f6e3d0699bb.png">

#### 동일 Qos 클래스를 갖는 컨테이너 중 우선순위
 - 실행중인 각 프로세스는 OOM 점수를 갖는다. 모든 실행중인 프로세스의 OOM score를 비교해 종료할 프로세스를 선정한다.
 - 노드에서 메모리 해제가 필요한 경우 가장 높은 점수의 프로세스가 종료됨.

#### OOM score 계산 방식
 - 프로세스가 소비하는 가용 메모리의 비율과 컨테이너의 요청된 메모리와 파드의 QoS 클래스를 기반으로 한 고정된 OOM score 기준이다.
 - 즉 요청된 메모리 중 사용률이 높은 컨테이너가 먼저 종료된다.(위 예제에서 파드B가 파드C보다 먼저 종료되는 이유)

## 14.4 네임스페이스별 파드에 대한 기본 요청과 제한 설정
> 컨테이너 리소스 요청과 제한을 설정하지 않으면 컨테이너는 이를 설정한 다른 컨테이너에 의해 좌지우지된다.
> 그래서 모든 컨테이에 리소스 요청과 제한을 설정하는 것이 좋다.

### 14.4.1 LimitRange 리소스
 - 모든 컨테이너에 리소스 요청과 제한을 설정하는 대신 LimitRage 리소스를 생성해 이를 수행할 수 있다.
 - 컨테이너의 각 리소스(네임스페이스별) 최소/최대 제한을 지정할 수 있고, 리소스 요청을 명시적으로 지정하지 않은 컨테이너의 기본 리소스 요청을 지정할 수 있다.
 - LimitRange는 파드 검증과 기본값 설정에 사용된다.
 - LimitRange 리소스는 LimitRange 어드미션 컨트롤 플러그인에서 사용된다.

<img width="600" alt="스크린샷 2020-09-15 오후 6 21 45" src="https://user-images.githubusercontent.com/6982740/93192166-53154980-f780-11ea-8b14-88337728d902.png">

### 14.4.2 LimitRange 오브젝트 생성
 - 컨테이너 레벨에서 보면 최소값, 최대값 뿐만 아니라 리소스 요청과 제한을 명시적으로 지정하지 않은 각 컨테이너에 적용될 기본 요청(defaultRequest)와 기본 제한(default)을 설정할 수 있다.
 - 또한 maxLimitREquestRatio을 통해 제한 대 요청 비율을 설정할 수 있다.(cpu 4의 의미는 cpu limit은 cpu request보다 4배 이상 큰 값이 될 수 없다.)
 - 단일 PVC에서 요청 가능한 스토리지 양을 제한핫ㄹ수도 있다.
 - 하나의 LimitRange 리소스로 관리할수도 있고, 각 type별로 쪼개서 관리할수도 있다.
 - 다른 쿠버네티스 매커니즘과 마찬가지로 나중에 수정된 사항은 기존 파드 및 PVC 등에 영향을 미치지 않는다.

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod         # 파드 전체에 리소스 제한 지정
    min: 
      cpu: 50m 
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
  - type: Container   # 컨테이너 전체에 요청 기본 지정
    defaultRequest:
      cpu: 100m
      memory: 10Mi
    default:          # 리소스 제한을 지정하지 않은 컨테이너의 기본 제한(limit)
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio:          # 각 리소스요청/제한간의 최대 비율
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim    # PVC에 대한 min/max
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

### 14.4.3 강제 리소스 제한

``` sh
# LimitRange 리소스 생성
kubectl create -f limits.yaml

# 요청이 큰 파드 생성
kubectl create -f limits-pod-too-big.yaml

The Pod "too-big" is invalid: spec.containers[0].resources.requests: Invalid value: "2": must be less than or equal to cpu limit
```

<img width="908" alt="스크린샷 2020-09-15 오후 6 30 29" src="https://user-images.githubusercontent.com/6982740/93193243-8e644800-f781-11ea-9d9e-81e44d47c5c2.png">

### 14.4.4 기본 리소스 요청과 제한 적용
 - 기본값이 정상 동작하는지 확인해보자.

``` sh
# 예전에 실습한 파드 생성
kubectl create -f ../Chapter03/kubia-manual.yaml

# 파드 상태 확인(limit, request가 정상적으로 반영된 것을 볼 수 있다.)
kubectl describe po kubia-manual
```

<img width="1027" alt="스크린샷 2020-09-15 오후 6 33 21" src="https://user-images.githubusercontent.com/6982740/93193544-f3b83900-f781-11ea-9c16-8c18e940dcae.png">

 - LimitRange에 설정된 제한은 각 개별 파드/컨테이너에만 적용이 되는데, 이 경우 많은 수의 파드가 만들어진다면 클러스터에서 사용 가능한 모든 리소스를 다 써버릴수 있다.(요청과 달리 제한은 총 리소스합을 넘어설수 있으므로)

## 14.5 네임스페이스의 사용 가능한 총 리소스 제한하기
 - 쿠버네티스에서는 리소스쿼터(ResourceQuota)를 통해 각 네임스페이스에서 사용 가능한 총 리소스를 제한할 수 있다.

### 14.5.1 리소스쿼터 오브젝트
 - 리소스쿼터 오브젝트는 리소스쿼터 어드미션 컨트롤 플러그인에 의해서 처리되고, 이는 생성하려는 파드가 설정된 리소스쿼터를 초과하는지 확인한다.
 - 리소트쿼터는 LimitRange와 마찬가지로 네임스페이스 기준으로 동작한다.
 - 파드가 사용할 수 있는 컴퓨팅 리소스 양과 퍼시스턴트볼륨클레임이 사용할 수 있는 스토리지 양 등을 제한할 수 있다.

#### CPU 및 메모리에 관한 리소스쿼터 생성
 - 리소스쿼터에서는 CPU 및 메모리에 대한 요청과 제한에 대한 별도 합계를 정의한다.
 - 네임스페이스 내에서 모든 파드들의 할 수 있는 요청과 제한의 최대값을 지정한다.
- 각 개별 파드나 컨테이너에 개별적으로 적용하지 않고 모든 파드의 리소스 요청과 제한의 총합에 적용된다.

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: cpu-and-mem
spec:
  hard:
    requests.cpu: 400m
    requests.memory: 200Mi
    limits.cpu: 600m
    limits.memory: 500Mi
```

<img width="575" alt="스크린샷 2020-09-15 오후 6 44 52" src="https://user-images.githubusercontent.com/6982740/93194896-8e654780-f783-11ea-8734-6cb89f10c5cc.png">

#### 쿼터와 쿼터 사용량 검사
``` sh
# 리소스쿼터 생성
kubectl create -f quota-cpu-memory.yaml

# 리소스쿼터 조회
kubectl describe quota
```

#### 리소스쿼터와 함께 LimitRange 생성
 - 리소스쿼터를 생성할 때 주의할 점은 LimitRange 오브젝트와 함꼐 생성해야 한다는 것이다.
 - LimitRange에 default가 없는 경우, 리소스 요청이나 제한도 명시하지 않은 파드는 아예 생성할 수 없게 된다.


### 14.5.2 퍼시스턴트 스토리지에 관한 쿼터 지정하기
 - 네임스페이스의 모든 퍼시스턴트볼륨클레임이 요청할 수 있는 스토리지 양을 제한할 수 있다.
 - PVC는 특정 스토리지 클래스에 동적 프로비저닝된 퍼시스턴트 볼륨을 요청할 수 있는데, 이때문에 각 스토리지 클래스에 개별적으로 스토리지 쿼터를 정의할 수 있도록 한 것이다.

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage
spec:
  hard:
    requests.storage: 500Gi
    ssd.storageclass.storage.k8s.io/requests.storage: 300Gi
    standard.storageclass.storage.k8s.io/requests.storage: 1Ti
```

### 14.5.3 생성 가능한 오브젝트 수 제한
 - 리소스쿼터는 네임스페이스 내의 파드, 레플리카셋, 서비스, 디플로이먼트 등에 대해서 오브젝트 수를 제한하는 것도 제공한다.

``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: objects
spec:
  hard:
    pods : 10
    replicationcontrollers: 5
    secrets: 10
    configmaps: 10
    services: 5
    services.loadbalancers: 10
    services.nodeports: 10
    standard.storageclass.storage.k8s.io/persistentvolumeclaims: 2
```

### 14.5.4 특정 파드 상태나 QoS 클래스에 대한 쿼터 지정
 - QoS클래스 기준으로 쿼터를 정할 수도 있다.

#### QoS 클래스 기준 쿼터 범위(quota scope)
 - BestEffort : BeftEffort QoS 클래스를 갖는 파드에 적용
 - NotBestEffort : BeftEffort가 아닌 QoS 클래스(Burstable, Guaranteed)를 갖는 파드에 적용
 - Terminating : 종료 과정의 파드에 적용
 - NotTerminating : 종료되지 않은 파드에 적용

#### Terminating 관련 참고사항
 - 각 파드가 종료되고 실패로 표시되기 까지 대기시간이 있는데 이 값에 따라 Terminating, NotTerminating이 구분된다. 
 - 파드 스펙의 activeDeadlineSeconds 필드에 이 값을 설정할 수 있다.
 - activeDeadlineSeconds가 설정된 파드에는 종료되는 과정에서 Terminating scope에 포함된다.
 - activeDeadlineSeconds가 설정되지 않은 파드는 NotTerminating scope에만 적용된다.

#### Qos 클래스 쿼터 뻠위를 지정한 리소스쿼터
``` yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: besteffort-notterminating-pods
spec:
  scopes:
  - BestEffort               # BestEffort Qos를 갖고 유효 데드라인이 없는 파드에만 적용
  - NotTerminating
  hard:
    pods: 4                   # 저 스코프에 해당하는 파드는 4개까지만 존재할수 있다.
```

## 14.6 파드 리소스 사용량 모니터링
> 쿠버네티스 클러스터를 최대한 활용하기 위해 리소스 요청과 제한을 적절하게 설정하는 것은 매우 중요한 일이다.
> 하지만 적정 값을 찾으려면 모니터링이 필요하다.
> 예상되는 부하 수준에서 컨테이너의 실제 리소스 사용량을 모니터링하고,  필요한 경우 리소스 요청과 제한을 조정해야 한다.

### 14.6.1 실제 리소스 사용량 수집과 검색
 - kubelet 내부에 cAdvisor라는 에이전트가 포함되어 있는데 이 에이전트는 노드에서 실행되는 개별 컨테이너와 노드 전체의 리소스 사용 데이터를 수집한다.
 - 물론 전체 클러스터를 이러한 통계를 중앙에서 수집할 수 있는 구성을 해야 한다.(metrics-server 등을 이용할 수 있다.)
 - metrics-server 관련된 자세한 내용은 [https://github.com/kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server) 에서 확인할 수 있다.

### 14.6.2 기간별 리소스 사용량 통계 저장 및 분석
 - 쿠버네티스 클러스터 내에 metric 데이터를 잘 처리하기 위해 인플럭스DB(InfluxDB)와 그라파나(Grafana)를 사용할 수 있다.

#### 인플럭스 DB와 그라파나
 - 인플럭스DB는 애플리케이션 메트릭과 기타 모니터링 데이터를 저장하는 데 이상적인 오픈소스 시계열 데이터베이스이다.

#### 클러스터에서 인플럭스DB와 그라파나 실행
 - minikube로 학습하는 경우 metrics-server addons를 활성화하면 쉽게 테스트를 해볼수 있다.
 - 이를 위한 자세한 내용은 metrics-server를 참조하라.

```  sh
# metrics-server 활성화
minikube addons enable metrics-server

# 쿠버네티스 클러스터 정보 조회
kubectl cluster-info

# 미니쿠베 모니터링 서비스 가동
minikube service monitoring-grafana -n kube-system
```

#### 그라파나로 리소스 사용량 분석하기
 - 차트를 보면 파드에 설정한 리소스 요청 또는 제한을 높여야 하는지 또는 더 많은 파드를 노드에 실행하도록 이들을 낮출수 있는지를 확인할 수 있다.
 - 실제 CPU사용량이 요청한 것보다 높다면, 다른 애플리케이션이 더 많은 CPU를 요청했을 때 해당 애플리케이션의 CPU 시간은 줄어들게 된다.
    - 이런 경우 해당 컨테이너의 CPU 요청을 늘리는 것이 합당하다.
 - 실제 메모리 사용량이 요청된 메모리보다 훨씬 적다면 메모리는 여기서 사용되지 않고, 다른 애플리케이션에서도 사용할 수 없게 된다. 이러면 클러스터 전체적으로 리소스가 낭비된다.
    - 이런 경우 해당 컨테이너의 메모리 요청을 감소하는 것이 합당하다.

## Reference
 - [https://github.com/kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server)
 - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
 - [kubernetes.io](https://kubernetes.io/ko/docs/home/)