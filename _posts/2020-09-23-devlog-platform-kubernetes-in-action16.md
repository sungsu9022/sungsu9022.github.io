---
layout: post
title: "[kubernetes-in-action] 16. 고급 스케줄링"
subtitle: "[kubernetes-in-action] 16. 고급 스케줄링"
categories: devlog
tags: platform
---

# 16. 고급 스케줄링
> 파드 스펙의 노드 셀렉터를 통해 특정 노드로 스케줄링되는 것을 수행할 수 있지만, 이것만으로는 특정 니즈를 처리할 수 없기 때문에 쿠버네티스에서는 이를 확장하는 매커니즘을 제공한다.

## 16.1 테인트와 톨러레이션을 사용해 특정 노드에서 파드 실행 제한
 - 노드 테인트(taint)와 테인트에 대한 파드 톨러레이션(toleration)은 어떤 파드가 특정 노드를 사용할 수 있는지를 제한할 수 있는 기능이다.
 - 노드의 테인트가 허용된(tolerate) 경우에만 파드가 노드에 스케줄링 될수 있음을 뜻한다.

### 테인트와 노드 어피니티 규칙의 차이점 이해
 - 노드 어피니티 규칙을 사용하면 특정 정보를 파드에 추가해 파드가 스케줄링되거나 스케줄링될 수 없는 노드를 선택할 수 있다.
 - 반면 테인트는 기존의 파드를 수정하지 않고, 노드에 테인트를 추가하는 것만으로도 파드가 특정 노드에 배포되지 않도록 한다.

### 16.1.1 테인트와 톨러레이션 소개
 - 클러스터 마스터 노드에 컨트롤 플레인 파드 배치를 어떻게 되어있는지를 살펴보자.

#### 노드의 테인트 구성
 - key, value, effect로 구성되어 있음.

<img width="582" alt="스크린샷 2020-09-22 오후 6 39 58" src="https://user-images.githubusercontent.com/6982740/93866737-080cb080-fd03-11ea-8d75-51fa343a4a25.png">

``` 
# <key>=<value>:<effect>
# key : node-role.kubernetes.io/master
# value : null
# effect : NoSchedule
# 파드가 이 테인트를 허용하지 않는 한 마스터 노드에 스케줄링되지 못하게 막는다는 의미를 가진다.(이 테인트를 허용하는 파드는 주로 시스템 파드에 해당된다.)
node-role.kubernetes.io/master:NoSchedule
```

<img width="554" alt="스크린샷 2020-09-22 오후 6 45 13" src="https://user-images.githubusercontent.com/6982740/93867276-c6c8d080-fd03-11ea-99b9-a27041fd8611.png">

 - 시스템파드의 톨러레이션과 노드의 테인트가 일치하기 때문에 마스터 노드에 스케줄링된다.
 - 톨러레이션이 없는 파드는 테인트가 없는 일반 노드에 스케줄링 된다.

#### 파드의 톨러레이션 표시하기
 - 파드로 실행되는 마스터 노드 컴포넌트도 쿠버네티스 서비스에 접근해야 할 수도 있기 때문에 kube-proxy가 마스터 노드에서도 실행된다. 이 정보를 확인해보면 다음과 같다.

<img width="560" alt="스크린샷 2020-09-22 오후 10 30 02" src="https://user-images.githubusercontent.com/6982740/93888591-2c788500-fd23-11ea-8448-5c829505659a.png">

 - ```node-role.kubernetes.io/master=:NoSchedule``` 톨러레이션은 마스터 노드의 테인트와 일치하므로 kube-proxy 파드가 마스터 노드에 스케줄링되도록 허용한다.
 - ```node.alpha.kubernetes.io/notReady=Exists:NoExecute```와 ```node.alpha.kubernetes.io/unreachable=Exists:NoExecute```는 준비되지 않았거나(notReady), 도달할 수 없는(unreachable)노드에서 파드를 얼마나 오래 실행할 수 있는지를 정의하는 용도의 톨러레이션이다.

#### 테인트 효과 이해
##### NoSchedule
 - 파드가 테인트를 허용하지 않는 경우 노드에 스케줄링되지 않는다.

##### PreferNoSchedule 
 - NoSchedule과 동일하지만, 다른 곳에 스케줄링할 수 없는 상황이면 해당 노드에 스케줄링이 될 수도 있는 효과이다.

