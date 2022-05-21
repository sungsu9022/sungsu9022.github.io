---
title: "[kubernetes-in-action] 10. 스테이트풀셋: 복제된 스테이트풀 애플리케이션 배포하기"
author: sungsu park
date: 2020-09-02 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 10. 스테이트풀셋: 복제된 스테이트풀 애플리케이션 배포하기
> 볼륨이나 퍼시스턴트볼륨클레임에 바인딩된 퍼시스턴트볼륨을 통해서 데이터베이스를 일반 파드에 실행했었는데, 이 파드들을 스케일아웃하려면 어떻게 할수 있을지 살펴본다.

## 10.1 스테이트풀 파드 복제하기
 - 스테이트풀 파드를 복제할떄 레플리카셋을 이용한다면 어떨까?
 - 레플리카셋은 하나의 파드 템플릿에서 여러 개의 파드 레플리카를 생성한다.
 - 여러 개개의 파드 레플리카를 복제하는 데 사용하는 파드 템플릿에는 클레임에 관한 참조가 있으므로 각 레플리카가 별도의 퍼시스턴트볼륨클레임을 사용하도록 만들 수가 없음.

<img width="414" alt="스크린샷 2020-09-01 오후 10 56 49" src="https://user-images.githubusercontent.com/6982740/91860275-70d5af80-eca6-11ea-9aaf-40f9737ac3be.png">

### 10.1.1 개별 스토리지를 갖는 레플리카 여러 개 실행하기
 - 이에 대한 방법은 여러가지가 있겠지만 현실성이 대부분 부족하다

#### 수동으로 파드 생성하기
 - 레플리카셋이 파드를 감시하지 못하므로 가능한 옵션이 아니다.

#### 파드 인스턴스별로 하나의 레플리카셋 사용하기
 - 이 경우 의도된 레플리카 수를 변경할 수 없고, 추가 레플리카셋을 생성해야 한다.(이러면 레플리카셋을 쓰는 의미가 있는가..? 없다.)

<img width="397" alt="스크린샷 2020-09-01 오후 10 59 00" src="https://user-images.githubusercontent.com/6982740/91860514-bb572c00-eca6-11ea-9553-93e618d80bdb.png">

#### 동일 볼륨을 여러 개 디렉터리로 사용하기
 - 인스턴스간 조정이 필요하고 올바르게 수행하기 쉽지 않다. 심지어 공유 스토리지 볼륨에서 병목이 발생할수도 있다.

<img width="442" alt="스크린샷 2020-09-01 오후 11 00 04" src="https://user-images.githubusercontent.com/6982740/91860644-e17ccc00-eca6-11ea-9950-104c8b541c71.png">

### 10.1.2 각 파드에 안정적인 아이덴티티 제공하기
 - 특정 애플리케이션이 안정적인 네트워크 아이덴티티를 요구하는 이유는 무엇일까? 분산 스테이트풀 애플리케이션에서는 이러한 요구사항이 상당히 일반적이다.
     - 이를테면 ACL을 구성하기도 하고..
 - 쿠버네티스에서는 매번 파드가 재스케줄링될 수 있고, 새로운 파드는 새로운 호스트 이름과 IP주소를 할당받으므로 구성 멤버가 재스케줄링될 때마다 모든 애플리케이션 클러스터가 재구성돼야 한다.

#### 각 파드 인스턴스별 전용 서비스 사용하기
 - 개별 파드는 자신이 어떤 서비스를 통해 노출되는지 알수 없으므로 안정적인 IP를  알수 있는것도 아니고, 모든 문제를 해결할 수 조차 없다.

<img width="569" alt="스크린샷 2020-09-01 오후 11 02 13" src="https://user-images.githubusercontent.com/6982740/91860907-2e60a280-eca7-11ea-8cf4-21ede74df5f2.png">


