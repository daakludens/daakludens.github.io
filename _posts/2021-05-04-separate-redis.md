---
title: "세션과 캐시 분리를 위한 Redis 분리와 Docker 사용"
excerpt: "Redis, Docker"
comments: true

categories:
  - Project
last_modified_at: 2021-05-06
---
## 세션과 캐시를 모두 담은 Redis
레디스는 디스크 IO보다 빠른 속도와 가용성을 보장하는 인메모리 db입니다. 이 장점들 때문에 프로젝트의 세션과 캐시 기능으로 Redis를 설정했습니다.             

지금으로썬 문제가 없지만 여기서 더 좋은 성능을 낼 수 있는 환경을 만들 수 있을지 고민해봤습니다.
결과적으로 `기능 분리의 필요성`과 `코어와 쓰레드 기능의 효율 극대화`를 위해 두 서버 인스턴스를 만들어 각각 세션과 캐시를 담기로 했습니다.

<br>

### 기능 분리
현 구조에서 발생할 수 있는 문제는 다음과 같습니다. 

1. Redis가 A라는 기능 때문에 다운될 경우 잘 수행 중이던 B 기능까지 중단된다.            
불필요한 복구 과정과 시간이 연장되며 서비스 운영에 차질
2. 기능 확장 시 다른 기능 때문에 제약이 걸린다.           
두 개의 상이한 기능을 동시에 확장하려면 복잡도 상승

이유는 독립적인 두 기능이 하나의 인스턴스에 있어서 의도치 않게 운명 공동체가 됐기 때문입니다.
독립적이라면 서로 영향을 받지 않도록 하는 게 기능 상 효율적입니다.

<br>

### Redis의 구조
위의 이론적인 이유를 봤다면 이번에는 구조적 이유를 살펴보겠습니다.                

Redis는 현재 나온 6버전까지 `싱글 쓰레드` 구조를 이어나가고 있습니다. 멀티 쓰레드를 사용하지 않는 이유는 여러 가지입니다.       

우선적으로 Redis는 메모리에 설치된 인메모리 데이터베이스로 메모리 IO가 발생합니다. 
Redis에서 병목 현상이 발생하는 구간은 메모리 또는 네트워크에서 거의 발생합니다.
그러니 CPU 성능을 올리기 위해 사용하는 멀티 코어 환경은 Redis 성능 개선에 크게 상관이 없습니다.

또한 멀티 쓰레드인 경우 서버를 확장할 때 확장이 어렵고 에러가 날 가능성이 큽니다. 
쓰레드 간의 이동으로 context switching이 발생하면서 지연 시간이 추가로 발생합니다.

결론적으로 멀티 쓰레드 구조로 얻는 이점이 거의 없으며 구조 복잡도만 늘어나기에 싱글 쓰레드 구조를 유지하는 겁니다.
그렇다면 싱글 쓰레드 구조가 멀티 코어 환경에서는 어떻게 동작하는지 알아야 합니다.

<br>

### 코어
싱글 쓰레드 구조에 싱글 코어를 사용하는 Redis는 하나의 command를 처리하고 있으면 뒤에 들어온 command는 대기해야 합니다.
각 command는 각기 다른 프로세스 시간이 걸리며 이는 Big O Notation 형태로 설명할 수 있습니다.

