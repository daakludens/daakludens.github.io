---
title: "MySQL 멀티 서버 환경 구축"
excerpt: "MySQL, replication"
comments: true

categories:
  - Project
last_modified_at: 2021-04-21
---
## 데이터 복구 문제
캐시를 적용해 db 부하를 줄였지만 여전히 문제점은 존재합니다. 
모든 데이터를 가지고 있는 하나의 저장소가 모종의 이유로 다운된다면 그동안 축적했던 데이터가 없어질 위험이 있고, 어찌 복구한다 하더라도 
그 기간 동안 서비스가 중단되며 금전적 손해와 서비스에 대한 신뢰감이 떨어질 수 있습니다. 
이런 일을 방지하기 위해서 MySQL의 replication(복제) 기능을 사용하기로 했습니다. 

<br>

## Replication
MySQL에서 replication은 두 개의 데이터베이스를 두고 데이터를 공유해 `부하 분산`과 `db 백업`을 위해 사용합니다. 
두 데이터베이스는 마스터(master)와 슬레이브(slave)로 마스터는 서비스의 쓰기(INSERT, UPDATE, DELETE) 기능을 담당하고 슬레이브는 읽기(SELECT) 기능을 담당합니다. 
평소 데이터를 읽어올 때는 슬레이브를 이용하고 데이터 변경 시에는 마스터에서 진행하며, 마스터는 변경 내역을 로그로 기록한 다음 해당 로그를 슬레이브 db로 보내서 데이터를 업데이트합니다.

