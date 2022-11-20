---
title: "[스파크 완벽 가이드] 6. 다양한 데이터 타입 다루기"
author: sungsu park
date: 2022-11-19 13:36:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# [스파크 완벽 가이드] 6. 다양한 데이터 타입 다루기
## 데이터 타입들
> 불리언 타입
> 수치 타입
> 문자열 타입
> date와 timestamp 타입
> null 값 다루기
> 복합 데이터 타입
> 사용자 정의 함수

## 6.1 API는 어디서 찾을까?
### DataFrame(DataSet) 메서드
- DataFrame은 Row 타입을 가진 Dataset이므로 Dataset 메서드를 보면 됨.
- Column 메서드
-  alias나 contains 같은 컬럼 관련된 여러 메소드를 제공하고, Column API 스파크 문서를 참고하자.

``` scala
val df = spark.read.format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("/data/retail-data/by-day/2010-12-01.csv")
df.printSchema()
df.createOrReplaceTempView("dfTable")
```

<img width="663" alt="스크린샷 2022-11-12 오후 6 51 03" src="https://user-images.githubusercontent.com/6982740/201468691-1bea8149-f905-4ec2-94b1-e69d7460b910.png">

## 6.2 스파크 데이터 타입으로 변환하기
- 프로그래밍 언어 고유 데이터 타입을 스파크 데이터 타입으로 변환해보자.
- `lit()` 은 다른 언어의 데이터 타입을 스파크 데이터 타입에 맞게 변환합니다.

``` scala
import org.apache.spark.sql.functions.lit

df.select(lit(5), lit("five"), lit(5.0))
```

## 6.3 불리언 데이터 타입 다루기
- 불리언 구문은 `and`, `or`, `true`, `false` 로 구성됨.

``` scala
import org.apache.spark.sql.functions.col

df.where(col("InvoiceNo").equalTo(536365))
  .select("InvoiceNo", "Description")
  .show(5, false)

df.where(col("InvoiceNo") === 536365)
  .select("InvoiceNo", "Description")
  .show(5, false)
```

- 스칼라에서는 not, equalsTo 를 사용하거나 `===`, `=!=`을 통해 동등성 비교를 할 수 있음.

``` scala
// 일치
df.where("InvoiceNo = 536365")
  .show(5, false)

// 불일치
df.where("InvoiceNo <> 536365")
  .show(5, false)
```

- 위와 같이 문자열 표현에 조건절을 명시하는 방법이 있음.( 가장 명확)

### `and`, `or`을 사용해 여러 조건 표현식

``` scala
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.where(col("StockCode").isin("DOT")).where(priceFilter.or(descripFilter))
  .show()
```

- `and`는 별도로 정의하지 않더라도 스파크 내부적으로 하나의 문장으로 변환됨.
- `or` 구문을 사용할 때는 반드시 동일한 구문에 조건을 정의해주어야 함.

### 불리언 표현식을 이용해 DataFrame 필터링
- 조회 필터링 조건 외에 DataFrame 데이터를 필터링하는데에도 이용할 수 있음.

``` scala
val DOTCodeFilter = col("StockCode") === "DOT"
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")

df.withColumn("isExpensive", DOTCodeFilter.and(priceFilter.or(descripFilter)))
  .where("isExpensive")
  .select("unitPrice", "isExpensive").show(5)
```

``` sql
SELECT UnitPrice, (StockCode = 'DOT' AND
  (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1)) as isExpensive
FROM dfTable
WHERE (StockCode = 'DOT' AND
       (UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1))
```

### 컬럼명을 사용해 필터를 정의할 수 있음.
``` scala
import org.apache.spark.sql.functions.{expr, not, col}

df.withColumn("isExpensive", not(col("UnitPrice").leq(250)))
  .filter("isExpensive")
  .select("Description", "UnitPrice").show(5)
df.withColumn("isExpensive", expr("NOT UnitPrice <= 250"))
  .filter("isExpensive")
  .select("Description", "UnitPrice").show(5)
```

## 6.4 수치형 데이터 타입 다루기
- `count`는 빅데이터 처리애서 필터링 다음으로 많이 수행하는 작업이다.
- 수치형 데이터 타입을 사용해 연산 방식을 정의하기만 하면 된다.

``` scala
import org.apache.spark.sql.functions.{expr, pow}

// 실제수량  = (현재 수량 * 단위가격)^2 + 5
val fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5

df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)

df.selectExpr(
  "CustomerId",
  "(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)
```

``` sql
SELECT customerId, (POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity
FROM dfTable
```

