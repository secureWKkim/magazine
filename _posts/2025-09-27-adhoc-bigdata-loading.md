---
layout: post
title:  "수습 과제: A사 3월~7월 데이터 adhoc 구축"
author: jane
categories: [airflow, bigdata, ETL, PostgreSQL, Problem Solving]
image: assets/images/6.jpg
tags: featured
---

# 1. 프로젝트 개요

## 1.1 프로젝트 목적

- **목적**: 운영 환경 DB에서 ?? 처리하기엔 데이터가 너무 많아 특정 기간의 데이터를 추출
- **주요 Mission**: 시간 효율 최우선. 운영 인덱스(timestamp, actor_id)와 파티셔닝 활용
- **구현 전략**: 월(파티션) 단위 → 키셋 페이지네이션 → COPY 스트리밍 → 타깃에 적재
- **검증**: EXPLAIN rows 추정치만 로그로 (가볍게)
- **중복 제어**: 기본 OFF(최대 속도). 필요 시 후처리 SQL 별도 실행.

## 1.2 수행 범위

## 1.3 기술 스택 및 환경

# 2. Sample Test 환경 구성 및 운영

### 2.1 Source DB / Stage DB / Airflow 컨테이너 정의

### 🔹 Source DB (원천 DB)

- **구성**: 회사 내부 PostgreSQL 16 서버 (외부에 별도 컨테이너 없음)
- **접속방식**: `SRC_PG_HOST`, `SRC_PG_PORT`, `SRC_PG_DB`, `SRC_PG_USER`, `SRC_PG_PASSWORD` 환경변수로 모든 서비스에 공통 주입
- **Airflow Connection**: `AIRFLOW_CONN_SRC_PG` URI를 통해 DAG 내 Operator들이 접근

### 🔹 Stage DB (적재 DB, 로컬 컨테이너)

- **구성**: `pg_stage` (Postgres 13 컨테이너)
- **환경변수**: `STG_PG_USER`, `STG_PG_PASSWORD`, `STG_PG_DB`
- **포트 매핑**: 호스트 5434 → 컨테이너 5432
- **데이터 저장소**: `pg-stage-data` 볼륨 마운트
- **헬스체크**: `pg_isready` 기반 DB 연결 상태 점검

### 🔹 Airflow 구성 (ETL 오케스트레이션)

- **공통 설정**: `x-airflow-common` 앵커 정의로 Executor, Broker, Backend, Volumes 등 일괄 적용
    - Executor: **CeleryExecutor**
    - Broker: **Redis**
    - Result Backend: **Postgres (airflow 메타DB)**
    - Volumes: `dags`, `logs`, `plugins`, `config` 공유
- **서비스별 컨테이너**
    - `airflow-webserver`: UI 제공 (포트 8080)
    - `airflow-scheduler`: DAG 스케줄링
    - `airflow-worker`: Celery 기반 작업 실행
    - `airflow-triggerer`: 이벤트 기반 트리거 관리
    - `airflow-init`: 초기 DB 마이그레이션 및 계정 생성
    - `airflow-cli`: 디버깅용 CLI
    - `flower`: Celery 모니터링 대시보드 (포트 5555)
- **의존성 관리**:
    - 모든 Airflow 서비스는 `postgres`, `redis`, `pg-remote-probe`, `pg_stage` 상태에 따라 기동

# 3. BigData TEST용 DB 관련 작업

## 3.1 PostgreSQL 환경 구축 (스키마 및 테이블 초기화)

### 3.1.1. DB서버 실행 환경

- 직접 설치 (도커 컨테이너 사용 x)
- 서버 사양
    - Ubuntu 24.02, 4Core CPU, 8GB RAM, 500GB **HDD**
- PostgreSQL 16  ***(Concern: 운영계 DB 버전은 13인 걸 고려하지 못함)***

### 3.1.2 DB 인스턴스 설정 변경

