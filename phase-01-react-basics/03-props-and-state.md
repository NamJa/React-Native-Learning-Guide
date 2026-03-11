# Props와 State — React 데이터 흐름 완전 이해

## 목차
1. [Props 개념: 부모 → 자식 데이터 전달](#1-props-개념)
2. [Props 구조 분해(Destructuring) 문법](#2-props-구조-분해)
3. [기본값(Default Props)](#3-기본값)
4. [TypeScript로 Props 타입 정의](#4-typescript-props-타입)
5. [Children Props 패턴](#5-children-props)
6. [State 개념: 컴포넌트 로컬 상태](#6-state-개념)
7. [useState 완전 가이드](#7-usestate-완전-가이드)
8. [State 불변성 규칙](#8-state-불변성-규칙)
9. [State 배칭 (React 19)](#9-state-배칭)
10. [상태 끌어올리기 (Lifting State Up)](#10-상태-끌어올리기)
11. [Controlled vs Uncontrolled 컴포넌트](#11-controlled-vs-uncontrolled)
12. [단방향 데이터 흐름](#12-단방향-데이터-흐름)
13. [Props vs State 결정 플로차트](#13-props-vs-state-결정)
14. [흔한 실수와 안티패턴 (5가지 이상)](#14-흔한-실수와-안티패턴)

---

## 1. Props 개념

### Props란?

**Props(Properties)**는 부모 컴포넌트가 자식 컴포넌트에게 전달하는 데이터입니다. 자식은 props를 **읽기 전용(read-only)**으로만 사용할 수 있습니다.

### Android와의 비교

```
┌──────────────────────────────────────────────────────────────────┐
│                  Android ↔ React Props 비교                      │
│                                                                  │
│  Android                           React                         │
│  ────────────────                 ────────────────                │
│  Intent extras                    Props                          │
│  (Activity → Activity)            (Parent → Child)               │
│                                                                  │
│  Fragment arguments               Props                          │
│  (Fragment.newInstance(args))      (Component에 속성으로 전달)      │
│                                                                  │
│  XML 속성                          JSX 속성                       │
│  android:text="Hello"             text="Hello"                   │
│  android:visibility="gone"        visible={false}                │
│                                                                  │
│  RecyclerView Adapter의           map() 안에서                    │
│  bind(item) 메서드                 <Item {...item} />             │
│                                                                  │
│  핵심 차이:                                                      │
│  - Android: Bundle/Intent는 직렬화 필요 (Parcelable/Serializable)│
│  - React: 어떤 타입이든 전달 가능 (함수, 객체, JSX 등)            │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 1: Props 기본 사용

```typescript
// 부모 컴포넌트
function App() {
  return (
    <div>
      {/* JSX 속성으로 데이터 전달 = props */}
      <UserGreeting
        name="김철수"
        age={25}
        isVip={true}
        hobbies={['독서', '코딩', '등산']}
        onLogout={() => console.log('로그아웃')}
      />
    </div>
  );
}

// 자식 컴포넌트 — props 객체로 받음
function UserGreeting(props: {
  name: string;
  age: number;
  isVip: boolean;
  hobbies: string[];
  onLogout: () => void;
}) {
  return (
    <div>
      <h1>안녕하세요, {props.name}님!</h1>
      <p>나이: {props.age}세</p>
      {props.isVip && <span className="badge">VIP</span>}
      <ul>
        {props.hobbies.map(hobby => (
          <li key={hobby}>{hobby}</li>
        ))}
      </ul>
      <button onClick={props.onLogout}>로그아웃</button>
    </div>
  );
}
```

### 예제 2: Kotlin Fragment arguments와 비교

```kotlin
// Android: Fragment에 데이터 전달
class UserFragment : Fragment() {
    companion object {
        private const val ARG_NAME = "name"
        private const val ARG_AGE = "age"

        fun newInstance(name: String, age: Int): UserFragment {
            return UserFragment().apply {
                arguments = Bundle().apply {
                    putString(ARG_NAME, name)
                    putInt(ARG_AGE, age)
                }
            }
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val name = arguments?.getString(ARG_NAME) ?: ""
        val age = arguments?.getInt(ARG_AGE) ?: 0
        // ... UI 업데이트
    }
}

// 사용
val fragment = UserFragment.newInstance("김철수", 25)
```

```typescript
// React: 컴포넌트에 데이터 전달 — 훨씬 간단!
function UserScreen() {
  return <UserProfile name="김철수" age={25} />;
}

function UserProfile({ name, age }: { name: string; age: number }) {
  return (
    <div>
      <h1>{name}</h1>
      <p>{age}세</p>
    </div>
  );
}
// → Bundle, putString, getString 같은 보일러플레이트가 필요 없음
// → 타입 안전성은 TypeScript가 보장
```

---

## 2. Props 구조 분해

JavaScript/TypeScript의 **구조 분해 할당(Destructuring)**을 사용하면 props를 깔끔하게 받을 수 있습니다.

### 예제 3: 구조 분해 할당 방식들

```typescript
interface ProfileProps {
  name: string;
  email: string;
  bio?: string;
  avatarUrl: string;
}

// 방법 1: 파라미터에서 바로 구조 분해 (가장 보편적)
function Profile({ name, email, bio, avatarUrl }: ProfileProps) {
  return (
    <div>
      <img src={avatarUrl} alt={name} />
      <h2>{name}</h2>
      <p>{email}</p>
      {bio && <p>{bio}</p>}
    </div>
  );
}

// 방법 2: 함수 본문에서 구조 분해
function Profile(props: ProfileProps) {
  const { name, email, bio, avatarUrl } = props;
  // 나머지는 동일
  return (
    <div>
      <img src={avatarUrl} alt={name} />
      <h2>{name}</h2>
    </div>
  );
}

// 방법 3: 나머지 props 수집 (rest operator)
function Profile({ name, email, ...rest }: ProfileProps) {
  // rest = { bio: '...', avatarUrl: '...' }
  console.log(rest);
  return <div>{name} - {email}</div>;
}

// Kotlin 비교:
// Kotlin에는 직접적인 구조 분해 할당 문법이 있음
// val (name, age) = Pair("김철수", 25)
// data class User(val name: String, val age: Int)
// val (name, age) = User("김철수", 25)
// React의 props 구조 분해도 같은 개념!
```

---

## 3. 기본값

### 예제 4: Props 기본값 설정

```typescript
interface ButtonProps {
  text: string;
  variant?: 'primary' | 'secondary' | 'danger';  // optional prop
  size?: 'small' | 'medium' | 'large';            // optional prop
  disabled?: boolean;                               // optional prop
  onClick?: () => void;
}

// 방법 1: 구조 분해 시 기본값 할당 (가장 보편적)
function Button({
  text,
  variant = 'primary',     // variant가 전달되지 않으면 'primary'
  size = 'medium',         // size가 전달되지 않으면 'medium'
  disabled = false,        // disabled가 전달되지 않으면 false
  onClick,
}: ButtonProps) {
  const backgroundColor =
    variant === 'primary' ? '#007AFF' :
    variant === 'secondary' ? '#6C757D' :
    '#DC3545';

  const padding =
    size === 'small' ? '4px 8px' :
    size === 'medium' ? '8px 16px' :
    '12px 24px';

  return (
    <button
      onClick={onClick}
      disabled={disabled}
      style={{ backgroundColor, padding, color: 'white', border: 'none', borderRadius: '6px' }}
    >
      {text}
    </button>
  );
}

// 사용법
function App() {
  return (
    <div>
      {/* 기본값 사용: variant='primary', size='medium' */}
      <Button text="기본 버튼" />

      {/* 일부 값만 변경 */}
      <Button text="위험" variant="danger" />

      {/* 모든 값 지정 */}
      <Button text="큰 보조 버튼" variant="secondary" size="large" />
    </div>
  );
}

// Kotlin 비교:
// fun Button(
//     text: String,
//     variant: Variant = Variant.PRIMARY,  ← Kotlin의 default parameter
//     size: Size = Size.MEDIUM,
// ) { ... }
// 개념이 완전히 동일!
```

---

## 4. TypeScript Props 타입

### 예제 5: 다양한 타입 정의 패턴

```typescript
// ── 기본 타입들 ──
interface BasicProps {
  // 원시 타입 — Kotlin 원시 타입과 1:1 대응
  name: string;           // String
  age: number;            // Int, Long, Float, Double (JS에는 숫자 타입이 하나)
  isActive: boolean;      // Boolean
}

// ── 선택적(Optional) Props ──
interface OptionalProps {
  required: string;        // 필수 — Kotlin의 non-null: String
  optional?: string;       // 선택적 — Kotlin의 nullable: String?
}

// ── 유니온 타입 (Kotlin의 sealed class / enum과 유사) ──
interface UnionProps {
  status: 'idle' | 'loading' | 'success' | 'error';
  // Kotlin: enum class Status { IDLE, LOADING, SUCCESS, ERROR }
  // 또는: sealed class Status { object Idle, object Loading, ... }

  size: 'small' | 'medium' | 'large';
  variant: 'filled' | 'outlined' | 'text';
}

// ── 함수 타입 Props — Kotlin의 고차함수와 동일 ──
interface CallbackProps {
  onClick: () => void;                              // () -> Unit
  onValueChange: (value: string) => void;           // (String) -> Unit
  onItemSelected: (item: Item, index: number) => void; // (Item, Int) -> Unit
  formatter?: (value: number) => string;            // ((Int) -> String)?
}

// ── 복합 타입 ──
interface ComplexProps {
  // 객체 — Kotlin data class
  user: {
    id: number;
    name: string;
    email: string;
  };

  // 배열 — Kotlin List<T>
  items: string[];           // List<String>
  users: User[];             // List<User>

  // 제네릭 — Kotlin의 제네릭과 동일
  // (아래에서 더 자세히)
}

// ── 제네릭 Props ──
// Kotlin: class ListComponent<T>(val items: List<T>, val renderItem: (T) -> Unit)
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

function GenericList<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <div>
      {items.map(item => (
        <div key={keyExtractor(item)}>
          {renderItem(item)}
        </div>
      ))}
    </div>
  );
}

// 사용법
function App() {
  const users = [
    { id: 1, name: '김철수' },
    { id: 2, name: '이영희' },
  ];

  return (
    <GenericList
      items={users}
      keyExtractor={user => String(user.id)}
      renderItem={user => <span>{user.name}</span>}
    />
  );
}
```

### 예제 6: 타입 확장과 조합

```typescript
// 기본 버튼 props
interface BaseButtonProps {
  text: string;
  disabled?: boolean;
}

// HTML button의 모든 속성을 상속 + 커스텀 props
interface ButtonProps extends BaseButtonProps {
  variant: 'primary' | 'secondary';
  size: 'small' | 'large';
}

// React.ComponentProps를 활용한 HTML 속성 상속
type InputProps = {
  label: string;
  error?: string;
} & React.InputHTMLAttributes<HTMLInputElement>;
// → HTML <input>의 모든 속성(type, placeholder, onChange 등)을 자동 상속

function TextInput({ label, error, ...inputProps }: InputProps) {
  return (
    <div>
      <label>{label}</label>
      <input {...inputProps} />  {/* 나머지 HTML 속성 전달 */}
      {error && <span style={{ color: 'red' }}>{error}</span>}
    </div>
  );
}

// 사용: HTML input의 모든 속성을 사용할 수 있음
<TextInput
  label="이메일"
  error="유효하지 않은 이메일입니다"
  type="email"
  placeholder="email@example.com"
  required
  autoComplete="email"
/>
```

---

## 5. Children Props

### 예제 7: Children의 다양한 활용

```typescript
// children의 타입 선택
// React.ReactNode: JSX, 문자열, 숫자, null, undefined, boolean 모두 허용 (가장 유연)
// React.ReactElement: JSX 요소만 허용
// string: 문자열만 허용
// () => React.ReactNode: 함수를 children으로 받는 패턴 (Render Props)

// 패턴 1: 기본 컨테이너
function Section({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <section>
      <h2>{title}</h2>
      <div className="section-content">{children}</div>
    </section>
  );
}

// 패턴 2: 조건부 children 래핑
function AuthGuard({ children, isLoggedIn }: {
  children: React.ReactNode;
  isLoggedIn: boolean;
}) {
  if (!isLoggedIn) {
    return <div>로그인이 필요합니다. <a href="/login">로그인</a></div>;
  }
  return <>{children}</>;
}

// 사용법
function App() {
  const [isLoggedIn] = useState(false);

  return (
    <AuthGuard isLoggedIn={isLoggedIn}>
      {/* isLoggedIn이 true일 때만 이 내용이 렌더링됨 */}
      <h1>비밀 페이지</h1>
      <p>로그인한 사용자만 볼 수 있습니다</p>
    </AuthGuard>
  );
}

// 패턴 3: Render Props — children을 함수로 받기
function DataFetcher<T>({ url, children }: {
  url: string;
  children: (data: T | null, isLoading: boolean) => React.ReactNode;
}) {
  const [data, setData] = useState<T | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(data => {
        setData(data);
        setIsLoading(false);
      });
  }, [url]);

  return <>{children(data, isLoading)}</>;
}

// 사용법
function UserList() {
  return (
    <DataFetcher<User[]> url="/api/users">
      {(users, isLoading) => {
        if (isLoading) return <p>로딩 중...</p>;
        if (!users) return <p>데이터 없음</p>;
        return (
          <ul>
            {users.map(u => <li key={u.id}>{u.name}</li>)}
          </ul>
        );
      }}
    </DataFetcher>
  );
}
```

---

## 6. State 개념

### State란?

**State**는 컴포넌트 내부에서 관리하는 변경 가능한 데이터입니다. State가 변경되면 컴포넌트가 다시 렌더링됩니다.

### Android와의 비교

```
┌──────────────────────────────────────────────────────────────────┐
│                  Android ↔ React State 비교                      │
│                                                                  │
│  Android                           React                         │
│  ────────────────────             ────────────────────            │
│  ViewModel의                      useState                       │
│  MutableStateFlow / MutableLiveData  → 상태 선언 + setter 함수    │
│                                                                  │
│  val _count = MutableStateFlow(0)  const [count, setCount]       │
│  val count: StateFlow<Int> =       = useState(0)                 │
│      _count.asStateFlow()                                        │
│                                                                  │
│  _count.value = 5                  setCount(5)                   │
│                                                                  │
│  Compose의 mutableStateOf          useState                      │
│  var count by remember {           const [count, setCount]       │
│    mutableStateOf(0)               = useState(0)                 │
│  }                                                               │
│  count = 5                         setCount(5)                   │
│                                                                  │
│  공통점:                                                         │
│  - 값이 변경되면 UI 자동 갱신                                     │
│  - 초기값 필요                                                    │
│  - 직접 변이(mutation) 불가 (setter를 통해서만)                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. useState 완전 가이드

### 예제 8: useState 기본 사용법

```typescript
import { useState } from 'react';

function Counter() {
  // useState(초기값) → [현재값, setter함수] 배열을 반환
  // 배열 구조 분해로 받음
  const [count, setCount] = useState(0);
  //     ^        ^               ^
  //     |        |               |
  //   현재값   setter함수      초기값

  return (
    <div>
      <p>카운트: {count}</p>

      {/* setter에 새 값 직접 전달 */}
      <button onClick={() => setCount(10)}>10으로 설정</button>

      {/* setter에 이전 값 기반 업데이트 함수 전달 (updater function) */}
      <button onClick={() => setCount(prev => prev + 1)}>+1</button>
      <button onClick={() => setCount(prev => prev - 1)}>-1</button>

      {/* 초기값으로 리셋 */}
      <button onClick={() => setCount(0)}>리셋</button>
    </div>
  );
}
```

**React Native에서 직접 실행해보세요:**

```jsx [snack]
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>useState 카운터</Text>
      <Text style={styles.count}>{count}</Text>

      <View style={styles.row}>
        <TouchableOpacity
          style={[styles.button, styles.danger]}
          onPress={() => setCount(prev => prev - 1)}
        >
          <Text style={styles.buttonText}>-1</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.button, styles.secondary]}
          onPress={() => setCount(0)}
        >
          <Text style={styles.buttonText}>리셋</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={[styles.button, styles.primary]}
          onPress={() => setCount(prev => prev + 1)}
        >
          <Text style={styles.buttonText}>+1</Text>
        </TouchableOpacity>
      </View>

      <TouchableOpacity
        style={[styles.button, styles.accent]}
        onPress={() => setCount(10)}
      >
        <Text style={styles.buttonText}>10으로 설정</Text>
      </TouchableOpacity>

      <Text style={styles.hint}>
        updater 함수: setCount(prev =&gt; prev + 1){'\n'}
        직접 설정: setCount(10)
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
    backgroundColor: '#f5f5f5',
  },
  title: {
    fontSize: 22,
    fontWeight: 'bold',
    marginBottom: 10,
  },
  count: {
    fontSize: 64,
    fontWeight: 'bold',
    color: '#007AFF',
    marginBottom: 20,
  },
  row: {
    flexDirection: 'row',
    gap: 12,
    marginBottom: 12,
  },
  button: {
    paddingVertical: 12,
    paddingHorizontal: 24,
    borderRadius: 8,
    minWidth: 80,
    alignItems: 'center',
  },
  primary: { backgroundColor: '#007AFF' },
  secondary: { backgroundColor: '#6C757D' },
  danger: { backgroundColor: '#DC3545' },
  accent: { backgroundColor: '#FF9500' },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  hint: {
    marginTop: 20,
    fontSize: 13,
    color: '#888',
    textAlign: 'center',
    lineHeight: 20,
  },
});

export default Counter;
```

### 예제 9: 다양한 타입의 state

```typescript
function StateExamples() {
  // 숫자
  const [count, setCount] = useState(0);

  // 문자열
  const [name, setName] = useState('');

  // boolean
  const [isVisible, setIsVisible] = useState(false);

  // 배열
  const [items, setItems] = useState<string[]>([]);

  // 객체
  const [user, setUser] = useState<{
    name: string;
    email: string;
  }>({ name: '', email: '' });

  // nullable (초기값이 null)
  const [selectedItem, setSelectedItem] = useState<string | null>(null);

  // 복합 타입
  interface FormState {
    username: string;
    password: string;
    rememberMe: boolean;
    errors: Record<string, string>;
  }

  const [form, setForm] = useState<FormState>({
    username: '',
    password: '',
    rememberMe: false,
    errors: {},
  });

  return <div>{/* ... */}</div>;
}
```

### 예제 10: Lazy Initialization — 비용이 큰 초기화

```typescript
// ❌ 매 렌더마다 비용이 큰 계산이 실행됨
function Bad() {
  // createExpensiveData()가 매 렌더마다 호출됨!
  // (반환값은 첫 렌더에서만 사용되고 이후에는 버려짐)
  const [data, setData] = useState(createExpensiveData());
  return <div>{data}</div>;
}

// ✅ 함수를 전달하면 첫 렌더에서만 실행됨 (Lazy Initialization)
function Good() {
  // 함수를 전달하면 초기 렌더 시 1회만 호출됨
  const [data, setData] = useState(() => createExpensiveData());
  return <div>{data}</div>;
}

// 실전 예시: localStorage에서 초기값 읽기
function SavedCounter() {
  const [count, setCount] = useState(() => {
    // 이 함수는 컴포넌트가 처음 마운트될 때 1번만 실행
    const saved = localStorage.getItem('count');
    return saved ? parseInt(saved, 10) : 0;
  });

  return (
    <button onClick={() => {
      const newCount = count + 1;
      setCount(newCount);
      localStorage.setItem('count', String(newCount));
    }}>
      {count}
    </button>
  );
}

// Kotlin 비교:
// val data by lazy { createExpensiveData() }
// → Kotlin의 lazy 초기화와 같은 개념
```

### 예제 11: Updater Function — 이전 값 기반 업데이트

```typescript
function UpdaterExample() {
  const [count, setCount] = useState(0);

  const incrementThree = () => {
    // ❌ 이렇게 하면 3번이 아니라 1번만 증가!
    // React는 같은 렌더 사이클에서의 state를 batching함
    // 모두 count(=0) 기준으로 0+1=1을 계산
    setCount(count + 1);  // 0 + 1 = 1
    setCount(count + 1);  // 0 + 1 = 1 (여전히 이 시점에서 count는 0)
    setCount(count + 1);  // 0 + 1 = 1

    // ✅ updater 함수를 사용하면 이전 값 기준으로 순차 적용
    setCount(prev => prev + 1);  // 0 → 1
    setCount(prev => prev + 1);  // 1 → 2
    setCount(prev => prev + 1);  // 2 → 3
  };

  return (
    <div>
      <p>{count}</p>
      <button onClick={incrementThree}>+3</button>
    </div>
  );
}

// Kotlin 비교:
// Compose에서도 비슷한 상황이 발생할 수 있음
// var count by remember { mutableStateOf(0) }
// count += 1
// count += 1  // Compose에서는 Snapshot System 덕분에 자동 처리됨
// React에서는 updater function을 명시적으로 사용해야 함
```

```exercise
type: bug-find
question: "이 코드에는 2개의 버그가 있습니다. 버그가 있는 라인을 클릭하세요."
code: |
  function Counter() {
    let [count, setCount] = useState(0);
    return (
      <View>
        <Text>{count}</Text>
        <Button
          onPress={setCount(count + 1)}
          title="증가"
        />
      </View>
    );
  }
bugLines: [2, 7]
explanations:
  2: "let 대신 const를 사용해야 합니다. useState의 반환값은 재할당하면 안 됩니다."
  7: "onPress에 함수 호출 결과가 아닌 화살표 함수를 전달해야 합니다: () => setCount(count + 1)"
xp: 10
```

```javascript [playground]
// 🧪 useState Updater 함수 개념 실습
// React의 useState updater 패턴을 순수 JavaScript로 이해해보세요

// 간단한 useState 시뮬레이터
function simulateState(initialValue) {
  let state = initialValue;
  const getState = () => state;
  const setState = (valueOrFn) => {
    if (typeof valueOrFn === 'function') {
      state = valueOrFn(state); // updater 함수: 이전 값 기반
    } else {
      state = valueOrFn; // 직접 값 설정
    }
  };
  return [getState, setState];
}

const [getCount, setCount] = simulateState(0);

// 직접 값 설정
setCount(5);
console.log("직접 설정:", getCount()); // 5

// updater 함수 사용 — 이전 값 기반 업데이트
setCount(prev => prev + 1); // 5 + 1
setCount(prev => prev + 1); // 6 + 1
setCount(prev => prev + 1); // 7 + 1
console.log("updater 3번:", getCount()); // 8

// ❌ 직접 값으로 3번 호출하면? (React에서의 함정)
setCount(0);
const currentCount = getCount(); // 0
setCount(currentCount + 1); // 0 + 1 = 1
setCount(currentCount + 1); // 0 + 1 = 1 (currentCount는 여전히 0!)
setCount(currentCount + 1); // 0 + 1 = 1
console.log("직접 값 3번:", getCount()); // 1 (3이 아님!)
```

---

## 8. State 불변성 규칙

### React에서 State를 직접 변경하면 안 되는 이유

React는 상태가 변경되었는지를 **참조(reference) 비교**로 판단합니다. 객체나 배열을 직접 변경하면 참조가 같으므로 React가 변경을 감지하지 못합니다.

### 예제 12: 배열 State 불변 업데이트

```typescript
function TodoList() {
  const [todos, setTodos] = useState<string[]>(['리액트 배우기']);

  // ❌ 직접 변경 — React가 변경을 감지하지 못함!
  const addTodoBad = () => {
    todos.push('새 할일');   // 원본 배열을 직접 변경 (mutation)
    setTodos(todos);         // 같은 참조 → React: "변경 없음" → 리렌더 안 됨!
  };

  // ✅ 새 배열 생성 — React가 변경을 감지함
  const addTodoGood = () => {
    setTodos([...todos, '새 할일']);  // 스프레드로 새 배열 생성
    // 또는
    setTodos(prev => [...prev, '새 할일']);  // updater 함수 사용
  };

  // ✅ 아이템 제거
  const removeTodo = (index: number) => {
    setTodos(prev => prev.filter((_, i) => i !== index));
  };

  // ✅ 아이템 수정
  const updateTodo = (index: number, newText: string) => {
    setTodos(prev => prev.map((todo, i) =>
      i === index ? newText : todo
    ));
  };

  // ✅ 특정 위치에 삽입
  const insertTodo = (index: number, text: string) => {
    setTodos(prev => [
      ...prev.slice(0, index),
      text,
      ...prev.slice(index),
    ]);
  };

  return (
    <ul>
      {todos.map((todo, i) => (
        <li key={i}>{todo}</li>
      ))}
    </ul>
  );
}
```

### 예제 13: 객체 State 불변 업데이트

```typescript
interface UserProfile {
  name: string;
  email: string;
  address: {
    city: string;
    zipCode: string;
  };
  tags: string[];
}

function ProfileEditor() {
  const [profile, setProfile] = useState<UserProfile>({
    name: '김철수',
    email: 'cs@mail.com',
    address: { city: '서울', zipCode: '06001' },
    tags: ['개발자', 'React'],
  });

  // ❌ 직접 변경 — 절대 하면 안 됨!
  const updateNameBad = () => {
    profile.name = '이영희';  // 직접 mutation
    setProfile(profile);       // 같은 참조 → 리렌더 안 됨!
  };

  // ✅ 스프레드 연산자로 새 객체 생성
  const updateName = (name: string) => {
    setProfile(prev => ({ ...prev, name }));
    // { ...prev } = 이전 객체의 모든 속성을 복사
    // name = name 속성만 새 값으로 덮어씀
  };

  // ✅ 중첩 객체 업데이트
  const updateCity = (city: string) => {
    setProfile(prev => ({
      ...prev,
      address: {
        ...prev.address,  // address 내부도 스프레드 필요!
        city,
      },
    }));
  };

  // ✅ 배열 속성에 아이템 추가
  const addTag = (tag: string) => {
    setProfile(prev => ({
      ...prev,
      tags: [...prev.tags, tag],
    }));
  };

  // ✅ 배열 속성에서 아이템 제거
  const removeTag = (tagToRemove: string) => {
    setProfile(prev => ({
      ...prev,
      tags: prev.tags.filter(tag => tag !== tagToRemove),
    }));
  };

  return (
    <div>
      <input
        value={profile.name}
        onChange={e => updateName(e.target.value)}
      />
      <input
        value={profile.address.city}
        onChange={e => updateCity(e.target.value)}
      />
      <div>
        {profile.tags.map(tag => (
          <span key={tag}>
            {tag}
            <button onClick={() => removeTag(tag)}>×</button>
          </span>
        ))}
      </div>
    </div>
  );
}
```

### Kotlin과의 비교

```kotlin
// Kotlin에서의 불변성 개념

// val list = listOf("a", "b")    // 불변 리스트 — React의 State와 유사
// list.add("c")  ← 컴파일 에러! 불변이므로 추가 불가

// val mutableList = mutableListOf("a", "b")  // 가변 리스트
// mutableList.add("c")  ← 가능하지만, React에서는 이런 방식이 금지됨

// React에서는 항상 "새 리스트"를 만들어야 함:
// Kotlin의 list + "c" (새 리스트 반환) 과 같은 패턴
// val newList = list + "c"  ← React의 [...items, newItem]과 동일

// data class에서의 copy()도 같은 패턴:
// data class User(val name: String, val age: Int)
// val user = User("김철수", 25)
// val updated = user.copy(age = 26)  ← React의 {...user, age: 26}과 동일
```

```javascript [playground]
// 🧪 State 불변성 패턴 실습
// React에서 State를 업데이트하는 올바른 패턴을 연습해보세요

// === 배열 불변 업데이트 ===
const todos = ["리액트 배우기", "RN 시작하기"];

// ✅ 추가 (push 대신 spread)
const added = [...todos, "앱 배포하기"];
console.log("추가:", added);

// ✅ 제거 (splice 대신 filter)
const removed = todos.filter((_, i) => i !== 0);
console.log("제거:", removed);

// ✅ 수정 (직접 변경 대신 map)
const updated = todos.map((todo, i) =>
  i === 0 ? "React Native 배우기" : todo
);
console.log("수정:", updated);

// ✅ 특정 위치에 삽입
const inserted = [...todos.slice(0, 1), "새 항목", ...todos.slice(1)];
console.log("삽입:", inserted);

console.log("원본 유지:", todos); // 원본은 변하지 않음!

// === 객체 불변 업데이트 ===
const user = {
  name: "김철수",
  address: { city: "서울", zip: "06001" },
  tags: ["개발자"]
};

// ✅ 속성 변경
const nameChanged = { ...user, name: "이영희" };
console.log("이름 변경:", nameChanged.name);

// ✅ 중첩 객체 변경 (address 내부)
const cityChanged = {
  ...user,
  address: { ...user.address, city: "부산" }
};
console.log("도시 변경:", cityChanged.address.city);
console.log("원본 도시:", user.address.city); // "서울" 유지!

// ✅ 배열 속성에 추가
const tagAdded = { ...user, tags: [...user.tags, "React"] };
console.log("태그 추가:", tagAdded.tags);
```

---

## 9. State 배칭

### React 19의 자동 배칭 (Automatic Batching)

React 19(및 18+)에서는 모든 상태 업데이트가 자동으로 **배칭(batching)**됩니다. 여러 `setState` 호출이 하나의 리렌더로 합쳐집니다.

### 예제 14: 배칭 동작

```typescript
function BatchingExample() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);
  const [text, setText] = useState('');

  console.log('렌더링!'); // 아래 버튼을 클릭하면 몇 번 출력될까?

  const handleClick = () => {
    // React 19: 이 세 개의 setState가 하나의 리렌더로 배칭됨
    setCount(c => c + 1);
    setFlag(f => !f);
    setText('updated');
    // → console.log('렌더링!')이 1번만 실행됨 (3번이 아님!)
  };

  // async 함수 안에서도 배칭됨 (React 18 이전에는 안 됐었음)
  const handleAsyncClick = async () => {
    const response = await fetch('/api/data');
    const data = await response.json();

    // React 19: async/await 뒤에서도 배칭됨!
    setCount(data.count);
    setFlag(data.flag);
    setText(data.text);
    // → 여전히 1번의 리렌더만 발생
  };

  // setTimeout/setInterval 안에서도 배칭됨
  const handleTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setFlag(f => !f);
      // → React 19에서는 배칭됨 (1번 리렌더)
      // → React 17에서는 배칭 안 됨 (2번 리렌더)
    }, 1000);
  };

  return (
    <div>
      <p>count: {count}, flag: {String(flag)}, text: {text}</p>
      <button onClick={handleClick}>동기 업데이트</button>
      <button onClick={handleAsyncClick}>비동기 업데이트</button>
      <button onClick={handleTimeout}>타임아웃 업데이트</button>
    </div>
  );
}

// Kotlin (Compose) 비교:
// Compose도 같은 프레임에서의 여러 상태 변경을 배칭함
// var a by remember { mutableStateOf(0) }
// var b by remember { mutableStateOf(false) }
// a = 1
// b = true
// → 하나의 recomposition만 발생
```

---

## 10. 상태 끌어올리기

### Lifting State Up

두 개 이상의 컴포넌트가 같은 데이터를 공유해야 할 때, 그 데이터를 가장 가까운 공통 부모로 "끌어올립니다."

### Android와의 비교

```
┌──────────────────────────────────────────────────────────────────┐
│           상태 끌어올리기 — Android ↔ React                       │
│                                                                  │
│  Android: SharedViewModel 패턴                                   │
│  ─────────────────────────────                                   │
│  Activity (또는 NavGraph)                                        │
│    ├── FragmentA  ← sharedViewModel.data.observe()               │
│    └── FragmentB  ← sharedViewModel.data.observe()               │
│  sharedViewModel은 Activity scope에서 공유                        │
│                                                                  │
│  React: Lifting State Up 패턴                                    │
│  ─────────────────────────────                                   │
│  ParentComponent  ← state를 여기서 관리                           │
│    ├── ChildA  ← props로 state 전달                              │
│    └── ChildB  ← props로 state 전달                              │
│  상태와 setter 함수를 props로 내려보냄                             │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 15: 온도 변환기 — 상태 끌어올리기 전

```typescript
// ❌ 각 입력이 독립적인 상태를 가짐 — 동기화 불가
function CelsiusInput() {
  const [temp, setTemp] = useState('');
  return (
    <div>
      <label>섭씨: </label>
      <input value={temp} onChange={e => setTemp(e.target.value)} />°C
    </div>
  );
}

function FahrenheitInput() {
  const [temp, setTemp] = useState('');
  return (
    <div>
      <label>화씨: </label>
      <input value={temp} onChange={e => setTemp(e.target.value)} />°F
    </div>
  );
}

// 이 상태에서는 섭씨를 입력해도 화씨가 자동으로 변환되지 않음!
```

### 예제 16: 온도 변환기 — 상태 끌어올리기 후

```typescript
// ✅ 상태를 부모로 끌어올림 — 두 입력이 동기화됨

// 자식: 상태를 갖지 않고, props로 받기만 함
function TemperatureInput({
  scale,
  temperature,
  onTemperatureChange,
}: {
  scale: '섭씨' | '화씨';
  temperature: string;
  onTemperatureChange: (temp: string) => void;
}) {
  return (
    <div>
      <label>{scale}: </label>
      <input
        value={temperature}
        onChange={e => onTemperatureChange(e.target.value)}
      />
      °{scale === '섭씨' ? 'C' : 'F'}
    </div>
  );
}

// 변환 함수
function toCelsius(fahrenheit: number): number {
  return (fahrenheit - 32) * 5 / 9;
}

function toFahrenheit(celsius: number): number {
  return celsius * 9 / 5 + 32;
}

// 부모: 상태를 관리하고 자식에게 전달
function TemperatureCalculator() {
  // "하나의 진실 공급원(Single Source of Truth)"
  const [temperature, setTemperature] = useState('');
  const [scale, setScale] = useState<'C' | 'F'>('C');

  const handleCelsiusChange = (temp: string) => {
    setScale('C');
    setTemperature(temp);
  };

  const handleFahrenheitChange = (temp: string) => {
    setScale('F');
    setTemperature(temp);
  };

  // 현재 값에서 다른 단위 계산
  const celsius = scale === 'F' && temperature
    ? toCelsius(parseFloat(temperature)).toFixed(2)
    : temperature;
  const fahrenheit = scale === 'C' && temperature
    ? toFahrenheit(parseFloat(temperature)).toFixed(2)
    : temperature;

  return (
    <div>
      <TemperatureInput
        scale="섭씨"
        temperature={celsius}
        onTemperatureChange={handleCelsiusChange}
      />
      <TemperatureInput
        scale="화씨"
        temperature={fahrenheit}
        onTemperatureChange={handleFahrenheitChange}
      />

      {temperature && (
        <p>
          {parseFloat(celsius) >= 100
            ? '물이 끓습니다!'
            : '물이 끓지 않습니다.'}
        </p>
      )}
    </div>
  );
}
```

**React Native 온도 변환기를 직접 실행해보세요:**

```jsx [snack]
import React, { useState } from 'react';
import { View, Text, TextInput, StyleSheet } from 'react-native';

function toCelsius(fahrenheit) {
  return ((fahrenheit - 32) * 5) / 9;
}

function toFahrenheit(celsius) {
  return (celsius * 9) / 5 + 32;
}

function TemperatureInput({ scale, temperature, onTemperatureChange }) {
  const unit = scale === '섭씨' ? '°C' : '°F';
  return (
    <View style={styles.inputGroup}>
      <Text style={styles.label}>{scale} ({unit})</Text>
      <TextInput
        style={styles.input}
        value={temperature}
        onChangeText={onTemperatureChange}
        keyboardType="numeric"
        placeholder={`${scale} 온도 입력`}
      />
    </View>
  );
}

function TemperatureCalculator() {
  const [temperature, setTemperature] = useState('');
  const [scale, setScale] = useState('C');

  const handleCelsiusChange = (temp) => {
    setScale('C');
    setTemperature(temp);
  };

  const handleFahrenheitChange = (temp) => {
    setScale('F');
    setTemperature(temp);
  };

  const celsius =
    scale === 'F' && temperature
      ? toCelsius(parseFloat(temperature)).toFixed(2)
      : temperature;
  const fahrenheit =
    scale === 'C' && temperature
      ? toFahrenheit(parseFloat(temperature)).toFixed(2)
      : temperature;

  const celsiusNum = parseFloat(celsius);
  const isBoiling = !isNaN(celsiusNum) && celsiusNum >= 100;

  return (
    <View style={styles.container}>
      <Text style={styles.title}>온도 변환기</Text>
      <Text style={styles.subtitle}>상태 끌어올리기 (Lifting State Up)</Text>

      <TemperatureInput
        scale="섭씨"
        temperature={String(celsius)}
        onTemperatureChange={handleCelsiusChange}
      />
      <TemperatureInput
        scale="화씨"
        temperature={String(fahrenheit)}
        onTemperatureChange={handleFahrenheitChange}
      />

      {temperature !== '' && (
        <View style={[styles.result, isBoiling && styles.resultBoiling]}>
          <Text style={styles.resultText}>
            {isBoiling ? '🔥 물이 끓습니다!' : '❄️ 물이 끓지 않습니다.'}
          </Text>
        </View>
      )}

      <Text style={styles.hint}>
        두 입력이 하나의 state를 공유합니다.{'\n'}
        부모가 state를 관리하고, 자식은 props로 받습니다.
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    padding: 24,
    backgroundColor: '#f5f5f5',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    textAlign: 'center',
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    marginBottom: 24,
  },
  inputGroup: {
    marginBottom: 16,
  },
  label: {
    fontSize: 16,
    fontWeight: '600',
    marginBottom: 6,
    color: '#333',
  },
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
    fontSize: 18,
    backgroundColor: 'white',
  },
  result: {
    marginTop: 20,
    padding: 16,
    borderRadius: 8,
    backgroundColor: '#E3F2FD',
    alignItems: 'center',
  },
  resultBoiling: {
    backgroundColor: '#FFEBEE',
  },
  resultText: {
    fontSize: 18,
    fontWeight: '600',
  },
  hint: {
    marginTop: 24,
    fontSize: 13,
    color: '#888',
    textAlign: 'center',
    lineHeight: 20,
  },
});

export default TemperatureCalculator;
```

### 예제 17: SharedViewModel 패턴과 비교

```kotlin
// Android: SharedViewModel으로 Fragment 간 데이터 공유
class SharedViewModel : ViewModel() {
    private val _temperature = MutableStateFlow("")
    val temperature: StateFlow<String> = _temperature.asStateFlow()

    private val _scale = MutableStateFlow("C")
    val scale: StateFlow<String> = _scale.asStateFlow()

    fun setCelsius(temp: String) {
        _scale.value = "C"
        _temperature.value = temp
    }

    fun setFahrenheit(temp: String) {
        _scale.value = "F"
        _temperature.value = temp
    }
}

// Fragment에서 사용
class CelsiusFragment : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        sharedViewModel.temperature.collect { temp ->
            binding.input.setText(temp)
        }
    }
}
```

```typescript
// React에서는 부모 컴포넌트가 ViewModel 역할
// SharedViewModel === 부모 컴포넌트의 state + handler 함수들
function TemperatureCalculator() {
  // ViewModel의 역할을 하는 부분
  const [temperature, setTemperature] = useState('');
  const [scale, setScale] = useState<'C' | 'F'>('C');

  // ViewModel의 메서드에 해당
  const setCelsius = (temp: string) => { setScale('C'); setTemperature(temp); };
  const setFahrenheit = (temp: string) => { setScale('F'); setTemperature(temp); };

  // Fragment에 해당하는 자식 컴포넌트에 props로 전달
  return (
    <>
      <CelsiusInput temperature={temperature} onChange={setCelsius} />
      <FahrenheitInput temperature={temperature} onChange={setFahrenheit} />
    </>
  );
}
```

---

## 11. Controlled vs Uncontrolled

### 예제 18: Controlled Component (권장)

```typescript
// Controlled: React state가 "진실의 근원"
// Android의 two-way data binding과 유사하지만, 단방향!
function ControlledForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    console.log('제출:', { name, email });
    // → name, email state가 항상 최신값을 가짐
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* value와 onChange를 모두 지정 = Controlled */}
      <input
        value={name}              // React state → input 표시값
        onChange={e => setName(e.target.value)}  // input 변경 → React state 업데이트
      />

      <input
        value={email}
        onChange={e => setEmail(e.target.value)}
      />

      {/* 실시간 검증이 가능 — state가 항상 최신이므로 */}
      {email && !email.includes('@') && (
        <p style={{ color: 'red' }}>유효한 이메일을 입력하세요</p>
      )}

      <button type="submit" disabled={!name || !email.includes('@')}>
        제출
      </button>
    </form>
  );
}
```

### 예제 19: Uncontrolled Component

```typescript
import { useRef } from 'react';

// Uncontrolled: DOM이 "진실의 근원" — React가 제어하지 않음
function UncontrolledForm() {
  const nameRef = useRef<HTMLInputElement>(null);
  const emailRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    // 제출 시점에 DOM에서 직접 값을 읽음
    console.log('제출:', {
      name: nameRef.current?.value,
      email: emailRef.current?.value,
    });
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* ref만 지정, value/onChange 없음 = Uncontrolled */}
      <input ref={nameRef} defaultValue="" />
      <input ref={emailRef} defaultValue="" />
      <button type="submit">제출</button>
    </form>
  );

  // ⚠️ Uncontrolled의 단점:
  // - 실시간 검증 어려움 (값이 state에 없으므로)
  // - 입력값 기반 조건부 렌더링 불가
  // - 여러 입력 간 연동 어려움
  // → 대부분의 경우 Controlled를 사용하는 것이 바람직
}
```

**React Native Controlled Form을 직접 실행해보세요:**

```jsx [snack]
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, StyleSheet, ScrollView } from 'react-native';

function ControlledForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [submitted, setSubmitted] = useState(false);
  const [result, setResult] = useState(null);

  const emailValid = email.length === 0 || email.includes('@');
  const canSubmit = name.length > 0 && email.includes('@');

  const handleSubmit = () => {
    setSubmitted(true);
    if (canSubmit) {
      setResult({ name, email });
    }
  };

  const handleReset = () => {
    setName('');
    setEmail('');
    setSubmitted(false);
    setResult(null);
  };

  return (
    <ScrollView contentContainerStyle={styles.container}>
      <Text style={styles.title}>Controlled Component</Text>
      <Text style={styles.subtitle}>React state가 "진실의 근원"</Text>

      <View style={styles.field}>
        <Text style={styles.label}>이름</Text>
        <TextInput
          style={[styles.input, submitted && !name && styles.inputError]}
          value={name}
          onChangeText={setName}
          placeholder="이름을 입력하세요"
        />
        {submitted && !name && (
          <Text style={styles.error}>이름을 입력해주세요</Text>
        )}
      </View>

      <View style={styles.field}>
        <Text style={styles.label}>이메일</Text>
        <TextInput
          style={[styles.input, submitted && !emailValid && styles.inputError]}
          value={email}
          onChangeText={setEmail}
          placeholder="email@example.com"
          keyboardType="email-address"
          autoCapitalize="none"
        />
        {!emailValid && (
          <Text style={styles.error}>유효한 이메일을 입력하세요</Text>
        )}
      </View>

      <View style={styles.statePreview}>
        <Text style={styles.previewTitle}>현재 State:</Text>
        <Text style={styles.previewText}>name: "{name}"</Text>
        <Text style={styles.previewText}>email: "{email}"</Text>
        <Text style={styles.previewText}>
          canSubmit: {canSubmit ? 'true ✅' : 'false ❌'}
        </Text>
      </View>

      <View style={styles.row}>
        <TouchableOpacity
          style={[styles.button, !canSubmit && styles.buttonDisabled]}
          onPress={handleSubmit}
        >
          <Text style={styles.buttonText}>제출</Text>
        </TouchableOpacity>
        <TouchableOpacity
          style={[styles.button, styles.buttonSecondary]}
          onPress={handleReset}
        >
          <Text style={styles.buttonText}>초기화</Text>
        </TouchableOpacity>
      </View>

      {result && (
        <View style={styles.resultBox}>
          <Text style={styles.resultTitle}>제출 완료!</Text>
          <Text style={styles.resultText}>이름: {result.name}</Text>
          <Text style={styles.resultText}>이메일: {result.email}</Text>
        </View>
      )}
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flexGrow: 1,
    justifyContent: 'center',
    padding: 24,
    backgroundColor: '#f5f5f5',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 14,
    color: '#666',
    textAlign: 'center',
    marginBottom: 24,
  },
  field: {
    marginBottom: 16,
  },
  label: {
    fontSize: 15,
    fontWeight: '600',
    marginBottom: 6,
    color: '#333',
  },
  input: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    backgroundColor: 'white',
  },
  inputError: {
    borderColor: '#DC3545',
  },
  error: {
    color: '#DC3545',
    fontSize: 13,
    marginTop: 4,
  },
  statePreview: {
    backgroundColor: '#E8F5E9',
    padding: 14,
    borderRadius: 8,
    marginBottom: 16,
  },
  previewTitle: {
    fontWeight: 'bold',
    marginBottom: 4,
    fontSize: 14,
  },
  previewText: {
    fontSize: 13,
    fontFamily: 'monospace',
    color: '#333',
    lineHeight: 20,
  },
  row: {
    flexDirection: 'row',
    gap: 12,
  },
  button: {
    flex: 1,
    backgroundColor: '#007AFF',
    paddingVertical: 14,
    borderRadius: 8,
    alignItems: 'center',
  },
  buttonSecondary: {
    backgroundColor: '#6C757D',
  },
  buttonDisabled: {
    opacity: 0.5,
  },
  buttonText: {
    color: 'white',
    fontSize: 16,
    fontWeight: '600',
  },
  resultBox: {
    marginTop: 20,
    backgroundColor: '#E3F2FD',
    padding: 16,
    borderRadius: 8,
  },
  resultTitle: {
    fontWeight: 'bold',
    fontSize: 16,
    marginBottom: 6,
  },
  resultText: {
    fontSize: 14,
    color: '#333',
  },
});

