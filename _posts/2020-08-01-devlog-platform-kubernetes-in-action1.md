---
title: "[kubernetes-in-action] 1. 쿠버네티스 소개"
author: sungsu park
date: 2020-08-01 16:34:00 +0800
categories: [DevLog, kubernetes]
tags: [Infra, kubernetes, kubernetes-in-action]
---

# 1. 쿠버네티스 소개
## 쿠버네티스 등장 배경
> 거대한 모놀리스 레거시 애플리케이션은 점차 마이크로 서비스라는 독립적으로 실행되는 더 작은 구성 요소로 세분화되고 있다.
> 마이크로 서비스는 서루 분리돼 있기 떄문에 개별적으로 개발, 배포, 업데이트, 확장할 수 있다. 이로써 오늘날 급변하는 비즈니스 요구사항을 충족시킬 만큼 신속하게 자주 구성 요소를 변경할 수 있게 되었다.
> 하지만 배포 가능한 구성 요소 수가 많아지고 데이터 센터의 규모가 커지면서 전체 시스템을 원활하게 구성, 관리 유지하는 일이 점점 더 어려워졌다.
> 이런 구성 요소의 서버 배포를 자동으로 스케줄링하고 구성, 관리, 장애 처리를 포함하는 자동화가 필요한데, 이것이 바로 "쿠버네티스"가 등장한 이유이다.

## 쿠버네티스 어원
> 쿠버네티스는 조종사, 조타수(선박의 핸들을 잡고 있는 사람)를 뜻하는 그리스어

## 쿠버네티스
> 쿠버네티스는 하드웨어 인프라를 추상화하고 데이터 센터 전체를 하나의 거대한 컴퓨팅 리소스로 제공한다. 실제 세세한 서버 정보를 알 필요 없이 애플리케이션 구성요소를 배포하고 실행할 수 있다.
> 쿠버네티스를 이용하여 하드웨어에서 실행되는 수만 개의 애플리케이션을 일일이 알 필요가 없게 되었다.

## 1.1 쿠버네티스와 같은 시스템이 필요한 이유
### 1.1.1 모놀리스 애플리케이션에서 마이크로 서비스 전환
 - 모놀리스 애플리케이션은 앱을 실행하는데 충분한 리소스를 제공할 수 있는 소수의 강력한 서버가 필요하다.
 - 여기서 시스템의 증가하는 부하를 처리할수 있는 방법은 Scale up과 Scale out이 있는데..

#### 수직 확장(scale up)
 - 시스템의 증가하는 부하를 처리하려고 하면 CPU, 메모리 등을 추가해 서버를 수직 확장(scale up)하거나 서버를 추가하는 방법
 - 비교적 비용이 많이 들고 실제 확장에 한계가 있음.

#### 수평 확장(scale out)
 - 애플리케이션의 복사본을 실행해 전체 시스템을 수평 확장(scale out)하는 방법
 - 상대적으로 저렴하지만, 애플리케이션 코드의 큰 변경이 필요할 수도 있고, 항상 가능하지 않음.(예를 들면 RDBMS 같은 경우에 scale out은 불가하고, HA 구성을 위해서는 다른 방식으로 사용)

#### 마이크로서비스로 애플리케이션 분할
 - 하나의 포르세스였던것을 독립적인 프로세스로 나누어서 실행하고, 잘 정의된 API로 상호 통신한다.(Resetful API를 제공하는 HTTP 또는  AMQP와 같은 비동기 프로토콜)

#### 마이크로서비스 확장
 - 전체 시스템을 함께 확장해야 하는 모놀리스 시스템과 달리 마이크로 서비스 확장은 서비스별로 수행되므로 리소스가 더 필요한 서비스만 별도로 확장할 수 있으며 다른 서비스는 그대로 둬도 된다.

#### 마이크로서비스 배포
 - 마이크로서비스에도 단점이 있는데, 구성요소가 많아지면 배포 조합의 수 뿐만 아니라 구성 요소 간의 상호 종속성 수가 훨씬 더 많아지므로 배포 관련 결정이 점점 더 어려워질 수 있음.
 - 마이크로서비스는 여러 개가 서로 함께 작업을 수행하므로 서로를 찾아 통신해야 하는데, 서비스 수가 증가함에 따라 특히 서버 장애 상황에서  해야 할 일을 생각해봤을 때 전체 아키텍쳐 구성의 어려움과 오류 발생 가능성이 높아진다.
 - 또한 실행 호출을 디버깅하고 추적하기 어려운 단점이 있을수 있는데 Zipkin과 같은 분산 추적 시스템으로 해결은 가능함.

