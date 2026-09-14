---
layout: post
title: '[DuckDB] 05 Integration with GA4 & BigQuery'
subtitle: 'Integration with GA4 & BigQuery'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 05 - Integration with GA4 & BigQuery](#duckdb-05---integration-with-ga4--bigquery)
- [실무 연동 — pandas · GA4 · BigQuery · dbt](#실무-연동--pandas--ga4--bigquery--dbt)
- [1부. DataFrame 연동](#1부-dataframe-연동)
    - [언제 SQL, 언제 pandas?](#언제-sql-언제-pandas)
- [2부. GA4 형태 데이터 다루기](#2부-ga4-형태-데이터-다루기)
    - [패턴 1. 파라미터를 행으로 펼치기 (`UNNEST`)](#패턴-1-파라미터를-행으로-펼치기-unnest)
    - [패턴 2. 특정 파라미터만 컬럼으로 뽑기](#패턴-2-특정-파라미터만-컬럼으로-뽑기)
    - [패턴 3. 파라미터를 컬럼으로 피벗하기 (재사용 가능한 형태)](#패턴-3-파라미터를-컬럼으로-피벗하기-재사용-가능한-형태)
    - [패턴 4. 실전 지표 뽑기](#패턴-4-실전-지표-뽑기)
    - [DuckDB ↔︎ BigQuery 문법 대응표 (GA4 작업용)](#duckdb--bigquery-문법-대응표-ga4-작업용)
- [3부. BigQuery ↔︎ DuckDB 비용 절감 워크플로우](#3부-bigquery--duckdb-비용-절감-워크플로우)
  - [왜 필요한가](#왜-필요한가)
  - [연결하는 3가지 방법](#연결하는-3가지-방법)
  - [워크플로우 시뮬레이션](#워크플로우-시뮬레이션)
    - [스캔량 감각 익히기](#스캔량-감각-익히기)
- [4부. dbt-duckdb](#4부-dbt-duckdb)
  - [무엇인가](#무엇인가)
  - [설정 예시](#설정-예시)
  - [주의할 점](#주의할-점)
  - [정리](#정리)


# DuckDB 05 - Integration with GA4 & BigQuery

Created: September 4, 2026 3:16 PM
Class: DuckDB
Jupyter Notebook: duckdb05.ipynb

# 실무 연동 — pandas · GA4 · BigQuery · dbt

> 실습 파일: `duckdb05.ipynb`
> 
> 
> 학습 로드맵 **7단계(실무 연동)** 에 해당합니다. 앞 단계들이 “DuckDB를 어떻게 쓰는가”였다면, 여기서는 **“입사 후 실제 업무 흐름에 어떻게 끼워 넣는가”** 를 다룹니다.
> 

다루는 것:

1. **DataFrame 연동** — pandas와 SQL을 오가는 실무 패턴
2. **GA4 형태 데이터 다루기** — 중첩 구조를 만들고 `UNNEST`로 펼치기 (**DuckDB vs BigQuery 문법 차이 포함**)
3. **BigQuery ↔︎ DuckDB 비용 절감 워크플로우**
4. **dbt-duckdb** — 로컬 개발 환경으로서의 DuckDB

> 3번의 실제 BigQuery 연결은 GCP 자격증명이 필요해 실행하지 않고, 로컬 Parquet를 “BigQuery 테이블”로 가정해 **워크플로우 자체를 시뮬레이션**합니다.
> 

---

# 1부. DataFrame 연동

`duckdb00.md`에서 문법은 봤으니, 여기서는 **실무에서 왜, 언제 그렇게 쓰는지**를 봅니다.

```python
import duckdb
import pandas as pd

# 실무에서 흔한 상황: 어딘가(API, CSV, BQ)에서 받아온 데이터가 DataFrame으로 들어옴
orders_df = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6],
    "user_id":  ["u1", "u2", "u1", "u3", "u2", "u1"],
    "amount":   [12000, 35000, 8000, 52000, 15000, 22000],
    "status":   ["paid", "paid", "cancelled", "paid", "refunded", "paid"],
})

# 변수 이름을 테이블처럼 그대로 SQL에 쓸 수 있음 (별도 등록 불필요)
duckdb.sql("""
    SELECT
        user_id,
        count(*)     AS order_cnt,
        sum(amount)  AS total_amount
    FROM orders_df
    WHERE status = 'paid'
    GROUP BY user_id
    ORDER BY total_amount DESC
""").show()
```

```
┌─────────┬───────────┬──────────────┐
│ user_id │ order_cnt │ total_amount │
├─────────┼───────────┼──────────────┤
│ u3      │         1 │        52000 │
│ u2      │         1 │        35000 │
│ u1      │         2 │        34000 │
└─────────┴───────────┴──────────────┘
```

- **여기서 짚고 넘어갈 부분**: DuckDB는 이 DataFrame을 **자기 저장소로 복사하지 않고 원본 메모리를 그대로 스캔**합니다(`duckdb00.md`의 zero-copy). 그래서 수백만 행짜리 DataFrame이어도 `SELECT * FROM df`에 복사 비용이 들지 않습니다. 대신 읽기 전용이라 `INSERT`/`UPDATE`는 안 됩니다.

```python
# 결과를 다시 DataFrame으로 받아서 이후 파이썬 작업(시각화 등)으로 넘기기
result_df = duckdb.sql("""
    SELECT status, count(*) AS cnt, avg(amount) AS avg_amount
    FROM orders_df
    GROUP BY status
    ORDER BY cnt DESC
""").df()

print(type(result_df))
print(result_df)
```

```
<class 'pandas.DataFrame'>
      status  cnt  avg_amount
0       paid    4     30250.0
1   refunded    1     15000.0
2  cancelled    1      8000.0
```

### 언제 SQL, 언제 pandas?

| 작업 | 추천 | 이유 |
| --- | --- | --- |
| 집계·조인·윈도우 함수 | **SQL** | 표현이 짧고, 옵티마이저가 알아서 최적화 |
| 여러 테이블/파일 결합 | **SQL** | 조인이 SQL의 본업 |
| 행 단위 복잡한 로직, ML 전처리 | **pandas** | 파이썬 함수·라이브러리 생태계 |
| 시각화 직전 마무리 | **pandas** | matplotlib/seaborn이 DataFrame을 받음 |

실무에서는 **“무거운 집계는 SQL로 줄이고, 줄어든 결과를 pandas로 받아 마무리”** 하는 패턴이 가장 흔합니다.

```python
# 무거운 집계는 DuckDB가, 결과 5행만 pandas로
summary = duckdb.sql("""
    SELECT category, count(*) AS cnt, avg(value) AS avg_value
    FROM 'output/parquet/perf_test.parquet'
    GROUP BY category
    ORDER BY avg_value DESC
    LIMIT 5
""").df()

print(f"pandas로 넘어온 행 수:{len(summary)}행 (원본은 500만 행)")
print(summary)
```

```
pandas로 넘어온 행 수: 5행 (원본은 500만 행)
   category    cnt  avg_value
0        57  50000      549.0
1        14  50000      548.0
2        71  50000      547.0
3        28  50000      546.0
4        85  50000      545.0
```

500만 행을 pandas로 통째로 올리면 메모리가 위험하지만, SQL로 5행까지 줄인 뒤 `.df()`로 받으면 안전합니다.

---

# 2부. GA4 형태 데이터 다루기

여기가 이 노트의 핵심입니다. GA4의 BigQuery 익스포트는 `event_params`가 **STRUCT를 원소로 갖는 LIST**(`ARRAY<STRUCT<key, value>>`) 형태입니다. `duckdb03.md`에서 배운 LIST/STRUCT 지식을 실제 GA4 모양에 적용합니다.

실제 GA4 익스포트 스키마(간소화):

```
event_date       STRING
event_timestamp  INT64
event_name       STRING
user_pseudo_id   STRING
event_params     ARRAY<STRUCT<
                     key   STRING,
                     value STRUCT<string_value STRING, int_value INT64>
                 >>
```

```python
# GA4 익스포트와 같은 모양의 테이블을 로컬에 재현
duckdb.sql("DROP TABLE IF EXISTS events")
duckdb.sql("""
    CREATE TABLE events AS
    SELECT * FROM (VALUES
        ('20240115', 1705276800000000, 'page_view', 'user_001',
         [{'key': 'page_location', 'value': {'string_value': 'https://shop.com/home', 'int_value': NULL}},
          {'key': 'ga_session_id', 'value': {'string_value': NULL, 'int_value': 1001}},
          {'key': 'engagement_time_msec', 'value': {'string_value': NULL, 'int_value': 3500}}]),
        -- ... (이하 5개 이벤트 생략, 노트북 참조)
    ) AS t(event_date, event_timestamp, event_name, user_pseudo_id, event_params)
""")

duckdb.sql("DESCRIBE events").show()
```

```
┌─────────────────┬──────────────────────────────────────────────────────────────────────────────────┐
│   column_name   │                                   column_type                                    │
├─────────────────┼──────────────────────────────────────────────────────────────────────────────────┤
│ event_date      │ VARCHAR                                                                          │
│ event_timestamp │ BIGINT                                                                           │
│ event_name      │ VARCHAR                                                                          │
│ user_pseudo_id  │ VARCHAR                                                                          │
│ event_params    │ STRUCT("key" VARCHAR, "value" STRUCT(string_value VARCHAR, int_value INTEGER))[] │
└─────────────────┴──────────────────────────────────────────────────────────────────────────────────┘
```

`event_params`의 타입이 **`STRUCT(...)[]`** — 즉 STRUCT의 배열입니다. `duckdb03.md`에서 본 중첩 타입이 실제 GA4 스키마에서 이렇게 쓰입니다.

원본은 한 행 안에 파라미터 배열이 통째로 들어있습니다:

```
│ page_view  │ user_001 │ [{'key': page_location, 'value': {'string_value': 'https://shop.com/home', ...}},
│            │          │  {'key': ga_session_id, 'value': {'string_value': NULL, 'int_value': 1001}}, ...]
```

### 패턴 1. 파라미터를 행으로 펼치기 (`UNNEST`)

```python
duckdb.sql("""
    SELECT
        event_name,
        user_pseudo_id,
        ep.key                  AS param_key,
        ep.value.string_value   AS string_value,
        ep.value.int_value      AS int_value
    FROM events, UNNEST(event_params) AS u(ep)
    ORDER BY event_timestamp, param_key
""").show()
```

```
┌────────────┬────────────────┬──────────────────────┬──────────────────────────┬───────────┐
│ event_name │ user_pseudo_id │      param_key       │       string_value       │ int_value │
├────────────┼────────────────┼──────────────────────┼──────────────────────────┼───────────┤
│ page_view  │ user_001       │ engagement_time_msec │ NULL                     │      3500 │
│ page_view  │ user_001       │ ga_session_id        │ NULL                     │      1001 │
│ page_view  │ user_001       │ page_location        │ https://shop.com/home    │      NULL │
│ view_item  │ user_001       │ ga_session_id        │ NULL                     │      1001 │
│ view_item  │ user_001       │ item_name            │ duck plushie             │      NULL │
│ view_item  │ user_001       │ page_location        │ https://shop.com/item/42 │      NULL │
│ purchase   │ user_001       │ ga_session_id        │ NULL                     │      1001 │
│ purchase   │ user_001       │ transaction_id       │ T-9001                   │      NULL │
│ purchase   │ user_001       │ value                │ NULL                     │     52000 │
│    ...     │      ...       │         ...          │           ...            │       ... │
├────────────┴────────────────┴──────────────────────┴──────────────────────────┴───────────┤
│ 18 rows                                                                         5 columns │
└───────────────────────────────────────────────────────────────────────────────────────────┘
```

이벤트 6개가 파라미터 개수만큼 늘어나 **18행**이 됐습니다.

> **⚠️ DuckDB와 BigQuery의 별칭 문법이 다릅니다 (직접 실행해 확인한 것)**
> 
> 
> 위 쿼리에서 별칭을 `AS u(ep)` 라고 **두 겹**으로 썼습니다. BigQuery에서 쓰는 `AS ep` 한 겹 형태를 DuckDB에 그대로 넣으면 **에러가 납니다**:
> 
> ```
> Binder Error: Table "ep" does not have a column named "key"
> ```
> 
> DuckDB에서 `AS ep`는 “테이블 별칭 `ep`, 그 안의 컬럼명은 `unnest`”로 해석되기 때문에 `ep.unnest.key`로 접근해야 합니다. 즉 BigQuery 쿼리를 DuckDB로 가져올 때 **`AS ep` → `AS u(ep)` 로 바꿔주는 손질이 필요**합니다.
> 

```python
# 참고: BigQuery식 한 겹 별칭을 DuckDB에서 굳이 쓴다면 이렇게 됨 (권장하지 않음)
duckdb.sql("""
    SELECT event_name, ep.unnest.key AS param_key
    FROM events, UNNEST(event_params) AS ep
    LIMIT 3
""").show()
```

```
┌────────────┬───────────────┐
│ event_name │   param_key   │
├────────────┼───────────────┤
│ view_item  │ page_location │
│ view_item  │ ga_session_id │
│ view_item  │ item_name     │
└────────────┴───────────────┘
```

`ep.unnest.key` — 한 단계가 더 들어가서 읽기 나쁩니다. `AS u(ep)` 쪽을 쓰세요.

### 패턴 2. 특정 파라미터만 컬럼으로 뽑기

GA4 분석에서 가장 많이 쓰는 패턴이자, **BigQuery와 가장 크게 갈리는 지점**입니다.

BigQuery에서는 스칼라 서브쿼리를 씁니다:

```sql
-- BigQuery 전용 (DuckDB에서는 에러)
SELECT
    event_name,
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location') AS page_location
FROM events
```

DuckDB에서는 이 문법이 동작하지 않습니다:

```
Binder Error: Referenced column "key" not found in FROM clause!
```

대신 **람다를 쓰는 `list_filter`** 로 같은 일을 합니다:

```python
duckdb.sql("""
    SELECT
        event_date,
        event_name,
        user_pseudo_id,
        list_filter(event_params, p -> p.key = 'page_location')[1].value.string_value  AS page_location,
        list_filter(event_params, p -> p.key = 'ga_session_id')[1].value.int_value     AS ga_session_id,
        list_filter(event_params, p -> p.key = 'value')[1].value.int_value             AS purchase_value
    FROM events
    ORDER BY event_timestamp
""").show()
```

```
┌────────────┬────────────┬────────────────┬──────────────────────────┬───────────────┬────────────────┐
│ event_date │ event_name │ user_pseudo_id │      page_location       │ ga_session_id │ purchase_value │
├────────────┼────────────┼────────────────┼──────────────────────────┼───────────────┼────────────────┤
│ 20240115   │ page_view  │ user_001       │ https://shop.com/home    │          1001 │           NULL │
│ 20240115   │ view_item  │ user_001       │ https://shop.com/item/42 │          1001 │           NULL │
│ 20240115   │ purchase   │ user_001       │ NULL                     │          1001 │          52000 │
│ 20240115   │ page_view  │ user_002       │ https://shop.com/home    │          1002 │           NULL │
│ 20240116   │ page_view  │ user_003       │ https://shop.com/cart    │          1003 │           NULL │
│ 20240116   │ purchase   │ user_003       │ NULL                     │          1003 │          18000 │
└────────────┴────────────┴────────────────┴──────────────────────────┴───────────────┴────────────────┘
```

- **여기서 짚고 넘어갈 부분**: `[1]`이 붙는 이유는 `list_filter`가 **리스트를 반환**하기 때문입니다(조건에 맞는 원소가 여러 개일 수 있으므로). GA4의 `event_params`는 키가 이벤트당 하나씩이라 항상 첫 원소를 꺼내면 됩니다. 그리고 `duckdb03.md`에서 확인했듯 DuckDB 인덱스는 **1부터** 시작하므로 `[0]`이 아니라 `[1]`입니다.

### 패턴 3. 파라미터를 컬럼으로 피벗하기 (재사용 가능한 형태)

매번 `list_filter`를 반복하는 대신, 한 번 펼쳐서 넓은 테이블(wide table)로 만들어두면 이후 분석이 편합니다. 실무에서는 이걸 **중간 테이블/뷰**로 만들어 재사용합니다.

```python
duckdb.sql("DROP VIEW IF EXISTS events_flat")
duckdb.sql("""
    CREATE VIEW events_flat AS
    SELECT
        event_date,
        event_timestamp,
        event_name,
        user_pseudo_id,
        list_filter(event_params, p -> p.key = 'ga_session_id')[1].value.int_value        AS ga_session_id,
        list_filter(event_params, p -> p.key = 'page_location')[1].value.string_value     AS page_location,
        list_filter(event_params, p -> p.key = 'item_name')[1].value.string_value         AS item_name,
        list_filter(event_params, p -> p.key = 'transaction_id')[1].value.string_value    AS transaction_id,
        list_filter(event_params, p -> p.key = 'value')[1].value.int_value                AS event_value,
        list_filter(event_params, p -> p.key = 'engagement_time_msec')[1].value.int_value AS engagement_time_msec
    FROM events
""")

duckdb.sql("SELECT * FROM events_flat ORDER BY event_timestamp").show()
```

```
┌────────────┬────────────┬────────────────┬───────────────┬──────────────────────────┬──────────────┬────────────────┬─────────────┬──────────────────────┐
│ event_date │ event_name │ user_pseudo_id │ ga_session_id │      page_location       │  item_name   │ transaction_id │ event_value │ engagement_time_msec │
├────────────┼────────────┼────────────────┼───────────────┼──────────────────────────┼──────────────┼────────────────┼─────────────┼──────────────────────┤
│ 20240115   │ page_view  │ user_001       │          1001 │ https://shop.com/home    │ NULL         │ NULL           │        NULL │                 3500 │
│ 20240115   │ view_item  │ user_001       │          1001 │ https://shop.com/item/42 │ duck plushie │ NULL           │        NULL │                 NULL │
│ 20240115   │ purchase   │ user_001       │          1001 │ NULL                     │ NULL         │ T-9001         │       52000 │                 NULL │
│ 20240115   │ page_view  │ user_002       │          1002 │ https://shop.com/home    │ NULL         │ NULL           │        NULL │                 1200 │
│ 20240116   │ page_view  │ user_003       │          1003 │ https://shop.com/cart    │ NULL         │ NULL           │        NULL │                 8800 │
│ 20240116   │ purchase   │ user_003       │          1003 │ NULL                     │ NULL         │ T-9002         │       18000 │                 NULL │
└────────────┴────────────┴────────────────┴───────────────┴──────────────────────────┴──────────────┴────────────────┴─────────────┴──────────────────────┘
```

*(`event_timestamp` 컬럼은 지면 관계로 생략)*

중첩 구조가 완전히 사라지고 **평범한 테이블**이 됐습니다. 이제부터는 SQLD에서 배운 그대로의 SQL을 쓸 수 있습니다.

### 패턴 4. 실전 지표 뽑기

```python
# 일자별 기본 지표
duckdb.sql("""
    SELECT
        event_date,
        count(DISTINCT user_pseudo_id)                            AS users,
        count(DISTINCT ga_session_id)                             AS sessions,
        count(*) FILTER (WHERE event_name = 'page_view')          AS page_views,
        count(*) FILTER (WHERE event_name = 'purchase')           AS purchases,
        sum(event_value) FILTER (WHERE event_name = 'purchase')   AS revenue
    FROM events_flat
    GROUP BY event_date
    ORDER BY event_date
""").show()
```

```
┌────────────┬───────┬──────────┬────────────┬───────────┬─────────┐
│ event_date │ users │ sessions │ page_views │ purchases │ revenue │
├────────────┼───────┼──────────┼────────────┼───────────┼─────────┤
│ 20240115   │     2 │        2 │          2 │         1 │   52000 │
│ 20240116   │     1 │        1 │          1 │         1 │   18000 │
└────────────┴───────┴──────────┴────────────┴───────────┴─────────┘
```

```python
# 세션별 퍼널: 조회 → 상품조회 → 구매
duckdb.sql("""
    SELECT
        ga_session_id,
        user_pseudo_id,
        count(*)                                                  AS events,
        max(CASE WHEN event_name = 'view_item' THEN 1 ELSE 0 END) AS viewed_item,
        max(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END)  AS purchased,
        sum(event_value)                                          AS revenue
    FROM events_flat
    GROUP BY ga_session_id, user_pseudo_id
    ORDER BY ga_session_id
""").show()
```

```
┌───────────────┬────────────────┬────────┬─────────────┬───────────┬─────────┐
│ ga_session_id │ user_pseudo_id │ events │ viewed_item │ purchased │ revenue │
├───────────────┼────────────────┼────────┼─────────────┼───────────┼─────────┤
│          1001 │ user_001       │      3 │           1 │         1 │   52000 │
│          1002 │ user_002       │      1 │           0 │         0 │    NULL │
│          1003 │ user_003       │      2 │           0 │         1 │   18000 │
└───────────────┴────────────────┴────────┴─────────────┴───────────┴─────────┘
```

- **여기서 짚고 넘어갈 부분**: `count(*) FILTER (WHERE ...)` 는 `duckdb01.md`에서 본 `GROUP BY`/`HAVING`의 연장선이지만 표준 SQL의 편의 문법입니다. BigQuery에는 `FILTER` 절이 없어서 `COUNTIF(event_name = 'page_view')` 를 씁니다 — 이것도 옮길 때 손봐야 하는 지점 중 하나입니다.

### DuckDB ↔︎ BigQuery 문법 대응표 (GA4 작업용)

로컬에서 DuckDB로 개발한 쿼리를 BigQuery로 옮길 때 손봐야 하는 지점들입니다. **직접 실행해서 확인한 내용입니다.**

| 목적 | BigQuery | DuckDB |
| --- | --- | --- |
| 파라미터 전부 펼치기 | `FROM t, UNNEST(event_params) AS ep` → `ep.key` | `FROM t, UNNEST(event_params) AS u(ep)` → `ep.key` |
| 특정 파라미터 뽑기 | `(SELECT value.string_value FROM UNNEST(event_params) WHERE key='x')` | `list_filter(event_params, p -> p.key='x')[1].value.string_value` |
| 조건부 카운트 | `COUNTIF(cond)` | `count(*) FILTER (WHERE cond)` |
| 날짜 테이블 와일드카드 | `FROM `proj.ds.events_*`` + `_TABLE_SUFFIX` | `FROM 'events_*.parquet'` (glob) + `filename` 가상 컬럼 |
| 리스트 인덱스 | `OFFSET(0)` / `ORDINAL(1)` | `[1]` (1부터 시작) |

> **실무 팁**: 두 엔진을 오갈 거라면 **파라미터를 펼친 중간 테이블(`events_flat` 같은)을 먼저 만들어두는 습관**이 좋습니다. 중첩 구조를 다루는 문법만 엔진마다 다르고, 일단 평평해진 다음의 집계 SQL은 양쪽이 거의 동일하기 때문입니다.
> 

---

# 3부. BigQuery ↔︎ DuckDB 비용 절감 워크플로우

## 왜 필요한가

BigQuery는 **스캔한 데이터량 기준으로 과금**합니다. GA4 원본 이벤트 테이블은 쉽게 수백 GB~TB가 되는데, 쿼리를 한 번에 완성하는 사람은 없습니다. 오타 고치고, 조건 바꾸고, 컬럼 추가하고… **이 반복 하나하나가 다 돈입니다.**

```
[BigQuery]  전체 데이터에서 일부만 추출 (하루치, 또는 샘플)
     ↓ Parquet로 export
[로컬]      DuckDB로 쿼리를 수십 번 고쳐가며 완성 (비용 0원)
     ↓ 완성된 쿼리
[BigQuery]  전체 기간에 대해 한 번만 실행
```

## 연결하는 3가지 방법

**방법 A. BigQuery → GCS → Parquet → 로컬** (가장 흔함, 자격증명 최소)

```sql
-- BigQuery 콘솔에서 실행: 하루치만 GCS로 내보내기
EXPORT DATA OPTIONS(
  uri = 'gs://my-bucket/ga4_sample/*.parquet',
  format = 'PARQUET'
) AS
SELECT * FROM `my_project.analytics_123456.events_20240115`;
```

내려받은 뒤 로컬에서는 그냥 파일로 읽습니다: `SELECT * FROM 'ga4_sample/*.parquet'`

**방법 B. `bigquery` 커뮤니티 확장으로 직접 연결**

```sql
INSTALL bigquery FROM community;
LOAD bigquery;

ATTACH 'project=my_gcp_project' AS bq (TYPE bigquery, READ_ONLY);

-- BigQuery 테이블을 로컬 테이블처럼 조회
SELECT * FROM bq.analytics_123456.events_20240115 LIMIT 100;

-- 또는 GoogleSQL 쿼리를 BigQuery 쪽에서 실행하고 결과만 받기
SELECT * FROM bigquery_query('bq', 'SELECT event_name, count(*) FROM ... GROUP BY 1');
```

⚠️ **주의**: `SELECT * FROM bq....` 는 **BigQuery 쪽에서 스캔이 일어나므로 과금됩니다.** 로컬에서 실행한다고 공짜가 아닙니다. 비용을 줄이려면 결국 방법 A처럼 **한 번 받아서 로컬에 두고** 반복해야 합니다.

**방법 C. Python 클라이언트 경유**

```python
from google.cloud import bigquery
import duckdb

client = bigquery.Client()
df = client.query("SELECT * FROM `proj.ds.events_20240115` LIMIT 100000").to_dataframe()

# 1부에서 본 것처럼 DataFrame을 그대로 SQL로 쿼리
duckdb.sql("SELECT event_name, count(*) FROM df GROUP BY 1").show()
```

## 워크플로우 시뮬레이션

`duckdb04`에서 만든 500만 행 Parquet를 **“BigQuery에 있는 큰 테이블”** 이라고 가정합니다.

```python
bq_table = 'output/parquet/perf_test.parquet'
print("'BigQuery 테이블' 크기:", round(os.path.getsize(bq_table) / 1024 / 1024, 2), "MB")
duckdb.sql(f"SELECT count(*) AS total_rows FROM '{bq_table}'").show()
```

```
'BigQuery 테이블' 크기: 5.25 MB
┌────────────┐
│ total_rows │
├────────────┤
│    5000000 │
└────────────┘
```

**[1단계] BigQuery에서 샘플만 추출해 로컬로 가져왔다고 가정**

```python
duckdb.sql("""
    COPY (
        SELECT * FROM 'output/parquet/perf_test.parquet'
        WHERE id <= 100000          -- 하루치만 뽑는다고 가정
    ) TO 'output/parquet/bq_sample.parquet' (FORMAT parquet)
""")
```

```
로컬 샘플 크기: 0.41 MB
```

5.25MB → **0.41MB**. 실제 GA4라면 수백 GB → 수 GB에 해당합니다.

**[2단계] 로컬 샘플로 쿼리를 반복 개발** (여기서 몇 번을 고쳐도 비용 0원)

```python
duckdb.sql("""
    SELECT
        category,
        count(*)              AS cnt,
        round(avg(value), 1)  AS avg_value,
        max(value)            AS max_value
    FROM 'output/parquet/bq_sample.parquet'
    WHERE value > 500
    GROUP BY category
    ORDER BY avg_value DESC
    LIMIT 5
""").show()
```

```
┌──────────┬───────┬───────────┬───────────┐
│ category │  cnt  │ avg_value │ max_value │
├──────────┼───────┼───────────┼───────────┤
│       57 │   500 │     799.0 │       999 │
│       14 │   500 │     798.0 │       998 │
│       71 │   500 │     797.0 │       997 │
│       28 │   500 │     796.0 │       996 │
│       85 │   500 │     795.0 │       995 │
└──────────┴───────┴───────────┴───────────┘
```

**[3단계] 완성된 쿼리를 전체 데이터에 한 번만 실행**

```python
duckdb.sql("""
    SELECT
        category,
        count(*)              AS cnt,
        round(avg(value), 1)  AS avg_value,
        max(value)            AS max_value
    FROM 'output/parquet/perf_test.parquet'   -- 샘플 → 전체로 경로만 교체
    WHERE value > 500
    GROUP BY category
    ORDER BY avg_value DESC
    LIMIT 5
""").show()
```

```
┌──────────┬───────┬───────────┬───────────┐
│ category │  cnt  │ avg_value │ max_value │
├──────────┼───────┼───────────┼───────────┤
│       57 │ 25000 │     799.0 │       999 │
│       14 │ 25000 │     798.0 │       998 │
│       71 │ 25000 │     797.0 │       997 │
│       28 │ 25000 │     796.0 │       996 │
│       85 │ 25000 │     795.0 │       995 │
└──────────┴───────┴───────────┴───────────┘
```

- **여기서 짚고 넘어갈 부분**: 2단계와 3단계 쿼리는 **`FROM` 경로 한 줄만 다릅니다.** 그리고 결과의 **순위와 평균값이 완전히 동일**합니다(`cnt`만 50배). 샘플이 전체를 잘 대표하도록 뽑았다면 **로컬에서 검증한 결론이 전체에서도 그대로 성립**한다는 뜻입니다 — 이게 이 워크플로우가 성립하는 근거입니다. 반대로 샘플이 편향되면(예: 특정 시간대만) 이 전제가 깨지므로, **샘플을 어떻게 뽑을지가 실무의 핵심 판단**입니다.

### 스캔량 감각 익히기

`duckdb04.md`에서 배운 `EXPLAIN`으로 “이 쿼리가 BigQuery였다면 얼마나 스캔했을까”를 가늠할 수 있습니다.

```python
plan = duckdb.sql("""
    EXPLAIN
    SELECT category, avg(value)
    FROM 'output/parquet/perf_test.parquet'
    WHERE value > 500
    GROUP BY category
""").fetchone()[1]

print(plan)
```

```
        ... (상위 연산자 생략) ...
┌─────────────┴─────────────┐
│       PARQUET_SCAN        │
│    ────────────────────   │
│         Function:         │
│        PARQUET_SCAN       │
│                           │
│        Projections:       │
│           value           │
│          category         │
│                           │
│     Filters: value>500    │
│                           │
│      ~1,000,000 rows      │
└───────────────────────────┘
```

`Projections: value, category` — 4개 컬럼 중 2개만 읽습니다.

**BigQuery도 컬럼 지향이라 과금 방식이 똑같습니다.** `SELECT *`를 쓰면 안 쓰는 컬럼까지 전부 스캔 비용에 포함됩니다. GA4 이벤트 테이블은 컬럼이 수십 개이므로, **필요한 컬럼만 명시하는 것만으로 비용이 몇 배 차이**납니다.

| 습관 | BigQuery 비용 영향 |
| --- | --- |
| `SELECT *` | 모든 컬럼 스캔 → 최대 비용 |
| 필요한 컬럼만 명시 | 그 컬럼만 스캔 |
| 날짜 파티션 필터 없음 | 전체 기간 스캔 |
| `WHERE event_date BETWEEN ...` | 해당 파티션만 스캔 |

> `duckdb04.md`의 zonemap 실습에서 “정렬된 컬럼은 프루닝이 되고 흩어진 컬럼은 안 된다”를 봤는데, BigQuery의 **날짜 파티셔닝**이 정확히 같은 원리입니다. GA4가 `events_YYYYMMDD`로 날짜별 테이블을 나눠 익스포트하는 이유이기도 합니다.
> 

---

# 4부. dbt-duckdb

## 무엇인가

**dbt(data build tool)** 는 SQL로 데이터 변환 파이프라인을 관리하는 도구입니다. GA4 → BigQuery 파이프라인을 dbt로 모델링하는 회사가 많습니다.

**dbt-duckdb**는 dbt의 DuckDB 어댑터입니다:

```
개발/테스트 →  dbt + DuckDB (로컬, 빠르고 무료)
운영        →  dbt + BigQuery (실제 데이터)
```

**같은 dbt 모델 SQL을 양쪽에서 돌리는 것**이 목표라, 3부의 “쿼리 본문은 그대로, 대상만 교체”가 도구 차원에서 자동화됩니다.

## 설정 예시

`profiles.yml`에 타겟 두 개를 두고 전환합니다:

```yaml
my_ga4_project:
target: dev
outputs:
dev:                          # 로컬 개발용
type: duckdb
path: ./dev.duckdb
threads:4

prod:                         # 운영용
type: bigquery
method: service-account
project: my-gcp-project
dataset: analytics_marts
keyfile: /path/to/key.json
threads:8
```

```bash
dbt run                  # 기본 타겟(dev, DuckDB)으로 로컬 실행
dbt run --target prod    # BigQuery로 실행
```

> dbt는 별도 설치(`pip install dbt-duckdb`)가 필요해 실행하지 않았습니다. **“이런 게 있고 왜 쓰는지”** 정도만 알아두면, 입사 후 팀이 dbt를 쓰고 있을 때 맥락을 바로 잡을 수 있습니다.
> 

## 주의할 점

같은 SQL을 양쪽에서 돌린다고 해도, **2부에서 확인한 문법 차이는 여전히 남습니다.** dbt에서는 보통 이렇게 분기합니다:

{% raw %}
```sql
{% if target.type == 'bigquery' %}
    (SELECT value.string_value FROM UNNEST(event_params) WHERE key = 'page_location')
{% else %}
    list_filter(event_params, p -> p.key = 'page_location')[1].value.string_value
{% endif %}
```
{% endraw %}

또는 아예 **중첩 구조를 다루는 레이어(staging)는 엔진별로 따로 두고, 그 위 레이어부터 공통 SQL을 쓰는** 구조로 설계합니다. 2부에서 만든 `events_flat` 뷰가 바로 그 staging 레이어에 해당합니다.

---

## 정리

7단계에서 실제로 손에 남는 것:

1. **DataFrame 연동** — 무거운 집계는 SQL로 줄이고, 줄어든 결과만 pandas로 받는다 (zero-copy라 복사 비용 없음)
2. **GA4 중첩 데이터** — `list_filter(params, p -> p.key='x')[1].value...` 로 파라미터를 뽑고, 평평한 뷰를 만들어 재사용한다
3. **엔진 간 문법 차이** — `UNNEST` 별칭, 파라미터 추출, `COUNTIF` vs `FILTER`가 다르다. 중첩을 다루는 레이어만 분리해두면 나머지 SQL은 공용으로 쓸 수 있다
4. **비용 절감 워크플로우** — BigQuery에서 샘플만 내려받아 로컬에서 반복 개발하고, 완성된 쿼리만 전체 데이터에 실행한다. `FROM` 경로 한 줄만 바뀌도록 짜두는 것이 요령
5. **dbt-duckdb** — 위 워크플로우를 도구 차원에서 자동화한 것

이것으로 로드맵 1~7단계를 모두 마쳤습니다. 남은 건 **실제 GA4 데이터로 직접 부딪히는 것**입니다 — 입사 후 실제 `events_YYYYMMDD` 테이블을 열었을 때, 이 노트의 `events_flat` 뷰를 만드는 것부터 시작하시면 됩니다.