### `round`(반올림), `bround`(내림)
``` scala
import org.apache.spark.sql.functions.{round, bround}
import org.apache.spark.sql.functions.lit

df.select(round(col("UnitPrice"), 1).alias("rounded"), col("UnitPrice")).show(5)

df.select(round(lit("2.5")), bround(lit("2.5"))).show(2)

```

``` sql
SELECT round(2.5), bround(2.5)
```

### 컬럼 사이의 상관관계 계산
- 피어슨 상관 계수를 계산해보고자 할 경우 내부적으로 제공해주는 함수와 메서드를 사용해 계산할 수 있음.

``` scala
import org.apache.spark.sql.functions.{corr}

df.stat.corr("Quantity", "UnitPrice")
df.select(corr("Quantity", "UnitPrice")).show()
```

``` sql
SELECT corr(Quantity, UnitPrice)
FROM dfTable
```

### 하나 이상의 컬럼에 대한 요약 통계

``` scala
df.describe().show()

// 아래 import를 통해 정확한 수치 집계 가능
import org.apache.spark.sql.functions.{count, mean, stddev_pop, min, max}
```

<img width="515" alt="스크린샷 2022-11-12 오후 7 36 28" src="https://user-images.githubusercontent.com/6982740/201470257-55e82270-3bdc-426c-9b54-fe07d5eda779.png">



### StatFunctions
- stat 속성을 사용해 다양한 통계값을 계산할 수 있음(approxQuantile 을 통한 데이터 백분위수 계산, 근사치 계산)

``` scala
val colName = "UnitPrice"
val quantileProbs = Array(0.5)
val relError = 0.05
df.stat.approxQuantile("UnitPrice", quantileProbs, relError) // 2.51

// 교차표 조회
df.stat.crosstab("StockCode", "Quantity").show()

// 자주 사용하는 항목쌍 조회
df.stat.freqItems(Seq("StockCode", "Quantity")).show()

// 모든 로우에 고유 ID값 추가
df.select(monotonically_increasing_id()).show(2)
```

## 6.5 문자열 데이터 타입 다루기
- 거의 모든 데이터 처리 과정에서 발생.
- 정규 표현식, 데이터 치환, 문자열 존재 여부, 대/소문자 변환 처리 등 작업이 가능하다.

### 대/소문자 변환
- `initcap()` : 주어진 문자열에서 공백으로 나뉘는 모든 단어의 첫글자를 대문자로 변경
-  `lowwer()` : 전체를 소문자로 변경
-  `upper()` :  전체를 대문자로 변경

``` scala
import org.apache.spark.sql.functions.{initcap}
df.select(initcap(col("Description"))).show(2, false)

df.select(col("Description"),
  lower(col("Description")),
  upper(lower(col("Description")))).show(2)
```

### 공백 제거/추가
- `lpad()`, `ltrim()`, `rpad()`, `rtrim()`, `trim()`

``` scala
import org.apache.spark.sql.functions.{lit, ltrim, rtrim, rpad, lpad, trim}
df.select(
    ltrim(lit("    HELLO    ")).as("ltrim"),
    rtrim(lit("    HELLO    ")).as("rtrim"),
    trim(lit("    HELLO    ")).as("trim"),
    lpad(lit("HELLO"), 3, " ").as("lp"),
    rpad(lit("HELLO"), 10, " ").as("rp")).show(2)
```

``` sql
SELECT
  ltrim('    HELLLOOOO  '),
  rtrim('    HELLLOOOO  '),
  trim('    HELLLOOOO  '),
  lpad('HELLOOOO  ', 3, ' '),
  rpad('HELLOOOO  ', 10, ' ')
FROM dfTable
```


### 6.5.1 정규 표현식
- 스파크에서는 `regexp_extract`, `regexp_replace` 함수를 제공합니다.
- 자바 정규 표현식 문법이 일반적인 문법과 약간 다르므로 사용 전 검토 필요

#### 정규 표현식을 이용한 문자열 치환
``` scala
import org.apache.spark.sql.functions.regexp_replace

val simpleColors = Seq("black", "white", "red", "green", "blue")
val regexString = simpleColors.map(_.toUpperCase).mkString("|")

// the | signifies `OR` in regular expression syntax
df.select(
  regexp_replace(col("Description"), regexString, "COLOR").alias("color_clean"),
  col("Description")).show(2)
```

``` sql
SELECT
  regexp_replace(Description, 'BLACK|WHITE|RED|GREEN|BLUE', 'COLOR') as
  color_clean, Description
FROM dfTable
```

