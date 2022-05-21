---
title: "[kubernetes-in-action] 9. 디플로이먼트 : 선언적 애플리케이션 업데이트"
author: sungsu park
date: 2020-08-30 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 9. 디플로이먼트 : 선언적 애플리케이션 업데이트
> 쿠버네티스 클러스터에서 실행되는 애플리케이션을 업데이트 하는 방법과 쿠버네티스가 어떻게 무중단 업데이트 프로세스로 전환하는 데 도움을 주는지 살펴본다.

## 9.1 파드에서 실행중인 애플리케이션 업데이트
 - 쿠버네티스에서 실행되는 애플리케이션 기본 구성은 아래와 같다. 여기서 파드에서 실행중인 컨테이너 이미지 버전을 업데이트한다고 할때 어떻게 해야할까?

<img width="552" alt="스크린샷 2020-08-26 오후 11 20 56" src="https://user-images.githubusercontent.com/6982740/91315580-cf0c1980-e7f2-11ea-9359-7d62389604ef.png">

### 모든 파드를 업데이트 하는 방법
 - 기존 파드를 모두 삭제한 다음 새 파드를 시작한다.
 - 새로운 파드를 시작하고, 기동하면 기존 파드를 삭제한다.

### 9.1.1 오래된 파드를 삭제하고 새 파드로 교체
 - (v1 -> v2로 업데이트한다고 했을때) v1 파드 세트를 관리하는 레플리카셋이 있는 경우 이미지의 버전 v2를 참조하도록 파드 템플릿을 수정한 다음 이전 파드 인스턴스를 삭제해 쉽게 교체할 수 있을것이다.

<img width="665" alt="스크린샷 2020-08-26 오후 11 24 08" src="https://user-images.githubusercontent.com/6982740/91315955-3fb33600-e7f3-11ea-9ca0-93ae10a85efa.png">

### 9.1.2 새 파드 기동과 이전 파드 삭제
 - 한 번에 여러 버전의 애플리케이션이 실행하는 것을 지원하는 경우(다른 버전의 애플리케이션이 같이 서빙되어도 문제가 없는 경우) 새 파드를 모두 기동한 후 이전 파드를 삭제할 수 있다. 잠시동안 동시에 두 배의 파드가 실행되므로 더 많은 하드웨어 리소스가 필요하다.

#### 한 번에 이전 버전에서 새 버전으로 전환
 - 새 버전을 실행하는 파드를 불러오는 동안 서비스는 파드의 이전 버전에 연결된다. 새 파드가 모두 실행되면 서비스의 레이블 셀렉터를 변경하고 서비스를 새 파드로 전환할 수 있다.(블루-그린 디플로이먼트)

<img width="650" alt="스크린샷 2020-08-26 오후 11 26 30" src="https://user-images.githubusercontent.com/6982740/91316264-9456b100-e7f3-11ea-8a26-e4eea672bb18.png">

> kubectl set selector 명령어를 사용해 서비스의 파드 셀렉터 변경이 가능

#### 롤링 업데이트 수행
 - 이전 파드를 한번에 삭제하는 방법 대신 파드를 단계적으로 교체하는 롤링 업데이트를 수행할 수도 있다.
 - 2개의 레플리카셋을 이용해서 상태를 보아가면서 수행할수도 있겠지만, 쿠버네티스에서는 하나의 명령으로 롱링 업데이트를 수행할 수 있다.

<img width="650" alt="스크린샷 2020-08-26 오후 11 29 02" src="https://user-images.githubusercontent.com/6982740/91316581-eef00d00-e7f3-11ea-9a50-15cfb0d4a358.png">

## 9.2 레플리케이션컨트롤러로 자동 롤링 업데이트 수행

#### 하나의 YAML에 여러개의 쿠버네티스 리소스를 정의하는 방법
 - ```---```(대시 3개)를 구분자로 여러 리소스 정의를 포함할 수 있다.
``` yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia-v1
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
---                                           # YAML 구분자
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  type: LoadBalancer
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

### 9.2.2 kubectl을 이용한 롤링 업데이트
``` sh
# v1 rc and service 생성
kubectl create -f kubia-rc-and-service-v1.yaml

# v2 롤링 업데이트 수행
kubectl rolling-update kubia-v1 kubia-v2 --image=luksa/kubia:v2
```

 - rolling-update 명령어를 수행하면 일단 kubia/v2에 대한 rc가 만들어지면서 롤링 업데이트가 수행된다.

<img width="532" alt="스크린샷 2020-08-30 오후 3 09 49" src="https://user-images.githubusercontent.com/6982740/91652497-dc672380-ead2-11ea-9da5-7afd07635217.png">

#### 롤링 업데이트 과정

<img width="804" alt="스크린샷 2020-08-30 오후 3 26 35" src="https://user-images.githubusercontent.com/6982740/91652760-34068e80-ead5-11ea-9ff9-f85977f9cc58.png">


#### 동일한 이미지 태그로 업데이트 푸시하기
 - 워커 노드에서 일단 이미지를 한번 가져오면 이미지는 노드에 저장되고, 동일한 이미지를 사용해 새 파드를 실행할 때 이미지를 리모트에서 다시 가져오지 않는다.(도커 이미지 기본 정책)
 - 즉 이미지의 변경한 내용을 같은 이미지 태그로 푸시하더라도 이미지가 변경되지 않는다. 이런 일을 해결할 수 있는 방법으로 컨테이너의 imagePullPolicy속성을 Always로 설정하면 가져올 수 있다.
 - 이미지의 latest 태그를 참조하는 경우에는 imagePullPolicy의 기반값은 always로 항상 리모트에서 가져오지만, 다른 태그인 경우에는 기본 정책인 ifNotPresnet이다.
 - 가장 좋은 방법은 이미지를 변경할 때마다 새로운 태그로 지정하는 방식이다.


#### 롤링 업데이트가 시작되기 전 kubeclt이 수행한 단계 이해하기
 - 롤링 업데이트 프로세스는 첫 번째 rc의 셀렉터도 수정한다.
 - 레이블에 deployment라는 키가 추가되고 value로 hashValue가 추가된다.
 - v2에는 다른 value를 가진 deployment 가 추가된다.

``` sh
# rc kubia-v1 상태 확인
kubectl describe rc kubia-v1