export default ControlledForm;
```

### 언제 어떤 것을 사용할까?

```
┌──────────────────────────────┬──────────────────────────────────┐
│     Controlled (권장)        │     Uncontrolled                  │
├──────────────────────────────┼──────────────────────────────────┤
│ 실시간 입력 검증 필요         │ 파일 업로드 (<input type="file">)│
│ 조건부 비활성화/표시          │ 비React 라이브러리 통합           │
│ 입력 형식 제한 (예: 숫자만)   │ 성능이 매우 중요한 경우          │
│ 여러 입력 간 연동             │ 간단한 일회성 폼                 │
│ 즉각적인 UI 반응             │                                  │
└──────────────────────────────┴──────────────────────────────────┘
```

---

## 12. 단방향 데이터 흐름

### 예제 20: 단방향 데이터 흐름 시각화

```
┌──────────────────────────────────────────────────────────────────┐
│                 React 단방향 데이터 흐름                          │
│                                                                  │
│                    ┌──────────────┐                               │
│                    │     App      │                               │
│                    │  state: {    │                               │
│                    │   users: []  │                               │
│                    │   filter: '' │                               │
│                    │  }           │                               │
│                    └──────┬───────┘                               │
│                           │                                       │
│              ┌────────────┼────────────┐                          │
│              │ props      │ props      │                          │
│              ▼            ▼            ▼                          │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                    │
│        │ SearchBar│ │ UserList │ │ Summary  │                    │
│        │          │ │          │ │          │                    │
│        │ filter   │ │ users    │ │ count    │                    │
│        │ onChange │ │ filter   │ │          │                    │
│        └────┬─────┘ └──────────┘ └──────────┘                    │
│             │                                                     │
│             │ onChange 콜백 호출                                   │
│             │ (자식 → 부모로 이벤트 전달)                          │
│             │                                                     │
│             └───→ App의 setFilter() 호출                          │
│                   → state 변경                                    │
│                   → 모든 자식 리렌더                               │
│                   → 새 props로 UI 갱신                            │
│                                                                  │
│   데이터: 위에서 아래로 (props) ↓                                 │
│   이벤트: 아래에서 위로 (콜백) ↑                                  │
│   → 양방향 바인딩이 아님!                                        │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 21: 전체 코드로 보는 단방향 흐름

