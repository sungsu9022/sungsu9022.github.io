---
title: "[kubernetes-in-action] 7. 컨피그맵과 시크릿 : 애플리케이션 설정"
author: sungsu park
date: 2020-08-22 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 7. 컨피그맵과 시크릿 : 애플리케이션 설정
> 빌드된 애플리케이션 자체에 포함되지 말아야 하는 설정(배포된 인스턴스별로 다른 세팅, 외부 시스템 엑세스를 위한 자격증명 등)이 필요하다.
> 쿠버네티스는 이런 앱을 실행할때 설정 옵션을 전달할수 있는 방법을 제공한다.

## 7.1 컨테이너화된 애플리케이션 설정
 - 필요한 모든 설정을 앱에 포함하는 경우를 제외하면 일반적으로 명령줄 인수를 통해 앱에 필요한 설정을 넘겨주면서 앱을 실행시킨다.
 - 옵션 목록이 커지면 이 옵션들을 파일에 저장하고 사용하기도 한다.
 - 아니면 서버의 환경변수를 통해 전달하기도 한다.
 - 도커 컨테이너를 기반으로 한다고 했을때 내부에 있는 설정 파일을 사용하는것은 약간 까다롭다.(컨테이너 이미지 안에 넣는것은 소스코드에 하드코딩하는것과 다를바가 없음)
 - 인증 정보나 암호화 키와 같이 비밀로 유지해야 하는 내용을 포함하게 되면 해당 이미지에 접근할 수 있는 모든 사람은 누구나 정보를 볼수 있게 되버린다.
 - 쿠버네티스에서는 설정 데이터를 최상위 레벨의 쿠버네티스 오브젝트에 저장하고 이를 기타 다른 리소스 정의와 마찬가지로 깃 저장소 혹은 다른 파일 기반 스토리지에 저장할 수 있다.
 - 이러한 목적으로 쿠버네티스는  컨피그맵이라는 리소스를 제공한다.
 - 대부분의 설정 옵션에서는 민감한 정보가 포함돼있지 않지만, 자격증명, 개인 암호화 키, 보안을 유지해야 하는 유사한 데이터들도 있다. 이를 위한 시크릿이라는 또다른 유형의 오브젝트도 제공한다.

### 애플리케이션에 설정을 전달하는 방법
 - 컨테이너에 명령줄 인수 전달
 - 각 컨테이너를 위한 사용자 정의 환경변수 지정
 - 특수한 유형의 볼륨을 통해 설정

## 7.2 컨테이너에 명령줄 인자 전달
### 7.2.1 도커에서 명령어와 인자 정의
#### ENTRYPOINT와 CMD 이해
 - ENTRYPOINT는 컨테이너가 시작될때 호출될 명령어 정의
 - CMD는 ENTRYPOINT에 전달되는 인자를 정의

#### shell과 exec 형식간의 차이점
 - shell 형식 : ENTRYPOINT node app.js

<img width="400" alt="스크린샷 2020-08-19 오후 11 39 55" src="https://user-images.githubusercontent.com/6982740/90649119-4bdc4800-e275-11ea-8c4e-d4a3142d1a23.png">

 - exec 형식 : ENTRYPOINT ["node", "app.js"]

<img width="329" alt="스크린샷 2020-08-19 오후 11 39 16" src="https://user-images.githubusercontent.com/6982740/90649077-3ff08600-e275-11ea-91a0-118cb55a8682.png">

#### fortune 이미지에서 간격을 설정할 수 있도록 만들기
 - INTERVAL 변수를 추가하고 첫 번쨰 명령줄 인자의 값으로 초기화

``` sh
# fortuneloop.sh
#!/bin/bash
trap "exit" SIGINT

INTERVAL=$1        # 인자
echo Configured to generate new fortune every $INTERVAL seconds

mkdir -p /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

```dockerfile
# Dockerfile
FROM ubuntu:latest

RUN apt-get update ; apt-get -y install fortune
ADD fortuneloop.sh /bin/fortuneloop.sh

