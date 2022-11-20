---
title: "[스파크 완벽 가이드] 8. 조인"
author: sungsu park
date: 2022-11-19 17:36:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---


# [스파크 완벽 가이드] 8. 조인
## 8.1 조인 표현식
- 스파크는 왼쪽과 오른쪽 데이터셋에 있는 하나 이상의 키값을 비교하고 왼쪽 데이터셋과 오른쪽 데이터셋의 결합 여부를 결정하는 조인 표현식의 평가 결과에 따라 2개의 데이터셋을 조인합니다.
- 일반적인 RDB의 JOIN과 유사한 방식

## 8.2 조인 타입
- 내부 조인(inner join) : 왼쪽과 오른쪽 데이터셋에 키가 있는 로우를 유지
- 외부 조인(outer join) : 왼쪽이나 오른쪽 데이터셋에 키가 있는 로우를 유지
- 왼쪽 외부 조인(left outer join) : 왼쪽 데이터셋에 키가 있는 로우를 유지
- 오른쪽 외부 조인(right outer join) : 오른쪽 데이터셋에 키가 있는 로우를 유지
- 왼쪽 세미 조인(left semi join) : 왼쪽 데이터셋의 키가 오른쪽 데이터셋에 있는 경우 키가 일치하는 왼쪽 데이터셋만 유지
- 왼쪽 안티 조인(left anti join) : 왼쪽 데이터셋의 키가 오른쪽 데이터셋에 없는 경우에는 키가 일치하지 않은 왼쪽 데이터셋만 유지
- 자연 조인(natural join) : 두 데이터셋에서 동일한 이름을 가진 컬럼을 암시적(implicit)으로 결합하는 조인을 수행
- 교차 조인(cross join) 또는 카테시안 조인(cartesian join) : 왼쪽 데이터셋의 모든 로우와 오른쪽 데이터셋의 모든 로우 조합을 표시

### base data
``` scala
val person = Seq(
    (0, "Bill Chambers", 0, Seq(100)),
    (1, "Matei Zaharia", 1, Seq(500, 250, 100)),
    (2, "Michael Armbrust", 1, Seq(250, 100)))
  .toDF("id", "name", "graduate_program", "spark_status")

val graduateProgram = Seq(
    (0, "Masters", "School of Information", "UC Berkeley"),
    (2, "Masters", "EECS", "UC Berkeley"),
    (1, "Ph.D.", "EECS", "UC Berkeley"))
  .toDF("id", "degree", "department", "school")

val sparkStatus = Seq(
    (500, "Vice President"),
    (250, "PMC Member"),
    (100, "Contributor"))
  .toDF("id", "status")

person.createOrReplaceTempView("person")
graduateProgram.createOrReplaceTempView("graduateProgram")
sparkStatus.createOrReplaceTempView("sparkStatus")
```

## 8.3 내부 조인(INNER JOIN)
- 테이블에 존재하는 키를 평가하고, 결과가 참(true)인 로우만 결합

``` scala
val joinExpression = person.col("graduate_program") === graduateProgram.col("id")
val wrongJoinExpression = person.col("name") === graduateProgram.col("school")

person.join(graduateProgram, joinExpression).show()
```

``` sql
SELECT *
FROM person
JOIN graduateProgram ON person.graduate_program = graduateProgram.id
```

<img width="581" alt="스크린샷 2022-11-20 오후 4 49 12" src="https://user-images.githubusercontent.com/6982740/202891557-9d41fe69-69f2-42b3-bb18-c9408cdfea5e.png">

### joinType 지정
``` scala
var joinType = "inner"

// 3번쨰 파라미터로 joinType을 명시해줄수 있다.
person.join(graduateProgram, joinExpression, joinType).show()
```

## 8.4 외부 조인(FULL OUTER  JOIN)
- Outer Join은 DataFrame이나 테이블에 존재하는 키를 평가하여 참이나 거짓으로 평가한 로우를 조인하고, 일치하는 로우가 없다면 해당 위치를 null로 채워주는 조인 방식

``` scala
joinType = "outer"
person.join(graduateProgram, joinExpression, joinType).show()
```

``` sql
SELECT *
FROM person
FULL OUTER JOIN graduateProgram ON graduate_program = graduateProgram.id
```

<img width="589" alt="스크린샷 2022-11-20 오후 4 51 38" src="https://user-images.githubusercontent.com/6982740/202891607-8dc9d0a2-91ea-4af9-b973-dbe9fbcda2ef.png">


## 8.5 왼쪽 외부 조인(LEFT  OUTER  JOIN)
- 왼쪽 DataFrame의 모든 로우를 표시하고, 이와 일치하는 오른쪽 DataFrame 로우를 함꼐 표시해주는 조인 방식
  - 오른쪽 DataFrame에 일치하는 로우가 없다면 해당 위치는 null로 채워짐

``` scala
joinType = "left_outer"
graduateProgram.join(person, joinExpression, joinType).show()
```

