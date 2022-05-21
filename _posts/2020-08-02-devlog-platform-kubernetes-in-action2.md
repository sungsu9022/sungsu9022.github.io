---
title: "[kubernetes-in-action] 2. 도커와 쿠버네티스 첫걸음"
author: sungsu park
date: 2020-08-02 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---


# 2. 도커와 쿠버네티스 첫걸음
## 2.1 도커를 사용한 컨테이너 이미지 생성, 실행, 공유하기
### 2.1.1 Hello World 컨테이너 실행하기

``` sh
docker run busybox echp "Hello world"
```

#### 백그라운드에 일어난 동작 이해하기
 - docker run 명령을 수행했을 떄 일어나는 일들

<img width="704" alt="스크린샷 2020-08-04 오전 1 36 51" src="https://user-images.githubusercontent.com/6982740/89205651-01ab6380-d5f3-11ea-8701-617c043e453f.png">


### 2.1.2 간단한 node.js 애플리케이션 생성
 - 예제 코드 있음.

### 2.1.3 이미지를 위한 DockerFile 생성
```
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
```

### 2.1.4 컨테이너 이미지 생성
 - 도커 이미지 조회
``` sh
docker images
```

 - 도커 이미지 생성
``` sh
docker buld -t kubia .
```

### 2.1.5 컨테이너 이미지 실행
 - 컨테이너 이미지 실행

``` sh
docker run --name kubia-container -p 8080:8080 -d kubia

curl localhost:8080
```

 - 실행중인 컨테이너 조회

``` sh
docker ps
```

 - 컨테이너에 관한 추가 정보 얻기

``` sh
docker inspect kubia-contaniner

[
    {
        "Id": "d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2",
        "Created": "2020-08-03T14:04:38.901107275Z",
        "Path": "node",
        "Args": [
            "app.js"
        ],
        "State": {
            "Status": "running",
            "Running": true,
            "Paused": false,
            "Restarting": false,
            "OOMKilled": false,
            "Dead": false,
            "Pid": 2660,
            "ExitCode": 0,
            "Error": "",
            "StartedAt": "2020-08-03T14:04:39.219869415Z",
            "FinishedAt": "0001-01-01T00:00:00Z"
        },
        "Image": "sha256:5b3381f90920e1c043628736040dca33fefeff5df30a14376e0d387e51439e0d",
        "ResolvConfPath": "/var/lib/docker/containers/d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2/hostname",
        "HostsPath": "/var/lib/docker/containers/d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2/hosts",
        "LogPath": "/var/lib/docker/containers/d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2/d9e72c4f7867bab17b77faaefc0148e3ce430a9335c0578c199a66efd2ba6fc2-json.log",
        "Name": "/kubia-contaniner",
        "RestartCount": 0,
        "Driver": "overlay2",
        "Platform": "linux",
        "MountLabel": "",
        "ProcessLabel": "",
        "AppArmorProfile": "",
        "ExecIDs": null,
        "HostConfig": {
            "Binds": null,
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {
                "8080/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "8080"
                    }
                ]
            },
            "RestartPolicy": {
                "Name": "no",
                "MaximumRetryCount": 0
            },
            "AutoRemove": false,
            "VolumeDriver": "",
            "VolumesFrom": null,
            "CapAdd": null,
            "CapDrop": null,
            "Capabilities": null,
            "Dns": [],
            "DnsOptions": [],
            "DnsSearch": [],
            "ExtraHosts": null,
            "GroupAdd": null,
            "IpcMode": "private",
            "Cgroup": "",
            "Links": null,
            "OomScoreAdj": 0,
            "PidMode": "",
            "Privileged": false,
            "PublishAllPorts": false,
            "ReadonlyRootfs": false,
            "SecurityOpt": null,
            "UTSMode": "",
            "UsernsMode": "",
            "ShmSize": 67108864,
            "Runtime": "runc",
            "ConsoleSize": [
                0,
                0
            ],
            "Isolation": "",
            "CpuShares": 0,
            "Memory": 0,
            "NanoCpus": 0,
            "CgroupParent": "",
            "BlkioWeight": 0,
            "BlkioWeightDevice": [],
            "BlkioDeviceReadBps": null,
            "BlkioDeviceWriteBps": null,
            "BlkioDeviceReadIOps": null,
            "BlkioDeviceWriteIOps": null,
            "CpuPeriod": 0,
            "CpuQuota": 0,
            "CpuRealtimePeriod": 0,
            "CpuRealtimeRuntime": 0,
            "CpusetCpus": "",
            "CpusetMems": "",
            "Devices": [],
            "DeviceCgroupRules": null,
            "DeviceRequests": null,
            "KernelMemory": 0,
            "KernelMemoryTCP": 0,
            "MemoryReservation": 0,
            "MemorySwap": 0,
            "MemorySwappiness": null,
            "OomKillDisable": false,
            "PidsLimit": null,
            "Ulimits": null,
            "CpuCount": 0,
            "CpuPercent": 0,
            "IOMaximumIOps": 0,
            "IOMaximumBandwidth": 0,
            "MaskedPaths": [
                "/proc/asound",
                "/proc/acpi",
                "/proc/kcore",
                "/proc/keys",
                "/proc/latency_stats",
                "/proc/timer_list",
                "/proc/timer_stats",
                "/proc/sched_debug",
                "/proc/scsi",
                "/sys/firmware"
            ],
            "ReadonlyPaths": [
                "/proc/bus",
                "/proc/fs",
                "/proc/irq",
                "/proc/sys",
                "/proc/sysrq-trigger"
            ]
        },
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/a3a88ce5bf878e4f64d8ae9e7db231c4b6e66b32b288ef882a2f105423eb4a74-init/diff:/var/lib/docker/overlay2/64e81c4fc48f121c861dbe5cc5cb5022a2ec2f7574e82e5f8948e04c87746b81/diff:/var/lib/docker/overlay2/5d2ddb4be75ae0be6e92a332006d173a1f20b92ec8688b10d6ac01bee176c542/diff:/var/lib/docker/overlay2/1a703255cdcdd1f16f5a3bd8a4b998be442fdb25c7a45108df017f1bb6c8960f/diff:/var/lib/docker/overlay2/f65dcb07c2b440dd4da92dc5536ee68a0952023878e59b335554a5d4ca9fdc8d/diff:/var/lib/docker/overlay2/57db638f03c99ab27a1426cc52ccc8911b103a5128a81b56fee29852dafe65bd/diff:/var/lib/docker/overlay2/af3ec0c3056b0a7ce999eff9a10084409f532b44e27418e736901749621ce194/diff:/var/lib/docker/overlay2/81250fe4e2e75f440c5aa17df795342744a5e70cd0e90ee4b54f1a20b36504aa/diff:/var/lib/docker/overlay2/5f0ac8d1e4a787d1229a165e4ea65375daa78d49a0ec9ec933ca3e5a2f7f1feb/diff:/var/lib/docker/overlay2/37442cc111e0254803eb97bc76e3e04ed907fa92aeacd37c10cf4b68c9c9ed94/diff",
                "MergedDir": "/var/lib/docker/overlay2/a3a88ce5bf878e4f64d8ae9e7db231c4b6e66b32b288ef882a2f105423eb4a74/merged",
                "UpperDir": "/var/lib/docker/overlay2/a3a88ce5bf878e4f64d8ae9e7db231c4b6e66b32b288ef882a2f105423eb4a74/diff",
                "WorkDir": "/var/lib/docker/overlay2/a3a88ce5bf878e4f64d8ae9e7db231c4b6e66b32b288ef882a2f105423eb4a74/work"
            },
            "Name": "overlay2"
        },
        "Mounts": [],
        "Config": {
            "Hostname": "d9e72c4f7867",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "ExposedPorts": {
                "8080/tcp": {}
            },
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "NPM_CONFIG_LOGLEVEL=info",
                "NODE_VERSION=7.10.1",
                "YARN_VERSION=0.24.4"
            ],
            "Cmd": null,
            "Image": "kubia",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": [
                "node",
                "app.js"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "b3896d08952f17bf1357487b848f07f3479552c4ccfce4c21a3a4e4f9c0d05b7",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {
                "8080/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "8080"
                    }
                ]
            },
            "SandboxKey": "/var/run/docker/netns/b3896d08952f",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "f07d0019848fb4f71c8be009ca864596dc3ca7a270264a07b0cbc2af9e7e430a",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.2",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:02",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "316f7a8d294ea277f1380cd6eda453adc349411009ecaa7b9e622c1f0910df78",
                    "EndpointID": "f07d0019848fb4f71c8be009ca864596dc3ca7a270264a07b0cbc2af9e7e430a",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

### 2.1.6 실행중인 컨테이너 내부 탐색하기
``` sh
# 컨테이너 내부 쉘 실행
docker exec -it kubia-contaniner bash
```

### 2.1.7 컨테이너 중지와 삭제
``` sh
# 컨테이너 중지
docker stop kubia-contaniner