ENTRYPOINT ["/bin/fortuneloop.sh"]    # exec 형태의 ENTRYPOINT 명령
CMD ["10"]                                                # 실행할때 사용할 기본 인자
```

### 7.2.2 쿠버네티스에서 명령과 인자 재정의
 - command와 args 필드로 맵핑된다. 이는 파드 생성 이후에는 업데이트 할수 없다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: some/image
    command: ["/bin"/commend"]
    args: ["arg1", "arg2", "arg3"]
```

#### 사용자 정의 주기로 fortune 파드 실행
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune2s
spec:
  containers:
  - image: luksa/fortune:args
    args: ["2"]                               # 스크립트가 2초마다 새로운 fortune 메시지를 생성하도록 인자 지정
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
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
    emptyDir: {}
```

#### 여러개의 인자 처리

``` yaml
    args:
     - foo
     - bar
     - "15"
```

#### 숫자 인자에 대한 처리
- 문자열인 경우는 ""로 묶을 필요 없지만 숫자는 묶어야 한다.

## 7.3 컨테이너의 환경변수 설정
 - 파드의 각 컨테이너를 위한 환경변수 리스트를 지정할 수 있다.
 - 컨테이너 명령이나 인자와 마찬가지로 환경변수 목록도 파드 생성 후에는 업데이트 불가
 - 컨테이너별로 다른 환경변수 설정도 가능하다.

<img width="253" alt="스크린샷 2020-08-19 오후 11 56 13" src="https://user-images.githubusercontent.com/6982740/90651055-94950080-e277-11ea-90b2-c228586c38b0.png">

### 환경변수로 fortune 이미지 안에 간격을 설정할 수 있도록 만들기
 - 변수 초기화하는 부분을 제거하면 끝.

``` sh
#!/bin/bash
trap "exit" SIGINT

echo Configured to generate new fortune every $INTERVAL seconds

mkdir -p /var/htdocs

while :
do
  echo $(date) Writing fortune to /var/htdocs/index.html
  /usr/games/fortune > /var/htdocs/index.html
  sleep $INTERVAL
done
```

### 7.3.1 컨테이너 정의에 환경변수 지정
 - 각 컨테이너를 설정할 때, 쿠버네티스는 자동으로 동일한 네임스페이스 안에 있는 각 서비스에 환경변수를 노출하는것은 알고 있어야 한다.(파드에 정의한 환경변수명이 같아 덮어써지는 부분이 있을수 있어서인듯)

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env
spec:
  containers:
  - image: luksa/fortune:env
    env:                                           # 환경변수 목록에 단일 변수 추가
    - name: INTERVAL
      value: "30"
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
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
    emptyDir: {}
```

### 7.3.2 변숫값에서 다른 환경변수 참조
``` yaml
env:
    - name: FIRST_VAR
      value: "foo"
    - name: SECOND_VAR
      value: "${FIRST_VAR}bar"     # foobar가 됨
```

### 7.3.3 하드코딩된 환경변수의 단점
 - 하드코딩된 값을 가져오는게 효율적일 수 있지만, 프로덕션과 개발환경의 파드를 별도로 정의해야할수도 있다는 의미
 - 여러 환경에서 동일한 파드 정의를 재사용하려면 파드 정의에서 설정을 분리하는것이 좋다.

## 7.4 컨피그맵으로 설정 분리
 - 환경에 따라 다르거나 자주 변경되는 설정 옵션을 애플리케이션 소스 코드와 별도로 유지하는 것.

### 7.4.1 컨피그맵 소개
 - 쿠버네티스에서는 설정 옵션을 컨피그맵이라 부르는  별도 오브젝트로 분리할 수 있다.
 - 짧은 문자열에서 전체 설정 파일에 이르는 값을 가지는 키/값 쌍으로 구성된 맵이다.
 - 쿠버네티스 REST API 엔드포인트를 통해 컨피그맵의 내용을 직접 읽는것이 가능하지만, 반드시 필요한 경우가 아니라면  애플리케이션 내부는 쿠버네티스와 무관하도록 유지해야 한다.