```typescript
interface User {
  id: number;
  name: string;
  department: string;
}

// 부모: 상태 관리
function App() {
  const [users] = useState<User[]>([
    { id: 1, name: '김철수', department: '개발팀' },
    { id: 2, name: '이영희', department: '디자인팀' },
    { id: 3, name: '박민수', department: '개발팀' },
    { id: 4, name: '최지영', department: '기획팀' },
  ]);
  const [filter, setFilter] = useState('');

  // 파생 데이터 (state로부터 계산)
  const filteredUsers = users.filter(u =>
    u.name.includes(filter) || u.department.includes(filter)
  );

  return (
    <div>
      {/* 데이터 ↓: filter 값과 변경 콜백을 props로 전달 */}
      <SearchBar
        value={filter}
        onChange={setFilter}  // 이벤트 ↑: 자식이 이 함수를 호출하면 부모 state 변경
      />

      {/* 데이터 ↓: 필터링된 결과를 props로 전달 */}
      <UserList users={filteredUsers} />

      {/* 데이터 ↓: 카운트를 props로 전달 */}
      <Summary
        total={users.length}
        filtered={filteredUsers.length}
      />
    </div>
  );
}

// 자식 1: 검색 입력
function SearchBar({ value, onChange }: {
  value: string;
  onChange: (value: string) => void;
}) {
  return (
    <input
      value={value}
      onChange={e => onChange(e.target.value)}  // 이벤트 ↑
      placeholder="이름 또는 부서 검색..."
    />
  );
}

// 자식 2: 사용자 리스트
function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name} ({user.department})</li>
      ))}
    </ul>
  );
}

// 자식 3: 요약
function Summary({ total, filtered }: { total: number; filtered: number }) {
  return <p>전체 {total}명 중 {filtered}명 표시</p>;
}
```

