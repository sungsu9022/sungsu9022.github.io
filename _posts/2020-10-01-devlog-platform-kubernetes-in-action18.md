---
layout: post
title: "[kubernetes-in-action] 18. 쿠버네티스 확장"
subtitle: "[kubernetes-in-action] 18. 쿠버네티스 확장"
categories: devlog
tags: platform
---

# 18. 쿠버네티스 확장
## 18.1 사용자 정의 API 오브젝트 정의
 - 쿠버네티스는 CustomResourceDefinition 오브젝트를 통해 디플로이먼트와 같은 높은 수준의 리소스를 직접 정의하여 사용할 수 있다.
 - 사용자 정의 컨트롤러는 이러한 높은 수준의 오브젝트를 관찰하고 이를 기반으로 낮은 수준의 오브젝트를 생성하도록 하여 의도된 동작을 이끌어낼 수 있다.

### 18.1.1 CustomResourceDefinition 소개
 - 새로운 리소스 유형을 정의하려면 CustomResourceDefinition(CRD) 오브젝트를 쿠버네티스 API 서버에 게시하면 된다.
 - CRD가 게시되면 사용자는 다른 쿠버네티스 리소스와 마찬가지로 JSON 또는 YAML 매니페스트를 API 서버에 게시해 사용자 정의 리소스 인스턴스를 생성할 수 있다.(1.7 이전에는 ThirdPartyResource 오브젝트를 통해서 정의했었다)
 - 각 CRD에는 일반적으로 관련 컨트롤러(사용자 정의 오브젝트를 기반으로 작업을 수행하는 활성화된 구성 요소)가 있어야 한다.
     - 다른 쿠버네티스 리소스들의 관련된 컨트롤러가 존재하는 것과 같은 방식이다.

#### CustomResourceDefinition 예제
 - 웹사이트의 HTML, CSS, PNG 등 정적 리소스를 포함하는 Website 유형의 오브젝트를 만들어 보자.

<img width="537" alt="스크린샷 2020-10-01 오후 9 54 01" src="https://user-images.githubusercontent.com/6982740/94811648-a0f0a980-0430-11eb-833c-a1de962e1203.png">

``` yaml
kind: Website
metadata:
  name: kubia    # 웹사이트 리소스 이름
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git   # 웹사이트 파일을 저장한 깃 리포지토리
```


#### CustomResourceDefinition 오브젝트 생성
 - 사용자 정의 오브젝트 인스턴스를 만들기 전에 CustomResourceDefinition을 API 서버에 게시해야 한다.

``` yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com
spec:
  scope: Namespaced
  group: extensions.example.com
  version: v1
  names:
    kind: Website
    singular: website
    plural: websites
```

 - 여기서 CustomResourceDefinition의 metadata.name이 긴 이유는 여러 CustomResourceDefinition간의 이름 충돌을 방지하기 위함이다. 위와 같이 길게 해주는 것이 좋다.
 - 위와 같이 정의한 이후에 사용자 정의 Website 리소스의 인스턴스를 만들때는 정의에 있는 api 그룹 ```extensions.example.com```을 지정해주어야 한다.

``` sh
# website crd 생성
kubectl create -f website-crd.yaml
```

#### 사용자 정의 리소스의 인스턴스 생성

``` sh
# website 인스턴스 생성
kubectl create -f kubia-website.yaml
```

#### 사용자 정의 리소스의 인스턴스 검색
```  sh
# 웹사이트 리소스 조회
kubectl get websites

# kubia website 리소스 조회
kubectl get website kubia -o yaml
```

<img width="893" alt="스크린샷 2020-10-01 오후 10 09 22" src="https://user-images.githubusercontent.com/6982740/94813315-cbdbfd00-0432-11eb-9b47-331f793ec865.png">

#### 사용자 정의 오브젝트의 인스턴스 삭제
``` yaml
# 삭제
kubectl delete website kubia
```

 - 사용자 정의 오브젝트도 다른 쿠버네티스 리소스들처럼 동일한 kubectl 클라이언트로 추가, 검색, 삭제 등을 할 수 있다.