#### `translate` 를 이용한 문자열 치환

``` scala
import org.apache.spark.sql.functions.translate

df.select(translate(col("Description"), "LEET", "1337"), col("Description"))
  .show(2)
```

#### 값의 존재여부 확인
``` scala
val containsBlack = col("Description").contains("BLACK")
val containsWhite = col("DESCRIPTION").contains("WHITE")

df.withColumn("hasSimpleColor", containsBlack.or(containsWhite))
  .where("hasSimpleColor")
  .select("Description").show(3, false)
```

``` sql
-- sql에서는 instr을 이용해 존재 여부 확인
SELECT Description
FROM dfTable
WHERE instr(Description, 'BLACK') >= 1
OR instr(Description, 'WHITE') >= 1

```

## 6.6 날짜와 타임스탬프 데이터 타입 다루기
- 날짜와 시간은 프로그래밍 언어와 DB 분야의 변함없는 과제여서 스파크에서는 복잡함을 피하고자 시간 관련 정보만 집중적으로 관리합니다.
- 달력 형태의 date, 날짜와 시간 정보를 모두 가지는 timestamp입니다.
- 스파크의 TimestampType 클래스는 초 단위 정밀도까지만 지원함.
  - 그 아래 단위까지 다뤄야 한다면 Long 데이터 타입으로 변환해 처리해야 한다.

``` scala
import org.apache.spark.sql.functions.{current_date, current_timestamp}
import org.apache.spark.sql.functions.{date_add, date_sub}

// 현재 날짜, 시간 구하기
val dateDF = spark.range(10)
  .withColumn("today", current_date())
  .withColumn("now", current_timestamp())

dateDF.createOrReplaceTempView("dateTable")

dateDF.printSchema()

// 현재 기준으로부터 5일 전후 날짜 구하기
dateDF.select(date_sub(col("today"), 5), date_add(col("today"), 5)).show(1)
```

### 두 날짜의 차이를 구하기
- `datediff` 함수를 사용

``` scala
import org.apache.spark.sql.functions.{datediff, months_between, to_date}

dateDF.withColumn("week_ago", date_sub(col("today"), 7))
  .select(datediff(col("week_ago"), col("today"))).show(1)

dateDF.select(
    to_date(lit("2016-01-01")).alias("start"),
    to_date(lit("2017-05-22")).alias("end"))
  .select(months_between(col("start"), col("end"))).show(1)
```

- `to_date` 함수는 문자열을 날짜로 변환할 수 있는 함수로 날짜 포맷은 Java의 `SimpleDateFormat` 클래스가 지원하는 포맷을 사용해야 한다.

``` scala
import org.apache.spark.sql.functions.{to_date, lit}

spark.range(5).withColumn("date", lit("2017-01-01"))
  .select(to_date(col("date"))).show(1)
```

- 날짜를 파싱할 수 없는 경우 `null`을 반환함.

## 6.7 null 값 다루기
- DataFrame에서 빠져 있거나 비어있는 데이터를 표현할 때는 항상 null 값을 사용하는 것이 좋음.
  - 사용자가 정의하는 대체 문자열이나 이런것을 사용하지 않아야 최적화 가능.
- DataFrame에서 `.na` 를 사용하는 것이 null을 다루는 기본적인 방식이다.

### 6.7.1 coalesce
- coalesce 함수는 인자로 지정한 여러 컬럼 중 null이 아닌 첫번째  값을 반환

``` scala
import org.apache.spark.sql.functions.coalesce

df.select(coalesce(col("Description"), col("CustomerId"))).show()
```

### 6.7.2 `ifnull`, `nullIf`, `nvl`, `nvl2`
- `coalesce` 함수와 유사한 결과를 얻을 수 있는 몇가지 SQL 함수
- `ifnull` : 첫번째 값이 null이면 두번쨰 값 반환하고, null이 아니면 첫번째 값을 반환.
- `nullif` : 두 값이 같으면 null 반환, 다르면 첫번째 값 반환
- `nvl` : 첫번째 값이 null이면 두번쨰 값 반환
- `nvl2` :  첫번쨰 값이 null이 아니면 2번째 값을 반화하고, null이면 세번쨰 인수로 지정된 값을 반환(else_value)

``` sql
SELECT
  ifnull(null, 'return_value'),
  nullif('value', 'value'),
  nvl(null, 'return_value'),
  nvl2('not_null', 'return_value', "else_value")
FROM dfTable LIMIT 1
```

<img width="364" alt="스크린샷 2022-11-12 오후 8 24 35" src="https://user-images.githubusercontent.com/6982740/201471737-0a731e63-3d10-449d-9509-d2e7889a396e.png">


