# React Hooks 완전 정복 — Android Lifecycle 개발자를 위한 가이드

## 목차
1. [왜 Hooks인가? Class에서 Hooks로의 역사](#1-왜-hooks인가)
2. [Hooks의 규칙 (Rules of Hooks)](#2-hooks의-규칙)
3. [useState: 상태 관리의 기본](#3-usestate)
4. [useEffect: 사이드 이펙트 처리](#4-useeffect)
5. [useCallback: 함수 메모이제이션](#5-usecallback)
6. [useMemo: 값 메모이제이션](#6-usememo)
7. [useRef: DOM 참조와 값 저장](#7-useref)
8. [useContext: 전역 데이터 공유](#8-usecontext)
9. [useReducer: 복잡한 상태 로직](#9-usereducer)
10. [useTransition (React 19): 동시성 기능](#10-usetransition)
11. [useDeferredValue (React 19)](#11-usedeferredvalue)
12. [Custom Hooks: 로직 재사용](#12-custom-hooks)
13. [전체 Hooks 비교 테이블](#13-전체-hooks-비교-테이블)

---

## 1. 왜 Hooks인가?

### Class Component 시대의 문제점

React 16.8 (2019년 2월) 이전에는 상태 관리와 생명주기가 필요한 컴포넌트를 Class로 작성해야 했습니다.

```typescript
// 예제 1: Class Component (옛날 방식 — 지금은 사용하지 않음)
// 참고용으로만 보세요. 레거시 코드를 읽을 때 필요합니다.
import React, { Component } from 'react';

interface CounterState {
  count: number;
}

class Counter extends Component<{}, CounterState> {
  constructor(props: {}) {
    super(props);
    this.state = { count: 0 };
    // ⚠️ 메서드마다 this 바인딩 필요!
    this.handleIncrement = this.handleIncrement.bind(this);
  }

  componentDidMount() {
    // 마운트 시 실행 (useEffect(fn, [])와 동일)
    console.log('마운트됨');
    document.title = `카운트: ${this.state.count}`;
  }

  componentDidUpdate(prevProps: {}, prevState: CounterState) {
    // 업데이트 시 실행
    if (prevState.count !== this.state.count) {
      document.title = `카운트: ${this.state.count}`;
    }
  }

  componentWillUnmount() {
    // 언마운트 시 실행 (useEffect cleanup과 동일)
    console.log('언마운트됨');
  }

  handleIncrement() {
    this.setState(prevState => ({ count: prevState.count + 1 }));
  }

  render() {
    return (
      <div>
        <p>{this.state.count}</p>
        <button onClick={this.handleIncrement}>+1</button>
      </div>
    );
  }
}
```

```typescript
// 예제 2: 같은 기능을 Hooks로 (현대 방식)
import React, { useState, useEffect } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    document.title = `카운트: ${count}`;
    console.log('마운트 또는 count 변경됨');

    return () => {
      console.log('정리(cleanup)');
    };
  }, [count]);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </div>
  );
}
// → 코드량 절반 이하, this 바인딩 불필요, 로직이 한 곳에 모임
```

### Hooks가 해결한 문제들

```
┌──────────────────────────────────────────────────────────────────┐
│                   Hooks가 해결한 3가지 핵심 문제                  │
│                                                                  │
│  1. 관련 로직의 분산                                              │
│     Class: componentDidMount + componentDidUpdate +              │
│            componentWillUnmount에 같은 기능의 코드가 흩어짐       │
│     Hooks: useEffect 하나에 관련 로직을 모두 모을 수 있음         │
│                                                                  │
│  2. 컴포넌트 간 상태 로직 재사용 어려움                            │
│     Class: HOC(Higher Order Component), Render Props 등           │
│            복잡한 패턴 필요 → "Wrapper Hell"                      │
│     Hooks: Custom Hook으로 로직만 깔끔하게 추출/재사용             │
│                                                                  │
│  3. this 바인딩과 보일러플레이트                                  │
│     Class: constructor, this.bind, this.state, this.setState     │
│     Hooks: 일반 함수와 변수만으로 모든 것을 표현                   │
│                                                                  │
│  Android 비유:                                                    │
│  Class Component → Activity (무거움, 생명주기 복잡)               │
│  Function + Hooks → Compose (가벼움, 선언적)                     │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Hooks의 규칙

### 반드시 지켜야 하는 2가지 규칙

```typescript
// 규칙 1: 최상위에서만 호출
// Hooks는 컴포넌트 함수의 최상위(top level)에서만 호출해야 합니다.
// 조건문, 반복문, 중첩 함수 안에서 호출하면 안 됩니다.

function MyComponent({ isLoggedIn }: { isLoggedIn: boolean }) {
  // ✅ 올바른 위치: 함수 최상위
  const [name, setName] = useState('');
  const [count, setCount] = useState(0);

  // ❌ 잘못된 위치: 조건문 안
  // if (isLoggedIn) {
  //   const [user, setUser] = useState(null);  // ❌ 에러!
  // }

  // ❌ 잘못된 위치: 반복문 안
  // for (let i = 0; i < 5; i++) {
  //   const [item, setItem] = useState('');  // ❌ 에러!
  // }

  // ❌ 잘못된 위치: 일반 함수 안
  // function handleClick() {
  //   const [temp, setTemp] = useState(0);  // ❌ 에러!
  // }

  // ✅ 조건부 로직은 Hook 호출 후에
  useEffect(() => {
    if (isLoggedIn) {
      // 조건부 로직은 Hook 내부에서 처리
      console.log('로그인됨');
    }
  }, [isLoggedIn]);

  return <div>{name}</div>;
}
```

### 왜 최상위에서만 호출해야 하는가?

```
┌──────────────────────────────────────────────────────────────────┐
│                    Hooks 호출 순서의 중요성                       │
│                                                                  │
│  React는 Hooks를 "호출 순서"로 구분합니다.                        │
│  이름이 아니라 순서로! 그래서 매 렌더마다 같은 순서가 보장되어야.   │
│                                                                  │
│  첫 번째 렌더:                                                    │
│    useState('') → Hook #1 = name                                 │
│    useState(0)  → Hook #2 = count                                │
│    useEffect    → Hook #3 = effect                               │
│                                                                  │
│  두 번째 렌더 (정상):                                             │
│    useState('') → Hook #1 = name   ✅ 순서 일치                  │
│    useState(0)  → Hook #2 = count  ✅ 순서 일치                  │
│    useEffect    → Hook #3 = effect ✅ 순서 일치                  │
│                                                                  │
│  조건부 Hook이 있는 경우 (버그!):                                  │
│    if (condition) useState('')  → Hook #1 = name (조건이 true)   │
│    useState(0)  → Hook #2 = count                                │
│    useEffect    → Hook #3 = effect                               │
│                                                                  │
│    다음 렌더에서 condition이 false가 되면:                         │
│    useState(0)  → Hook #1 = ??? 🔥 name이어야 하는데 count!      │
│    useEffect    → Hook #2 = ??? 🔥 완전히 꼬여버림                │
│                                                                  │
│  Kotlin(Compose) 비유:                                           │
│  Compose도 @Composable 함수를 조건부로 호출하면 문제가 생길 수 있음│
│  → key를 사용하여 identity를 관리하는 방식으로 해결                │
│  React Hooks는 호출 순서로 identity를 관리                        │
└──────────────────────────────────────────────────────────────────┘
```

```typescript
// 규칙 2: React 함수(컴포넌트 또는 Custom Hook) 안에서만 호출
// 일반 JavaScript 함수에서는 Hook을 사용할 수 없습니다.

// ✅ React 컴포넌트에서 사용
function MyComponent() {
  const [count, setCount] = useState(0);  // ✅
  return <div>{count}</div>;
}

// ✅ Custom Hook에서 사용 (use로 시작하는 함수)
function useCounter(initialValue: number) {
  const [count, setCount] = useState(initialValue);  // ✅
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

// ❌ 일반 함수에서 사용 불가
// function calculateSomething() {
//   const [value, setValue] = useState(0);  // ❌ 에러!
// }
```

---

## 3. useState

`useState`는 03-props-and-state.md에서 자세히 다루었으므로, 여기서는 추가적인 고급 패턴만 다룹니다.

### 예제 3: 복잡한 상태를 위한 패턴

```typescript
// 패턴 1: 객체 상태의 부분 업데이트 유틸리티
function useFormState<T extends Record<string, unknown>>(initialState: T) {
  const [state, setState] = useState(initialState);

  const setField = <K extends keyof T>(field: K, value: T[K]) => {
    setState(prev => ({ ...prev, [field]: value }));
  };

  const resetForm = () => setState(initialState);

  return { formData: state, setField, resetForm, setFormData: setState };
}

// 사용법
function RegistrationForm() {
  const { formData, setField, resetForm } = useFormState({
    username: '',
    email: '',
    password: '',
    age: 0,
  });

  return (
    <form>
      <input
        value={formData.username}
        onChange={e => setField('username', e.target.value)}
      />
      <input
        value={formData.email}
        onChange={e => setField('email', e.target.value)}
      />
      <button type="button" onClick={resetForm}>초기화</button>
    </form>
  );
}
```

### 예제 4: useState로 토글/사이클 패턴

```typescript
function ToggleExamples() {
  // 불리언 토글
  const [isOpen, setIsOpen] = useState(false);
  const toggle = () => setIsOpen(prev => !prev);

  // 값 사이클 (다음 값으로 순환)
  const themes = ['light', 'dark', 'auto'] as const;
  type Theme = typeof themes[number];
  const [theme, setTheme] = useState<Theme>('light');
  const nextTheme = () => {
    setTheme(prev => {
      const currentIndex = themes.indexOf(prev);
      return themes[(currentIndex + 1) % themes.length];
    });
  };

  return (
    <div>
      <button onClick={toggle}>{isOpen ? '닫기' : '열기'}</button>
      <button onClick={nextTheme}>테마: {theme}</button>
    </div>
  );
}
```

---

## 4. useEffect

### 핵심 개념

`useEffect`는 **사이드 이펙트(side effect)**를 처리하는 Hook입니다. "사이드 이펙트"란 컴포넌트의 렌더링과 직접 관련 없는 작업을 말합니다: API 호출, 구독, 타이머, DOM 직접 조작 등.

### Android Lifecycle과의 매핑

```
┌──────────────────────────────────────────────────────────────────┐
│              Android Lifecycle ↔ useEffect 매핑                   │
│                                                                  │
│  Android                          React                          │
│  ─────────────────               ─────────────────               │
│  onCreate() / onViewCreated()    useEffect(() => {               │
│  → 초기 설정, API 호출              // 초기 설정                   │
│                                  }, [])  ← 빈 배열 = 마운트 1회   │
│                                                                  │
│  onResume() / onPause()          useEffect(() => {               │
│  → 구독 시작 / 구독 해제            // 구독 시작                   │
│                                    return () => {                │
│                                      // 구독 해제 (cleanup)       │
│                                    };                            │
│                                  }, [])                           │
│                                                                  │
│  onDestroy() / onCleared()       useEffect cleanup 함수          │
│  → 리소스 해제                    → 컴포넌트 언마운트 시 실행       │
│                                                                  │
│  LiveData.observe(lifecycle) {    useEffect(() => {               │
│    // 데이터 변경 시 실행            // 의존성 변경 시 실행          │
│  }                                }, [dependency])               │
│                                                                  │
│  LifecycleObserver               useEffect 의존성 배열            │
│  → 생명주기 이벤트 관찰            → 특정 값 변경 감지              │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 5: useEffect 의존성 배열의 3가지 형태

```typescript
function EffectDependencyDemo({ userId }: { userId: number }) {

  // ── 형태 1: 의존성 배열 없음 ──
  // 매 렌더마다 실행 (거의 사용하지 않음)
  useEffect(() => {
    console.log('매 렌더마다 실행됨');
    // ⚠️ 여기서 setState를 하면 무한 루프!
  });

  // ── 형태 2: 빈 배열 [] ──
  // 마운트 시 1회만 실행 (Android onCreate와 동일)
  useEffect(() => {
    console.log('컴포넌트가 화면에 나타남 (마운트)');

    return () => {
      console.log('컴포넌트가 화면에서 사라짐 (언마운트)');
    };
  }, []);

  // ── 형태 3: 의존성 지정 [dep1, dep2] ──
  // 지정된 값이 변경될 때만 실행
  useEffect(() => {
    console.log(`userId가 ${userId}로 변경됨, 데이터 다시 로드`);
    fetchUserData(userId);

    return () => {
      console.log(`userId ${userId}에 대한 정리`);
      // 이전 userId에 대한 정리 작업
    };
  }, [userId]);  // userId가 바뀔 때만 실행

  return <div>사용자 ID: {userId}</div>;
}
```

### 예제 6: API 호출 패턴

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    // 비동기 함수를 내부에 정의 (useEffect에 async를 직접 사용 불가)
    let isCancelled = false;  // cleanup에서 사용할 플래그

    async function fetchUser() {
      setIsLoading(true);
      setError(null);

      try {
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error('사용자를 찾을 수 없습니다');

        const data: User = await response.json();

        // ⚠️ 컴포넌트가 언마운트되었거나 userId가 바뀌었으면 state 업데이트 안 함
        if (!isCancelled) {
          setUser(data);
        }
      } catch (err) {
        if (!isCancelled) {
          setError(err instanceof Error ? err.message : '알 수 없는 에러');
        }
      } finally {
        if (!isCancelled) {
          setIsLoading(false);
        }
      }
    }

    fetchUser();

    // Cleanup: userId가 바뀌거나 언마운트 시 이전 요청 무시
    return () => {
      isCancelled = true;
    };
  }, [userId]);

  if (isLoading) return <div>로딩 중...</div>;
  if (error) return <div>에러: {error}</div>;
  if (!user) return <div>사용자 정보 없음</div>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

// Kotlin 비교: ViewModel에서의 API 호출
// class UserViewModel(private val api: UserApi) : ViewModel() {
//     private val _user = MutableStateFlow<User?>(null)
//     val user: StateFlow<User?> = _user.asStateFlow()
//
//     fun loadUser(userId: Int) {
//         viewModelScope.launch {
//             try {
//                 _user.value = api.getUser(userId)
//             } catch (e: Exception) { ... }
//         }
//     }
// }
```

### 예제 7: 구독과 이벤트 리스너 패턴

```typescript
// 패턴 1: 윈도우 이벤트 리스너
function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    // 마운트: 이벤트 리스너 등록
    window.addEventListener('resize', handleResize);

    // 언마운트: 이벤트 리스너 해제 (메모리 누수 방지!)
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);  // 빈 배열: 마운트/언마운트 시만 실행

  return <p>화면 크기: {size.width} x {size.height}</p>;
}

// 패턴 2: 타이머
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      setSeconds(prev => prev + 1);  // ⚠️ updater 함수 사용 필수!
    }, 1000);

    // cleanup: 타이머 해제
    return () => clearInterval(intervalId);
  }, []);

  return <p>{seconds}초</p>;
}

// 패턴 3: WebSocket 연결
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    // roomId가 바뀔 때마다 새 연결
    const ws = new WebSocket(`wss://chat.example.com/rooms/${roomId}`);

    ws.onmessage = (event) => {
      setMessages(prev => [...prev, event.data]);
    };

    ws.onopen = () => console.log(`${roomId} 연결됨`);

    // cleanup: 이전 room 연결 해제
    return () => {
      ws.close();
      console.log(`${roomId} 연결 해제`);
    };
  }, [roomId]);  // roomId 변경 시 재연결

  return (
    <div>
      {messages.map((msg, i) => <p key={i}>{msg}</p>)}
    </div>
  );
}
```

### useEffect 함정과 해결책

### 예제 8: 무한 루프 방지

```typescript
// ❌ 무한 루프 1: 의존성 배열 없이 setState
function InfiniteLoop1() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(count + 1);  // state 변경 → 리렌더 → useEffect 재실행 → 무한 반복!
  });  // 의존성 배열 없음 = 매 렌더마다 실행

  return <div>{count}</div>;
}

