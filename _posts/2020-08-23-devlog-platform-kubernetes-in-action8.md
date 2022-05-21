---
title: "[kubernetes-in-action] 8. 애플리케이션에서 파드 메타데이터와 그 외의 리소스에 엑세스하기"
author: sungsu park
date: 2020-08-23 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 8. 애플리케이션에서 파드 메타데이터와 그 외의 리소스에 엑세스하기
> Downward API사용방법과 쿠버네티스 REST API 사용방법, 인증과 서버 검증을 kubectl proxy에 맡기는 방법, 컨테이너 내에서 API 서버에 접근하는 방법, 앰배서더 컨테이너 패턴의 이해, 쿠버네티스 클라이언트 라이브러리 사용방법 등을 살펴본다.
 - 특정 파드와 컨테이너 메타데이터를 컨테이너로 전달하는 방법과 컨테이너 내에서 실행중인 애플리케이션이 쿠버네티스 API 서버와 통신해 클러스터에 배포된 리소스의 정보를 얻는 것이 얼마나 쉬운지, 이런 리소스를 생하거나 수정하는 방법을 살펴보자.

## 8.1 Downward API로 메타데이터 전달
 - 파드의 IP, 호스트 노드 이름, 파드 자체의 이름과 같이 실행 시점까지 알려지지 않은 데이터는 어떻게 얻어와야할까?
 - 이러한 정보들을 여러 곳에서 반복해서 설정하는건 말이 안된다.
 - 위 2가지 문제는 쿠버네티스의 Downward API를 사용하면 해결할 수 있다.
 - Downward API는 애플리케이션이 호출해서 데이터를 가져오는 REST 엔드포인트와는 다르다.
 - 파드 매니페스트에 정의한 메타 데이터를 기준으로 volume을 정의하고 이를 환경변수에 할당하여 사용할 수 있다.

<img width="539" alt="스크린샷 2020-08-23 오후 9 27 39" src="https://user-images.githubusercontent.com/6982740/90978257-7c7af500-e587-11ea-8fc2-481d19ab510d.png">

### 8.1.1 사용 가능한 메타데이터 이해
 - Downward API를 사용하면 파드 자체의 메타데이터를 해당 파드 내에서 실행중인 프로세스에 노출시킬 수 있다.
 - 이러한 데이터는 OS로 직접 얻을수도 있겠지만, Downward API는 더 간단한 대안을 제공한다.

#### 메타데이터 종류
 - 파드의 이름
 - 파드의 IP 주소
 - 파드가 속한 네임스페이스
 - 파드가 실행중인 노드의 이름
 - 파드가 실행중인 서비스 어카운트 이름 ( 일단 파드가 API 서버와 통신할 때 인증하는 계정 정도로 이해하면 된다.)
 - 각 컨테이너의 CPU와 메모리 요청
 - 각 컨테이너의 CPU와 메모리 제한
 - 파드의 레이블
 - 파드의 어노테이션

### 8.1.2 환경변수로 메타데이터 노출하기
 - 환경변수로 파드와 컨테이너의 메타데이터를 컨테이너에 전달하는 방법

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 20Mi                       # 메모리 사이즈 4Mi로는 파드가 안뜨고 20으로 올려줘야 정상동작( https://github.com/kubernetes/minikube/issues/6160 )
    env:
    - name: POD_NAME                       # 특정 값을 설정하는 대신 파드 매니페스트의 metadata.name을 참조
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:                     # 컨테이너의 CPU/메모리 요청과 제한은 fieldRef 대신 resourceFieldRef를 사용해 참조
          resource: requests.cpu
          divisor: 1m                         # 리소스 필드의 경우 필요한 단위의 값을 얻으려면 제수(divisor)을 정의한다.
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

 - 제수는 어떤 수를 나누는 수라는 뜻으로 위에서 CPU 메모리 요청의 용량 단위

<img width="644" alt="스크린샷 2020-08-23 오후 9 33 13" src="https://user-images.githubusercontent.com/6982740/90978351-42f6b980-e588-11ea-897b-44e69cfb7b58.png">

``` sh
# downward API를 사용하는 pod 생성
kubectl create -f downward-api-env.yaml

# pod의 env 조회
kubectl exec downward env

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=downward
POD_NAMESPACE=default
POD_IP=172.17.0.4
NODE_NAME=minikube
SERVICE_ACCOUNT=default
CONTAINER_CPU_REQUEST_MILLICORES=15
CONTAINER_MEMORY_LIMIT_KIBIBYTES=20480
POD_NAME=downward
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBIA_PORT=tcp://10.102.194.76:80
KUBIA_PORT_80_TCP_ADDR=10.102.194.76
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBIA_SERVICE_HOST=10.102.194.76
KUBIA_PORT_80_TCP=tcp://10.102.194.76:80
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBIA_SERVICE_PORT=80
KUBIA_PORT_80_TCP_PORT=80
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBIA_PORT_80_TCP_PROTO=tcp
HOME=/root
```

