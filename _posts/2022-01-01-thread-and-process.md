---
layout: post
title: 스레드와 프로세스
date: 2022-01-01
excerpt: "Thread and Process in Python"
tags: [Concurrency]
comment: true
---


# 개요
회사에서 병렬 프로그래밍을 통해, 모듈의 학습 시간을 50% 정도 감소시킨 경험이 있다. 그 때는 제대로 모르는 상태에서 구글링을 통해 겨우겨우 해결했는데, 성능 이슈에 따른 동시성 프로그래밍은 언제 발생할지 모르므로 관련 공부를 미리미리 해 두고 기록으로 남겨두려고 한다. 주로 에이콘에서 출판한 '파이썬 동시성 프로그래밍(Learning Concurrency in Python)'을 학습해서 정리하였고, '좋은사람' 님의 인프런 동시성 강의 ([링크](https://www.inflearn.com/course/%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%ED%8C%8C%EC%9D%B4%EC%8D%AC-%EC%99%84%EC%84%B1-%EC%9D%B8%ED%94%84%EB%9F%B0-%EC%98%A4%EB%A6%AC%EC%A7%80%EB%84%90))도 참고하였다.


## 포스팅 연재 순서
1. 스레드와 프로세스 개념
2. 스레드 동기화 문제 및 동기화 장치
3. 스레드 사이 통신
4. 동시성 관련  사용법 정리
	- Ray
	- Pathos
	- AsyncIO
	- multiprocessing
	- concurrent.futures


## 스레드
- 스레드는 프로세서가 시간을 할당할 수 있는 최소 단위의 흐름. 정확히 말하면, 운영체제에서 작동되며 스케쥴링될 수 있는 인스트럭션의 순차적인 흐름을 의미한다.

- 스레드의 특징은 다음과 같다.
  - 메모리를 공유한다. 구체적으로, Code, Data, Heap은 서로 공유하며 stack은 독립적이다.
  - 공유 자원 사이에서 서로 상호 작용할 수 있기 때문에, 읽기 및 쓰기가 가능하다. 즉, 한 스레드의 결과가 다른 스레드에 영향을 끼치게 된다.
  - 서로 통신 가능하다

- 스레드의 생명주기는 New, Runnable, Running/Not Running, Dead로 나뉜다.
  - New : 스레드 객체만 생성되고, 스레드 실행에 필요한 자원을 할당받지 못한 상태이다.

  - Runnable : start() 메서드가 호출되면, 스레드는 '실행 가능' 상태에 들어간다. 운영체제로부터 필요한 자원을 할당받고, 작업 스케쥴링이 진행된다.

  - Running : run() 함수가 호출되면 스레드는 '실행' 상태로 바뀌어 작업을 시작하는데, run() 함수는 start() 함수를 호출되면 내부적으로 실행된다. Thread 클래스를 상속해서 커스텀 스레드를 만들면, run 함수를 직접 정의할 수 있다.

  - Not Running : sleep() 함수 등으로 작업이 정지되면 스레드는 '비실행' 상태로 바뀐다.

  - Dead : 작업이 다 완료되면, 스레드는 '정지' 상태에 들어간다.


## 스레드 클래스
스레드 클래스를 어떻게 사용하는지 확인하기 위해, 스레드 클래스 생성자를 확인해보자.
```python
# 생성자
class Thread():
    def __init__(self, group=None, target=None, name=None, args=(), kwargs = None, verbose=None):
        # target : run() 함수에 호출되는 함수. 즉, 스레드가 작업할 내용이 들어간다.
        # name : 스레드 이름
        # args : target 함수에서 받는 인자. 튜플 형태로 넘겨받는다.
        # kwargs : 기본 클래스 생성자를 실행하는 키워드 인자 딕셔너리
```

```python
# 간단한 예시
from threading import Thread

class MyWorkerThread(Thread):
    def __init__(self):
        print("init my thread")
        Thread.__init__(self)

    def run(self): #Thread 클래스을 상속받으면, run 함수를 수정할 수 있다.
        print("Thread is running")

if __name__ == "__main__":
	# main thread가 생성된다.
    myThread = MyWorkerThread() # main thread를 실행하는동안, mtThread가 생성됐다. myThread는 New 상태이다.
    myThread.start() # myThread는 Runnable 상태로 바뀌며, 내부적으로 run이 호출되어 Running이 된다.
    myThread.join() # join()을 호출하면, myThread가 종료될 때까지 myThread를 호출한 main thread는 멈추게 된다.
    # 이후 main thread도 종료되고, 프로그램이 종료된다.
```


## 프로세스
- 프로세스는 실행 중인 프로그램이며, 프로그램을 실행하면 OS로부터 자원을 할당받아 프로세스가 된다.
- 프로세스의 특징은 다음과 같다.

  - 스레드와 다르게 기본적으로 서로 독립되어 있다. 독립적인 주소 공간과 메모리 영역(Code, Data, Stack, Heap)을 가진다

  - 파이프, 파일, 소켓 등을 활용해서 프로세스 간 통신(IPC)할 수 있다. 다만, 비용이 높다.

  - 각 프로세스는 기본적으로 최소 하나의 주-스레드(Main Thread)를 가지지만, 자체의 레지스터와 스택을 가지고 있는 여러 개의 서브스레드를 만들 수도 있다. 프로세스에서 수행하는 프로그램의 코드, 이에 필요한 데이터와 파일은 스레드끼리 서로 공유하며, 레지스터와 스택은 스레드별로 각각 가지게 된다


## 멀티스레딩
- 한 개의 단일 어플리케이션에서 여러 스레드를 구성 후 작업을 처리한다. 스레드가 컨텍스트 스위칭을 하면서 여러 작업을 동시에 처리하는 것이며, 회사에서 직원 한 명이 여러 작업을 변경해가며 처리하는 것과 유사하다.

- 장점
  - Blocking I/O 발생 시 멀티스레딩을 사용하면 작업 속도가 올라간다.
  - 멀티프로세싱에 비해 멀티스레딩 환경에서 메모리를 적게 사용하며, 서로 통신하기 쉽다.

- 단점
  - 자원을 서로 공유하므로, 경합 조건이나 교착 상태 등을 고려해서 코드를 작성해야 한다. 즉, 신경 써야 할 부분들이 많다.
  -  파이썬의 경우 CPU Bounded 작업을 수행할 경우 GIL로 인해 속도 차이가 없거나 오히려 더 늦어질 수 있다.


## 멀티프로세싱
- 한 개의 단일 어플리케이션에서 여러 프로세스를 구성 후 작업을 처리한다. 여러 명의 직원이 각각 다른 일을 처리하는 것과 같다. 멀티스레드와 달리 한 프로세스에서 문제가 생기더라도 다른 프로세스까지 피해가 확산되지 않지만, 그만틈 프로세스 간 통신(IPC)이 필요한 경우에는 스레드 통신보다 비용이 크다.