// ❌ 무한 루프 2: 매 렌더마다 새로 생성되는 객체를 의존성에
function InfiniteLoop2() {
  const [data, setData] = useState<string[]>([]);

  const options = { page: 1, limit: 10 };  // 매 렌더마다 새 객체!

  useEffect(() => {
    fetch('/api/data?' + new URLSearchParams(options as any))
      .then(res => res.json())
      .then(setData);
  }, [options]);  // ❌ options는 매 렌더마다 새 참조 → 무한 루프!

  return <div>{data.length}</div>;
}

// ✅ 해결 1: 원시 값을 의존성에 사용
function Fixed1() {
  const [data, setData] = useState<string[]>([]);
  const page = 1;
  const limit = 10;

  useEffect(() => {
    fetch(`/api/data?page=${page}&limit=${limit}`)
      .then(res => res.json())
      .then(setData);
  }, [page, limit]);  // ✅ 원시값은 비교 가능

  return <div>{data.length}</div>;
}

// ✅ 해결 2: useMemo로 객체 안정화
function Fixed2() {
  const [data, setData] = useState<string[]>([]);

  const options = useMemo(() => ({ page: 1, limit: 10 }), []);

  useEffect(() => {
    fetch('/api/data?' + new URLSearchParams(options as any))
      .then(res => res.json())
      .then(setData);
  }, [options]);  // ✅ useMemo 덕분에 같은 참조 유지

  return <div>{data.length}</div>;
}
```

### 예제 9: Stale Closure (낡은 클로저) 문제

```typescript
// ❌ Stale Closure: 오래된 값이 캡처됨
function StaleClosureDemo() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      console.log('현재 count:', count);  // ❌ 항상 0을 출력!
      // 이유: 이 클로저는 count=0일 때 생성되었고,
      // 빈 의존성 배열이므로 다시 생성되지 않음
      setCount(count + 1);  // ❌ 항상 0 + 1 = 1 → count가 1을 넘지 않음
    }, 1000);

    return () => clearInterval(intervalId);
  }, []);  // count가 의존성에 없음!

  return <div>{count}</div>;
}