## 10.2 스테이트풀셋
 - 10.1에서 나열한 문제들을 해결할 수 있도록 쿠버네티스에서는 스테이트풀셋(StatefulSet)을 제공한다.
 - 스테이트풀셋은 애플리케이션의 인스턴스가 각각 안정적인 이름과 상태를 가지며 개별적으로 취급돼야 하는 애플리케이션에 알맞게 만들어졌다.

### 10.2.1 스테이트풀셋과 레플리카셋 비교
#### 애완동물 vs 가축
##### 스테이트풀셋 - 애완동물
 - 각 인스턴스에 이름을 부여하고 개별적으로 관리한다
 - 스테이트풀 파드가 종료되면 새로운 파드 인스턴스는 교체되는 파드와 동일한 이름, 네트워크 아이덴티티, 상태 그대로 다른 노드에서 되살아나야 한다.

##### 레플리카셋 - 가축
 - 스테이트리스 애플리케이션의 인스턴스는 이름을 정확히 알고 있을 필요가 없고 몇마리가 있는지만 중요하다.
 - 언제든 완전히 새로운 파드로 교체되어도 된다.

### 10.2.2 안정적인 네트워크 아이덴티티 제공하기
 - 스테이트풀셋으로 생성된 파드는 서수 인덱스(0부터 시작)가 할당되고 파드의 이름과 호스트 이름, 안정적인 스토리지를 붙이는 데 사용된다.

<img width="490" alt="스크린샷 2020-09-01 오후 11 07 09" src="https://user-images.githubusercontent.com/6982740/91861514-dfffd380-eca7-11ea-9b1d-0a83b4ec6162.png">

#### 거버닝 서비스
 - 스테이트풀 파드는 때떄로 호스트 이름을 통해 다뤄져야 할 필요가 있다.(스테이트리스는 이런 니즈가 없음)
 - 스테이트풀 파드는 각각 서로 다르므로(다른 상태를 가지거나) 요구사항에 따라 특정 스테이트풀 파드에서 동작하기를 원할수도 있다.
 - 위와 같은 이유로 스테이트풀셋은 거버닝 헤드리스 서비스(governing headless service)를 생성해서 각 파드에게 실제 네트워크 아이덴티티를 제공해야 한다.(헤드리스 서비스를 통해 고정된 하나의 IP가 아닌 서비스에 맵핑된 모든 파드의 IP 목록을 얻는다.)

#### 스테이트풀셋 교체하기
 - 스테이트풀셋은 레플리카셋이 하는 것과 비슷하게 새로운 인스턴스로 교체되도록 한다. 하지만 교체되더라도 이전에 사라진 파드와 동일한 호스트 이름을 갖는다. 그렇기 떄문에 파드가 다른 노드로 재스케줄링되더라도 같은 클러스터 내에서 이전과 동일한 호스트 이름으로 접근이 가능하다.

<img width="596" alt="스크린샷 2020-09-01 오후 11 13 48" src="https://user-images.githubusercontent.com/6982740/91862353-cd39ce80-eca8-11ea-8f58-1b5c598939a7.png">

#### 스테이트풀셋 스케일링
 - 스테이트풀셋의 스케일 다운의 좋은 점은 항상 어떤 파드가 제거될지 알수 있다는 점이다.
 - 스테이트풀셋의 스케일 다운은 항상 가장 높은 서수 인덱스의 파드를 먼저 제거한다.
 - 스테이트풀셋은 인스턴스 하나라도 비정상인 경우 스케일 다운 작업을 허용하지 않는다. 그 이유는 여러개 노드가 동시에 다운되는 경우 데이터를 잃을수도 있기 떄문이다.)

<img width="570" alt="스크린샷 2020-09-01 오후 11 15 05" src="https://user-images.githubusercontent.com/6982740/91862484-fa867c80-eca8-11ea-9b1c-28cd9f4053e0.png">

### 10.2.3 각 스테이트풀 인스턴스에 안정적인 전용 스토리지 제공하기
 - 스테이트풀 파드의 스토리지는 영구적이어야 하고 파드와는 분리되어야 한다.
 - 스테이트풀셋의 각 파드는 별도의 퍼시스턴트볼륨을 갖는 다른 퍼시스턴트볼륨클레임을 참조해야 한다.(이를 스테이트풀셋에서 제공함.)