- data_directory 경로 변경(회사 파티션 서버임을 감안), 외부 접속 허용(listen_address), 사용자 계정 생성

### 3.1.3 스키마 구조 정의

- 원천 DB 테이블

```sql
CREATE TABLE acid.lrs_statement_new (
  "timestamp" timestamptz NOT NULL,
  full_statement jsonb NOT NULL,
  actor_id bigint
) PARTITION BY RANGE ("timestamp");

CREATE TABLE acid.lrs_statement_2025_03
  PARTITION OF acid.lrs_statement_new
  FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');

...

CREATE TABLE acid.lrs_statement_2025_07
  PARTITION OF acid.lrs_statement_new
  FOR VALUES FROM ('2025-07-01') TO ('2025-08-01');
```

- 원천 DB 인덱스

```sql
-- 1) 자식 파티션 인덱스 생성 (CONCURRENTLY)
CREATE INDEX CONCURRENTLY IF NOT EXISTS lrs_statement_03_ts_actor_idx ON acid.lrs_statement_2025_03 ("timestamp", actor_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS lrs_statement_04_ts_actor_idx ON acid.lrs_statement_2025_04 ("timestamp", actor_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS lrs_statement_05_ts_actor_idx ON acid.lrs_statement_2025_05 ("timestamp", actor_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS lrs_statement_06_ts_actor_idx ON acid.lrs_statement_2025_06 ("timestamp", actor_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS lrs_statement_07_ts_actor_idx ON acid.lrs_statement_2025_07 ("timestamp", actor_id);

-- 2) 부모에 파티션드 인덱스 생성(메타데이터; CONCURRENTLY 불가)
CREATE INDEX lrs_statement_ts_actor_idx
  ON acid.lrs_statement_new ("timestamp", actor_id);

-- 3) 자식 인덱스들을 부모 인덱스에 ATTACH
ALTER INDEX acid.lrs_statement_ts_actor_idx ATTACH PARTITION lrs_statement_03_ts_actor_idx;
ALTER INDEX acid.lrs_statement_ts_actor_idx ATTACH PARTITION lrs_statement_04_ts_actor_idx;
ALTER INDEX acid.lrs_statement_ts_actor_idx ATTACH PARTITION lrs_statement_05_ts_actor_idx;
ALTER INDEX acid.lrs_statement_ts_actor_idx ATTACH PARTITION lrs_statement_06_ts_actor_idx;
ALTER INDEX acid.lrs_statement_ts_actor_idx ATTACH PARTITION lrs_statement_07_ts_actor_idx;
```

- 적재 DB

```sql
CREATE SCHEMA IF NOT EXISTS pipeline;

CREATE TABLE IF NOT EXISTS pipeline.aidt_raw (
    id UUID,
    user_id VARCHAR,
    object_type VARCHAR,
    "statement" JSONB,
    "timestamp" TIMESTAMPTZ,
    inserted_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    flag CHAR(1) DEFAULT 'N',
    partner_id TEXT
);
```

- 운영 환경과 동일하게 맞춘 부분 vs 실험 목적으로 바꾼 부분 (필요한 컬럼만 넣음)

## 3.2 샘플 원천 데이터 이전

- 상황
    - acid.lrs_statement 테이블은 dump를 뜰 수 없는 상황 ⇒ pipeline.aidt_raw 안에 있
- 해결
    - 파일 생성 후 bulk update 보단 DB to DB 통신이 빠를 것 같다고 생각해 실행에 옮김