``` sql
SELECT *
FROM graduateProgram
LEFT OUTER JOIN person ON person.graduate_program = graduateProgram.id
```

<img width="577" alt="스크린샷 2022-11-20 오후 4 53 29" src="https://user-images.githubusercontent.com/6982740/202891672-c84d1fd6-5470-450d-ad47-9e7348182c36.png">

## 8.6 오른쪽 외부 조인(RIGHT OUTER JOIN)
- LEFT OUTER JOIN과 드라이빙 테이블이 왼쪽이냐 오른쪽이냐만 차이가 있고 동일.


<img width="584" alt="스크린샷 2022-11-20 오후 4 54 40" src="https://user-images.githubusercontent.com/6982740/202891695-b1d28b5d-24e1-4af5-b41d-87020e3c1a46.png">

## 8.7 왼쪽 세미 조인
- 왼쪾 DataFrame의 어떤 값도 포함하지 않기 떄문에 조금 다르지만, 오른쪽 DataFrame의 존재여부에 따라 결과가 달라질수 있는 조인 타입
- 결과 데이터에는 왼쪽 DataFrame만 표시됨.

``` scala
joinType = "left_semi"
graduateProgram.join(person, joinExpression, joinType).show()

val gradProgram2 = graduateProgram.union(Seq(
    (0, "Masters", "Duplicated Row", "Duplicated School")).toDF())

gradProgram2.createOrReplaceTempView("gradProgram2")

gradProgram2.join(person, joinExpression, joinType).show()
```

``` sql
SELECT *
FROM gradProgram2
LEFT SEMI JOIN person ON gradProgram2.id = person.graduate_program
```

<img width="332" alt="스크린샷 2022-11-20 오후 5 06 28" src="https://user-images.githubusercontent.com/6982740/202891991-d1d0a95a-17e3-415d-84ed-6a364362324f.png">

## 8.8 왼쪽 안티 조인(LEFT ANTI JOIN)
-  세미 조인의 반대 개념
- 오른쪽 DataFrame의 어떤 값과도 일치되지 않은 왼쪽 DataFrame만 표시

``` scala
joinType = "left_anti"
graduateProgram.join(person, joinExpression, joinType).show()
```

``` sql
SELECT *
FROM graduateProgram
LEFT ANTI JOIN person ON graduateProgram.id = person.graduate_program
```

<img width="267" alt="스크린샷 2022-11-20 오후 5 11 11" src="https://user-images.githubusercontent.com/6982740/202892170-b6fad4e6-5126-4af6-9823-3298b201773a.png">

## 8.9 자연 조인(NATURAL JOIN)
- 조인하려는 컬럼을 암시적으로 추정하여 JOIN하는 방식
- 별도로 조인 키를 지정해주지 않고 동일한 컬럼명으로 참조되기 때문에 의도치 않은 결과를 반환할수 있어서 사용시 주의 필요.

``` sql
SELECT *
FROM graduateProgram
NATURAL JOIN person
```

## 8.10 교차 조인(CROSS JOIN) 또는 카테시안 조인(CARTESIAN JOIN)
- 교차 조인은 조건절을 기술하지 않은 내부 조인
- DataFrame의 모든 로우를 오른쪽 모든 로우와 결합
- 교차 조인을 하게 되면 엄청나게 많은 수의 로우가 생성될 수 있음.
  - `1,000 X 1,000`의 교차 조인 결과는 1,000,000개의 로우가 생성됨.

``` scala
joinType = "cross"
graduateProgram.join(person, joinExpression, joinType).show()

person.crossJoin(graduateProgram).show()
```

``` sql
SELECT *
FROM graduateProgram
CROSS JOIN person ON graduateProgram.id = person.graduate_program
```

<img width="587" alt="스크린샷 2022-11-20 오후 5 15 13" src="https://user-images.githubusercontent.com/6982740/202892276-ea40a9df-e106-4151-a560-f12141d266c8.png">

## 8.11 조인 사용시 문제점
### 8.11.1 복합 데이터 타입의 조인
- 복합 데이터 타입의 조인이 어려워보일수 있지만, 실제로 블리언을 반환하는 모든 표현식을 이용해서 조인을 할수 있다.

``` scala
import org.apache.spark.sql.functions.expr

person.withColumnRenamed("id", "personId")
  .join(sparkStatus, expr("array_contains(spark_status, id)")).show()
```

``` sql
SELECT *
FROM (
    SELECT id as personId, name, graduate_program, spark_status
    FROM person
)
INNER JOIN sparkStatus ON array_contains(spark_status, id)
```

<img width="556" alt="스크린샷 2022-11-20 오후 5 17 13" src="https://user-images.githubusercontent.com/6982740/202892346-d492bcc3-fdbc-4cd9-93cd-6ebcfeda3691.png">


