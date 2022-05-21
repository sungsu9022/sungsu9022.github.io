---
title: "[kubernetes-in-action] 4. 레플리케이션과 그 밖의 컨트롤러 : 관리되는 파드 배포"
author: sungsu park
date: 2020-08-07 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 4. 레플리케이션과 그 밖의 컨트롤러 : 관리되는 파드 배포
## 4.1 파드를 안정적으로 유지하기
 - 쿠버네티스를 사용하면 얻을 수 있는 주요 이점은 쿠버네티스에 컨테이너 목록을 제공하면 해당 컨테이너를 클러스터 어딘가에서 계속 실행되도록 할수 있는 것이다.
 - 파드가 노드에 스케줄링되면 노드의 Kubelet은 이 파드가 존재하는 한 컨테이너가 계속 실행되도록 하고, 컨테이너의 주 프로세스에 크래시가 발생하면 Kubelet이 컨테이너를 다시 시작시킨다.
 - 하지만 JVM 기준으로 OOM이 발생하거나 애플리케이션 무한 루프나 교착상태에 빠져서 응답을 주지 못하는 경우 앱을 재실행하려면 앱 내부의 기능에 의존해서는 안되고, 외부에서 앱의 상태를 체크해야 한다.

### 4.1.1. 라이브니스 프로브 (Liveness probe)
 - 쿠버네티스는 라이브니스 프로브를 통해 컨테이너가 살아 있는지 확인할 수 있다.
 - 파드의 스펙에 각 컨테이너의 라이브니스 프로브를 지정하면 된다.
 - 쿠버네티스는 주기적으로 프로브를 실행하고 프로브가 실패할 경우 컨테이너를 다시 시작한다.

#### 쿠버네티스의 프로브 매커니즘
##### 1) HTTP GET 프로브
 - 지정한 IP, 포트, 경로에 HTTP GET 요청을 수행
 - 프로브가 응답을 수신하고 응답코드가 오류가 아닌 경우 성공(2xx, 3xx)
 - 오류 응답코드이면 실패로 간주(4xx, 5xx)

##### 2) TCP 소켓 프로브
 - 컨테이너의 지정된 포트에 TCP 연결을 수행
 - 연결이 성공하면 성공, 실패하면 실패

##### 3) Exec 프로브
 - 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드 확인
 - 상태 코드가 0이면 성공, 다른코드는 모두 실패로 간주

### 4.1.2 HTTP 기반 라이브니스 프로브 생성

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:  # 라이브니스 프로브 추가
      httpGet:
        path: /            # HTTP 요청 경로
        port: 8080    # 프로브가 연결해야 하는 포트
```

### 4.1.3 동작중인 라이브니스 프로브 확인
``` sh
# 라이브니스 프로브 생성
kubectl create -f kubia-liveness-probe.yaml

# 라이브니스 프로브 확인
kubectl get po kubia-liveness

# 크래시된 컨테이너의 애플리케이션 로그 조회
kubectl logs {podName} --previous

# pod이 다시 시작되는 이유 확인
kubectl describe po kubia-liveness
```

#### Pod Last State Exit Code
 - 정상적으로 종료된 경우 0
 - 외부에 의해서 종료된 경우 128 + x의 값을 가진다.
    - x는 프로세스에 전송된 시그널 번호임. SIGKILL 번호인 9로 강제종료된다면 128+9 = 137이 된다.

### 4.1.4 라이브니스 프로브의 추가 속성 설정
#### kubectl describe을 통해 라이브니스 추가 정보를 확인할 수 있다.

```  yaml
Liveness : http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
```

 - delay(지연) - 컨테이너가 시작된 후 바로 프로브 시작된다.
 - timeout(제한시간) - 컨테이너가 제한시간 안에 응답이 와야 한다.
 - period(기간) - 기간마다 프로브를 수행한다
 - #failure(실패수) - n번 연속 실패시 컨테이너가 다시 시작한다.

#### 설정 정의
 - initialDelaySeconds 옵션을 정의하여 설정할 수 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15
```

#### 라이브니스 프로브 정의시 주의사항
 - 애플리케이션 시작시간을 고려해서 초기 지연을 설정해야 한다.

