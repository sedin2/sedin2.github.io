---
title: Redis를 이용한 Ranking Board
date: 2024-03-16 00:00:00 +09:00
categories: [redis, cache, 캐시]
tags: [redis]
image: /assets/img/redis.png
---

### 📌소개

이전 [포스트](https://sedin2.github.io/posts/redis-data-types-02/)에서 다룬 Redis의 Sorted Set을 활용해 간단한 Ranking Board를 만들어보자.  
동점자 처리까지 해보기

---

### 🚀프로젝트 세팅

**build.gradle**에 의존성 추가

```
# build.gradle

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'  # redis

    //...
}
```

**RedisTemplate**을 **Spring Bean**에 등록

```java
// RedisConfiguration.java

@Configuration
@EnableRedisRepositories
public class RedisConfiguration {

    @Value("${spring.redis.port}")
    private int port;

    @Value("${spring.redis.host}")
    private String host;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();

        // key, value에 대해 Serializer 세팅
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());

        return redisTemplate;
    }
}
```

```
# application.properties

spring.redis.port=6379
spring.redis.host=localhost
```

**Redis 응답**을 담을 **ResponseRankingDto** 작성

```java
// ResponseRankingDto.java

@Getter
@NoArgsConstructor
@AllArgsConstructor
public class ResponseRankingDto {

    private String member;

    private double score;

    public static ResponseRankingDto convertToResponseRankingDto(ZSetOperations.TypedTuple<String> tuple) {
        return new ResponseRankingDto(tuple.getValue(), tuple.getScore());
    }

}
```

---

### ✅테스트1. **`동점자가 없을 때`**

```java
@SpringBootTest
class RankingServiceTest {

    private final static String KEY = "ranking";

    @Autowired
    private RedisTemplate redisTemplate;

    @AfterEach
    public void tearDown() {
        Set<String> keys = redisTemplate.keys("*");
        redisTemplate.delete(keys);
    }

    @Test
    @DisplayName("Sorted Set - 동점자가 없을 때")
    void withoutSameScores() {
        // given
        redisTemplate.opsForZSet().add(KEY, "cafe", 80);
        redisTemplate.opsForZSet().add(KEY, "latte", 60);
        redisTemplate.opsForZSet().add(KEY, "mocha", 100);
        redisTemplate.opsForZSet().add(KEY, "americano", 70);
        redisTemplate.opsForZSet().add(KEY, "cold brew", 90);

        // when
        Long rank1 = getRank("mocha");
        Long rank2 = getRank("latte");

        // then
        assertThat(rank1).isEqualTo(1);  // 1등
        assertThat(rank2).isEqualTo(5);  // 5등
    }

    private Long getRank(String name) {
        return redisTemplate.opsForZSet().reverseRank(KEY, name) + 1; // index가 0부터 시작해서 1을 더해준다
    }
}
```

![성공 - 동점자 없을 때](/assets/img/success_without_same_scores.png)

**동점자가 없을 땐, 랭킹을 잘 구해온다.**

---

### ✅테스트2. **`동점자가 있을 때`**

```java
@SpringBootTest
class RankingServiceTest {

    //...
    @Test
    @DisplayName("Sorted Set - 동점자가 있을 때")
    void withSameScores() {
        // given
        redisTemplate.opsForZSet().add(KEY, "cafe", 80);
        redisTemplate.opsForZSet().add(KEY, "latte", 80);
        redisTemplate.opsForZSet().add(KEY, "mocha", 100);
        redisTemplate.opsForZSet().add(KEY, "americano", 80);
        redisTemplate.opsForZSet().add(KEY, "cold brew", 60);

        // when
        Long rank1 = getRank("cafe");
        Long rank2 = getRank("latte");
        Long rank3 = getRank("americano");

        // then
        // 2등 3명을 조회
        assertAll(
                () -> assertThat(rank1).isEqualTo(2),   // 3등
                () -> assertThat(rank2).isEqualTo(2),   // 2등
                () -> assertThat(rank3).isEqualTo(2)    // 4등
        );
    }

    private Long getRank(String name) {
        return redisTemplate.opsForZSet().reverseRank(KEY, name) + 1; // index가 0부터 시작해서 1을 더해준다
    }
}
```

![실패 - 동점자 있을 때](/assets/img/fail_with_same_scores.png)
mocha - 100 -> 1등  
cafe, latte, americano - 80 -> **공동 2등 3명**  
cold brew - 60 -> 5등  
**opsForZSet().reverseRank 메서드**는 **같은 스코어일 경우 value값을 기준으로 정렬**하여 리턴해서 실패한다.  
따라서 **정확한 랭킹을 구하기 위해서는 추가적인 처리**가 필요하다.

---

### ✅테스트3. **`동점자가 있을 때(추가 로직)`**

```java
@SpringBootTest
class RankingServiceTest {

    //...
    @Test
    @DisplayName("Sorted Set - 동점자가 있을 때")
    void withSameScores() {
        // given
        redisTemplate.opsForZSet().add(KEY, "cafe", 80);
        redisTemplate.opsForZSet().add(KEY, "latte", 80);
        redisTemplate.opsForZSet().add(KEY, "mocha", 100);
        redisTemplate.opsForZSet().add(KEY, "americano", 80);
        redisTemplate.opsForZSet().add(KEY, "cold brew", 60);

        // when
        Long rank1 = getRank("cafe");
        Long rank2 = getRank("latte");
        Long rank3 = getRank("americano");

        // then
        // 2등 3명을 조회
        assertAll(
                () -> assertThat(rank1).isEqualTo(2),   // 2등
                () -> assertThat(rank2).isEqualTo(2),   // 2등
                () -> assertThat(rank3).isEqualTo(2)    // 2등
        );
    }

    private Long getRank(String name) {
        Long ranking = 0L;
        Double score = redisTemplate.opsForZSet().score(KEY, name); // 해당 이름의 스코어를 구해온다

        /*
            구해온 스코어 범위 내에 속하는 멤버들을 역순으로 가져오는 메서드
            여기선 offset이 0, count가 1이므로 역순으로 정렬된 순서대로 첫 번째 멤버 집합을 가져온다
         */
        Set<String> highRank = redisTemplate.opsForZSet().reverseRangeByScore(KEY, score, score, 0, 1);

        for (String highRankName : highRank) {
            ranking = redisTemplate.opsForZSet().reverseRank(KEY, highRankName);    // 해당 멤버의 랭킹을 구한다
        }

        return ranking + 1; // index가 0부터 시작되어서 1 더해준다
    }
}
```

![성공 - 동점자 있을 때](/assets/img/success_with_same_scores.png)

위 케이스에선 80점 스코어인 멤버는 3명이 있다.  
80점 스코어 집합 - **`{"latte", "cafe", "americano"}`**

예를들어 **cafe**의 순위를 구할 때,

1. cafe로 **score 80을 구한다.**
2. 해당 socre 80으로 집합에서 **첫 번째 멤버만 포함한 집합(**`{"latte"}`**)을 구한다.**
3. 구해온 집합(**`{"latte"}`**)을 통해 **다시 랭킹을 구한다.**

---

### 📝마무리

이렇게 Redis를 통해 간단한 랭킹 보드를 만들어 봤다.  
손쉽게 구현 가능했지만, 같은 스코어가 있을 때 정확한 랭킹을 구하기가 까다로웠다.  
추가적인 로직 처리가 필요했는데 이는 각자의 상황이나 비즈니스 로직에 맞게 구현해서 사용하면 될 것 같다.

---

### 🌐References

- [Redis](https://redis.io/docs/)

---