# pod label 확인
kubectl get po --show-labels
```

<img width="573" alt="스크린샷 2020-08-30 오후 4 18 57" src="https://user-images.githubusercontent.com/6982740/91653512-9dd66680-eadc-11ea-9a62-4f693afb10fa.png">

#### 레플리케이션컨트롤러 두 개를 스케일업해 새 파드로 교체
 - service selector는 app=kubia로만 참조되므로, rc에서 하나씩 scale up / scale down이 일어나면서 파드가 교체되고,  롤링 업데이트가 이루어진다.
 - 롤링 업데이트를 계속하면 v2 파드에 대한 요청 비율이 점점 더 높아지기 시작한다.
 - 마지막 v1 파드가 삭제되고, 서비스가 이제 v2 파드에 의해서만 지원하게 되는데, 이때 kubectl은 v1 rc를 삭제하고 업데이트 프로세스가 완료된다.

``` sh
Scaling kubia-v2 up to 1
Scaling kubia-v1 down to 2
```

<img width="517" alt="스크린샷 2020-08-30 오후 4 21 25" src="https://user-images.githubusercontent.com/6982740/91653550-de35e480-eadc-11ea-9e83-cd71e274d90d.png">

### 9.2.3 kubectl rolling-update를 더 이상 사용하지 않는 이유
#### 1) 스스로 만든 오브젝트를 쿠버네티스가 수정하기 떄문에
 - 클러스터 개발자가 등록한 매니페스트를 무시하고 쿠버네티스가 변경한다.(deployment label 같은 경우)

#### 2) 롤링 업데이트를 수행하는 레벨이 클라이언트에서 이루어지기 때문
 - ```--v``` 옵션을 사용해 자세한 로딩을 켜면 이를 확인할 수 있다.
 - kubectl 클라이언트가 쿠버네티스 마스터 대신 스케일링을 수행하는 중임을 볼수 있다.
 - 서버가 아닌 클라이언트가 업데이트 프로세스를 수행하면 왜 문제일까?
     - 업데이트를 수행하는 동안 네트워크 연결이 끊어지는 경우, 중간 상태로 프로세스가 종료될수 있기 떄문에 리스크가 있다.

#### 3) rolling-update 자체가 명령(imperative)을 나타내기 떄문
 - 쿠버네티스에 파드를 추가하거나 초과된 파드를 제거하라고 지시하지 마라(대신 레플리카 수를 변경하여 쿠버네티스 서버가 알아서 하도록 맡겨야 한다.)

## 9.3 애플리케이션을 선언적으로 업데이트 하기 위한 디플로이먼트 사용하기
 - 낮은 수준의 개념으로 간주되는 RC, RS을 통해 수행하는 대신 애플리케이션을 배포하고 선언적으로 업데이트 하기 위한 높은 수준의 리소스
 - 디플로이먼트를 생성하면 레플리카셋 리소스가 그 아래에 생성된다.
 - 애플리케이션 업데이트할 때는 추가 레플리케이션컨트롤러를 도입하고, 두 컨트롤러가 잘 조화하도록 조정해야 하는데, 이를 전체적으로 통제하는 것이 디플로이먼트이다.

<img width="582" alt="스크린샷 2020-08-30 오후 4 36 45" src="https://user-images.githubusercontent.com/6982740/91653839-032b5700-eadf-11ea-85af-89fe5c497711.png">

### 9.3.1 디플로이먼트 생성
 - 디플로이먼트는 레이블 셀렉터, 원하는 레플리카수, 파드 템플릿으로 구성된다.
 - 리소스가 수정될 때 업데이트 수행 방법을 정의하는 디폴로이먼트 전략을 지정할수 도 있음.

#### 디플로이먼트 매니페스트 생성
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
  selector:
    matchLabels:
      app: kubia
```

``` sh
# rc 제거
kubectl delete rc --all

# deployment 생성
# --recode 명령줄을 포함시켜 개정 이력(revision history)에 기록하여야 한다.
kubectl create -f kubia-deployment-v1.yaml --record

# deployment 조회
kubectl get deploy

# deployment에 의해 생성된 rc 조회
kubectl get rs

# deployment, rs에 의해 생성된 pod 조회
ubectl get po | grep kubia-59d857
```

#### 디플로이먼트 롤아웃 상태 출력
``` sh
# 롤아웃 상태 출력
kubectl rollout status deploy kubia

# pod 확인
kubectl get po
```

#### 디플로이먼트가 레플리카셋을 생성하는 방법과 레플리카셋이 파드를 생성하는 방식 이해
 - 컨트롤러 이름과 임의로 생성된 문자열로 구성된다. 디플로이먼트에서 생성한 파드 3개에는 이름 중간에 숫자 값이 추가로 포함된다.
 - 이 숫자 값은 파드 템플릿의 해시값을 나타낸다.

<img width="732" alt="스크린샷 2020-08-30 오후 4 47 45" src="https://user-images.githubusercontent.com/6982740/91654016-987b1b00-eae0-11ea-8697-08f6bd38f276.png">


``` sh
# 레플리카셋 조회
kubectl get rs
```

