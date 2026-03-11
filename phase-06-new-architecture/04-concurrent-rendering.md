# Concurrent Rendering — React 19의 동시성 기능 활용하기

## 목차
1. [동시성 렌더링이란?](#1-동시성-렌더링이란)
2. [Automatic Batching](#2-automatic-batching)
3. [Transitions: useTransition, startTransition](#3-transitions)
4. [Suspense](#4-suspense)
5. [useDeferredValue](#5-usedeferredvalue)
6. [Error Boundaries](#6-error-boundaries)
7. [실전 패턴 모음](#7-실전-패턴-모음)
8. [성능 가이드: 언제 무엇을 쓸까](#8-성능-가이드)

---

## 1. 동시성 렌더링이란?

### 1.1 기존 렌더링의 문제

기존(Legacy) React 렌더링은 **동기적이고 중단 불가능**했다. 한 번 렌더링이 시작되면 끝날 때까지 다른 작업을 할 수 없었다.

```
┌────────────────────────────────────────────────────────────────┐
│             동기 렌더링 (Legacy)의 문제                           │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시나리오: 검색 필드 타이핑 중 1000개 결과 필터링                  │
│                                                                 │
│  시간 → ─────────────────────────────────────────────►          │
│                                                                 │
│  사용자: "a" 입력                                               │
│  JS Thread: [━━━━━━━ 1000개 필터링 + 렌더 (200ms) ━━━━━━━━]    │
│                                                                 │
│  사용자: "ab" 입력 (50ms 후)                                    │
│            ↑ 이 입력이 무시됨! JS Thread가 바쁘니까              │
│            └── 사용자는 "키보드가 먹통"이라고 느낌                 │
│                                                                 │
│  결과: 입력이 뚝뚝 끊기는 최악의 UX                              │
│                                                                 │
│  Android 비유:                                                   │
│  Main Thread에서 1000개 아이템을 DiffUtil 없이                   │
│  notifyDataSetChanged() 하는 것과 유사                           │
│  → ANR까지는 아니더라도 극심한 프레임 드롭                        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 동시성 렌더링의 해결

```
┌────────────────────────────────────────────────────────────────┐
│             동시성 렌더링 (Concurrent)의 해결                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시나리오: 동일한 검색 필드 타이핑                                │
│                                                                 │
│  시간 → ─────────────────────────────────────────────►          │
│                                                                 │
│  사용자: "a" 입력                                               │
│  JS Thread:                                                     │
│    [입력 즉시 반영(5ms)]                                        │
│    [━━ 필터링 시작 ━━╗ ← 긴급 업데이트 도착!                    │
│                      ║                                          │
│  사용자: "ab" 입력   ║                                          │
│    [입력 즉시 반영]   ║ ← 필터링 중단, 입력 먼저 처리            │
│                      ║                                          │
│  사용자: "abc" 입력  ║                                          │
│    [입력 즉시 반영]   ║ ← 또 중단, 입력 먼저                     │
│                      ║                                          │
│    [━━━━━━ "abc" 필터링 ━━━━━━] → 최종 결과만 렌더              │
│                                                                 │
│  결과: 입력은 항상 즉각 반응 + 최종 결과만 표시                   │
│                                                                 │
│  Android Compose 비유:                                          │
│  Compose의 recomposition도 중단 가능하다.                        │
│  LaunchedEffect 안에서 collectAsState를 사용할 때                │
│  더 긴급한 recomposition이 오면 현재 것을 중단할 수 있다.         │
│  → React의 Concurrent Rendering과 동일한 개념                    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 핵심 개념: 업데이트 우선순위

```
┌────────────────────────────────────────────────────────────────┐
│              React 19의 업데이트 우선순위                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─── 긴급 (Urgent) ──────────────────────────────────┐        │
│  │ • 텍스트 입력 (onChangeText)                        │        │
│  │ • 클릭/터치 (onPress)                               │        │
│  │ • 키보드 이벤트                                     │        │
│  │ → 즉시 처리, 다른 렌더를 중단할 수 있음              │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  ┌─── 전환 (Transition) ──────────────────────────────┐        │
│  │ • startTransition(() => setState(...))              │        │
│  │ • 검색 결과 필터링                                  │        │
│  │ • 탭 전환 콘텐츠                                    │        │
│  │ → 중단 가능, 긴급 업데이트에 양보                    │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  ┌─── 지연 (Deferred) ────────────────────────────────┐        │
│  │ • useDeferredValue                                  │        │
│  │ • 대량 리스트 렌더링                                │        │
│  │ → 가장 낮은 우선순위, 여유 있을 때 처리              │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                 │
│  Android 비유:                                                   │
│  긴급 = Choreographer.CALLBACK_INPUT                             │
│  전환 = Choreographer.CALLBACK_TRAVERSAL                         │
│  지연 = MessageQueue.IdleHandler                                 │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

```javascript [playground]
// 🧪 동시성 렌더링 개념 실습 — 우선순위 스케줄링

// 작업 우선순위 스케줄러 구현
function createScheduler() {
  const queues = { high: [], normal: [], low: [] };

  return {
    schedule(priority, task) {
      queues[priority].push(task);
    },
    flush() {
      // 높은 우선순위부터 처리
      const order = ['high', 'normal', 'low'];
      const results = [];

      for (const priority of order) {
        while (queues[priority].length > 0) {
          const task = queues[priority].shift();
          results.push({ priority, result: task() });
        }
      }
      return results;
    }
  };
}

const scheduler = createScheduler();

// useTransition 시뮬레이션: 긴급 업데이트 vs 전환 업데이트
console.log("=== 검색 입력 시나리오 ===");

// 사용자가 "hello" 입력
const input = "hello";

// 긴급: 입력 필드 업데이트 (즉시 반영)
scheduler.schedule('high', () => {
  console.log(`[긴급] 입력 필드 → "${input}"`);
  return input;
});

// 낮은 우선순위: 검색 결과 필터링 (지연 가능)
scheduler.schedule('low', () => {
  const items = ["hello world", "help me", "helmet", "hero", "heaven"];
  const filtered = items.filter(item => item.startsWith(input));
  console.log(`[전환] 검색 결과 필터링: ${filtered.length}개`);
  return filtered;
});

// 일반: UI 상태 업데이트
scheduler.schedule('normal', () => {
  console.log(`[일반] 로딩 스피너 표시`);
  return "loading";
});

// 스케줄러 실행
console.log("\n처리 순서:");
const results = scheduler.flush();
results.forEach((r, i) =>
  console.log(`${i + 1}. [${r.priority}] → ${JSON.stringify(r.result)}`)
);

console.log("\n→ 긴급 업데이트(입력)가 먼저, 전환(검색)이 나중에 처리됨");
console.log("→ React의 useTransition이 바로 이 원리!");
```

---

## 2. Automatic Batching

### 2.1 배칭이란?

여러 개의 `setState` 호출을 하나의 렌더링으로 합치는 것이다.

```tsx
// ═══ React 17 이전 — 이벤트 핸들러 내부만 배칭 ═══

function OldBehavior() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // 이벤트 핸들러 → 배칭 O (리렌더 1회)
  const handleClick = () => {
    setCount(c => c + 1);
    setFlag(f => !f);
    // → 리렌더 1회 (배칭 됨)
  };

  // setTimeout/Promise → 배칭 X (리렌더 2회!)
  const handleAsync = () => {
    fetch('/api/data').then(() => {
      setCount(c => c + 1);  // → 리렌더 1회
      setFlag(f => !f);      // → 리렌더 또 1회 (총 2회!)
    });
  };
}

// ═══ React 19 (0.84) — 모든 곳에서 자동 배칭 ═══

function NewBehavior() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  // 이벤트 핸들러 → 배칭 O
  const handleClick = () => {
    setCount(c => c + 1);
    setFlag(f => !f);
    // → 리렌더 1회
  };

  // setTimeout → 자동 배칭 O!
  const handleTimeout = () => {
    setTimeout(() => {
      setCount(c => c + 1);
      setFlag(f => !f);
      // → 리렌더 1회! (React 19에서는 자동 배칭)
    }, 1000);
  };

  // Promise → 자동 배칭 O!
  const handleAsync = async () => {
    const data = await fetch('/api/data');
    setCount(c => c + 1);
    setFlag(f => !f);
    // → 리렌더 1회! (React 19에서는 자동 배칭)
  };

  // Native 이벤트 리스너 → 자동 배칭 O!
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (state) => {
      setCount(c => c + 1);
      setFlag(f => !f);
      // → 리렌더 1회! (React 19에서는 자동 배칭)
    });
    return () => subscription.remove();
  }, []);
}
```

### 2.2 배칭을 원하지 않는 경우

```tsx
import { flushSync } from 'react-dom';

function ManualFlush() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  const handleClick = () => {
    // 각 setState마다 즉시 리렌더를 강제하고 싶을 때
    flushSync(() => {
      setCount(c => c + 1);
    });
    // 여기서 DOM이 이미 업데이트됨

    flushSync(() => {
      setFlag(f => !f);
    });
    // 여기서 또 DOM 업데이트됨

    // 총 2회 리렌더
    // 주의: 거의 사용할 일이 없음. 성능에 악영향.
  };
}
```

---

## 3. Transitions

### 3.1 useTransition

`useTransition`은 **낮은 우선순위 업데이트를 표시**하는 Hook이다.

```tsx
import { useState, useTransition } from 'react';
import { View, TextInput, FlatList, Text, ActivityIndicator, StyleSheet } from 'react-native';

// 큰 데이터셋 (10,000개 아이템)
const allItems = Array.from({ length: 10000 }, (_, i) => ({
  id: String(i),
  name: `아이템 ${i}`,
  category: ['음식', '전자', '의류', '도서'][i % 4],
}));

function SearchScreen() {
  const [query, setQuery] = useState('');
  const [filteredItems, setFilteredItems] = useState(allItems);
  const [isPending, startTransition] = useTransition();
  //     ↑ 전환 중인지    ↑ 전환 시작 함수

  const handleSearch = (text: string) => {
    // 1. 입력 필드 업데이트 — 긴급! 즉시 반영!
    setQuery(text);

    // 2. 필터링 결과 업데이트 — 전환! 중단 가능, 낮은 우선순위
    startTransition(() => {
      const filtered = allItems.filter(item =>
        item.name.includes(text) || item.category.includes(text)
      );
      setFilteredItems(filtered);
    });
  };

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        value={query}
        onChangeText={handleSearch}
        placeholder="검색어를 입력하세요..."
      />

      {/* isPending: transition이 진행 중일 때 true */}
      {isPending && (
        <View style={styles.loadingBar}>
          <ActivityIndicator size="small" color="#4A90D9" />
          <Text style={styles.loadingText}>검색 중...</Text>
        </View>
      )}

      <FlatList
        data={filteredItems}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <View style={styles.item}>
            <Text style={styles.itemName}>{item.name}</Text>
            <Text style={styles.itemCategory}>{item.category}</Text>
          </View>
        )}
        // 성능 최적화
        initialNumToRender={20}
        maxToRenderPerBatch={20}
        windowSize={5}
        style={{ opacity: isPending ? 0.7 : 1 }}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  input: { height: 48, borderWidth: 1, borderColor: '#ddd', borderRadius: 8,
    paddingHorizontal: 16, fontSize: 16, marginBottom: 8 },
  loadingBar: { flexDirection: 'row', alignItems: 'center', padding: 8 },
  loadingText: { marginLeft: 8, color: '#666' },
  item: { padding: 12, borderBottomWidth: 1, borderBottomColor: '#eee' },
  itemName: { fontSize: 16 },
  itemCategory: { fontSize: 12, color: '#888', marginTop: 4 },
});
```

### 3.2 동작 과정 상세

```
┌────────────────────────────────────────────────────────────────┐
│           useTransition 동작 과정                                │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  사용자가 "ab"를 빠르게 입력:                                    │
│                                                                 │
│  t=0ms: "a" 입력                                               │
│  ├── setQuery("a")           → 긴급 업데이트 → 즉시 렌더        │
│  ├── startTransition(() => { → 전환 업데이트 → 스케줄링          │
│  │     setFilteredItems(...)                                    │
│  │   })                                                         │
│  ├── isPending = true                                           │
│  └── 입력 필드에 "a" 즉시 표시                                   │
│                                                                 │
│  t=16ms: 전환 렌더링 시작 (필터링)                               │
│  ├── 10000개 중 "a" 포함 아이템 필터링 중...                     │
│  │                                                              │
│  t=30ms: "b" 입력 (전환 렌더링 중단!)                            │
│  ├── setQuery("ab")          → 긴급! 즉시 렌더!                 │
│  ├── 이전 전환 렌더링 폐기                                       │
│  ├── 새 startTransition 시작                                     │
│  └── 입력 필드에 "ab" 즉시 표시                                  │
│                                                                 │
│  t=46ms: 새 전환 렌더링 시작 ("ab" 필터링)                       │
│  │                                                              │
│  t=80ms: 전환 렌더링 완료                                        │
│  ├── isPending = false                                          │
│  └── 필터 결과 화면에 표시                                       │
│                                                                 │
│  핵심: "a" 필터링은 완료되지 않고 폐기됨                         │
│        "ab" 필터링만 실제로 화면에 반영됨                         │
│        입력은 항상 즉각 반응                                     │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 3.3 startTransition (단독 사용)

