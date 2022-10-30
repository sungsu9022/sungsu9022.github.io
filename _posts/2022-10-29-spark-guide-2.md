---
title: "[스파크 완벽 가이드] 2. 스파크 간단히 살펴보기"
author: sungsu park
date: 2022-10-29 15:46:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 2. 스파크 간단히 살펴보기
## 2.1 스파크의 기본 아키텍쳐
### 2.1.1 스파크 애플리케이션
- driver 프로세스와 executor 프로세스로 구성됨.
- driver 프로세스는 zeppelin 같은 BI 툴이 될수도 있고, spring application같은것이 될수도 있음.

<img width="654" alt="스크린샷 2022-10-30 오후 4 14 40" src="https://user-images.githubusercontent.com/6982740/198866921-f04eca6a-9bdc-4187-93ee-f82ee85d87fb.png">

#### 로컬 모드
- 위 그림과 같이 클러스터 모드로도 동작할수 있으나, `로컬 모드`도 지원함.
- 드라이버, 익스큐터는 단순 프로세스 단위로 동작할 수 있음.
- 로컬 모드로 실행될 경우 단일 머신에서 스레드 형태로 실행됨.


## 2.2 스파크의 다양한 언어 API
- 스칼라, 자바, 파이썬, SQL, R 등 다양한 언어를 제공함.
- kotlin 진영에서 https://github.com/Kotlin/kotlin-spark-api 와 같이 spark-api 라이브러리를 제공하기도 하여 최근 JVM  진형 트랜드에 맞게 개발하는 것도 가능.

### SparkSession과 다른언어 API와의 관계

<img width="470" alt="스크린샷 2022-10-30 오후 4 24 31" src="https://user-images.githubusercontent.com/6982740/198867219-98441787-a323-4516-90e7-e10d35a828a6.png">

- 스파크가 사용자를 대신해 다른 언어로 작성된 코드를 JVM에서 실행될수 있는 코드로 변환하여 실행됨.


## 2.3 Spark API
- 저수준 비구조적 API : RDD
- 고수준 구조적 API, Dataset, DataFrame

## 2.4 Spark Session
- 스파크 애플리케이션은 SparkSession이라 불리는 드라이버 프로세스를 통해 제어됨.
- SparkSession은 사용자가 정의한 처리 명령을 클러스터에서 실행함.


## 2.5 DataFrame
- 가장 대표적인 구조적 API
- 테이블 데이터를 로우와 컬럼으로 단순하게 표현(스키마)

<img width="477" alt="스크린샷 2022-10-30 오후 4 32 18" src="https://user-images.githubusercontent.com/6982740/198867479-0ebff921-af48-4122-89d1-17ec1b771f35.png">

- 그림과 같이 분산된 데이터를 하나의 Dataset으로 프로세싱 할수 있음.

### 2.5.1 파티션
- 스파크는 모든 익스큐터가 병렬로 작업을 수행할수 있도록 파티션이라 불리는 chunk 단위로 데이터를 분할
- 파티션은 클러스터의 물리 머신에 존재하는 로우의 집합을 의미함.
- 파티션이 1이라면 수천개의 익스큐터가 있는 분산환경이더라도 병렬성은 1이 됨.(그 반대도 마찬가지)

## 2.6 트랜스포메이션
- 스파크의 핵심 데이터 구조는 불변성(immutable)을 가짐.
- DataFrame을 변경하는 방법을 트랜스포메이션이라 지칭.

### 2.6.1 좁은 의존성(narrow dependency)
- 하나의 입력 파티션이 하나의 출력 파티션에만 영향을 미치는 경우
- 이 케이스에서는 모든 작업들이 메모리를 통해 파이프라이닝 처리됨.

<img width="379" alt="스크린샷 2022-10-30 오후 4 37 34" src="https://user-images.githubusercontent.com/6982740/198867655-4f00a096-b088-422e-92a3-0ee1bb443ccd.png">

### 2.6.2 넓은 의존성(wide dependency)
- 하나의 입력 파티션이 여러개의 출력 파티션에 영향을 미치는 경우
- 좁은 의존성과 달리 셔플이 발생하면 이를 셔플 결과를 디스크에 저장하여 처리됨.
  - 이 최적화를 어떻게 하냐에 따라 작업의 성능에 큰 영향이 미침.

<img width="535" alt="스크린샷 2022-10-30 오후 4 38 12" src="https://user-images.githubusercontent.com/6982740/198867678-269b8812-4376-4ee9-b10a-4b8a41ff85d0.png">

### 2.6.3. 지연 연산
- 스파크는 연산명령을 즉시 처리하는 방식이 아니라 실행계획을 먼저 생성하면서  최적화가 이루어진다.

#### 조건절 푸시다운
- 하나의 로우만 가져오는 필터를 가지고있다면 실제 데이터를 1개만 읽는것이 효율적인데 이를 자동적으로 수행하고 최적화됨.

## 2.7 액션
- 실제 연산 수행을 위해서는 액션 명령을 실행해야 함.
- Java Stream API의 중단연산과 종단연산과 유사하다고 이해할수 있음.

``` java
divisBy .count()
```

## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)


