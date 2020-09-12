---
layout: post
title: "[kubernetes-in-action] 12. 쿠버네티스 API 서버 보안"
subtitle: "[kubernetes-in-action] 12. 쿠버네티스 API 서버 보안"
categories: devlog
tags: platform
---

# 12. 쿠버네티스 API 서버 보안
## 12.1 인증 이해
 - API 서버는 하나 이상의 인증 플러그인으로 실제 사람 구성할 수 있다.
 - API 서버가 요청을 받으면 인증 플러그인 목록을 거치면서 요청이 전달되고, 각각의 인증 플러그인이 요청을 검사해서 보낸 사람이 누구인지를 밝혀내려 시도한다.
 - 요청에서 해당 정보를 처음으로 추출해낸 플러그인은 사용자 이름, 사용자 OD와 클라이언트가 속한 그룹을 API 서버 코어에 반환한다.

### 인증 플러그인이 클라이언트 아이덴티티를 얻는 방식
 - 클라이언트의 인증서
 - HTTP 헤더로 전달된 인증토큰
 - 기본 HTTP 인증
 - 기타

### 12.1.1 사용자와 그룹
 - 인증 플러그인은 인증된 사용자의 사용자 이름과 그룹을 반환한다.

#### 사용자
 - 클라이언트 유형
   - 실제 사람
   - 파드(파드 내부에서 실행되는 애플리케이션)

 - 이 두 가지 유형의 클라이언트는 모두 인증 플러그인을 사용해 인증된다.
 - 사용자는 싱글 사인 온(SSO, SSingle Sign On)과 같은 외부 시스템에 의해 관리돼야 함.(API 서버를 통해 사용자를 생성하거나 업데이트/삭제할수 없다.)
 - 파드는 서비스 어카운트(service account)라는 매커니즘을 사용(클러스터에 서비스 어카운트 리소스를 생성할 수 있다.)


#### 그룹
 - 휴먼 사용자와 서비스 어카운트는 하나 이상의 그룹에 속할 수 있다.
 - 개별 사용자에게 권한을 부여하지 않고 한 번에 여러 사용자에게 권한을 부여하는데 사용된다.

#### 인증플러그인에서 반환하는 내장 그룹들
> system:unauthneticated = 어떤 인증 플러그인에서도 클라이언트를 인증할 수 없는 요청
> system:authneticated = 성공적으로 인증된 사용자에게 자동으로 할당
> system:serviceaccounts = 시스템의 모든 서비스어카운트를 포함.
> system:serviceaccounts:<namespace> = 특정 네임스페이스의 모든 서비스어카운트를 포함.

### 12.1.2 서비스어카운트
 - 시크릿 볼륨으로 각 컨테이너의 파일시스템에 마운트된 ```/var/run/secrets/kubernetes.io/serviceaccount/token``` 파일
 - 모든 파드는 파드에서 실행중인 애플리케이션의 아이덴티티를 나타내는 서비스어카운트와 연계되어 있고, 위 토큰 파일은 서비스어카운트의 인증 토근을 가지고 있음.

#### 서비스어카운트의 사용자 이름 규칙
 - ```system:serviceaccount:<namespace>:<service account name>```

#### 서비스어카운트 리소스
 - 서비스 어카운트는 파드, 시크릿, 컨피그맵 등과 같은 리소스로 범위는 네임스페이이다.
 - 각 네임스페이스마다 default 서비스어카운트가 자동으로 생성됨.
 - 필요한 경우 서비스어카운트를 추가해서 관리할 수 있다.
 - 각 파드는 파드의 네임스페이스에 있는 하나의 서비스 어카운트와 연계된다.

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92995322-7a300900-f53d-11ea-88c7-4ce736c59444.png">

``` sh
# 서비스 어카운트 조회
kubectl get sa
```

#### 서비스어카운트가 인가와 어떻게 밀접하게 연계돼 있는지 이해하기
 - 파드 매니패스트에 서비스어카운트 이름을 지정해 파드에 서비스어카운트를 할당할 수 있다.
 - 명시적으로 할당하지 않으면 default를 사용한다.
 - 파드에 서로 다른 서비스어카운트를 할당하면 각 파드가 엑세스할 수 있는 리소스를 제어할 수 있다.
 - API 서버는 토큰을 사용해 요청을 보낸 클라이언트를 인증한 다음 관련 서비스어카운트가 요청된 작업을 수행할 수 있는지 여부를 결정한다.
    - 역할 기반 엑스세 제어(RBAC) 플러그인과 같은 인가 플러그인으로 처리가 가능.