### 6.7.3 drop
- `drop` 메서드는 null 값을 가진 로우를 제거하는 가장 간단한 함수
- null 값을 가진 모든 로우를 제거

``` scala
df.na.drop()
// 로우 컬럼 값 중 하나라도 nul인경우
df.na.drop("any")
// 모든 컬럼 값이 null 또는 NaN인 경우
df.na.drop("all")
```

``` sql
SELECT *
FROM dfTable
WHERE Description IS NOT NULL
```


### 6.7.4 fill
- fill 함수를 사용해 하나 이상의 컬럼을 특정 값으로 채울 수 있습니다.

``` scala
// String 데이터 타입의 컬럼이 존재하는 null 값을 5명으로 채워 넣는 방법
df.na.fill("All Null values become this string")

// 다수의 컬럼에 적용하고 싶다면 다음으로 적용
df.na.fill(5, Seq("StockCode", "InvoiceNo"))

// Map을 이용해서 다수의 컬럼에 fill 메소드 적용
val fillColValues = Map("StockCode" -> 5, "Description" -> "No Value")
df.na.fill(fillColValues)
```

### 6.7.5 replace
- `replace` 메소드를 이용해서 다른값으로 대체할 수 있습니다.

``` scala
df.na.replace("Description", Map("" -> "UNKNOWN"))
```

## 6.8 정렬하기
- `asc_nulls_first`, `desc_nulls_first`, `asc_nulls_last`, `desc_nulls_last` 함수를 사용해 DataFrame을 정렬할때  null 값이  표시되는 기준을 지정할 수 있습니다.


## 6.9  복합 데이터 타입 다루기
### 6.9.1 구조체
- DataFrame 내부의 DataFrame을 구조체라 생각할 수 있습니다.

``` scala
df.selectExpr("(Description, InvoiceNo) as complex", "*")

df.selectExpr("struct(Description, InvoiceNo) as complex", "*")

import org.apache.spark.sql.functions.struct

val complexDF = df.select(struct("Description", "InvoiceNo").alias("complex"))
complexDF.createOrReplaceTempView("complexDF")

complexDF.select("complex.Description")
complexDF.select(col("complex").getField("Description"))

complexDF.select("complex.*")
```

- 복합 데이터 타입을 가진 DataFrame, 유일한 차이점은 문법에 `.`을 사용하거나 getField 메소드를 사용한다는 점만 차이가 있음.
- `*` 문자를 사용해 모든 값을 조회할 수 있고, 모든 컬럼을 DataFrame의 최상위 수준으로 끌어 올릴 수 있음.


### 6.9.2 배열
#### split
- 구분자를 인수로 전달해 배열로 변환

``` scala
import org.apache.spark.sql.functions.split

df.select(split(col("Description"), " ")).show(2)
```


<img width="191" alt="스크린샷 2022-11-12 오후 8 35 50" src="https://user-images.githubusercontent.com/6982740/201472116-7150347c-0d68-4aba-af09-958f0db7fc6e.png">


``` sacala
df.select(split(col("Description"), " ").alias("array_col"))
  .selectExpr("array_col[0]").show(2)
```

<img width="128" alt="스크린샷 2022-11-12 오후 8 36 05" src="https://user-images.githubusercontent.com/6982740/201472129-a33bcc13-5c95-4916-91d3-1f87581fc0d2.png">


#### 배열의 길이
``` scala
import org.apache.spark.sql.functions.size

df.select(size(split(col("Description"), " "))).show(2) // shows 5 and 3
```

#### array_contatins
- 값의 존재 유무 확인

``` scala
import org.apache.spark.sql.functions.array_contains

df.select(array_contains(split(col("Description"), " "), "WHITE")).show(2)
```

####  explode
- 배열 타입의 컬럼을 받아 포함된 모든 로우를 반환


<img width="659" alt="스크린샷 2022-11-12 오후 8 39 20" src="https://user-images.githubusercontent.com/6982740/201472250-6043f027-11a8-4e1f-8aa4-b80259d411e0.png">


``` scala
import org.apache.spark.sql.functions.{split, explode}

df.withColumn("splitted", split(col("Description"), " "))
  .withColumn("exploded", explode(col("splitted")))
  .select("Description", "InvoiceNo", "exploded").show(2)
```

<img width="326" alt="스크린샷 2022-11-12 오후 8 38 54" src="https://user-images.githubusercontent.com/6982740/201472230-37d46d56-3070-437e-96f3-995ec080dfc5.png">


