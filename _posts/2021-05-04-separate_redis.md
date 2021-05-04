---
title: "세션과 캐시 분리를 위한 Redis 분리와 Docker 사용"
excerpt: "Redis, Docker"
comments: true

categories:
  - Project
last_modified_at: 2021-05-04
---
## 세션과 캐시를 모두 담은 Redis

레디스는 디스크 IO보다 빠른 속도와 가용성을 보장하는 인메모리 db입니다. 이 장점들 때문에 제 프로젝트의 세션과 캐시로 설정했습니다. 
지금으로썬 문제가 없긴 하지만 여기서 더 좋은 성능을 낼 수 있는 환경을 만들 수 있을지 고민해봤습니다.

Redis는 현재 나온 6버전까지 많은 사용자들의 요구에도 불구하고 `single thread` 구조를 이어나가고 있습니다. 
Single thread는 원자성(atomic)을 가지며 한 번에 하나의 command만 수행해 빠른 속도를 낼 수 있습니다. 
하지만 순차적으로 오는 command는 앞의 command가 끝나지 않으면 대기해야 합니다. 

Command가 밀리지 않을 만큼 처리 속도가 빠르면 괜찮지만, 만약 서비스 내 트래픽이 너무 늘어나서 대기 상태가 길어지게 되면 어떨까요? 
그리고 많은 유저가 유입되는 동시에 데이터 캐싱을 해야 한다면? 
전혀 관계 없는 기능들이 단지 single thread 구조의 한 서버에 있다는 이유로 대기해야 하는 비효율적인 상황에 놓입니다.

<br>

이렇게 현 구조에서 발생할 수 있는 문제를 짚어보자면

1. 동시 요청이 많아질 수록 전체 애플리케이션의 대기 시간(latency)이 증가한다.        
세션과 캐시는 독립적인 기능이며 동시에 요청이 들어올 수 있는 작업이다.
2. Redis가 A라는 기능 때문에 다운될 경우 잘 수행 중이던 B 기능까지 중단된다.            
불필요한 복구 과정과 시간이 연장되며 서비스 운영에 차질
3. 기능 확장 시 다른 기능 때문에 제약이 걸린다.           
두 개의 상이한 기능을 동시에 확장하려면 복잡도 상승

<br>

해결책으로 간단하게 두 기능을 분리 시켜 각각의 독립적인 주체로 만들어 주기로 했습니다.
여러 Redis 서버 인스턴스를 생성해 넣어주면 각 기능에만 집중하며 필요 시 확장 또는 관리가 가능한 환경이 형성됩니다. 
스프링을 배우면서 한 번 쯤 들어봤을 `OCP(Open-Closed Principle)` 개념이 이 상황에도 통용될 수 있을 것 같네요.

기존에 설치 됐던 Redis는 session용으로 두고 새로 만드는 인스턴스에 cache 기능을 넣기로 했습니다.

<br>

## 첫번재 서버 - Session
복기 겸 제가 설치했던 방법을 다시 한 번 적어봤습니다.

