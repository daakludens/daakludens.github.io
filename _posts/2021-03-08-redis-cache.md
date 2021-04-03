---
title: "부하 분산과 성능 개선을 위한 캐시 적용"
excerpt: "redis, cache"
comments: true

categories:
  - Project
last_modified_at: 2021-03-31
---

## 왜 Cache를 사용하게 되었는가?
첫 번째로 로그인과 회원 관리 기능을 구축한 후 본격적으로 게임 목록을 가져오는 로직 구축을 시작했습니다. 해당 기능은 사용자가 로그인하면 db에 저장되어 있는 게임 리스트를 가져와 보여줍니다. 단순 조회 기능이라 SELECT * from문을 사용하고 보니 문제점이 생겼습니다. 제 서비스는 수 천명의 사용자가 동시에 여러 게임 데이터를 조회한다는 가정 하에 작성하고 있습니다. 사용자가 점점 늘어나게 된다면 db에 엄청난 부담을 주면서 성능이 느려집니다. 몇 초의 로딩 시간에도 불만을 가질 수 있는데 만약 대기 시간이 길어진다면 사용자들은 더 나은 환경의 게임 사이트를 찾아 떠날 겁니다. 

처음에는 SELECT 쿼리문에 페이징 처리로 한 번에 불러오는 게임 수를 제한했습니다. 하지만 접근 개수를 줄인 거지 db에 가해지는 부하는 똑같이 적용됩니다. 그래서 부하 분산을 위해서 캐싱을 도입하기로 했습니다.

<br>

## Cache
MySQL 같은 디스크 저장소에서 데이터를 불러오려면 네트워크를 타고 접근해 쿼리를 파싱하는 등 복잡한 연산이 생깁니다. 캐시는 고속 저장소로 대용량 데이터, 복잡한 수학적 연산 결과, 정적 컨텐츠 등을 연산 없이 데이터를 불러올 수 있어 CPU 기능 부하와 지연 시간을 줄여줄 수 있고 퍼포먼스를 향상 시킬 수 있습니다. 보통 RAM이나 인메모리 엔진 같이 가볍고 빠른 하드웨어에 설치되어 실행됩니다.

이렇게만 본다면 느린 디스크 대신 캐시 만을 활용하는 게 오히려 효율적으로 보입니다. 하지만 캐시 적용에 중요한 부분 중 캐시 히트율과 인메모리 db의 특징에 대해 주목했습니다. 

<p align="center"><img src="/assets/images/cache-hit-fail.png" with="800" height="400"></p>

<br>

캐시 히트율(cache hit rate)은 요청이 들어왔을 때 캐싱된 데이터가 있는 확률로, 히트된 개수에서 전체 응답 수를 나눠서 계산합니다. 캐시 히트율이 높다는 건 그만큼 캐싱 데이터가 쓰여서 유용성을 입증할 수 있습니다. 하지만 무조건 높다는 게 좋은 건 아닙니다. DB인 MySQL과 캐시 DB인 Redis는 실시간 싱크가 이뤄지지 않기에 바로 반영된 데이터가 필요한 경우 캐시 사용이 적합하지 않을 수 있습니다. @CachePut과 @CacheEvict를 사용해 캐시의 UPDATE나 DELETE가 가능하지만 프로세스 중 변경된 데이터가 반영되는 시간 차이가 발생합니다. 

때문에 데이터 일관성을 위해서 TTL(Time To Live)을 이용해 일정 시간이 지나면 캐시를 소멸하고 다음 요청 때 db에서 가져온 값을 저장해 주기적으로 업데이트할 수 있습니다. 또한 캐시는 휘발성 저장소입니다. 높은 IOPS(Input/Output Operations Per Second)을 가져 요청에 대한 입출력이 빠르게 진행되지만 예기치 않은 에러로 종료가 된다면 데이터는 삭제됩니다. 데이터의 일관성과 안정성을 위해서 디스크 db와 캐시를 함께 사용하는 경우가 많습니다.

<br>
<br>

## 로컬 캐시와 글로벌 캐시
캐싱 타입은 여러 종류가 있지만 그 중에서 자주 언급되는 로컬 캐시(local cache)와 글로벌 캐시(distributed cache)에 대해 알아보겠습니다.

