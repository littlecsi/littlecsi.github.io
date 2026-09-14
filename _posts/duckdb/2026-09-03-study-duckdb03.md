---
layout: post
title: '[DuckDB] 03 Data Types'
subtitle: 'Data Types'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 03 - Data Types](#duckdb-03---data-types)
- [데이터 타입](#데이터-타입)
  - [General-Purpose Data Types](#general-purpose-data-types)
  - [중첩/복합 타입 (Nested / Composite Types)](#중첩복합-타입-nested--composite-types)
    - [중첩(Nesting)](#중첩nesting)
  - [List 타입](#list-타입)
    - [리스트 생성하기](#리스트-생성하기)
    - [리스트에서 값 가져오기](#리스트에서-값-가져오기)
    - [비교와 정렬](#비교와-정렬)
  - [Struct 데이터 타입](#struct-데이터-타입)
    - [Struct 생성하기](#struct-생성하기)
    - [필드 추가/갱신하기](#필드-추가갱신하기)
    - [Struct에서 값 가져오기](#struct에서-값-가져오기)
    - [`unnest` / `STRUCT.*`](#unnest--struct)
    - [점 표기법의 연산 순서](#점-표기법의-연산-순서)
    - [`row` 함수로 Struct 만들기](#row-함수로-struct-만들기)
    - [비교와 정렬](#비교와-정렬-1)
    - [스키마 갱신하기 (v1.3.0+)](#스키마-갱신하기-v130)
  - [정리](#정리)


# DuckDB 03 - Data Types

Created: September 3, 2026 4:33 PM
Class: DuckDB
Jupyter Notebook: duckdb03.ipynb

# 데이터 타입

> 원본: [duckdb.org/docs/current/sql/data_types/overview](https://duckdb.org/docs/current/sql/data_types/overview), […/list](https://duckdb.org/docs/current/sql/data_types/list), […/struct](https://duckdb.org/docs/current/sql/data_types/struct)
실습 파일: `duckdb03.ipynb`
> 

---

## General-Purpose Data Types

| 이름 | 별칭 | 설명 |
| --- | --- | --- |
| `BIGINT` | `INT8`, `LONG` | 부호 있는 8바이트 정수 |
| `BIT` | `BITSTRING` | 0과 1로 이루어진 문자열 |
| `BLOB` | `BYTEA`, `BINARY`, `VARBINARY` | 가변 길이 바이너리 데이터 |
| `BOOLEAN` | `BOOL`, `LOGICAL` | 논리값 (true/false) |
| `DATE` |  | 달력 날짜 |
| `DECIMAL(prec, scale)` | `NUMERIC(prec, scale)` | 고정 정밀도 숫자. 기본값 prec=18, scale=3 |
| `DOUBLE` | `FLOAT8` | 배정밀도 부동소수점 (8바이트) |
| `FLOAT` | `FLOAT4`, `REAL` | 단정밀도 부동소수점 (4바이트) |
| `HUGEINT` |  | 부호 있는 16바이트 정수 |
| `INTEGER` | `INT4`, `INT`, `SIGNED` | 부호 있는 4바이트 정수 |
| `INTERVAL` |  | 날짜/시간 간격 |
| `JSON` |  | JSON 객체 (json 확장) |
| `SMALLINT` | `INT2`, `SHORT` | 부호 있는 2바이트 정수 |
| `TIME` / `TIMESTAMP` / `TIMESTAMP WITH TIME ZONE` | `DATETIME` / `TIMESTAMPTZ` | 시각 / 날짜+시각 / 시간대 포함 날짜+시각 |
| `TINYINT` | `INT1` | 부호 있는 1바이트 정수 |
| `UBIGINT`/`UHUGEINT`/`UINTEGER`/`USMALLINT`/`UTINYINT` |  | 각 크기의 부호 없는 정수 |
| `UUID` |  | UUID |
| `VARCHAR` | `CHAR`, `BPCHAR`, `TEXT`, `STRING` | 가변 길이 문자열 |

---

## 중첩/복합 타입 (Nested / Composite Types)

| 이름 | 설명 | 컬럼 규칙 | 값으로 생성 | DDL 정의 |
| --- | --- | --- | --- | --- |
| `ARRAY` | 같은 타입, 길이 고정 시퀀스 | 모든 행이 같은 타입·같은 개수 | `[1, 2, 3]` | `INTEGER[3]` |
| `LIST` | 같은 타입, 길이 가변 시퀀스 | 타입은 같아야 하나 개수는 자유 | `[1, 2, 3]` | `INTEGER[]` |
| `MAP` | 이름 붙은 값들의 딕셔너리 (키 타입 통일, 값 타입 통일) | 행마다 다른 키 가능 | `map([1, 2], ['a', 'b'])` | `MAP(INTEGER, VARCHAR)` |
| `STRUCT` | 이름 붙은 값들의 딕셔너리 (키마다 값 타입이 달라도 됨) | 모든 행이 같은 키 | `{'i': 42, 'j': 'a'}` | `STRUCT(i INTEGER, j VARCHAR)` |
| `UNION` | 여러 대안 타입 중 하나를 저장, tag로 현재 타입 구분 | 행마다 다른 멤버 타입 가능 | `union_value(num := 2)` | `UNION(num INTEGER, text VARCHAR)` |
| `VARIANT` | 값마다 자기 타입 정보를 담는 반정형 타입 | 행마다 다른 타입 가능 | `42::VARIANT` | `VARIANT` |
- **MAP**의 키는 대소문자를 구분하지만, **UNION**과 **STRUCT**의 키는 구분하지 않습니다.
- 중첩 타입 값을 갱신할 때 DuckDB는 내부적으로 삭제 후 삽입(delete-then-insert)으로 처리합니다. ART 인덱스(명시적 인덱스, PK/UNIQUE 제약)가 걸린 테이블에서는 이 때문에 예상치 못한 제약 위반이 날 수 있습니다.
- **여기서 짚고 넘어갈 부분**: `MAP`과 `STRUCT`는 둘 다 “이름-값” 컬렉션이라 헷갈리기 쉽습니다. 표의 컬럼 규칙이 핵심 차이입니다 — `MAP`은 **행마다 키 집합이 달라도 되는** 동적 딕셔너리(런타임에 키가 결정되는 데이터용), `STRUCT`는 **모든 행이 같은 키를 가져야 하는** 고정 스키마(컬럼처럼 미리 알고 있는 필드용)입니다. GA4의 `event_params`처럼 키가 이벤트마다 달라지는 데이터는 `MAP`이나 “STRUCT의 LIST”로 표현되는 이유가 여기 있습니다. 그리고 `VARIANT`는 이 다섯 중 유일하게 **한 컬럼 안에서 행마다 아예 다른 타입**을 허용하는 catch-all 타입이라, 나머지 넷과는 결이 다릅니다.

### 중첩(Nesting)

`ARRAY`, `LIST`, `MAP`, `STRUCT`, `UNION`은 타입 규칙만 지키면 임의 깊이로 중첩됩니다.

```python
import duckdb

# LIST를 담은 STRUCT
duckdb.sql("SELECT {'birds': ['duck', 'goose', 'heron'], 'aliens': NULL, 'amphibians': ['frog', 'toad']}").show()
```

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                          s                                          │
│  struct(birds varchar[], aliens integer, amphibians varchar[])                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│ {'birds': [duck, goose, heron], 'aliens': NULL, 'amphibians': [frog, toad]}          │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

```python
# MAP의 LIST를 담은 STRUCT
duckdb.sql("SELECT {'test': [MAP([1, 5], [42.1, 45]), MAP([1, 5], [42.1, 45])]}").show()
```

```
┌────────────────────────────────────────────────┐
│                        s                        │
│      struct(test map(integer, decimal(11,1))[]) │
├────────────────────────────────────────────────┤
│ {'test': [{1=42.1, 5=45.0}, {1=42.1, 5=45.0}]}  │
└────────────────────────────────────────────────┘
```

```python
# UNION의 LIST
duckdb.sql("SELECT [union_value(num := 2), union_value(str := 'ABC')::UNION(str VARCHAR, num INTEGER)]").show()
```

```
┌───────────────────────────────────┐
│      union(str varchar, num integer)[]  │
├───────────────────────────────────┤
│ [2, ABC]                          │
└───────────────────────────────────┘
```

이 세 예제가 바로 GA4의 `event_params`(STRUCT의 LIST) 구조가 문법적으로 성립하는 근거입니다.

---

## List 타입

`LIST` 컬럼은 값들의 리스트를 인코딩합니다. 필드마다 길이는 달라도 되지만 기저 타입은 같아야 합니다. PostgreSQL의 `ARRAY`와 비슷하며, 호환을 위해 `array_` 계열 함수도 제공됩니다. (길이가 **고정**된 리스트는 `ARRAY` 타입을 씁니다.)

### 리스트 생성하기

```python
import duckdb

duckdb.sql("SELECT [1, 2, 3]").show()
```

```
┌──────────────────────────┐
│ main.list_value(1, 2, 3) │
│         int32[]          │
├──────────────────────────┤
│ [1, 2, 3]                │
└──────────────────────────┘
```

```python
duckdb.sql("SELECT ['duck', 'goose', NULL, 'heron']").show()
```

```
┌───────────────────────────┐
│         varchar[]         │
├───────────────────────────┤
│ [duck, goose, NULL, heron]│
└───────────────────────────┘
```

```python
duckdb.sql("SELECT [['duck', 'goose', 'heron'], NULL, ['frog', 'toad'], []]").show()
```

```
┌──────────────────────────────────────────────────┐
│                     varchar[][]                   │
├──────────────────────────────────────────────────┤
│ [[duck, goose, heron], NULL, [frog, toad], []]    │
└──────────────────────────────────────────────────┘
```

```python
duckdb.sql("SELECT list_value(1, 2, 3)").show()
```

```
┌─────────────────────┐
│ list_value(1, 2, 3) │
│       int32[]       │
├─────────────────────┤
│ [1, 2, 3]           │
└─────────────────────┘
```

```python
# INTEGER 리스트 컬럼과 VARCHAR 리스트 컬럼을 가진 테이블
duckdb.sql("DROP TABLE IF EXISTS list_table")
duckdb.sql("CREATE TABLE list_table (int_list INTEGER[], varchar_list VARCHAR[])")
```

### 리스트에서 값 가져오기

대괄호+슬라이싱 표기법, 또는 `list_extract`/`list_slice` 같은 함수를 씁니다.

```python
duckdb.sql("""
    SELECT
        ['a', 'b', 'c'][3]              AS idx_3,
        ['a', 'b', 'c'][-1]             AS idx_neg1,
        ['a', 'b', 'c'][2 + 1]          AS idx_expr,
        list_extract(['a', 'b', 'c'], 3) AS list_extract_3,
        ['a', 'b', 'c'][1:2]            AS slice_1_2,
        ['a', 'b', 'c'][:2]             AS slice_to_2,
        ['a', 'b', 'c'][-2:]            AS slice_neg2,
        list_slice(['a', 'b', 'c'], 2, 3) AS list_slice_2_3
""").show()
```

```
┌─────────┬──────────┬──────────┬────────────────┬───────────┬────────────┬────────────┬────────────────┐
│  idx_3  │ idx_neg1 │ idx_expr │ list_extract_3 │ slice_1_2 │ slice_to_2 │ slice_neg2 │ list_slice_2_3 │
├─────────┼──────────┼──────────┼────────────────┼───────────┼────────────┼────────────┼────────────────┤
│ c       │ c        │ c        │ c              │ [a, b]    │ [a, b]     │ [b, c]     │ [b, c]         │
└─────────┴──────────┴──────────┴────────────────┴───────────┴────────────┴────────────┴────────────────┘
```

- **여기서 짚고 넘어갈 부분**: `idx_3`(인덱스 3)와 `idx_neg1`(인덱스 -1)이 둘 다 `'c'`인 걸 보면 알 수 있듯, 이전에 확인한 대로 DuckDB는 **1부터 세는(1-based) 인덱싱**입니다. Python 배경이 있으시면 `[3]`을 “네 번째 원소”로 착각하지 않도록 주의하세요.

### 비교와 정렬

`LIST`는 위치 기준으로 비교됩니다: 같은 위치의 원소를 앞에서부터 비교하다가 다른 값이 나오면 그걸로 대소가 갈리고, 끝까지 같으면 더 긴 리스트가 큽니다.

```python
duckdb.sql("SELECT [1, 2] < [1, 3] AS result").show()          # true
duckdb.sql("SELECT [[1], [2, 4, 5]] < [[2]] AS result").show() # true
duckdb.sql("SELECT [ ] < [1] AS result").show()                # true
duckdb.sql("SELECT [1, 2] < [1, NULL, 4] AS result").show()    # true

duckdb.sql("SELECT [ ] < [ ] AS result").show()   # false
duckdb.sql("SELECT [1, 2] < [1] AS result").show() # false

duckdb.sql("SELECT NULL = [1] AS result").show()  # NULL
```

- **여기서 짚고 넘어갈 부분**: `NULL` 처리 방식이 두 층위로 다릅니다. 리스트 **자체**가 `NULL`과 비교되면(`NULL = [1]`) 결과는 `NULL`(PostgreSQL과 동일한 3값 논리). 하지만 리스트 **안의 원소**가 `NULL`이면(`[1, 2] < [1, NULL, 4]`) 그 `NULL`은 다른 모든 값보다 “큰” 것으로 취급되어 비교가 확정적으로 진행됩니다. 같은 `NULL`인데 최상위냐 중첩 레벨이냐에 따라 동작이 달라지는 셈이라 헷갈리기 쉬운 지점입니다.

리스트 함수 전체 목록은 별도의 List Functions 문서를 참고하세요.

---

## Struct 데이터 타입

`STRUCT` 컬럼은 “entries”라는 순서 있는 컬럼 목록을 담고, 각 entry는 문자열 키로 참조됩니다. **모든 행이 같은 키**를 가져야 하며(이 점이 PostgreSQL의 `ROW`와 다른 핵심 차이), 이 제약 덕분에 벡터화 실행 엔진을 온전히 활용해 성능과 타입 일관성을 모두 얻습니다. 키는 대소문자를 구분하지 않습니다.

### Struct 생성하기

```python
import duckdb

# struct_pack 함수 (키에 따옴표 없음, := 연산자)
duckdb.sql("SELECT struct_pack(key1 := 'value1', key2 := 42) AS s").show()

# 배열 표기법 (동일 결과)
duckdb.sql("SELECT {'key1': 'value1', 'key2': 42} AS s").show()

# row 변수로부터
duckdb.sql("SELECT d AS s FROM (SELECT 'value1' AS key1, 42 AS key2) d").show()
```

```
┌────────────────────────────────────┐
│                 s                  │
│ struct(key1 varchar, key2 integer) │
├────────────────────────────────────┤
│ {'key1': value1, 'key2': 42}       │
└────────────────────────────────────┘
```

(세 방식 모두 동일한 결과)

```python
duckdb.sql("SELECT {'yes': 'duck', 'maybe': 'goose', 'huh': NULL, 'no': 'heron'} AS s").show()
```

```
┌───────────────────────────────────────────────────────────────┐
│ struct("yes" varchar, maybe varchar, huh integer, "no" varchar) │
├───────────────────────────────────────────────────────────────┤
│ {'yes': duck, 'maybe': goose, 'huh': NULL, 'no': heron}         │
└───────────────────────────────────────────────────────────────┘
```

```python
duckdb.sql("SELECT {'key1': 'string', 'key2': 1, 'key3': 12.345} AS s").show()
```

```
┌───────────────────────────────────────────────────────┐
│ struct(key1 varchar, key2 integer, key3 decimal(5,3))  │
├───────────────────────────────────────────────────────┤
│ {'key1': string, 'key2': 1, 'key3': 12.345}            │
└───────────────────────────────────────────────────────┘
```

```python
duckdb.sql("""
    SELECT {
            'birds': {'yes': 'duck', 'maybe': 'goose', 'huh': NULL, 'no': 'heron'},
            'aliens': NULL,
            'amphibians': {'yes': 'frog', 'maybe': 'salamander', 'huh': 'dragon', 'no': 'toad'}
        } AS s
""").show()
```

```
{'birds': {'yes': duck, 'maybe': goose, 'huh': NULL, 'no': heron}, 'aliens': NULL, 'amphibians': {'yes': frog, 'maybe': salamander, 'huh': dragon, 'no': toad}}
```

### 필드 추가/갱신하기

```python
duckdb.sql("SELECT struct_update({'a': 1, 'b': 2}, b := 3, c := 4) AS s").show()
```

```
┌─────────────────────────────────────────┐
│ struct(a integer, b integer, c integer)  │
├─────────────────────────────────────────┤
│ {'a': 1, 'b': 3, 'c': 4}                 │
└─────────────────────────────────────────┘
```

`struct_update`는 기존 필드를 갱신하면서 새 필드도 추가합니다 (`b`는 3으로 갱신, `c`는 새로 추가). `struct_insert`는 추가만 가능하고 갱신은 안 됩니다.

### Struct에서 값 가져오기

```python
import duckdb

# 점 표기법
duckdb.sql("SELECT a.x FROM (SELECT {'x': 1, 'y': 2, 'z': 3} AS a)").show()

# 키에 공백이 있으면 큰따옴표로
duckdb.sql("""
    SELECT a."x space"
    FROM (SELECT {'x space': 1, 'y': 2, 'z': 3} AS a)
""").show()

# 대괄호 표기법 (문자열 키는 작은따옴표, 상수만 가능)
duckdb.sql("SELECT a['x space'] FROM (SELECT {'x space': 1, 'y': 2, 'z': 3} AS a)").show()

# struct_extract 함수 (위와 동등)
duckdb.sql("SELECT struct_extract({'x space': 1, 'y': 2, 'z': 3}, 'x space')").show()
```

네 방식 모두 `1`을 반환합니다 (`x`, `x space` 필드 조회).

### `unnest` / `STRUCT.*`

```python
duckdb.sql("""
    SELECT unnest(a)
    FROM (SELECT {'x': 1, 'y': 2, 'z': 3} AS a)
""").show()
```

```
┌───────┬───────┬───────┐
│   x   │   y   │   z   │
├───────┼───────┼───────┤
│     1 │     2 │     3 │
└───────┴───────┴───────┘
```

```python
duckdb.sql("""
    SELECT a.* EXCLUDE ('y')
    FROM (SELECT {'x': 1, 'y': 2, 'z': 3} AS a)
""").show()
```

```
┌───────┬───────┐
│   x   │   z   │
├───────┼───────┤
│     1 │     3 │
└───────┴───────┘
```

> **경고**: 별표(`*`) 표기법은 현재 최상위(top-level) struct 컬럼과 비집계 표현식에서만 쓸 수 있습니다.
> 
- **여기서 짚고 넘어갈 부분**: `unnest(a)`와 `a.* EXCLUDE(...)`는 결과가 비슷해 보이지만 용도가 다릅니다. `unnest`는 **struct의 키를 모를 때**(동적으로 만들어진 struct를 그대로 펼치고 싶을 때) 쓰고, `.* EXCLUDE`는 **키를 알고 있고 일부만 제외/조정하고 싶을 때** 씁니다. GA4의 `event_params`를 펼칠 때는 보통 `key`가 무엇이 올지 미리 모르니 `unnest` 계열 접근이 자연스럽습니다.

### 점 표기법의 연산 순서

`part1.part2` 같은 점 표기법은 스키마/테이블 참조와 모호할 수 있습니다. DuckDB는 **컬럼을 먼저 찾고, 그다음 struct 키**를 찾는 순서로 해석합니다 (점이 하나면 “테이블.컬럼”으로 먼저 시도한 뒤 “컬럼.속성”으로 재시도, 점이 둘이면 “스키마.테이블.컬럼” → “테이블.컬럼.속성” → “컬럼.속성.속성” 순). `.part4` 이후의 추가 부분은 항상 속성으로 취급됩니다.

### `row` 함수로 Struct 만들기

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS t1")
duckdb.sql("CREATE TABLE t1 (s STRUCT(v VARCHAR, i INTEGER))")
duckdb.sql("INSERT INTO t1 VALUES (row('a', 42))")
duckdb.sql("SELECT * FROM t1").show()
```

```
┌──────────────────────────────┐
│ struct(v varchar, i integer) │
├──────────────────────────────┤
│ {'v': a, 'i': 42}            │
└──────────────────────────────┘
```

`row('a', 42)::STRUCT(v VARCHAR, i INTEGER)`로 캐스팅해도 동일한 결과입니다.

`row` 함수로 **테이블의 struct 컬럼 자체**를 초기화하려 하면 실패합니다 — 아래는 실제로 재현되는 에러입니다:

```python
duckdb.sql("DROP TABLE IF EXISTS t2")
duckdb.sql("CREATE TABLE t2 AS SELECT row('a')")
```

```
InvalidInputException: Invalid Input Error: A table cannot be created from an unnamed struct
```

struct 간 캐스팅은 **최소 하나의 필드 이름이 일치**해야 합니다. 그래서 아래도 실패합니다:

```python
duckdb.sql("""
    SELECT a::STRUCT(y INTEGER) AS b
    FROM
        (SELECT {'x': 42} AS a)
""")
```

```
BinderException: Binder Error: STRUCT to STRUCT cast must have at least one matching member
```

- **여기서 짚고 넘어갈 부분**: 두 에러 모두 “이름 없는(unnamed) struct”가 원인입니다. `row('a')`는 키가 빈 문자열인 struct를 만드는데, 테이블 컬럼은 반드시 이름 붙은 필드가 필요해서 첫 에러가 납니다. 두 번째는 `{'x': 42}`(키 `x`)를 `STRUCT(y INTEGER)`(키 `y`)로 캐스팅하려는 것인데, 겹치는 키가 하나도 없어서 실패합니다. 우회법은 `struct_pack(y := a.x)`처럼 **명시적으로 이름을 다시 붙여주는 것**입니다:

```python
duckdb.sql("""
    SELECT struct_pack(y := a.x) AS b
    FROM
        (SELECT {'x': 42} AS a)
""").show()
```

`row`는 이름 없는 struct를 반환하는 용도로도 쓸 수 있고, 표현식이 여러 개면 `row`를 생략한 `(x, x + 1, y)` 형태도 동일하게 동작합니다:

```python
duckdb.sql("SELECT row(x, x + 1, y) FROM (SELECT 1 AS x, 'a' AS y) AS s").show()
# (1, 2, a)

duckdb.sql("SELECT (x, x + 1, y) AS s FROM (SELECT 1 AS x, 'a' AS y)").show()
# 위와 동일한 결과
```

### 비교와 정렬

`STRUCT`는 사전식(lexicographical) 순서로 비교되며, `NULL`은 다른 모든 값보다 큰 것으로 취급됩니다. 서로 다른 키를 가진 struct끼리 비교하면 키의 합집합으로 암묵 변환된 뒤 비교됩니다.

```python
duckdb.sql("SELECT {'k1': 0, 'k2': 0} < {'k1': 1, 'k2': 0}").show()          # true
duckdb.sql("SELECT {'k1': 'hello'} < {'k1': 'world'}").show()               # true
duckdb.sql("SELECT {'k1': 0, 'k2': 0} < {'k1': 0, 'k2': NULL}").show()      # true
duckdb.sql("SELECT {'k1': 0} < {'k2': 0}").show()                           # true
duckdb.sql("SELECT {'k1': 0, 'k2': 0} < {'k2': 0, 'k3': 0}").show()         # true
duckdb.sql("SELECT {'k1': 1, 'k2': 0} > {'k3': 0, 'k1': 0}").show()         # true

duckdb.sql("SELECT {'k1': 1, 'k2': 0} < {'k1': 0, 'k2': 1}").show()         # false
duckdb.sql("SELECT {'k1': [0]} < {'k1': [0, 0]}").show()                    # true
duckdb.sql("SELECT {'k1': 1} > {'k2': 0}").show()                           # false
duckdb.sql("SELECT {'k1': 0, 'k2': 0} < {'k3': 0, 'k1': 1}").show()         # true
duckdb.sql("SELECT {'k1': 1, 'k2': 0} > {'k2': 0, 'k3': 0}").show()         # false
```

### 스키마 갱신하기 (v1.3.0+)

`ALTER TABLE`로 struct의 하위 스키마를 바로 고칠 수 있습니다.

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS test")
duckdb.sql("CREATE TABLE test (s STRUCT(i INTEGER, j INTEGER))")
duckdb.sql("INSERT INTO test VALUES (ROW(1, 1)), (ROW(2, 2))")

# 필드 추가
duckdb.sql("ALTER TABLE test ADD COLUMN s.k INTEGER")
duckdb.sql("FROM test").show()
```

```
┌─────────────────────────────────────────┐
│ struct(i integer, j integer, k integer)  │
├─────────────────────────────────────────┤
│ {'i': 1, 'j': 1, 'k': NULL}              │
│ {'i': 2, 'j': 2, 'k': NULL}              │
└─────────────────────────────────────────┘
```

```python
# 필드 삭제
duckdb.sql("ALTER TABLE test DROP COLUMN s.i")
duckdb.sql("FROM test").show()
```

```
┌──────────────────────────────┐
│ struct(j integer, k integer) │
├──────────────────────────────┤
│ {'j': 1, 'k': NULL}          │
│ {'j': 2, 'k': NULL}          │
└──────────────────────────────┘
```

```python
# 필드 이름 변경
duckdb.sql("ALTER TABLE test RENAME s.j TO v1")
duckdb.sql("FROM test").show()
```

```
┌───────────────────────────────┐
│ struct(v1 integer, k integer) │
├───────────────────────────────┤
│ {'v1': 1, 'k': NULL}          │
│ {'v1': 2, 'k': NULL}          │
└───────────────────────────────┘
```

- **여기서 짚고 넘어갈 부분**: 이 기능은 `duckdb02.md`(Parquet)에서 다뤘던 **field ID 기반 스키마 진화**와 맞닿아 있습니다. 거기서는 “컬럼 이름이 바뀌어도 file ID로 추적된다”는 원리를 봤는데, 여기 `ALTER TABLE ... RENAME s.j TO v1`도 같은 문제(스키마가 시간이 지나며 바뀐다)를 STRUCT 레벨에서 SQL 문법으로 직접 푸는 방법입니다. 둘 다 “저장은 이미 됐는데 이름/구조가 바뀐다”는 상황을 다루는 도구라는 점에서 같은 계열입니다.

Struct 관련 함수 전체 목록은 별도의 Struct Functions 문서를 참고하세요.

---

## 정리

세 문서를 이어서 보면 GA4/BigQuery 맥락에서 필요한 그림이 완성됩니다: GA4의 `event_params`는 **STRUCT를 원소로 갖는 LIST** (`ARRAY<STRUCT<key, value>>`)이고, 이를 다루려면

1. **Overview**에서 본 “타입은 임의 깊이로 중첩 가능하다”는 원칙,
2. **List**에서 본 리스트 생성·인덱싱(1-based)·비교 방식,
3. **Struct**에서 본 키 접근(dot/bracket notation)과 `unnest`로 모든 키를 컬럼으로 펼치는 방법

이 셋이 합쳐져야 합니다. 다음 단계로 `sql/query_syntax/unnest` 문서를 보면, 여기서 다룬 `unnest(struct)`가 LIST에도 어떻게 적용되어 GA4의 `UNNEST(event_params)` 패턴으로 이어지는지 확인할 수 있습니다.
