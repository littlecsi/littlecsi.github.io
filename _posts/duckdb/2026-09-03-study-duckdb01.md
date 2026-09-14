---
layout: post
title: '[DuckDB] 01 SQL Introduction'
subtitle: 'SQL Introduction'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 01 - SQL Introduction](#duckdb-01---sql-introduction)
- [SQL Introduction](#sql-introduction)
  - [1. 개념 (Concepts)](#1-개념-concepts)
  - [2. 새 테이블 만들기 (Creating a New Table)](#2-새-테이블-만들기-creating-a-new-table)
  - [3. 테이블에 행 채우기 (Populating a Table with Rows)](#3-테이블에-행-채우기-populating-a-table-with-rows)
    - [`COPY`가 `INSERT`보다 대량 적재에 빠른 이유](#copy가-insert보다-대량-적재에-빠른-이유)
  - [4. 테이블 조회하기 (Querying a Table)](#4-테이블-조회하기-querying-a-table)
  - [5. 테이블 간 조인 (Joins between Tables)](#5-테이블-간-조인-joins-between-tables)
  - [6. 집계 함수 (Aggregate Functions)](#6-집계-함수-aggregate-functions)
    - [`WHERE`에 집계 함수를 못 쓰는 이유 — 실제 에러로 확인](#where에-집계-함수를-못-쓰는-이유--실제-에러로-확인)
    - [`GROUP BY` / `HAVING`](#group-by--having)
  - [7. 갱신과 삭제 (Updates / Deletions)](#7-갱신과-삭제-updates--deletions)
  - [정리](#정리)


# DuckDB 01 - SQL Introduction

Created: September 3, 2026 10:39 AM
Class: DuckDB
Jupyter Notebook: duckdb01.ipynb

# SQL Introduction

> 원본: [duckdb.org/docs/current/sql/introduction](https://duckdb.org/docs/current/sql/introduction)
실습 파일: `duckdb01.ipynb`
> 
> 
> 이 페이지도 “Introduction(입문)” 튜토리얼이므로, `duckdb00.md`와 같은 기준으로 **내부 동작 원리까지 깊게 파고들지는 않습니다.** 이미 SQLD 수준의 SQL 지식이 있으신 만큼, `SELECT`/`WHERE`/`ORDER BY` 같은 기본 문법 자체보다는 **문서에서 설명이 얕은 부분, 또는 DuckDB 특유의 맥락**을 위주로 살을 붙였습니다.
> 

---

## 1. 개념 (Concepts)

- DuckDB는 관계형 데이터베이스 관리 시스템(RDBMS)입니다. **Relation**은 수학적으로 테이블을 가리키는 용어이고, 테이블은 스키마(schema) 안에 저장되며, 여러 스키마의 모음이 전체 데이터베이스를 구성합니다.
- **여기서 짚고 넘어갈 부분**: `duckdb00.md`에서 다룬 Python Relation(지연 평가되는 쿼리 결과)과 이름은 같지만 **다른 개념**입니다. 여기서 말하는 “relation”은 수학·SQL 이론에서 “테이블”을 부르는 용어이고, Python 클라이언트의 `Relation` 객체는 그 결과를 감싸는 지연 실행 래퍼입니다. 헷갈리지 않도록 구분해두시면 좋습니다.

---

## 2. 새 테이블 만들기 (Creating a New Table)

```python
import duckdb

duckdb.sql("""
    CREATE TABLE weather (
        city    VARCHAR,
        temp_lo INTEGER, -- 하루 중 최저 기온
        temp_hi INTEGER, -- 하루 중 최고 기온
        prcp    FLOAT,
        date    DATE
    )
""")
```

```python
duckdb.sql("""
    CREATE TABLE cities (
        name VARCHAR,
        lat  DECIMAL,
        lon  DECIMAL
    )
""")
```

- SQL은 대소문자를 구분하지 않고, `-`는 한 줄 주석이라는 점, `VARCHAR`/`INTEGER`/`FLOAT`/`DATE` 같은 표준 SQL 타입을 지원한다는 점은 이미 익숙하실 내용이라 그대로 넘어갑니다.
- 참고용으로만 제시된 문법(`DROP TABLE tablename;`)은 실제 테이블 이름이 아닌 자리표시자를 썼기 때문에 노트북에서 실행하지 않았습니다.

---

## 3. 테이블에 행 채우기 (Populating a Table with Rows)

```python
duckdb.sql("""
    INSERT INTO weather
    VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27')
""")

duckdb.sql("""
    INSERT INTO cities
    VALUES ('San Francisco', -194.0, 53.0)
""")

duckdb.sql("""
    INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
    VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29')
""")

duckdb.sql("""
    INSERT INTO weather (date, city, temp_hi, temp_lo)
    VALUES ('1994-11-29', 'Hayward', 54, 37)
""")
```

- 컬럼을 명시적으로 나열하는 스타일(`INSERT INTO weather (city, temp_lo, ...)`)이 암묵적 순서 의존보다 권장된다는 점은 실무에서도 그대로 통용되는 관례입니다.

### `COPY`가 `INSERT`보다 대량 적재에 빠른 이유

```sql
COPY weather
FROM 'weather.csv';
```

*(이 노트북에는 `weather.csv` 파일이 없어 실행하지 않았습니다.)*

- **여기서 짚고 넘어갈 부분**: 원문은 “`COPY`가 대량 로딩에 최적화되어 있어 더 빠르다”고만 말하고 이유는 설명하지 않습니다. `INSERT`는 한 문장마다 파싱·검증·트랜잭션 처리 오버헤드가 붙는 반면, `COPY`는 파일을 통째로 스트리밍하며 벌크 전용 경로로 데이터를 적재해 이 오버헤드를 건너뜁니다. [duckdb00.md](/study/2026/09/03/study-duckdb00/)에서 다룬 “파일을 직접 쿼리/적재”하는 능력과 같은 계열의 기능입니다.

---

## 4. 테이블 조회하기 (Querying a Table)

```python
duckdb.sql("SELECT * FROM weather").show()
```

```
┌───────────────┬─────────┬─────────┬───────┬────────────┐
│     city      │ temp_lo │ temp_hi │ prcp  │    date    │
│    varchar    │  int32  │  int32  │ float │    date    │
├───────────────┼─────────┼─────────┼───────┼────────────┤
│ San Francisco │      46 │      50 │  0.25 │ 1994-11-27 │
│ San Francisco │      43 │      57 │   0.0 │ 1994-11-29 │
│ Hayward       │      37 │      54 │  NULL │ 1994-11-29 │
└───────────────┴─────────┴─────────┴───────┴────────────┘
```

`SELECT`, 표현식(`(temp_hi + temp_lo) / 2 AS temp_avg`), `WHERE`, `ORDER BY`, `DISTINCT`는 표준 SQL 그대로이므로 결과만 요약합니다.

| 쿼리 | 결과 요약 |
| --- | --- |
| `WHERE city = 'San Francisco' AND prcp > 0.0` | San Francisco의 비 온 날 1건만 반환 |
| `ORDER BY city` | 도시명 기준 정렬 (동일 도시 내 순서는 비결정적일 수 있음) |
| `ORDER BY city, temp_lo` | 정렬 기준을 추가해 결과 순서를 완전히 결정 |
| `DISTINCT city` / `DISTINCT city ORDER BY city` | 중복 도시명 제거 |

---

## 5. 테이블 간 조인 (Joins between Tables)

```python
duckdb.sql("""
    SELECT *
    FROM weather, cities
    WHERE city = name
""").show()
```

```
┌───────────────┬─────────┬─────────┬───────┬────────────┬───────────────┬───────────────┬───────────────┐
│     city      │ temp_lo │ temp_hi │ prcp  │    date    │     name      │      lat      │      lon      │
├───────────────┼─────────┼─────────┼───────┼────────────┼───────────────┼───────────────┼───────────────┤
│ San Francisco │      46 │      50 │  0.25 │ 1994-11-27 │ San Francisco │      -194.000 │        53.000 │
│ San Francisco │      43 │      57 │   0.0 │ 1994-11-29 │ San Francisco │      -194.000 │        53.000 │
└───────────────┴─────────┴─────────┴───────┴────────────┴───────────────┴───────────────┴───────────────┘
```

- `FROM weather, cities WHERE city = name`처럼 콤마로 테이블을 나열하는 **옛 스타일 조인 문법**과, `INNER JOIN ... ON ...`이라는 **명시적 조인 문법**이 결과가 동일함을 보여줍니다.
- **여기서 짚고 넘어갈 부분**: 원문도 “콤마 스타일은 덜 흔히 쓰인다”고만 언급하고 넘어가는데, 실무에서 콤마 스타일이 기피되는 이유는 **조인 조건(`WHERE city = name`)과 필터 조건(`WHERE prcp > 0`)이 같은 `WHERE` 절에 섞여버려서, 테이블이 늘어날수록 어떤 조건이 조인용이고 어떤 게 필터용인지 코드만 보고 구분하기 어려워지기 때문**입니다. `INNER JOIN ... ON`은 조인 조건을 `ON`에, 필터는 `WHERE`에 분리해 이 문제를 없앱니다. 이후 SQL을 짤 땐 콤마 스타일보다 명시적 `JOIN`을 쓰시는 걸 권합니다.

```python
duckdb.sql("""
    SELECT *
    FROM weather
    LEFT OUTER JOIN cities ON weather.city = cities.name
""").show()
```

```
┌───────────────┬─────────┬─────────┬───────┬────────────┬───────────────┬───────────────┬───────────────┐
│     city      │ temp_lo │ temp_hi │ prcp  │    date    │     name      │      lat      │      lon      │
├───────────────┼─────────┼─────────┼───────┼────────────┼───────────────┼───────────────┼───────────────┤
│ San Francisco │      46 │      50 │  0.25 │ 1994-11-27 │ San Francisco │      -194.000 │        53.000 │
│ San Francisco │      43 │      57 │   0.0 │ 1994-11-29 │ San Francisco │      -194.000 │        53.000 │
│ Hayward       │      37 │      54 │  NULL │ 1994-11-29 │ NULL          │          NULL │          NULL │
└───────────────┴─────────┴─────────┴───────┴────────────┴───────────────┴───────────────┴───────────────┘
```

- `cities`에 없는 Hayward가 `LEFT OUTER JOIN`에서는 `NULL`로 채워져 살아남는 것을 확인할 수 있습니다 (INNER JOIN에서는 사라졌던 행).

---

## 6. 집계 함수 (Aggregate Functions)

```python
duckdb.sql("SELECT max(temp_lo) FROM weather").show()
```

```
┌──────────────┐
│ max(temp_lo) │
├──────────────┤
│           46 │
└──────────────┘
```

### `WHERE`에 집계 함수를 못 쓰는 이유 — 실제 에러로 확인

```python
duckdb.sql("""
    SELECT city
    FROM weather
    WHERE temp_lo = max(temp_lo)
""").show()
```

```
BinderException: Binder Error: WHERE clause cannot contain aggregates!
```

- `WHERE`는 **집계가 계산되기 전에** 어떤 행을 집계 대상으로 넣을지 걸러내는 단계이므로, 아직 계산되지 않은 집계값을 `WHERE` 안에서 참조하는 것 자체가 논리적으로 성립하지 않습니다. 서브쿼리로 바꾸면 해결됩니다:

```python
duckdb.sql("""
    SELECT city
    FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather)
""").show()
```

```
┌───────────────┐
│     city      │
├───────────────┤
│ San Francisco │
└───────────────┘
```

### `GROUP BY` / `HAVING`

```python
duckdb.sql("""
    SELECT city, max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40
""").show()
```

```
┌─────────┬──────────────┐
│  city   │ max(temp_lo) │
├─────────┼──────────────┤
│ Hayward │           37 │
└─────────┴──────────────┘
```

```python
duckdb.sql("""
    SELECT city, max(temp_lo)
    FROM weather
    WHERE city LIKE 'S%'
    GROUP BY city
    HAVING max(temp_lo) < 40
""").show()
```

```
┌─────────┬──────────────┐
│  city   │ max(temp_lo) │
├─────────┴──────────────┤
│         0 rows         │
└─────────────────────────┘
```

- **여기서 짚고 넘어갈 부분**: 마지막 쿼리가 `0 rows`를 반환하는 이유를 원문은 별도로 짚지 않는데, 직접 실행해보면 이게 왜 `HAVING`을 이해하는 데 좋은 예시인지 드러납니다. `city LIKE 'S%'`(`WHERE`)를 통과하는 도시는 San Francisco뿐인데, San Francisco의 `max(temp_lo)`는 46으로 `HAVING max(temp_lo) < 40` 조건을 만족하지 못합니다. 즉 `WHERE`로 걸러진 뒤에도 `HAVING`이 한 번 더 걸러내서 최종 0건이 된 것 — **`WHERE`와 `HAVING`이 서로 다른 단계에서 순차적으로 작동한다**는 걸 결과로 직접 확인할 수 있는 대목입니다.
- `WHERE`와 `HAVING`의 근본 차이(그룹·집계 계산 **이전** 행 필터 vs **이후** 그룹 필터)와, 가능하면 조건을 `WHERE`에 두는 게 더 효율적이라는 설명(불필요한 행을 그룹화·집계 전에 미리 제외)은 원문 설명이 이미 충분히 명확합니다.

---

## 7. 갱신과 삭제 (Updates / Deletions)

```python
duckdb.sql("""
    UPDATE weather
    SET temp_hi = temp_hi - 2, temp_lo = temp_lo - 2
    WHERE date > '1994-11-28'
""")
duckdb.sql("SELECT * FROM weather").show()
```

```
┌───────────────┬─────────┬─────────┬───────┬────────────┐
│     city      │ temp_lo │ temp_hi │ prcp  │    date    │
├───────────────┼─────────┼─────────┼───────┼────────────┤
│ San Francisco │      46 │      50 │  0.25 │ 1994-11-27 │
│ San Francisco │      41 │      55 │   0.0 │ 1994-11-29 │
│ Hayward       │      35 │      52 │  NULL │ 1994-11-29 │
└───────────────┴─────────┴─────────┴───────┴────────────┘
```

```python
duckdb.sql("""
    DELETE FROM weather
    WHERE city = 'Hayward'
""")
duckdb.sql("SELECT * FROM weather").show()
```

```
┌───────────────┬─────────┬─────────┬───────┬────────────┐
│     city      │ temp_lo │ temp_hi │ prcp  │    date    │
├───────────────┼─────────┼─────────┼───────┼────────────┤
│ San Francisco │      46 │      50 │  0.25 │ 1994-11-27 │
│ San Francisco │      41 │      55 │   0.0 │ 1994-11-29 │
└───────────────┴─────────┴─────────┴───────┴────────────┘
```

- **경고 (원문 그대로)**: 조건 없이 `DELETE FROM table_name;`을 실행하면 해당 테이블의 모든 행이 제거되며, 시스템은 실행 전에 확인을 요청하지 않습니다. 이 예시는 임의의 존재하지 않는 테이블명을 썼기 때문에 노트북에서 실행하지 않았습니다.

---

## 정리

이 페이지는 DuckDB가 **표준 SQL(PostgreSQL 방언)을 그대로 지원한다**는 걸 보여주는 입문 문서입니다. SQLD 지식으로 이미 아시는 `SELECT`/`JOIN`/`GROUP BY` 문법 자체보다, **DuckDB Python 클라이언트 안에서 이 문법들을 어떻게 실행하고 에러를 관찰하는지**가 이 노트가 실제로 더한 값입니다. 이후 로드맵에서 예고했던 `UNNEST`/`STRUCT`/`QUALIFY`/`PIVOT` 같은 DuckDB 고유 확장 문법은 이 “Introduction” 페이지 범위 밖이라 여기서는 다루지 않았습니다.