##### NoExecute 
 - NoSchedule,PreferNoSchedule 와 달리 이미 실행중인 파드에도 영향을 주고, 해당 노드에서 이미 실행중이지만 해당 테인트를 허용하지 않은 파드는 노드에서 제거되는 효과이다.

### 16.1.2 노드에 사용자 정의 테인트 추가하기
``` sh
# 노드에 테인트  추가
kubectl taint node node1.k8s node-type=production:NoSchedule 

# 파드 생성(deployment)
kubectl run test --image busybox --replicas 5 -- sleep 9999
```

<img width="599" alt="스크린샷 2020-09-22 오후 10 38 51" src="https://user-images.githubusercontent.com/6982740/93889637-67c78380-fd24-11ea-8f6e-bc3bd5a8babd.png">

### 16.1.3 파드에 톨러레이션 추가
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prod
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: prod
    spec:
      containers:
      - args:
        - sleep
        - "99999"
        image: busybox
        name: main
      tolerations:        # 프로덕션 노드에 파드가 스케줄링 될수 있도록 톨러레이션 추가
      - key: node-type
        operator: Equal
        value: production
        effect: NoSchedule
```

<img width="602" alt="스크린샷 2020-09-22 오후 10 40 48" src="https://user-images.githubusercontent.com/6982740/93889860-ac531f00-fd24-11ea-93c7-269ccca2ad2a.png">

 - 의도대로라면 프로덕션 노드인 node1에만 해당 파드가 배포되어야하지만 node2에도 배포되었다. 왜그럴까?
     - 이 부분을 strict하게 막으려면 node2에서 ```node-type=dev:NoSchedule```과 같은 테인트를 추가해야 한다.

### 16.1.4 테인트와 톨러레이션의 활용 방안
 - 노드는 하나 이상의 테인트를 가질 수 있고, 파드는 하나 이상의 톨러레이션을 가질 수 있다.
 - 추가로 톨러레이션 정의에서 Equals 연산자를 사용하는것 외에도 Exists 연산자 등을 활용하여 특정 테인트 키에 여러 값을 허용하도록 할수 있다.

#### 스케줄링에 테인트와 톨러레이션 사용하기
 - 스케줄링을 방지하고(NoSchedule 효과), 선호하지 않는 노드를 정의하고(PreferNoSchedule 효과), 노드에서 기존 파드를 제거하는(NoExecute) 데에도 사용할 수 있다.

#### 노드 실패 후 파드를 재스케줄링하기까지의 시간 설정
 - 톨러레이션을 사용해 쿠버네티스가 다른 노드로 파드를 다시 스케줄링하기 전에 대기해야 하는 시간을 지정할 수도 있다.

<img width="597" alt="스크린샷 2020-09-22 오후 10 47 25" src="https://user-images.githubusercontent.com/6982740/93890704-98f48380-fd25-11ea-8253-420ccbd8af4b.png">

 - 톨러레이션을 별도로 정의하지 않은 파드는 이 두 톨러레이션이 자동으로 추가되고 tolerationSeconds 기본값이 300초인데 이 값을 변경하고 싶다면 파드 스펙에  이 2개의 톨러레이션을 추가한 뒤 지연 시간을 변경할 수 있다.

## 16.2 노드 어피니티를 사용해 파드를 특정 노드로 유인하기
 - 테인트는 파드를 특정 노드에서 떨어뜨려 놓는데 주로 사용한다. 반면 노드 어피니티(node affinity)를 활용해서 특정 노드 집합에만 파드를 스케줄링하도록 지시할 수 있다.

### 노드 어피니티와 노드 셀렉터 비교
 - 노드 셀렉터는 결국 사용이 중단될 것이므로, 새로운 노드 어피니티 규칙을 사용하는것이 권장된다.
 - 노드 어피니티 규칙은 노드 셀렉터와 유사하게 정의가 가능하다.
 - 차이점은 노드 어피니티 규칙을 이용하면 더이상 가용한 노드 리소스가 없어서 파드가 Pending되는 상황을 예방할수 있다는 것이다.
    - 어떤 노드가 특정한 파드를 선호한다는것을 알려주면 쿠버네티스는 해당 노드 중에 하나의 파드를 스케줄링하려고 시도하고, 스케줄링이 불가능할 경우 다른 노드를 선택하도록 할수 있다.

### 디폴트 노드의 레이블 검사
 - 노드 어피니티는 노드 셀렉터와 같은 방식으로 레이블을 기반으로 노드를 선택한다.
 - 디폴트 노드의 레이블과 그 외 다른(사용자가 정의한) 레이블을 파드 어피니티 규칙에 사용할 수 있다.

<img width="602" alt="스크린샷 2020-09-22 오후 11 24 18" src="https://user-images.githubusercontent.com/6982740/93895406-bf68ed80-fd2a-11ea-827a-044c3bbf72ca.png">

#### 디폴트 노드의 레이블들 중 중요한 요소
 - failure-domain-beta.kubernetes.io/region : 리전을 지정
 - failure-domain-beta.kubernetes.io/zone : 가용 영역을 지정
 - kubernetes.io/hostname : 노드의 호스트 이름

### 16.2.1 하드 노드 어피니티 규칙 지정
 - 노드 셀렉터를 이용하는 방식과 노드 어피티니를 이용하는 방식의 차이는 노드 어피니티를 이용하는 방식이 훨씬 복잡하지만 표현력이 풍부하여 노드 셀렉터로 할 수 없는 것들을 할 수 있다.

#### 노드 셀렉터를 이용한 방식
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: luksa/kubia
    name: kubia
```