<img width="725" alt="스크린샷 2020-08-30 오후 4 50 03" src="https://user-images.githubusercontent.com/6982740/91654087-2525d900-eae1-11ea-81a1-8fea6c029cce.png">

 - 디플로이먼트에서 생성된 레플리카셋의 이름을 보면 여기에도 파드 템플릿의 해시값이 포함되어 있다.
 - 디플로이먼트는 파드 템플릿의 각 버전마다 하나씩 여러 개의 레플리카셋을 만든다.
 - 이 파드 템플릿의 해시 값을 사용하면 디플로이먼트에서 지정된 버전의 파드 템플릿에 관해 항상 동일한 레플리카셋을 사용할 수 있다.

### 9.3.2 디플로이먼트 업데이트
 - 디플로이먼트 리소스에 정의된 파드 템플릿을 수정하기만 하면 쿠버네티스가 실제 시스템 상태를 리소스에 정의된 상태로 만드는 데 필요한 모든 단계를 수행한다.

#### 사용 가능한 디플로이먼트 전략
 - 기본 전략은 RollingUpdate 전략이다.  RollingUpdate 전략은 이전 버전과 새 버전을 동시에 실행할 수 있는 경우에만 사용해야 한다.
 - 대안으로 존재하는 Recreate 전략은 한 번에 기존 모든 파드를 삭제한 뒤 새로운 파드를 만드는 전략이다. 이는 앱이 여러 버전을 병렬로 실행하는 것을 지원하지 않고 새 버전을 시작하기 전에 이전 버전을 완전히 중지해야 하는 경우 사용할 수 있는데, 짧게 서비스 다운타임이 발생하는 문제가 있다.

#### 롤링 업데이트 속도 느리게 하기
 - 디플로이먼트의 minReadySeconds 속성을 설정하여 롤링 업데이트 속도를 느리게 만들 수 있다.

``` sh
# deployment spec 수정
kubectl patch deployment kubia -p '{"spec":{"minReadySeconds":10}}'

# deployment 상태 확인
kubectl get deploy -o yaml
```

#### 롤링 업데이트 시작
 - kubectl set image 명령어를 사용해 컨테이너가 포함된 모든 리소스(rc, rs, deployment 등등)을 수정할 수 있다.

``` sh
# image 변경
kubectl set image deployment kubia nodejs=luksa/kubia:v2 --record

# deployment 상태 확인
kubectl get deploy -o yaml
```

 - depolyment의 파드 템플릿이 업데이트돼 nodejs 컨테이너에 사용된 이미지가 kubia:v2로 변경된다.

<img width="613" alt="스크린샷 2020-08-30 오후 5 03 09" src="https://user-images.githubusercontent.com/6982740/91654317-b184cb80-eae2-11ea-880e-654d70568717.png">


#### 디플로이먼트와 그 외의 리소스를 수정하는 방법

| 명령              | 설명                                                   | example                                                             |
|-------------------|--------------------------------------------------------|---------------------------------------------------------------------|
| kubectl edit      | 기본 편집기로 수정                                     | kubectl edit deploy kubia                                           |
| kubectl patch     | 오브젝트의 개별 속성 수정                              | kubectl patch deployment kubia -p '{"spec":{"minReadySeconds":10}}' |
| kubectl apply     | 전체 yaml/json 파일의 속성 값을 적용해 오브젝트를 수정 | kubectl apply -f kubia-deployment-v2.yaml                           |
| kubectl replace   | yaml / json 파일로 오브젝트를 새것으로 교체            | kubectl replace -f kubia-deployment-v2.yaml                         |
| kubectl set image | 정의된 컨테이너 이미지 변경                            | kubectl set image deployment kubia nodejs=luksa/kubia:v2            |


#### 디플로이먼트의 놀라움
 - 파드 템플릿을 변경하는 것만으로 애플리케이션을 최신 버전으로 업데이트할 수 있음.
 - 디플로이먼트의 파드 템플릿이 컨피그맵(또는 시크릿)을 참조하는 경우 컨피그맵은 수정하더라도 업데이트를 시작하지 않는다.(단, 새 컨피그맵을 만들고 파드 템플릿이 새 컨피그맵을 참조하도록 수정하면 업데이트가 수행된다.)
 - 여기서 중요한 사실 중 하나는 기존 rs도 여전히 남아있는다는 것이다. 이는 롤백이나 이런 부분에서 재사용될 수 있음.
 - 단일 디플로이먼트 오브젝트를 관리하는 것이 여러 레플리케이션 컽느롤러를 처리하고 추적하는것보다 훨씬 쉬움

