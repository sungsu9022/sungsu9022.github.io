---
title: "[kubernetes-in-action] 3. 파드 : 쿠버네티스에서 컨테이너 실행"
author: sungsu park
date: 2020-08-03 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 3. 파드 : 쿠버네티스에서 컨테이너 실행
## 3.1 파드 소개
 - 파드는 함께 배치된 컨테이너 그룹이며, 쿠버네티스의 기본 빌딩 블록.
 - 컨테이너를 개별적으로 배포하기보다는 컨테이너를 가진 파드를 배포하고, 운영한다.
 - 무조건 2개 이상을 컨테이너를 포함시키라는 의미는 아니고 일반적으로는 하나의 컨테이너만 포함된다.
 - 파드의 핵심은 파드가 여러 컨테이너를 가지고 있을 경우 모든 컨테이너는 항상 하나의 워커 노드에서 실행되고, 여러 워커 노드에 걸쳐 실행되지 않는다는것.

<img width="624" alt="스크린샷 2020-08-04 오전 1 40 29" src="https://user-images.githubusercontent.com/6982740/89205974-88f8d700-d5f3-11ea-9b35-03282b3c5f3e.png">


### 3.1.1 파드가 필요한 이유
- 여러 프로세스를 실행하는 단일 컨테이너보다 다중 컨테이너가 좋기 때문
 - 쿠버네티스에서는 프로세스를 항상 컨테이너에서 실행시키고, 각각의 컨테이너는 격리된 머신과 비슷하다. 그래서 여러 프로세스를 단일 컨테이너 안에서 실행하는것이 타당한것처럼 보일수 있는데.
 - 컨테이너는 단일 프로세스를 실행하는 것을 목적으로 설계되었음.
 - 그래서 각 프로세스를 자체의 개별 컨테이너로 실행해야 한다.
 - 하지만 실제 서비스 상황에서는 여러개의 컨테이너를 포함시킬 수 있는 개념이 필요할 수 있음.
    - 예를 들면 WAS를 쿠버네티스로 서빙한다고 했을떄 쿠버네티스 스케줄러에 의해서 pod이 존재하는 노드는 언제든 수시로 변경될 수 있는데 이 경우 로그가 유실될 수 있기 때문에 이런 로그를 유실되지 않기 위해서는 로그를 중앙저장소로 보내주는 별도의 프로세스가 있어야 함. 이 경우 파드 구성을 WAS 컨테이너 + 로그전송 프로세스 컨테이너로 하여 하기 위함이 아닌가 싶음.

### 3.1.2 파드 이해하기
 - 여러 프로세스를 단일 컨테이너로 묶지 않기 때문에 컨테이너를 함께 묶고 하나의 단위로 관리할 수 있는 또 다른 상위 구조가 필요하다. (파드가 필요한 이유)
 -  컨테이너가 제공하는 모든 기능을 활용하는 동시에 프로세스가 함께 실행되는것처럼 보이게 할수 있음.

#### 같은 파드에서 컨테이너 간 부분 격리
 - 쿠버네티스는 파드 안에 있는 모든 컨테이너가 자체 네임스페이스가 아닌 동일한 리눅스 네임스페이스를 공유하도록 도커를 설정
 - 즉 동일한 네트워크 네임스페이스, UTS(Unix Timesharing System, 호스트 이름을 네임스페이스로 격리) 안에서 실행되므로 같은 파드 내에서 이것들을 서로 공유할 수 있다.
 - 여기서 주의할 점은 파일시스템같은 경우는 컨테이너 이미지에서 나오기 때문에 기본적으로 완전히 분리된다고 보면 된다.
 - 쿠버네티스 볼륨 개념을 이용하면 컨테이너간 파일 디렉터리를 공유하는것도 가능

