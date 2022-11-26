---
title: "[스파크 완벽 가이드] 9. 데이터소스"
author: sungsu park
date: 2022-11-25 21:52:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---


# [스파크 완벽 가이드] 9. 데이터소스
## 9.1 데이터소스 API 구조
- 특정 포맷을 읽고 쓰는 방법을 알아보기 전에 데이터소스 API 전체적인 구조를 알아보자.

### 9.1.1 읽기 API 구조
``` scala
dataframe.write.format("csv")
  .option("mode", "OVERWRITE")
  .option("dateFormat", "yyyy-MM-dd")
  .option("path", "path/to/file(s)")
  .save()
```

- 스파크에서는 모든 데이터소스를 읽을때 위와 같은 포맷을 사용한다.
- `format`을 선택적으로 사용할수 있고, default는 파케이 포맷을 사용 한다.

### 9.1.2 데이터 읽기 기초
- 스파크에서 데이터를 읽을때는 기본적으로 DataFrameReader를 사용하여` SparkSession.read` 소성으로 접근한다.
- 전바적인 코드 구성은 아래 포맷을 참고하면 된다.

``` scala
spark.read.format("csv")
  .option("header", "true")
  .option("mode", "FAILFAST")
  .option("inferSchema", "true")
  .load("some/path/to/file.csv")
```

#### 읽기 모드
- 외부 데이터소스에서 데이터를 읽다보면 자연스럽게 형식에 맞지 않는 데이터를 만날 수 있다.
  -` 읽기 모드`란 형식에 맞지 않는 데이터를 만났을때의 동작방식을 지정하는 옵션이다.


<img width="609" alt="스크린샷 2022-11-20 오후 5 44 06" src="https://user-images.githubusercontent.com/6982740/202893256-f699094c-1191-4cba-8b79-fdcd2128e7b1.png">

- 기본값은  `permissive`이다.

### 9.1.3 쓰기 API 구조
- format은 읽기와 마찬가지로 default. 파케이 포맷입니다.
- `partitionBy`, `bucketBy`, `sortBy`는 파일 기반 데이터소스에서만 동작한다.

``` scala
DataFrameWriter
    .format(...)
    .option(...)
    .partitionBy(...)
    .bucketBy(...)
    .sortBy(...)
    .save()
```

### 9.1.4 데이터 쓰기의 기초
- 읽기 API와 매우 유사하고, DataFrameReader 대신 DataFrameWriter를 사용하면 된다.

``` scala
csvFile.write
   .format("csv")
   .mode("overwrite")
   .option("sep", "\t")
  .save("/tmp/my-tsv-file.tsv")
```

#### 저장 모드
- `저장 모드`란 스파크가 지정된 위치에서 동일한 파일이 발견되었을ㄷ 떄의 동작방식을 지정하는 옵션입니다.

<img width="600" alt="스크린샷 2022-11-20 오후 5 48 38" src="https://user-images.githubusercontent.com/6982740/202893403-1af5e584-ef8b-4cce-9986-acbfdbf50253.png">

- 기본값은 `errorIfExists`


## 9.2 CSV 파일
- `,` 구분된 데이터 포맷

### 9.2.1 CSV 옵션
- CSV reader에서 사용할 수 있는 옵션

<img width="414" alt="스크린샷 2022-11-20 오후 5 50 08" src="https://user-images.githubusercontent.com/6982740/202893483-f8a90698-cc91-43e7-82c0-c14ad62c1886.png">

<img width="406" alt="스크린샷 2022-11-20 오후 5 50 18" src="https://user-images.githubusercontent.com/6982740/202893487-15e60263-1799-4269-afe4-d9a00eae3ff1.png">


### 9.2.2 CSV 파일 읽기

``` scala
import org.apache.spark.sql.types.{StructField, StructType, StringType, LongType}

val myManualSchema = new StructType(Array(
  new StructField("DEST_COUNTRY_NAME", StringType, true),
  new StructField("ORIGIN_COUNTRY_NAME", StringType, true),
  new StructField("count", LongType, false)
))

spark.read.format("csv")
  .option("header", "true")
  .option("mode", "FAILFAST")
  .schema(myManualSchema)
  .load("/data/flight-data/csv/2010-summary.csv")
  .show(5)
```