### 4.1.5 효과적인 라이브니스 프로브 생성
 - 운영환경이라면 반드시 라이브니스 프로브를 정의하는 것이 좋다.(L7 health check 같은 느낌으로)
 - 정의하지 않으면 쿠버네티스가 앱이 살아 있는지 알 수 있는 방법이 없음.(프로세스가 떠있더라도 실제 문제 상황이 있을 수 있기 때문)

#### 라이브니스 프로브가 확인해야 할 사항
 - URL에 요청하도록 프로브를 구성해 앱 내 실행중인 모든 주요 구성요소가 살아있는지 또는 응답이 없는지 확인하도록 구성할 수도 있다.
    - 특정 가벼운 API를 호출해본다던지 별도의 헬스체크 URL을 백엔드 엔드포인트로 구성한다던지
- 라이브니스 프로브는 앱 내부만 체크하고 외부 요인의 영향을 받지 않도록 구성해야 한다.
   - 예를 들면 DB 서버에 문제가 있는 경우 이 프로브가 실패하도록 구성하는것은 지양해야 한다. 그 이유는 근본적인 원인이 DB라면 앱 컨테이너를 재시작한다 하더라도 문제가 해결되지 않기 때문이다.

#### 프로브를 가볍게 유지하기
 - 기본적으로 프로브는 비교적 자주 실행되며 1초 내에 완료되어야 한다.
 - 너무 많은 일을 하는 프로브는 컨테이너 속도를 저하 시킬수 있다.
 - 컨테이너에서 JVM 과 같은 기동 절차에 상당한 연산 리소스가 필요한 경우 Exec 프로브보다는 HTTP GET 라이브니스 프로브가 적합하다.

#### 프로브에 재시도 루프를 구현하지 마라.
 - 프로브의 실패 임계값을 설정할수 있기도 하고, 실패 임계값이 1이더라도 쿠버네티스는 실패를 한번 했다고 간주하기 전에 여러번 재시도를 한다.
 - 따라서 자체적인 재시도 루프를 구현하지 말아야 한다.

#### 라이브니스 프로브 요약
 - 라이브니스 프로브에 대한 로직은 해당 워커 노드의 Kubelet에서 수행이 된다.
 - 만약 워커 노드 자체에 크래시가 발생한 경우 해당 노드의 중단된 모든 파드의 대체 파드를 생성해야 하는것은 컨트롤 플레인의 역할
 - 레플리케이션컨트롤러같은 리소스 외에 직접 생선한 Pod들은 워커 노드 자체가 고장나면 아무것도 할 수 없음.

## 4.2 레플리케이션 컨트롤러 소개
 - 레플리케이션 컨트롤러는 파드가 항상 실행되도록 보장하는 쿠버네티스 리소스이다.
 - 파드의 여러 복제본(레플리카)을 작성하고 관리하기 위한 리소스이다.

### 동작 원리
 - 노드1의 파드 A는 종료된 이후 레플리케이션 컨트롤러가 관리하지 않기 때문에 다시 생성되지 않는다.
 - 레플리케이션컨트롤러는 파드B가 사라진것을 인지하고 새로운 파드 인스턴스를 생성한다.

<img width="692" alt="스크린샷 2020-08-05 오후 11 34 46" src="https://user-images.githubusercontent.com/6982740/89425688-43afe300-d774-11ea-8bc6-071256ac3164.png">

### 4.2.1 레플리케이션컨트롤러의 동작
 - 실행중인 파드 목록을 지속적으로 모니터링하고, 특정 유형의 실제 파드 수가 의도하는 수와 일치하는지 항상 확인한다.

#### 의도하는 수의 복제본수가 변경되는 케이스
> 파드 유형이란 특정 레비을 셀렉터와 일치하는 파드 세트(실제로 유형이라는 개념은 특별히 없다.)
 - 누군가 같은 유형의 파드를 수동으로 만드는 경우
 - 누군가 기존 파드의 유형을 변경하는 경우
 - 누군가 의도하는 파드 수를 줄이는 경우