#### 환경 요구 사항의 다양성
 - 애플리케이션이 서로 다른 버전의 동일한 라이브러리를 필요로 하는 경우 애플리케이션 구성 요소간 종속성의 차이는 불가피하다.
 - 동일한 호스트에 배포해야 하는 구성 요소 수가 많을수록 모든 요구사항을 충족시키려 모든 종속성을 관리하기가 더 어려워진다.

<img width="541" alt="스크린샷 2020-08-01 오후 9 27 02" src="https://user-images.githubusercontent.com/6982740/89101745-c287e180-d43d-11ea-8d26-891ef455f357.png">

### 1.1.2 애플리케이션에 일관된 환경 제공
 - 애플리케이션을 실행하는 환경이 다른것은 사실 심각한 문제 중에 하나이다.
 - 개발과 프로덕션 환겨 사이의 큰 차이 뿐만 아니라 각 프로덕션 머신간에도 차이가 있다.
 - 이런 차이는 하드웨어에서 운영체제, 각 시스템에서 사용 가능한 라이브러리에 이르기까지 다양하다.
 - 프로덕션 환경에서만 나타나는 문제를 줄이려면 애플리케이션 개발과 프로덕션이 정확히 동일한 환경에서 실행돼 운영체제, 라이브러리, 시스템 구성, 네트워킹 환경, 기타 모든 것이 동일한 환경을 만들 수 있다면 사실 이상적이다.

### 1.1.3 지속적인 배포로 전환 : 데브옵스와 노옵스
 - 과거 개발 팀의 업무는 애플리케이션을 만들고 이를 배포(CD)하고 관리하며 계속 운영하는 운영팀에 넘기는것(?, 회사마다 다름)이었다고 한다.
 - 개발팀이 애플리케이션을 배포하고 관리하는 것이 모두가 낫다고 생각한다.
 - 개발자, 품질보증(QA), 운영팀이 전체 프로세스에서 협업해야 하는데 이를 "데브옵스"라고 한다.

#### 개발자와 시스템 관리자 각자가 최고로 잘하는 것을 하게 하는것!
 - 하드웨어 인프라를 전혀 알지 못하더라도 관리자를 거치지 않고 개발자는 애플리케이션을 직접 배포할 수 있다. 이는 가장 이상적인 방법이며, 노옵스(NoOps)라고 부른다,
 - 쿠버네티스를 사용하면 이런것들을 해결할 수 있음.
 - 하드웨어를 추상화하고 이를 애플리케이션 배포, 실행을 위한 플랫폼으로 제공함으로써,
    - 개발자는 시스템 관리자의 도움 없이도 애플리케이션을 구성, 배포할 수 있으며, 시스템 관리자는 실제 실행되는 애플리케이션을 알 필요 없이 인프라를 유지하고 운영하는 데 집중할 수 있다.

## 1.2 컨테이너 기술 소개
> 쿠버네티스는 애플리케이션을 격리하는 기능을 제공하기 위해 리눅스 컨테이너 기술을 사용한다.
> 쿠버네티스 자체를 깊이 파고들기 전에 먼저 컨테이너의 기본에 익숙해져야 한다.
> 또한 도커나 rkt(rock-it)과 같은 컨테이너 기술이 어떤 문제를 해결하는지 이해해야 한다.

### 1.2.1 컨테이너 이해
 - 애플리케이션이 더 작은 수의 커다란 구성요소로만 이뤄진 경우 구성요소에 전용 가상머신을 제공하고 고유한 운영체제 인스턴스를를 제공해 환경을 격리할수 있다.
 - 하지만 MSA로 인해 구성요소가 작아지면서 숫자가 많아지기 시작하면 하뒈어 리소스 낭비가 생길 수 있음.
 - 일반적으로 각각의 가상머신을 개별적으로 구성하고 관리해야 해서 시스템 관리자의 작업량도 상당히 증가한다.