### 12.1.3 서비스어카운트 생성
 - 서비스어카운트를 생성하는 이유는 클러스터 보안을 위해서다.
 - 클러스터의 메타데이터를 읽을 필요가 없는 파드는 클러스터에 배포된 리소스를 검색하거나 수정할 수 없는 제한된 계정으로 실행해야 한다.
 - 리소스의 메타 데이터를 검색해야 하는 파드는 해당 오브젝트의 메타데이터만 읽을 수 있는 서비스어카운트로 실행되어야 한다.
 - 모든 권한은 최소로 유지되어야 한다.

#### 생성
``` sh
# 서비스어카운트 생성
# kubectl create serviceaccount foo
kubectl create sa foo

# 서비스어카운트 정보 조회
kubectl describe sa foo

# 시크릿 조회
kubectl describe secret foo-token-8kjb4
```

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92995396-46a1ae80-f53e-11ea-86a8-228b6190d21f.png">

> image pull secrets : 서비스어카운트를 사용하는 파드에 이 필드의 값이 자동으로 추가된다.
> Mountable secrets : 마운트 가능한 시크릿이 강제화된 경우 이 서비스 어카운트를 사용하는 파드만 해당 시크릿을 마운트할 수 있다.
> Tokens : 인증 토큰, 첫 번째 토큰이 컨테이너에 마운트된다.( 내부적으로 JSON Web Tokens[JWT]를 사용한다.)

#### 서비스어카운트의 마운트 가능한 시크릿
 - 기본적으로 파드는 원하는 모든 시크릿을 마운트할 수 있으나, 파드가 서비스어카운트 마운트 가능한 시크릿 목록에 있는 시크릿만 마운트하도록 설정할 수 있다.

> kubernetes.io/enforce-mountable-secrets="true" 어노테이션이 달린 경우 이를 사용하는 모든 파드는 서비스어카운트의 마운트 가능한 시크릿만 마운트할 수 있다.

#### 서비스어카운트 이미지 풀 시크릿(image pull secret)
 - 프라이빗 이미지 리포지터리에서 컨테이너 이미지를 가져오는 데 필요한 자격증명에 대한 시크릿
 - 서비스어카운트를 사용해 모든 파드에 특정 이미지 풀 시크릿을 자동으로 추가할 수 있다.(각 파드에 개별적으로 정의할 필요 없음)

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
imagePullSecrets:
- name: my-dockerhub-secret
```

### 12.1.4 파드에 서비스 어카운트 할당
 - spec.serviceAccountName 필드에 서비스어카운트를 명시하여 할당할 수 있다.(파드 만들때만 유효하고 파드가 살아있는동안은 변경이 불가)

#### 사용자 정의 서비스어카운트를 사용하는 파드 생성
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo          # default 대신 다른 서비스 어카운트 사용
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

#### API 서버와 통신하기 위해 사용자 정의 서비스어카운트 토큰 사용

``` sh
# 파드 생성
kubectl create -f curl-custom-sa.yaml