#### 컨테이너가 동일한 IP와 포트 공간을 공유하는 방법
 - 파드 안의 컨테이너가 동일한 네트워크 네임스페이스에서 실행되기 때문에 도일한 IP주소와 포트공간을 공유한다.
 - 그래서 동일한 파드 안 컨테이너에서 실행되는 프로세스들은 같은 포트번호를 사용하지 않도록 해야 한다.
 - 파드 안의 컨테이너들끼리는 로컬호스트를 통해 서로 통신이 가능(동일한 루프백 네트워크 인터페이스를 갖음)

#### 파드간 플랫 네트워크 소개
 - 모든 파드는 하나의 플랫한 공유 네트워크 주소 공간에 상주
 - 모든 파드는 다른 파드의 IP주소를 사용해 접근이 가능하다.
 - LAN과 유사한 방식으로 상호 통신이 가능
 - 파드는 논리적인 호스트로서 컨테이너가 아닌 환경에서의 물리적 호스트 혹은 VM과 매우 유사하게 동작함.

<img width="569" alt="스크린샷 2020-08-04 오전 1 39 48" src="https://user-images.githubusercontent.com/6982740/89205891-68308180-d5f3-11ea-8c93-1449b61e04ea.png">


### 3.1.3 파드에서 컨테이너의 적절한 구성

#### 다계층 애플리케이션을 여러 파드로 분할
 - 프론트엔드와 백엔드 예제에서 파드를 2개로 분리하면 쿠버네티스가 프론트엔드를 한 노드로 백엔드는 다른 노드에 스케줄링해서 인프라스트러겨의 활용도를 향상시킬수 있음.
 - 이게 만약 같은 파드에 있다면 둘중 항상 같은 노드에서 실행되겠지만, 서로 필요한 리소스의 양은 다를수 있기에 분리하는것이 좋다.

#### 개별 확장이 가능하도록 여러 파드로 분할
 - 두 컨테이너를 하나의 파드에 넣지 말아야하는 이유 중 하나는 "스케일링" 때문
 - 쿠버네티스 스케줄링 기본 단위는 파드이다.
 - 쿠버네티스에서는 컨테이너 단위로 수평 확장할수 없고 파드 단위로만 수평확장(scale out)이 가능하다.
 - 컨테이너를 개별적으로 스케일링하는 것이 필요하다고 판단되는 경우 별도 파드에 배포하자.

#### 파드에서 여러 컨테이너를 사용하는 경우
 - 애플리케이션이 하나의 주요 프로세스와 하나 이상의 보안 프로세스로 구성된 경우
 - 특정 디렉터리에서 파일을 제공하는 웹서버로 예를 들자면 주 컨테이너는 웹서버라고 하고, 추가 컨테이너(사이드카 컨테이너)는 외부 소스에서 주기적으로 콘텐츠를 받아 웹서버 디렉터리에 저장하는 방식을 예로 들수 있다.(ex. sitemap을 받아와서 디렉터리에서 저장하는 것과 같은)
 - 다른 에제로는 로그 로테이터와 수집기, 데이터 프로세서, 통신 어댑터 등이 있을수 있다.

<img width="573" alt="스크린샷 2020-08-04 오전 1 38 03" src="https://user-images.githubusercontent.com/6982740/89205749-2a335d80-d5f3-11ea-9b3d-825fe1bfb0b4.png">

#### 파드에서 여러 컨테이너를 사용하는 경우 결정
 - 컨테이너를 함께 실행해야 하는가? 혹은 서로 다른 호스트에서 실행할 수 있는가?
 - 여러 컨테이너가 모여 하나의 구성 요소로 나타내는가? 혹은 개별적인 구성 요소인가?
 - 컨테이너가 함꼐, 혹은 개별적으로 스케일링돼야 하는가?
 - 컨테이너는 여러 프로세스를 실행시키지 말아야하고, 파드는 동일한 머신에서 실행할 필요가 없다면 여러 컨테이너를 포함하지 말아야 한다.

## 3.2 YAML 또는 JSON 디스크립터로 파드 생성
 - kubectl run 명령어를 이용해서 리소스를 만들수도 있지만, yaml 파일에 모든 쿠버네티스 오브젝트를 정의하면 버전 관리 시스템에 넣는것도 가능하고, 모든 이점을 누릴수 있다.