#### 컨트롤러 조정 루프
 - 레플리케이션컨트롤러의 역할은 정확한 수의 파드가 항상 레이블 셀렉터와 일치하는지 확인하는 것이다.

<img width="555" alt="스크린샷 2020-08-05 오후 11 39 06" src="https://user-images.githubusercontent.com/6982740/89426209-dea8bd00-d774-11ea-9f71-601232939e9c.png">

#### 레플리케이션컨트롤러의 3가지 요소
 - 레이블 셀렉터는 범위에 있는 파드를 결정
 - 레플리카수는 실행할 파드의 의도하는 수를 지정
 - 파드 템플릿은 새로운 파드 레플리카 만들때 사용(파드 정의)

#### 컨트롤러의 레이블 셀렉터 또는 파드 템플릿 변경의 영향 이해
 - 레이블 셀렉터와 파드 템플릿을 변경하더라도 기존 파드에는 영향을 미치지 않음.
 - 레플리케이션컨트롤러는 파드를 생성한 후에는 실제 컨텐츠(컨테이너 이미지, 환경변수 등)에 신경 쓰지 않음.
 - 그래서 변경 이후에 새 파드가 생성되는 시점에서만 영향을 미친다.

#### 레플리케이션컨트롤러 사용시 이점
 - 기존 파드가 사라지면 새 파드를 시작해 파드가 항상 실행되도록 보장할 수 있다.
 - 클러스터 노드에 장애가 발생하면 장애가 발생한 노드에서 실행중인 모든 파드에 관한 교체 복제본이 생성된다.
 - 수동 또는 자동으로 파드를 쉽게 수평 확장할수 있다.

### 4.2.2 레플리케이션컨트롤러 생성
``` yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia        # 레플리케이션컨트롤러 이름
spec:
  replicas: 3           # 의도한 파드 인스턴스 수
  selector:             # 관리하는 파드 셀렉터
    app: kubia
  template:              # 파드 템플릿
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```
 - 레플리케이션 spec.selector를 지정하지 않을 수도 있다.(Optional)
 - 셀렉터를 지정하지 않으면 템플릿의 레이블로 자동 설정된다.
 - 레플리케이션컨트롤러를 정의할 때 셀렉터를 지정하지 않는것이 좋다.(쿠버네티스가 자동으로 추출하도록 하는게 간결하고 더 단순하다)

#### 레플리케이션 컨트롤러 생성
``` sh
kubectl create -f kubia-rc.yaml
```

### 4.2.3 레플리케이션 컨트롤러 동작 확인
``` sh
kubectl get pods
```

#### 삭제된 파드에 관한 레플리케이션 컨트롤러의 반응
``` sh
# 삭제
kubectl delete pod kubia-2lqjr

# 삭제된 팟 확인
kubectl get pods

# 레플리케이션 정보 확인
kubectl get rc

# 레플리케이션컨트롤러 추가 정보 확인
kubectl describe rc kubia
```

#### 컨트롤러가 새로운 파드를 생성한 원인 정확히 이해하기
 - 레플레키에션컨트롤러가 삭제 액션에 반응한 것이 아니다.
 - 결과적인 상태(부족한 파드수)에 대응한 것을 알아야 한다.
 - 파드 삭제가 일어난 이후 컨트롤러의 실제 파드 수 확인하였고, 이에 대한 적절한 조치로 새로운 파드가 생성된 것

<img width="700" alt="스크린샷 2020-08-05 오후 11 53 24" src="https://user-images.githubusercontent.com/6982740/89428005-dc476280-d776-11ea-9724-d8e7e63ffdb4.png">

#### 노드 장애 대응
 - 쿠버네티스를 사용하지 않는 환경에서 노드에 장애가 발생하면 앱을 수동으로 다른 시스템에 마이그레이션해야 함.(매우 오랜 시간이 걸리고 큰 문제)
 - 레플리케이션컨트롤러는 노드의 다운을 감지하자마자 파드를 대체하기 위해 새 파드를 기동시킨다.