#### 리눅스 컨테이너 기술로 구성 요소 격리
 - 가상머신을 사용해 각 마이크로서비스의 환경을 격리하는 대신 개발자들은 리눅스 컨테이너 기술로 눈을 돌림
 - 컨테이너 기술은 가상머신과 유사하게 서로 격리하지만 오버헤드가 훨씬 적음.
 - 컨테이너에서 실행되는 프로세스는 다른 모든 프로세스와 마찬가지로 호스트 운영체제 내에서 실행된다.(가상머신은 별도의 운영체제에서 실행됨)

#### 컨테이너와 가상머신 비교
##### 컨테이너
 - 컨테이너는 훨씬더 가벼워서 동일한 하드웨어 스펙으로 더 많은 수의 소프트웨어 구성 요소를 실행할 수 있음.
 - 호스트 OS에서 실행되는 하나의 격리된 프로세스일뿐, 앱이 소비하는 리소스만 소비하고 추가 프로세스의 오버헤드가 없음.
 - 호스트 OS에서 실행되는 동일한 커널에서 시스템 콜을 사용한다.
 - 어떠한 종류의 가상화도 필요없음.
 - 각각의 컨테이너가 모두 동일한 커널을 호출함으로 보안 위협이 발생할 수 있다.
 - 컨테이너를 실행할때 VM처럼 부팅할 필요가 없고, 즉시 프로세스를 실행시킬 수 있음.

##### 가상머신
 - 구성 요소 프로세스 뿐만 아니라 시스템 프로세스를 실행해야 하기 때문에 추가 컴퓨팅 리소스가 필요하다.
 - 호스트에 가상머신 3개를 실행하면 3개의 완전히 분리된 운영체제가 실행되고 동일한 하드웨어를 공유한다.
 - 시스템콜을 하이퍼 바이저를 통해 받아서 수행
 - OS를 필요로 하는 하이퍼바이저 타입이 있고, 호스트 타입을 필요로 하지 않는 타입이 있음.
 - 독립적인 커널을 호출하므로 보안 위험이 컨테이너에 비해 적다.


#### 컨테이너 격리를 가능하게 하는 매커니즘
 - 첫번째는 리눅스 네임스페이스로 각 프로세스가 시스템(파일, 프로세스, 네트워크 인터페이스, ㅎ스트 이름 등)에 대한 독립된 뷰만 볼 수 있도록 하는것
 - 두번째는 리눅스 컨트롤 그룹(cgroups)으로 프로세스가 사용할 수 있는 리소스(CPU, 메모리, 네트워크 대역폭 등)의 양을 제한하는 것

#### 네임 스페이스의 종류
- 마운트(mnt)
- 프로세스 ID(pid)
- 네트워크(net)
- 프로세스간 통신(ipc)
 - 호스트와 도메인 이름(uts - Unix Time Sharing)
 - 사용자 ID(user)

#### 리눅스 네임스페이스로 프로세스 격리
 - 여러 종류의 네임스페이스가 있기 때문에 프로세스는 하나의 네임스페이스에만 속하는 것이 아니라 여러 네임스페이스에 속할 수 있다.
 - 각 네임스페이스는 특정 리소스 그룹을 격리하는데 사용됨.
    - 2개의 서로 다른 UTS 네임스페이스를 프로세스에 각각 지정하면 서로 다른 로컬 호스트 이름을 보게할 수 도 있음.(두 프로세스는 마치 두 개의 다른 시스템에서 실행중인것처럼 보이게 할 수 있음.)
 - 각 컨테이너는 고유한 네트워크 네임스페이슬르 사용하므로 각 컨테이너는 고유한 네트워크 인터페이스 세트를 보수 있음.

#### 프로세스의 가용 리소스 제한
 - 프로세스의 리소스 사용을 제한하는 리눅스 커널 기능인 cgroups으로 이뤄진다.

### 1.2.2 도커 컨테이너 플랫폼 소개
 - 도커는 컨테이너를 여러 시스템에 쉽게 이식 가능하게 하는 최초의 컨테이너 시스템
 - 앱, 라이브러리, 여러 종속성, 심지어 전체 OS 파일시스템까지도 도커를 실행하는 다른 컴퓨터에 애플리케ㅣ션을 프로비저닝하는 데 사용할 수 있다.
 - 도커로 패키징된 앱을 실행하면 함께 제공된 파일 시스템 내용을 정확하게 볼수 있다.
