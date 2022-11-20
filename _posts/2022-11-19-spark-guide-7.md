---
title: "[스파크 완벽 가이드] 7. 집계 연산"
author: sungsu park
date: 2022-11-19 16:20:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 7. 집계 연산
## 7.1 집계 함수
- ` org.apache.Spark.sql.functions` 패키지에서 제공되는 함수를 찾아볼수 있음.

### 7.1.1 count
- count는 액션이 아닌 트랜스포메이션으로 동작
  - 지연 연산이 아니라 즉각 결과를 처리한다는 의미

``` scala
import org.apache.spark.sql.functions.count
df.select(count("StockCode")).show() // 541909
```

``` sql
SELECT COUNT(*) FROM dfTable
```

> null 값이 포함된 데이터 레코드의 count를 구할때는 유의사항이 있음.
> `count(*)`을 사용하면 null 값을 가진 로우도 포함해서 카운트하고, 특정 컬럼을 지정하면 null 값은 카운트 하지 않음.

### 7.1.2 countDistinct
- 전체 레코드 수가 아니라 고유 레코드 수를 구해야 하는 경우 사용

``` scala
import org.apache.spark.sql.functions.countDistinct

df.select(countDistinct("StockCode")).show() // 4070
```

``` sql
SELECT COUNT(DISTINCT *)
FROM DFTABLE
```

### 7.1.3 approx_count_distinct
- 정확한 고유 개수가 무의미한 경우 어느 정도 수준의 정확도를 가지는 근사치만으로 유의미할 경우 사용할수 있는 집계함수
  - `최대 추정 오류율`(maximum estimation error)이라는 한가지 파라미터를 더 사용함.
  - 대규모 데이터셋을 처리할때 오차가 어느정도 발생할수 있지만 countDistinct함수보다 더 빠르게 결과를 반환받을수 있음.

``` scala
import org.apache.spark.sql.functions.approx_count_distinct

df.select(approx_count_distinct("StockCode", 0.1)).show() // 3364
```

``` sql
SELECT approx_count_distinct(StockCode, 0.1)
FROM DFTABLE
```

### 7.1.4 `first`와 `last`
- DataFrame의 첫번째 값이나 마지막 값을 조회

``` scala
import org.apache.spark.sql.functions.{first, last}
df.select(first("StockCode"), last("StockCode")).show()
```

``` sql
SELECT first(StockCode), last(StockCode)
FROM dfTable
```

### 7.1.5 `min` 과 `max`
- 최솟값과 최댓값 추출

``` scala
import org.apache.spark.sql.functions.{min, max}

df.select(min("Quantity"), max("Quantity")).show()

```

``` sql
SELECT min(Quantity), max(Quantity)
FROM dfTable
```

### 7.1.6 sum
- DataFrame에서 특정 컬럼의 모든 값을 합산할때 사용

``` scala
import org.apache.spark.sql.functions.sum

df.select(sum("Quantity")).show() // 5176450
```

``` sql
SELECT sum(Quantity)
FROM dfTable
```

### 7.1.7 sumDistinct
- 특정 컬럼의 고윳값을 합산

``` scala
import org.apache.spark.sql.functions.sumDistinct

df.select(sumDistinct("Quantity")).show() // 29310

```

``` sql
SELECT SUM(DISTINCT Quantity)
FROM dfTable -- 29310
```

### 7.1.8 avg
- `sum()` / `count()` 대신 스파크의 avg, mean 함수를 사용하면 평균값을 더 쉽게 구할 수 있음.

``` scala
import org.apache.spark.sql.functions.{sum, count, avg, expr}

df.select(
    count("Quantity").alias("total_transactions"),
    sum("Quantity").alias("total_purchases"),
    avg("Quantity").alias("avg_purchases"),
    expr("mean(Quantity)").alias("mean_purchases"))
  .selectExpr(
    "total_purchases/total_transactions",
    "avg_purchases",
    "mean_purchases").show()
```

### 7.1.9 분산과 표준편차
- 평균을 구하다 보면 자연스럽게 분산과 표준편차가 궁금할 수 있다.
- 스파크에서 표본표준편차, 모표준편차과 같은 처리도 지원함.

