# Zustand 완전 가이드 — Android ViewModel 개발자를 위한 전역 상태 관리

## 목차
1. [왜 Zustand인가?](#1-왜-zustand인가)
2. [설치와 기본 사용법](#2-설치와-기본-사용법)
3. [Store에서 상태 읽기: Selector 패턴](#3-store에서-상태-읽기-selector-패턴)
4. [상태 업데이트: set()과 get()](#4-상태-업데이트)
5. [파생 상태 (Computed/Derived)](#5-파생-상태)
6. [미들웨어](#6-미들웨어)
7. [비동기 액션](#7-비동기-액션)
8. [Store 분리 (Slices)](#8-store-분리)
9. [테스팅](#9-테스팅)
10. [실전 패턴: Auth Store](#10-실전-패턴-auth-store)
11. [실전 패턴: Cart Store](#11-실전-패턴-cart-store)
12. [실전 패턴: UI Settings Store](#12-실전-패턴-ui-settings-store)
13. [Android ViewModel+StateFlow vs Zustand 비교](#13-android-viewmodel-vs-zustand-비교)

---

## 1. 왜 Zustand인가?

### 다른 상태 관리 라이브러리와 비교

| 특성 | Redux | Context | Zustand | Jotai | MobX |
|------|-------|---------|---------|-------|------|
| 보일러플레이트 | 매우 많음 | 보통 | 최소 | 최소 | 보통 |
| 번들 크기 | ~7KB | 0 (내장) | ~1.2KB | ~3KB | ~16KB |
| 러닝 커브 | 높음 | 낮음 | 매우 낮음 | 낮음 | 보통 |
| TypeScript 지원 | 좋음 | 보통 | 매우 좋음 | 좋음 | 좋음 |
| Selector 지원 | O | X | O | O | 자동 |
| 미들웨어 | O | X | O | X | X |
| DevTools | O | X | O | O | O |
| Provider 필요 | O | O | X | O | O |
| React 외부 사용 | O (복잡) | X | O (간단) | X | O |

**Zustand가 Android 개발자에게 적합한 이유**:

1. **ViewModel과 유사한 패턴**: Store가 ViewModel 역할을 하며, 상태와 액션을 한 곳에 정의한다
2. **최소한의 코드**: Redux의 action/reducer/dispatch 대신 직관적인 `set()`
3. **Provider 불필요**: Hilt처럼 어디서든 바로 사용 가능 (Context는 Provider 트리 필요)
4. **Selector로 최적화**: StateFlow의 `map`/`distinctUntilChanged`와 동일한 최적화
5. **React 외부에서 사용 가능**: Axios 인터셉터, 네비게이션 등에서 `getState()` 호출 가능

### Redux와의 비교

```tsx
// Redux: 카운터 하나에 이 모든 코드가 필요
// actions.ts
const INCREMENT = 'INCREMENT';
const DECREMENT = 'DECREMENT';
export const increment = () => ({ type: INCREMENT });
export const decrement = () => ({ type: DECREMENT });

// reducer.ts
export function counterReducer(state = { count: 0 }, action) {
  switch (action.type) {
    case INCREMENT: return { ...state, count: state.count + 1 };
    case DECREMENT: return { ...state, count: state.count - 1 };
    default: return state;
  }
}

// store.ts
import { createStore } from 'redux';
const store = createStore(counterReducer);

// Component.tsx
function Counter() {
  const count = useSelector(state => state.count);
  const dispatch = useDispatch();
  return <Button title={`${count}`} onPress={() => dispatch(increment())} />;
}
```

```tsx
// Zustand: 동일한 기능을 이렇게 간단하게
import { create } from 'zustand';

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
}));

function Counter() {
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  return <Button title={`${count}`} onPress={increment} />;
}
```

---

## 2. 설치와 기본 사용법

### 2-1. 설치

```bash
npm install zustand
```

그것이 전부다. peer dependency도 없고, 네이티브 모듈도 없다. Zustand는 순수 JavaScript 라이브러리다.

### 2-2. Store 생성

```tsx
// stores/counterStore.ts
import { create } from 'zustand';

// TypeScript 인터페이스 정의 (Android의 data class / sealed class와 유사)
interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setCount: (value: number) => void;
}

// create<타입>()(콜백) — 타입과 함께 store 생성
// Android의 ViewModel 클래스를 정의하는 것과 동일
const useCounterStore = create<CounterState>()((set) => ({
  // 초기 상태 (ViewModel의 초기 StateFlow 값)
  count: 0,

  // 액션 (ViewModel의 메서드)
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  setCount: (value) => set({ count: value }),
}));

export default useCounterStore;
```

### 2-3. 컴포넌트에서 사용

```tsx
// screens/CounterScreen.tsx
import React from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';
import useCounterStore from '../stores/counterStore';

function CounterScreen() {
  // Store에서 필요한 상태와 액션 가져오기
  // Android: viewModel.count.collectAsState()
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);
  const reset = useCounterStore((state) => state.reset);

  return (
    <View style={styles.container}>
      <Text style={styles.count}>{count}</Text>
      <View style={styles.buttons}>
        <Button title="-" onPress={decrement} />
        <Button title="+" onPress={increment} />
        <Button title="Reset" onPress={reset} />
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  count: { fontSize: 72, fontWeight: 'bold', marginBottom: 30 },
  buttons: { flexDirection: 'row', gap: 20 },
});
```

```exercise
type: code-arrange
question: "Zustand 스토어를 생성하는 코드를 조립하세요"
tokens:
  - "const useStore"
  - "= create"
  - "((set) => ({"
  - "count: 0,"
  - "increment: () =>"
  - "set((state) => ({ count: state.count + 1 }))"
  - "}))"
distractors:
  - "new Store"
  - "this.setState"
  - "MutableStateFlow"
answer: ["const useStore", "= create", "((set) => ({", "count: 0,", "increment: () =>", "set((state) => ({ count: state.count + 1 }))", "}))"]
hint: "Zustand는 create 함수에 set을 받는 콜백을 전달합니다"
xp: 8
```

---

## 3. Store에서 상태 읽기: Selector 패턴

Selector는 Zustand의 핵심 성능 최적화 기법이다. Android의 `StateFlow.map { }.distinctUntilChanged()`와 동일한 역할을 한다.

### 3-1. 개별 Selector (권장)

```tsx
// 각 상태를 개별 selector로 구독
// count가 변경되면 CountDisplay만 리렌더링, ActionButtons는 리렌더링 안 됨
function CountDisplay() {
  const count = useCounterStore((state) => state.count);
  return <Text>{count}</Text>;
}

function ActionButtons() {
  const increment = useCounterStore((state) => state.increment);
  return <Button title="+" onPress={increment} />;
}
```

### 3-2. 여러 값 한 번에 가져오기

```tsx
// 방법 1: 객체로 가져오기 (useShallow 사용)
import { useShallow } from 'zustand/react/shallow';

function TodoStats() {
  const { todos, total } = useCounterStore(
    useShallow((state) => ({
      todos: state.todos,
      total: state.total,
    }))
  );
}

// 방법 2: 배열로 가져오기 (useShallow 사용)
function TodoStats() {
  const [todos, total] = useCounterStore(
    useShallow((state) => [state.todos, state.total])
  );
}
```

**왜 useShallow가 필요한가?** selector가 새 객체/배열을 반환하면 매번 참조가 달라져서 불필요한 리렌더링이 발생한다. `useShallow`는 얕은 비교(shallow comparison)를 수행하여 실제 값이 변경되었을 때만 리렌더링한다. Android의 `distinctUntilChanged()`와 동일한 개념이다.

### 3-3. 전체 Store 가져오기 (비추천)

```tsx
// 비추천: Store의 어떤 값이라도 변경되면 리렌더링
function BadComponent() {
  const store = useCounterStore(); // 전체 store 구독
  return <Text>{store.count}</Text>;
}

// 추천: 필요한 값만 selector로 구독
function GoodComponent() {
  const count = useCounterStore((state) => state.count);
  return <Text>{count}</Text>;
}
```

---

## 4. 상태 업데이트

### 4-1. set() 함수

```tsx
const useStore = create<StoreState>()((set, get) => ({
  count: 0,
  name: 'Zustand',
  items: [],

  // 방법 1: 객체 전달 (부분 업데이트 — 기존 상태와 자동 merge)
  setName: (name: string) => set({ name }),

  // 방법 2: 콜백 전달 (이전 상태 기반 업데이트)
  increment: () => set((state) => ({ count: state.count + 1 })),

  // 방법 3: 배열/객체 업데이트 (불변성 유지 필수!)
  addItem: (item: Item) => set((state) => ({
    items: [...state.items, item],  // 새 배열 생성 (push 금지!)
  })),

  removeItem: (id: number) => set((state) => ({
    items: state.items.filter((item) => item.id !== id),
  })),

  updateItem: (id: number, updates: Partial<Item>) => set((state) => ({
    items: state.items.map((item) =>
      item.id === id ? { ...item, ...updates } : item
    ),
  })),
}));
```

### 4-2. get() 함수 (현재 상태 읽기)

```tsx
const useStore = create<StoreState>()((set, get) => ({
  items: [],
  selectedId: null,

  // get()으로 현재 상태에서 값 읽기
  // Android의 ViewModel에서 _state.value로 현재 상태를 읽는 것과 동일
  getSelectedItem: () => {
    const { items, selectedId } = get();
    return items.find((item) => item.id === selectedId) ?? null;
  },

  // set() 안에서 다른 상태를 참조할 때
  selectNextItem: () => {
    const { items, selectedId } = get();
    const currentIndex = items.findIndex((i) => i.id === selectedId);
    const nextIndex = (currentIndex + 1) % items.length;
    set({ selectedId: items[nextIndex]?.id ?? null });
  },
}));
```

### 4-3. React 외부에서 상태 접근

```tsx
// Android에서 Service나 BroadcastReceiver에서 ViewModel 상태를 접근하는 것과 유사
// Zustand는 React 바깥에서도 상태를 읽고 쓸 수 있다!

// API 인터셉터에서 토큰 가져오기
import { useAuthStore } from '../stores/authStore';

const apiClient = axios.create({ baseURL: 'https://api.example.com' });

apiClient.interceptors.request.use((config) => {
  // React Hook이 아닌 getState()로 직접 접근
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 응답 인터셉터에서 로그아웃 처리
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // React Hook이 아닌 getState()로 직접 액션 호출
      useAuthStore.getState().logout();
    }
    return Promise.reject(error);
  }
);
```

### 4-4. Store 구독 (React 외부)

```tsx
// Store 변경을 구독 (Android의 Flow collect와 유사)
const unsubscribe = useAuthStore.subscribe(
  (state) => state.isLoggedIn,
  (isLoggedIn, prevIsLoggedIn) => {
    console.log('로그인 상태 변경:', prevIsLoggedIn, '→', isLoggedIn);
    if (!isLoggedIn) {
      // 네비게이션 리셋 등
      navigationRef.reset({ index: 0, routes: [{ name: 'Login' }] });
    }
  }
);

// 구독 해제
unsubscribe();
```

---

## 5. 파생 상태

Android의 `Flow.map()`이나 `Flow.combine()`으로 만드는 파생 상태에 해당한다.

### 5-1. Store 내부 파생 상태

```tsx
interface CartState {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: number) => void;
  // 파생 상태를 getter 메서드로 정의
  getTotal: () => number;
  getItemCount: () => number;
}

const useCartStore = create<CartState>()((set, get) => ({
  items: [],

  addItem: (item) => set((state) => ({
    items: [...state.items, item],
  })),

  removeItem: (id) => set((state) => ({
    items: state.items.filter((i) => i.id !== id),
  })),

  // Kotlin의 Flow.map { items -> items.sumOf { it.price } }와 동일
  getTotal: () => {
    return get().items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  },

  getItemCount: () => {
    return get().items.reduce((sum, item) => sum + item.quantity, 0);
  },
}));
```

### 5-2. 컴포넌트에서 파생 상태 (useMemo)

```tsx
function CartSummary() {
  const items = useCartStore((state) => state.items);

  // useMemo로 파생 값 계산 (items가 변경될 때만 재계산)
  // Android의 derivedStateOf와 유사
  const total = useMemo(
    () => items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );

  const itemCount = useMemo(
    () => items.reduce((sum, item) => sum + item.quantity, 0),
    [items]
  );

  return (
    <View>
      <Text>총 {itemCount}개 상품</Text>
      <Text>합계: {total.toLocaleString()}원</Text>
    </View>
  );
}
```

```javascript [playground]
// 🧪 파생 상태(Derived State) 실습

// Zustand store를 순수 JS로 시뮬레이션
const cartState = {
  items: [
    { id: 1, name: "노트북", price: 1500000, qty: 1 },
    { id: 2, name: "키보드", price: 80000, qty: 2 },
    { id: 3, name: "마우스", price: 50000, qty: 3 },
  ]
};

// 파생 상태: store에 저장하지 않고 계산으로 얻는 값
const totalItems = cartState.items.reduce((sum, item) => sum + item.qty, 0);
const totalPrice = cartState.items.reduce((sum, item) => sum + item.price * item.qty, 0);
const avgPrice = totalPrice / totalItems;

console.log(`총 ${totalItems}개 상품`);
console.log(`총 금액: ${totalPrice.toLocaleString()}원`);
console.log(`평균 단가: ${Math.round(avgPrice).toLocaleString()}원`);

// 파생 상태는 원본이 변하면 자동으로 업데이트
const updatedItems = cartState.items.map(item =>
  item.id === 1 ? { ...item, qty: 2 } : item // 노트북 2개로 변경
);

const newTotal = updatedItems.reduce((sum, item) => sum + item.price * item.qty, 0);
console.log(`\n수량 변경 후: ${newTotal.toLocaleString()}원`);
console.log(`차이: +${(newTotal - totalPrice).toLocaleString()}원`);
```

---

## 6. 미들웨어

### 6-1. persist: 상태 영속화

Android의 SharedPreferences/DataStore에 ViewModel 상태를 저장하는 것과 동일하다.

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface SettingsState {
  theme: 'light' | 'dark';
  language: string;
  notifications: boolean;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (lang: string) => void;
  toggleNotifications: () => void;
}

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'ko',
      notifications: true,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      toggleNotifications: () =>
        set((state) => ({ notifications: !state.notifications })),
    }),
    {
      name: 'settings-storage',          // 저장 키 (SharedPreferences의 키)
      storage: createJSONStorage(() => AsyncStorage), // 저장소

      // 특정 값만 영속화 (함수는 저장할 필요 없음)
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
        notifications: state.notifications,
      }),

      // 저장소 버전 관리 (마이그레이션)
      version: 1,
      migrate: (persistedState: any, version: number) => {
        if (version === 0) {
          // v0 → v1 마이그레이션
          persistedState.notifications = true; // 새 필드 기본값
        }
        return persistedState as SettingsState;
      },
    }
  )
);
```

MMKV를 사용한 고성능 영속화 (추천):

```tsx
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// MMKV 어댑터
const mmkvStorage = {
  getItem: (name: string) => {
    const value = storage.getString(name);
    return value ?? null;
  },
  setItem: (name: string, value: string) => {
    storage.set(name, value);
  },
  removeItem: (name: string) => {
    storage.delete(name);
  },
};

const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      // ... 상태 정의
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);
```

### 6-2. devtools: 개발자 도구 연결

```tsx
import { devtools } from 'zustand/middleware';

const useStore = create<StoreState>()(
  devtools(
    (set) => ({
      count: 0,
      increment: () =>
        set(
          (state) => ({ count: state.count + 1 }),
          undefined,      // replace 여부
          'increment'     // 액션 이름 (DevTools에 표시)
        ),
    }),
    {
      name: 'CounterStore', // DevTools에서 표시되는 Store 이름
      enabled: __DEV__,     // 개발 모드에서만 활성화
    }
  )
);
```

### 6-3. immer: 불변성 자동 관리

Kotlin의 data class `copy()`처럼 불변 상태를 쉽게 업데이트할 수 있다.

```bash
npm install immer
```

```tsx
import { immer } from 'zustand/middleware/immer';

interface NestedState {
  user: {
    profile: {
      name: string;
      address: {
        city: string;
        zipCode: string;
      };
    };
    settings: {
      theme: string;
      notifications: boolean;
    };
  };
  updateCity: (city: string) => void;
}

// immer 없이: 깊은 중첩 객체 업데이트가 매우 번거로움
const useStoreWithoutImmer = create<NestedState>()((set) => ({
  user: {
    profile: { name: 'Kim', address: { city: 'Seoul', zipCode: '12345' } },
    settings: { theme: 'dark', notifications: true },
  },
  updateCity: (city) => set((state) => ({
    user: {
      ...state.user,
      profile: {
        ...state.user.profile,
        address: {
          ...state.user.profile.address,
          city, // 이 하나를 바꾸기 위해 이 모든 spread...
        },
      },
    },
  })),
}));

// immer 사용: 직관적인 직접 수정 (내부적으로 불변성 보장)
const useStoreWithImmer = create<NestedState>()(
  immer((set) => ({
    user: {
      profile: { name: 'Kim', address: { city: 'Seoul', zipCode: '12345' } },
      settings: { theme: 'dark', notifications: true },
    },
    updateCity: (city) => set((state) => {
      state.user.profile.address.city = city; // 직접 수정!
      // Kotlin의 copy()보다도 간단
    }),
  }))
);
```

### 6-4. 미들웨어 조합

```tsx
// 여러 미들웨어를 함께 사용
const useStore = create<StoreState>()(
  devtools(
    persist(
      immer(
        (set) => ({
          // store 정의
        })
      ),
      { name: 'store', storage: createJSONStorage(() => AsyncStorage) }
    ),
    { name: 'MyStore' }
  )
);
```

```javascript [playground]
// 🧪 미들웨어 패턴 실습 — Zustand middleware를 순수 JS로 이해하기

// 기본 store
function createStore(initialState) {
  let state = initialState;
  return {
    getState: () => state,
    setState: (partial) => {
      state = typeof partial === 'function'
        ? { ...state, ...partial(state) }
        : { ...state, ...partial };
    }
  };
}

// 미들웨어 1: Logger — 상태 변경을 콘솔에 기록
function withLogger(createStoreFn) {
  return (initial) => {
    const store = createStoreFn(initial);
    const originalSetState = store.setState;
    store.setState = (partial) => {
      const prev = store.getState();
      originalSetState(partial);
      const next = store.getState();
      console.log('%c이전:', 'color: gray', JSON.stringify(prev));
      console.log('%c다음:', 'color: green', JSON.stringify(next));
    };
    return store;
  };
}

// 미들웨어 2: Persist — 상태를 저장소에 유지
function withPersist(createStoreFn, storageKey) {
  return (initial) => {
    const store = createStoreFn(initial);
    const originalSetState = store.setState;
    store.setState = (partial) => {
      originalSetState(partial);
      console.log(`[persist] "${storageKey}" 저장됨`);
    };
    return store;
  };
}

// 미들웨어 적용
const store = withLogger(
  (initial) => withPersist(createStore, 'my-store')(initial)
)({ count: 0, name: "테스트" });

console.log("=== 상태 변경 ===");
store.setState({ count: 1 });
store.setState(prev => ({ count: prev.count + 1 }));
```

---

## 7. 비동기 액션

Android ViewModel에서 `viewModelScope.launch { }`를 사용하는 것처럼, Zustand에서도 비동기 작업을 수행할 수 있다.

### 7-1. 기본 비동기 액션

```tsx
interface UserState {
  user: User | null;
  isLoading: boolean;
  error: string | null;

  fetchUser: (id: string) => Promise<void>;
  updateUser: (data: Partial<User>) => Promise<void>;
}

const useUserStore = create<UserState>()((set, get) => ({
  user: null,
  isLoading: false,
  error: null,

  // Android: viewModelScope.launch { repository.getUser(id) }
  fetchUser: async (id: string) => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch(`https://api.example.com/users/${id}`);
      if (!response.ok) throw new Error('사용자를 찾을 수 없습니다');
      const user = await response.json();
      set({ user, isLoading: false });
    } catch (error) {
      set({
        error: error instanceof Error ? error.message : '알 수 없는 오류',
        isLoading: false,
      });
    }
  },

  updateUser: async (data: Partial<User>) => {
    const currentUser = get().user;
    if (!currentUser) return;

    set({ isLoading: true });
    try {
      const response = await fetch(
        `https://api.example.com/users/${currentUser.id}`,
        {
          method: 'PATCH',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(data),
        }
      );
      const updatedUser = await response.json();
      set({ user: updatedUser, isLoading: false });
    } catch (error) {
      set({
        error: error instanceof Error ? error.message : '업데이트 실패',
        isLoading: false,
      });
    }
  },
}));
```

### 7-2. Android ViewModel 비동기 패턴 비교

```kotlin
// Android ViewModel
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun fetchUser(id: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                val user = repository.getUser(id)
                _uiState.update { it.copy(user = user, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }
}
```

```tsx
// Zustand: 거의 1:1 대응!
const useUserStore = create<UserState>()((set) => ({
  user: null,
  isLoading: false,
  error: null,

  fetchUser: async (id: string) => {
    set({ isLoading: true, error: null });         // _uiState.update { ... }
    try {
      const user = await userApi.getUser(id);       // repository.getUser()
      set({ user, isLoading: false });              // _uiState.update { ... }
    } catch (e: any) {
      set({ error: e.message, isLoading: false });  // _uiState.update { ... }
    }
  },
}));
```

---

## 8. Store 분리 (Slices)

앱이 커지면 하나의 Store에 모든 상태를 넣기 어렵다. Zustand에서는 Store를 여러 Slice로 분리할 수 있다.

### 8-1. 별도 Store 분리 (가장 간단, 권장)

```tsx
// stores/authStore.ts
export const useAuthStore = create<AuthState>()(/* ... */);

// stores/cartStore.ts
export const useCartStore = create<CartState>()(/* ... */);

// stores/settingsStore.ts
export const useSettingsStore = create<SettingsState>()(/* ... */);

// 사용: 각 Store를 독립적으로 사용
function Header() {
  const user = useAuthStore((s) => s.user);
  const itemCount = useCartStore((s) => s.items.length);
  const theme = useSettingsStore((s) => s.theme);
}
```

### 8-2. Slice 패턴 (하나의 Store에 합치기)

```tsx
// slices/authSlice.ts
export interface AuthSlice {
  token: string | null;
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
}

export const createAuthSlice = (
  set: any,
  get: any
): AuthSlice => ({
  token: null,
  user: null,
  login: async (email, password) => {
    const data = await authApi.login(email, password);
    set({ token: data.token, user: data.user });
  },
  logout: () => {
    set({ token: null, user: null });
  },
});

// slices/cartSlice.ts
export interface CartSlice {
  cartItems: CartItem[];
  addToCart: (item: CartItem) => void;
  clearCart: () => void;
}

export const createCartSlice = (
  set: any,
  get: any
): CartSlice => ({
  cartItems: [],
  addToCart: (item) => set((state: any) => ({
    cartItems: [...state.cartItems, item],
  })),
  clearCart: () => set({ cartItems: [] }),
});

// stores/appStore.ts — Slice들을 합치기
type AppStore = AuthSlice & CartSlice;

const useAppStore = create<AppStore>()((...args) => ({
  ...createAuthSlice(...args),
  ...createCartSlice(...args),
}));

export default useAppStore;
```

대부분의 경우 **별도 Store 분리**(8-1)가 더 간단하고 관리하기 쉽다. Slice 패턴은 Store 간 상태를 공유해야 할 때만 사용한다.

---

## 9. 테스팅

### 9-1. Store 단위 테스트

```tsx
// __tests__/counterStore.test.ts
import useCounterStore from '../stores/counterStore';

describe('Counter Store', () => {
  // 각 테스트 전에 Store 초기화
  beforeEach(() => {
    useCounterStore.setState({ count: 0 });
  });

  test('초기 count는 0이다', () => {
    const { count } = useCounterStore.getState();
    expect(count).toBe(0);
  });

  test('increment는 count를 1 증가시킨다', () => {
    useCounterStore.getState().increment();
    expect(useCounterStore.getState().count).toBe(1);
  });

  test('decrement는 count를 1 감소시킨다', () => {
    useCounterStore.getState().decrement();
    expect(useCounterStore.getState().count).toBe(-1);
  });

  test('setCount는 특정 값으로 설정한다', () => {
    useCounterStore.getState().setCount(42);
    expect(useCounterStore.getState().count).toBe(42);
  });

  test('reset은 0으로 초기화한다', () => {
    useCounterStore.getState().setCount(100);
    useCounterStore.getState().reset();
    expect(useCounterStore.getState().count).toBe(0);
  });
});
```

### 9-2. 비동기 액션 테스트

```tsx
// __tests__/userStore.test.ts
import useUserStore from '../stores/userStore';

// fetch를 모킹
global.fetch = jest.fn();

describe('User Store', () => {
  beforeEach(() => {
    useUserStore.setState({ user: null, isLoading: false, error: null });
    (global.fetch as jest.Mock).mockClear();
  });

  test('fetchUser 성공 시 user가 설정된다', async () => {
    const mockUser = { id: '1', name: 'Kim', email: 'kim@test.com' };
    (global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUser),
    });

    await useUserStore.getState().fetchUser('1');

    const state = useUserStore.getState();
    expect(state.user).toEqual(mockUser);
    expect(state.isLoading).toBe(false);
    expect(state.error).toBeNull();
  });

  test('fetchUser 실패 시 error가 설정된다', async () => {
    (global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: false,
    });

    await useUserStore.getState().fetchUser('999');

    const state = useUserStore.getState();
    expect(state.user).toBeNull();
    expect(state.isLoading).toBe(false);
    expect(state.error).toBeTruthy();
  });
});
```

---

## 10. 실전 패턴: Auth Store

```tsx
// stores/authStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
}

interface AuthState {
  // 상태
  token: string | null;
  refreshToken: string | null;
  user: User | null;
  isLoading: boolean;
  isLoggedIn: boolean;

  // 액션
  login: (email: string, password: string) => Promise<void>;
  register: (name: string, email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshAuth: () => Promise<boolean>;
  updateProfile: (data: Partial<User>) => Promise<void>;
  checkAuthStatus: () => Promise<void>;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      token: null,
      refreshToken: null,
      user: null,
      isLoading: true,
      isLoggedIn: false,

      login: async (email, password) => {
        set({ isLoading: true });
        try {
          const response = await fetch('https://api.example.com/auth/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password }),
          });

          if (!response.ok) {
            const error = await response.json();
            throw new Error(error.message || '로그인 실패');
          }

          const data = await response.json();
          set({
            token: data.accessToken,
            refreshToken: data.refreshToken,
            user: data.user,
            isLoggedIn: true,
            isLoading: false,
          });
        } catch (error) {
          set({ isLoading: false });
          throw error; // 컴포넌트에서 에러 처리
        }
      },

      register: async (name, email, password) => {
        set({ isLoading: true });
        try {
          const response = await fetch('https://api.example.com/auth/register', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ name, email, password }),
          });

          if (!response.ok) throw new Error('회원가입 실패');

          const data = await response.json();
          set({
            token: data.accessToken,
            refreshToken: data.refreshToken,
            user: data.user,
            isLoggedIn: true,
            isLoading: false,
          });
        } catch (error) {
          set({ isLoading: false });
          throw error;
        }
      },

      logout: () => {
        set({
          token: null,
          refreshToken: null,
          user: null,
          isLoggedIn: false,
          isLoading: false,
        });
      },

      refreshAuth: async () => {
        const { refreshToken } = get();
        if (!refreshToken) return false;

        try {
          const response = await fetch('https://api.example.com/auth/refresh', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ refreshToken }),
          });

          if (!response.ok) {
            get().logout();
            return false;
          }

          const data = await response.json();
          set({ token: data.accessToken, refreshToken: data.refreshToken });
          return true;
        } catch {
          get().logout();
          return false;
        }
      },

      updateProfile: async (data) => {
        const { token, user } = get();
        if (!token || !user) return;

        const response = await fetch(`https://api.example.com/users/${user.id}`, {
          method: 'PATCH',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
          },
          body: JSON.stringify(data),
        });

        if (response.ok) {
          const updatedUser = await response.json();
          set({ user: updatedUser });
        }
      },

      checkAuthStatus: async () => {
        const { token } = get();
        if (!token) {
          set({ isLoading: false });
          return;
        }
        try {
          const response = await fetch('https://api.example.com/auth/me', {
            headers: { Authorization: `Bearer ${token}` },
          });
          if (response.ok) {
            const user = await response.json();
            set({ user, isLoggedIn: true, isLoading: false });
          } else {
            const refreshed = await get().refreshAuth();
            if (!refreshed) {
              set({ isLoading: false });
            } else {
              await get().checkAuthStatus();
            }
          }
        } catch {
          set({ isLoading: false });
        }
      },
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({
        token: state.token,
        refreshToken: state.refreshToken,
      }),
    }
  )
);
```

---

## 11. 실전 패턴: Cart Store

```tsx
// stores/cartStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface CartItem {
  id: number;
  name: string;
  price: number;
  quantity: number;
  imageUrl: string;
}

