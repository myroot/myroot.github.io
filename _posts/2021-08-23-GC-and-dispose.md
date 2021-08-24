---
title: "Garbage collection and dispose"
layout: post
comments: true
---
C++로 개발하던 개발자가 C#을 사용하면서 자주 실수 하는것 중에 소멸자와 관련된 부분이 있습니다. C++의 소멸자와 C#의 소멸자는 그 역할과 호출 타이밍에 있어서 많은 차이가 있어서 C++의 소멸자를 생각하고 C# 소멸자를 사용하게 되는 경우 원하지 않은 결과가 될 수 있습니다. C#은 GC(Garbage Collector)에 의해 메모리가 관리되는 Managed language입니다. 메모리 해제와 소멸자의 호출은 GC에 의해 수행되며, 그 시점이 C++의 그것과 많이 차이가 있습니다. 또한 C++에서 shared pointer를 사용한 래퍼런스 카운트 기반의 메모리 관리와 C#에서의 메모리 관리와는 차이가 있어서 이런 차이점을 숙지하지 않은 경우 잘못된 코드를 작성할 수 있습니다. 이번 글에서는 이러한 차이점과 C#언어에서 GC를 통해 메모리를 관리하는 방법을 알아보려고 합니다.


## 소멸자의 호출
 C++에서는 소멸자는 `delete`/`delete[]` 연산자에 의해 객체가 삭제되기 전에 호출이 됩니다. 따라서 소멸자가 호출되는 thread는 `delete`를 호출한 thread가 되고 callstack도 `delete`의 호출에 소멸자 호출이 쌓이는 구조가 됩니다. 이런 구조적인 특징으로 C++에서는 생성자와 소멸자를 활용한 많은 이디엄(idiom)이 존재합니다. 대표적으로 RAII(Resource Acquisition Is Initialization) 이디엄은 자동으로 로그를 기록하거나 Lock을 자동으로 해제 하기 위한 방법으로 많이 사용되고 있습니다.

```c++
class ScopedLock : NonCopyable// Scoped Lock idiom
{
  public:
    ScopedLock (Lock & l) : lock_(l) { lock_.acquire(); }
    ~ScopedLock () throw () { lock_.release(); } 
  private:
    Lock& lock_;
};
//....
{
    ScopedLock(mylock);
    // a critical section
}
```
이런 동작을 상상하고 C#의 소멸자를 활용하는 경우 문제가 됩니다. 

C#의 경우 소멸자는 GC(Garbage collection) 과정에서 일정한 시점에 메모리 해제가 필요한 객체에 대하여 일괄적으로 호출이 됩니다. 따라서 어떤 시점에 호출이 되는지 예측이 불가능하며, 수행되는 thread 또한 GC thread에서 수행됩니다. 따라서 이를 고려하지 않고 소멸자에 의존하여 작성한 코드는 문제가 될 수 있습니다.

C#에서는 이러한 소멸자의 불확실한 수행 시점과 GC에 의한 자원해제를 좀 더 명시적으로 수행하는 방법을 제공하기 위해 `IDisposable`이라는 인터페이스를 제공하고 있습니다. 이 인터페이스를 사용함으로서 C++에서의 소멸자와 유사한 동작을 구현 할 수 있습니다.

## IDisposable 인터페이스
 `IDisposable` 인터페이스는 클래스에서 소유한 자원에 대해 해제할 수 있는 방법을 제공하는 표준 인터페이스입니다.
 `IDisposable` 인터페이스는 `Dispose`메소드를 정의하며, 이 인터페이스를 구현한 클래스는 Dispose메소드가 수행될 때, 자신이 소유한 자원을 해제하도록 구현하게 됩니다. 또한 IDisposable인터페이스를 구현한 객체는 `using` 블럭과 결합하여 Dispose에 대한 호출을 자동으로 할 수 있습니다.

 ```c#
 using (var obj = GetSomeDisposableObject())
 {
     // use obj 
 }
 // obj is disposed

 {
     using var obj = GetSomeDisposableObject();
     // use obj
 }
 // obj is disposed
 ```
 위와 같이 두 가지 구조로 using과 결합하여 Dispose를 자동으로 호출 되도록 할 수 있습니다. using문의 괄호 안에 IDisposable객체를 넣고 새로운 블럭을 작성하여 해당 블럭의 스코프가 벗어난 경우 Dispose가 호출되게하거나, 이미 존재하는 블럭 위에서 지역변수 선언 앞에 using키워드를 넣어서 해당 지역변수가 스코프를 벗어날때 Dispose가 불리게 하는 방법이 있습니다.

 이렇게 IDisposable인터페이스와 using을 결합하여 C++에서 구현하였던 RAII이디엄을 동일하게 구현할 수 있습니다.


