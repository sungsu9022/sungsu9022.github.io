---
title: OSX에서 MongoDB 설치하기
author: sungsu park
date: 2020-06-08 16:34:00 +0800
categories: [DevLog, MongoDB]
tags: [Infra, MongoDB]
---

# OSX에서 MongoDB 설치하기
## 1) brew를 통한 mongoDB 설치
 - OSX에 Mongo DB를 설치하는 방법은 다양합니다. 공식 사이트( [https://www.mongodb.com/download-center/community](https://www.mongodb.com/download-center/community) ) 에서 tgz 파일을 다운받아서 설치할수도 있는데요. OSX를 사용한다면 brew와 같은 패키지 매니저를 통해 여러 애플리케이션을 설치하고 관리하는것이 조금더 일반적이라 brew를 이용한 방법으로 작성하였습니다.

``` bash
brew install mongodb-community
```

- mongodb로 설치하는 경우 설치가 되지 않는 문제가 있는데요. homebrew PR-43770( [https://github.com/Homebrew/homebrew-core/pull/43770](https://github.com/Homebrew/homebrew-core/pull/43770) )  에 의해 제거되었기 떄문입니다.

``` bash
brew install mongodb
```

- 추가로 혹시 인스톨상에 문제가 있다면 아래와 같이 시도해보세요.

``` bash
brew services stop mongodb
brew uninstall mongodb

brew tap mongodb/brew
brew install mongodb-community
```

## 2) mongoDB 간략 설정

 - brew를 통해 mongodb가 설치되었다면 ```/usr/local/etc/mongod.conf``` 파일이 생성된것을 알수 있습니다.
 - 해당 파일의 log와 dbPath를 적절한 위치로 수정합니다.

``` conf
# 적절히 설정 변경
systemLog:
  destination: file
  path: /Users/user/logs/mongodb/mongo.log
  logAppend: true
storage:
  dbPath: /Users/user/data/mongodb
net:
  bindIp: 127.0.0.1
```

## 3) mongoDB 실행
### 버전 확인
``` sh
mongo -version
```

<img width="388" alt="스크린샷 2020-06-04 오전 11 45 53" src="https://user-images.githubusercontent.com/6982740/83709383-5bc29380-a659-11ea-8801-af33f1396cb2.png">

### (3.1) mongoDB server 실행
 - brew를 이용해 실행시키는 방법(이 경우는 ```/usr/local/etc/mongod.conf``` 설정을 들고 뜨도록 되어있습니다.)

``` sh
brew services start mongodb-community
```

<img width="916" alt="스크린샷 2020-06-04 오전 11 55 46" src="https://user-images.githubusercontent.com/6982740/83709863-5fa2e580-a65a-11ea-9195-8f94305ccf16.png">

 -  mongod를 직접 사용하여 server를 띄워도 됩니다.

``` sh
mongod --config /usr/local/etc/mongod.conf &

ps -ef | grep mongo
```

<img width="679" alt="스크린샷 2020-06-04 오전 11 57 51" src="https://user-images.githubusercontent.com/6982740/83709982-a264bd80-a65a-11ea-9450-385b3487656a.png">

### (3.2) mongoDB server 종료
``` sh
brew services stop mongodb-community
```

 - mongod를 직접 사용하여 프로세스를 올린 경우에는 아래와 같이 프로세스를 종료시킬수 있습니다.

```  sh
# commend parsing error 발생
mongod --shutdown

kill <mongod process ID>
```

 - ``` mongod --shutdown```은 document 업데이트가 안된것인지 작동하지 않아 그냥 kill을 이용해서 프로세스를 종료하시면 됩니다.


### (3.3) mongoDB client
``` sh
mongo
```

<img width="997" alt="스크린샷 2020-06-04 오후 12 04 52" src="https://user-images.githubusercontent.com/6982740/83710412-a1805b80-a65b-11ea-9006-2e2be8870d2c.png">



## Refference
 - [https://docs.mongodb.com/manual/tutorial/manage-mongodb-processes/#stop-mongod-processes](https://docs.mongodb.com/manual/tutorial/manage-mongodb-processes/#stop-mongod-processes)
 - [https://stackoverflow.com/questions/57856809/installing-mongodb-with-homebrew](https://stackoverflow.com/questions/57856809/installing-mongodb-with-homebrew)
