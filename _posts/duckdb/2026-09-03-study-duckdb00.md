---
layout: post
title: '[DuckDB] 00 Overview'
subtitle: 'Overview'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 00 - Overview](#duckdb-00---overview)
- [DuckDB Python Client — Overview 강의 노트](#duckdb-python-client--overview-강의-노트)
  - [0. 환경 설정](#0-환경-설정)
  - [1. 기본 API 사용법 (Basic API Usage)](#1-기본-api-사용법-basic-api-usage)
    - [Relation: 지연 평가(lazy evaluation)되는 쿼리 결과](#relation-지연-평가lazy-evaluation되는-쿼리-결과)
  - [2. 데이터 입력 (Data Input)](#2-데이터-입력-data-input)
  - [3. DataFrame 연동 (Pandas / Polars / PyArrow)](#3-dataframe-연동-pandas--polars--pyarrow)
  - [4. 결과 변환 (Result Conversion)](#4-결과-변환-result-conversion)
  - [5. 디스크에 쓰기 (Writing Data to Disk)](#5-디스크에-쓰기-writing-data-to-disk)
  - [6. 연결 옵션 (Connection Options)](#6-연결-옵션-connection-options)
    - [In-Memory Database](#in-memory-database)
    - [Persistent Storage](#persistent-storage)
    - [Configuration](#configuration)
  - [7. Connection Object와 Module의 관계](#7-connection-object와-module의-관계)
  - [8. 병렬 Python 프로그램에서의 연결 사용 (Thread Safety)](#8-병렬-python-프로그램에서의-연결-사용-thread-safety)
    - [`cursor()`](#cursor)
  - [9. Community Extensions](#9-community-extensions)
  - [10. Unsigned Extensions](#10-unsigned-extensions)
  - [정리](#정리)


# DuckDB 00 - Overview

Created: September 3, 2026 9:24 AM
Class: DuckDB
Jupyter Notebook: duckdb00.ipynb

# DuckDB Python Client — Overview 강의 노트

> 원본: [duckdb.org/docs/current/clients/python/overview](https://duckdb.org/docs/current/clients/python/overview)
실습 파일: `duckdb00.ipynb`
> 
> 
> 이 문서는 “Overview” 페이지 하나를 다루는 노트이므로, 각 개념의 **내부 동작 원리까지 깊게 파고들지는 않습니다.** 대신 공식 문서에서 설명이 얕거나 왜 그런지 이유가 생략된 부분 위주로 살을 붙였습니다.
> 

---

## 0. 환경 설정

```python
# LTS 버전 설치
%pip install -q duckdb==1.4.5 pandas polars pyarrow
```

- `duckdb` 외에 `pandas`, `polars`, `pyarrow`를 함께 설치한 이유는 뒤에 나올 **“DataFrame 연동”** 섹션 때문입니다. DuckDB 자체는 이 라이브러리들 없이도 동작하지만, 셋 중 하나라도 설치돼 있으면 DuckDB가 이를 감지해 해당 객체를 SQL에서 바로 쿼리할 수 있게 해줍니다.

---

## 1. 기본 API 사용법 (Basic API Usage)

```python
import duckdb

duckdb.sql("SELECT 42").show()
```

```
┌───────┐
│  42   │
│ int32 │
├───────┤
│    42 │
└───────┘
```

- `duckdb.sql()`은 별도로 연결을 만들지 않고, **모듈 안에 전역으로 존재하는 in-memory DB**에 대고 쿼리를 실행합니다. 즉 파일 하나 안 만들고도 바로 SQL을 써볼 수 있는 가장 빠른 진입점입니다.

### Relation: 지연 평가(lazy evaluation)되는 쿼리 결과

```python
r1 = duckdb.sql("SELECT 42 AS i")
duckdb.sql("SELECT i * 2 AS k FROM r1").show()
```

```
┌───────┐
│   k   │
│ int32 │
├───────┤
│    84 │
└───────┘
```

- `duckdb.sql()`의 반환값은 실제 데이터가 아니라 **Relation**이라는, “이 쿼리를 실행하면 이런 결과가 나올 것”이라는 **쿼리의 상징적 표현**입니다.
- **여기서 짚고 넘어갈 부분**: 공식 문서는 “실행되지 않는다”고만 말하는데, 실무적으로 중요한 건 이게 **pandas의 `.query()`나 SQL의 `VIEW`와 비슷한 지연 평가 개념**이라는 점입니다. `r1`은 그 자체로는 아무 계산도 하지 않고, 위 코드처럼 **다른 쿼리 안에서 테이블처럼 재사용**될 때 비로소 SQL 옵티마이저가 두 단계를 하나로 합쳐 실행합니다. `.show()`, `.df()`, `.fetchall()` 등 **결과를 실제로 요청하는 순간**에만 실행이 트리거됩니다.

---

## 2. 데이터 입력 (Data Input)

```python
import duckdb

duckdb.read_csv("data/csv/example.csv")                  # CSV 파일을 Relation으로 읽기
duckdb.read_parquet("data/parquet/example.parquet")      # Parquet 파일을 Relation으로 읽기
duckdb.read_json("data/json/example.json")               # JSON 파일을 Relation으로 읽기

duckdb.sql("SELECT * FROM 'data/csv/example.csv'")           # CSV 파일을 SQL로 직접 쿼리
duckdb.sql("SELECT * FROM 'data/parquet/example.parquet'")   # Parquet 파일을 SQL로 직접 쿼리
duckdb.sql("SELECT * FROM 'data/json/example.json'")         # JSON 파일을 SQL로 직접 쿼리
```

```
┌───────┬─────────┬───────┐
│  id   │  name   │ price │
│ int64 │ varchar │ int64 │
├───────┼─────────┼───────┤
│     1 │ apple   │  1200 │
│     2 │ banana  │   800 │
│     3 │ cherry  │  5000 │
└───────┴─────────┴───────┘
```

- **여기서 짚고 넘어갈 부분**: 원본 문서는 두 가지 방식(`read_csv()` 계열 함수 vs SQL의 `FROM 'file'` 구문)을 나란히 보여주기만 하고 **왜 두 가지나 존재하는지는 설명하지 않습니다.** 이 둘은 동일한 기능에 대한 **두 개의 다른 진입점**입니다:
    - `duckdb.read_csv(...)`: **Python다운(functional) API** — 옵션을 Python 함수 인자로 넘기고, 반환값을 변수에 담아 이후 코드에서 계속 다룰 때 편리합니다.
    - `duckdb.sql("... FROM 'file.csv'")`: **SQL 네이티브 방식** — 이미 SQL을 짜고 있는 흐름 안에서 파일을 테이블처럼 자연스럽게 섞어 쓰고 싶을 때 편리합니다.
    - 둘 다 내부적으로는 같은 파일 스캔 로직을 타므로, 성능 차이는 없고 **코드 스타일의 선택 문제**입니다.

---

## 3. DataFrame 연동 (Pandas / Polars / PyArrow)

```python
import pandas as pd
pandas_df = pd.DataFrame({"a": [42]})
duckdb.sql("SELECT * FROM pandas_df")
```

```python
import polars as pl
polars_df = pl.DataFrame({"a": [42]})
duckdb.sql("SELECT * FROM polars_df")
```

```python
import pyarrow as pa
arrow_table = pa.Table.from_pydict({"a": [42]})
duckdb.sql("SELECT * FROM arrow_table")
```

세 경우 모두 동일한 결과:

```
┌───────┐
│   a   │
│ int64 │
├───────┤
│    42 │
└───────┘
```

- 세 라이브러리 모두 **파이썬 변수 이름을 그대로 SQL의 테이블명처럼 인식**해서 쿼리할 수 있다는 점이 핵심입니다. 별도의 `CREATE TABLE`이나 import 과정이 필요 없습니다.
- **문서에서 설명이 약한 부분**: 왜 “read-only”인지에 대한 이유가 생략돼 있습니다. DuckDB는 이 DataFrame들을 자기 저장소로 **복사해오지 않고, 원본 메모리를 그 자리에서 그대로 훑어보는(scan) 방식**으로 동작합니다. 그래서 `INSERT`/`UPDATE`로 그 데이터를 바꿀 수 없는 겁니다 — DuckDB 입장에서는 자신이 소유한 테이블이 아니라 “남의 메모리를 잠깐 들여다보는 것”이기 때문입니다. (이게 바로 데이터 복사 없이 여러 도구를 오가는 “zero-copy” 연동의 핵심 아이디어입니다.)

---

## 4. 결과 변환 (Result Conversion)

```python
duckdb.sql("SELECT 42").fetchall()   # 순수 Python 객체(튜플 리스트)로 변환
duckdb.sql("SELECT 42").df()         # Pandas DataFrame으로 변환
duckdb.sql("SELECT 42").pl()         # Polars DataFrame으로 변환
duckdb.sql("SELECT 42").arrow()      # Arrow Table로 변환
duckdb.sql("SELECT 42").fetchnumpy() # NumPy 배열로 변환
```

출력 예시 (`fetchnumpy()`):

```
{'42': array([42], dtype=int32)}
```

- Relation 하나에서 목적에 맞는 형태로 결과를 뽑아낼 수 있다는 게 핵심입니다. 예를 들어 시각화 라이브러리에 넘길 땐 `.df()`, 다른 Arrow 기반 시스템과 주고받을 땐 `.arrow()`처럼 **뒤에 어떤 도구를 쓸지에 따라 골라 쓰면 됩니다.**

---

## 5. 디스크에 쓰기 (Writing Data to Disk)

```python
duckdb.sql("SELECT 42").write_parquet("output/parquet/out.parquet")  # Relation 메서드로 Parquet 쓰기
duckdb.sql("SELECT 42").write_csv("output/csv/out.csv")              # Relation 메서드로 CSV 쓰기
duckdb.sql("COPY (SELECT 42) TO 'output/parquet/out.parquet'")       # SQL COPY 구문으로 쓰기
```

- 위 3~4단계와 마찬가지로, **Relation 메서드(`.write_parquet()` 등)**와 **SQL `COPY` 구문**이라는 두 개의 동등한 경로가 또 등장합니다. 지금까지의 패턴을 보면, DuckDB Python API는 대부분의 기능을 **“Python 메서드 방식”과 “순수 SQL 방식”** 두 가지로 항상 제공한다는 설계 원칙을 확인할 수 있습니다.

---

## 6. 연결 옵션 (Connection Options)

### In-Memory Database

```python
con = duckdb.connect()
con.sql("SELECT 42 AS x").show()
```

- 인자 없이 `duckdb.connect()`를 호출하면, 1단계의 `duckdb.sql()`이 쓰던 것과 같은 성격의 **in-memory DB**에 대한 새 연결을 얻습니다. 단, 이건 전역 연결과는 **별개의 독립적인 in-memory DB**입니다.

### Persistent Storage

```python
con = duckdb.connect("db/file.db")
con.sql("DROP TABLE IF EXISTS test")
con.sql("CREATE TABLE test (i INTEGER)")
con.sql("INSERT INTO test VALUES (42)")
con.table("test").show()
con.close()
```

```python
with duckdb.connect("db/file.db") as con:
    con.sql("DROP TABLE IF EXISTS test")
    con.sql("CREATE TABLE test (i INTEGER)")
    con.sql("INSERT INTO test VALUES (42)")
    con.table("test").show()
```

- 파일 경로를 넘기면 **파일 기반의 영속 DB**가 됩니다. 이 파일은 SQLite의 `.db` 파일처럼, 프로세스가 끝나도 남아있고 나중에 같은 경로로 다시 `connect()`하면 데이터가 그대로 복원됩니다.
- `with` 구문(context manager)을 쓰면 블록이 끝날 때 `con.close()`가 자동 호출됩니다 — 파일 핸들을 명시적으로 닫아야 하는 자원이라는 뜻이므로, 노트북/스크립트에서는 가급적 `with`를 쓰는 습관을 들이는 게 안전합니다.

### Configuration

```python
con = duckdb.connect(config={'threads': 1})
con = duckdb.connect(config={'storage_compatibility_version': 'latest'})
```

- `config` 딕셔너리로 스레드 수 제한, 저장 포맷 버전 같은 엔진 옵션을 조정할 수 있다는 정도만 “Overview” 수준에서는 알아두면 충분합니다.

---

## 7. Connection Object와 Module의 관계

- `duckdb` 모듈 자체(`duckdb.sql(...)`)와 `duckdb.connect()`로 얻은 연결 객체(`con.sql(...)`)는 **동일한 메서드 집합을 지원**합니다.
- 유일한 차이는 **모듈을 직접 쓰면 전역 in-memory DB 하나를 공유**하고, `connect()`는 **독립된 새 연결(또는 영속 파일)** 을 만든다는 점입니다.

---

## 8. 병렬 Python 프로그램에서의 연결 사용 (Thread Safety)

```python
# 안전한 사용: 스레드마다 새 연결을 만듦
def good_use():
    con = duckdb.connect()
    con.sql("SELECT 1").fetchall()

# 위험한 사용: 전역/공유 연결에 의존
def bad_use():
    con = duckdb.connect(':default:')
    return con.sql("SELECT 1").fetchall()

def also_bad():
    return duckdb.sql("SELECT 1").fetchall()
```

- 결론: `duckdb.sql()`이나 `duckdb.connect(':default:')`가 쓰는 **전역 연결은 스레드 세이프하지 않으므로, 여러 스레드에서 DuckDB를 쓴다면 스레드마다 자기만의 `connect()`를 가져야 합니다.**
- **문서에서 설명이 약한 부분**: 이게 왜 문제인지에 대한 배경이 빠져 있습니다. 헷갈리기 쉬운 점은, DuckDB는 **쿼리 실행 자체는 내부적으로 멀티스레드로 병렬 처리**합니다 (Python GIL도 쿼리 실행 중에는 해제됩니다). 여기서 말하는 “thread-safe하지 않다”는 것은 그 병렬 실행 능력과는 별개로, **하나의 연결(connection) 객체에 여러 Python 스레드가 동시에 쿼리를 “제출”하는 행위** 자체가 안전하지 않다는 뜻입니다. 즉 “연결 객체를 공유하지 마라”는 것이지 “DuckDB가 느리다”는 뜻이 아닙니다.

### `cursor()`

- `con.cursor()`는 **새 연결을 만드는 게 아니라, 같은 연결에 대한 또 다른 핸들**을 만드는 것뿐입니다. 그래서 한 연결에서 나온 cursor들끼리는 여전히 동시에 쿼리를 실행할 수 없습니다.
- (참고: `sqlite3` 같은 다른 DB 라이브러리의 `cursor()`와 동작 의미가 다르니, 다른 라이브러리 경험이 있다면 헷갈리지 않도록 주의가 필요합니다.)

---

## 9. Community Extensions

```python
con = duckdb.connect()
con.install_extension("h3", repository="community")
con.load_extension("h3")
```

- DuckDB는 지리 정보(`h3`)처럼 코어에 포함되지 않은 기능을 **확장(extension)** 형태로 설치해 붙일 수 있습니다. `repository="community"`는 DuckDB 팀이 아니라 **커뮤니티가 관리하는 확장 저장소**에서 가져온다는 뜻입니다.

## 10. Unsigned Extensions

```python
con = duckdb.connect(config={"allow_unsigned_extensions": "true"})
```

- **문서에서 설명이 약한 부분**: 왜 이게 별도 옵션으로 분리돼 있는지가 안 나와 있습니다. 확장은 결국 **DuckDB 프로세스 안에서 실행되는 바이너리 코드**이기 때문에, 기본적으로는 서명(sign)이 확인된 신뢰할 수 있는 확장만 로드를 허용합니다. `allow_unsigned_extensions`는 이 검증을 끄는 옵션이라 **보안 경계를 낮추는 설정**입니다 — 출처가 확실한 확장(예: 직접 빌드한 것)에서만 켜는 것이 안전합니다.

---

## 정리

이 Overview 페이지가 다룬 범위는 딱 **“Python에서 DuckDB를 어떻게 부르는가”** 까지입니다. 여기 나온 CSV/Parquet/JSON 읽기, DataFrame 연동, 저장 방식은 이후 학습 로드맵의 **SQL 문법(중첩 데이터, UNNEST 등)이나 파일 쿼리 최적화**와는 별개의 층위이니, 이 문서를 SQL 학습의 출발점이 아니라 **“연결 방법 참고서”** 정도로 활용하시면 됩니다.