# api 호출
kubectl exec -it curl-custom-sa -c main curl localhost:8001/api/v1/pods
```

## 12.2 역할 기반 엑세스 제어로 클러스터 보안
 - 1.8.0 버전에서 RBAC 인가 플러그인이 GA(General Availability)로 승격되어 기본적으로 활성화 되어 있다.
 - RBAC는 권한이 없는 사용자가 클러스터 상태를 보거나 수정하지 못하게 한다.
 - 쿠버네티스에서는 RBAC 외에도 속성 기반 엑세스 제어(ABAC) 플러그인, 웹훅(Web Hook) 플러그인, 사용자 정의 플러그인 구현 등 여러 인가 플러그인이 포함되어 있다.(표준은 RBAC)

### 12.2.1 RBAC 인가 플러그인
 - 액션을 요청하는 사용자가 액션을 수행할 수 있는지 점검하는 플러그인

#### 액션 이해하기
 - REST 클라이언트는 GET, POST< PUT, DELETE HTTP 요청을 통해 리소스를 조회하거나 생성,수정할 수 있다.

| HTTP 메서드 | 단일 리소스에 관한 동사     | 컬렉션에 관한 동사 |
|-------------|-----------------------------|--------------------|
| GET, HEAD   | get(a d watch for watching) | list(and watch)    |
| POST        | create                      | n/a                |
| PUT         | update                      | n/a                |
| PATCH       | patch                       | n/a                |
| DELETE      | delete                      | delete collection  |


 - RBAC 규칙은 특정 리소스 인스턴스에도 적용할 수 있다.(예 myservcie라는 서비스)
 - 리소스가 아닌 URL path에도 권한을 적용할 수 있다.(API 서버가 노출하는 모든 경로가 리소스를 매핑한 것은 아니기 때문)

#### RBAC 플러그인 이해 이해
 - 사용자 롤(user role)을 사용
 - 주체(사람, 서비스어카운트, 또는 서비스어카운트 그룹일수 있음)는 하나 이상의 롤과 연계돼 있으며 각 롤은 특정 리소스에 특정 동사를 수행할 수 있다.

### 12.2.2 RBAC 리소스
> 롤(Role), 클러스터롤(ClusterRole) : 리소스에 수행할 동사를 지정
> 롤바인딩(RoleBinding), 클러스터롤바인딩(ClusterRoleBinding)  : 특정 사용자 그룹 또는 서비스 어카운트에 바인딩

 - 롤과 롤바인딩은 네임스페이스에 지정된 리소스
 - 클러스터롤과 클러스터롤 바인딩은 네임스페이스를 지정하지 않는 클러스터 수준의 리소스
 - 롤은 권한을 부여하며, 롤바인딩은 롤을 주체에 바인딩한다.

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92995984-03960a00-f543-11ea-872b-cce1bdb2af4c.png">

#### 롤과 롤 바인딩, 클러스터롤과 클러스터바인딩 구조 

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92996031-4657e200-f543-11ea-86b5-8b6bd0eee7bb.png">


#### 네임스페이스와 파드를 통한 실습
``` sh
# foo 네임스페이스 생성
kubectl create ns foo

# foo - pod 생성
kubectl run test --image=luksa/kubectl-proxy -n foo
# kubectl run --generator=run-pod/v1 test --image=luksa/kubectl-proxy -n foo


# bar 네임스페이스 생성
kubectl create ns bar

# bar - pod 생성
# kubectl run test --image=luksa/kubectl-proxy -n bar
kubectl run --generator=run-pod/v1 test --image=luksa/kubectl-proxy -n bar

# 파드 sh 접속
kubectl exec -it test-bf9d475cc-6qxks -n foo sh

# 파드 서비스 목록 나열
# minikube version: v1.12.1을 사용하는 경우 즉각적으로 권한이 없기 때문에 조회가 불가능하다.
curl localhost:8001/api/v1/namespaces/foo/services
```

### 12.2.3 롤과 롤 바인딩 사용

####  서비스를 읽을수 있는 Role 정의
 ``` yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]                  # 서비스는 이름이 없는 core apiGroup의 리소스이므로 ""
  verbs: ["get", "list"]
  resources: ["services"]      # 리소스 지정시에는 복수형 사용해야 함.
```

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92996323-bbc4b200-f545-11ea-8e15-d6ba3c43e651.png">

 - 사용자가 다른 네임스페이스에 있는 서비스의 get과 list는 허용하지 않는다.

``` sh
# bar 네임스페이스에 서비스리더 롤 생성
kubectl create role service-reader --verb=get --verb=list --resource=services -n bar
```

#### 서비스어카운트에 롤 바인딩하기
 - 롤바인딩 리소스를 만들어 롤을 바인딩한다.
 - 서비스어카운트 대신 사용자에게 롤을 바인딩하려면 --user 인수를 사용해서 할수 있다.(그룹은 --group)
 - 롤바인딩은 항상 하나의 롤을 참조하지만, 여러 주체에 롤을 바인딩할 수 있다. 

``` sh
# foo 네임스페이스에 있는 "foo:default" 서비스어카운트에 "service-reader" role을 바인딩하는 롤바인딩 리소스 생성
kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
```

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92996400-4e655100-f546-11ea-8439-7302d9417a66.png">

#### 롤바인딩 후 service 조회
 - 롤바인딩을 완료한 이후에 API를 호출해보면 정상 응답을 받을 수 있다.

<img width="668" alt="스크린샷 2020-09-12 오후 9 18 09" src="https://user-images.githubusercontent.com/6982740/92996464-bcaa1380-f546-11ea-8b39-d8b0e450cf04.png">

#### 롤바인딩에서 다른 네임스페이스의 서비스어카운트 포함하기
 - foo 네임스페이스에 있는 롤바이딩을 편집해서 다른 네임스페이스에 있는 다른 파드의 서비스 어카운트를 추가한 후 테스트
 - 두 서비스어카운트는 foo 네임스페이스의 서비스를 get하거나 list하도록 허용한다.(롤바인딩에서 가능한 범위)

``` sh
kubectl edit rolebiding test -n foo
```

``` yaml
subjects:
- kind: ServiceAccount
  name: default
  namespace: bar
