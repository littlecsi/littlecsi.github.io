---
layout: post
title: '[DuckDB] 02 Parquet and CSV Files'
subtitle: 'Parquet and CSV Files'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB 02 - Parquet and CSV Files](#duckdb-02---parquet-and-csv-files)
- [Parquet 파일 읽기와 쓰기 — 강의 노트](#parquet-파일-읽기와-쓰기--강의-노트)
  - [Parquet란?](#parquet란)
    - [Apache Parquet란 무엇인가](#apache-parquet란-무엇인가)
    - [Apache Parquet의 동작 방식](#apache-parquet의-동작-방식)
    - [Apache Parquet의 이점](#apache-parquet의-이점)
    - [(요약) 주요 활용 사례](#요약-주요-활용-사례)
    - [(요약) 다른 파일 포맷과의 비교](#요약-다른-파일-포맷과의-비교)
    - [(요약) 지원 생태계](#요약-지원-생태계)
  - [예제 모음 (Examples) — 빠른 참고용](#예제-모음-examples--빠른-참고용)
  - [`read_parquet` 함수](#read_parquet-함수)
  - [파라미터 (Parameters)](#파라미터-parameters)
    - [`schema` 파라미터 — 왜 컬럼 이름이 아니라 Field ID인가](#schema-파라미터--왜-컬럼-이름이-아니라-field-id인가)
  - [일부만 읽기 (Partial Reading) — 위에서 배운 구조가 실제로 쓰이는 곳](#일부만-읽기-partial-reading--위에서-배운-구조가-실제로-쓰이는-곳)
  - [INSERT와 View — 언제 무엇을 쓰나](#insert와-view--언제-무엇을-쓰나)
  - [Parquet 파일에 쓰기 (Writing to Parquet Files)](#parquet-파일에-쓰기-writing-to-parquet-files)
  - [암호화 (Encryption)](#암호화-encryption)
  - [지원되는 기능 (Supported Features)](#지원되는-기능-supported-features)
  - [Parquet 확장 설치하기](#parquet-확장-설치하기)
  - [정리](#정리)
- [CSV 파일 가져오기 — 강의 노트](#csv-파일-가져오기--강의-노트)
  - [예제 모음 (Examples)](#예제-모음-examples)
  - [CSV 로딩(CSV Loading)](#csv-로딩csv-loading)
  - [파라미터 (Parameters)](#파라미터-parameters-1)
    - [`auto_type_candidates` 상세](#auto_type_candidates-상세)
  - [CSV 함수 (CSV Functions)](#csv-함수-csv-functions)
  - [COPY 문으로 쓰기 (Writing Using the COPY Statement)](#copy-문으로-쓰기-writing-using-the-copy-statement)
  - [잘못된 CSV 파일 읽기 (Reading Faulty CSV Files)](#잘못된-csv-파일-읽기-reading-faulty-csv-files)
  - [삽입 순서 보존 (Order Preservation)](#삽입-순서-보존-order-preservation)
  - [CSV 파일 쓰기 (Writing CSV Files)](#csv-파일-쓰기-writing-csv-files)
  - [정리](#정리-1)


# DuckDB 02 - Parquet and CSV Files

Created: September 3, 2026 1:49 PM
Class: DuckDB
Jupyter Notebook: duckdb02.ipynb

# Parquet 파일 읽기와 쓰기 — 강의 노트

> 원본: [duckdb.org/docs/current/data/parquet/overview](https://duckdb.org/docs/current/data/parquet/overview)
참고 자료: [IBM Think - What is Apache Parquet?](https://www.ibm.com/think/topics/parquet)
실습 파일: `duckdb02.ipynb`
> 

---

## Parquet란?

### Apache Parquet란 무엇인가

Apache Parquet는 대용량 데이터셋을 효율적으로 저장·관리·분석하기 위한 **오픈소스 컬럼 지향(columnar) 저장 포맷**입니다. CSV나 JSON 같은 행 지향(row-based) 포맷과 달리, 데이터를 컬럼 단위로 정리해 쿼리 성능을 높이고 저장 비용을 줄입니다.

예를 들어 고객 거래 데이터를 분석할 때, Parquet를 쓰는 소매업체 데이터베이스는 전체 고객 레코드를 로드하지 않고도 구매 날짜·금액 같은 특정 컬럼만 바로 읽을 수 있습니다.

Parquet 포맷은 다음 세 영역에서 특히 가치가 있습니다:

- 수십억 건의 레코드를 대상으로 복잡한 쿼리를 처리하는 **분석 워크로드**
- 효율적인 저장과 빠른 데이터 조회가 필요한 **데이터 레이크·데이터 웨어하우스**
- 대규모 학습 데이터셋에서 특정 속성만 분석하는 **머신러닝(ML) 파이프라인**

Apache Spark, Apache Hive, Apache Hadoop 같은 분산 시스템·데이터 도구와의 호환성도 Parquet가 널리 채택된 이유입니다.

**핵심 특징 세 가지**

- **컬럼 지향 저장 포맷**: 관련 있는 컬럼 값만 읽을 수 있어, 쿼리 시간을 몇 시간에서 몇 분으로 줄이면서 저장 비용도 낮춥니다.
- **스키마와 메타데이터 관리**: 모든 Parquet 파일에는 자체 기술 스키마(self-describing schema)가 포함되어, 비즈니스 요구가 바뀔 때 데이터 모델이 진화할 수 있습니다 (예: 기존 레코드 재구축 없이 새 컬럼 추가).
- **효율적인 압축**: 중복 정보를 제거하고 공간 효율적인 형태로 저장해, 다양한 워크로드에서 최적의 저장 효율과 연산 성능을 보장합니다.

### Apache Parquet의 동작 방식

원본 데이터를 최적화된 컬럼 포맷으로 변환하는 5단계입니다.

1. **데이터 구성(Data organization)**: 데이터를 **row group**(행 묶음)으로 나눕니다. 각 row group은 독립적인 단위라 병렬 처리와 효율적인 메모리 관리가 가능합니다.
2. **컬럼 청킹(Column chunking)**: 각 row group 안에서 데이터를 컬럼 기준으로 재정리해 **column chunk**로 묶고, 데이터 특성에 맞는 전용 인코딩을 적용합니다.
3. **압축과 인코딩**: 먼저 run-length encoding(RLE) 같은 기법으로 반복값을 효율적으로 표현하고, 그다음 Snappy/Gzip 등 압축 알고리즘을 적용합니다.
4. **메타데이터 생성**: 파일 스키마, 데이터 타입, 컬럼별 통계, row group 위치 등을 담은 메타데이터를 생성합니다.
5. **쿼리 실행**: 쿼리 엔진이 메타데이터를 먼저 확인해 필요한 column chunk만 읽고, 필요 시 압축을 해제합니다.

> 이 5단계는 “행 위치(offset) 정렬로 행을 재구성한다”는 컬럼 지향 저장의 원리를 실제 파일 처리 과정으로 구체화한 것입니다.
> 

### Apache Parquet의 이점

- **쿼리 성능**: 관련 컬럼만 접근해 쿼리 시간이 시간 단위에서 분 단위로 줄어듭니다.
- **복잡한 데이터 처리**: 중첩 데이터 구조·배열을 효율적으로 다룹니다 (웹 분석의 JSON형 구조, IoT 센서의 중첩 배열 등).
- **저장 효율성**: 데이터 타입별로 다른 인코딩을 적용해 CSV/JSON보다 나은 압축률을 냅니다.
- **프레임워크 통합**: pandas, Java, Spark 등에서 일관된 데이터 접근을 보장합니다.
- **Hadoop 생태계 최적화**: HDFS를 위해 만들어져 Hadoop 환경에서 특히 좋은 성능을 냅니다.

### (요약) 주요 활용 사례

- **데이터 레이크·웨어하우스**: 대용량 데이터를 저장하면서도 BI 툴·SQL 쿼리에 빠르게 접근해야 하는 조직이 주 저장 포맷으로 채택 (예: 소매 체인의 매장별 매출 분석)
- **분석 워크로드**: Spark나 pandas를 쓰는 데이터 과학자·분석가에게 적합 (예: 금융사의 실시간에 가까운 리스크 지표 계산)
- **ETL 파이프라인**: 스키마 진화를 지원해, 데이터 구조가 바뀌는 상황에서도 안정적으로 중간/목적 포맷으로 활용 (예: 여러 시스템의 환자 기록을 통합하는 헬스케어 조직)

### (요약) 다른 파일 포맷과의 비교

- **vs. CSV/JSON**: 행 지향은 컬럼 하나만 봐도 전체 행을 스캔해야 하지만, Parquet는 필요한 컬럼 청크만 읽습니다.
- **vs. Avro**: Avro는 행 지향이라 직렬화·스트리밍, 이벤트/트랜잭션 단위 기록에 강하고, Parquet는 컬럼 단위 분석 워크로드에 최적화되어 있습니다.
- 참고로 앞서 다룬 **“행 지향 RDBMS와의 차이”**(Parquet는 엔진이 아니라 파일 포맷이라는 점)는 여기서 다룬 CSV/JSON/Avro와의 비교와는 다른 축의 비교입니다.

### (요약) 지원 생태계

- **처리 프레임워크**: Spark(고성능 분석), Hadoop(분산 처리), Arrow(시스템 간 빠른 데이터 공유)
- **언어·인터페이스**: Python은 pandas, Java는 네이티브 라이브러리
- **클라우드**: AWS/GCP/Azure/IBM Cloud의 네이티브 지원과 Athena/BigQuery/Db2 Warehouse 같은 쿼리 엔진 호환 — DuckDB도 이 생태계의 일원입니다.

---

## 예제 모음 (Examples) — 빠른 참고용

문서 최상단의 예제 모음입니다. `test.parquet`, `file1.parquet`처럼 존재하지 않는 파일을 참조하는 것이 대부분이라 문법 참고용으로만 남겨둡니다.

```sql
-- 단일 Parquet 파일 읽기
SELECT * FROM 'test.parquet';

-- 컬럼/타입 확인
DESCRIBE SELECT * FROM 'test.parquet';

-- Parquet 파일로부터 테이블 생성
CREATE TABLE test AS SELECT * FROM 'test.parquet';

-- 확장자가 .parquet가 아니면 read_parquet 함수 사용
SELECT * FROM read_parquet('test.parq');

-- 여러 파일을 하나의 테이블처럼 읽기
SELECT * FROM read_parquet(['file1.parquet', 'file2.parquet', 'file3.parquet']);

-- glob 패턴으로 읽기
SELECT * FROM 'test/*.parquet';

-- filename 가상 컬럼 포함 (v1.3.0+ 기본 제공)
SELECT *, filename FROM read_parquet('test/*.parquet');

-- 두 폴더의 파일을 glob 목록으로 읽기
SELECT * FROM read_parquet(['folder1/*.parquet', 'folder2/*.parquet']);

-- HTTPS로 읽기
SELECT * FROM read_parquet('https://some.url/some_file.parquet');

-- 메타데이터/파일 메타데이터/key-value 메타데이터/스키마 조회
SELECT * FROM parquet_metadata('test.parquet');
SELECT * FROM parquet_file_metadata('test.parquet');
SELECT * FROM parquet_kv_metadata('test.parquet');
SELECT * FROM parquet_schema('test.parquet');

-- 기본 압축(Snappy)으로 쓰기
COPY (SELECT * FROM tbl) TO 'result-snappy.parquet' (FORMAT parquet);

-- 압축 방식 + row group 크기 지정해서 쓰기
COPY (FROM generate_series(100_000)) TO 'test.parquet'
    (FORMAT parquet, COMPRESSION zstd, ROW_GROUP_SIZE 100_000);

-- 데이터베이스 전체를 Parquet로 export
EXPORT DATABASE 'target_directory' (FORMAT parquet);
```

---

## `read_parquet` 함수

| 함수 | 설명 |
| --- | --- |
| `read_parquet(path_or_list_of_paths)` | Parquet 파일(들)을 읽음 |
| `parquet_scan(path_or_list_of_paths)` | `read_parquet`의 별칭(alias) |

```python
import duckdb

duckdb.sql("SELECT * FROM 'data/parquet/example.parquet'").show()
```

```
┌───────┬─────────┬───────┐
│  id   │  name   │ price │
│ int32 │ varchar │ int32 │
├───────┼─────────┼───────┤
│     1 │ apple   │  1200 │
│     2 │ banana  │   800 │
│     3 │ cherry  │  5000 │
└───────┴─────────┴───────┘
```

- `read_parquet('file.parquet')`와 `'file.parquet'`를 `FROM`에 직접 쓰는 것 모두 같은 결과입니다. `duckdb00.md`에서 CSV/JSON에 대해 짚었던 것과 같은 패턴 — 확장자가 `.parquet`면 함수 호출 없이도 DuckDB가 자동 인식합니다.
- 여러 파일은 glob이나 파일 목록으로 한 번에 읽을 수 있습니다 (자세한 내용은 Multiple Files 문서 참고).

---

## 파라미터 (Parameters)

| 이름 | 설명 | 타입 | 기본값 |
| --- | --- | --- | --- |
| `binary_as_string` | 레거시 writer가 UTF8 플래그를 잘못 설정해 문자열이 BLOB으로 로드되는 경우, `true`로 문자열로 강제 로드 | BOOL | false |
| `can_have_nan` | `FLOAT`/`DOUBLE`에 NaN이 있을 수 있는지 (필터 pushdown 시 min/max 통계 판단에 영향) | BOOL | false |
| `encryption_config` | Parquet 암호화 설정 | STRUCT | - |
| `filename` | `filename` 컬럼 포함 여부 (v1.3.0+부터는 자동 추가되어 호환용으로만 남음) | BOOL | false |
| `file_row_number` | `file_row_number` 컬럼 포함 여부 | BOOL | false |
| `hive_partitioning` | 경로를 Hive 파티션 경로로 해석할지 | BOOL | (자동 감지) |
| `union_by_name` | 여러 스키마의 컬럼을 이름 기준으로 통합할지 | BOOL | false |
| `schema` | 지정한 스키마로 읽기. Field ID 필요 | MAP | NULL |

### `schema` 파라미터 — 왜 컬럼 이름이 아니라 Field ID인가

```python
duckdb.sql("COPY (SELECT 42::INTEGER AS i) TO 'output/parquet/integers.parquet' (FIELD_IDS{i: 0})")

duckdb.sql("""
    SELECT *
    FROM read_parquet('output/parquet/integers.parquet', schema = MAP {
                        0: {name: 'renamed_i', type: 'BIGINT', default_value: NULL},
                        1: {name: 'new_column', type: 'UTINYINT', default_value: 43}
                      })
""").show()
```

```
┌───────────┬────────────┐
│ renamed_i │ new_column │
│   int64   │   uint8    │
├───────────┼────────────┤
│        42 │         43 │
└───────────┴────────────┘
```

- **여기서 짚고 넘어갈 부분**: 문서는 “field ID가 필요하다”고만 말하고 이유는 설명하지 않습니다. 컬럼 **이름**은 나중에 얼마든지 바뀔 수 있지만, 컬럼에 한 번 부여된 **field ID는 파일이 존재하는 한 고정**됩니다. 그래서 스키마가 시간이 지나며 진화하는 상황(컬럼명 변경, 추가)에서도 “이 데이터가 원래 어떤 컬럼이었는지”를 안정적으로 추적할 수 있습니다. (Iceberg/Avro 같은 다른 데이터 포맷들도 같은 이유로 필드 ID 기반 스키마 진화를 씁니다.) 즉 이 예제는 단순 문법 데모가 아니라, 위에서 다룬 “스키마 진화”를 Parquet가 실제로 어떻게 구현하는지 보여주는 대목입니다.
- `union_by_name=true`와 함께 쓸 수 없는 이유도 같은 맥락입니다 — 하나는 ID로, 하나는 이름으로 컬럼을 맞추는 서로 다른 전략이라 동시에 쓸 수 없습니다.

---

## 일부만 읽기 (Partial Reading) — 위에서 배운 구조가 실제로 쓰이는 곳

DuckDB는 Parquet 스캔에 **프로젝션 pushdown**(필요한 컬럼만 읽기)과 **필터 pushdown**(조건에 안 맞는 부분을 zonemap으로 건너뛰기)을 모두 지원합니다. 이는 DuckDB가 자동으로 처리하며, 상당한 성능 이점을 제공합니다.

- **여기서 짚고 넘어갈 부분**: 이 두 최적화는 추상적인 기능이 아니라, 앞서 “Parquet 동작 방식”에서 설명한 **컬럼 청킹**과 **메타데이터(row group별 min/max 통계)**가 있어야 가능한 것들입니다. 프로젝션 pushdown은 컬럼이 애초에 분리 저장되어 있기 때문에 가능하고, 필터 pushdown은 row group 메타데이터의 zonemap을 미리 확인해 아예 읽지 않아도 되는 row group을 걸러내는 방식입니다. 다만 zonemap이 없는 파일에는 필터 pushdown이 적용되지 않을 수 있습니다.

---

## INSERT와 View — 언제 무엇을 쓰나

```sql
-- people 테이블이 이미 존재한다고 가정 (원문 예제, 실행하지 않음)
INSERT INTO people
    SELECT * FROM read_parquet('test.parquet');
```

```python
duckdb.sql("""
    CREATE TABLE people AS
    SELECT * FROM read_parquet('data/parquet/example.parquet')
""")
duckdb.sql("SELECT * FROM people").show()
```

```
┌───────┬─────────┬───────┐
│  id   │  name   │ price │
│ int32 │ varchar │ int32 │
├───────┼─────────┼───────┤
│     1 │ apple   │  1200 │
│     2 │ banana  │   800 │
│     3 │ cherry  │  5000 │
└───────┴─────────┴───────┘
```

```python
duckdb.sql("DROP TABLE IF EXISTS people")
duckdb.sql("""
    CREATE VIEW people AS
    SELECT * FROM read_parquet('data/parquet/example.parquet')
""")
duckdb.sql("SELECT * FROM people").show()
```

```
┌───────┬─────────┬───────┐
│  id   │  name   │ price │
│ int32 │ varchar │ int32 │
├───────┼─────────┼───────┤
│     1 │ apple   │  1200 │
│     2 │ banana  │   800 │
│     3 │ cherry  │  5000 │
└───────┴─────────┴───────┘
```

- **여기서 짚고 넘어갈 부분**: 두 결과가 똑같아 보이지만 성격이 다릅니다. `CREATE TABLE ... AS`는 Parquet 파일의 데이터를 **DuckDB 내부 저장소로 복사**합니다 — 이후 Parquet 파일이 바뀌어도 테이블은 그대로입니다. `CREATE VIEW ... AS`는 복사하지 않고 **쿼리할 때마다 Parquet 파일을 다시 읽습니다** — 원본 파일이 갱신되면 뷰의 결과도 즉시 반영되지만, 매번 파일을 다시 스캔하는 비용이 붙습니다. “원본 파일을 최신 상태로 유지하며 매번 최신 데이터를 보고 싶다”면 View, “지금 시점의 데이터를 고정해두고 빠르게 반복 조회하고 싶다”면 Table AS가 맞는 선택입니다.

---

## Parquet 파일에 쓰기 (Writing to Parquet Files)

```sql
-- tbl이 정의돼 있지 않아 아래 세 예제는 실행하지 않음
COPY (SELECT * FROM tbl) TO 'result-snappy.parquet' (FORMAT parquet);
COPY tbl TO 'result-zstd.parquet' (FORMAT parquet, COMPRESSION zstd);
COPY tbl TO 'result-zstd.parquet' (FORMAT parquet, COMPRESSION zstd, COMPRESSION_LEVEL 1);
```

```python
duckdb.sql("""
    COPY (
        SELECT 42 AS number, true AS is_even
    ) TO 'output/parquet/kv_metadata.parquet' (
        FORMAT parquet,
        KV_METADATA {
            number: 'Answer to life, universe, and everything',
            is_even: 'not ''odd'''
        }
    )
""")
```

```sql
-- tbl이 정의돼 있지 않아 실행하지 않음
COPY tbl TO 'result-v2.parquet' (FORMAT parquet, PARQUET_VERSION 'V2');
```

```python
duckdb.sql("""
    COPY
        'data/csv/example.csv'
        TO 'output/parquet/result-uncompressed.parquet'
        (FORMAT parquet, COMPRESSION uncompressed)
""")

duckdb.sql("""
    COPY
        (FROM generate_series(100_000))
        TO 'output/parquet/row-groups-zstd.parquet'
        (FORMAT parquet, COMPRESSION zstd, ROW_GROUP_SIZE 100_000)
""")

duckdb.sql("""
    COPY
        (FROM generate_series(100_000))
        TO 'output/parquet/result-lz4.parquet'
        (FORMAT parquet, COMPRESSION lz4)
""")

duckdb.sql("""
    COPY
        (FROM generate_series(100_000))
        TO 'output/parquet/result-brotli.parquet'
        (FORMAT parquet, COMPRESSION brotli)
""")
```

```sql
-- lz4_raw는 위 lz4 결과와 동일하므로 별도 실행하지 않음
COPY (FROM generate_series(100_000)) TO 'result-lz4.parquet' (FORMAT parquet, COMPRESSION lz4_raw);

-- lineitem 테이블이 정의돼 있지 않아 실행하지 않음
COPY lineitem TO 'lineitem-with-custom-dictionary-size.parquet'
    (FORMAT parquet, STRING_DICTIONARY_PAGE_SIZE_LIMIT 100_000);

-- 현재 DB 상태에 따라 결과가 달라지고 여러 파일을 생성하는 부수효과가 있어 실행하지 않음
EXPORT DATABASE 'target_directory' (FORMAT parquet);
```

- **여기서 짚고 넘어갈 부분 — 압축 알고리즘 선택**: 문서는 각 압축 알고리즘을 쓰는 예제만 나열할 뿐 언제 무엇을 쓰라는 안내가 없습니다. 대략적인 기준은 **압축 속도 · 압축률 · 해제 속도 사이의 트레이드오프**입니다 — Snappy는 압축/해제가 빨라 기본값으로 무난하고, ZSTD는 Snappy보다 느리지만 압축률이 훨씬 좋아 저장 비용이 중요할 때 유리하며, Brotli는 압축률은 가장 좋지만 압축 자체가 느려 “자주 쓰지 않는 아카이브용 데이터”에 적합합니다.
- **`ROW_GROUP_SIZE`도 트레이드오프가 있습니다**: row group을 작게 쪼갤수록 필터 pushdown이 더 세밀하게(불필요한 부분을 더 잘 건너뜀) 동작하지만, row group마다 메타데이터 오버헤드가 붙어 파일이 비대해집니다.

---

## 암호화 (Encryption)

DuckDB는 암호화된 Parquet 파일을 읽고 쓰는 것을 지원합니다.

## 지원되는 기능 (Supported Features)

지원되는 Parquet 기능 목록은 Parquet 문서의 “Implementation status” 페이지에서 확인할 수 있습니다.

## Parquet 확장 설치하기

```python
duckdb.sql("INSTALL parquet")
```

- Python 클라이언트에서는 `parquet` 확장이 기본으로 번들되어 자동 로드되므로, 이 명령은 대부분 이미 설치돼 있다는 메시지만 내고 아무 일도 하지 않습니다. CLI나 확장이 번들되지 않은 환경에서 필요한 명령이라고 이해하시면 됩니다.

---

## 정리

이 문서에서 실제로 새로 배운 것은 “Parquet가 컬럼 지향 포맷이다”라는 사실 자체가 아니라, 그 사실이 **`schema`의 field ID 기반 진화**, **프로젝션/필터 pushdown**, **압축·row group 크기의 트레이드오프**, **Table AS vs View**라는 네 가지 실무적 선택으로 구체화된다는 점입니다. 다음 학습 단계에서 `UNNEST`/`STRUCT` 같은 중첩 데이터 문법을 배울 때, 그 데이터가 실제로는 이런 Parquet의 컬럼·row group 구조 위에 얹혀 있다는 걸 염두에 두시면 좋습니다.

---

# CSV 파일 가져오기 — 강의 노트

> 원본: [duckdb.org/docs/current/data/csv/overview](https://duckdb.org/docs/current/data/csv/overview)
> 
> 
> 이 노트북에는 문서와 정확히 같은 스키마의 `data/csv/flights.csv`(파이프(`|`)로 구분된 항공편 데이터, 헤더 포함)가 실제로 있어서, 아래 예제 대부분을 그대로 실행할 수 있습니다.
> 

```
FlightDate|UniqueCarrier|OriginCityName|DestCityName
1988-01-01|AA|New York, NY|Los Angeles, CA
1988-01-02|AA|New York, NY|Los Angeles, CA
1988-01-03|AA|New York, NY|Los Angeles, CA
```

---

## 예제 모음 (Examples)

```python
import duckdb

# 옵션을 자동으로 추론해서 CSV 파일 읽기
duckdb.sql("SELECT * FROM 'data/csv/flights.csv'").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

```python
import duckdb

# read_csv 함수로 옵션을 직접 지정해서 읽기
duckdb.sql("""
    SELECT *
    FROM read_csv('data/csv/flights.csv',
        delim = '|',
        header = true,
        columns = {
            'FlightDate': 'DATE',
            'UniqueCarrier': 'VARCHAR',
            'OriginCityName': 'VARCHAR',
            'DestCityName': 'VARCHAR'
        })
""").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

표준 입력(stdin)으로 CSV를 흘려보내며 읽는 예제도 있습니다. 이건 터미널(CLI)에서 실행하는 명령이라 Jupyter 노트북에서는 실행하지 않습니다:

```bash
cat data/csv/flights.csv | duckdb -c "SELECT * FROM read_csv('/dev/stdin')"
```

```python
import duckdb

# CSV 파일을 스키마를 지정한 테이블에 적재하기
duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime (
        FlightDate DATE,
        UniqueCarrier VARCHAR,
        OriginCityName VARCHAR,
        DestCityName VARCHAR
    )
""")
duckdb.sql("COPY ontime FROM 'data/csv/flights.csv'")
duckdb.sql("SELECT * FROM ontime").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

```python
import duckdb

# 스키마를 직접 지정하지 않고 CREATE TABLE ... AS SELECT로 생성
duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime AS
    SELECT * FROM 'data/csv/flights.csv'
""")
duckdb.sql("SELECT * FROM ontime").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

```python
import duckdb

# SELECT *를 생략하는 FROM-first 문법도 동일하게 동작합니다
duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime AS
    FROM 'data/csv/flights.csv'
""")
duckdb.sql("SELECT * FROM ontime").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

- **여기서 짚고 넘어갈 부분**: 이 섹션에 등장하는 세 가지 진입점 — `'file.csv'` 직접 조회, `read_csv()` 함수, `COPY ... FROM`은 서로 다른 용도입니다. `SELECT * FROM 'file.csv'`/`read_csv()`는 **탐색적으로 빠르게 훑어볼 때**, `COPY`는 **스키마를 이미 알고 있고 대량의 데이터를 정해진 테이블에 밀어 넣을 때** 적합합니다. `duckdb01.md`에서 다뤘던 “`COPY`가 `INSERT`보다 대량 적재에 빠른 이유”와 같은 맥락으로, `COPY`는 스키마 검증·타입 캐스팅 오버헤드를 최소화하도록 만들어졌기 때문입니다.

---

## CSV 로딩(CSV Loading)

CSV 로딩, 즉 CSV 파일을 데이터베이스로 가져오는 작업은 매우 흔하면서도 의외로 까다로운 작업입니다. CSV는 겉보기엔 단순해 보이지만, 실제로는 로딩을 어렵게 만드는 여러 비일관성이 파일 안에 존재합니다. CSV 파일은 다양한 변형이 존재하고, 종종 손상돼 있으며, 스키마가 없습니다. CSV 리더는 이런 모든 상황에 대응해야 합니다.

DuckDB의 CSV 리더는 CSV sniffer를 이용해 파일을 분석함으로써 어떤 설정 플래그를 쓸지 자동으로 추론할 수 있습니다. 이는 대부분의 상황에서 잘 동작하므로 가장 먼저 시도해볼 방법입니다. CSV 리더가 올바른 설정을 알아내지 못하는 드문 상황에서는 CSV 리더를 수동으로 설정해 파일을 올바르게 파싱할 수 있습니다 (자세한 내용은 Auto Detection 문서 참고).

- **여기서 짚고 넘어갈 부분**: 위 예제에서 구분자를 명시하지 않은 첫 `SELECT * FROM 'data/csv/flights.csv'`도 실제로는 잘 동작합니다 — 파일이 쉼표가 아니라 파이프(`|`)로 구분돼 있는데도 sniffer가 이를 자동으로 알아낸 것입니다. Parquet가 “자체 기술 스키마”로 구조를 알려주는 것과 달리, CSV는 스키마가 전혀 없는 텍스트 파일이라 이런 sniffer가 그 공백을 메우는 역할을 합니다 — 다만 항상 100% 신뢰할 수는 없으니, 프로덕션 파이프라인에서는 `delim`/`header`/`columns`를 명시하는 두 번째 방식이 더 안전합니다.

---

## 파라미터 (Parameters)

`read_csv` 함수에 전달할 수 있는 파라미터입니다. 의미상 적용 가능한 경우 `COPY` 문에도 이 파라미터들을 쓸 수 있습니다.

| 이름 | 설명 | 타입 | 기본값 |
| --- | --- | --- | --- |
| `all_varchar` | 타입 감지를 건너뛰고 모든 컬럼을 `VARCHAR`로 취급 (`read_csv` 함수 전용) | BOOL | false |
| `allow_quoted_nulls` | 따옴표로 감싼 값을 NULL로 변환하도록 허용 | BOOL | true |
| `auto_detect` | CSV 파라미터 자동 감지 | BOOL | true |
| `auto_type_candidates` | sniffer가 컬럼 타입을 감지할 때 고려하는 타입 목록. `VARCHAR`는 항상 폴백으로 포함됨 | TYPE[] | 기본 타입 목록 |
| `buffer_size` | 파일을 읽을 때 쓰는 버퍼 크기(바이트). 최소 4줄을 담을 수 있어야 하며 성능에 큰 영향을 줌 | BIGINT | `16 * max_line_size` |
| `columns` | 컬럼 이름과 타입을 struct로 지정. 이 옵션을 쓰면 스키마 자동 감지가 꺼짐 | STRUCT | (비어있음) |
| `comment` | 주석을 시작하는 문자. 이 문자로 시작하는 줄은 완전히 무시됨 | VARCHAR | (비어있음) |
| `compression` | CSV 압축 방식. 기본은 확장자로 자동 감지. `none`/`gzip`/`zstd` 중 선택 | VARCHAR | auto |
| `dateformat` / `date_format` | 날짜를 파싱/쓸 때 쓰는 포맷 (`date_format`은 `COPY` 전용 별칭) | VARCHAR | (비어있음) |
| `decimal_separator` | 숫자의 소수점 구분자 | VARCHAR | `.` |
| `delim` / `sep` / `delimiter` | 컬럼 구분자 (최대 4바이트까지 가능) | VARCHAR | `,` |
| `escape` | 따옴표로 감싼 값 안에서 따옴표 문자를 이스케이프하는 문자열 | VARCHAR | `"` |
| `encoding` | CSV 인코딩. `utf-8`/`utf-16`/`latin-1` 중 선택 (`COPY`는 항상 utf-8) | VARCHAR | utf-8 |
| `filename` | 각 행에 파일 경로를 컬럼으로 추가 (v1.3.0+부터는 자동, 호환용으로만 남음) | BOOL | false |
| `files_to_sniff` | 여러 파일을 읽을 때 스키마 감지에 사용할 파일 수. `-1`이면 전부 | BIGINT | 10 |
| `force_not_null` | 지정한 컬럼은 빈 값을 NULL이 아니라 길이 0인 문자열로 읽음 | VARCHAR[] | [] |
| `header` | 첫 줄이 컬럼 이름인지 여부 | BOOL | false |
| `hive_partitioning` | 경로를 Hive 파티션 경로로 해석할지 여부 | BOOL | (자동 감지) |
| `ignore_errors` | 파싱 에러를 무시 | BOOL | false |
| `max_line_size` | 한 줄의 최대 크기(바이트). `COPY`에서는 사용 불가 | BIGINT | 2000000 |
| `names` / `column_names` | 컬럼 이름을 리스트로 지정 | VARCHAR[] | (비어있음) |
| `new_line` | 줄바꿈 문자 | VARCHAR | (비어있음) |
| `normalize_names` | 컬럼 이름 정규화(비영숫자 제거, 예약어는 `_` 접두) | BOOL | false |
| `null_padding` | 컬럼 수가 부족한 줄의 나머지를 NULL로 채움 | BOOL | false |
| `nullstr` / `null` | NULL 값을 나타내는 문자열 | VARCHAR/VARCHAR[] | (비어있음) |
| `parallel` | 병렬 CSV 리더 사용 여부 | BOOL | true |
| `quote` | 값을 감싸는 문자열 | VARCHAR | `"` |
| `rejects_scan` / `rejects_table` / `rejects_limit` | 잘못된 스캔/줄 정보를 저장할 임시 테이블 이름과 상한 | VARCHAR / VARCHAR / BIGINT | reject_scans / reject_errors / 0 |
| `sample_size` | 파라미터 자동 감지에 쓰는 샘플 줄 수 | BIGINT | 20480 |
| `skip` | 각 파일 시작 부분에서 건너뛸 줄 수 | BIGINT | 0 |
| `store_rejects` | 에러 있는 줄을 건너뛰고 rejects 테이블에 저장 | BOOL | false |
| `strict_mode` | `true`면 문제 시 바로 에러, `false`면 구조적으로 잘못된 파일도 읽기 시도 | BOOL | true |
| `thousands` | 숫자의 천 단위 구분자 (단일 문자, `decimal_separator`와 달라야 함) | VARCHAR | (비어있음) |
| `timestampformat` / `timestamp_format` | 타임스탬프 파싱/쓰기 포맷 | VARCHAR | (비어있음) |
| `types` / `dtypes` / `column_types` | 컬럼 타입을 리스트(위치 기준) 또는 struct(이름 기준)로 지정 | VARCHAR[]/STRUCT | (비어있음) |
| `union_by_name` | 여러 파일의 컬럼을 이름 기준으로 정렬 (메모리 사용량 증가) | BOOL | false |

> **팁**: UTF-8(기본), UTF-16, Latin-1 외 인코딩은 encodings 확장을 쓰거나 `iconv` 같은 커맨드라인 도구로 변환하면 됩니다.
> 

### `auto_type_candidates` 상세

```sql
SELECT * FROM read_csv('csv_file.csv', auto_type_candidates = ['BIGINT', 'DATE']);
```

기본값은 `['NULL', 'BOOLEAN', 'BIGINT', 'DOUBLE', 'TIME', 'DATE', 'TIMESTAMP', 'VARCHAR']`입니다.

- **여기서 짚고 넘어갈 부분**: `columns`(타입을 직접 못박기)와 `auto_type_candidates`(sniffer가 고려할 후보만 좁히기)를 헷갈리기 쉽습니다. `columns`는 자동 감지를 아예 끄고 스키마를 통째로 지정하는 것이고, `auto_type_candidates`는 자동 감지는 유지하되 후보 타입 목록만 좁혀서 오탐(예: 우편번호가 정수로 감지되는 것)을 줄이는 용도입니다. 강제냐 힌트냐의 차이입니다.

---

## CSV 함수 (CSV Functions)

`read_csv`는 CSV sniffer를 이용해 CSV 리더의 올바른 설정을 자동으로 알아내려 시도합니다. 컬럼 타입도 자동으로 추론합니다. CSV 파일에 헤더가 있으면 그 헤더에 있는 이름을 컬럼명으로 쓰고, 없으면 `column0`, `column1`, … 식으로 이름 붙입니다.

```python
import duckdb

duckdb.sql("SELECT * FROM read_csv('data/csv/flights.csv')").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

경로는 상대 경로(현재 작업 디렉터리 기준)나 절대 경로 모두 가능합니다. `read_csv`로 영속 테이블을 만들 수도 있습니다:

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime AS
    SELECT * FROM read_csv('data/csv/flights.csv')
""")
duckdb.sql("DESCRIBE ontime").show()
```

```
┌────────────────┬─────────────┬─────────┬─────────┬─────────┬─────────┐
│  column_name   │ column_type │  null   │   key   │ default │  extra  │
│    varchar     │   varchar   │ varchar │ varchar │ varchar │ varchar │
├────────────────┼─────────────┼─────────┼─────────┼─────────┼─────────┤
│ FlightDate     │ DATE        │ YES     │ NULL    │ NULL    │ NULL    │
│ UniqueCarrier  │ VARCHAR     │ YES     │ NULL    │ NULL    │ NULL    │
│ OriginCityName │ VARCHAR     │ YES     │ NULL    │ NULL    │ NULL    │
│ DestCityName   │ VARCHAR     │ YES     │ NULL    │ NULL    │ NULL    │
└────────────────┴─────────────┴─────────┴─────────┴─────────┴─────────┘
```

```python
import duckdb

duckdb.sql("SELECT * FROM read_csv('data/csv/flights.csv', sample_size = 20_000)").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

`delim`(`sep`), `quote`, `escape`, `header`를 명시적으로 지정하면 해당 파라미터의 자동 감지만 선택적으로 건너뛸 수 있습니다:

```python
import duckdb

duckdb.sql("SELECT * FROM read_csv('data/csv/flights.csv', header = true)").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ FlightDate │ UniqueCarrier │ OriginCityName │  DestCityName   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

여러 파일은 glob이나 파일 목록을 지정해서 한 번에 읽을 수 있습니다 (자세한 내용은 Multiple Files 문서 참고).

---

## COPY 문으로 쓰기 (Writing Using the COPY Statement)

`COPY` 문으로 CSV 파일의 데이터를 테이블로 적재할 수 있습니다. PostgreSQL과 동일한 문법입니다. `COPY`로 데이터를 적재하려면 먼저 (CSV 파일의 컬럼 순서와 일치하고, 값에 맞는 타입을 쓰는) 올바른 스키마의 테이블을 만들어야 합니다. `COPY`는 CSV의 설정 옵션을 자동으로 감지합니다.

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime (
        flightdate DATE,
        uniquecarrier VARCHAR,
        origincityname VARCHAR,
        destcityname VARCHAR
    )
""")
duckdb.sql("COPY ontime FROM 'data/csv/flights.csv'")
duckdb.sql("SELECT * FROM ontime").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ flightdate │ uniquecarrier │ origincityname │  destcityname   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

CSV 포맷을 직접 지정하고 싶다면 `COPY`의 설정 옵션을 사용하면 됩니다.

```python
import duckdb

duckdb.sql("DROP TABLE IF EXISTS ontime")
duckdb.sql("""
    CREATE TABLE ontime (flightdate DATE, uniquecarrier VARCHAR, origincityname VARCHAR, destcityname VARCHAR)
""")
duckdb.sql("COPY ontime FROM 'data/csv/flights.csv' (DELIMITER '|', HEADER)")
duckdb.sql("SELECT * FROM ontime").show()
```

```
┌────────────┬───────────────┬────────────────┬─────────────────┐
│ flightdate │ uniquecarrier │ origincityname │  destcityname   │
│    date    │    varchar    │    varchar     │     varchar     │
├────────────┼───────────────┼────────────────┼─────────────────┤
│ 1988-01-01 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-02 │ AA            │ New York, NY   │ Los Angeles, CA │
│ 1988-01-03 │ AA            │ New York, NY   │ Los Angeles, CA │
└────────────┴───────────────┴────────────────┴─────────────────┘
```

---

## 잘못된 CSV 파일 읽기 (Reading Faulty CSV Files)

DuckDB는 형식이 잘못된(erroneous) CSV 파일을 읽는 것도 지원합니다. 자세한 내용은 별도의 “Reading Faulty CSV Files” 문서를 참고하세요.

- 위 Parameters 표의 `ignore_errors`, `store_rejects`, `rejects_table`/`rejects_scan`이 바로 이 기능과 연결됩니다 — 파일 전체를 실패시키는 대신, 문제 있는 줄만 별도 테이블에 모아두고 나머지는 정상 로드하는 방식입니다.

## 삽입 순서 보존 (Order Preservation)

CSV 리더는 `preserve_insertion_order` 설정을 따릅니다. 기본값 `true`에서는 결과 행 순서가 파일에서 읽은 줄 순서와 동일합니다. `false`면 순서 보장이 없습니다.

- **여기서 짚고 넘어갈 부분**: 왜 순서를 포기하는 옵션이 있는지 문서에는 이유가 없습니다. DuckDB의 병렬 CSV 리더는 파일을 여러 조각으로 나눠 스레드별로 동시에 파싱하는데, 원래 순서를 유지하려면 그 조각들을 다시 원래 순서로 재조립해야 합니다. `preserve_insertion_order = false`로 이 재조립 과정을 생략하면, 특히 파일이 크고 순서가 중요하지 않은 배치 처리에서 성능 이득을 볼 수 있습니다.

## CSV 파일 쓰기 (Writing CSV Files)

DuckDB는 `COPY ... TO` 문으로 CSV 파일을 쓸 수 있습니다.

> `duckdb00.md`에서 이미 `.write_csv()`와 `COPY (...) TO '...csv'` 형태로 CSV 쓰기를 다뤘으니, 여기서는 같은 기능이 CSV 읽기 문서에서도 짧게 재확인되는 것으로 이해하시면 됩니다.
> 

---

## 정리

이 문서의 핵심은 “CSV는 스키마가 없는 텍스트 파일이라, DuckDB가 sniffer로 그 공백을 자동으로 메운다”는 것입니다. Parquet가 파일 자체에 스키마·통계를 담아 **정확성**을 보장하는 쪽이라면, CSV는 **추론에 의존**하는 쪽이라 `columns`/`auto_type_candidates`로 그 추론을 통제하거나, `ignore_errors`/`rejects_table`로 추론 실패에 대비하는 옵션들이 왜 필요한지가 이 노트의 핵심 맥락입니다.
