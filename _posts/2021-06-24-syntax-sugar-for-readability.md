---
title: "가독성을 높여주는 C# syntax sugar"
layout: post
comments: true
---
Syntax sugar는 의미적으로는 동일하면서 코드를 좀 더 간결하게 할 수 있는 문법을 말합니다.

C#에서는 다양한 syntax sugar를 제공하고 있으며, 이 중 가독성에 큰 도움이 되는 문법에 대해 알아보려고 합니다.


## Object initializer
 Object initializer는 객체를 생성한 후 할당문을 별도로 추가하지 않고 객체의 Field나 Property를 생성과 함께 초기화 하는 방법입니다.

```c#
    public class StudentName
    {
        // This constructor has no parameters. The parameterless constructor
        // is invoked in the processing of object initializers.
        // You can test this by changing the access modifier from public to
        // private. The declarations in Main that use object initializers will
        // fail.
        public StudentName() { }

        // The following constructor has parameters for two of the three
        // properties.
        public StudentName(string first, string last)
        {
            FirstName = first;
            LastName = last;
        }

        // Properties.
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public int ID { get; set; }

        public override string ToString() => FirstName + "  " + ID;
    }
```
위와 같이 클래스가 정의되어 있을 때
일반적인 문법으로는 아래와 같은 코드로 생성 및 초기화를 작성합니다.
```c#
    var name = new StudentName();
    name.FirstName = "Seungkeun";
    name.LastName = "Lee";
    name.ID = 0;
```
Object initializer를 적용한 경우
아래와 같이 한 statement로 표현할 수 있습니다.
```c#
    var name = new StudentName
    {
        FirstName = "Seungkeun",
        LastName = "Lee",
        ID = 0
    };
```
이렇게 한 문(statement)로 코드를 묶음으로서 좀 더 가속성이 높아지며 간략한 코드 작성이 가능해집니다. 
```c#
{
    var name = new StudentName();
    name.FirstName = "Seungkeun";
    name.LastName = "Lee";
    name.ID = 0;
    Run(name);
}
{
    Run(new StudentName
    {
        FirstName = "Seungkeun",
        LastName = "Lee",
        ID = 0
    });
}

static void Run(StudentName name);
```
위 두 코드를 비교해보면, 모두 Run이라는 메소드에 StudentName객체를 인자로서 전달하기 위해 생성하고 초기화 하는 코드입니다. 
그런데, 첫번째 코드는 실제로는 사용되지 않을 name이라면 변수를 객체의 초기화를 위해 선언하고 있습니다. object initializer를 사용한 경우에는 변수를 선언할 필요 없이 바로 Run함수에 객체를 생성하고 초기화한 객체를 전달할 수 있어서 좀 더 가독성을 높일 수 있습니다.


## is operator
is 연산자는 런타임에 타입 체크를 위한 방법으로 많이 사용됩니다. 그런데 is 연산자 뒤에 [var 패턴](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/patterns#var-pattern)을 사용하여 타입 체크와 동시에 타입 케스팅이 되도록 코드를 작성할 수 있습니다.

이 문법은 C# 7.0에 도입되었으며, 이를 통해 아래와 같은 코드는
```c#
    object name = ...
    if (name is StudentName)
    {
        StudentName studentName = (StudentName)name;
        Console.WriteLine("Hello " + studentName.FirstName);
    }    
```
다음과 같이 더 간략한 코드로 표현이 됩니다.
```c#
    object name = ...
    if (name is StudentName studentName)
    {
        Console.WriteLine("Hello " + studentName.FirstName);
    }
```
## null-conditional operator
 `?.`로 사용되는 null conditional operator는 객체를 사용하기 전에 null검사가 필요한 경우에 매우 유용하게 사용될 수 있는 syntax surgar입니다.

null검사를 하여 null이 아닌 경우에 `.` 뒤에 붙이 메소드가 실행됩니다.
만약 null인 경우라면 해당 `.`뒤의 메소드는 실행이 되지 않으면서 해당 문이 null로 치환됩니다.

```c#
    StudentName name = GetStudentNameOrNull();
    if (name != null)
    {
        name.SayHello();
    }
```
위 코드처럼 함수 결과가 null이 아닌 경우에 해당 객체의 특정 메소드를 호출하려고 한다면 아래 코드처럼 매우 간략하게 줄일 수 있습니다.
```c#
    GetStudentNameOrNull()?.SayHello();
```

## switch 문에서 패턴 사용
``` c#
switch (expr)
```
c# 7.0부터는 switch문에서 expr에 올 수 있는 형식의 제한이 null이 아닌 모든 식으로 완화가 되었습니다. 이에 따라 case 레이블에 올 수 있는 형식이 더 이상 상수로 제한되지 않고 여러가지 표현식을 사용할 수 있도록 되었으며, 매치할 수 있는 표현이 더욱 풍부해졌습니다.

아래와 같은 형식 패턴은 타입을 검사하고 해다 타입에 적합한 작업을 하는데 유용하게 사용될 수 있습니다.
>  case type varname

``` c#
    People people = GetPeople();
    switch (people)
    {
        case Student student:
            GoToClassRoom(student);
            break;
        case Employee employee:
            GoToOffice(employee);
            break;
    }
```

## 마무리
 이번 글에서는 Syntax sugar를 이용하여 동일한 표현에 대해 더 간략하고 가독성 높은 코드를 알아 보았습니다. Syntax sugar는 C# 언어 버전이 올라갈 수록 더 많은 기능들이 포함되어 언어의 표현력이 더 높아지고 있습니다. 자신의 프로젝트에서 사용하는 언어 버전을 가능한 올려서 더 강력한 표현으로 코드를 작성하면 좋을것 같습니다.

## 링크

* Object initializer : [https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/how-to-initialize-objects-by-using-an-object-initializer](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/how-to-initialize-objects-by-using-an-object-initializer)
* type testing with pattern matching : [https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/type-testing-and-cast#type-testing-with-pattern-matching](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/type-testing-and-cast#type-testing-with-pattern-matching)
* null-conditional operator : [https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)
* switch 형식 패턴 [https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/keywords/switch#type-pattern](https://docs.microsoft.com/ko-kr/dotnet/csharp/language-reference/keywords/switch#type-pattern)