```

<img width="713" alt="스크린샷 2020-09-12 오후 11 49 21" src="https://user-images.githubusercontent.com/6982740/92998073-b6219900-f552-11ea-9845-9441e94655af.png">


### 12.2.4 클러스터롤과 클러스터롤바인딩 사용하기
 - 롤과 롤바인딩은 네임스페이스가 지정된 리소스로, 하나의 네임스페이스상에 상주하며 해당 네임스페이스의 리소스에 적용된다는 것을 의미하지만, 롤바인딩은 다른 네임스페이스의 서비스어카운트도 참조할 수 있다.

#### 클러스터롤과 클러스터롤바인딩이 필요한 이유
 - 하지만 일반 롤은 롤이 위치하고 있는 동일한 네임스페이스의 리소스에만 엑세스할 수 있기 때문에 다른 네임스페이스의 리소스에 누군가가 엑세스할 수 있게 하려면 해당 네임스페이스마다 롤과 롤바인딩을 만들어야 한다.(클러스터롤과 클러스터롤바인딩이 필요한 이유)
 - 또어떤 리소스는 전혀 네임스페이스를 지정하지 않는 리소스들이 존재하기 때문에 이들을 위한 롤이 필요할수 있다.
    - 노드, 퍼시스턴트볼륨, 네임스페이스 등)
 - 리소스를 나타내지 않는 일부 URL을 위한 처리
 - 위와 같은 엑세스 권한을 위해 클러스터롤이 필요하다.

#### 클러스터 수준 리소스에 엑세스 허용
``` sh
# persistentvolumes에 대한 클러스터롤 생성(퍼시스턴트볼륨은 클러스터 수준의 리소스이므로 네임스페이스가 없음)
kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes

# 클러스터롤 정보 조회
kubectl get clusterrole pv-reader -o yaml

# pod 컨테이너 sh 진입
kubectl exec -it test -n bar sh

# api 조회(데이터 조회 불가)
curl localhost:8001/api/v1/persistentvolumes

# 클러스터롤 롤바인딩
kubectl create rolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default -n foo

# 롤바인딩 정보 조회
kubectl get rolebindings pv-test -o yaml -n foo
```

<img width="621" alt="스크린샷 2020-09-13 오전 12 06 25" src="https://user-images.githubusercontent.com/6982740/92998493-fd108e00-f554-11ea-849f-004623e8a4b3.png">

 - 퍼시스턴트볼륨은 클러스터 수준 리소스이므로 클러스터롤을 참조하고 있는 롤바인딩은(네임스페이스)는 접근 권한을 부여하지 못한다.

``` sh
# 롤바인딩 삭제
kubectl delete rolebinding pv-test -n foo

# 클러스터롤 클러스터롤바인딩
kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default

# api 조회(데이터 조회 가능)
curl localhost:8001/api/v1/persistentvolumes
```

<img width="656" alt="스크린샷 2020-09-13 오전 12 11 17" src="https://user-images.githubusercontent.com/6982740/92998611-a8214780-f555-11ea-9a1b-63747f756eb1.png">

 - 클러스터 수준 리소스에 엑세스 권한을 부여하려면 클러스터롤바인딩과 클러스터롤을 사용해야 한다.

#### 리소스가 아닌 URL에 엑세스 허용하기
 - 리소스가 아닌 URL의 클러스터롤은 클러스터롤로 바인딩되어야 한다.(네임스페이스 기준의 리소스가 아니기 떄문)

``` sh
kubectl get clusterrolebinding system:discovery -o yaml
```

<img width="706" alt="스크린샷 2020-09-13 오전 12 15 33" src="https://user-images.githubusercontent.com/6982740/92998689-41505e00-f556-11ea-9850-84ce0f5d752d.png">

 - 인증된 사용자와 인증이 되지 않은 사용자(모든 사용자)를 이 클러스터롤에 바인딩한다.(system:authenticated system:unauthenticated 2개의 그룹)
 - 모든 사람이 클러스터롤에 나열된 URL에 엑세스 할수 있음을 의미힌다.

#### 특정 네임스페이스의 리소스에 엑세스 권한을 부여하기 위해 클러스터롤 사용하기
 - 대부분의 리소스를 가져오고 나열하고 볼수있는 권한인 view(모든 네임스페이스의 리소스에 접근할수 있는 롤)를 이용한 롤 테스트
 - 권한이 없던 파드의 서비스어카운트에 전체 리소스를 조회할수 있는 view 클러스터롤을 바인딩 추가 권한을 얻은것을 확인할 수 있다.

``` sh
# api 조회(데이터 조회 불가)
curl localhost:8001/api/v1/pods

