---
layout: post
title: "VSCode Remote SSH 접속 실패 문제 해결"
author: jane
categories: [ ssh, docker ]
image: assets/images/2.jpg
---

### 문제 상황 (어떤 오류/이슈였는지)
회사에서 사용하는 AWS EC2 .pem 키를 받아 ~/.ssh/config에 필요한 설정을 해주었음에도 불구하고 계속 키 권한 문제 에러로 인한 Connection fail 발생.

### 원인 분석
`chmod 600 (파일명)`을 해줬고, 네트워크 설정상의 접근 권한 문제(인바운드/아웃바운드 등)가 없는데도 같은 문제가 발생한다는 건 다른 게 원인이라는 것. 그래서 접속 실패하기까지의 로그 전체를 분석

### 해결
IdentityFile에 적힌 pem키 파일 경로명을 상대 경로가 아닌 절대 경로로 정확하게 명시했더니 해결됐다.
```
Host airflow-ec2
  HostName (IP주소)
  User ubuntu
  IdentityFile (pem키 파일 경로)
```

### 주의할 점 / 대체 방법
- 파일/폴더 경로가 문제되는 다른 경우들에도 저런 식의 방법이 통할 때가 있었다.

### 참고 자료 (링크, 문서 등)
ChatGPT5.0