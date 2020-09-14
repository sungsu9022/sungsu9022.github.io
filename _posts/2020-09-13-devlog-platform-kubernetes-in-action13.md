---
layout: post
title: "[kubernetes-in-action] 13. 클러스터 노드와 네트워크 보안"
subtitle: "[kubernetes-in-action] 13. 클러스터 노드와 네트워크 보안"
categories: devlog
tags: platform
---

# 13. 클러스터 노드와 네트워크 보안
> 파드가 노드의 리소스에 엑세스할 수 있게 하는 방법

## 13.1 파드에서 호스트 노드의 네임스페이스 사용
 - 각 파드는 고유한 PID 네임스페이스가 있기 때문에 고유한 프로세스 트리가 있으며 고유한 IPC 네임스페이스도 사용하므로 동일한 파드의 프로세스간 통신 매커니즘(IPC, Inter-Process Communication mechanism)으로 서로 통신할 수 있다.

### 13.1.1 파드 노드의 네트워크 네임스페이스 사용
 - 특정 파드(시스템 파드 같은 경우)는 가상 네트워크 어댑터 대신 노드의 실제 네트워크 어댑터를 사용해야 하는 경우가 있을 수 있다.
    - 이 경우 파드 스펙에서 hostNetwork 속성을 true로 설정하면 가능하다.(이러면 프로세스는 노드의 포트에 직접 바인드된다.)

<img width="726" alt="스크린샷 2020-09-13 오전 1 18 58" src="https://user-images.githubusercontent.com/6982740/92999836-1c141d80-f55f-11ea-8503-03b9b7cb922b.png">

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-network
spec:
  hostNetwork: true     # 호스트 노드 네트워크 네임스페이스 사용
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
```

``` sh
# 파드 생성
kubectl create -f pod-with-host-network.yaml

# 파드 내부 이더넷  조회
kubectl exec pod-with-host-network ifconfig
```

<img width="1071" alt="스크린샷 2020-09-13 오전 1 22 42" src="https://user-images.githubusercontent.com/6982740/92999971-1834cb00-f560-11ea-957e-429577236d54.png">

 - 호스트 네트워크 네임스페이스를 사용하는 파드 네트워크 인터페이스를 대략 확인해볼 수 있다.

### 13.1.2 호스트 네트워크 네임스페이스를 사용하지 않고 호스트 포트에 바인딩
 - 파드는 hostNetwork 옵션으로 노드의 기본 네임스페이스의 포트에 바인딩할 수 있지만 여전히 고유한 네트워크 네임스페이스를 갖는다.
 - spec.containers.ports 필드 안에 postPort 속성을 사용해서 할수도 있다.

### hostPort를 사용하는 파드와 NodePort 서비스로 노출된 파드를 혼동하면 안됨.
 - 파드가 hostPort를 사용하는 경우 노드포트에 대한 연결은 해당 노드에서 실행중인 파드로 직접 전달되고,
 - NodePort 서비스의 경우 노드포트의 연결은 임의의 파드로 전달된다.

<img width="812" alt="스크린샷 2020-09-13 오전 1 32 50" src="https://user-images.githubusercontent.com/6982740/93000109-0b64a700-f561-11ea-913e-d634dc0e32b8.png">

 - 파드가 특정 호스트 포트를 사용하는 경우 두 프로세스가 동일한 호스트 포트에 바인딩될 수 없으므로 파드 인스턴스 하나만 노드에 스케줄링될 수 있다는 점을 이해해야 한다.

<img width="725" alt="스크린샷 2020-09-13 오전 1 35 31" src="https://user-images.githubusercontent.com/6982740/93000157-6d251100-f561-11ea-8369-4b1c42dad650.png">

#### 노드 포트에 파드 포트 바인딩
 - 노드가 여러개인 경우 다른 노드의 해당 포트로 파드에 엑세스할 수는 없다.
 - hostPort 기능은 기본적으로 데몬셋을 사용해 모든 노드에 배포되는 시스템 서비스를 노출하는 데 사용된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-hostport
spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080   # 컨테이너에 파드 IP의 포트 8080으로 연결될 수 있다.
      hostPort: 9000         # 배포된 노드포트 9000에서도 연결될 수 있다.
      protocol: TCP
```