interface CartState {
  items: CartItem[];
  addItem: (product: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: number) => void;
  updateQuantity: (id: number, quantity: number) => void;
  incrementQuantity: (id: number) => void;
  decrementQuantity: (id: number) => void;
  clearCart: () => void;
  getTotal: () => number;
  getItemCount: () => number;
  isInCart: (id: number) => boolean;
}

export const useCartStore = create<CartState>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (product) => set((state) => {
        const existing = state.items.find((i) => i.id === product.id);
        if (existing) {
          return {
            items: state.items.map((i) =>
              i.id === product.id
                ? { ...i, quantity: i.quantity + 1 }
                : i
            ),
          };
        }
        return { items: [...state.items, { ...product, quantity: 1 }] };
      }),

      removeItem: (id) => set((state) => ({
        items: state.items.filter((i) => i.id !== id),
      })),

      updateQuantity: (id, quantity) => set((state) => {
        if (quantity <= 0) {
          return { items: state.items.filter((i) => i.id !== id) };
        }
        return {
          items: state.items.map((i) =>
            i.id === id ? { ...i, quantity } : i
          ),
        };
      }),

      incrementQuantity: (id) => set((state) => ({
        items: state.items.map((i) =>
          i.id === id ? { ...i, quantity: i.quantity + 1 } : i
        ),
      })),

      decrementQuantity: (id) => set((state) => {
        const item = state.items.find((i) => i.id === id);
        if (item && item.quantity <= 1) {
          return { items: state.items.filter((i) => i.id !== id) };
        }
        return {
          items: state.items.map((i) =>
            i.id === id ? { ...i, quantity: i.quantity - 1 } : i
          ),
        };
      }),

      clearCart: () => set({ items: [] }),

      getTotal: () =>
        get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),

      getItemCount: () =>
        get().items.reduce((sum, i) => sum + i.quantity, 0),

      isInCart: (id) => get().items.some((i) => i.id === id),
    }),
    {
      name: 'cart-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({ items: state.items }),
    }
  )
);
```

---

## 12. 실전 패턴: UI Settings Store

```tsx
// stores/settingsStore.ts
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Appearance } from 'react-native';

