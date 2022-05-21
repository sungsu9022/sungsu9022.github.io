---
title: "[kubernetes-in-action] 5. 서비스 : 클라이언트가 파드를 검색하고 통신을 가능하게 함."
author: sungsu park
date: 2020-08-16 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---


# 5. 서비스 : 클라이언트가 파드를 검색하고 통신을 가능하게 함.
> 파드가 다른 파드에게 제공하는 서비스를 사용하려면 다른 파드를 찾는 방법이 필요하다.
> 쿠버네티스에서는 서비스를 제공하는 서버의 정확한 IP 주소나 호스트 이름을 지정해 각 클라이언트 앱을 구성하는것과는 다른 방식이 필요하다.

## 기존의 방식과 다른 방식이 필요한 이유
 - 쿠버네티스 파드는 일시적이고, 언제든 다른 노드로 이동될 수 있기 때문(클러스터 IP는 언제든 변경될수 있다.)
 - 쿠버네티스는 노드에 파드를 스케줄링한 후 파드가 시작되기 바로 직전에 파드의 IP주소를 할당함. 따라서 클라이언트에서 특정 파드의 IP주소를 미리 알 수 없다.
 - 오토 스케일링 등을 위해서는 여러 파드가 동일한 서비스로 제공할 수 있어야 하는데, 각 파드마다 고유한 IP주소가 있고, 클라이언트는 서비스를 지원하는 파드의 수와 IP에 상관없이 단일 IP주소로 모든 파드에 엑세스할수 있어야 한다.

## 5.1 서비스 소개
 - 쿠버네티스의 서비스는 동일한 서비스를 제공하는 파드 그룹에 지속적인 단일 접점을 만들어주는 리소스이다.

### 서비스 설명
 - 서비스를 만들고 클러스터 외부에서 엑세스할 수 있도록 구성하면 외부 클라이언트가 파드에 연결할 수 있는 하나의 고정 IP가 노출
 - 서비스가 관리하는 파드들의 IP가 변경되더라도 서비스의 IP주소는 변경되지 않는다.
 - 내부 클라이언트와 외부 클라이언트 모두 서비스로 파드에 접속

<img width="616" alt="스크린샷 2020-08-08 오후 10 16 46" src="https://user-images.githubusercontent.com/6982740/89711371-e363b000-d9c4-11ea-9cbe-4c75809e6e7f.png">

### 5.1.1 서비스 생성
 - 서비스를 지원하는 파드가 한개 혹은 그 이상일 수 있다.
 - 서비스 연결은 뒷단의 모든 파드로 로드밸런싱된다.
 - 서비스에서도 레플리카셋과 동일하게 레이블 셀렉터 매커니즘을 그대로 적용된다.

<img width="618" alt="스크린샷 2020-08-08 오후 10 18 48" src="https://user-images.githubusercontent.com/6982740/89711417-24f45b00-d9c5-11ea-96ad-9472313548e8.png">

#### kubectl expose
 - 서비스를 생성하기 가장 쉬운 방법은 kubectl expose 명령어를 사용하는 것이다.
 - expose 명령어를 사용하는 것 대신 kubernetes 명령어를 이용해 서비스 리소스를 생성할 수도 있다.

#### YAML 디스크립터를 통한 서비스 생성
``` sh
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80                   # 서비스가 사용할 포트
    targetPort: 8080     # 서비스가 포워드할 컨테이너 포트
  selector:
    app: kubia                # app=kubia 레이블이 있는 모든 파드가 이 서비스에 포함된다는것을 의미
```

#### 새 서비스 검사하기
``` sh
# 서비스 생성
kubectl create -f kubia-svc.yaml

# 서비스 정보 조회 ( 내부 클러스터 IP가 할당되었는지 확인 )
kubectl get svc
```
 - 이렇게 클러스터 IP가 할당되면 클러스터 내부에서는 바로 엑세스할 수 있다.

#### 실행인 컨테이너에 원격으로 명령어 실행
 - kubectl exec 명령어를 사용하면 기존 파드의 컨테이너 내에서 원격으로 임의의 명령어를 실행할 수 있다.

``` sh
# 특정 Pod에 접속하여 서비스의 클러스터 IP로 http 요청
kubectl exec kubia-4dkws -- curl -s http://10.102.206.16

# pod shell 접근
kubectl exec --stdin --tty kubia-4dkws -- /bin/bash
```

