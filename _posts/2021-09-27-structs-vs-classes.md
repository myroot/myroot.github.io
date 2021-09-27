---
title: "Structs vs Classes"
layout: post
comments: true
---
C++에서 class와 struct차이는 기본 접근자가 public이냐 private냐 정도의 차이만 있을뿐 class와 struct사이에 차이가 거의 없었습니다. Goolge의 C++ style guide에서는 
>Use a struct only for passive objects that carry data; everything else is a class.

C++에서는 struct의 사용을 수동적인 데이터 저장소로만 제한적으로 선택하도록 하고 대부분의 상황에서 class를 사용하도록 가이드 하고 있습니다.

하지만 C#에서는 struct는 값 타입으로 동작하고 class는 참조 타입으로 동작하고 있어서 이 둘간의 동작에 있어서 차이가 많이 있기 때문에 어떤 것을 선택하는데 있어서 이 둘간의 특징을 잘 알고 선택해야 합니다.


### 할당 공간의 차이
참조 타입인 class는 힙에 할당되고 GC에 의해서 해제됩니다. 값 타입인 struct는 스택 공간에 할당되고 스택이 팝되면서 자동으로 해제가 됩니다. 

이러한 할당 공간의 차이는 배열의 할당 방법에 있어서도 차이가 존재합니다. 참조 타입을 위한 배열은 단순히 참조를 위한 공간만을 할당하고 값 타입을 위한 배열은 값 타입 자체를 저장할 공간을 할당하게 됩니다.