### 3.2.1 기존 파드의 YAML 디스크립터 살펴보기
``` yaml
# 메타 데이터
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-08-03T14:21:44Z"
  generateName: kubia-
  labels:
    run: kubia
  name: kubia-5wsx8
  namespace: default
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicationController
    name: kubia
    uid: cb311984-2974-4434-8979-b7a11f07388d
  resourceVersion: "4551"
  selfLink: /api/v1/namespaces/default/pods/kubia-5wsx8
  uid: ed4f042e-f547-4003-9a89-8005bca02f60
# 스펙
spec:
  containers:
  - image: sungsu9022/kubia
    imagePullPolicy: Always
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-zszt7
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: docker-desktop
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-zszt7
    secret:
      defaultMode: 420
      secretName: default-token-zszt7
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2020-08-03T14:21:44Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2020-08-03T14:21:49Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2020-08-03T14:21:49Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2020-08-03T14:21:44Z"
    status: "True"
    type: PodScheduled
# Status
  containerStatuses:
  - containerID: docker://796528475e929b9f1ec28fc5ae3ca722f1279b8f852b262dfef549bbdebdcd7a
    image: sungsu9022/kubia:latest
    imageID: docker-pullable://sungsu9022/kubia@sha256:3811562a9c1c45f2e03352bf99cacf4e024f9c3a874a3d862f0208367eace763
    lastState: {}
    name: kubia
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2020-08-03T14:21:48Z"
  hostIP: 192.168.65.3
  phase: Running
  podIP: 10.1.0.13
  podIPs:
  - ip: 10.1.0.13
  qosClass: BestEffort
  startTime: "2020-08-03T14:21:44Z"

```

#### 파드를 정의하는 주요 부분 소개
 - Metadata : 이름, 네임스페이스, 레이블 및 파드에 관한 기타 정보를 포함
 - Spec : 파드 컨테이너, 볼륨, 기타 데이터 등 파드 자체에 관한 실제 명세를 가진다.
 - Status : 파드 상태, 각 컨테이너 설명과 상태, 파드 내부 IP, 기타 기본 정보 등 현재 실행중인 파드에 관한 정보 포함(새 파드를 만들때는 작성하지 않는 내용)

### 3.2.2 파드를 정의하는 간단한 YAML 정의
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual  # 파드 이름
spec:
  containers:
  - image: luksa/kubia # 컨테이너 이미
    name: kubia # 컨테이너 이름
    ports:
    - containerPort: 8080 # 애플리케이션 수신 포트
      protocol: TCP
```

#### 컨테이너 포트 지정
 - 스펙에 포트를 명시하지않아도 해당 파트에 접속할 수 있으나, 명시적으로 정의해주면 클러스터를 사용하는 모든 사람이 파드에 노출한 포트를 빠르게 볼 수 있어서 정의하는 것이 좋음.

#### 오브젝트 필드들에 대한 설명 확인(도움말같은)
``` sh
kubectl explain pods
kubectl explain pod.spec
```

### 3.2.3 kubectl create 명령으로 파드 만들기
 - yaml 또는 json 파일로 리소스를 동일하게 만들 수 있음.

``` sh
kubectl create -f kubia-manual.yaml
```

#### 실행중인 파드의 전체 정의 가져오기
``` sh
kubectl get po kubia-manual -o yaml
kubectl get po kubia-manual -o json
```

### 3.2.4 애플리케이션 로그 보기
 - 컨테이너 런타임(도커)은  스트림을 파일로 전달하고 아래 명령을 이용해 컨테이너 로그를 가져온다.

``` sh
docker log <container id>
```

 - 쿠버네티스 logs를 이용해 파드 로그 가져오기
``` sh
kubectl logs kubia-manual
```

 - 파드에 컨테이너가 하나만 있다면 로그를 가져오는 것은 매우 간단함
 - 컨테이너 로그는 하루 단위로, 로그 파일이 10MD 크기에 도달할때마다 로테이션 되기 떄문에 별도로 로그 관리하는 방식이 필요할듯

#### 컨테이너 이름을 지정해 다중 컨테이너 파드에서 로그 가져오기
 ``` sh