- 위와 같이 schema 정의하고 모드를 `FAILFAST`로 지정했으므로,  스키마와 데이터가 일치하지 않는 경우 오류가 발생합니다.
  - 컴파일 단계에서는 알수 없고, 스파크가 실행되는 RunTime에 오류가 발생함.


### 9.2.3 CSV 파일 쓰기
- `maxColumns`, `inferSchema` 옵션 같이 데이터 쓰기에는 적용되지 않는 옵션을 제외하면 동일한 옵션을 제공한다.

#### csv 파일을 읽어서 tsv 파일로 내보내기

``` scala

val csvFile = spark.read.format("csv")
  .option("header", "true").option("mode", "FAILFAST").schema(myManualSchema)
  .load("/data/flight-data/csv/2010-summary.csv")


csvFile.write.format("csv").mode("overwrite").option("sep", "\t")
  .save("/tmp/my-tsv-file.tsv")
```

## 9.3 JSON 파일
### JSON파일을 Spark에서 다룰때 중요한 부분
- JSON 파일 내부는 개행으로 구분된 것을 기본으로 합니다.
- JSON 객체나 배열을 하나씩 가지고 있는 파일을 다루는 것에 차이를 두고 처리해야 함.
- `multiLine` 옵션을 통해 줄로 구분된 방식과 여러줄로 구성된 방식을 선택적으로 사용할 수 있음.

### 9.3.1 JSON 옵션

<img width="549" alt="스크린샷 2022-11-20 오후 6 15 37" src="https://user-images.githubusercontent.com/6982740/202894392-1254902b-02b3-4669-97a2-f42e25ee7a87.png">

<img width="535" alt="스크린샷 2022-11-20 오후 6 15 46" src="https://user-images.githubusercontent.com/6982740/202894405-43a84e4e-e77d-4eb2-a993-e6b88cb1caf0.png">

### 9.3.2 JSON 파일 읽기
``` scala
spark.read
    .format("json")
    .option("mode", "FAILFAST")
    .schema(myManualSchema)
    .load("/data/flight-data/json/2010-summary.json")
    .show(5)
```

### 9.3.3 JSON 파일 쓰기
- input 데이터소스에 관계 없이 JSON 파일에 저장할 수 있습니다.

``` scala
csvFile.write
   .format("json")
   .mode("overwrite")
   .save("/tmp/my-json-file.json")
```

## 9.4 파케이(Parquet) 파일
- 다양한 스토리지 최적화 기술을 제공하는 오픈소스로 만들어진 컬럼 기반 데이터 저장 방식
- 분석 워크로드에 최적화되어있고, 저장소 공간을 절약할 수 있음.
- 또한, 전체 파일을 읽는 대신 개별 컬럼을 읽을수있으며, 컬럼 기반의 압축 기능을 제공한다.
- 스파크와 잘 호환되기 떄문에 기본 파일 포맷이기도 함.

### 9.4.1 파케이 파일 읽기
- 데이터를 저장할 때 자체 스키마를 사용해 데이터를 저장하기 때문에 옵션이 거의 없다.
- 따라서 포맷을 설정하는 것만으로도 충분하다.

``` scala
spark.read
   .format("parquet")
   .load("/data/flight-data/parquet/2010-summary.parquet")
   .show(5)
```

#### 파케이 옵션
- 옵션이 거의 없지만, 간혹 호환되지 않은 파케이 파일이 존재할 수 있는데, 이는 다른 버전(특히 오래된 버전)의 스파크를 사용해 만든 파케이 파일의 경우 조심해야 한다는 점 외에 특이사항은 없다.

<img width="540" alt="스크린샷 2022-11-20 오후 6 21 19" src="https://user-images.githubusercontent.com/6982740/202894583-39723bc9-1840-484a-93c0-7ebf7a3fa7d0.png">

### 9.4.2 파케이 파일 쓰기
``` scala
csvFile.write
   .format("parquet")
   .mode("overwrite")
   .save("/tmp/my-parquet-file.parquet")
```