**React Native 단방향 데이터 흐름을 직접 실행해보세요:**

```jsx [snack]
import React, { useState } from 'react';
import { View, Text, TextInput, FlatList, StyleSheet } from 'react-native';

const USERS = [
  { id: '1', name: '김철수', department: '개발팀' },
  { id: '2', name: '이영희', department: '디자인팀' },
  { id: '3', name: '박민수', department: '개발팀' },
  { id: '4', name: '최지영', department: '기획팀' },
  { id: '5', name: '정하늘', department: '디자인팀' },
  { id: '6', name: '강다솜', department: '기획팀' },
];

function SearchBar({ value, onChange }) {
  return (
    <TextInput
      style={styles.searchInput}
      value={value}
      onChangeText={onChange}
      placeholder="이름 또는 부서 검색..."
    />
  );
}

function UserItem({ user }) {
  return (
    <View style={styles.userItem}>
      <Text style={styles.userName}>{user.name}</Text>
      <Text style={styles.userDept}>{user.department}</Text>
    </View>
  );
}

function Summary({ total, filtered }) {
  return (
    <Text style={styles.summary}>
      전체 {total}명 중 {filtered}명 표시
    </Text>
  );
}

function App() {
  const [filter, setFilter] = useState('');

  const filteredUsers = USERS.filter(
    (u) => u.name.includes(filter) || u.department.includes(filter)
  );

  return (
    <View style={styles.container}>
      <Text style={styles.title}>단방향 데이터 흐름</Text>
      <Text style={styles.subtitle}>
        데이터 ↓ (props) / 이벤트 ↑ (콜백)
      </Text>

      {/* 데이터 ↓: filter, onChange를 props로 전달 */}
      <SearchBar value={filter} onChange={setFilter} />

      {/* 데이터 ↓: 필터링 결과를 props로 전달 */}
      <Summary total={USERS.length} filtered={filteredUsers.length} />

      <FlatList
        data={filteredUsers}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <UserItem user={item} />}
        ListEmptyComponent={
          <Text style={styles.empty}>검색 결과가 없습니다</Text>
        }
      />

      <View style={styles.flowDiagram}>
        <Text style={styles.flowTitle}>데이터 흐름:</Text>
        <Text style={styles.flowText}>App (state: filter)</Text>
        <Text style={styles.flowArrow}>  ↓ props     ↑ onChange 콜백</Text>
        <Text style={styles.flowText}>SearchBar / UserList / Summary</Text>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    paddingTop: 50,
    backgroundColor: '#f5f5f5',
  },
  title: {
    fontSize: 22,
    fontWeight: 'bold',
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 13,
    color: '#666',
    textAlign: 'center',
    marginBottom: 16,
  },
  searchInput: {
    borderWidth: 1,
    borderColor: '#ccc',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    backgroundColor: 'white',
    marginBottom: 8,
  },
  summary: {
    fontSize: 14,
    color: '#555',
    marginBottom: 12,
    textAlign: 'right',
  },
  userItem: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    backgroundColor: 'white',
    padding: 14,
    borderRadius: 8,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: '#eee',
  },
  userName: {
    fontSize: 16,
    fontWeight: '600',
  },
  userDept: {
    fontSize: 14,
    color: '#007AFF',
  },
  empty: {
    textAlign: 'center',
    color: '#999',
    marginTop: 20,
    fontSize: 15,
  },
  flowDiagram: {
    marginTop: 16,
    backgroundColor: '#E8EAF6',
    padding: 14,
    borderRadius: 8,
  },
  flowTitle: {
    fontWeight: 'bold',
    fontSize: 13,
    marginBottom: 4,
  },
  flowText: {
    fontSize: 12,
    fontFamily: 'monospace',
    color: '#333',
  },
  flowArrow: {
    fontSize: 12,
    fontFamily: 'monospace',
    color: '#666',
  },
});

export default App;
```

---

## 13. Props vs State 결정

```
┌──────────────────────────────────────────────────────────────────┐
│              Props vs State 결정 플로차트                         │
│                                                                  │
│   이 데이터는 시간이 지남에 따라 변하나요?                         │
│     │                                                            │
│     ├─ 아니요 → Props (또는 상수)                                 │
│     │           예: 사용자 이름, 컴포넌트 설정값                   │
│     │                                                            │
│     └─ 예 → 부모가 이 데이터를 관리하나요?                        │
│              │                                                    │
│              ├─ 예 → Props (부모에서 전달받음)                     │
│              │       예: 부모의 state를 자식에게 전달               │
│              │                                                    │
│              └─ 아니요 → 이 데이터는 이 컴포넌트만 사용하나요?     │
│                          │                                        │
│                          ├─ 예 → State (이 컴포넌트의 로컬 상태)  │
│                          │       예: 폼 입력값, 토글 상태          │
│                          │                                        │
│                          └─ 아니요 → 공통 부모로 State를 끌어올림  │
│                                     (Lifting State Up)            │
│                                     예: 형제 컴포넌트 간 공유     │
│                                                                  │
│                                                                  │
│   다른 state/props에서 계산할 수 있나요?                          │
│     │                                                            │
│     ├─ 예 → State/Props 아님! 그냥 변수로 계산                    │
│     │       예: const fullName = firstName + ' ' + lastName;     │
│     │       예: const total = items.reduce((a, b) => a + b, 0); │
│     │                                                            │
│     └─ 아니요 → 위의 플로차트 따르기                              │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 22: 무엇이 State이고 무엇이 아닌가

```typescript
function ShoppingCart() {
  // ✅ State: 시간에 따라 변하고, 다른 것에서 계산할 수 없음
  const [items, setItems] = useState<CartItem[]>([]);
  const [couponCode, setCouponCode] = useState('');
  const [isCheckingOut, setIsCheckingOut] = useState(false);

  // ❌ State가 아님: items에서 계산 가능 → 그냥 변수로 선언
  const totalPrice = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const itemCount = items.length;
  const isEmpty = items.length === 0;

  // ❌ State가 아님: totalPrice와 couponCode에서 계산 가능
  const discount = couponCode === 'SAVE10' ? totalPrice * 0.1 : 0;
  const finalPrice = totalPrice - discount;

  return (
    <div>
      <p>상품 수: {itemCount}</p>          {/* 계산된 값 */}
      <p>합계: {totalPrice.toLocaleString()}원</p>       {/* 계산된 값 */}
      <p>할인: -{discount.toLocaleString()}원</p>        {/* 계산된 값 */}
      <p>최종가: {finalPrice.toLocaleString()}원</p>     {/* 계산된 값 */}
    </div>
  );
}
```

---

## 14. 흔한 실수와 안티패턴

### 안티패턴 1: Props를 State에 복사하기

```typescript
// ❌ 안티패턴: Props를 State에 복사
function UserProfile({ name }: { name: string }) {
  // 부모의 name이 바뀌어도 이 state는 초기값 그대로!
  const [localName, setLocalName] = useState(name);
  return <p>{localName}</p>;  // 부모에서 name이 바뀌면 반영 안 됨!
}