<img width="292" alt="스크린샷 2020-08-23 오후 4 16 28" src="https://user-images.githubusercontent.com/6982740/90973329-0ca34500-e55c-11ea-965b-3ddd2ea53d39.png">

 - 파드는 컨피그맵을 이름으로 참조하여 모든 환경에서 동일한 파드 정의를 사용해 각 환경에서 서로 다른 설정을 사용할 수 있다.

<img width="507" alt="스크린샷 2020-08-23 오후 4 18 37" src="https://user-images.githubusercontent.com/6982740/90973361-555afe00-e55c-11ea-8213-7c313345b59a.png">

### 7.4.2 컨피그맵 생성

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fortune-config
data:
  sleep-interval: "25"
```

 - kubectl create configmap로 생성 가능
 - 컨피그맵 키는 유효한 DNS 서브도메인이어야 한다.(영숫자, 대시, 밑줄, 점만 포함 가능)

``` sh
# 컨피그맵 생성 (file)
kubectl create -f fortune-config.yaml

# 컨피그맵 생성 (literal)
kubectl create configmap fortune-config --from-literal=sleep-interval=25

# 컨피그맵 생성 (여러개의 literal)
kubectl create configmap myconfigmap --from-literal=foo=bar --from-literal=bar=baz --from-literal=one=two

# 컨피그맵 목록 조회
kubectl get cm

# 컨피그맵 정의 확인
kubectl get configmap fortune-config -o yaml
```

#### 파일 내용으로 컨피그맵 생성
 - 전체 설정 파일 같은 데이터를 통째로 컨피그맵에 저장할 수 있다.

``` sh
# 파일 통째로 컨피그맵 생성
#  - 파일 자체가 값으로 지정됨
kubectl create configmap my-config --from-file=config-file.conf

# 파일 통째로 저장하되 customkey  지정
kubectl create configmap my-config --from-file=customkey=config-file.conf
```

#### 디렉터리에 있는 파일 컨피그맵 생성
``` sh
kubectl create configmap my-config --from-file=/path/to/dir

#### 다양한 옵션 결합
``` sh
kubectl create configmap my-config
    --from-file=foo.json                  # 단일 파일
    --from-file=bar=foobar.conf    # 사용자 정의 키 밑에 파일 저장
    --from-file=config-opts/          # 전체 디렉터리
    --from-literal=some=thing       # 문자열 값
```

<img width="650" alt="스크린샷 2020-08-23 오후 4 31 07" src="https://user-images.githubusercontent.com/6982740/90973560-1a59ca00-e55e-11ea-982a-364cb2fbe1a6.png">


### 7.4.3 컨피그맵 항목을 환경변수로 컨테이너에 전달
 - 생성한 맵의 값을 어떻게 파드 안의 컨테이너를 전달할수 있는 방법을 살펴보자

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-env-from-configmap
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL               # INTERVAL 환경변수 설정
      valueFrom:
        configMapKeyRef:            # 컨피그맵 키에서 값을 가져와 초기화
          name: fortune-config     # 참조하는 컨피그맵 이름
          key: sleep-interval          # 컨피그맵에서 해당 키 아래에 저장된 값으로 변수 셋팅
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
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
    emptyDir: {}
```

<img width="577" alt="스크린샷 2020-08-23 오후 4 45 18" src="https://user-images.githubusercontent.com/6982740/90973768-0adb8080-e560-11ea-9de7-0dc283009381.png">

#### 파드에 존재하지 않는 컨피그맵 참조
 - 존재하지 않는 컨피그맵을 참조하려고 하면 컨테이너는 시작하는데 실패한다.
 - 하지만 참조하지 않는 다른 컨테이너는 정상적으로 시자된다. 누락된 컨피그맵을 생성하면 실패했던 컨테이너는 파드를 다시 만들지 않아도 자동으로 시작된다.
 - 컨피그맵 참조를 옵션으로 표시할수도 있다. ( configMapKeyRef.optional : true로 지정), 이런 경우는 컨피그맵이 존재하지 않아도 컨테이너가 시작된다.

### 7.4.4 컨피그맵의 모든 항목을 한번에 환경변수로 전달
 - 컨피그맵의 모든 항목을 환경변수로 노출할 수 있는 방법을 제공한다.
 - 접두사는 선택사항이고, 이를 생략하면 환경변수의 이름은 키와 동일한 이름을 갖게 된다.
 - CONFIG_FOO-BAR는 대시를 가지고 있어 올바른 환경변수 이름이 아니기 때문에 이런 경우 환경변수로 변환되지 않는다.(올바른 형식이 아닌 경우 쿠버네티스에서 생략함)

``` yaml
spec:
  containers:
  - image: some-image
    envForm:                               # env 대신 envForm 사용
    - prefix: CONFIG_                 # 모든 환경변수는 CONFIG_ prefix로 설정됨.
        configMapRef:                  # my-config-map 이름의 컨피그맵 참조
          name: my-config-map
