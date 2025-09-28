---
layout: post
title: "airflow UID가 생성?되지 않는 문제"
author: jane
categories: [ airflow, docker ]
image: assets/images/1.jpg
---

### 문제 상황 (어떤 오류/이슈였는지)
docker compose 파일에 `user: "${AIRFLOW_UID:-50000}:0"` 라는 부분이 포함돼있음. 그럼에도 불구하고 AIRFLOW_UID가 생성되지 않아 `WARNING`이 뜨고, 실제 airflow-init 컨테이너를 비롯하여 여러 컨테이너를 띄울 때 에러를 보이며 airflow 구동에 실패함.

### 원인 분석

### 해결 방법 (코드/명령어 포함)
프로젝트 루트(Compose 파일이 있는 폴더)에 .env 파일을 만들어 항상 로드되게 한다.
```
echo "AIRFLOW_UID=$(id -u)" > .env
```

### 주의할 점 / 대체 방법
- 신기한 건, 실행 환경에 따라 AIRFLOW_UID가 항상 달랐다는 것.
- .env는 자동으로 읽히므로, 쉘에서 export 안 해도 됨

### 참고 자료 (링크, 문서 등)
ChatGPT5.0