Redis 웹사이트에 들어가셔서 다운을 받으면 되고, 만약 Windows를 쓴다면 [Windows 전용 사이트](https://github.com/microsoftarchive/redis/releases/tag/win-3.0.504)에서 다운 받으면 됩니다. 

제가 글을 쓸 때 최신 LTS 버전은 3.0.504여서 해당 버전을 이용했습니다.
설치하실 때 더 최신 버전이 있는지 한 번 확인하시고 설치하세요.

redis-cli를 이용하고 싶다면 cmd를 통해 설치된 파일에 들어가 `redis-cli.exe`를 가동 시킵니다.
포트번호는 기본으로 6379이니 그대로 써줍니다.

```jsx
cd C:/Program Files/redis
redis-cli.exe -p 6379
```

명령어 Enter를 쳤을 때 커서가 [로컬 IP 주소(localhost)]:[포트 번호]로 바뀌게 되면 정상 작동하는 겁니다.

<br>

## 두번째 서버 - Cache
이번 Redis는 Docker에 가동시키려 합니다. 
Docker는 사용해보니 한 두줄의 명령어로 서버를 올려 바로 사용할 수 있는 편리함과 개발 시간 단축이 됩니다.

우선 Docker Hub에 가입하고 Docker Desktop을 다운 받습니다. 
Docker는 기본적으로 이미지를 먼저 생성한 다음 이미지를 토대로 컨테이너를 생성하는 과정을 거칩니다. 먼저 이미지를 만듭니다. 

참고로 여기서 먼저 docker search redis를 입력해서 Docker에서 공식 기능으로 인정하는 이미지를 사용하는 게 안전 상 제일 좋습니다. 
official[ok]가 붙어 있는 이미지를 고르면 됩니다(사실 해당 이미지가 유저가 주는 별도 제일 많기도 하고요). 
간단하게 docker pull redis 명령어를 치면 됩니다. 

하지만 Redis서버는 기본적으로 하나의 인스턴스가 모든 메모리를 사용합니다. 
설령 인스턴스를 나눴어도 메모리 전체를 소요하려 들기 때문에 메모리 설정이 필요해 보였습니다. 
또한 Memcached와 달리 여러 캐시 알고리즘을 제공하는 Redis이기에 캐시 알고리즘을 변경해보고 싶었습니다.

Redis의 서버 설정을 바꾸려면 redis.conf 파일에서 하면 됩니다. 
직접적으로 수정한 redis.conf를 사용하는 Redis 이미지를 만들기 위해서 파일을 Dockerfile로 생성했습니다. 

<br>

### redis.conf 생성
Docker에서 제공하는 기본 Redis 이미지는 redis.conf 파일이 없습니다. 
그래서 github의 공식 사이트에서 볼 수 있습니다. 
다만 tag가 unstable이라 조금 불안한 관계로 redis에서 제공하는 redis.conf stable 버전을 다운 받아 사용했습니다.         
URL : [https://download.redis.io/redis-stable/redis.conf](https://download.redis.io/redis-stable/redis.conf)

중간 즈음 내려가면 `MEMORY MANAGEMENT`라는 이름으로 메모리 관련 설정 칸이 보입니다.
필요한 부분만 주석 해제를 하고 메모리 최대 사용량과 이빅션(eviction) 정책을 설정합니다.

```java
maxmemory 2mb
maxmemory-policy allkeys-lfu
```

<br>

### Dockerfile 생성
Dockerfile은 Docker에서 build 명령어를 내릴 때 이미지 생성에 참고하는 일종의 가이드라인입니다.
제가 쓴 Dockerfile 내용은 [도커 공식 문서](https://hub.docker.com/_/redis)에 나와 있는 명령어를 썼습니다.

```java
FROM redis
COPY redis.conf /usr/local/etc/redis/redis.conf
CMD [ "redis-server", "/usr/local/etc/redis/redis.conf" ]
```

<br>

명령어를 해석하자면 redis 이미지를 다운 받아 현재 위치의 redis.conf 파일을 지정한 경로에 복사한 다음 파일을 토대로 redis-server을 실행시킨다는 의미입니다.

<br>

### Docker에서 이미지와 컨테이너 생성
이제 이미지와 컨테이너를 만들어 줍니다.
방금 만든 redis.conf와 Dockerfile을 한 폴더에 넣어 cli로 해당 폴더 위치에서 시작하면 됩니다.

여러 개의 동일한 DB 서버를 띄우려면 포트 번호가 달라야 합니다. 
세션용 포트 번호는 6379이니 다른 포트 번호로 설정하면 됩니다. 

Docker build 명령어에서 현재 위치에서 build를 한다는 걸 명시하기 위해 뒤에 .을 붙여줘야 합니다.
그리고 컨테이너를 생성할 때 컨테이너의 이름과 포트 번호만 지정해주면 됩니다.

```java
docker build -t ludens-cache .

docker run -d -p 6380:6379 --name ludens-cache ludens-cache
```

<br>

`docker ps` 명령어를 입력하면 현재 실행되는 도커 컨테이너를 확인할 수 있습니다. 
리스트에 아까 만든 컨테이너가 보인다면 성공입니다.

이제 저희가 설정한 redis.conf도 잘 적용됐는지 확인 해봐야죠.

```java
docker exec -it ludens-cache bash
```

위 명령어로 컨테이너로 들어가 `/usr/local/etc/redis` 경로를 통해 문서를 확인할 수 있습니다.

<br>

## 마치며
사실 현 단계에서는 단순히 redis 서버 분리만 필요한 기능일지도 모릅니다. 
하지만 미리 튜닝할 수 있는 부분을 고려하며 처음 설치하는 것도 굉장히 중요하다 생각했습니다.

마지막으로 Redis 관련해 여기 저기 찾다가 [Reddit](https://www.reddit.com/r/redis/comments/5q5ddr/learn_redis_the_hard_way_in_production/)에서 봤던 어떤 분의 말로 이번 블로그 글을 마치려고 합니다.

> I have never found a software component that could handle any kind of data backup/persistence policy, could handle any command in its client/server command set, and could handle any configuration setting, and do these things at all scales from one command per minute up to a million commands per second. 
They all require you to do something different at different points on that scale. **You can't just install the software and never think about it again.**

제가 종종 놓치는 포인트이기도 합니다. 처음 시도하거나 너무 어려웠던 기술을 적용하고 나면 뿌듯함과 동시에 한동안 안 봐도 되겠지? 하면서 방치해두곤 합니다.
하지만 서비스를 키워나가며 동적으로 변하는 외부 환경과 내부 시스템적인 변화로 제가 처음에 쓴 기술이 오히려 발목을 잡을 수 있을 거란 생각이 듭니다.
모든 상황에 최고의 설정은 없으며 항상 변화에 기민하게 반응하며 돌아보는 습관을 가져야 겠다는 마인드를 주는 글이었습니다.

읽어주셔서 감사합니다.

<br>

## 출처
1. [https://redis.io/topics/faq](https://redis.io/topics/faq)

2. [https://stackoverflow.com/questions/10110660/misunderstanding-the-difference-between-single-threading-and-multi-threading-pro](https://stackoverflow.com/questions/10110660/misunderstanding-the-difference-between-single-threading-and-multi-threading-pro)

3. [https://groups.google.com/g/redis-db/c/DkSiBcvceuo/m/A5KMt_wDAQAJ](https://groups.google.com/g/redis-db/c/DkSiBcvceuo/m/A5KMt_wDAQAJ)

4. [https://redis.io/topics/config](https://redis.io/topics/config)

5. [https://redis.io/topics/lru-cache](https://redis.io/topics/lru-cache)
