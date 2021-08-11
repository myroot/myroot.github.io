---
title: "Event loop on C#"
layout: post
comments: true
---

이번 글에서는 .net framework에서는 어떻게 Event loop과 협력하는 방법을 제공하는지에 대해 알아보려고 합니다. Event loop은 제공하는 플랫폼에 따라 그 구현이 다양하고 사용방법에 있어서도 사용해야할 API가 모두 다르기 때문에, 멀티 플랫폼에서 동작하는 소프트웨어 개발이 필요한 곳에서는 이러한 다양한 EventLoop구현에 대응하기 위하여 추상화된 Eventloop 인터페이스와 각 구현에 맞게 구현된 서브 클래스로 제공되어집니다. 이를 동작하는 플랫폼에 맞게 선택하여 사용하도록 하는것이 일반적으로 event loop을 멀티플랫폼에서 공용으로 활용하는 방법입니다.
 .net framework에서는 이를 framework래벨에서 제공하고 있는데, event loop의 다양한 역할 중에 Task Qeueue의 역할을 하는 기능에 대하여 SynchronizationContext라는 클래스를 제공하고 있습니다.


## SynchronizationContext
SynchronizationContext는 지정된 특정 Thread와 협력하여 동작하며, 해당 thread에 작업을 전달하여 그 작업이 해당 Thread에서 동작할 수 있도록 합니다.

우리는 이 SynchronizationContext을 사용하여 작업을 원하는 Thread로 전달 할 수 있습니다. 

그러면 SynchronizationContext는 어떻게 획득할 수 있을까요?

SynchronizationContext는 Current라는 ThreadLocal변수 제공하고 있습니다. 그래서 해당 Thread와 협력하는 SynchronizationContext는 Current 변수를 통해 획득할 수 있습니다. 

이는 해당 Thread가 특정 Event loop와 결합되어 있을 경우에 해당 프레임워크 및 런타임을 제공하는 곳에서 Current에 해당 Thread와 협력하는 SynchronizationContext객체를 생성하여 설정하고 있기 때문입니다.

따라서, Event loop이 동작하지 않은 thread의 경우 SynchronizationContext.Current는 null로 설정되어 있으며 이는 해당 Thread에는 작업을 전달 할 수 없음을 의미합니다.
즉 thread에 작업을 전달하는 기능은 모두 Event loop의 구현에 의존하고 있으며, SynchronizationContext는 이런 Event loop의 사용을 추상화하여 제공하고 있을 뿐 입니다.

## SynchronizationContext와 비동기 작업
그러면 어떻게 SynchronizationContext를 사용하여 비동기 작업의 결과를 main thread에 전달 하는지에 대해 알아보려고 합니다.

일반적으로 비동기로 동작하는 작업은 main thread가 아닌 새로운 Thread를 생성하여 해당 thread에서 작업을 수행하게 되고 여기서 생성된 작업의 결과는 다시 SynchronizationContext를 통해 main thread로 전달되어 결과를 동기적으로 안정하게 사용할 수 있게 됩니다.

```c#
static void RunHeavyTaskAndReturnOnCurrentThreadContext()
{
    var context = SynchronizationContext.Current;
    Task.Run(() =>
    {
        // run a heavy task
        Print("Run Async task");

        context.Post((s) =>
        {
            // this code run on main thread
            Print("Return result");
        }, null);
    });
}
```

위 코드는 Task.Run을 사용하여 Worker thread를 생성하고 비동기 작업을 수행한 후 그 작업의 결과물을 다시 SynchronizationContext의 Post를 사용하여 Main thread로 전달하는 코드를 보여줍니다. 

## SynchronizationContext의 구현
위 예제에서는 SynchronizationContext를 사용하는 방법에 대해 알아보았다면, 이번에는 SynchronizationContext가 어떻게 Event loop을 이용하여 동작하는지 SynchronizationContext를 직접 구현해 보려고 합니다. SynchronizationContext는 동작의 특성상 특정 Event loop구현에 의존적으로 코드를 작성할 수 밖에 없습니다. 이번 예제에서는 매우 간단한 Event loop을 만들고 이와 동작하는 SynchronizationContext를 만들어보로독 하겠습니다.

### Event Loop의 인터페이스
Event loop은 크게 아래 2가지 메소드를 제공합니다. 
1. Event loop의 구동을 시작하는 메소드
 ```c#
  public void Run()
 ```
  >이는 해당 메소드를 수행하는 Thread에 event loop을 구동시키는 것으로 해당 메소드는 event loop이 종료되기 전까지 리턴되지 않고 계속 thread를 선점한 상태로 동작하게 됩니다.
2. Event loop에 작업을 등록하는 메소드
 ```c#
  public void Post(Action task)
 ```
  >이 메소드를 통해서 전달하고자하는 Task를 해당 event loop으로 전달할 수 있습니다.

위 두가지 메소드 중에서 SynchronizationContext를 작성하기 위해서는 작업을 Event loop으로 전달하는 방법을 제공하는 Post메소드가 필요합니다.

### SynchronizationContext 상속
SynchronizationContext 클래스를 상속하여 새로운 SynchronizationContext 클래스를 만듭니다. 그리고 Post와 Send메소드를 재정의(override)하여 클래스를 완성시킵니다.
```c#
public override void Post(SendOrPostCallback d, object state)
{
    _eventLoop.Post(() =>
    {
        d.Invoke(state);
    });
}
```
Post로 전달된 callback을 Event loop의 Post를 사용하여 구현하고 있습니다. 사실 구현은 단순히 callback을 자신이 사용하는 event loop의 API를 사용하여 해당 thread로 대신 전달해주는것일 뿐 특별한 것은 없습니다.
(Send도 같은 방식으로 재정의합니다.)

### SynchronizationContext 설정
SynchronizationContext는 동작하는 thread의 thread local변수로 저장되어 있어야 합니다. 따라서 SynchronizationContext를 생성하고 해당 Thread에 설정하는 절차가 필요합니다. 이는 [SetSynchronizationContext](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext.setsynchronizationcontext?view=net-5.0) 정적 메소드를 사용하여 등록되며 수행되는 thread의 threadlocal변수로 SynchronizationContext.Current에 해당 객체를 설정합니다.

이 과정은 일반적으로 event loop을 제공하는 Framework제공자가 수행합니다.
```c#
// 단순화된 버전의 초기화 코드
loop = new EventLoop();
SynchronizationContext.SetSynchronizationContext(new MockSynchronizationContext(loop));
```

## 마무리
이처럼 .net framework에서는 하위 구현에서 사용되는 event loop의 종류와 상관 없이 특정 작업을 원하는 thread로 전달하는 표준화된 방법을 제공하고 있기 때문에 소프트웨어 개발에 있어서 구체화된 구현체에 의존하지 않고 공용의 방법으로 좀 더 유연한 코드를 작성할 수 있습니다.

이런 동작을 가능하게 하는 SynchronizationContext를 제대로 알고 활용한다면 좀 더 효율적이고 재사용 가능한 소프트웨어 개발을 쉽게 할 수 있습니다.

# 링크
 예제 코드 [https://github.sec.samsung.net/sngnlee/ExampleCode/tree/main/EventLoop](https://github.sec.samsung.net/sngnlee/ExampleCode/tree/main/EventLoop)