![image](https://media.github.sec.samsung.net/user/376/files/dd4edb00-1f9a-11ec-8618-f0c2fe94d0cc)

### 할당 방식의 차이
할당 되는 공간의 차이로 할당 방식에 있어서도 차이가 발생합니다. 참조 타입의 경우 새로운 변수에 할당되는 경우 단순하게 힙에 존재하는 인스턴스에 대한 참조만 추가됩니다. 따라서 새로 할당된 변수를 사용하여 값을 업데이트 하는 경우 해당 인스턴스를 가르키는 모든 참조에 영향을 줍니다. 반면 값 타입의 경우 새로운 변수에 할당하는 경우 기존 인스턴스의 모든 값이 복사되어 할당되게 됩니다. 

![image](https://media.github.sec.samsung.net/user/376/files/6d405500-1f9a-11ec-82ef-cdb4fc7d3936)

이런 방식의 차이는 메소드의 인자로 인스턴스가 전달될 때도 동일한 방식으로 적용됩니다. 즉 struct타입의 인스턴스를 전달할때는 값이 복사하되 전달되며, class타입의 인스턴스를 전달할때는 참조만 전달이 됩니다. 따라서 메소드 안에서 전달된 인자에 대한 업데이트가 발생한 경우 strcut는 호출 시 사용된 인스턴스에는 영향을 미치지 않지만, class타입의 경우 호출 시 사용된 인스턴스에 영향을 미치게 됩니다.

### 자질구래한 차이
 * struct에서는 == 및 != 연산자는 명시적으로 오버로드하지 않는 한 struct 에 대한 == 및 != 연산을 수행할 수 없습니다.
 * 인스턴스의 Field, Property가 모두 동일한 경우

    |method|struct|class|
|-|-|-|
|Equals| true | false |
|GetHashCode | 동일한 값 | 서로 다른 값|


### Rule of thumb
위와 같은 차이로 인해 우리는 표현하려는 객체의 특징에 맞게 struct나 calss를 선택하여 사용해야 합니다. 
일반적으로 Immutable한 객체를 표현하는데 struct가 적합합니다 Immutable한 객체는 값 자체가 무언가를 표현하는 객체로 생각할 수 있습니다. 값 자체가 중요하기 때문에 동일한 값을 갖고 있다면 동일한 객체로 판단해야 합니다. 구체적인 예를 들면 크기를 나타내는 Size라는 객체의 경우 Width와 Height두가지 요소가 존재하는 객체로 표현할 수 있는데, 이 객체는 Width와 Height가 동일하다면 그 값이 어디에 저장되어 있더라도 동일하다고 판단해야 할 것입니다. 그리고 만약 Size에 대한 변경이 필요한 경우라면, Size 객체를 업데이트하는것이 아니라 Size 객체 자체를 새로운 객체로 대체해야 합니다. 이러한 Immutable한 특징을 갖는 객체의 동작은 strcut를 사용하여 매우 적절하게 구현할 수 있습니다.

아래 예제를 통해 struct가 적합한 Size객체를 class로 표현했을때 얼마나 많은 비효율이 발생하고 직관적이지 못 한 상황이 발생하는지 확인해 보려고 합니다.

### Case study (type for size)
#### 기본 구현
 * struct
```c#
    public struct SizeStruct
    {
        public int Width;
        public int Height;
    }
``` 

 * class
```c#
    public class SizeClass
    {
        public int Width;
        public int Height;
    }
```

#### 비교(Equals, ==) 구현하기
```c#
SizeStruct var1 = new SizeStruct
{
    Width = 100,
    Height = 100,
};
SizeStruct var2 = new SizeStruct
{
    Width = 100,
    Height = 100,
};

Debug.Assert(var1 == var2); // compile error, == 오버로딩 필요
Debug.Assert(var1.Equals(var2)); // pass
Debug.Assert(var1.GetHashCode() == var2.GetHashCode()); // pass
```
 struct에서 기본으로 구현된 Equals와 GetHashCode는 모두 테스트를 통과합니다.
 == 연산자의 경우 오버로딩이 필요한데, `Equals`를 사용하여 쉽게 구현 가능합니다. 다음과 같은 간단한 구현 추가로 모든 테스트를 통과할 수 있습니다.
```c#
public static bool operator ==(SizeStruct lhs, SizeStruct rhs) => lhs.Equals(rhs);
public static bool operator !=(SizeStruct lhs, SizeStruct rhs) => !(lhs == rhs);
```        
 하지만 class의 경우 동일한 값을 표현하고 있음에도 `Equals`결과와 `==`결과가 false가 되어 테스트에 실패합니다.
```c#
SizeClass var1 = new SizeClass
{
    Width = 100,
    Height = 100,
};
SizeClass var2 = new SizeClass
{
    Width = 100,
    Height = 100,
};

Debug.Assert(var1 == var2); // fail
Debug.Assert(var1.Equals(var2)); // fail
Debug.Assert(var1.GetHashCode() == var2.GetHashCode()); // fail
```
때문에 `Equals`와 `GetHashCode`를 재정의(override) 하고 이를 사용하여 `==`연산자를 오버로딩 하여야 합니다.
SizeClass를 위한 Equals와 GetHashCode == 연산자 구현은 아래 코드와 같습니다.

```c#
public override bool Equals(object obj) => Equals(obj as SizeClass);

public bool Equals(SizeClass obj)
{
    if (obj == null)
        return false;
    return Width == obj.Width && Height == obj.Height;
}

public override int GetHashCode() => (Width, Height).GetHashCode();

public static bool operator ==(SizeClass lhs, SizeClass rhs)
{
    if (lhs is null)
    {
        if (rhs is null)
        {
            return true;
        }

        // Only the left side is null.
        return false;
    }
    // Equals handles case of null on right side.
    return lhs.Equals(rhs);
}

public static bool operator !=(SizeClass lhs, SizeClass rhs) => !(lhs == rhs);
```
struct를 사용했을때와 비교하여 상대적으로 매우 많은 코드를 추가하여야지 테스트에 통과합니다.

#### 다른 객체의 멤버로서 포함될 때 사용성
Size객체는 값이 동일하다면 비교의 결과가 true이어야 하지만 그것이 동일한 대상을 지칭하고 있음을 의미하는 것은 아닙니다. 즉 == 결과가 true이더라도 한쪽 값을 변경한 경우 다른 값이 동일하게 변경되는 것을 기대하지는 않습니다.

그래서 아래와 같은 코드의 의도는
```c#
var obj1 = new ClassContainer
{
    Size = new SizeClass
    {
        Width = 100,
        Height = 100,
    }
};

var obj2 = new ClassContainer
{
    Size = obj1.Size
};
```
obj1이 갖고 있는 Size의 값과 동일한 값으로 obj2의 Size를 설정하라는 의미이지 동일한 값을 가르키라는 의미는 아니라는 것을 직관적으로 판단 할 수 있게 됩니다.

하지만, class를 사용하는 경우라면 이러한 직관성과는 맞지 않는 동작을 보여줍니다. 즉 아래와 같은 테스트 코드에서 실패하게 됩니다.

```c#
obj2.Size.Height = 200;
Debug.Assert(obj1.Size.Height == 100); // Fail
```
그래서 Size를 소유한 객체에게만 변경이 적용되도록 하려고 한다면, 객체를 할당할때 복사본을 만들어 할당하거나 Immutable객체로 만들어서 한번 만들어진 객체는 변경이 불가능하도록 해야 합니다. Immutalbe객체로 만드는것이 일반적으로 직관적이고 유효한 방법입니다.
이를 위해 아래와 같이 생성자를 만들고 Width/Height에 대해 get property만 제공하도록 변경합니다.

```c#
public SizeClass(int width, int height)
{
    Width = width;
    Height = height;
}
public int Width { get; }
public int Height { get; }
```
이렇게 변경되면 테스트 코드도 Size객체에 대한 값을 inline으로는 변경하지 못하고 새로운 객체를 생성하여 교체하는 방식으로 변경하도록 수정해야 합니다. (이것이 Immutable객체의 특징입니다)

```c#
obj2.Size = new SizeClass(100, 200);
Debug.Assert(obj1.Size.Height == 100); // pass
```

하지만 struct를 사용한 경우라면 객체를 굳이 Immutable객체로 만들지 않더라도 Get메소드나 Property get을 통해 사용되는 경우 리턴되는 값이 복사 된다는 특징으로 인해 Immutable 객체처럼 수정할 수 없도록 동작하게 됩니다. 따라서 아래와 같은 코드는 허용되지 않습니다.

```c#
var obj2 = new StructContainer
{
    Size = obj1.Size
};

obj2.Size.Width = 200; // 변수가 아니므로 해당 반환 값을 수정할 수 없습니다. compile error
```

#### 최종 결과 코드 비교
 * struct : 처음 구현에서 크게 변경된것 없이 `==`/`!=` 연산자 오버로딩만 추가 하였습니다.
    ```c#
public struct SizeStruct
{
    public int Width;
    public int Height;

    public static bool operator ==(SizeStruct lhs, SizeStruct rhs) => lhs.Equals(rhs);
    public static bool operator !=(SizeStruct lhs, SizeStruct rhs) => !(lhs == rhs);
}   
```

 * class : 비교를 재정의하기 위해 많은 코드가 추가 되었으며 기본적으로 객체가 Immutable하도록 setter를 제거하고 생성자를 통해서만 초기화 가능하도록 변경되었습니다.
 ```c#
public class SizeClass
{
    public SizeClass(int width, int height)
    {
        Width = width;
        Height = height;
    }
    public int Width { get; }
    public int Height { get; }

    public override bool Equals(object obj) => Equals(obj as SizeClass);

    public bool Equals(SizeClass obj)
    {
        if (obj == null)
            return false;
        return Width == obj.Width && Height == obj.Height;
    }

    public override int GetHashCode() => (Width, Height).GetHashCode();

    public static bool operator ==(SizeClass lhs, SizeClass rhs)
    {
        if (lhs is null)
        {
            if (rhs is null)
            {
                return true;
            }

            // Only the left side is null.
            return false;
        }
        // Equals handles case of null on right side.
        return lhs.Equals(rhs);
    }

    public static bool operator !=(SizeClass lhs, SizeClass rhs) => !(lhs == rhs);
} 
 ```

### 마무리
 예제 코드를 통해 값으로서 표현해야하는 타입을 참조 타입으로 표현할 경우 발생하는 많은 문제를 확인하였습니다. C++개발 경험만을 갖은 개발자분들이 C++코드를 C#으로 이식하는 경우에 있어서 이러한 특징을 모른체 strcut는 struct로 class는 class로 기계적으로 이식하는 경우가 많이 있었고 이를 해결하기 위해 변경을 하기도 하지만 API 호환성을 문제로 수정하지 못하고 계속 사용하기 불편한체로 남아 있는 경우도 많이 있었습니다. C#에서의 struct와 class의 차이를 좀 더 명확히 알고 이를 잘 선택하여 구현하여 좀 더 사용하기 좋은 API와 구현이 되었으면 합니다.


### 참고 자료
 * Choosing Between Class and Struct : [https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct](https://docs.microsoft.com/en-us/dotnet/standard/design-guidelines/choosing-between-class-and-struct)
 * How to define value equality for a class or struct : [https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/how-to-define-value-equality-for-a-type](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/how-to-define-value-equality-for-a-type)