### 13.1.3 노드의 PID와 네임스페이스 사용
 - hostNetwork 옵션과 유사한 파드 스펙 속성으로 hostPID와 hostIPC가 있다.
 - 노드의 PDI와 IPC 네임스페이스를 사용해 컨테이너에서 실행중인 프로세스가 노드의 다른 프로세스를 보거나 IPC로 통신할 수 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-host-pid-and-ipc
spec:
  hostPID: true   # 파드가 호스트의 PID 네임스페이스를 사용하도록 한다.
  hostIPC: true   # 파드가 호스트의 IPC 네임스페이스를 사용하도록 한다.
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
```
 
 - 컨테이너의 프로세스를 조회하면 컨테이너에서 실행중인 프로세스뿐만 아니라 호스트 노드에서 실행중인 모든 프로세스가 조회된다.

``` sh
# 파드 생성
kubectl create -f pod-with-host-pid-and-ipc.yaml

# 파드에서 프로세스 확인
kubectl exec pod-with-host-pid-and-ipc ps aux
```
 
 - hostIPC 속성을 true로 하면 노드에서 실행중인 다른 모든 프로세스와 IPC로 통신할 수도 있다.

## 13.2 컨테이너의 보안 컨텍스트 구성
 - securityContext속성으로 다른 보안 관련 기능을 파드와 파드의 컨테이너에 구성할수 있다.

### 보안 컨텍스트에서 설정할 수 있는 사항
 - 컨테이너의 프로세스를 실행할 사용자 지정
 - 컨테이너가 루트로 실행되는것 방지하기
 - 컨테이너를 특권(privileged mode)에서 실행해 노드의 커널에 관한 모든 접근 권한을 갖도록 함.
 - 특권모드에서 컨테이너를 실행해 기능을 추가하거나 삭제해 세분화된 권한 구성하기
 - 컨테이너의 권한 확인을 강력하게 하기 위해 SELinux(Security Enhanced Linux)옵션 설정하기
 - 프로세스가 컨테이너의 파일시스템에 쓰기 방지하기

### 보안 컨텍스트를 지정하지 않고 파드 실행

``` sh
# 기본 보안 컨텍스트 옵션을 사용해 파드 실행
kubectl run pod-with-defaults --image alpine --restart Never -- /bin/sleep 999999

# 컨테이너가 어떤 사용자와 그룹 ID로 실행되고 있는지 확인
kubectl exec pod-with-defaults id

uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```
 
 - 컨테이너가 사용자 ID(uid) 0과 그룹 ID(gid) 0인 루트 사용자로 실행중이다.(여러 다른 그룹의 구성원이기도 하면서)
 - 보통 컨테이너 이미지에 지정된 사용자로 컨테이너가 실행되는데 Dockerfile에서 이것은 USER 지시문을 이용하고 생략시 루트 권한으로 실행된다.

### 13.2.1 컨테이너를 특정 사용자로 실행
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-user-guest
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:       
      runAsUser: 405  # 사용자 이름이 아닌 사용자 ID를 지정(ID 405는 게스트 사용자)
```

``` sh
# 파드 생성
kubectl create -f pod-as-user-guest.yaml

# id 확인
kubectl exec pod-as-user-guest id

uid=405(guest) gid=100(users)
```

### 13.2.2 컨테이너가 루트로 실행되는 것 방지
 - 대부분의 컨테이너는 호스트 스시틈과 분리되어있기는 하지만, 프로세스를 루트로 실행하는 것은 나쁜 관행이다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-run-as-non-root
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsNonRoot: true     # 이 컨테이너는 루트가 아닌 사용자로만 실행할 수 있다.
```

``` sh
# pod 상태 확인
kubectl get po pod-run-as-non-root

NAME                  READY   STATUS                       RESTARTS   AGE
pod-run-as-non-root   0/1     CreateContainerConfigError   0          14s
```

### 13.2.3 특권 모드에서 파드 실행
 - 때때로 파드는 일반 컨테이너에서 접근할 수 없는 보호된 시스템 장치나 커널의 다른 기능을 사용하는 것과 같이 그드이 실행중인 노드가 할 수 있는 모든 것을 해야 할 수도 있다.
 - 대표적으로는 kube-proxy 파드가 있다.(노드의 iptables 규칙을 수정해야 함)
 - 노드 커널의 모든 엑세스 권한을 얻기 위해 파드의 컨테이너는 특권 모드(privileged mode)로 실행된다.
 - securityContext 속성에서 privileged 속성을 true로 설정하고 실행하면 된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-privileged
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      privileged: true    # 이 컨테이너는 특권 모드에서 실행된다.
```

 - /dev 하위에는 장치와 통신하는 데 사용되는 특별한 파일들이 있는데 일반 파드와 특권모드의 파드에서 접근할수 있는 데이터의 양이 다르다.
     - 권한이 있는 컨테이너는 모든 호스트 노드의 장치를 볼수 있음

