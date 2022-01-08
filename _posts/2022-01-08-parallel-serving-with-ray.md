---
layout: post
title: 동시성 프레임워크 Ray를 이용한 서빙 병렬 처리
date: 2022-01-08
excerpt: "parallel serving with ray"
tags: [Concurrency]
comment: true
---

# 개요
- 서빙된 모델을 이용해 추론할 때, Ray를 도입해서 AIOps 모듈의 추론 시간을 줄여 안정적으로 서빙하도록 테스트하는 작업


## 1. Ray 테스트 배경
- 실시간 모니터링 서비스는 학습된 모델을 이용해 1분 단위로 수행되며, 구체적으로 다음과 같다.
  - 서버에서 Web Application(e.g. Flask, Django)을 실행시킨다.
  - Application에서 분석 모듈 클래스의 인스턴스를 생성하고, 초기화한다.
  - 분석 모듈 인스턴스에서 사용할 해당 타겟(instance, infra, code) 알고리즘 모델들을 로딩한다.
  - API 통신을 통해, 서버에서 데이터를 전달하면서 Web Application에 서빙 요청을 한다.
  - Web application은 요청을 받으면 분석 모듈에 정의된 serving 함수를 호출해 서빙을 완료하고, 서버로 결과를 전달한다.

- 현재 서빙프로세스는 몇 가지 문제가 있다.
  - 유입되는 데이터를 순차 처리한다. 유입되는 트랜잭션 데이터가 많아지면 그만큼 inference 시간도 늘어나며, 1분 안에 serving을 못 할 위험성이 존재한다.
  - 병렬처리를 위한 파이썬 STL인 multiprocessing, concurrent.futures는 다음 이유로 사용할 수 없다.
    - Web Application에서 분석 모듈 인스턴스를 초기화하고, API를 통해 서빙하므로 " __name__ == '__main__' "이 아니다.
    - 클래스에 정의된 인스턴스 메서드들은 Unpicklable하므로, 내부적으로 pickle을 사용하는 multiprocessing으로는 서빙할 수 없다.

- Ray는 이런 조건들을 만족시키면서, 수행 시간을 줄일 수 있을 것으로 보인다.
  - 클래스에 정의된 인스턴스 메서드들도 병렬 처리할 수 있으며, 초기화 할 때 전역 변수로 일정 개수의 Actor를 생성해두면 ProcessPool과 유사한 효과를 볼 수 있다.
  - 시스템에서 동작하는 Ray Worker 프로세스는 함수를 수행할 때만 자원을 사용하고, 그렇지 않으면 최소한의 자원만 쓰며 IDLE 상태를 유지한다.
  - API가 간단해서 사용하기 편하다.