...
```

### 7.4.5 컨피그맵 항목을 명령줄 인자로 전달
 - pod.spec.containers.args 필드에서 직접 컨피그맵 항목을 참조할 수는 없지만 컨피그맵 항목을 환경변수로 먼저 초기화하고 이 벼수를 인자로 참조할 수 있다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-args-from-configmap
spec:
  containers:
  - image: luksa/fortune:args          # 환경변수가 아닌 첫번째 인자에서 간격을 가져오는 이미지
    env:
    - name: INTERVAL                       # 컨피그맵에서 환경변수 정의
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    args: ["$(INTERVAL)"]                  # 인자에 앞에서 정의한 환경변수를 지정
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
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
    emptyDir: {}
```

<img width="566" alt="스크린샷 2020-08-23 오후 4 51 12" src="https://user-images.githubusercontent.com/6982740/90973854-dc11da00-e560-11ea-87ae-2d60bd828f16.png">

### 7.4.6 컨피그맵 볼륨을 사용해 컨피그맵 항목을 파일로 노출
 - 컨피그맵은 모든 설정 파일을 포함한다.
 - 컨피그맵 볼륨을 사용해서도 적용할 수가 있다.
 - 컨피그맵 볼륨은 파일로 컨피그맵의 각 항목을 노출한다.

#### 컨피그맵 생성
 - nginx 서버가 클라이언트로 응답을 gzip 압축해서 보내는 니즈가 있다고 해보자.
 - nginx gzip 압축 옵션을 활성화하고 이에 대한 컨피그맵을 생성해야 한다.


``` conf
# nginx gzip config 정의
server {
    listen              80;
    server_name         www.kubia-example.com;

    gzip on;                                                         # gzip 압축 활성화
    gzip_types text/plain application/xml;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
}
```

``` sh
# configMap 삭제
kubectl delete cm fortune-config

# configMap dir로 생성
kubectl create configmap fortune-config --from-file=configmap-files

# configMap 확인
kubectl get cm fortune-config -o yaml
```

``` yaml
# 컨피그맵 내용
apiVersion: v1
data:
  my-nginx-config.conf: |                      # 파이프라인(|) 문자는 여러 줄의 문자열이 이어진다는 것을 의미한다.
    server {
        listen              80;
        server_name         www.kubia-example.com;

        gzip on;
        gzip_types text/plain application/xml;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

    }
  sleep-interval: |
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-08-23T07:55:47Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:my-nginx-config.conf: {}
        f:sleep-interval: {}
    manager: kubectl
    operation: Update
    time: "2020-08-23T07:55:47Z"
  name: fortune-config
  namespace: default
  resourceVersion: "3405"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: bc35aea1-618a-4961-a8ef-06136d71825c
```

#### 볼륨 안에 있는 컨피그맵 항목 사용
 - 컨피그맵 항목에서 생성된 파일로 볼륨을 초기화하는 방법

<img width="512" alt="스크린샷 2020-08-23 오후 4 58 05" src="https://user-images.githubusercontent.com/6982740/90973957-d2d53d00-e561-11ea-9f68-bd3017f4d11a.png">

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume
spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:                                           # 컨피그맵 볼륨 마운트
    - name: html
      mountPath: /usr/share/nginx/html          # 컨피그맵 볼륨을 마운트하는 컨테이너 위치
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: config
      mountPath: /tmp/whole-fortune-config-volume
      readOnly: true
    ports:
      - containerPort: 80
        name: http
        protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:                             # 이 볼륨은 fortune-config 컨피그맵을 참조하는 볼륨
      name: fortune-config
