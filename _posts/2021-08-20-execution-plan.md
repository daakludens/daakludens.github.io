---
title: "DB 쿼리 성능 향상을 위한 실행 계획"
excerpt: "실행 계획, DB"
comments: true

categories:
  - Project
last_modified_at: 2021-08-20
---
# 서문
전반적인 사업 분야에서 폭발적인 데이터 생성량과 데이터 분석의 중요성이 높아지는 만큼 귀중한 데이터를 저장하고 활용할 데이터베이스의 중요성 또한 대두되고 있습니다. 
안정적이고 정확한 데이터를 입력하고 필요한 최소한의 데이터를 빠르게 호출하는 게 db 레이어의 업무 중 하나일 겁니다.

데이터를 다루기 위해선 SQL문에 대해 알아야 합니다. 
개발을 시작했다면 어느 시점엔 SQL문을 무조건 한 번은 다뤄보게 됩니다. 
요즘은 ORM 기술을 이용한 JPA가 많이 쓰이긴 합니다만, 기본적인 SQL 문법에 대해 아는 게 좋다고 합니다. 
왜냐면 JPA도 결국은 SQL문으로 실행되고 데이터베이스 실행 방식에 대해 더 잘 이해할 수 있기 때문입니다. 
JPA 사용 중 예상치 못한 에러가 났을 때 에러가 발생한 부분을 독해해 디버깅할 수 있어야 하기도 하고요. 

아무튼, 다시 SQL 얘기로 돌아오자면 XML 형식으로 쿼리문을 작성해 데이터를 다루는 건 다소 쉽습니다. 
다른 프로그래밍 언어와 달리 사람이 쓰는 자연 언어와 더 유사하기에 키워드만 알면 대충 어떤 흐름으로 조합하면 될 거라는 감이 옵니다. 
대신 쿼리 연산 능력은 키워드에 따라 엄청난 성능 차이를 냅니다. 
결국 더 효율적으로 연산하기 위한 방법으로 `실행 계획`을 조회한 쿼리 튜닝을 진행할 수 있습니다.

<br>

# 실행 계획
MySQL에서 실행계획은 DB의 뇌를 담당하는 옵티마이저가 전달된 쿼리문을 분석해 어떤 방식으로 데이터를 가져올지 예상하는 계획표입니다. 

왜 실행 계획을 쓰는가?
- 디스크 IO와 시간을 감소해 더 효율적인 쿼리문 작성
- 쿼리문 로직이나 기타 수정할 곳 보완

실행 계획으로 데이터 정렬 방식 또는 사용되는 인덱스 등을 파악해 데이터를 더 신속하게 가져올 수 있게 튜닝할 수 있습니다. 
실행 계획을 보려면 기존의 쿼리문 앞에 `EXPLAIN` 키워드를 추가해 실행시키면 됩니다.

<br>

# 문제 및 해결
제 프로젝트에서 쿼리문 대부분은 테이블의 데이터를 가공 없이 그대로 가져오게 되서 특별히 문제가 될 법한 쿼리는 없습니다. 
하지만 모든 쿼리문의 실행 계획을 돌렸을 때 튜닝이 필요한 구간이 2곳 있었습니다.

<br>

## 문제1
첫번째는 게임 리스트(getGameList 메서드)를 가져오는 쿼리문입니다. 
캐시를 적용해 반복적인 쿼리에는 성능 문제가 없지만, 처음 쿼리가 실행될 때 실행 계획에 눈에 밟히는 부분이 있습니다.

<br>

### 기존 쿼리문
```
SELECT G.id,
       G.title,
       G.genre,
       G.description,
       G.release_date releaseDate,
       G.price,
       G.developer,
       G.publisher,
       G.rating,
       G.sales,
       G.status,
       R.name genreName,
       D.name developerName,
       P.name publisherName,
       S.name statusName
FROM game G
       LEFT OUTER JOIN genre R ON G.genre = R.id
       LEFT OUTER JOIN developer D ON G.developer = D.id
       LEFT OUTER JOIN publisher P ON G.publisher = P.id
       LEFT OUTER JOIN status S ON G.status = S.id
WHERE G.id > 45
  AND G.status = 3 
   OR G.status = 4
ORDER BY G.release_date
LIMIT 50;
```

<br>

