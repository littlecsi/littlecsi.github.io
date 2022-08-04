---
layout: post
title: '[Binance Auto Trading] Research and Analysis'
subtitle: 'Analysing what problems I will be facing'
categories: devlog
tags: binance
comments: true
---

![Binance x Python]()

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Analysis](#analysis)
  - [Volatility Breakout Strategy](#volatility-breakout-strategy)
- [Implementation](#implementation)
  - [Step 1: Get Current Price Frequently](#step-1-get-current-price-frequently)
  - [Step 2: Calculate Target Price](#step-2-calculate-target-price)
  - [Step 3: Updating Variables](#step-3-updating-variables)
  - [Step 4: Trade Crypto](#step-4-trade-crypto)


# Introduction

Python을 사용해서 Binance 계정과 연결해 비트코인 자동 거래 서비스를 만들어 보겠다. 이를 시작하기 위해서는 Binance의 API key를 먼저 받아야한다. 이는 [여기서](https://www.binancezh.top/en/my/settings/api-management) 생성하면된다. 그 다음, [Binance API Documentation](https://binance-connector.readthedocs.io/en/latest/getting_started.html)을 읽으면서 initialising 하면 된다.

여기서 주의할 점이 있는데 python-binance라는 패키지는 비공식 라이브러리이기 때문에 사용하지 않는 것을 추천한다. 다른 사람들의 말을 들어보면 futures 거래가 API 에러를 일으켜 작동하지 않는다고 했다. 따라서, `binance-connector`라는 바이낸스의 공식 라이브러리를 사용하기를 권장한다.

# Analysis

사실 이번 프로젝트는 [조코딩 유튜브 채널](https://www.youtube.com/c/조코딩JoCoding)의 [파이썬 비트코인 투자 자동화](https://youtube.com/playlist?list=PLU9-uwewPMe3KKFMiIm41D5Nzx_fx2PUJ) 재생목록을 따라한다. 다른 점이라면, 조코딩님은 업비트 Upbit 플랫폼을 사용하셨고, 나는 바이낸스 Binance를 사용한다는 점이다. API가 다르기 때문에 이것을 제외하면 조코딩님이 사용하시는 "변동성 돌파 전략"을 같이 구현할 것이다.

[변동성 돌파 전략](https://wellsw.tistory.com/m/131)은 링크를 통해 들어가 자세한 설명을 볼 수 있지만 안 그럴것이니 여기에 적겠다. 

## Volatility Breakout Strategy

래리 윌리엄스(Larry Williams)가 Long-Term Secrets to Short-Term Trading에서 소개한 변동성 돌파 전략은 최근 '가즈아! 가상화폐 투자 마법공식 (강환국 저)' 등에서 소개된 이후로 국내에서도 유명해졌습니다. 변동성 돌파 전략을 간략히 요약해 보면 아래와 같습니다.

1) 가격 변동폭 계산: 투자하려는 가상화폐의 전일 고가(high)에서 전일 저가(low)를 빼서 가상화폐의 가격 변동폭을 구합니다.
2) 매수 기준: 당일 시간에서 (변동폭 * 0.5) 이상 상승하면 해당 가격에 바로 매수합니다.
3) 매도 기준: 당일 종가에 매도합니다.

간단한 예제와 함께 변동성 돌파 전략을 이해해 봅시다. 가상화폐는 24시간 거래되고 있기 때문에 먼저 시가와 종가를 계산할 기준 시간을 잡아야 합니다. 00:00:00 분부터 그날의 거래가 시작되고 23:59:59에 거래가 종료된다고 가정합시다.
1) 가격 변동폭은 00:00:00~23:59:59 사이의 거래 중에서 가장 높았던 금액과 가장 낮았던 금액을 빼주면 되겠지요? 예를 들어 12월 10일의 비트코인의 고가가 460만 원이고 저가가 300만 원이었다면 이날의 가격 변동폭은 160만 원입니다.
2) 12월 11일 비트코인의 시가(open)가 400만 원이라면 12월 11일의 매수 목표가는 400 + 160 x 0.5 = 480만 원이 됩니다. 12월 11일 하루 종일 일정한 시간마다 현재가를 계속 조회하다가 현재가가 매수 목표가인 480만 원을 넘어서면 바로 매수합니다. 
3) 매도는 12월 12일 00:00:00초에 갖고 있는 비트코인을 시장가로 전량 매도합니다.

출처: https://wikidocs.net/21888

# Implementation

이 자동매매 프로그램은 24시간 구동되어야 하기 때문에 학교에서 받은 교육용 AWS 계정을 이용해 EC2 instance를 생성한 뒤, 이 instance에서 구동할 것이다. EC2 instance 생성과정은 찾아보면 많이 나오고 그렇게 어렵지 않으니 이곳에 적지 않겠다.

[파이썬을 이용한 비트코인 자동매매 ch06-2](https://wikidocs.net/21888)를 참고하면서 코딩을 시작했다. 하지만, 이 책은 빗썸 거래소를 사용하고 나는 바이낸스를 사용하기 때문에 바이낸스 라이브러리로 변환해서 코딩을 했다. 바이낸스 API가 가독성이 조금 떨어질 수 있으므로 조심히 살펴가면서 코딩하자. 

## Step 1: Get Current Price Frequently

이 프로젝트에서 쓰이는 모든 함수들은 *biat.py* 파일에 적고 *main.py* 에서 import해서 사용한다. 

아래 *get_current_price* 함수는 바이낸스 API에서 사용자가 지정한 코인의 현재 값을 불러온다. 바이낸스 API의 거의 모든 반환값들이 문자열이기 때문에 거의 모든 반환값들에게 타입 변환을 해주어야한다. 

```python
def get_current_price(client: Spot, asset: str="BTCUSDT") -> float:
    """"
  Returns the current price of a specific asset.
  Asset type is "BTC" by default.
  """
  assert isinstance(client, Spot)
  assert isinstance(asset, str)

  # Add "USDT" if user just inputs coin symbol
  if len(asset) <= 4: asset += "USDT"

  return float(client.ticker_price(asset)["price"])
```

## Step 2: Calculate Target Price

`pybithumb` 라이브러리에는 너무 편하게도 **get_ohlcv()** 함수가 있지만, `binance-connector`에는 없다. 따라서, 내가 따로 **get_ytd_ohlcv()** 랑 **get_tdy_ohlcv()** 함수를 생성했다. 

```python
def get_ytd_ohlcv(client: Spot, asset: str="BTCUSDT") -> list:
  """
  Returns open, high, low, close, volume data from yesterday.
  """
  assert isinstance(client, Spot)
  assert isinstance(asset, str)

  # Add "USDT" if user just inputs coin symbol
  if len(asset) <= 4: asset += "USDT"

  # Receives today's timestamp and convert to "ms"
  today = int(get_today()) * 1000

  result = client.klines(asset, "1d", endTime=today)

  return result[-1][1:6]
```

나는 이 함수들을 이용해 변동성 돌파 전략에 필요한 *open*, *high*, *low*, *close*, *volume* 데이터들을 수집했고, 목표가를 계산했다. 

```python
def get_target_price(client: Spot, asset: str="BTCUSDT") -> float:
  """
  Returns target price of today.
  """
  assert isinstance(client, Spot)
  assert isinstance(asset, str)
  
  today = get_tdy_ohlcv(client, asset)
  yesterday = get_ytd_ohlcv(client, asset)

  # Volatility Breakout Target calculation
  target = float(today[0]) + (float(yesterday[1]) - float(yesterday[2])) * 0.5

  return float(target)
```

다시 말하지만, 바이낸스 API가 반환하는 값들이 대부분 dictionary 타입이며, value들 대부분이 string 타입이기 때문에 산수를 목적으로 데이터를 수집했다면 타입 변환을 꼭 해주자. 아니면 문자열 결합이 되기 때문이다.

## Step 3: Updating Variables

래리 윌리엄스의 변동성 돌파 전략에서 목표가는 프로그램이 시작될 때 한 번 그리고 매일 자정마다 갱신해야 한다. 목표가를 계산하는 **get_target_price()** 함수를 구현했기 떄문에 자정에 이 함수를 호출하기만 하면 된다. 이때, 파이썬의 `datetime` 모듈을 사용했다.

참고로 바이낸스는 서버 시간을 ms로 알려주기 때문에 꼭 1000을 나눠줘야 한다. 그리고 헷갈리지 않기 위해 모두 UTC 타임존을 사용했다. 

```python
now = datetime.datetime.utcnow()
mid = datetime.datetime(now.year, now.month, now.day) + datetime.timedelta(1)

target_price = biat.get_target_price(client, asset)

while True:
  now = datetime.datetime.utcnow()
  
  # When the day ends
  if mid < now < mid + datetime.timedelta(seconds=10):
    # Calculate new target price for the next day.
    target_price = biat.get_target_price(client, asset)

    # Update next day
    mid = datetime.datetime(now.year, now.month, now.day) + datetime.timedelta(1)
```

## Step 4: Trade Crypto

```python
def buy_crypto(client: Spot, balance: float, price: float, asset: str="BTCUSDT") -> dict:
  """
  Attemps to purchase crypto at target price.
  """
  assert isinstance(client, Spot)
  assert isinstance(balance, float)
  assert isinstance(price, float)
  assert isinstance(asset, str)

  # Calculate the quantity of crypto to buy
  quantity = balance // price

  try:
    response = client.new_order(asset, "BUY", "MARKET", quantity=quantity)
    return response
  except:
    response = {}
    raise Exception("Order not successful.\nPlease try again.")

def sell_crypto(client: Spot, quantity: float, asset: str="BTCUSDT") -> dict:
  """
  Attempts to sell crypto at market price.
  """
  assert isinstance(client, Spot)
  assert isinstance(quantity, float)
  assert isinstance(asset, str)

  try:
    response = client.new_order(asset, "SELL", "MARKET", quantity=quantity)
    return response
  except:
    response = {}
    raise Exception("Order not successful.\nPlease try again.")
```

변수 타입만 신경쓰면 어려움 없는 코드다. 마지막으로 `main.py`를 살펴보자. 

```python
def main():
  now = datetime.datetime.utcnow()
  mid = datetime.datetime(now.year, now.month, now.day) + datetime.timedelta(1)

  target_price = biat.get_target_price(client, asset)

  while True:
    now = datetime.datetime.utcnow()
    
    # When the day ends
    if mid < now < mid + datetime.timedelta(seconds=10):
      # Calculate new target price for the next day.
      target_price = biat.get_target_price(client, asset)

      # Update next day
      mid = datetime.datetime(now.year, now.month, now.day) + datetime.timedelta(1)

      # Sell crypto
      biat.sell_crypto(client, biat.get_balance(client, asset), asset)

    current_price = biat.get_current_price(client, asset)

    # If the current price reaches the target price
    if current_price >= target_price:
      # Get the entire balance of USDT dollars
      balance = biat.get_balance(client, "USDT")

      # Purchase the maximum amount of crypto user can order.
      biat.buy_crypto(client, balance, target_price, asset)

    time.sleep(1)
```

첫 if문 조건이 <현재시간이 다음날 자정과 자정 + 10초 사이>인 이유는 AWS 서버의 일처리 속도 때문이다. 매 while문이 얼마동안 돌아갈지 모르기 때문에 적당한 간격의 10초를 정하고 그 사이를 "자정"이라 부르는 것이다. 마지막에 `time.sleep(1)`를 통해 이 while문은 iteration 당 1초가 조금 넘는 시간이 걸릴 것이다. 