type ThemeMode = 'light' | 'dark' | 'system';
type FontSize = 'small' | 'medium' | 'large';
type Language = 'ko' | 'en' | 'ja';

interface SettingsState {
  themeMode: ThemeMode;
  fontSize: FontSize;
  language: Language;
  notificationsEnabled: boolean;
  biometricEnabled: boolean;
  onboardingCompleted: boolean;

  setThemeMode: (mode: ThemeMode) => void;
  setFontSize: (size: FontSize) => void;
  setLanguage: (lang: Language) => void;
  toggleNotifications: () => void;
  toggleBiometric: () => void;
  completeOnboarding: () => void;
  resetSettings: () => void;

  // 파생 상태
  getEffectiveTheme: () => 'light' | 'dark';
  getFontScale: () => number;
}

const defaultSettings = {
  themeMode: 'system' as ThemeMode,
  fontSize: 'medium' as FontSize,
  language: 'ko' as Language,
  notificationsEnabled: true,
  biometricEnabled: false,
  onboardingCompleted: false,
};

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set, get) => ({
      ...defaultSettings,

      setThemeMode: (mode) => set({ themeMode: mode }),
      setFontSize: (size) => set({ fontSize: size }),
      setLanguage: (lang) => set({ language: lang }),
      toggleNotifications: () =>
        set((s) => ({ notificationsEnabled: !s.notificationsEnabled })),
      toggleBiometric: () =>
        set((s) => ({ biometricEnabled: !s.biometricEnabled })),
      completeOnboarding: () => set({ onboardingCompleted: true }),
      resetSettings: () => set(defaultSettings),

      getEffectiveTheme: () => {
        const { themeMode } = get();
        if (themeMode === 'system') {
          return Appearance.getColorScheme() ?? 'light';
        }
        return themeMode;
      },

      getFontScale: () => {
        const { fontSize } = get();
        switch (fontSize) {
          case 'small': return 0.85;
          case 'medium': return 1.0;
          case 'large': return 1.2;
        }
      },
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({
        themeMode: state.themeMode,
        fontSize: state.fontSize,
        language: state.language,
        notificationsEnabled: state.notificationsEnabled,
        biometricEnabled: state.biometricEnabled,
        onboardingCompleted: state.onboardingCompleted,
      }),
    }
  )
);
```

---

## 13. Android ViewModel+StateFlow vs Zustand 비교

동일한 기능(상품 목록 + 필터 + 즐겨찾기)을 Android ViewModel과 Zustand로 구현하여 직접 비교한다.

### 13-1. Android ViewModel + StateFlow

```kotlin
// Android: ProductViewModel.kt
data class ProductUiState(
    val products: List<Product> = emptyList(),
    val favorites: Set<Int> = emptySet(),
    val filter: String = "",
    val isLoading: Boolean = false,
    val error: String? = null,
)