## GC대상의 객체
C++에서 레퍼런스 카운트 방식의 메모리 관리(shared_ptr)는 내가 참조하는 객체는 반드시 유효하다는 보장이 있습니다. 그래서 일부 잘못된 구조에서 상호 참조가 발생하는 경우 아무도 해당 객체들을 사용하지 않고 있음에도 각각의 객체를 참조하는 객체가 존재하는 것으로 판단하므로 삭제가 되지 않은 상황이 발생하기도 합니다. 

C#에서의 객체의 유효성은 약간 다른 방식으로 계산됩니다. 각 객체가 글로벌 시작점이라고 불리는 지역변수, 전역변수, static변수로부터 도달 가능한지에 대한 Reachable 여부를 판단하여 객체의 유효성을 판단합니다. 그래서 상호 참조로 서로 참조하고 있더라도 이렇게 서로 참조하는 객체에 도달하지 못하는 경우 이 객체들도 유효하지 않다고 판단하여 GC의 대상이 됩니다.
이런 특징으로 소멸자가 불리는 상황에서 특별히 주의해야할 상황이 있습니다.

### 소멸자에서 하면 안되는 일
C++에서는 소멸자에서도 자신이 참조하는 모든 객체에 접근이 가능하였습니다. 이는 레퍼런스 카운트 방식의 메모리 해제 방식을 따르고 있기 때문에 내 자신이 참조하는 객체는 레퍼런스 카운트가 유지되므로 삭제되지 않은 객체라는 보장이 있기 때문입니다. 하지만, C#방식의 메모리 해제는 Unreachable한 객체를 구분한 후, 소멸자가 존재하는 객체만을 별도로 관리하여 추후 소멸자를 호출하는 방식입니다. 만약 소멸자가 없는 객체라면 바로 사용되는 메모리는 해제됩니다. 그리고 소멸자가 존재하는 객체의 경우에 소멸자가 호출되는 순서는 정해져 있지 않으며, 어떤 순서대로 호출되는지 예측이 불가능합니다. 따라서 소멸자가 호출되는 시점에는 해당 객체가 참조하고 있는 모든 객체들이 정상적으로 사용가능한 상태인지 이미 소멸자가 호출되어 삭제된 이후인지 예측할 수 없습니다. 그렇기 때문에 자신의 소멸자 안에서 다른 멤버 변수를 사용하거나 접근하여 업데이트 하는 일이 없어야 합니다.

설령 소멸자에서 이미 GC가 된 객체를 대상으로 무언가하였고 잘 동작하는것처럼 보여지더라도 이런 코드는 작성해서는 안됩니다. 이 동작은 보장된 동작이 아니고 내 실행환경에서 운이 좋게 돌아간 것이거나 내가 사용하는 특정 런타임 버전에서만 잘 동작하는 것일 수 있습니다. 즉, 스팩화 되지 않은 동작에 대해 의존하여 코드를 작성하지 않는 것이 좋습니다.

그리고 소멸자에서 해당 객체를 다시 접근 가능한 곳으로 옮겨 놓는것도 하면 안되는 일입니다. 그렇게 다시 reachable객체가 되더라도 객체의 상태는 이미 소멸자가 호출되어 삭제된 상태로 인식하기 때문에 다시 unreachable이 되어 소멸자가 호출되어야 할때 소멸자는 호출되지는 않습니다.