#### 노드 어피니티 규칙을 이용한 방식
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
  containers:
  - image: luksa/kubia
    name: kubia
```

#### 긴 nodeAffinity 속성 이름에 관하여
 - ... IgnoredDuringExecution : 노드에서 이미 실행중인 파드에는 영향을 미치지 않는다는 의미이다.
 -  현재는 어피니티가 파드 스케줄링에만 영향을 미치고, 파드가 노드에서 제거되지 않는다. 그래서 postfix로 IgnoredDuringExecution가 붙는 속성들이 많은데 추후에는 현존하는 파드에 영향을 미치는 옵션이 추가될 가능성이 있다.

#### nodeSelectorTerm 이해
 - nodeSelectorTerms필드와 matchExpressions필드를 이용해서 파드를 의도한 노드에 스케줄링되도록 정의할 수 있다.

<img width="599" alt="스크린샷 2020-09-22 오후 11 47 52" src="https://user-images.githubusercontent.com/6982740/93898315-0ad0cb00-fd2e-11ea-9229-f7bf647c793d.png">


### 16.2.2 파드의 스케줄링 시점에 노드 우선순위 지정
 - ```preferredDuringSchedulingIgnoredDuringExecution``` 필드를 통해 스케줄러가 선호할 노드를 지정하는 것도 가능하다.
 - 특정 노드 집합에 스케줄링 되는것을 선호하지만, 이게 필수가 아닌 경우라면 이 속성을 사용할 수 있다.

#### 노드 레이블링
 - 먼저 동작 확인을 위해 노드에 레이블링을 실시한다.

<img width="586" alt="스크린샷 2020-09-22 오후 11 50 09" src="https://user-images.githubusercontent.com/6982740/93898622-5daa8280-fd2e-11ea-878b-d4a95adfe7dd.png">

#### 선호하는 노드 어피니티 규칙 지정
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: pref
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: pref
    spec:
      affinity:
        nodeAffinity:        
          preferredDuringSchedulingIgnoredDuringExecution:    # 필수가 아닌 선호 어피니티 속성 지정
          - weight: 80                   # 가중치 80의 availability-zone 선호도 지정
            preference:
              matchExpressions:
              - key: availability-zone
                operator: In
                values:
                - zone1
          - weight: 20                   # 가중치 20의 share-type 선호도 지정
            preference:
              matchExpressions:
              - key: share-type
                operator: In
                values:
                - dedicated
      containers:
      - args:
        - sleep
        - "99999"
        image: busybox
        name: main
```

#### 노드 선호도 작동 방법
 - 이 파드는 어느 노드에나 스케줄링될 수 있지만 파드의 레이블에 따라 특정 노드를 선호하기 때문에 각 노드 그룹별로 우선순위가 존재한다.

<img width="677" alt="스크린샷 2020-09-22 오후 11 53 02" src="https://user-images.githubusercontent.com/6982740/93898954-c4c83700-fd2e-11ea-9449-91d26ca3a495.png">

#### 노드가 두 개인 클러스터에 파드 배포하기
 - 다섯 개의 파드 중 4개는 node1에 배포되었지만 하나는 node2에 배포되었다. 그 이유는 스케줄러가 노드를 결정하는 데 있어서 어피니티 우선순위 지정 기능 외에도 다른 우선순위 지정 기능을 사용했기 때문이다.
    - 이는 Selector-SpreadPriority 기능인데, 동일한 레플리카셋 또는 서비스에 속하는 파드를 여러 노드에 분산시켜 노드 장애로 인해 전체 서비스 중단을 막기 위함이다.