#### 볼륨 클레임 템플릿과 파드 템플릿을 같이 구성
 - 스테이트풀셋이 파드를 생성하는 것과 같은 방식으로 퍼시스턴트볼륨클레임 또한 생성할 수 있다.

<img width="512" alt="스크린샷 2020-09-01 오후 11 23 14" src="https://user-images.githubusercontent.com/6982740/91863471-1e968d80-ecaa-11ea-90d1-3bcb3da0320b.png">

#### 퍼시스턴트볼륨클레임의 생성과 삭제의 이해
 - 생성할떄는 파드와 퍼시스턴트볼륨클레임 등 2개 이상의 오브젝트가 생성이 되는데..
 - 스케일 다운을 할 떄는 파드만 삭제하고 클레임은 남겨둔다.(클레임이 삭제된 후 바인딩됐던 퍼시스턴트볼륨은 재활용되거나 삭제돼 콘텐츠가 손실될 수 있기 때문)
 - 그래서 기반 퍼시스턴트볼륨을 해제하려면 퍼시스턴트볼륨클레임을 수동을 삭제해주어야 한다.

#### 동일 파드의 새 인스턴스에 퍼시스턴트볼륨클레임 다시 붙이기
 - 스테이트풀셋은 스케일 다운되었을 때 기존에 있던 퍼시스턴트볼륨클레임을 유지했다가 스케일 업될떄 다시 해당 볼륨클레임을 연결한다.

<img width="577" alt="스크린샷 2020-09-01 오후 11 26 04" src="https://user-images.githubusercontent.com/6982740/91863856-8351e800-ecaa-11ea-9d0e-d38d237657d4.png">

### 10.2.4 스테이트풀셋 보장(guarantee)
#### 안정된 아이덴티티와 스토리지의 의미
 - 쿠버네티스상에서 보장해주지 않는다고 가정할때, 동일한 아이덴티티를 가지는 교체 파드를 생성되면 애플리케이션의 두 개 인스턴스가 동일한 아이덴티티로 시스템에서 실행하게 되면 큰 문제가 될수도 있다.

#### 스테이트풀셋 최대 하나의 의미
 - 쿠버네티스는 두 개의 스테이트풀 파드 인스턴스가 절대 동일한 아이덴티티로 실행되지 않고, 동일한 퍼시스턴트볼륨클레임에 바인딩되지 않는것을 보장한다.
 - 스테이트풀셋은 교체 파드를 생성하기 전에 파드가 더 이상 실행중이지 않는다는 점을 절대적으로 확신해야 처리가 이루어진다.

## 10.3 스테이트풀셋 사용하기
### 10.3.1 스테이트풀셋을 위해 준비사항
 - 데이터 파일을 저장하기 위한 퍼시스턴트볼륨(동적 프로비저닝을 해도됨)
 - 스테이트풀셋에 필요한 거버닝 헤드리스 서비스
 - 스테이트풀셋

### 10.3.2 스테이트풀셋 배포
#### 퍼시스턴트볼륨 생성
``` yaml
kind: List                        # 여러개의 리소스를 정의할때 List를  사용할 수 있다.
apiVersion: v1
items:
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-a
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /tmp/pv-a
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-b
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /tmp/pv-b
- apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-c
  spec:
    capacity:
      storage: 1Mi
    accessModes:
      - ReadWriteOnce
    persistentVolumeReclaimPolicy: Recycle
    hostPath:
      path: /tmp/pv-c
```

``` sh
# 볼륨 생성
kubectl create -f persistent-volumes-hostpath.yaml
```

#### 거버닝 서비스 생성하기
``` yaml
 apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  clusterIP: None          # 헤드리스 서비스를 위함
  selector:
    app: kubia
  ports:
  - name: http
    port: 80
```

``` sh
#  거버닝 헤드리스 서비스 생성
kubectl create -f kubia-service-headless.yaml
```

