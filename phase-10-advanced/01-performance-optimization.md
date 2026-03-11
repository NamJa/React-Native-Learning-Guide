# 성능 최적화 완전 가이드 — Android 성능 튜닝 경험을 활용하기

## 목차
1. [성능 마인드셋: Android와 다른 점](#1-성능-마인드셋-android와-다른-점)
2. [렌더링 성능](#2-렌더링-성능)
3. [리스트 성능](#3-리스트-성능)
4. [이미지 최적화](#4-이미지-최적화)
5. [번들 크기 최적화](#5-번들-크기-최적화)
6. [메모리 관리](#6-메모리-관리)
7. [앱 시작 시간 최적화](#7-앱-시작-시간-최적화)
8. [네이티브 성능](#8-네이티브-성능)
9. [성능 모니터링 도구](#9-성능-모니터링-도구)

---

## 1. 성능 마인드셋: Android와 다른 점

### Android vs React Native 성능 모델

```
[Android 네이티브]
단일 스레드 모델 (UI 스레드 중심)
┌─────────────────────────┐
│      UI Thread          │  ← 렌더링 + 이벤트 처리 + 비즈니스 로직
│  (Main Thread)          │     무거운 작업은 Coroutine/Worker로 분리
└─────────────────────────┘

[React Native — 기존 아키텍처 (Bridge)]
┌─────────────────────────┐     ┌─────────────────────────┐
│      JS Thread          │ ←→  │      UI Thread          │
│  React 렌더링           │ Bridge│  네이티브 UI 렌더링     │
│  비즈니스 로직          │     │  터치 이벤트 처리       │
│  상태 관리              │     │  애니메이션 (일부)      │
└─────────────────────────┘     └─────────────────────────┘
       ↕ Bridge (병목 지점!)

[React Native — 새 아키텍처 (Fabric + TurboModules)]
┌─────────────────────────┐     ┌─────────────────────────┐
│      JS Thread          │ ←→  │      UI Thread          │
│  React 렌더링           │ JSI  │  네이티브 UI 렌더링     │
│  비즈니스 로직          │(직접)│  터치 이벤트 처리       │
│  상태 관리              │     │  애니메이션 (Reanimated) │
└─────────────────────────┘     └─────────────────────────┘
       ↕ JSI (훨씬 빠름, 동기 호출 가능)

[핵심 차이]
1. JS 스레드가 바쁘면 UI가 버벅임 → JS 부하 최소화가 핵심
2. 두 스레드 간 통신이 성능 병목 → 통신 횟수 줄이기
3. 새 아키텍처(JSI)로 통신 오버헤드 대폭 감소
```

---

## 2. 렌더링 성능

### 2.1 React.memo — 불필요한 리렌더링 방지

Android Compose의 `remember`와 `derivedStateOf`에 해당한다.

```typescript
// 문제: 부모 리렌더링 시 자식도 무조건 리렌더링
function ParentScreen() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('홍길동');

  return (
    <View>
      <Text>{count}</Text>
      <Button title="증가" onPress={() => setCount(c => c + 1)} />
      {/* count가 바뀔 때마다 ExpensiveChild도 리렌더링됨 */}
      <ExpensiveChild name={name} />
    </View>
  );
}

// ❌ 매번 리렌더링 (name이 바뀌지 않았는데도)
function ExpensiveChild({ name }: { name: string }) {
  // 무거운 렌더링 로직
  return <Text>{name}</Text>;
}

// ✅ React.memo로 감싸면 props가 바뀔 때만 리렌더링
// Compose의 @Stable과 유사한 효과
const ExpensiveChild = React.memo(function ExpensiveChild({ name }: { name: string }) {
  return <Text>{name}</Text>;
});

// 커스텀 비교 함수 (복잡한 객체 props일 때)
const UserCard = React.memo(
  function UserCard({ user }: { user: User }) {
    return (
      <View>
        <Text>{user.name}</Text>
        <Text>{user.email}</Text>
      </View>
    );
  },
  (prevProps, nextProps) => {
    // true를 반환하면 리렌더링 안 함
    return prevProps.user.id === nextProps.user.id;
  }
);
```

```jsx [snack]
import React, { useState, useRef } from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';

// React.memo 없이 — 부모 리렌더링 시 매번 리렌더링
function NormalChild({ name }) {
  const renderCount = useRef(0);
  renderCount.current++;
  return (
    <View style={styles.card}>
      <Text style={styles.label}>memo 없음</Text>
      <Text>이름: {name}</Text>
      <Text style={styles.count}>렌더링 횟수: {renderCount.current}</Text>
    </View>
  );
}

// React.memo 사용 — props가 바뀔 때만 리렌더링
const MemoChild = React.memo(function MemoChild({ name }) {
  const renderCount = useRef(0);
  renderCount.current++;
  return (
    <View style={[styles.card, styles.memoCard]}>
      <Text style={styles.label}>React.memo 적용</Text>
      <Text>이름: {name}</Text>
      <Text style={styles.count}>렌더링 횟수: {renderCount.current}</Text>
    </View>
  );
});

export default function App() {
  const [count, setCount] = useState(0);
  const [name] = useState('홍길동');

  return (
    <View style={styles.container}>
      <Text style={styles.title}>React.memo 비교 데모</Text>
      <Text style={styles.subtitle}>카운터: {count}</Text>
      <Button title="카운터 증가 (부모 리렌더링)" onPress={() => setCount(c => c + 1)} />
      <View style={styles.row}>
        <NormalChild name={name} />
        <MemoChild name={name} />
      </View>
      <Text style={styles.hint}>
        카운터를 눌러도 name은 변하지 않습니다.{'\n'}
        memo 적용된 컴포넌트는 리렌더링되지 않습니다.
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, justifyContent: 'center', backgroundColor: '#f5f5f5' },
  title: { fontSize: 20, fontWeight: 'bold', textAlign: 'center', marginBottom: 8 },
  subtitle: { fontSize: 16, textAlign: 'center', marginBottom: 12, color: '#555' },
  row: { flexDirection: 'row', justifyContent: 'space-around', marginTop: 20 },
  card: { padding: 16, borderRadius: 12, backgroundColor: '#fff', elevation: 2, alignItems: 'center', width: '45%' },
  memoCard: { borderColor: '#4CAF50', borderWidth: 2 },
  label: { fontSize: 12, fontWeight: 'bold', color: '#888', marginBottom: 4 },
  count: { marginTop: 8, fontSize: 18, fontWeight: 'bold', color: '#e53935' },
  hint: { marginTop: 20, textAlign: 'center', color: '#777', fontSize: 13, lineHeight: 20 },
});
```

### 2.2 useMemo — 비싼 계산 메모이제이션

```typescript
// Compose의 remember { expensiveCalculation() }에 해당

function ProductList({ products, sortBy }: Props) {
  // ❌ 매 렌더링마다 정렬 (O(n log n))
  const sorted = products.sort((a, b) => a[sortBy] - b[sortBy]);

  // ✅ products나 sortBy가 바뀔 때만 정렬
  const sorted = useMemo(
    () => [...products].sort((a, b) => a[sortBy] - b[sortBy]),
    [products, sortBy]
  );

  // ✅ 필터링 + 정렬 같이 무거운 연산
  const processedProducts = useMemo(() => {
    const filtered = products.filter(p => p.isAvailable);
    const sorted = filtered.sort((a, b) => a.price - b.price);
    const grouped = groupByCategory(sorted);
    return grouped;
  }, [products]);

  return <FlatList data={sorted} renderItem={...} />;
}
```

### 2.3 useCallback — 함수 참조 안정화

```typescript
// React.memo와 함께 사용해야 효과가 있음

function TodoList({ todos }: { todos: Todo[] }) {
  // ❌ 매 렌더링마다 새 함수 생성 → TodoItem이 memo여도 리렌더링
  const handleDelete = (id: string) => {
    deleteTodo(id);
  };

  // ✅ 함수 참조가 안정적 → TodoItem이 memo면 리렌더링 방지
  const handleDelete = useCallback((id: string) => {
    deleteTodo(id);
  }, []);  // 의존성이 없으면 [] 사용

  return (
    <FlatList
      data={todos}
      renderItem={({ item }) => (
        <TodoItem todo={item} onDelete={handleDelete} />
      )}
    />
  );
}

const TodoItem = React.memo(function TodoItem({
  todo,
  onDelete,
}: {
  todo: Todo;
  onDelete: (id: string) => void;
}) {
  return (
    <View>
      <Text>{todo.title}</Text>
      <Button title="삭제" onPress={() => onDelete(todo.id)} />
    </View>
  );
});
```

```jsx [snack]
import React, { useState, useCallback, useRef } from 'react';
import { View, Text, Button, FlatList, TouchableOpacity, StyleSheet } from 'react-native';

const TodoItem = React.memo(function TodoItem({ todo, onDelete }) {
  const renderCount = useRef(0);
  renderCount.current++;
  return (
    <View style={styles.item}>
      <Text style={styles.todoText}>{todo.title}</Text>
      <Text style={styles.renderBadge}>R:{renderCount.current}</Text>
      <TouchableOpacity onPress={() => onDelete(todo.id)} style={styles.deleteBtn}>
        <Text style={styles.deleteTxt}>삭제</Text>
      </TouchableOpacity>
    </View>
  );
});

let nextId = 4;

export default function App() {
  const [todos, setTodos] = useState([
    { id: '1', title: '장보기' },
    { id: '2', title: '운동하기' },
    { id: '3', title: 'React Native 공부' },
  ]);

  // useCallback으로 함수 참조 안정화 → TodoItem이 memo여도 리렌더링 방지
  const handleDelete = useCallback((id) => {
    setTodos(prev => prev.filter(t => t.id !== id));
  }, []);

  const addTodo = () => {
    setTodos(prev => [...prev, { id: String(nextId++), title: `할 일 ${nextId}` }]);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>useCallback + React.memo 데모</Text>
      <Text style={styles.hint}>R:N = 해당 아이템의 렌더링 횟수</Text>
      <Button title="할 일 추가" onPress={addTodo} />
      <FlatList
        data={todos}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <TodoItem todo={item} onDelete={handleDelete} />}
        style={styles.list}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, paddingTop: 60, backgroundColor: '#f5f5f5' },
  title: { fontSize: 20, fontWeight: 'bold', textAlign: 'center', marginBottom: 4 },
  hint: { textAlign: 'center', color: '#888', marginBottom: 12, fontSize: 13 },
  list: { marginTop: 12 },
  item: { flexDirection: 'row', alignItems: 'center', backgroundColor: '#fff', padding: 14, borderRadius: 10, marginBottom: 8, elevation: 1 },
  todoText: { flex: 1, fontSize: 16 },
  renderBadge: { backgroundColor: '#E3F2FD', paddingHorizontal: 8, paddingVertical: 2, borderRadius: 8, fontSize: 12, color: '#1565C0', marginRight: 10 },
  deleteBtn: { backgroundColor: '#ffebee', paddingHorizontal: 12, paddingVertical: 6, borderRadius: 8 },
  deleteTxt: { color: '#c62828', fontWeight: 'bold' },
});
```

### 2.4 useMemo/useCallback을 사용하지 말아야 할 때

```typescript
// ❌ 불필요한 useMemo — 간단한 계산
const fullName = useMemo(
  () => `${firstName} ${lastName}`,
  [firstName, lastName]
);
// ✅ 그냥 직접 계산
const fullName = `${firstName} ${lastName}`;

// ❌ 불필요한 useCallback — React.memo 없는 자식에게 전달
const handlePress = useCallback(() => {
  doSomething();
}, []);
// 자식이 React.memo가 아니면 어차피 리렌더링됨

// [규칙]
// 1. useMemo: O(n) 이상의 연산, 큰 배열 처리, 복잡한 객체 생성
// 2. useCallback: React.memo 자식에게 전달하는 함수, useEffect 의존성
// 3. 성능 문제가 없으면 사용하지 않아도 됨 (조기 최적화 금지)
```

### 2.5 Key 최적화

```typescript
// FlatList/map에서 key 사용 방법

// ❌ index를 key로 사용 — 아이템 순서가 변경되면 전체 리렌더링
data.map((item, index) => <Card key={index} item={item} />);

// ✅ 고유 ID를 key로 사용 — 변경된 항목만 리렌더링
data.map((item) => <Card key={item.id} item={item} />);

// FlatList의 keyExtractor
<FlatList
  data={items}
  keyExtractor={(item) => item.id}  // ✅ 고유 ID
  renderItem={({ item }) => <Card item={item} />}
/>
```

### 2.6 컴포넌트 분리 전략

```typescript
// ❌ 하나의 거대한 컴포넌트 — 어느 상태든 바뀌면 전체 리렌더링
function ProfileScreen() {
  const [name, setName] = useState('');
  const [bio, setBio] = useState('');
  const [avatar, setAvatar] = useState('');
  const [followers, setFollowers] = useState(0);
  const [posts, setPosts] = useState<Post[]>([]);

  return (
    <ScrollView>
      <Image source={{ uri: avatar }} />
      <TextInput value={name} onChangeText={setName} />
      <TextInput value={bio} onChangeText={setBio} />
      <Text>팔로워: {followers}</Text>
      <PostList posts={posts} />
    </ScrollView>
  );
}

// ✅ 상태별로 컴포넌트를 분리 — 각 부분만 리렌더링
function ProfileScreen() {
  return (
    <ScrollView>
      <ProfileAvatar />        {/* avatar 변경 시 여기만 리렌더링 */}
      <ProfileForm />          {/* name/bio 변경 시 여기만 리렌더링 */}
      <FollowerCount />        {/* followers 변경 시 여기만 리렌더링 */}
      <PostList />             {/* posts 변경 시 여기만 리렌더링 */}
    </ScrollView>
  );
}
```

---

## 3. 리스트 성능

### 3.1 FlatList 최적화

Android의 `RecyclerView` 최적화와 직접 비교한다.

```typescript
// RecyclerView 최적화 → FlatList 최적화 대응표
//
// recyclerView.setHasFixedSize(true)  → getItemLayout
// adapter.setHasStableIds(true)       → keyExtractor
// recyclerView.setItemViewCacheSize() → maxToRenderPerBatch
// recyclerView.setNestedScrollingEnabled(false) → 중첩 방지

function OptimizedList({ data }: { data: Item[] }) {
  // renderItem을 컴포넌트 외부에서 정의하거나 useCallback 사용
  const renderItem = useCallback(
    ({ item }: { item: Item }) => <ListItem item={item} />,
    []
  );

  const keyExtractor = useCallback(
    (item: Item) => item.id,
    []
  );

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={keyExtractor}

      // 아이템 크기가 고정인 경우 (RecyclerView의 setHasFixedSize)
      getItemLayout={(data, index) => ({
        length: 80,   // 각 아이템 높이
        offset: 80 * index,
        index,
      })}

      // 한 번에 렌더링할 아이템 수 (기본: 10)
      maxToRenderPerBatch={10}

      // 뷰포트 밖의 아이템을 메모리에 유지하는 범위 (기본: 21)
      // 작게 하면 메모리 절약, 크게 하면 스크롤 부드러움
      windowSize={5}

      // 초기 렌더링 아이템 수
      initialNumToRender={10}

      // 화면 밖 아이템의 네이티브 뷰를 제거 (메모리 절약)
      // RecyclerView의 뷰 재활용과 유사
      removeClippedSubviews={true}

      // 리스트 끝 도달 감지 임계값 (무한 스크롤용)
      onEndReachedThreshold={0.5}
      onEndReached={loadMore}

      // 아이템 구분자
      ItemSeparatorComponent={() => <View style={{ height: 8 }} />}

      // 빈 리스트 표시
      ListEmptyComponent={<EmptyState />}
    />
  );
}

// 아이템 컴포넌트는 반드시 React.memo로 감쌈
const ListItem = React.memo(function ListItem({ item }: { item: Item }) {
  return (
    <View style={styles.item}>
      <Text>{item.title}</Text>
    </View>
  );
});
```

```jsx [snack]
import React, { useState, useCallback, useMemo } from 'react';
import { View, Text, FlatList, TouchableOpacity, StyleSheet } from 'react-native';

const COLORS = ['#E3F2FD', '#FFF3E0', '#E8F5E9', '#FCE4EC', '#F3E5F5', '#E0F7FA'];

// 더미 데이터 생성
const DATA = Array.from({ length: 200 }, (_, i) => ({
  id: String(i),
  title: `아이템 #${i + 1}`,
  subtitle: `이것은 ${i + 1}번째 항목의 설명입니다`,
  color: COLORS[i % COLORS.length],
}));

const ListItem = React.memo(function ListItem({ item, onPress }) {
  return (
    <TouchableOpacity onPress={() => onPress(item.id)} style={[styles.item, { backgroundColor: item.color }]}>
      <Text style={styles.itemTitle}>{item.title}</Text>
      <Text style={styles.itemSub}>{item.subtitle}</Text>
    </TouchableOpacity>
  );
});

export default function App() {
  const [selected, setSelected] = useState(null);

  const handlePress = useCallback((id) => {
    setSelected(id);
  }, []);

  const renderItem = useCallback(({ item }) => (
    <ListItem item={item} onPress={handlePress} />
  ), [handlePress]);

  const keyExtractor = useCallback((item) => item.id, []);

  // getItemLayout으로 고정 높이 아이템 최적화
  const getItemLayout = useCallback((data, index) => ({
    length: 80,
    offset: 80 * index,
    index,
  }), []);

  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.title}>FlatList 최적화 데모</Text>
        <Text style={styles.subtitle}>200개 아이템 | 선택: {selected ?? '없음'}</Text>
      </View>
      <FlatList
        data={DATA}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
        getItemLayout={getItemLayout}
        maxToRenderPerBatch={10}
        windowSize={5}
        initialNumToRender={10}
        removeClippedSubviews={true}
        ItemSeparatorComponent={() => <View style={{ height: 1 }} />}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#fafafa' },
  header: { padding: 20, paddingTop: 50, backgroundColor: '#fff', borderBottomWidth: 1, borderBottomColor: '#eee' },
  title: { fontSize: 20, fontWeight: 'bold' },
  subtitle: { fontSize: 14, color: '#888', marginTop: 4 },
  item: { height: 79, paddingHorizontal: 16, justifyContent: 'center' },
  itemTitle: { fontSize: 16, fontWeight: '600' },
  itemSub: { fontSize: 13, color: '#666', marginTop: 2 },
});
```

### 3.2 FlashList — 고성능 대안

RecyclerView의 뷰 재활용 메커니즘을 JavaScript에서 구현한 라이브러리다. FlatList보다 훨씬 빠르다.

```bash
npm install @shopify/flash-list
```

```typescript
// FlashList — RecyclerView에 더 가까운 동작
import { FlashList } from '@shopify/flash-list';

function HighPerformanceList({ data }: { data: Item[] }) {
  return (
    <FlashList
      data={data}
      renderItem={({ item }) => <ListItem item={item} />}
      estimatedItemSize={80}  // 필수! 아이템의 예상 높이
      keyExtractor={(item) => item.id}

      // FlashList 전용 옵션
      drawDistance={300}  // 미리 렌더링할 거리 (px)
      overrideItemLayout={(layout, item) => {
        // 아이템별로 다른 크기 지정 가능
        layout.size = item.type === 'header' ? 120 : 80;
      }}
    />
  );
}

// FlatList vs FlashList 성능 비교:
// - 1000개 아이템 렌더링: FlashList이 약 5배 빠름
// - 빠른 스크롤 시: FlashList은 빈 공간이 거의 없음
// - 메모리 사용량: FlashList이 약 50% 적음
```

### 3.3 SectionList 최적화

```typescript
// RecyclerView + ItemDecoration으로 섹션을 구현하는 것에 해당
import { SectionList } from 'react-native';

interface Section {
  title: string;
  data: Item[];
}

function GroupedList({ sections }: { sections: Section[] }) {
  return (
    <SectionList
      sections={sections}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <ListItem item={item} />}
      renderSectionHeader={({ section }) => (
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>{section.title}</Text>
        </View>
      )}
      stickySectionHeadersEnabled={true}  // 섹션 헤더 고정
      getItemLayout={getSectionListItemLayout}
    />
  );
}
```

---

## 4. 이미지 최적화

### 캐싱 전략

```typescript
// expo-image의 캐싱 (Glide/Coil의 캐싱과 유사)
import { Image } from 'expo-image';

function OptimizedImage({ url }: { url: string }) {
  return (
    <Image
      source={{ uri: url }}
      style={{ width: 200, height: 200 }}
      // 캐시 정책
      cachePolicy="memory-disk"  // 메모리 + 디스크 캐시 (기본)
      // cachePolicy="memory"    // 메모리 캐시만
      // cachePolicy="disk"      // 디스크 캐시만
      // cachePolicy="none"      // 캐시 없음

      // 로딩 중 표시 (Glide의 placeholder에 해당)
      placeholder={require('../assets/placeholder.png')}

      // 로딩 전환 효과 (Glide의 crossFade에 해당)
      transition={200}

      // 이미지 크기 조절 (Glide의 centerCrop/fitCenter에 해당)
      contentFit="cover"
    />
  );
}
```

### 점진적 로딩

```typescript
// 저해상도 → 고해상도 로딩 (thumbnail → full)
function ProgressiveImage({ thumbnailUrl, fullUrl }: Props) {
  return (
    <Image
      source={{ uri: fullUrl }}
      placeholder={{ uri: thumbnailUrl }}  // 먼저 썸네일 표시
      placeholderContentFit="cover"
      contentFit="cover"
      transition={300}
      style={{ width: '100%', height: 300 }}
    />
  );
}

// BlurHash 사용 (Instagram 스타일 로딩)
function BlurHashImage({ url, blurhash }: { url: string; blurhash: string }) {
  return (
    <Image
      source={{ uri: url }}
      placeholder={{ blurhash }}  // 로딩 중 블러 프리뷰
      contentFit="cover"
      transition={200}
      style={{ width: '100%', height: 200 }}
    />
  );
}
```

---

## 5. 번들 크기 최적화

### Tree Shaking

```typescript
// ❌ 전체 라이브러리 임포트
import _ from 'lodash'; // 전체 lodash가 번들에 포함 (~70KB)
_.debounce(fn, 300);

// ✅ 필요한 함수만 임포트
import debounce from 'lodash/debounce'; // debounce만 포함 (~2KB)
debounce(fn, 300);

// ❌ 전체 아이콘 임포트
import { Ionicons } from '@expo/vector-icons'; // 모든 Ionicons

// ✅ 아이콘 트리 쉐이킹 (번들 사이즈에 영향은 expo-font 캐싱으로 최소화됨)
// @expo/vector-icons는 자동으로 필요한 글리프만 로드
```

### React.lazy + Suspense (코드 분할)

```typescript
// 무거운 화면을 지연 로딩 (Android Fragment의 동적 로딩과 유사)
import React, { Suspense, lazy } from 'react';
import { ActivityIndicator, View } from 'react-native';

// ❌ 모든 화면을 앱 시작 시 로드
import { SettingsScreen } from './SettingsScreen';
import { ProfileEditScreen } from './ProfileEditScreen';
import { AnalyticsScreen } from './AnalyticsScreen';

// ✅ 필요할 때 로드 (번들 분할)
const SettingsScreen = lazy(() => import('./SettingsScreen'));
const ProfileEditScreen = lazy(() => import('./ProfileEditScreen'));
const AnalyticsScreen = lazy(() => import('./AnalyticsScreen'));

// Suspense로 로딩 상태 표시
function Navigator() {
  return (
    <Suspense fallback={
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
      </View>
    }>
      <Stack.Navigator>
        <Stack.Screen name="Settings" component={SettingsScreen} />
        <Stack.Screen name="ProfileEdit" component={ProfileEditScreen} />
        <Stack.Screen name="Analytics" component={AnalyticsScreen} />
      </Stack.Navigator>
    </Suspense>
  );
}
```

### 번들 분석

```bash
# 번들 크기 시각화
npx react-native-bundle-visualizer

# 또는 metro 번들 통계 보기
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js \
  --sourcemap-output /tmp/bundle.map

# source-map-explorer로 분석
npx source-map-explorer /tmp/bundle.js /tmp/bundle.map
```

---

## 6. 메모리 관리

### 메모리 누수 탐지

```typescript
// Android의 LeakCanary에 해당하는 패턴

// ❌ 메모리 누수 — 컴포넌트 언마운트 후에도 상태 업데이트 시도
function UserProfile() {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    fetchUser().then((data) => {
      setUser(data); // 컴포넌트가 이미 언마운트되었으면 경고/누수!
    });
  }, []);
}

// ✅ cleanup 함수로 방지
function UserProfile() {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    let isMounted = true;

    fetchUser().then((data) => {
      if (isMounted) {
        setUser(data);
      }
    });

    // cleanup 함수 (Android의 onCleared()에 해당)
    return () => {
      isMounted = false;
    };
  }, []);
}

