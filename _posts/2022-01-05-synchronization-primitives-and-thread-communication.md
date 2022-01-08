---
layout: post
title: 동기화 프리미티브 Event와 Condition을 이용한 스레드 통신
date: 2022-01-05
excerpt: "Thread communication using Event and Condition"
tags: [Concurrency]
comment: true
---

# 개요
- 동기화 프리미티브인 Event, Condition을 소개하며 생산자-소비자 패턴을 구현해본다. 생산자 스레드들과 소비자 스레드들 사이의 통신은 Queue를 이용한다.


## Event
- 동시적으로 실행되는 여러 스레드 사이의 간단한 통신에 사용하기 좋다. 여러 쓰레드 사이에 순서를 보장할 때, 사용하면 된다.
- wait()를 호출한 스레드는 대기하고 있다가, wait() 걸리지 않은 다른 스레드에서 작업을 마치고 Event.Set()이라는 시그널을 주면 작업을 시작하게 된다. 일종의 달리기 출발선과 비슷하다.
- 내부적으로 Flag가 있으며, 초기값은 0이다.
- 주요 메서드는 다음 4가지이다.
  - set() : Flag를 1로 바꾼다.
  - clear() : Flag를 0으로 바꾼다.
  - wait() : Flag가 1일 때는 리턴, 0이면 대기
  - isSet() : 현재 Flag 상태를 그대로 리턴한다.

```python
import threading
import time
 
def firstThread(event):
    print(f"[{threading.currentThread().getName()}] do job(sleeping 5 seconds) before setting Event")
    time.sleep(5)
    print(f"[{threading.currentThread().getName()}] has the event been set? {event.isSet()}")
    print(f"[{threading.currentThread().getName()}] set Event")
    event.set()
    print(f"[{threading.currentThread().getName()}] has the event been set? {event.isSet()}")
    print(f"[{threading.currentThread().getName()}] the event has been set")
    print(f"[{threading.currentThread().getName()}] finish job")
 
def secondThread(event):
    print(f"[{threading.currentThread().getName()}] waiting for the event to be set")
    event.wait()
    print(f"[{threading.currentThread().getName()}] has the event been set? {event.isSet()}")
    print(f"[{threading.currentThread().getName()}] do job(sleep 5 seconds)")
    time.sleep(5)
    print(f"[{threading.currentThread().getName()}] finish job")
 
def thirdThread(event):
    print(f"[{threading.currentThread().getName()}] waiting for the event to be set")
    event.wait()
    print(f"[{threading.currentThread().getName()}] has the event been set? {event.isSet()}")
    print(f"[{threading.currentThread().getName()}] do job(sleep 5 seconds)")
    time.sleep(5)
    print(f"[{threading.currentThread().getName()}] finish job")
 
def main():
    myEvent = threading.Event()
    print(f"[{threading.currentThread().getName()}] Event Object : {myEvent}")
    print(f"[{threading.currentThread().getName()}] Is the event set? : {myEvent.isSet()}")
    thread1 = threading.Thread(target=firstThread, args=(myEvent,))
    thread2 = threading.Thread(target=secondThread, args=(myEvent,))
    thread3 = threading.Thread(target=thirdThread, args=(myEvent,))
     
    print(f"[{threading.currentThread().getName()}] start Thread {thread1.getName()}")
    print(f"[{threading.currentThread().getName()}] start Thread {thread2.getName()}")
    print(f"[{threading.currentThread().getName()}] start Thread {thread3.getName()}")
    thread1.start()
    thread2.start()
    thread3.start()
     
    print(f"[{threading.currentThread().getName()}] sleep 20 seconds")
    time.sleep(20)
    print(f"[{threading.currentThread().getName()}] finish job")
 
if __name__ == "__main__":
    main()
 
'''
[MainThread] Event Object : <threading.Event object at 0x000001528AD9BBB0>
[MainThread] Is the event set? : False
[MainThread] start Thread Thread-34
[MainThread] start Thread Thread-35
[MainThread] start Thread Thread-36
[Thread-34] do job(sleeping 5 seconds) before setting Event # 34번 쓰레드는 wait()없이 작업을 먼저 진행하며, 작업이 끝나면 Event를 setting한다.
[Thread-35] waiting for the event to be set #35번 쓰레드는 wait() 함수를 만났으므로, Event가 Set되기 전까지 작업을 멈췄다.
[Thread-36] waiting for the event to be set #36번 쓰레드도 wait() 함수를 만났으므로, Event가 Set되기 전까지 작업을 멈춘다.
[MainThread] sleep 20 seconds
[Thread-34] has the event been set? False #34번 쓰레드 작업 중
[Thread-34] set Event                     #34번 쓰레드에서 Event Set
[Thread-34] has the event been set? True
[Thread-34] the event has been set
[Thread-34] finish job                    #34번 쓰레드 작업 완료
[Thread-36] has the event been set? True
[Thread-35] has the event been set? True
[Thread-36] do job(sleep 5 seconds)      # Event가 Set되고 나서 36번 쓰레드 작업 시작.
 
[Thread-35] do job(sleep 5 seconds)      # Event가 Set되고 나서 35번 쓰레드 작업 시작
[Thread-35] finish job
 
[Thread-36] finish job
 
[MainThread] finish job
'''
```


