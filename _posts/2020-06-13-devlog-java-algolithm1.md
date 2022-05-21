---
title: "[HackerRank] Java Loops II"
author: sungsu park
date: 2020-06-13 16:34:00 +0800
categories: [DevLog, Algorithm]
tags: [Java, Algorithm]
---

# [HackerRank] Java Loops II
## Ploblem
<img width="578" alt="스크린샷 2020-06-13 오후 5 47 26" src="https://user-images.githubusercontent.com/6982740/84564618-faf43300-ad9d-11ea-9e9c-e5c82cbb2d06.png">

<img width="566" alt="스크린샷 2020-06-13 오후 5 47 33" src="https://user-images.githubusercontent.com/6982740/84564617-f9c30600-ad9d-11ea-8291-8c4990ccb2df.png">

## My Solution

``` java
import java.util.*;
import java.io.*;
import java.lang.Math;

class Solution{
    public static void main(String []argh){
        Scanner in = new Scanner(System.in);
        int t=in.nextInt();
        for(int i=0;i<t;i++){
            int a = in.nextInt();
            int b = in.nextInt();
            int n = in.nextInt();

            for(int j=1;j<=n;j++) {
                int value = a;
                for(int k=0;k<j;k++) {
                    value += Math.pow(2, k) * b;
                }
                System.out.printf("%d ", value);
            }

            System.out.println("");
        }
        in.close();
    }
}
```

## Best Solution
 - 내가 문제를 보면서 푸는것 말고 더 좋은 방법이 없나 검색해보니 더 좋은 방법이 있었다.
 - ```2^0 + 2^1 + ... 2^j = 2^(j+1) - 1``` 수학 공식을 이용해 푸는 방법인데 저는 재귀함수를 써야하나 loop 한번 더 돌아야할것 같다고만 생각했네요

``` java
class Solution{
    public static void main(String []argh){
        Scanner in = new Scanner(System.in);
        int t=in.nextInt();
        StringBuilder sb = new StringBuilder();
        for(int i=0;i<t;i++){
            int a = in.nextInt();
            int b = in.nextInt();
            int n = in.nextInt();
            sb.setLength(0);
            for(int j=0; j<n; ++j) {
                // 2^0 + 2^1 + ... 2^j = 2^(j+1) - 1
                sb.append((int) (a + b*(Math.pow(2, j+1) - 1))).append(" ");

            }
            System.out.println(sb.toString());
        }
        in.close();
    }
}
```

## Code Repo
 - [https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_Loops_II.java](https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_Loops_II.java)


## Refference
 - [https://www.hackerrank.com/challenges/java-loops/forum/comments/214175](https://www.hackerrank.com/challenges/java-loops/forum/comments/214175)
