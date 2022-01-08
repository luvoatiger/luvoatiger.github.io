---
layout: post
title: 교착 상태와 경합 조건
date: 2022-01-02
excerpt: "Deadlock and Race Condition"
tags: [Concurrency]
comment: true
---


# 개요
스레드는 서로 메모리 영역을 공유하기 때문에, 공유 자원을 동시에 여러 스레드가 접근해서 변경할 수 있다. 이 경우 예상치 못한 문제가 발생하게 된다.

## 교착 상태(Deadlock)
- 스레드가 필요한 공유 자원을 획득하기까지 대기하며, 프로그램이 정상적으로 수행되지 못하는 상태를 말한다.
- 공유 자원은 하나의 스레드만 점유해야 하며 이미 해당 자원을 선점한 스레드가 존재하면, 다른 스레드가 자원을 강제로 가져올 수 없기 때문에 무한 대기하게 되는 것이다.


## 철학자의 저녁식사 문제
- 교착 상태의 대표적인 예시는 철학자의 저녁식사 문제
- 식사 규칙은 다음 두 가지다.
  - 두 손에 포크를 쥐어야 한다.
  - 포크는 한 명의 철학자만 가질 수 있으며, 강제로 뺏을 수 없다. 따라서, 포크를 소유한 철학자가 포크를 내려놓기 전까지 다른 철학자는 대기해야 한다.

```python
import threading
import time
import random

class Philosopher(threading.Thread):
  def __init__(self, name, leftFork, rightFork):
    print("{} Has Sat Down At the Table".format(name))
    threading.Thread.__init__(self, name=name)
    self.leftFork = leftFork
    self.rightFork = rightFork

  def run(self):
    print("{} has started thinking".format(threading.currentThread().getName()))
    while True:
      time.sleep(random.randint(1,5))
      print("{} has finished thinking".format(threading.currentThread().getName()))
      self.leftFork.acquire()
      time.sleep(random.randint(1,5))
      try:
        print("{} has acquired the left fork".format(threading.currentThread().getName()))
 
        self.rightFork.acquire()
        try:
          print("{} has attained both forks, currently eating".format(threading.currentThread().getName()))
        finally:
          self.rightFork.release()
          print("{} has released the right fork".format(threading.currentThread().getName()))
      finally:
        self.leftFork.release()
        print("{} has released the left fork".format(threading.currentThread().getName()))

fork1 = threading.RLock()
fork2 = threading.RLock()
fork3 = threading.RLock()
fork4 = threading.RLock()
fork5 = threading.RLock()

philosopher1 = Philosopher("Kant", fork1, fork2)
philosopher2 = Philosopher("Aristotle", fork2, fork3)
philosopher3 = Philosopher("Spinoza", fork3, fork4)
philosopher4 = Philosopher("Marx", fork4, fork5)
philosopher5 = Philosopher("Russell", fork5, fork1)
 
philosopher1.start()
philosopher2.start()
philosopher3.start()
philosopher4.start()
philosopher5.start()
 
philosopher1.join()
philosopher2.join()
philosopher3.join()
philosopher4.join()
philosopher5.join()

'''
Kant has started thinking
Aristotle has started thinking
Spinoza has started thinking
Marx has started thinking
Russell has started thinking
Aristotle has finished thinking
Russell has finished thinkingSpinoza has finished thinking
Aristotle has acquired leftFork # 포크를 획득한 스레드가 존재
 
 
Marx has finished thinking
Kant has finished thinking
Marx has acquired leftForkSpinoza has acquired leftFork
 
Kant has acquired leftFork
Russell has acquired leftFork # 서로 포크를 획득

이후 포크를 획득하기까지 기다리느라 아무도 식사하지 못함
'''
```

## Deadlock 해결법
- Deadlock 발생 조건을 발생시키지 않으면 된다. 하지만, 시스템 처리량이나 효율성을 떨어뜨리게 된다.
  - 공유 자원을 여러 스레드가 점유할 수 있도록 한다. 하지만, 이 조치는 경합조건이 발생할 빌미를 제공한다.
  - 해당 스레드가 필요한 자원을 획득하고 작업 후 반환할 때까지, 다른 스레드의 작업을 멈춘다. 하지만, 이 경우 순차 처리와 같아질 수 있다.
  - 자원을 선점했더라도, 우선순위가 높은 스레드가 해당 자원을 소유하려고 하면 소유권을 넘겨준다. 소유권이 사라진 스레드는 작업을 정지시킨다.


## 경합 조건(Race Condition)
- 시스템의 출력이 제어할 수 없는 이벤트에 영향을 받는 상태이다. 공유 자원을 여러 스레드가 접근할 때, 어떻게 점유되는지에 따라 결과가 달라지는 현상이다.
- Bonus를 계산하는 간단한 로직인데, 이 프로그램은 연산이 끝나지 않을 수도 있고 연산이 끝나더라도 어떤 값이 나올지 모른다.

```python
import threading
import logging
 
initial_bonus = 0
def increase_bonus():
    global initial_bonus
    while initial_bonus < 1000:
        initial_bonus = initial_bonus + 1
 
def decrease_bonus():
    global initial_bonus
    while initial_bonus >- 1000:
        initial_bonus = initial_bonus - 1
 
def main():   
    formatter = "%(asctime)s : %(message)s"
    logging.basicConfig(format=formatter, level=logging.INFO, datefmt="%H:%M:%S")
    logging.info("Main Thread : before creating thread!")
 
    global initial_bonus
 
    increasing_thread = threading.Thread(target=increase_bonus)
    decreasing_thread = threading.Thread(target=decrease_bonus)   
 
    increasing_thread.start()
    decreasing_thread.start()       
         
    print(initial_bonus)
     
# main scope(python entry point)
if __name__ == "__main__":
    main()
```


## 동기화 프리미티브
- 교착 상태와 경합 조건을 방지하기 위해, 공유 자원 접근을 제한하는 장치들이다.
- 여러 가지가 있는데, 다음 포스팅에 정리할 계획이다. 