``` sh
# 특권모드 파드 생성
kubectl create -f pod-privileged.yaml

# 기본 파드 /dev 확인
kubectl exec -it pod-with-defaults ls /dev
```

### 13.2.4 컨테이너에 개별 커널 기능 추가
 - 수년동안 리눅스 커널 기능의 발달로 훨씬 세분화된 권한 시스템을 지원하게 되었다.
 - 보안 관점에서 훨씬 안전한 방법은 컨테이너를 만들때 실제로 필요한 커널 기능만 엑세스하도록 하는 것이다.
 - 컨테이너 권한을 미세 조정하고 공격자의 잠재적인 침입의 영향을 제한할 수 있다.

#### 시스템 시간을 변경하는 예제
 - 컨테이너는 일반적으로 시스템 시간(하드웨어 시계 시간)을 변경할 수 없음

```  sh
# 파드 시스템 시간 변경(불가능함)
kubectl exec -it pod-with-defaults -- date +%T -s "12:00:00"
```

<img width="1187" alt="스크린샷 2020-09-13 오후 5 28 23" src="https://user-images.githubusercontent.com/6982740/93013783-8e7c1080-f5e6-11ea-8f73-07c14681eee8.png">

#### 컨테이너 기능에 TIME 기능 추가
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-add-settime-capability
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      capabilities:
        add:
        - SYS_TIME    # SYS_TIME 기능을  추가
```

 - 리눅스 커널 기능은 일반적으로 CAP_ 접두어로 시작한다.(파드 스펙에서 지정한 경우 접두어를 생략해야 함)

``` sh
# 시간 변경 기능을 탑재한 컨테이너를 가진 파드 생성
kubectl create -f pod-add-settime-capability.yaml

# 컨테이너 시간 변경(변경 명령이 동작함)
kubectl exec -it pod-add-settime-capability -- date +%T -s "12:00:00"

# 시간 조회(명령은 동작하나 실제 시간이 변경되지는 않음)
kubectl exec -it pod-add-settime-capability -- date
```

<img width="1275" alt="스크린샷 2020-09-13 오후 5 32 05" src="https://user-images.githubusercontent.com/6982740/93013875-21b54600-f5e7-11ea-8f13-35a7ec70cdf3.png">

 - 모든 권한을 부여하는 특권모드(privileged:true)를 하는것보다 위와 같이 기능을 추가하여 컨테이너에 부여하는 것이 훨씬 좋은 방법이다.

### 13.2.5 컨테이너에서 기능 제거
 - 컨테이너에 커널 기능을 추가한것처럼 기능을 제거할수도 있따.
 - 파일의 소유권을 변경할 수 있는 CAP_CHOWN 권한은 기본적으로 포함되는 기능인데 이를 제거할수도 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-drop-chown-capability
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      capabilities:
        drop:
        - CHOWN     # 이 컨테이너는 파일 소유권을 변경할 수 없다.(기능 제거)
```

``` sh
# CHOWN 권한이 없는 파드생성
kubectl create -f pod-drop-chown-capability.yaml

#  CHOWN 권한이 없는 파드에서 소유권 변경 시도
kubectl exec pod-drop-chown-capability chown guest /tmp

#  CHOWN 권한이 있는 파드에서 소유권 변경 시도(정상동작)
kubectl exec pod-with-defaults chown guest /tmp
```

<img width="1213" alt="스크린샷 2020-09-13 오후 5 37 44" src="https://user-images.githubusercontent.com/6982740/93013981-f1ba7280-f5e7-11ea-96a5-b99b60bd039f.png">


