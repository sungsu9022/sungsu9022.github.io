---
title: "[스파크 완벽 가이드] 4. 구조적 API 개요"
author: sungsu park
date: 2022-10-29 18:22:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 4. 구조적 API 개요
## 4.1 DataFrame과 Dataset
- 2가지 모두 잘 정의된 로우와 컬럼을 가지는 분산 테이블 형태의 컬렉션.
- 각 컬럼은 다른 컬럼과 동일한 수의 로우를 가져야 함.(값이 없는 경우 null일수는 있음)

## 4.2 스미카
- DataFrame의 컬럼명과 데이터 타입을 정의
- 스키마는 데이터 소스로부터 얻거나 직접 정의할 수 있음.

## 4.3 스파크의 구조적 데이터 타입 개요
- 스파크는 하나의 프로그래밍 언어로 봐도 좋음.
- 실행계획 수립과 처리에 사용하는 자체 데이터 타입 정보를 가지고 있는 카탈리스트 엔진을 사용함.

``` scala
val df = spark.range(500).toDF("number")
df.select(df.col("number") + 10)
```

- 이런 덧셈이 가능한 이유는 카탈리스트 엔진에서 스파크의 데이터 타입으로 변환해 명령을 처리하기 때문이다.

### 4.3.1 DataFrame과 Dataset 비교
- DataFrame : 비타입형  / Dataset : 타입형
- 실제 코드레벨에서 DataFrame에도 데이터 타입이 있으나, 스키마에 명시된 타입의 일치 여부는 런타임이 되어야 확인함.
- 반면, Dataset은 스키마에 명시된 데이터 타입의 일치 여부를 컴파일 타임에 확인함.(Java / Scala에서만 지원)
- 중요한 점은 DataFrame을 사용하면 스파크의 최적화된 내부 포맷을 사용할수 있다는 사실이고, 이를 사용하면 스파크가 지원하는 어떤 언어의 API를 사용하더라도 동일한 효과와 효율성을 얻을 수 있음.

### 4.3.2 컬럼
- 정수형, 문자열 같은 단순 데이터 타입과 배열이나 맵 같은 복합 데이터 타입, null 값 표현

### 4.3.3 로우
- 데이터 레코드, DataFrame의 레코드는 Row 타입으로 구성.
- 로우는 SQL, RDD, 데이터 소스에서 얻거나 직접 만들수 있음.

``` scala
spark.range(2).toDF().collect()
```

### 4.3.4 스파크 데이터 타입
- 스파크는 내부 데이터 타입을 가지고 있음.
- 실제 langauge 데이터 타입과의 맵핑을 통해 처리할 수 있음.

<img width="567" alt="스크린샷 2022-10-30 오후 6 12 09" src="https://user-images.githubusercontent.com/6982740/198871060-919c2151-049f-41bc-bf03-1c957bcd5c6f.png">

## 4.4 구조적 API의 실행 과정
- 스파크 코드가 클러스터에서 실제 처리되는 과정을 설명

### 처리 과정
1. DataFrame/DataSet/SQL을 이용해 코드 작성
2. 정상적인 코드라면 스파크가 논리적 실행 계획 변환
3. 논리적 실행 계획을 물리적 실행 계획으로 변환, 추가적인 최적화 확인
4. 클러스터에서 물리적 실행계획(RDD 처리) 실행

### 카탈리스트 옵티마이저
- 코드를 넘겨 받아 실행 계획을 생성하는 역할

<img width="561" alt="스크린샷 2022-10-30 오후 6 14 34" src="https://user-images.githubusercontent.com/6982740/198871159-223144ab-9cb9-416f-8ad0-247cd2c42940.png">

### 4.4.1 논리적 실행 계획
- 추상적 트랜스포메이션만 포함되는 단계
- 이 단계에서는 드라이버/익스큐터 정보를 고려하지 않음.
  -먼저 검증 없이 논리적 실행 계획으로 변환한 뒤 카탈로그, 저장소, DataFrame 정보를 활용하여 검증된 논리적 실행계획을 생성.
- 이 논리적 실행 계획에 조건절 푸시 다운 등의 최적화 단계를 거쳐서 최종 논리적 실행 계획이 만들어진다.

<img width="557" alt="스크린샷 2022-10-30 오후 6 15 27" src="https://user-images.githubusercontent.com/6982740/198871185-7aeb526b-ee28-43de-9af4-f98a78f769b3.png">


### 4.4.2 물리적 실행 계획
- 이어서 스파크 실행 계획이라고 불리는 물리적 실행 계획이 생성됨.
- 클러스터 환경에서 실행하는 방법에 대한 정의
- 물리적 실행 전략을 생성하고, 비용 모델을 이용해 비교 후 최적의 전략을 선택함.(테이블 크기, 팣티션 수 등 물리적 속성 고려)
- 일련의 RDD와 트랜스포메이션으로 컴파일되어 실행됨.

<img width="560" alt="스크린샷 2022-10-30 오후 6 19 21" src="https://user-images.githubusercontent.com/6982740/198871306-eba9ccb7-43d2-417a-b884-58eec1092cd1.png">

### 4.4.3 실행
- 런타임에 전체 태스크나 스테이지를 제거할 수 있는 자바 바이트 코드를 생성해 추가적인 최적화를 수행하고, 결과를 사용자에게 반환


## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
