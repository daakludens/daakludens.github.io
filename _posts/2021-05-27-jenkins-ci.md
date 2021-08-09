---
title: "빌드와 단위 테스트 자동화를 위한 Jenkins CI 도입"
excerpt: "Jenkins, CI, Agile"
comments: true

categories:
  - Project
last_modified_at: 2021-06-22
---
## 효율적으로 일한다는 건
처음 개발을 시작했을 때는 혼자만의 기술 지식과 코딩 능력에만 집중했었습니다. 최대한 빠른 시간 내에 동작하는 함수를 만들어 내 업무 할당량을 마무리 짓는 걸 우선 사항으로 여기기도 했습니다. 그러다 지인 분의 추천으로 산드로 만쿠소가 쓴 소프트웨어 장인이라는 책을 읽으면서 개발은 단순 코딩이 아닌 전체적인 개발 주기 관리와 발전에 의거함을 보여줍니다.

개발의 전체 프로세스를 소프트웨어 개발 생명주기라고 하며 계획부터 개발, 테스트, 배포, 후 유지 보수까지 전체 과정을 의미합니다. 제가 처음 중요하게 여겼던 개발 과정은 전체 프로세스의 일부분에 지나지 않습니다. 이 전체 과정을 불필요한 시간 낭비 없이 효율적으로 움직일 수 있는 게 이상적인 업무 방법입니다.

개발자들도 이런 중요성을 인지하고 더 나은 방법을 연구해왔습니다. 여러 방식 중 오랫동안 사용된 폭포수 모델이 있습니다. 변경 횟수가 적은 계획엔 효과적이지만 최근 고객의 요구 사항이 잦고 산업이 빠르게 변화되는 만큼 즉각적인 대처가 가능한 모델의 필요성이 대두되고 있습니다. 각광 받는 모델은 만쿠소의 책에서도 소개됐던 에자일 모델이 있습니다.

<br>

## 폭포수 모델 vs 에자일 모델
폭포수 모델은 기획, 개발, 테스트, 배포 단계를 순서대로 거치는 단방향 프로세스를 의미합니다. 단계의 범위는 전체 애플리케이션을 기준으로 돌아갑니다. 각 단계마다 걸리는 시간이 길기 때문에 기획 단계에서 의도치 않은 변경이 생기지 않도록 구조를 짜야 합니다. 

에자일 모델은 범위를 전체 애플리케이션이 아닌 한 기능을 기준으로 진행합니다. 네이버에 나온 정의에 의하면 정해진 계획이 아닌 개발 주기 또는 소프트웨어 환경에 유연하게 대처합니다. 그러기에 실행 가능한 기능을 바로 고객에게 넘겨 받은 피드백을 기능에 바로 반영할 수 있다는 장점이 있습니다.