### 8.11.2 중복 컬럼명 처리
- JOIN을 수행할때 가장 까다로운 것중 하나는 결과 DataFrame에서 중복된 컬럼명을 다루는 것이다.
- 각 컬럼은 스파크 SQL엔진인 카탈리스트 내에 고유 ID를 가지고 있으나, 이는 카탈리스트 내부에서만 관리되고, 직접 참조할 수 없다.
  - 따라서 중복된 컬럼명이 존재하는 DataFrame을 사용할 떄는 특정 컬럼을 참조할 수 없어서 별도 처리가 필요하다.

<img width="599" alt="스크린샷 2022-11-20 오후 5 22 02" src="https://user-images.githubusercontent.com/6982740/202892551-ddb7429c-73d2-4c54-a22a-eca4bce7f39e.png">


#### 해결방법1 : 다른 조인 표현식 사용
- 불리언 형태의 조인 표현식을 문자열이나 시퀀스 형태로 변경한다.
- 조인할때 두 컬럼 중 하나가 자동으로 제거 됨.

``` scala
val gradProgramDupe = graduateProgram.withColumnRenamed("id", "graduate_program")

person.join(gradProgramDupe,"graduate_program").select("graduate_program").show()
```

#### 해결방법2 : 조인 후 컬럼 제거
``` scala
person.join(gradProgramDupe, joinExpr).drop(person.col("graduate_program"))
  .select("graduate_program").show()

val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(graduateProgram, joinExpr).drop(graduateProgram.col("id")).show()
```

#### 해결 방법3 : 조인 전 컬럼명 변경(가장 좋은 방법이라고 생각됨)
``` scala
val gradProgram3 = graduateProgram.withColumnRenamed("id", "grad_id")
val joinExpr = person.col("graduate_program") === gradProgram3.col("grad_id")
person.join(gradProgram3, joinExpr).show()
```

## 8.12 스파크의 조인 수행 방식
- 스파크의 조인 수행 방식을 이해하기 위해서는 두 가지 핵심 전략을 이해해야 한다.

### 핵심 전략
- 노드간 네트워크 통신 전략
- 노드별 연산 전략

### 8.12.1 네트워크 통신 전략
- 스파크는 조인시 2가지 클러스터 통신 방식을 활용한다.
  - `셔플 조인(shuffle join)`, `브로드캐스트 조인(broadcast join)`

#### 큰 테이블과 큰 테이블 조인
- 하나의 큰 테이블을 다른 큰 테이블과 조인하면 아래와 같은 셔플 조인이 발생합니다.

<img width="606" alt="스크린샷 2022-11-20 오후 5 27 50" src="https://user-images.githubusercontent.com/6982740/202892730-b2d33efa-835b-4c94-8f73-7190a27d8253.png">

- 셔플 조인은 전체 노드 간 통신이 발생하는데, 사용된 특정 키나 키 집합이 어떤 노드에 있느냐에 따라 해당 노드와 데이터를 공유해야 합니다.
- 이런 방식 떄문에 네트워크가 복잡해지고, 많은 자원을 사용해야 합니다.
- 이런 데이터가 자주 사용된다면  네트워크 비용을 줄이기 위해 1차적으로 비정규화된 데이터셋을 하나 만들어서 프로세싱하는것도 방법이 될수 있다.

#### 큰 테이블과 작은 테이블 조인
- 작은 테이블이 단일 워커 노드의 메모리 크기에 적합할 정도로 충분히 작다면 조인 연산을  최적화 할수 있습니다.
- 이 경우 작은 DataFrame을 클러스터 전체 워커노드에 복제하는 `브로드캐스트 조인`을 수행한다면 효율적으로 처리가 간으합니다.

<img width="611" alt="스크린샷 2022-11-20 오후 5 31 04" src="https://user-images.githubusercontent.com/6982740/202892840-ef91c717-a6a7-4365-bc4a-3a4dcb4e1215.png">

- 최초 전체 워커노드에 데이터를 복제하는 과정에서는 I/O가 발생하겠지만  그이후로는 추가적인 네트워크 비용이 들기 않기 때문에 작업 전체적인 리소스 효율을 증대시킬수 있음.

##### broadcast join 수행하지 않는 경우
``` scala

val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(graduateProgram, joinExpr).explain()
```

<img width="424" alt="스크린샷 2022-11-20 오후 5 32 39" src="https://user-images.githubusercontent.com/6982740/202892875-82e81f28-a254-494c-bdc5-27820729d220.png">


##### broadcast join hint
- 아래와 같이 힌트를 주어 broadcast join을 수행할 수 있습니다.
  - 다만, 강제성이 있는것은 아니라서 옵티마이저 판단에 의해 무시될 수 있습니다.

``` scala
import org.apache.spark.sql.functions.broadcast

val joinExpr = person.col("graduate_program") === graduateProgram.col("id")
person.join(broadcast(graduateProgram), joinExpr).explain()
```

#### 아주 작은 테이블 사이의 조인
- 이 경우는 스파크 옵티마이저가 조인 방식을 결정하도록 내버려 두는게 가장 좋은 방법


## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