```

#### nginx 서버가 마운트한 설정 파일을 사용하는지 확인
``` sh
# pod 생성
kubectl create -f fortune-pod-configmap-volume.yaml

# port forward
 kubectl port-forward fortune-configmap-volume 8080:80 &

# reqeuest
curl -H "Accept-Encoding: gzip" -I localhost:8080

# 마운트된 컨피그맵 볼륨 내용 살펴보기
kubectl exec fortune-configmap-volume -c web-server ls /etc/nginx/conf.d
```

#### 볼륨에 특정 컨피그맵 항목 노출
 - 아래와 같이 설정하면 컨테이너 마운트 위치 '/etc/nginx/conf.d/' 디렉터리에는 gzip.conf 파일만 포함된다.

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-configmap-volume-with-items
spec:
  containers:
  - image: luksa/fortune:env
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d/     # 컨테이너 마운트 path
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:                            # 볼륨에 포함할 항목을 조회해 선택
      - key: my-nginx-config.conf       # 해당 키 아래에 항목을 포함
        path: gzip.conf                 # 항목 값이 지정된 파일에 저장
```

#### 디렉터리를 마운트할 때 디렉터리의 기존 파일을 숨기는 것에 대한 이해
 - 리눅스 파일시스템을 비어 있지 않은 디렉터리에 마운트할떄 발생할 수 있는 문제로, 디렉터리는 마운트한 파일시스템에 있는 파일만 포함하고, 원래 있던 파일은 해당 파일시스템이 마운트돼 있는 동안 접근할 수 없게 된다.
 - 중요한 파일을 포함하는  /etc 디렉터리에 볼륨을 마운트한다고 하면 /etc 디렉터리에 있어야 하는 모든 원본 파일이 더 이상 존재하지 않게 되어 전체 컨테이너가 손상될 수 있다.

#### 디렉터리 안에 다른 파일을 숨기지 않고 개별 컨피그맵 항목을 파일로 마운트
 - 전체 볼륨을 마운트하는 대신 volumeMount에 subPath 속성으로 파일이나 디렉터리 하나를 볼륨에 마운트할 수 있다.

<img width="545" alt="스크린샷 2020-08-23 오후 5 06 37" src="https://user-images.githubusercontent.com/6982740/90974099-04023d00-e563-11ea-8e05-648a5c6bdbfe.png">

``` yaml
spec:
  containers:
  - image: some/image
    volumeMounts:
    - name: myvolume
      mountPath: /etc/someconfig.conf      # 디렉터리가 아닌 파일을 마운트
      subPath: myconfig.conf               # 전체 볼륨을 마운트하는 대신 myconfig.conf 항목만 마운트
```

 - subPath 속성은 모든 종류의 볼륨을 마운트할때 사용할 수 있다. 하지만 개별 파일을 마운트하는 이 방법은 파일 업데이트와 관ㄹ녀해 상대적으로 큰 결함을 가지고 있다.

#### 컨피그맵 볼륨 안에 있는 파일 권한 수정
 - 기본적으로 컨피그맵 볼륨의 모든 파일 권한은 644(-rw-r-r--)로 설정된다.
  - defaultMode 속성을 설정하여 변경이 가능하다.

``` yaml
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      defaultMode: 0660           # 모든 파일 권한을 660(-rw-rw---) 로 설정
```


### 7.4.7 애플리케이션을 재시작하지 않고 설정 업데이트
 - 컨피그맵을 사용해 볼륨으로 노출하면 파드를 다시 만들거나 컨테이너를 다시 시작할 필요 없이 설정을 업데이트할 수 있다.
 - 컨피그맵을 업데이트한 후에 파일이 업데이트되기까지는 생각보다 오랜 시간이 걸릴수 있다.(최대 1분)

