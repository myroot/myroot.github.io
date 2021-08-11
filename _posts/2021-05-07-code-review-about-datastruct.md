---
title: "[코드리뷰사례] 의미에 맞지 않은 자료형 사용"
layout: post
comments: true
---
코드 리뷰 사례를 공유하기 위하여 글을 작성하였습니다. 
이번 사례는 표현하려는 데이터가 사용하는 자료형과 맞지 않는 상황에서 해당 자료형을 사용하므로서 발생하는 문제에 대하여 리뷰한 사례입니다.


### 문제 코드
```c#
/// TouchArea can expand the view's touchable area.
/// If you set the TouchAreaOffset on an view, when you touch the view,
/// the touch area is used rather than the size of the view.
/// TouchAreaOffset = new Rectangle(left, right, bottom, top);
Rectangle TouchAreaOffset {get;set;}
```
### 문제 코드에 관한 설명
터치 영역을 확장하기 위한 API. 
4방향 (left,right,bottom, top)에 대한 터치 영역의 offset(+/-)을 설정할 수 있습니다.

### 해당 코드의 문제점
1. Rectangle 자료형은 X/Y, Width/Height를 표현하는 자료형인데, 작성자가 의도한 의미의 값을 나타내지 못합니다.
따라서, API를 사용하는 입장에서는 문서를 보지 않고서는 저 API를 정상적으로 사용할 수 없으며, 코드적으로도
``` c#
 view.TouchAreaOffset.Width // 이것은 터치 영역의 넓이 인가?
```
위와 같이 전혀 다른 의미의 값으로 오해할 수 있습니다.

2. offset이 어떤 의미일까? +값이 영역의 확장인지, -값이 영역의 축소인지 직관적이지 않습니다.
>작성자의 답변은 그래픽 좌표계에 따라, Left영역은 -값이 영역의 확대를 의미하고, Bottom영역은 +값이 영역의 확대를 의미한다고 합니다.

이처럼 `확대`에 대한 의도가 영역에 따라 +/-로 서로 다른 값을 표기 해야 합니다.

### 대안으로 재시한 방법
 대부분의 UI API에서 padding/margin을 표현하는 방법이 있는데, 4방향 (Left,Top,Bottom,Right)의 두께를 표현합니다. 이 방법은 API사용자에게도 익숙하고, API사용에 있어서 오해가 없는 방법으로 제공할 수 있습니다.


### 결론
 API의 직관성은 API문서를 보지 않고서도 API의 동작을 예측할 수 있어야 합니다. 이러한 예측 가능성은 다른 API가 어떻게 동작하는지에 대한 경험으로부터 오며, 또한 해석의 여지가 여러 개가 아니고 단일한 해석이 가능한 설계에서 높은 예측 가능성이 발현됩니다. 이를 통해 API의 직관성과 사용성을 높을 수 있습니다.