`useTransition` Hook 없이 `startTransition`만 단독으로 사용할 수도 있다. `isPending` 상태가 필요 없을 때 사용한다.

```tsx
import { startTransition, useState } from 'react';

function TabScreen() {
  const [activeTab, setActiveTab] = useState('home');

  const switchTab = (tab: string) => {
    // 탭 전환은 무거운 렌더를 유발할 수 있으므로 transition으로
    startTransition(() => {
      setActiveTab(tab);
    });
    // isPending 상태가 필요 없다면 이것으로 충분
  };

  return (
    <View>
      <View style={styles.tabBar}>
        {['home', 'search', 'profile'].map(tab => (
          <Pressable key={tab} onPress={() => switchTab(tab)}>
            <Text style={activeTab === tab ? styles.activeTab : styles.tab}>
              {tab}
            </Text>
          </Pressable>
        ))}
      </View>
      {activeTab === 'home' && <HeavyHomeScreen />}
      {activeTab === 'search' && <HeavySearchScreen />}
      {activeTab === 'profile' && <HeavyProfileScreen />}
    </View>
  );
}
```

```exercise
type: output-predict
question: "useTransition을 사용한 다음 코드에서 검색어 입력 시 어떤 일이 발생하나요?"
code: |
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleSearch = (text) => {
    setQuery(text);
    startTransition(() => {
      setResults(filterItems(text));
    });
  };
options:
  - "query 즉시 업데이트, results는 낮은 우선순위로 업데이트"
  - "query와 results 모두 즉시 업데이트"
  - "query와 results 모두 지연 업데이트"
  - "에러 발생: startTransition 내에서 setState 불가"
answer: "query 즉시 업데이트, results는 낮은 우선순위로 업데이트"
explanation: "startTransition 안의 상태 업데이트는 낮은 우선순위로 처리되어 UI 응답성을 유지합니다."
xp: 6
```

