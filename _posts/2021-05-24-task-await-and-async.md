---
title: "Task, await and async"
layout: post
comments: true
---
.net framework에서는 비동기 작업을 위해 여러가지 built-in된 방법을 제공하고 있으며, 특히 Task, await 그리고 async등은 높은 수준의 개념으로  비동기 작업을 다룰수 있는 방법을 제공합니다. 이를 적극적으로 활용하면비동기 작업을 위해 callback 지옥에 빠지는 등의 문제를 겪지 않고도 효율적인 비동기 코드를 작성할 수 있습니다.

이번 글에서는 .net에서 효율적인 비동기 프로그래밍을 위한 Task, await 그리고 async에 대한 주제로 설명하려고 합니다.


## Task
C#에서의 Task는 특정 행위/동작과 그 결과에 대해 추상적인 표현입니다. 다른 언어에서 제공하는 기능과 비교하자면, Javascript에서의 Promise, Java에서의 Future와 같이 비동기적인 작업의 결과를 담는 용도로서 사용됩니다.

C#에서는 Task클래스 자체에서 비동기적인 작업에 대한 결과로서의 역할 뿐만 아니라 thread pool에 작업을 시작하는 기능 또한 제공하고 있어서 다른 언어의 그것보다 더 많은 역할을 갖고 있지만, 이 글에서 다루고자하는 주요 기능은 비동기적인 작업의 결과를 표현하는 기능입니다.

### Task 상태
Task는 특정 작업과 그 작업의 결과에 대한 추상적인 표현이라고 위에서 설명하였습니다. 따라서 Task는 그가 표현하는 작업의 완료 여부에 따라서 상태가 [TaskStatus](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskstatus?view=net-5.0)로 표현됩니다. 다음 RanToCompletion, Faulted, 또는 Canceled 상태인 경우는 Task가 완료되었다고 표현할 수 있으며, 그 외의 상태에서는 Task가 완료상태가 될때까지 다양한 방법으로 대기할 수 있습니다.

### Task 객체의 생성
일반적으로 개발자는 Task객체를 직접 생성하지 않습니다. 대부분의 경우에 .net framework에 의해 내부적으로 생성되며, 다양한 메소드의 결과값으로 Task를 받아서 사용하는것이 주된 Task의 사용 시나리오입니다.

그렇지만 작업의 완료를 표현하려는 용도로 Task를 생성할 필요가 있으며, 이런 경우에 TaskCompletionSource를 사용하여 Task를 생성할 수 있습니다.

```c#
TaskCompletionSource<bool> tcs = new TaskCompletionSource<bool>();
AsyncMethodWithCallback(() =>
{
    tcs.SetResult(true);
});

var task = tcs.Task;
task.ContinueWith((t) => {
    Print("Task complete");
});
```
TaskCompletionSource클래스를 사용하여 Task를 생성하고 해당 작업의 완료를 TaskCompletionSource의 SetResult 메소드를 사용하여 생성된 Task에 전달 할 수 있습니다.

