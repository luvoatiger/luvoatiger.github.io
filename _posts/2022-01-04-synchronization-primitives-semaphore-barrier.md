---
layout: post
title: 동기화 프리미티브 (2) Semaphore, Barrier
date: 2022-01-04
excerpt: "Semaphore and Barrier"
tags: [Concurrency]
comment: true
---


## Semaphore
- Lock처럼 내부의 카운터를 가지고 있으면서, 하나의 스레드가 아니라 n개의 Thread만 공유 자원에 접근할 수 있도록 동기화하는 장치이다.
- 초기값은 Semaphore를 정의할 때, 입력하는 값에 따라 counter가 결정된다.
- counter가 0이 되면 다른 thread가 작업을 마치고 release하지 않는 이상, semaphore를 획득하지 못한 Thread는 Blocking 되어 코드 실행이 정지된다.

```python
import threading
import time
import logging
import random
 
class TicketSeller(threading.Thread):
    ticketSold = 0
    def __init__(self, semaphore):
        threading.Thread.__init__(self)
        self.semaphore = semaphore
        print(f"[{threading.currentThread().getName()}] semaphore value in __init__ : {self.semaphore._value}")
 
    def run(self):
        global ticketsAvailable
        running = True
        while running:
            time.sleep(random.randint(0, 1))
            print(f"[{threading.currentThread().getName()}] acquire {self.semaphore.acquire()}")
            print(f"[{threading.currentThread().getName()}] semaphore value in run : {self.semaphore._value}")
            if ticketsAvailable <= 0:
                running = False
            else:
                TicketSeller.ticketSold += 1
                ticketsAvailable -= 1
            print(f"[{threading.currentThread().getName()}] release {self.semaphore.release()}")
            print(f"[{threading.currentThread().getName()}] semaphore value in run : {self.semaphore._value}")
                 
if __name__ == "__main__":
    semaphore = threading.Semaphore(3)
    ticketsAvailable = 10
    print(f"[{threading.currentThread().getName()}] semaphore value in main : {semaphore._value}")
     
    ticketSellers = []
    for i in range(4):
        seller = TicketSeller(semaphore)
        seller.start()
        ticketSellers.append(seller)
         
    for seller in ticketSellers:
        seller.join()
 
'''
[MainThread] semaphore value in main : 3
[MainThread] semaphore value in __init__ : 3 # semaphore 내부 counter는 3
[Thread-30] acquire True
[MainThread] semaphore value in __init__ : 2 # 획득하면 counter는 줄어든다.
 
[Thread-30] semaphore value in run : 2 
[Thread-31] acquire True
[Thread-31] semaphore value in run : 1
[Thread-31] release None
[Thread-31] semaphore value in run : 2 # counter를 반환하면 다시 증가
 
[Thread-30] release None
[MainThread] semaphore value in __init__ : 3
[Thread-30] semaphore value in run : 3
 
[MainThread] semaphore value in __init__ : 3
[Thread-33] acquire True
[Thread-33] semaphore value in run : 2
[Thread-33] release None
[Thread-33] semaphore value in run : 3
[Thread-31] acquire True
[Thread-32] acquire True
[Thread-32] semaphore value in run : 1
[Thread-32] release None
[Thread-32] semaphore value in run : 2
[Thread-30] acquire True
[Thread-33] acquire True
[Thread-30] semaphore value in run : 0
[Thread-30] release None
[Thread-30] semaphore value in run : 1
[Thread-31] semaphore value in run : 1
[Thread-31] release None
[Thread-31] semaphore value in run : 2
 
[Thread-33] semaphore value in run : 2
[Thread-33] release None
[Thread-33] semaphore value in run : 3
[Thread-32] acquire True
[Thread-31] acquire True
[Thread-32] semaphore value in run : 1
[Thread-30] acquire True
[Thread-31] semaphore value in run : 0 # 3개의 Thread가 semaphore를 획득
[Thread-31] release None
[Thread-31] semaphore value in run : 1
 
[Thread-30] semaphore value in run : 1
[Thread-33] acquire True
[Thread-32] release None
[Thread-32] semaphore value in run : 1
 
[Thread-30] release None
[Thread-33] semaphore value in run : 2
[Thread-33] release None
[Thread-33] semaphore value in run : 3
 
[Thread-30] semaphore value in run : 3
[Thread-30] acquire True
[Thread-32] acquire True
[Thread-31] acquire True
[Thread-32] semaphore value in run : 0
[Thread-32] release None
[Thread-32] semaphore value in run : 1
 
[Thread-31] semaphore value in run : 1
[Thread-31] release None
[Thread-31] semaphore value in run : 2
 
[Thread-30] semaphore value in run : 2
[Thread-30] release None
[Thread-30] semaphore value in run : 3
'''
```