// ✅ 올바른 방법: Props를 직접 사용
function UserProfile({ name }: { name: string }) {
  return <p>{name}</p>;  // 항상 최신 값 반영
}

// 예외: props를 "초기값"으로만 사용하고 이후 독립적으로 관리할 때
function EditableField({ initialValue }: { initialValue: string }) {
  const [value, setValue] = useState(initialValue);
  // 이 경우 의도적으로 부모와 분리됨 — 사용자가 편집 가능
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```

### 안티패턴 2: State 직접 변경 (Mutation)

```typescript
// ❌ 안티패턴: State 직접 변경
function TodoApp() {
  const [todos, setTodos] = useState([{ id: 1, text: '할일', done: false }]);

  const toggleBad = (id: number) => {
    // 원본 배열의 객체를 직접 변경!
    const todo = todos.find(t => t.id === id);
    if (todo) todo.done = !todo.done;  // ❌ mutation!
    setTodos([...todos]);  // 새 배열이지만 내부 객체는 같은 참조
  };

  // ✅ 올바른 방법: 새 객체 생성
  const toggleGood = (id: number) => {
    setTodos(prev => prev.map(todo =>
      todo.id === id
        ? { ...todo, done: !todo.done }  // 새 객체 생성
        : todo
    ));
  };
}
```

### 안티패턴 3: 불필요한 State

```typescript
// ❌ 안티패턴: 계산 가능한 값을 State로 관리
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');
  const [fullName, setFullName] = useState('');  // ❌ 불필요한 state!

  const handleFirstNameChange = (name: string) => {
    setFirstName(name);
    setFullName(name + ' ' + lastName);  // ❌ 동기화 코드 필요
  };

  const handleLastNameChange = (name: string) => {
    setLastName(name);
    setFullName(firstName + ' ' + name);  // ❌ 동기화 코드 필요
  };
}