- 앱은 실행중인 서버의 내용은 볼 수 없으므로 서버에 개발 컴퓨터와 다른 설치 라이브러리가 설치돼 있는지는 중요하지 않다.
 - 가상머신에 운영체제를 설치하고 그 안에 앱을 설치한 다음 가상 머신 이미지를 배포하고 실행하는 가상머신 이미지를 만드는것과 유사함.
 - 도커 기반 컨테이너 이미지와 가상머신 이미지의 큰 차이점은 컨테이너 이미지가 여러 이미지에서 공유되고 재사용될 수 있는 레이러로 구성되어 있다는것
    - 동일한 레이러르 포함하는 다른 컨테이너 이미지를 실행할 때 다른 레이어에서 이미 다운로드된 경우 이미지의 특정 레이어만 다운로드 하면됨.

#### 도커 개념 이해
> 도커는 앱을 패키징, 배포, 실행하기 위한 플랫폼
> 전체 환경과 함꼐 패키지화할 수 있음.
> 도커를 사용하여 패키지를 중앙 저장소로 전송할 수 있고, 이를 도커를 실행하는 모든 컴퓨터에 전송할 수 있음.

##### 이미지
 - 애플리케션과 해당 환경을 패키지화한 것
 - 앱에서 사용할 수 있는 파일시스템과 이미지가 실행될때 실행돼야 하는 실행파일 경로와 같은 메타데이터를 포함한다.

##### 레지스트리
 - 도커 이미지를 저장하고 다른 사람이나 컴퓨터 간에해당 이미지를 쉽게 공유할 수 있는 저장소 ( push / pull)

##### 컨테이너
 - 실행중인 컨테이너는 도커를 실행하는 호스트에서 실행되는 프로세스이지만 호스트와 호스트에서 실행중인 다른 프로세스와는 완전히 격리돼 있음.
 - 리소스 사용이 제한돼 있으므로 할당된 리소스의 양(CPU, RAM 등)만 엑세스하고 사용할 수 있다.

#### 도커 이미지의 빌드 및 배포, 실행

<img width="718" alt="스크린샷 2020-08-02 오후 3 43 31" src="https://user-images.githubusercontent.com/6982740/89117192-efd49e00-d4d6-11ea-9bbe-e2b1b727010e.png">

#### 가상 머신과 도커 컨테이너 비교