kubectl logs kubia-manual -c kubia
```

 - 현재 존재하는 파드의 컨테이너 로그만 가져올수 있음.
 - 파드가 삭제되면 로그도 같이 삭제된다.
 - 일반적인 서비스 환경에서는 클러스터 전체의 중앙집중식 로깅을 설정하는 것이 좋음

### 3.2.5 파드에 요청 보내기
 - 기존에 ```kubectl expose``` 명령으로 외부에서 파드에 접속할 수 있는 서비스를 만들었었는데
 - "포트 포워딩" 방식을 이용해 테스트와 디버깅 목적으로 연결할수 있음.

#### 로컬 네트워크 포트를 파드의 포트로 포워딩
 - 서비스를 거치지 않고 특정 파드와 통신하고 싶을때 쿠버네티스는 해당 파드로 향하나느 포트 포워딩을 구성해준다.
 - 아래와 같인 하면 localhost:8888로 해당 파드의 8080포트와 연결시킬 수 있다.

``` sh
kubectl port-forward kubia-manual 8888:8080
```

<img width="619" alt="스크린샷 2020-08-04 오전 1 41 51" src="https://user-images.githubusercontent.com/6982740/89206075-afb70d80-d5f3-11ea-8fb4-68cb03125e1e.png">



## 3.3 레이블을 이용한 파드 구성
 - 파드 수가 증가함에 따라 파드를 부분집합으로 분류할 필요가 있음.
 - MSA에서 배포된 서비스가 매우 많고 여기서 여러 버전 혹은 릴리스(안정, 베타 카나리 등)이 동시에 실행될수도 있는데 이러면 시스템에 수백개의 파드가 생길수 있고 이를 정리할 매커니즘이 필요한데 이것이 레이블이다.
 - 레이블을 통해 파드와 기타 다른 쿰버네티스 오브젝트의 조직화를 할 수 있음(파드 뿐만 아니라 다른 쿠버네티스 리소스 가능)

<img width="695" alt="스크린샷 2020-08-04 오전 1 38 37" src="https://user-images.githubusercontent.com/6982740/89205789-3b7c6a00-d5f3-11ea-92af-77e37a044e09.png">


### 3.3.1 레이블 소개
 - 파드와 모든 쿠버네티스 리소스를 조직화할 수 있는 단순하면서 강력한 기능
 - 리소스에 첨부하는 키-값 쌍
 - 레이블 셀렉터를 사용해 리소스를 선택할때 호라용
 - 레이블 키가 해당 리소스 내에서 고유하다면 하나 이상 원하는 만큼의 레이블을 가질 수 있다.

<img width="697" alt="스크린샷 2020-08-04 오전 1 41 19" src="https://user-images.githubusercontent.com/6982740/89206037-a0d05b00-d5f3-11ea-8a22-0b50caabf600.png">


### 3.3.2 파드를 생성할 때 레이블 지정
 - kubia-manual-with-labels.yaml

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels: # 레이블 2개 지정
    creation_method: manual
    env: prod
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

``` sh
# Pod 생성
kubectl create -f kubia-manual-with-labels.yaml

# pod label 정보 조회
kubectl get po --show-labels

# 특정 레이블에만 관심 있는 경우 -L 스위치를 지정해 각 레이블을 자체 열에 표시할 수 있음.
kubectl get po -L creation_method,env
```

### 3.3.3 기존 파드 레이블 수정
``` sh
# 기존 pods 레이블 추가
kubectl label po kubia-manual creation_method=manual

# pods 레이블 수정
kubectl label po kubia-manual-v2 env=debug --overwrite