``` scala
import org.apache.spark.sql.functions.{var_pop, stddev_pop}
import org.apache.spark.sql.functions.{var_samp, stddev_samp}

df.select(var_pop("Quantity"), var_samp("Quantity"),
  stddev_pop("Quantity"), stddev_samp("Quantity")).show()
```

``` sql
SELECT
   var_pop(Quantity),
  var_samp(Quantity),
  stddev_pop(Quantity),
   stddev_samp(Quantity)
FROM dfTable
```

### 7.1.10 비대칭도와 첨도
- 비대칭도(skewness), 첨도(kurtosis)와 같은 데이터의 변곡점(extreme point)을 측정하는 방법도 제공.

``` scala
import org.apache.spark.sql.functions.{skewness, kurtosis}

df.select(skewness("Quantity"), kurtosis("Quantity")).show()
```

``` sql
SELECT skewness(Quantity), kurtosis(Quantity)
FROM dfTable
```

### 7.1.11 공분산과 상관관계
- 두 컬럼값 사이의 영향도 비교하는 함수들도 제공.
- `cov()` :  공분산(covariance) 계산
- `corr()` :  상관관계(correlation) 계산
- 표본공분산(sample covariance) 방식이나 모공분산(population covariance) 방식으로 공분산 계산도 가능.

``` scala
import org.apache.spark.sql.functions.{corr, covar_pop, covar_samp}

df.select(corr("InvoiceNo", "Quantity"), covar_samp("InvoiceNo", "Quantity"),
    covar_pop("InvoiceNo", "Quantity")).show()
```

``` sql
SELECT corr(InvoiceNo, Quantity), covar_samp(InvoiceNo, Quantity),
  covar_pop(InvoiceNo, Quantity)
FROM dfTable
```

### 7.1.12 복합 데이터 타입의 집계
- 복합 데이터 타입을 사용해 집계를 수행할 수 있다.
  - 특정 컬럼의 값을 리스트로 수집, 셋 데이터 타입으로 고윳값만 수집

``` scala
import org.apache.spark.sql.functions.{collect_set, collect_list}

df.agg(collect_set("Country"), collect_list("Country")).show()

```

``` sql
SELECT collect_set(Country), collect_set(Country)
FROM dfTable
```

## 7.2 그룹화
- 실제로는 DataFrame 수준을 넘어서서 그룹 기반 집계를 수행하는 경우가 많이 있음.
  - 단일 컬럼의 데이터를 그룹화하고, 해당 그룹의 다른 여러 컬럼을 사용해 계산하기 위해 카테고맇여 데이터를 사용한다.

### 그룹화 과정
- `RelationalGroupedDataset` -> `DataFrame` 반환

``` scala
df.groupBy("InvoiceNo", "CustomerId").count().show()
```

``` sql
SELECT count(*) FROM dfTable GROUP BY InvoiceNo, CustomerId
```

<img width="230" alt="스크린샷 2022-11-20 오후 2 43 25" src="https://user-images.githubusercontent.com/6982740/202887522-e8c943c0-9a64-4da4-85a9-283942c0a0ac.png">

### 7.2.1 표현식을 이용한 그룹화
- count 함수를 select 구문에 표현식으로 지정하는것보다 agg 메소드를 사용하는 것이 좋음.
- `agg` 메서드는 여러 집계 ㄹ처리를 한번에 지정할 수 있고, 집계에 표현식을 사용할 수 있음.
- 트랜스포메이션이 완료된 컬럼에서 `alias` 메서드를 사용할 수 있음.

``` scala
df.groupBy("InvoiceNo").agg(
  count("Quantity").alias("quan"),
  expr("count(Quantity)")).show()
```

<img width="266" alt="스크린샷 2022-11-20 오후 2 45 08" src="https://user-images.githubusercontent.com/6982740/202887559-51e60689-55b9-48a8-adf5-8ea1bc4c1fe4.png">

### 7.2.2 맵을 이용한 그룹화
- 컬럼을 키로, 수행할 집계 함수의 문자열을 값으로 하는 맵 타입을 사용해서 트랜스포메이션을 정의할 수 있음.

``` scala
df.groupBy("InvoiceNo").agg("Quantity"->"avg", "Quantity"->"stddev_pop").show()
```