```sql
-- 2-1) FDW 확장 설치
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- 2-2) 소스 DB를 가리키는 서버 객체 생성 (도커 네트워크 호스트명 사용)
DROP SERVER IF EXISTS src_srv CASCADE;
CREATE SERVER src_srv FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'pg_source', port '5432', dbname 'srcdb');

-- 2-3) 인증 매핑 (소스 DB 계정/비번)
CREATE USER MAPPING FOR stguser SERVER src_srv
OPTIONS (user 'srcuser', password 'srcpass');

-- 2-4) 외부 테이블 가져올 스키마 (분리 권장)
CREATE SCHEMA IF NOT EXISTS src_fdw;

-- 2-5) 필요한 테이블만 임포트 (여기서 source_schema/source_table을 정확히 지정)
IMPORT FOREIGN SCHEMA source_schema
  LIMIT TO (source_table)
  FROM SERVER src_srv
  INTO src_fdw;
```

## 3.3 균일 분포 샘플 데이터 제작

### 3.3.1 배경

- 원본 `lrs_statement.insert.sql`에는 `actor.account.name`이 특정 값(`00066347-5ed0-5305-b2c4-82f28cd72560`)에 과도하게 치중.
- 동일한 이름은 동일한 `actor_id`를 갖기 때문에, 결과적으로 `actor_id` 값도 편향 → 인덱스 `(actor_id, timestamp)` 효율 저하.

### 3.3.2 목표

- 100가지 서로 다른 `actor.account.name`을 생성.
- 각 이름이 균등한 레코드 수를 갖도록 분배.
- 동일한 이름은 동일한 `actor_id`(1~100 사이 정수)를 가지도록 매핑.
- `timestamp` 및 다른 필드 값은 그대로 유지.

### 3.3.3 절차

1. 원본 SQL 파일에서 `INSERT` 구문을 모두 파싱.
2. 행 인덱스 기준으로 **`idx % 100`** 방식으로 100개의 버킷에 균등 분배.
3. 각 버킷에 대해:
    - `actor.account.name`을 UUIDv4 형식으로 새로 부여.
    - `actor_id`는 1~100까지의 정수를 대응시킴.
4. JSON 내부에서는 `actor.account.name`만 교체, 나머지 구조는 유지.
5. 최종적으로 수정된 `INSERT` 구문들을 새로운 `.sql` 파일로 출력.

### 3.3.4 결과

- 100개의 이름이 균등하게 분포된 샘플 데이터 생성.
- 인덱스 `(actor_id, timestamp)`의 분포 다양성이 확보됨.
- 이후 동일 데이터를 timestamp만 변경하여 대용량 데이터셋 생성 가능.
- 개발한 DAG task 모두 성공
    
    ![20.jpg](http://localhost:4000/magazine/assets/images/20.jpg)
    

## 3.4 대용량 샘플 데이터(8천만 건) 생성 및 적재

### 3.4.1 배경

- AI DT 파이프라인의 3~7월 데이터 예상 최대 산정량 = 3억 건
⇒ 시간, 서버 자원 허용 수준에서 최대한 유사한 조건을 재현

### 3.4.2 방법 설계

- 판단
    - 모든 값을 다 새롭게 생성하는 것은 비효율적 ⇒ DB 안의 기존 레코드 내 timestamp만 변경하는 동시 복제하여 샘플을 뻥튀기
- 결론

굳이 파일을 쓰는 **COPY로 돌아가지 말고 `INSERT … SELECT`(또는 CTAS→ATTACH)로 직접 자식 파티션에 적재**하는 게 더 빠르고 간단합니다. 왜냐면,

- COPY를 쓰려면 DB→파일(write)→DB(read)로 왕복 IO가 추가됩니다(특히 HDD면 더 손해)
    - **원천도, 목표도 같은 DB** → 파일 왕복이 순수 오버헤드
    - 파일 5개 만들고 반복 COPY도 가능하나, **HDD에서 파일 IO**는 큰 이점이 없습니다.
- `INSERT … SELECT`는 **엔진 내부에서 바로 읽고 바로 씁니다**(파일 왕복 없음 ⇒ ?)
    - INSERT…SELECT는 **단일 SQL**로 실행 경로가 짧고, 파이프라이닝/버퍼링이 효율적
- 대용량일수록 “부모→라우팅”보단 **자식 파티션에 직접 적재**가 유리합니다.
    - 파티션을 이미 월별 5개로 생성 → 라우팅 오버헤드 미미
- PostgreSQL 내부 처리)의 실제 장점:
    - 네트워크 I/O 제로: 모든 처리가 DB 내부
    - 최적화된 메모리 관리: PostgreSQL의 shared_buffers 활용
    - 트랜잭션 오버헤드 최소: 대용량 단일 트랜잭션 가능

### 3.4.3 절차

```sql
INSERT INTO acid.lrs_statement_new("timestamp", full_statement, actor_id)
SELECT
  (DATE '2025-03-01' + (floor(random()*153))::int)::timestamptz,
  s.full_statement,
  s.actor_id
FROM acid.lrs_statement AS s
TABLESAMPLE SYSTEM (3.7) REPEATABLE (1004) -- 균일 분포를 위한 랜덤 샘플링
LIMIT 1000000;
```

### 3.4.4 결과

- **생성+적재+AUTO VACUUM+WAL+트랜잭션 커밋** 소요 시간 = 최소 20분 ~ 최대 1시간 (250만 건 기준)
- **총 7,975만 건 | 152GB**

![21.png](http://localhost:4000/magazine/assets/images/21.png)

# 4. 데이터 파이프라인 구축 (Airflow)

## 4.1 DAG 구조 설계

### 4.1.1 Extract → Transform → Load 단계별 구성

- **Extract**: `SELECT_SQL`로 원천 테이블에서 키셋 페이지네이션(정렬 키: `timestamp, actor_id`)을 사용해 고정 크기(`BATCH_SIZE`)로 끊어 읽음.
    - **의도**: 오프셋 기반보다 인덱스-친화적이고 대용량에서도 성능이 선형적. 운영 인덱스 `(timestamp, actor_id)`만 타도록 WHERE/ORDER BY를 설계.
- **Transform(경량)**: SELECT 단계에서 필요한 필드만 JSON 연산자로 즉시 추출(`full_statement->...`). 애플리케이션 단 변환은 최소화해 I/O를 우선 병목으로 보고 처리량 확보.
- **Load**: `io.StringIO` 버퍼에 누적 후 `COPY ... FROM STDIN`을 **청크 단위(`COPY_CHUNK`)**로 흘려보내는 **스트리밍 로드**.
    - **의도**: row-by-row `INSERT`보다 수십 배 빠른 경향. 메모리 압박을 막기 위해 중간 flush(청크) 설계

## 4.2 DAG 구현

### **4.2.1 PythonOperator, PostgresHook 활용**

- **PythonOperator**로 전체 월 범위를 순회(`run_all_months`) → 10일 단위로 세분(`split_range`) → 각 윈도우마다 `run_one_month` 실행.
    - **의도**: 로직을 파이썬에서 제어하면 **키셋 페이지네이션**, **청크 COPY**, **워터마크**(마지막 `(timestamp, actor_id)`) 같은 세밀 제어가 쉬움.
- **PostgresHook** 두 개를 써서 **원천/타깃 커넥션 분리**. 커넥션 당 커서 수명도 짧게 잡아 리소스 점유 최소화.
- **빠른 사전 검증**: `EXPLAIN (FORMAT JSON)`으로 월 윈도우의 예상 rows를 **로그만 남김**(실 COUNT로 테이블 풀스캔을 피함).

### **4.2.2 task 설계 및 실행 주기**

- 현 버전은 **단일 태스크**(`run_etl`)로 설계.
    - **의도**: 초기 대용량 이관/적재에서는 **최대 처리량 확보**와 **코드 단순성**을 우선. COPY 스트리밍은 태스크 내부 루프가 유리.
- **스케줄**: 현재 `schedule=None, max_active_runs=1`
    - **의도**: 수동 기동/리런 시 중복 실행 방지. 대량 이관 종료 후에는 일 배치(예: `@daily`)로 전환 가능.

## 4.3 에러 대응 및 최적화

### **4.3.1 XCom 에러 트러블슈팅**

Airflow 2.x는 **PythonOperator의 리턴값을 XCom으로 자동 푸시**. 대용량 오브젝트(리스트/데이터프레임/문자열 버퍼)를 반환하면 XCom 사이즈 한도를 넘어 **직렬화/DB 에러**가 발생할 수 있음.

**1) 문제**: 특정 태스크(`extract_actor_id_list_task`) 실행 시 **`UnmappableXComLengthPushed`** 예외 발생

- 에러 로그:
    
    ```
    airflow.exceptions.UnmappableXComLengthPushed: unmappable return value length: 2031 > 1024
    ```
    
- 결과적으로 동적 매핑(`.expand()`)에 필요한 XCom 값이 비정상으로 처리되어 다운스트림 태스크 전부가 SKIPPED 처리됨.

**2) 원인**

- **XCom 자동 사용**
    - Airflow의 `@task` 또는 `PythonOperator` 반환값은 자동으로 XCom에 저장된다.
    - 코드에서 `xcom_push`, `xcom_pull`을 직접 호출하지 않아도 내부적으로 XCom이 항상 사용된다.
- **동적 태스크 매핑(expand)**
    - 매핑 대상 값은 반드시 XCom을 통해 전달된다.
    - 이 값의 길이가 Airflow 코어 설정값 `max_map_length`(기본 1024)를 초과하면 매핑 불가능 에러가 발생한다.
- **실제 상황**
    - `actor_id` 리스트가 2,031개 반환됨 → 제한(1024) 초과
    - 따라서 스케줄러가 매핑 태스크를 생성하지 못하고 에러 발생

**3) 해결 방법**

단편적인 해결 방법이라고 생각하는 `AIRFLOW__CORE__MAX_MAP_LENGTH` 환경 변수 조정은 차순위 방법

- 근본적인 해결 방법이 아니기에 나중에 다른 문제가 생길 수 있다고 생각
    - 태스크 인스턴스 수가 급격히 늘어날 경우 스케줄러/워커 부하가 커짐
    
    ```yaml
    environment:
      AIRFLOW__CORE__MAX_MAP_LENGTH: "5000"
    ```
    

⇒ 따라서 코드 구조를 개선.

**1) 청크 단위 반환**

- 리스트를 잘게 나눠 반환 → 매핑 개수 축소

```python
CHUNK_SIZE = 200
chunks = [actor_ids[i:i+CHUNK_SIZE] for i in range(0, len(actor_ids), CHUNK_SIZE)]
return chunks
```

- 다운스트림은 청크 단위 태스크 실행

**2) 외부 저장소 활용**

- 대량 데이터는 DB/파일에 저장
- XCom에는 페이지 번호, 범위 등 소량 메타데이터만 전달

**3) 매핑 단위 상향**

- 개별 actor 단위 대신 actor 그룹/청크 단위 태스크로 재설계

### **4.3.2 장애 발생 대응 전략**

- **워터마크 재시도**: 키셋 페이지네이션의 마지막 포인터 `(last_ts, last_actor_id)`를 로그로 남겨 **재기동 시 이어 받기** 가능
    - 권장: 마지막 포인터를 **타깃 테이블 또는 별도 체크포인트 테이블**에 저장해 재시작 시 자동 복구
- **트랜잭션 경계**: COPY 청크 flush마다 `commit`(현재는 윈도우 내 1커밋)
    - 대량 적재에서 장애 발생 시 **부분 재처리** 범위를 줄이려면 **청크별 커밋**으로 조정 가능(대신 커밋 오버헤드 증가)
