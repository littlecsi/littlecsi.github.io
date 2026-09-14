---
layout: post
title: '[DuckDB] 04 Performance Intuition'
subtitle: 'Performance Intuition'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 04 - Performance Intuition](#duckdb-04---performance-intuition)
- [성능 감각 — EXPLAIN으로 쿼리 들여다보기](#성능-감각--explain으로-쿼리-들여다보기)
  - [1. `EXPLAIN` — 실행 계획 들여다보기](#1-explain--실행-계획-들여다보기)
    - [트리 읽는 법 (아래에서 위로 실행됩니다)](#트리-읽는-법-아래에서-위로-실행됩니다)
    - [추가 설정 (Additional Explain Settings)](#추가-설정-additional-explain-settings)
  - [2. `EXPLAIN ANALYZE` — 실제로 어디서 시간이 새는지 보기](#2-explain-analyze--실제로-어디서-시간이-새는지-보기)
  - [3. “My Workload Is Slow” 체크리스트](#3-my-workload-is-slow-체크리스트)
- [실습 — 500만 행으로 직접 확인하기](#실습--500만-행으로-직접-확인하기)
    - [실습 1. 프로젝션 pushdown을 계획에서 확인하기](#실습-1-프로젝션-pushdown을-계획에서-확인하기)
    - [실습 2. 필터 pushdown과 예상 카디널리티](#실습-2-필터-pushdown과-예상-카디널리티)
    - [실습 3. `EXPLAIN ANALYZE`로 실제 시간 보기](#실습-3-explain-analyze로-실제-시간-보기)
    - [실습 4. zonemap의 실체 보기 (이 노트의 하이라이트)](#실습-4-zonemap의-실체-보기-이-노트의-하이라이트)
    - [실습 5. 프로젝션 pushdown 효과 실측](#실습-5-프로젝션-pushdown-효과-실측)
    - [실습 6. 멀티스레드 확인](#실습-6-멀티스레드-확인)
  - [정리](#정리)


# DuckDB 04 - Performance Intuition

Created: September 4, 2026 1:33 PM
Class: DuckDB
Jupyter Notebook: duckdb04.ipynb

# 성능 감각 — EXPLAIN으로 쿼리 들여다보기

> 원본: [EXPLAIN: Inspect Query Plans](https://duckdb.org/docs/current/guides/meta/explain), [EXPLAIN ANALYZE: Profile Queries](https://duckdb.org/docs/current/guides/meta/explain_analyze), [My Workload Is Slow](https://duckdb.org/docs/current/guides/performance/my_workload_is_slow)
실습 파일: `duckdb04.ipynb`
> 

로드맵에 적힌 “컬럼 지향 + 벡터화 실행”, “Parquet 통계 기반 프루닝”, “자동 멀티스레드”의 **원리**는 이미 다뤘습니다:

- 컬럼 지향/벡터화 → Kersten 외 논문(*Everything You Always Wanted to Know About Compiled and Vectorized Queries*)
- Parquet 통계 기반 프루닝 → `duckdb02.md`의 “일부만 읽기(Partial Reading)” 섹션

그래서 이 노트의 목적은 개념이 아니라 **“실제로 느린 쿼리를 앞에 두고 무엇을 봐야 하는가”** 하는 도구 사용법입니다.

---

## 1. `EXPLAIN` — 실행 계획 들여다보기

`EXPLAIN` 문은 **물리적 계획(physical plan)**, 즉 실제로 실행될 쿼리 계획을 보여줍니다. 물리적 계획은 정해진 순서로 실행되는 **연산자(operator)들의 트리**이며, 쿼리 옵티마이저가 이 계획을 더 나은 형태로 변형합니다.

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS students")
duckdb.sql("DROP TABLE IF EXISTS exams")
duckdb.sql("CREATE TABLE students (name VARCHAR, sid INTEGER)")
duckdb.sql("CREATE TABLE exams (eid INTEGER, subject VARCHAR, sid INTEGER)")
duckdb.sql("INSERT INTO students VALUES ('Mark', 1), ('Joe', 2), ('Matthew', 3)")
duckdb.sql("INSERT INTO exams VALUES (10, 'Physics', 1), (20, 'Chemistry', 2), (30, 'Literature', 3)")
```

- **여기서 짚고 넘어갈 부분**: `EXPLAIN` 결과를 `.show()`로 보면 트리 전체가 표 한 칸에 뭉개져서 읽을 수 없습니다. **`.fetchone()[1]`로 문자열을 꺼내 `print`** 해야 트리 모양이 제대로 보입니다. 문서에는 이 얘기가 없는데, Python 클라이언트에서는 이걸 모르면 첫 시도부터 막힙니다.

```python
# EXPLAIN 결과는 (key, value) 한 행으로 나오며, 트리 문자열은 두 번째 컬럼에 들어있음
plan = duckdb.sql("""
    EXPLAIN
        SELECT name
        FROM students
        JOIN exams USING (sid)
        WHERE name LIKE 'Ma%'
""").fetchone()[1]

print(plan)
```

```
┌───────────────────────────┐
│         PROJECTION        │
│    ────────────────────   │
│            name           │
│                           │
│           ~1 row          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         HASH_JOIN         │
│    ────────────────────   │
│      Join Type: INNER     │
│   Conditions: sid = sid   ├──────────────┐
│                           │              │
│           ~1 row          │              │
└─────────────┬─────────────┘              │
┌─────────────┴─────────────┐┌─────────────┴─────────────┐
│         SEQ_SCAN          ││         SEQ_SCAN          │
│    ────────────────────   ││    ────────────────────   │
│        Table: exams       ││      Table: students      │
│   Type: Sequential Scan   ││   Type: Sequential Scan   │
│      Projections: sid     ││                           │
│                           ││        Projections:       │
│                           ││            sid            │
│                           ││            name           │
│                           ││                           │
│                           ││          Filters:         │
│                           ││  name>='Ma' AND name<'Mb' │
│                           ││                           │
│          ~3 rows          ││           ~1 row          │
└───────────────────────────┘└───────────────────────────┘
```

### 트리 읽는 법 (아래에서 위로 실행됩니다)

- **`SEQ_SCAN`**: 테이블 스캔. `Projections`(어떤 컬럼을 읽는지)와 `Filters`(어떤 조건이 스캔 단계로 밀려 들어갔는지)가 표시됩니다.
- **`HASH_JOIN`**: 해시 조인. 조인 종류(`INNER`)와 조인 조건(`sid = sid`)이 표시됩니다.
- **`PROJECTION`**: 최종적으로 내보낼 컬럼 정리.
- **`~N rows`**: **예상 카디널리티**. 이 연산자가 몇 행을 내보낼 것으로 옵티마이저가 추정했는지입니다.
- **여기서 짚고 넘어갈 부분 (문서와 실제 출력의 차이)**: 공식 문서는 예상 카디널리티를 **`EC: 1`** 형태로 표기한다고 설명하지만, 실제로 DuckDB 1.4.5에서 실행하면 **`~1 row`** 형태로 나옵니다. 문서 예제가 구버전 출력 형식이라 그렇습니다 — 표기만 다를 뿐 의미는 같습니다(`~` 물결표가 “추정치”라는 뜻).
- 또 하나: 문서 예제 트리에는 `FILTER` 연산자가 별도로 등장하는데, 실제 실행에서는 `name LIKE 'Ma%'` 조건이 **`SEQ_SCAN` 안의 `Filters:`로 완전히 밀려 들어가서** 별도 `FILTER` 연산자가 사라졌습니다. 이게 바로 필터 pushdown이 동작한 증거입니다.

`EXPLAIN`은 쿼리를 **실제로 실행하지 않으므로**, 여기 보이는 행 수는 전부 통계와 휴리스틱으로 계산한 **추정치**입니다.

### 추가 설정 (Additional Explain Settings)

| 설정 | 의미 |
| --- | --- |
| `PRAGMA explain_output = 'physical_only';` | 기본값. 물리적 계획만 표시 |
| `PRAGMA explain_output = 'optimized_only';` | 최적화된 논리 계획만 표시 |
| `PRAGMA explain_output = 'all';` | 물리적 계획과 최적화 계획을 모두 표시 |

```python
duckdb.sql("PRAGMA explain_output = 'all'")

plan_all = duckdb.sql("""
    EXPLAIN
        SELECT name
        FROM students
        JOIN exams USING (sid)
        WHERE name LIKE 'Ma%'
""").fetchone()[1]

print(plan_all)

duckdb.sql("PRAGMA explain_output = 'physical_only'")  # 기본값 복구
```

```
┌───────────────────────────┐
│         PROJECTION        │
│     Expressions: name     │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│           FILTER          │
│      (name ~~ 'Ma%')      │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│      COMPARISON_JOIN      │
│      Join Type: INNER     │
│    Conditions: sid = sid  ├──────────────┐
└─────────────┬─────────────┘              │
┌─────────────┴─────────────┐┌─────────────┴─────────────┐
│          SEQ_SCAN         ││          SEQ_SCAN         │
│      Table: students      ││        Table: exams       │
└───────────────────────────┘└───────────────────────────┘
```

- **여기서 짚고 넘어갈 부분**: 이 논리 계획에는 `FILTER` 연산자와 `COMPARISON_JOIN`이 그대로 남아 있습니다. 물리적 계획에서는 이게 각각 **스캔 내부의 `Filters:`** 와 **`HASH_JOIN`** 으로 바뀌었죠. 두 출력을 나란히 보면 **옵티마이저가 무엇을 바꿨는지**가 드러납니다 — 논리 계획은 “무엇을 할지”, 물리 계획은 “어떻게 할지”입니다.

---

## 2. `EXPLAIN ANALYZE` — 실제로 어디서 시간이 새는지 보기

`EXPLAIN ANALYZE`는 계획을 출력하는 동시에 **실제로 쿼리를 실행**해서 각 연산자의 **실행 시간**과 **실제 처리 행 수**를 보여줍니다.

```python
profile = duckdb.sql("""
    EXPLAIN ANALYZE
        SELECT name
        FROM students
        JOIN exams USING (sid)
        WHERE name LIKE 'Ma%'
""").fetchone()[1]

print(profile)
```

```
┌────────────────────────────────────────────────┐
││              Total Time: 0.0010s             ││
└────────────────────────────────────────────────┘
┌───────────────────────────┐
│      EXPLAIN_ANALYZE      │
│           0 rows          │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         PROJECTION        │
│            name           │
│           2 rows          │
│          (0.00s)          │
└─────────────┬─────────────┘
┌─────────────┴─────────────┐
│         HASH_JOIN         │
│      Join Type: INNER     │
│   Conditions: sid = sid   ├──────────────┐
│           2 rows          │              │
│          (0.00s)          │              │
└─────────────┬─────────────┘              │
┌─────────────┴─────────────┐┌─────────────┴─────────────┐
│         TABLE_SCAN        ││         TABLE_SCAN        │
│        Table: exams       ││      Table: students      │
│      Projections: sid     ││   Projections: sid, name  │
│                           ││          Filters:         │
│                           ││  name>='Ma' AND name<'Mb' │
│           3 rows          ││           2 rows          │
│          (0.00s)          ││          (0.00s)          │
└───────────────────────────┘└───────────────────────────┘
```

`EXPLAIN`과 비교했을 때 달라진 점:

- 맨 위에 **`Total Time`** 이 표시됨
- 각 연산자에 **실제 처리한 행 수**와 **`(0.00s)` 형태의 실행 시간**이 추가됨
- 연산자 이름이 `SEQ_SCAN` → `TABLE_SCAN`으로 바뀜 (실행 시점 표기)
- **여기서 짚고 넘어갈 부분 (문서와 실제 출력의 차이)**: 문서는 `EXPLAIN ANALYZE`가 “예상 카디널리티(EC)와 실제 카디널리티를 함께” 보여준다고 설명하지만, 실제 1.4.5 출력에는 **실제 행 수만** 찍히고 추정치는 없습니다. 따라서 **추정 vs 실제를 비교하려면 `EXPLAIN`(추정 `~N rows`)과 `EXPLAIN ANALYZE`(실제 `N rows`)를 각각 돌려서 눈으로 대조**해야 합니다. 이 대조가 중요한 이유는, **추정과 실제가 크게 어긋나면** 옵티마이저가 잘못된 조인 순서를 골랐을 가능성이 높기 때문입니다.

---

## 3. “My Workload Is Slow” 체크리스트

공식 문서가 제시하는, 느린 워크로드를 만났을 때 순서대로 점검할 항목입니다.

1. **메모리가 충분한가?** DuckDB는 **스레드당 1~4 GB**일 때 가장 잘 동작합니다.
2. **OS가 스왑하고 있지는 않은가?** 기본값(RAM의 80%)에서 `SET memory_limit = '...'`로 **낮춰보세요**. 직관에 반하지만, 다른 프로세스가 20% 이상 쓰는 환경에서는 오히려 빨라질 수 있습니다.
3. **빠른 디스크인가?** 네트워크 연결 디스크(클라우드 블록 스토리지)는 쓰기 위주·메모리 초과 워크로드를 느리게 만듭니다. 클라우드에서는 인스턴스 직결 NVMe SSD를 권장합니다.
4. **인덱스/제약조건을 쓰고 있는가?** 가능하면 꺼보세요. 적재·갱신 성능이 올라갑니다.
5. **올바른 타입인가?** 예: 날짜시각은 `TIMESTAMP`로.
6. **Parquet를 읽는가?** **row group 100k~1M**, **파일 100MB~10GB** 범위인지 확인하세요.
7. **계획이 제대로 나왔는가?** `EXPLAIN`으로 확인.
8. **병렬로 돌고 있는가?** `htop`이나 작업 관리자로 관찰.
9. **스레드를 너무 많이 쓰고 있지는 않은가?** 스레드 수를 제한해보세요.

> 6번이 `duckdb02.md`에서 다룬 `ROW_GROUP_SIZE` 트레이드오프와 직결됩니다 — 너무 작으면 메타데이터 오버헤드, 너무 크면 프루닝 입도가 나빠집니다.
> 

---

# 실습 — 500만 행으로 직접 확인하기

앞선 노트북의 예제 파일(3행짜리)은 너무 작아서 모든 게 `0.00s`로 나옵니다. `generate_series`로 500만 행짜리 Parquet를 만들어 씁니다.

```python
import duckdb, time, os

# 500만 행 테스트 데이터 생성
#   id       : 1부터 순차 증가 (row group 경계와 정렬됨)
#   category : 0~99 순환
#   value    : 0~999 순환 (전 구간에 고르게 흩어짐)
#   name     : 문자열 컬럼 (읽기 비용이 큰 편)
duckdb.sql("""
    COPY (
        SELECT
            i AS id,
            i % 100 AS category,
            (i * 7) % 1000 AS value,
            'name_' || (i % 50) AS name
        FROM generate_series(1, 5000000) t(i)
    ) TO 'output/parquet/perf_test.parquet' (FORMAT parquet, COMPRESSION zstd, ROW_GROUP_SIZE 100000)
""")
```

```
생성 시간: 0.54 초
파일 크기: 5.25 MB
```

500만 행이 **0.5초 만에 생성되고 5.25MB**로 압축됩니다. 원본을 CSV로 저장했다면 100MB를 넘겼을 데이터입니다 — `duckdb02.md`에서 다룬 “컬럼 단위 저장이라 압축률이 높다”가 실제 숫자로 나타난 것입니다.

### 실습 1. 프로젝션 pushdown을 계획에서 확인하기

```python
# 컬럼 4개짜리 파일에서 id 하나만 조회
plan = duckdb.sql("EXPLAIN SELECT id FROM 'output/parquet/perf_test.parquet'").fetchone()[1]
print(plan)
```

```
┌───────────────────────────┐
│       PARQUET_SCAN        │
│    ────────────────────   │
│         Function:         │
│        PARQUET_SCAN       │
│                           │
│      Projections: id      │
│                           │
│      ~5,000,000 rows      │
└───────────────────────────┘
```

`PARQUET_SCAN` 블록에 **`Projections: id`** 만 찍힙니다. 나머지 세 컬럼(`category`, `value`, `name`)은 **파일에서 아예 읽지 않습니다.** 이게 컬럼 지향 저장의 실질적 이득입니다.

### 실습 2. 필터 pushdown과 예상 카디널리티

```python
plan = duckdb.sql("EXPLAIN SELECT id FROM 'output/parquet/perf_test.parquet' WHERE value > 990").fetchone()[1]
print(plan)
```

```
┌───────────────────────────┐
│       PARQUET_SCAN        │
│    ────────────────────   │
│         Function:         │
│        PARQUET_SCAN       │
│                           │
│      Projections: id      │
│     Filters: value>990    │
│                           │
│      ~1,000,000 rows      │
└───────────────────────────┘
```

달라진 점 두 가지:

- **`Filters: value>990`** 이 스캔 연산자 안으로 들어갔습니다. 스캔한 **뒤에** 거르는 게 아니라 **스캔하면서** 거릅니다.
- 예상 행 수가 `~5,000,000` → **`~1,000,000`** 으로 줄었습니다. 옵티마이저가 통계를 보고 “이 필터를 통과하는 건 약 20%”라고 추정한 것입니다.

### 실습 3. `EXPLAIN ANALYZE`로 실제 시간 보기

```python
profile = duckdb.sql("""
    EXPLAIN ANALYZE
        SELECT category, avg(value) AS avg_value, count(*) AS cnt
        FROM 'output/parquet/perf_test.parquet'
        WHERE value > 500
        GROUP BY category
        ORDER BY avg_value DESC
        LIMIT 5
""").fetchone()[1]

print(profile)
```

```
┌────────────────────────────────────────────────┐
││              Total Time: 0.0092s             ││
└────────────────────────────────────────────────┘
┌───────────────────────────┐
│           TOP_N           │
│           Top: 5          │
│  Order By: avg(value) DESC│
│           5 rows          │
│          (0.00s)          │
└─────────────┬─────────────┘
              │      ... (중간 PROJECTION 연산자 생략) ...
┌─────────────┴─────────────┐
│   PERFECT_HASH_GROUP_BY   │
│         Groups: #0        │
│   Aggregates: avg(#1),    │
│        count_star()       │
│          100 rows         │
│          (0.01s)          │
└─────────────┬─────────────┘
              │
┌─────────────┴─────────────┐
│         TABLE_SCAN        │
│         Function:         │
│        PARQUET_SCAN       │
│  Projections: value,      │
│               category    │
│     Filters: value>500    │
│    Total Files Read: 1    │
│       2,495,000 rows      │
│          (0.08s)          │
└───────────────────────────┘
```

여기서 눈여겨볼 것:

- **`TABLE_SCAN`에 시간이 가장 많이 찍힙니다 (0.08s).** 집계(`0.01s`)나 정렬(`0.00s`)보다 훨씬 큽니다. OLAP 쿼리는 대부분 **계산이 아니라 데이터를 읽어오는 데** 시간을 씁니다. Kersten 논문이 “대부분의 OLAP 쿼리는 memory-bound라서 SIMD 이득이 작다”고 한 것이 정확히 이 현상입니다.
- `Filters: value>500`이 스캔에 들어갔고, 실제로 **2,495,000행**이 통과했습니다 (500만 행의 약 절반).
- `Total Files Read: 1` — 여러 파일을 읽었다면 여기 개수가 표시됩니다.
- **여기서 짚고 넘어갈 부분 (문서 설명이 실제로 확인되는 지점)**: `Total Time`이 **0.0092초**인데 `TABLE_SCAN` 하나가 **0.08초**입니다. 부분이 전체보다 큰 게 이상해 보이지만, 이게 바로 문서가 경고한 **“각 연산자 시간은 누적 wall-clock이라, 병렬 처리 시 전체 시간이 개별 연산자 시간의 합보다 작을 수 있다”** 는 상황입니다. 32개 스레드가 스캔을 나눠 처리했기 때문에, **스레드별 시간을 다 더하면 0.08초지만 실제로 흐른 시간은 0.0092초**인 것입니다. 연산자 시간을 **절대 시간이 아니라 “어디에 일이 몰렸는지”를 보는 비율 지표**로 읽어야 하는 이유입니다.

### 실습 4. zonemap의 실체 보기 (이 노트의 하이라이트)

`duckdb02.md`에서 “row group마다 min/max 통계(zonemap)가 있어서 조건에 안 맞는 row group을 통째로 건너뛴다”고 배웠습니다. 그 통계를 **직접 눈으로** 봅니다.

```python
duckdb.sql("""
    SELECT count(DISTINCT row_group_id) AS row_groups
    FROM parquet_metadata('output/parquet/perf_test.parquet')
""").show()
```

```
┌────────────┐
│ row_groups │
├────────────┤
│         50 │
└────────────┘
```

```python
# row group별 min/max 통계 = zonemap 의 실체
duckdb.sql("""
    SELECT row_group_id, path_in_schema AS col_name, stats_min, stats_max
    FROM parquet_metadata('output/parquet/perf_test.parquet')
    WHERE path_in_schema IN ('id', 'value')
      AND row_group_id < 3
    ORDER BY row_group_id, col_name
""").show()
```

```
┌──────────────┬──────────┬───────────┬───────────┐
│ row_group_id │ col_name │ stats_min │ stats_max │
├──────────────┼──────────┼───────────┼───────────┤
│            0 │ id       │ 1         │ 100352    │
│            0 │ value    │ 0         │ 999       │
│            1 │ id       │ 100353    │ 200704    │
│            1 │ value    │ 0         │ 999       │
│            2 │ id       │ 200705    │ 301056    │
│            2 │ value    │ 0         │ 999       │
└──────────────┴──────────┴───────────┴───────────┘
```

**이 표가 핵심입니다.**

| 컬럼 | row group 0 | row group 1 | row group 2 | 프루닝 가능? |
| --- | --- | --- | --- | --- |
| `id` | 1 ~ 100352 | 100353 ~ 200704 | 200705 ~ 301056 | **가능** — 범위가 서로 겹치지 않음 |
| `value` | 0 ~ 999 | 0 ~ 999 | 0 ~ 999 | **불가능** — 모든 row group이 같은 범위 |

```python
# 두 필터의 결과 행 수는 비슷하지만, 읽어야 하는 row group 수는 완전히 다름
duckdb.sql("""
    SELECT
        (SELECT count(*) FROM 'output/parquet/perf_test.parquet' WHERE id < 100000)  AS id_filter_rows,
        (SELECT count(*) FROM 'output/parquet/perf_test.parquet' WHERE value < 20)   AS value_filter_rows
""").show()
```

```
┌────────────────┬───────────────────┐
│ id_filter_rows │ value_filter_rows │
├────────────────┼───────────────────┤
│          99999 │            100000 │
└────────────────┴───────────────────┘
```

결과 행 수는 **99,999 vs 100,000으로 거의 같습니다.** 하지만:

- `WHERE id < 100000` → 통계만 보고 **row group 0 하나만 읽고 나머지 49개는 건너뜁니다.**
- `WHERE value < 20` → 어느 row group에도 값이 있을 수 있으므로 **50개를 전부 읽어야 합니다.**
- **여기서 짚고 넘어갈 부분**: “Parquet는 통계 기반 프루닝을 지원한다”는 문장만 보면 항상 이득인 것 같지만, 실제로는 **데이터가 그 컬럼 기준으로 정렬(또는 상관)되어 저장돼 있어야만** 효과가 있습니다. 똑같이 10만 행을 뽑는 두 필터가 읽는 데이터 양이 50배 차이 나는 이유가 여기 있습니다. 그래서 실무에서 큰 Parquet를 만들 때 **자주 필터링하는 컬럼(주로 날짜) 기준으로 정렬해서 쓰거나 파티셔닝**하는 것이 중요합니다. GA4 데이터에서 `event_date`로 파티션을 나누는 관행이 정확히 이 이유에서 나옵니다.

### 실습 5. 프로젝션 pushdown 효과 실측

```python
import duckdb, time

def measure(sql, label, runs=3):
    """캐시 영향을 줄이려고 여러 번 실행한 뒤 최소 시간을 사용"""
    times = []
    for _ in range(runs):
        t0 = time.time()
        duckdb.sql(sql).fetchall()
        times.append(time.time() - t0)
    print(f"{label}:{round(min(times), 4)}초")

measure("SELECT sum(id) FROM 'output/parquet/perf_test.parquet'",
        "컬럼 1개(id)만 필요")

measure("SELECT sum(id), sum(category), sum(value), max(name) FROM 'output/parquet/perf_test.parquet'",
        "컬럼 4개 전부 필요")
```

```
컬럼 1개(id)만 필요: 0.008초
컬럼 4개 전부 필요: 0.0136초
```

컬럼 4개를 다 읽는 쿼리가 **약 1.7배** 느립니다. 특히 `name`은 문자열이라 읽는 비용이 큽니다.

> 실무 교훈: `SELECT *`를 습관적으로 쓰지 마세요. 컬럼 지향 저장에서 `SELECT *`는 **컬럼 지향의 이점을 스스로 포기하는 것**과 같습니다. 컬럼 수가 수십 개인 GA4 이벤트 테이블에서는 이 배수가 훨씬 커집니다.
> 

### 실습 6. 멀티스레드 확인

```python
duckdb.sql("SELECT current_setting('threads') AS threads").show()
```

```
┌─────────┐
│ threads │
├─────────┤
│      32 │
└─────────┘
```

```python
heavy = """
    SELECT category, avg(value), count(*)
    FROM 'output/parquet/perf_test.parquet'
    GROUP BY category
"""

for n in [1, 2, 4]:
    duckdb.sql(f"SET threads ={n}")
    times = []
    for _ in range(3):
        t0 = time.time()
        duckdb.sql(heavy).fetchall()
        times.append(time.time() - t0)
    print(f"threads={n}:{round(min(times), 4)}초")

duckdb.sql("RESET threads")  # 기본값(코어 수)으로 복구
```

```
threads=1: 0.0188초
threads=2: 0.0108초
threads=4: 0.0065초
복구 후 threads: 32
```

1 → 2 → 4 스레드에서 **0.0188 → 0.0108 → 0.0065초**, 거의 선형에 가까운 스케일링입니다 (4스레드에서 약 2.9배).

- **여기서 짚고 넘어갈 부분**: 병렬화 코드를 **한 줄도 쓰지 않았는데** 엔진이 알아서 코어를 나눠 썼습니다. 이게 Kersten 논문에서 다룬 **morsel-driven 병렬화**가 실제로 동작하는 모습입니다. 다만 데이터가 작으면 스레드 생성·조율 오버헤드가 이득을 넘어서 오히려 느려질 수 있는데, 체크리스트 9번(“스레드를 너무 많이 쓰고 있지 않은가”)이 바로 그 경우를 가리킵니다.

---

## 정리

**“왜 이 쿼리가 느린가”를 판단하는 최소 절차:**

1. **`EXPLAIN`으로 계획을 본다** → 필터와 프로젝션이 스캔 단계로 밀려 들어갔는가? (`Filters:`, `Projections:`가 스캔 블록 안에 있는지)
2. **`EXPLAIN ANALYZE`로 실행한다** → 어느 연산자가 시간을 다 먹는가? 1번의 추정 행 수(`~N rows`)와 실제 행 수(`N rows`)가 크게 다른가?
3. **스캔이 병목이면** → 읽는 컬럼을 줄였는가(`SELECT *` 지양)? 필터 컬럼 기준으로 데이터가 정렬/파티셔닝돼 있는가(`parquet_metadata`로 zonemap 확인)?
4. **그래도 느리면** → “My Workload Is Slow” 체크리스트(메모리·디스크·타입·row group 크기·스레드 수)

이 4단계가 로드맵 6단계에서 목표했던 “성능 감각”의 실체입니다. 다음 **7단계(실무 연동)** 에서 BigQuery ↔︎ DuckDB를 오갈 때, “BigQuery에서 스캔 비용이 큰 쿼리를 로컬 DuckDB로 먼저 다듬는” 작업이 바로 여기서 익힌 도구 위에서 이뤄집니다.