# 컨테이너 삭제
docker rm  kubia-contaniner
```

### 2.1.8 이미지 레지스트리에 이미지 푸시
``` sh
# 추가 태그로 이미지 태그 지정
docker tag kubia sungsu9022/kubia

# 도커 허브에 이미지 푸시
docker push sungsu9022/kubia

# 다른머신에서 이미지 실행하기
docker run -p 8080:8080 -d sungsu9022/kubia
```

## 2.2 쿠버네티스 설치하기
 - 공식문서에 minikube로 시작하는 가이드가 있음. 그게 가장 간편해보임.
 - docker desktop이 설치되어있다면 거기 옵션으로 kubernetes를 시작할수도 있음.

## 2.2.3 쿠버네티스 자동완성 설정하기
 - ...

## 2.3 쿠버네티스 앱 실행
``` sh
# 파드 실행
kubectl run kubia --image=sungsu9022/kubia --port=8080 --generator=run/v1

# 파드 조회
kubectl get pods

# 서비스 오브젝트 생성
kubectl expose rc kubia --type=LoadBalancer --name kubia-http

# 서비스 조회
kubectl get services
kubectl get svc
```

> minikube에서는 서비스 지원하지 않아 외부 포트를 통해 서비스 접근해야 함.

``` sh
# replicationcontrollers 정보 조회
kubectl get replicationcontrollers

# replicationcontrollers 수늘리기
kubectl scale rc kubia --replicas=3

# 실행중인 노드까지 표시
kubectl get pods -p wide

# 파드 세부 정보 살펴보기
kubectl describe pod kubia-5wsx8

# dashboard
minikube dashboard
```

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)