---

## 4. Suspense

### 4.1 Suspense란?

Suspense는 **비동기 작업의 로딩 상태를 선언적으로 처리**하는 React 기능이다. 자식 컴포넌트가 아직 준비되지 않았을 때 자동으로 fallback UI를 보여준다.

```
┌────────────────────────────────────────────────────────────────┐
│                 Suspense 개념                                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  이전 방식 (명령적):                                             │
│  if (isLoading) {                                               │
│    return <LoadingSpinner />;                                   │
│  }                                                              │
│  if (error) {                                                   │
│    return <ErrorView />;                                        │
│  }                                                              │
│  return <DataView data={data} />;                               │
│                                                                 │
│  Suspense 방식 (선언적):                                        │
│  <ErrorBoundary fallback={<ErrorView />}>                       │
│    <Suspense fallback={<LoadingSpinner />}>                     │
│      <DataView />  {/* 내부에서 데이터가 준비되면 자동 표시 */}    │
│    </Suspense>                                                  │
│  </ErrorBoundary>                                               │
│                                                                 │
│  Android 비유:                                                   │
│  Compose의 produceState + CircularProgressIndicator와 유사       │
│  하지만 더 선언적 — 로딩 상태를 컴포넌트 자체가 아닌               │
│  Suspense 경계에서 처리                                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 Suspense 경계(Boundary)

```tsx
import { Suspense } from 'react';
import { View, Text, ActivityIndicator, StyleSheet } from 'react-native';