<img width="600" alt="스크린샷 2020-09-22 오후 11 53 54" src="https://user-images.githubusercontent.com/6982740/93899059-e32e3280-fd2e-11ea-8a9e-3624b486c5ed.png">

## 16.3 파드 어피니티와 안티-어피니티를 이용해 파드 함께 배치하기
 - 쿠버네티스에서는 파드-노드간이 아닌 파드-파드간의 어피니티를 지정할 니즈를 처리할 수도 있다.
 - 파드 어피니티(pod affinity)는 2개의 파드를 서로 가깝게 유지하면서 적절한 곳에 배포하기 위해서 사용할 수 있다.

### 16.3.1 파드 간 어피니티를 사용해 같은 노드에 파드 배포하기

#### 대상 파드 생성
``` sh
# app=backend 레이블을 붙인 파드 생성
kubectl run backend -l app=backend --image busybox -- sleep 99999
```

#### 파드 정의에 파드 어피니티 지정
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:   # 필수 파드 어피니티 속성
          - topologyKey: kubernetes.io/hostname             # 동일한 노드에 배포하기 위해 topologyKey를 호스트이름으로 지정
            labelSelector:
              matchLabels:
                app: backend
      containers:
      - name: main
        image: busybox
        args:
        - sleep
        - "99999"
```

 - 위 예제는  app=backend 레이블이 있는 파드와 동일한 노드(topology key 필드로 지정)에 배포되도록 하는 필수 요구 사항을 갖는 파드를 생성하는 매니페스트 정의이다.

<img width="491" alt="스크린샷 2020-09-23 오전 12 01 40" src="https://user-images.githubusercontent.com/6982740/93900103-f8f02780-fd2f-11ea-8645-d08d5a8dd13c.png">

 - 모든 프론트엔드 파드는 백엔드 파드가 스케줄링되는 노드에만 스케줄링되도록 지정한다.
 - matchLabels 대신 matchExpressions 필드를 써도 된다.

#### 파드 어피니티를 갖는 파드 배포
<img width="626" alt="스크린샷 2020-09-23 오전 12 20 45" src="https://user-images.githubusercontent.com/6982740/93902758-e6c3b880-fd32-11ea-93ec-e6c92740cec7.png">

<img width="600" alt="스크린샷 2020-09-23 오전 12 20 50" src="https://user-images.githubusercontent.com/6982740/93902767-e9261280-fd32-11ea-8c69-6a7894230e22.png">


#### 스케줄러가 파드 어피니티 규칙을 사용하는 방법
 - 흥미로운 사실은 파드 어피니티 규칙을 정의하지 않은 백엔드 파드를 삭제하더라도 스케줄러가 백엔드 파드를 node2에 스케줄링 한다는 것이다.(규칙은 프론트엔드 파드에만 있음)
 - 백엔드 파드가 실수로 삭제돼서 다른 노드로 다시 스케줄링된다면 프론트엔드 파드의 어피니티 규칙이 깨지기 때문에 이와 같은 처리를 스케줄러 내부적으로 수행해준다.
 - 자세한 로그를 확인해보면 다음과 같은 절차를 거치게 된다.

<img width="575" alt="스크린샷 2020-09-23 오전 12 25 04" src="https://user-images.githubusercontent.com/6982740/93903142-45893200-fd33-11ea-8190-5010f456a867.png">


### 16.3.2 동일한 랙, 가용 영역 또는 리전에 파드 배포
 - 동일한 머신에서 실행되는 것을 바라는 케이스는 실제로 니즈가 많지는 않을 것이다. 실제로는 같은 가용 영역에서 실행하는 것과 같은 니즈가 훨씬 더 많을 것이다.

#### 동일한 가용 영역에 파드 함께 배포하기
 - 이떄는 topologyKey 속성만 failure-domain.beta.kubernetes.io/zone으로 변경하여 간단히 처리할 수 있다.

#### 동일한 리전에 파드 함께 배포하기
 - 이떄는 topologyKey 속성만 failure-domain.beta.kubernetes.io/region으로 변경하여 간단히 처리할 수 있다.

#### topologyKey 작동 방식
 - topologyKey 작동 방식은 단순히 레이블을 기준으로 동작하고 특별하지 않다.
 - 만약 서버 랙을 기준으로 처리하고 싶다면, 자체적으로 서버랙 label을 정의하고, 이를  topologyKey에 사용해서 쉽게 스케줄링할 수 있다.

<img width="581" alt="스크린샷 2020-09-23 오전 12 29 48" src="https://user-images.githubusercontent.com/6982740/93903653-e5df5680-fd33-11ea-9258-d4f89925e4fb.png">

#### 다른 네임스페이스의 파드 참조방법
 - 기본적으로 레이블 셀렉터는 스케줄링돼 있는 파드와 동일한 네임스페이스의 파드만 일치시킨다.
 - 그러나 lable-Selector와 동일한 레벨에 namespace 필드를 추가하면 다른 네임스페이스에 있는 파드도 선택할 수 있다.

### 16.3.3 필수 요구 사항 대신 파드 어피니티 선호도 표시하기
 - podAffinity의 ```preferredDuringSchedulingIgnoredDuringExecution``` 속성을 이용하여 노드 어피티니 설정한것과 동일한 처리를 할 수 있다.
 - 파드 어피니티에 정의한 가중치대로 스케줄링되는걸 희망하지만, 여의치 않으면 다른곳에 스케줄링되어도 무방하다고 알리는 것이다.

``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:  # 선호 파드 어피니티
          - weight: 80               # 가중치 80의 availability-zone 선호도 지정
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: backend
      containers:
....
```

