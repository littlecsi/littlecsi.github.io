---
layout: post
title: '[BigData] Naver News Analysis'
subtitle: 'Naver News Analysis using R'
categories: portfolio
tags: bigdata
comments: true
---
![Naver x RStudio](/assets/img/portfolio/portfolio-naver_news_analysis_01.png)

R을 사용하여서 네이버 뉴스를 분석, 그리고 뉴스 분야를 예측해 보겠습니다. 그러기에 필요한 문제를 파악하고 분석방법 및 예측 모델을 기획한 다음 분석을 시작하겠습니다. 마지막으로, 도출해낸 결과를 시각화함과 동시에 광고 효율을 올리는 것에 대한 제안을 하겠습니다.

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Analysis](#analysis)
- [Data Requirement](#data-requirement)
- [Development](#development)
  - [1 Crawling](#1-crawling)

# Analysis

> 문제 파악 및 기획 방향 분석

네이버 뉴스를 크롤링하기 전에 웹사이트를 분석하였습니다.



# Data Requirement

Data source : https://news.naver.com/

| Column | Data Type | Description | Variable Name |
|:---:|:---:|:---:|:---:|
| Rank | Integer | 뉴스 랭킹 순위 | rank |
| Title | String | 뉴스 제목 | title | 
| Subtitle | String | 뉴스 부제목 | subti |
| Source | String | 뉴스를 적은 회사 | source | 
| View | Integer | 뉴스 조회수 | view |
| Comment | Integer | 뉴스 총 댓글수 | cmt |
| Date | Integer | 뉴스 날짜 | date |
| Current Comment | Integer | 현재 댓글수 | currCmt |
| Deleted Comment | Integer | 삭제된 댓글수 | deleted |
| Broken Policy | Integer | 규정 미준수 댓글수 | brokenPolicy |
| Male Ratio | Integer | 남자 댓글수 비율 | maleRatio |
| Female Ratio | Integer | 여자 댓글수 비율 | femaleRatio |
| 10s Ratio | Integer | 10대 댓글수 비율 | X10 |
| 20s Ratio | Integer | 20대 댓글수 비율 | X20 |
| 30s Ratio | Integer | 30대 댓글수 비율 | X30 |
| 40s Ratio | Integer | 40대 댓글수 비율 | X40 |
| 50s Ratio | Integer | 50대 댓글수 비율 | X50 |
| 60s Ratio | Integer | 60대 댓글수 비율 | X60 |

2018년 11월 ~ 2019년 10월은 예측 모델을 위한 **training** 데이터로 사용할 것이고,<br>
2019년 11월은 예측 모델을 위한 **testing** 데이터로 사용할 것이다.

# Development

## 1 Crawling

> Data Requirement에 있는 데이터 웹크롤링하기

우선 크롤링을 하기 위해 `rvest` 패키지를 사용했습니다.

![News Website Analysis 01](/assets/img/portfolio/portfolio-naver_news_analysis_02.jpg)