### 13.2.6 프로세스가 컨테이너의 파일시스템에 쓰는 것 방지
 - 컨테이너의 파일시스템에 쓰지 못하게 하고 마운트된 볼륨에만 쓰도록 할 수 있다.
 - 만약 공격자가 파일시스템에 쓸수 있도록 숨겨진 취약점이 있는 애플리케이션을 실행한다고 했을때 공격자가 파일을 수정해 악성 코드를 삽입할 수도 있는데, 이런 공격 유형은 컨테이너가 파일 시스템을 쓰지 못하게 함으로써 방지할 수 있다.
 - securityContext.readOnlyRootFileSystem 속성을 true로 하면 된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readonly-filesystem
spec:
  containers:
  - name: main
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      readOnlyRootFilesystem: true      # 이 컨테이너는 파일시스템에 쓸수 없다.
    volumeMounts:
    - name: my-volume
      mountPath: /volume              # 마운트된 볼륨인 /volume에는 쓸 수 있음.
      readOnly: false
  volumes:
  - name: my-volume
    emptyDir:
```

``` sh
# 파일시스템에 파일을 쓸수 없는 파드 생성
kubectl create -f pod-with-readonly-filesystem.yaml

# 파일시스템에 파일을 쓸수 없는 파드에서 파일 생성(불가)
kubectl exec -it pod-with-readonly-filesystem touch /new-file

# 볼륨에 파일 생성(성공)
kubectl exec -it pod-with-readonly-filesystem touch /volume/new-file

# 볼륨 파일 확인
kubectl exec -it pod-with-readonly-filesystem -- ls -la /volume/new-file

# 파일시스템에 쓸수있는 파드에서 파일 생성(성공)
kubectl exec -it pod-with-defaults touch /new-file

# 파일시스템에 쓸수 있는 파드 파일 확인
kubectl exec -it pod-with-defaults -- ls -la /new-file
```

<img width="1262" alt="스크린샷 2020-09-13 오후 5 58 45" src="https://user-images.githubusercontent.com/6982740/93014299-fc2a3b80-f5ea-11ea-90a7-e17b6912bd2d.png">

 - 컨테이너의 파일시스템을 읽기 전용으로 만들 경우 애플리케이션이 쓰는 모든 디렉터리에 볼륨을 마운트해야 한다.
 - 프로덕션 환경에서 보안을 강화하려면 파드를 실행할 때 컨테이너의 readOnlyRootFilesystem 속성을 true로 설정하는 것이 좋다.

#### 파드 수준의 보안 컨텍스트 옵션 설정
 - 컨테이너 수준에서 설정하듯이 파드 수준에서도 보안 컨텍스트를 사용할수도 있다.

### 13.2.7 컨테이너가 다른 사용자로 실행될 때 볼륨 공유
 - 볼륨을 통해 파드의 컨테이너간 데이터 공유를 쉽게 할 수 있는데 이 케이스는 사실 두 컨테이너 모두 루트 권한으로 실행되어 볼륨의 모든 파일에 대한 전체 엑세스 권한을 부여받았기 떄문이다.
 - runAsUser 옵션을 통해 2개의 컨테이너를 두 명의 다른 사용자로 실행해야 하는 경우가 있다.(이 경우에 항상 서로의 파일을 읽고 쓸수 있는것은 아니다)
 - 쿠버네티스에서는 실행중인 모든 파드에 supplementalGroups 속성을 지정해 실행중인 사용자 ID에 상관없이 파일을 공유할 수 있다.
     -  fsGroup
     - supplementalGroups

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-shared-volume-fsgroup
spec:
  securityContext:
    fsGroup: 555                # fsGorup과 supplementalGroups는 파드 레벨의 보안 컨텍스트로 정의
    supplementalGroups: [666, 777]
  containers:
  - name: first
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 1111      # 첫번쨰 컨테이너의 사용자 ID : 1111
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  - name: second
    image: alpine
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 2222        # 두번째 컨테이너의 사용자 ID : 2222
    volumeMounts:
    - name: shared-volume     # 각 컨테이너는 같은 볼륨을 사용
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-volume
    emptyDir:
```

``` sh
# 파드 생성
kubectl create -f pod-with-shared-volume-fsgroup.yaml

# id 확인
kubectl exec -it pod-with-shared-volume-fsgroup -c first sh

# 파드 내부에서 볼륨 확인
 ls -l / | grep volume
```