<img width="512" alt="스크린샷 2020-09-23 오전 12 35 30" src="https://user-images.githubusercontent.com/6982740/93904322-b250fc00-fd34-11ea-9890-3683529acc5f.png">

 - 스케줄러는 프론트엔드 파드를 노드2에 스케줄링하는 것을 선호하지만 노드1에 스케줄링할 수도 있다.

<img width="595" alt="스크린샷 2020-09-23 오전 12 36 05" src="https://user-images.githubusercontent.com/6982740/93904391-c72d8f80-fd34-11ea-9bec-aa956d1f24b5.png">


### 16.3.4 파드 안티-어피니티를 사용해 파드들이 서로 떨어지게 스케줄링하기
 - 파드를 서로 멀리 떨어뜨려 놓고 싶은 경우 파드 안티-어피니티(anti-affinity) 속성을 사용해서 처리할 수 있다.

<img width="491" alt="스크린샷 2020-09-23 오전 12 38 38" src="https://user-images.githubusercontent.com/6982740/93904755-225f8200-fd35-11ea-984a-377c5cb1c56c.png">

 - 위 파드는 app=foo 레이블이 있는 파드가 실행중인 노드로 스케줄링 되지 않는다.

#### 파드 안티-어피니티를 언제 사용하는가?
 - 두 개의 파드 세트가 동일한 노드에서 실행되는 경우 서로의 성능을 방해할수도 있는 경우에 이 기능을 사용할 수 있다.
 - 스케줄러가 동일한 그룹의 파드를 다른 가용 영역 또는 리전에 분산시켜 전체 가용 영역에 장애가 발생해도 서비스가 완전히 중단되지 않도록 하는 경우 사용할 수 있다.

#### 같은 디플로이먼트의 파드를 분산시키기 위해 안티-어피니티 사용
``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: frontend
    spec:
      affinity:
        podAntiAffinity:         
          requiredDuringSchedulingIgnoredDuringExecution:   # 필수의 파드 안티 어피니티 사용
          - topologyKey: kubernetes.io/hostname             # 같은 노드에서 실행되지 않도록 설정
            labelSelector:
              matchLabels:
                app: frontend
      containers:
      - name: main
        image: busybox
        args:
        - sleep
        - "99999"
```

<img width="592" alt="스크린샷 2020-09-23 오전 12 41 54" src="https://user-images.githubusercontent.com/6982740/93905159-9b5ed980-fd35-11ea-83b7-f7afeff07246.png">

 - 자기 자신의 레이블을 가진 파드에 대해서 같은 노드에 스케줄링되지 않도록 안티 어피니티를 지정했기 때문에 node1, node2에 각각 하나씩만 스케줄링 되고 나머지는 pending 상태가 된다.

### 선호하는 파드 안티-어피니티 사용하기
 - ```preferredDuringSchedulingIgnoredDuringExecution``` 속성을 사용해서 동일하게 처리할 수 있다.


## Reference
 - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
 - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
