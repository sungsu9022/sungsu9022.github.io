---
layout: post
title: "[kubernetes-in-action] 15. 파드와 클러스터 노드의 오토스케일링"
subtitle: "[kubernetes-in-action] 15. 파드와 클러스터 노드의 오토스케일링"
categories: devlog
tags: platform
---

# 15. 파드와 클러스터 노드의 오토스케일링
- 스케일을 수동으로 제어하는 건 순간적인 부하(load spikes)를 예측할 수 있거나, 부하가 장시간에 걸쳐 점진적으로 변화하는 경우에는 괜찮다.
- 하지만 갑자기 발생하는 예측할 수 없는 트래픽 증가 현상을 수동으로 개입해 처리하는 것은 이상적이지 않다.
- 쿠버네티스는 파드를 모니터링하다가 CPU 사용량이나 다른 메트릭이 증가하는 것을 감지하는 즉시 확장할 수 있다.
- 클라우드 환경에서 기존 노드가 파드를 더 이상 수용할 수 없을 때 추가 노드를 생성하는 것도 가능하다.

## 15.1 수평적 파드 오토스케일링
 - 수평적 파드 오토스케일링은 컨트롤러가 관리 파드의 레플리카 수를 자동으로 조정하는 것을 말한다.
 - Horizontal 컨트롤러에 의해 수행되며, HorizontalPodAutoscaler 리소스를 작성해 활성화한다.
 - 컨트롤러는 주기적으로 파드의 메트릭(metrics)을 확인해 이 리소스에 설정되어 있는 대상 메트릭 값을 만족하는 레플리카 수를 계산하고 레플리카 수를 조정한다.

### 15.1.1 오토스케일링 프로세스 이해

#### 1) 파드 메트릭 얻기
 - Kubelet에서 실행되는 cAdvisor 에이전트에 의해 수집된다.
 - 이 수집된 데이터를 HorizontalPodAutoscaler에서 가져다 사용한다.
    - 보통 일련의 API 집합(metrics.k8s.io, custom.metrics.k8s.io, external.metrics.k8s.io)에서 메트릭을 가져온다. 
     - metrics.k8s.io API는 대개 별도로 시작해야 하는 메트릭-서버에 의해 제공된다.

##### 메트릭 얻는 부분에 대한 추가내용
 - 힙스터는 Kubernetes v1.11 버전부터 Deprecated되었음.
 - 1.8에서부터는 오토스케일러가 리소스 메트릭의 집계된 버전 API를 통해 얻을 수 있다.
    - 이를 위해서 컨트롤러 매니저에 ```--horizontal-pod-autoscaler-use-rest-clients=true```인자 옵션을 주고 실행할 수 있으며, 1.9부터 기본으로 동작한다고 한다.
 - core API 서버는 메트릭 자체를 노출하지 않는데, 1.7부터 다중 API 서버를 등록하고 한 API서버를 통해서만 메트릭을 노출할 수 있음.(18장 API 서버 애그리게이션에 추가 설명)
 - 클러스터 관리자가 어떤 메트릭 수집기를 사용할지 결정할 수 있으며, 메트릭을 적절한 API 경로와 형식으로 노출하려면 변환 레이어가 필요하다고 함.

#### 2) 필요한 파드수 계산
 - 파드의 모든 메트릭을 가지고 있으면 필요한 레플리카 수를 파악할 수 있다.
 - 메트릭의 평균 값을 이용해 지정한 목표 값과 가능한 가깝게 하는 숫자를 찾는 방식이다.
    - 실제 계산에서는 메트릭 값이 불안정한 상태에서 빠르게 변하는 상황 등 더 복잡하지만 일반적으로 저런식으로 동작한다.

<img width="602" alt="스크린샷 2020-09-16 오후 11 34 36" src="https://user-images.githubusercontent.com/6982740/93351885-33ac1880-f875-11ea-864f-f4f9d7351184.png">
 
 - 위 예제에서는 HorizontalPodAutoscaler에 설정한 메트릭은 CPU와 QPS인데
    - 목표 CPU 사용량이 50%이고, 3개의 파드의 CPU사용량의 합을 구한뒤 목표값으로 나눈 값이 레플리카수 후보가 된다.
    - 목표 QPS는 20이고, 3개의 파드의 QPS의 합을 구한 뒤 목표값으로 나눈 값이 레플리카수 후보가 된다.
    - 이렇게 2개의 후보 중 MAX을 기준으로 레플리카수가 결정된다.