### 6.9.3 Map
- `key-value` 쌍을 이용해 생성

``` scala
import org.apache.spark.sql.functions.map

df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map")).show(2)

df.select(map(col("Description"), col("InvoiceNo")).alias("complex_map"))
  .selectExpr("complex_map['WHITE METAL LANTERN']").show(2)
```


<img width="267" alt="스크린샷 2022-11-12 오후 8 41 08" src="https://user-images.githubusercontent.com/6982740/201472317-e21d56cb-6852-4bbc-89f8-a873471a5ef1.png">

## 6.10 JSON 다루기
- 스파크에서 문자열 형태의 JSON을 직접 조작하거나 파싱하여 JSON 객체로 만들 수 있습니다.

``` scala
val jsonDF = spark.range(1).selectExpr("""
  '{"myJSONKey" : {"myJSONValue" : [1, 2, 3]}}' as jsonString""")


import org.apache.spark.sql.functions.{get_json_object, json_tuple}

jsonDF.select(
    get_json_object(col("jsonString"), "$.myJSONKey.myJSONValue[1]") as "column",
    json_tuple(col("jsonString"), "myJSONKey")).show(2)

jsonDF.selectExpr(
  "get_json_object(jsonString, '$.myJSONKey.myJSONValue[1]') as column",
  "json_tuple(jsonString, 'myJSONKey')").show(2)
```

<img width="238" alt="스크린샷 2022-11-12 오후 8 42 50" src="https://user-images.githubusercontent.com/6982740/201472379-962ff209-83fe-4a77-841b-3f740b5a510b.png">

### to_json
- StructType을 JSON 문자열로 변환

``` scala
import org.apache.spark.sql.functions.to_json

df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")))

import org.apache.spark.sql.functions.from_json
import org.apache.spark.sql.types._

val parseSchema = new StructType(Array(
  new StructField("InvoiceNo",StringType,true),
  new StructField("Description",StringType,true)))

df.selectExpr("(InvoiceNo, Description) as myStruct")
  .select(to_json(col("myStruct")).alias("newJSON"))
  .select(from_json(col("newJSON"), parseSchema), col("newJSON")).show(2)
```

<img width="353" alt="스크린샷 2022-11-12 오후 8 43 54" src="https://user-images.githubusercontent.com/6982740/201472418-39134611-ea4b-4401-85c6-775431f7fc0d.png">


## 6.11 사용자 정의 함수
- 스파크의 가장 강력한 기능 중 하나는 사용 자 정의 함수(User Defined Function, UDF)를 사용할 수 있다.
- 파이썬이나 스칼라 등 외부 라이브러리를 사용해 사용자가 원하는 형태로 프랜스포메이션을 만들수 있게 한다.
- SparkSession이나 Context에서 사용할수 있도록 임시 함수 형태로 등록된다.

``` scala
val udfExampleDF = spark.range(5).toDF("num")

def power3(number:Double):Double = number * number * number

power3(2.0)
```

- 이렇게 만들어진 함수는 모든 워커 노드에서 사용하려면 등록해야 한다.
- 함수를 개발한 언어에 따라 근본적으로 동작방식이 달라질 수 있는데, 스칼라나 자바로 함수를 작성했다면 JVM 환경에서만 사용할 수 있습니다.

### 파이썬으로 UDF를 작성하는 경우
- 파이썬으로 작성한 함수라면 스파크는 워커 노드에 파이썬 프로세스를 실행하고 파이썬이 이해할 수 있는 포맷으로 모든 데이터를 직렬화해야 합니다.

<img width="648" alt="스크린샷 2022-11-12 오후 8 48 09" src="https://user-images.githubusercontent.com/6982740/201472562-7c76f4b8-1f57-4c05-9e18-fc1b3df628ff.png">

- 이 과정에서 파이썬 프로세스에 대한 부하도 있고, 데이터 직렬화 문제가 있을수 있습니다.
- 가급적이면 자바나 스칼라로 사용자 정의 함수를 작성하는 것이 좋음.

## 6.12 Hive UDF
- 하이브 문법을 사용해서 만든 UDF / UDAF도 사용할 수 있음.
- SparkSession 생성시 `.enableHiveSupport()` 명시해야함.
  - 이렇게하면 SQL로 UDF를 정의등록할 수 있음.
  - `TEMPORARY` 키워드 여부에 따라 하이브 메타스토어에 영구 함수로 등록할수도 있음.

``` sql
CREATE TEMPORARY FUNCTION myFunc AS `com.organization.hive.udf.FunctionName`
```


## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