``` sql
SELECT avg(Quantity), stddev_pop(Quantity), InvoiceNo
FROM dfTable
GROUP BY InvoiceNo
```

<img width="419" alt="스크린샷 2022-11-20 오후 2 46 12" src="https://user-images.githubusercontent.com/6982740/202887578-ecb9a132-7229-446d-88de-6b08192b67e6.png">


## 7.3. 윈도우 함수
- 윈도우 함수를 집계에 사용할 수 있다.
- 윈도우 함수는 데이터의 특정 윈도우(window)를 대상으로 고유의 집계 연산을 수행하는 것을 말함.
  - 윈도우는 현재 데이터에 대한 참조를 사용해서 정의

### `group-by`와의 차이점
- `group-by` 를 사용하면 모든 로우 레코드가 단일 그룹으로만 이동
- 윈도우 함수는  frame에 입력되는 모든 로우에 대해 결과값을 계싼.
  - frame은 로우 그룹 단위 테이블을 의미하고, 각 로우는 하나 이상의 프레임에 할당될 수 있다.

### 지원되는 윈도우 함수
- 랭크 함수(raking function)
- 분석 함수(analytic function)
- 집계 함수(aggregate function)

### 윈도우 함수 시각화

<img width="675" alt="스크린샷 2022-11-20 오후 2 47 37" src="https://user-images.githubusercontent.com/6982740/202887637-25fc86cb-55c4-48f2-b1fa-221ba324fc4f.png">

### 윈도우 함수 스펙 정의 및 사용

``` scala
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions.col

val windowSpec = Window
  .partitionBy("CustomerId", "date")  // 파티셔닝 스키마와 개념적으로 관련 없는 Window 명세
  .orderBy(col("Quantity").desc)       // 파티션의 정렬 방식 지정
  .rowsBetween(Window.unboundedPreceding, Window.currentRow)    // 입력된 로우의 참조를 기반으로 프레임에 로우가 포함될 수 있는지 결정
```

- 윈도우의 그룹을 어떻게 나눌지 결정하는것과 유사한 개념.

``` scala
import org.apache.spark.sql.functions.max
import org.apache.spark.sql.functions.{dense_rank, rank}
import org.apache.spark.sql.functions.col

// 최대 구매 개수 구하기
val maxPurchaseQuantity = max(col("Quantity")).over(windowSpec)

// 구매량 순위 구하기
val purchaseDenseRank = dense_rank().over(windowSpec)
val purchaseRank = rank().over(windowSpec)

// 시간대별 최대 구매 개수 구하기
dfWithDate.where("CustomerId IS NOT NULL").orderBy("CustomerId")
  .select(
    col("CustomerId"),
    col("date"),
    col("Quantity"),
    purchaseRank.alias("quantityRank"),
    purchaseDenseRank.alias("quantityDenseRank"),
    maxPurchaseQuantity.alias("maxPurchaseQuantity")).show()
```

``` sql
SELECT CustomerId, date, Quantity,
  rank(Quantity) OVER (PARTITION BY CustomerId, date
                       ORDER BY Quantity DESC NULLS LAST
                       ROWS BETWEEN
                         UNBOUNDED PRECEDING AND
                         CURRENT ROW) as rank,
  dense_rank(Quantity) OVER (PARTITION BY CustomerId, date
                             ORDER BY Quantity DESC NULLS LAST
                             ROWS BETWEEN
                               UNBOUNDED PRECEDING AND
                               CURRENT ROW) as dRank,
  max(Quantity) OVER (PARTITION BY CustomerId, date
                      ORDER BY Quantity DESC NULLS LAST
                      ROWS BETWEEN
                        UNBOUNDED PRECEDING AND
                        CURRENT ROW) as maxPurchase
FROM dfWithDate
WHERE CustomerId IS NOT NULL
ORDER BY CustomerId
```

<img width="637" alt="스크린샷 2022-11-20 오후 3 06 00" src="https://user-images.githubusercontent.com/6982740/202888439-eab43069-625a-4f0c-a1cc-2017aa993088.png">


## 7.4 그룹화 셋
- 때로는 여러 그룹에 걸쳐 집계할 수 있는 무언가가 필요할수 있는데 이때 그룹화 셋을 사용할 수 있다.
- 그룹화 셋은 여러 집계를 결합하는 저수준 기능입니다.

