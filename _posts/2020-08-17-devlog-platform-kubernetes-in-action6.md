---
title: "[kubernetes-in-action] 6. 볼륨 : 컨테이너에 디스크 스토리지 연결"
author: sungsu park
date: 2020-08-17 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 6. 볼륨 : 컨테이너에 디스크 스토리지 연결
- 파드는 내부에 프로세스가 실행되고 CPU, RAM, 네트워크 인터페이스 등의 리소스를 공유한다. 하지만 디스크는 공유되지 않는다. 파드 내부의 각 컨테이너는 고유하게 분리된 파일 시스템을 가지기 때문이다.(컨테이너 이미지로부터 제공되는)
- 새로 시작한 컨테이너는 이전에 실행했던 컨테이너에 쓰여진 파일 시스템의 어떤 것도 볼수 없다.
- 전체 파일 시스템이 유지될 필요는 없지만 실제 데이터를 가진 디렉터리를 보존하고 싶을 수 있음.
- 이를 위해 쿠버네티스는 스토리지 볼륨으로 기능을 제공한다.
- 볼륨은 파드와 같은 최상위 리소스는 아니지만 파드의 일부분으로 정의되며 파드와 일반적으로는 동일한 라이프 사이클을 가진다.

## 6.1 볼륨 소개
 - 쿠버네티스 볼륨은 파드의 구성 요소로 컨테이너와 동일하게파드 스펙에서 정의된다.
 - 볼륨은 독립적인 쿠버네티스 오브젝트가 아니므로 자체적으로 생성, 삭제될 수 없다.
 - 접근하려는 컨테이너에서 각각 마운트 되어야 한다.

### 6.1.1 볼륨 예제
 - 볼륨 2개를 파드에 추가하고, 3개의 컨테이너 내부의 적절한 경로에 마운트
 - 리눅스에서 파일시스템을 파일 트리의 임의 경로에 마운트할 수 있는 방식을 이용
 - 같은 볼륨을 2개의 컨테이너에 마운트하면 컨테이너는 동일한 파일로 동작 가능하다.
 - 마운트되지 않은 볼륨이 같은 파드안에 있더라도 접근할수 없고, 접근하려면 volumeMount를 컨테이너 스펙에 정의해야 한다.

<img width="329" alt="스크린샷 2020-08-17 오후 5 41 21" src="https://user-images.githubusercontent.com/6982740/90376084-e2134100-e0b0-11ea-9e1f-4d2e231669ea.png">

### 6.1.2 사용 가능한 볼륨 유형 소개
 - emptyDir : 일시적인 데이터를 저장하는 데 사용되는 간단한 빈 디렉터리
 - hostPath : 워커 노드의 파일시스템을 파드의 디렉터리로 마운트
 - gitRepo : 깃 리포지터리의 콘텐츠를 체크아웃해 초기화한 볼륨
 - nft : NFS 공유를 파드에 마운트
 - gcePersistentDisk, awsElasticBlockStore, azureDisk 등 : 클라우드 제공자의 전용 스토리지 마운트
 - cinder, cephfs, iscsi, flocker, glusterfs, quobyte, rdb, flexVolume, vsphereVolume, photonPersistentDisk, scaleIO : 다른 유형의 네트워크 스토리지 마운트
 - configMap, secret, downwardAPI : 쿠버네티스 리소스나 클러스터 정보를 파드에 노출하는 데 사용되는 특별한 유형의 볼륨
 - persistentVolumeClaim : 사전 혹은 동적으로 프로비저닝된 퍼시스턴트 스토리지를 사용하는 방법

## 6.2 볼륨을 사용한 컨테이너간 데이터 공유
### 6.2.1 emptyDir 볼륨 사용
 - 빈 디렉터리로 시작되며, 볼륨의 라이프사이클이 파드에 묶여 있으므로 파드가 삭제되면 볼륨의 콘텐츠도 같이 사라진다.
 -  컨테이너에서 가용한 메모리에 넣기에 큰 데이터 세트의 정렬 작업을 수행하는 것과 같은 임시 데이터를 디스크에 쓰는 목적인 경우 사용할 수 있다.

#### 파드에 emptyDir 볼륨 사용(동일한 볼륨을 공유하는 컨테이너 2개가 있는 파드)
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator         # 첫번째 컨테이너 html-generator
    volumeMounts:                # html이라는 이름의 볼륨을 컨테이너 /var/htdocs에 마운트
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server             # 두번째 컨테이너 web-server
    volumeMounts:                # html이라는 이름의 볼륨을 컨테이너 /usr/share/nginx/html에 마운트
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:                       # html이라는 단일 emptyDir 볼륨을 위의 컨테이너 2개에 마운트하기 위한 정의
  - name: html
    emptyDir: {}
