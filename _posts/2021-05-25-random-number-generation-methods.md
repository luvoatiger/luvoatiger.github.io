---
layout: post
title: 적분 예제
date: 2021-05-25
excerpt: "6. Random Number Generation Methods"
tags: [Bayesian]
comment: true
---



﻿# 1. Inverse Transform Method

- 누적분포함수(cdf)의 형태를 구할 수 있을 때, 사용할 수 있는 방법. 누적분포함수의 역함수를 구한 다음에, 균일분포를 따르는 난수를 입력해주면 된다. 가장 간단한 방법이지만, 누적분포함수를 closed form 형태로 구할 수 있는 분포가 많지 않아서 일반적으로 사용하기 어렵다.

- 이 방법의 근거는 확률적분변환이며, 다음과 같다.

$$ If \quad U \sim Uniform(0, 1) \quad and \quad F_{X}(x) \quad is \quad target \quad CDF, $$

$$ Then \quad F_{X}^{-1}(u) = inf\{x : F(x) \ge u\} \quad \overset{d}{\to} \quad F_{X}(x) $$


- 증명

$$ Let \quad X \quad be \quad random \quad variable \quad from \quad target \quad distribution \quad F_{X}(x) $$

$$ Let \quad U \quad be \quad a \quad random \quad variable \quad s.t \quad U= F_{X}(x) = P_{X}(X \le x)$$

$$ Then, \quad F_{U}(u) = P(U \le u) = P(F_{X}(x) \le u) = P(x \le F_{X}^{-1}(u)) = F_{X}(F_{X}^{-1}(u)) = u$$

  

- 요약

$$ Therefore \quad F_{X}(x) \quad follows \quad uniform \quad distribution \quad and \quad we \quad can \quad generate \quad x \quad using \quad F_{X}^{-1}(u)$$

## 적용 아이디어

- $P(X = x_{j})=p_{j}$를 따르는 이산확률분포를 생성하는 방법은 다음과 같다.

$$ X = x_{j} \quad if \quad U = p_{j} \quad i.e. \quad F(x_{j-1}) \le U \le F(x_{j}) \quad where \quad F(x_{j}) = \sum_{i=0}^j {p_{i}}$$

- 타겟 분포의 확률 구간에 따라, Uniform Random Variable에 확률 변수 값을 역으로 부여하는 아이디어이다. 즉, Uniform Random Variable이 확률이다.

- 연속확률분포를 생성하려면 $U$를 생성하고, $F_{X}^{-1}$를 구해서 $U$를 입력하면 된다.

```python
# generate random variabe X with pmf p1 = 0.2, p2 = 0.15, p3 = 0.25, p4 = 0.4

random_number_x = np.random.uniform(size=10000)
random_variable_x = np.where(random_number_x < 0.2,  1, np.where(random_number_x < 0.35,  2, np.where(random_number_x < 0.6,  3,  4)))  #random_number의 값이 0.2보다 작으면 확률변수 1을 부여합니다. 마찬가지로, random_number값이 다른 구간에 속하게 되면 그에 맞는 확률변수를 부여합니다.

pd.DataFrame(random_variable_x).value_counts().sort_index()
# 결과
1    1997
2    1409
3    2494
4    4100
dtype: int64
----------
```

# 2. Composition Approach

- 타겟 분포가 다음과 같다고 하자. $α$에 따라서, 두 분포의 값이 섞인 분포이다.

$$ P(X = j) = p_{j} =α p_{j}^{(1)} + (1-α)p_{j}^{(2)} $$

- 이 분포는 $α$ 확률로 $p_{j}^{(1)}$ 그리고 $(1-α)$ 확률로 $p_{j}^{(2)}$가 된다. 따라서, 타겟 확률변수열은 $α$ 확률로 $X_{1}$, $(1-α)$ 확률로 $X_{2}$가 된다. 여기서 $X_{1}$은 $p_{j}^{(1)}$을 따르는 확률변수이고, $X_{2}$는 $(1-α)p_{j}^{(2)}$를 따르는 확률 변수이다.

```python
# 예제

# 1, 2, 3, 4, 5가 나올 확률은 0.05, 6,7,8,9,10이 나올 확률은 0.15가 나오는 이산확률분포 시뮬레이션

# 즉, 6,7,8,9,10의 개수가 1,2,3,4,5의 개수보다 3배 많아야 한다.


# 아이디어 : 그림을 그려서 확률 분포를 관찰해보면, 1,2,3,4,5,6,7,8,9,10이 모두 0.1인 uniform 분포와 6,7,8,9,10만 0.2인 uniform 분포를 섞으면 되는 것을 알 수 있다. 단, 섞을 때 alpha 를 0.5로 주면 된다.

  

u1 = pd.DataFrame(np.random.uniform(size=100000), columns=['u1'])  # 확률 alpha에 해당하는 난수 정의
u2 = pd.DataFrame(np.random.uniform(size=100000), columns=['u2'])  # 확률변수를 할당할 난수 정의
df = pd.concat([u1, u2], axis=1)

x1 = df['u2'][df['u1']<0.5].apply(lambda x :  int(10*x) + 1)
x2 = df['u2'][df['u1']>=0.5].apply(lambda x :  int(5*x) + 6)

df['target'] = pd.concat([x1, x2])
df['target'].value_counts().sort_index()

#결과
1      4895
2      4914
3      5025
4      4977
5      5042
6     14976
7     15156
8     14943
9     14978
10    15094
Name: target, dtype: int64

----------
```

# 3. Acceptance Rejection Technique