![Untitled (1)](https://user-images.githubusercontent.com/71559880/122907878-867ba900-d38e-11eb-8dc4-a7b63719c0a9.png)

<br>

에자일 개발 방식을 회사 문화로 정착 시키려면 잦은 커밋에도 에러 없이 동작하며, 기능을 구분해 확장성과 독립성을 부여할 수 있는 환경이 구성되야 합니다. 이를 위해 많은 회사에서 CI를 적용합니다.

<br>

## CI(Continuous Integration)
직역하면 지속적인 통합으로, 팀의 통합 프로젝트에 개인이 작업했던 코드 내용을 합치는 과정으로 위에서 언급한 에자일 모델에 적합한 방식입니다. CI를 위한 여러 툴이 존재하지만 저는 Jenkins를 사용하기로 했습니다.

<br>

## Jenkins
지속적인 통합을 위한 소프트웨어 도구입니다. Jenkins는 CI와 CD(Continuous Deployment)에서 자주 언급되는 툴입니다. Jenkins를 이용하면 CI 과정을 자동화 할 수 있기에 개발자는 개발에 더 신경을 쓸 수 있다는 장점이 있습니다. 

원래대로라면 매 커밋마다 프로젝트 빌드, 단위 테스트, jar 파일 생성 과정을 개발자가 직접 해줘야 합니다. 굉장히 지루한 작업이기도 하지만 중간에 개발자의 실수로 에러가 생길 가능성도 배제할 수 없습니다. 그렇기에 Jenkins로 에러 없는 방식으로 자동화를 설정하면 전체 개발 주기에 시간을 단축시키고 로직 작업에 더 집중할 수 있습니다.

<br>

## 적용
제가 하는 프로젝트는 Git Flow 방식을 차용합니다. [프로젝트 페이지](https://github.com/f-lab-edu/ludensdomain)에 Git Flow에 대한 설명이 간략하게 되어 있습니다. 브랜치를 기능 별로 분리해 원격 저장소에 커밋해 머지(merge) 승인을 받으면 메인 브랜치인 main에 머지하는 형식입니다. 여기서 CI를 적용해 로컬 환경에서 원격 저장소로 push를 했을 때, GitHub는 Jenkins로 신호(트리거)를 보내 자동으로 빌드, 테스트, 파일 생성까지 완성 시킵니다.

![test  Untitled 2](https://user-images.githubusercontent.com/71559880/128740421-cf2a6eaa-54b8-4ce8-a06b-65f6d4115e14.jpg)

<br>

그렇다면 제가 구현할 부분은 다음과 같습니다.
- Jenkins 설치
- 형상관리 툴(GitHub) 연결
- CI 자동화

<br>

### Jenkins 설치
저는 Jenkins를 새로운 서버에 설치해 운영하기로 했습니다. 네이버 클라우드 플랫폼에서 우분투 16.04 서버를 만들었습니다. 클라우드 서비스는 구글에서 운영하는 GCP도 있지만 NCP에서 추가 크레딧을 받을 수 있는 기회가 생겨서 네이버를 사용했습니다. 생각보다 서버 비용이 많이 들고 무료 크레딧이 소진되면 개인 돈으로 지불해야 되서 조금 부담되었기 때문입니다. 서버에 대해 더 자세한 얘기는 기회가 된다면 후에 더 얘기하도록 하고, 일단은 CI 과정에 집중하겠습니다. 

젠킨스를 사용하려면 자바를 설치해줘야 합니다. 제 경우 우분투 16.04에서 젠킨스를 사용할 때 자바 11 버전이 에러를 많이 일으켜서 8버전을 명시해 설치했습니다. 충돌 이유는 [젠킨스 도큐먼트](https://www.jenkins.io/doc/administration/requirements/java/)에서 설명한 대로 빌드 도구로 메이븐을 사용하면 메이븐에 지정된 자바 버전과 Jenkins에 설치된 자바 버전이 같아야 합니다.

```
sudo apt-get update // 설치 전 우분투 서버 업데이트
sudo apt-get install openjdk-8-jdk
java -version // 설치한 자바 버전 확인

update-java-alternatives -l // JAVA_HOME의 path를 알기 위한 명령어
java-1.8.0-openjdk-amd64 1081 /usr/lib/jvm/java-1.8.0-openjdk-amd64 // 위 명령어로 나오는 결과. /usr부터 경로를 복사

sudo nano /etc/environment // 환경 파일 edit하기 위한 명령어
JAVA_HOME="/usr/lib/jvm/java-1.8.0-openjdk-amd64" // 해당 라인 작성하고 저장
```

<br>

그 다음에 젠킨스를 설치합니다.

```
wget --no-check-certificate -q -O - https://pkg.jenkins.io/debian-stable/jenkins-ci.org.key | sudo apt-key add -

echo deb http://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list
```

<br>

설치가 끝나면 해당 서버의 ip 주소와 기본 포트인 8080을 url에 입력하면 젠킨스 메인 화면이 나옵니다. 그러면 아이디와 패스워드를 만들어 로그인하면 본인 만의 젠킨스를 갖게 됩니다.

만약에 포트 번호를 변경하고 싶다면 /etc/default 경로에 있는 jenkins 파일에서 설정을 바꿀 수 있습니다. 설정을 바꾸면 sudo service jenkins restart로 서버를 재부팅해줘야 제대로 작동합니다.

그리고 가끔 터지는 에러를 방지하기 위해 젠킨스 관리 → 시스템 설정에서 Jenkins location url명을 localhost에서 해당 서버의 ip주소로 바꿔줍니다.

<br>

### GitHub 연결
그 다음은 방금 개설한 젠킨스와 GitHub 내 프로젝트를 연결합니다.

**webhook**             
webhook은 연결된 레포지토리에 온 신호(트리거)를 인식해 프로젝트를 자동으로 빌드해주는 역할을 합니다. 여기서 트리거는 푸시나 풀 같은 행동을 의미합니다. 

본인의 깃허브 계정의 프로젝트에 Settings에 왼쪽 내비게이션을 보면 webhook 설정이 있습니다.

![Untitled](https://user-images.githubusercontent.com/71559880/128742211-665cd21e-4e07-485e-94ec-40621e795945.png)

<br>

payload url은 GitHub webhook에 연결할 URL을 의미합니다. 젠킨스 서버의 IP 주소와 포트 번호를 넣고 /github-webook/까지 넣어줘야 제대로 작동합니다.(맨 뒤에 슬래시 기호도 꼭 넣어줘야 합니다!) 

```
// 예시
http://127.0.0.1:8080/github-webhook/
```
<br>

밑에 webhook 동작 조건은 기호에 따라 선택하면 됩니다. 저는 모든 브랜치의 푸시 이벤트에 Jenkins와 연결이 되도록 설정했습니다.

<br>

**Personal Access Token**               
Jenkins를 사용할 때 가장 많은 시행착오를 겪은 부분이 credential 지정입니다. Jenkins는 클라우드 서비스 같은 다른 서드 파티 시스템과 연결되어 작업을 진행하기도 합니다. credential을 등록해두면 Jenkins는 그 대상을 신뢰 가능한 서드 파티로 인식해 막지 않고 정상적으로 작동될 수 있게 해줍니다. 

특히 Jenkins의 Pipeline 프로젝트는 Jenkinsfile로 명령할 때 바로 실행 가능하려면 credential이 필요합니다.  

credential은 아래와 같이 여러 방식으로 지정될 수 있습니다.

![Untitled (1)](https://user-images.githubusercontent.com/71559880/128742364-7fc5c258-4563-454f-a495-6bc8c09eea54.png)

<br>

GitHub에서 token을 생성해야 합니다. 본인 GitHub 계정의 Developer Setting에서 personal access tokens를 선택합니다. 

새로운 토큰을 생성하며 credential에 사용될 범위(scope)를 지정합니다. Jenkins에서 필요한 옵션은repo와 admin : repo_hook으로, 전부 체크해주면 됩니다.

![Untitled (2)](https://user-images.githubusercontent.com/71559880/128742543-ddfc3d5c-ff7e-4c54-9594-494af4e86021.png)

생성되면 만들어진 토큰명을 무조건 어딘가에 저장해두세요! Jenkins에서 credential 생성시 꼭 필요한 정보이고, 한 번 이 페이지를 나가면 다시 볼 수 없습니다.

<br>

### CI 자동화 기능             
이제 젠킨스에서 GitHub과 연결 설정을 합니다. 우선 webhook 연결을 위해서는 Github Integeration Plugin이 필요합니다. 플러그인 메뉴로 들어가서 설치해주면 됩니다. 그 다음 webhook으로 깃허브의 신호와 소스 코드가 오면 실행할 동작을 지정하기 위해 젠킨스에 파이프라인을 설치합니다. 제 프로젝트는 여러 브랜치를 사용하고 브랜치들의 push 이벤트에 동작할 거기 때문에 multi pipeline 프로젝트를 생성해줬습니다.

프로젝트 깃허브 설정에서 리포지토리 주소를 넣어주고 소스 코드 관리에서도 주소를 넣어줍니다. 바로 밑에 credential을 추가하는 공간이 있는데 username with password 옵션에 아무 아이디와 personal access token으로 생성한 패스워드를 넣어줍니다.

<br>

**Jenkinsfile 설정**                 
Jenkinsfile은 Jenkins 프로젝트가 수행할 명령어의 흐름을 적어 놓은 파일입니다. Agent를 통해 일련의 과정을 거치게 하며, 필요한 부분만 적어 보내기 때문에 굳이 복잡한 건 없습니다.

```
pipeline {
    agent any

    tools {
        maven 'maven'
    }

    stages {
        stage('Poll') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    sh 'mvn clean verify -DskipITs=true';
                }
            }
        }

        stage('Unit Test') {
            steps {
                script {
                    sh 'mvn surefire:test'
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }
    }
}
```

<br>

**확인**           
젠킨스의 로그에서 모든 단계가 정상적으로 처리되면 각 step마다 초록색으로 표시됨을 볼 수 있습니다.

GitHub 상에서도 push 이력 옆에 초록색 체크 마크가 붙어있으면, CI 성공 결과를 GitHub에 알려주는 겁니다.
![Untitled (3)](https://user-images.githubusercontent.com/71559880/128743217-510f3b8d-8a15-4d42-b71d-8cd9f48de972.png)

<br>

지금까지 글 읽어주셔서 감사합니다.
프로젝트는 이 [링크](https://github.com/f-lab-edu/ludensdomain)로 들어오시면 더 자세한 진행 상황을 볼 수 있습니다.

<br>

## 출처
Learing Continous Integration with Jenkins 2nd Edition
[https://www.notion.so/CI-30b9c81190634736b2d7c4afb2b97a89](https://www.notion.so/CI-30b9c81190634736b2d7c4afb2b97a89)
[**https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials**](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#handling-credentials)