<img width="1172" alt="스크린샷 2020-09-13 오후 6 08 20" src="https://user-images.githubusercontent.com/6982740/93014448-23cdd380-f5ec-11ea-9561-535198f554b5.png">
 
 - 파드 정의에 지정한대로 컨테이너가 사용자 ID 1111로 실행되며, 유효 그룹은 0(root)이지만 그룹 ID 555,666,777도 사용자와 연관이 돼 있다.
 - fsGroup을 55로 설정했기 때문에 volume은 555 그룹 소유가 된다.
 - 마운트된 볼륨 디렉터리에 파일을 작성하면 사용자 ID 1111과 그룹 ID 555가 소유하게 된다.

<img width="433" alt="스크린샷 2020-09-13 오후 6 13 13" src="https://user-images.githubusercontent.com/6982740/93014542-def66c80-f5ec-11ea-9249-7aa46741d7bb.png">

 - 볼륨이 아닌 파일시스템에 생성한다면 root 그룹이 소유하게 된다.(일반적인 사용자 유효 그룹ID가 0[root]이므로)

<img width="456" alt="스크린샷 2020-09-13 오후 6 15 21" src="https://user-images.githubusercontent.com/6982740/93014568-18c77300-f5ed-11ea-8112-9f9b661f5359.png">

 - fsGroup 보안 컨텍스트 속성은 프로세스가 볼륨에 파일을 생성할 때 사용됨.
 - supplementalGroups 속성은 사용자와 관련된 추가 그룹 ID 목록을 정의하는 데 사용된다.
    - 아마도 다른 파드같은곳에서 다른 fsGroup으로 해당 그룹ID들을 사용할수 있고, 이에 따른 권한 이용을 위한 목적으로 쓸수 있는듯?

## 13.3 파드의 보안 관련 기능 사용 제한
 - 쿠버네티스는 하나 이상의 PodSecurityPolicy 리소스를 생성해 보안 관련 기능의 사용을 제한할 수 있다.

### 13.3.1 PodSecurityPolicy 리소스
 - API 서버에서 실행되는 PodSecurityPolicy 어드미션 컨트롤 플러그인으로 수행된다.
 - 파드 서버 리소를 API 서버에 게시하면 PodSecurityPolicy 어드미션 컨트롤 플러그인은 구성된 PodSecurityPolicyes로 파드 정의의 유효성을 검사한다.


#### minikube에서 pod security policy plugin 활성화 방법
 - minikube를 기본 셋팅으로 사용하는 경우 pod security policy plugin이 비활성화되어 있어 테스트를 해보려면 이를 활성화 시켜야 한다.

``` sh
#  minikbue addon 목록 조회
minikube addons list 

# pod-security-policy 활성화
minikube addons enable pod-security-policy
```

#### PodSecurityPolicy가 할 수 있는 작업
 - 파드가 호스트으 IPC, PID또는 네트워크 네임스페이스를 사용할 수 있는지 여부
 - 파드가 바인딩할 수 있는 호스트 포트
 - 컨테이너가 실행할 수 있는 사용자 ID
 - 특권을 갖는 컨테이너가 있는 파드를 만들 수 있는지 여부
 - 어떤 커널 기능이 허용되는지, 어떤 기능이 기본으로 추가되거나 혹은 항상 삭제되는지 여부
 - 컨테이너가 사용할 수 있는 SELinux(Security Enhanced Linux) 레이블
 - 컨테이너가 쓰기 가능한 루트 파일 시스템을 사용할 수 있는지 여부
 - 컨테이너가 실행할 수 있는 파일시스템 그룹
 - 파드가 사용할 수 있는 볼륨 유형

#### PodSecurityPolicy 예제
``` yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false           # 호스트 IPC 사용여부
  hostPID: false           # 호스트 PID 사용여부
  hostNetwork: false       # 호스트 네트워크 네임스페이스 사용여부
  hostPorts:               # 호스트 포트 제한
  - min: 10000
    max: 11000
  - min: 13000
    max: 14000
  privileged: false               # 특권모드 실행여부
  readOnlyRootFilesystem: true    # 읽기전용 루트 파일시스템 적용여부
  runAsUser:                   # 컨테이너 사용자 룰(모든 사용자 가능)
    rule: RunAsAny               
  fsGroup:                       
    rule: RunAsAny      
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny        # 원하는 SELinux 그룹 사용
  volumes:                # 파드에 사용가능한 볼륨 유형 정의
  - '*'
```