// ✅ 해결 1: updater 함수 사용
function FixedWithUpdater() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const intervalId = setInterval(() => {
      setCount(prev => prev + 1);  // ✅ 항상 최신 값 기준으로 업데이트
    }, 1000);

    return () => clearInterval(intervalId);
  }, []);

  return <div>{count}</div>;
}

// ✅ 해결 2: useRef로 최신 값 참조
function FixedWithRef() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count;  // 매 렌더마다 ref를 최신 값으로 업데이트

  useEffect(() => {
    const intervalId = setInterval(() => {
      console.log('현재 count:', countRef.current);  // ✅ 항상 최신 값
      setCount(prev => prev + 1);
    }, 1000);

    return () => clearInterval(intervalId);
  }, []);

  return <div>{count}</div>;
}
```

```exercise
type: output-predict
question: "다음 코드에서 컴포넌트 마운트 시 console.log 출력 순서는?"
code: |
  function MyComponent() {
    console.log('1: render');
    useEffect(() => {
      console.log('2: effect');
      return () => console.log('3: cleanup');
    }, []);
    return <Text>Hello</Text>;
  }
options:
  - "1: render → 2: effect"
  - "2: effect → 1: render"
  - "1: render → 2: effect → 3: cleanup"
  - "2: effect → 3: cleanup → 1: render"
answer: "1: render → 2: effect"
explanation: "useEffect는 렌더링 후에 실행됩니다. cleanup은 언마운트 시에만 실행되므로 마운트 시에는 1, 2 순서입니다."
xp: 6
```

---

## 5. useCallback

### 개념

`useCallback`은 함수를 메모이제이션합니다. 의존성이 변경되지 않으면 이전에 생성한 함수와 동일한 참조를 반환합니다.

### 예제 10: useCallback이 필요한 상황

```typescript
import React, { useState, useCallback, memo } from 'react';

// memo로 감싸진 자식 컴포넌트
// props가 변경되지 않으면 리렌더를 건너뜀
const ExpensiveChild = memo(function ExpensiveChild({
  onClick,
  label,
}: {
  onClick: () => void;
  label: string;
}) {
  console.log(`${label} 렌더링됨`);
  return <button onClick={onClick}>{label}</button>;
});

// ❌ useCallback 없이
function ParentBad() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // 매 렌더마다 새 함수 생성 → ExpensiveChild가 매번 리렌더됨!
  const handleClick = () => {
    console.log('clicked');
  };

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <p>{count}</p>
      {/* text가 바뀔 때마다 handleClick이 새 참조 → 불필요한 리렌더 */}
      <ExpensiveChild onClick={handleClick} label="버튼" />
    </div>
  );
}

// ✅ useCallback 사용
function ParentGood() {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');

  // 의존성 []이므로 마운트 시 1회만 생성, 이후 같은 참조 유지
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  // count를 사용하는 함수: count가 바뀔 때만 새 함수 생성
  const handleIncrement = useCallback(() => {
    setCount(c => c + 1);
    // updater 함수를 사용하면 count를 의존성에 넣지 않아도 됨!
  }, []);

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <p>{count}</p>
      {/* handleClick 참조가 유지되므로 ExpensiveChild 리렌더 방지 */}
      <ExpensiveChild onClick={handleClick} label="버튼" />
    </div>
  );
}
```

### 예제 11: useCallback을 사용하지 말아야 할 때

```typescript
// ❌ 불필요한 useCallback — 오히려 가독성만 떨어뜨림
function SimpleComponent() {
  const [count, setCount] = useState(0);

  // 이 컴포넌트에 memo된 자식이 없다면 useCallback은 불필요
  // useCallback 자체도 비용이 있음 (의존성 비교, 메모리)
  const increment = useCallback(() => {
    setCount(c => c + 1);
  }, []);  // ❌ 과잉 최적화

  // ✅ 이렇게 하면 됨 — 단순하고 읽기 쉬움
  const incrementSimple = () => setCount(c => c + 1);

  return <button onClick={incrementSimple}>{count}</button>;
}

// ✅ useCallback이 필요한 경우들:
// 1. memo()로 감싸진 자식 컴포넌트에 콜백 전달할 때
// 2. useEffect의 의존성 배열에 함수가 들어갈 때
// 3. 리스트의 각 아이템에 콜백을 전달할 때
```

### Compose와의 비교

```kotlin
// Compose에서는 컴파일러가 자동으로 람다를 최적화함
// 개발자가 명시적으로 "이 함수를 메모이제이션해라"고 할 필요가 적음

@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }

    // Compose 컴파일러가 이 람다를 자동으로 remember 처리
    // (변경이 없으면 재생성 안 함)
    ChildButton(onClick = { count++ })
}

// React에서는 개발자가 직접 useCallback으로 메모이제이션해야 함
// → 이것이 React의 "수동 최적화" 성격
```

---

## 6. useMemo

### 개념

`useMemo`는 **계산 결과값**을 메모이제이션합니다. 의존성이 변경되지 않으면 이전 계산 결과를 재사용합니다.

### 예제 12: useMemo 사용

```typescript
import { useState, useMemo } from 'react';

interface Product {
  id: number;
  name: string;
  category: string;
  price: number;
}