### 8.1.3 downwardAPI 볼륨에 파일로 메타데이터 전달
 - 환경변수 대신 파일로 메타데이터를 노출하려는 경우 downwwardAPI 볼륨을 정의해서 컨테이너에 마운트할 수 있다.
 - downward라는 볼륨을 정의하고 컨테이너의 /etc/downward 아래에 마운트하는 예제

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward
  labels:                          # 이 레이블과 어노테이션은 downwardAPI 볼륨으로 노출된다.
    foo: bar
  annotations:
    key1: value1
    key2: |
      multi
      line
      value
spec:
  containers:
  - name: main
    image: busybox
    command: ["sleep", "9999999"]
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 100m
        memory: 20Mi
    volumeMounts:                   # downward 볼륨 /etc/downward에 마운트
    - name: downward
      mountPath: /etc/downward
  volumes:
  - name: downward                  # downwardAPI 볼륨 정의
    downwardAPI:
      items:
      - path: "podName"             #  metadata.name에  정의한 이름은 podName 파일에 기록된다.
        fieldRef:
          fieldPath: metadata.name
      - path: "podNamespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main
          resource: requests.cpu
          divisor: 1m
      - path: "containerMemoryLimitBytes"
        resourceFieldRef:
          containerName: main
          resource: limits.memory
          divisor: 1
```

<img width="584" alt="스크린샷 2020-08-23 오후 9 46 32" src="https://user-images.githubusercontent.com/6982740/90978616-1e034600-e58a-11ea-8456-cae5e1d73bc7.png">


``` sh
# pod 생성
kubectl create -f downward-api-volume.yaml

# pod 데이터 확인
kubectl exec downward -- ls -al /etc/downward/
kubectl exec downward -- cat /etc/downward/labels
kubectl exec downward -- cat /etc/downward/annotations
```

#### 레이블과 어노테이션 업데이트
 - 파드가 실행되는 동안 레이블, 어노테이션 값을 업데이트될 수 있다.
 - downwardAPI 볼륨을 이용하는 경우에는 업데이트시에도 최신 데이터를 볼수 있다.
 - 환경변수를 사용하는 경우에는 나중에 업데이트할 수 없다.

#### 볼륨 스펙에서 컨테이너 수준의 메타데이터 참조
 - 리소스 제한 또는 요청(resourceFieldRef)과 같은 컨테이너 수준의 메타데이터를 노출하는 경우 리소스 필드를 참조하는 컨테이너의 이름을 필수로 지정해야 한다.(컨테이너가 하나인 파드에서도 필수 지정)
 - 볼륨이 컨테이너가 아니라 파드 수준에서 정의되었지만, 리소스 제한은 컨테이너 기준이기 때문
 - 환경변수를 사용하는 것보다 약간 더 복잡하지만 필요할 경우 한 컨테이너의 리소스 필드를 다른 컨테이너에 전달할 수 있는 장점이 있다.
 - 환경변수로는 컨테이너 자신의 리소스 제한과 요청만 전달할 수 있다.

``` yaml
spec:
  volumes:
  - name: downward
    downwardAPI:
      items:
      - path: "containerCpuRequestMilliCores"
        resourceFieldRef:
          containerName: main           # 컨테이너 이름이 필수로 지정되어야 한다.
          resource: requests.cpu
          divisor: 1m
```

#### Downward API 사용 시기 이해
 - Downward API를 사용하면 애플리케이션은 쿠버네티스에 독립적으로 유지할 수 있게 한다.(기존에 환경변수의 특정 데이터를 활용하고 있는 경우 유용할 수도 있다.)
 - Downward API로 가져올 수 없는 다른 데이터들이 필요한 경우 쿠버네티스 API를 통해 가져와야 한다.


## 8.2 쿠버네티스 API 서버와 통신하기
 - Downward API는 단지 파드 자체의 메타데이터와 모든 파드의 데이터 중 일부만 노출한다.
 - 애플리케이션에서 클러스터에 정의된 다른 파드나 리소스에 대한 정보가 필요한 경우도 있는데 이 경우에는 쿠버네티스 API를 이용해야 한다.

### 8.2.1 쿠버네티스 REST API 살펴보기
``` sh
# 쿠버네티스 클러스터 정보 조회
kubectl cluster-info

