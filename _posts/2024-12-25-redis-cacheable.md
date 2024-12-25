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
**Redis 세팅**
```java
@Configuration
public class RedisConfig {
    @Value("${spring.data.redis.host}")
    private String host;

    @Value("${spring.data.redis.port}")
    private int port;

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(new RedisStandaloneConfiguration(host, port));
    }
}
```
Lettuce라는 라이브러리를 활용해 Redis 연결을 관리하는 객체를 생성하고 Redis의 host와 port를 넣는다.  
<br/>
<br/>
### Redis 사용
#### Service의 구성  
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
* @Cacheable은 [캐시 사용 흐름](#캐시-사용-흐름)처럼 캐시에 데이터가 없으면 DB에서 데이터를 가져온다.  
* cacheNames : 이름을 지정함으로써 각각의 다른 캐시 공간을 사용한다. 'getBoards'라는 별도의 공간에서 key와 value 사용  
* key : boards:page:1:size:10 이런 형태로 key가 저장이 되며 #page, #size는 아래의 매개변수 page와 size를 가져와 동적으로 표현  
* cacheManager : Redis의 캐시를 관리하는 [CacheManager](#cachemanager의-구성)에 접근한다. 메서드명 boardCacheManager에 접근

<br/>
#### CacheManager의 구성  
**아래의 방법은 엔터티의 특정 필드를 전역적으로 직렬화/역직렬화 설정하는 방식**  
```java
@Configuration
@EnableCaching // Spring Boot의 캐싱 설정을 활성화
public class RedisCacheConfig {
    @Bean
    public CacheManager boardCacheManager(RedisConnectionFactory redisConnectionFactory) {
        ObjectMapper objectMapper = new ObjectMapper()
                .registerModule(new JavaTimeModule()) // Java 8 날짜 및 시간 지원
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS); // ISO-8601 형식으로 날짜 직렬화

        Jackson2JsonRedisSerializer<Object> serializer = new Jackson2JsonRedisSerializer<>(objectMapper, Object.class);

        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(serializer))
                .entryTtl(Duration.ofMinutes(1L));

        return RedisCacheManager
                .RedisCacheManagerBuilder
                .fromConnectionFactory(redisConnectionFactory)
                .cacheDefaults(redisCacheConfiguration)
                .build();
    }
}
```
* 빈에 등록된 [RedisConfig](#간단한-redis-세팅)의 설정값을 매개변수로 사용  
* Redis에 접근할 때 데이터를 주고 받는 형식 등을 설정한다.  
* @EnableCaching : Spring Boot의 캐싱 설정을 활성화
* serializeKeysWith는 key 저장시 StringRedisSerializer를 사용하여 String으로 직렬화해서 저장한다.
  * 문자열로 저장된 key를 가져올 때 역직렬화를 한다.
  * 만일 설정안하면 key는 바이너리 형태로 저장이 된다.
* entryTtls는 만료시간(TTL) 설정
  * 1L은 1분을 가리킴

1. ObjectMapper를 사용하여 특정 클래스를 커스터마이징한다.
2. ObjectMapper를 Jackson2JsonRedisSerializer에 대입
3. serializeValuesWith는 value 저장시 Jackson2JsonRedisSerializer를 사용하여 Json으로 직렬화해서 저장한다.
  * Jackson2JsonRedisSerializer는 객체를 JSON 문자열로 변환하여 저장하고, JSON 문자열을 다시 객체로 역직렬화하는 데 사용
    *  여기서 객체는 엔터티를 가리키고 클래스 형태의 엔터티를 json 형태로 직렬화하여 변환
  * json으로 저장된 value는 가져올 때 역직렬화를 한다
  * 만일 설정안하면 value는 바이너리 형태로 저장이 된다.  

<br/>
**엔터티**
```java
@Entity
@Table(name = "boards")
@Getter
public class Board {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    private String content;

    private LocalDateTime createdAt;
}
```
* LocalDateTime은 redis에 저장시 적합하지 않은 데이터 타입이지만 ObjectMapper를 통해 커스터마이징하여 변환
* 역직렬화시 LocalDateTime에 맞추어 변환  

<br/>
**아래의 방법은 엔터티의 특정 필드를 직렬화/역직렬화 설정하는 방식**  
```java
@Configuration
@EnableCaching // Spring Boot의 캐싱 설정을 활성화
public class RedisCacheConfig {
  @Bean
  public CacheManager boardCacheManager(RedisConnectionFactory redisConnectionFactory) {
    RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
        .defaultCacheConfig()
	      // Redis에 Key를 저장할 때 String으로 직렬화(변환)해서 저장
        .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
        // Redis에 Value를 저장할 때 Json으로 직렬화(변환)해서 저장
        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new Jackson2JsonRedisSerializer<Object>(Object.class)))
        // 데이터의 만료기간(TTL) 설정(1분)
        .entryTtl(Duration.ofMinutes(1L));

    return RedisCacheManager
        .RedisCacheManagerBuilder
        .fromConnectionFactory(redisConnectionFactory)
        .cacheDefaults(redisCacheConfiguration)
        .build();
  }
}
```
* 빈에 등록된 [RedisConfig](#간단한-redis-세팅)의 설정값을 매개변수로 사용  
* Redis에 접근할 때 데이터를 주고 받는 형식 등을 설정한다.  
* @EnableCaching : Spring Boot의 캐싱 설정을 활성화  
* serializeKeysWith는 key 저장시 StringRedisSerializer를 사용하여 String으로 직렬화해서 저장한다.
  * 문자열로 저장된 key를 가져올 때 역직렬화를 한다.
  * 만일 설정안하면 key는 바이너리 형태로 저장이 된다.
* serializeValuesWith는 value 저장시 Jackson2JsonRedisSerializer를 사용하여 Json으로 직렬화해서 저장한다.
  * Jackson2JsonRedisSerializer는 객체를 JSON 문자열로 변환하여 저장하고, JSON 문자열을 다시 객체로 역직렬화하는 데 사용
    *  여기서 객체는 엔터티를 가리키고 클래스 형태의 엔터티를 json 형태로 직렬화하여 변환
  * json으로 저장된 value는 가져올 때 역직렬화를 한다
  * 만일 설정안하면 value는 바이너리 형태로 저장이 된다.
* entryTtls는 만료시간(TTL) 설정
  * 1L은 1분을 가리킴

<br/>
**엔터티**  
```java
@Entity
@Table(name = "boards")
@Getter
public class Board {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    private String content;

    @JsonFormat(pattern = "yyyy-MM-dd' 'HH:mm:ss")
    @JsonSerialize(using = LocalDateTimeSerializer.class)
    @JsonDeserialize(using = LocalDateTimeDeserializer.class)
    private LocalDateTime createdAt;
}
```  
* @JsonFormat은 json으로 변환시 원하는 형태로 변환
* @JsonSerialize은 redis에 value 저장시 LocalDateTime은 적합하지 않기 때문에 직렬화 해줘야함
* @JsonDeserialize은 redis에 저장된 직렬화 데이터를 역직렬화 해준다.
  * 역직렬화시 기본적으로 배열형태로 변환하기 때문에 형태를 맞추기 위해 @JsonFormat 사용
* @JsonSerialize, @JsonDeserialize이 없으면 Redis 사용시 에러

<br/>
#### 참고
@RestController를 통해 객체를 response시 스프링이 LocalDateTime타입은 'yyyy-MM-dd'T'HH:mm:ss' 형태로 자동으로 직렬화하여 내보낸다.  
반대로 request시에도 자동으로 역직렬화하여 값을 읽어들인다.  
그래서 단순히 전송시 @JsonSerialize, @JsonDeserialize 사용할 필요는 없다.