#### 3) 스케일링된 리소스의 레플리카 수 갱신
 - 수평적 파드 오토스케일러는 스케일 서브 리소스만을 수정한다.(디플로이먼트와 같은)
 - 모든 확장 가능한 리소스를 대상으로 오토스케일링이 가능하다.
    - 디플로이먼트
    - 레플리카셋(레플리케이션컨트롤러)
    - 스테이트풀셋

<img width="583" alt="스크린샷 2020-09-16 오후 11 38 08" src="https://user-images.githubusercontent.com/6982740/93352351-b339e780-f875-11ea-943e-535cc5ecd263.png">

#### 전체 오토스케일링 과정 이해
 - 각 구성요소는 메트릭을 다른 구성요소에서 주기적으로 가져온다.(무한 루프 안에서 동작)
 - 하지만 실제로 메트릭 데이터는 전파되고, 재조정 작업이 수행되기까지는 시간이 걸린다.(즉각적으로 이루어지는게 아님.)

<img width="689" alt="스크린샷 2020-09-16 오후 11 39 03" src="https://user-images.githubusercontent.com/6982740/93352445-cfd61f80-f875-11ea-9194-2cb8320859bf.png">

### 15.1.2 CPU 사용률 기반 스케일링
 - CPU 사용량이 100%에 도달하면 더 이상 요구에 대응할 수 없어 스케일 업(수직적 스케일링, 파드의 CPU 양 증가와 같은)이나 스케일 아웃(수평적 스케일링, 파드수 증가)이 필요하다.
 - CPU가 치솟는 상황에서 스케일 아웃이 이루어지면 평균 CPU사용량은 감소할 것이다.
 - CPU 사용량을 기존으로 오토 스케일링을 설정할 때는 100%에 훨씬 못미치게 설정하는 것이 좋다.
    - CPU사용량이 고정적이지 않고 불안정하므로 완전히 바쁜상태에 도달하기 전에 스케일아웃이 이루어지는것이 안전함.
    - 갑자기 순간적인 부하가 발생하는 것을 제어하는 데 필요한 공간을 확보해야 한다.

#### CPU사용률을 판단하는 기준
 - 오토슼일러에 한해서 파드의 CPU 사용률을 결정할 때는 파드가 요청을 통해 보장받은 CPU 기준 사용량만을 이용한다.
 - 이 동작방식때문에 오토스케일링이 필요한 파드는 직접 또는 간접적으로 CPU 요청을 설정해야 한다.(LimitRange 오브젝트를 이용하는것이 안전할 것.)

#### CPU 사용량을 기반으로 HorizontalPodAutoscaler 생성
 - 오토스케일링 대상은 디플로이먼트를 기준으로 하는 것이 좋다.(수동/자동 마찬가지)
    - 애플리케이션 업데이트시에도 원하는 레플리카 수를 계속 유지할 수 있기 때문

##### 오토스케일링을 위한 deployment 준비
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v1
        name: nodejs
        resources:
          requests:
            cpu: 100m
  selector:
    matchLabels:
      app: kubia
```

##### minikube metrics-server 설정
``` sh
minikube addons list

minikube addons enable metrics-server
```


##### HorizontalPodAutoscaler 생성
 - kubia deployment 대상
 - cpu 사용률 30% 유지
 - 레플리카수 1 ~ 5

``` sh
# HorizontalPodAutoscaler 생성
kubectl autoscale deployment kubia --cpu-percent=30 --min=1 --max=5

