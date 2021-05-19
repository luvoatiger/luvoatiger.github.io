---
layout: post
title: 몬테카를로 시뮬레이션
date: 2021-05-08
excerpt: "1. Monte Carlo Approach"
tags: [simulation]
comment: true
---


# Monte Carlo Approach
몬테카를로 접근법은 무작위로 발생시킨 난수를 이용해서 함수의 값을 구하는 접근법을 말합니다. 최적화, 수치적분, 확률분포 근사 등등 여러 가지 분야에 쓰이고 있습니다. 이번 포스팅에서는 간단한 이산확률분포를 근사하는 예제를 통해, 몬테카를로 접근법을 한 번 살펴 보도록 하겠습니다. 그럼 시작하겠습니다.


# 핵심 아이디어
몬테카를로 시뮬레이션을 할 때는 주로 표준균일분포의 난수를 발생시키는데, 그 이유는 표준균일분포를 이용하면 특정 이산확률분포 및 연속확률분포 그리고 기대값 등을 근사하기 편리하기 때문입니다. 또한, 표준균일분포를 따르는 난수는 $$0$$ 과 $$1$$ 사이의 값을 갖기 때문에 확률과 유사하게 생각할 수 있습니다. 

표준균일분포를 따르는 확률변수 $$U$$ 의 밀도함수 $$f(u)$$는 $$0$$과 $$1$$ 사이에서 $$1$$의 값을 갖습니다. 그리고 분포함수 $$F(u)$$는 $$u < 0$$이면 $$0$$, $$0 \leq u \leq 1$$이면 $$u$$, $$u > 1$$이면 $$1$$의 값을 가집니다. 그렇기에 다음 관계가 성립합니다.

<p align="center">
$$	
	P(a \leq U \leq b) = F(b) - F(a) = b - a
$$
</p>

이 수식을 다음처럼 확률에 적용해 볼 수 있습니다.

<p align="center">
$$
    P(p_{1} \leq U < p_{1} + p_{2}) = p_{2}
$$
</p>

이 수식은 난수 $$U$$의 값이 $$p_{1}$$ 과 $$p_{1} + p_{2}$$ 사이에 있을 확률이 $$p_{2}$$라는 것을 알려줍니다. 


# 예제
다음과 같은 이산확률분포를 생성하고 싶다고 합시다. 해당 분포를 따르는 이산확률변수 X는 1, 2, 3, 4의 값을 가지며, X의 질량함수 $$P(X=i)=p_{i}$$는 다음과 같습니다.

<p align="center">
    $$
 	p_{1} = 0.2,\quad p_{2} = 0.15,\quad p_{3} = 0.25,\quad p_{4} = 0.4
	$$
</p>

즉, 확률변수 값이 $$i$$일 확률은 $$p_{i}$$임을 알 수 있습니다.

따라서 $$p_{i}$$ 값에 맞게 난수 $$U$$가 위치하는 구간을 설계하고 임의의 난수가 속한 구간에 맞는 확률변수 값을 생성하면, 위의 분포를 따르는 확률변수열을 만들 수 있습니다. 전체 알고리즘은 다음과 같습니다.

<p align="center">	
$$
If \quad U < p_{1}, \quad then \quad X = 1\\If \quad p_{1} \leq U < p_{1} + p_{2}, \quad then \quad X = 2\\If \quad p_{1} + p_{2} \leq U < p_{1} + p_{2} + p_{3}, \quad then \quad X = 3\\If \quad p_{1} + p_{2} + p_{3} \leq U < p_{1} + p_{2} + p_{3} + p_{4}, \quad then \quad X = 4
$$
</p>


# 코드
```python
# generate random variabe X with pmf p1 = 0.2, p2 = 0.15, p3 = 0.25, p4 = 0.4
random_number_x = np.random.uniform(size=10000)
random_variable_x = np.where(random_number_x < 0.2, 1, np.where(random_number_x < 0.35, 2, np.where(random_number_x < 0.6, 3, 4))) #random_number의 값이 0.2보다 작으면 확률변수 1을 부여합니다. 마찬가지로, random_number값이 다른 구간에 속하게 되면 그에 맞는 확률변수를 부여합니다.

# 결과 표시를 위한 코드
number_counter = dict()
for random_variable in list(set(random_variable_x)):
    number_counter[random_variable] = list(random_variable_x).count(random_variable)
#결과 : {1: 2035, 2: 1525, 3: 2485, 4: 3955}
```

결과를 보니 약 40%가 값이 4이므로 원하는 분포가 잘 만들어진 것을 알 수 있습니다.


# Conclusion
이처럼 몬테카를로 접근법은 표준균일분포를 따르는 난수를 무작위로 추출해서 이를 변형하여 분포를 근사하거나 손으로 구하기 어려운 적분값을 구하는 방법입니다. 추가적인 이산확률분포 및 연속확률분포를 근사할 때도 표준균일분포를 이용하는데, 수치적으로 적분하는 예제도 보도록 하겠습니다.
