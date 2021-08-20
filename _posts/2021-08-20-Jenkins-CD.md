---
title: "빌드와 단위 테스트 자동화를 위한 Jenkins CI 도입"
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
또한 통합 과정과 마찬가지로 매 배포마다 개발자가 일일이 배포하면서 소비되는 시간과 노력을 비즈니스 로직에 투자할 수 있도록 도와줍니다. 
이런 면에서 Jenkins로 배포 자동화 기능을 추가해줬습니다.

<br>

## 흐름
Jenkinsfile은 저번 프로젝트(url)에 이어 배포 관련 step만 추가해주면 됩니다. 
배포 과정이 제대로 동작한다면 빌드로 만든 JAR 파일을 배포 서버로 이동 시켜서 실행시킵니다. 
실행은 배포 서버 내 쉘 스크립트로 명령어를 주었습니다.

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

## 쉘 스크립트 작성법
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

## 회고
너무 많은 TRY로 인해 더러워진 커밋 내역이 문제였다.
이걸 해결할 방법을 찾아야 겠다.

<br>

전체 코드는 아래 링크에서 확인할 수 있습니다.           
[프로젝트 링크](https://github.com/f-lab-edu/ludensdomain)

읽어주셔서 감사합니다.