# 레이블 조회
kubectl get po -L creation_method,env
```

## 3.4 레이블 셀렉터를 이용한 파드 부분 집합 나열
 - 레이블이 중요한 이유는 레이블 셀렉터와 함꼐 사용된다는 것이다.
 - 레이블 셀렉터는 리소스 중에서 다음 기준에 따라 리소르를 선택한다.

> 1. 특정한 키를 포함하거나 포함하지 않는 레이블
> 2. 특정한 키와 값을 가진 레이블
> 3. 특정한 키를 갖고 있지만, 다른 값을 가진 레이블


### 3.4.1 레이블 셀렉터를 사용해 파드 나열
``` sh
# creation_method 레이블 manual을 가지고 있는 파드 나열
kubectl get po -l creation_method=manual

# env를 가지고 있는 파드 나열
kubectl get po -l env

# env를 가지고 있지 않은 파드 나열
kubectl get po -l '!env'

# creation_method 레이블을 가진 것중에 값이 manual이 아닌것
kubectl get po -l 'creation_method!=manual'

kubectl get po -l 'env in (prod,devel)'

kubectl get po -l 'env notin (prod,devel)'
```

### 3.4.2 레이블 셀렉터에서 여러 조건 사용
``` sh
# 셀렉터는 쉼표로 구분된 여러 기준을 포함하는 것도 가능하다.
kubectl get po -l creation_method=manual,env=debug
```

 - 레이블 셀렉터를 이용해 여러 파드를 한번에 삭제할수도 있음.


## 3.5 레이블과 셀렉터를 이용해 파드 스케줄링 제한
 - 파드 스케줄링할 노드를 결정할 떄 영향을 미치고 싶은 상황이 있을수 있음.
 - SSD를 가지고 있는 워커 노드에 할당해야 하는 경우
 - 이때 특정노드를 지정하는 방법은 좋은것은 아님(아마도 노드이름이나 별도의 정보를 기준으로 하는것을 말하는듯)
 - 노드를 지정하는 대신 필요한 노드 요구사항을 기술하고 쿠버네티스가 요구사항을 만족하는 노드를 직접 선택하도록 해야 하는데 이때 노드 레이블과 레이블 셀렉터를 통해서 할 수 있다.
    - SSD 노드가 2개가 있을떄 A노드로 지정을 해버리면 혹시 A노드에 문제가 생기면 스케줄링을 재대로 못할수 있다는 의미(만약 저런 지정이 없었다면 B로 스케줄링 되었을것)

### 3.5.1 워커 노드 분류에 레이블 사용
 - 노드를 포함한 모든 쿠버네티스 오브젝트에 레이블을 부착할수 있기 때문에 노드 분류에 레이블을 사용하고 쿠버네티스가 스케줄링할때 활용할 수 있다.

``` sh
# 이런식으로 노드에 label을 붙인다.(로컬에서 docker-desktop밖에 없어서 직접해보지는 ㅇ낳음)
kubectl label node gke-kubia-85f6-node-0rrx gpu=true

# 노드 조회
kubectl get nodes -l gpu=true
```

### 3.5.2 특정 노드에 파드 스케줄링

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true" # gpu=true 레이블을 포함한 노드에 이 파드라 배포하도록 지시
  containers:
  - image: luksa/kubia
    name: kubia

```

### 3.5.3 하나의 특정 노드 스케줄링
 - nodeSelector에 실제 호스트 이름을 지정할 경우 해당 노드가 오프라인 상태인 경우 파드가 스케줄링 되지 않을수 있어서 레이블 셀렉터를 통해 지정한 특정 기준을 만족하는 노드의 논리적인 그룹으로 처리될 수 있도록 해야 한다.
 - 추가로 노드에 파드를 스케줄링할 때 영향을 줄수 있는 방법은 16장에서 또 나옴