#### 스테이트풀셋 생성하기
 - 첫번째 파드가 생성되고 준비가 완료되어야만 두 번째 파드가 생성된다.
 - 두 개 이상의 멤버가 동시에 생성되면 레이스 컨디션에 빠질 가능성이 있기 떄문에 스테이트풀셋은 순차적으로 하나씩만 처리된다.

``` yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kubia
spec:
  serviceName: kubia
  replicas: 2
  selector:
    matchLabels:
      app: kubia # has to match .spec.template.metadata.labels
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia-pet
        ports:
        - name: http
          containerPort: 8080
        volumeMounts:                 # 볼륨 마운트
        - name: data
          mountPath: /var/data
  volumeClaimTemplates:          # 볼륨클레임 템플릿
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 1Mi
      accessModes:
      - ReadWriteOnce
```

``` sh
# 스테이트풀셋 생성
kubectl create -f kubia-statefulset.yaml

# statefulset 조회
kubectl get statefulset

# pod 조회
kubectl get po | grep kubia-

# 생성된 statefulset pod 조회
kubectl get po kubia-0 -o yaml
```

### 10.3.3 생성된 파드 확인
 - 파드에 피기백(exec를 통해 파드 내부로 진입해서 처리하는 방식)하거나 API 서버를 통해 데이터를 확인해본다.

``` sh
# api server local 프록시
kubectl proxy

# 파드 엔드포인트 조회
curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
curl localhost:8001/api/v1/namespaces/default/pods/kubia-1/proxy/
```

<img width="600" alt="스크린샷 2020-09-01 오후 11 45 43" src="https://user-images.githubusercontent.com/6982740/91866247-42a79e00-ecad-11ea-9406-8e8a5199047f.png">

#### 스테이트풀셋 파드를 삭제해 재스케줄링된 파드가 동일한 스토리지에 연결되는지 확인
 - 새 파드는 클러스터의 어느 노드에서나 스케줄링될 수 있으며 이전 파드가 스케줄링 됐던 동일한 노드일 필요가 없다.
 - 이전 파드의 모든 아이덴티티(이름, 호스트 이름, 스토리지)는 새 노드로 효과적으로 이동된다.

``` sh
# 스테이트풀셋 파드 데이터 수정
curl -X POST -d "Hey three! This greeting was submitted to kubia-0." localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/

# 변경한 파드 삭제(삭제하면 직후 스테이트풀셋이 새로운 파드를 생성한다.)
kubectl delete po kubia-0

# 재확인(수정한 데이터가 그대로 남아있음.)
curl localhost:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
```

#### 스테이트풀셋 스케일링
 - 가장 중요한 점은 스케일 업/다운이 점진적으로 수행되며 스테이트풀셋이 초기에 생성됐을 때 개별 파드가 생성되는 방식과 유사하다는 점이다.
 - 또한 스케일 다운시 가장 높은 서수의 파드가 먼저 삭제되고, 파드가 완전히 종료된 이후부터 다음 스케일다운이 수행된다.

#### 스테이트풀 파드를 헤드리스가 아닌 일반적인 서비스로 노출하기
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-public
spec:
  selector:
    app: kubia
  ports:
  - port: 80
    targetPort: 8080
```

#### API 서버를 통해 클러스터 내부 서비스에 연결
``` sh
# cluster ip service 호출
curl localhost:8001/api/v1/namespaces/default/services/kubia-public/proxy/
``

## 10.4 스테이트풀셋의 피어 디스커버리
 - 클러스터된 애플리케이션의 중요한 요구사항은 피어 디스커버리(클러스터의 다른 멤버를 찾는 기능)이다.
 - API서버와 통신해서 찾을수 있지만, 쿠버네티스의 목표 중 하나는 애플리케이션을 완전히 쿠버네티스에 독립적으로 유지하며 기능을 노출하는 것이다.

