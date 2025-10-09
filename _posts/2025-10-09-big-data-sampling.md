---
layout: post
title:  "RDB에 1억 건의 샘플 데이터를 생성하고 적재하기"
author: jane
categories: [ bigdata, PostgreSQL ]
image: assets/images/12.jpg
tags: featured
---

이전 발행 글 <<수습 과제: A사 3월~7월 데이터 adhoc 구축>>의 `3.4 대용량 샘플 데이터(8천만 건) 생성 및 적재` 부분에서 `TABLESAMPLE SYSTEM (3.7) REPEATABLE (1004)`이란 SQL 문법을 사용해 기존에 생성된 데이터를 샘플링하는 방식으로 대용량 샘플 데이터를 만둘었다.

```
INSERT INTO acid.lrs_statement_new("timestamp", full_statement, actor_id)
SELECT
  (DATE '2025-03-01' + (floor(random()*153))::int)::timestamptz,
  s.full_statement,
  s.actor_id
FROM acid.lrs_statement AS s
TABLESAMPLE SYSTEM (3.7) REPEATABLE (1004) -- 균일 분포를 위한 랜덤 샘플링
LIMIT 1000000;
```

이 방법을 사용하기까지 수많은 의사결정과 번복이 있었기에, 어떤 과정을 거쳐 이 방법을 채택하게 되었는지 더 자세히 얘기해보고자 한다.


### 1. ~와의 비교 (무슨 row를 반복하는 방식 있었는데 그거)
### 2. 파일 COPY 방식과의 비교
### 3. `TABLESAMPLE BERNOULLI` 와의 비교