##### 더블 대시를 사용하는 이유
 - 명령어의 더블 대시(--)는 kubectl 명령줄 옵션의 끝을 의미함.
 - 더블 대시 뒤의 모든 것은 파드 내에서 실행돼야 하는 명령이다.
 - ```kubectl exec kubia-4dkws -- curl -s http://10.102.206.16``` 의 예제에서 더블 대시가 없다면 -s 옵션은 kubectl exec의 옵션으로 해석하여 처리 되지 않는다.

<img width="702" alt="스크린샷 2020-08-16 오후 7 48 01" src="https://user-images.githubusercontent.com/6982740/90332623-6904e280-dff9-11ea-908a-12e6f00ba2f2.png">

#### 서비스의 세션 어피니티 구성
 - 동일한 클라이언트에서 요청하더라도 서비스 프록시가 각 연결을 임의의 파드를 선택해서 연결을 다시 전달(forward)하기 때문에 요청할 때마다 다른 파드가 선택된다.
 - 특정 클라이언트의 모든 요청을 매번 같은 파드로 리디렉션하려면 서비스의 세션 어피니티(sessionAffinity)속성을 기본값 None 대신 ClientIP로 설정하면 된다.
 - 쿠버네티스에서는 None, ClientIP두 가지 유형의 서비스 세션 어피니티만 지원.
 - 서비스 레벨에서는 HTTP 수준에서는 작동하지 않고 TPC / UDP 패킷을 처리하고 그들이 가지고 있는 payload는 신경쓰지 않는다.(쿠키 기반으로 할 수 없음)

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP # 동일한 클라이언트 IP의 모든 요청을 동일한 파드로 전달
```

#### 동일한 서비스에서 여러 개의 포트 노출
 - 파드가 2개의 포트(http : 8080, https : 8443)을 수신한다면 하나의 서비스를 사용해 포트 80과 433을 파드의 포트 8080과 8443으로 전달할 수 있음.
 - 하나의 서비스를 사용해 멀티 포트 서비스를 사용하면 단일 클러스터 IP로 모든 서비스 포트가 노출된다.

#### 이름이 지정된 포트 사용
 - 포트 번호가 잘 알려진 경우가 아니더라도 서비스 스펙을 좀 더 명확히 할 수 있는 방법.
 - 나중에 서비스 스펙을 변경하지 않고도 pod의 포트 번호를 변경할 수 있다는 큰 장점이 있음.

``` yaml
# 파드 정의에 포트 이름 사용
kind: Pod
spec:
  ports:
  - name: http
    containerPort: 8080
  - name: https
    containerPort: 8443

# 서비스에 이름이 지정된 포트 참조
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - name: http
    port: 80
    targetPort: http       # 포트 80은 컨테이너 포트의 이름이 http인 것에 매핑
  - name: https
    port: 443
    targetPort: https      # 포트 443은 컨테이너 포트의 이름이 https인 것에 매핑
  selector:
    app: kubia

```

### 5.1.2 서비스 검색
 - 서비스의 파드는 생성되기도 하고 사라지기도 하고, 파드 IP가 변경되거나 파드 수는 늘어나거나 줄어들 수도 있다.
 - 이러한 상황 속에서 항상 서비스의 IP주소로 엑세스할 수 있어야 한다.
 - 쿠버네티스는 클라이언트 파드가 서비스의 IP와 포트를 검색할 수 있는 방법을 제공한다.

#### 환경변수를 통한 서비스 검색
 - 파드가 시작되면 쿠버네티스는 해당 시점에 존재하는 각 서비스를 가리키는 환경변수 세트를 초기화한다.
 - 환경변수 네이밍 규칙은 서비스 이름의 대시(-)는 밑줄(_)로 변환되고 서비스 이름이 환경변수 이름의 접두어로 쓰이면서 모든 문자는 대문자로 표시한다.

``` sh
kubectl exec kubia-4dkws env