``` sh
# PodSecurityPolicy 생성
kubectl create -f pod-security-policy.yaml

# PodSecurityPolicy 조회
kubectl get psp

# 특권모드 파드 생성(원래 불가해야하는데 생성됨;;)
kubectl create -f pod-privileged.yaml
```

### 13.3.2 runAsUser, fsGroup, supplementalGroups 정책
#### MustRunAs 규칙 사용
``` yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  hostIPC: false
  hostPID: false
  hostNetwork: false
  hostPorts:
  - min: 10000
    max: 11000
  - min: 13000
    max: 14000
  privileged: false
  readOnlyRootFilesystem: true
  runAsUser:
    rule: MustRunAs     # 하나의 특정 ID를 설정하는 경우 min&max를 같은값으로 설정한다.
    ranges:
    - min: 2
      max: 2
  fsGroup:
    rule: MustRunAs
    ranges:
    - min: 2
      max: 10
    - min: 20
      max: 30
  supplementalGroups:
    rule: MustRunAs 
    ranges:           # 여러 범위를 지정할 수 있다. (2~10 또는 20~30)
    - min: 2
      max: 10
    - min: 20
      max: 30
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
``` 

#### 정책 범위를 벗어나는 RunAsUser를 사용하는 파드 배포
``` sh
# psp 생성
kubectl create -f psp-must-run-as.yaml

# 파드 생성(원래는 psp에서 걸려야하지만 성공으로 처리됨)
kubectl create -f pod-as-user-guest.yaml
```

#### 범위를 벗어난 사용자 ID를 가진 컨테이너 이미지가 있는 파드 배포
 - psp를 사용해 컨테이너 이미지에 하드코딩된 사용자 ID를 재정의하는것도 가능하다.

``` sh
# 범위를 벗어난 사용자 ID를 가진 컨테이너 이미지가 있는 파드 배포
kubectl run run-as-5 --image luksa/kubia-run-as-user-5 --restart Never

# id 확인
kubectl exec run-as-5 -- id

# uid=2(bin) gid=2(bin) groups=2(bin)이 나와야함..
# uid=5(games) gid=60(games) groups=60(games) 
# psp가  재대로 먹지 않은듯..
```

#### RunAsUser 필드의 MustAsNotRoot 규칙 사용
 - MustAsNotRoot 규칙을 사용해서 사용자는 루트로 실행되는 컨테이너를 배포할 수 없다.

### 13.3.3 allowed, default, disallowed 기능 구성
 - 컨테이너의 기능(사용 또는 불가능)에 영향을 줄수 있다.

``` yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  allowedCapabilities:           # 컨테이너에 SYSMTIME 기능을 사용하도록 허용
  - SYS_TIME
  defaultAddCapabilities:     # 컨테이너에 CHOEN 기능을 자동으로 추가
  - CHOWN
  requiredDropCapabilities:  # 컨테이너에 SYS_ADMIN과 SYS_MODULE 기능을 삭제하도록 요구
  - SYS_ADMIN
  - SYS_MODULE

...

```

#### 컨테이너에 어떤 기능을 추가할지 지정
 - PodSecurityPolicy 어드미션 컨트롤 플러그인이 활성화된 경우에는 allowedCapabilities에 허용되지 않은 기능은 아예 추가가 불가능해진다.

#### 모든 컨테이너에 기능 추가
 - defaultAddCapabilities 필드 아래에 나열된 모든 기능은 배포되는 모든 파드 컨테이너에 추가된다.

#### 컨테이너에서 기능 제거
 - requiredDropCapabilities 이 필드에 나열된 기능은 모든 컨테이너에서 자도으로 삭제되도록 한다.

``` sh
# psp 추가
kubectl create -f pod-add-sysadmin-capability.yaml
```

### 13.3.4 파드가 사용할 수 있는 볼륨 유형 제한
 - 사용자가 파드에 추가할 수 있는 볼륨 유형을 정의한다.
 - 이 PSP 볼륨 정의에서 최소한 emptyDir,  컨피그맵, 시크릿, 다운워드API, 퍼시스턴볼륨클레임 볼륨 사용은 허용해야 한다.
 - 리소스가 여러개 있는 경우 파드는 모든 정책에 정의된 모든 볼륨 유형을 사용할 수 있다.

``` yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: default
spec:
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:      # 사용가능한 볼륨 리소스 타입 정의
  - emptyDir
  - configMap
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

### 13.3.5 각각의 사용자와 그룹에 다른 PodSecurityPolicies 할당
 - PodSecurityPolicy는 클러스터 수준의 리소스의 리소스이므로 특정 네임스페이스를 지정하거나 적용할 수 없다.
 - 보통 다른 사용자에게 다른 정책을 할당하는 것은 RBAC 매커니즘으로 수행되는데 클러스터롤 리소르르 만들고 이름으로 개별 정책을 지정해 필요한 만큼 많은 정책을 작성하고 개별 사용자 또는 그룹이 사용할 수 있도록 할 수 있다.
 - 이런 클러스터롤을 클러스터롤바인딩을 사용해 특정 사용자나 그룹에 바인당하면 PodSecurityPolicy 어드미션 컨트롤 플러그인이 파드 정의를 승인할지 여부를 결정해야 할 때 파드를 생성하는 사용자가 엑세스할 수 있는 정책만 고려하면 된다.

#### 특권을 가진 컨테이너를 배포할 수 있는 PodSecurityPolicy 만들기
``` yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged        # 정책 이름
spec:
  privileged: true         # 특권을 갖는 컨테이너를 실행할 수 있음.
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  volumes:
  - '*'
```

``` sh
#  psp 생성
kubectl create -f psp-privileged.yaml

# psp 조회
kubectl get psp
```

<img width="1199" alt="스크린샷 2020-09-13 오후 7 08 11" src="https://user-images.githubusercontent.com/6982740/93015444-8dea7680-f5f4-11ea-81a7-221789d41808.png">

 - PRIV 열을 통해 각 정책이 어떤것을 허용하는지 확인할 수 있다.(privileged 정책만 특권모드 컨테이너를 실행할 수 있다.)

#### RBAC를 사용해 다른 사용자에게 다른 psp 할당
``` sh
# 클러스터롤 생성(기본)
kubectl create clusterrole psp-default --verb=use --resource=podsecuritypolicy --resource-name=default

# 클러스터롤 생성(특권모드)
kubectl create clusterrole psp-privileged --verb=use --resource=podsecuritypolicy --resource-name=privileged

# 클러스터롤 바인딩(기본 - 인증된 유저그룹에 할당)
kubectl create clusterrolebinding psp-all-users --clusterrole=psp-default --group=system:authenitated

# 클러스터롤 바인딩(특권모드 - bob 유저에게만 할당)
kubectl create clusterrolebinding psp-bob --clusterrole=psp-privileged --user=bob
```

#### kubectl을 위해 추가 사용자 생성
``` sh
# alice 유저 추가
kubectl config set-credentials alice --username=alice --password=password

# bob 유저 추가
kubectl config set-credentials bob --username=bob --password=password
```

#### 다른 사용자로 파드 생성
 - SPS 어드미션 컨트롤러부터 뭔가 정상동작을 확인하지는 못함..

``` sh
# alice로 파드 생성(실패)
kubectl --user alice create -f pod-privileged.yaml

# bob으로 파드 생성(성공)
kubectl --user bob create -f pod-privileged.yaml
```

## 13.4 파드 네트워크 격리
 - 어떤 파드가 어떤 파드와 대화할 수 있는지를 제한해 파드간 네트워크를 보호할 수 있는 방법을 제공한다.
 - 구성 가능 여부는 컨테이너 네트워킹 플러그인에 따라 다를 수 있다. ( NetworkPolicy 리소스를 만들어 네트워크 격리를 구성)
 - NetworkPolicy는 해당 레이블 셀렉터와 일치하는 파드에 적용됨.
 - 엑세스할 수 있는 소스나 파드에서 엑세스할 수 있는 대상을 지정하는 방식
 - 인그레스(ingress)와 이그레스(egress) 규칙으로 구성
 - 파드 셀렉터와 일치하는 파드나 레이블이 네임스페이스 셀렉터와 일치하는 네임스페이스의 모든 파드, 또는 CIDR(Classless Inter-Domain Routing) 표기법(예 : 192.168.1.0/24) 을 사용해 지정된 네트워크 IP 대역과 일치하는 파드에 대해 적용된다.

### 13.4.1 네임스페이스에서 네트워크 격리 사용
 - default-deny NetworkPolicy를 생성하면 모든 클라이언트가 네임스페이스의 모든 파드에 연결할 수 없게 된다.
 - 특정 네임스페이스에서 이 NetworkPolicy를 만들면 아무도 해당 네임스페이스 파드에 연결할 수 없다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:   {}  # 빈 파드 셀렉터는 동일한 네임스페이스의 모든 파드와 매치 
```