```

#### 실행중인 파드 보기

``` sh
# pod 조회
kubectl get po

# fortune pod 포트 포워딩
kubectl port-forward fortune 8080:80

# request
curl http://localhost:8080

# html-generator 컨테이너 내부 확인
kubectl exec -i -t fortune -c html-generator -- /bin/bash

# web-server 컨테이너 index.html 파일 확인
kubectl exec -i -t fortune -c web-server -- cat /usr/share/nginx/html/index.html
```

#### emptyDir을 사용하기 위한 매체 지정하기
 - 워커 노드의 실제 디스크에 생성하는 경우 노드 디스크가 어떤 유형인지에 따라 성능이 결정될수 있음.
 - 쿠버네티스에 emptyDir을 디스크가 아닌 메모리를 사용하는 tmpfs 파일시스템으로 생성하도록 요청할수도 있음.

``` yaml
  volumes:
  - name: html
    emptyDir:
        medium: Memory   # 이 emptyDir의 파일들은 메모리에 저장된다.
```

### 6.2.2 깃 리포지터리를 볼륨으로 사용하기
 - gitRepo 볼륨은 emptyDir base이고, 파드가 시작되면 깃 리포를 복제하여 데이터를 채운다.
 - 볼륨이 생성된 후에 참조하는 리포지터리와 동기화되지는 않는다. 파드가 삭제되고 새 파드가 생성되면 그 파드는 최신 커밋을 포함하게 된다.
 - 최신 변경사항을 동기화하고 싶은 경우 github web hook 같은 것을 이용하면 될듯

<img width="603" alt="스크린샷 2020-08-17 오후 6 18 08" src="https://user-images.githubusercontent.com/6982740/90379695-03c2f700-e0b6-11ea-9d25-44582f973fb1.png">

#### 복제된 깃 리포지터리 파일을 서비스하는 웹 서버 실행하기
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:               # gitRepo 정의하여 볼륨 정의 가능
      repository: https://github.com/luksa/kubia-website-example.git
      revision: master
      directory: .
```

#### gitRepo 볼륨에 대한 정리
 - gitRepo 볼륨은 emptyDir 볼륨과 유사하게 기본적으로 볼륨을 포함하는 파드를 위해 특별히 생성되고 독점적으로 사용되는 전용 디렉터리이다.
 - 파드가 삭제되면 볼륨과 콘텐츠는 모두 삭제된다.

## 6.3 워커 노드 파일시스템의 파일 접근
- 대부분의 파드는 호스트 노드를 인식하지 못하므로 노드의 파일 시스템에 있는 어떤 파일에도 접근하면 안 된다.
- 하지만 특정 시스템 레벨의 파드(데몬셋과 같은)는 이런 파일 시스템 접근이 필요할 수 있다.
- 쿠버네티스는 hostPath 볼륨으로 이 기능을 지원한다.

### 6.3.1 hostPath 볼륨 소개
- hostPath 볼륨은 노드 파일 시스템의 특정 파일이나 디렉터리를 가리킨다.(퍼시스턴트 스토리지)
- gitRepo나 emptyDir 볼륨의 콘텐츠는 파드가 종료되면 삭제되지만, hostPath 볼륨의 콘텐츠는 삭제되지 않는다.
- 이전 파드와 동일한 노드에서 새롭게 스케줄링 되는 새로운 파드는 이전 파드가 남긴 모든 항목을 볼 수 있다.
- 다만 데이터베이스의 데이터 디렉터리를 지정할 위치로 사용하기에는 적절하지 않다.(db pod은 다른 노드로 스케줄링 될 가능성이 있으므로)
- hostPath 볼륨은 파드가 어떤 녿에 스케줄되느냐에 따라 민감하기 때문에 일반적인 파드에서는 사용하지 않는것이 좋다.


<img width="503" alt="스크린샷 2020-08-17 오후 6 40 24" src="https://user-images.githubusercontent.com/6982740/90381984-20acf980-e0b9-11ea-8939-59fec001a61f.png">

### 6.3.2 hostPath 볼륨을 사용하는 시스템 파드 검사하기
- 노드의 로그 파일이나 kubeconfig(쿠버네티스 구성 파일), CA 인증서를 접근하기 위한 데이터들을 hostPath로 구성되어있음
- 노드의 시스템 파일에 읽기/쓰기를 하는 경우에만 hostPath 볼륨을 사용해야 한다.(여러 파드에 걸쳐 데이터를 유지하기 위해서는 사용 금지)

