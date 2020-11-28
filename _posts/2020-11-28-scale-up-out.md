---
title: "대용량 트래픽을 위한 유통 시스템 설계"
excerpt: "어떻게 ESD 시스템에서 대용량 트래픽을 견디게 할까"
comments: true

tags:
  - scale up
  - scale out

categories:
  - Project
last_modified_at: 2020-11-19
---
### ESD 시스템

진행하게 될 프로젝트 주제는 게임 ESD 시스템 구현입니다.

게임 ESD 시스템이란 본 업체 또는 타 업체가 만든 게임들을 한 곳에 모아 놓고 이용자가 그 안에서 게임을 구매하고 다운로드해 플레이할 수 있는 플랫폼을 말합니다. 
최근에는 게임 구매 뿐만 아니라 게이머 커뮤니티나 유튜브 플레이 영상이 올라오는 등 다양한 서비스를 제공합니다. 

가장 유명한 ESD 시스템으로는 미국 밸브 사가 만든 Steam입니다. 
그 외도 Epic Games가 있고 자회사 게임 컨텐츠만 관리하는 Origin, BattleNet 등도 있습니다. 
그 중에서도 제가 가장 많이 사용하고 시도할 수 있는 컨텐츠가 다양한 스팀을 모델로 삼고 진행하기로 했습니다. 

혹시 관심 가시는 분들은 아래에 달아둔 깃허브 링크에서 보실 수 있습니다.