#### 컨피그맵 편집
``` sh
# configmap 수정
kubectl edit configmap fortune-config

# web-server 컨테이너 내 설정파일 확인
kubectl exec fortune-configmap-volume -c web-server cat /etc/nginx/conf.d/my-nginx-config.conf
```

#### 설정을 다시 로드하기 위해 nginx에 신호 전달
``` sh
kubectl exec fortune-configmap-volume -c web-server -- nginx -s reload
```

#### 파일이 한꺼번에 업데이트되는 방법 이해
 - 쿠버네티스 컨피그맵은 모든 파일이 한번에 업데이트된다. (심볼릭 링크 방식으로 동작하기 때문)

``` sh
# 컨테이너 디렉토리 확인
kubectl exec -i -t fortune-configmap-volume -c web-server -- ls -al /etc/nginx/conf.d

# 결과
drwxrwxrwx    3 root     root          4096 Aug 23 08:14 .
drwxr-xr-x    3 root     root          4096 Aug 14 00:37 ..
drwxr-xr-x    2 root     root          4096 Aug 23 08:14 ..2020_08_23_08_14_11.042426742
lrwxrwxrwx    1 root     root            31 Aug 23 08:14 ..data -> ..2020_08_23_08_14_11.042426742
lrwxrwxrwx    1 root     root            27 Aug 23 08:14 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root            21 Aug 23 08:14 sleep-interval -> ..data/sleep-interval
```

#### 이미 존재하는 디렉터리에 파일만 마운트했을 때 업데이트가 되지 않는 것 이해하기
 - 단일 파일만 컨테이너에 마운트한 경우 파일이 업데이트 되지 않는다.(단순 컨피그맵 + 볼륨 기능만 이용하는 경우에 한하여)

#### 컨피그맵 업데이트의 결과 이해하기
 - 컨테이너의 가장 주용한 기능은 불변성(immutability)이다.
 - 앱이 설정을 다시 읽는 기능을 지원하지 않는 경우에 심각한 문제가 발생한다.
 - 컨피그맵을 변경한 이후 생성된 파드는 새로운 설정을 사용하지만 예전 파드는 계속 예전 설정을 사용하기 때문.
 - 애플리케이션이 설정을 자동으로 다시 읽는 기능을 가지고 있지 않다면 이미 존재하는 컨피그맵을 수정하는것은 좋은 방법이 아니다.

## 7.5 시크릿으로 민감한 데이터 컨테이너에 전달
### 7.5.1 시크릿 소개
 - 쿠버네티스는 민감한 정보를 보관하고 배포하기 위하여 시크릿이라는 오브젝트를 제공한다.
 - 시크릿은 키-값 쌍을 가진 맵으로 컨피그맵과 매우 비슷하다.
 - 컨피그맵과 마찬가지로 환경변수로 시크릿 항목을 컨테이너에 전달하거나 볼륨 파일로 노출시킬 수 있다.
 - 시크릿에 접근해야 하는 파드가 실행되고 있는 노드에만 개별 시크릿을 배포해 시크릿을 안전하게 유지한다.
 - 노드 자체적으로 시크릿을 항상 메모리에만 저장하게 되고 물리 저장소에는 기록되지 않도록 처리한다.
 - 마스터 노드의 etcd에는 시크릿을 암호화되지 않는 형식으로 저장하므로 시크릿에 저장한 민감한 데이터를 보호하려면 마스터 노드를 보호해야 한다.(쿠버네티스 1.7 이하에서만, 그 이후로는 암호화된 형태로 저장함.)

#### 시크릿 vs 컨피그맵 어떤것을 사용해야 할지에 대한 기준
 - 민감하지 않고, 일반 설정 데이터는 컨피그맵을 사용하라.
 - 본질적으로 민감한 데이터는 시크릿을 사용해 키 아래에 보관하는 것이 필요하다.
 - 민감한 데이터와 그렇지 않는 데이터를 모두 가지고 있는 경우 해당 파일은 시크릿 안에 저장해야 한다.