``` sh

# 노드의 eth0 을 다운시킨 후 노드 확인
# 이 경우 해당 노드는 NotReady 상태로 변경된다.
kubectl get node


# 파드 확인
# NotReady 상태의 노드에 있던 Pod는 Unknown 상태로 변경되고 삭제된다.(새로운 파드가 다른 노드에서 생성됨)
kubectl get pods
```

### 4.2.4 레플리케이션컨트롤러 범위 안팎으로 파드 이동하기
 - 레플리케이션컨트롤러는 레이블 셀렉터와 일치하는 파드만을 관리한다.
 - 파드의 레이블을 변경하면 범위에서 제거되거나 추가시킬 수 있다.
 - 파드는 metadata.ownerReferences 필드에서 레플리케이션컨트롤러 참조 정보를 확인할 수 있다.
 - 파드의 레이블을 변경하여 범위에서 제거하면 수동으로 만든 다른 파드처럼 변경된다.

#### 레플리케이션 컨트롤러가 관리하는 파드에 레이블 추가
- 관리하지 않는 레이블이라면 아무런 영향이 없다.

#### 관리되는 파드의 레이블 변경
 - 레플리케이션컨트롤러에서 관리하지 않는 레이블로 변경하게 되면 범위에서 제거된것으로 간주하고 레플리케이션컨트롤러는 새로운 파드를 생성한다.

#### 컨트롤러에서 파드를 제거하는 실제 사례
 - 특정 파드에서만 어떤 작업을 하려는 경우 레이블을 변경하여 범위에서 제거하면 작업이 수월해질 수 있다.
 - 오동작하는 파드가 하나 있을때 이를 범위 밖으로 빼내고(이때 레플리케이션컨트롤러에 의해 레플리카수는 유지될것이다.) 원하는 방식으로 파드를 디버그하거나 문제를 재연해볼 수 있다.

#### 레플리케이션컨트롤러의 레이블 셀렉터 변경
 - 컨트롤러의 레이블을 변경하면 모든 파드들이 범위를 벗어나게 되므로 새로운 N개의 파드를 생성하게 된다.

### 4.2.5 파드 템플릿 변경
 - 템플릿을 변경하는것은 쿠키 커터를 다른것으로 교체하는 것과 같다.
 - 나중에 잘라낼 쿠키에만 영향을 줄뿐 이미 잘라낸 쿠키에는 아무런 영향을 미치지 않는다.

#### 템플릿을 수정하는 방법
``` sh
kubectl edit rc kubia
```

 - 기본 텍스트 편집기에서 레플리케이션컨트롤러의 YMAL 정의가 열려서 이를 수정할 수 있다.
 - KUBE_EDITOR 환경변수를 설정해서 텍스트 편집기를 커스텀할 수 있다.

### 4.2.6 수평 파드 스케일링
#### 레플리케이션컨트롤러 스케일 업(확장) / 스케일 다운(축소)
``` sh
# 확대
kubectl scale rc kubia --replicas=10

# 조회
kubectl get rc

# 축소
kubectl scale rc kubia --replicas=3
```

 - 위와 같이 수정을 하게 되면 레플리케이션컨트롤러가 업데이트되고 즉시 파드 수가 10개로 확장되었다가 다시 3개로 축소된다.

#### 스케일링에 대한 선언적 접근 방법 이해
 - 쿠버네티스에게 무엇을 어떻게 하라고 말하는게 아니라 의도하는 상태로 변경하는 것뿐.
 - 쿠버네티스는 상태를 보고 상태에 맞게 조정한다.


### 4.2.7 레플리케이션컨트롤러 삭제
 - kubectl delete를 통해 컨트롤러를 삭제하면 파드도 같이 삭제된다.

``` sh
# 파드와 컨트롤러 모두 삭제
kubectl delete rc kubia

# 파드를 삭제하지 않고 레플리케이션 컨트롤러를 삭제하는 방법
kubectl delete rc kubia --cascade=false
```

## 4.3 레플리카셋
 - 레플리카셋은 레플리케이션컨트롤러와 유사한 리소스이고 레플리케이션컨트롤러를 대체하기 위한 나온 리소스이다.
 - 일반적으로는 레플리카셋을 직접 생성하지는 않고 상위 수준의 디플로이먼트 리소스를 생성하면 자동으로 생성된다.