// 로딩 컴포넌트 (fallback)
function LoadingScreen() {
  return (
    <View style={styles.loading}>
      <ActivityIndicator size="large" color="#4A90D9" />
      <Text style={styles.loadingText}>로딩 중...</Text>
    </View>
  );
}

// 앱의 최상위
function App() {
  return (
    <View style={styles.app}>
      {/* Header는 항상 즉시 표시 */}
      <Header />

      {/* Suspense 경계: 내부 컴포넌트가 준비될 때까지 fallback 표시 */}
      <Suspense fallback={<LoadingScreen />}>
        <MainContent />
      </Suspense>

      {/* Footer도 항상 즉시 표시 */}
      <Footer />
    </View>
  );
}
```

### 4.3 중첩된 Suspense 경계

```tsx
function DashboardScreen() {
  return (
    <View style={styles.dashboard}>
      {/* 각 섹션에 독립적인 Suspense 경계 */}
      {/* → 각 섹션이 독립적으로 로딩/표시 */}

      <Suspense fallback={<SkeletonCard />}>
        <UserProfile />
        {/* 유저 정보가 로드되면 표시 */}
      </Suspense>

      <Suspense fallback={<SkeletonList />}>
        <RecentOrders />
        {/* 주문 목록이 로드되면 표시 */}
      </Suspense>

      <Suspense fallback={<SkeletonChart />}>
        <Statistics />
        {/* 통계 데이터가 로드되면 표시 */}
      </Suspense>
    </View>
  );
}
```

```
┌────────────────────────────────────────────────────────────────┐
│             중첩 Suspense의 로딩 순서                             │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  t=0ms:    ┌────────────────────────────┐                      │
│            │ [SkeletonCard          ]   │                      │
│            │ [SkeletonList          ]   │                      │
│            │ [SkeletonChart         ]   │                      │
│            └────────────────────────────┘                      │
│                                                                 │
│  t=200ms:  ┌────────────────────────────┐                      │
│  (프로필    │ [UserProfile ✓ 완료    ]   │ ← 프로필 먼저 로드   │
│   로드)    │ [SkeletonList          ]   │                      │
│            │ [SkeletonChart         ]   │                      │
│            └────────────────────────────┘                      │
│                                                                 │
│  t=500ms:  ┌────────────────────────────┐                      │
│  (주문     │ [UserProfile ✓         ]   │                      │
│   로드)    │ [RecentOrders ✓ 완료   ]   │ ← 주문 목록 로드     │
│            │ [SkeletonChart         ]   │                      │
│            └────────────────────────────┘                      │
│                                                                 │
│  t=800ms:  ┌────────────────────────────┐                      │
│  (통계     │ [UserProfile ✓         ]   │                      │
│   로드)    │ [RecentOrders ✓        ]   │                      │
│            │ [Statistics ✓ 완료     ]   │ ← 통계 로드          │
│            └────────────────────────────┘                      │
│                                                                 │
│  각 섹션이 독립적으로 로딩됨 → 점진적 콘텐츠 표시                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 4.4 Suspense + TanStack Query 통합

TanStack Query (React Query)와 Suspense를 함께 사용하면 데이터 fetching이 매우 깔끔해진다.

