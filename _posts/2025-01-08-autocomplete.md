---
title:  "Redis를 활용한 검색 자동완성 기능 구현"
layout: single
categories: Redis
tag: [Java, Redis]
toc: true
toc_sticky: true
author_profile: false
sidebar:
    nav: "docs"
---
# 참고
이 글은 Spring boot 3.0를 사용하여 Redis 코드를 구현

# Redis의 ZSet 사용
Redis의 ZSet 기능을 이용하여 자동완성 기능을 구현할 수 있다.  
* Redis는 key와 value로 구성이 되어있고 value에는 ZSet의 자료 구조가 있다.  
* ZSet은 자바의 Set처럼 중복 허용이 안되어 있고 오름차순으로 나열된 List라고 보면 된다.  
* b, c, a, c 순서대로 데이터를 넣을 시  
  확인해보면 [a, b, c] 이렇게 구성되어 있다.  
* 데이터를 넣을 시 score라는 점수도 함께 넣는데  
  작은 점수 순서대로 오름차순이 된다.  
  b의 score가 0  
  c의 score가 2  
  a의 score가 1 이면  
  [b, a, c] 로 구성이 되어있다.  
* 단순 글자 오름차순을 원하면 점수를 다 똑같이 주면 된다.  

# 자동완성 구현
상품을 등록한다고 쳤을 때 DB뿐만 아니라 redis에도 데이터를 넣는다.  
예를 들어 "라이터"와 "라이더자켓"을 redis에 등록하는 과정을 보자  
```java
List<String> makePrefixes(String term) {
    List<String> results = new ArrayList<>();
    for (int i = 1; i <= term.length(); i++) {
        results.add(term.substring(0, i)); // 글자 분해
    }
    return results;
}
```
위 코드에서 보면 term에 "라이터"와 "라이더자켓"이 들어갈 시  
List에 [라, 라이, 라이터] , [라, 라이, 라이더, 라이더자, 라이더자켓]  
이렇게 담긴다.  
<br>
List에 담긴 분할된 글자들을 Redis의 key로 설정하고 value값을 넣는다.  
아래는 Redis에 자동완성 데이터 삽입 코드다.  
```java
@Service
@RequiredArgsConstructor
public class SearchProductService {
    private final RedisTemplate<String, String> redisTemplate;
    private static final String AUTO_COMPLETE_KEY = "autocomplete:";

    // term에 "라이터"와 "라이더자켓" 담기
    public void add(String term) {
        List<String> prefixes = makePrefixes(term); // 여기서 글자분해하여 List에 담기
        for (String prefix : prefixes) {
            String redisKey = AUTO_COMPLETE_KEY + prefix; // 분해된 글자 key로 설정
            redisTemplate.opsForZSet().add(redisKey, term, Integer.MAX_VALUE);
        }
    }
}
```
"라이터", "라이더자켓" 순으로 넣으면
* 라 : [라이터]  
  라이 : [라이터]  
  라이터 : [라이터]  
* 라 : [라이더자켓, 라이터]  
  라이 : [라이더자켓, 라이터]  
  라이더 : [라이더자켓]  
  라이더자 : [라이더자켓]  
  라이더자켓 : [라이더자켓]  

여기서 "라"와 "라이"는 둘다 일치하므로 한 ZSet에 담는다.  
그리고 "라이더자켓"이 "라이터"보다 글자 오름차순이다.  
<br>
opsForZSet()은 ZSet을 가리키며  
add에서 (key, value, score) 순서대로 넣는다.  
필자는 score에 max int 값을 넣고 해당 자동완성 데이터 선택시 -1씩 숫자를 줄여 우선순위를 높였다.  
<br>
Redis에 담긴 키를 호출시 아래코드는 이렇다.  
```java
 private static final String AUTO_COMPLETE_KEY = "autocomplete:";

 public List<String> getAutoComplete(String keyword) {
      String redisKey = AUTO_COMPLETE_KEY + keyword; // 호출할 키

      return redisTemplate.opsForZSet().range(redisKey, 0, 9);
}
```
결과
* "라" 입력시 [라이더자켓, 라이터]가 출력  
* "라이" 입력시 [라이더자켓, 라이터]가 출력  
* "라이터" 입력시 [라이터]가 출력  
<br>
* range에서 (key, start, end) 이렇게 값을 넣고  
  ZSet에 담긴 리스트의 index 0번부터 9번까지 총 10개의 데이터만 호출  
* range에서 전체를 불러오고 싶을 땐 end에 -1을 넣는다. 