``` sh
# kube-system 네임스페이스의 시스템 파드 조회
kubectl get pods --namespace kube-system

# 시스템 파드 Path 확인
kubectl describe po kube-controller-manager-minikube --namespace kube-system

# path 내용들
ca-certs:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/ssl/certs
    HostPathType:  DirectoryOrCreate
  flexvolume-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /usr/libexec/kubernetes/kubelet-plugins/volume/exec
    HostPathType:  DirectoryOrCreate
  k8s-certs:
    Type:          HostPath (bare host directory volume)
    Path:          /var/lib/minikube/certs
    HostPathType:  DirectoryOrCreate
  kubeconfig:
    Type:          HostPath (bare host directory volume)
    Path:          /etc/kubernetes/controller-manager.conf
    HostPathType:  FileOrCreate
```

## 6.4 퍼시스턴트 스토리지 사용
 - 파드에서 실행중인 애플리케이션이 디스크에 데이터를 유지해야 하고 파드가 다른 노드로 재스케줄링된 경우에도 동일한 데이터를 사용해야 하는 경우를 NAS 같은 유형에 데이터를 저장해야 한다.
 - 이를 위한 방법을 쿠버네티스가 제공한다.
 - minikube로 연습하는 경우에는 hostPath 볼륨으로 사용하면 된다.

### 6.4.1 GCE 퍼시스턴트 디스크를 파드 볼륨에 사용하기
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    gcePersistentDisk:                 # 볼륨의 유형은 GCE 퍼시스턴트 디스크
      pdName: mongodb
      fsType: ext4                         # 리눅스 파일시스템 유형
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db           # 컨테이너 내 마운트 되는 path
    ports:
    - containerPort: 27017
      protocol: TCP
```

<img width="593" alt="스크린샷 2020-08-17 오후 6 57 35" src="https://user-images.githubusercontent.com/6982740/90383744-87cbad80-e0bb-11ea-95b7-f3534c77f9b0.png">

### 6.4.2 기반 퍼시스턴트 스토리지로 다른 유형의 볼륨 사용하기
 - awsElasticBlockStore, azureFile이나 azureDisk 볼륨을 사용할 수 있음.

#### AWS Elastic Block Store 볼륨 사용
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-aws
spec:
  volumes:
  - name: mongodb-data
    awsElasticBlockStore:                 # awsElasticBlockStore
      volumeID: my-volume
      fsType: ext4
...
```

#### NFS 볼륨 사용
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-nfs
spec:
  volumes:
  - name: mongodb-data
    nfs:
      server: 1.2.3.4
      path: /some/path
...
```

#### 다른 스토리지 기술 사용
 - 쿠버네티스는 왠만한 모든 기술의 다양한 스토리지를 지원한다.


## 6.5 기반 스토리지 기술과 파드 분리
 - 쿠버네티스에 앱을 배포하는 개발자는 기저에 어떤 종류의 스토리지 기술이 사용되는 알 필요 없어야 하고, 동일한 방식으로 파드를 실행하기 위해 어떤 유형의 물리 서버가 사용되는지 알 필요 없어야 한다.(이상적)
 - 파드의 볼륨이 실제 기반 인프라스르럭처를 참조한다는 것은 쿠버네티스가 추구하는 바가 아님
 - 인프라 스트럭처 관련 정보를 파드 정의에 포함한다는 것은 파드 정의가 특정 쿠버네티스 클러슽에 밀접하게 연결됨을 의미한다. 동일한 파드 정의를 다른 클러스터에서는 사용할 수 없다.

### 6.5.1 퍼시스턴트볼륨(PV, PersistentVolume)과 퍼시스턴트볼륨클레임(PVC, PersistentVolumeClaim)
 - 인프라스트럭처의 세부 사항을 처리하지 않고 앱이 스토리지를 요청할 수 있도록 하기 위한 리소스 유형
 - 관리자는 네트워크 스토리지 유형을 서정하고, PV 디스크립터를 게시하여 퍼시스턴트볼륨을 생성한다.
 - 사용자는 퍼시스턴트볼륨클레임(PVC)을 생성하면, 쿠버네티스가 적당한 크기와 접근모드의 PV를 찾아서 PVC를 PV에 바인딩시킨다.
 - 사용자는 이제 PVC를 참조하는 볼륨을 가진 파드를 생성한다.

<img width="574" alt="스크린샷 2020-08-17 오후 7 07 36" src="https://user-images.githubusercontent.com/6982740/90384690-ee050000-e0bc-11ea-8704-dc68264d66e7.png">

### 6.5.2 퍼시스턴트볼륨 생성
 - 퍼시스턴트볼륨을 생성할 때 동시에 단일 또는 다수 노드에 읽기나 쓰기가 가능한지 여부 등을 지정해야 하고, 퍼시스턴트볼륨 해제시 어떤 동작을 해야 할지 정의해야 한다.
 - 퍼시스턴트 볼륨을 지원하는 실제 스토리지의 유형, 위치, 그 밖의 속성 정보를 지정
 - 퍼시스턴트 볼륨은 특정 네임스페이스에 속하지 않고, 노드와 같은 수준의 클러스터 리소스이다.

#### 정의
``` yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce                        # 단일 클라이언트의 읽기/쓰기용으로 마운트
    - ReadOnlyMany                         # 여러 클라이언트의 읽기 전용으로 마운트
  persistentVolumeReclaimPolicy: Retain    # 클레임이 해제된 후 퍼시스턴트볼륨을 유지한다.
  hostPath:                                # ohstPath 볼륨 (minikube)
    path: /tmp/mongodb
