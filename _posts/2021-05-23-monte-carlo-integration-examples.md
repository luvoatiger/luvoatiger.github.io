---
layout: post
title: 적분 예제
date: 2021-05-23
excerpt: "5. Monte Carlo Integration Examples"
tags: [Bayesian]
comment: true
---



# 적분 예제


## 예제 1. $$∫_{-2}^{2} e^{(x + x^2)}\, dx $$


```python
def  integrate_function(sample_size, integral_function, start = 0, end = 1):
	random_number = np.random.uniform(0,  1, size=sample_size)
	final_result = integral_function(start, end, random_number)

	return np.mean(final_result)

  

def  ingetrate_exp_x_plus_xsquare(a, b, u):
	X = a + (b - a) * u
	exp_x_plus_xsquare = np.exp(X + np.square(X))
	transformed_result = (b - a) * exp_x_plus_xsquare
	
	return transformed_result

start = -2
end = 2
sample_size = 100000
manual_solution = integrate_function(sample_size, ingetrate_exp_x_plus_xsquare, start, end)
scipy_solution = integrate.quad(lambda x : math.exp(x + math.pow(x,  2)), start, end)

print("manual : {}, scipy : {}".format(manual_solution, scipy_solution))
# manual : 93.69589724019725, scipy : (93.16275329244199, 1.6178564393124623e-09)
```

## 예제 2 $$\int_{0}^{1} \int_{0}^{1}e^{(x+y)^2}dydx\, $$

```python
x = np.random.uniform(0,  1,  1000000)
y = np.random.uniform(0,  1,  1000000)
print(np.mean(np.exp(np.square(x + y))))
```

## 예제 3 $$\int_{0}^{∞} \int_{0}^{x} e^{-(x+y)} dydx\, $$

배운 대로 다음 변환을 취해서 각 적분을 0부터 1사이의 이중적분으로 변환시키자.

$$ t = \frac{1}{x+1}, \quad s = \frac{y}{x} $$

역 변환을 취하면 다음과 같다.

$$ x = \frac{1}{t}-1, \quad y = {s}({\frac{1}{t}-1}), \quad J = \begin{vmatrix} ∂x\over∂t & ∂ x \over ∂s \\ ∂y \over ∂t & ∂y \over ∂s \end{vmatrix} = \frac{1}{t}-1 $$

변수변환 정리에 의해서, 원래 식은 다음처럼 변형된다.

$$ \int_{0}^{1} \int_{0}^{1} exp \left\{ -(\frac{1}{t} -1)(1+s)t^{-2}\left|\frac{1}{t}-1\right| \right\} $$

```python
t = np.random.uniform(0,  1,  100000)
s = np.random.uniform(0,  1,  100000)

transformed = np.exp(-(-1 + 1/t) * (1 + s) * (1/np.square(t)) * np.abs((-1 + 1/t)))

print(np.mean(transformed)) #0.308
```

