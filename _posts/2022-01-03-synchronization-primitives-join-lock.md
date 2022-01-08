---
layout: post
title: 동기화 프리미티브 (1) join, Lock, RLock
date: 2022-01-03
excerpt: "Join and Lock"
tags: [Concurrency]
comment: true
---


# 동기화 프리미티브
경합 조건을 방지하기 위해, 스레드 사이의 공유 자원 접근을 제한하는 장치들을 의미한다. 여러 가지가 있는데, 하나씩 정리해보려고 한다.

## Join 메서드
- join이 호출된 스레드가 종료되기까지, join을 호출한 스레드는 작업을 멈추게 된다.
- 같은 sub thread 2개를 만들고, 하나는 join을 호출하고 다른 하나는 join을 호출하지 않았을 때 어떤 순서로 Thread가 실행되는지 확인하면 된다.

```python
import threading
import time
 
def execute_by_fisrtthread(name):
    print(f"{name} start job")
    time.sleep(15)
    print(f"{name} finish job")
 
def execute_by_secondthread(name):
    print(f"{name} start job")
    time.sleep(5)
    print(f"{name} finish job")
     
if __name__ == "__main__":
    print(f"{threading.currentThread().getName()} started")
    sub_thread_1 = threading.Thread(target=execute_by_fisrtthread, args=("first",))
    sub_thread_1.start()
    print(f"Is first Finished?")
    sub_thread_2 = threading.Thread(target=execute_by_secondthread, args=("second",))
    sub_thread_2.start()
    sub_thread_2.join()
     
    print(f"Is second Finished?")
    print(f"{threading.currentThread().getName()} finished")
 
'''
MainThread started
first start job
Is first Finished?
second start job
second finish job # join() 걸린 second thread는 5초 동안 sleep 된 다음에, Main Thread로 순서가 넘어간다.
Is second Finished?
MainThread finished
first finish job # join() 걸리지 않은 first thread는 15초를 sleep하는데 그 사이에 Main과 Second는 전부 자신의 일을 끝냈다.
'''
```

## Lock
- 락의 기본값은 1이며, 특정 Thread가 락 획득(acquire)을 요청하면 값을 0으로 바꾸고 락을 획득한 쓰레드가 락 해제(release)를 호출하기 전까지 다른 쓰레드의 접근을 막는다.
- 락 해제를 호출하면, 값을 1로 바꾸고 다른 Thread가 락을 획득할 수 있도록 제어한다.
- 락 객체에 정의된 주요 메서드는 다음 세 가지이다.
  - acqurie(blocking=True, timeout=-1)
  - release()
  - locked()


```python
import threading
import time
 
class MyWorker():
    def __init__(self):
        self.a = 1
        self.b = 2
        self.c = 3
        self.RLock = threading.Lock()
 
    def modifyAinA(self):
        with self.RLock:
            print(f"[modifyAinA] Lock acquired? {self.RLock.locked()}")
            print(f"[modifyAinA] {self.RLock}")
            self.a = self.a + 1
            time.sleep(5)
        print(f"[modifyAinA] Lock acquired? {self.RLock.locked()}")
        print(f"[modifyAinA] {self.RLock}")
         
    def modifyA(self):
        with self.RLock:
            print(f"[modifyA] Lock acquired? {self.RLock.locked()}")
            print(f"[modifyA] {self.RLock}")
            self.modifyAinA()
            self.a = self.a + 1
            time.sleep(5)
        print(f"[modifyA] Lock acquired? {self.RLock.locked()}")
        print(f"[modifyA] {self.RLock}")
             
    def modifyB(self):
        with self.RLock:
            print(f"[modifyB] Lock acquired? {self.RLock.locked()}")
            print(f"[modifyB] {self.RLock}")
            self.b = self.b - 1
            time.sleep(5)
        print(f"[modifyB] RLock acquired? {self.RLock.locked()}")
        print(f"[modifyB] {self.RLock}")
             
    def modifyBoth(self):
        with self.RLock:
            print(f"[modifyBoth] Lock acquired? {self.RLock.locked()}")
            print(f"[modifyBoth] {self.RLock}")
            self.modifyA()
            self.modifyB()
        print(f"[modifyBoth] Lock acquired? {self.RLock.locked()}")
        print(f"[modifyBoth] {self.RLock}")
             
if __name__ == "__main__":
    workerA = MyWorker()
    workerA.modifyBoth()
'''
[modifyBoth] RLock acquired? True
[modifyBoth] <locked _thread.lock object at 0x000001D32E07A210>
이후 modifyA() 함수의 컨텍스트 매니저(with self.Rlock)에서 Deadlock이 걸리게 된다.
'''
```