function ProductList({ products }: { products: Product[] }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');
  const [theme, setTheme] = useState('light');  // UI 관련 state

  // ✅ useMemo: products, searchTerm, sortBy가 변경될 때만 재계산
  // theme이 바뀔 때는 재계산하지 않음
  const filteredAndSorted = useMemo(() => {
    console.log('필터링 + 정렬 실행'); // 이 로그로 재계산 여부 확인 가능

    const filtered = products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );

    const sorted = [...filtered].sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.price - b.price;
    });

    return sorted;
  }, [products, searchTerm, sortBy]);
  //    ↑ 이 값들이 변경될 때만 위의 계산 함수가 다시 실행됨

  // ✅ useMemo로 파생 값 계산
  const stats = useMemo(() => ({
    total: filteredAndSorted.length,
    avgPrice: filteredAndSorted.reduce((sum, p) => sum + p.price, 0) /
              (filteredAndSorted.length || 1),
    maxPrice: Math.max(...filteredAndSorted.map(p => p.price), 0),
  }), [filteredAndSorted]);

  return (
    <div className={theme}>
      <input
        value={searchTerm}
        onChange={e => setSearchTerm(e.target.value)}
        placeholder="검색..."
      />
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        테마 전환
      </button>
      {/* theme 전환 시 필터링/정렬은 재계산되지 않음 */}

      <p>총 {stats.total}개, 평균 {stats.avgPrice.toLocaleString()}원</p>

      {filteredAndSorted.map(product => (
        <div key={product.id}>
          {product.name} — {product.price.toLocaleString()}원
        </div>
      ))}
    </div>
  );
}
```

```javascript [playground]
// 🧪 useMemo 개념 실습 — 메모이제이션을 순수 JS로 이해하기

// 간단한 memoize 함수 구현
function memoize(fn) {
  let lastArgs = null;
  let lastResult = null;
  let callCount = 0;

  return function(...args) {
    // 이전 인자와 같으면 캐시된 결과 반환
    if (lastArgs && args.every((arg, i) => arg === lastArgs[i])) {
      console.log(`  → 캐시 사용 (계산 스킵)`);
      return lastResult;
    }
    callCount++;
    console.log(`  → 계산 실행 #${callCount}`);
    lastResult = fn(...args);
    lastArgs = args;
    return lastResult;
  };
}

// 비싼 계산을 시뮬레이션
const expensiveFilter = memoize((items, search) => {
  return items.filter(item =>
    item.toLowerCase().includes(search.toLowerCase())
  );
});

const products = ["노트북", "키보드", "마우스", "노트패드", "모니터"];

console.log("1차 검색 '노트':");
console.log(expensiveFilter(products, "노트"));

console.log("2차 같은 검색 '노트':");
console.log(expensiveFilter(products, "노트")); // 캐시 사용!

console.log("3차 다른 검색 '마':");
console.log(expensiveFilter(products, "마"));    // 다시 계산
```

### useMemo vs useCallback

```typescript
// useMemo: 값을 메모이제이션
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);

// useCallback: 함수를 메모이제이션
const memoizedFn = useCallback(() => { doSomething(a, b); }, [a, b]);

// 실제로 useCallback은 useMemo의 특수한 경우:
// useCallback(fn, deps) === useMemo(() => fn, deps)

// 언제 어떤 것을 사용?
// - 비용이 큰 계산 결과를 캐시 → useMemo
// - memo된 자식에게 전달하는 콜백 함수 → useCallback
// - 단순한 값이나 간단한 계산 → 둘 다 불필요 (과잉 최적화 피하기)
```

---

## 7. useRef

### 개념

`useRef`는 두 가지 용도로 사용됩니다:
1. **DOM 요소에 직접 접근** (Android의 findViewById와 유사)
2. **리렌더를 발생시키지 않는 값 저장** (변경해도 UI가 다시 그려지지 않음)

### 예제 13: DOM 참조

```typescript
import { useRef, useEffect } from 'react';

function TextInputWithFocus() {
  // useRef<타입>(초기값)으로 생성
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // 마운트 시 자동 포커스 (Android의 requestFocus()와 동일)
    inputRef.current?.focus();
  }, []);

  const handleButtonClick = () => {
    // 버튼 클릭 시 입력 필드에 포커스
    inputRef.current?.focus();
    // Android: editText.requestFocus()
  };

  return (
    <div>
      <input ref={inputRef} placeholder="여기에 포커스" />
      <button onClick={handleButtonClick}>포커스 이동</button>
    </div>
  );
}