## 9.5 ORC 파일
- ORC는 하둡 워크로드를 위해 설계된 자기 기술적(self-describing)이고, 데이터 타입을 인식할 수 있는 컬럼 기반의 파일 포맷
- 파케이와 매우 유사한하나 근본적인 차이점은 스파크에 최적화되어있느냐 하이브에 최적화되어있느냐 차이가 있다.

### 9.5.1 ORC 파일 읽기
``` scala
spark.read
   .format("orc")
   .load("/data/flight-data/orc/2010-summary.orc")
   .show(5)
```

### 9.5.2 ORC 파일 쓰기
``` scala
csvFile.write
   .format("orc")
   .mode("overwrite")
   .save("/tmp/my-orc-file.orc")
```

## 9.6 SQL 데이터베이스
- SQL 데이터베이스는 매우 강력한 커넥터 중 하나.
- 사용자는 SQL을 지원하는 다양한 시스템에 SQL 게이터소스를 연결할 수 있습니다.(MySQL, ProstgreSQL, Oracle 등)

### 데이터베이스 연결
- 스파크 classpath에 데이터베이스 JDBC 드라이버를 추가하고, 적ㅈ러한 JDBC 드라이버 jar파일을 제공해야 함.

``` sh
./bin/5park-Shell \
--driver-class-path postgresql-9.4.1207.jar \
--jars postgresql-9.4.1207.jar
```

### JDBC 데이터소스 옵션

<img width="539" alt="스크린샷 2022-11-20 오후 6 28 32" src="https://user-images.githubusercontent.com/6982740/202894840-89ce5bd8-f7ff-4cfb-aabf-b2b15a9203e8.png">

<img width="536" alt="스크린샷 2022-11-20 오후 6 28 39" src="https://user-images.githubusercontent.com/6982740/202894850-0d362c27-1e80-4071-a9e8-f6bc033041ec.png">

### 9.6.1 SQL 데이터베이스 읽기
``` scala
val driver =  "org.sqlite.JDBC"
val path = "/data/flight-data/jdbc/my-sqlite.db"
val url = s"jdbc:sqlite:/${path}"
val tablename = "flight_info"

import java.sql.DriverManager
val connection = DriverManager.getConnection(url)
connection.isClosed()
connection.close()
```

``` scala
val dbDataFrame = spark.read
   .format("jdbc")
   .option("url", url)
   .option("dbtable", tablename)
   .option("driver",  driver).load()
```

- 위와 같은 방식으로 생성된 DataFrame은 기존에 생성된 DataFrame과 전혀다르지 않음.

### 9.6.2 쿼리 푸시다운
- 스파크에서는 DataFrame을 만들기 전에 데이터베이스 자체에서 데이터를 필터링하도록 만들 수 있습니다.
- 쿼리 실행계획을 보면 테이블의 컬럼 중 관련 있는 컬럼만 선택한다는것을 알수 있습니다.

``` scala
dbDataFrame
    .select("DEST_COUNTRY_NAME")
   .distinct()
   .explain
```

<img width="438" alt="스크린샷 2022-11-20 오후 6 36 15" src="https://user-images.githubusercontent.com/6982740/202895116-ce4b41f4-c3b4-4ff0-8bc0-99382cdf1b54.png">


#### SQL 쿼리 명시
- 모든 스파크 함수를 SQL 데이터베이스에 맞게 변환하지는 못하기 때문에  SQL 쿼리를 직접 명시해서 처리하는 경우도 필요할 수 있습니다.


``` scala
val pushdownQuery = """(SELECT DISTINCT(DEST_COUNTRY_NAME) FROM flight_info)
  AS flight_info"""

val dbDataFrame = spark.read.format("jdbc")
  .option("url", url).option("dbtable", pushdownQuery).option("driver",  driver)
  .load()
```

#### 데이터베이스 병렬로 읽기
- 스파크는 파일 크기, 유형, 압축 방식에 따른 분할 가능성에 따라 여러 파일을 읽어 하나의 파티션으로 만들거나 여러 파티션을 하나의 파일로 만드는 기본 알고리즘을 가지고 있습니다.
- SQL 데이터베이스에서는 `numPartitons` 옵션을 사용해 읽기 및 쓰기용 동시작업 수를 제한 할 수 있습니다.

