# 컴포넌트와 JSX — React의 빌딩 블록 완전 가이드

## 목차
1. [컴포넌트란 무엇인가?](#1-컴포넌트란-무엇인가)
2. [Function Component — 현대 React의 표준](#2-function-component)
3. [JSX 문법 완전 가이드](#3-jsx-문법-완전-가이드)
4. [조건부 렌더링 패턴 (5가지 이상)](#4-조건부-렌더링-패턴)
5. [리스트 렌더링과 key](#5-리스트-렌더링과-key)
6. [Fragment: 불필요한 래퍼 제거](#6-fragment)
7. [이벤트 핸들링](#7-이벤트-핸들링)
8. [컴포넌트 합성과 children](#8-컴포넌트-합성과-children)
9. [컴포넌트 파일 구조 패턴](#9-컴포넌트-파일-구조-패턴)

---

## 1. 컴포넌트란 무엇인가?

### 정의

React에서 **컴포넌트**는 UI의 독립적이고 재사용 가능한 조각입니다. 각 컴포넌트는 자신만의 로직과 외관을 가지며, 다른 컴포넌트와 조합하여 복잡한 UI를 구성합니다.

### Android와 비교

```
┌──────────────────────────────────────────────────────────────────┐
│                    Android vs React 컴포넌트 비교                 │
│                                                                  │
│  Android                          React                          │
│  ────────────────────            ────────────────────             │
│  Activity                        Page/Screen 컴포넌트             │
│  Fragment                        일반 컴포넌트                    │
│  Custom View / ViewGroup         일반 컴포넌트                    │
│  XML Layout                      JSX 반환값                      │
│  RecyclerView.ViewHolder         리스트 아이템 컴포넌트            │
│  Dialog / BottomSheet            Modal 컴포넌트                   │
│  Adapter                         map() + 컴포넌트                │
│                                                                  │
│  핵심 차이:                                                      │
│  - Android는 Activity/Fragment/View 등 여러 단위가 존재           │
│  - React는 "컴포넌트" 단 하나의 개념으로 모든 것을 표현            │
│  - 페이지든, 버튼이든, 리스트 아이템이든 전부 "함수 컴포넌트"       │
└──────────────────────────────────────────────────────────────────┘
```

### Android Fragment vs React Component

```kotlin
// Android Fragment — 상당한 보일러플레이트
class ProfileFragment : Fragment() {
    private var _binding: FragmentProfileBinding? = null
    private val binding get() = _binding!!
    private val viewModel: ProfileViewModel by viewModels()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentProfileBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewModel.profile.observe(viewLifecycleOwner) { profile ->
            binding.tvName.text = profile.name
            binding.tvEmail.text = profile.email
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

```typescript
// 예제 1: React Component — 같은 기능, 훨씬 간결
import React, { useState, useEffect } from 'react';

function ProfileScreen() {
  const [profile, setProfile] = useState<{
    name: string;
    email: string;
  } | null>(null);

  useEffect(() => {
    // 데이터 로딩 (ViewModel의 역할도 컴포넌트 안에서 가능)
    fetchProfile().then(setProfile);
  }, []);

  if (!profile) return <div>로딩 중...</div>;

  return (
    <div>
      <h1>{profile.name}</h1>
      <p>{profile.email}</p>
    </div>
  );
}
```

---

## 2. Function Component

### 문법과 작성 규칙

React 컴포넌트는 **대문자로 시작하는 JavaScript 함수**입니다. JSX를 반환합니다.

### 예제 2: 가장 단순한 컴포넌트

```typescript
// 방법 1: function 선언 (가장 보편적, 권장)
function Greeting() {
  return <h1>안녕하세요!</h1>;
}

// 방법 2: 화살표 함수 (팀 컨벤션에 따라 사용)
const Greeting = () => {
  return <h1>안녕하세요!</h1>;
};

// 방법 3: 화살표 함수 + 암시적 반환 (단순한 경우)
const Greeting = () => <h1>안녕하세요!</h1>;
```

### 예제 3: TypeScript를 사용한 props 타입 정의

```typescript
// Props 인터페이스 정의 — Kotlin의 data class와 유사한 역할
interface GreetingProps {
  name: string;
  age?: number;  // ?는 optional — Kotlin의 Int? (nullable)와 유사
}

// function 선언 방식 + 구조 분해(destructuring)
function Greeting({ name, age }: GreetingProps) {
  return (
    <div>
      <h1>안녕하세요, {name}님!</h1>
      {age !== undefined && <p>나이: {age}세</p>}
    </div>
  );
}

// 화살표 함수 방식 + React.FC 타입 (일부 프로젝트에서 사용)
const Greeting: React.FC<GreetingProps> = ({ name, age }) => {
  return (
    <div>
      <h1>안녕하세요, {name}님!</h1>
      {age !== undefined && <p>나이: {age}세</p>}
    </div>
  );
};
// 참고: React.FC 사용은 논쟁이 있음. 최근에는 일반 함수 선언을 선호하는 추세
```

### 네이밍 컨벤션

```typescript
// ✅ 올바른 컴포넌트 이름: PascalCase (대문자로 시작)
function UserProfile() { ... }
function TodoListItem() { ... }
function NavigationBar() { ... }

// ❌ 잘못된 이름: 소문자로 시작하면 React가 HTML 태그로 인식
function userProfile() { ... }  // React가 <userProfile>을 HTML 태그로 처리
function todo_item() { ... }    // 언더스코어 사용 안 함

// Kotlin 비교:
// Kotlin 클래스: class UserProfile — PascalCase (동일)
// Kotlin 함수: fun getUserProfile() — camelCase (다름!)
// Compose: @Composable fun UserProfile() — PascalCase (React와 동일!)
```

### 예제 4: 완전한 컴포넌트 파일 구조

```typescript
// UserCard.tsx
import React from 'react';

// 1. 타입 정의
interface UserCardProps {
  name: string;
  email: string;
  avatarUrl: string;
  isOnline: boolean;
  onPress?: () => void;  // 콜백 함수도 props로 전달 가능
}

// 2. 컴포넌트 정의
function UserCard({ name, email, avatarUrl, isOnline, onPress }: UserCardProps) {
  return (
    <div onClick={onPress} style={styles.container}>
      <img src={avatarUrl} alt={name} style={styles.avatar} />
      <div>
        <strong>{name}</strong>
        <p style={styles.email}>{email}</p>
        <span style={{ color: isOnline ? 'green' : 'gray' }}>
          {isOnline ? '● 온라인' : '● 오프라인'}
        </span>
      </div>
    </div>
  );
}

// 3. 스타일 정의
const styles = {
  container: {
    display: 'flex',
    alignItems: 'center',
    padding: '12px',
    borderRadius: '8px',
    border: '1px solid #eee',
    cursor: 'pointer',
  },
  avatar: {
    width: '48px',
    height: '48px',
    borderRadius: '50%',
    marginRight: '12px',
  },
  email: {
    color: '#666',
    fontSize: '14px',
  },
} as const;

// 4. export
export default UserCard;
```

---

## 3. JSX 문법 완전 가이드

### JSX란?

JSX는 **JavaScript XML**의 약자입니다. JavaScript 코드 안에서 HTML과 유사한 문법으로 UI를 작성할 수 있게 해줍니다. 빌드 시 일반 JavaScript 함수 호출로 변환됩니다.

### 예제 5: JSX는 어떻게 변환되는가?

```typescript
// 우리가 작성하는 JSX
const element = <h1 className="title">안녕하세요</h1>;

// 빌드 후 변환되는 실제 JavaScript (React 19 기준)
// (자동 변환이므로 직접 작성할 일은 없음)
import { jsx as _jsx } from 'react/jsx-runtime';
const element = _jsx('h1', { className: 'title', children: '안녕하세요' });

// Kotlin 비교: Compose도 컴파일러 플러그인이 코드를 변환함
// @Composable fun → 실제로는 $composer 파라미터가 추가된 함수로 변환
```

### 예제 6: JSX 안에서 JavaScript 표현식 사용 `{}`

```typescript
function ExpressionExamples() {
  const name = '김철수';
  const items = ['사과', '바나나', '체리'];
  const isLoggedIn = true;

  return (
    <div>
      {/* 변수 삽입 */}
      <h1>안녕하세요, {name}님!</h1>

      {/* 계산식 */}
      <p>2 + 3 = {2 + 3}</p>

      {/* 함수 호출 */}
      <p>대문자: {name.toUpperCase()}</p>

      {/* 삼항 연산자 */}
      <p>{isLoggedIn ? '환영합니다' : '로그인해주세요'}</p>

      {/* 배열 길이 */}
      <p>아이템 수: {items.length}개</p>

      {/* 즉시 실행 — 복잡한 로직은 변수로 빼는 것이 더 깔끔 */}
      <p>{(() => {
        const hour = new Date().getHours();
        return hour < 12 ? '오전' : '오후';
      })()}</p>

      {/* ⚠️ 주의: {} 안에는 "표현식"만 가능 (값을 반환하는 것) */}
      {/* ❌ if문, for문 같은 "문(statement)"은 사용 불가 */}
      {/* {if (isLoggedIn) return '환영'} ← 에러! */}
      {/* {for (const item of items) ...} ← 에러! */}
    </div>
  );
}
```

### 예제 7: JSX vs Android XML — 핵심 차이점

```typescript
// JSX 문법 규칙 비교
function JsxVsXml() {
  return (
    <div>
      {/* ─── 차이 1: class vs className ─── */}
      {/* Android XML: android:layout_width="match_parent" */}
      {/* HTML:        class="container" */}
      {/* JSX:         className="container"  ← class는 JS 예약어이므로 */}
      <div className="container">

        {/* ─── 차이 2: camelCase 속성명 ─── */}
        {/* HTML:  onclick, tabindex, maxlength */}
        {/* JSX:   onClick, tabIndex, maxLength  ← camelCase */}
        <button onClick={() => alert('clicked')} tabIndex={0}>
          클릭
        </button>

        {/* ─── 차이 3: style은 객체로 전달 ─── */}
        {/* HTML:  style="font-size: 16px; background-color: red;" */}
        {/* JSX:   style={{ fontSize: '16px', backgroundColor: 'red' }} */}
        {/*        ↑ 바깥 {} = JSX 표현식, 안쪽 {} = JS 객체 리터럴 */}
        <p style={{ fontSize: '16px', backgroundColor: 'red', color: 'white' }}>
          스타일 예시
        </p>

        {/* ─── 차이 4: self-closing 태그 필수 ─── */}
        {/* HTML:  <img src="photo.jpg"> <br> <input type="text"> */}
        {/* JSX:   <img src="photo.jpg" /> <br /> <input type="text" /> */}
        {/*        ↑ /> 로 반드시 닫아야 함 (Android XML과 동일!) */}
        <img src="photo.jpg" alt="사진" />
        <br />
        <input type="text" />

        {/* ─── 차이 5: for vs htmlFor ─── */}
        {/* HTML: <label for="email"> */}
        {/* JSX:  <label htmlFor="email">  ← for는 JS 예약어 */}
        <label htmlFor="email">이메일</label>
        <input id="email" type="email" />

        {/* ─── 차이 6: 주석 ─── */}
        {/* HTML:  <!-- 주석 --> */}
        {/* JSX:   {/* 주석 */}
        {/* Android XML: <!-- 주석 --> */}

        {/* ─── 차이 7: boolean 속성 ─── */}
        {/* HTML:  <button disabled> */}
        {/* JSX:   <button disabled> 또는 <button disabled={true}> */}
        <button disabled>비활성 버튼</button>
        <button disabled={false}>활성 버튼</button>
      </div>
    </div>
  );
}
```

### 예제 8: JSX의 반환 규칙

```typescript
// 규칙 1: 단일 루트 요소만 반환 가능
// ❌ 에러
function Bad() {
  return (
    <h1>제목</h1>
    <p>내용</p>
  );
}

// ✅ div로 감싸기
function GoodWithDiv() {
  return (
    <div>
      <h1>제목</h1>
      <p>내용</p>
    </div>
  );
}

// ✅ Fragment로 감싸기 (불필요한 DOM 노드 생성 방지)
function GoodWithFragment() {
  return (
    <>
      <h1>제목</h1>
      <p>내용</p>
    </>
  );
}

// 규칙 2: 여러 줄 JSX는 괄호 ()로 감싸기
function MultiLine() {
  return (   // ← 괄호 시작
    <div>
      <h1>제목</h1>
    </div>
  );         // ← 괄호 끝
}

// 규칙 3: null, undefined, false, true는 렌더링되지 않음
function NullRendering() {
  return (
    <div>
      {null}       {/* 아무것도 렌더링되지 않음 */}
      {undefined}  {/* 아무것도 렌더링되지 않음 */}
      {false}      {/* 아무것도 렌더링되지 않음 */}
      {true}       {/* 아무것도 렌더링되지 않음 */}
      {0}          {/* ⚠️ 주의: 0은 렌더링됨! "0"이 화면에 표시 */}
      {''}         {/* 빈 문자열: 아무것도 렌더링되지 않음 */}
    </div>
  );
}
```

```exercise
type: code-arrange
question: "함수형 컴포넌트를 올바르게 조립하세요"
tokens:
  - "function"
  - "Greeting"
  - "({ name })"
  - "{"
  - "return"
  - "<Text>Hello {name}</Text>"
  - "}"
distractors:
  - "class"
  - "render()"
  - "this.props"
answer: ["function", "Greeting", "({ name })", "{", "return", "<Text>Hello {name}</Text>", "}"]
hint: "React Native에서는 함수형 컴포넌트를 사용하며, props를 구조 분해로 받습니다"
xp: 8
```

---

## 4. 조건부 렌더링 패턴

Android에서는 `View.VISIBLE`, `View.GONE`, `View.INVISIBLE`을 사용했지만, React에서는 JavaScript의 조건 표현식을 그대로 사용합니다.

### 패턴 1: && 연산자 (가장 흔한 패턴)

```typescript
// 예제 9: && 연산자 — 조건이 true일 때만 렌더링
function Notifications({ count }: { count: number }) {
  return (
    <div>
      <h1>알림</h1>
      {/* count > 0일 때만 배지 표시 */}
      {count > 0 && (
        <span className="badge">{count}개의 새 알림</span>
      )}

      {/* ⚠️ 흔한 실수: 숫자 0은 falsy이지만 렌더링됨! */}
      {/* ❌ */ count && <span>{count}개</span>}
      {/* count가 0이면 화면에 "0"이 표시됨 */}

      {/* ✅ 올바른 방법: 명시적으로 비교 */}
      {count > 0 && <span>{count}개</span>}
    </div>
  );
}
```

### 패턴 2: 삼항 연산자 (조건에 따라 다른 UI)

```typescript
// 예제 10: 삼항 연산자 — true/false에 따라 다른 요소 렌더링
function LoginStatus({ isLoggedIn }: { isLoggedIn: boolean }) {
  return (
    <div>
      {isLoggedIn ? (
        <div>
          <p>환영합니다!</p>
          <button>로그아웃</button>
        </div>
      ) : (
        <div>
          <p>로그인이 필요합니다</p>
          <button>로그인</button>
        </div>
      )}
    </div>
  );
}
```

### 패턴 3: 조기 반환 (Early Return)

```typescript
// 예제 11: 조기 반환 — 로딩/에러 상태 처리에 유용
interface DataScreenProps {
  isLoading: boolean;
  error: string | null;
  data: string[];
}

function DataScreen({ isLoading, error, data }: DataScreenProps) {
  // Android의 when 문과 비슷한 패턴
  if (isLoading) {
    return <div className="spinner">로딩 중...</div>;
  }

  if (error) {
    return <div className="error">에러: {error}</div>;
  }

  if (data.length === 0) {
    return <div className="empty">데이터가 없습니다</div>;
  }

  // 정상 케이스
  return (
    <ul>
      {data.map(item => (
        <li key={item}>{item}</li>
      ))}
    </ul>
  );
}
```

### 패턴 4: 변수에 JSX 저장

```typescript
// 예제 12: JSX를 변수에 할당하여 복잡한 조건부 렌더링 정리
function Dashboard({ user }: { user: User | null }) {
  let content: React.ReactNode;

  if (!user) {
    content = <p>로그인이 필요합니다</p>;
  } else if (user.role === 'admin') {
    content = <AdminPanel user={user} />;
  } else if (user.role === 'editor') {
    content = <EditorPanel user={user} />;
  } else {
    content = <UserPanel user={user} />;
  }

  return (
    <div className="dashboard">
      <Header />
      {content}
      <Footer />
    </div>
  );
}
```

### 패턴 5: 즉시 실행 함수 (IIFE)

```typescript
// 예제 13: IIFE — JSX 안에서 복잡한 로직이 필요할 때
// (권장하지는 않지만, 알아두면 유용)
function StatusDisplay({ status }: { status: string }) {
  return (
    <div>
      {(() => {
        switch (status) {
          case 'loading': return <Spinner />;
          case 'error': return <ErrorMessage />;
          case 'empty': return <EmptyState />;
          case 'success': return <DataList />;
          default: return null;
        }
      })()}
    </div>
  );
}
```

### 패턴 6: 객체 매핑 (가장 깔끔한 다중 조건)

```typescript
// 예제 14: 객체 매핑 — switch/when 대안으로 매우 깔끔
type Status = 'idle' | 'loading' | 'success' | 'error';

const STATUS_COMPONENTS: Record<Status, React.ReactNode> = {
  idle: <p>시작하려면 버튼을 누르세요</p>,
  loading: <div className="spinner">로딩 중...</div>,
  success: <p style={{ color: 'green' }}>✓ 완료!</p>,
  error: <p style={{ color: 'red' }}>✗ 오류가 발생했습니다</p>,
};

function StatusBanner({ status }: { status: Status }) {
  return <div className="banner">{STATUS_COMPONENTS[status]}</div>;
}
```

### 패턴 7: CSS를 활용한 조건부 표시 (VISIBLE/INVISIBLE 대응)

```typescript
// 예제 15: display:none — Android의 View.INVISIBLE과 유사
// (DOM에서 제거하지 않고 숨기기만 함)
function Panel({ isVisible }: { isVisible: boolean }) {
  return (
    <div>
      {/* 방법 A: 조건부 렌더링 — 아예 DOM에서 제거 (GONE 효과) */}
      {isVisible && <div>패널 A</div>}

      {/* 방법 B: CSS hidden — DOM은 유지, 보이지만 않음 (INVISIBLE 효과) */}
      <div style={{ display: isVisible ? 'block' : 'none' }}>
        패널 B
      </div>

      {/* 방법 C: visibility hidden — 공간은 차지하지만 안 보임 */}
      <div style={{ visibility: isVisible ? 'visible' : 'hidden' }}>
        패널 C
      </div>
    </div>
  );
}
```

```
┌──────────────────────────────────────────────────────────────────┐
│              Android View Visibility ↔ React 매핑                │
│                                                                  │
│  View.VISIBLE     →  그냥 렌더링 (기본 상태)                      │
│  View.GONE        →  {condition && <Component />}                │
│                      (조건이 false면 DOM에서 아예 제거)            │
│  View.INVISIBLE   →  style={{ visibility: 'hidden' }}            │
│                      (보이지 않지만 공간은 차지)                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 5. 리스트 렌더링과 key

### Android RecyclerView vs React map()

Android에서 리스트를 표시하려면 RecyclerView + Adapter + ViewHolder라는 상당한 보일러플레이트가 필요합니다. React에서는 `map()` 한 줄이면 됩니다.

### 예제 16: 기본 리스트 렌더링

```typescript
interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([
    { id: 1, text: '리액트 배우기', completed: false },
    { id: 2, text: '프로젝트 만들기', completed: false },
    { id: 3, text: '배포하기', completed: true },
  ]);

  return (
    <ul>
      {/* 배열의 각 요소를 JSX로 변환 */}
      {todos.map(todo => (
        <li
          key={todo.id}  // ← key는 필수! 아래에서 자세히 설명
          style={{
            textDecoration: todo.completed ? 'line-through' : 'none',
            color: todo.completed ? '#999' : '#000',
          }}
        >
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

### key가 왜 중요한가?

```
┌──────────────────────────────────────────────────────────────────┐
│                    key의 역할 — 핵심 개념                        │
│                                                                  │
│  React는 리스트를 다시 렌더링할 때, 어떤 항목이 추가/삭제/이동      │
│  되었는지 알아야 효율적으로 DOM을 업데이트할 수 있습니다.            │
│  key는 각 항목의 "신분증" 역할을 합니다.                          │
│                                                                  │
│  Android의 DiffUtil.ItemCallback과 같은 목적:                    │
│  → areItemsTheSame(oldItem, newItem) ≈ key 비교                 │
│                                                                  │
│  ────────────────────────────────────────────────────            │
│                                                                  │
│  key가 없거나 index를 key로 사용할 때 문제:                       │
│                                                                  │
│  Before:  [A, B, C]     index: [0, 1, 2]                        │
│  After:   [X, A, B, C]  index: [0, 1, 2, 3]                     │
│                                                                  │
│  index를 key로 사용하면:                                         │
│  → React는 key=0인 "A"가 "X"로 바뀌었다고 판단                   │
│  → key=1인 "B"가 "A"로 바뀌었다고 판단                           │
│  → key=2인 "C"가 "B"로 바뀌었다고 판단                           │
│  → key=3인 "C"가 새로 추가되었다고 판단                           │
│  → 모든 항목의 내용을 업데이트! (매우 비효율적)                    │
│                                                                  │
│  고유 id를 key로 사용하면:                                       │
│  → key="X"가 새로 추가됨을 즉시 인식                              │
│  → key="A", "B", "C"는 그대로 유지                               │
│  → 새 항목 1개만 DOM에 추가! (효율적)                             │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 17: key 사용 규칙

```typescript
interface User {
  id: string;     // 고유 식별자
  name: string;
  email: string;
}

function UserList({ users }: { users: User[] }) {
  return (
    <div>
      {users.map(user => (
        // ✅ 좋음: 데이터 고유 ID를 key로 사용
        <UserCard key={user.id} user={user} />
      ))}

      {/* ⚠️ 피해야 할 패턴들 */}

      {users.map((user, index) => (
        // ❌ 나쁨: index를 key로 사용 — 순서가 바뀌면 문제 발생
        <UserCard key={index} user={user} />
      ))}

      {users.map(user => (
        // ❌ 나쁨: Math.random() — 매 렌더마다 key가 바뀌어 전체 재생성
        <UserCard key={Math.random()} user={user} />
      ))}
    </div>
  );
}

// ✅ key 규칙 정리:
// 1. 형제 요소 사이에서 고유해야 함 (전역적으로 고유할 필요는 없음)
// 2. 변경되면 안 됨 (Math.random() ❌)
// 3. 데이터의 고유 ID 사용이 최선
// 4. index는 "리스트 순서가 절대 바뀌지 않는 경우"에만 사용
```

```javascript [playground]
// 🧪 리스트 데이터 변환 실습 — React의 map/filter를 순수 JS로 연습

const users = [
  { id: 1, name: "홍길동", age: 30, role: "개발자" },
  { id: 2, name: "김철수", age: 25, role: "디자이너" },
  { id: 3, name: "이영희", age: 35, role: "개발자" },
  { id: 4, name: "박민수", age: 28, role: "PM" },
  { id: 5, name: "최수진", age: 32, role: "개발자" },
];

// 1) map — 표시할 데이터 변환 (React에서 JSX를 반환하는 것과 동일한 로직)
const displayNames = users.map(u => `${u.name} (${u.age}세)`);
console.log("이름 목록:", displayNames);

// 2) filter + map — 조건부 렌더링
const developers = users
  .filter(u => u.role === "개발자")
  .map(u => u.name);
console.log("개발자:", developers);

// 3) sort — 정렬 (원본 유지!)
const byAge = [...users].sort((a, b) => a.age - b.age);
console.log("나이순:", byAge.map(u => `${u.name}(${u.age})`));

// 4) key의 중요성 — 인덱스 vs 고유 ID
const items = ["A", "B", "C"];
// ❌ 인덱스를 key로 사용하면 중간에 삽입 시 문제
const withIndex = items.map((item, i) => ({ key: i, value: item }));
console.log("인덱스 key:", JSON.stringify(withIndex));

// 중간에 "X" 삽입 → 인덱스가 밀림!
const inserted = ["A", "X", "B", "C"];
const afterInsert = inserted.map((item, i) => ({ key: i, value: item }));
console.log("삽입 후:", JSON.stringify(afterInsert));
console.log("→ key=1이 B에서 X로 바뀜! React가 전체를 다시 렌더링");

// ✅ 고유 ID를 key로 사용하면 정확한 업데이트 가능
```

### 예제 18: 필터링과 정렬이 있는 리스트

```typescript
function FilterableList() {
  const [items] = useState([
    { id: 1, name: '사과', category: 'fruit', price: 1500 },
    { id: 2, name: '당근', category: 'vegetable', price: 800 },
    { id: 3, name: '바나나', category: 'fruit', price: 2000 },
    { id: 4, name: '브로콜리', category: 'vegetable', price: 1200 },
    { id: 5, name: '체리', category: 'fruit', price: 5000 },
  ]);

  const [filter, setFilter] = useState<'all' | 'fruit' | 'vegetable'>('all');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  // 필터링 + 정렬된 리스트 계산
  const filteredAndSorted = items
    .filter(item => filter === 'all' || item.category === filter)
    .sort((a, b) => {
      if (sortBy === 'name') return a.name.localeCompare(b.name);
      return a.price - b.price;
    });

  return (
    <div>
      {/* 필터 버튼 */}
      <div>
        <button onClick={() => setFilter('all')}>전체</button>
        <button onClick={() => setFilter('fruit')}>과일</button>
        <button onClick={() => setFilter('vegetable')}>채소</button>
      </div>

      {/* 정렬 버튼 */}
      <div>
        <button onClick={() => setSortBy('name')}>이름순</button>
        <button onClick={() => setSortBy('price')}>가격순</button>
      </div>

      {/* 리스트 렌더링 */}
      {filteredAndSorted.length === 0 ? (
        <p>결과가 없습니다</p>
      ) : (
        <ul>
          {filteredAndSorted.map(item => (
            <li key={item.id}>
              {item.name} — {item.price.toLocaleString()}원
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## 6. Fragment

Fragment는 불필요한 DOM 노드를 생성하지 않고 여러 요소를 그룹화하는 방법입니다. Android의 ViewGroup 없이 여러 View를 반환하는 것과 같습니다.

### 예제 19: Fragment 사용법

```typescript
import React from 'react';

// 방법 1: 단축 문법 <></>  (가장 많이 사용)
function UserInfo({ name, email }: { name: string; email: string }) {
  return (
    <>
      <dt>{name}</dt>
      <dd>{email}</dd>
    </>
  );
  // 이렇게 하면 <div>로 감싸지 않아도 됨
  // <dl> 안에서 사용할 때 유효한 HTML 구조를 유지할 수 있음
}

// 방법 2: React.Fragment — key가 필요할 때 사용
function UserList({ users }: { users: Array<{ id: string; name: string; email: string }> }) {
  return (
    <dl>
      {users.map(user => (
        // <></> 에는 key를 줄 수 없으므로 React.Fragment 사용
        <React.Fragment key={user.id}>
          <dt>{user.name}</dt>
          <dd>{user.email}</dd>
        </React.Fragment>
      ))}
    </dl>
  );
}

// Android 비교:
// Compose에서도 비슷한 상황이 있음
// Column { ... } 으로 감싸지 않고 여러 Composable을 반환하고 싶을 때
// Compose에서는 별도의 Fragment 없이 그냥 나열 가능 (content 람다)
```

---

## 7. 이벤트 핸들링

### Android vs React 이벤트 처리 비교

```
┌──────────────────────────────┬──────────────────────────────────┐
│     Android                  │     React                        │
├──────────────────────────────┼──────────────────────────────────┤
│ setOnClickListener { }       │ onClick={() => { }}              │
│ setOnLongClickListener { }   │ onContextMenu (웹) /             │
│                              │ onLongPress (React Native)       │
│ addTextChangedListener       │ onChange                         │
│ setOnFocusChangeListener     │ onFocus / onBlur                 │
│ setOnScrollChangeListener    │ onScroll                         │
│ setOnTouchListener           │ onTouchStart/Move/End            │
│ setOnKeyListener             │ onKeyDown / onKeyUp              │
└──────────────────────────────┴──────────────────────────────────┘
```

### 예제 20: 이벤트 핸들링 패턴들

```typescript
function EventExamples() {
  // 패턴 1: 인라인 핸들러 (간단한 로직)
  // ✅ 한 줄 로직에 적합
  const handleSimpleClick = () => alert('클릭!');

  // 패턴 2: 별도 함수 (복잡한 로직)
  const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();  // 폼 기본 동작(페이지 새로고침) 방지
    console.log('폼 제출됨');
  };

  // 패턴 3: 파라미터가 필요한 핸들러
  const handleItemClick = (id: number) => {
    console.log(`아이템 ${id} 클릭됨`);
  };

  return (
    <div>
      {/* 패턴 1: 인라인 */}
      <button onClick={() => console.log('클릭!')}>
        인라인 핸들러
      </button>

      {/* 패턴 2: 함수 참조 — ()를 붙이지 않음! */}
      <button onClick={handleSimpleClick}>
        함수 참조
      </button>

      {/* ❌ 흔한 실수: ()를 붙이면 렌더링 시점에 즉시 실행됨 */}
      {/* <button onClick={handleSimpleClick()}>잘못된 예</button> */}

      {/* 패턴 3: 파라미터 전달 — 화살표 함수로 감싸기 */}
      <button onClick={() => handleItemClick(42)}>
        아이템 42
      </button>

      {/* 패턴 4: 이벤트 객체 사용 */}
      <input
        onChange={(e: React.ChangeEvent<HTMLInputElement>) => {
          console.log('입력값:', e.target.value);
          // Android: editText.addTextChangedListener의 afterTextChanged와 유사
        }}
        placeholder="입력해보세요"
      />

      {/* 패턴 5: 폼 제출 */}
      <form onSubmit={handleSubmit}>
        <input type="text" />
        <button type="submit">제출</button>
      </form>

      {/* 패턴 6: 키보드 이벤트 */}
      <input
        onKeyDown={(e: React.KeyboardEvent) => {
          if (e.key === 'Enter') {
            console.log('엔터 키 눌림');
          }
        }}
      />
    </div>
  );
}
```

### 예제 21: React Native의 이벤트 핸들링

```typescript
// React Native에서는 웹과 이벤트 이름이 다름
import { View, Text, TouchableOpacity, TextInput, Pressable } from 'react-native';

function RNEventExamples() {
  return (
    <View>
      {/* 웹의 onClick → React Native의 onPress */}
      <TouchableOpacity onPress={() => console.log('탭!')}>
        <Text>터치해주세요</Text>
      </TouchableOpacity>

      {/* Pressable: 더 세밀한 터치 이벤트 제어 */}
      <Pressable
        onPress={() => console.log('짧은 탭')}
        onLongPress={() => console.log('길게 누름')}
        onPressIn={() => console.log('손가락 닿음')}
        onPressOut={() => console.log('손가락 뗌')}
      >
        <Text>Pressable 컴포넌트</Text>
      </Pressable>

      {/* 텍스트 입력 */}
      <TextInput
        onChangeText={(text) => console.log('입력:', text)}
        onFocus={() => console.log('포커스 획득')}
        onBlur={() => console.log('포커스 잃음')}
        placeholder="입력해주세요"
      />
    </View>
  );
}
```

---

## 8. 컴포넌트 합성과 children

### children props — Android의 ViewGroup 자식 요소와 유사

React에서 컴포넌트 사이에 다른 JSX를 넣으면, 그것이 `children` props로 전달됩니다. Android의 `LinearLayout` 안에 `TextView`를 넣는 것과 같은 개념입니다.

### 예제 22: children 기본 사용법

```typescript
// 카드 컨테이너 컴포넌트 — Android의 CardView와 유사
interface CardProps {
  title: string;
  children: React.ReactNode;  // ReactNode = JSX, 문자열, 숫자, null 등 모든 렌더 가능한 것
}

function Card({ title, children }: CardProps) {
  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '12px',
      padding: '16px',
      margin: '8px',
      boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
    }}>
      <h2 style={{ borderBottom: '1px solid #eee', paddingBottom: '8px' }}>
        {title}
      </h2>
      <div style={{ marginTop: '12px' }}>
        {children}  {/* ← 여기에 전달받은 자식 요소가 렌더링됨 */}
      </div>
    </div>
  );
}

// 사용법 — Card 사이에 넣은 모든 것이 children으로 전달됨
function App() {
  return (
    <div>
      <Card title="사용자 정보">
        {/* 이 영역이 children */}
        <p>이름: 김철수</p>
        <p>이메일: kim@mail.com</p>
      </Card>

      <Card title="주문 내역">
        {/* 어떤 JSX든 children으로 전달 가능 */}
        <OrderList />
        <button>더 보기</button>
      </Card>
    </div>
  );
}
```

### 예제 23: Render Props / Slot 패턴

```typescript
// 좀 더 유연한 합성 — 여러 "슬롯"에 컨텐츠 배치
// Android의 include, merge 또는 Compose의 slot API와 유사
interface PageLayoutProps {
  header: React.ReactNode;    // 헤더 영역에 넣을 내용
  sidebar: React.ReactNode;   // 사이드바에 넣을 내용
  children: React.ReactNode;  // 메인 컨텐츠 영역
  footer?: React.ReactNode;   // 선택적 푸터
}

function PageLayout({ header, sidebar, children, footer }: PageLayoutProps) {
  return (
    <div style={{ display: 'flex', flexDirection: 'column', minHeight: '100vh' }}>
      <header style={{ padding: '16px', borderBottom: '1px solid #ddd' }}>
        {header}
      </header>

      <div style={{ display: 'flex', flex: 1 }}>
        <aside style={{ width: '240px', padding: '16px', borderRight: '1px solid #ddd' }}>
          {sidebar}
        </aside>
        <main style={{ flex: 1, padding: '16px' }}>
          {children}
        </main>
      </div>

      {footer && (
        <footer style={{ padding: '16px', borderTop: '1px solid #ddd' }}>
          {footer}
        </footer>
      )}
    </div>
  );
}

// 사용법
function App() {
  return (
    <PageLayout
      header={<h1>내 앱</h1>}
      sidebar={<NavMenu />}
      footer={<p>© 2026 My App</p>}
    >
      <h2>메인 컨텐츠</h2>
      <p>여기가 본문 영역입니다.</p>
    </PageLayout>
  );
}
```

### 예제 24: 재사용 가능한 래퍼 패턴

```typescript
// 에러 경계 래퍼 (개념적 예시)
interface ErrorBoundaryProps {
  fallback: React.ReactNode;
  children: React.ReactNode;
}

// 조건부 래핑 패턴
function ConditionalWrapper({
  condition,
  wrapper,
  children,
}: {
  condition: boolean;
  wrapper: (children: React.ReactNode) => React.ReactNode;
  children: React.ReactNode;
}) {
  return condition ? wrapper(children) : children;
}

// 사용법: 조건에 따라 링크로 감싸거나 그냥 텍스트로 표시
function UserName({ name, profileUrl }: { name: string; profileUrl?: string }) {
  return (
    <ConditionalWrapper
      condition={!!profileUrl}
      wrapper={(children) => <a href={profileUrl}>{children}</a>}
    >
      <span>{name}</span>
    </ConditionalWrapper>
  );
}
```

---

## 9. 컴포넌트 파일 구조 패턴

### 패턴 A: 파일당 하나의 컴포넌트 (가장 보편적)

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx        ← 컴포넌트
│   │   ├── Button.styles.ts  ← 스타일 (선택)
│   │   ├── Button.test.tsx   ← 테스트 (선택)
│   │   └── index.ts          ← re-export
│   ├── Card/
│   │   ├── Card.tsx
│   │   └── index.ts
│   └── Header/
│       ├── Header.tsx
│       └── index.ts
├── screens/                   ← 화면 단위 (Android의 Activity/Fragment 대응)
│   ├── HomeScreen/
│   │   ├── HomeScreen.tsx
│   │   ├── components/        ← 이 화면에서만 사용하는 하위 컴포넌트
│   │   │   └── FeaturedList.tsx
│   │   └── index.ts
│   └── ProfileScreen/
│       ├── ProfileScreen.tsx
│       └── index.ts
└── App.tsx
```

### 패턴 B: index.ts re-export 패턴

```typescript
// components/Button/index.ts
export { default } from './Button';
export type { ButtonProps } from './Button';

// 이렇게 하면 import할 때 깔끔:
// import Button from '../components/Button';
// (Button/Button 이 아니라 Button만으로 import 가능)
```

### 예제 25: 완전한 컴포넌트 파일 예시

```typescript
// components/TodoItem/TodoItem.tsx
import React, { useCallback } from 'react';

// ── 타입 정의 ──
export interface TodoItemProps {
  id: number;
  text: string;
  completed: boolean;
  onToggle: (id: number) => void;
  onDelete: (id: number) => void;
}

// ── 컴포넌트 ──
function TodoItem({ id, text, completed, onToggle, onDelete }: TodoItemProps) {
  const handleToggle = useCallback(() => onToggle(id), [id, onToggle]);
  const handleDelete = useCallback(() => onDelete(id), [id, onDelete]);

  return (
    <div style={styles.container}>
      <input
        type="checkbox"
        checked={completed}
        onChange={handleToggle}
      />
      <span style={{
        ...styles.text,
        textDecoration: completed ? 'line-through' : 'none',
        color: completed ? '#999' : '#333',
      }}>
        {text}
      </span>
      <button onClick={handleDelete} style={styles.deleteButton}>
        삭제
      </button>
    </div>
  );
}

// ── 스타일 ──
const styles = {
  container: {
    display: 'flex',
    alignItems: 'center',
    gap: '8px',
    padding: '8px 12px',
    borderBottom: '1px solid #eee',
  },
  text: {
    flex: 1,
    fontSize: '16px',
  },
  deleteButton: {
    backgroundColor: '#ff4444',
    color: 'white',
    border: 'none',
    borderRadius: '4px',
    padding: '4px 8px',
    cursor: 'pointer',
  },
} as const;

// ── export ──
export default TodoItem;
```

### 예제 26: 컴포넌트 조합 — 전체 화면 구성

```typescript
// screens/TodoScreen.tsx
import React, { useState, useCallback } from 'react';
import TodoItem from '../components/TodoItem';

interface Todo {
  id: number;
  text: string;
  completed: boolean;
}

function TodoScreen() {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [inputText, setInputText] = useState('');

  const addTodo = useCallback(() => {
    if (inputText.trim() === '') return;
    setTodos(prev => [
      ...prev,
      { id: Date.now(), text: inputText.trim(), completed: false },
    ]);
    setInputText('');
  }, [inputText]);

  const toggleTodo = useCallback((id: number) => {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, completed: !todo.completed } : todo
      )
    );
  }, []);

  const deleteTodo = useCallback((id: number) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }, []);

  const completedCount = todos.filter(t => t.completed).length;

  return (
    <div style={{ maxWidth: '500px', margin: '0 auto', padding: '20px' }}>
      <h1>할 일 목록</h1>

      {/* 입력 영역 */}
      <div style={{ display: 'flex', gap: '8px', marginBottom: '16px' }}>
        <input
          value={inputText}
          onChange={e => setInputText(e.target.value)}
          onKeyDown={e => e.key === 'Enter' && addTodo()}
          placeholder="할 일을 입력하세요"
          style={{ flex: 1, padding: '8px' }}
        />
        <button onClick={addTodo}>추가</button>
      </div>

      {/* 요약 */}
      {todos.length > 0 && (
        <p style={{ color: '#666', marginBottom: '8px' }}>
          {completedCount}/{todos.length} 완료
        </p>
      )}

      {/* 리스트 */}
      {todos.length === 0 ? (
        <p style={{ color: '#999', textAlign: 'center' }}>
          할 일이 없습니다. 위에서 추가해보세요!
        </p>
      ) : (
        todos.map(todo => (
          <TodoItem
            key={todo.id}
            id={todo.id}
            text={todo.text}
            completed={todo.completed}
            onToggle={toggleTodo}
            onDelete={deleteTodo}
          />
        ))
      )}
    </div>
  );
}

export default TodoScreen;
```

---

## 핵심 요약

```
┌──────────────────────────────────────────────────────────────────┐
│                    기억해야 할 핵심 개념                          │
│                                                                  │
│  1. 컴포넌트 = 대문자로 시작하는 함수 + JSX 반환                  │
│     → Android의 Fragment/View/ViewHolder 역할을 하나로 통합       │
│                                                                  │
│  2. JSX = JavaScript 안의 HTML-like 문법                         │
│     → {} 안에 JavaScript 표현식 사용 가능                         │
│     → className, htmlFor 등 일부 속성명이 HTML과 다름             │
│                                                                  │
│  3. 조건부 렌더링 = JavaScript 조건 표현식                        │
│     → &&, 삼항, 조기 반환, 변수 할당, 객체 매핑 등 다양한 패턴    │
│     → visibility GONE/INVISIBLE 대신 조건부 렌더링 또는 CSS       │
│                                                                  │
│  4. 리스트 = map() + key                                         │
│     → RecyclerView + Adapter 대신 map() 한 줄                   │
│     → key는 DiffUtil처럼 효율적 업데이트를 위해 필수              │
│                                                                  │
│  5. children = 컴포넌트의 자식 요소                               │
│     → Android의 ViewGroup이 자식 View를 가지는 것과 동일          │
│     → 유연한 컴포넌트 합성의 핵심                                 │
│                                                                  │
│  6. 이벤트 = camelCase 이벤트 이름 + 함수 전달                   │
│     → onClick, onChange, onSubmit 등                             │
│     → 함수 "참조"를 전달 — ()를 붙이면 안 됨!                    │
└──────────────────────────────────────────────────────────────────┘
```

---

> **이전 문서:** [01-declarative-ui-paradigm.md](./01-declarative-ui-paradigm.md)
> **다음 문서:** [03-props-and-state.md](./03-props-and-state.md)