```

#### persistentVolumeReclaimPolicy
>  현재 NFS 및 HostPath만 재활용을 지원한다. AWS EBS, GCE PD, Azure Disk 및 Cinder 볼륨은 삭제를 지원한다.

 - Retain(보존) -- 수동 반환
 - Recycle(재활용) -- 기본 스크럽 (rm -rf /thevolume/*)
 - Delete(삭제) -- AWS EBS, GCE PD, Azure Disk 또는 OpenStack Cinder 볼륨과 같은 관련 스토리지 자산이 삭제됨

#### 생성 및 조회
``` sh
# hostPath PV 생성
kubectl create -f mongodb-pv-hostpath.yaml

# pv 조회
kubectl get pv
```

<img width="612" alt="스크린샷 2020-08-17 오후 7 22 00" src="https://user-images.githubusercontent.com/6982740/90386041-f0685980-e0be-11ea-8ce2-33cdac104986.png">


### 6.5.3 퍼시스턴트볼륨클레임 생성을 통한 퍼시스턴트볼륨 요청
 - 파드가 재스케줄링되더라도 동일한 퍼시스턴트볼륨클레임이 사용 가능한 상태로 유지되기를 원하므로 퍼시스턴트 볼륨에 대한 클레임은 파드를 생성하는 것과 별개의 프로세스이다.

#### 퍼시스턴트볼륨클레임 생성하기
 - 퍼시스턴트볼륨클레임이 생성되자마자 쿠버네티스는 적절한 퍼시스턴트볼륨을 찾고 클레임에 바인딩한다.
 - 용량은 퍼시스턴트볼륨클레임의 요청을 수용할만큼 충분히 커야하고, 볼륨 접근 모드는 클레임에서 요청한 접근모드를 포함하는 상태여야 바인딩이 이루어진다.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:                         # 1GiB의 스토리지를 요청
      storage: 1Gi
  accessModes:
  - ReadWriteOnce               # 단일 클라이언트를 지원하는 읽기/쓰기 스토리지
  storageClassName: ""
```

#### 퍼시스턴트볼륨클레임 조회하기
``` sh
# 퍼시스턴트볼륨클레임 생성
kubectl create -f mongodb-pvc.yaml

# 퍼시스턴트볼륨클레임 조회
kubectl get pvc
```

#### 퍼시스턴트볼륨 접근모드
 - RWO(ReadWriteOnce) : 단일 노드만이 읽기/쓰기용으로 볼륨을 마운트
 - ROX(ReadOnlyMany) : 다수 노드가 읽읽기용으로 볼륨을 마운트
 - RWX(ReadWriteMany) : 다수 노드가 읽기/쓰기용으로 볼륨을 마운트

### 6.5.4 파드에서 퍼시스턴트볼륨클레임 사용하기
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data                # 파드 볼륨에서 이름으로 퍼시스턴트볼륨클레임을 참조
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

#### mongodb test
``` sh
# 퍼시스턴트클레임을 이용하는 mongodb pod 생성
kubectl create -f mongodb-pod-pvc.yaml

# mongodb 셸 접속
kubectl exec -it mongodb mongo

# mongodb db 변경
use mystore

# 데이터 insert
db.foo.insert({name:'foo'})

# 데이터 조회
db.foo.find()
```