## RLock
- Lock을 호출한 함수가 자기 자신을 재귀적으로 호출할 경우에 사용한다.
- 해당 함수가 Lock을 획득(acquire)하고, release하기 전에 자기 자신을 다시 호출한다고 가정해보자. 계속해서 Lock을 획득(acquire)하려고 할 텐데, Lock은 한 번만 획득할 수 있으므로 해당 쓰레드는 멈추게 될 것이다.
- 이럴 때 RLock을 사용하면 Lock을 계속 획득(acquire)해서 작업을 처리할 수 있다. RLock을 획득한 횟수만큼 RLock 내부 Counter가 증가한다. 다만, 획득한 횟수만큼 락 해제(release)를 호출해야한다.
- 주요 메서드는 다음 2가지이다.
  - acquire(blocking=True, timeout=-1))
  - release()

```python
import threading
import time
 
class MyWorker():
    def __init__(self):
        self.a = 1
        self.b = 2
        self.c = 3
        self.RLock = threading.RLock()
 
    def modifyAinA(self):
        with self.RLock:
            print(f"[modifyAinA] RLock acquired? {self.RLock._is_owned()}")
            print(f"[modifyAinA] {self.RLock}")
            self.a = self.a + 1
            time.sleep(5)
        print(f"[modifyAinA] RLock acquired? {self.RLock._is_owned()}")
        print(f"[modifyAinA] {self.RLock}")
         
    def modifyA(self):
        with self.RLock:
            print(f"[modifyA] RLock acquired? {self.RLock._is_owned()}")
            print(f"[modifyA] {self.RLock}")
            self.modifyAinA()
            self.a = self.a + 1
            time.sleep(5)
        print(f"[modifyA] RLock acquired? {self.RLock._is_owned()}")
        print(f"[modifyA] {self.RLock}")
             
    def modifyB(self):
        with self.RLock:
            print(f"[modifyB] RLock acquired? {self.RLock._is_owned()}")
            print(f"[modifyB] {self.RLock}")
            self.b = self.b - 1
            time.sleep(5)
        print(f"[modifyB] RLock acquired? {self.RLock._is_owned()}")
        print(f"[modifyB] {self.RLock}")
             
    def modifyBoth(self):
        with self.RLock:
            print(f"[modifyBoth] RLock acquired? {self.RLock._is_owned()}")
            print(f"[modifyBoth] {self.RLock}")
            self.modifyA()
            self.modifyB()
        print(f"[modifyBoth] RLock acquired? {self.RLock._is_owned()}")
        print(f"[modifyBoth] {self.RLock}")
             
if __name__ == "__main__":
    workerA = MyWorker()
    workerA.modifyBoth()
 
'''
[modifyBoth] RLock acquired? True
[modifyBoth] <locked _thread.RLock object owner=20632 count=1 at 0x000001D32DC7E840> #count가 계속 증가한다.
[modifyA] RLock acquired? True
[modifyA] <locked _thread.RLock object owner=20632 count=2 at 0x000001D32DC7E840>   #count가 계속 증가한다. 이전과 다르게 Deadlock이 걸리지 않았따.
[modifyAinA] RLock acquired? True
[modifyAinA] <locked _thread.RLock object owner=20632 count=3 at 0x000001D32DC7E840>
[modifyAinA] RLock acquired? True
[modifyAinA] <locked _thread.RLock object owner=20632 count=2 at 0x000001D32DC7E840>
[modifyA] RLock acquired? True                                                      #with문에서는 Lock을 반환했으나, count가 0이 아니어서 계속 획득된 상태로 표시
[modifyA] <locked _thread.RLock object owner=20632 count=1 at 0x000001D32DC7E840>
[modifyB] RLock acquired? True
[modifyB] <locked _thread.RLock object owner=20632 count=2 at 0x000001D32DC7E840>
[modifyB] RLock acquired? True
[modifyB] <locked _thread.RLock object owner=20632 count=1 at 0x000001D32DC7E840>
[modifyBoth] RLock acquired? False                                                  #count가 0이 되어야 lock 획득을 False로 처리
[modifyBoth] <unlocked _thread.RLock object owner=0 count=0 at 0x000001D32DC7E840>  #count가 0이 되어야 unlock으로 처리
'''
```
