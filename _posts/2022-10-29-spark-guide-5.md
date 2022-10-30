---
title: "[스파크 완벽 가이드] 5. 구조적 API 기본 연산"
author: sungsu park
date: 2022-10-29 19:21:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 5. 구조적 API 기본 연산

## 5.1 스키마
- 스키마는 DataFrame의 컬럼명과 데이터 타입을 정의(데이터소스로부터 얻거나 직접 정의)
- 스키마는 여러 개의 StructField 타입 필드로 구성된 StructType 객체
- 이름, 데이터 타입, 컬럼의 nullable 여부를 가짐.

``` scala
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}
import org.apache.spark.sql.types.Metadata

val myManualSchema = StructType(Array(
  StructField("DEST_COUNTRY_NAME", StringType, true),
  StructField("ORIGIN_COUNTRY_NAME", StringType, true),
  StructField("count", LongType, false,
    Metadata.fromJson("{\"hello\":\"world\"}"))
))

val df = spark.read.format("json").schema(myManualSchema)
  .load("/data/flight-data/json/2015-summary.json")
```

- 프로그래밍 언어의 데이터 타입을 스파크 데이터 타입으로 설정할 수 없다는점에 유의

## 5.2 컬럼과 표현식
- 사용자는 표현식으로 DataFrame의 컬럼을 선택, 조작, 제거할 수 있음.
- DataFrame을 통해 컬럼에 접근하여야 하고, 수정하려면 DataFrame의 트랜스포메이션을 사용해야 함.

### 5.2.1 컬럼
- `col()`, `colum()`을 사용하는것이 가장 간단

``` scala
import org.apache.spark.sql.functions.{col, column}
col("someColumnName")
column("someColumnName")
```

#### 명시적 컬럼 참조
- DataFrame의 컬럼은 col 메서드로 참조하고, 이는 데이터 JOIN시 유용하게 사용할 수 있음.

### 5.2.2 표현식
- 표현식은 DataFrame 레코드의 여러 값에 대한 트랜스포메이션의 집합을 의미.
- `expr("someCol")`, `col("someCol")`

#### 표현식으로 컬럼 표현
- 트랜스포메이션을 수행하려면 반드시 컬럼 참조를 사용해야 함.

``` scala
(((col("someCol") + 5) * 200) - 6) < col("otherCol")
```

<img width="572" alt="스크린샷 2022-10-30 오후 6 44 25" src="https://user-images.githubusercontent.com/6982740/198872282-54866627-4fae-4117-b45a-1e3c38a68fe0.png">


#### DataFrame 컬럼에 접근하기
``` scala
spark.read.format("json").load("/data/flight-data/json/2015-summary.json")
  .columns
```

## 5.3 레코드와 로우
- DataFrame의 각 로우는 하나의 레코드이며, 스파크에서는 Row 객체로 표현된다.

``` scala
df.first()
```

### 5.3.1 로우 생성
- Row 객체는 스키마 정보를 가지고 있지 않음.
- Row 객체를 직접 생성하려면 DataFrame의 스키마와 같은 순서로 값을 명시해야 함.

``` scala
import org.apache.spark.sql.Row
val myRow = Row("Hello", null, 1, false)


// 스칼라 버전
myRow(0) // type Any
myRow(0).asInstanceOf[String] // String
myRow.getString(0) // String
myRow.getInt(2) // Int
```

## 5.4 DataFrame의 트랜스포메이션
- 로우나 컬럼 추가/제거, 로우->컬럼 변환, 로우 순서 변경 등 처리

<img width="560" alt="스크린샷 2022-10-30 오후 6 48 26" src="https://user-images.githubusercontent.com/6982740/198872505-14418e51-9de8-47f1-8da5-9cbdb557fbca.png">

### 5.4.1 DataFrame 생성
- 원시 데이터소스에서 DataFrame을 생성할 수 있음.

``` scala
val df = spark.read.format("json")
  .load("/data/flight-data/json/2015-summary.json")
df.createOrReplaceTempView("dfTable")

import org.apache.spark.sql.Row
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}

val myManualSchema = new StructType(Array(
  new StructField("some", StringType, true),
  new StructField("col", StringType, true),
  new StructField("names", LongType, false)))

val myRows = Seq(Row("Hello", null, 1L))
val myRDD = spark.sparkContext.parallelize(myRows)
val myDf = spark.createDataFrame(myRDD, myManualSchema)
myDf.show()
```

<img width="208" alt="스크린샷 2022-10-30 오후 6 49 48" src="https://user-images.githubusercontent.com/6982740/198872573-3981814c-51b2-49ed-9f29-e3916fa09bbb.png">