// ✅ 올바른 방법: 계산된 값은 변수로
function Form() {
  const [firstName, setFirstName] = useState('');
  const [lastName, setLastName] = useState('');

  // fullName은 firstName + lastName에서 항상 계산 가능
  const fullName = `${firstName} ${lastName}`.trim();

  return (
    <div>
      <input value={firstName} onChange={e => setFirstName(e.target.value)} />
      <input value={lastName} onChange={e => setLastName(e.target.value)} />
      <p>전체 이름: {fullName}</p>
    </div>
  );
}
```

### 안티패턴 4: State 업데이트 후 즉시 값 사용

```typescript
// ❌ 안티패턴: setState 직후에 새 값을 기대
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    console.log(count);  // ❌ 여전히 0! (아직 리렌더 전)

    // React에서 setState는 비동기적으로 처리됨
    // setCount를 호출해도 count 변수는 이 렌더 사이클에서 변하지 않음

    // ✅ 새 값이 필요하면 직접 계산
    const newCount = count + 1;
    setCount(newCount);
    console.log(newCount);  // ✅ 1

    // 또는 useEffect에서 count 변경을 감지
  };
}
```

### 안티패턴 5: 과도한 State 분리

```typescript
// ❌ 안티패턴: 관련된 state를 너무 많이 분리
function Form() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  const [address, setAddress] = useState('');
  const [city, setCity] = useState('');
  const [zipCode, setZipCode] = useState('');
  // 7개의 독립적인 state! 관리가 어려움

  const handleSubmit = () => {
    // 모든 state를 하나하나 모아야 함
    const data = { username, password, email, phone, address, city, zipCode };
    // ...
  };
}

// ✅ 개선: 관련된 값을 객체로 묶기
function Form() {
  const [formData, setFormData] = useState({
    username: '',
    password: '',
    email: '',
    phone: '',
    address: '',
    city: '',
    zipCode: '',
  });

  // 범용 변경 핸들러
  const handleChange = (field: keyof typeof formData, value: string) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  const handleSubmit = () => {
    console.log(formData);  // 모든 데이터가 하나의 객체에
  };

  return (
    <form onSubmit={e => { e.preventDefault(); handleSubmit(); }}>
      <input
        value={formData.username}
        onChange={e => handleChange('username', e.target.value)}
        placeholder="사용자명"
      />
      <input
        value={formData.email}
        onChange={e => handleChange('email', e.target.value)}
        placeholder="이메일"
      />
      {/* ... 나머지 필드들 */}
    </form>
  );
}
```

### 안티패턴 6: useEffect로 파생 State 동기화

```typescript
// ❌ 안티패턴: useEffect로 state를 다른 state와 동기화
function ProductList({ products }: { products: Product[] }) {
  const [searchTerm, setSearchTerm] = useState('');
  const [filteredProducts, setFilteredProducts] = useState(products);

  // ❌ 불필요한 useEffect + 불필요한 state
  useEffect(() => {
    setFilteredProducts(
      products.filter(p => p.name.includes(searchTerm))
    );
  }, [products, searchTerm]);

  return <div>...</div>;
}