## Barrier
- Barrier는 장벽처럼 스레드를 대기시키다가 정해진 개수 이상의 스레드가 대기하게 되면, 무너지는 출발선이라고 생각하면 된다.
- 스레드가 Barrier에서 점점 대기하다가 n개의 스레드가 대기하게 되면, 자동적으로 장벽(Barrier)가 해제된다.
- Barrier 객체에 정의된 주요 메서드는 다음 세 가지이다.
  - wait()
  - reset()
  - abort()
- Barrier 객체에 정의된 주요 변수는 다음 세 가지이다.
  - n_waiting
  - parties
  - broken


```python
import threading
import time
import random
 
class myThread(threading.Thread):
    def __init__(self, barrier):
        threading.Thread.__init__(self)
        self.barrier = barrier
        print(f"[{threading.currentThread().getName()}] barrier can accept {self.barrier.parties} threads")
     
    def run(self):
        print(f"[{threading.currentThread().getName()}] run")
        time.sleep(random.randint(1, 10))
        print(f"[{threading.currentThread().getName()}] joining {self.barrier.n_waiting + 1} on Barrier")
        print(f"[{threading.currentThread().getName()}] Is the barrier broken? {self.barrier.broken}")
        self.barrier.wait()
        print(f"[{threading.currentThread().getName()}] Is the barrier broken? {self.barrier.broken}")
        print(f"[{threading.currentThread().getName()}] The barrier has been lifted, continuing with work")
         
if __name__ == "__main__":
    barrier = threading.Barrier(4)
    threads = []
    print(f"[{threading.currentThread().getName()}] barrier in __main__ : {barrier}")
     
    for i in range(4):
        thread = myThread(barrier)
        thread.start()
        threads.append(thread)
         
    for thread in threads:
        thread.join()
 
'''
[MainThread] barrier in __main__ : <threading.Barrier object at 0x0000028FC2D4B730>
[MainThread] barrier can accept 4 threads                       # Barrier에는 총 4개의 쓰레드가 대기할 수 있다.
[Thread-78] run
[MainThread] barrier can accept 4 threads
[Thread-79] run
[MainThread] barrier can accept 4 threads
[Thread-80] run
[MainThread] barrier can accept 4 threads
[Thread-81] run
[Thread-79] Is the barrier broken? False
[Thread-80] Is the barrier broken? False
[Thread-79] joining 1 on Barrier
[Thread-80] joining 2 on Barrier
[Thread-81] Is the barrier broken? False
[Thread-81] joining 3 on Barrier
[Thread-78] Is the barrier broken? False
[Thread-78] joining 4 on Barrier                               # 총 4개의 쓰레드가 도달
[Thread-78] The barrier has been lifted, continuing with work  # 장벽 해제(Barrier Lifted)되며, 4개의 쓰레드가 동시에 작업하게 된다.
[Thread-80] The barrier has been lifted, continuing with work
[Thread-81] The barrier has been lifted, continuing with work
[Thread-79] The barrier has been lifted, continuing with work
'''
```