// 스크롤 제어 예시
function ScrollToBottom() {
  const bottomRef = useRef<HTMLDivElement>(null);
  const [messages, setMessages] = useState<string[]>([]);

  useEffect(() => {
    // 메시지가 추가될 때마다 맨 아래로 스크롤
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
    // Android: recyclerView.smoothScrollToPosition(adapter.itemCount - 1)
  }, [messages]);

  return (
    <div style={{ height: '300px', overflow: 'auto' }}>
      {messages.map((msg, i) => <p key={i}>{msg}</p>)}
      <div ref={bottomRef} />  {/* 스크롤 앵커 */}
    </div>
  );
}
```

### 예제 14: 리렌더 없는 값 저장

```typescript
function StopWatch() {
  const [time, setTime] = useState(0);
  const [isRunning, setIsRunning] = useState(false);

  // intervalId를 ref로 저장 — state로 저장하면 불필요한 리렌더 발생
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // 이전 값 추적 (렌더 간 값 유지, 변경해도 리렌더 안 됨)
  const prevTimeRef = useRef(0);

  const start = () => {
    if (isRunning) return;
    setIsRunning(true);
    intervalRef.current = setInterval(() => {
      setTime(t => t + 10);
    }, 10);
  };

  const stop = () => {
    if (!isRunning) return;
    setIsRunning(false);
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
    }
    prevTimeRef.current = time;
  };

  const reset = () => {
    stop();
    setTime(0);
    prevTimeRef.current = 0;
  };

  // cleanup on unmount
  useEffect(() => {
    return () => {
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  return (
    <div>
      <p>{(time / 1000).toFixed(2)}초</p>
      <p>이전 기록: {(prevTimeRef.current / 1000).toFixed(2)}초</p>
      <button onClick={start} disabled={isRunning}>시작</button>
      <button onClick={stop} disabled={!isRunning}>정지</button>
      <button onClick={reset}>리셋</button>
    </div>
  );
}
```

### useState vs useRef 비교

```
┌──────────────────────────────┬──────────────────────────────────┐
│     useState                 │     useRef                        │
├──────────────────────────────┼──────────────────────────────────┤
│ 값 변경 시 리렌더 발생 ✅     │ 값 변경 시 리렌더 없음 ❌        │
│ 렌더링에 사용되는 데이터      │ 렌더링에 사용되지 않는 데이터     │
│ 예: 화면에 표시할 카운트      │ 예: interval ID, 이전 값         │
│ 불변 업데이트 필요            │ .current 직접 변경 가능          │
│ Compose: mutableStateOf      │ 일반 변수 (단, 렌더 간 유지)     │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 8. useContext

### 개념

`useContext`는 **props를 일일이 전달하지 않고** 컴포넌트 트리 깊숙이 데이터를 전달할 수 있게 합니다. Android의 **Hilt/Koin을 통한 의존성 주입**과 유사한 목적입니다.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Props Drilling 문제                            │
│                                                                  │
│  App (theme='dark')                                              │
│   └── Layout (theme 전달)                                        │
│       └── Sidebar (theme 전달)                                   │
│           └── Menu (theme 전달)                                  │
│               └── MenuItem (theme 사용!)                         │
│                                                                  │
│  MenuItem이 theme을 쓰려면 중간의 Layout, Sidebar, Menu가         │
│  모두 theme을 props로 받아서 전달해야 함 → "Props Drilling"       │
│                                                                  │
│  useContext 사용 시:                                              │
│  App (ThemeProvider value='dark')                                │
│   └── Layout                    ← theme 전달 불필요              │
│       └── Sidebar               ← theme 전달 불필요              │
│           └── Menu              ← theme 전달 불필요              │
│               └── MenuItem (useContext로 직접 접근!)              │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 15: Context 기본 사용

```typescript
import React, { createContext, useContext, useState } from 'react';

// 1. Context 생성 (Android: Module에서 @Provides 정의하는 것과 유사)
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

// 2. Provider 컴포넌트 (Android: @Module, @InstallIn과 유사)
function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const toggleTheme = () => setTheme(t => t === 'light' ? 'dark' : 'light');

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Custom Hook으로 Context 사용을 안전하게 (Android: @Inject와 유사)
function useTheme(): ThemeContextType {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme은 ThemeProvider 안에서 사용해야 합니다');
  }
  return context;
}

// 4. 사용
function Header() {
  const { theme, toggleTheme } = useTheme();

  return (
    <header style={{
      backgroundColor: theme === 'dark' ? '#333' : '#fff',
      color: theme === 'dark' ? '#fff' : '#333',
    }}>
      <h1>내 앱</h1>
      <button onClick={toggleTheme}>
        {theme === 'dark' ? '🌞' : '🌙'}
      </button>
    </header>
  );
}

function DeepNestedComponent() {
  const { theme } = useTheme();  // props 전달 없이 직접 접근!
  return <p>현재 테마: {theme}</p>;
}

// 5. 앱 루트에서 Provider로 감싸기
function App() {
  return (
    <ThemeProvider>
      <Header />
      <main>
        <DeepNestedComponent />
      </main>
    </ThemeProvider>
  );
}
```

### 예제 16: 인증 Context — 실전 패턴

```typescript
// AuthContext.tsx
interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // 앱 시작 시 저장된 토큰으로 사용자 정보 로드
  useEffect(() => {
    async function loadUser() {
      try {
        const token = localStorage.getItem('token');
        if (token) {
          const res = await fetch('/api/me', {
            headers: { Authorization: `Bearer ${token}` },
          });
          if (res.ok) {
            setUser(await res.json());
          }
        }
      } finally {
        setIsLoading(false);
      }
    }
    loadUser();
  }, []);

  const login = async (email: string, password: string) => {
    const res = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify({ email, password }),
      headers: { 'Content-Type': 'application/json' },
    });
    const data = await res.json();
    localStorage.setItem('token', data.token);
    setUser(data.user);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}

// 사용: 어떤 깊이의 컴포넌트에서든 접근 가능
function ProfileScreen() {
  const { user, logout } = useAuth();

  if (!user) return <p>로그인해주세요</p>;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <button onClick={logout}>로그아웃</button>
    </div>
  );
}
```

### Hilt/Koin과의 비교

```kotlin
// Android Hilt
@Module
@InstallIn(SingletonComponent::class)
object AuthModule {
    @Provides
    @Singleton
    fun provideAuthRepository(api: AuthApi): AuthRepository {
        return AuthRepositoryImpl(api)
    }
}

// Fragment에서 주입
@AndroidEntryPoint
class ProfileFragment : Fragment() {
    @Inject lateinit var authRepository: AuthRepository
    // 또는
    private val viewModel: ProfileViewModel by viewModels()
}
```

```typescript
// React Context — 같은 목적, 다른 메커니즘
// Provider로 값을 제공하고, useContext(또는 custom hook)로 주입받음
// Hilt의 @Provides ≈ Context.Provider
// Hilt의 @Inject ≈ useContext()
// Hilt의 @Singleton scope ≈ App 레벨 Provider
// Hilt의 @ActivityScope ≈ 특정 화면 레벨 Provider
```

---

## 9. useReducer

### 개념

`useReducer`는 `useState`의 대안으로, 복잡한 상태 로직을 관리할 때 사용합니다. Android의 **MVI(Model-View-Intent) 패턴** 또는 **Redux 패턴**과 매우 유사합니다.

### 예제 17: useReducer 기본 사용

```typescript
import { useReducer } from 'react';

// 1. State 타입 정의
interface CounterState {
  count: number;
  step: number;
  history: number[];
}

// 2. Action 타입 정의 (Android MVI의 Intent와 동일)
type CounterAction =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET_STEP'; payload: number }
  | { type: 'SET_COUNT'; payload: number };

// Kotlin sealed class 비교:
// sealed class CounterIntent {
//   object Increment : CounterIntent()
//   object Decrement : CounterIntent()
//   object Reset : CounterIntent()
//   data class SetStep(val step: Int) : CounterIntent()
//   data class SetCount(val count: Int) : CounterIntent()
// }

// 3. Reducer 함수 정의 (순수 함수: 이전 상태 + 액션 → 새 상태)
function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'INCREMENT':
      return {
        ...state,
        count: state.count + state.step,
        history: [...state.history, state.count + state.step],
      };
    case 'DECREMENT':
      return {
        ...state,
        count: state.count - state.step,
        history: [...state.history, state.count - state.step],
      };
    case 'RESET':
      return { count: 0, step: 1, history: [] };
    case 'SET_STEP':
      return { ...state, step: action.payload };
    case 'SET_COUNT':
      return {
        ...state,
        count: action.payload,
        history: [...state.history, action.payload],
      };
    default:
      return state;
  }
}

// Kotlin (MVI) 비교:
// fun reduce(state: CounterState, intent: CounterIntent): CounterState = when (intent) {
//     is CounterIntent.Increment -> state.copy(count = state.count + state.step)
//     is CounterIntent.Decrement -> state.copy(count = state.count - state.step)
//     is CounterIntent.Reset -> CounterState()
//     is CounterIntent.SetStep -> state.copy(step = intent.step)
//     is CounterIntent.SetCount -> state.copy(count = intent.count)
// }

// 4. 컴포넌트에서 사용
function Counter() {
  const [state, dispatch] = useReducer(counterReducer, {
    count: 0,
    step: 1,
    history: [],
  });

  return (
    <div>
      <p>카운트: {state.count} (단계: {state.step})</p>

      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>리셋</button>

      <div>
        <label>단계: </label>
        <input
          type="number"
          value={state.step}
          onChange={e => dispatch({
            type: 'SET_STEP',
            payload: parseInt(e.target.value) || 1,
          })}
        />
      </div>

      <h3>히스토리</h3>
      <ul>
        {state.history.map((value, i) => (
          <li key={i}>{value}</li>
        ))}
      </ul>
    </div>
  );
}
```

### 예제 18: 실전 — 비동기 데이터 로딩 Reducer

```typescript
// 복잡한 비동기 상태를 깔끔하게 관리

interface FetchState<T> {
  data: T | null;
  isLoading: boolean;
  error: string | null;
}

type FetchAction<T> =
  | { type: 'FETCH_START' }
  | { type: 'FETCH_SUCCESS'; payload: T }
  | { type: 'FETCH_ERROR'; payload: string }
  | { type: 'RESET' };

function fetchReducer<T>(state: FetchState<T>, action: FetchAction<T>): FetchState<T> {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, isLoading: true, error: null };
    case 'FETCH_SUCCESS':
      return { data: action.payload, isLoading: false, error: null };
    case 'FETCH_ERROR':
      return { data: null, isLoading: false, error: action.payload };
    case 'RESET':
      return { data: null, isLoading: false, error: null };
    default:
      return state;
  }
}

function UserList() {
  const [state, dispatch] = useReducer(fetchReducer<User[]>, {
    data: null,
    isLoading: false,
    error: null,
  });

  useEffect(() => {
    async function loadUsers() {
      dispatch({ type: 'FETCH_START' });
      try {
        const res = await fetch('/api/users');
        if (!res.ok) throw new Error('서버 에러');
        const data = await res.json();
        dispatch({ type: 'FETCH_SUCCESS', payload: data });
      } catch (err) {
        dispatch({
          type: 'FETCH_ERROR',
          payload: err instanceof Error ? err.message : '알 수 없는 에러',
        });
      }
    }
    loadUsers();
  }, []);

  if (state.isLoading) return <p>로딩 중...</p>;
  if (state.error) return <p>에러: {state.error}</p>;
  if (!state.data) return <p>데이터 없음</p>;

  return (
    <ul>
      {state.data.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

```javascript [playground]
// 🧪 useReducer 패턴 실습 — 순수 JavaScript로 Reducer 이해하기

// Reducer는 순수 함수: (이전상태, 액션) => 새상태
function todoReducer(state, action) {
  switch (action.type) {
    case 'ADD':
      return {
        ...state,
        todos: [...state.todos, {
          id: state.nextId,
          text: action.payload,
          done: false
        }],
        nextId: state.nextId + 1
      };
    case 'TOGGLE':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload
            ? { ...todo, done: !todo.done }
            : todo
        )
      };
    case 'DELETE':
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload)
      };
    case 'CLEAR_DONE':
      return {
        ...state,
        todos: state.todos.filter(todo => !todo.done)
      };
    default:
      return state;
  }
}