<img width="668" alt="스크린샷 2020-08-30 오후 5 11 09" src="https://user-images.githubusercontent.com/6982740/91654458-cf066500-eae3-11ea-9717-ebccc399c8b1.png">

``` sh
# rs 조회 ( 2개가 그대로 남아있는것을 알 수 있다.)
kubectl get rs
```

### 9.3.3 디플로이먼트 롤백
``` sh
# 이미지 변경
kubectl set image deploy kubia nodejs=luksa/kubia:v3 --record

# rollout 상태 확인
kubectl rollout status deploy kubia
```

#### 롤아웃 되돌리기
 - 업데이트 된 v3가 에러를 발생하기 시작할떄 롤백을 수행할 수 있다.
 - 롤아웃 프로세스가 진행중인 동안에도 롤아웃을 중단하려면 실행 취소 명령을 사용해서 중단시킬 수 있다. (롤아웃 중에 생성된 파드는 제거되고 이전 파드로 다시 교체된다.)

``` sh
# 이전 버전으로 롤백
kubectl rollout undo deploy kubia
```

#### 디플로이먼트 롤아웃 이력 표시
 - 롤아웃 이력에 포함시키려면 --record 명령줄을 포함시켜야만 한다.

``` sh
# 롤아웃 이력 표시
kubectl rollout history deploy kubia
```

#### 특정 디플로이먼트 개정으로 롤백
 - 개정(revision) 번호를 지정해 특정 개정으로 롤백할 수 있다.
 - 디플로이먼트를 처음 수정했을 때 비활성화된 레플리카셋이 남아있던 이유는 이 롤백을 위함이다.
 - 모든 개정 내역의 수는 디폴로이먼트 리소스의 editionHistoryLimit 속성에 의해 제한된다.(쿠버네티스 버전이 올라가면서 revisionHistoryLimit로 변경된듯 하다)
 - 현재 쿠버네티스 버전 기준으로 revisionHistoryLimit의 기본값은 10이다.
 - https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/

``` sh
# 1번 revision으로 롤백
kubectl rollout undo deployment kubia --to-revision=1
```

<img width="559" alt="스크린샷 2020-08-30 오후 5 30 35" src="https://user-images.githubusercontent.com/6982740/91654746-869c7680-eae6-11ea-898d-4c8fc6a7e415.png">


### 9.3.4 롤아웃 속도 제어
 - 롤링 업데이트 전략의 두 가지 추가 속성을 통해 새 파드를 만들고 기존 파드를 삭제하는 과정에서 속도를 제어할 수 있다.

#### 롤링 업데이트 전략의 maxSurge와 maxUnavailable 속성 소개
 - 이 2개의 속성에 의해서  한 번에 몇개의 파드를 교체할지를 결정된다.
 - maxSurge : 레플리카 수보다 얼마나 많은 파드 인스턴스 수를 허용할지를 나타낸다.(기본값 : 25%), 만약 레플리카수가 4로 설정한 경우 4의 25%인 1개만큼 더 허용될수 있다. 즉 5개의 파드까지 생성될 수 있음을 의미한다. 백분위로 설정하기 떄문에 이 값은 반올림 처리 된다.
 - maxUnavailable : 업데이트 중에 의도하는 레플리카 수를 기준으로 사용할 수 없는 파드 인스턴스 수(기본값 25%), 레플리카수가 4인 경우 25%인 1개만큼 unavailable되는것을 허용한다. 업데이트 과정에서 사용할 수 없는 파드는 최대 1개여야 한다. 즉 최소 3개의 파드는 항상 가용한 상태를 유지하면서 롤링 업데이트가 수행되어야 한다는 의미이다. 이  백분위 값을 처리할떄는 내림으로 처리된다.

``` yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
```

 - 위 에제에서 레플리카수가 3이고, maxSurge가 1이고, maxUnavailable을 0으로 설정한 경우에는 다음과 같이 동작한다.
 - 모든 파드수는 4개가 되는 것까지 허용했으며, maxUnavailable은 0이기 떄문에 available한 파드는 레플리카수와 동일한 3개여야 한다.(항상 3개의 파드를 사용할 수 있어야 한다.)

<img width="665" alt="스크린샷 2020-08-30 오후 5 41 50" src="https://user-images.githubusercontent.com/6982740/91654952-1a227700-eae8-11ea-8228-2a7485382a34.png">


##### maxUnavailable 속성 이해
 - 위 예제에서 maxUnavailable만 1로 변해보면 어떻게 동작할까?
 - 레플리카수는 3개인데 maxUnavailable이 1이기 떄문에, 롤링 업데이트 과정에서 최소 2개의 파드가 사용가능한 상황을 허용한다.
 - 최대 동시에 실행될 수 있는 파드는 기존과 동일하게 4개이고, 2개가 가용하지 않아도 된다.
 - 위 예제와 비교했을떄 maxUnavailable=1로 인해서 2개까지만 가용한 상태면 되기 때문에 결과적으로는 롤링 업데이트가 더 빠르게 수행될수 있다.


<img width="590" alt="스크린샷 2020-08-30 오후 5 43 02" src="https://user-images.githubusercontent.com/6982740/91654972-43430780-eae8-11ea-9168-0a0a5a3ed443.png">


### 9.3.5 롤아웃 프로세스 일시 중지
 - 롤아웃 프로세스를 일시정지해서 일종의 카나리 릴리스를 실행할 수가 있다.(카나리 릴리스를 하는데 아주 좋은 방법은 아닌듯 하다.)
    - 정확히 내가 원하는 레플리카수를 보장하기 힘들다. 타이밍 이슈가 발생하기 떄문에 좋은 방법이 아니다.(책의 저자가 아닌 글 작성자의 개인적인 생각)
 - n개의 파드가 있을 때 v4의 파드는 하나만 구동시키고 나머지는 기존 버전으로 구동시킨다.(이러한 방식으로 사용자들에게 영향을 최소화하면서 변경된 로직이 문제없는지 체크하는 방식)

``` sh
# v4로 버전 변경
kubectl set image deploy kubia nodejs=luksa/kubia:v4 --record

# 롤아웃 일시 정지
kubectl rollout pause deploy kubia

# pod 확인
kubectl get po
```

<img width="725" alt="스크린샷 2020-08-30 오후 5 48 52" src="https://user-images.githubusercontent.com/6982740/91655081-13e0ca80-eae9-11ea-8531-ad7cd3f54379.png">

 - 위 파드 목록을 보면 v4로 구동된 파드 ( kubia-586b45dbdc-5cgc6) 1개가 추가로 구동중인것을 볼수 있다.

#### 롤아웃 재개
 - 새 버전이 제대로 작동한다고 확신하면 디플로이먼트를 다시 시작해 이전 파드를 모두 새 파드로 교체할 수 있다.
 - 책이 쓰여진 시점 기준,  카나리 릴리스를 수행하는 적절한 방법은 두 가지 다른 디플로이먼트를 사용해 적절하게 확장하는 것이다.

``` sh
# 롤아웃 재개
kubectl rollout resume deploy kubia
```

#### 롤아웃을 방지하기 위한 일시 중지 기능 사용
 - 일시 중지 기능을 사용하면 롤아웃 프로세스가 시작돼 디플로이먼트를 업데이트하는 것을 막을 수 있고, 여러 번 변경하면서 필요한 모든 변경을 완료한 후에 롤아웃을 시작하도록 할 수 있다.

