---
title: jenv를 활용한 JDK 버전 관리
author: sungsu park
date: 2020-05-30 16:34:00 +0800
categories: [DevLog, Java]
tags: [Java]
---


# jenv를 활용한 JDK 버전 관리
## jenv란 무엇인가?
 - jenv는 rbenv에서 사용하는 방식을 본떠서 만든  java version 관리 도구입니다.
 - 다양한 버전의 java application 관리 하는 경우에 적절히 손쉽게 version 변경할 수 있습니다.

## jenv 설치하기
```
brew install jenv
```

##  sh 설정 추가
 - jenv를 설치하면 아래와 같은 문구가 나옵니다.

<img width="557" alt="스크린샷 2020-05-30 오후 5 54 03" src="https://user-images.githubusercontent.com/6982740/83324224-c2812f00-a29e-11ea-9d3a-e060301f1f47.png">


``` sh
# for bash(terminal, iTerm)
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(jenv init -)"' >> ~/.bash_profile
source ~/.bash_profile
```

``` sh
# for zsh
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
echo 'eval "$(jenv init -)"' >> ~/.zshrc
source ~/.zshrc
```

- 여기서  ``` eval "$(jenv init -)"``` 이 부분은 사실 .bash_profile에까지 추가할 필요는 없다. 한번만 initialize 해주면 되기 때문에 profile에 넣지 말고 커맨드라인에서 한번만 실행시켜주셔도 됩니다.


## jenv에 java version 추가하기
 - Java를 개인이 원하는 별도의 위치에 설치하고 쓰는 경우도 있겠지만 자동 설치파일을 통해 설치한 경우
"/Library/Java/JavaVirtualMachines/" 하위에 들어가게 되는데요. 이를 추가해주시면 됩니다.
 - 만약 별도의 디렉토리에서 관리하고 있다면 해당 디렉토리 Home을 기준으로 추가해주면 됩니다.

``` sh
jenv add /Library/Java/JavaVirtualMachines/jdk-11.0.2.jdk/Contents/Home
jenv add /Library/Java/JavaVirtualMachines/jdk-14.jdk/Contents/Home
jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_201.jdk/Contents/Home
```

<img width="845" alt="스크린샷 2020-05-30 오후 6 04 20" src="https://user-images.githubusercontent.com/6982740/83324405-0de80d00-a2a0-11ea-913c-571b6389625e.png">

## jenv 등록된 버전 확인
``` sh
jenv versions
```
<img width="367" alt="스크린샷 2020-05-30 오후 6 15 38" src="https://user-images.githubusercontent.com/6982740/83324630-a03ce080-a2a1-11ea-8b67-88f69cee2c41.png">


## jenv 사용하기
 - jenv를 사용하는 방법은 global, local 단위로 설정하는 방법이 있다.
 - global은 말그대로 전체에 사용할 버전을 명시한것이고, local을 디렉토리 단위에서 사용할 java version을 지정하여 관리하는 방법이다.
 - global을 지정하게 되면 ```~/.jenv/version``` 파일에 사용자가 설정한 버전이 입력되어 관리된다.
``` sh
jenv global 14
```

 - local을 지정한 경우에는 지정한 디렉토리의 ``` .java-version ``` 파일이 생성되고  local java version이 관리된다. local 지정한 버전를 제거하고 싶다면 해당 파일을 삭제하면 된다.
``` sh
jenv local 14
```

<img width="769" alt="스크린샷 2020-05-30 오후 6 17 23" src="https://user-images.githubusercontent.com/6982740/83324651-d1b5ac00-a2a1-11ea-9ee6-f32a1a048d67.png">

## Reference
 -  https://github.com/jenv/jenv