```tsx
// 설치: npm install @tanstack/react-query

import { QueryClient, QueryClientProvider, useSuspenseQuery } from '@tanstack/react-query';
import { Suspense } from 'react';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 2,
      staleTime: 5 * 60 * 1000, // 5분
    },
  },
});

// 앱 최상위
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MainScreen />
    </QueryClientProvider>
  );
}

// 데이터를 사용하는 컴포넌트 — 로딩/에러 처리가 없다!
function UserList() {
  // useSuspenseQuery: 데이터가 준비되지 않으면 Suspense로 위임
  const { data: users } = useSuspenseQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('https://api.example.com/users');
      return response.json();
    },
  });

  // 여기에 도달하면 data는 항상 존재함 (undefined 체크 불필요!)
  return (
    <FlatList
      data={users}
      keyExtractor={item => item.id}
      renderItem={({ item }) => (
        <View style={styles.userRow}>
          <Text style={styles.userName}>{item.name}</Text>
          <Text style={styles.userEmail}>{item.email}</Text>
        </View>
      )}
    />
  );
}

// 메인 화면
function MainScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.header}>사용자 목록</Text>
      <Suspense fallback={<UserListSkeleton />}>
        <UserList />
      </Suspense>
    </View>
  );
}

// 스켈레톤 로딩 UI
function UserListSkeleton() {
  return (
    <View>
      {Array.from({ length: 5 }).map((_, i) => (
        <View key={i} style={styles.skeletonRow}>
          <View style={styles.skeletonName} />
          <View style={styles.skeletonEmail} />
        </View>
      ))}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  header: { fontSize: 24, fontWeight: 'bold', marginBottom: 16 },
  userRow: { padding: 12, borderBottomWidth: 1, borderBottomColor: '#eee' },
  userName: { fontSize: 16, fontWeight: '600' },
  userEmail: { fontSize: 14, color: '#666', marginTop: 4 },
  skeletonRow: { padding: 12, borderBottomWidth: 1, borderBottomColor: '#eee' },
  skeletonName: { width: 150, height: 16, backgroundColor: '#e0e0e0', borderRadius: 4 },
  skeletonEmail: { width: 200, height: 14, backgroundColor: '#e0e0e0', borderRadius: 4,
    marginTop: 6 },
});
```

### 4.5 Suspense + useTransition 조합

```tsx
function TabContainer() {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  const switchTab = (tab: string) => {
    startTransition(() => {
      setActiveTab(tab);
    });
  };

  return (
    <View style={styles.container}>
      {/* 탭 바 — 항상 즉시 반응 */}
      <View style={styles.tabBar}>
        {['home', 'search', 'settings'].map(tab => (
          <Pressable
            key={tab}
            onPress={() => switchTab(tab)}
            style={[styles.tab, activeTab === tab && styles.activeTab]}
          >
            <Text>{tab}</Text>
          </Pressable>
        ))}
      </View>

      {/* isPending이면 현재 콘텐츠를 dim 처리 */}
      <View style={{ opacity: isPending ? 0.6 : 1, flex: 1 }}>
        <Suspense fallback={<ActivityIndicator size="large" />}>
          {activeTab === 'home' && <HomeTab />}
          {activeTab === 'search' && <SearchTab />}
          {activeTab === 'settings' && <SettingsTab />}
        </Suspense>
      </View>
    </View>
  );
}
```

```
동작 과정:
1. "search" 탭 클릭
2. startTransition → 낮은 우선순위로 activeTab 변경
3. isPending = true → 현재 화면 dim 처리 (사용자에게 "전환 중" 힌트)
4. SearchTab이 로딩 필요하면 → Suspense fallback 표시
   SearchTab이 이미 캐시됨면 → 즉시 표시
5. 로딩 완료 → isPending = false → 정상 opacity
6. 만약 로딩 중 다른 탭 클릭 → 이전 전환 중단, 새 전환 시작!
```

---

## 5. useDeferredValue

### 5.1 useDeferredValue란?

`useDeferredValue`는 값의 "지연 복사본"을 만든다. 긴급한 렌더에서는 이전 값을 사용하고, 여유 있을 때 새 값으로 업데이트한다.

```tsx
import { useState, useDeferredValue, useMemo } from 'react';

function ProductSearch() {
  const [query, setQuery] = useState('');

  // query의 지연 복사본 — 긴급 렌더에서는 이전 값 유지
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;
  // isStale = true면 아직 이전 결과를 보여주는 중

  return (
    <View style={styles.container}>
      <TextInput
        style={styles.input}
        value={query}
        onChangeText={setQuery}
        placeholder="상품 검색..."
      />

      <View style={{ opacity: isStale ? 0.5 : 1 }}>
        {/* deferredQuery는 query보다 "느리게" 따라옴 */}
        <ProductList query={deferredQuery} />
      </View>
    </View>
  );
}

// 무거운 리스트 컴포넌트
const ProductList = React.memo(function ProductList({ query }: { query: string }) {
  // 이 계산은 deferredQuery가 변경될 때만 실행됨
  // query가 빠르게 변해도 deferredQuery는 여유 있을 때만 따라옴
  const filtered = useMemo(() => {
    return hugeProductList.filter(product =>
      product.name.toLowerCase().includes(query.toLowerCase())
    );
  }, [query]);

  return (
    <FlatList
      data={filtered}
      keyExtractor={item => item.id}
      renderItem={({ item }) => (
        <View style={styles.product}>
          <Text style={styles.productName}>{item.name}</Text>
          <Text style={styles.productPrice}>{item.price}원</Text>
        </View>
      )}
    />
  );
});
```