로컬 캐시는 애플리케이션 메모리 안에 캐싱한 데이터를 보관하는 저장소입니다. 저장 가능한 용량은 크지 않지만 로컬에서 빠르게 접근이 가능해 지연 시간을 줄일 수 있습니다. 이런 특징은 잦은 호출 빈도 수를 가지는 작은 크기의 데이터를 호출하는 서비스에 적합합니다.

글로벌 캐시는 두 개 이상의 애플리케이션이 공통된 캐시 저장소를 가집니다.

<br>

## Redis vs. Memcached
캐시용 인메모리 db를 찾게 되면 가장 많이 마주하는 게 Redis와 Memcached입니다. 두 기능 모두NoSQL 형식으로 키-값 형태를 이룹니다. 현재 제가 사용하고 있는 자바 언어의 클라이언트를 제공하며 Memcached는 Xmemcached과 Memcached-java-client를 제공하고 Redis는 Jedis, Lettuce, Redisson을 제공합니다. 또한 늘어나는 데이터 양을 수용할 확장성을 제공해줍니다.

언뜻 보기에는 비슷해 보이지만 차이점이 여럿 있습니다.

### 데이터 자료형과 그에 따른 메모리 사용량           
Memcached는 key와 value가 String 자료형으로 최대 1MB까지 저장합니다. Redis는 5개의 데이터 자료형(String, Hash, List, Set, Sorted set)을 사용하며 키와 값 모두 512MB까지 저장 가능합니다.
String으로만 구성된 Memcached는 일반적으로 Redis보다 빠르지만 Redis의 Hash로 직렬화, 역직렬화 과정을 거치지 않고 객체를 저장할 수 있어 애플리케이션 개발이 수월해지고 IO 과정이 줄어들어 효율적입니다.

### 구조와 확장법                
Memcached는 멀티 코어 구조로 된 멀티 쓰레드를 지원합니다. 따라서 용량을 늘리려면 코어나 쓰레드를 추가하는 수평멀티 쓰레드를 지원합니다. 따라서 용량을 늘리려면 코어나 쓰레드를 추가하는 수평적 확장이 가능합니다. 큰 데이터셋을 다뤄야 하는 경우 Redis보다 더 빠른 성능을 보입니다.
그에 비해 Redis는 싱글 쓰레드 구조이며 수평적, 수직적 확장 둘 다 가능합니다. 수평적 확장의 경우 노드 그룹 (샤드) 개수를 조정하면 되고, 수직적 확장은 노드의 클러스터 크기를 늘리면 됩니다.

### 데이터 지속성                
데이터 지속성(data persistence)은 어떤 변화 후에도 데이터의 유지 여부를 뜻하며 가변적(volatile)이지 않은 데이터베이스에 담겨져야 지속성을 가질 수 있습니다.
Memcached는 완전한 인메모리 캐시로 volatile한 성격을 띕니다. 반면 Redis는 완전한 인메모리가 아닌 데이터 스토어로 데이터 지속성을 유지할 수 있는 방법이 2개 있습니다.
먼저 RDB snapshot는 특정 시간에 모든 데이터셋을 스냅샷으로 찍어 디스크 내에 있는 파일에 저장하는 기능입니다. 복원할 데이터 양이 많아질 수록 응답 시간이 길어지는 특징이 있어 AOF가 사용될 수 있습니다. AOF(Append-Only File) log는 레디스 서버에서 수행된 모든 쓰기 기능을 기록하며 디스크에 순서대로 적히기 때문에 후에 그대로 수행하면 데이터를 복원할 수 있다. 지워지면 안 되는 데이터의 경우 매우 유용한 기능이다. 파일에 append로 계속해 붙이기 때문에 데이터가 변질될 가능성은 없지만 레디스를 계속 돌릴 수록 RDB보다 저장할 데이터가 훨씬 빨리 증가합니다.