// 초기 상태
let state = { todos: [], nextId: 1 };

// dispatch 시뮬레이션
function dispatch(action) {
  state = todoReducer(state, action);
  console.log(`[${action.type}]`, JSON.stringify(state.todos));
}

dispatch({ type: 'ADD', payload: 'React 배우기' });
dispatch({ type: 'ADD', payload: 'RN 앱 만들기' });
dispatch({ type: 'ADD', payload: '앱스토어 배포' });
dispatch({ type: 'TOGGLE', payload: 1 }); // 첫 번째 완료
dispatch({ type: 'DELETE', payload: 2 }); // 두 번째 삭제

console.log("\n완료된 할일:", state.todos.filter(t => t.done).map(t => t.text));
console.log("미완료 할일:", state.todos.filter(t => !t.done).map(t => t.text));

dispatch({ type: 'CLEAR_DONE' }); // 완료된 항목 정리
console.log("정리 후:", state.todos.map(t => t.text));
```

### useState vs useReducer

```
┌──────────────────────────────┬──────────────────────────────────┐
│     useState                 │     useReducer                    │
├──────────────────────────────┼──────────────────────────────────┤
│ 단순한 상태 (1-2개 값)       │ 복잡한 상태 (여러 관련 값)        │
│ 독립적인 업데이트             │ 상태 전이 로직이 복잡할 때        │
│ 간단한 토글, 입력값           │ 폼 전체 상태, 비동기 로딩 상태    │
│ 적은 업데이트 패턴            │ 많은 업데이트 패턴 (action 타입)  │
│ 컴포넌트 안에서 로직 처리     │ Reducer 함수로 로직 분리 (테스트)│
│                              │ MVI/Redux 패턴과 동일             │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 10. useTransition

### React 19의 동시성 기능

`useTransition`은 **긴급하지 않은 상태 업데이트**를 "전환(transition)"으로 표시하여, 사용자 입력 같은 긴급한 업데이트가 차단되지 않게 합니다.

### 예제 19: useTransition

```typescript
import { useState, useTransition } from 'react';

function SearchWithTransition() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<string[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const value = e.target.value;

    // 긴급한 업데이트: 입력 필드는 즉시 반영되어야 함
    setQuery(value);

    // 긴급하지 않은 업데이트: 검색 결과는 약간 늦어도 됨
    startTransition(() => {
      // 이 안의 setState는 "낮은 우선순위"로 처리
      // 사용자가 계속 타이핑하면 이 업데이트는 중단되고 최신 것만 처리
      const filtered = hugeList.filter(item =>
        item.toLowerCase().includes(value.toLowerCase())
      );
      setResults(filtered);
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} placeholder="검색..." />

      {/* isPending: transition이 진행 중인지 여부 */}
      {isPending && <p>검색 중...</p>}

      <ul>
        {results.map(item => (
          <li key={item}>{item}</li>
        ))}
      </ul>
    </div>
  );
}

// 큰 리스트 생성 (시뮬레이션)
const hugeList = Array.from({ length: 100000 }, (_, i) => `Item ${i + 1}`);

// Android 비교:
// Compose에서 LaunchedEffect + Dispatchers.Default로 무거운 계산을 백그라운드로 처리
// React의 useTransition은 프레임워크 레벨에서 우선순위를 관리
```

---

## 11. useDeferredValue

### 개념

`useDeferredValue`는 값의 업데이트를 지연시킵니다. 새 값이 준비될 때까지 이전 값을 유지합니다.

### 예제 20: useDeferredValue

```typescript
import { useState, useDeferredValue, useMemo } from 'react';

function SearchResults() {
  const [query, setQuery] = useState('');

  // query의 "지연된 버전" — UI가 바쁠 때 이전 값을 유지
  const deferredQuery = useDeferredValue(query);

  // deferredQuery를 사용하여 비용이 큰 계산 수행
  const results = useMemo(() => {
    return hugeList.filter(item =>
      item.toLowerCase().includes(deferredQuery.toLowerCase())
    );
  }, [deferredQuery]);

  // 현재 query와 deferredQuery가 다르면 "로딩 중" 상태
  const isStale = query !== deferredQuery;

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="검색..."
      />
      <div style={{ opacity: isStale ? 0.5 : 1 }}>
        {/* 지연된 결과가 보이는 동안 투명도로 "로딩 중" 표시 */}
        <p>{results.length}개 결과</p>
        <ul>
          {results.slice(0, 100).map(item => (
            <li key={item}>{item}</li>
          ))}
        </ul>
      </div>
    </div>
  );
}

// useTransition vs useDeferredValue:
// useTransition: setState를 감싸서 "이 업데이트는 긴급하지 않다"고 표시
// useDeferredValue: 값을 감싸서 "이 값의 변경은 급하지 않다"고 표시
// 두 방법 모두 사용자 입력의 응답성을 유지하면서 비용이 큰 업데이트를 처리
```

---

## 12. Custom Hooks

### 개념

Custom Hook은 **컴포넌트 로직을 재사용 가능한 함수로 추출**하는 것입니다. `use`로 시작하는 함수 안에서 다른 Hooks를 사용합니다.

```
┌──────────────────────────────────────────────────────────────────┐
│                Custom Hook = 로직의 재사용                        │
│                                                                  │
│  Android에서의 로직 재사용:                                      │
│  - BaseViewModel → 공통 로직                                     │
│  - Repository 패턴 → 데이터 로직                                 │
│  - UseCase 클래스 → 비즈니스 로직                                │
│  - Extension Functions → 유틸리티                                │
│                                                                  │
│  React에서의 로직 재사용:                                        │
│  - Custom Hook → 상태 + 사이드 이펙트 로직                       │
│  - 일반 함수 → 순수 유틸리티 (Hooks 필요 없는 경우)               │
│                                                                  │
│  Custom Hook 네이밍 규칙:                                        │
│  - 반드시 "use"로 시작해야 함                                    │
│  - useDebounce, useLocalStorage, useNetworkStatus 등             │
│  - "use"로 시작해야 React가 Hook 규칙을 적용함                    │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 21: useDebounce — 디바운스 커스텀 Hook

```typescript
import { useState, useEffect } from 'react';

// 값의 변경을 지정된 시간만큼 지연
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    // delay 후에 값 업데이트
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // value가 변경되면 이전 타이머 취소
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 사용법: 검색 API 호출 최적화
function SearchPage() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);  // 300ms 디바운스
  const [results, setResults] = useState<string[]>([]);

  // 디바운스된 값이 변경될 때만 API 호출
  useEffect(() => {
    if (debouncedQuery) {
      fetch(`/api/search?q=${debouncedQuery}`)
        .then(res => res.json())
        .then(setResults);
    } else {
      setResults([]);
    }
  }, [debouncedQuery]);

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="검색어 입력 (300ms 디바운스)"
      />
      {/* 사용자가 타이핑을 멈추고 300ms 후에만 API 호출 */}
      <ul>
        {results.map(r => <li key={r}>{r}</li>)}
      </ul>
    </div>
  );
}
```

### 예제 22: useLocalStorage — 로컬 저장소 연동

```typescript
function useLocalStorage<T>(key: string, initialValue: T) {
  // 로컬 스토리지에서 초기값 읽기 (lazy initialization)
  const [value, setValue] = useState<T>(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  // 값이 변경될 때마다 로컬 스토리지에 저장
  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error('로컬 스토리지 저장 실패:', error);
    }
  }, [key, value]);

  return [value, setValue] as const;
}

