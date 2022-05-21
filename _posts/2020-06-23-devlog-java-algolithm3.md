---
title: "[HackerRank] Java Anagrams"
author: sungsu park
date: 2020-06-23 16:34:00 +0800
categories: [DevLog, Algorithm]
tags: [Java, Algorithm]
---

# [HackerRank] Java Anagrams
## Ploblem
<img width="974" alt="스크린샷 2020-06-24 오전 2 33 07" src="https://user-images.githubusercontent.com/6982740/85435956-36270b00-b5c3-11ea-81f8-dbcb4997bd93.png">

<img width="900" alt="스크린샷 2020-06-24 오전 2 33 26" src="https://user-images.githubusercontent.com/6982740/85435966-38896500-b5c3-11ea-9707-3594a5074d7e.png">

## My solution
 - char별로 카운트해야 된다고 생각해서 Map에 넣고 Count를 한다음에 각 맵끼리 비교하는것으로 문제를 풀었다.


``` java
private void anagram(String a, String b) {
		boolean anagram = isAnagram(a,b);
		if(anagram) {
			System.out.println("Anagrams");
		} else {
			System.out.println("Not Anagrams");
		}
	}

	private boolean isAnagram(String a, String b) {
		if(isValidParam(a, b)) {
			return false;
		}

		Map<Character, Integer> aCountMap = makeCountMap(a);
		Map<Character, Integer> bCountMap = makeCountMap(b);

		log.debug("aCountMap : {}", aCountMap);
		log.debug("bCountMap : {}", bCountMap);
		return aCountMap.equals(bCountMap);
	}

	private Map<Character, Integer> makeCountMap(String a) {
		Map<Character, Integer> countMap = new HashMap<>();
		for(Character c : a.toLowerCase().toCharArray()) {
			Integer count = Optional.ofNullable(countMap.get(c))
				.map(cnt -> cnt + 1)
				.orElse(1);
			countMap.put(c, count);
		}
		return countMap;
	}

	private boolean isValidParam(String a, String b) {
		return (a.length() <= 1 && a.length() <= 50 && b.length() <=1 && b.length() <= 50);
	}
```

## Simple solution
 - 다른사람들이 손쉽게 풀이한 문제를 공유한다.
 - 공간복잡도도 더 조금쓰면서 코드도 심플..(시무룩)

``` java
private void othersAnagram(String a, String b) {
		if (a.length() != b.length()) {
			System.out.println("Not Anagrams");
			return;
		}
		char[] a1 = a.toLowerCase().toCharArray();
		char[] a2 = b.toLowerCase().toCharArray();
		Arrays.sort(a1);
		Arrays.sort(a2);

		if(Arrays.equals(a1, a2)) {
			System.out.println("Anagrams");
		} else {
			System.out.println("Not Anagrams");
		}
	}
```

## Code Repo
- [https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_Anagrams.java](https://github.com/sungsu9022/study-algorithm/blob/master/src/test/java/com/sungsu/algorithm/hackerrank/Java_Anagrams.java)

## Refference
 - [https://www.baeldung.com/java-strings-anagrams](https://www.baeldung.com/java-strings-anagrams)