### 4.3.1 레플리카셋과 레플리케이션컨트롤러 비교
 - 레플리카셋이 좀 더 풍부한 파드 셀렉터 표현식을 사용할 수 있다.
    - 특정 레이블이 없는 파드나 레이블의 값과 상관없이 특정 레이블의 키를 갖는 파드를 매칭시킬 수도 있다.
    - 또한 하나의 레플리카셋으로 두 파드 세트를 모두 매칭시켜 하나의 그룹으로 취급하는것도 가능하다.
 - 위 부분을 제외하고는 다르지 않음.

### 4.3.2 레플리카셋 정의하기
``` yaml
apiVersion: v1
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
#    matchLabels:
#      app: kubia
    matchExpressions:
      - key: app
        operator: In
        values:
         - kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```

#### API 버전 속성
 - API 그룹, API 버전으로 구분되며 "apps/v1bet2"인 경우 그룹은 apps, 버전은 v1beta2이다.
 - core API 그룹에 속할 경우에는 그룹을 명시하지 않아도 된다. (v1)

### 4.3.3. 레플리카셋 생성 및 검사
 - 레플리카셋 생성은 api v1으로만 해야한다.(버전업되면서 변경됨)

``` sh
# 레플리카셋 생성
kubectl create -f kubia-replicaset.yaml

# 레플리카셋 조회
kubectl get rs
```

### 4.3.4 레플리카셋의 더욱 표현적인 레이블 셀렉터
 - In은 레이블의 값이 지정된 값 중 하나와 일치해야 함.
 - NotIn은 레이블의 값이 지정된 값과 일치하지 않아야 함.
 - Exists는 파드의 지정된 키를 가진 레이블이 포함되어야 한다.(값은 관계없기에 value 필드를 지정하지 않는다.)
 - DoesNotExists는 파드에 지정된 키를 가진 레이블이 포함돼 있지 않아야 한다.(value 지정 X)

### 4.3.5 레플리카셋 삭제
``` sh
kubectl delete rs kubia
```

## 4.4. 데몬셋
 - 데몬셋을 사용하면 각 노드에서 정확히 한 개의 파드만 실행시킬 수 있다.
 - 레플리카셋은 노드는 관계없이 지정된 수만큼 파드를 실행하는데 데몬셋은 이런 수를 지정하는것이 없고 클러스터의 모든 노드에 노드당 하나의 파드만 실행시키는 리소스이다.
 - 시스템 수준의 작업을 수행하는 인프라 관련 파드가 있다고 하면 데몬셋이 가장 적합할것이다.
 - 다른 예는 kube-proxy 프로세스가 데몬셋의 예이다. 서비스를 작동시키기 위해 모든 노드에서 실행되어야 하기 때문이다.

### 4.4.1 데몬셋으로 모든 노드에 파드 실행하기
 - 모든 클러스 노드마다 파드를 하나만 실행하고자 할때 사용하면 되는 리소스이다.
 - 만약 하나의 노드가 다운되더라도 다른곳에서 파드를 생성하지 않고, 새로운 노드가 클러스터에 추가되면 즉시 새 파드 인스턴스를 해당 노드에 배포한다.

<img width="574" alt="스크린샷 2020-08-06 오전 1 23 08" src="https://user-images.githubusercontent.com/6982740/89438132-672e5a00-d783-11ea-8d67-1ed2a7e3025c.png">

### 4.4.2 데몬셋을 사용해 특정 노드에서만 파드 실행하기
 - 파드가 노드의 일부에서만 실행되도록 지정하지 않으면 데몬셋은 클러스터 모든 노드에 파드를 배포한다.
 - 하지만 파드 템플릿에서 node-Selector 속성을 지정하면 특정 노드에만 배포할 수 있다.

#### 데몬셋과 파드 스케줄링
 - 쿠버네티스를 이용하면 노드에 스케줄링 되지 않게 해서 파드가 노드에 배포되지 않도록 할수도 있는데 이는 스케줄링 기반으로 동작하는 처리방식이다.
 - 데몬셋이 관리하는 파드의 경우는 스케줄러와는 무관하기 때문에 스케줄링이 되지 않는 노드에서도 파드가 실행된다.

