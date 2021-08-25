---
title: "EventLoop 이해하기"
layout: post
comments: true
---

Event Loop(aka. Main Loop)은 싱글 쓰레드 기반 어플리케이션에서 주로 사용되는 어플리케이션 동작 모델입니다.
대부분의 어플리케이션은 프로그램 생명 주기 동안 외부의 이벤트에 대기하고 발생된 이벤트를 처리하는 방식으로 동작합니다.
외부의 이벤트를 기다리는 가장 간단한 방법은 반복문 안에서 이벤트를 계속 확인하는 것입니다.


```c
 int readn;
 do {
    readn = read(fd, buffer, BUF_MAX);
    // processing buffer
 } while (readn > 0)
```

하지만 이런 방법으로는 여러개의 이벤트 소스에 대기하기 위해서는 이벤트 소스 개수 만큼의 쓰레드가 필요합니다.
이를 해결하는 방법으로는 Multiplexing I/O(select, poll, epoll)를 사용하는 것입니다. 

```c
while (true) {
    select(nfds, &readfds, &writefds, &exceptfds, timeout);
    // processing fds
}
```

하지만, 이러한 Multiplexing I/O는 OS의 종류와 버전에 따라 의존적이고 사용하는데 어려운 점이 있습니다.
event loop 라이브러리들은 각 OS가 제공하는 Multiplexing I/O를 사용하여 I/O이벤트에 대기하고 이벤트가 발생했을때 원하는 처리를 할 수 있도록 callback 핸들러를 호출하는 구조를 제공합니다. 그리고 Multiplexing I/O를 직접 사용 하는것 보다 쉽게 사용할 수 있습니다.

대부분의 Event Loop 라이브러리는 Timer, I/O event handler, idler를 제공합니다. 
 * Timer : 지정한 시간이 경과된 후 handler를 호출해 준다.
 지정한 시간의 정확도는 loop가 한번 순회하는데 걸리는 시간만큼이다. 따라서 0으로 timer를 지정하더라도, loop가 한번 순회하는데 걸리는 시간만큼 경과된 시간에 handler가 호출된다. 
 * I/O event handler : 지정한 fd에 이벤트가 발생한 경우 handler를 호출해 준다. 
 * idler : event loop이 I/O event에 대기하기 위해 idle상태로 들어가기 전에 호출한다.

Event loop에서 각 이벤트가 발생했을때 사용자가 이를 처리하기 위해 사용자가 등록한 handler 호출해 줍니다. 
이때 알아야할 가장 중요한 개념 중 하나는 "run-to-completion"입니다.

### Run-to-completion
이벤트 핸들러는 이전 이벤트 핸들러가 모두 처리된 이후에서야 호출 될 수 있습니다. 즉, 자신이 작성한 핸들러(callback)의 코드가 모두 실행되고 종료(리턴)될때까지 다른 작업에 의해 선점되지 않고 수행됩니다. 만약 특정 handler안에서 blocking API를 사용한다면, 모든 event loop이 blocking됩니다. 따라서 반응성 높은 어플리케이션 작성을 위해서는 handler안에서는 blocking API를 사용하지 않아야 하며, 수행이 오래걸리는 작업의 경우에도 별도의 thread에서 수행하여 어플리케이션의 반응성에 방해를 해서는 안됩니다.

### Race condition
보통 event loop을 사용하는 어플리케이션은 싱글 쓰레드를 사용하여 이벤트를 처리합니다. 이러한 특징으로 얻는 장점은 
단일 쓰레드에서만 접근하는것을 약속하고 사용하기 때문에 handler에서 접근하는 데이터에 대해 race condition 상황을 걱정하지 않아도 됩니다. 만약 event loop을 사용하면서, worker thread를 사용하여 event loop에서 사용하는 데이터에 접근하려고 한다면, 앞선 가정이 깨지게 되어 race condition에 놓인 데이터에 대해 lock을 사용하여 race condition상황을 회피하도록 해야 합니다. 대부분의 UI작업은 Main event loop thread에서만 하도록 허용하고 있다. 그 이유가 이러한 race condition상황을 원천적으로 배제를 하기 위함입니다.

### Task Queue
 Event Loop의 또 다른 특징은 해당 Thread에 작업을 전달 하는 방법을 제공하는것 입니다. Event loop은 구현체에 따라 다양한 종류의 queue를 관리합니다. 예를 들어 IO작업이 완료된 이후에 바로 핸들러가 호출되어 처리되는 것이 아니라 IO완료 큐를 통해 순차적으로 작업이 처리됩니다. 그리고 Idler를 통해 등록된 작업들도, Idle상태에 진입하기 전에 순차적으로 수행이 됩니다. 이런 큐 구조의 특징으로 다른 Thread에서 event thread로 작업을 전달 할때도, 이 큐를 사용하게 됩니다. Idler에 task를 전달하는 방법으로는 구현체에 따라 `g_idle_add`, `ecore_idler_add`와 같은 함수가 제공됩니다. 