- 많이 사용되는 방법이다. Rejection Sampling 이라고도 부른다. 샘플링하려고 하는 타겟 분포 $p$의 확률 밀도 함수를 알고 있지만, p에서 샘플링하기 어려울 떄 사용한다.

- Rejection Sampling은 $q$에서 샘플을 추출하고, 해당 샘플의 분포를 $p$에 맞게 수정하는 방법이다. 구체적으로는 샘플링하기 쉬운 분포 $q$에서 샘플을 추출한 다음, $q$에서 추출된 샘플들이 $p$에서 추출됐다고 봐도 되는 샘플이면 채택하고 그렇지 않고 그대로 $q$에서 나왔다고 봐야한다면 기각하게 된다.

- MCMC에서 나오는 Gibbs Sampler, Metropolis-Hastings에서도 비슷한 수식과 논리가 전개되므로 이 알고리즘은 증명까지 알아두는 게 좋다.

## 알고리즘 세팅

- Assume that it is hard to generate $X$ from the target density $p$

- Assume that it is easy to generate $Z$ from the density $q$ which satisfies that $sup\frac{q(x)}{p(x)} \le \frac{1}{π}$. 다르게 표현하면, $π \le sup\frac{p(x)}{q(x)}$ 일 것이다.

- Accept $Z_{i}$ as a target random variable $X_{i} \quad if \quad U_{i} \le \frac{πp(Z_{i})}{q(Z_{i})}$

$$Where \quad Z_{1}, Z_{2}, Z_{3}, ... \overset{\underset{\mathrm{iid}}{}}{\thicksim} q \quad \perp \quad U_{1}, U_{2}, U_{3}, ... \overset{\underset{\mathrm{iid}}{}}{\thicksim} Uniform(0, 1) $$

- 여기서 π는 acceptance probability이며, q는 p보다 tail이 두꺼워야 한다.

## 증명

### Acceptance Probability가 π인 이유

- 첫 번째 괄호에서 conditioning이 된 이유는 conditioning principle 때문이다.

$$ E_{Y}(P(X|Y))= \int_{-\infty}^{\infty}P(X|Y)P(Y)dy = \int_{-\infty}^{\infty}P(X,Y)dy = P(X)$$

- 증명

$$ P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})}\right) = E_{Z}\left[P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})} | Z_{i}\right)\right] = E_{Z}\left(\pi \frac{p(z_{i})}{q(z_{i})}\right) = \pi\int_{-∞}^{∞}\frac{ p(z_{i})}{q(z_{i})}{q(z_{i})dz} = \pi \int_{-∞}^{∞} p(z_{i})dz = π$$

- Conditioning을 쓴 이유는 U도 확률변수, Z도 확률변수이므로 Z를 고정시켜야 확률을 구할 수 있기 때문이다.

- U와 Z는 독립이므로, 조건부 확률이 계산된다.

- Z의 분포는 q이므로 기댓값 공식을 쓰면, 파이가 나오게 된다.

  

### Accepted된 난수 $U$가 $X$의 분포를 따라가는 이유

- X의 분포함수는 $F_{X}(x)$이다. 그리고 X는 첫 번째에 바로 Accept될 수도 있고, Accept되지 않다가 한참 뒤에 Accpet 될 수도 있다. 따라서, 각 경우의 확률을 모두 합해야 한다. 먼저, 뽑힌 Zi가 샘플 Xi로 채택될 확률은 다음과 같다.

$$ (1-\pi)^{i-1}P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})}, Z_{i} \le x \right)$$

- 따라서, 전체 횟수에 대해서 생각해보면, 다음과 같아진다.

$$ P(X \le x) = \sum_{i=1}^{\infty}(1-\pi)^{i-1} P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})}, Z_{i} \le x \right)$$

- 수식을 풀면 다음과 같다.

$$ = \sum_{i=1}^{\infty}(1-\pi)^{i-1} E_{Z}\left[P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})}, Z_{i} \le x |Z_{i} \right)\right]$$

$$ = \sum_{i=1}^{\infty}(1-\pi)^{i-1} E_{Z}\left[P\left(U_{i} \le \frac{\pi p(z_{i})}{q(z_{i})}\right)*I(Z_{i} \le x)\right] \quad since \quad if \quad Z_{i} \ge x, \quad then \quad probability = 0 \quad else \quad trivial$$

$$ = \sum_{i=1}^{\infty}(1-\pi)^{i-1} E_{Z}\left[\frac{\pi p(z_{i})}{q(z_{i})}*I(Z_{i} \le x)\right]$$

$$ = \sum_{i=1}^{\infty}(1-\pi)^{i-1} \left[\int_{0}^{x}\frac{\pi p(z_{i})}{q(z_{i})}{q(z_{i})dz}\right]$$

$$ = \pi \sum_{i=1}^{\infty}(1-\pi)^{i-1} F_{X}(x) = F_{X}(x)$$

```python
def  p(x):
	return st.norm.pdf(x, loc=30, scale=10) + st.norm.pdf(x, loc=80, scale=20)

def  q(x):
	return st.norm.pdf(x, loc=50, scale=30)

def rejection_sampling(iter=1000):
	samples = []

	for i in  range(iter):
		z = np.random.normal(50,  30)
		u = np.random.uniform(0, k*q(z))
		
		if u <= p(z):
			samples.append(z)
			
	return np.array(samples)

x = np.arange(-50,  151)
k = max(p(x) / q(x))

sns.kdeplot(rejection_sampling())
plt.title("sampling data")
plt.show()
```
