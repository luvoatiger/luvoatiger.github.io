---
layout: post
title: 동시성 라이브러리 pathos를 이용한 학습 병렬 처리
date: 2022-01-07
excerpt: "parallel training with pathos"
tags: [Concurrency]
comment: true
---

# 개요
- 모니터링 시스템 전체를 실시간 이상 탐지하는 모듈(sys-e2e)의 학습 시간을 pathos 라이브러리로 단축시켰던 업무 경험을 정리하는 포스팅

## 학습 과정 구조
- 사용자가 대시보드에서 sys-e2e 모듈을 학습 요청하면, 서버가 학습 프로세스를 실행시킨다.

```python
import parser

'''
중간생략
'''
target_class = aicommon.Utils.get_module_class(
    module_name, class_name, str(Path(py_path) / "analyzer")
)
instance = target_class(param, logger) # 분석 모듈 인스턴스 생성

header = None
body = None
errno = 0
errmsg = None

logger.info("============== Start Training ==============")
logger.info("Train meta info (%s)", param)

try:
    instance.init_training()
    header, body, errno, errmsg = instance.training(stat_logger) # 분석 모듈에 정의된 training 함수 호출

'''
후처리
'''

    instance.end_training() # 학습 종료

except MemoryError as error:
    logger.exception("트레이닝 과정에 MemoryError 오류가 발생하였습니다.", error)
    aicommon.Utils.print_memory_usage(logger)

except:
    logger.exception("트레이닝 과정에 오류가 발생하였습니다.")

logger.info("============== End Training ==============")
```


- sys-e2e 인스턴스 학습 로직은 다음과 같다. 아래는 pseudo code이다.

```python
# 기존 로직
class Analyzer:
    def training(logger):
        def load_data_and_get_target(csv_list) # 데이터 로딩 및 학습 대상 선정
        def algorithms.fit() # 알고리즘 학습
        def save() # 모델 저장

# 신규 로직
class Analyzer:
    def training(logger):
        def load_data_and_get_target(csv_list) # 데이터 로딩 및 학습 대상 선정
        def map_target_and_data() # 학습 대상과 해당 데이터 매핑. 순차처리일 경우, 이 과정에서 각 대상별 학습
        def algorithms.fit() # 여기서는 병렬학습
        def preprocess_result() # 결과 후처리
        def save() # 모델 저장
```

## multiprocessing 트러블슈팅
- 병렬 처리에 많이 사용하는 파이썬 STL인 multiprocessing, concurrent.futures는 사용할 수 없다.

  - 해당 Analyzer 파일을 직접 실행시키면 이상없이 작동하지만, 인스턴스를 생성하고 인스턴스 메서드를 호출하는 구조에서는 에러가 발생한다. 즉, name이 main이면 정상 동작하지만 그렇지 않으면 에러가 발생한다.

  - 이는 윈도우와 리눅스의 자식프로세스 생성 과정이 다르기 때문에, 멀티프로세싱 라이브러리에서는 entry-point(프로그램 시작지점)을 보호하도록 제약을 두었기 때문이다.
    - 리눅스
      - 파이썬 코드를 실행하다가 pool.map함수 등 병렬처리 작업을 만나게 되면 내부적으로 fork() 메서드를 사용해서 자식프로세스 생성한다.
      - fork()를 하면, 부모프로세스를 그대로 에서 지금까지 처리했던 변수값이나 내용들이 자식프로세스도 그대로 접근 및 사용, 변경 가능하다.
      - 따라서 자식프로세스는 부모프로세스에서 지금까지 처리한 걸 바탕으로, 작업을 다른 프로세스(부모프로세스 포함)들과 나눠서 처리한다.

    - 윈도우
      - 자식프로세스를 생성할 때, fork()를 사용할 수 없다고 한다.
      - 자식프로세스에서는 부모프로세스에서 처리했던 값들을 이용할 수 없기에 부모프로세스가 처리하는 모듈의 코드를 그대로 import해서 처음부터 다시 처리해야 한다는 뜻이다.
      - 새롭게 생성된 자식프로세스가 처음부터 코드를 실행하다가 다시 프로세스를 생성하는 코드를 만나게 되면 새로운 손자프로세스를 생성하게 된다. 손자프로세스는 처음부터 코드를 실행하다가 증손자 프로세스를 생성하게 되는 무한 재귀 방식이 되므로, 프로그램 시작지점을 보호해야 한다.

  - TypeError : can't pickle thread.RLockObjects 에러가 발생한다.
    - multiprocessing 라이브러리에서는 내부적으로 pickle 라이브러리를 사용해서 serializing을 하는데, 클래스 내부에서 선언한 인스턴스의 메서드는 pickle할 수 없다. pickle 대신 dill을 사용하거나, 커스텀 메서드를 만들면 pickle에서 인스턴스 메서드도 serializing 가능하지만 공수가 만만치 않을 것으로 보여 다른 대안을 조사하기 시작했다.

