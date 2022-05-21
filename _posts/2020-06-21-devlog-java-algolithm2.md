---
title: "[HackerRank] Java String Reverse"
author: sungsu park
date: 2020-06-21 16:34:00 +0800
categories: [DevLog, Algorithm]
tags: [Java, Algorithm]
---

# [HackerRank] Java String Reverse
## Ploblem
<img width="894" alt="스크린샷 2020-06-21 오후 7 12 16" src="https://user-images.githubusercontent.com/6982740/85221993-45575e80-b3f3-11ea-8bf8-972f63835b3d.png">

## My solution
 - 예전에 동일한 문제 해결 방법으로 Stack에 넣었다가 pop()으로 reverse 한다는 이야기를 들은적이 있어서 그렇게 구현

``` java
private void solution(String s) {
		if(isPalindrome(s)) {
			System.out.println("Yes");
		} else {
			System.out.println("No");
		}
	}

	private boolean isPalindrome(String s) {
		Stack<Character> stack = new Stack<>();
		for (char c : s.toCharArray()) {
			stack.push(c);
		}

		StringBuilder sb = new StringBuilder();
		while(!stack.isEmpty()) {
			sb.append(stack.pop());
		}

		return sb.toString().equals(s);
	}
```

## Simple solution
 - StringBuilder 객체로 만든 뒤 reverse() 호출

``` java
private void solution2(String s) {
		System.out.println( s.equals( new StringBuilder(s).reverse().toString()) ? "Yes" : "No" );
	}
```

## 추가 내용
  - 내가 풀이한 방법대로 했을때 공간복잡도를 생각했을때 더 많이 사용될 수 있음. Stack 인스턴스를 만들어야 하기 떄문.. 만약 StringBulder 없이 하더라면 비슷할것으로 생각된다.
 - StringBuilder.reverse() 코드를 보면 for-loop를 돌면서 처리한다.

## Code Repo
- [https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_String_Reverse.java](https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_String_Reverse.java)

## Refference
 - [https://www.hackerrank.com/challenges/java-string-reverse/forum](https://www.hackerrank.com/challenges/java-string-reverse/forum)