``` scala
val dbDataFrame = spark.read
   .format("jdbc")
   .option("url", url)
   .option("dbtable", tablename)
   .option("driver", driver)
   .option("numPartitions", 10)
   .load()
```

#### 슬라이딩 윈도우 기반의 파티셔닝
- 조건절을 기반으로 분할할수 있는 방법을 제공합니다.
- lowerBound(min), upperBound(max) 값과 `numPartitions`을 기준으로 쿼리를 N번으로 쪼개서 병렬처리 가능.

``` scala
val colName = "count"
val lowerBound = 0L
val upperBound = 348113L // this is the max count in our database
val numPartitions = 10

spark.read
   .jdbc(url,tablename,colName,lowerBound,upperBound,numPartitions,props)
  .count() // 255
```

### 9.6.3 SQL 데이터베이스 쓰기
- 데이터 쓰는것은 읽기만큼 쉬움.

``` scala
val newPath = "jdbc:sqlite://tmp/my-sqlite.db"
csvFile.write
   .mode("overwrite")
   .jdbc(newPath, tablename, props)

```

## 9.7 텍스트 파일
- 스파크에서는 일반 텍스트 파일도 읽을 수 있습니다.
  - 파일의 각 줄은 DataFrame의 레코드로 맵핑됨.

### 9.7.1 텍스트 파일 읽기
- `textFile` 메소드에 텍스트 파일을 지정하기만 하면 됨.
  - 파티션 수행 결과로 만들어진 디렉터리명을 무시(파티션된 텍스트 파일을 읽거나 쓰려면 `text` 메소드 사용

``` scala
spark.read.textFile("/data/flight-data/csv/2010-summary.csv")
  .selectExpr("split(value, ',') as rows").show()
```

<img width="166" alt="스크린샷 2022-11-26 오후 9 27 21" src="https://user-images.githubusercontent.com/6982740/204088912-db8aa999-546a-46e4-b2bc-e9a7a1702da0.png">

### 9.7.2 텍스트 파일  쓰기
- 텍스트 파일을 쓸때는 문자열 컬럼이 하나만 존재해야 함.(아닌 경우 작업 실패)
- 텍스트 파일에 데이터를 저장할때 파티셔닝을 수행하면 더 많은 컬럼을 저장할 수 있음.
  - 디렉터리에 컬럼별로 별도 저장됨

``` scala
csvFile.select("DEST_COUNTRY_NAME").write.text("/tmp/simple-text-file.txt")

csvFile.limit(10).select("DEST_COUNTRY_NAME", "count")
  .write.partitionBy("count").text("/tmp/five-csv-files2.csv")
```

## 9.8 고급 I/O 개념
- 쓰기 작업 전 파티션 수를 조절함으로써 병렬 처리할 파일 수를 제어할 수 있음.
  - `버텟팅`과 `파티셔닝`의 조절

### 9.8.1 분할 가능한 파일 타입과 압축 방식
- 특정 파일 포맷은 기본적으로 분할을 지원함.
  - 따라서 스파크에서 전체 파일이 아닌 쿼리에 필요한 부분만 읽을수 있어 성능 향상됨.
- 스파크에서는 `파케이` 파일 포맷과 `GZIP` 압축 방식을 권장함.

### 9.8.2 병렬로 데이터 읽기
- 여러 익스큐터가 같은 파일을 동시에 읽을 수는 없지만, 여러 파일을 동시에 읽을 수는 있음.(?)
  - 다수의 파일이 존재하는 폴더를 읽을ㄷ 때 폴더의 개별 파일은 Dataframe의 파티션이 되는데 이걸 기준으로 병렬로 읽기가 가능

### 9.8.3 병렬로 데이터 쓰기
- 파일이나 데이터 수는 쓰는 시점에 DataFrame이 가진 파티션 수에 따라 달라질 수 있음.
- 기본적으로 데이터 파티션당 하나의 파일이 작성됨.
  - 실제로 옵션에 지정된 파일명은 다수의 파일을 가진 디렉터리

``` scala
// 디렉토리 안에 5개의 파일이 생성
csvFile.repartiton(5).write.format("csv").save("/tmp/multiple.csv")
```

<img width="519" alt="스크린샷 2022-11-26 오후 9 40 54" src="https://user-images.githubusercontent.com/6982740/204089462-b2e6238f-13e7-41b4-b047-c7d18a6dc96b.png">

#### 파티셔닝
- 어떤 데이터를 어디에 저장할 것인지 제어할 수 있는 기능
- 파티셔닝된 디렉터리 또는 테이블에 파일을 쓸때 디렉터리별로 컬럼 데이터를 인코딩해 저장함.
  -  데이터를 읽을 때 전체 데이터셋을 스캔하지 않고 필요한 컬럼의 데이터만 읽을수 있는 이유.

``` scala
csvFile.limit(10).write.mode("overwrite").partitionBy("DEST_COUNTRY_NAME")
  .save("/tmp/partitioned-files.parquet")
```

<img width="300" alt="스크린샷 2022-11-26 오후 9 44 00" src="https://user-images.githubusercontent.com/6982740/204089561-be379dab-5e75-434d-a562-e4cd287e74f1.png">

- 각 폴더는 조건절을 폴더명으로 사용하며, 만족한 데이터가 저장된 파케이 파일을 가지고 있다.
- 파티셔닝은 필터링을 자주 사용하는 테이블을 가진 경우에 사용할 수 있는 가장 손쉬운 최적화 방법.

#### 버켓팅
- 각 파일에 저장된 데이터를 제어할 수 있는 또 다른 파일 조직화 기법
- 동일한 버킷 ID를 가진 데이터가 하나의 물리적 파티션에 모두 모여 있게 하므로 데이터를 읽을때 셔플을 피할 수 있다.
- 특정 컬럼을 파티셔닝했을때 수억개의 디렉터리가 만들어질수도 있는데, 이런 경우  `버켓팅` 방법을 찾아야 한다.

``` scala
// 버켓 단위로 데이터를 모아 일정 수의 파일로 저장하는 예제

val numberBuckets = 10
val columnToBucketBy = "count"

csvFile.write.format("parquet").mode("overwrite")
  .bucketBy(numberBuckets, columnToBucketBy).saveAsTable("bucketedFiles")
```

### 9.8.4 복합 데이터 유형 쓰기
- 스파크는 다양한 자체 데이터 타입을 제공하는데, 이는 스파크에서는 잘 동작하지만, 모든 데이터 파일 포맷에 적합하지는 않음.
  - CSV파일은 복합 데이터 타입을 지원하지 않음
  - 파케이나 ORC에서는 지원함.

### 9.8.5 파일 크기 관리
- 파일크기는 데이터를 저장할 때는 중요한 요소는 아니다.
- 하지만, 데이터를 읽을때는 중요한 소요

#### 적은 크기의 파일 문제(작은 크기의 파일이 많은 경우)
- 메타데이터에 엄청난 관리 부하가 발생할 수 있음.
- HDFS 등 많은 파일 시스템에서 작은 크기의 많은 파일을 잘 다루지 못한다.(스파크도 포함)

#### 파일크기가 큰 파일을 다루는 경우
- 몇개의 로우가 필요하더라도 전체 데이터 블록을 읽어야 하기 때문에 비효율적이므로 지양해야 함.

#### `maxRecordsPerFile`(파일당 레코드 수)
- 스파크 2.2 부터 해당 기능을 사용해서 각 파일에 기록될 레코드 수를 조절할 수 있으므로 파일 크기를 효과적으로 제어할 수 있다.

``` scala
df.write.option("maxRecordsPerFile", 5000)
```

## Reference
- [Spark 완벽 가이드](https://www.coupang.com/vp/products/164359777?itemId=471497435&vendorItemId=4215695264&src=1042503&spec=10304982&addtag=400&ctag=164359777&lptag=10304982I471497435&itime=20221030164504&pageType=PRODUCT&pageValue=164359777&wPcid=16589055075836517214634&wRef=&wTime=20221030164504&redirect=landing&gclid=Cj0KCQjwwfiaBhC7ARIsAGvcPe7kxytxjJU9Ylxpe5l8Jk9zhXhknDFceRzD80Zn6IzxUaF-RPn5OKAaAnGxEALw_wcB&campaignid=18626086777&adgroupid=)