# KUBIA_SERVICE_HOST=10.102.206.16     # 서비스 클러스터 IP
# KUBIA_PORT_80_TCP_PORT=80              # 서비스가 제공되는 포트
```

#### DNS를 통한 서비스 검색
 - kube-system 네임스페이스에 파드 중 kube-dns라는 게 있었다.
 - kube-dns는 DNS 서버를 실행하며 클러스터에서 실행중인 다른 모든 파드는 자동으로 이를 사용하도록 구성되는 파드이다.
 - 파드에서 실행중인 프로세스에서 수행된 모든 DNS 쿼리는 시스템에서 실행중인 모든 서비스를 알고 있는 쿠버네티스의 자체 DNS 서버로 처리된다.
 - 각 서비스는 내부 DNS 서버에서 항목을 가져오고 서비스 이름을 알고 있는 클라이언트 파드는 환경변수 대신 FQDN(정규화된 도메인 이름)으로 엑세스 할수 있다.

#### FQDN을 통한 서비스 연결
> backend-database.default.svc.cluster.local
> "backend-database" -> 서비스 이름
> "default" -> 네임스페이스
> "svc.cluster.local"  -> 모든 클러스트의 로컬 서비스 이름에 사용되는 도메인 접미사

 - 클라이언트는 여전히 서비스의 포트번호를 알아야 한다. 표준 포트가 아닌 경우 문제가 될수 있다.(환경변수에서 포트 번호를 얻을수 있어야 함)
 - 접미사와 네임스페이스는 생략이 가능하다.
 - 위 예제에서는 "backend-database" FQDN만으로 서비스에 엑세스할 수 있다.

#### 파드의 컨테이너 내에서 셸 실행
 - 파드 컨테이너 내부의 DNS resolver가 구성되어 있기 때문에 네임스페이스와 접미사를 생략할 수 있다.

``` sh
# 파드 컨테이너 냉서 셸 실행
kubectl exec -it kubia-4dkws bash

# DNS resolver 확인
cat /etc/resolv.conf

# FQDN을 통한 서비스 호출(모두 같은 결과)
curl http://kubia.default.svc.cluster.local
curl http://kubia.default
curl http://kubia
```

#### 서비스 IP에 핑을 할 수 없는 이유
 - 서비스로 crul은 동작하지만 핑은 응답이 오지 않는다.
 - 이는 서비스의 클르서트 IP가 가상 IP이므로 서비스 포트와 결합된 경우에만 의미가 있기 때문

## 5.2 클러스터 외부에 있는 서비스 연결
 - 클러스터에서 실행중인 파드는 내부 서비스에 연결하는 것처럼 외부 서비스에 연결할 수 있다.

### 5.2.1 서비스 엔드포인트 소개
 - 서비스는 파드에 직접 연결(link)되지 않는다. 대신 엔드포인트 리소스가 그 사이에 있다.
 - 파드 셀렉터는 서비스 스펙에 정의돼 있지만 들어오는 연결을 전달할 때 직접 사용하지 않고, IP와 포트 목록을 작성하는데 사용되며, 엔드포인트 리소스에 저장된다.

``` sh
# service 정보 조회
kubectl describe svc kubia

# kubia endpoints 조회
kubectl get endpoints kubia
```

### 5.2.2 서비스 엔드포인트 수동 구성
 - 서비스의 엔드포인트를 서비스와 분리하면 엔드포인트를 수동으로 구성하고 업데이트할수 있다.
 - 수동으로 관리되는 엔드포인트를 사용해 서비스를 만들려면 서비스와 엔드포인트 리소스를 모두 만들어야 한다.

#### 셀렉터 없이 서비스 생성
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service          # 엔드포인트 오브젝트 이름과 일치해야 함.
spec:                                          # spec에   selector를 정의하지 않음.
  ports:
  - port: 80
```

#### 셀렉터가 없는 서비스에 관한 엔드포인트 리소스 생성
 - 엔드포인트 오브젝트는 서비스 이름과 같아야 하고, 서비스를 제공하는 대상 IP주소와 포트 목록을 가져야 함.

``` yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service         # 서비스 이름과 일치시킴.
subsets:
  - addresses:                           # 서비스가 연결을 전달할 엔드포인트 IP 생성
    - ip: 11.11.11.11
    - ip: 22.22.22.22
    ports:
    - port: 80                              # 엔드포인트의 대상 포트
```

<img width="572" alt="스크린샷 2020-08-16 오후 8 31 40" src="https://user-images.githubusercontent.com/6982740/90333286-8177fb80-dfff-11ea-81a4-86a2872095ed.png">