#### 데몬셋 적용 예제
 - 노드에 레레이블을 지정하여 노드2를 제외한곳에만 데몬셋 파드가 실행되도록 처리한 예제이다.

<img width="595" alt="스크린샷 2020-08-06 오전 1 27 12" src="https://user-images.githubusercontent.com/6982740/89438586-f8053580-d783-11ea-982d-442f5f209826.png">

#### 데몬셋 YAML 정의
 - apiVerison이 쿠버네티스 업데이트 따라 변경되어서 "apps/v1" 을 이용해야 함.

``` yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

#### 데몬셋 생성
``` sh
# 데몬셋 생성
kubectl create -f ssd-monitor-daemonset.yaml

# 데몬셋 조회
kubectl get ds
```

#### 노드 레이블 추가 및 삭제
``` sh
# ssd 레이블 노드에 추가
kubectl label node minikube disk=ssd

# 데몬셋 pod 조회(추가됨)
kubectl get po

# ssd 레이블 노드를 hdd로 변경
kubectl label node minikube disk=hdd --overwrite

# 데몬셋 pod 조회(제거됨)
kubectl get po
```

## 4.5 Job 리소스
 - 완료 가능한 단일 태스크를 수행하는 파드를 실행하기 위한 리소스로 Job이 있다.
 - 완료 가능한 단일 태스크에서는 프로세스가 종료된 후에 다시 시작되지 않는다.

### 4.5.1 Job리소스 특징
 - 다른 리소스와 유사하지만 잡은 파드의 컨테이너 내부에서 실행중인 프로세스가 성공적으로 완료되면 컨테이너를 다시 시작하지 않는 파드를 실행시킬 수 있다.
 - 작업이 재대로 완료되는 것이 중요한 임시 작업에 유용하다.
 - 이러한 잡 리소스에 정의하기에 좋은 예로는 데이터를 어딘가에 저장하고 있고, 이 데이터를 변환해서 어딘가로 전송하는 케이스를 들수 있다.
 - 잡에서 과관리하느 파드는 성공적으로 끝날 때까지 다시 스케줄링이 된다.


<img width="666" alt="스크린샷 2020-08-06 오전 1 37 47" src="https://user-images.githubusercontent.com/6982740/89439643-71e9ee80-d785-11ea-9aa3-2e2ba26e74d9.png">

### 4.5.2 잡 리소스 정의
``` yaml
apiVersion: batch/v1       # batchAPI그룹의 버전을 선택해야 한다.
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure      # 재시작 정책을 사용할 수 있음.(Always, OnFailure, Naver)
      containers:
      - name: main
        image: luksa/batch-job
```

### 4.5.3 파드를 실행한 잡 확인
``` sh
# Job 생성
kubectl create -f batch-job.yaml

# Job 확인
kubectl get jods

# Job pod 확인 (completed된 job도 표시됨)
kubectl get po

# 로그 확인
kubectl logs batch-job-9dhsc

# job 삭제(job을 삭제하면 pod도 삭제된다)
kubectl delete job batch-job
```

#### 완료된 파드를 삭제하지 않는 이유
 - 파드가 완료될 때 파드가 삭제되지 않는 이유는 해당 파드의 로그를 검사할 수 있도록 하기 위함이다.

### 4.5.4 잡에서 여러 파드 인스턴스 실행하기
 - 2개 이상의 파드 인스턴스를 생성해 병렬 또는 순차 처리를 구성할 수 있다.(completions, parallelism 속성 이용)

#### 순차적으로 잡 파드 실행하기
 - spec에 completions값 지정
 - 이렇게 처리하면 5개의 파드가 성공적으로 완료될 때까지 과정을  계속한다.
 - 중간에 실패하는 파드가 있다면 잡은 새 파드를 생성하여 실제 5개 이상의 파드가 생성될 수 있다.

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

#### 병렬로 잡 파드 실행하기
 - parallelism을 2로 설정하면 동시에 2개의 파드가 생성되어 병렬처리로 실행된다.

``` sh
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5
  parallelism: 2
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

