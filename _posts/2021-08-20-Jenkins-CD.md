---
title: "안정적인 배포 자동화를 위한 Jenkins CD 도입"
excerpt: "Jenkins, CD, Agile"
comments: true

categories:
  - Project
last_modified_at: 2021-08-20
---
# CD(지속적인 배포, Continuous Deployment)
## 왜 배포를 자동화하는가?
저번에 CI를 통해 빌드와 유닛 테스트 자동화를 완성했습니다. 이 과정으로 통합된 코드는 에러 없이 동작 가능한 게 증명되었습니다. 
그럼 이제 증명된 애플리케이션을 일반 사용자가 쓸 수 있도록 서버에 배포해야 합니다.

저는 CI를 위해 이미 작성한 Jenkinsfile에 덧붙여 자동 배포화까지 완성 시키기로 했습니다. 
물론 배포는 개발자가 직접 실행할 수 있습니다. 하지만 모종의 이유로 배포에 실패할 가능성이 있습니다. 
빌드나 유닛 테스트의 경우 에러를 확인하면 코드 수정만 하고 다시 푸시하면 되지만 배포는 일반 사용자가 실제로 사용하는 애플리케이션에 영향이 가기 때문에 더 조심해야 한다고 생각됩니다. 
또한 통합 과정과 마찬가지로 매 배포마다 개발자가 일일이 배포하면서 소비되는 시간과 노력을 절감하여 그 절감된 시간을 비즈니스 로직에 투자할 수 있도록 도와줍니다.
이런 면에서 Jenkins로 배포 자동화 기능을 추가해줬습니다.

<br>

## 흐름
Jenkinsfile은 저번에 작성한 [CI](https://daakludens.github.io/project/jenkins-ci/)에 이어 배포 관련 step만 추가해주면 됩니다. 
배포 과정이 제대로 동작한다면 빌드로 만든 JAR 파일을 배포 서버로 이동 시켜서 실행시킵니다. 
실행은 배포 서버 내 쉘 스크립트로 명령어를 주었습니다.

![test  Copy of Untitled 2](https://user-images.githubusercontent.com/71559880/130258871-01135997-ce32-419a-abc7-00c4aed8aec2.jpg)

<br>

## Jenkinsfile
운영 서버로 JAR 파일을 보내기 전에 아래 두 조건에 부합하는지 확인합니다.
- 빌드의 성공 여부
- push된 브랜치명 확인

여기서 브랜치명 확인이란, Git flow 방식에 의해 배포에 쓰일 브랜치를 다른 용도의 브랜치와 분리해 에러 없는 깔끔한 코드만 배포 브랜치에 담는 방식을 의미합니다.
우선 통합용 브랜치에 제대로 담기는 지 확인하기 위해서 develop 브랜치일 때 동작하도록 시범해봤습니다.

```
when {
      allOf {
        expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
        branch 'develop'
      }
  }
```

<br>

그 다음엔 JAR 파일을 넘길 서버의 정보를 입력해주고 실행시킬 쉘 스크립트 호출 코드를 더해줍니다.

```
steps([$class: 'BapSshPromotionPublisherPlugin']) {
                sshPublisher(
                    continueOnError: false, failOnError: true,
                    publishers: [
                        sshPublisherDesc(
                            configName: "ludens-deploy",
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "target/*.jar",
                                    removePrefix: "target",
                                    remoteDirectory: "/",
                                    execCommand: "sh /scripts/ludens-deploy.sh"
                                )
                            ]
                        )
                    ]
                )
            }
```

<br>

## 쉘 스크립트
리눅스 서버에 쉘 스크립트를 실행시키면 작성한 bash 명령어가 동작하게 됩니다.
반복적으로 고정된 작업을 호출해야 한다면 쉘스크립트가 적합 합니다.

아래에 나와 있는 명령어의 흐름은 다음과 같습니다.
1. 쉘스크립트에 쓸 변수(JAR 파일 루트와 JAR 파일명) 선언
2. JAR 파일이 있는 루트로 이동
3. 동일한 JAR 파일명으로 기동 중인 프로세스의 PID 가져오기
4. PID 확인 후 있으면 종료시키고 없다면 그대로 진행 
5. java -jar 명령어로 JAR 파일 배포

```
#!/bin/bash

REPOSITORY=/root
JAR_NAME=ludensdomain-0.0.1-SNAPSHOT.jar

echo "Move To Repository"
echo $REPOSITORY
cd $REPOSITORY

echo "Find Current PID"
CURRENT_PID=$(ps -ef|grep "$JAR_NAME"|awk '{print $2}')

echo "$CURRENT_PID"

if [ -z "$CURRENT_PID" ]
then
  echo "Start New Process"
else
  echo "Kill Current Process"
  kill -15 $CURRENT_PID
  sleep 3
fi

echo "Deploy Application"
java -Dspring.profiles.active=prod -jar $JAR_NAME &
```

<br>

마지막에 java -jar 명령어 끝에 &를 붙여 애플리케이션이 백그라운드에서 돌도록 해줍니다.
안 그러면 포그라운드에서 실행해도 쉘스크립트가 끝나면서 애플리케이션이 같이 종료됩니다.

<br>

## 회고
지금까지 프로젝트를 진행하면서 가장 많은 시도를 한 단계가 CI와 CD였습니다.
CI와 CD 테스팅 과정에는 푸시가 필연적으로 들어가게 되고, 그만큼의 커밋 내역이 쌓이게 됐습니다.

![Screenshot_1](https://user-images.githubusercontent.com/71559880/130262192-4fe3ed72-55c5-42aa-a911-91cc823af31c.png)

<br>

한 브랜치의 커밋 내역만 해도 이렇게 길고 지저분합니다. 그런데 이 브랜치를 다른 브랜치에 merge하게 되면 이 커밋 내역까지 합쳐집니다.
커밋 내역은 형상 관리에도 쓰이지만 브랜치 개발 과정을 한 눈에 볼 수 있고 유의미한 정보만 담기는 것이 중요합니다.

그래서 아예 테스팅용 브랜치로 따로 정의하는 게 좋을 것 같다는 생각이 들었습니다.
다음에 이러한 테스팅 과정이 길법한 기능 추가가 필요한 경우, 똑같이 브랜치를 생성해 진행하되 테스팅용 브랜치는 삭제하고 새 feature 브랜치에 깔끔한 코드를 커밋해 머지하려 합니다.

<br>

# 마치며
전체 코드는 아래 링크에서 확인할 수 있습니다.           
[프로젝트 링크](https://github.com/f-lab-edu/ludensdomain)

읽어주셔서 감사합니다.