@HiltViewModel
class ProductViewModel @Inject constructor(
    private val repository: ProductRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()

    // 파생 상태
    val filteredProducts: StateFlow<List<Product>> = _uiState
        .map { state ->
            state.products.filter {
                it.name.contains(state.filter, ignoreCase = true)
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    init { loadProducts() }

    fun loadProducts() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            try {
                val products = repository.getProducts()
                _uiState.update { it.copy(products = products, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }

    fun setFilter(filter: String) {
        _uiState.update { it.copy(filter = filter) }
    }

    fun toggleFavorite(productId: Int) {
        _uiState.update { state ->
            val newFavorites = if (productId in state.favorites) {
                state.favorites - productId
            } else {
                state.favorites + productId
            }
            state.copy(favorites = newFavorites)
        }
    }
}
```

### 13-2. Zustand 동일 구현

```tsx
// React Native: productStore.ts
import { create } from 'zustand';

interface Product {
  id: number;
  name: string;
  price: number;
  category: string;
}

interface ProductState {
  products: Product[];
  favorites: Set<number>;
  filter: string;
  isLoading: boolean;
  error: string | null;

  loadProducts: () => Promise<void>;
  setFilter: (filter: string) => void;
  toggleFavorite: (productId: number) => void;
  getFilteredProducts: () => Product[];
}

export const useProductStore = create<ProductState>()((set, get) => ({
  products: [],
  favorites: new Set<number>(),
  filter: '',
  isLoading: false,
  error: null,

  loadProducts: async () => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch('https://api.example.com/products');
      const products = await response.json();
      set({ products, isLoading: false });
    } catch (e: any) {
      set({ error: e.message, isLoading: false });
    }
  },

  setFilter: (filter) => set({ filter }),

  toggleFavorite: (productId) => set((state) => {
    const newFavorites = new Set(state.favorites);
    if (newFavorites.has(productId)) {
      newFavorites.delete(productId);
    } else {
      newFavorites.add(productId);
    }
    return { favorites: newFavorites };
  }),

  getFilteredProducts: () => {
    const { products, filter } = get();
    return products.filter((p) =>
      p.name.toLowerCase().includes(filter.toLowerCase())
    );
  },
}));
```

### 13-3. 비교 요약

```
Android ViewModel                    Zustand Store
─────────────────────────────────   ─────────────────────────────
class ProductViewModel               create<ProductState>()
MutableStateFlow(initialState)       초기값을 create 내부에 직접 정의
StateFlow<T>                         useStore(selector)
_uiState.update { it.copy(...) }     set((state) => ({ ... }))
viewModelScope.launch { }            async 함수 (자동 스코프)
stateIn(scope, started, initial)     selector + useMemo
@Inject constructor(repo)            직접 import (DI 불필요)
hiltViewModel()                      useStore() 호출 (Provider 불필요)
distinctUntilChanged()               useShallow() 또는 개별 selector
```

**Zustand의 압도적 장점**: 동일한 기능을 구현하는데 Android는 ViewModel + UiState data class + Repository + Hilt Module + StateFlow + stateIn 등 여러 파일과 보일러플레이트가 필요하지만, Zustand는 하나의 파일에 모든 것이 들어간다.

다음 문서에서는 TanStack Query를 사용한 서버 상태 관리를 다룬다.