### 그룹화셋을 사용하지 않은 방식

``` scala
val dfNoNull = dfWithDate.drop()

dfNoNull.createOrReplaceTempView("dfNoNull")
```

``` sql
SELECT CustomerId, stockCode, sum(Quantity)
FROM dfNoNull
GROUP BY customerId, stockCode
ORDER BY CustomerId DESC, stockCode DESC
```

<img width="296" alt="스크린샷 2022-11-20 오후 3 07 27" src="https://user-images.githubusercontent.com/6982740/202888481-ac9212ad-a35c-41d2-84d0-e1da5b0ceec6.png">

### 그룹화셋 사용방식
``` scala
SELECT CustomerId, stockCode, sum(Quantity) FROM dfNoNull
GROUP BY customerId, stockCode GROUPING SETS((customerId, stockCode))
ORDER BY CustomerId DESC, stockCode DESC
```

- 책 내용을 보면 무슨차이인지 모르겠음
  -  그룹화셋을 사용할떄는 `dfNoNull` 대신 `dfWithDate`을 사용할수 있다는것인지 추후에 스파크 개발을 해보면서 업데이트 하겠다.

#### null과 그룹화 셋
- null 값에 따라 집계 수준이 달라지는데, null값이 제거되지 않으면 부정확한 결과값을 얻을수 있음.
- `CustomerId`, `stockCode`과 관계 없이 총 수량의 합산 결과를 추가하려고 하는경우 `group-by`를 이용해서는 불가능하지만 그룹화셋을 사용하면 가능함.

``` sql
SELECT CustomerId, stockCode, sum(Quantity) FROM dfNoNull
GROUP BY customerId, stockCode GROUPING SETS((customerId, stockCode),())
ORDER BY CustomerId DESC, stockCode DESC
```

#### `GROUPING SETS` 제약사항
- 이 구문은 SQL에서만 사용 가능하다.
- DataFrame에서 동일한 연산을 수행하려면 `rollup`, `cube` 메소드를 사용해야 한다.


### 7.4.1 롤업(rollup)
- 다양한 컬럼을 그룹화 키로 설정하면 그룹화 키로 설정된 조합 뿐만 아니라 데이터셋에서 볼수 있는 실제 조합을 모두 살펴볼 수 있다.
- 롤업은 group-by 스타일의 다양한 연산을 수행할 수 있는 다차원 집계 기능이다.

``` scala
val rolledUpDF = dfNoNull.rollup("Date", "Country").agg(sum("Quantity"))
  .selectExpr("Date", "Country", "`sum(Quantity)` as total_quantity")
  .orderBy("Date")

rolledUpDF.show()
```


<img width="309" alt="스크린샷 2022-11-20 오후 3 20 07" src="https://user-images.githubusercontent.com/6982740/202888828-f44c544a-7968-45cf-aa18-3c835dba3d99.png">

- null 값을 가진 로우에서 전체 날짜의 합계를 확인할 수 있다.
  - 두 개의 컬럼값이 모두 null인 로우는 두 컬럼에 속한 레코드의 전체 합계

``` scala
rolledUpDF.where("Country IS NULL").show()

rolledUpDF.where("Date IS NULL").show()
```

<img width="221" alt="스크린샷 2022-11-20 오후 3 21 21" src="https://user-images.githubusercontent.com/6982740/202888851-7caacc7b-86a6-4f7b-9739-72983620b454.png">


### 7.4.2 큐브(cube)
- 롤업을 고차원적으로 사용할수 있도록 해주는 기능.
- 요소들을 계층적으로 다루는 대신 모든 차원에 대해 동일한 작업을 수행한다.

``` scala
dfNoNull.cube("Date", "Country").agg(sum(col("Quantity")))
  .select("Date", "Country", "sum(Quantity)").orderBy("Date").show()
```

<img width="301" alt="스크린샷 2022-11-20 오후 3 22 17" src="https://user-images.githubusercontent.com/6982740/202888884-5fbf2254-b481-4ded-b621-d7df74572263.png">