# HorizontalPodAutoscaler 정보 조회
kubectl get hpa.v2beta1.autoscaling kubia -o yaml
```

<img width="739" alt="스크린샷 2020-09-16 오후 11 59 03" src="https://user-images.githubusercontent.com/6982740/93355105-aa96e080-f878-11ea-915a-f7b74c472b3f.png">

 - 최소, 최대 레플리카수, CPU 메트릭에 대한 정보, 현재 오토스케일러의 상태 등을 확인할 수 있다.


#### 최초 오토 리스케일 이벤트 보기

``` sh
# 오토스케일러 조회(레플리카수가 1인것을 확인할 수 있다.)
kubectl get hpa

# deployment 조회
kubectl get deploy

# pod 조회
kubectl get pod
```

 - 로컬에서 생성된 파드는 CPU 사용량이 0에 가깝기 때문에 오토스케일러가 파드 하드를 하나만 남도록 했다.

<img width="880" alt="스크린샷 2020-09-17 오전 12 51 36" src="https://user-images.githubusercontent.com/6982740/93361558-f5682680-f87f-11ea-9a92-f96d85bc6a3a.png">

``` sh
# hpa 상태 조회
kubectl describe hpa kubia
```

<img width="1277" alt="스크린샷 2020-09-17 오전 12 52 20" src="https://user-images.githubusercontent.com/6982740/93361760-2d6f6980-f880-11ea-818d-5198f2a4cf91.png">

 - 초반에 metric-server 설정을 안킨 상태로 해서 그런건지는 모르겠으나 스케일다운에 대한 이벤트가 안보이네요..

#### 스케일 아웃 일으키기
``` sh
#  service 생성
kubectl expose deployment kubia --port=80 --target-port=8080

# deploy watch
kubectl get deploy --watch

# hpa watch
kubectl get hpa --watch

# 부하를 줄수있는 일회성 파드 생성
kubectl run -it --rm --restart=Never loadgenerator --image=busybox -- sh -c "while true; do wget -O - -q http://kubia.default; done"
```

#### 오토스케일러가 디플로이먼트를 스케일 아웃하는지 확인
 - 이때 CPU는 컨테이너에서 사용 가능한 최대 사용량이 아닌 최소 사용량의 정의이고, 이를 넘어설수 있기 떄문에 컨테이너의 요청 CPU보다 더 많은 CPU를 소비할 수 있다.(100% 넘는것도 가능하나 로컬에서는 그렇게까지는 안나왔음.)

##### watch 내역
<img width="1276" alt="스크린샷 2020-09-17 오전 1 13 40" src="https://user-images.githubusercontent.com/6982740/93364044-08c8c100-f883-11ea-9019-1aa6b99a7b6b.png">

##### hpa event 확인
<img width="1279" alt="스크린샷 2020-09-17 오전 1 12 23" src="https://user-images.githubusercontent.com/6982740/93363923-e171f400-f882-11ea-86a1-6dd3c32f879a.png">


#### 최대 스케일링 비율 이해
 - 한번 확장 단계에서 추가할 수 있는 레플리카의 수는 제한되어 있다.
 - 오토스케일러는 2개를 초과하는 레플리카가 존재할 경우 한 번의 수행에서 최대 두배의 레플리카를 생성할 수 있다.
    - 2개의 파드가 있었을때 한번의 오토스케일링에서 2배인 4개까지 생성될 수 있다는 의미
 - 스케일 아웃 3분 단위로 리스케일링이 일어난다.
 - 스케일 인은 5분 간격으로 일어난다.

#### 기존 HPA 오브젝트에서 목표 메트릭 값 변경
 - 리소스를 변경하면 오토스케일러 컨트롤러에 의해 감지돼 동작한다.(다른 리소스들과 동일한 셈)
 - HPA 리소스를 삭제하는 것은 대상 리소스의 오토스케일링을 비활성화하는것 뿐. 레플리카수는 종료시점 그대로 계속 유지된다.

### 15.1.3 메모리 소비량에 기반을 둔 스케일링
 - 메모리 기반 오토스케일링은 CPU 기반 오토스케일링에 비해 훨씬 문제가 많다.
 - 스케일 업 후에 오래된 파드는 어떻게든 메모리를 해제하는 것이 필요하기 떄문이다. 
     - 그러나 메모리 해제는 파드 컨테이너 내부의 애플리케이션이 해야 하는 일이기 때문에 애플리케이션을 종료하고 다시 시작해야 한다.(원하는 방식이 아님)

#### 쿠버네티스 1.8 메모리 기반 오토스케일링
##### 공식 문서
 - ```메모리 및 사용자 정의 메트릭에 대한 스케일링 지원을 포함하는 베타 버전은 autoscaling/v2beta2에서 확인할 수 있다.``` 라고만 공식 document에 나와 있다.

##### 블로그(hpa를 사용하지 않고 직접 스크립트를 만들어 처리하는 방법)
 - [https://www.powerupcloud.com/autoscaling-based-on-cpu-memory-in-kubernetes-part-ii/](https://www.powerupcloud.com/autoscaling-based-on-cpu-memory-in-kubernetes-part-ii/)

##### google cloud 가이드
  - [https://cloud.google.com/kubernetes-engine/docs/how-to/horizontal-pod-autoscaling?hl=ko](https://cloud.google.com/kubernetes-engine/docs/how-to/horizontal-pod-autoscaling?hl=ko)

``` yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 100Mi
  # Uncomment these lines if you create the custom packets_per_second metric and
  # configure your app to export the metric.
  # - type: Pods
  #   pods:
  #     metric:
  #       name: packets_per_second
  #     target:
  #       type: AverageValue
  #       averageValue: 100