## await
생성된 Task가 완료될때까지 대기하거나 완료 후 원하는 작업을 진행하는 방법에는 여러 가지 방법이 있습니다.

 * [ContinueWith Method](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.continuewith?view=net-5.0)
   > Task가 완료된 후 실행할 작업을 Callback으로 전달합니다.
 * [Wait Method](https://docs.microsoft.com/ko-kr/dotnet/api/system.threading.tasks.task.wait?view=net-5.0)
   > Task가 완료되는것을 현재 Thread를 점유한 체 대기합니다. (busy waiting). 따라서 Task의 완료에 현재 Thread에서의  동작이 필요한 경우에 Deadlock상황이 발생할 수 있습니다.
 * [await operator](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await)
   > Task가 완료되는것을 현재 Thread를 점유하지 않은 체 대기합니다.

이 중 await 연산자를 사용하는 것이 코드의 가독성을 해치지 않으면서 효율적으로 Task에 대기하는 방법입니다.

await는 Task의 완료를 대기할 때, 어떤 Thread도 점유하지 않습니다. 그리고 Task가 완료된 이후에는 다시 돌아갈 Thread를 선택하고 해당 Thread에서 await 이후의 코드를 수행합니다.
 이 때 await가 돌아갈 Thread를 선택하는 방법이 바로 이 전 글에서 다루었던 SynchronizationContext([Event loop on C#](/event-loop-on-csharp/))입니다. await를 통해 대기 상태로 진입할 당시의 Thread의 SynchronizationContext.Current에 설정된 SynchronizationContext는 추후 Task가 완료된 시점에 사용되어 await로 대기 상태로 진입했던 Thread로 돌아가서 나머지 코드를 수행 할 수 있게 됩니다.
 
 만약 SynchronizationContext.Current가 없는 상태의 Thread였다면 await 이후 코드는 새로운 Thread(workert thread)를 통해 수행이 됩니다. 그리고 Task를 완료하는 Thread가 Task를 대기하는 Thread와 동일한 경우 Post과정 없이, Task를 완료하는 과정에 이어서 수행 됩니다.


## async method
 일반적인 함수의 경우 실행에 필요한 local variable은 callstack에 저장되고, 함수 수행이 완료되면 stack 포인터가 pop되면서 local variable들은 사라지게 됩니다. 그럼 어떻게 await는 코드 수행이 진행 중인 메소드를 중단하면서 callstack에 쌓여 있는 local variable은 어떻게 유지 될 수 있는 것일까요? 그리고 어떻게 원래 중단된 지점으로 되돌아 갈 수 있는것일까요?

 이런 동작을 위해 C#은 async 키워드가 붙은 async method를 정의하였고, 이 async method는 컴파일러에 의해 상태를 갖는 객체 형태로 각 코드가 변환되게 됩니다.

```c#
static async void AwaitAsyncExample()
{
    Print("Before await code");
    await ASomeAsyncMethod();
    Print("After await code");
}
```
위 코드와 같이 await를 사용한 async 메소드를 작성하여 나온 dll을 디컴파일러를 통해 생성된 코드를 확인하면 (.pdb파일을 삭제하고 확인해야 실제 생성된 코드로부터 디컴파일 한 결과를 볼 수 있습니다.)

```c#
private static void AwaitAsyncExample()
{
    Program.<AwaitAsyncExample>d__3 stateMachine = new Program.<AwaitAsyncExample>d__3();
    stateMachine.<>t__builder = AsyncVoidMethodBuilder.Create();
    stateMachine.<>1__state = -1;
    stateMachine.<>t__builder.Start<Program.<AwaitAsyncExample>d__3>(ref stateMachine);
}
```
위 코드는 AwaitAsyncExample에 해당하는 코드가 변환된 결과입니다. 원래의 코드는 없어지고, 메소드 구현이 stateMachine객체를 생성하고 해당 객체의 state를 -1로 설정하고 Start하는 것으로 대체가 되었습니다.

원래의 코드는 아래와 같이 별도의 객체로서 표현되어 변환 되었습니다.
```c#
    private sealed class <AwaitAsyncExample>d__3 : IAsyncStateMachine
    {
      public int <>__state;
      private int <localVariable>5__1;
      
      void IAsyncStateMachine.MoveNext()
      {
        int num1 = this.<>1__state;
        try
        {
          TaskAwaiter awaiter;
          int num2;
          if (num1 != 0)
          {
            this.<localVariable>5__1 = 1;
            Program.Print("Before await code " + this.<localVariable>5__1++.ToString());
            awaiter = Program.ASomeAsyncMethod().GetAwaiter();
            if (!awaiter.IsCompleted)
            {
              this.<>1__state = num2 = 0;
              this.<>u__1 = awaiter;
              Program.<AwaitAsyncExample>d__3 stateMachine = this;
              this.<>t__builder.AwaitUnsafeOnCompleted<TaskAwaiter, Program.<AwaitAsyncExample>d__3>(ref awaiter, ref stateMachine);
              return;
            }
          }
          else
          {
            awaiter = this.<>u__1;
            this.<>u__1 = new TaskAwaiter();
            this.<>1__state = num2 = -1;
          }
          awaiter.GetResult();
          Program.Print("After await code " + this.<localVariable>5__1.ToString());
        }
        catch (Exception ex)
        {
          this.<>1__state = -2;
          this.<>t__builder.SetException(ex);
          return;
        }
        this.<>1__state = -2;
        this.<>t__builder.SetResult();
      }
    }
```
(이해하는데 불필요한 코드는 생략하였습니다.)
원래의 코드 블럭이 await의 호출을 기준으로 두 블럭으로 나뉘어서 IAsyncStateMachine.MoveNext 메소드 표현되어 있습니다. 그리고 local variable로 선언한 int타입의 localVariable은 객체의 instance variable로 선언되어 있습니다. 이렇게 표현된 객체를 통해 await전 코드를 수행하고 종료되어 있다가 이후 Task가 완료되면 state를 업데이트하여 다시 MoveNext를 호출하여 await이후 코드가 실행 될수 있도록 합니다. 그리고 local variable들은 모두 instance variable로 변환되어 저장되기 때문에 함수가 다시 수행되어도 원래 값을 유지한체 다음 코드를 수행 할 수 있습니다.

## 마무리
이번 글에서는 c#에서 비동기 작업을 표현하는 방법과 이를 효율적으로 사용하는 방법에 대해 다루었습니다. 비동기 함수는 빠른 반응성에 있어서 매우 중요한 메커니즘으로 잘 이해하고 적극적으로 사용해야합니다. 그리고 await/async 또 한 코드의 가독성을 높이면서 비동기 함수를 다룰 수 있는 좋은 방법입니다. 그럼에도 변환된 코드로 인해 의도하지 않은 결과가 발생하기도 하고, 이를 잘 이해하지 못한 경우에 디버깅하기도 어렵게 됩니다. 이 글을 통해 동작 메커니즘을 잘 이해하고 효율적인 코드 작성에 도움이 되었으면 합니다.

## 링크
 * 디컴파일러(dot peek) : [https://www.jetbrains.com/ko-kr/decompiler/](https://www.jetbrains.com/ko-kr/decompiler/)
 * await operator : [https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/await)