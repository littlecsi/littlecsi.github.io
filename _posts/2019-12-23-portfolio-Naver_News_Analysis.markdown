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
    - [1.1 Initialisation](#11-initialisation)
    - [1.2 URL analysis](#12-url-analysis)
    - [1.3 Exceptions](#13-exceptions)
    - [1.4 Rvest Crawling](#14-rvest-crawling)
- [Sources](#sources)

# Analysis

> 문제 파악 및 기획 방향 분석

네이버 뉴스를 크롤링하기 전에 웹사이트를 분석하였습니다.



# Data Requirement

Data source : [https://news.naver.com](https://news.naver.com)

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

### 1.1 Initialisation

> Data Requirement에 있는 데이터 웹크롤링하기

우선 크롤링을 하기 위해 `rvest` 와 `RSelenium` 패키지를 사용했습니다. Selenium을 사용함으로써 `rvest`로 GET 하거나 POST 하여 가져올 수 없는 웹페이지를 크롤링할 떄 편리합니다. 예를 들어 클릭해서 로그인 후 내용을 크롤링 한다든지, 검색어를 입력해서 크롤링 하는 경우, 웹표준을 지키지 않아서 크롤링이 어려운 경우 등에 사용하면 편리합니다.

RSelenium 을 사용하기 위해서 3개의 드라이버를 다운 받았습니다:

- [Selenium Standalone Server](http://selenium-release.storage.googleapis.com/index.html)

자바 jar 파일입니다. Selenium 서버를 실행하기 위한 툴입니다.

- [Gecko Driver](https://github.com/mozilla/geckodriver/releases/tag/v0.17.0)

W3C WebDriver 프로토콜을 사용하여 Selenium 과 통신합니다. W3C 는 보편적으로 정의된 표준입니다. 즉, Selenium 개발자들이 각 브라우저 버전별로 Selenium 버전을 계속 개발하지 않아도 된다는 뜻입니다.

- [Chrome Driver](https://sites.google.com/a/chromium.org/chromedriver/)

이 또한 W3C WebDriver standard를 시행한 독립 서버입니다.

RStudio 에서 gecko driver 를 사용하려면 먼저 관리자 권한으로 CMD를 실행시켜 위파일이 설취된 폴더에 접근하였습니다. 그리고, 다음 명령어를 입력하여 Gecko WebDriver 를 이용해 Selenium 서버를 4445 포트에 실행하였습니다.

```
java -Dwebdriver.gecko.driver="geckodriver.exe" -jar selenium-server-standalone-3.9.1.jar -port 4445
```

서버가 성공적으로 실행되면 다음과 같은 출력을 확인할 수 있습니다.

```
INFO - Selenium Server is up and running on port 4445
```

다음으로, RStudio에서 원격 드라이버를 실행시키는 코드를 작성하였습니다.

```R
library(RSelenium)

remDr <- remoteDriver(remoteServerAdd='localhost', port=4445L, browserName='chrome')
remDr$open()
```

저는 'Chrome' 브라우저를 사용하여 로컬 호스트 서버에서 미리 명시한 4445번 포트로 remoteDriver 를 실행시켰습니다.

성곡적으로 실행시킨 경우, 새로운 Chrome 창이 떴습니다.

### 1.2 URL analysis

크롤링할 사이트 url 을 분석 후 baseurl 을 *url* 변수에 저장하였습니다.

```R
url <- 'https://news.naver.com/main/ranking/popularDay.nhn?rankingType=popular_day&sectionId=100&date='
```

- sectionId 는 뉴스 분야를 나타내는 고유값이다. 100부터 105까지 총 6개의 section(정치, 경제, 사회, 생활/문화, 세계, IT/과학) 입니다.
- date 는 날짜를 나타내므로, 형식은 20181101 처럼 8자리 숫자를 입력합니다.

### 1.3 Exceptions

'Data Requirement' 에서 말했듯이 Training 데이터는 1년치 데이터이고, Testing 데이터는 1개월 데이터입니다. 먼저 for 문을 돌리기 위한 변수들을 만들었습니다.

```R
year <- as.character(c(2018:2019))
month <- c('01','02','03','04','05','06','07','08','09','10','11','12')
day <- c(c('01','02','03','04','05','06','07','08','09'), 10:31)
month_eng <- c('Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec')
```

더 나아가, 크롤링 시작하는 날짜와 끝나는 날짜도 변수에 하고 크롤링 시작 및 탈출을 하기 위한 flag 논리 변수도 생성하였습니다.

```R
startDate <- '20191101'
stopDate <-  '20191130'

startFlag = F
finFlag = F
```

이제 for 문 속 예외 처리를 확인해 보겠습니다.

```R
for(y in year) {
    if(finFlag == T) { break }
    if(as.integer(y) < as.integer(str_sub(startDate, 1, 4)) & isFALSE(startFlag)) { next }
    
    mcnt <- 0 # Index to iterate month_eng vector
    for(m in month) {
        mcnt <- mcnt + 1 # Incrementing index 

        if(finFlag == T) { break }
        if(as.integer(m) < as.integer(str_sub(startDate, 5, 6)) & isFALSE(startFlag)) { next }
        
        for(d in day) {
            if(finFlag == T) { break }
            if(as.integer(d) < as.integer(str_sub(startDate, 7, 8)) & isFALSE(startFlag)) { next }
            # Discard months with no 31st (except February)
            if(m %in% c('04','06','09','11') & d == 31) { next }
            # Discard February (special cases)
            if(m %in% c('02') & d >= 30) { next }
            
            if(isFALSE(startFlag)) { startFlag = T }

            dat <- paste(y, m, d, sep='')

            if(dat == stopDate) { finFlag = T }
        }
    }
}
```

먼저 for 문을 도는 변수가 크롤링 시작날짜 *startDate* 변수와 일치할 때까지 다음 반복문으로 넘어가고, 크롤링 마치는날짜 *stopDate* 변수와 일치할 경우 그 반복문을 마지막으로 실행하고 반복문을 탈출 할 수 있게 설계하였습니다.

*day* 변수를 반복처리 하면서 이런경우 다음 반복문으로 넘어가도록 설계했습니다:
- 31일이 없는 월일 경우 (4월, 6월, 9월, 11월)
- 2월 30일 또는 31일인 경우

이 모든 예외처릴 거치고도 *day* 변수 반복문에 있다는 것은 3개의 반복문이 startDate와 일치하다는 뜻으로 크롤링이 시작할 준비가 되었다는 뜻입니다. 그러므로 이때 *startFlag* 변수를 True 으로 바꾸어 줍니다.
그리고, *dat* 변수에 *url* 변수 속 date를 지정할 때 넣을 값 (8자리 숫자배열) 을 저장합니다.

마지막으로, *dat* 변수가 *stopDate* 와 일치할 경우 *finFlag* 를 True 값으로 바꾸어줍니다.

*mcnt* 변수는 마지막에 모은 크롤링 데이터를 가지고 엑셀 파일에 저장할 때 필요한 index 변수입니다.
이는 나중에 데이터 저장을 다루는 챕터에서 다루겠습니다.

### 1.4 Rvest Crawling



# Sources
[R을 이용한 Selenium 실행 (Windows 10 기준)](https://hmtb.tistory.com/5)