### 9.3.6 잘못된 버전의 롤아웃 방지
 - minReadySeconds 속성으로 롤아웃 속도를 늦춰 롤링 업데이트 과정을 직접 볼수 있는데, 이 기능은 오작동 버전의 배포를 방지하는 목적으로도 사용할 수 있다.

#### minReadySeconds의 적용 가능성 이해
 - minReadySeconds는 파드를 사용 가능한  것으로 취급하기 전에 새로 만든 파드를 준비할 시간을 지정하는 속성이다.
 - 이것과 레디니스 프로브를 함께 이용하여 오작동 버전의 롤아웃을 효과적으로 차단할 수 있다.
 - 모든 파드의 레디니스 프로브가 성공하면 파드가 준비상태가 되는데, minReadySeconds가 지나기전에 레디니스 프로브가 실패하기 시작하면 새 버전의 롤아웃은 차단이 된다.
 - 적절하게 구성된 레디니스 프로브와 적절한 minReadySeconds 설정으로 쿠버네티스는 버그가 있는 버전을 배포하지 못하게 할 수 있다.

#### 적절한 minReadySeconds와 레디니스 프로브 정의
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  minReadySeconds: 10           # minReadySeconds를 10초로 설정
  selector:
    matchLabels:
      app: kubia
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0             # 디플로이먼트가 파드를 하나씩만 교체하도록 0으로 설정
    type: RollingUpdate
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
      - image: luksa/kubia:v3
        name: nodejs
        readinessProbe:               # 레디니스 프로브 정의
          periodSeconds: 1           # 매 초마다 레디니스 프로브 수행
          httpGet:
            path: /
            port: 8080
```

### kubectl apply 를 통한 deploy 업데이트
 - apply를 통해 업데이트할 때 원하는 레플리카 수를 변경하지 않으려면 replicas 필드를 포함시키면 안된다.

``` sh
# kubectl apply 를 통한 deploy 업데이트
kubectl apply -f kubia-deployment-v3-with-readinesscheck.yaml

# 롤아웃 상태 확인
kubectl rollout status deploy kubia
```

#### 레디니스 프로브가 잘못된 버전으로 롤아웃되는 것을 방지하는 법
 - 잘못된 버전은 레디니스 프로브 단계에서 차단되어 파드가 생성되지 않는다.
 - rollout status 명령어느 하나의 새 레플리카만 시작됐음을 보여준다.
 - 사용 가능한 것으로 간주되려면 10초 이상 준비돼 있어야 하기 때문에 해당 파드가 사용 가능할 때까지 롤아웃 프로세스는 새 파드를 만들지 않는다. 여기서 maxUnavailable 속성이 0으로 설정되었기 때문에 원래 파드도 제거되지 않는다.
 - 만약 위 상황에서 minReadySeconds를 짧게 설정했더라면 레디니스 프로브의 첫 번쨰 호출이 성공한 후 즉시 새 파드가 사용 가능한것으로 간주해버릴 수도 있다. 그러면 잘못된 버전으로 롤아웃이 일어나기 때문에 이 값을 적절하게 잘 설정해야 한다.

<img width="554" alt="스크린샷 2020-08-30 오후 6 13 02" src="https://user-images.githubusercontent.com/6982740/91655561-74253b80-eaec-11ea-92bf-58e6407cbc12.png">

#### 롤아웃 데드라인 설정
 - 기본적으로 쿠버네티스에서는 롤아웃이 10분동안 진행되지 않으면 실패한 것으로 간주된다.
 - 이 값은 progressDeadlineSeconds 속성을 통해 설정할 수 있다.
 - progressDeadlineSeconds에 지정된 시간이 초과되면 롤아웃이 자동으로 중단된다.

``` sh
# deploy 정보 확인
kubectl describe deploy kubia
```

#### 잘못된 롤아웃 중지
``` sh
# 롤아웃 중지
kubectl rollout undo deployment kubia
```


## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/)