![스크린샷 2020-08-02 오후 3 49 44](https://user-images.githubusercontent.com/6982740/89117324-cff1aa00-d4d7-11ea-9293-1ee1543dcb2e.png)

 - 가상머신에서 실행될 떄와 두 개의 별도 컨테이너로 실행될 때 앱 A,B가 동일한 바이너리, 라이브러리에 접근할 수 있음.
 - 컨테이너는 격리된 자체 파일시스템이 있는데 이떄 같은 파일을 공유할 수 있음.

#### 이미지 레이어의 이해
 - 모든 도커 이미지는 다른 이미지 위에 빌드되고, 2개의 다른 이미지는 동일한 부모 이미지를 사용할 수 있으므로, 서로 정확히 동일한 레이어가 포함될 수 있음.
 - 레이어는 배포를 효율적으로 할 뿐만 아니라 이미지의 스토리지 공간을 줄이는 데에도 도움이 된다.
   - 각 레이어는 동일 호스트에서 한번만 저장됨.
   - 동일한 기본 레이러를 기반으로 한 2개의 이미지에서 생성한 2개의 컨테이너는 동일한 파일을 읽을 수 있지만, 그중 하나가 해당 파일을 덮어쓰면 다른 컨테이너에서는 그 변경 사항을 바라보지 않는다.
   - 파일을 공유하더라도 여전히 서로 격리돼 있는데 이것은 컨테이너 이미지 레이어가 읽기 전용이기 때문
   - 컨테이너가 실행될때 이미지 레이어 위에 새로운 쓰기 가능한 레이어가 만들어진다.

#### 컨테이너 이미지의 제한적인 이식성 이해
 - 이론적으로 컨테이너 이미지는 도커를 실행하는 모든 리눅스 시스템에서 실행될 수 있지만, 호스트에서 실행되는 모든 컨테이너가 호스트의 리눅스 커널을 사용한다는 사실과 관련해 주의해야 한다.
 - 컨테이너화된 앱이 특정 커널 버전이 필요하다면 모든 시스템에서 작동하지 않을수 있음.
 - 머신이 다른 버전의 리눅스 커널로 실행되거나 동일한 커널 모듈을 사용할 수 없는 경우에는 앱이 실행될 수 없다.
 - 컨테이너는 가상머신에 비해 훨씬 가볍지만 컨테이너 내부에서 실행되는 앱은 일정한 제약이 있따.
 - 하드웨어 아키텍처용으로 만들어진 컨테이너화된 앱은 해당 아키텍처 시스템에서만 실행될 수 있다는 점을 분명히 해야 한다.

### 1.2.3 도커의 대안으로 rkt 소개
 - rkt는 현시점 deprecated됨.
 - 도커와 마찬가지로 컨테이너를 실행하기 위한 플랫폼
 - 2018년 이후로 업데이트 없음.

## 1.3 쿠버네티스 소개
### 1.3.1 쿠버네티스의 기원
 - 구글은 보그(Borg, 이후 오메가(Omega)로 바뀐 시스템)라는 내부 시스템을 개발해 애플리케이션 개발자와 시스템 관리자가 수천 개의 애플리케이션과 서비스를 관리하는데 도움을 주었다.
 - 개발과 관리를 단순화할 뿐만 아니라 인프라 활용률을 크게 높일 수 있었다.
 - 2014년 보그, 오메가, 기타 내부 구글 시스템으로 얻은 경험을 기반으로 하는 오픈소스 시스템인 쿠버네티스를 출시

### 1.3.2 넓은 시각으로 쿠버네티스 바라보기
 - 쿠버네티스는 컨테이너화된 애플리케이션을 쉽게 배포하고 관리할 수 있게 해주는 소프트웨어 시스템
 - 애플리케이션은 컨테이너에서 실행되므로 동일한 서버에서 실행되는 다른 앱에 영향을 미치지 않으며, 이는 동일한 하드웨어에서 완전히 다른 조직의 앱을 실행할때 매우 중요하다.
 - 호스팅된 앱을 완전히 격리하면서 하드웨어를 최대한 활용한다.
 - 모든 노드가 마치 하나의 거대한 컴퓨터인 것처럼 수천대의 컴퓨터 노드에서 스포트웨어 애플리케이션을 실행할 수 있다.

#### 쿠버네티스 핵심 이해
 - 시스템은 마스터 노드와 여러 워커 노드로 구성
 - 구성요소가 어떤 노드에 배포되든지 개발자나 시스템 관리자에게 중요하지 않다.
 - 개발자는 앱 디스크립터를 쿠버네티스 마스터에게 게시하면 쿠버네티스는 해당 앱을 워커 노드 클러스터에 배포한다.

<img width="656" alt="스크린샷 2020-08-02 오후 5 35 38" src="https://user-images.githubusercontent.com/6982740/89119118-9a07f200-d4e6-11ea-9e80-b007cc72370c.png">

#### 개발자가 앱 핵심 기능에 집중할 수 있도록 지원
 - 쿠버네티스는 마치 클러스터의 운영체제로 생각할 수 있음.
 - 서비스 디스커버리 ,스케일링, 로드밸런싱, 자가 치유, 리더 선출 같은 것들을 포함하여 지원한다.

#### 운영 팀이 효과적으로 리소스를 활용할 수 있도록 지원
 - 각 앱들은 어떤 노드에서 실행되든 상관이 없기 때문에 쿠버네티스는 언제든지 앱을 재배치하고, 조합함으로써 리소스를 수동 스케줄링보다 훨씬 잘 활용할 수 있다.

### 1.3.3 쿠버네티스 클러스터 아키텍쳐 이해
 - 하드웨어 수준에서 쿠버네티스 클러스터는 여러 노드로 구성되며, 2가지 유형으로 나눌 수 있다.
> 마스터 노드 : 전체 쿠버네티스 시스템을 제어하고 관리하는 쿠버네티스 컨트롤 플레인을 실행
> 워커 노드 : 실제 배포되는 컨테이너 애플리케이션을 실행

<img width="643" alt="스크린샷 2020-08-02 오후 5 41 52" src="https://user-images.githubusercontent.com/6982740/89119225-772a0d80-d4e7-11ea-99fd-3dbd04beb866.png">

#### 컨트롤 플레인(Control Plane)
 - 클러스터를 제어하고 작동시키는 역할
 - 하나의 마스터 노드에서 실행하거나, 여러 노드로 분할되고 복제하여 고가용성을 보장할 수 있는 여러 구성 요소로 구성할 수 있다.
 - API서버는 사용자, 컨트롤 플레인 구성 요소와 통신
 - 스케줄러는 앱의 배포를 담당(앱의 배포 가능한 각 구성요소를 워크 노드에 할당)
 - 컨트롤러 매니저는 구성요소 복제본, 워커 노드 추적, 노드 장애 처리 등과 같은 클러스터단의 기능을 수행
 - Etcd는 클러스터 구성을 지속적으로 저장하는 신뢰할 수 있는 분산 데이터 저장소

#### 노드(Worker Node)
 - 컨테이너화된 애플리케이션을 실행하는 시스템
 - 컨테이너 런타임은 도커, rkt 또는 다른 컨테이너 런타임이 될 수 있다.
 - kebelet은 API 서버와 통힌하고 노드의 컨테이너를 관리한다.
 - kebe-proxy(쿠버네티스 서비스 프록시)는 앱 구성요소간 네트워크 트래픽을 로드밸런싱한다.

### 1.3.4 쿠버네티스에서 애플리케이션 실행
 - 애플리케이션을 하나 이상의 컨테이너 이미지로 패키징하고 해당 이미지를 레지스트리로 푸시한 다음에 쿠버네티스 API 서버에 앱 디스크립션을 게시해야 한다.

#### 앱 디스크립션에 포함되는 내용들
 - 컨테이너 이미지
 - 앱 구성요소가 포함된 이미지
 - 해당 구성요소가 서로 통신하는 방법
 - 동일 서버에 함께 배치되어야 하는 구성 요소
 - 실행될 각 구성 요소의 본제본 수
 - 내부 또는 외부 클라이언트에 서비스를 제공하는 구성요소
 - 하나의 IP주소로 노출해 다른 구성 요소에서 검색가능하게 해야 하는 구성요소 등

#### 디스크립션으로  컨테이너를 실행하는 방법 이해
 - 1) API 서버가 앱 디스크립션을 처리할때 스케줄러는 각 컨테이너에 필요한 리소스를 계산하고 각 노드에 할당되지 않은 리소스를 기반으로 사용 가능한 워커 노드에 지정된 컨테이너를 할당한다.
 - 2) kebulet은 컨테이너 런타임(도커)에 필요한 컨테이너 이미지를 가져와 컨테이너를 실행하도록 지시한다.