### 7.5.2 기반 토큰 시크릿
 - 모든 파드에는 sercret 볼륨이 자동으로 연결되어 있다.

``` yaml
Containers:        # default-token (Secret) 마운트
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-xkr6d (ro)

Volumes:
  default-token-xkr6d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-xkr6d
    Optional:    false
```

``` sh
# pod 정보 조회
kubectl describe po kubia-8pw7z

# 시크릿 리소스 조회
kubectl get secrets

# 시크릿 정보 조회
kubectl describe secrets
```

 - 시크릿이 갖고 있는 3가지 항목(ca.crt, namespace, token)은 파드 안에서 쿠버네티스 API 서버와 통신할 때 필요한 것이다.
 - 기본적으로 default-token 시크릿은 모든 컨테이너에 마운트된다.
 - 파드 스펙 안에 auto mountService-AccountToken 필드 값을 false로 지정하거나 파드가 사용하는 서비스 어카운트를 false로 지정해 비활성화할 수 있다.

<img width="523" alt="스크린샷 2020-08-23 오후 5 37 00" src="https://user-images.githubusercontent.com/6982740/90974536-44fc5080-e567-11ea-9dd1-0b77f396ae57.png">


### 7.5.3 시크릿 생성
```  sh
# 인증서와 개인키 생성
openssl genrsa -out https.key 2048
openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.kubia-example.com

# secret 생성 ( fortune-https 이름을 가진 generic 시크릿을 생성 )
kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
```

#### 시크릿의 유형
 - docker-registry ( 도커 레지스트리 사용을 위한)
 - tls ( TLS 통신을 위한)
 - generic (일반적인 상황)

### 7.5.4 컨피그맵과 시크릿 비교
 - 시크릿 항목의 내용은 base64 인코딩 문자열로 표시되고, 컨피그맵의 내용은 일반 텍스트로 표시된다.

``` sh
# 시크릿 조회
kubectl get secret fortune-https -o yaml

# 컨피그맵 조회
kubectl get configmap fortune-config -o yaml
```

#### 바이너리 데이터 시크릿 사용
 - base64 인코딩을 사용하는 이유는 일반 텍스트 뿐만 아니라 바이너리 값도 담을 수 있기 때문이다.
 - 민감하지 않은 데이터도 시크릿을 사용할수 있지만 시크릿의 최대 크기는 1MB로 제한된다.


#### stringData 필드 소개
 - 쿠버네티스는 시크릿의 값을 stringData 필드로 설정할 수 있게 해준다.
 - stringData 필드는 쓰기 전용이다.(값을 설정할 때만 사용 가능)

``` yaml
apiVersion: v1
kind: Secret
stringData                # 바이너리 데이터가 아닌 시크릿 데이터에 사용할 수 있다.
  foo: plain text         # "plain text"는 base64 인코딩되지 않는다.
data:
   https.cert: ...
   https.key: ...
```

#### 파드에서 시크릿 항목 읽기
 - secret 볼륨을 통해 시크릿을 컨테이너에 노출하면, 시크릿 항목의 값이 일반 텍스트인지 바이너리 데이터인지에 관계 없이 실제 형식으로 디코딩돼 파일에 기록된다.


### 7.5.5 파드에서 시크릿 사용
 - configmap의 nignx 설정에 https 인증서와 개인키를 추가해주고 시크릿을 파드에 마운트
 - 참고 : 시크릿도 defaultMode 속성을 통해 볼륨에 노출된 파일 권한을 지정할 수 있음

#### fortune-https 시크릿을 파드에 마운트
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune-https
spec:
  containers:
  - image: luksa/fortune:env
    name: html-generator
    env:
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: fortune-config
          key: sleep-interval
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    - name: config
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs                       # nginx 서버가 인증서와 키를 /etc/nginx/certs에서 읽을수 있도록 해당 위치에 secret 마운트
      mountPath: /etc/nginx/certs/
      readOnly: true
    ports:
    - containerPort: 80
    - containerPort: 443
  volumes:
  - name: html
    emptyDir: {}
  - name: config
    configMap:
      name: fortune-config
      items:
      - key: my-nginx-config.conf
        path: https.conf
  - name: certs                       # 시크릿 볼륨 정의
    secret:
      secretName: fortune-https
