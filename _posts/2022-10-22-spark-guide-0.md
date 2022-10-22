---
title: "[스파크 완벽 가이드] 0. 스파크 시작하기"
author: sungsu park
date: 2022-10-22 15:46:00 +0800
categories: [DevLog, Spark]
tags: [Spark]

---

# 스파크 시작하기
- 곧 업무에서 스파크를 사용해야 해서 스파크를 재대로 공부해봐야겠다고 생각했다.
- 그래서 일단 local 환경에서 spark를 설치하고, 간단한 작업들을 해보려고 한다.

## 1. 스파크 설치
> 2020 m1 osx 를 기준으로 작성되었음을 알려드립니다.

``` sh
# java  설치
arch -arm64 brew install openjdk@11

# scala 설치
arch -arm64 brew install scala

# Apache spark
arch -arm64 brew install apache-spark
```

- 이렇게만 설치하면 `spark-shell`을 사용할 준비가 모두 끝났습니다.

## 2. 트러블 슈팅
- 위처럼 설칠하고 spark-shell을 실행시켜보니 정상적으로 구동되지 않았다.

<img width="1427" alt="스크린샷 2022-10-22 오후 3 37 51" src="https://user-images.githubusercontent.com/6982740/197324431-23f7ce89-ffc0-4575-bd8e-b39a781b9926.png">

- 원인은 hostname 설정 관련 문제가 있는듯 했다.

``` sh
sudo hostname -s 127.0.0.1
```

## 3. `spark-shell` 실행

``` sh
import spark.implicits._
val data = Seq(("Java", "20000"), ("Python", "100000"), ("Scala", "3000"))
val df = data.toDF()
df.show()
```

<img width="1015" alt="스크린샷 2022-10-22 오후 3 42 32" src="https://user-images.githubusercontent.com/6982740/197324586-ddf96487-fff5-49e9-87d0-3997d71c00d5.png">

- 이제 책 예제를 실행시켜볼 준비 완료!




## Reference
- https://sparkbyexamples.com/spark/install-apache-spark-on-mac/
- https://stackoverflow.com/questions/34601554/mac-spark-shell-error-initializing-sparkcontext