#### 사용자 정의 오브젝트의 동작
 - 사용자 정의 오브젝트를 만든다는 것이 오브젝트를 생성할 때 항상 어떤 일이 일어나게 해야 한다는 것은 아니다.
 - 특정 사용자 정의 오브젝트는 컨피그맵과 같이 일반적인 매커니즘을 사용하는 대신, 데이터를 저장하는 용도로만 사용할 수도 있다.
 - 또한 이렇게 인스턴스가 생성되더라도 컨트롤러가 없다면 아무런 동작을 하지 않을 것이다. 

### 18.1.2 사용자 정의 컨트롤러로 사용자 정의 리소스 자동화
 - Website 오브젝트를 의도한대로 동작시키기 위해서는 Website 컨트롤러를 생성하고 배포해야 한다.
 - 컨트롤러는 관리되지 않는 파드(unmanaged pod) 대신 디플로이먼트 리소스를 직접 생성하도록 하는 것이 좋다.
 - 어쩃든 이를 위해 완벽하지는 않지만 간단한 버전의 컨트롤러를 만든다면 다음과 같다.

<img width="603" alt="스크린샷 2020-10-01 오후 10 13 43" src="https://user-images.githubusercontent.com/6982740/94813759-60def600-0433-11eb-86e7-a1dea754e5b3.png">

#### Website 컨트롤러의 동작 이해
 - 컨트롤러는 Website 리소스 오브젝트를 감시해야 한다. 
   - ``` http://localhost:8001/apis/extensions.example.com/v1/ewbsites?watch=true```
 - 컨트롤러는 API 서버에 직접 연결하지 않지만 동일한 파드에서 사이드카 컨테이너로  실행돼 API 서버로의 앰배서더 역할을 하는 kubectl proxy 프로세스에 연결하여 통신한다.

<img width="659" alt="스크린샷 2020-10-01 오후 10 16 34" src="https://user-images.githubusercontent.com/6982740/94814058-c6cb7d80-0433-11eb-99ba-206b090c42b2.png">

 - API 서버는 Website 오브젝트의 모든 변경에 대한 감시 이벤트를 컨트롤러 파드로 보내게 된다.
 - 오브젝트 생성 -> ADDED 감시 이벤트 발송 -> 이벤트 핸들러는 Website 오브젝트에서 Website의 이름과 깃 리포지터리의 URL을 추출하고 JSON  매니페스트를 API서버에 게시해 디플로이먼트와 서비스 오브젝트를 생성한다.
 - 여기서 디플로이먼트 리소스는 2개의 컨테이너를 갖는 파드 템플릿으로 구성된다.
     - 하나는 nginx, 다른 하나는 gitsync 프로세스를 실행하여 로컬 디렉터리를 깃 리포지터리의 콘텐츠와 동기화한 상태를 유지시킨다.  
 - 서비스는 노드포트 서비스 타입으로 각 노드의 임시 포트로 웹서버 파드를 노출한다.

<img width="603" alt="스크린샷 2020-10-01 오후 10 20 21" src="https://user-images.githubusercontent.com/6982740/94814448-4eb18780-0434-11eb-9ed1-7082594b8f0e.png">

 - Website 리소스 인스턴스가 삭제되면 API 서버는 DELETED 감시 이벤트를 보내고, 컨트롤러는 이전에 작성한 디플로이먼트와 서비스 리소스를 삭제한다.

 - 이 예제에서는 빠졌지만 실제로 이런 컨트롤러를 구현한다면 API 서버를 통해 오브젝트 감시 이벤트를 받는것 뿐만 아니라 주기적으로 모든 오브젝트를 조회해야 한다.(감시 이벤트가 누락될 경우에도 시스템 이상이 없도록 하기 위해서)