### 5.2 useTransition vs useDeferredValue — 언제 어떤 것을?

```
┌────────────────────────────────────────────────────────────────┐
│         useTransition vs useDeferredValue                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  useTransition:                                                 │
│  • setState를 감싸서 사용                                       │
│  • isPending 상태 제공                                          │
│  • 상태 업데이트를 직접 제어할 수 있을 때 사용                    │
│  • 예: 검색어 입력 핸들러에서 결과 필터링                        │
│                                                                 │
│  useDeferredValue:                                              │
│  • 값을 감싸서 사용                                             │
│  • 상태 업데이트를 직접 제어할 수 없을 때 유용                    │
│  • 부모에서 props로 받은 값을 지연시킬 때                        │
│  • 예: 부모가 전달한 query prop을 지연                           │
│                                                                 │
│  선택 기준:                                                     │
│  ┌──────────────────────────────────────────────────┐          │
│  │ 내가 setState를 제어할 수 있나?                    │          │
│  │   YES → useTransition                             │          │
│  │   NO (props로 받음) → useDeferredValue             │          │
│  └──────────────────────────────────────────────────┘          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

```tsx
// useTransition이 적합한 경우:
function SearchScreenOwnedState() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (text: string) => {
    setQuery(text); // 긴급
    startTransition(() => {
      setResults(filterData(text)); // 전환
    });
  };
  // ...
}

// useDeferredValue가 적합한 경우:
function SearchResults({ query }: { query: string }) {
  // query는 부모에서 props로 받음 — 내가 제어할 수 없음!
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => filterData(deferredQuery), [deferredQuery]);
  // ...
}
```

---

## 6. Error Boundaries

### 6.1 Error Boundary란?

렌더링 중 발생하는 에러를 포착하여 앱 전체가 크래시되는 것을 방지하는 컴포넌트이다.

```
┌────────────────────────────────────────────────────────────────┐
│               Error Boundary 없을 때 vs 있을 때                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Error Boundary 없음:                                           │
│  ┌─────────────────────────┐                                   │
│  │ App                     │                                   │
│  │  ├── Header ✓           │                                   │
│  │  ├── Content            │                                   │
│  │  │    └── BuggyWidget ✗ │ ← 에러 발생!                     │
│  │  └── Footer ✓           │                                   │
│  └─────────────────────────┘                                   │
│  → 앱 전체 하얀 화면 (크래시)                                    │
│                                                                 │
│  Error Boundary 있음:                                           │
│  ┌─────────────────────────┐                                   │
│  │ App                     │                                   │
│  │  ├── Header ✓           │ ← 정상 표시                       │
│  │  ├── ErrorBoundary      │                                   │
│  │  │    └── BuggyWidget ✗ │ ← 에러 발생!                     │
│  │  │    → "문제가 발생함"  │ ← fallback UI 표시               │
│  │  └── Footer ✓           │ ← 정상 표시                       │
│  └─────────────────────────┘                                   │
│  → 에러 부분만 대체 UI, 나머지 정상 작동                          │
│                                                                 │
│  Android 비유:                                                   │
│  try-catch로 특정 뷰만 에러 처리하는 것                           │
│  또는 Fragment별 에러 처리와 유사                                 │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 6.2 Error Boundary 구현

React Native에서 Error Boundary는 클래스 컴포넌트로만 만들 수 있다. (Hook으로는 만들 수 없다.)

```tsx
// src/components/ErrorBoundary.tsx

import React, { Component, ErrorInfo, ReactNode } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  // 에러 발생 시 호출 — state 업데이트
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  // 에러 로깅 (Crashlytics, Sentry 등에 보고)
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('ErrorBoundary caught:', error, errorInfo);
    this.props.onError?.(error, errorInfo);
    // 예: Sentry.captureException(error);
  }

  // 재시도 기능
  handleRetry = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      // 커스텀 fallback이 있으면 사용
      if (this.props.fallback) {
        return this.props.fallback;
      }

      // 기본 에러 UI
      return (
        <View style={styles.errorContainer}>
          <Text style={styles.errorTitle}>문제가 발생했습니다</Text>
          <Text style={styles.errorMessage}>
            {this.state.error?.message || '알 수 없는 오류'}
          </Text>
          <Pressable style={styles.retryButton} onPress={this.handleRetry}>
            <Text style={styles.retryText}>다시 시도</Text>
          </Pressable>
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = StyleSheet.create({
  errorContainer: { flex: 1, justifyContent: 'center', alignItems: 'center',
    padding: 32 },
  errorTitle: { fontSize: 20, fontWeight: 'bold', marginBottom: 8, color: '#D32F2F' },
  errorMessage: { fontSize: 14, color: '#666', textAlign: 'center', marginBottom: 16 },
  retryButton: { backgroundColor: '#4A90D9', paddingHorizontal: 24, paddingVertical: 12,
    borderRadius: 8 },
  retryText: { color: 'white', fontSize: 16, fontWeight: '600' },
});

export default ErrorBoundary;
```