### SRV 레코드
 - srvlookup이라 부르는 일회용 파드(--restart=Never)를 실행하고 콘솔에 연결하며(-it) 명령어를 수행하고 종료되며, 바로 삭제된다.(--rm)
 - ANSWER SECTION에는 헤드리스 서비스를 뒷받침하는 두 개의 파드를 가리키는 두 개의 SRV 레코드를 확인할 수 있다.
 - 파드가 스테이트풀셋의 다른 모든 파드의 목록을 가져오려면 SRC DNS 룩업을 수행해서 얻을수 있다는 것이다.

``` sh
# srvlookup을 할수 있는 dnsutils pod을 하나 수행시켜서 명령어를 수행하고 파드 종료
kubectl run -it srvlookup --image=tutum/dnsutils --rm --restart=Never -- dig SRV kubia.default.svc.cluster.local


;; ANSWER SECTION:
kubia.default.svc.cluster.local. 30 IN	SRV	0 50 80 kubia-0.kubia.default.svc.cluster.local.
kubia.default.svc.cluster.local. 30 IN	SRV	0 50 80 kubia-1.kubia.default.svc.cluster.local.

;; ADDITIONAL SECTION:
kubia-1.kubia.default.svc.cluster.local. 30 IN A 172.17.0.8
kubia-0.kubia.default.svc.cluster.local. 30 IN A 172.17.0.3
```

### 10.4.1 DNS를 통한 피어 디스커버리
 - 스테이트풀셋과 SRV 레코드를 사용하여 헤드리스 서비스의 모든 파드를 직접 찾는다.

``` js
import dns from 'dns'

dns.resolveSrv(serviceName, (err, addresses) {
    // TODO 무언가 처리
})
```

<img width="592" alt="스크린샷 2020-09-02 오후 10 05 36" src="https://user-images.githubusercontent.com/6982740/91987026-7267ac00-ed68-11ea-811f-d0dc7228d03f.png">

### 10.4.2 스테이트풀셋 업데이트
 - 편집기로 statefulset 정의 내에서 레플리카 수를 수정한다면 수정 직후 파드가 생성되는걸 확인할 수 있다.
 - 만약 이미지를 변경한다면 디플로이먼트처럼 롱링 업데이트도 필요할텐데 쿠버네티스 1.7부터 이 기능도 지원한다.

``` sh
# 편집기로 statefulset 정의 열어서 수정
kubectl edit statefulset kubia

# pod 확인
kubectl get po
```

### 10.4.3 클러스터된 데이터 저장소 사용하기
 - SRV lookup을 통해 서비스에 포함된 모든 파드를 찾을수 있기 때문에 스테이트풀셋을 스케일 업하거나 스케일 다운하더라도 클라이언트 요청을 서비스하는 파드는 항상 그 시점에 실행중인 모든 피어를 찾을 수 있다.

## 10.5 스테이트풀셋이 노드 실패를 처리하는 과정
 - 스테이트풀셋은 노드가 실패한 경우 동일한 아이덴티티와 스토리지를 가진 두 개의 파드가 절대 실행되지 않는 것을 보장하므로, 스테이트풀셋은 파드가 더 이상 실행되지 않는다는 것을 확실할 때까지 대체 파드를 생성할 수 없으며, 생성해서도 안된다.
 - 이 경우 오직 클러스터 관리자가 알려줘야만 알 수 있고, 이를 위해 관리자는 파드를 삭제하거나 전체 노드를 삭제해야 한다.(노드 삭제시 노드에 스케줄링된 모든 파드가 삭제됨)
 - 노드가 다운된 상태에서 파드를 삭제하게 되면 쿠버네티스 클러스터(마스터) 기준으로는 파드가 삭제됐으나, 다운된 노드에서는 이를 알 방법이 없기 때문에 실제 노드에 스케줄링된 파드가 삭제되지는 않는다. 이 경우 파드를 강제로 삭제해야 할 수 있다.
 - 노드가 더 이상 실행중이 아니거나 연결 불가함을 아는 경우가 아니라면, 스테이트풀 파드를 강제로 삭제해서는 안된다.(영구적으로 유지되는 좀비 파드가 생성될수 있다.)


## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