## Dispose Pattern
 확장 가능한 클래스를 설계하면서 IDisposable인터페이스를 구현할때는 클래스를 상속하는 하위 클래스에서도 Dispose상황에서 자신의 자원을 해제할 수 있는 방법을 제공해야 합니다. 또한 비관리 자원을 소유한 경우에는 Dispose를 명시적으로 호출하지 않고 객체가 삭제되는 상황을 대비하여 소멸자를 통해서 비관리 자원이 해제될수 있도록 해야 합니다. 이를 고려하여 IDisposable을 구현하는 가장 좋은 방법은 MS에서 가이드 하는 [Dispose pattern](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose#implement-the-dispose-pattern)을 적용하는것 입니다. Dispose패턴의 가장 중요한 부분은 protected virtual로 제공하는 Dispose(bool) 메소드입니다.
 ```c#
 protected virtual void Dispose(bool disposing)
 ```
 해당 메소드를 하위 클래스에서는 override하여 자신이 소유한 자원을 해제하고, 다시 base.Dispose(bool)을 호출하여 부모 클래스에서 관리하는 자원 또한 해제가 되도록 해야합니다.

 여기서 disposing의 의미가 중요합니다. disposing이 true인 경우 Dispose를 명시적으로 호출하는 상황이며, disposing이 false인 경우는 Dispose가 소멸자에 의해서 호출되는 상황입니다. 따라서 disposing이 true인 상황에서는 모든 managed객체에 접근이 가능하며 managed객체에 대한 자원 해제 작업도 필요합니다. 하지만 disposing이 false인 상황에서는 모든 managed객체는 이미 해제가 된 상태이므로 managed 객체에 대한 자원 해제는 할 필요가 없습니다. 단지, 관리되지 않은(unmanaged) 객체는 GC에 의해 관리되지 않으므로 disposing이 true이든 false이든 항상 해제해야 합니다.

Dispose pattern에서 Dispose메소드는 아래와 같이 구현됩니다.
 ```c#
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
 ```
 약간 특별한 `GC.SuppressFinalize(this);` API가 사용되었습니다. 해당 API의 기능은 인자로 전달된 객체의 소멸자가 호출되지 않도록 하는것입니다. 우선 C#에서의 소멸자는 앞서 설명했듯이 C++와 달리 객체를 정리하는 목적으로는 사용하지 않기 때문에 대부분의 경우 소멸자를 구현할 필요가 없습니다. 또 한 GC는 소멸자가 존재하는 객체를 특별하게 다루고 있어서 소멸자를 선언하는것 만으로도 성능에 있어서 손해가 발생합니다. 때문에 소멸자를 사용하는 것을 가능한 피해야합니다. 하지만 관리되지 않은 자원을 소유하고 있는 객체는 최종적으로는 소멸자를 통해서만 비관리 자원을 해제할 수 있으므로 어쩔수 없이 소멸자를 선언해야 합니다. 이런 상황에서 Dispose를 통해 이미 비관리 자원을 해제하였다면, 이때는 소멸자의 호출이 더 이상 필요 없기 때문에 `GC.SuppressFinalize(this);`를 호출하여 소멸자 호출이 안되도록 하는 것입니다.
그리고 만약 디자인 하는 클래스에 비관리 자원이 없어서 소멸자가 필요 없는 상황이라도, `GC.SuppressFinalize(this);`를 호출하는 부분은 유지해야 합니다. 이미 소멸자가 없는 객체에 대해 `GC.SuppressFinalize(this);` API호출은 아무런 변화가 없지만, 해당 클래스를 상속 받은 클래스에서 비관리 자원을 사용하고 소멸자를 추가한 상황에서 Dispose를 통해 이미 자원이 해제된 경우 소멸자 호출이 되지 않도록 하기 위해서 `GC.SuppressFinalize(this);`를 유지해야 합니다.


## 링크
 * Dispose Pattern : [https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose#implement-the-dispose-pattern](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose#implement-the-dispose-pattern)

 * Fundamentals of garbage collection : [https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals)