---
layout: post
title: "Apache Airflow 완전 정리"
date: 2026-07-09
categories: [Data Engineering, Airflow]
tags: [airflow, dag, etl, data-pipeline, python, scheduler]
---

## 목차
1. [DAG 기본 파라미터](#1-dag-기본-파라미터)
2. [BashOperator](#2-bashoperator)
3. [PythonOperator](#3-pythonoperator)
4. [XCom](#4-xcom)
5. [DAG Run](#5-dag-run)
6. [Executor](#6-executor)
7. [Airflow 아키텍처](#7-airflow-아키텍처)
8. [Context](#8-context)
9. [DAG Processor & Serialization](#9-dag-processor--serialization)
10. [Operator, Hook, Connection](#10-operator-hook-connection)
11. [HttpOperator & HttpHook](#11-httpoperator--httphook)
12. [SQLExecuteQueryOperator](#12-sqlexecutequeryoperator)
13. [MySQL vs PostgreSQL 구조](#13-mysql-vs-postgresql-구조)
14. [데이터 처리 도구 비교](#14-데이터-처리-도구-비교)
15. [Spark + Airflow 연결](#15-spark--airflow-연결)
16. [도커 환경에서 Connection 설정](#16-도커-환경에서-connection-설정)

---

## 1. DAG 기본 파라미터

**DAG(Directed Acyclic Graph)** 는 작업(Task)들의 실행 순서와 의존관계를 정의하는 워크플로우입니다.

```python
from airflow import DAG
from datetime import datetime

dag = DAG(
    dag_id="sales_report",           # DAG 이름
    start_date=datetime(2024, 1, 1), # 시작일
    schedule="0 9 * * *",            # 매일 오전 9시
    catchup=False,                   # 밀린 실행 무시
)
```

| 파라미터 | 설명 |
|---|---|
| `dag_id` | DAG 고유 이름 |
| `start_date` | DAG 첫 실행 시작 날짜 |
| `schedule` | 실행 주기 (`@daily`, `@hourly`, cron 표현식 등) |
| `catchup` | 밀린 실행 소급 여부 (`True`/`False`) |

### catchup 예시

```
start_date = 2026-01-01, 오늘 = 2026-01-05, schedule = @daily

catchup=True  → 1/1, 1/2, 1/3, 1/4 총 4번 실행 (밀린 것 전부 실행)
catchup=False → 가장 최근 1번만 실행
```

---

## 2. BashOperator

Shell 명령어를 실행하는 Operator입니다.

```python
from airflow.operators.bash import BashOperator

task = BashOperator(
    task_id="print_date",
    bash_command="date",
)
```

### Jinja 템플릿으로 실행 날짜 활용

```python
task = BashOperator(
    task_id="with_date",
    bash_command="python etl.py --date {{ ds }}",
)
```

### 종료 코드와 실패 처리

BashOperator는 exit code 기준으로 성공/실패를 판단합니다.

```bash
exit 0  # 성공
exit 1  # 실패 → Airflow가 FAILED 처리
```

### 복잡한 스크립트 디버깅 방법

```python
# 복잡한 스크립트는 .sh 파일로 분리
BashOperator(
    task_id="complex_task",
    bash_command="/opt/airflow/scripts/etl.sh"
)
```

```bash
#!/bin/bash
set -e  # 에러 발생 시 즉시 중단
echo "===== 시작 ====="
python etl.py
echo "===== ETL 완료 ====="
```

---

## 3. PythonOperator

Python 함수를 직접 실행하는 Operator입니다.

```python
from airflow.operators.python import PythonOperator

def greet(name, age):
    print(f"안녕하세요, {name}! 나이: {age}")

task = PythonOperator(
    task_id="greet_task",
    python_callable=greet,
    op_kwargs={"name": "홍길동", "age": 30},
)
```

| 파라미터 | 설명 |
|---|---|
| `task_id` | 태스크 고유 ID |
| `python_callable` | 실행할 Python 함수 |
| `op_args` | 함수에 전달할 위치 인자 (리스트) |
| `op_kwargs` | 함수에 전달할 키워드 인자 (딕셔너리) |

---

## 4. XCom

태스크 간에 데이터를 주고받는 메커니즘입니다.

```
Task A → return 값 저장 → [XCom 저장소] → Task B가 꺼내서 사용
```

### push (return으로 자동 저장)

```python
def push_value():
    return "Hello From task_push"  # 자동으로 XCom에 저장
```

### pull (xcom_pull로 가져오기)

```python
def pull_value(ti):
    pulled = ti.xcom_pull(task_ids='task_push')
    print(f"pulled: {pulled}")
```

### 주의사항

| 항목 | 내용 |
|---|---|
| 저장 위치 | Airflow 메타DB |
| 용량 제한 | 소용량 데이터만 권장 |
| 대용량 데이터 | S3, GCS 등 외부 저장소 사용 권장 |

> 대용량 데이터는 S3/GCS에 저장하고 **경로만 XCom에 넘기는 패턴**이 일반적입니다.

---

## 5. DAG Run

DAG가 실제로 실행된 인스턴스 하나를 의미합니다.

```
DAG     = 레시피 (설계도)
DAG Run = 레시피로 실제 요리한 것 (실행 기록)
```

### DAG Run 종류

| 종류 | 설명 |
|---|---|
| Scheduled Run | `schedule`에 따라 자동 실행 |
| Manual Run | UI나 CLI에서 수동 실행 |
| Backfill Run | 과거 날짜 소급 실행 (`catchup`) |

### 주요 상태

```
running  → 실행 중
success  → 성공
failed   → 실패
queued   → 대기 중
```

---

## 6. Executor

### 종류 비교

| Executor | 병렬 실행 | 멀티 서버 | 용도 |
|---|---|---|---|
| `SequentialExecutor` | X | X | 테스트/개발 |
| `LocalExecutor` | O | X | 단일 서버 소규모 |
| `CeleryExecutor` | O | O | 운영 환경 대규모 |
| `KubernetesExecutor` | O | O | 쿠버네티스 환경 |

### CeleryExecutor 동작 방식

```
Scheduler → [Celery Broker (Redis/RabbitMQ)] → Worker 1
                                              → Worker 2
                                              → Worker 3
```

---

## 7. Airflow 아키텍처

```
DAG파일 감지 → Scheduler가 실행 결정
→ Executor가 Worker에 배분 (Queue 경유)
→ Worker가 API서버에서 정보 받아 실행
→ 결과 Metadata DB 저장
```

| 컴포넌트 | 역할 |
|---|---|
| DAG Processor | DAG 폴더 감시, 파싱 후 메타DB 저장 |
| Scheduler | 실행할 DAG 결정 후 Executor에 지시 |
| Executor | Worker 슬롯 관리 및 배분 |
| Worker | 실제 태스크 실행 |
| Metadata DB | 모든 실행 정보 저장 |
| API Server | 컴포넌트들이 정보 요청하는 창구 |

---

## 8. Context

Task가 실행될 때 Airflow가 자동으로 넘겨주는 실행 환경 정보입니다.

### 주요 Context 변수

| 변수 | 설명 |
|---|---|
| `logical_date` | DAG 논리적 실행 시점 |
| `ds` | logical_date의 날짜 문자열 (YYYY-MM-DD) |
| `data_interval_start/end` | 데이터 처리 범위의 시작과 끝 |
| `dag` | DAG 관련 객체 정보 |
| `dag_run` | 현재 실행 중인 DAG Run 객체 |
| `ti` | 현재 실행 중인 Task Instance 객체 |
| `params` | 사용자가 DAG 실행 시 입력한 파라미터 |

### Context 가져오는 3가지 방법

```python
# 방법 1: **context로 전부 받기
def my_task(**context):
    ds = context["ds"]
    ti = context["ti"]

# 방법 2: 필요한 것만 인자로 받기
def my_task(ds, ti, dag_run):
    print(ds)

# 방법 3: get_current_context() 사용
from airflow.operators.python import get_current_context
def my_task():
    context = get_current_context()
    ds = context["ds"]
```

---

## 9. DAG Processor & Serialization

### 동작 주기

| 설정 | 기본값 | 역할 |
|---|---|---|
| `refresh_interval` | 5분 | 새로운 DAG 파일 감지 |
| `min_file_process_interval` | 30초 | 기존 DAG 파일 변경 감지 |

### Serialization이란?

```
DAG 파이썬 코드
    ↓ JSON 형태로 변환
Metadata DB에 저장
    ↓
Worker가 DAG 파일 없어도 DB에서 꺼내서 실행 가능!
```

### DAG 버전이 바뀌는 조건

```python
# 버전 바뀌는 것 O
task_new = PythonOperator(...)       # 태스크 추가/삭제
task1 >> task2  →  task2 >> task1   # 태스크 순서 변경
schedule="@daily" → schedule="@hourly"  # 스케줄 변경

# 버전 안 바뀌는 것 X
# 주석 추가/삭제
# 함수 내부 로직 변경
# 공백, 들여쓰기 변경
```

---

## 10. Operator, Hook, Connection

```
Operator   = 주문자 (우리가 직접 쓰는 것)
Hook       = 실제 일꾼 (외부 시스템 연결 도구)
Connection = 접속 정보 (주소, 비밀번호)
```

### 전체 흐름

```
MySqlOperator 실행
    ↓
내부의 MysqlHook 호출
    ↓
Connection에서 접속 정보 꺼냄
    ↓
실제 MySQL 접속 & 쿼리 실행!
```

### Hook 단독으로 Task가 안 되는 이유

```python
# X Hook 단독 → Task 아님
hook = MySqlHook(mysql_conn_id="my_db")
hook.run("SELECT * FROM orders")

# O @task로 감싸야 Task가 됨
@task
def query():
    hook = MySqlHook(mysql_conn_id="my_db")
    return hook.get_records("SELECT * FROM orders")
```

---

## 11. HttpOperator & HttpHook

### 비교

| | HttpOperator | HttpHook |
|---|---|---|
| 태스크 수 | 2개 | 1개 |
| 데이터 가공 | 별도 태스크 필요 | 바로 가능 |
| 응답 형태 | 문자열 (XCom) | response 객체 |
| JSON 변환 | `json.loads()` 필요 | `response.json()` 바로 가능 |
| 디버깅 | 어려움 | 유연함 |

### HttpOperator 예시

```python
run_fetch = HttpOperator(
    task_id="fetch_data",
    http_conn_id="my_api",
    endpoint="api/v1/data",
    method="GET",
    data={
        "from": f"{{{{ ds }}}}T00:00",  # Jinja 템플릿
        "to":   f"{{{{ ds }}}}T23:59",
    },
)
```

### HttpHook 예시

```python
@task
def fetch_data(**context):
    date_str = context["ds"]
    hook = HttpHook(method="GET", http_conn_id="my_api")
    response = hook.run(endpoint="api/v1/data", data={"from": f"{date_str}T00:00"})
    data = response.json()  # 바로 dict로 변환!
    return data
```

### Jinja 템플릿을 쓰는 이유

```
HttpOperator는 객체라서 DAG 로딩 시점에 생성됨
→ 이 시점엔 ds(실행 날짜)를 모름!
→ {{ ds }}로 자리만 표시해두고
→ 실행 시점에 Airflow가 날짜로 채워줌
```

---

## 12. SQLExecuteQueryOperator

DB 종류 상관없이 하나로 통일된 SQL 실행 Operator입니다.

```python
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

create_table_op = SQLExecuteQueryOperator(
    task_id="create_table",
    conn_id="mysql_ecom_conn",
    sql="mysql_drop_create_users.sql"  # SQL 파일 사용 가능
)
```

### 예전 방식 vs 현재 방식

```
예전: MySQL    → MySqlOperator
     Postgres → PostgresOperator

현재: 모두     → SQLExecuteQueryOperator O
```

### 주의사항

```
SELECT 결과 → XCom에 저장됨
대용량 SELECT → Hook 사용 권장!
```

### get_conn() vs get_pandas_df()

```python
# get_pandas_df() - 간단하지만 제한적
df = hook.get_pandas_df(sql)

# get_conn() - 번거롭지만 유연
conn = hook.get_conn()
df = pd.read_sql(sql, conn)
conn.close()  # 반드시 닫아야 함!
```

| | get_pandas_df() | get_conn() |
|---|---|---|
| 코드 | 간단 | 길어짐 |
| 세션 설정 (SET search_path) | X | O |
| SQL 여러 개 실행 | X | O |
| conn.close() 필요 | X | O |

---

## 13. MySQL vs PostgreSQL 구조

### 구조 비교

```
MySQL      DB → Table (2단계)
PostgreSQL DB → Schema → Table (3단계)
```

### MySQL의 DB = PostgreSQL의 Schema

| MySQL | PostgreSQL |
|---|---|
| DB_Ecommerce | Schema_public |
| DB_Sales | Schema_stage |

### PostgreSQL 테이블 접근

```sql
-- schema명.테이블명
SELECT * FROM public.users;
SELECT * FROM stage.users;

-- search_path 설정 시 schema명 생략 가능
SET search_path TO stage;
SELECT * FROM users;
```

### Schema를 쓰는 이유

```
1. 용도별 분리 (운영/개발/테스트)
2. 권한 관리 (Schema 단위로 권한 부여)
3. 이름 충돌 방지 (같은 이름의 테이블을 Schema로 구분)
4. ETL 단계별 분리 (raw → stage → public)
```

---

## 14. 데이터 처리 도구 비교

| 도구 | 특징 | 적합한 규모 |
|---|---|---|
| pandas | 인메모리, 편리 | 수백만 건 이하 |
| Polars | 인메모리, Rust 기반으로 빠름 | 중규모 |
| Spark | 분산처리, 여러 서버 | 수억 건 이상 |
| Kafka | 실시간 데이터 전달 | 실시간 스트리밍 |

```
pandas  = 혼자 일하는 직원
Polars  = 혼자인데 엄청 빠른 직원
Spark   = 여러 명이 나눠서 일하는 팀
Kafka   = 데이터 배달부 (전달 전문)
```

---

## 15. Spark + Airflow 연결

```python
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator

spark_job = SparkSubmitOperator(
    task_id="run_spark_job",
    conn_id="spark_default",
    application="/opt/spark/etl.py",
    name="etl_job",
)
```

```
Airflow = 지휘자
Spark   = 오케스트라

Airflow가 "지금 Spark 실행해!"
→ Spark가 대용량 데이터 처리!
```

---

## 16. 도커 환경에서 Connection 설정

```
컨테이너마다 localhost가 따로 있음!

Airflow 컨테이너에서
localhost:5432 X (자기 자신한테 접속 시도)
postgres:5432  O (도커가 postgres 컨테이너 찾아줌)
```

```yaml
# docker-compose.yml
services:
  airflow:
    networks:
      - my_network
  postgres:          # ← 이 이름을 Host에 입력!
    networks:
      - my_network

networks:
  my_network:
```

> 같은 도커 네트워크 안이면 **IP 대신 컨테이너 이름으로 접속** 가능합니다.