### 13.4.2 네임스페이스의 일부 클라이언트 파드만 서버 파드에 연결하도록 허용
 - 네임스페이스 foo에서 실행되는 파드A와 이를 이용하는 다른 파드B가 있다고 할때, A,B 외의 다른 파드들도 같은 네임스페이스 있지만 A에 B만 연결되기를 희마하는 경우 NetworkPolicy을 이용해서 이를 처리할 수 있다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-netpolicy
spec:
  podSelector:
    matchLabels:
      app: database      # app=database 레이블을 가진 파드에 대한 엑세스 보호
  ingress:               # app=webserver 레이블이 있는 파드에서 들어오는 연결 허용
  - from:
    - podSelector: 
        matchLabels:
          app: webserver
    ports:
    - port: 5432         # 5432 포트에만 연결할 수 있다.
```

<img width="638" alt="스크린샷 2020-09-13 오후 8 42 59" src="https://user-images.githubusercontent.com/6982740/93017185-b9279280-f601-11ea-9a2c-d389a6e74454.png">

 - 일부 파드만 다른 파드에 엑세스할 수 있는 Network Policy

### 13.4.3 쿠버네티스 네임스페이스 간 네트워크 격리
 - 여러 테넨트(네임스페이스 레이블 구분자로 쓰기 위한 회사이름 대명사 같은 느낌)가 동일한 쿠버네티스 클러스터를 사용하는 예시
 - 네임스페이스는 해당 테넨트를 지정하는 레이블이 있다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoppingcart-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart        # app=shoping-card로 표시된 파드들에게 적용
  ingress:
  - from:
    - namespaceSelector:           # tenant=manning의 레비을이 지정된 네임스페이스에서 시행중인 파드만 이 서비스 파드에 엑세스할 수 있다.
        matchLabels:
          tenant: manning
    ports:
    - port: 80     # 포트는 80만 허용
```

 - 새로운 엑세스 권한을 부여하려는 경우 추가 NetworkPolicy 리소스를 만들거나 기존 NetworkPolicy에 인그레스 규칙을 추가할 수 있다.
 - 이러한 클러스터 구성에서는 일반적으로 자신이 속한 네임스페이스에 레이블을 직접 추가할 수 없도록 해야 한다. 만약 추가할 수 있다면 네임스페이스 셀렉터 기반 인그레스 규칙을 우회하여 접근을 가능하게 만들수 있기 때문이다.
 - 네임스페이스 셀렉터와 일치하는 네임스페이스의 파드만 특정 파드에 엑세스 하도록 하는 NetworkPolicy 동작 방식은 아래와 같다.

<img width="671" alt="스크린샷 2020-09-13 오후 8 46 35" src="https://user-images.githubusercontent.com/6982740/93017260-394df800-f602-11ea-9a75-1caf387726e9.png">


### 13.4.4 CIDR 표기법으로 격리
 - 192.168.1.1~.255 범위의 IP에서만 엑세스할 수 있게 하려면 다음과 같은 인그레스 규칙을 지정할 수 있다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ipblock-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.1.0/24     # 192.168.1.0/24 IP대역의 클라이언트 트래픽만을 허용한다.
```

### 13.4.5 파드의 아웃바운드 트래픽 제한
 - 이그레스 규칙으로 아웃바운드 트래픽을 제한할 수 있다.

``` yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-net-policy
spec:
  podSelector:
    matchLabels:
      app: webserver
  egress:       # 파드의 아웃바운드 트래픽을 제한한다.
  - to:
    - podSelector:     # 웹 서버 파드는 app=database 레이블이 있는 파드만 연결할 수 있다.(5432 포트를 통해)
        matchLabels:
          app: database
    ports:
    - port: 5432
```


## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)