![image](https://user-images.githubusercontent.com/71559880/117562373-83c43d80-b0d9-11eb-9b22-9468e7631394.png)

Big O Notation은 n개의 요청(command) 수 만큼 얼마나 더 많은 시간이 걸리는지 측정하는 도구입니다.
위 그래프를 보면 O(n)이나 더 낮은 복잡도의 경우 요청이 늘어나도 선 기울기가 수평을 유지하는 반면 O(n logn)부터는 기울기가 올라가는 걸 볼 수 있습니다.
Redis 공식 문서에서도 command를 정할 때 최소 상수 시간(O(n))이 걸리는 명령어를 사용하면 CPU로 일어나는 지연 상태는 거의 일어나지 않는다고 합니다.

하지만 요새 대부분 멀티 코어를 탑재한 컴퓨터를 사용하는데 Redis로는 코어를 1개만 사용할 수 있습니다.
그럼 나머지 CPU 코어는 불필요하게 대기 상태에 놓이게 됩니다.

<br>

## Redis 분리 결정
기존에 가지고 있는 CPU를 더 효과적으로 사용하며 기능 분리를 위해 Redis 여러 서버를 동시에 띄워 서버 당 하나의 코어를 사용할 수 있습니다.
그러면 각 기능에만 집중하며 필요 시 확장 또는 관리가 가능한 환경이 형성됩니다. 
스프링을 배우면서 한 번 쯤 들어봤을 `OCP(Open-Closed Principle)` 개념이 이 상황에도 통용될 수 있을 것 같습니다.

기존에 설치 됐던 Redis는 session용으로 두고 새로 만드는 인스턴스에 cache 기능을 넣기로 했습니다.
Cache는 단기간 저장 후 계속해서 파기되는 과정을 거쳐서 굳이 많은 메모리 공간이 필요하진 않습니다.
하지만 `세션은 지속적으로 변경되는 값`이기 때문에 세션을 위한 메모리 여유 공간이 언제나 남아 있어야 합니다.
그래서 session 서버가 최대한 많은 메모리를 챙길 수 있도록 cache 서버에 메모리 최대 한도를 정했습니다.

<br>

## 첫번재 서버 - Session
복기 겸 제가 설치했던 방법을 다시 한 번 적어봤습니다.

Redis 공식 웹사이트에 들어가셔서 다운을 받으면 되고, 만약 Windows를 쓴다면 [Windows 전용 사이트](https://github.com/microsoftarchive/redis/releases/tag/win-3.0.504)에서 다운 받으면 됩니다. 

제가 글을 쓸 때 최신 LTS 버전은 3.0.504여서 해당 버전을 이용했습니다.
설치하실 때 더 최신 버전이 있는지 한 번 확인하시고 설치하세요.

redis-cli를 이용하고 싶다면 cmd를 통해 설치된 파일에 들어가 `redis-cli.exe`를 가동 시킵니다.
포트번호는 기본으로 6379이니 그대로 써줍니다.

```jsx
redis-cli.exe -p 6379
```

명령어 Enter를 쳤을 때 커서가 [로컬 IP 주소(localhost)]:[포트 번호]로 바뀌게 되면 정상 작동하는 겁니다.

<br>

## 두번째 서버 - Cache
이번 Redis는 Docker에 가동시키려 합니다. 
Docker는 사용해보니 한 두줄의 명령어로 서버를 올려 바로 사용할 수 있는 편리함과 개발 시간 단축이 됩니다.

우선 Docker Hub에 가입하고 Docker Desktop을 다운 받습니다. 
Docker는 기본적으로 이미지를 먼저 생성한 다음 이미지를 토대로 컨테이너를 생성하는 과정을 거칩니다. 먼저 이미지를 만듭니다. 

<br>

### 도커 이미지 선택 기준
참고로 여기서 `docker search [기술명]`을 입력해 도커가 가지고 있는 해당 기술의 모든 이미지 리스트를 확인할 수 있습니다.
저는 Docker에서 공식(official[ok]가 붙어 있는) 이미지를 사용했습니다. 
도커 공식 사이트에서도 공식 이미지를 강력 추천하는데 그 이유는 다음과 같습니다.
- Ubuntu 또는 CentOS 같은 기본 OS 저장소를 제공해 프로그램 시작점 제공
- 지속적인 보안 업데이트 보장
- 널리 쓰이는 프로그래밍 언어의 풀 패키지 제공
- Dockerfile 레퍼런스 제공

또한 공식 이미지의 업로드와 리뷰 만을 담당하는 Docker 사내 전담팀이 있어 보안과 품질이 보장됩니다.       

다만 무조건 공식 이미지를 사용해야 하는 건 아닙니다.
처음 시작하는 사용자라면 공식 이미지를 이용해 잘 쓰여진 공식 문서와 베스트 프랙티스(best practice)를 연습할 수 있습니다.
하지만 공식 이미지는 용량이 크기 때문에 경험이 어느 정도 쌓인다면 공식 이미지의 Dockerfile을 참고해 본인에게 맞는 커스텀 이미지를 제작하는 게 더 도움이 됩니다.

원하는 이미지를 간단하게 `docker pull [이미지 이름]`으로 다운 받을 수 있습니다. 

<br>

### 메모리 설정
하지만 Redis 서버는 기본적으로 하나의 인스턴스가 모든 메모리를 사용합니다. 
설령 인스턴스를 나눴어도 메모리는 공유하기 때문에 서로가 전체 메모리 용량을 소요하려 들기 때문에 메모리 설정이 필요해 보였습니다. 
또한 Memcached와 달리 여러 캐시 알고리즘을 제공하는 Redis이기에 캐시 알고리즘을 변경해보고 싶었습니다.

Redis의 서버 설정을 바꾸려면 redis.conf 파일에서 하면 됩니다. 
직접적으로 수정한 redis.conf를 Redis 이미지에 적용하기 위해 Dockerfile을 이용했습니다. 

<br>

### redis.conf 생성
컨테이너 외부에서 배포할 때는 Dockerfile의 COPY 명령어로 파일을 가져오거나 볼륨/마운트로 주입할 수 있습니다.
저는 미리 파일을 생성해 COPY로 이미지 생성 때 같이 올리는 방식을 택했습니다. 
Redis에서 제공하는 [redis.conf stable 버전](https://download.redis.io/redis-stable/redis.conf)을 다운 받아 필요한 부분한 수정하면 됩니다.         

중간 즈음 내려가면 `MEMORY MANAGEMENT`라는 이름으로 메모리 관련 설정 칸이 보입니다.
필요한 부분만 주석 해제를 하고 메모리 최대 사용량과 이빅션(eviction) 정책을 설정합니다.
우선 기본적으로 2mb로 설정하고 후에 성능 튜닝으로 적합한 값을 찾아나갈 예정입니다.

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

Redis를 적용한 프로젝트는 [여기](https://github.com/f-lab-edu/ludensdomain)서 볼 수 있습니다.

<br>

## 출처
1. [https://redis.io/topics/faq](https://redis.io/topics/faq)

2. [https://stackoverflow.com/questions/10110660/misunderstanding-the-difference-between-single-threading-and-multi-threading-pro](https://stackoverflow.com/questions/10110660/misunderstanding-the-difference-between-single-threading-and-multi-threading-pro)

3. [https://groups.google.com/g/redis-db/c/DkSiBcvceuo/m/A5KMt_wDAQAJ](https://groups.google.com/g/redis-db/c/DkSiBcvceuo/m/A5KMt_wDAQAJ)

4. [https://redis.io/topics/config](https://redis.io/topics/config)

5. [https://redis.io/topics/lru-cache](https://redis.io/topics/lru-cache)

6. [https://docs.docker.com/docker-hub/official_images/](https://docs.docker.com/docker-hub/official_images/)

7. [https://redis.io/commands/](https://redis.io/commands/)