- 큐브를 사용해 테이블에 있는 모든 정보를 빠르고 쉽게 조회할 수 있는 요약 정보 테이블을 만들수 있습니다.

### 7.4.3 그룹화 메타데이터
- 큐브와 롤업을 사용하다보면 수준에 따라 쉽게 필터링하기 위해 집계 수준을 조회하는 경우가 발생한다.
- `grouping_id`을 사용해서 결과 데이터셋의 집계 수준을 명시하는 컬럼을 제공할 수 있음.

#### 그룹화 ID의 의미

<img width="616" alt="스크린샷 2022-11-20 오후 4 12 43" src="https://user-images.githubusercontent.com/6982740/202890407-8fde26a3-0af3-4199-b0f9-4ed26c715743.png">

#### 그룹화 ID 사용 예시
``` scala
import org.apache.spark.sql.functions.{grouping_id, sum, expr}

dfNoNull.cube("customerId", "stockCode").agg(grouping_id(), sum("Quantity"))
.orderBy(col("grouping_id()").desc)
.show()
```

<img width="358" alt="스크린샷 2022-11-20 오후 4 13 37" src="https://user-images.githubusercontent.com/6982740/202890425-ac561b3e-5ed2-4af0-8f4c-f3825af671ea.png">


### 7.4.4 피벗
- 피벗(pivot)을 사용해 로우를 컬럼으로 변환할 수 있다.

``` scala
val pivoted = dfWithDate.groupBy("date").pivot("Country").sum()

pivoted.where("date > '2011-12-05'").select("date" ,"`USA_sum(Quantity)`").show()

// data             | USA_sum(Quantity)
// 2011-12-06 | null
// 2011-12-09 | null
// 2011-12-08 | -196
// 2011-12-07 | null
```

- 컬럼의 모든 값을 단일 그룹화해서 계산할 수 있음.

## 7.5 사용자 정의 집계 함수
- UDAF(user-defined aggregation function)은 직접 제작한 함수나 비즈니스 규칙에 기반을 둔 자쳬 집계 함수를 정의하는 방법.
- 입력 데이터 그룹에 직접 개발한 연산을 수행할 수 있다.
- 스파크는 입력 데이터의 모든 그룹의 중간 결과를 단일 AggregationBuffer에 저장해 관리

### UDAF 정의
- UserDefinedAggregateFunction을 상속하고 아래 메서드를 정의

<img width="563" alt="스크린샷 2022-11-20 오후 4 24 03" src="https://user-images.githubusercontent.com/6982740/202890764-2df1e22d-4263-46e6-9869-1e0028956685.png">

``` scala
import org.apache.spark.sql.expressions.MutableAggregationBuffer
import org.apache.spark.sql.expressions.UserDefinedAggregateFunction
import org.apache.spark.sql.Row
import org.apache.spark.sql.types._

class BoolAnd extends UserDefinedAggregateFunction {
  def inputSchema: org.apache.spark.sql.types.StructType =
    StructType(StructField("value", BooleanType) :: Nil)
  def bufferSchema: StructType = StructType(
    StructField("result", BooleanType) :: Nil
  )
  def dataType: DataType = BooleanType
  def deterministic: Boolean = true
  def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = true
  }
  def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    buffer(0) = buffer.getAs[Boolean](0) && input.getAs[Boolean](0)
  }
  def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getAs[Boolean](0) && buffer2.getAs[Boolean](0)
  }
  def evaluate(buffer: Row): Any = {
    buffer(0)
  }
}
```

### UDAF 등록
``` scala
import org.apache.spark.sql.functions._

val ba = new BoolAnd
spark.udf.register("booland", ba)


spark.range(1)
  .selectExpr("explode(array(TRUE, TRUE, TRUE)) as t")
  .selectExpr("explode(array(TRUE, FALSE, TRUE)) as f", "t")
  .select(ba(col("t")), expr("booland(f)"))
  .show()
```

<img width="168" alt="스크린샷 2022-11-20 오후 4 29 06" src="https://user-images.githubusercontent.com/6982740/202890911-c1b970ef-ffaa-49b6-a298-7d7be7815853.png">

### UDAF 유의사항
- 스파크 완벽 가이드 기준으로 스칼라와 자바로만 사용할 수 있음.


## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