// ✅ 올바른 방법: 그냥 렌더 중에 계산
function ProductList({ products }: { products: Product[] }) {
  const [searchTerm, setSearchTerm] = useState('');

  // 렌더링할 때마다 자동으로 최신 값 계산
  const filteredProducts = products.filter(p =>
    p.name.includes(searchTerm)
  );
  // 만약 계산이 비싸다면 useMemo 사용 (04-hooks-deep-dive.md 참고)

  return <div>...</div>;
}
```

### 예제 23: 올바른 폼 관리 전체 예시

```typescript
interface FormData {
  name: string;
  email: string;
  age: string;
  agreeToTerms: boolean;
}

interface FormErrors {
  name?: string;
  email?: string;
  age?: string;
  agreeToTerms?: string;
}

function RegistrationForm() {
  const [formData, setFormData] = useState<FormData>({
    name: '',
    email: '',
    age: '',
    agreeToTerms: false,
  });

  const [submitted, setSubmitted] = useState(false);

  // ✅ 에러는 state가 아닌 계산된 값으로 (매 렌더마다 최신 검증)
  const errors: FormErrors = {};
  if (formData.name.length < 2) errors.name = '이름은 2자 이상이어야 합니다';
  if (!formData.email.includes('@')) errors.email = '유효한 이메일을 입력하세요';
  if (formData.age && (Number(formData.age) < 0 || Number(formData.age) > 150)) {
    errors.age = '유효한 나이를 입력하세요';
  }
  if (!formData.agreeToTerms) errors.agreeToTerms = '약관에 동의해야 합니다';

  const isValid = Object.keys(errors).length === 0;

  const handleChange = <K extends keyof FormData>(field: K, value: FormData[K]) => {
    setFormData(prev => ({ ...prev, [field]: value }));
  };

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitted(true);
    if (isValid) {
      console.log('제출:', formData);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <input
          value={formData.name}
          onChange={e => handleChange('name', e.target.value)}
          placeholder="이름"
        />
        {submitted && errors.name && (
          <span style={{ color: 'red' }}>{errors.name}</span>
        )}
      </div>

      <div>
        <input
          value={formData.email}
          onChange={e => handleChange('email', e.target.value)}
          placeholder="이메일"
          type="email"
        />
        {submitted && errors.email && (
          <span style={{ color: 'red' }}>{errors.email}</span>
        )}
      </div>

      <div>
        <input
          value={formData.age}
          onChange={e => handleChange('age', e.target.value)}
          placeholder="나이"
          type="number"
        />
        {submitted && errors.age && (
          <span style={{ color: 'red' }}>{errors.age}</span>
        )}
      </div>

      <div>
        <label>
          <input
            type="checkbox"
            checked={formData.agreeToTerms}
            onChange={e => handleChange('agreeToTerms', e.target.checked)}
          />
          약관에 동의합니다
        </label>
        {submitted && errors.agreeToTerms && (
          <span style={{ color: 'red' }}>{errors.agreeToTerms}</span>
        )}
      </div>

      <button type="submit" disabled={submitted && !isValid}>
        가입하기
      </button>
    </form>
  );
}
```

### 예제 24: Props 콜백으로 부모에게 데이터 전달

```typescript
// 자식 → 부모 데이터 전달 패턴
// Android의 interface callback / Compose의 람다 콜백과 동일

interface SearchResult {
  query: string;
  resultCount: number;
}

// 자식: 검색 결과를 부모에게 알림
function SearchBox({ onSearch }: { onSearch: (result: SearchResult) => void }) {
  const [query, setQuery] = useState('');

  const handleSearch = () => {
    // 검색 수행 후 결과를 부모 콜백으로 전달
    const resultCount = Math.floor(Math.random() * 100);
    onSearch({ query, resultCount });
  };

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        onKeyDown={e => e.key === 'Enter' && handleSearch()}
      />
      <button onClick={handleSearch}>검색</button>
    </div>
  );
}

// 부모: 자식으로부터 데이터 수신
function App() {
  const [lastSearch, setLastSearch] = useState<SearchResult | null>(null);

  return (
    <div>
      <SearchBox onSearch={setLastSearch} />
      {lastSearch && (
        <p>"{lastSearch.query}" 검색 결과: {lastSearch.resultCount}건</p>
      )}
    </div>
  );
}
```

### 예제 25: 복수의 하위 컴포넌트와 상태 관리

```typescript
// 실전 패턴: 여러 자식 컴포넌트가 상태를 공유하는 경우
function ShoppingApp() {
  const [cart, setCart] = useState<CartItem[]>([]);

  const addToCart = (product: Product) => {
    setCart(prev => {
      const existing = prev.find(item => item.productId === product.id);
      if (existing) {
        return prev.map(item =>
          item.productId === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      }
      return [...prev, { productId: product.id, name: product.name, price: product.price, quantity: 1 }];
    });
  };

  const removeFromCart = (productId: number) => {
    setCart(prev => prev.filter(item => item.productId !== productId));
  };

  const cartTotal = cart.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const cartCount = cart.reduce((sum, item) => sum + item.quantity, 0);

  return (
    <div>
      <Header cartCount={cartCount} />
      <ProductGrid onAddToCart={addToCart} />
      <CartSidebar
        items={cart}
        total={cartTotal}
        onRemove={removeFromCart}
      />
    </div>
  );
}
```

```javascript [playground]
// 🧪 안티패턴 vs 올바른 패턴 실습

// 안티패턴 1: State 직접 변경 vs 새 객체 생성
const original = [
  { id: 1, text: "할일 1", done: false },
  { id: 2, text: "할일 2", done: false },
];

// ❌ 직접 변경 (mutation) — React에서 변경 감지 불가
const bad = original;
bad[0].done = true; // 원본도 변경됨!
console.log("mutation:", original[0].done); // true (원본 오염!)

// 원복
original[0].done = false;

// ✅ 새 객체 생성 (immutable update)
const good = original.map(todo =>
  todo.id === 1 ? { ...todo, done: true } : todo
);
console.log("immutable 원본:", original[0].done); // false (원본 유지!)
console.log("immutable 새 객체:", good[0].done);  // true

// 참조 비교로 변경 감지
console.log("같은 참조?:", original === good); // false → React가 변경 감지!

// 안티패턴 3: 계산 가능한 값은 State 대신 변수로
const firstName = "길동";
const lastName = "홍";
// ❌ fullName을 별도 state로 관리할 필요 없음
// ✅ 그냥 계산하면 됨
const fullName = `${lastName}${firstName}`;
console.log("계산된 값:", fullName);

// 파생 상태 예시
const items = [
  { name: "사과", price: 3000, qty: 2 },
  { name: "바나나", price: 2000, qty: 3 },
];
// ✅ 파생 계산 (별도 state 불필요)
const totalPrice = items.reduce((sum, item) => sum + item.price * item.qty, 0);
const totalItems = items.reduce((sum, item) => sum + item.qty, 0);
console.log(`총 ${totalItems}개, ${totalPrice.toLocaleString()}원`);
```

---

## 핵심 요약

```
┌──────────────────────────────────────────────────────────────────┐
│                    기억해야 할 핵심 개념                          │
│                                                                  │
│  Props:                                                          │
│  ─────                                                           │
│  • 부모 → 자식 데이터 전달 (읽기 전용)                            │
│  • TypeScript interface로 타입 정의                               │
│  • 구조 분해 + 기본값으로 깔끔하게 사용                            │
│  • Android의 Bundle/Intent extras, Fragment arguments 대체        │
│                                                                  │
│  State:                                                          │
│  ──────                                                          │
│  • 컴포넌트 내부의 변경 가능한 데이터                              │
│  • useState(초기값) → [현재값, setter]                            │
│  • 불변성 필수: 직접 변경 ❌, 새 값/객체 생성 ✅                  │
│  • updater 함수 사용: setCount(prev => prev + 1)                 │
│  • Android의 MutableStateFlow / Compose의 mutableStateOf 대체    │
│                                                                  │
│  데이터 흐름:                                                     │
│  ────────                                                        │
│  • 단방향: 데이터 ↓ (props), 이벤트 ↑ (콜백)                     │
│  • 공유 상태 → 공통 부모로 끌어올리기 (Lifting State Up)          │
│  • 계산 가능한 값 → State로 만들지 말고 렌더 중에 계산             │
│  • Controlled 컴포넌트 = React state가 진실의 근원                │
└──────────────────────────────────────────────────────────────────┘
```

---

> **이전 문서:** [02-components-and-jsx.md](./02-components-and-jsx.md)
> **다음 문서:** [04-hooks-deep-dive.md](./04-hooks-deep-dive.md)