### 실행 계획 결과
![1](https://user-images.githubusercontent.com/71559880/130112497-db31d002-ebf6-48dd-ab4e-a838435680f3.png)

<br>

표를 보면 문제가 두 군데 있습니다. type이 `ALL`로 표시되고 Extra에 `using filesort`가 나옵니다.

간단하게 말하자면 ALL은 테이블 전체를 훑어서 조건에 맞는 데이터를 가져오고 있음을 의미하고 using filesort는 인덱스를 사용하지 않고 정렬하고 있음을 의미합니다. 

그렇다면 고려할 점은 다음과 같습니다.
- 쿼리에서 사용되는 컬럼은 인덱스를 사용하는지
- 만약 인덱스가 있다면 왜 인덱스를 사용 못하는지

인덱스는 FROM부터 사용하는 의미가 있기에 FROM부터 집중적으로 분석하면 됩니다.

<br>

![2](https://user-images.githubusercontent.com/71559880/130112588-7a696e2e-d12b-4411-af72-b81ef4454e24.png)

<br>

DB의 ERD를 보게 되면 game, genre, developer, publisher, status 각 테이블의 id 컬럼이 있으며 id 컬럼은 primary key가 걸려있어 자동적으로 index의 역할을 겸합니다. 
그렇다면 FROM절의 OUTER JOIN은 인덱스를 무조건 사용하고 있으니 WHERE부터 무언가 문제가 있음을 알 수 있습니다. 

우선 type을 튜닝해보겠습니다. 
ALL은 최후의 수단으로써 사용되는 설정이긴 하나, 실제로 풀 스캔 테이블이 필요한, 한꺼번에 많은 데이터를 읽어오는 식의 쿼리를 사용한다면 굳이 튜닝할 필요는 없습니다. 
하지만 잘못 썼다면 개선하는 게 맞는 거죠. 이 경우엔, 잘못 쓴 게 맞습니다.

해당 문제점은 AND와 OR 부분에서 발견할 수 있습니다. 
해당 쿼리는 판매 상태가 3(판매 중) 혹은 4(삭제 예정)인 게임 리스트를 가져오는 조건문입니다. 
WHERE에서 조건을 붙여 정해진 구간에 있는 데이터만 조회하도록 합니다. 
하지만 위에 쓰인 OR 조건문은 모든 데이터 중 게임 판매 현황이 4인 값을 찾습니다. 
그러니 당연히 모든 레코드를 조회하는 ALL이 실행 계획에 뜨는 겁니다.

<br>

### 해결1
그래서 쿼리문을 수정하게 되면 type이 테이블 전체를 스캔하는 ALL에서 조건문에 해당되는 값만 스캔하는 range로 변경된 걸 볼 수 있습니다.

**수정한 쿼리문1**
```
AND (G.status = 3 OR G.status = 4)
```

<br>

**결과**
![3](https://user-images.githubusercontent.com/71559880/130112834-ee04dbaa-b60a-4c71-8138-b319feec70aa.png)

<br>

### 해결2
두 번째로 filesort를 튜닝해보겠습니다. 

인덱스를 활용하지 못하는 정렬 기준이기에 ORDER BY절에 인덱스를 사용할 수 있도록 game 테이블의 id를 정렬 기준으로 변경했습니다.

<br>

**수정한 쿼리문2**
```
ORDER BY G.id
```

<br>

그러면 이제 정렬을 할 때 filesort를 사용하지 않아 Extra 컬럼에서 없어진 걸 볼 수 있습니다.

<br>

### 문제2
두번째 문제는 게임을 삭제(deleteGame 메서드)하는 쿼리에 있었습니다

**기존 쿼리문**
```
DELETE
FROM game
WHERE status = 4;
```
<br>

**실행 계획 결과**
![4](https://user-images.githubusercontent.com/71559880/130113193-d39426df-b8f7-425e-8789-ac2e8465c8e8.png)

<br>

**해결**
해당 쿼리문은 모든 게임 리스트를 훑어 삭제 예정인 상태를 확인해야 하니 type으로 ALL이 나옵니다. 
그래서 status에 인덱스를 추가로 줘서 전체를 스캔하지 않아도 되도록 합니다.

![5](https://user-images.githubusercontent.com/71559880/130113314-3acf4494-8829-42c0-a37c-6769c66b5ebb.png)

<br>

**결과**
인덱스를 적용한 후 결과로 type이 range로 변한 걸 볼 수 있습니다.

![6](https://user-images.githubusercontent.com/71559880/130113433-db792978-5cbf-44dc-a611-846cf23e0896.png)

<br>

# 회고
실제로 실행 계획을 해보니 그동안 제가 적어왔던 SQL문에 문제가 있음을 알았습니다.
기존에 회사에서 SQL문을 적을 때 무조건 쿼리문에서 모든 데이터를 가공해서 그대로 스크립트 단으로 넘기라는 주문이 있어 그대로 이행하고 있었습니다.
하지만 실행 계획에서 인덱스 사용 유무에 따른 쿼리 성능이 바뀌는 걸 보게 됐습니다.
데이터 사용량이 많아질 수록 이 성능 차이가 벌어질 테고, 데이터가 얼마 없더라도 좋은 SQL 문법을 지키는 게 굉장히 중요함을 배웠습니다.
그래서 우선 제 프로젝트에서라도 인덱스가 걸려 있는 컬럼이나 primary key 컬럼은 가공하지 않고 인덱스에 맞춰 가져오려고 노력 중입니다.

<br>

지금까지 제가 해온 프로젝트는 다음 [링크](https://github.com/f-lab-edu/ludensdomain)를 선택하면 볼 수 있습니다.          
긴 글 읽어 주셔서 감사합니다.

<br>

# 출처
Real MySQL