![image](https://user-images.githubusercontent.com/71559880/115433101-1f346200-a242-11eb-9dee-95db0451704c.png)

<br>

MySQL 서버를 2개 띄우려면 여러 방법이 있습니다. 한 컴퓨터 내에서 `포트 번호가 다른 MySQL 서버를 추가`하거나 Docker나 클라우드 플랫폼의 `가상 서버에 추가 서버를 구축`하는 것입니다. 
이왕 새 기술 배울 때 흐름을 제대로 파악해보자는 마음에 먼저 로컬 환경에서 MySQL 서버를 추가한 다음, 추후 배포 환경을 고려해 클라우드 플랫폼의 가상 서버를 쓰는 두 가지 방법을 모두 해봤습니다. 

<br>

## 로컬 환경에서 replication 설정
로컬 환경 내 replication을 다음과 같은 단계로 진행했습니다.
- MySQL Slave 서버 파일 생성 및 가동
- Master와 Slave 서버 설정 변경 및 user 추가
- Master 서버의 log 덤프 뜨기
- Slave 서버에 적재

<br>

여기서 더 자세한 사항을 말씀 드리기 전 유의할 점이 있습니다.
- 제가 사용하는 MySQL 버전은 8.0.23입니다. 각자가 사용하는 MySQL의 버전 확인은 필수입니다. (사용하시는 MySQL 서버에 "select version( );"이라고 치면 나옵니다.)
- 로컬 환경 구축을 하며 cmd와 MySQL Command Line을 섞어 썼으니 주의해주세요. 대체로 shell 명령어가 필요하면 cmd를, MySQL 명령어가 필요하면 Command Line을 사용했습니다.

<br>

우선 기존의 MySQL 서버 파일을 찾아 복제해줍니다. 복제한 파일명은 원하시는 대로 지으시면 됩니다. 
- C:/ProgramData/MySQL/MySQL Server 8.0 파일을 다른 이름으로 복사
- C:/ProgramFiles/MySQL/MySQL Server 8.0 파일을 다른 이름으로 복사

![image](https://user-images.githubusercontent.com/71559880/115435877-46406300-a245-11eb-8390-2ae6fa2b7fda.png)
![image](https://user-images.githubusercontent.com/71559880/115435924-52c4bb80-a245-11eb-8a09-e9b85e893773.png)

현재 두 서버는 같은 설정을 갖고 있기에 이대로 진행을 하면 당연히 충돌이 납니다. 
그래서 MySQL 설정을 변경해야 하는데 제가 사용하는 윈도우의 경우 `my.ini` 파일을 사용합니다. 리눅스나 UNIX 환경에서는 `my.cnf` 파일에서 설정을 변경해야 합니다. 
my.ini의 경우 보안에 민감한 파일이다 보니 함부로 수정이 불가능합니다. 
그래서 현재 os 유저에게 쓰기 권한을 부여해야 합니다. 
먼저 windows + R 버튼으로 services.msc에 들어가 실행 중인 MySQL80을 끈 다음 우클릭을 해 해당 파일의 속성으로 들어갑니다.
보안 → 유저 선택 → 편집 → 유저 다시 선택 → 아래 창의 모든 권한 체크박스에 체크 → 적용 → 확인을 누릅니다.
이제 메모장 프로그램으로 해당 파일을 수정할 수 있게 됩니다. 그림과 같이 설명된 깔끔한 글이 있으니 [여기](https://woowaa.net/131)를 보시며 참고하시면 됩니다.

쓰기 권한을 킨 다음 서버가 사용할 포트 번호, path 설정에 필요한 base directory와 data directory를 각 my.ini 파일에 적어주면 됩니다.

master 서버 my.ini 설정
```
# CLIENT SECTION
...
port=3307

# SERVER SECTION
...
# The TCP/IP Port the MySQL Server will listen on
port=3307
```

<br>

slave 서버 my.ini 설정
```
# CLIENT SECTION
...
port=3308

# SERVER SECTION
...
# The TCP/IP Port the MySQL Server will listen on
port=3308

# Path to installation directory. All paths are usually resolved relative to this.
basedir="C:/Program Files/MySQL/MySQL Server 8.0_Sub/"

# Path to the database root
datadir="C:/ProgramData/MySQL/MySQL Server 8.0_Sub/Data"
```

<br>

주의할 점
- basedir은 주석 처리 되어있으므로 #을 빼줘야 합니다.
- 제가 사용한 MySQL 8.0.23은 Secure File에 경로 지정이 있어 바꿔야 했습니다. 다른 버전에서는 어느 부분에 추가적인 경로 지정이 필요한지는 모릅니다. 본인이 처음부터 끝까지 찬찬히 훑어보면서 체크해보시기 바랍니다.
- 포트 번호는 총 2군데를 바꿔야 합니다.

설정이 끝나면 슬레이브 서버 서비스를 생성하고 실행시켜야 합니다. 
cmd에서 슬레이브 서버의 Program Files 안에 있는 폴더의 bin에 들어가 시스템 설치 명령을 내립니다.

```
cd C:/Program Files/MySQL/MySQL Server 8.0_Sub/bin
"C:/Program Files/MySQL/MySQL Server 8.0_Sub/bin/mysqld.exe" --install MySQL80_Sub --defaults-file="C:/ProgramData/MySQL/MySQL Server 8.0_Sub/my.ini"
```

<br>

만약 여기서 install/remove of the Service Denied 가 나오신 분이 있다면, 그건 해당 권한이 없기 때문에 발생한 일입니다. cmd를 관리자 권한으로 실행해주면 됩니다.
이제 Windows + R에 services.msc에 들어가면 방금 만든 MySQL80_Sub이라는 서비스를 볼 수 있습니다.

![image](https://user-images.githubusercontent.com/71559880/115433819-ee086180-a242-11eb-878b-aa8378ef253c.png)

더블 클릭으로 실행시키면 이제 MySQL Server 8.0_Sub를 이용할 수 있습니다.

접속 가능한지 확인해봅니다. 
MySQL의 계정 로그인 시 두 서버의 포트 번호가 다르기에 포트 번호를 명시해줘야 합니다. 
만약 포트 번호를 쓰지 않으면 자동으로 기본 포트로 설치된 MySQL에 접속하게 됩니다.  
여기서 처음 로그인할 때 계정은 root를 사용하고 비밀번호는 기존 MySQL 서버의 root 계정의 비밀번호를 그대로 쓰면 됩니다.

```
cd C:/Program Files/MySQL/MySQL Server 8.0_Sub/bin
mysql -u root -p --port=3308

Enter password: **********
```

<br>

서버 구동 확인이 됐다면 replication 설정을 위해 my.ini를 다시 들어갑니다. 

마스터 서버
```
***** Group Replication Related *****
...
log-bin="C:/log/binary_log"
server-id=101

# Additional Options for Replication
sync_binlog=1
binlog_cache_size=5M
max_binlog_size=512M
expire_logs_days=14
log-bin-trust-function-creators=1
```

<br>

슬레이브 서버
```
# ***** Group Replication Related *****
...
log-bin="C:/log/binary_log"
server-id=102
relay-log=relay_log
relay_log_purge=TRUE
read_only
```

<br>

이제 복제 계정을 만들어야 합니다.
복제용 계정은 슬레이브가 마스터 서버에 접속하고 로그인하기 위해 필요한 계정입니다. 
MySQL Command Line에 들어가 유저 확인을 해봤습니다.

```
select User, Host from mysql.user;
SHOW GRANTS;
SHOW GRANTS FOR '[유저명]';
```

<br>

밑의 명령어는 서버의 정보를 확인할 수 있는 명령어 입니다. 하지만 아직 설정을 하지 않아 에러가 납니다.

replication client이라는 명령어로 privilege grant를 해줘야 합니다. 
저는 모든 권한을 가지고 있는 root 계정이 아닌 다른 계정을 사용하고 있어 권한이 없는 겁니다.

권한 설정을 위해 root 계정으로 들어가서 명령어를 쳐줍니다. 
여기서 replication은 권역(global) 권한에 포함되기 때문에 특정 데이터베이스가 아닌 서버 전체에 영향을 줍니다. 
만약 특정 데이터베이스만 주고 싶다고 명령어를 치면 에러가 나옵니다.

```
show master status;
grant replication client on *.* to '[유저명]'@'%';
```

<br>

이제 show master status 확인이 가능합니다.

![image](https://user-images.githubusercontent.com/71559880/115434442-a7673700-a243-11eb-843e-9062b9181748.png)

복제용 계정 생성
```
CREATE USER '[slave 유저명]'@'%' IDENTIFIED BY '[slave 패스워드]';
GRANT REPLICATION SLAVE ON *.* TO '[slave 유저명]'@'%';
```

이제 master DB의 바이러니 로그 파일을 .sql 파일에 dump해서 slave DB로 복제하면 됩니다. 
만약 .sql 파일을 특정 위치에 보관하고 싶다면 절대 경로를 사용해 명시해야 합니다. 
그리고 서버가 2개 있기에 어떤 포트를 사용해야 할지 모르기 때문에 포트 번호를 명시해야 됩니다.


```
mysqldump -uludens -p --port=3307 --opt --single-transaction --hex-blob --master-data=2 --routines --triggers --all-databases > c:/log/master_data.sql
```

<br>

그리고 해당 명령어를 쳤는데 비밀번호를 안 물어보고 바로 처리된다? 
오류입니다. 파일 생성됐다고 그냥 넘어가지 말고 열어서 내용물을 봐야 합니다. 전혀 엉뚱한 값이 들어가기도 합니다.


이제 MySQL command line에서 적재 명령어를 내립니다.
```
SOURCE c:/log/master_data.sql
```

<br>

master_data.sql 파일에 들어가면 헤더 부분에 CHANGE MASTER로 시작하는 문장을 볼 수 있습니다
이 명령어를 그대로 복사한 다음 마스터 MySQL의 호스트명, 포트, 복제용 사용자 계정, 비밀번호를 추가해 명령어를 만듭니다.

```
CHANGE MASTER TO MASTER_LOG_FILE='binary_log.000004', MASTER_LOG_POS=156, master_host='[master 계정 이름]', master_port=3307, master_user='[복제용 계정 이름]', master_password='[복제용 계정 비밀번호]';
```

<br>

슬레이브 MySQL에 로그인해 만든 명령어를 실행하고 SHOW SLAVE STATUS를 실행하면 복제 관련 정보가 뜨는 걸 볼 수 있습니다. 
그러면 등록된 정보를 실행하기 위해 START SLAVE 명령어를 칩니다.

## 결과
![image](https://user-images.githubusercontent.com/71559880/115434986-54da4a80-a244-11eb-858d-427035941790.png)

<br>

## 여담
### 1
MySQL을 다운 받을 때 zip과 msi 두 가지 방법이 있습니다.
MariaDB(MySQL 개발자들이 나와 구조가 거의 비슷한) 공식 웹사이트에 해당 방법의 차이점에 대한 설명이 있습니다.

zip은 MySQL에 필요한 모든 파일을 담고 있고, Perl을 다루는 데 필요한 test suite과 nix 쉘 스크립트 등이 있습니다. 
그러나 윈도우 유저의 경우에는 나열한 기능을 사용할 환경이 제공되지 않기에 `쓸 일이 없다`고 합니다. 
msi는 윈도우 환경에 필요한 파일을 갖추고 있으며 Unix 배경에서는 쓸 수 없다고 합니다. 즉, msi는 윈도우 유저용, zip은 리눅스 유저용이라고 이해하면 될 것 같습니다.             
링크 : [https://mariadb.com/kb/en/zip-or-msi/](https://mariadb.com/kb/en/zip-or-msi/)                 

### 2
제가 마주했던 문제 중 하나가 윈도우 MySQL을 다루기 위한 MySQL Command Line의 문제점이었습니다.
첫번째는, Command Line을 실행해 root의 패스워드를 정확히 넣었는데 그냥 프로그램이 종료되는 에러가 있었습니다.
그럴 때는 Windows + R에 services.msc에 들어가 MySQL을 찾습니다. 
제 경우에는 MySQL 80이 있었습니다. 더블 클릭해서 들어가면 시작 유형이 있는데 이를 자동으로 바꿔줘야 합니다.

![image](https://user-images.githubusercontent.com/71559880/115436097-8f90b280-a245-11eb-87d7-3192b2460fc1.png)

애초에 프로그램이 종료되는 이유는 MySQL 서버가 동작을 안 하고 있기 때문입니다. 
그래서 시작 유형을 자동으로 켜주면 컴퓨터가 시작될 때 자동으로 MySQL도 같이 시작을 합니다.
두번째로, Command Line에서 root 유저에서 다른 유저로 전혀 사용하지 못하는 겁니다. 
root로 먼저 들어간 다음 껐다가 다시 시작하면 무조거 root에서 시작해 패스워드를 입력하는 것부터 시작합니다. 
인터넷 상에서 user를 바꾸려면 system mysql -u 유저명 -p를 사용하면 된다고 하는데, 저는 비밀번호 입력까지는 가지만 그 다음 localhost를 인식할 수 없다는 에러만 계속 뿜더군요. 
여러 방법을 찾던 중 Command Line Client의 속성으로 들어가서 대상(Target)에 보이는 저 "=uroot" 부분을 "-u[유저명]"으로 변경하면 된다고 합니다.

![image](https://user-images.githubusercontent.com/71559880/115436152-a2a38280-a245-11eb-98e6-043f55cccb66.png)

<br>

## 츨처
Real MySQL

[https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_replication-client](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_replication-client)

[https://mudchobo.github.io/posts/spring-boot-jpa-master-slave](https://mudchobo.github.io/posts/spring-boot-jpa-master-slave)

[https://jsonobject.tistory.com/383](https://jsonobject.tistory.com/383)
