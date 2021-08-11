---
title: "[코드리뷰사례] 이벤트 Propagation 방법에 대한 구조 개선 리뷰"
layout: post
comments: true
---
코드 리뷰 사례를 공유하기 위하여 글을 작성하였습니다. 
이번 사례는 이벤트 Propagation방법에 대해 구조를 개선을 위해 리뷰했던 사례입니다.

## 문제 코드
``` c#
        /// <summary>
        /// This is used when the parent view wants to listen to gesture events.<br/>
        /// For example, the child view is overlapped on the parent view.<br/>
        /// Currently, if you tap a child, the parent cannot listen to the tap event.<br/>
        /// Now, If set to NeedGesturePropagation = true, the parent can receive gesture events.<br/>
        /// Please call this setting inside a gesture callback, it will be reset after the gesture callback is called.<br/>
        /// </summary>
        /// <code>
        /// {
        ///    View parent = new View();
        ///    View child = new View();
        ///    parent.Add(child);
        ///    TapGestureDetector parentTapDetector = new TapGestureDetector();
        ///    TapGestureDetector childTapDetector = new TapGestureDetector();
        ///    parentTapDetector.Attach(parent);
        ///    childTapDetector.Attach(child);
        ///    parentTapDetector.Detected += OnParentTap;
        ///    childTapDetector.Detected += OnChildTap;
        /// }
        /// void OnChildTap(object source, TapGestureDetector.DetectedEventArgs e)
        /// {
        ///    // If you set NeedGesturePropagation to true here, the parent view can also listen to events
        ///    child.NeedGesturePropagation = true;
        /// }
        /// </code>
        public bool NeedGesturePropagation

```

## 문제 코드에 대한 설명
 Gesture이벤트가 발생하였을때, 이를 View tree상 상위 view로 이벤트를 전파하기 위한 방법이 필요하여 View클래스에 NeedGesturePropagation Property가 추가되어 이를 설정하도록 API가 설계되었습니다.
 그런데, 해당 기능은 Gesture이벤트가 발생하였을때만 의미 있는 기능이므로, 이벤트가 발생하지 않는 시점에서의 NeedGesturePropagation Property는 아무런 동작을 하지 않는 Property가 되어 버립니다. 그리고 변수의 scope는 가능한 가장 짧게 유지하는것이 좋기 때문에 Gesture가 발생되고 처리되는 scope로 NeedGesturePropagation의 유효 범위를 한정시키는것이 clean code에 더 적합하다고 판단 하였습니다

## 대안으로 제시한 방법
 Gesture이벤트가 전달하는 EventArgs에 해당 이벤트를 Propagation 시킬지에 대한 여부를 결정하는 argument를 추가하여 이를 앱에서 설정하는 방법을 제시하였습니다.
 이는 C#을 사용하는 다른 플랫폼에서도 사용하는 방법으로, 해당 언어에 대한 경험을 갖춘 개발자에게 좀 더 익숙한 API를 제공할 수 있어서 API에 대한 사용성과 이해도를 높일 수 있다고 판단하였습니다.
 [https://docs.microsoft.com/en-us/dotnet/api/system.componentmodel.handledeventargs.handled?view=net-5.0#System_ComponentModel_HandledEventArgs_Handled](https://docs.microsoft.com/en-us/dotnet/api/system.componentmodel.handledeventargs.handled?view=net-5.0#System_ComponentModel_HandledEventArgs_Handled)


## 결론
 특정 기능을 위한 API를 추가 할때는, 해당 기능에 대한 컨택스트가 가장 적합한 곳에 추가해야 합니다. 이는 GRASP패턴의 Information Expert 원칙에 약간 근접한 내용으로 해당 행위에 대해 책임을 수행할 수 있는 객체에 그리고 그 Scope가 가장 최소가 되는 범위로 추가되는것이 바람직하다고 생각합니다.
 또 한 API 사용자의 개발 경험에 있어서 이미 존재하는 구조에 대해 동일한 방식으로 제공하게 될 경우 더 이해하기 쉽고 사용하기 편리한 API가 될 것 같습니다.