// 사용법: 새로고침해도 값 유지
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');
  const [fontSize, setFontSize] = useLocalStorage('fontSize', 16);

  return (
    <div>
      <select value={theme} onChange={e => setTheme(e.target.value)}>
        <option value="light">라이트</option>
        <option value="dark">다크</option>
      </select>

      <input
        type="range"
        min={12}
        max={24}
        value={fontSize}
        onChange={e => setFontSize(Number(e.target.value))}
      />
      <span>{fontSize}px</span>
    </div>
  );
}

// Kotlin 비교:
// Android에서 SharedPreferences를 ViewModel에서 관리하는 것과 동일
// class SettingsRepository(private val prefs: SharedPreferences) {
//     fun getTheme(): String = prefs.getString("theme", "light") ?: "light"
//     fun setTheme(theme: String) = prefs.edit().putString("theme", theme).apply()
// }
```

### 예제 23: useNetworkStatus — 네트워크 상태 감지

```typescript
function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(
    typeof navigator !== 'undefined' ? navigator.onLine : true
  );

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
}

// 사용법
function App() {
  const isOnline = useNetworkStatus();

  return (
    <div>
      {!isOnline && (
        <div style={{ backgroundColor: 'red', color: 'white', padding: '8px' }}>
          네트워크 연결이 끊어졌습니다
        </div>
      )}
      <MainContent />
    </div>
  );
}

// Android 비교:
// class NetworkStatusLiveData(context: Context) : LiveData<Boolean>() {
//     private val connectivityManager = context.getSystemService(...)
//     private val callback = object : ConnectivityManager.NetworkCallback() {
//         override fun onAvailable(network: Network) { postValue(true) }
//         override fun onLost(network: Network) { postValue(false) }
//     }
// }
```

### 예제 24: useFetch — 범용 데이터 패칭 Hook

```typescript
interface UseFetchResult<T> {
  data: T | null;
  isLoading: boolean;
  error: string | null;
  refetch: () => void;
}

function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchData = useCallback(async () => {
    setIsLoading(true);
    setError(null);

    try {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const json = await response.json();
      setData(json);
    } catch (err) {
      setError(err instanceof Error ? err.message : '에러 발생');
    } finally {
      setIsLoading(false);
    }
  }, [url]);

  useEffect(() => {
    fetchData();
  }, [fetchData]);

  return { data, isLoading, error, refetch: fetchData };
}

// 사용법 — 매우 간결!
function UserList() {
  const { data: users, isLoading, error, refetch } = useFetch<User[]>('/api/users');

  if (isLoading) return <p>로딩 중...</p>;
  if (error) return <p>에러: {error} <button onClick={refetch}>재시도</button></p>;
  if (!users) return null;

  return (
    <ul>
      {users.map(user => <li key={user.id}>{user.name}</li>)}
    </ul>
  );
}
```

### 예제 25: useToggle — 간단한 유틸리티 Hook

```typescript
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse } as const;
}

// 사용법
function Modal() {
  const { value: isOpen, toggle, setFalse: close } = useToggle();

  return (
    <div>
      <button onClick={toggle}>모달 열기</button>
      {isOpen && (
        <div className="modal">
          <p>모달 내용</p>
          <button onClick={close}>닫기</button>
        </div>
      )}
    </div>
  );
}
```

### 예제 26: usePrevious — 이전 값 추적

```typescript
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;  // 이전 렌더의 값을 반환
}

// 사용법: 값 변경 방향 감지
function PriceDisplay({ price }: { price: number }) {
  const previousPrice = usePrevious(price);

  const direction = previousPrice === undefined ? 'none' :
    price > previousPrice ? 'up' :
    price < previousPrice ? 'down' : 'none';

  return (
    <div>
      <span>{price.toLocaleString()}원</span>
      {direction === 'up' && <span style={{ color: 'red' }}> ▲</span>}
      {direction === 'down' && <span style={{ color: 'blue' }}> ▼</span>}
    </div>
  );
}
```

### 예제 27: useInterval — 안전한 반복 타이머

```typescript
function useInterval(callback: () => void, delay: number | null) {
  const savedCallback = useRef(callback);

  // 매 렌더마다 최신 콜백을 ref에 저장
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  useEffect(() => {
    if (delay === null) return;  // delay가 null이면 타이머 중지

    const intervalId = setInterval(() => {
      savedCallback.current();  // ref에서 최신 콜백 호출
    }, delay);

    return () => clearInterval(intervalId);
  }, [delay]);
}

// 사용법
function AutoCounter() {
  const [count, setCount] = useState(0);
  const [isRunning, setIsRunning] = useState(true);

  useInterval(
    () => setCount(c => c + 1),
    isRunning ? 1000 : null,  // null이면 타이머 중지
  );

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setIsRunning(!isRunning)}>
        {isRunning ? '중지' : '시작'}
      </button>
    </div>
  );
}
```

### 예제 28: useMediaQuery — 반응형 레이아웃

```typescript
function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() =>
    typeof window !== 'undefined'
      ? window.matchMedia(query).matches
      : false
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (event: MediaQueryListEvent) => setMatches(event.matches);

    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// 사용법
function Layout() {
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');

  if (isMobile) return <MobileLayout />;
  if (isTablet) return <TabletLayout />;
  return <DesktopLayout />;
}
```

### 예제 29: Custom Hook 조합하기

```typescript
// Custom Hook은 다른 Custom Hook을 사용할 수 있음
function useSearchWithDebounce(items: string[]) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);  // 다른 custom hook 사용!
  const previousQuery = usePrevious(debouncedQuery);  // 또 다른 custom hook!

  const results = useMemo(() =>
    items.filter(item =>
      item.toLowerCase().includes(debouncedQuery.toLowerCase())
    ),
    [items, debouncedQuery]
  );

  const isSearching = query !== debouncedQuery;

  return {
    query,
    setQuery,
    results,
    isSearching,
    previousQuery,
  };
}

// 사용법 — 복잡한 로직이 Hook 안에 캡슐화됨
function SearchPage({ items }: { items: string[] }) {
  const { query, setQuery, results, isSearching } = useSearchWithDebounce(items);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {isSearching ? <p>검색 중...</p> : <p>{results.length}개 결과</p>}
      <ul>
        {results.map(r => <li key={r}>{r}</li>)}
      </ul>
    </div>
  );
}
```

### 예제 30: useForm — 폼 관리 Custom Hook

```typescript
interface UseFormOptions<T> {
  initialValues: T;
  validate: (values: T) => Partial<Record<keyof T, string>>;
  onSubmit: (values: T) => void | Promise<void>;
}

function useForm<T extends Record<string, any>>({
  initialValues,
  validate,
  onSubmit,
}: UseFormOptions<T>) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = useCallback((field: keyof T, value: any) => {
    setValues(prev => ({ ...prev, [field]: value }));
  }, []);

  const handleBlur = useCallback((field: keyof T) => {
    setTouched(prev => ({ ...prev, [field]: true }));
    // 포커스 잃을 때 검증
    const fieldErrors = validate(values);
    setErrors(prev => ({ ...prev, [field]: fieldErrors[field] }));
  }, [values, validate]);

  const handleSubmit = useCallback(async (e: React.FormEvent) => {
    e.preventDefault();
    const validationErrors = validate(values);
    setErrors(validationErrors);
    setTouched(
      Object.keys(values).reduce((acc, key) => ({ ...acc, [key]: true }), {} as Record<keyof T, boolean>)
    );

    if (Object.keys(validationErrors).length === 0) {
      setIsSubmitting(true);
      try {
        await onSubmit(values);
      } finally {
        setIsSubmitting(false);
      }
    }
  }, [values, validate, onSubmit]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
    reset,
  };
}