- **타입/스키마 미스매치**(실무에서 자주 나는 오류)
    - 예: `invalid input syntax for type uuid` 류 → **타깃 컬럼 타입**과 COPY 텍스트 직렬화 포맷 불일치
    - 대응: 버퍼에 쓰기 전 **모든 필드를 문자열로 안전 변환**(탭/뉴라인 이스케이프, `None → \N`), **타깃 컬럼 순서**와 **형 변환** 명시. `to_char(timestamp, ...)` 같은 서버 사이드 캐스팅을 SELECT에 포함하면 더 안전
- **타임존 일관성**
    - 모든 커넥션 시작 시 `SET TIME ZONE 'UTC'`로 고정. SELECT 조건/워터마크/정렬이 **절대 시각** 기준으로 일치
- **운영 안전장치**
    - Airflow `retries=1`, `retry_delay=3m`로 **단기 네트워크 흔들림** 흡수
    - 원천/타깃 각각 **connect_timeout**, **statement_timeout**를 Hook 연결 문자열에 부여하는 것도 권장

# 5. Test: 대용량 데이터 대상 DAG 실행

### 5.1 테스트 목적 및 시나리오

- DAG를 통해 대용량 데이터(1억 건) 적재 시 안정적으로 동작하는지 확인
- Source DB → Stage DB로 Extract-Transform-Load 전 과정 부하 테스트(?)
- *(운영 환경과 유사한) 주기/동시성 조건에서의 성능 검증*

### 5.2 DAG 실행 조건

- 실행 주기: 없음
- 병렬 처리: CeleryExecutor 기반 worker 분산 실행
- 주요 태스크:
    - Extract (원천 DB에서 페이징/청크 단위 추출)
    - Transform (JSON 파싱 및 actor_id 매핑)
    - Load (Stage DB에 COPY 방식으로 적재)

### 5.4 모니터링 및 검증 방법

- 로그 → 배치 처리 및 적재 소요 시간, 처리 row 수, 에러 확인
- DB 모니터링 → `pg_stat_activity`, 디스크 I/O, 락 충돌 여부
- *성능 지표 → 처리 속도 (row/s), 총 소요 시간*

### 5.5 결과 및 분석

- 총 실행 시간 (예: 1억 건 적재에 N 시간 소요)
    - 한 배치 당 실행 시간: 8~10분 → 20분+ → 40분 → …
    - CPU 점유율: 왔다갔다 했다가, 후반부에 보니까 airflow triggerer 의 점유율이 꾸준히 1등
- 태스크별 병목 구간 확인 (Extract, Load, …)
- 튜닝 포인트: 배치 단위 크기, COPY vs INSERT, 파티셔닝 여부의 성능 차이, 인덱스의 효과

![22.png](http://localhost:4000/magazine/assets/images/22.png)

![23.png](http://localhost:4000/magazine/assets/images/23.png)

### 5.6 개선 방안 및 한계

- 개선 방안
    - 현재 개선이 아닌 문제 해결이 필요한 상황
- 한계
    - 로컬 테스트 환경과 운영 환경의 DB 사양 차이
    - 데이터 무결성 무시, 키 미설정
    ex) id/statement_id 미할당, (timestamp, actor_id)의 고유성 상실
        - timestamp의 날짜 뿐 아니라 시간까지 랜덤 생성해야 했음
        
        ⇒ 데이터 중복 검사 및 UPSERT 처리 x
        

# 6. 회고

## 6.1 잘한 점

## 6.2 아쉬운 점

- 과제가 막연히 쉬울 거라 생각하여 작업 시간 분배에 실패했다.
    - 대용량 샘플 데이터 생성과 DAG 실행에 대해 좀더 여러 번의 trial-error가 필요했다.
- DB 중고급 지식&경험 부재
    - 이를테면 적재량이 늘어날수록 Auto Vacuum, WAL이 잡아먹는 시간이 늘어나며 적재 속도가 느려진다는 것을 처음 알게 됨

## 6.3 다음 할 일

- 모니터링 구축 후 병목 원인/지점 발굴, 제대로 된 테스트 및 기록