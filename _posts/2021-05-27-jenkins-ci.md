---
title: "빌드와 단위 테스트 자동화를 위한 Jenkins CI 도입"
excerpt: "Jenkins, CI, Agile"
comments: true

categories:
  - Project
last_modified_at: 2021-05-28
---
## Agile 개발 환경
에자일 환경은 하나의 기능 또는 동일한 목적의 여러 기능을 한 단위로 보고 단위 씩 개발을 하는 개발 방식이다.
해당 방식은 기능을 수정할 때 다른 기능에 영향을 줄 위험 없이 신속하고 독립적으로 변경할 수 있는 장점이 있다.
하나의 기능이 개발되면 바로 적용이 가능하기에, 고객의 경우 각 기능을 테스트해보고 피드백을 바로 줄 수 있다.
즉, 고객은 사용 가능한 기능을 계속 경험할 수 있고, 개발자는 지속적으로 주어진 피드백을 빠르게 해결할 수 있다는 장점이 있다.

<br>

## CI(Continuous Integration)
CI는 로컬 환경에서 개발한 기능을 원격 저장소에 있는 통합 프로젝트에 추가하는 반복적인 과정이다.

ci -> integrate personal work into team's integration branch
detect problem early in process(build, unit test, etc) up until package

<br>

## Jenkins

hud pros
pipeline(flow of jobs) -> automatic sequence
useful plugins provided
repetitive build -> reduces time and enhances speed of whole operation