// ✅ AbortController 사용 (더 나은 방법)
function UserProfile() {
  const [user, setUser] = useState<User | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch('/api/user', { signal: controller.signal })
      .then(res => res.json())
      .then(setUser)
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error(err);
        }
      });

    return () => controller.abort(); // cleanup: 요청 취소
  }, []);
}
```

### 타이머/구독 정리

```typescript
// ❌ 타이머 미정리 — 메모리 누수
function Countdown() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('tick');
    }, 1000);
    // cleanup이 없음!
  }, []);
}

// ✅ 타이머 정리
function Countdown() {
  useEffect(() => {
    const timer = setInterval(() => {
      console.log('tick');
    }, 1000);

    return () => clearInterval(timer); // 반드시 정리!
  }, []);
}

// ✅ 이벤트 리스너 정리
function KeyboardTracker() {
  useEffect(() => {
    const subscription = Keyboard.addListener('keyboardDidShow', (e) => {
      console.log('키보드 높이:', e.endCoordinates.height);
    });

    return () => subscription.remove(); // cleanup
  }, []);
}

// ✅ WebSocket 정리
function ChatRoom({ roomId }: { roomId: string }) {
  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/chat/${roomId}`);
    ws.onmessage = (event) => { /* 메시지 처리 */ };

    return () => {
      ws.close(); // cleanup: WebSocket 연결 종료
    };
  }, [roomId]);
}
```

### 대용량 데이터 처리

```typescript
// 큰 배열을 화면에 표시할 때 — 모든 데이터를 메모리에 올리지 않기

// ❌ 모든 데이터를 한 번에 로드
const [allItems, setAllItems] = useState<Item[]>([]); // 10,000개!

// ✅ 페이지네이션으로 점진적 로드
function useInfiniteItems() {
  return useInfiniteQuery({
    queryKey: ['items'],
    queryFn: ({ pageParam = 1 }) => fetchItems(pageParam, 20),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextPage,
  });
}

// ✅ FlatList의 windowSize로 화면 밖 아이템 메모리 해제
<FlatList
  data={items}
  windowSize={5}           // 화면의 5배만 메모리에 유지
  removeClippedSubviews    // 화면 밖 뷰 제거
/>
```

---

## 7. 앱 시작 시간 최적화

### Hermes 바이트코드 사전 컴파일

```
[Hermes가 시작 시간을 줄이는 원리]

JSC (JavaScriptCore) 엔진:
  소스 코드 → [파싱] → [컴파일] → [실행]
  앱 시작 시 모든 과정을 거침 → 느림

Hermes 엔진:
  빌드 시: 소스 코드 → [파싱] → [컴파일] → 바이트코드 (.hbc 파일)
  앱 시작 시: 바이트코드 → [실행]  (파싱/컴파일 생략!)
  → 시작 시간 50-70% 단축

// Hermes는 React Native 0.70+에서 기본 활성화
// android/app/build.gradle에서 확인:
// hermesEnabled = true (기본값)
```

### 지연 모듈 초기화

```typescript
// ❌ 앱 시작 시 모든 것을 한 번에 초기화
import { initSentry } from './sentry';
import { initAnalytics } from './analytics';
import { initPushNotifications } from './pushNotifications';
import { preloadFonts } from './fonts';
import { loadTranslations } from './i18n';

// 앱 시작 시 모두 실행 → 느린 시작
initSentry();
initAnalytics();
initPushNotifications();
preloadFonts();
loadTranslations();

// ✅ 필수 초기화만 먼저, 나머지는 지연
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();

export default function App() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    async function init() {
      // 1단계: 필수 초기화 (스플래시 중)
      await loadTranslations();  // UI 표시에 필수
      await preloadFonts();      // UI 표시에 필수

      setIsReady(true);
      await SplashScreen.hideAsync();

      // 2단계: 지연 초기화 (스플래시 이후, 비동기)
      initSentry();              // 크래시 리포팅 (약간 늦어도 됨)
      initAnalytics();           // 분석 (백그라운드)
      initPushNotifications();   // 푸시 (사용자가 앱을 사용 중일 때)
    }

    init();
  }, []);

  if (!isReady) return null;
  return <App />;
}
```

### 스플래시 스크린 최적화

```typescript
// 스플래시 스크린 동안 초기 데이터를 미리 로드
import * as SplashScreen from 'expo-splash-screen';

SplashScreen.preventAutoHideAsync();

export default function App() {
  const [appIsReady, setAppIsReady] = useState(false);

  useEffect(() => {
    async function prepare() {
      try {
        // 병렬로 초기화 작업 수행 (Promise.all)
        await Promise.all([
          loadFonts(),
          loadTranslations(),
          checkAuthToken(),
          prefetchCriticalData(),  // 홈 화면 데이터 미리 로드
        ]);
      } catch (e) {
        console.warn(e);
      } finally {
        setAppIsReady(true);
      }
    }
    prepare();
  }, []);

  const onLayoutRootView = useCallback(async () => {
    if (appIsReady) {
      // 첫 번째 화면이 렌더링된 후 스플래시 숨김
      await SplashScreen.hideAsync();
    }
  }, [appIsReady]);

  if (!appIsReady) return null;

  return (
    <View style={{ flex: 1 }} onLayout={onLayoutRootView}>
      <AppProviders>
        <RootNavigator />
      </AppProviders>
    </View>
  );
}
```

---

## 8. 네이티브 성능

### 새 아키텍처의 장점

```
[구 아키텍처 — Bridge]
JS Thread → JSON 직렬화 → Bridge → JSON 역직렬화 → Native
                    ↑ 비동기, 배치, 오버헤드 큼

[새 아키텍처 — JSI (JavaScript Interface)]
JS Thread → JSI → Native
         ↑ 동기 호출 가능, 직렬화 불필요, 매우 빠름

[성능 향상]
- 네이티브 모듈 호출: ~10배 빠름
- UI 업데이트: 동기 렌더링 가능 (Fabric)
- 메모리: 공유 메모리로 복사 제거
```

### Worklets (Reanimated) — UI 스레드에서 실행

```typescript
// 무거운 계산을 UI 스레드에서 실행하여 JS 스레드 부하를 피함
import Animated, {
  useSharedValue,
  useAnimatedScrollHandler,
  useAnimatedStyle,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

function ParallaxHeader() {
  const scrollY = useSharedValue(0);

  // 스크롤 핸들러가 UI 스레드에서 실행됨 (worklet)
  // JS 스레드를 거치지 않아 60fps 유지
  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      // 이 코드는 UI 스레드에서 실행됨!
      scrollY.value = event.contentOffset.y;
    },
  });

  const headerStyle = useAnimatedStyle(() => ({
    height: interpolate(
      scrollY.value,
      [0, 200],
      [300, 60],
      Extrapolation.CLAMP
    ),
    opacity: interpolate(
      scrollY.value,
      [0, 100],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));

  return (
    <View style={{ flex: 1 }}>
      <Animated.View style={[styles.header, headerStyle]}>
        <Text>패럴랙스 헤더</Text>
      </Animated.View>
      <Animated.FlatList
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        data={items}
        renderItem={renderItem}
      />
    </View>
  );
}
```

---

## 9. 성능 모니터링 도구

### React DevTools Profiler 사용법

```
[Profiler 사용 단계]

1. React Native DevTools 열기 (Metro에서 'j' 키)
2. Profiler 탭 선택
3. 설정(톱니바퀴): "Record why each component rendered" 활성화
4. ⏺ 녹화 시작
5. 앱에서 성능이 느린 작업 수행
6. ⏹ 녹화 중지
7. 결과 분석:

[Flamegraph 해석]
- 넓은 막대: 렌더링 시간이 긴 컴포넌트
- 회색 막대: 이 커밋에서 리렌더링되지 않은 컴포넌트
- 노란/빨간 막대: 렌더링이 느린 컴포넌트 (최적화 필요!)

[Why did this render? 결과]
- "Props changed": props가 변경됨
- "State changed": 상태가 변경됨
- "Hooks changed": 훅의 값이 변경됨
- "Parent rendered": 부모가 리렌더링되어 같이 렌더링됨
  → React.memo로 방지 가능!
```

### 커스텀 성능 측정

```typescript
// 렌더링 횟수 추적 (디버그용)
function useRenderCount(name: string) {
  const count = useRef(0);
  count.current++;

  useEffect(() => {
    if (__DEV__) {
      console.log(`[Render] ${name}: ${count.current}번째`);
    }
  });
}

// 성능 측정 유틸
function usePerformanceTracker(name: string) {
  useEffect(() => {
    const start = performance.now();

    return () => {
      const duration = performance.now() - start;
      if (__DEV__ && duration > 16.67) {
        // 16.67ms = 1 프레임 (60fps)
        console.warn(`[Perf] ${name}: ${duration.toFixed(1)}ms (> 1 frame)`);
      }
    };
  });
}

// React의 <Profiler> 컴포넌트 활용
import { Profiler, ProfilerOnRenderCallback } from 'react';

const onRender: ProfilerOnRenderCallback = (
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) => {
  if (__DEV__) {
    console.log(`[Profiler] ${id}: ${phase} ${actualDuration.toFixed(1)}ms`);
  }
};

function App() {
  return (
    <Profiler id="HomeScreen" onRender={onRender}>
      <HomeScreen />
    </Profiler>
  );
}
```

### 성능 최적화 체크리스트

```
[React Native 성능 최적화 체크리스트]

렌더링:
[ ] React.memo로 불필요한 리렌더링 방지
[ ] useMemo/useCallback을 적절히 사용
[ ] 컴포넌트를 적절한 크기로 분리
[ ] key prop으로 고유 ID 사용

리스트:
[ ] FlatList 또는 FlashList 사용 (ScrollView + map 금지)
[ ] getItemLayout 설정 (고정 높이 아이템)
[ ] renderItem을 useCallback으로 메모이제이션
[ ] removeClippedSubviews 활성화
[ ] windowSize 조정

이미지:
[ ] 적절한 크기의 이미지 사용 (원본 크기가 아닌)
[ ] 캐싱 활성화 (expo-image)
[ ] 지연 로딩 적용
[ ] WebP 포맷 사용

메모리:
[ ] useEffect cleanup 함수로 리소스 해제
[ ] 타이머/구독 정리
[ ] 대용량 데이터는 페이지네이션

앱 시작:
[ ] Hermes 엔진 활성화
[ ] 지연 초기화 패턴 적용
[ ] 스플래시 화면 동안 사전 로드

네이티브:
[ ] 새 아키텍처(Fabric + TurboModules) 활용
[ ] 애니메이션은 Reanimated (UI 스레드)
[ ] 무거운 계산은 네이티브 모듈로
```

```javascript [playground]
// 🧪 성능 최적화 패턴 실습

// 1) 메모이제이션 — 비싼 계산 캐싱
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

// 피보나치 — 메모이제이션 효과 비교
function fibSlow(n) {
  if (n <= 1) return n;
  return fibSlow(n - 1) + fibSlow(n - 2);
}

const fibFast = memoize(function fib(n) {
  if (n <= 1) return n;
  return fibFast(n - 1) + fibFast(n - 2);
});

let t1 = performance.now();
const slow = fibSlow(30);
const slowTime = performance.now() - t1;

let t2 = performance.now();
const fast = fibFast(30);
const fastTime = performance.now() - t2;

console.log(`fib(30) = ${slow}`);
console.log(`일반: ${slowTime.toFixed(2)}ms`);
console.log(`메모: ${fastTime.toFixed(2)}ms`);
console.log(`속도 향상: ${(slowTime / Math.max(fastTime, 0.01)).toFixed(0)}배\n`);

// 2) 배치 업데이트 — 불필요한 중간 계산 방지
function batchProcess(items, batchSize = 100) {
  const batches = [];
  for (let i = 0; i < items.length; i += batchSize) {
    batches.push(items.slice(i, i + batchSize));
  }
  return batches;
}

const bigList = Array.from({ length: 1000 }, (_, i) => i);
const batches = batchProcess(bigList, 200);
console.log(`1000개 → ${batches.length}개 배치`);
batches.forEach((batch, i) =>
  console.log(`  배치 ${i}: ${batch.length}개 (${batch[0]}~${batch[batch.length-1]})`)
);

// 3) 객체 비교 최적화 — 얕은 비교(shallow compare)
function shallowEqual(a, b) {
  if (a === b) return true;
  const keysA = Object.keys(a);
  const keysB = Object.keys(b);
  if (keysA.length !== keysB.length) return false;
  return keysA.every(key => a[key] === b[key]);
}

const props1 = { name: "홍길동", count: 5 };
const props2 = { name: "홍길동", count: 5 };
const props3 = { name: "홍길동", count: 6 };
console.log("\n얕은 비교:");
console.log("동일 값:", shallowEqual(props1, props2)); // true
console.log("다른 값:", shallowEqual(props1, props3)); // false
```

## ✅ 학습 확인 퀴즈

```quiz
type: mcq
question: "FlatList에서 항목 높이가 고정일 때 성능을 크게 향상시키는 prop은?"
options:
  - "windowSize"
  - "maxToRenderPerBatch"
  - "getItemLayout"
  - "removeClippedSubviews"
answer: "getItemLayout"
explanation: "getItemLayout을 제공하면 FlatList가 항목의 위치를 미리 계산할 수 있어, 동적 측정 비용을 제거하고 스크롤 성능을 크게 향상시킵니다."
```

```quiz
type: mcq
question: "React.memo의 역할은?"
options:
  - "컴포넌트의 state를 메모이제이션"
  - "props가 변경되지 않으면 re-render를 건너뜀"
  - "useEffect의 실행을 최적화"
  - "네트워크 응답을 캐싱"
answer: "props가 변경되지 않으면 re-render를 건너뜀"
explanation: "React.memo는 고차 컴포넌트로, 이전 props와 새 props를 얕은 비교(shallow comparison)하여 변경이 없으면 re-render를 건너뜁니다."
```
