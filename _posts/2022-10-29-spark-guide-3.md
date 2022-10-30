---
title: "[스파크 완벽 가이드] 3. 스파크 기능 둘러보기"
author: sungsu park
date: 2022-10-29 17:52:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 3. 스파크 기능 둘러보기
## 3.1 스파크의 기능

<img width="548" alt="스크린샷 2022-10-30 오후 5 01 09" src="https://user-images.githubusercontent.com/6982740/198868524-e88db775-542a-4643-8f52-f93966588d4c.png">

## 3.2 Dataset : 타입 안정성을 제공하는 구조적 API
- 자바와 스칼라의 정적 데이터 타입 코드를 지원하기 위해 고안된 구조적 API
- 이는 타입 안정성을 지원하고, 동적 타입 언어인 파이썬과 R에서는 사용할수 없음.

``` scala
case class Flight(DEST_COUNTRY_NAME: String, ORIGIN_COUNTRY_NAME : String, count: BigInt) {
   val flightsDF = spark.read
       .parquet("/data/flight-data/parquet/2010-summray.parquet/")

   val flights = flightsDF.as[Flight]

   flights
       .filter(flightRow => flightRow.0RIGIN_COUNTRY_NAME !=
  ''Canada'')
       .map(flightRow => flightRow)
       .take(5)
}
```

## 3.3 구조적 스트리밍
- 구조적 스트리밍은 안정화된 스트림 처리용 고수준 API
- 구조적 APi로 개발된 배치 모드의 연산을 스트리밍 방식으로 실행할 수 있음.

### 스트리밍이 아닌 일반적인 read

``` scala

val staticDataFrame = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("/data/retail-data/by-day/*.csv")

staticDataFrame.createOrReplaceTempView("retail_data")
val staticSchema = staticDataFrame.schema

import org.apache.spark.sql.functions.{window, column, desc, col}
staticDataFrame
  .selectExpr(
    "CustomerId",
    "(UnitPrice * Quantity) as total_cost",
    "InvoiceDate")
  .groupBy(
    col("CustomerId"), window(col("InvoiceDate"), "1 day"))
  .sum("total_cost")
  .show(5)
```

### 스트리밍 방식으로 처리
- `read` 대신 `readStream`을 사용한 것이 가장 큰 차이점.
- 아래 예제와 같은 방식을 운영 환경에 적용하는것은 추천하지 않음.

``` scala
val streamingDataFrame = spark.readStream
    .schema(staticSchema)
    .option("maxFilesPerTrigger", 1)
    .format("csv")
    .option("header", "true")
    .load("/data/retail-data/by-day/*.csv")

//  총 판매금액 계산
val purchaseByCustomerPerHour = streamingDataFrame
  .selectExpr(
    "CustomerId",
    "(UnitPrice * Quantity) as total_cost",
    "InvoiceDate")
  .groupBy(
    col("CustomerId"), window(col("InvoiceDate"), "1 day"))
  .sum("total_cost")


purchaseByCustomerPerHour.writeStream
    .format("memory") // memory = store in-memory table
    .queryName("customer_purchases") // the name of the in-memory table
    .outputMode("complete") // complete = all the counts should be in the table
    .start()
```

- 스트림이 시작되면 실행 결과가 어떠한 형태로 인메모리 테이블에 기록되는지 확인할수 있음.

## 3.4 머신러닝과 고급 분석
- 스파크는 내장된 MLlib를 사용해 대규모 머신러닝을 수행할 수 있음.
- 대용량 데이터를 대상으로 preprocessing, munging(rawdata를 접근/분석이 쉽도록 가공하는 행위) , model training, prediction 등을 할수 있음.
- classfication, regression, clustering, deep learning 등 정교한 API 제공


## 3.5 저수준 API
- 스파크는 RDD를 통해 자바와 파이썬 객체를 다루는 데 필요한 다양한 기능을 제공함.
- 스파크의 거의 모든 기능은 RDD를 기반으로 만들어졌음.
- 파티션 제어 등 DataFrame보다 더 세밀한 제어가 가능함.
- RDD는 `toDF()` 와 같은 API를 통해 손쉽게 DataFrame으롭 변환할 수 있음.
- 기본적으로 사용이 권장되지는 않으나, 비정형 데이터나 정제되지 않은 rawdata를 처리해야 한다면 사용할 일이 있을수도 있음.

``` scala
spark.sparkContext.parallelize(Seq(1, 2, 3)).toDF()

```

## 3.6 SparkR
- R 언어를 사용하기 위한 기능을 제공함.

## 3.7 스파크 에코시스템과 패키지
- 스파크의 최고 장점은 커뮤니티가 만들어낸 패키지 에코시스템과 다양한 기능.
- 개발자 누구나 자신이 개발한 패키지를 공개할수 있는 `spark-packages.org` 저장소가 있음.


## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