### 5.4.2 `select`, `selectExpr`
- 데이터 테이블에 SQL을 사용한것처럼 DataFrame에서도 사용할 수 있음.


``` scala
df.select("DEST_COUNTRY_NAME", "ORIGIN_COUNTRY_NAME").show(2)
```

``` sql
SELECT DEST_COUNTRY_NAME, 0RIGIN_COUNTRY_NAME
FROM dfTable
LIMIT 2
```

#### `selectExpr`
- 새로운 DataFrame을 생성하는 복잡한 표현식으로 간단하게 만들어주는 도구
- 모든 유효한 비집계형 SQL 구문을 지정할 수 있음.

``` scala
df.selectExpr(
    "*", // include all original columns
    "(DEST_COUNTRY_NAME = ORIGIN_COUNTRY_NAME) as withinCountry")
  .show(2)
```

``` sql
SELECT *, (DEST=COUNTRY NAME = 0RIGIN COUNTRY_NAME) as withinCountry
FROM dfTable
LIMIT2
```

### 5.4.3 스파크 데이터 타입으로 변환하기
- 때로는 새로운 컬럼이 아닌 명시적인 값을 스파크에 전달해야 함.(ex. 상수)
- 리터럴을 사용할 수 있음.

``` scala
import org.apache.spark.sql.functions.lit
df.select(expr("*"), lit(1).as("One")).show(2)
```

``` sql
SELECT *, 1 as One
FROM dfTable
LIMIT 2
```

### 5.4.4 컬럼 추가하기
- DataFrame의 withColumn 메서드롤 사용해서 신규 컬럼을 추가할 수 있음.

``` scala
df.withColumn("numberOne", lit(1)).show(2)
```

``` sql
SELECT *, 1 as numberOne
FROM dfTable
LIMIT 2
```

### 5.4.5 컬럼명 변경하기
- `withColumn` 대신 `withColumnRenamed` 메서드로 컬럼명을 변경할 수 있음.

``` scala
df.withColumnRenamed("DEST_COUNTRY_NAME", "dest").columns

// ... dest, ORIGIN-COUNTRY_NAME, count
```

