---
layout: post
title: 몬테카를로 시뮬레이션
date: 2021-05-22
excerpt: "4. Monte Carlo Integration from zero to infinity"
tags: [simulation]
comment: true
---

> 지난 포스팅에서는 구간 $$a​$$과 $$b​$$사이에서 함수의 적분값을 수치적으로 구하는 과정을 정리했습니다. 이번에는 구간 $$0​$$와 $$\infty​$$사이에서 함수의 적분값을 수치적으로 구하는 방법을 설명하도록 하겠습니다.



# 배운 것을 활용하자
이번에는 구간 $$[0, \infty ]$$ 사이에서 함수 $$g(x)$$를 적분하려고 합니다. 마찬가지로, 앞에서 배운 $$[0, 1]$$ 사이에서 함수 $$h(u)$$를 적분하는 방법을 사용하면 됩니다. 다만 구간을 맞춰줘야 하는데 $$[a, b]$$에서 적분했던 방법을 떠올려보면, 먼저 적분구간이 다르므로 구간을 맞춰주고 그에 맞춰서 함수 $$g(x)$$를 어떻게 변형하면 되는지 생각해보면 됩니다.

즉, 함수 $$g(x)​$$를 함수 $$h(u)​$$로 변형하는데 다음 조건을 만족해야 합니다.

<p align='center'>
    $$
    \int_{0}^{\infty}g(x)dx = \int_{0}^{1}h(u)du
    $$
</p>


# 함수를 변형하자

먼저 적분구간이 다르므로 이를 맞춰줍시다. 구간 $$[0, \infty ]$$를 구간 $$[0, 1]$$로 맞추려면, $$x$$가 $$0$$ 일 때 $$1$$이 되고, $$x$$가 $$\infty$$ 이면, $$0$$이 되면 됩니다. $$\infty$$ 를 $$1$$로 변환하는 것보다, $$0$$으로 변환하는 것이 편하기 때문입니다. 이런 변환을 생각해보면, 다음과 같습니다



<p align='center'>
	$$
    u = \frac{1}{x + 1}, du = -u^2 dx 
    $$
</p>




따라서 함수 $$g(x)​$$의 구간을 $$[0, \infty ]​$$에서 $$[0, 1]​$$사이로 바꾸면 원래의 함수는 다음처럼 변형됩니다.



<p align='center'>
	$$
    \int_{a}^{b}g(x)dx = \int_{0}^{1} \frac{g(\frac{1}{u} - 1)}{u^2} du
    $$
</p>



여기서 $$u$$는 표준균일분포에서 생성한 난수입니다.



# Next

다음 시간에는 다중적분을 소개하도록 하겠습니다.