Kubernetes master is running at https://192.168.64.2:8443
KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# 호출 ( Forbidden 403 에러)
curl https://192.168.64.2:8443 -k
```

#### kubectl proxy로 API 서버 엑세스하기
``` sh
# proxy 실행
kubectl proxy

# local proxy 호출
curl http://localhost:8001


{
  "paths": [
    "/api",
    "/api/v1",          # 대부분의 리소스 타입을 여기서 확인
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```

#### 배치 api 그룹의 REST 엔드포인트 살펴보기
``` sh
# apis/batch 엔드포인트 조회
curl http://localhost:8001/apis/batch

{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "batch",
  "versions": [                         # 제공되는 groupVersion 종류(batch API그룹은 2가지 버전을 갖는다는것을 의미함)
    {
      "groupVersion": "batch/v1",
      "version": "v1"
    },
    {
      "groupVersion": "batch/v1beta1",
      "version": "v1beta1"
    }
  ],
  "preferredVersion": {                  # 클라이언트는 preferredVersion을 사용하는것을 권장한다는 의미
    "groupVersion": "batch/v1",
    "version": "v1"
  }
}
```

 ``` sh
# batch/v1 리소스 유형
curl http://localhost:8001/apis/batch/v1

{
  "kind": "APIResourceList",         # batch/v1 API 그룹 내의 API 리소스 목록
  "apiVersion": "v1",
  "groupVersion": "batch/v1",
  "resources": [                     # 이 그룹의 모든 리소스 유형을 담는 배열
    {
      "name": "jobs",
      "singularName": "",
      "namespaced": true,            # 네임스페이스에 속하는 리소스라는 의미 ( persistentvolumes같은 것들은 false)
      "kind": "Job",
      "verbs": [                     # 이 리소스와 함꼐 사용할 수 있는 제공되는 API(단일, 여러개를 한꺼번에 추가 삭제할수 있고, 검색, 감시 업데이트 할수 있음)
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "mudhfqk/qZY="
    },
    {
      "name": "jobs/status",       # 리소스의 상태를 수정하기 위한 특수한 REST 엔드포인트
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

#### 클러스터 안에 있는 모든 잡 인스턴스 나열하기
 - items 하위에 나열된다.

``` sh
# job 생성
kubectl create -f my-job.yaml

# 모든 잡 인스턴스 조회
curl http://localhost:8001/apis/batch/v1/jobs
```

#### 이름별로 특정 잡 인스턴스 검색
``` sh
# api를 통한 job 조회
curl http://localhost:8001/apis/batch/v1/namespaces/default/jobs/my-job

# job 정보 조회
kubectl get job my-job -o json

# 위 2가지 결과는 같다.
```

### 8.2.2 파드 내에서 API 서버와 통신
 - 파드 내에서 통신하려면  API 서버의 위치를 찾아야하고, 서버로 인증을 해야 한다.

#### API 서버와의 통신을 시도하기 위해 파드 실행
``` sh
kubectl create -f curl.yaml

# 파드의 shell 접근
kubectl exec -it curl bash

```

#### API 서버 주소 찾기
 - 실제 애플리케이션에서는 서버 인증서 확인을 절대로 건너뛰면 안된다. 중간자 공격(man-in-the-middle attack)으로 인증 토큰을 공격자에게 노출할수 있기 때문
     - 중간자 공격(man-in-the-middle attack)은 통신을 연결하는 두 사람 사이에 중간자가 침입해 두 사람은 상대방에 연결했다고 생각하지만 실제로는 두 사람은 중간자에게 연결돼 있으며, 중간자가 한쪽에서 전달된 정보를 도청 및 조작한 후 다른쪽으로 전달하는 방식

``` sh
# kubernetes 서비스
kubectl get svc

# KUBERNETES_SERVICE_HOST, KUBERNETES_SERVICE_PORT 변수를 통해 얻을수 있다.
env | grep KUBERNETES_SERVICE

KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443

# FQDN을 이용한 방법(403)
curl https://kubernetes -k
```

#### 서버의 아이덴티티 검증
 - 각 컨테이너의 /var/run/secrets/kubernetes.io/serviceaccount/에 마운트되는 자동 생성된 default-token-xyz 라는 이름의 시크릿을 기준으로 처리할수 있다.

``` sh
# 컨테이너 내부에서 조회
ls /var/run/secrets/kubernetes.io/serviceaccount/

ca.crt	namespace  token

# --cacert 옵션을 통해 인증서 지정 ( 여전히 403)
curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes

# 환경변수 지정
export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# 재호출
curl https://kubernetes
```

#### API 서버로 인증
 - Authorization HTTP 헤더 내부에 토큰을 전달하여 토큰을 인증된 것으로 인식하여 적절한 응답을 받을 수 있다.
 - 이런 방식을 통해 네임스페이스 내에 있는 모든 파드를 조회할 수 있다. 그러나 먼저 curl 파드가 어떤 네임스페이서에서 실행중인지 알아야 한다.

``` sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# 호출  ( 왜안되느지 모르겠지만 안됨 ㅠ)
curl -H "Authorization: Bearer $TOKEN" https://kubernetes
```

#### 역할 기반 엑세스 제어(RBAC) 비활성화
 - RBAC가 활성화된 쿠버네티스 클러스터를 사용하는 경우 서비스 어카운트가 API 서버에 엑세스할 권한이 없을 수 있다.

``` sh
# 모든 서비스 어카운트에 클러스터 관리자 권한이 부여됨
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --group=system:serviceaccounts

# 위험하고 프로덕션 클러스터에서는 해서는 안됨.
```

#### 파드가 실행중인 네임스페이스 얻기
 - 시크릿 볼륨 디렉터리에 있는 3개의 파일을 사용해 파드와 동일한 네임스페이스에서 실행중인 모든 파드를 나열할 수 있다.
 - GET 대신 PUT이나 PATCH를 전송해 업데이트도 가능하다.

``` sh
# namespace 지정 ( 실제로 default인데 아무값도 없다..)
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

# 호출
curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v/namespaces/$NS/pods
```

#### 파드가 쿠버네티스와 통신하는 방법 정리
 - 애플리케이션 API 서버의 인증서가 인증기관으로부터 서명됐는지를 검증해야 하고, 인증 기관의 인증서는 ca.cart 파일에 있다.
 - 애플리케이션은 token 파일의 내용을 Authorization HTTP 헤더에 Bearer 토큰으로 넣어 전송해서 자신을 인증해야 한다.
 - namespace 파일은 파드의 네임스페이스 안에 있는 API 오브젝트의 CRUD 작업을 수행할 때 네임스페이스를 API 서버로 전달하는데 사용해야 한다.

<img width="575" alt="스크린샷 2020-08-23 오후 11 02 49" src="https://user-images.githubusercontent.com/6982740/90980181-cae2c080-e594-11ea-9c01-59946ba4ece6.png">

### 8.2.3 앰배서더 컨테이너를 이용한 API 서버 통신 간소화
 - 보안을 유지하면서 통신을 훨씬 간단하게 만들수 있다.(kubectl proxy 활용)

#### 앰배서더 컨테이너 패턴 소개
 - 메인 컨테이너 옆의 앰배서더 컨테이너에서 kubectl proxy를 실행하고 이를 통해 APi 서버와 통신할 수 있다.
 - 메인 컨테이너의 애플리케이션은 HTTPS 대신 HTTP로 앰배서더에 연결하고 앰배서더 프록시가 APi 서버에 대한 HTtPS 연결을 처리하도록해 보안을 투명하게 관리할 수 있다.
 - 파드의 모든 컨테이너는 동일한 루프백 네트워크 인터페이스를 공유하므로 애플리케이션은 localhost의 포트로 프록세이 엑세스할 수 있다.

<img width="474" alt="스크린샷 2020-08-23 오후 11 03 46" src="https://user-images.githubusercontent.com/6982740/90980192-e8178f00-e594-11ea-9768-db60048e3bc9.png">

#### 추가적인 앰배서더 컨테이너를 사용한 curl 파드 실행
 - kubectl proxy는 8001에 바인딩되며 curl localhost:8001에 접속할 수 있다.
 - 외부 서비스에 연결하는 복잡성을 숨기고 메인 컨테이너에서 실행되는 애플리케이션을 단순화하기 위해 앰배서더 컨테이너를 사용하는 좋은 예시이다.
 - 단점은 추가 프로세스를 실행해야 해서 리소스가 추가로 소비된다는 것이다.


``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-with-ambassador
spec:
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador                             # kubectl-proxy 이미지를 실행하는 앰배서더 컨테이너
    image: luksa/kubectl-proxy:1.6.2
```

``` sh
# pod 생성
kubectl create -f curl-with-ambassador.yaml


# shell 접속
kubectl exec -it curl-with-ambassador -c main bash

# 호출
curl localhost:8001
```

<img width="577" alt="스크린샷 2020-08-23 오후 11 09 33" src="https://user-images.githubusercontent.com/6982740/90980360-b9e67f00-e595-11ea-81e0-4118cbe6ad08.png">


### 8.2.4 클라이언트 라이브러리를 사용해 API 서버와 통신
 - 단순한 API 요청 이상을 수행하려면 쿠버네티스 API 클라이언트 라이브러리 중 하나를 사용하는 것이 좋다.
 - [https://kubernetes.io/ko/docs/reference/using-api/client-libraries/](https://kubernetes.io/ko/docs/reference/using-api/client-libraries/) ( 다양한 언어를 지원함.)
 - 현재 SIG(Special Interest Group)에서 지원하는 API는 Go, Python, Java ,.net, JavaScript, Haskell이 있다.
 - 이 라이브러리를 사용하는 경우 기본적으로 HTTPS를 지원하고, 인증을 관리하므로 앰배서더 컨테이너를 사용할 필요가 없다.

#### Java 예제 ( https://github.com/kubernetes-client/java/ )
 - 책에 있는 Fabric8 java 클라이언트가 아니라 SIG에서 지원하는 Java 클라이언트 라이브러리를 첨부

``` java
// list all pods
public class Example {
    public static void main(String[] args) throws IOException, ApiException{
        ApiClient client = Config.defaultClient();
        Configuration.setDefaultApiClient(client);

        CoreV1Api api = new CoreV1Api();
       // 라이브러리 메소드 설계는 좀 이상하게 해놓은듯, 모두 null이면 arguments 정의를 안했어야지..)
        V1PodList list = api.listPodForAllNamespaces(null, null, null, null, null, null, null, null, null);
        for (V1Pod item : list.getItems()) {
            System.out.println(item.getMetadata().getName());
        }
    }
}

// watch on namespace object:
public class WatchExample {
    public static void main(String[] args) throws IOException, ApiException{
        ApiClient client = Config.defaultClient();
        Configuration.setDefaultApiClient(client);

        CoreV1Api api = new CoreV1Api();

        Watch<V1Namespace> watch = Watch.createWatch(
                client,
                api.listNamespaceCall(null, null, null, null, null, 5, null, null, Boolean.TRUE, null, null),
                new TypeToken<Watch.Response<V1Namespace>>(){}.getType());

        for (Watch.Response<V1Namespace> item : watch) {
            System.out.printf("%s : %s%n", item.type, item.object.getMetadata().getName());
        }
    }
}
```

#### 스웨거와 Open API를 사용해 자신의 라이브러리 구축
 - 쿠버네티스 API 서버는 /swaggerapi 에서 스웨거 API 정의를 공개하고 /swagger.json에서 OepnAPI 스펙을 공개하고 있다.

#### 스웨거 UI로 API 살펴보기
 - 스웨거 UI로 REST API를 더 나은 방식으로 탐색할 수 있다.
 - API 서버를  ```--enable-swagger-ui=true```옵션으로 실행하면 활성화된다.

``` sh
# minikube swaggerUI = true 적용
minikube start --extra-config=apiserver.Features.EnableSwaggerUI=true


# kubectl proxy
kubectl proxy --port=8080 &

# 호출 ( swagger-ui 안됨..)
http://localhost:8080/swagger-ui
http://localhost:8080/swagger.json
http://192.168.64.2:8443/swagger-ui

```

## 8.3 요약
 - 파드의 이름, 네임스페이스 및 기타 메타데이터가 환경변수 또는 downward API 볼륨의 파일로 컨테이너 내부의 프로세스에 노출시키는 방법
 - CPU와 메모리의 요청 및 제한이 필요한 단위로 애플리케이션에 전달되는 방법
 - 파드에서 downward API 볼륨을 사용해 파드가 살아 있는 동안 변경될 수 있는 최신 메타데이터를 얻는 방법(레이블과 어노테이션 등)
 - kubectl proxy로 쿠버네티스 REST API를 탐색하는 방법
 - 쿠버네티스에 정의된 다른 서비스와 같은 방식으로 파드가 환경변수 또는 DNS로 API 서버의 위치를 찾는 방법
 - 파드에서 실행되는 애플리케이션이 API 서버와 통신하는지 검증하고, 자신을 인증하는 방법
 - 앰배서더 컨테이너를 사용해 애플리케이션 내에서 API 서버와 훨씬 간단하게 통신하는 방법
 - 클라이언트 라이브러리ㅏ로 쉽게 쿠버네티스와 상호작용할 수 있는 방법

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
