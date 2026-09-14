---
layout: post
title: '[DuckDB] Intro Roadmap and some Reading'
subtitle: 'Roadmap and some Reading'
categories: study
tags: duckdb
comments: true
---

![DuckDB](/assets/img/study/duckdb/duckdb.svg)

- [DuckDB Intro - Roadmap and some Reading](#duckdb-intro---roadmap-and-some-reading)
  - [DuckDB 학습 로드맵](#duckdb-학습-로드맵)
    - [1단계 — 개념 이해 (반나절)](#1단계--개념-이해-반나절)
    - [2단계 — Python 클라이언트 기본 (10~15분, 얇게)](#2단계--python-클라이언트-기본-1015분-얇게)
    - [3단계 — SQL 문법 차이 (2~3일)](#3단계--sql-문법-차이-23일)
    - [4단계 — 파일 직접 쿼리 (2일)](#4단계--파일-직접-쿼리-2일)
    - [5단계 — 중첩 데이터: UNNEST / STRUCT (GA4 대비 최우선 ★)](#5단계--중첩-데이터-unnest--struct-ga4-대비-최우선-)
    - [6단계 — 성능 감각 (1일)](#6단계--성능-감각-1일)
    - [7단계 — 실무 연동](#7단계--실무-연동)
    - [8단계 — 실습](#8단계--실습)
- [Dissertation Reading](#dissertation-reading)
  - [“One Size Fits All”: An Idea Whose Time Has Come and Gone](#one-size-fits-all-an-idea-whose-time-has-come-and-gone)
    - [논문 개요](#논문-개요)
    - [핵심 주장 (Thesis)](#핵심-주장-thesis)
    - [근거 1: 데이터 웨어하우징 (OLTP vs OLAP)](#근거-1-데이터-웨어하우징-oltp-vs-olap)
    - [근거 2: 스트림 처리 (논문의 메인 실험, 3~4절)](#근거-2-스트림-처리-논문의-메인-실험-34절)
    - [근거 3: 그 외 특화 시장 (5절, 짧게 언급)](#근거-3-그-외-특화-시장-5절-짧게-언급)
    - [결론 (7절)](#결론-7절)
    - [왜 지금 이 논문이 의미 있는가](#왜-지금-이-논문이-의미-있는가)
  - [Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask](#everything-you-always-wanted-to-know-about-compiled-and-vectorized-queries-but-were-afraid-to-ask)
    - [논문 개요](#논문-개요-1)
    - [배경: 두 가지 경쟁 패러다임](#배경-두-가지-경쟁-패러다임)
    - [실험 방법론](#실험-방법론)
    - [핵심 발견](#핵심-발견)
    - [원시 성능 외의 실무적 차이 (8절)](#원시-성능-외의-실무적-차이-8절)
    - [결론](#결론)
    - [DuckDB와의 직접적 연결](#duckdb와의-직접적-연결)


# DuckDB Intro - Roadmap and some Reading

Created: September 2, 2026 3:35 PM
Class: DuckDB

## DuckDB 학습 로드맵

> **학습 원칙**: 문서는 순서대로 완독하지 않는다. 아래 단계 순서대로 **필요한 페이지만 골라 읽고, 읽자마자 `.duckdb` 가상환경에서 바로 실행**한다.
> 

---

### 1단계 — 개념 이해 (반나절)

- **왜 배우는가**: In-process OLAP 엔진, "분석가용 SQLite" 포지셔닝
- 문서: `Why DuckDB`

---

### 2단계 — Python 클라이언트 기본 (10~15분, 얇게)

- `duckdb.connect()`, `con.sql()`, `.df()` — 연결/실행/결과 반환 방식만
- 문서: `clients/python/overview` (여기서 SQL 문법까지 배우려 하지 말 것 — 얕게 훑기)

---

### 3단계 — SQL 문법 차이 (2~3일)

표준 SQL과 다른, DuckDB 고유 편의 문법 위주:

| 기능 | 용도 |
| --- | --- |
| `SELECT * EXCLUDE / REPLACE` | 특정 컬럼만 빼거나 변형해서 조회 |
| `GROUP BY ALL / ORDER BY ALL` | 자동 그룹핑/정렬 |
| `QUALIFY` | 윈도우 함수 결과 필터링 |
| `PIVOT / UNPIVOT` | 엑셀식 피벗 |
- 문서: `sql/` → *Friendly SQL*

---

### 4단계 — 파일 직접 쿼리 (2일)

- 테이블 로드 없이 CSV/Parquet를 바로 `FROM 'file.parquet'`로 쿼리
- Parquet 우선순위로 학습 (실무에서 CSV보다 압도적으로 많이 씀)
- 문서: `data/parquet`, `data/csv`

---

### 5단계 — 중첩 데이터: UNNEST / STRUCT (GA4 대비 최우선 ★)

- GA4 BigQuery 익스포트의 `event_params`, `items`가 STRUCT 배열 형태라, 이 문법이 **가장 레버리지 높은 학습 지점**

```sql
SELECT event_name, ep.key, ep.value.string_value
FROM events, UNNEST(event_params) AS ep;  -- 배열을 행으로 펼침
```

- 문서: `sql/data_types/nested`, `sql/query_syntax/unnest`

---

### 6단계 — 성능 감각 (1일)

- 컬럼 지향 + 벡터화 실행, Parquet 통계 기반 프루닝, 자동 멀티스레드
- "왜 이 쿼리가 느린가"를 판단할 최소한의 내부 구조 이해

---

### 7단계 — 실무 연동

- **pandas 연동**: `SELECT * FROM df` 형태로 DataFrame 직접 쿼리
- **비용 절감 워크플로우**: BigQuery에서 샘플을 Parquet로 내려받아 DuckDB로 쿼리를 빠르게 반복 검증 → 완성된 쿼리만 BigQuery에 실행 (스캔 비용 절감)
- **dbt-duckdb**: 이름과 동작 원리 정도 이해 (로컬 dbt 개발 환경으로 흔히 쓰임)

---

### 8단계 — 실습

- **NYC Taxi 데이터셋**(Parquet)으로 3~6단계 문법 종합 연습
- GA4 스키마를 흉내낸 샘플 데이터로 `UNNEST` 실전 연습

---

**예상 소요**: 집중 시 1.5~2주. 5단계(UNNEST/STRUCT)에 시간을 가장 많이 배분하시길 권합니다.

---

# Dissertation Reading

## “One Size Fits All”: An Idea Whose Time Has Come and Gone

### 논문 개요

- **제목**: *"One Size Fits All": An Idea Whose Time Has Come and Gone*
- **저자**: Michael Stonebraker (MIT, INGRES/Postgres 설계자), Uğur Çetintemel (Brown University)
- **학회**: ICDE 2005 (사용하신 파일명은 2015로 돼있지만 논문 본문 기준 실제 발표는 2005년입니다)
- **위상**: DB 시스템 역사에서 자주 인용되는 **포지션 페이퍼**입니다. DuckDB, ClickHouse, Snowflake 같은 컬럼 지향 분석 엔진들이 등장하게 된 사상적 배경이 되는 논문이라, 지금 학습 중인 DuckDB와 직접 연결됩니다.

---

### 핵심 주장 (Thesis)

**"하나의 DBMS 아키텍처로 모든 워크로드를 처리한다"는 지난 25년간의 업계 전략이 더 이상 통하지 않으며, 시장은 워크로드별 특화 엔진들로 분화될 것이다.**

저자들은 이 전략이 유지된 이유가 기술적 우월성이 아니라 **영업/마케팅/유지보수 비용** 문제였다고 지적합니다 (코드라인이 여러 개면 유지보수비 증가, 영업 혼란, 포지셔닝 문제). 즉 "One size fits all"은 애초에 성능 최적화가 아니라 비즈니스 편의를 위한 선택이었다는 게 논지의 출발점입니다.

---

### 근거 1: 데이터 웨어하우징 (OLTP vs OLAP)

SQLD 범위에서 익숙하실 정규화/인덱스 개념을 확장해서 이해하시면 됩니다.

- **OLTP**는 쓰기(update) 최적화, **웨어하우스**는 읽기(ad-hoc 쿼리) 최적화라는 근본적으로 다른 워크로드입니다.
- **스키마 차이**: 웨어하우스는 **스타 스키마**(fact table + dimension table)가 표준인데, OLTP에서는 이런 구조가 거의 안 쓰입니다.
- **인덱스 차이**: OLTP는 **B-tree**, 웨어하우스는 **비트맵 인덱스**가 유리 — 이유는 웨어하우스가 소수 컬럼에 대한 대량 집계가 많기 때문.
- **핵심 기술 포인트 (5.1절)**: 저장 방식 자체가 다릅니다.
    - **Row-store**(행 지향): 레코드 하나를 디스크에 한 번에 쓰기 좋음 → OLTP에 적합
    - **Column-store**(열 지향): 쿼리에 필요한 컬럼만 읽으면 되므로 대량 집계에 압도적으로 유리 → 웨어하우스에 적합
    - **바로 이 column-store 아이디어가 DuckDB의 핵심 아키텍처입니다.** 지난 대화에서 설명드린 "컬럼 지향 + 벡터화 실행"이 이 논문에서 이미 예견된 방향입니다.

---

### 근거 2: 스트림 처리 (논문의 메인 실험, 3~4절)

가장 구체적인 성능 데이터가 나오는 부분입니다.

#### 실험 설계

- 금융 피드 지연 알림 애플리케이션(주식 시세 지연 감지)을 **StreamBase**(전문 스트림 처리 엔진)와 **상용 RDBMS**로 각각 구현해 처리량 비교
- 4500개 종목 (500개 빠른 종목: 5초 지연 기준 / 4000개 느린 종목: 60초 지연 기준), 2개 피드 제공사

#### 결과

- **StreamBase: 초당 160,000 메시지 처리**
- **RDBMS: 초당 900 메시지 처리**
- → **약 178배(두 자릿수, 논문 표현으로는 "two orders of magnitude") 성능 차이**

#### 이 차이의 4가지 원인 (4.1~4.5절) — 논문의 실질적 핵심

1. **Inbound vs Outbound 처리 모델**: RDBMS는 "저장 후 조회"(outbound, pull 방식)가 기본 설계인데, 스트림 처리는 "데이터가 흐르면서 즉시 처리"(inbound, push 방식)가 필요. RDBMS의 트리거는 이를 흉내내려는 "덧붙인 기능"일 뿐이라 근본적으로 느림.
2. **적합한 프리미티브의 부재**: SQL의 `GROUP BY`는 "테이블의 끝"을 전제로 하는데, 스트림은 끝이 없음. 스트림 엔진은 **시간 윈도우(sliding window)** 개념을 기본 내장하지만 RDBMS는 이를 SQL로 흉내내야 해서 비효율적.
3. **DBMS 처리와 애플리케이션 로직의 통합**: RDBMS는 보안을 위해 클라이언트-서버 구조(별도 주소 공간)로 설계됐지만, 스트림 처리는 신뢰된 단일 애플리케이션이므로 **같은 프로세스 안에서** DB 로직과 앱 로직을 섞는 게 프로세스 전환(context switch) 비용을 없애 훨씬 빠름.
4. **고가용성(HA)과 동기화**: 로그 기반 복구는 스트림 환경에 부적합(다운타임 발생), ACID 트랜잭션도 대부분의 스트림 앱엔 과한 스펙(단일 사용자 환경이라 세마포어 수준의 가벼운 동기화로 충분).

---

### 근거 3: 그 외 특화 시장 (5절, 짧게 언급)

- **센서 네트워크**: 전력/대역폭이 핵심 제약이라 기존 DBMS 최적화 전략 자체가 무의미
- **텍스트 검색**: Google이 자체 GFS를 만든 사례 — append-only 쓰기, 순차 읽기, 장애 상시 발생을 전제로 한 아키텍처가 필요해 RDBMS로는 부적합
- **과학 데이터베이스**: 다차원 인덱싱, 데이터 계보(lineage) 추적 등 특수 요구
- **XML 데이터베이스**: 관계형 모델에 잘 안 맞는 반정형 데이터 — 확장이냐 별도 엔진이냐 논쟁 중이라고만 언급 (결론 없이 open question으로 남김)

---

### 결론 (7절)

앞으로 도메인별 특화 DB 엔진들이 다수 등장할 것이며, "One size fits all"은 더 이상 지속 가능한 전략이 아니다 — 라는 예측으로 마무리됩니다. 저자들 스스로 "흥미로운 시대가 올 것"이라며 다소 낙관적/도발적인 톤으로 끝맺습니다.

---

### 왜 지금 이 논문이 의미 있는가

이 논문이 2005년에 예측한 흐름이 이후 20년간 그대로 실현됐습니다:

- 웨어하우스 → **컬럼 스토어** (Vertica, Snowflake, BigQuery, **DuckDB**)
- 스트림 처리 → Kafka Streams, Flink
- 텍스트 검색 → Elasticsearch
- NoSQL/문서 DB → MongoDB 등 (XML 논쟁의 후신)

**DuckDB는 정확히 이 논문의 5.1절(column-store)이 예견한 방향의 산물**입니다 — "분석 워크로드는 row-store RDBMS로는 한계가 있다"는 이 논문의 주장이, 20년 뒤 "로컬 환경에서도 쓸 수 있는 임베디드 column-store"라는 형태로 구현된 게 DuckDB라고 보시면 됩니다. 앞서 배우신 "왜 DuckDB가 컬럼 지향이라 빠른가"의 이론적 뿌리가 바로 이 논문입니다.

---

## Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask

### 논문 개요

- **제목**: *Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask*
- **저자**: Timo Kersten, Viktor Leis, Alfons Kemper, Thomas Neumann (TU München), Andrew Pavlo (CMU), **Peter Boncz (CWI)**
- **학회**: VLDB 2018, Vol. 11
- **중요 포인트**: 공저자 **Peter Boncz는 DuckDB의 공동 창시자**이자, 이 논문이 다루는 "vectorized execution" 패러다임을 처음 제안한 MonetDB/X100 연구의 원저자입니다. 즉 이 논문은 **DuckDB가 왜 지금의 실행 엔진 구조를 택했는지에 대한 이론적·실험적 근거 그 자체**입니다.

---

### 배경: 두 가지 경쟁 패러다임

SQLD 수준에서는 "쿼리 옵티마이저가 실행 계획을 짠다"까지만 다루지만, 이 논문은 그 **실행 계획을 실제로 어떻게 실행하느냐** — 즉 DB 엔진의 CPU 레벨 구현 방식을 다룹니다. 기존의 Volcano 스타일(한 번에 한 행씩 처리)은 요즘 CPU에서 비효율적이라는 전제 아래, 현대 OLAP 엔진들은 두 방식 중 하나를 씁니다.

| 방식 | 대표 시스템 | 핵심 아이디어 |
| --- | --- | --- |
| **Vectorized (벡터화)** | MonetDB/X100 → VectorWise → **DuckDB** | 한 번에 한 행이 아니라, **~1000개 단위의 배치(vector)**로 처리. 각 연산은 하나의 자료형에 특화된 단순 함수("primitive")로 쪼개짐 |
| **Data-centric Compilation (컴파일)** | HyPer, Peloton | 쿼리마다 **전용 머신코드를 생성**해서, 파이프라인 전체를 하나의 타이트한 루프로 융합. 중간값을 CPU 레지스터에 그대로 유지 |

---

### 실험 방법론

기존 비교 연구들은 HyPer vs VectorWise처럼 **실제 상용 시스템끼리 비교**했는데, 문제는 두 시스템이 실행 모델 외에도 압축 방식, 병렬화 전략, 최적화기 등 너무 많은 부분이 달라서 "실행 모델 자체"의 차이를 순수하게 볼 수 없었습니다.

- 그래서 저자들은 **동일한 알고리즘·자료구조**를 쓰는 두 엔진을 직접 구현합니다: 컴파일 방식 **"Typer"**, 벡터화 방식 **"Tectorwise"**
- 실행 모델 하나만 다르게 두고 나머지는 통제 → **apples-to-apples 비교**
- TPC-H 벤치마크 중 5개 쿼리를 대표로 선정 (산술 집계, 선택적 필터, 조인 2종, 고카디널리티 집계)

---

### 핵심 발견

#### 1. 전반적으로는 막상막하

두 모델의 성능 차이는 크지 않습니다 (Q1에서 컴파일이 74% 빠름 ~ Q9에서 벡터화가 32% 빠름 수준). 참고로 HyPer와 PostgreSQL의 차이는 10~100배인 것과 비교하면, **두 최신 패러다임 사이의 격차는 미미한 수준**입니다.

#### 2. 컴파일이 유리한 경우: 연산이 많은 쿼리

Typer(컴파일)는 중간 계산값을 **CPU 레지스터**에 그대로 들고 있을 수 있어서, 벡터화보다 **실행 명령어 수 자체가 적습니다**(최대 2.4배 적음). Q1처럼 산술 연산이 지배적인 쿼리에서 압도적으로 유리한 이유입니다.

#### 3. 벡터화가 유리한 경우: 조인처럼 메모리 접근이 많은 쿼리

직관과 달리, 벡터화는 명령어 수는 더 많이 실행하는데도 **해시 조인 같은 작업에서는 더 빠릅니다**. 이유는:

- 벡터화 코드는 루프가 단순해서, CPU의 **비순차 실행(out-of-order execution)** 엔진이 **더 많은 메모리 요청을 미리 동시에 던질 수 있음**
- 즉 **캐시 미스로 인한 대기 시간(memory stall)을 더 잘 숨김**
- 컴파일 코드는 한 루프 안에 스캔+필터+조인+집계가 다 섞여있어서 이 "선행 요청"의 여지가 줄어듦

> SQLD에서 인덱스/조인 성능을 "논리적 비용"으로만 배우셨다면, 이 부분이 그 아래 **물리적 계층(CPU 캐시, 메모리 지연시간)**에서 실제로 무슨 일이 벌어지는지를 보여주는 대목입니다.
> 

#### 4. SIMD(하나의 명령어로 여러 데이터를 동시 처리)는 기대만큼 효과가 크지 않음

- 마이크로벤치마크에서는 최대 8.4배 향상되지만, **실제 TPC-H 쿼리에서는 조인 쿼리 기준 약 10% 향상에 그침**
- 이유: 대부분의 OLAP 쿼리는 **계산이 아니라 메모리 접근(=데이터를 얼마나 빨리 가져오느냐)이 병목**이라, 계산을 빠르게 하는 SIMD의 이점이 메모리 대기 시간에 묻혀버림

#### 5. 멀티코어 병렬화: 둘 다 잘 확장되고, 격차가 오히려 줄어듦

- "morsel-driven parallelism"(HyPer가 제안한 작업 분배 방식)을 두 엔진에 동일 적용 → 10코어에서 8~11배 정도의 준수한 스케일링
- 흥미로운 점: **하이퍼스레딩을 켜면 두 모델 간 성능 격차가 줄어듭니다.** 스레드가 늘수록 벡터화의 "명령어 수가 많다"는 단점이 하드웨어 차원에서 가려지기 때문

#### 6. 하드웨어를 바꿔도 결론은 동일

Intel Skylake, AMD Threadripper, Intel Xeon Phi(Knights Landing) 세 플랫폼에서 실험했지만, 어느 하드웨어에서도 한쪽이 일방적으로 우세하지 않았습니다.

---

### 원시 성능 외의 실무적 차이 (8절)

저자들은 **"성능 차이가 크지 않으니, 실제 선택은 다른 요인이 좌우한다"**고 결론짓습니다.

|  | 컴파일 (Typer 계열) | 벡터화 (Tectorwise 계열) |
| --- | --- | --- |
| **OLTP** | 유리 (한 행짜리 쿼리를 저장 프로시저처럼 통째로 컴파일 가능) | 불리 (배치 처리 전제라 이점이 없음) |
| **컴파일 시간** | 불리 (쿼리마다 코드 생성/컴파일 필요) | 유리 (primitive가 이미 사전 컴파일됨) |
| **프로파일링/디버깅** | 어려움 (생성된 코드라 추적 힘듦) | 쉬움 (각 primitive 단위로 시간 측정 가능) |
| **적응형 실행(런타임 최적화 변경)** | 매우 어려움 | 가능 (VectorWise의 micro-adaptive 최적화 사례) |

---

### 결론

**두 패러다임 사이에 절대적 승자는 없다.** OLAP 워크로드에서는 성능이 비슷하고, 실질적 선택은 워크로드 성격(OLTP 지원 필요 여부), 프로파일링/디버깅 편의성, 컴파일 시간 리스크 같은 **엔지니어링 트레이드오프**에 달려있다는 것이 논문의 최종 메시지입니다. 실제로 최신 시스템들(HyPer + Data Blocks, Peloton 등)은 두 방식을 섞은 **하이브리드**로 수렴하는 추세라고 언급합니다.

---

### DuckDB와의 직접적 연결

이번 논문은 지난번 Stonebraker 논문보다 **DuckDB 엔진 내부와 훨씬 더 직접적으로 연결**됩니다.

- **DuckDB는 이 논문이 다루는 "Tectorwise" 계열, 즉 벡터화 실행 엔진**입니다. 실제로 DuckDB는 MonetDB/X100 혈통을 그대로 이어받았고(공동 창시자 Boncz가 X100/VectorWise의 원저자), 내부적으로 튜플을 **~2048개 단위 벡터**로 처리합니다 — 이 논문 4.3절에서 실험한 "vector size" 튜닝이 실제로 DuckDB 소스코드에 반영돼 있는 값입니다.
- **왜 DuckDB가 컴파일 방식(HyPer, Umbra류)이 아니라 벡터화를 택했는가**에 대한 답이 이 논문에 있습니다: 벡터화는 (1) 임베디드 환경에서 **컴파일 시간 오버헤드가 없고**, (2) **디버깅·프로파일링이 쉬우며**, (3) 구현 복잡도가 상대적으로 낮으면서도, (4) 대부분의 실제 워크로드가 이 논문이 보여준 대로 **메모리 바운드**라 컴파일 방식 대비 성능 손해가 크지 않기 때문입니다. 즉 "속도는 거의 동등한데 엔지니어링 비용이 훨씬 적은" 선택인 셈입니다.

앞서 배우신 "DuckDB는 컬럼 지향 + 벡터화 실행이라 빠르다"는 설명의 **"벡터화"라는 단어가 정확히 이 논문에서 정의되고 실증된 개념**입니다.

---