### 6.5.5 퍼시스턴트볼륨과 퍼시스턴트볼륨클레임 사용의 장점 이해하기
 - GCE 퍼시스터턴트 디스크를 직접 사용하는 경우와 PVC + PV를 사용하는 경우 비교
 - 개발자가 직접 인프라스트럭처에서 스토리지를 가져오는 방식보다는 PV + PVC을 통해간접적으로 가져오는 방식이 더 간단하다(인프라스트럭처를 몰라도됨)
 - 또한 동일한 파드와 클레임 매니페스트는 인프라스트럭처와는 관련된 어떤것도 참조하지 않으므로 다른 쿠버네티스 클러스터에서도 그대로 사용할 수 있다.
 - "클레임은 x만큼의 스토리지가 필요하고 한 번에 하나의 클라이언트에서 읽기와 쓰기를 할 수 있어야 한다"만 명시한다.

<img width="595" alt="스크린샷 2020-08-17 오후 11 34 27" src="https://user-images.githubusercontent.com/6982740/90408022-371b7b00-e0e2-11ea-9d36-ba056fbc2b43.png">

### 6.5.6 퍼시스턴트볼륨 재사용
``` sh
# mongodb pod 삭제
kubectl delete pod mongodb

# pvc 삭제
kubectl delete pvc mongodb-pvc

#pvc, pod 재생성 ( 이 경우 pvc는 바로 volume을 할당받지 못하고 Pending 상태가 된다.)
kubectl create -f mongodb-pvc.yaml
kubectl create -f mongodb-pod-pvc.yaml

# pvc 조회
kubectl get pvc

# pv 조회 ( 퍼시스턴트볼륨의 상태가 Released로 표시되고 Available이 아니다. 그 이유는 이미 볼륨을 사용했기 떄문에 데이터를 가지고 있어서 새로운 클레임을 바인딩할 수 없는 상태)
kubectl get pv
```

#### 퍼시스턴트볼륨을 수동으로 다시 클레임하기
 - persistentVolumeClaimPolicy를 Retain으로 설정하면 퍼시스턴트볼륨클레임이 해제되더라도 데이터가 남아있으면 상태가 Available로 풀리지 않는다.

#### 퍼시스턴트볼륨을 자동으로 다시 클레임하기
 - 다른 리클레임 정책인 Recycle과 Delete가 있는데 Recycle은 볼륨의 콘텐츠를 삭제하고 다시 클레임될수 있도록 만드는 옵션이다.
 - Delete 정책은 쿠버네티스에서 퍼시스턴트볼륨 오브젝트와 외부 인프라(예: AWS EBS, GCE PD, Azure Disk 또는 Cinder 볼륨)의 관련 스토리지 자산을 모두 삭제한다.
 - Recycle과 Delete의 차이는 pvc가 삭제될때 pv까지 삭제하느냐 안하느냐에 대한 차이가 있음(Delete는 pvc를 삭제하면 pv까지 삭제함)

<img width="684" alt="스크린샷 2020-08-18 오전 12 29 06" src="https://user-images.githubusercontent.com/6982740/90413802-ed369300-e0e9-11ea-9768-cbf5f29b2f5b.png">


## 6.6 퍼시스턴트볼륨의 동적 프로비저닝
### 6.6.2 퍼시스턴트볼륨클레임에서 스토리지 클래스 요청하기
#### 특정 스토리지클래스를 요청하는 pvc 정의
 - 클레임을 생성하면 fast 스토리지클래스 리소스에 참조된 프로비저너가 퍼시스턴트볼륨을 생성한다.
 - PVC에서 존재하지 않는 스토리지클래스를 참조하면 PV 프로비저닝은 실패한다.
 - kubectl describe 로 확인해보면 ProvisioningFailed 이벤트 표시됨.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  storageClassName: fast           # PVC는 사용자 정의 스토리지 클래스를 요청
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

#### 스토리지클래스 정의
``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd

```

#### 동적 프로비저닝된 PV와 생성된 PVC 조회
 - 이렇게 생성된 PV는 리클레임 정책 Delete을 가지며, PVC가 삭제되면 PV도 삭제된다.

``` sh
# 스토리지클래스 생성
kubectl create -f storageclass-fast-hostpath.yaml

# PVC 생성
kubectl create -f mongodb-pvc-dp.yaml

# pvc 조회
kubectl get pvc

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
mongodb-pvc   Bound    pvc-b767134f-218a-48cc-b1a4-4787a661fd09   100Mi      RWO            fast           16m

