---
layout: post
title: 몬테카를로 시뮬레이션
date: 2021-05-19
excerpt: "3. Monte Carlo Integration from a to b"
tags: [simulation]
comment: true
---

> 지난 포스팅에서는 구간 $$0$$과 $$1$$사이에서 함수의 적분값을 수치적으로 구하는 과정을 정리했습니다. 이번에는 구간 $$a$$와 $$b$$사이에서 함수의 적분값을 수치적으로 구하는 방법을 설명하도록 하겠습니다.



# 배운 것을 활용하자
구간 $$[a, b]​$$ 사이에서 함수 $$g(x)​$$를 적분하려고 합니다. 이 때, 앞에서 배운 $$[0, 1]​$$ 사이에서 함수 $$h(u)​$$를 적분하는 방법을 어떻게 사용할 수 있을까요? 먼저 적분구간이 다르므로 구간을 맞춰줘야 할 것 같습니다. 그리고 그에 맞춰서 함수 $$g(x)​$$를 어떻게 변형하면 되는지 생각해보면 될 것 같아보입니다.

즉, 함수 $$g(x)​$$를 함수 $$h(u)​$$로 변형하는데 다음 조건을 만족해야 합니다.

<p align='center'>
    $$
    \int_{a}^{b}g(x)dx = \int_{0}^{1}h(u)du
    $$
</p>



# 함수를 변형하자

먼저 적분구간이 다르므로 이를 맞춰줍시다. 구간 $$[a, b]​$$를 구간 $$[0, 1]​$$로 맞추는 게 개인적으로 생각하기 더 편해서 다음 관계가 성립하는 것을 알 수 있었습니다.



<p align='center'>
	$$
    u = \frac{x-a}{b-a}
    $$
</p>



따라서 함수 $$g(x)​$$의 구간을 $$[a, b]​$$에서 $$[0, 1]​$$사이로 바꾸면 원래의 함수는 다음처럼 변형됩니다.



<p align='center'>
	$$
    \int_{a}^{b}g(x)dx = \int_{0}^{1}g(a + [b - a]u)(b - a)du
    $$
</p>



여기서 $$u$$는 표준균일분포에서 생성한 난수입니다.



# 예제
```python
# integral of exp(-x^2) from a to b
def integrate_exp_negative_square(a, b, size):
    random_number = np.random.uniform(size=size)
    transformed_random_number = a + (b - a) * random_number
    exp_x_square = np.exp(-np.square(transformed_random_number))
    final_result = (b - a) * exp_x_square

    return np.mean(final_result)

manual_solution = integrate_exp_negative_square(1, 10, 1000000)
scipy_solution = integrate.quad(lambda x : math.exp(-x**2), 1, 10)
print("manual : {}, scipy : {}".format(manual_solution, scipy_solution))  
# manual : 0.13933370564933364, scipy : (0.13940279264033098, 3.928467470000696e-15)  
```
1,000,000개의 표준균일분포 난수 $U$가 담긴 배열을 생성한 다음에 수작업으로 구한 결과와 scipy 패키지에서 구한 결과와 비슷한 것을 알 수 있습니다.