#### 실행된 컨테이너 유지
 - 앱이 실행되면 쿠버네티스는 앱의 배포 상태가 사용자가 제공한 디스크립션과 일치하는지 지속적으로 확인한다.
    - 예를 들어 5개의 웹 서버 인스턴스를 실행하도록 지정하면 쿠버네티스는 항상 정확히 5개의 인스턴스를 계속 실행
    - 프로세스 중단 등 인스턴스가 제대로 동작하지 않으면 자동으로 다시 시작
    - 워커 노드 전체가 종료되거나 하면 이 노드에서 실행중인 모든 컨테이너 노드를 새로 스케줄링하고, 새로 선택한 노드에서 실행한다.

#### 복제본 수 스케일링
 - 실행되는 동안 복제본 수를 늘릴지 줄일지 결정할 수 있음.
 - 최적의 본제본 수를 결정하는 작업을 쿠버네티스에게 위힘할 수 있다.
 - 쿠버네티스는 CPU부하, 메모리 사용량, 초당 요청수 등 실시간 메트릭을 기반으로 복제본 수를 자동으로 조정할 수 있다.

#### 이동한 애플리케이션에 접근하기
 - 쿠버네티스는 클라이언트가 특정 서비르르 제공하는 컨테이너를 쉽게 찾을 수 있도록 동일한 서비스를 제공하는 컨테이너를 알려주면 하나의 고정 IP주소로 모든 컨테이너를 노출하고 해당 주소를 클러스터에서 실행중인 모든 앱에 노출한다.
 - DNS로 서비스 IP를 조회할 수도 있음.
 - kube-proxy는 서비스를 제공하는 모든 컨테이너에서 서비스 연결이 로드밸런싱되도록 한다.
 - 이런 매커니즘에 의해서 컨테이너들이 클러스터 내에서 이동하더라도 컨테이너에 항상 연결할 수 있음.