#### 5.4.6 예약 문자와 키워드
- 공백이나 하이픈(-) 같은 예약 문자는 컬럼명에 사용할 수 없음.
- 이를 사용하기 위해서는 백틱(`) 문자을 이용해 escaping해야 함.

``` scala
dfWithLongColName.selectExpr(
    "`This Long Column-Name`",
    "`This Long Column-Name` as `new col`")
  .show(2)
```

``` sql
SELECT `This Long Column-Name`, `This Long CoIumn-Name` as `new-col`
FROM dfTableLong
LIMIT 2
```

- 표현식 대신 문자열을 사용해서 명시적으로 컬럼을 참조하면 리터럴로 해석되기 떄문에 예약 문자가 포함된 컬럼을 참조할 수 있음.

### 5.4.7 대소문자 구분
- 기본적으로 스파크에서는 대소문자를 구별하지 않으나, 설정을 통해 구분하게 할 수 있음.

``` sql
set spark.sql.caseSensitive true
```

### 5.4.8 컬럼 제거하기

``` scala
df.drop("ORIGIN_COUNTRY_NAME").columns

// 다수의 컬럼 한번에 제거
dfWithLongColName.drop("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")
```

### 5.4.9 컬럼의 데이터 타입 변경

``` scala
df.withColumn("count2", col("count").cast("long"))
```

``` sql
5ELECT *, cast(count as string) AS count2
FROM dfTable
```

### 5.4.10 로우 필터링하기
- true / false을 판별하는 표현식을 만들어 필터링을 할 수 있음.
- `where` 또는 `filter` 메소드롤 이용해서 필터링(같은 동작)

``` scala
df.filter(col("count") < 2).show(2)
df.where("count < 2").show(2)
```

``` sql
SELECT *
FROM dfTable
WHERE count < 2
LIMIT 2
```

#### 여러 필터 적용
``` scala
df
   .where(col("count") < 2)
   .where(col("ORIGIN_COUNTRY_NAME") =!= "Croatia")
  .show(2)
```

``` sql
SELECT *
FROM dfTable
WHERE count < 2
AND 0RIGIN_COUNTRY_NAME != 'Croatia'
LIMIT 2
```

### 5.4.11 고유한 로우 얻기
``` scala
df
    .select("ORIGIN_COUNTRY_NAME", "DEST_COUNTRY_NAME")
    .distinct()
    .count()
```

``` sql
SELECT COUNT(DISTINCT(0RIGIN_COUNTRY_NAME, DEST_COUNTRY_NAME))
FROM dfTable
```

### 5.4.12 무작위 샘플 만들기
- sample 메서드를 이용해서 무작위 샘플 데이터를 얻을수 있음.
- 표본 데이터 추출 비율, 복원 추출 또는 비복원 추출의 사용 여부를 지정할 수 있음.

``` scala
val seed = 5
val withReplacement = false
val fraction = 0.5
df.sample(withReplacement, fraction, seed).count()
```

### 5.4.13 임의 분할하기
- 원본 DataFrame의 임의 크기로 분할할때 유용하다.

``` scala
val dataFrames = df.randomSplit(Array(0.25, 0.75), seed)
dataFrames(0).count() > dataFrames(1).count() // False

```

### 5.4.14 로우 합치기와 추가하기
- DataFrame은 불변성이기 때문에 레코드 추가하는 작업은 구조적으로 불가능.
- 레코드를 추가하려면 원본 DF에 새로운 DF를 통합해야 함.
- 반드시 동일한 스키마와 컬럼수를 가져야 함.

``` scala
import org.apache.spark.sql.Row
val schema = df.schema
val newRows = Seq(
  Row("New Country", "Other Country", 5L),
  Row("New Country 2", "Other Country 3", 1L)
)

val parallelizedRows = spark.sparkContext.parallelize(newRows)
val newDF = spark.createDataFrame(parallelizedRows, schema)

df.union(newDF)
  .where("count = 1")
  .where($"ORIGIN_COUNTRY_NAME" =!= "United States")
  .show() // get all of them and we'll see our new rows at the end
```

- 컬럼 표현식과 문자열을 비교할떄 `=!=` 연산자를 사용하면 컬럼의 실제 값을 비교 대상 문자열과 비교함.
  -   `=!=`, `===`


### 5.4.15 로우 정렬하기
- `sort`와 `orderBy` 메소드를 사용해 최대값 혹은 최솟값이 상단에 위치하도록 정렬할 수 있음.

``` scala
import org.apache.5park.Sql.functions.{desc, asc}


df.sort("count").show(5)
df.orderBy("count", "DEST_COUNTRY_NAME").show(5)
df.orderBy(col("count"), col("DEST_COUNTRY_NAME")).show(5)
df.orderBy(desc("count"), asc("DEST_COUNTRY_NAME")).show(2)

```

- `asc_nulls_first`, `desc_null_first`와 같은 메소드를 사용하여 렬된 DataFrame에서 null 값 표시 기준을 지정할 수 있음.

### 5.4.16 로우 수 제한하기
- 추출할 로우수 제한을 하는 경우 사용

``` scala
df.limit(5).show()
```

### 5.4.17 repartition와 coalesce
- 자주 필터링하는 컬럼을 기준으로 데디터를 분할하여 최적화할 수 있다.
- 파티셔닝 스키마와 파티션 수를 포함한 물리적인 데이터 구성 제어
- repartition을 하면 무조건 전체 데이터 셔플이 발생함. 파티션 수가 현재보다 많거나 컬럼을 기준으로 파티션을 만들어야 하는 경우에만 사용

``` scala
df.rdd.getNumPartitions // 1

df.repartition(5)

//  특정 컬럼을 기준으로 자주 필터링한다면 이를 기준으로 파티션을 재분배하는것이 좋음.
df.repartition(col("DEST_COUNTRY_NAME"))

// 선택적으로 파티션 수를 지정할 수 있음.
df.repartition(5, col("DEST_COUNTRY_NAME"))
```

#### `coalesec`
- 전체 데이터를 셔플하지 않고 파티션을 병합하는 경우에 사용

``` scala
df.repartition(5, col("DEST_COUNTRY_NAME")).coalesce(2)
```

### 5.4.18 드라이버로 로우 데이터 수집하기
- 로컬환경에서 데이터를 다루려면 드라이버로 데이터를 수집해야 함.

``` scala
val collectDF = df.limit(10)
collectDF.take(5) // take works with an Integer count
collectDF.show() // this prints it out nicely
collectDF.show(5, false)
collectDF.collect()

```

#### 전체 데이터셋에 대한 반복 처리를 위해 드라이버로 로우를 모으는 다른 방법

``` scala
collectDF.toLocalIterator()
```

- 드라이버로 모든 데이터 컬렉션을 수집하는 경우 매우 큰 비용(CPU, 메모리, 네트워크 등)이 발생하므로 데이터와 필요에 따라 적절히 선택되어야 한다.

## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