## Condition
- 같은 역할을 하는 여러 스레드끼리 하나의 자원을 경합(acquire)한 다음 이긴 스레드가 작업을 마치고(release) 알림을(notify) 주면, 다른 역할을 하는 여러 쓰레드가 알림을 대기(wait)하다가 서로 경합해서(acquire) 이긴 스레드가 작업을 끝마치게 된다(release).
- 이벤트 객체와 락의 혼합으로 이해하면 된다.
- condition 객체의 주요 메서드는 다음과 같다.
  - acquire(*kwargs)
  - release()
  - wait(timeout=None)
  - wait_for(predicate, timeout=None)
  - notify(n=1)
  - notify_all()

## Condition을 이용한 생산자-소비자 패턴
- Event를 이용해서도 구현할 수도 있다.


```python
import threading
import time
import queue
import random
 
class Producer(threading.Thread):
    def __init__(self, pipeline, condition):
        self.pipeline = pipeline
        self.condition = condition
        threading.Thread.__init__(self)
         
    def run(self):
        while True:
            self.condition.acquire()
            print(f"[Producer {threading.currentThread().getName()}] acquired condition")   
            message = random.randint(1, 10)
            self.pipeline.put(message)
            print(f"[Producer {threading.currentThread().getName()}] put message {message}")   
            self.condition.notify()
            print(f"[Producer {threading.currentThread().getName()}] notify condition")   
            self.condition.release()
            print(f"[Producer {threading.currentThread().getName()}] release condition")   
            time.sleep(30)
            print(f"[Producer {threading.currentThread().getName()}] finish run after waiting for 1 seconds")   
             
class Consumer(threading.Thread):
    def __init__(self, pipeline, condition):
        self.pipeline = pipeline
        self.condition = condition
        threading.Thread.__init__(self)
         
    def run(self):
        while True:
            self.condition.acquire()
            print(f"[Consumer {threading.currentThread().getName()}] acquired condition")   
            while True:
                if not self.pipeline.empty():
                    message = self.pipeline.get()
                    print(f"[Consumer {threading.currentThread().getName()}] get message {message}")   
                    break
                print(f"[Consumer {threading.currentThread().getName()}] wait condition")   
                self.condition.wait()
            print(f"[Consumer {threading.currentThread().getName()}] release condition")   
            self.condition.release()       
            print(f"[Consumer {threading.currentThread().getName()}] finish run")   
     
def main():
    pipeline_queue = queue.Queue(maxsize=10000)
    condition = threading.Condition()
    print(f"[{threading.currentThread().getName()}] condition object : {condition}")
    producer_thread_1 = Producer(pipeline_queue, condition)
    print(f"[{threading.currentThread().getName()}] create producer_thread_1 : {producer_thread_1.getName()}")
    producer_thread_2 = Producer(pipeline_queue, condition)
    print(f"[{threading.currentThread().getName()}] create producer_thread_2 : {producer_thread_2.getName()}")
    producer_thread_3 = Producer(pipeline_queue, condition)
    print(f"[{threading.currentThread().getName()}] create producer_thread_3 : {producer_thread_3.getName()}")
    consumer_thread_1 = Consumer(pipeline_queue, condition)
    print(f"[{threading.currentThread().getName()}] create consumer_thread_1 : {consumer_thread_1.getName()}")
    consumer_thread_2 = Consumer(pipeline_queue, condition)
    print(f"[{threading.currentThread().getName()}] create consumer_thread_2 : {consumer_thread_2.getName()}")
 
    print(f"[{threading.currentThread().getName()}] start producer_thread_1")   
    producer_thread_1.start()
    print(f"[{threading.currentThread().getName()}] start producer_thread_2")   
    producer_thread_2.start()
    print(f"[{threading.currentThread().getName()}] start producer_thread_3")   
    producer_thread_3.start()
    print(f"[{threading.currentThread().getName()}] start consumer_thread_1")   
    consumer_thread_1.start()
    print(f"[{threading.currentThread().getName()}] start consumer_thread_2")   
    consumer_thread_2.start()
 
    print(f"[{threading.currentThread().getName()}] join all")       
    producer_thread_1.join()
    producer_thread_2.join()
    producer_thread_3.join()
    consumer_thread_1.join()
    consumer_thread_2.join()
 
if __name__ == "__main__":
    main()
'''
[MainThread] condition object : <Condition(<unlocked _thread.RLock object owner=0 count=0 at 0x00000286DB87CAE0>, 0)>
[MainThread] create producer_thread_1 : Thread-7
[MainThread] create producer_thread_2 : Thread-8
[MainThread] create producer_thread_3 : Thread-9
[MainThread] create consumer_thread_1 : Thread-10
[MainThread] create consumer_thread_2 : Thread-11
[MainThread] start producer_thread_1
[Producer Thread-7] acquired condition
[Producer Thread-7] put message 4
[Producer Thread-7] notify condition
[Producer Thread-7] release condition
[MainThread] start producer_thread_2
[Producer Thread-8] acquired condition
[MainThread] start producer_thread_3
[Producer Thread-8] put message 10
[Producer Thread-8] notify condition
[Producer Thread-8] release condition
[Producer Thread-9] acquired condition
[Producer Thread-9] put message 2
[Producer Thread-9] notify condition
[Producer Thread-9] release condition
[MainThread] start consumer_thread_1
[Consumer Thread-10] acquired condition
[Consumer Thread-10] get message 4
[MainThread] start consumer_thread_2
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] get message 10
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[MainThread] join all                    # 여기서부터 main은 작업을 멈추고, 경합하게 놔 둔다.
[Consumer Thread-10] get message 2
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] wait condition
[Consumer Thread-11] acquired condition
[Consumer Thread-11] wait condition
[Producer Thread-7] acquired condition  # 경합. Thread-7이 queue를 획득해 message 입력
[Producer Thread-7] put message 5
[Producer Thread-7] notify condition
[Producer Thread-7] release condition
[Producer Thread-9] acquired condition
[Producer Thread-9] put message 6
[Producer Thread-9] notify condition
[Producer Thread-9] release condition
[Producer Thread-8] acquired condition
[Producer Thread-8] put message 2
[Producer Thread-8] notify condition
[Producer Thread-8] release condition
[Consumer Thread-10] get message 5     # 경합. Thread-10이 queue를 획득해 queue에 있던 모든 message 처리
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] get message 6
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] get message 2
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] wait condition    # 처리 완료 후 consumer는 알림 대기
[Consumer Thread-11] wait condition    # 처리 완료 후 consumer는 알림 대기
[Producer Thread-7] acquired condition
[Producer Thread-7] put message 5
[Producer Thread-7] notify condition
[Producer Thread-7] release condition
[Consumer Thread-10] get message 5
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] wait condition
[Producer Thread-8] acquired condition # producer가 입력 후 알림
[Producer Thread-8] put message 3
[Producer Thread-8] notify condition
[Producer Thread-8] release condition
[Consumer Thread-11] get message 3     # 알림 받고 즉시, consumer가 처리 
[Consumer Thread-11] release condition
[Consumer Thread-11] acquired condition
[Consumer Thread-11] wait condition
[Producer Thread-9] acquired condition
[Producer Thread-9] put message 6
[Producer Thread-9] notify condition
[Producer Thread-9] release condition
[Consumer Thread-10] get message 6
[Consumer Thread-10] release condition
[Consumer Thread-10] acquired condition
[Consumer Thread-10] wait condition
[Producer Thread-7] acquired condition   # 경합. Thread-7이 queue를 획득해서 message를 입력
[Producer Thread-7] put message 2
[Producer Thread-7] notify condition
[Producer Thread-7] release condition
[Producer Thread-9] acquired condition   # 경합. 이번에는 Thread-8이 아닌 Thread-9가 queue를 획득해서 message를 입력
[Producer Thread-9] put message 4
[Producer Thread-9] notify condition
[Producer Thread-9] release condition
[Producer Thread-8] acquired condition  # 경합. Thread-8이 queue를 획득해서 message를 입력
[Producer Thread-8] put message 7
[Producer Thread-8] notify condition
[Producer Thread-8] release condition
[Consumer Thread-11] get message 2     # 경합. 이번엔 Thread-11이 queue를 획득해 queue에 있던 모든 message 처리
[Consumer Thread-11] release condition
[Consumer Thread-11] acquired condition
[Consumer Thread-11] get message 4
[Consumer Thread-11] release condition
[Consumer Thread-11] acquired condition
[Consumer Thread-11] get message 7
[Consumer Thread-11] release condition
[Consumer Thread-11] acquired condition
[Consumer Thread-11] wait condition
[Consumer Thread-10] wait condition
'''
```
