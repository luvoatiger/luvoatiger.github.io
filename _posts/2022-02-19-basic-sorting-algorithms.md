---
layout: post
title: 기본 정렬 알고리즘
date: 2022-02-19
excerpt: "coding test basic sorting algorithms"
tags: [Algorithms]
comment: true
---


# 개요
- 코딩 테스트의 가장 기초적인 정렬 알고리즘을 구현해보고, 정리해두는 페이지


## 1. 버블 정렬
- 코드


```python
def bubble_sort(input_data):
    """
	인접한 값끼리 비교하면서, 순서가 정렬되지 않은 경우 정렬하는 방식을 반복한다.
	turn을 돌면서 가장 큰 원소가 제일 끝 부분으로 이동하게 된다.
	인접한 값끼리 비교하며 이동하는 모습을 거품(bubble)이 터지는 것처럼 보여 버블 정렬 이름이 붙었다.
	최악의 경우에는 len(input_data) * (len(input_data) - 1) / 2 번의 swap연산을 해야 하므로, O(N^2)의 시간복잡도를 가진다.
    """
    for turn in range(len(input_data) - 1):
        need_swap = False
        for index in range(len(input_data) - turn - 1):
            if input_data[index] > input_data[index + 1]:
                input_data[index], input_data[index + 1] = input_data[index + 1], input_data[index]
                need_swap=True

        if need_swap is False:
            break

    return input_data
```

## 2. 삽입 정렬
- 코드

```python
def insertion_sort(input_data):
    """
    두 번째 인덱스에 있는 원소부터 마지막 인덱스에 있는 원소를 대상으로
    그 앞 쪽에 있는 원소들 중 자신보다 작은 원소를 발견할 때까지 크기를 비교한 다음
    기존에 해당 인덱스에 있는 원소부터 앞쪽에 있는 원소들과 자신을 swap한다.

    최악의 경우에는 input_data * (input_data - 1) / 2 번의 swap연산을 해야 하므로, O(N^2)의 시간복잡도를 가진다.
    """
    for turn in range(len(input_data) - 1):
        for inner in range(turn + 1, 0, -1):
            if input_data[inner] < input_data[inner - 1]:
                input_data[inner], input_data[inner - 1] = input_data[inner - 1], input_data[inner]

    return input_data
```


## 3. 선택 정렬
- 코드

```python
def selection_sort(input_data):
    """
    주어진 데이터 중 최솟값을 찾고, 해당 최솟값을 맨 앞의 값과 교체한다.
    이를 나머지 원소들에 대해서도 반복한다.

    최악의 경우에는 input_data * (input_data - 1) / 2 번의 swap연산을 해야 하므로, O(N^2)의 시간복잡도를 가진다.
    """
    for turn in range(len(input_data) - 1):
        min_value = input_data[turn]
        for inner in range(turn + 1, len(input_data)):
            if input_data[inner] < min_value:
                min_value = input_data[inner]
        input_data[turn], min_value = min_value, input_data[turn]

    return input_data
```

## 4. 실행 및 테스트
- 실행 및 테스트 코드

```python
import random
import pytest
def execute_sorting_function(sort_func, input_data):
    """
    정렬 함수와 데이터를 입력받아, 정렬
    """
    return sort_func(input_data)

def test_execute_sorting_function():
    """
    정렬 테스트 코드
    0 ~ 100 사이의 값 50개를 뽑아서, 직접 작성한 메서드의 수행 결과와 내장 함수 sorted의 실행 결과를 비교함
    """
    input_data = list(random.sample(range(100), 50))
    return_data = execute_sorting_function(selection_sort, input_data)
    sorted_data = sorted(input_data)
    assert return_data == sorted_data
```

- 실행 커맨드 및 결과


```bash
실행 커맨드(해당 파이썬 파일의 이름은 sorting.py)
pytest sorting.py

결과

C:\Users\kwonk>pytest sorting.py
================================================= test session starts =================================================
platform win32 -- Python 3.8.8, pytest-6.2.3, py-1.10.0, pluggy-0.13.1
rootdir: C:\Users\kwonk
plugins: anyio-3.3.4
collected 1 item

sorting.py .                                                                                                     [100%]

================================================== warnings summary ===================================================
Anaconda3\lib\site-packages\pyreadline\py3k_compat.py:8
  C:\Users\kwonk\Anaconda3\lib\site-packages\pyreadline\py3k_compat.py:8: DeprecationWarning: Using or importing the ABCs from 'collections' instead of from 'collections.abc' is deprecated since Python 3.3, and in 3.9 it will stop working
    return isinstance(x, collections.Callable)

-- Docs: https://docs.pytest.org/en/stable/warnings.html
============================================ 1 passed, 1 warning in 0.04s =============================================
```