## 2. Ray 소개
*   Ray는 버클리 대학의 RISE 연구실에서 만들어진 병렬 & 분산처리 프레임워크
*   최근에 해당 연구실 소속 연구원들이 [AnyScale](https://www.anyscale.com/)이라는 회사를 설립해 Ray를 이용한 다양한 기능들을 만들고 있다.



## 3. Ray 장점
*   단순하고 범용적인 API로 분산 처리 및 병렬 처리 할 수 있다.
*   multiprocessing 및 threading과 다르게 코드를 조금만 수정하면, 쉽게 병렬 처리를 할 수 있다.
*   ML/DL 모델 서빙 이외에도 분산 강화학습, 분산 하이퍼파라미터 튜닝, 분산 학습 등 다양한 기능을 제공한다.
*   다른 라이브러리와도 연동이 잘 되어 있어, Hadoop Ecosystem처럼 Ray Ecosystem이 만들어지고 있다.
*   ![](/imgs/ray_ecosystem.PNG)


## 4. Ray Core
*   핵심적인 개념은 Task, Actor, Object이며 그 외의 요소는 차차 알아가도 괜찮다.


### Task

*   호출하는 곳과 다른 프로세스에서 실행되는 함수. Remote Task 라고도 부른다.
*   @ray.remote로 감싼 파이썬 함수
*   stateless함. 즉, 이전 상태를 저장하고 있지 않음
*   비동기로 실행된다.



### Actor

*   @ray.remote로 감싼 파이썬 클래스 인스턴스
*   stateful함. 즉, 이전 상태를 저장하고 있음


### Object


*   Task를 통해 반환된 값
*   ray.get을 호출해서 사용할 수 있는 값으로 변환할 수 있다.


## 5. Ray 주의사항

### 1) 작은 작업은 큰 단위로 합쳐서 처리
```python
import time
import ray

ray.init(num_cpus = 4)

# -----------------------------------------Bad Usage----------------------------------------------
@ray.remote
def tiny_work(x):
    time.sleep(0.0001) # Replace this with work you need to do.
    return x

start = time.time()
result_ids = [tiny_work.remote(x) for x in range(100000)]
results = ray.get(result_ids)
print("duration =", time.time() - start)

# ----------------------------------------Good Usage----------------------------------------------
def tiny_work(x):
    time.sleep(0.0001) # replace this is with work you need to do
    return x

@ray.remote
def mega_work(start, end):
    return [tiny_work(x) for x in range(start, end)]

start = time.time()
result_ids = []
[result_ids.append(mega_work.remote(x*1000, (x+1)*1000)) for x in range(100)]
results = ray.get(result_ids)
print("duration =", time.time() - start)
```

### 2) 비동기적으로 작업을 요청하고, 결과는 한 번에 동기적으로 받기
```python
import time
import ray

ray.init(num_cpus = 4) # Specify this system has 4 CPUs.

@ray.remote
def do_some_work(x):
    time.sleep(1) # Replace this with work you need to do.
    return x

# -----------------------------------------Bad Usage----------------------------------------------
start = time.time()
bad_results = [ray.get(do_some_work.remote(x)) for x in range(4)]
print("duration =", time.time() - start)
print("results = ", bad_results )

# ----------------------------------------Good Usage----------------------------------------------
start = time.time()
good_results = ray.get([do_some_work.remote(x) for x in range(4)])
print("duration =", time.time() - start)
print("results = ", good_results)
```


### 3) Multi-Actor를 생성하기

```python
import ray

# Start Ray. If you're connecting to an existing cluster, you would use
# ray.init(address=<cluster-address>) instead.
ray.init()

@ray.remote
class Counter(object):
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
        return self.value

# Create ten Counter actors.
counters = [Counter.remote() for _ in range(10)]

# Increment each Counter once and get the results. These tasks all happen in
# parallel.
results = ray.get([c.increment.remote() for c in counters])
print(results)  # prints [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]

# Increment the first Counter five times. These tasks are executed serially
# and share state.
results = ray.get([counters[0].increment.remote() for _ in range(5)])
print(results)  # prints [2, 3, 4, 5, 6]
```


## 5. 적용 코드

### 1) Actor 클래스
*   ray를 import하고, 클래스에 데코레이터를 붙여준다.

```python
from algorithms import aimodel
from common import aicommon
from common import constants as bc
from common.error_code import Errors

import ray

@ray.remote
class Algorithm(aimodel.AIModel):
    def __init__(self, name, config, logger):
        self.config = config
        self.logger = logger

'''
이하 생략
'''

    def predict_by_ray(self, input_param):
        return [self.predict(*x) for x in input_param]


    def predict(self, target, serving_time, sbiz_df, fail_type=None):
        self.logger.debug(f"input data: {target}:\n{serving_time}")

        if target not in self.models:
            self.logger.debug(
                f"return None to module since target {target} not found in model"
            )
            return None

        model = self.models[target]
'''
이하 생략
'''

```


### 2) 호출 클래스
---------

Ray Actor클래스들의 Pool을 만들고, 모듈 초기화 및 모델 로드 Task를 실행한다. 그리고 serving에서 처리할 Task를 호출한다.

```python
'''
생략
'''
from algorithms import Algorithm
import ray

ray.init()

class Analyzer(aimodule.AIModule):
    def __init__(self, config, logger):
        self.config = config
        self.logger = logger

        # Ray Actor 개수 설정
        self.number_of_ray_actor = int(psutil.cpu_count(logical=False) * 0.3)

        # Ray Actor Pool 생성
        self.serving_algorithm_actor_pool = [Algorithm.remote(self.service_id, self.config, self.logger) 
for i in range(self.number_of_ray_actor)]

'''
이하 생략
'''
    def serving(self, header, data_dict):
        # API에서 호출되는 서빙 함수
        self.logger.info("=========== Start Serving ===========")
'''
중간 생략
'''
        # 기존 순차 처리 방식에 필요한 정보를 모은다.
        test_target_id = list(input_df.index.values)[0]
        test_count = 500
        input_param = [(test_target_id, serving_time, df, 'top') for i in range(test_count)]

        # Actor 개수에 맞게 작업들을 나눈다.
        chunk_size, remainder = divmod(len(input_param), self.number_of_ray_actor)
        splitted_input_param = [input_param[i : (i + chunk_size)] for i in range(0, len(input_param), chunk_size)]
        task_per_actor = zip(splitted_input_param, self.serving_algorithm_actor_pool)

        # Task 파라미터에 담아 remote task를 호출한다.
		with TimeLogger("ray serving takes : ", self.logger):
            ray_serving_result = ray.get([acter.predict_by_ray.remote(task) for task, acter in task_per_actor])
            self.logger.info(f"ray serving result : {ray_serving_result}")
```