ludensdomain : [https://github.com/f-lab-edu/ludensdomain](https://github.com/f-lab-edu/ludensdomain)

<br>

### 문제: 트래픽 부하

프로젝트를 구상하며 가장 먼저 맞닥뜨린 문제는 트래픽에 대한 서버 관리였습니다. 
고객이 서비스 이용 시 바라는 점은 언제 어디서나 접속이 가능한 접근성입니다. 
그러면 제공자의 입장에서 10명이든 10만 명이든 각 사용자가 애플리케이션을 켤 때 최소한의 로딩 시간으로 화면이 출력되도록 하는 게 주목적입니다.

Steam은 평소에도 2천만 명의 온라인/오프라인 유저들이 접속하고 있고 1년에 6번 진행하는 세일 기간에는 세계 각국의 유저들이 동시다발적으로 접속해 게임을 쓸어 담습니다. 
대량의 사용자와 단기간에 폭발적으로 증가하는 접속률을 견디기 위해선 평소에 코드를 짜던 방식으로는 도저히 불가능해 보였습니다. 
이를 위한 해결책으로 서버를 확장하는 스케일링 방법인 `scale up`과 `scale out`을 찾았습니다.

<br>

### 방법: Scale Up과 Scale Out

scale up과 scale out은 애플리케이션에서 대용량 서버 부하와 변화하는 트래픽량에 유연하게 대처하기 위해 사용하는 확장성(scalability) 기술입니다. 
둘의 차이점과 장단점은 아래와 같습니다.

<br>

#### Scale Up(Vertical Scaling)     

<br>

![Untitled_Diagram](https://user-images.githubusercontent.com/58535669/99605037-316b6380-2a4a-11eb-941a-a3b59e22de23.png)

Scale up은 수직 확장 방식으로 사용하는 서버 1대를 업그레이드해 성능을 높이는 방법입니다. 
서버의 하드웨어인 CPU, RAM, SSD 같은 부품을 여유 슬롯에 추가하거나 고성능 부품으로 대체할 수 있습니다.

서버 1개만 다루기에 설계, 관리 및 확장이 쉽고 데이터 일관성 유지를 위한 추가적인 작업이 필요하지 않습니다. 
더 강력한 서버 기능이 필요하다 싶으면 부품을 사서 갈아끼우면 되고 네트워크 비용이 적다는 장점이 있습니다. 

대신 성능을 높이기 위해 부품을 교체하며 드는 비용이 큽니다. 
또한 가동하는 하나의 서버에 모든 트래픽이 몰리기 때문에 만일 서버가 다운된다면 막대한 데이터 손실이 일어날 수 있습니다. 
무엇보다 전체 시스템의 성능 최대치는 서버 1대의 최대 스펙과 같아 무한정으로 성능을 올릴 수 없다는 단점이 있습니다.

Scale up 방식은 오라클, MySQL, Postgres 같은 RDBMS를 이용하는 데이터베이스 서버에 적합한데 하나의 서버가 있다는 가정 하에 디자인 되어 있기 때문입니다.

아래에 IBM Red Book에서 정리한 scale up에 대한 장점입니다.

<br>

![Untitled](https://user-images.githubusercontent.com/58535669/99605005-20225700-2a4a-11eb-968d-cad185b4a885.png)

<br>

#### Scale Out(Horizontal Scaling)     

<br>

![Untitled_Diagram_(1)](https://user-images.githubusercontent.com/58535669/99605073-4cd66e80-2a4a-11eb-8ad9-6389cdabe871.png)

Scale out은 수평 확장 방식으로 서버의 개수를 늘려서 성능을 높이는 방법입니다. 
부품을 교체하지 않고 보통 성능의 서버를 여러 대 들여서 병렬 형식으로 서버를 관리합니다.

서버 1대가 다운되더라도 가동 중인 다른 서버가 넘겨 받아 일을 진행할 수 있어 데이터 손실을 최소화 할 수 있습니다. 
트래픽 부하를 나눠 갖기 때문에 서버 다운 가능성이 낮아지고 하드웨어적으로 큰 비용이 필요하지 않습니다.

대신 서버 분배와 관리를 위한 관련 기술이 추가로 필요해집니다. 대표적으로 서버 부하를 분산시켜주는 로드 밸런싱을 사용합니다. 
OSI 7계층의 각 층에 맞는 로드 밸런서가 있으며 포트 정보를 이용하는 L4과 L7을 많이 사용합니다. 
그 중 다수의 서버 프로그램을 다루는 경우 4번(L4) 이상의 로드밸런서를 사용해야 합니다. 

이외에도 DB의 쓰기 전용 서버인 마스터 서버를 분리하는 샤딩, 고속 버퍼 메모리 방식인 메모리 캐시, 비관계형 데이터베이스 NoSQL 등을 사용할 수 있습니다.

또한 scale out은 필요 이상으로 개수를 늘릴 경우 가지고 있는 리소스를 충분히 활용할 수 없는 server sprawl 현상이 일어날 수 있어 각 프로젝트에 맞는 최적의 서버 개수도 고려해야 합니다.

IBM Red Book에서 정리한 scale out에 대한 장점입니다.

<br>

![Untitled](https://user-images.githubusercontent.com/58535669/99605189-8b6c2900-2a4a-11eb-8b15-49db74e18e3e.png)

<br>

### 나의 선택: scale out

두 방법 중 저는 scale out을 이용하기로 했습니다.

이유로는,

1. 게임 서비스에선 속도도 중요하지만 유저의 데이터 손실 최소화와 서버 안전성이 더 중요하다고 판단했습니다. 
만에 하나 서버가 다운되면 애플리케이션 접근 자체가 아예 불가합니다. 
막대한 데이터 손실과 더불어 서비스에 불만을 느낀 유저가 탈퇴하는 최악의 수가 생길 수 있습니다. 
2. 주기적으로 새로운 CPU나 RAM을 교체하기 보다 동일한 사양의 서버를 더 들이면 기능 대비 훨씬 경제적입니다. 
하드웨어는 수직적 상승이 일어날 때마다 가격이 급등하기 때문입니다. 
제가 구현할 프로젝트는 접속이 급증하는 특정 시기 외에는 트래픽의 변동폭이 좁을 예정이기에 찰나를 위해 비싼 하드웨어로 교체하기엔 부담이 생깁니다.
3. 오픈 소스 기술을 이용한 수평적 서버 확장이 1인 프로젝트로써 비용적 측면에 메리트가 있습니다.

<br>

지금까지 프로젝트 [ludensdomain](https://github.com/f-lab-edu/ludensdomain)에서 대용량 트래픽을 견디기 위한 조사 과정에 대해서 설명했습니다.

여기까지 읽어주신 분들 감사합니다.

<br>

### 출처
- IBM red books. Scale up for Linux on LinuxONE 2019     
<https://books.google.co.kr/books?id=SxqjDwAAQBAJ&pg=PA6&dq=scale+up+scale+out&hl=ko&sa=X&ved=2ahUKEwiIpcKO8d3sAhXLyYsBHbMECfYQ6AEwAXoECAkQAg#v=onepage&q&f=true>
- 로드 밸런싱     
<https://dheldh77.tistory.com/entry/네트워크-로드-밸런싱Load-Balancing>
- scale up과 scale out     
<https://dheldh77.tistory.com/entry/네트워크-스케일-업Scale-Up과-스케일-아웃Scale-Out>
- 서버 관련 기초 용어, 개념 정리     
<https://wkdtjsgur100.github.io/internet-terms-2/>