# pv 조회
kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
pvc-b767134f-218a-48cc-b1a4-4787a661fd09   100Mi      RWO            Delete           Bound    default/mongodb-pvc   fast                    16m
```

#### 스토리지 클래스 사용하는 법 이해하기
 - 스토리지클래스의 좋은 점은 클레임 이름으로 이를 참조한다는 사실. 그래서 다른 클러스터간 스토리지클래스 이름을 동일하게 사용한다면 PVC 정의를 다른 클러스터로 이식도 가능하다.


### 6.6.3 스토리지 클래스를 지정하지 않는 동적 프로비저닝
``` sh
# 스토리지 클래스 조회
kubectl get sc

NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
fast                 k8s.io/minikube-hostpath   Delete          Immediate           false                  18m
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  26h

# 기본 스토리지 클래스 확인
kubectl get sc standard -o yaml
```

#### 스토리지 클래스를 지정하지 않고 퍼시스턴트볼륨클레임 생성하기
``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc2
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

 - storageClassName 속성을 지정하지 않고 PVC를 생성하면 구글 쿠버네티스 엔진에서는 pd-standard 유형의 퍼시스턴트 디스크가 프로비저닝된다.


``` sh
# 스토리지 클래스를 지정하지 않고 pvc 생성
kubectl create -f mongodb-pvc-dp-nostorageclass.yaml

# pvc 확인
kubectl get pvc

# pv 확인
kubectl get pv
```

#### 퍼시스턴트볼륨클레임을 미리 프로비저닝된 퍼시스턴트볼륨으로 바인딩 강제화하기
 - storageClassName 속성을 빈 문자열로 지정하지 않으면 미리 프로비저닝된 퍼시스턴트볼륨이 있다고 할지라도 동적 볼륨 프로비저너는 새로운 퍼시스턴트볼륨을 프로비저닝한다.
 - 미리 프로비저닝된 PV에 바인딩하기 위해서는(미리 만들어둔 PV에 바인딩하려면) 명시적으로 storageClassName을 ""로 지정해야 한다.

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""      # 빈 문자열을 스토리지클래스 이름으로 지정하면 PVC가 새로운 PV를 동적 프로비저닝하지 않고 미리 프로비저닝된 PV에 바인딩된다.
```

#### 퍼시스턴트볼륨 동적 프로비저닝의 플로우
 - 클러스터 관리자는 퍼시스턴트볼륨 프로비저너를 설정하고, 하나 이상의 스토리지 클래스를 생성하고 기본값 정의
 - 사용자는 스토리지클래스 중 하나를 참조해 PVC를 생성
 - PVC는 스토리지클래스와 거기서 참조된 프로비저너를 보고 PVC로 요청된 접근모드, 스토리지 크기, 파라미터를 기반으로 새 PV를 프로비저닝하도록 요청
 - 프로비저너는 스토리지를 프로비저닝하고 PV를 생성한 후 PVC에 바인딩한다.
 - 사용자는 PVC를 이름으로 참조하는 볼륨과 파드를 생성

<img width="682" alt="스크린샷 2020-08-18 오전 12 30 12" src="https://user-images.githubusercontent.com/6982740/90413845-fe7f9f80-e0e9-11ea-8bf2-7be6c28b94dc.png">

## 6.7 요약
 - 다중 컨테이너 파드 생성과 파드의 컨테이너들이 볼륨을 파드에 추가하고 각 컨테이너에 마운트해 동일한 파일로 동작하게 할 수 있다.
 - emptyDir 볼륨을 사용해 임시, 비영구 데이터를 저장할 수 있다.
 - gitRepo 볼륨을 사용해 파드의 시작 시점에 깃 리포지터리의 콘텐츠로 디렉터리를 쉽게 채울수 있다.
 - hostpath 볼륨을 사용해 호스트 노드의 파일에 접근한다.
 - 외부 스토리지를 볼륨에 마운트해 파드가 재시작돼도 파드의 데이터를 유지한다.
 - 퍼시스턴트볼륨과 퍼시스턴트볼륨클레임을 사용해 파드와 스토리지 인프라스트럭처를 분리할수 있다.
 - 스토리지클래스를 이용하면 PVC가 원하는 만큼의 PV를 프로비저닝할 수 있다.
 - PVC을 미리 프로비저닝된 PV에 바인딩하고자 할 때 동적 프로비저너가 간섭하는 것을 막을 수도 있다.

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