#### 외부 엔드포인트를 가지는 서비스를 만들어야 하는 목적
 - 나중에 외부 서비스를 쿠버네티스 내에서 실행되는 파드로 마이그레이션하기로 한 경우 서비스에 셀렉터를 추가해 엔드포인트를 자동으로 관리 할 수 있다.
 - 이를 통해 서비스의 실제 구현이 변경되는 동안에도 서비스 IP 주소가 일정하게 유지될 수 있다.

### 5.2.3 외부 서비스를 위한 별칭 생성
#### ExternalName 서비스 생성
 - 외부 서비스의 별칭으로 하려는 경우 유형(type) 필드를 ExternalName으로 설정하면 된다.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName                                     # ExternalName 유형
  externalName: api.somecompany.com      # FQDN(Fully Qualified Domain Name) 이름 지정
  ports:
  - port: 80
```

 - 여기서 FQDN을 사용하는 대신 external-service.default.svc.clster.local 도메인 이름으로 외부 서비스에 연결할수도 있다.
 - ExternalName 서비스는 DNS 레벨에서만 구현된다.
 - 서비스에 연결하는 클라이언트는 서비스 프록시를 완전히 무시하고 외부 서비스에 직접 연결된다.
 - 이러한 이유로 Cluster IP를 얻을 수 없음.

## 5.3 외부 클라이언트에 서비스 노출
 - 쿠버네티스느 외부에서 서비스를 엑세스할 수 있는 방법을 몇가지 제공해준다.

<img width="556" alt="스크린샷 2020-08-16 오후 8 37 34" src="https://user-images.githubusercontent.com/6982740/90333393-5510af00-e000-11ea-92ce-03a41d66bbf9.png">


### 5.3.1 노트포트 서비스
 - 서비스를 생성하고 유형을 노드포트로 설정하는 방법

#### 노드포트 서비스 생성
 - nodePort를 생략할 경우 쿠버네티스가 임의의 포트를 선택한다.

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort                     # 노드포트 서비스 유형
  ports:
  - port: 80                               # 서비스 클러스터 IP 포트
    targetPort: 8080                 # 서비스 대상 파드의 포트
    nodePort: 30123                 # 각 클러스터 노드의 포트 30123을 통해 서비스에 엑세스 할수 있음.
  selector:
    app: kubia
```

#### 노드포트 서비스 확인
``` sh
# 노드포트 서비스 생성
kubectl create -f kubia-svc-nodeport.yaml

# 노드포트 서비스 확인
# kubectl get svc kubia-nodeport

# 노드 Ip 조회(minikube에서는 안됨)
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# minikube nodeport service(노드포트 서비스를 로컬에서 접근 가능하도록 처리)
minikube service kubia-nodeport
crul http://127.0.0.1:53446/

|-----------|----------------|-------------|-------------------------|
| NAMESPACE |      NAME      | TARGET PORT |           URL           |
|-----------|----------------|-------------|-------------------------|
| default   | kubia-nodeport |          80 | http://172.17.0.2:30123 |
|-----------|----------------|-------------|-------------------------|
🏃  Starting tunnel for service kubia-nodeport.
|-----------|----------------|-------------|------------------------|
| NAMESPACE |      NAME      | TARGET PORT |          URL           |
|-----------|----------------|-------------|------------------------|
| default   | kubia-nodeport |             | http://127.0.0.1:53446 |
|-----------|----------------|-------------|------------------------|
```

<img width="609" alt="스크린샷 2020-08-16 오후 8 42 49" src="https://user-images.githubusercontent.com/6982740/90333485-10394800-e001-11ea-87ec-1c7d6e0559fa.png">

#### 노드포트 서비스의 단점
 - 클라이언트가 하나의 노드에만 요청하는 경우 노드에 장애가 발생할 경우 더 이상 서비스에 엑세스할 수 없으므로, 노드 앞에 로드밸런서를 배치하는 것이 좋다.

### 5.3.2 외부 로드밸런서로 서비스 노출
 - 쿠버네티스 클러스터는 로드밸런서를 자동으로 프로비저닝하는 기능을 제공한다.
 - 쿠버네티스가 로드밸런서 서비스를 지원하지 않는 환경에서 실행중인 경우 로드밸런서는 프로비저닝되지는 않지만 서비스는 여전히 노드포트 서비스처럼 작동한다.

#### 로드밸런서 서비스 생성
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer              # 쿠버네티스 클러스터를 호스팅하는 인프라에서 로드밸런서를 얻을수 있다.
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

