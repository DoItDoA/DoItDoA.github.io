---
title:  "Redis의 Cacheable 사용"
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
이 글은 Spring boot 3.0과 Jpa를 사용하여 Redis 코드를 구현

# 캐시 사용 흐름
![cache_miss](/assets/images/redis/cache_miss.png)  
위의 그림과 같이 캐시에 데이터가 없으면 데이터베이스로부터 가져오고 캐시에 저장을 한다.  
그리고 다음에 데이터 가져올 시 캐시로부터 데이터를 가져와 빠르게 조회가 가능하다.  

# Redis 구성
key와 value 형태로 데이터를 구성한다.  
Java의 Map처럼 해당 key를 통해 value 값을 호출하는 방식이다.  
<br/>
Redis의 key 네이밍 방식은 단순히 한 단어를 써서 키이름으로 저장이 가능하지만 (ex : users)    
콜론(:)을 사용해 계층적으로 의미를 구분하여 많이 사용한다.  
(ex : users:100:profile) <- 사용자들(users)중에서 PK가 100인 프로필(profile)  
장점으로 가독성과 서로 다른 key와 겹쳐 충돌할 일이 적어진다.  


# 사용코드
### 간단한 Redis 세팅
코드 사용전에 아래의 코드를 build.gradle에 추가  
```
implementation 'org.springframework.boot:spring-boot-starter-data-redis'
```
application.properties에 아래 코드 추가  
```
data.redis.host=localhost
data.redis.port=6379
```
### Redis 자바 사용코드
Service의 구성  
```java
@Service
@RequiredArgsConstructor
public class BoardService {
    private final BoardRepository boardRepository;

    @Cacheable(cacheNames = "getBoards", key = "'boards:page:' + #page + ':size:' + #size", cacheManager = "boardCacheManager")
    public List<Board> getBoards(int page, int size) {
        // boardRepository로부터 데이터 호출
    }
}
```
@Cacheable은 위 그림의 [ㅁㅁㅁㅁ](#캐시-사용-흐름) 캐시처럼
