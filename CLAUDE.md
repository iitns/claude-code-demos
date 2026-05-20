# Overview
- Claude Code를 이용해 Zero to One을 한 경험을 공유하는 발표 자료를 준비하는 프로젝트

# Goal
- 15-30분 정도짜리 영어로 된 슬라이드
  - 15분씩 두번에 나눠서 해도 됨.

# Audience
- Junior - Senior Software engineer including one eng manager
- Claude code 사용은 대부분 회사 계정을 통해 회사 업무만 했을 것으로 예상
- 기본적으로 매일 claude code를 사용하고 있음

# 주의사항
- 모든 슬라이드는 영어로 되어 있어야 함
- 영어로 스크립트도 준비 해야함
- Sharing learnings, knowledges
- Do not get distracted by the engineering decisions such as why used this and that, but focus on the progress on building things

# Format
- HTML
  - Some image files or video will be attached later by me. Keep an placeholder for the media
- Keep the script aside
- Keep the text low as possible. But hold more pictures and diagrams
- 어떤 프레임웤을 써도 괜찮아. 가장 잘 표현할 수 있는 어떤 것이든 써도 좋아. 회사 내부 Github에 올려서 서빙할 예정이라 뭐든 상관 없어

# 내용
## Overview
- 무엇을 할지 보여줌
- 어떤 결과물이 필요했는지, 그러기 위해서 뭐가 있어야 했는지, 가지고 있는 인프라가 무엇이 있었는지, 거기서 클로드 코드를 어떻게 활용 했는지
- 도움이 되었던 경험: 크게 작업을 계획하고, 서버별로 작업을 나눴으며, agent 별로 태스크를 나누어서 겹치지 않게 했다.

## Motivation
- 밥 먹거나 잠깐 머리 식힐 때 유튜브 쇼츠도 보지만, 요새 어떤 일들이 벌어지고 있는지 궁금해서 그런 글들을 찾아보곤 한다.
- 커뮤니티가 나뉘어져 있으니, 이곳 저것 찾아보는데 시간도 걸리고 번거로워서 한군데에 몰아두면 좋겠다는 생각을 했다.
- 데이터를 모아두면, 이걸 가지고 사용할 것들이 생긴다.

## Infra
- 구성:
  - Kubernetes가 운용 중인 homelab이 하나 있고, Argo CD도 구성되어 있었음.
  - Local LLM을 위한 mac-studio가 있다.
  - Health check을 위해 GCP에 무료로 vm이 하나 떠있다.
- 모두가 서로에게 접속할 수 있게 하기 위해 VPN으로 연결할 필요가 있었다.
- 이 부분은 다이어그램으로 개발용 맥북, homelab과 mac-studio는 집 네트워크에, GCP는 외부 네트워크에 표현해주고, 네트워크 연결을 VPN으로 해서 그려줘
  - 이 다이어그램을 다시 쓸거니까 잘 그려줘. 여기에 claude code와 codex가 SSH로 붙어서 작업하는 것을 이후로 묘사할건데, 컴퓨터 위에 agent가 올라타서 직접 작업하는 모양이면 좋겠어.

## Architectural design
- Provided the context in CLAUDE.md about what's needed, what infra I have, what platforms needed, what data stores needed
- Specified what platform to use: Airflow, Postgres DB, Elastic Search, Frontend page
- Asked claude code to split the tasks into by machines so that I can expertise the process with multiple sessions

## Operations
- DB
  - Provided schemas on what to crawl, what to store. Helps claude to understand the crawling codes and DB store schema
  - Executes the query on DB schema changes, data pruning, and normalization/denormalization
- Deployments: Synced with github and ArgoCD for automated deployment.
  - Pushes the code triggers github action, building docker image, pushing them to docker/github registry. ArgoCD synces
  - But sometimes, agents need to SSH into servers to manually update and deploy

## Working with multiple agents
- Claude code and Codex
  - Claude code: Infrastructures and operations
  - Codex: Writing codes of airflow dags, frontend app. 처음에는 Claude code가 infra setup을 다 할때까지 기다려야 했다.
- 여기서, infra diagram을 가지고 애니메이션을 만들면 좋은데:
  - 개발용 맥북에서 Claude code로 작업을 지시하면, claude code가 SSH로 homelab이나 mac-studio 등 다른 장비에 접속하여 작업하는 것
  - Codex는 github repo의 코드를 받아서 코드를 수정해서 github으로 푸시하는 것, 그리고 푸시된 코드가 빌드 되어 registry에 저장되면, 그걸 sync 하여 코드가 배포되는 것

## 몇가지 note들
- 처음에는 동작하지 않는 부분이 많았다.
  - 서버가 kubernetes에 잘 떴으나, 리소스가 너무 적게 할당되어 있었고, 
  - ElasticSearch에서 term extraction 옵션을 지정하지 않아서, 제대로 검색되지 않는 문제가 있었고,
  - airflow DAG들은 병렬처리가 되지 않거나, DAG 구성이 잘못되어 하나라도 실패하면 그 다음이 제대로 동작하지 않거나
  - 로그가 제대로 찍히지 않거나
  - 수집이 제대로 되지 않는 등의 문제
- 커뮤니티 별로 수집 로직을 주지 않고, 주소만 주었는데도 알아서 목록을 잘 파싱하긴 했다.
  - 시차 문제가 생기기도 해서, 시간 저장 방식, 본문 추출 HTML selector 수정 등의 개선을 했다.

## 수집 후
- Local LLM을 이용해 하루치 요약을 할 수 있게 되었다.
- 커뮤니티별로 인기 있는 탑10 글들을 모아서 요약하고, 이걸 다시 요약하는 방식으로 전날 있었던 토픽들을 요약해서 텔레그램으로 받도록 했다.
- 시간을 써서 무슨 일이 있었는지 캐치업 하지 않아도 내가 원하는 도메인들의 내용을 전달 받을 수 있었다.

## 응용
- 금융 소식을 받아서 요약 뉴스를 만들 수 있다. 데이터 수집하는게 필요하다.
- 해외 소식을 번역해서 유투브 영상으로 만들 수 있다.