# view 권한 부여 
kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default

# api 조회(데이터 조회 가능)
curl localhost:8001/api/v1/pods
```

<img width="718" alt="스크린샷 2020-09-13 오전 12 43 33" src="https://user-images.githubusercontent.com/6982740/92999233-34ce0480-f55a-11ea-8aab-564e2f2684a0.png">


 - 클러스터롤인 view을 rolebinding으로 바인딩할 경우에는 자신의 네임스페이스에 있는 파드 조회는 가능하지만 다른 네임스페이스에 대한것은 볼수 없게 된다.

<img width="735" alt="스크린샷 2020-09-13 오전 12 46 22" src="https://user-images.githubusercontent.com/6982740/92999272-8f676080-f55a-11ea-92ce-5839b4b1c3e3.png">

### 12.2.5 디폴트 클러스터롤과 클러스터롤바인딩 이해
 - 기본적으로 쿠버네티스가 제공하는 클러스터롤과 클러스터롤바인딩이 있는데 이에 대해서도 알고 있어야 하는 부분이 있다.

``` sh
# 클러스터롤 바인딩 조회
kubectl get clusterrolebindings

# 클러스터롤 조회
kubectl get clusterrole
```

 - 기본 룰중에 가장 중요한것은 view, edit, admin, cluster-admin 롤이다.

#### view 클러스터롤
 - 이를 사용해 리소스에 읽기 전용 엑세스 허용할 수 잇다.
 - 롤, 롤바인딩과 시크릿을 제외한 네임스페이스 내의 거의 모든 리소스를 읽을수 있는 권한

#### edit 클러스터롤
 - 이를 사용해 리소스에 변경을 허용할 수 있다.
 - 롤 또는 롤바인딩을 보거나 수정하는 것은 허용되지 않는다.(이걸 제한하는 이유는 권한 상을을 방지하기 위함)

#### admin 클러스터롤
 - 네임스페이스에 있는 리소스에 관한 완전한 제어 권한을 가진다.
 - 리소스쿼터와 네임스페이스 리소스 자체를 제외한 모든 리소스를 읽고 수정할 수 있다.
 - 네임스페이스에서 롤과 롤바인딩을 보고 수정할 수 있다.

#### cluster-admin 클러스터롤
 - 쿠버네티스 클러스터를 완전하게 제어할 수 이쓴 롤

#### 그 밖의 디폴트 클러스터롤
 - system: 으로 시작하는 클러스터롤들은 쿠버네티스 구성 요소에서 사용된다.
 - 스케줄러에서 사용하는 system:kube-scheduler와 kubelet에서 사용되는 system:node 등이 있음.

### 12.2.6 인가 권한을 현명하게 부여하기
 - 기본적으로 파드는 클러스터의 상태를 볼 수 없는 것이 쿠버네티스 기본 정책이다. 적절한 권한을 부여하는 것은 사용자의 몫이다.
 - 모든 서비스어카운트에 cluster-admin 클러스터롤을 부여하는 것은 잘못된 생각이다.
 - 모든 사람에게 자신의 일을 하는 데 꼭 필요한 권한만 최소한으로 주고, 한 가지 이상의 권한을 주지 않는 것이 좋다.(최소 권한의 원칙)

#### 각 파드에 특정 서비스어카운트 생성
 - 각 파드를 위한 특정 서비스 어카운트를 생성한 다음 롤바인딩으로 맞춤형 롤과 연계하는 것이 바람직한 접근 방법이다.

#### 애플리케이션이 탈취될 가능성을 염두에 두기
 - 원치 않는 사람이 결국 서비스어카운트의 인증 토큰을 손에 넣을 수 있다는 가능성을 예상해야 하며, 실제로 피해를 입지 않도록 항상 서비스어카운트에 제한을 둬야 한다.


## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)