#### 파드 형태로 컨트롤러 실행
 - 컨트롤러를 프로덕션 환경에 배포할 준비가 되었을 때 가장 좋은 방법은 다른 모든 코어 컨트롤러와 마찬가지로 쿠버네티스 안에서 컨트롤러를 실행하는 것이다.
 - 항상 컨트롤러가 실행됨을 보장하기 위해서 디플로이먼트 리소스로 컨트롤러를 배포할 수 있다.

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: website-controller
spec:
  replicas: 1       # 하나의 레플리카
  selector: 
    matchLabels:
      app: website-controller
  template:
    metadata:
      name: website-controller
      labels:
        app: website-controller
    spec:
      serviceAccountName: website-controller    # 특별한 서비스어카운트 아래에서 실행
      containers:
      - name: main
        image: luksa/website-controller
      - name: proxy
        image: luksa/kubectl-proxy:1.6.2
```

``` sh
# 서비스어카운트 생성
kubectl create serviceaccount website-controller

# 서비스 어카운트에 클러스터롤바인딩(cluster-admin 권한을 준다.)
kubectl create clusterrolebinding website-controller --clusterrole=cluster-admin --serviceaccount=default:website-controller

# website 생성
kubectl create -f kubia-website.yaml
```

#### 동작 중인 컨트롤러 살펴보기

``` sh
kubectl logs website-controller-6d5b8c49b7-rz25k -c main
```

<img width="1148" alt="스크린샷 2020-10-01 오후 10 32 24" src="https://user-images.githubusercontent.com/6982740/94815909-07c49180-0436-11eb-8b42-0aad35451eca.png">

``` sh
# 디플로이먼트, 서비스, 파드 조회
kubectl get deploy,svc,po
```

<img width="939" alt="스크린샷 2020-10-01 오후 10 33 14" src="https://user-images.githubusercontent.com/6982740/94815971-2034ac00-0436-11eb-859f-f3ce598d255a.png">

### 18.1.3 사용자 정의 오브젝트 유효성 검증
 - Website CustomResourceDefinition에서 어떤 종류의 유효성 스키마도 지정하지 않았는데, 이로 인해 사용자는 유효하지 않은 Website 오브젝트 생성을 요청할지도 모른다.
 - 생성시점에 유효성 검사를 추가하고 API 서버가 유효하지 않은 오브젝트를 승인하지 않도록 할 방법은 과거에는 없었으나 현재는 CustomResourceValidation기능이 추가되어 가능하다.

### 18.1.4 사용자 정의 오브젝트를 위한 사용자 정의 API 서버 제공
 - CustomResourceValidation 기능이 있기 때문에 CustomResourceDefinition을 사용하는것도 괜찮지만, 사용자 정의 오브젝트에 대한 지원을 추가하는 다른 방법이 있는데, 이는 사용자 정의 오브젝트를 위한 자체 API 서버를 구현하고 클라이언트와 직접 통신하도록 하는 것이다.

#### API 서버 애그리게이션
 - 쿠버네티스 버전 1.7부터 여러 애그리게이션된 API 서버를 단일 경로로 노출할 수 있다.
 - 클라이언트는 여러 API 서버가 뒤에서 서로 다른 오브젝트를 처리한다는 사실을 알지 못하고, 코어 쿠버네티스 API서버도 결국 여러 작은 API 서버로 분할되며, 에그리게이터를 통해 단일 서버로 노출된다.

<img width="541" alt="스크린샷 2020-10-01 오후 10 40 24" src="https://user-images.githubusercontent.com/6982740/94816680-1cedf000-0437-11eb-9fc6-684eee8b5788.png">

 - 이렇게 하면 사용자 정의 API 서버에 직접 구현하므로 더 이상 해당 오브젝트를 나타내기 위해 CRD를 만들지 않아도 된다.
 - 또한 자체 etcd 인스턴스를 실행하거나, 코어 API 서버에 CRD 인스턴스를 생성해 코어 API 서버의 etcd 저장소에 리소스를 저장할 수 있다.

#### 사용자 정의 API 서버 등록
 - 사용자 정의 API 서버를 클러스터에 추가하려면 파드로 배포하고 서비스를 통해 노출해야 한다.
 - 이를 위해 쿠버네티스는 APIService 리소르를 제공한다.

``` yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.extensions.example.com
spec:
  group: extensions.example.com
  version: v1alpha1
  groupPriorityMinimum: 10    # 그룹이 최소한 가져야하는 우선 순위
  versionPriority: 150        # 그룹 내에서이 API 버전의 순서를 제어
  service:
    name: website-api
    namespace: default
