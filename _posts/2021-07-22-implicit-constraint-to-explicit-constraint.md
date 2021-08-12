---
title: "암시적 제약을 명시적 제약으로"
layout: post
comments: true
---
API를 설계하다보면, 어떤 객체에서 특정 상태에 있을때만 사용 가능한 API를 추가하는 경우가 발생합니다. 이 상황에 해당하는 대표적인 API가 멀티미디어 API중 Player API입니다. 

![player_state_cs](https://github.sec.samsung.net/storage/user/376/files/7efea200-fb7e-11eb-9a35-6f6fa38800c1)


Player API는 Prepare를 통해 Ready상태에 도달해야지만 `Start()`, `Stop()`, `SetPlayPositionAsync()`과 같은 API가 동작하고 Ready상태가 아닌 경우에는 해당 API를 사용한 경우 예외가 발생합니다.

이 처럼 명확히 동작에 대한 제약이 존재하지만, 그 제약에 대한 명세가 단지 문서에만 설명되어 있을 뿐, 구조적으로 잘못된 상태일때 API의 사용을 못하는 것이 아니기 때문에 API만 봐서는 그러한 제약이 존재하는 알 수 가 없습니다. 
그래서 많은 상황에서 잘못된 상태에서 API를 호출하는 실수가 발생하게 됩니다.

저는 이렇게 암시적으로만 존재하는 제약사항을 좀 더 구조적인 제약으로 만들어서 API사용자가 절대 이러한 제약사항을 어길수 없도록 하는 구조로 API를 제공하게 된다면 API 사용에 있어서 더 실수가 없을 것으로 생각합니다. 그래서 이런 상황에서 적합하게 적용할 수 있는 구조에 대해 다루어 보려고 합니다.

저는 설명을 위해 아래와 같은 가상의 API를 디자인해 보았습니다.

```c#
    public class Player
    {
        
        enum State
        {
            Idle,
            Prepared,
        }

        State _state;

        public Player()
        {
            _state = State.Idle;
        }

        /// <summary>
        /// Change internal state to prepared
        /// </summary>
        public void Prepare()
        {
            if (_state != State.Idle)
                throw new InvalidOperationException("wrong state");

            // process to prepare
            _state = State.Prepared;
        }

        /// <summary>
        /// Change internal state to idle
        /// </summary>
        public void Unprepare()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to unprepare
            _state = State.Idle;
        }

        /// <summary>
        /// Start player, it only valid on prepared state
        /// </summary>
        public void Start()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to start
        }

        /// <summary>
        /// Stop player, it only valid on prepared state
        /// </summary>
        public void Stop()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to stop
        }

        /// <summary>
        /// Seek position, it only valid on prepared state
        /// </summary>
        public void Seek(int position)
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to seek
        }
    }
```

내부적으로 상태를 갖으며, 특정 상태에 도달한 경우에만 특정 API가 정상적으로 동작합니다. 이런 제약에 대한 명세는 오직 문서에만 의존하여 API사용자에게 알리 수 밖에 없는 상황입니다.
```c#
    Player player = new Player();
    player.Prepare();
    player.Start();
    player.Seek(10);
    player.Stop();
    player.Unprepare();
    player.Start(); // Exception occurred
```

이런 문제를 개선하기 위해 저는 내부의 상태 변화를 명시적으로 새로운 객체의 생성으로 표현해보려고 합니다. 

상태가 `Idle`에서 `Prepared`으로 전이가 발생한다면 명시적으로 `Prepared`의 상태를 표현하는 새로운 객체를 생성하고 이 객체에 새로운 상태 `Prepared`에서 해야할 책임을 위임하는것입니다.

기존의 상태를 전이 시키는 함수는 새로운 상태를 위한 객체를 생성하는 함수로서 역할을 하게 되고 그렇게 생성된 객체를 통해 그 상태에 적합한 API를 제공하게 됩니다.

이를 실제 코드로 반영을 한다면 아래와 같은 구조의 코드로 작성 할 수 있습니다.

```c#
    public interface IPlaybackSession : IDisposable
    {
        void Start();
        void Stop();
        void Seek(int position);
    }

    public class Player2
    {

        enum State
        {
            Idle,
            Prepared,
        }

        State _state;

        public Player2()
        {
            _state = State.Idle;
        }

        /// <summary>
        /// Change internal state to prepared
        /// </summary>
        public IPlaybackSession Prepare()
        {
            if (_state != State.Idle)
                throw new InvalidOperationException("wrong state");

            // process to prepare
            _state = State.Prepared;
            return new InternalPlaybackSession(this);
        }

        void Unprepare()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to unprepare
            _state = State.Idle;
        }

        void Start()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to start
        }

        void Stop()
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to stop
        }

        void Seek(int position)
        {
            if (_state != State.Prepared)
                throw new InvalidOperationException("wrong state");

            // process to seek
        }

        class InternalPlaybackSession : IPlaybackSession
        {
            Player2 _player;
            public InternalPlaybackSession(Player2 player)
            {
                _player = player;
            }

            public void Dispose()
            {
                _player.Unprepare();
                _player = null;
            }

            public void Seek(int position)
            {
                _player.Seek(position);
            }

            public void Start()
            {
                _player.Start();
            }

            public void Stop()
            {
                _player.Stop();
            }
        }
    }
```
`IPlaybackSession`는 새로운 상태에서 사용 가능한 API에 대한 인터페이스입니다. 그리고 그 인터페이스는 inner 클래스를 통해 구현되었습니다. 기존에 public으로 제공하던 상태에 따라 제약이 발생하는 API는 private으로 변경되어 `IPlaybackSession`를 구현한 inner클래스에 의해서 사용되도록 다시 디자인되었습니다.

그래서, 기존의 상태 전이 함수인 `Prepare`가 리턴값으로 `IPlaybackSession`인터페이스를 구현한 객체를 생성하게 되고 해당 인터페이스를 통해 Player의 내부 함수에 접근하여 해당 상태에서 제공하는 API를 사용할 수 있게 됩니다.

이 API를 앱에서 사용되게 된다면 아래와 같은 모습이 됩니다.
```c#
    Player2 player = new Player2();
    var session = player.Prepare();
    session.Start();
    session.Seek(100);
    session.Stop();
    session.Dispose();
```
`Prepare`를 통해 명시적으로 새로운 객체가 생성되고 해당 객체를 통해 해당 상태에서 동작할 수 있는 API를 사용할 수 있습니다. 만약 해당 상태에 도달하지 않은 경우라면 당연히 해당 객체를 획득하지 않은 상황이므로 해당 API를 구조적으로 호출하지 못하기 때문에 앞서 문제가 되었던 구조적으로 잘못된 상태에서 API호출이 가능했던 부분을 구조적으로 호출이 불가능하게 되었습니다.

## 마무리
API를 디자인 하는데 있어서 API사용자가 원천적으로 잘못된 접근이 불가능 하도록하는 구조를 제공한다면 좀 더 사용하기 쉬운 API가 될것입니다. 그리고 대부분의 API 사용자는 API에 문서로 설명된 명세를 세심하게 읽지 않고 API를 사용하는 경우가 많습니다. 따라서 문서로만 존재하는 제약을 만들기 보다는 구조적인 제약을 만들어서 잘못된 상황에 빠지지 않도록 API를 디자인하면 사용성이 더 높은 API가 될것 같습니다.

## 링크
 예제 코드 : [https://github.sec.samsung.net/sngnlee/ExampleCode/tree/main/Implict-to-explict](https://github.sec.samsung.net/sngnlee/ExampleCode/tree/main/Implict-to-explict)