```

##### autoscaler github
 - [https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
 - 현재 알려진 제약사항으로 CPU 또는 메모리 기반  HPA와 VPA(Vertical Pod Autoscaler)를 함께 쓰지 말라고 하네요.

### 15.1.4 기타 그리고 사용자 정의 메트릭 기반 스케일링
 - 과거 CPU 기반 오토스케일링을 넘어 다른 메트릭을 사용하도록 하는 것이 쉽지 않아 쿠버네티스 오토스케일링 SIG에서 완전히 다시 설계한 이력이 있음.
 - 사용자 정의 메트릭을 사용하는 방법이 어렵다고 한 저자의 블로그 글 : [https://medium.com/@marko.luksa/kubernetes-autoscaling-based-on-custom-metrics-without-using-a-host-port-b783ed6241ac](https://medium.com/@marko.luksa/kubernetes-autoscaling-based-on-custom-metrics-without-using-a-host-port-b783ed6241ac)

#### 메트릭 유형으로 들어갈 수 있는 항목
##### 리소스  : 리소스 메트릭 기반(CPU 등)

<img width="524" alt="스크린샷 2020-09-17 오전 1 45 40" src="https://user-images.githubusercontent.com/6982740/93367484-81318100-f887-11ea-87d8-d271cf0ac766.png">

##### 파드 
 - 초당 질의수(QPS), 메시지 브로커의 큐 메시지 수 같은 것을 사용할 수 있다.

<img width="508" alt="스크린샷 2020-09-17 오전 1 46 45" src="https://user-images.githubusercontent.com/6982740/93367604-a6be8a80-f887-11ea-92da-1bcd1dd8e77f.png">

 - QPS가 100을 유지하도록 설정


##### 오브젝트
 
<img width="642" alt="스크린샷 2020-09-17 오전 1 47 07" src="https://user-images.githubusercontent.com/6982740/93367643-b5a53d00-f887-11ea-932f-8145befe1628.png">

 - 인그레스 오브젝트의 latencyMillis 메트릭을 사용하고, 이 값을 20이 되도록 유지한다. 오토스케일러는 인그레스의 메트릭을 모니터링 하다가 해당 값이 높이 올라갈 경우 디플로이먼트 리소스를 확장한다.

### 15.1.5 오토스케일링에 적합한 메트릭 결정
 - 파드에 있는 컨테이너의 메모리 소비량은 오토스케일링에 적합한 메트릭이 아니다. 레플리카 수를 늘려도 관찰중인 메트릭의 평균값이 선형적으로 감소하지 않거나 최소한 선형에 가깝게 줄어들지 않는다면 오토스케일러가 제대로 동작하지 않는다.
 - 하나의 파드 인스턴스와 메트릭 X 값이 있는 상황에서 오토스케일러가 레플리카를 2개로 확장했을때 메트릭 X 값이 X/2에 가깝게 줄어들때 의미가 있다.(예 QPS)
 - 오토스케일러 메트릭을 정할때는 해당 메트릭 값이 파드 수 증감에 어떻게 변화하는지를 보고 판단해야 한다.

### 15.1.6 레플리카를 0으로 감소
 - 수평적 파드 오토스케일러는 현재 minReplicas 필드로 0으로 설정할 수 없다. 즉 파드 수가 0으로 감소할 일은 없다.
 - 쿠버네티스에 아직 유휴(idling), 유휴 해제(un-idling)이 구현되지는 않았는데 레드햇 오픈시프트에서 구현하고 있다고 한다.

## 15.2 수직적 파드 오토스케일링
 - 수평적 확장이 불가능한 애플리케이션의 경우 유일한 옵션은 수직적으로 확장하는 것이다.(더 많은 CPU, 메모리 리소스 부여)
 - 현재 베타 기능으로 별도로 추가 설치를 해야 함.
 - 자세한 내용은 구글 문서 참조 : [https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler?hl=ko](https://cloud.google.com/kubernetes-engine/docs/concepts/verticalpodautoscaler?hl=ko)
     - github보다 설정에 대한 자세한 정보가 여기 더 많음.

``` yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       my-app
  updatePolicy:
    updateMode: "Auto"
```

### 15.2.1 리소스 요청 자동 설정
 - 새로 생성한 파드의 컨테이너가 CPU와 메모리 요청을 명시적으로 정의하고 있지 않은 경우에 이를 설정한다. 이는 IntialResoruces라는 어드미션컨트롤러 플러그인에 의해 제공된다.
 - 새로운 파드가 리소스 요청 없이 생성되면 플러그인은 파드의 컨테이너의 과거 리소스 사용량 데이터를 살펴보고 요청값을 적절하게 설정한다.
 - 리소스 요청 값을 지정하지 않고 파드를 배포한 뒤 쿠버네티스에 의존해 컨테이너 리소스 요구사항을 파악할 수도 있다.
 - 이를 통해 쿠버네티스는 효과적으로 파드의 수직적 확장을 수행한다.

> 이 내용은 공식문서나 레퍼런스가 없음(?)

### 15.2.2 파드가 실행되는 동안 리소스 요청 수정
 - 실행중인 파드를 수직적으로 확장하는 기능
 - 구글 문서 기준으로 "updateMode"을 조정해서 할수 있는 것으로 보인다.
    -  updateMode 필드 값은 Auto이면 VerticalPodAutoscaler가 Pod 수명 동안 CPU 및 메모리 요청을 업데이트할 수 있다는 의미입니다. 즉, VerticalPodAutoscaler는 Pod를 삭제하고 CPU 및 메모리 요청을 조정한 후에 새 Pod를 시작할 수 있습니다.

## 15.3 수평적 클러스터 노드 확장
 - 모든 노드가 한계에 도달해 더 이상 파드를 추가할 수 없는 상황에서는 노드가 추가되어야 하는데 이를 쿠버네티스가 제공해준다.
 - 쿠버네티스는 추가적인 노드가 필요한 것을 탐지하면 가능한 빠르게 추가 노드를 클라우드 제공자(provider)에게 요청하는 기능을 가지고 있다.

### 15.3.1 클러스터 오토스케일러
 - 노드에 리소스가 부족해서 스케줄링할 수 없는 파드를 발견하면 추가 노드를 자동으로 공급한다.
 - 또한 오랜 시간 동안 사용률이 낮으면 노드를 줄인다.

#### 클라우드 인프라스트럭처에 추가 노드 요청
 - 노드를 추가를 수행하기 전에 새로운 노드가 파드를 수용할 수 있는지 확인하는 절차를 거쳐야 한다.
 - 그리고 동일한 크기의  노드들은 그룹으로 묶여야 하고, 클러스터 오토스케일러는 단순히 추가 노드를 달라고 말할 수 없으며, 어떤 그룹의 노드 유형인지 지정해야 한다.
 - 클러스터 오토스케일러는 사용 가능한 노드 그룹을 검사해 노드 유형에서 스케줄링되지 않은 파드를 수용할수 있는지 확인하고, 그 노드 유형 중 하나를 선택해서 노드 그룹을 스케일아웃한다.
 - 새 노드가 시작되면 해당 노드의 kubelet이 API서버에 접속해 노드 리소스를 만들어 노드를 등록한다.

<img width="617" alt="스크린샷 2020-09-17 오전 2 15 13" src="https://user-images.githubusercontent.com/6982740/93370341-a1633f00-f88b-11ea-96e7-2cd79648333e.png">

#### 노드 종료
 - 클러스터 오토스케일러는 노드 리소스가 충충분히 활용되고 있지 않을 때 노드 수를 줄여야 한다.
 - 기본 원칙은 특정 노드에서 실행중인 모든 파드의 CPU와 메모리 요청이 50% 미만이면 해당 노드는 필요하지 않은 것으로 간주한다.
 - 클러스터 오토스케일러가 노드에서 실행중인 파드가 다른 노드로 다시 스케줄링될 수 있다는 것을 아는 경우에만 해당 노드를 반환할 수 있다.
     - 해당 노드에서만 실행중인 시스템 파드가 있ㄴ는 경우에는 해당 노드를 종료할 수 없다.
     - 또 관리되지 않는 파드, 로컬 저장소를 가진 파드가 실행되는 경우에도 종료될 수 없다.
 - 노드를 종료할때는 먼저 해당 노드에 스케줄링할 수 없다는 표시를 하고나서 파드를 제거한 뒤 진행된다.(이 과정에서 종료할 노드에 다른 파드가 스케줄링 되면 안되기 떄문에 이런 마킹을 처리한다.

#### 수동으로 노드 금지(cordoning), 배출(draining) 하기
 - ```kubectl cordon <node>``` 명령을 통해 스케줄링할 수 없음을 표시할 수 있다.
 - ```kubectl drain <node>``` 명령을 통해 스케줄링할 수 없음을 표시하고, 노드에서 실행중인 모든 파드를 종료할 수 있다.
 - ```kubectl uncordon <node>```을 통해 금지 해제를 할수 있다.


### 15.3.3 클러스터 스케일 다운 동안에 서비스 중단 제한
 - 특정 서비스에서는 최소 개수의 파드가 항상 실행되어야 한다. 이는 쿼럼(quorum) 기반 클러스터 애플리케이션인 경우 특히 그렇다.(합의 알고리즘 등)
 - 쿠버네티스는 이런 문제해결을 위해 스케일 다운 등의 작업을 수행할 경우에도 유지되어야 하는 최소 파드 개수를 지정하는 PodDisruptionBudget 리소스를 제공한다.

``` sh
# PodDisruptionBudget 생성
kubectl create pdb kubia-pdb --selector=app=kubia --min-available=3
``` 

<img width="353" alt="스크린샷 2020-09-17 오전 2 25 23" src="https://user-images.githubusercontent.com/6982740/93371278-0e2b0900-f88d-11ea-9e98-5bbaeb6e7612.png"> 
 
 - minAvailable 필드에 절대값이 아닌 백분율을 사용할 수도 있다. 
 - maxUnavailable 필드를 이용하면 minAvailable  대신 일정 개수 이상의 파드가 종료되지 않도록 지정할 수도 있다.


## Reference
 - [https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/)
 - [https://www.powerupcloud.com/autoscaling-based-on-cpu-memory-in-kubernetes-part-ii/](https://www.powerupcloud.com/autoscaling-based-on-cpu-memory-in-kubernetes-part-ii/)
 - [https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
 - [https://cloud.google.com/kubernetes-engine/docs/how-to/horizontal-pod-autoscaling?hl=ko](https://cloud.google.com/kubernetes-engine/docs/how-to/horizontal-pod-autoscaling?hl=ko)
 - [https://medium.com/@marko.luksa/kubernetes-autoscaling-based-on-custom-metrics-without-using-a-host-port-b783ed6241ac](https://medium.com/@marko.luksa/kubernetes-autoscaling-based-on-custom-metrics-without-using-a-host-port-b783ed6241ac)
 - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
 - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