### 6.3 Error Boundary + Suspense 조합

```tsx
function RobustScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.header}>대시보드</Text>

      {/* 에러와 로딩을 모두 처리하는 완전한 패턴 */}
      <ErrorBoundary
        fallback={
          <View style={styles.errorSection}>
            <Text>프로필을 불러올 수 없습니다</Text>
          </View>
        }
      >
        <Suspense fallback={<ProfileSkeleton />}>
          <UserProfile />
        </Suspense>
      </ErrorBoundary>

      <ErrorBoundary
        fallback={
          <View style={styles.errorSection}>
            <Text>주문 내역을 불러올 수 없습니다</Text>
          </View>
        }
      >
        <Suspense fallback={<OrderListSkeleton />}>
          <RecentOrders />
        </Suspense>
      </ErrorBoundary>
    </View>
  );
}
```

```
처리 흐름:
1. 최초 렌더: Suspense fallback(Skeleton) 표시
2. 데이터 로딩 완료: 실제 컴포넌트 표시
3. 데이터 로딩 실패: ErrorBoundary fallback(에러 메시지) 표시
4. 렌더 중 에러: ErrorBoundary fallback 표시
→ 어떤 상황에서도 앱이 크래시되지 않음!
```

---

## 7. 실전 패턴 모음

### 패턴 1: 디바운스 검색 (useTransition 활용)

```tsx
function DebouncedSearch() {
  const [input, setInput] = useState('');
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (text: string) => {
    setInput(text);

    startTransition(() => {
      // startTransition 안에서는 자동으로 중단 가능
      // → 별도 debounce 라이브러리 불필요!
      if (text.length >= 2) {
        const filtered = performSearch(text);
        setResults(filtered);
      } else {
        setResults([]);
      }
    });
  };

  return (
    <View style={styles.container}>
      <TextInput
        value={input}
        onChangeText={handleSearch}
        placeholder="2글자 이상 입력..."
        style={styles.input}
      />
      {isPending && <Text style={styles.searching}>검색 중...</Text>}
      <SearchResultList results={results} />
    </View>
  );
}
```

### 패턴 2: 스켈레톤 로딩 (Suspense + React.lazy)

```tsx
import { Suspense, lazy } from 'react';

// 무거운 화면을 lazy로 분리 → 필요할 때 로드
const SettingsScreen = lazy(() => import('./screens/SettingsScreen'));
const AnalyticsScreen = lazy(() => import('./screens/AnalyticsScreen'));

function Navigator({ currentScreen }: { currentScreen: string }) {
  return (
    <ErrorBoundary>
      <Suspense fallback={<FullScreenSkeleton />}>
        {currentScreen === 'settings' && <SettingsScreen />}
        {currentScreen === 'analytics' && <AnalyticsScreen />}
      </Suspense>
    </ErrorBoundary>
  );
}

function FullScreenSkeleton() {
  return (
    <View style={styles.skeleton}>
      <View style={styles.skeletonHeader} />
      <View style={styles.skeletonBody}>
        {Array.from({ length: 6 }).map((_, i) => (
          <View key={i} style={styles.skeletonRow} />
        ))}
      </View>
    </View>
  );
}
```

### 패턴 3: 점진적 콘텐츠 로딩

```tsx
function ProgressiveLoadingScreen() {
  return (
    <ScrollView style={styles.container}>
      {/* 즉시 표시: 정적 헤더 */}
      <View style={styles.header}>
        <Text style={styles.title}>마이페이지</Text>
      </View>

      {/* 빠르게 로드: 캐시된 사용자 정보 */}
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile />
      </Suspense>

      {/* 중간 속도: 최근 활동 */}
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />
      </Suspense>

      {/* 느리게 로드: 추천 콘텐츠 (AI 생성 등) */}
      <Suspense fallback={<RecommendationSkeleton />}>
        <Recommendations />
      </Suspense>
    </ScrollView>
  );
}
```

### 패턴 4: 대량 리스트 필터링 (useDeferredValue)

```tsx
function LargeListFilter({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');
  const deferredFilter = useDeferredValue(filter);
  const isStale = filter !== deferredFilter;

  const filteredItems = useMemo(() => {
    if (!deferredFilter) return items;
    return items.filter(item =>
      item.title.toLowerCase().includes(deferredFilter.toLowerCase()) ||
      item.description.toLowerCase().includes(deferredFilter.toLowerCase())
    );
  }, [items, deferredFilter]);

  return (
    <View style={styles.container}>
      <TextInput
        value={filter}
        onChangeText={setFilter}
        placeholder="필터..."
        style={styles.input}
      />

      {isStale && (
        <View style={styles.staleIndicator}>
          <Text style={styles.staleText}>업데이트 중...</Text>
        </View>
      )}

      <FlatList
        data={filteredItems}
        keyExtractor={item => item.id}
        renderItem={({ item }) => <ItemCard item={item} />}
        style={{ opacity: isStale ? 0.7 : 1 }}
        initialNumToRender={15}
        maxToRenderPerBatch={10}
      />
    </View>
  );
}
```