### 1.3.5 쿠버네티스 사용의 장점
 - 시스템 관리자는 앱을 배포하고 실행하기 위해 아무것도 설치할 필요가 없음.
 - 개발자는 시스템 관리자의 도움 없이 즉시 앱을 실행할 수 있다.

#### 애플리케이션 배포의 단순화
 - 개발자는 클러스트를 구성하는 서버에 관해 알 필요가 전혀 없다.
 - 모든 노드는 단순히 전체 컴퓨팅 리소스일뿐, 앱에 적절한 시스템 리소를 제공할 수 있는 한 어느 서버에서 실행중인지는 신경쓰지 않아도 된다.
 - 특정 앱이 특정 종류의 하드웨어에서 실행해야 하는 경우에도 지원이 가능(예를 들면 SSD를 이용해야 하는 경우)

#### 하드웨어 활용도 높이기
 - 인프라와 애플리케이션을 분리해서 생ㅇ각할 수 있다.
 - 쿠버네티스는 요구사항에 대한 디스크립션과 노드에서 사용 가능한 리소스에 따라 앱을 실행한 가장 적합한 노드를 선택할 수 있다.
 - 쿠버네티스는 언제든지 클러스터 간에 앱이 이동할 수 있으모로 수동으로 수행하는것보다 훨씬 더 인프라를 잘 활용할 수 있다.

#### 상태 확인과 자가 치유
 - 서버 장애 발생시 언제든지 클러스터 간에 앱을 이동시킬수 있는 시스템이 갖출수 있다.
 - 쿠버네티스는 앱 구성요소와 구동중인 워커 노드를 모니터링하다가 노드 장애 발생시 자동으로 앱을 다른 노드로 스케줄링한다.

#### 오토스케일링
 - 급격한 부하 급증에 대응하기 위해 개별 앱의 부하를 운영팀이 지속적으로 모니터링할 필요가 없다.
 - 각 앱이 사용하는 리소스를 모니터링하고, 실행중인 인스턴스 수를 자동으로 조정하도록 지시할수 있다.

#### 애플리케이션 개발 단순화
 - 개발과 프로덕션 환경이 모두 동일한 환경에서 실행된다는걸 보장할 수 있어서 버그발견시 큰 효과를 얻을수 있음.
 - 또한 서비스 디스커버리와 같은 일반적으로 구현해야 하는 기능들을 구현할 필요가 없어졌다.
 - 쿠버네티스 API 서버를 직접 쿼리하면 개발자가 리더 선정 같은 복잡한 메커니즘을 구현하지 않아도 된다.
 - 새로운 버전의 앱을 출시할때 신버전이 잘못됐는지 자동으로 감지하고 즉시 롤아웃을 중지할수 있어서 신뢰성을 증가시켜 CD(continuous delivery)을 가속화할수 있음.

## 1.4 요약
 - 모놀리스 애플리케이션은 구축하기 쉽지만 시간이 지남에 따라 유지 관리가 어려워지고 때로는 확장이 불가능할수 있다.
 - 마이크로서비스 기반 애플리케이션 아키텍처는 각 구성요소의 개발을 용이하게 하지만, 하나의 시스템으로 작동하도록 배포하고 구성하기가 어렵다.
 - 리눅스 컨테이너는 가상머신과 동일한 이점을 제공하지만 훨씬 더 가볍고 하드웨어 활용도를 높일 수 있다.
 - 도커는 OS환경과 함꼐 컨테이너화된 애플리케이션을 좀 더 쉽고 빠르게 프로비저닝할 수 있도록 지원해 기존 리눅스 컨테이너 기술을 개선했다.
 - 쿠버네티스는 전체 데이터 센터를 앱 실행을 위한 컴퓨팅 리소스로 제공한다.
 - 개발자는 시스템 관리자의 도움 없이도 쿠버네티스로 앱을 배포할 수 있다.
 - 시스템 관리자는 쿠버네티스가 고장 난 노드를 자동으로 처리하도록 할수 있다.

## Reference
  - [kubernetes-in-action](https://www.manning.com/books/kubernetes-in-action)
  - [kubernetes.io](https://kubernetes.io/ko/docs/home/)