#### 로드밸런서를 통한 서비스 연결
``` sh
# 로드밸런서 서비스 생성
kubectl create -f kubia-svc-loadbalancer.yaml

# 확인
kubectl get svc kubia-loadbalancer
```

<img width="617" alt="스크린샷 2020-08-16 오후 8 57 20" src="https://user-images.githubusercontent.com/6982740/90333732-17f9ec00-e003-11ea-88c5-41a070370f14.png">


#### 세션 어피니티와 웹 브라우저
 - 웹 브라우저에서 세션 어피티니가 None이더라도 같은 파드로 계속 요청하는 현상을 볼수 있음.
 - 그 이유는 브라우저의 http keep-alive header을 사용하기 때문이다.

### 5.3.3 외부 연결의 특성 이해
#### 불필요한 네트워크 홉의 이해와 예방
 - 외부 클라이언트가 노드포트로 서비스에 요청할 경우 임의로 선택된 파드가 연결을 수신한 동일한 노드에서 실행중일 수도 있고, 그렇지 않을 수도 있다.
 - 파드에 도달하려면 추가적인 네트워크 홉이 필요할 수 있으며 이것이 항상 바람직한 것은 아니다.
 - 요청을 수신한  노드에서 실행중인 파드로만 외부 트래픽을 전달하도록 서비스를 구성해 추가 홉을 방지할 수 있는 옵션을 제공한다.

``` yaml
spec:
   externalTrafficPolicy: Local
```

#### externalTrafficPolicy을 이용할때 주의사항
 - 서비스 프록시는 로컬에 실행중인 파드를 선택하는데 로컬 파드가 업으면 요청을 중단시킨다.(로드밸런서는 파드가 하나 이상 있는 노드에만 연결을 전달하도록 해야 함)
 - 모든 파드에 균등하게 분산되지 않을 수 있다.

<img width="451" alt="스크린샷 2020-08-16 오후 9 01 04" src="https://user-images.githubusercontent.com/6982740/90333794-9c4c6f00-e003-11ea-88fc-6314f2e18726.png">