## 3.6 파드에 어노테이션 달기
 - 어노테이션은 키-값 쌍으로 레이블과 거의 비슷하지만 식별 정보로 사용되지 않음.
 - 반면 훨씬 더 많은 정보를 보유할 수 있다.
 - 쿠버네티스 새로운 기능 릴리스시 어노테이션을 사용하곤 한다.
 - 어노테이션을 유용하게 사용하는 방법은 파드나 다른 API 오브젝트에 설명을 추가하는 것을 예를 들수 있음.
    - 이렇게 하면 클러스터를 사용하는 모든 사람이 개별 오브젝트에 관한 정보를 쉽게 찾을수 있음.

### 3.6.1 오브젝트의 어노테이션 조회
 - 레이블에 넣을 데이터는 보통 짧은 데이터이고, 어노테이션에는 상대적으로 큰 데이터를 넣을 수 있음

``` sh
# kubectl describe {리소스} {name}
kubectl get po kubia-xxxxx -o yarml  # 1.9부터 이 정보는 yaml안에서 확인할수 없도록 변경됨

```

### 3.6.2 어노테이션 추가 및 수정
``` sh
# 어노테이션 추가
kubectl annotate pod kubia-manual naver.com/someannotation="naver foo bar"

# describe을 통해 pod의 어노테이션 정보 조회
kubectl describe pod kubia-manual
```

## 3.7 네임스페이스를 사용한 리소스 그룹화
 - 각 오브젝트는 여러 레이블을 가질 수 있기 때문에 오브젝트 그룹은 서로 겹쳐질 수 있다.
 - 클러스터에서 작업을 수행할 때 레이블 셀렉터를 명시적으로 지정하지 않으면 항상 모든 오브젝트를 보게 된다.
 - 이런 문제를 해결하기 위해 쿠버네티스는 네임스페이스 단위로 오브젝트를 그룹화한다.
 - docker에서 이야기했던 리눅스 네임스페이스와는 별개임

### 3.7.1 네임스페이스의 필요성
 - 네임스페이스를 사용하면 많은 구성 요소를 가진 복잡한 시스템을 좀 더 작은 개별 그룹으로 분리할 수 있음.
 - 멀티테넌트환경처럼 리소르르 분리하는 데 사용된다.
 - 리소스를 프로덕션 개발, QA환경 혹은 원하는 다른 방법으로 나누어 사용할 수 있다.
- 단 노드의 경우는 전역 리소스이고, 단일 네임스페이스에 얽매이지 않는다.(이러한 이유는 4장에서 다룸)

### 3.7.2 다른 네임스페이스와 파드 살펴보기
 - 명시적으로 지정하지 않았다면  default 네임스페이스를 사용했을 것
``` sh
kubectl get ns

kubectl get po -n kube-system
```
 - 쿠베 시스템에 대한 네임스페이스가 분리되어 있어서 뭔가 유저가 만든 리소스들과 깔끔하게 분리하여 관리할수 있음.
 - 매우 큰 규모에서 쿠버네티스를 사용한다면 네임스페이스를 사용해서 서로 관계없는 리소스를 겹치지 않는 그룹으로 분리할 수 있다.
 - 네임스페이스 기준으로 리소스 이름에 관한 접근범위를 제공하므로 리소스 이름 충돌에 대한 걱정도 할 필요 없음.
 - 이 외에도 특정 사용자에 대한 리솟 ㅡ접근 권한을 관리할수도 있고, 개별 사용자가 사용할 수 있는 컴퓨팅 리소스를 제한하는 데에도 사용된다.

### 3.7.3 네임스페이스 생성

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```


``` sh
# yaml 파일을 이용한 네임스페이스 생성
kubectl create -f custom-namespace.yaml

# 커맨드라인 명령어로 생성
kubectl create namespace custom-namespace
```

### 3.7.4 다른 네임스페이스의 오브젝트 관리
 - 생성한 네임스페이스 안에 리소스를 만들기 위해서는 metadata 섹션에 namespace항목을 넣거나 kubectl create 명령시 네임스페이스를 지정하면 된다.

``` sh
# custom-namespace에 pod 생성
kubectl create -f kubia-manual.yaml -n custom-namespace
```

- 네임스페이스 context 이동

``` sh
# custom-namespace 네임스페이스로 컨텍스트 변경
kubectl config set-context --current --namespace=custom-namespace

