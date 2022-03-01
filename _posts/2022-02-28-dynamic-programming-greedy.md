---
layout: post
title: 동적 계획법과 탐욕법
date: 2022-02-28
excerpt: "dynamic programming and greedy algorithm"
tags: [Algorithms]
comment: true
---


# 개요
- 동적 계획법과 탐욕법을 구현해보고, 정리해두는 페이지


## 1. 동적 계획법(Dynamic Programming)
- 최적해를 구할 때 전체 문제를 여러 개의 부분문제로 나누어서 해결하는 기법
- 이 때, 기존에 계산한 부분은 재활용하고 새롭게 계산해야 하는 부분만 계산해서 답을 구한다.
- 피보나치 수열을 계산할 때 동적 계획법을 활용하면, 시간 복잡도가 O(2^N)에서 O(N)으로 줄어든다.
- 아래의 두 가지 조건이 만족될 때 동적 계획법을 활용하며, Bottom-up 방식으로 계산할 수도 있고 Top-down 방식으로 계산할 수도 있다.

### 최적 부분 구조
- 전체 문제의 해는 부분 문제들의 해만으로도 구할 수 있어야 한다.

### 중복되는 부분문제
- 전체 문제를 해결할 때, 중복되는 부분 문제들이 존재한다.

#### 예제
```python
"""
피보나치 수열은 이전에 계산했던 문제들의 해를 활용해서 구할 수 있으며, 구하는 수가 커질수록 중복되는 문제들이 존재한다.
"""
import sys
def fibo_dp_from_down(n):
    dp_table = [0 for i in range(n + 1)]
    dp_table[0] = 0
    dp_table[1] = 1
    
    for index in range(2, n + 1):
        dp_table[index] = dp_table[index - 1] + dp_table[index - 2]
        
    return dp_table[n]

dp_memo = {0 : 0, 1 : 1}
def fibo_dp_from_top(n):
    if n > sys.getrecursionlimit():
        raise Exception("input exceeds system recursion limit")
    else:
        if n not in dp_memo.keys():
            dp_memo[n] = fibo_dp_from_top(n - 1) + fibo_dp_from_top(n - 2)
        return dp_memo[n]
```

## 2. 탐욕 알고리즘(Greedy Algorithm)
- 각 단계에서 가장 최선의 방법을 찾아가는 방식으로 해를 구하는 알고리즘
- 현재 시점에서 가장 최선의 선택을 하지만, 항상 최적해를 구할 수 있는 것은 아니다.
- 여기서 '가장 최선'의 의미는 문제마다 다르게 정의되며, 탐욕 알고리즘으로 문제를 해결하려면 탐욕스러운 선택 조건과 최적 부분 구조 조건이 만족되어야 한다.


### 최적 부분 구조
- 전체 문제의 해는 부분 문제들의 해만으로도 구할 수 있어야 한다.

### 탐욕스러운 선택 조건
- 부분 문제에서 최적의 선택이 전체 문제의 해에 반드시 포함되어야 한다.


#### 예제
```python
def minimize_coin(amount, coin_list):
    """
    해당 동전보다 잔돈이 더 작아질 때까지, 남아 있는 동전 중 가장 큰 동전으로 지불한다.
    남은 동전 중 잔돈을 가장 빠르게 거스를 수 있는 동전을 써서 동전 개수를 덜 쓰면, 필요한 전체 동전의 개수도 줄어든다.
    """
    coin_count = dict()
    for coin in coin_list:
        coin_count[coin] = 0
        while amount >= coin:
            amount = amount - coin
            coin_count[coin] = coin_count[coin] + 1

    return sum(coin_count.values())

def minimize_fractional_knapsack(capacity, item_info):
    """
    이 문제에서는 두 변수(무게, 가치) 모두가 영향을 끼치므로, 무게 대비 가치가 가장 큰 상품이 최선이다.
    무게 대비 가치가 가장 큰 상품부터 채우면, 가치가 가장 크게 배낭을 채울 수 있다.
    """
    
    item_info = sorted(item_info, key = lambda x : x[1] / x[0], reverse=True)
    total_value = 0
    details = {}
    
    for item in item_info:
        weight, value = item
        if capacity >= weight:
            capacity = capacity - weight
            total_value = total_value + value
            details[weight] = 1
        else:
            fraction = capacity / weight
            total_value = total_value + value * fraction
            details[weight] = fraction
            break
        
    return total_value
```