## pathos 라이브러리
- 대안을 조사한 결과 발견한 라이브러리이다.


- pickle 대신 dill 라이브러리를 사용해서 serializing을 하므로 인스턴스 메서드도 실행 가능하며, main에서 실행하지 않고도 병렬 처리 할 수 있도록 multiprocessing 모듈을 수정해서 포팅했다고 한다.


- 위의 이슈는 해결됐으나, 두 가지 이슈가 추가로 발생해서 다시 트러블슈팅을 진행했다.

  - 전역 변수에 등록된 로거(self.logger)를 동시에 참조하다가 Lock이 걸리는 현상이 발생했다.
    - self.logger가 아닌 pathos에서 제공하는 processLogger를 사용해서 해결하였다.
  - 여러 프로세스가 컨텍스트 스위칭을 하면서 한 번에 하나의 프로세스가 대상 하나를 처리하며 순차 처리보다 느려지는 현상이 발생했다.
    - chunksize를 명시적으로 전달하도록 수정하였다.

## 성능 테스트 비교 결과
- 3000개의 대상에 대해서 테스트 한 결과이다. 절반 정도로 학습 시간이 줄어든 것을 확인하였다.

||2-process|3-process|4-process|
|--|------|---|---|
|순차처리|5700초|5700초|5700초|
|병렬처리|3000초|2147초|1680초|

- 자식프로세스도 정상적으로 떴으며, 각 프로세스가 하나의 코어를 사용한다.
![코어 사용량](/imgs/core.png)


## 적용 코드
```python
import numpy as np
import pandas as pd
import pathos
import psutil
# 사용 가능한 코어 개수 조절
current_process = psutil.Process()
current_process.cpu_affinity(list(range(psutil.cpu_count(logical=False))))

# 중간생략
class Analyzer(aimodule.AIModule):
    def __init__(self, config, logger):
    	# 설정값, 로거 입력
        self.config = config
        self.logger = logge

        # 병렬처리 정보
        self.number_of_child_processes = int(psutil.cpu_count(logical=False) * 0.3)

        # 분석 모델 인스턴스 생성
        self.all_algorithm = AllAlorithm(self.service_id, self.config, self.logger)
        self.top_algorithm = TopAlgorithm(self.service_id, self.config, self.logger)

    def training(self, train_logger):
        self.logger.info(f"module {self.service_id} start training")

		df_all, train_target_xcodes = self.load_data_and_get_target(csv_list)
        
        train_prog.log("process train data", 100)

        dbsln_result, body, code, message = self.all_algorithm.fit(df_all)

        xcode_df_mapper = {}
        is_multiprocessing_mode = True if self.number_of_child_processes >= 2 else False

        total_train_time = 0

        for i in range(len(target)):
            xcode = target[i]

            self.logger.info(f"[training] preprocessing data of target no.{xcode}")
            if len(df_1day_all) > 0:
                df_xcd = df_1day_all.query(f"{self.target_id_field} == '{xcode}'")
                df_xcd = df_xcd.set_index("time")
                df_xcd = df_xcd.interpolate()

                if not is_multiprocessing_mode:
                    result = self.top_algorithm.fit(xcode, df_xcd, multiprocessing=False) # 순차 처리

                xcode_df_mapper[xcode] = df_xcd # 타겟과 데이터 매핑

        if is_multiprocessing_mode:
            pool = pathos.multiprocessing.Pool(processes=self.number_of_child_processes) # 병렬 처리를 위한 pool 생성

            input_iterable = [(key, value) for key, value in xcode_df_mapper.items()] # input_parameter 만들기
            chunk_size, remainder = divmod(len(input_iterable), self.number_of_child_processes)
            if remainder != 0:
                chunk_size += 1


            txn_model = zip(*pool.starmap(self.top_algorithm.fit, input_iterable, chunksize=chunk_size)) # 병렬 학습

            pool.close()
            pool.join())

            models = {}
            for element in txn_models:
                if element is None:
                    continue
                else:
                    for xcode in list(element.keys()):
                        models[xcode] = element[xcode]

            self.logger.info(
                "[training] finish postprocessing  multiprocessed model"
            )

        # save_model
        self._save_model()

        result = {"all": all_result, "top": {}}

        return result, train_target_xcodes, 0, None

```

