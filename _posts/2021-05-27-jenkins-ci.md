---
title: "빌드와 단위 테스트 자동화를 위한 Jenkins CI 도입"
excerpt: "Jenkins, CI, Agile"
comments: true

categories:
  - Project
last_modified_at: 2021-05-28
---
## Agile 개발 환경과 CI
CI는 로컬 환경에서 개발한 기능을 원격 저장소에 있는 통합 프로젝트에 추가하는 반복적인 과정이다.

agile -> deals with one feature or group of related features one at a time
change in feature does not affect the entire project
customer holds a executable feature and provides feedback
short-term feedback based on one feature at a time & customer has running feature

ci -> integrate personal work into team's integration branch
detect problem early in process(build, unit test, etc) up until package

hud pros
pipeline(flow of jobs) -> automatic sequence
useful plugins provided
repetitive build -> reduces time and enhances speed of whole operation