---

## 8. 성능 가이드

### 8.1 언제 무엇을 사용할까

```
┌────────────────────────────────────────────────────────────────┐
│              동시성 기능 선택 가이드                               │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  문제: "입력 시 UI가 버벅인다"                                   │
│  → useTransition (입력 = 긴급, 결과 = 전환)                     │
│                                                                 │
│  문제: "부모에서 받은 props가 너무 자주 변한다"                   │
│  → useDeferredValue (props의 지연 복사본)                        │
│                                                                 │
│  문제: "데이터 로딩 상태를 깔끔하게 처리하고 싶다"                 │
│  → Suspense + TanStack Query (useSuspenseQuery)                 │
│                                                                 │
│  문제: "에러가 나면 화면 전체가 하얗게 된다"                      │
│  → ErrorBoundary (섹션별 에러 격리)                              │
│                                                                 │
│  문제: "탭 전환 시 버벅인다"                                     │
│  → useTransition (탭 전환을 전환으로) + Suspense                 │
│                                                                 │
│  문제: "setState를 여러 번 하면 렌더가 여러 번 된다"              │
│  → React 19 자동 배칭이 알아서 처리 (추가 코드 불필요)            │
│                                                                 │
│  문제: "무거운 화면 코드를 분리하고 싶다"                         │
│  → React.lazy + Suspense (코드 분할)                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 8.2 주의사항

```
┌────────────────────────────────────────────────────────────────┐
│              동시성 기능 사용 시 주의사항                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 모든 곳에 useTransition을 쓰지 마라                          │
│     → 실제로 무거운 렌더가 있을 때만 의미 있음                    │
│     → 간단한 setState에는 오히려 오버헤드                        │
│                                                                 │
│  2. useDeferredValue는 React.memo와 함께 써야 효과적             │
│     → memo 없으면 부모 리렌더 시 자식도 리렌더                   │
│     → deferredValue의 의미가 없어짐                             │
│                                                                 │
│  3. Suspense fallback은 가벼워야 한다                            │
│     → Skeleton UI는 정적이어야 함                                │
│     → fallback 안에서 또 데이터 fetch하면 무한 루프 위험          │
│                                                                 │
│  4. ErrorBoundary는 이벤트 핸들러 에러를 잡지 못한다             │
│     → 렌더링 중 에러만 잡음                                     │
│     → 이벤트 핸들러는 try-catch 사용                             │
│                                                                 │
│  5. startTransition 안에서 await 하지 마라                       │
│     → transition은 동기 콜백만 받음                              │
│     ✗ startTransition(async () => { await fetch(...) })         │
│     ✓ startTransition(() => { setState(newData) })              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 8.3 Android Compose와의 최종 비교

```
┌───────────────────────────────┬──────────────────────────────┐
│      Android Compose           │     React Native 0.84        │
├───────────────────────────────┼──────────────────────────────┤
│ mutableStateOf → recomposition│ useState → re-render          │
│ Concurrent composition        │ Concurrent rendering          │
│ Snapshot system               │ React Fiber tree              │
│ LaunchedEffect + loading flag │ Suspense + fallback           │
│ remember { } (메모이제이션)    │ useMemo, useCallback          │
│ derivedStateOf (파생 상태)     │ useDeferredValue              │
│ 없음 (수동 구현)              │ useTransition (우선순위 분리)  │
│ try-catch in ViewModel        │ ErrorBoundary                 │
│ collectAsStateWithLifecycle   │ useSuspenseQuery              │
│ Modifier.animateContentSize() │ LayoutAnimation               │
└───────────────────────────────┴──────────────────────────────┘
```

---

## 요약

```
┌────────────────────────────────────────────────────────────────┐
│           Concurrent Rendering 핵심 정리                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  React 19 + Fabric = 동시성 렌더링 가능                          │
│                                                                 │
│  ┌──────────────────┐ 여러 setState → 1회 렌더                  │
│  │ Automatic Batching│ 추가 코드 불필요, 자동 적용               │
│  └──────────────────┘                                           │
│                                                                 │
│  ┌──────────────────┐ 긴급/전환 업데이트 분리                    │
│  │ useTransition    │ 입력은 즉시, 결과는 여유있게               │
│  └──────────────────┘                                           │
│                                                                 │
│  ┌──────────────────┐ 비동기 로딩 선언적 처리                    │
│  │ Suspense         │ fallback으로 스켈레톤 표시                 │
│  └──────────────────┘                                           │
│                                                                 │
│  ┌──────────────────┐ 값의 지연 복사본                           │
│  │ useDeferredValue │ props 기반 최적화에 유용                   │
│  └──────────────────┘                                           │
│                                                                 │
│  ┌──────────────────┐ 렌더 에러 격리                             │
│  │ Error Boundary   │ 섹션별 에러 처리, 앱 크래시 방지            │
│  └──────────────────┘                                           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