#### 클라이언트 IP가 보존되지 않음 인식
 - 노드포트로 연결을 수신하면 패킷에서 소스 네트워크 주소 변환(SNAT)이 수행되므로 패킷의 소스 IP가 변경된다.
 - 웹 서버의 경우 엑세스 로그에 브라우저의 IP를 표시할 수 없다는 것은 의미함..
 - 로컬 외부 트래픽 정책(Local External Traffic Policy은 연결을 수신하는 노드와 대상 파드를 호스팅하는 노드 사이에 추가 홉이 없기 때문에 클라이언트 IP 보존에 영향을 미친다.

## 5.4 인그레스 리소스로 서비스 외부 노출
<img width="617" alt="스크린샷 2020-08-16 오후 9 03 57" src="https://user-images.githubusercontent.com/6982740/90333839-0b29c800-e004-11ea-8afd-0bcb04165574.png">

### 인그레스가 필요한 이유
 - 인그레스는 한 IP 주소로 수십 개의 서비스에 접근이 가능하도록 지원한다.
 - 네트워크 스택의 애플리케이션 계층(HTTP)에서 작동하며, 서비스가 할 수 없는 쿠키 기반 세션 어피니티 등과 같은 기능 제공이 가능하다.

#### Minikube에서 인그레스 애드온 활성화
``` sh
# minikube addons 확인
minikube addons list

# ingress addons 활성화( minikube start --vm=true 로 시작해야 가능)
minikube addons enable ingress

# 기존에 설치된 docker 기반 minikube delete
minikube delete

# minikube vm 모드로 시작
minikube start --vm=true --driver=hyperkit

# 모든 네임스페이스 pods 조회
kubectl get po --all-namespaces
```

### 5.4.1 인그레스 리소스 생성
``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com                       # 인그레스는 kubia.example.com 도메인 이름으로 서비스에 매핑된다.
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport          # 모든 요청은 kubia-nodeport 서비스의 포트 80으로 전달된다.
          servicePort: 80
```

### 5.4.2 인그레스 서비스 엑세스
``` sh
# ingress 리소스 생성
kubectl create -f kubia-ingress.yaml

# ingresses 조회
kubectl get ingresses

# host 설정
sudo vi etc/hosts
192.168.64.2	kubia.example.com

# 호출
curl http://kubia.example.com
```

#### 인그레스 동작 방식
<img width="699" alt="스크린샷 2020-08-16 오후 9 38 49" src="https://user-images.githubusercontent.com/6982740/90334484-e2580180-e008-11ea-8be5-115b7310f360.png">

### 5.4.3 하나의 인그레스로 여러 서비스 노출
 - 규칙과 경로가 모두 배열이라 여러 항목을 가질 수 있다.
 - 여러 호스트(host)와 경로(path)를 여러 서비스(backend.serviceName)에 매핑할 수 있다.

#### 동일한 호스트의 다른 경로로 여러 서비스 매핑
``` yaml
   - host: kubia.example.com
      http:
          paths:
            - path: /kubia
               backend:                            # kubia.example.com/kubia으로의 요청을 kubia 서비스로 라우팅된다.
                    serviceName: kubia
                    servicePort: 80
            - path: /bar
               backend:                            # kubia.example.com/bar으로의 요청을 bar 서비스로 라우팅된다.
                    serviceName: bar
                    servicePort: 80
```

#### 서로 다른 호스트로 서로 다른 서비스 매핑하기
``` yaml
spec:
   rules:
    - host: foo.example.com
       http:
          paths:
            - path: /
               backend:                            # foo.example.com으로의 요청을 foo 서비스로 라우팅된다.
                    serviceName: foo
                    servicePort: 80
    - host: bar.example.com
       http:
          paths:
            - path: /
               backend:                            # bar.example.com으로의 요청을 bar서비스로 라우팅된다.
                    serviceName: bar
                    servicePort: 80
```

### 5.4.4 TLS 트래픽을 처리하도록 인그레스 구성
#### 인그레스를 위한 TLS 인증서 생성
``` sh
# 개인키 생성
openssl genrsa -out tls.key 2048

# 인증서 생성
openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com

# 시크릿 리소스 생성
kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
```

#### CertificateSigningRequest 리소스로 인증서 서명
 - 인증서를 직접 서명하는 대신 CSR 리소스를 만들어 인증서에 서명할 수 있음.

``` sh
kubectl certificate approve <name of th CSR>
```

#### tls 트래픽을 처리하는 인그레스 생성
``` yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia-tls
spec:
  tls:                                           # TLS 구성
  - hosts:
    - kubia.example.com
    secretName: tls-secret       # 개인키와 인증서는 위에서 생성한 tls-secret 참조
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```

``` sh
# https request 호출
curl -k -v https://kubia.example.com/kubia
```

## 5.5 레디니스 프로브 ( 파드가 연결을 수락할 준비가 됐을 때 신호 보내기 )
 - 파드는 구성에 시간이 걸릴 수 있다. 데이터를 로드하는 데 시간이 걸리거나, 첫번 째 사용자 요청이 너무 오래걸리거나 사용자 경험에 영향을 미치는 것을 방지하고자 준비 절차를 수행해야 할 수도 있다.

### 5.5.1 레디니스 프로브 소개
 - 라이브니스 프로브와 비슷하게 파드에 레디니스 프로브를 정의할 수 있다.
 - 주기적으로 호출되며 특정 파드가 클라이언트 요청을 수신할 수 있는지를 결정한다.
 - 애플리케이션 특성에 따라 상세한 레디니스 프로브를 작성하는 것은 개발자의 몫

#### 레디니스 프로브 유형
 - Exec 프로브 : 프로세스를 실행
 - HTTP GET 프로브 : HTTP GET 요청 수행
 - TCP 소켓 프로브 : 컨테이너에 TCP 연결 확인

#### 레디니스 프로브의 동작
 - 컨테이너가 시작될 때 쿠버네티스는 첫 번째 레디니스 점검을 수행하기 전에 구성 가능한 시간이 경과하기를 기다릴 수 있도록 구성 가능
 - 주기적으로 프로브를 호출하고 레디니스 프로브의 결과에 따라 작동
 - 파드가 준비되지 않았다고 하면 서비스에서 제거하고, 파드가 준비되면 서비스에 다시 추가한다.
 - 컨테이너가 준비 상태 점검에 실패하더라도 컨에티너가 종료되거나 다시 시작시키지 않는다.

#### 라이브니스 프로브 vs 레디니스 프로브
 - 라이브니스 프로브는 상태가 좋지 않은 컨테이너를 제거하고 새롭고 건강한 컨테이너로 교체해 파드의 상태를 정상으로 유지시킨다.
 - 레디니스 프로브는 요청을 처리할 준비가 된 파드의 컨테이너만 요청을 수신하도록 한다.(L7 health check와 유사)

<img width="586" alt="스크린샷 2020-08-16 오후 9 55 14" src="https://user-images.githubusercontent.com/6982740/90334809-306e0480-e00b-11ea-9cd2-04189b331957.png">

#### 레디니스 프로브가 중요한 이유
 - 파드 그룹이 다른 파드에서 제공하는 서비스에 의존한다고 했을때(웹 앱 -> 백엔드 데이터베이스) 웹 앱 파드중 하나만 DB에 연결할 수 없는 경우, 요청을 처리할 준비가 되지 않았다고 신호를 주는게 현명할 수 있다.
 - 레디니스 프로브를 사용하면 클라이언트가 정상 상태인 파드하고만 통신할 수 있다.
 - 그래서 시스템에 문제가 있다는 것을  절대 알아차리지 못한다.

### 5.5.2 파드에 레디니스 프로브 추가
``` sh
# replicaSet 수정
kubectl edit rs kubia


# yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
        - name: kubia
          image: sungsu9022/kubia
          ports:
            - name: http
              containerPort: 8080
            readinessProbe:                      # 파드의 각 컨테이너에 레디니스 프로브를 정의
              exec:                                      # ls /var/ready 명령어를 주기적으로 수행하여 존재하면 0(성공), 그렇지 않으면 다른 값(실패)
                command:
                  - ls
                  - /var/ready
```

 - 이렇게 하면 /var/ready 파일이 없으므로 READY에 준비된 컨테이너가 없다고 표시된다.

``` sh
# /var/ready 파일 생성
kubectl exec kubia-4wjqj -- touch /var/ready

# 확인(kubia-4wjqj 가 READY 1/1로 변경됨)
kubectl get pods

# 레디니스 프로브 조회
kubectl describe pod kubia-4wjqj

# 하나의 READY 파드로 서비스를 호출
curl http://kubia.example.com
```

### 5.5.3 실제 환경에서 레디니스 프로브가 수행해야 하는 기능
 - 서비스에서 파드를 수동으로 추가하거나 제거하려면 파드와 서비스의 레이블 셀렉터에 enabled=true 레이블을 추가한다. 서비스에서 파드를 제거하려면 레이블을 제거하라.

#### 레디니스 프로브를 항상 정의하라
 - 파드의 레디니스 프로브를 추가하지 않으면 파드가 시작하는 즉시 서비스 엔드포인트가 된다.
 - 여전히 시작 단계로 수신 연결을 수락할 준비가 되지 않은 상태에서 파드로 전달된다.
 - 따라서 클라이언트가 Connection Refused 유형의 에러를 보게 된다.
 - 기본 URL에 HTTP 요청을 보내더라도 항상 레디니스 프로브를 정의해야 한다.

#### 레디니스 프로브에 파드의 종료 코드를 포함하지 마라
 - 파드가 종료할 때, 실행되는 앱은 종료 신호를 받자마자 연결 수단을 중단한다.
 - 쿠버네티스는 파드를 삭제하자마자 모든 서비스에서 파드를 제거하기 때문에 굳이 별도로 이런 처리를 할 필요가 없다.

## 5.6. 헤드리스 서비스로 개별 파드 찾기
 - 클라이언트가 모든 파드에 연결해야 하는 경우 어떻게 할수 있을까? 파드가 다른 파드에 각각 연결해야 하는 경우 어떻게 해야 할까?
 - 클라이언트가 모든 파드에 연결하려면 각 파드의 IP를 알아야 한다.
 - 쿠버네티스는 클라이언트가 DNS 조회로 파드 IP를 찾을 수 있도록 한다.
 - 쿠버네티스 서비스에 클러스터 IP가 필요하지 않다면 ClusterIP 필드를 None으로 설정하여 DNS 서버는 하나의 서비스 IP 대신 파드 IP 목록들을 반환한다.
 - 이때 각 레코는 현재시점 기준으로 서비스를 지원하는 개별 파드의 IP를 가리킨다.
 - 따라서 클라이언트는 간단한 DNS A 레코드 조회를 수행하고 서비스에 포함된 모든 파드의 IP를 얻을 수 있다.

### 5.6.1 헤드리스 서비스 생성
 - 서비스 스펙의 clusterIP필드를 None으로 설정하면 클라이언트가 서비스의 파드에 연결할 수 있는 클러스터 IP를 할당하지 않기 떄문에 서비스가 헤드리스 상태가 된다.
 - 클러스터 Ip가 없고 엔드포인트에 파드 셀렉터와 일치하는 파드가 포함돼 있음을 확인할 수 있다.

``` yarml
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  clusterIP: None                     # 헤드리스 서비스로 만드는 spec 옵션
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

### 5.6.2 DNS로 파드 찾기
 - 실제로 클라이언트 관점에서는 헤드리스 서비스를 사용하나 일반 서비시를 사용하나 관계 없이 서비스의 DNS 이름에 연결해 파드에 연결할 수 있다.
 - 차이점이 있다면 헤드리스 서비스의 경우 클라이언트는 서비스 프록시 대신 파드에 직접 연결한다.
  - 헤드리스 서비스는 여전히 파드간에 로드밸런싱을 제공하지만 서비스 프록시 대신 DNS 라운드 로빈 매커니즘으로 처리된다.

``` sh
# dnsutils pod 생성
kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity

# 헤드리스 서비스를 위해 반환된 DNS A 레코드( 레코드 목록이 표시된다. )
kubectl exec dnsutils nslookup kubia-headless

# 일반 서비스 nslookup ( 클러스터 IP가 표시된다. )
kubectl exec dnsutils nslookup kubia
```

### 5.6.3 모든 파드 검색 - 준비되지 않은 파드도 포함
 - 쿠버네티스가 파드의 레디니스 상태에 관계 없이 모든 파드를 서비스에 추가되게 하려면 서비스에 다음 어노테이션을 추가해야 한다.
 - tolerate-unready-endpoints는 deprecated되었고, publishNotReadyAddresses를 사용해서 동일한 기능을 처리할 수 있다.( https://github.com/kubedb/project/issues/242 )

``` yaml
kind: Service
#metadata:
#  annotations:
#      service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec
   publishNotReadyAddresses: true

```

## 5.7 서비스 문제 해결 FAQ
> 서비스로 파드에 엑세스할 수 없는 경우 다음 내용을 확인해보면 도움이 된다.
 - 외부가 아닌 클러스터 내에서 서비스의 클러스터 IP에 연결되는지 확인
 - 서비스에 엑세스할 수 있는지 확인하려고 서비스 IP로 핑을 하는 케이스(핑 동작 안함)
 - 레디니스 프로브를 정의했다면 성공했는지 확인하라, pod의 READY 여부 확인
 - 파드가 서비스의 일부인지 확인하려면 kubectl get endpoints를 사용해 해당 엔드포인트 오브젝트를 확인하라.
 - FQDN이나 그 일부로 서비스에 엑세스하려고 하는데 작동하지 않는 경우 FQDN 대신 클러스터 IP를 사용해 엑세스할수 있는지 확인하라
 - 대상 포트가 아닌 서비스로 노출된 포트에 연결하고 있는지 확인하라
 - 파드 IP에 직접 연결해 파드가 올바른 포틍 연결돼있는지 확인하라
 - 앱이 로컬호스트에만 바인딩하고 있는지 확인하라.

## 5.8 요약
 - 서비스는 안정된 단일 IP 주소와 포트로 특정 레이블 셀렉터와 일치하는 여러 개의 파드를 노출
 - 기본적으로 클러스터 내부에서 서비스에 엑세스할 수 있지만 유형을 노드포트 또는 로드밸런서로 설정해 클러스터 외부에서 서비스에 엑세스할 수 있다.
 - 파드는 환경변수를 검색해 IP 주소와 포트로 서비스를 검색 할수 있다.
 - 엔드포인트 리소스를 만드는 대신 셀렉터 설정 없이 서비스 리소스를 생성해 클러스터 외부에 있는 서비스를 검색하고 통신할 수 있다.
 - ExternalName 서비스 유형으로 외부 서비스에 대한 DNS CNAME(별칭)을 제공할 수 있다.
 - 단일 인그레스로 여러 HTTP 서비스를 노출할 수 있다.
 - 파드 컨테이너의 레디니스 프로브는 파드를 서비스 엔드포인트에 포함해야 하는지 여부를 결정한다.
 - 헤드리스 서비스를 생성하면 DNS로 파드 IP를 검색할 수 있다.

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
