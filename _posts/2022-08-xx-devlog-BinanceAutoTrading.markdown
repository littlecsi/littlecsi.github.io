---
layout: post
title: 'Binance Auto Trading [2]'
subtitle: '바이낸스 자동 매매 구현하기 [2]'
categories: devlog
tags: crypto
comments: true
---

![Binance x Python](/assets/img/devlog/Avis-Binance.jpeg)

# Table of Contents

# Introduction
tutorial 편은 조코딩님의 튜토리얼과 책을 거의 따라했다. 하지만, 모두 미숙한 점이 많다고 생각되어 코드 개선을 하고자 내가 조금 더 덧붙이려고 한다.

# Implementation

## Slack Debugging
코드가 AWS 서버에서 돌아가고 있고, 내가 24시간 감시를 할 수 없는 상황이기에 슬랙앱을 통해서 프로그램 이상 감지를 할 생각이다. 

```python
def post_message(token, channel, text):
    response = requests.post("https://slack.com/api/chat.postMessage",
        headers={"Authorization": "Bearer " + token},
        data={"channel": channel,"text": text}
    )
    print(response)
```

이렇게 **post_message()** 함수를 통해서 바이낸스 API에 접근하는 코드를 `try...except`문에 넣고 문제가 있으면 슬랙봇을 통해 나에게 경고를 보내는 형식이다. 

```python
    try:
        balances = client.account()["balances"]
    except:
        msg = "get_balance() - cannot get " + asset + " information."
        post_message(config.slack_token, "#debug", msg)
```

이렇게 한다면 서버에서 코드가 멈췄을때도 슬랙 알림을 통해 알 수 있다.

## Sell Algorithm Fixed

현재 코인 판매 알고리즘에는 조금 문제가 있다. 만약 12시에 판매할 때, 현재 가격이 목표가보다 높으면 팔자마자 바로 다시 구매하기 때문이다. 이를 방지 하기 위해서 조건문을 현재가가 새로운 목표가보다 같거나 높으면 팔지 않도록 했다.따라서, 내가 코인을 가지고 있고, 현재가가 목표가보다 낮으면 판매하도록 바꿨다.

```python
balance = biat.get_balance(client, asset)
if (balance != 0) and (current_price < target_price):
    # Sell crypto
    biat.sell_crypto(client, balance, asset)
else:
    biat.post_message(config.slack_token, "#trade-alert", "NOTHING TO SELL")
```

물론, 매일 12시에 판매하는 게 부적절할 수도 있지만, 일단은 이렇게 두고 프로그램을 돌리면서 개선 사항을 고려해 보겠다. 

튜토리얼을 따라하면서 왜 굳이 MARKET 매매를 하는지 의문이다. 12시에 LIMIT 구매 주문을 넣어놓고 시장가의 변화를 보며 주문을 취소해 변동하거나 하면 되지 않냐는 의문이 들었다. 따라서, 12시에 새로운 목표가가 생기면서 LIMIT 주문을 하는 방향으로 코딩 해보려고 한다. 이렇게 하게 되면 좋은 점이라면, 코드를 처음 실행했을 때, 프로그램이 실수로 코인을 높은 가격에 구매하지 않게 할 수 있다는 점이다.

이렇게 코드에 큰 수정이 들어가니 branch를 새로 따서 코딩을 해주자. 나는 main에서 `develop` branch를 따로 땄다.