```
 - 위와 같이 정의할 경우 extensions.example.com API그룹과 버전 v1alpha1의 리소스를 포함하는 주 API 서버로 전달된 클라이언트 요청은 website-api 서비스로 노출된 사용자 정의 API서버 파드로 포워딩 된다.
 - APIService 관련된 자세한 설정을 알기 위해는 [APIService Doc](https://docs.openshift.com/container-platform/4.4/rest_api/extension_apis/apiservice-apiregistration-k8s-io-v1.html) 을 참조하기를 바란다.


#### 사용자 정의 클라이언트 생성
 - 사용자 정의 API 서버에 덧붙여 사용자 정의 CLI 도구를 빌드하는 방법도 제공한다.
 - kubectl이 시크릿, 디플로이먼트, 기타 리소스를 생성하는 방법과 유사하게 해당 오브젝트를 조작하기 위한 전용 명령어를 추가할 수 있다.


## 18.2 쿠버네티스 서비스 카탈로그를 통한 쿠버네티스 확장
 - 지금까지 살핀 쿠버네티스 모든 내용들은 기준으로 쿠버네티스 클러스터에서 애플리케이션을 실행하려면 이 모든 지식들을 다 알고 있어야 한다. 하지만 쿠버네티스는 사용하기 쉬운 셀프 서비스 시스템을 지향한다.
 - 사용자가 쿠버네티스에게 "PostgresSQL 데이터베이스가 필요해. 하나를 프로비저닝하고 어디에서 연결하는지 알려줘" 라고만 하면 관련된 리소스를 내부적으로 알아서 처리하고 connetion 정보을 바로 얻는 이상적인 방법이 필요하다. 이는 쿠버네티스 서비스 카탈로그를 통해 가능해질 것이다.

### 18.2.1 서비스 카탈로그 소개
 - 사용자는 카탈로그를 탐색할 수 있고, 서비스 실행에 필요한 파드, 서비스, 컨피그맵, 기타 리소스를 다루지 않고도 카탈로그에 나열된 서비스 인스턴스를 프로비저닝할 수 있다.

#### 서비스 카탈로그 4가지 API 리소스
##### ClusterServiceBroker 
 - 서비스를 프로비저닝할 수 있는 외부 시스템을 기술한다.

##### ClusterServiceClass
 - 프로비저닝할 수 있는 서비스 유형을 기술한다.

##### ServiceInstance 
 - 프로비저닝된 서비스의 한 인스턴스이다.

##### ServiceBinding
 - 클라이언트(파드) 세트와 ServiceInstance 간의 바인딩을 나타낸다.

#### 서비스 카탈로그 API 리소스간의 관계

<img width="667" alt="스크린샷 2020-10-01 오후 10 58 38" src="https://user-images.githubusercontent.com/6982740/94818949-a8688080-0439-11eb-9a01-0cc30395d74c.png">

 - 클러스터 관리자는 클러스터에서 서비스를 제공하고자 하는 각 서비스 브로커 ClusterServiceBroker 리소스를 생성한다.
 - 사용자는 서비스를 프로비저닝해야 하는 경우 ServiceInstance 리소스를 생성한 다음 ServiceBinding을 사용해 해당 ServiceInstance를 파드에 바인딩한다.
 - 해당 파드에는 프로비저닝된 ServiceInstance에 연결하는 데 필요한 모든 자격증명과 기타 데이터가 포함된 시크릿이 주입된다.

<img width="594" alt="스크린샷 2020-10-01 오후 11 00 35" src="https://user-images.githubusercontent.com/6982740/94819184-ecf41c00-0439-11eb-80cb-426d6f9c2885.png">

 - 클라이언트 파드는 프로비저닝된 서비스를 사용한다.

### 18.2.2 서비스 카탈로그 API 서버 및 컨트롤러 매니저 소개

#### 서비스 카탈로그 구성 요소
 - 서비스 카탈로그 API 서버
 - etcd 스토리지
 - 모든 컨트롤러를 실행하는 컨트롤러 매니저

#### 대략적인 동작 방식
 - 서비스 카탈로그 관련 리소스를 API 서버에 게시해 생성하고, 이는 etcd 스토리지에 저장한다.
- 컨트롤러는 요청된 서비스를 스스로 프로비저닝하지 않고, 서비스 카탈로그 API에서 ServiceBroker 리소스를 생성해 등록된 외부 서비스 브로커에게 이를 위임한다.

### 18.2.3 Service Broker와 OpenServiceBroker API 소개
 - 모든 브로커는 OpenServiceBroker API를 구현해야 한다.

#### OpenServiceBroker API 소개
 - 서비스 목록 검색(GET /v2/catalog)
 - 서비스 인스턴스 프로비저닝(PUT /v2/service_instances/:id)
 - 서비스 인스턴스 업데이트(PATCH, /v2/service_instances/:id)
 - 서비스 인스턴스 바인딩(PUT, /v2/service_instances/:id/service_bindings/:binding_id)
 - 서비스 인스턴스 바인딩 해제(DELETE, /v2/service_instances/:id/service_bindings/:binding_id)
 - 서비스 인스턴스 디프로비저닝(DELETE, /v2/service_instances/:id)

> 자세한 내용은 [https://github.com/openservicebrokerapi/servicebroker](https://github.com/openservicebrokerapi/servicebroker) 을 참조하라.

#### 서비스 카탈로그에 브로커 등록
``` yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: Broker
metadata:
  name: database-broker   # 브로커 이름