# default 네임스페이스로 컨텍스트 변경
kubectl config set-context --current --namespace=default
```

### 3.7.5 네임스페이스가 제공하는 격리이해
 - 실행중인 오브젝트에 대한 격리는 제공하지 않음.
 - 네임스페이스에서 네트워크 격리를 제공하는지는 쿠버네티스와 함께 배포하는 네트워킹 솔루션에 따라 다름.
 - 서로 다른 네임스페이스 안에 있는 파드끼리 서로 IP주소를 알고있다면 HTTP 요청과 같은 트래픽을 다른 파드로 보내는데에는 전혀 제약이 없다.

## 3.8 파드 중지와 제거
 - 파드를 삭제하면 쿠버네티스 파드 안에 있는 모든 컨테이너를 종료하도록 지시
 - 이때 SIGTERM 신호를 프로세스에 보내고, 지정된 시간(기본 30초)동안 기다린다.
 - 시간 내에 종료되지 않으면 SIGKILL 신호를 통해 종료시킨다.
 - 프로세스가 항상 정상적으로 종료되게 하기 위해서는 SIGTERM 신호를 올바르게 처리해야 한다.

### 3.8.1 이름으로 파드 제거
``` sh
# 이름으로 삭제
kubectl delete po kubia-gpu

# 공백을 구분자로 여러개를 한번에 지울수 있음.
kubectl delete po kubia-gpu kubia-gpu2
```

### 3.8.2 레이블 셀렉터를 이용한 파드 삭제
``` sh
# 레이블 셀렉터를 이요한 삭제
kubectl delete po -l creation_method=manual
kubectl delete po -l rel=canary
```

### 3.8.3 네임스페이스를 삭제한 파드 제거
``` sh
kubectl delete ns custom-namespace
```

### 3.8.4 네임스페이슬 유지하면서 네임스페이스 안에 모든 파드 삭제
 - 여기서 파드를 삭제하더라도 레플리케이션컨트롤러로 생긴 파드가 있었다면 삭제하더라도 다시 스케줄링 될것(이 경우 파드를 삭제하기 위해서는 리플리케이션컨트롤러도 같이 삭제해야 한다.
``` sh
kubectl delete po --all
```

### 3.8.5 네임스페이스에서 (거의) 모든 리소스 삭제
 - all 키워드를 쓰면 대부분의 리소스는 삭제할 수 있지만, 7장에 나올 시크릿 같은 특정 리소스는 그대로 보존되어 있다.
 - 이 키워드를 통해  kubernetes 서비스도 삭제가 가능하지만 잠시 후에 다시 생성된다.
``` sh
kubectl delete all --all
```

## 3.9 요약
 - 특정 컨테이너를 파드로 묶어야 하는지 여부를 결정하는 방법
 - 파드는 여러 프로세스를 실행할 수 있으며 컨테이너가 아닌 세계의 물리적 호스트와 비슷하다.
 - yaml 또는 json 디스크립터를 작성해 파드를 작성하고 파드 정의와 상태를 확인할 수 있다.
 - 레이블과 레이블 셀렉터를 사용해 파드를 조직하고 한번에 여러 파드에서 작업을 쉽게 수행할 수 있다.
 - 노드 레이블과 셀렉터를 사용해 특정 기능을 가진 노드에 파드를 스케줄링 할 수 있다.
 - 어노테이션을 사용하면 사람 또는 도구, 라이브러리에서 더 큰 데이터를 파드에 부착할 수 있다.
 - 네임스페이스는 다른 팀들이 동일한 클러스터를 별도 클러스터를 사용하는 것처럼 이용할 수 있게 해준다.
 - kubectl explain 명령으로 쿠버네티스 리소스 도움말을 볼 수 있음.

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