### 데이터 축출 정책에 대한 알고리즘                        
데이터 축출 정책(eviction policy)은 메모리에 여유 공간이 남지 않았을 때 축출되는 대상의 우선 순위를 정하는 방법을 뜻합니다. Memcached는 가장 오랫동안 사용되지 않은 데이터를 축출하는 LRU(Least Recently Used)만 지원되지만 Redis는 8가지 정책 중 고를 수 있습니다. 
알고리즘 | 방식
---- | ----
noeviction | 메모리가 다 찬 경우 에러 표시
allkeys-LRU | 가장 사용되지 않은 데이터 축출
volatile-LRU | 가장 사용되지 않음 + 만료 기간 설정
allkeys-random | 랜덤하게 축출
volatile-random | 랜덤 + 만료 기간 축출
volatile-TTL | 제일 짧은 TTL + 만료 기간 설정
volatile-lfu | 제일 사용되지 않음 + 만료 기간 설정(Redis 4.0)
allkeys-lfu | 제일 사용되지 않는 데이터 축출(Redis 4.0)

출처 : [https://redis.io/topics/lru-cache](https://redis.io/topics/lru-cache)

<br>

### 트랜잭션                       
Memcached는 원자성(atomic)은 있지만 트랜잭션을 제공하지 않는 반면 Redis는 트랜잭션 기능을 제공합니다.

<br>

## 선택
Cache를 적용하기 위해 Redis를 사용하기로 했습니다. 
이미 세션 분산 이슈로 Redis를 사용하고 있는 상태이고, Spring에서는 3.1 버전부터 cache를 기존에 있던 스프링 애플리케이션에 코드 변화 없이 적용할 수 있도록 해줍니다. 
또한 데이터 저장 시 다양한 자료형을 지원하고 IO 횟수를 줄일 수 있는 Redis가 유용하다고 봤습니다.
데이터 지속성 옵션의 경우는 MySQL로 replication을 설치할 예정이라 영속성을 보장할 필요가 없으니 기능이 필요없다고 판단되어 사용하지 않았습니다.

<br>

## 적용
먼저 Redis 캐시를 적용하기 위해 CacheManger를 configure 해줘야 합니다.
```
@Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration
                .defaultCacheConfig()
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext
                        .SerializationPair
                        .fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext
                        .SerializationPair
                        .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfiguration = new HashMap<>();
        cacheConfiguration.put(GAME_LIST, redisCacheConfiguration.entryTtl(Duration.ofSeconds(5)));

        return RedisCacheManager
                .RedisCacheManagerBuilder
                .fromConnectionFactory(redisConnectionFactory)
                .cacheDefaults(redisCacheConfiguration)
                .build();
    }
```

<br>

제 경우는 게임 리스트를 별도로 캐시하기 위해 GAME_LIST라는 이름과 5초 후에 캐싱 데이터를 삭제하는 TTL을 지정해줬습니다.
그리고 캐시를 사용하고 싶은 메서드에 @Cacheable을 사용하면 첫 요청 시에 디스크에서 가져온 데이터를 저장해 다음 호출 때 저장된 데이터를 그대로 불러오게 됩니다. Redis는 키-값 형태로 데이터를 저장하기에 키를 argument로 받는 listInfo로 지정하고, 위에서 지정했던 redisCacheManager를 캐시 매니저로 지정해주면 됩니다.

```
@Cacheable(key = "#listInfo", value = GAME_LIST, cacheManager = "redisCacheManager")
@LoginCheck(authLevel = AuthLevel.USER)
@GetMapping("/")
public List<GameDto> selectGameList(GamePagingDto listInfo) {

    return gameService.getGameList(listInfo);
}
```

<br>

프로젝트 링크 : [링크](https://github.com/f-lab-edu/ludensdomain)

### 출처
1. https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache
2. https://aws.amazon.com/caching/?nc1=h_ls
3. https://www.imaginarycloud.com/blog/redis-vs-memcached/
4. https://www.baeldung.com/memcached-vs-redis#:~:text=Memcached and Redis,-Often%2C we think&text=Memcached is a distributed memory,%2C message broker%2C and queue
5. https://docs.oracle.com/cd/E18686_01/coh.37/e18677/cache_intro.htm#COHDG320
6. https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/scaling-redis-cluster-mode-enabled.html