spec:
  url: http://database-osbapi.myorganization.org   # 서비스 카탈로그가 브로커에 연결할 수 있는 위치 (OpenServiceBroker[OSB] API URL
```

 - 해당 예제는 다른 유형의 데이터베이스를 프로비저닝할 수 있는 가상의 브로커를 설명한다.
 - 서비스 목록을 검색한 후 각 서비스에 대한 ClusterServiceClass 리소스를 생성한다.

#### 클러스터에서 사용 가능한 서비스 조회
 - ```kubectl get clusterserviceclasses``` 을 사용해 클러스터에서 프로비저닝할 수 있는 모든 서비스 목록을 검색할 수 있다.
 - 예제에는 가상의 데이터베이스 브로커가 제공할 수 있는 서비스에 대한 ClusterServiceClasses가 표시된다.

<img width="597" alt="스크린샷 2020-10-01 오후 11 26 33" src="https://user-images.githubusercontent.com/6982740/94822388-8ec93800-043d-11eb-93fc-45d90b9fd65d.png">

<img width="608" alt="스크린샷 2020-10-01 오후 11 27 42" src="https://user-images.githubusercontent.com/6982740/94822511-b6b89b80-043d-11eb-8db0-82014c3d4e3f.png">

### 18.2.4 프로비저닝과 서비스 사용
#### ServiceInstance 프로비저닝

``` yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: Instance
metadata:
  name: my-postgres-db      # 인스턴스의 이름
spec:
  serviceClassName: postgres-database    # serviceClass와 원하는 플랜 지정
  planName: free
  parameters:
    init-db-args: --data-checksums      # 브로커에 전달되는 추가 파라메터 정의 
```

 - 위 정보를 기준으로 무엇을 해야 하는지 아는 것은 전적으로 브로커에게 달려 있다.
- 데이터베이스 브로커는 아마도 어딘가에 PostgreSQL 데이터베이스의 새 인스턴스를 기동할 것이다.


<img width="627" alt="스크린샷 2020-10-01 오후 11 30 00" src="https://user-images.githubusercontent.com/6982740/94822825-0d25da00-043e-11eb-8e35-78fe5f395e9b.png">

#### 서비스 인스턴스 바인딩
 - 파드에서 사용하기 위해서는 ServiceBinding 리소스를 생성해서 바인딩 해야 한다.

``` yaml
apiVersion: servicecatalog.k8s.io/v1alpha1
kind: Binding
metadata:
  name: my-postgres-db-binding