```

<img width="590" alt="스크린샷 2020-08-23 오후 5 50 29" src="https://user-images.githubusercontent.com/6982740/90974768-2434fa80-e569-11ea-8e2d-57792e1aacd2.png">

#### nginx가 시크릿의 인증서와 키를 사용하는지 테스트
``` sh
# https 파드 생성
kubectl create -f fortune-pod-https.yaml

# port forward
kubectl port-forward fortune-https 8443:443 &

# curl call
curl  https://localhost:8443 -k -v
```

#### 시크릿 볼륨을 메모리에 저장하는 이유
 - 시크릿 볼륨은 시크릿 파일을 저장하는 데 인메모리 파일시스템(tmpfs)을 사용한다.
 - tmpfs를 사용하는 이유는 민감한 데이터를 노출시킬 수도 있는 디스크에 저장하지 않기 위해서이다.

``` sh
# 컨테이너 확인
kubectl exec fortune-https -c web-server -- mount | grep certs

tmpfs on /etc/nginx/certs type tmpfs (ro,relatime)
```

#### 환경변수로 시크릿 항목 노출
 - configMapKeyRef 대신 secretKeyRef를 사용해 컨피그맵과 유사한 방식으로 참조가 가능하다.
 - 시크릿을 환경변수로 노출할 수 있게 해주기는 하지만, 이 기능 사용은 권장하지는 않는다.
 - 앱에서 일반적으로 오류 보고서에 환경변수를 기록하거나 로그에 환경변수를 남겨 의도치 않게 시크릿이 노출될 가능성이 있다.
 - 또한 자식 프로세스는 부모 프로세스의 모든 환경변수를 상속받는데, 앱이 타사(third-party) 바이너리를 실행할 경우 시크릿 데이터를 어떻게 사용하는지 알 수 있는 방법이 없다.

``` yaml
    env:                               # 변수는 시크릿 항목에서 설정
    - name: FOO_SECRET
      valueFrom: "30"
         secretKeyRef:
            name: fortune-https        # 시크릿 이름 지정
            key: foo                   # 시크릿의 키 이름

```

### 7.5.6 이미지를 가져올 때 사용하는 시크릿 이해
 - 쿠버네티스에서 자격증명을 전달하는것이 필요할때가 있다.(프라이빗 컨테이너 이미지 레지스트리)

#### 도커 허브에서 프라이빗 이미지 사용
 - 도커 레지스트리 자격증명을 가진 시크릿 생성
 - 파드 매니페스트 안에 imagePullSecrets 필드에 해당 시크릿 참조

#### 도커 레지스트리 인증을 위한 시크릿 생성
``` yaml
# 도커 레지스트리용 시크릿 생성
kubectl create secret docker-registry mydockerhubsecret --docker-username=myusername --docker-password=mypassword --docker-email=my.email@providercom
```

#### 파드 정의에서 도커 레지스트리 시크릿 사용
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:                 # 프라이빗 이미지 레지스트리에서 이미지를 가져올 수 있도록 설정
  - name: mydockerhubsecret
  containers:
  - image: username/private:tag
    name: main
```

#### 모든 파드에서 이미지를 가져올 때 사용할 시크릿을 모두 지정할 필요는 없다.
 - 이미지를 가져올 때 사용할 시크릿을 서비스어카운트에 추가해 모든 파드에 자동으로 추가되도록 할수도 있다.(12장)

## 7.6 요약
 - 컨테이너 이미지에 정의된 기본 명령어를 파드 정의 안에 재정의
 - 주 컨테이너 프로세스에 명령줄 인자 전달
 - 컨테이너에서 사용할 환경변수 설정
 - 파드 사양에서 설정을 분리해 컨피그맵 안에 넣기
 - 민감한 데이터를 시크릿 안에 넣고 컨테이너에 안전하게 전달
 - docker-registry 시크릿을 만들고 프라이빗 이미지 레지스트리에서 이미지를 가져올 때 사용

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