// 사용법
function RegistrationForm() {
  const form = useForm({
    initialValues: { name: '', email: '', password: '' },
    validate: (values) => {
      const errors: Record<string, string> = {};
      if (!values.name) errors.name = '이름은 필수입니다';
      if (!values.email.includes('@')) errors.email = '유효한 이메일을 입력하세요';
      if (values.password.length < 8) errors.password = '비밀번호는 8자 이상이어야 합니다';
      return errors;
    },
    onSubmit: async (values) => {
      await fetch('/api/register', {
        method: 'POST',
        body: JSON.stringify(values),
      });
      alert('가입 완료!');
    },
  });

  return (
    <form onSubmit={form.handleSubmit}>
      <div>
        <input
          value={form.values.name}
          onChange={e => form.handleChange('name', e.target.value)}
          onBlur={() => form.handleBlur('name')}
          placeholder="이름"
        />
        {form.touched.name && form.errors.name && (
          <span style={{ color: 'red' }}>{form.errors.name}</span>
        )}
      </div>

      <div>
        <input
          value={form.values.email}
          onChange={e => form.handleChange('email', e.target.value)}
          onBlur={() => form.handleBlur('email')}
          placeholder="이메일"
          type="email"
        />
        {form.touched.email && form.errors.email && (
          <span style={{ color: 'red' }}>{form.errors.email}</span>
        )}
      </div>

      <div>
        <input
          value={form.values.password}
          onChange={e => form.handleChange('password', e.target.value)}
          onBlur={() => form.handleBlur('password')}
          placeholder="비밀번호"
          type="password"
        />
        {form.touched.password && form.errors.password && (
          <span style={{ color: 'red' }}>{form.errors.password}</span>
        )}
      </div>

      <button type="submit" disabled={form.isSubmitting}>
        {form.isSubmitting ? '처리 중...' : '가입하기'}
      </button>
      <button type="button" onClick={form.reset}>초기화</button>
    </form>
  );
}
```

```javascript [playground]
// 🧪 Custom Hook 패턴 실습 — 로직 재사용을 순수 JS로 이해하기

// Custom Hook의 핵심: 상태 로직을 재사용 가능한 함수로 추출

// useToggle 시뮬레이션
function createToggle(initial = false) {
  let value = initial;
  return {
    get: () => value,
    toggle: () => { value = !value; return value; },
    setTrue: () => { value = true; return value; },
    setFalse: () => { value = false; return value; },
  };
}

const modal = createToggle();
console.log("초기:", modal.get());
console.log("토글:", modal.toggle());
console.log("토글:", modal.toggle());

// useLocalStorage 시뮬레이션
function createStorage(key, initialValue) {
  const storage = {};
  storage[key] = initialValue;

  return {
    get: () => storage[key],
    set: (value) => {
      storage[key] = typeof value === 'function' ? value(storage[key]) : value;
      return storage[key];
    },
  };
}

const theme = createStorage('theme', 'light');
console.log("테마:", theme.get());
theme.set('dark');
console.log("변경:", theme.get());
theme.set(prev => prev === 'dark' ? 'light' : 'dark');
console.log("토글:", theme.get());

// usePrevious 시뮬레이션
function createPrevTracker() {
  let current = undefined;
  let previous = undefined;
  return {
    update: (value) => { previous = current; current = value; },
    getCurrent: () => current,
    getPrevious: () => previous,
  };
}

const counter = createPrevTracker();
[0, 1, 2, 5, 3].forEach(v => {
  counter.update(v);
  console.log(`현재: ${counter.getCurrent()}, 이전: ${counter.getPrevious()}`);
});
```

---

## 13. 전체 Hooks 비교 테이블

| Hook | 용도 | Android 비교 | 사용 빈도 | 핵심 포인트 |
|------|------|-------------|---------|------------|
| `useState` | 컴포넌트 로컬 상태 관리 | `mutableStateOf` (Compose), `MutableStateFlow` | ★★★★★ | 가장 기본, 불변 업데이트 필수 |
| `useEffect` | 사이드 이펙트 (API, 구독, 타이머) | `LaunchedEffect` (Compose), Lifecycle callbacks | ★★★★★ | 의존성 배열 이해 필수, cleanup 함수 |
| `useCallback` | 함수 메모이제이션 | Compose 컴파일러 자동 최적화 | ★★★☆☆ | memo된 자식에 콜백 전달 시 필수 |
| `useMemo` | 값 메모이제이션 (비용 큰 계산 캐시) | `derivedStateOf` (Compose) | ★★★☆☆ | 비용 큰 계산에만, 과잉 최적화 주의 |
| `useRef` | DOM 참조, 리렌더 없는 값 저장 | `findViewById`, 일반 변수 | ★★★☆☆ | .current로 접근, 변경해도 리렌더 없음 |
| `useContext` | Props drilling 없이 깊은 데이터 전달 | Hilt/Koin DI | ★★★★☆ | Provider + Consumer 패턴 |
| `useReducer` | 복잡한 상태 로직 (다수의 액션) | MVI 패턴 / Redux | ★★★☆☆ | useState 대안, 테스트 용이 |
| `useTransition` | 긴급하지 않은 업데이트 지연 | Dispatchers.Default | ★★☆☆☆ | React 18+, isPending 상태 제공 |
| `useDeferredValue` | 값의 업데이트 지연 | debounce / throttle | ★★☆☆☆ | React 18+, useTransition의 값 버전 |
| Custom Hooks | 로직 재사용 | BaseViewModel, UseCase | ★★★★★ | `use` 접두사 필수, 조합 가능 |

### 자주 사용하는 Hook 조합 패턴

```typescript
// 패턴 1: 데이터 패칭 (가장 흔한 패턴)
// useState + useEffect
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
useEffect(() => { fetchData().then(setData).finally(() => setLoading(false)); }, []);

// 패턴 2: 폼 관리
// useState + useCallback
const [value, setValue] = useState('');
const handleChange = useCallback((e) => setValue(e.target.value), []);

// 패턴 3: 최적화된 리스트
// useState + useMemo + useCallback + memo
const filtered = useMemo(() => items.filter(...), [items, filter]);
const handleClick = useCallback((id) => setItems(...), []);

// 패턴 4: 전역 상태 공유
// createContext + useContext + useReducer
const [state, dispatch] = useReducer(reducer, initialState);
<Context.Provider value={{ state, dispatch }}>{children}</Context.Provider>
```

---

## 핵심 요약

```
┌──────────────────────────────────────────────────────────────────┐
│                    기억해야 할 핵심 개념                          │
│                                                                  │
│  1. Hooks 규칙: 최상위에서만, React 함수 안에서만 호출             │
│     → 호출 순서로 Hook을 식별하기 때문                            │
│                                                                  │
│  2. useState: 단순한 상태, useReducer: 복잡한 상태               │
│     → 둘 다 불변 업데이트 필수                                    │
│                                                                  │
│  3. useEffect: 의존성 배열이 핵심                                │
│     → [] = 마운트 1회, [dep] = dep 변경 시, 없음 = 매번           │
│     → cleanup 함수로 정리 (메모리 누수 방지)                      │
│                                                                  │
│  4. 성능 최적화: useMemo + useCallback + React.memo              │
│     → 필요할 때만 사용, 과잉 최적화 주의                          │
│                                                                  │
│  5. useRef: 리렌더 없는 값 저장 + DOM 접근                       │
│     → useState와의 차이 이해 필수                                 │
│                                                                  │
│  6. useContext: 전역 상태 공유 (Props Drilling 해결)              │
│     → Provider + Custom Hook 패턴                                │
│                                                                  │
│  7. Custom Hooks: React의 핵심 강점                              │
│     → 컴포넌트 로직을 재사용 가능한 함수로 추출                    │
│     → use 접두사 필수, 다른 Hook과 자유롭게 조합                  │
│                                                                  │
│  8. React 19 동시성: useTransition, useDeferredValue             │
│     → 사용자 입력 응답성을 유지하면서 비용 큰 업데이트 처리        │
└──────────────────────────────────────────────────────────────────┘
```

---

> **이전 문서:** [03-props-and-state.md](./03-props-and-state.md)
> **다음 단계:** [Phase 02 - 환경 설정](../phase-02-environment-setup/)