spec:
  instanceRef:
    name: my-postgres-db        # 이전에 생성한 인스턴스 참조
  secretName: postgres-secret   # 서비스에 접근하기 위해 시크릿에 저장된 자격증명을 사용한다.
```

 - ServiceInstnace를 파드의 어디에 바인딩하거나 하는 것은 없다.
 - 현재 서비스 카탈로그는 아직 ServiecInstnace의 자격증명을 사용해 파드에 주입할 수 없다. 현재는 PodPresets 이라는 쿠버네티스 알파 기능을 사용하여 할 수 있다.
    - 이는 spec.container 아래 필드를 미리 정의해놓고 기본값으로 사용할 수 있다.

#### 클라이언트 파드에서 새로 생성된 시크릿 허용
 - 시크릿의 이름을 직접 선택할 수 있으므로 서비스를 프로비저닝하거나 바인딩하기 전에 파드를 배포할 수 있다.
    - 이렇게 되면 시크릿이 존재할 때까지 파드는 시작되지 않는다.
 - 바인딩 인스턴스마다 새로운 자격증명 세트를 만들어서 하는 것이 좋다.

<img width="602" alt="스크린샷 2020-10-01 오후 11 33 32" src="https://user-images.githubusercontent.com/6982740/94823171-87eef500-043e-11eb-8f3c-750696628bf6.png">

### 18.2.5 바인딩 해제와 프로비저닝 해제
 - ServiceBinding이 더 이상 필요하지 않으면 이를
 삭제할수 있다.

``` sh
kubectl delete servicebinding my-postgres-db-binding
```

 - 서비스 카탈로그 컨트롤러는 시크릿을 삭제하고 브로커를 호출해 바인딩 해제 작업을 수행한다.
 - 데이터베ㅣ스 인스턴스가 더 이상 필요하지 않으면 ServiceInstnace 리소스도 함꼐 삭제해야 한다.

``` sh
kubectl delete serviceinstnace my-postgres-db
```

 - ServiceInstnace 자원을 삭제하면 서비스 카탈로그는 서비스 브로커에게 프로비저닝 해제 작업을 수행한다.

### 18.2.6 서비스 카탈로그의 이점
 - 서비스 공급자는 해당 클러스터에 브로커를 등록해 모든 쿠버네티스 클러스터에 해당 서비스를 노출할 수 있다.
 - 일반적으로 서비스 브로커는 쿠버네티스에서 서비스를 쉽게 프로비저닝하고 노출할수 있게 만들며 쿠버네티스를 애플리케이션 배포를 위한 더욱 뛰어난 플랫폼으로 만들 것이다.

## 18.3 쿠버네티스 기반 플랫폼
 - 쿠버네티스는 차세대 PaaS의 토대를 널리 인정받으며, 쿠버네티스를 기반으로 구축한 PaaS 시스템 중에는 Deis Workflow(Azure Kubernetes Service, AKS)와 레드햇의 오픈시프트가 있다. 

### 18.3.1 레드햇 오픈시프트 컨테이너 플랫폼
 - 레드햇 오픈시프트는 PaaS로 개발자 경험에 중점을 두며, 애플리케이션의 빠른 개발 뿐만 아니라 애플리케이션의 쉬운 배포, 스케일링, 장기적인 유지 관리를 가능하게 하는 것을 목표로 한다.
 - 오픈시프트측은 쿠버네티스 위에서 오픈시프트 버전 3을 만들기 위해 처음부터 다시 구축하기로 결정하기도 한다.
 - 쿠버네티스가 롤아웃과 애플리케이션 스케일링을 자동화한다면, 오픈시프트는 클러스터에 지속적 통합(Continuous Integration) 솔루션을 별도로 통합할 필요 없이 애플맄이션의 이미지 빌드와 배포를 자동화한다.

#### 오픈시프트에서 이용할 수 있는 추가 리소스 소개
 - 사용자 및 그룹
 - 프로젝트
 - 템플릿
 - BuildConfigs
 - DeploymentConfigs
 - ImageStreeams
 - Routes

##### 사용자, 그룹, 프로젝트의 이해
 - 멀티 테넌트 환경을 제공한다.
 - 오픈시프트는 강력한 사용자 관리 기능을 제공해 각 사용자가 무엇을 할 수 있고 무엇을 할 수 없는지 지정할 수 있다.(현재 쿠버네티스 표준인 RBAC(Role Based Access Control)의 이전 버전이기는 하다)

##### 애플리케이션 템플릿
 - 오픈시프트는 JSON, YAML 매니페스트를 통해 자원을 배치할 수 있는 것에서 더 나아가 이 매니페스트를 파라미터화할 수 있다.
 - 템플릿은 정의에 자리 표시자를 갖는 오브젝트의 목록이며 템플릿을 처리(process)해서 인스턴스화할 때 자리 표시자가 파라미터 값으로 대체된다.

<img width="665" alt="스크린샷 2020-10-01 오후 11 54 20" src="https://user-images.githubusercontent.com/6982740/94825709-6f340e80-0441-11eb-8d6b-c49dc34f607b.png">

 - 템플릿이 인스턴스화되기 전에 먼저 처리되어야 한다.
 - 오픈시프트는 사용자가 몇개의 인자를 지정해 복잡한 애플리케이션을 빠르게 실행할 수 있도록 사전 제작된 템플릿 목록을 제공한다.
 - 이를 통해 모든 구성 요소를 명령어 하나로 배포할 수 있다.
 
##### BuildConfig를 사용한 소스로 이미지 빌드하기
 - 오픈시프트가 수행하므로 컨테이너 이미지를 직접 만들 필요가 없고, 변경 내용이 소스의 깃 리포지터리에 커밋된 직후 컨테이너 이미지 빌드를 트리거하도록 구성된 BuildConfig라는 리소스를 생성하면 된다.
 - 오픈시프트는 깃 리포지터리에서 변경사항을 가져와서 빌드 프로세스를 시작한다. Source To Image라는 빌드 매커니즘은 깃 저장에서 어떤 유형의 애플리케이션이 있는지 감지하고 적절한 빌드 프로시저를 실행할 수 있다.( pom.xml -> mvn build)
 - 따라서 BuildConfig 오브젝트를 생성함으로써 개발자는 깃 리포지터리만 지정하면 컨테이너 이미지 빌드를 걱정할 필요가 없다.

##### DeploymentConfig로 새 빌드 이미지를 자동으로 배포
  - DeploymentConfig 오브젝트를 생성하고 ImageStream을 가리키면 이 기능이 활성화 된다.
 - ImageStream은 이미지의 스트림이고, 이미지가 만들어지면 ImageStream에 추가된다.
 - 이를 통해 DeploymentConfig는 새로 빌드된 이미지를 인식하고 필요한 행동을 취해 새 이미지의 롤아웃을 시작할 수 있다.

<img width="593" alt="스크린샷 2020-10-01 오후 11 59 11" src="https://user-images.githubusercontent.com/6982740/94826280-1ca72200-0442-11eb-9aa9-0533323764e7.png">

 - 쿠버네티스의 Deployment와 유사지만 DeploymentConfig은 사전에 배포된다는 차이가 있다.
 - DeploymentConfig에서는 ReplicaSet 대신 ReplicationController를 생성하고 몇 가지 추가 기능을 제공한다.

##### Route를 사용해 외부로 서비스 노출하기
 - Route는 인그레스와 비슷하지만 TLS termination과 트래픽 분할과 관련된 추가적인 구성을 할 수 있다.

#### 오픈시프트 시험해보기
 - Minikube에 해당하는 Minishift를 제공한다.
 - [http://manage.openshift.com/](http://manage.openshift.com/) 에서 무료 멀티 테넌트를 사용해볼 수 있다.

### 18.3.2 Deis Workflow
 - 현재는 Azure Kubernetes Service(AKS)로 통합되었다. 쿠버네티스 기반으로 구축된 PaaS를 제공한다.

#### Azure Kubernetes Service(AKS) 소개
 - Workflow를 실행하면 일련의 서비스와 레플리케이션컨트롤러가 만들어져 개발자에게 간단하고 친숙한 환경을 제공한다.
 - ```git push deis master```로 변경사항을 푸시해 애플리케이션의 새로운 버전 배포가 시작되면 Workflow가 나머지 작업을 처리한다.
 - 코어 쿠버네티스에서는 사용할 수 없는 소스에서 이미지로의(source to image) 매커니즘, 애플리케이션 롤아웃과 롤백, 에지 라우팅, 로그 집계, 메트릭 및 알람을 제공한다.

### 18.3.3 Helm
#### Helm으로 리소스 배포
 - Helm은 쿠버네티스 패키지 관리자이다.(리눅스의 yum, osx의 brew와 유사)

#### Helm의 구성요소
 - helm CLI 도구(클라이언트)
 - 서버 구성 요소인 Tiller(쿠버네티스 클러스터 내에서 파드로 실행됨)

#### Helm의 동작 원리
 - Helm 애플리케이션 패키지를 차트(Chart)라고 한다.
 - 구성 정보가 포함된 컨피그와 결합돼 있으며 차트로 병합돼 애플리케이션 인스턴스가(차트와 Config의 결합) 실행 중인 릴리스(Release)를 생성한다.
 - Tiller는 차트에 정의된 모든 쿠버네티스 리소스를 생성하는 구성 요소이다.
 - 차트를 직접 생성해 사용할 수 있다.
 - [http://github.com/kubernetes/charts](http://github.com/kubernetes/charts) 에 있는 차트를 사용하여 쉽게 애플리케이션을 구동할 수도 있다.
   - PostgreSQL, MySQL, MariaDB, Magento, Memcached, MongoDB, OpenVPN, PHPBB, RabbitMQ, Redis, WordPress 등

<img width="471" alt="스크린샷 2020-10-02 오전 12 08 20" src="https://user-images.githubusercontent.com/6982740/94827437-647a7900-0443-11eb-90cc-e68d3b2c119d.png">

 - 쿠버네티스 클러스터에 애플리케이션을 실행할 때 매니페스트를 작성하는것부터 시작하지 말고, 다른 사람이 이미 같은 문제를 겪었는지와 이를 위한 Helm 차트가 준비돼 있는지를 확인하라.
 - 특정 애플리케이션에 대한 Helm 차트를 준비해 Helm 차트를 깃허브 리포지터리에 추가했다면 전체 애플리케이션을 설치하는 데 명령어 한 줄이면 된다.

``` sh
helm install --name my-database stable/mysql
```

 - 필요한 구성 요소가 무엇인지와 애플리케이션을 올바르게 실행하도록 구성할 방법을 걱정할 필요가 없다.
 - 가장 흥미로운 차트 중 하나는 OpenVPN 차트이다. 이는 쿠버네티스 클러스터 내부에서 OpenVPN 서버를 실행하고 로컬 머신이 클러스터의 파드인 것처럼 VPN을 통해 파드 네트워크에 들어가고 서비스에 엑세스할 수 있도록 한다.
    - 애플리케이션을 개발하고 로컬에서 실행할 때 유용하다.


## Reference
 - [https://docs.openshift.com/container-platform/4.4/rest_api/extension_apis/apiservice-apiregistration-k8s-io-v1.html](https://docs.openshift.com/container-platform/4.4/rest_api/extension_apis/apiservice-apiregistration-k8s-io-v1.html)
 - [https://github.com/openservicebrokerapi/servicebroker](https://github.com/openservicebrokerapi/servicebroker)
 - [http://manage.openshift.com/](http://manage.openshift.com/)
 - [https://azure.microsoft.com/en-us/services/kubernetes-service/](https://azure.microsoft.com/en-us/services/kubernetes-service/)
 - [http://github.com/kubernetes/charts](http://github.com/kubernetes/charts)
 - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
 - [kubernetes.io](https://kubernetes.io/ko/docs/home/)