#### 잡 스케일링
 - 잡이 실행되는 동안 parallelism 속성을 변경하면 동시에 처리되는 파드 수를 조절할 수 있다.
``` sh
kubectl scale job multi-completion-batch-job --replicas 3
```

### 4.5.5 잡 파드가 완료되는데 걸리는 시간 제한하기 및 재시도 횟수 설정
 - activeDeadlineSeconds 속성을 설정하면 파드의 실행 시간에 제한을 두고 시간을 넘어서면 잡을 실패한것으로 처리할수도 있다.
 - backoffLimit 필드를 지정해 실패한것으로 표시되기 전에 잡을 재시도할수 있는 횟수도 설정할 수 있다.(기본값 6)

## 4.6 크론잡(CronJon)
 - 잡을 주기적으로 또는 한번만 실행되도록 스케줄링하기
 - 많은 배치 잡이 미래의 특정 시간 또는 지정된 간격으로 반복 실행해야 한다. 쿠버네티스를 이를 지원하기 위한 크론잡 리소스 기능을 제공한다.

### 4.6.1 크론잡 정의
``` yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"     # 매일 매시간 0,15,30,45분에 실행되는 cronJob
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

#### 크론잡 생성 및 확인
``` sh
# 크론잡 생성
kubectl create -f cronjob.yaml

# 크론잡 확인
kubectl get cronjob

# 크론잡 삭제
kubectl delete cronjob batch-job-every-fifteen-minutes
```

### 4.6.2 스케줄된 잡의 실행 방법 이해
 - 예정된 시간을 너무 초과해서 시작되서는 안되는 엄격한 요구사항이 요구될떄도 있는데 이를 위한 옵션을 제공한다.
    - startingDeadlineSeconds 필드를 지정( 초단위)
 - 일반적인 상황에서 크론잡은 스케줄에 설정한 각 실행에 항상 하나의 잡만 생성하지만, 2개의 잡이 동시에 생성되거나 전혀 생성되지 않을수도 있다.
    - 이런 문제를 해결하기 위해 멱등성(한번 실행이 아니라 여러번 실행해도 동일한 결과가 나타나야함)가지도록 개발해야 한다.
    - 이전에 누락된 잡 실행이 있다면 다음번 작업에서 해당 작업을 같이 수행해주도록 개발하는것이 좋다.

## 4.7 요약
 - 컨테이너가 더 이상 정상적이지 않으면 즉시 쿠버네티스가 컨테이너를 다시 시작하도록 라이브니스 프로브를 지정할 수 있다.
 - 직접 생성한 파드는 실수로 삭제되거나 실행중인 노드에 장애가 발생하거나 노드에서 퇴출되면 다시 생성되지 않기 때문에 직접 생성해서 사용하면 안된다.
 - 레플리케이션컨트롤러는 의도하는 수의 파드 복제본을 항상 실행 상태로 유지해준다.
 - 파드를 수평으로 스케일링(확장)하려면 쉽게 레플리케이션컨트롤러에 의도하는 레플리카 수를 변경하는 것만으로도 가능하다.
 - 파드는 레플리케이션컨트롤러가 소유하지 않으며, 필요한 경우 레플리케이션컨트롤러간에 이동할 수 있다.
 - 템플릿을 변경해도 기존의 파드에는 영향을 미치지 않는다.
 - 레플리카셋과 디폴로이먼트로 교체해야 하며, 레플리카셋과 디폴로이먼트는 동일한 기능을 제공하면서 추가적인 강력한 기능을 제공한다.
 - 레플리카셋은 임의의 클러스턴 ㅗ드에 파드를 스케줄링하는 반면, 데몬셋은 모든 노드에 데몬셋이 정의한 파드의 인스턴스 하나만 실행되도록 한다.
 - 배치 작업을 수행하는 파드는 쿠버네티스의 잡 리소스로 생성해야 한다.
 - 특정 시점에 주기적으로 수행해야 하는 잡은 크론잡 리소스를 통해 생성할 수 있다.


## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
