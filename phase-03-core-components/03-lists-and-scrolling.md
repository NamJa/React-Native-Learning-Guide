# 리스트와 스크롤 — RecyclerView에서 FlatList로

> Android 개발자에게 가장 익숙한 RecyclerView를 React Native의 FlatList로 대체하는 방법을 학습합니다.
> 가상화(Virtualization), 무한 스크롤, 당겨서 새로고침 등 실전 패턴을 모두 다룹니다.

---

## 1. ScrollView — 기본 스크롤 컨테이너

### Android 대응: `ScrollView` / `NestedScrollView`

React Native의 `ScrollView`는 Android의 `ScrollView`와 동일한 역할을 합니다. **모든 자식을 한 번에 렌더링**합니다.

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `horizontal` | `boolean` | 수평 스크롤 | `HorizontalScrollView` |
| `showsVerticalScrollIndicator` | `boolean` | 세로 스크롤바 표시 | `android:scrollbars` |
| `showsHorizontalScrollIndicator` | `boolean` | 가로 스크롤바 표시 | `android:scrollbars` |
| `pagingEnabled` | `boolean` | 페이지 단위 스크롤 | `ViewPager` |
| `bounces` | `boolean` | iOS 바운스 효과 | — (Android에는 없음) |
| `overScrollMode` | `string` | Android 오버스크롤 | `android:overScrollMode` |
| `contentContainerStyle` | `ViewStyle` | 내부 콘텐츠 컨테이너 스타일 | — |
| `keyboardShouldPersistTaps` | `string` | 키보드와 탭 동작 | — |
| `keyboardDismissMode` | `string` | 스크롤 시 키보드 동작 | — |
| `refreshControl` | `RefreshControl` | 당겨서 새로고침 | `SwipeRefreshLayout` |
| `onScroll` | `(event) => void` | 스크롤 이벤트 | `OnScrollChangeListener` |
| `scrollEventThrottle` | `number` | onScroll 호출 빈도 (ms) | — |
| `stickyHeaderIndices` | `number[]` | 고정 헤더 인덱스 | `isNestedScrollingEnabled` |
| `nestedScrollEnabled` | `boolean` | 중첩 스크롤 허용 (Android) | `NestedScrollView` |

### 예제 1: 기본 세로 스크롤

```tsx
import React from 'react';
import { ScrollView, View, Text, StyleSheet } from 'react-native';

const BasicScrollExample = () => {
  return (
    <ScrollView
      style={styles.container}
      contentContainerStyle={styles.contentContainer}
      showsVerticalScrollIndicator={true}
    >
      {Array.from({ length: 20 }, (_, i) => (
        <View key={i} style={styles.item}>
          <Text style={styles.itemText}>항목 {i + 1}</Text>
        </View>
      ))}
    </ScrollView>
  );
};

// Android 대응:
// <ScrollView>
//     <LinearLayout android:orientation="vertical">
//         <TextView android:text="항목 1" />
//         <TextView android:text="항목 2" />
//         ...
//     </LinearLayout>
// </ScrollView>

const styles = StyleSheet.create({
  container: { flex: 1 },
  contentContainer: { padding: 16 },
  item: {
    padding: 16,
    marginBottom: 8,
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
  },
  itemText: { fontSize: 16 },
});
```

```jsx [snack]
import React from 'react';
import { ScrollView, View, Text, StyleSheet } from 'react-native';

const BasicScrollExample = () => {
  return (
    <ScrollView
      style={styles.container}
      contentContainerStyle={styles.contentContainer}
      showsVerticalScrollIndicator={true}
    >
      {Array.from({ length: 20 }, (_, i) => (
        <View key={i} style={styles.item}>
          <Text style={styles.itemText}>항목 {i + 1}</Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  contentContainer: { padding: 16 },
  item: {
    padding: 16,
    marginBottom: 8,
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
  },
  itemText: { fontSize: 16 },
});

export default BasicScrollExample;
```

### 예제 2: 수평 스크롤 (카러셀)

```tsx
import React from 'react';
import { ScrollView, View, Text, Dimensions, StyleSheet } from 'react-native';

const { width: screenWidth } = Dimensions.get('window');

const HorizontalScrollExample = () => {
  const colors = ['#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7'];

  return (
    <ScrollView
      horizontal={true}
      pagingEnabled={true}
      showsHorizontalScrollIndicator={false}
      decelerationRate="fast"
      snapToInterval={screenWidth * 0.85 + 16}
      snapToAlignment="center"
      contentContainerStyle={{ paddingHorizontal: screenWidth * 0.075 }}
    >
      {colors.map((color, index) => (
        <View
          key={index}
          style={[styles.page, { backgroundColor: color, width: screenWidth * 0.85 }]}
        >
          <Text style={styles.pageText}>페이지 {index + 1}</Text>
        </View>
      ))}
    </ScrollView>
  );
};

const styles = StyleSheet.create({
  page: {
    height: 200,
    borderRadius: 12,
    justifyContent: 'center',
    alignItems: 'center',
    marginHorizontal: 8,
  },
  pageText: { color: '#fff', fontSize: 24, fontWeight: 'bold' },
});
```

### ScrollView를 사용해야 할 때 vs FlatList를 사용해야 할 때

| 상황 | ScrollView | FlatList |
|------|-----------|---------|
| 아이템 수 적음 (< 20-30) | 적합 | 가능하지만 과도 |
| 아이템 수 많음 (> 50) | 성능 문제 | 적합 (가상화) |
| 동적 데이터 (API) | 부적합 | 적합 |
| 혼합 콘텐츠 (텍스트+이미지+버튼) | 적합 | 부적합 |
| 무한 스크롤 | 부적합 | 적합 |
| 당겨서 새로고침 | 가능 | 내장 지원 |

> **핵심**: ScrollView는 **모든 자식을 한 번에 렌더링**합니다. 1000개의 아이템이 있으면 1000개를 모두 렌더링합니다. 이는 RecyclerView 없이 LinearLayout에 1000개의 View를 넣는 것과 같습니다.

---

## 2. FlatList — 효율적인 리스트

### Android 대응: `RecyclerView`

FlatList는 React Native에서 가장 중요한 리스트 컴포넌트입니다. Android의 `RecyclerView`와 동일한 가상화(Virtualization) 개념을 사용합니다.

### 가상화(Virtualization) 개념

```
[화면 밖 - 위]
  ┌─────────────────┐
  │   (렌더링 안 됨) │  ← 윈도우 밖: 메모리에서 제거
  │   (렌더링 안 됨) │
  └─────────────────┘
      ─ ─ ─ ─ ─ ─ ─ ─   ← windowSize 경계 (위)
  ┌─────────────────┐
  │   Item 5 (렌더됨) │  ← 윈도우 안: 렌더링됨 (보이지는 않음)
  │   Item 6 (렌더됨) │
  └─────────────────┘
╔═════════════════════╗
║   Item 7 (보임)     ║
║   Item 8 (보임)     ║  ← 화면에 보이는 영역 (Viewport)
║   Item 9 (보임)     ║
║   Item 10 (보임)    ║
╚═════════════════════╝
  ┌─────────────────┐
  │   Item 11 (렌더됨)│  ← 윈도우 안: 렌더링됨 (보이지는 않음)
  │   Item 12 (렌더됨)│
  └─────────────────┘
      ─ ─ ─ ─ ─ ─ ─ ─   ← windowSize 경계 (아래)
  ┌─────────────────┐
  │   (렌더링 안 됨) │  ← 윈도우 밖: 메모리에서 제거
  │   (렌더링 안 됨) │
  └─────────────────┘
[화면 밖 - 아래]
```

### RecyclerView와의 비교

| 개념 | RecyclerView (Android) | FlatList (React Native) |
|------|----------------------|------------------------|
| 가상화 방식 | ViewHolder 재활용 | Window 기반 렌더링/언마운트 |
| 아이템 렌더링 | `onBindViewHolder()` | `renderItem` 함수 |
| 아이템 식별 | `DiffUtil` | `keyExtractor` |
| 레이아웃 관리 | `LayoutManager` | `numColumns`, `horizontal` |
| 아이템 구분선 | `ItemDecoration` | `ItemSeparatorComponent` |
| 헤더/푸터 | Adapter에서 ViewType 분기 | `ListHeaderComponent`, `ListFooterComponent` |
| 빈 상태 | 직접 구현 | `ListEmptyComponent` |
| 당겨서 새로고침 | `SwipeRefreshLayout` 래핑 | `refreshing` + `onRefresh` props |
| 무한 스크롤 | `OnScrollListener` | `onEndReached` |
| 성능 최적화 | `setHasFixedSize`, prefetch | `getItemLayout`, `windowSize` |

### FlatList 주요 Props 전체 목록

| Prop | 타입 | 설명 |
|------|------|------|
| `data` | `T[]` | 표시할 데이터 배열 |
| `renderItem` | `({item, index, separators}) => ReactElement` | 각 아이템 렌더링 함수 |
| `keyExtractor` | `(item, index) => string` | 각 아이템의 고유 키 |
| `ItemSeparatorComponent` | `ComponentType` | 아이템 사이 구분선 |
| `ListHeaderComponent` | `ComponentType \| ReactElement` | 리스트 상단 헤더 |
| `ListFooterComponent` | `ComponentType \| ReactElement` | 리스트 하단 푸터 |
| `ListEmptyComponent` | `ComponentType \| ReactElement` | 데이터가 비었을 때 표시 |
| `onEndReached` | `() => void` | 리스트 끝에 도달 시 콜백 |
| `onEndReachedThreshold` | `number` | onEndReached 호출 임계값 (0~1) |
| `refreshing` | `boolean` | 새로고침 중 여부 |
| `onRefresh` | `() => void` | 당겨서 새로고침 콜백 |
| `horizontal` | `boolean` | 수평 리스트 |
| `numColumns` | `number` | 그리드 열 수 |
| `getItemLayout` | `(data, index) => {length, offset, index}` | 아이템 크기 미리 제공 (성능) |
| `initialNumToRender` | `number` | 초기 렌더링 아이템 수 (기본: 10) |
| `maxToRenderPerBatch` | `number` | 배치당 최대 렌더링 수 (기본: 10) |
| `windowSize` | `number` | 렌더링 윈도우 크기 (뷰포트 배수, 기본: 21) |
| `columnWrapperStyle` | `ViewStyle` | numColumns 사용 시 행 스타일 |
| `extraData` | `any` | data 외 변경 감지용 추가 데이터 |
| `inverted` | `boolean` | 리스트 뒤집기 (채팅 UI) |
| `onViewableItemsChanged` | `(info) => void` | 보이는 아이템 변경 콜백 |
| `viewabilityConfig` | `object` | 가시성 판단 기준 설정 |

### 예제 1: 기본 FlatList

```tsx
import React from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

interface User {
  id: string;
  name: string;
  email: string;
}

const users: User[] = Array.from({ length: 100 }, (_, i) => ({
  id: String(i + 1),
  name: `사용자 ${i + 1}`,
  email: `user${i + 1}@example.com`,
}));

const BasicFlatListExample = () => {
  const renderItem = ({ item }: { item: User }) => (
    <View style={styles.userItem}>
      <View style={styles.avatar}>
        <Text style={styles.avatarText}>{item.name[0]}</Text>
      </View>
      <View style={styles.userInfo}>
        <Text style={styles.userName}>{item.name}</Text>
        <Text style={styles.userEmail}>{item.email}</Text>
      </View>
    </View>
  );

  return (
    <FlatList
      data={users}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
    />
  );
};

// Android 대응:
// class UserAdapter(private val users: List<User>) :
//     RecyclerView.Adapter<UserAdapter.ViewHolder>() {
//
//     class ViewHolder(view: View) : RecyclerView.ViewHolder(view) { ... }
//
//     override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder { ... }
//     override fun onBindViewHolder(holder: ViewHolder, position: Int) { ... }
//     override fun getItemCount() = users.size
// }

const styles = StyleSheet.create({
  userItem: {
    flexDirection: 'row',
    padding: 16,
    alignItems: 'center',
  },
  avatar: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: '#E3F2FD',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  avatarText: { fontSize: 18, fontWeight: 'bold', color: '#1565C0' },
  userInfo: { flex: 1 },
  userName: { fontSize: 16, fontWeight: '600' },
  userEmail: { fontSize: 14, color: '#666', marginTop: 2 },
  separator: { height: 1, backgroundColor: '#f0f0f0', marginLeft: 76 },
});
```

```jsx [snack]
import React from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

const users = Array.from({ length: 50 }, (_, i) => ({
  id: String(i + 1),
  name: '사용자 ' + (i + 1),
  email: 'user' + (i + 1) + '@example.com',
}));

const BasicFlatListExample = () => {
  const renderItem = ({ item }) => (
    <View style={styles.userItem}>
      <View style={styles.avatar}>
        <Text style={styles.avatarText}>{item.name[0]}</Text>
      </View>
      <View style={styles.userInfo}>
        <Text style={styles.userName}>{item.name}</Text>
        <Text style={styles.userEmail}>{item.email}</Text>
      </View>
    </View>
  );

  return (
    <FlatList
      data={users}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
    />
  );
};

const styles = StyleSheet.create({
  userItem: {
    flexDirection: 'row',
    padding: 16,
    alignItems: 'center',
  },
  avatar: {
    width: 48,
    height: 48,
    borderRadius: 24,
    backgroundColor: '#E3F2FD',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  avatarText: { fontSize: 18, fontWeight: 'bold', color: '#1565C0' },
  userInfo: { flex: 1 },
  userName: { fontSize: 16, fontWeight: '600' },
  userEmail: { fontSize: 14, color: '#666', marginTop: 2 },
  separator: { height: 1, backgroundColor: '#f0f0f0', marginLeft: 76 },
});

export default BasicFlatListExample;
```

### 예제 2: 당겨서 새로고침 (Pull to Refresh)

```tsx
import React, { useState, useCallback } from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

interface Post {
  id: string;
  title: string;
  body: string;
}

const PullToRefreshExample = () => {
  const [posts, setPosts] = useState<Post[]>(
    Array.from({ length: 10 }, (_, i) => ({
      id: String(i + 1),
      title: `게시글 ${i + 1}`,
      body: `게시글 ${i + 1}의 내용입니다.`,
    }))
  );
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = useCallback(async () => {
    setRefreshing(true);

    // API 호출 시뮬레이션
    await new Promise(resolve => setTimeout(resolve, 2000));

    // 새 데이터로 교체
    const newPosts = Array.from({ length: 10 }, (_, i) => ({
      id: String(Date.now() + i),
      title: `새 게시글 ${i + 1}`,
      body: `새로고침된 게시글 ${i + 1}의 내용입니다.`,
    }));
    setPosts(newPosts);
    setRefreshing(false);
  }, []);

  return (
    <FlatList
      data={posts}
      renderItem={({ item }) => (
        <View style={styles.postItem}>
          <Text style={styles.postTitle}>{item.title}</Text>
          <Text style={styles.postBody}>{item.body}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}
      refreshing={refreshing}
      onRefresh={onRefresh}
      // Android에서 새로고침 인디케이터 색상 변경
      // (iOS는 tintColor 사용)
      progressViewOffset={0}
    />
  );
};

// Android 대응:
// <SwipeRefreshLayout
//     android:id="@+id/swipeRefresh"
//     app:layout_behavior="@string/appbar_scrolling_view_behavior">
//     <RecyclerView ... />
// </SwipeRefreshLayout>
//
// swipeRefreshLayout.setOnRefreshListener {
//     viewModel.refresh()
// }
// viewModel.isLoading.observe(this) { isLoading ->
//     swipeRefreshLayout.isRefreshing = isLoading
// }

const styles = StyleSheet.create({
  postItem: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  postTitle: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  postBody: { fontSize: 14, color: '#666' },
});
```

```jsx [snack]
import React, { useState, useCallback } from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

const PullToRefreshExample = () => {
  const [posts, setPosts] = useState(
    Array.from({ length: 10 }, (_, i) => ({
      id: String(i + 1),
      title: '게시글 ' + (i + 1),
      body: '게시글 ' + (i + 1) + '의 내용입니다.',
    }))
  );
  const [refreshing, setRefreshing] = useState(false);

  const onRefresh = useCallback(async () => {
    setRefreshing(true);
    await new Promise((resolve) => setTimeout(resolve, 2000));
    const newPosts = Array.from({ length: 10 }, (_, i) => ({
      id: String(Date.now() + i),
      title: '새 게시글 ' + (i + 1),
      body: '새로고침된 게시글 ' + (i + 1) + '의 내용입니다.',
    }));
    setPosts(newPosts);
    setRefreshing(false);
  }, []);

  return (
    <FlatList
      data={posts}
      renderItem={({ item }) => (
        <View style={styles.postItem}>
          <Text style={styles.postTitle}>{item.title}</Text>
          <Text style={styles.postBody}>{item.body}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}
      refreshing={refreshing}
      onRefresh={onRefresh}
    />
  );
};

const styles = StyleSheet.create({
  postItem: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  postTitle: { fontSize: 16, fontWeight: '600', marginBottom: 4 },
  postBody: { fontSize: 14, color: '#666' },
});

export default PullToRefreshExample;
```

### 예제 3: 무한 스크롤 (Infinite Scroll / Pagination)

```tsx
import React, { useState, useCallback } from 'react';
import { FlatList, View, Text, ActivityIndicator, StyleSheet } from 'react-native';

interface Item {
  id: string;
  title: string;
}

const InfiniteScrollExample = () => {
  const [data, setData] = useState<Item[]>(
    Array.from({ length: 20 }, (_, i) => ({
      id: String(i + 1),
      title: `항목 ${i + 1}`,
    }))
  );
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [page, setPage] = useState(1);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;

    setLoading(true);

    // API 호출 시뮬레이션
    await new Promise(resolve => setTimeout(resolve, 1500));

    const nextPage = page + 1;
    const newItems = Array.from({ length: 20 }, (_, i) => ({
      id: String(page * 20 + i + 1),
      title: `항목 ${page * 20 + i + 1}`,
    }));

    if (nextPage > 5) {
      setHasMore(false);  // 5페이지 후 더 이상 데이터 없음
    }

    setData(prev => [...prev, ...newItems]);
    setPage(nextPage);
    setLoading(false);
  }, [loading, hasMore, page]);

  const renderFooter = () => {
    if (!loading) return null;
    return (
      <View style={styles.footer}>
        <ActivityIndicator size="small" color="#2196F3" />
        <Text style={styles.footerText}>더 불러오는 중...</Text>
      </View>
    );
  };

  return (
    <FlatList
      data={data}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text style={styles.itemText}>{item.title}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}    // 끝에서 50% 지점에서 호출
      ListFooterComponent={renderFooter}
    />
  );
};

// Android 대응:
// recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
//     override fun onScrolled(rv: RecyclerView, dx: Int, dy: Int) {
//         val layoutManager = rv.layoutManager as LinearLayoutManager
//         val totalItemCount = layoutManager.itemCount
//         val lastVisibleItem = layoutManager.findLastVisibleItemPosition()
//         if (!isLoading && totalItemCount <= lastVisibleItem + THRESHOLD) {
//             loadNextPage()
//         }
//     }
// })

const styles = StyleSheet.create({
  item: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  itemText: { fontSize: 16 },
  footer: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    padding: 16,
    gap: 8,
  },
  footerText: { fontSize: 14, color: '#666' },
});
```

```jsx [snack]
import React, { useState, useCallback } from 'react';
import { FlatList, View, Text, ActivityIndicator, StyleSheet } from 'react-native';

const InfiniteScrollExample = () => {
  const [data, setData] = useState(
    Array.from({ length: 20 }, (_, i) => ({
      id: String(i + 1),
      title: '항목 ' + (i + 1),
    }))
  );
  const [loading, setLoading] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  const [page, setPage] = useState(1);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;
    setLoading(true);
    await new Promise((resolve) => setTimeout(resolve, 1500));
    const nextPage = page + 1;
    const newItems = Array.from({ length: 20 }, (_, i) => ({
      id: String(page * 20 + i + 1),
      title: '항목 ' + (page * 20 + i + 1),
    }));
    if (nextPage > 5) {
      setHasMore(false);
    }
    setData((prev) => [...prev, ...newItems]);
    setPage(nextPage);
    setLoading(false);
  }, [loading, hasMore, page]);

  const renderFooter = () => {
    if (!loading) return null;
    return (
      <View style={styles.footer}>
        <ActivityIndicator size="small" color="#2196F3" />
        <Text style={styles.footerText}>더 불러오는 중...</Text>
      </View>
    );
  };

  return (
    <FlatList
      data={data}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text style={styles.itemText}>{item.title}</Text>
        </View>
      )}
      keyExtractor={(item) => item.id}
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}
      ListFooterComponent={renderFooter}
    />
  );
};

const styles = StyleSheet.create({
  item: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  itemText: { fontSize: 16 },
  footer: {
    flexDirection: 'row',
    justifyContent: 'center',
    alignItems: 'center',
    padding: 16,
    gap: 8,
  },
  footerText: { fontSize: 14, color: '#666' },
});

export default InfiniteScrollExample;
```

### 예제 4: 수평 리스트

```tsx
import React from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

interface Category {
  id: string;
  name: string;
  color: string;
}

const categories: Category[] = [
  { id: '1', name: '전체', color: '#FF6B6B' },
  { id: '2', name: '음식', color: '#4ECDC4' },
  { id: '3', name: '여행', color: '#45B7D1' },
  { id: '4', name: '기술', color: '#96CEB4' },
  { id: '5', name: '스포츠', color: '#FFEAA7' },
  { id: '6', name: '음악', color: '#DDA0DD' },
  { id: '7', name: '영화', color: '#98D8C8' },
];

const HorizontalListExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>카테고리</Text>
      <FlatList
        data={categories}
        horizontal={true}
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={styles.listContent}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={[styles.categoryChip, { backgroundColor: item.color }]}>
            <Text style={styles.categoryText}>{item.name}</Text>
          </View>
        )}
      />
    </View>
  );
};

// Android 대응:
// RecyclerView with LinearLayoutManager(context, HORIZONTAL, false)

const styles = StyleSheet.create({
  container: { paddingVertical: 16 },
  title: { fontSize: 18, fontWeight: 'bold', paddingHorizontal: 16, marginBottom: 12 },
  listContent: { paddingHorizontal: 16, gap: 8 },
  categoryChip: {
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 20,
  },
  categoryText: { color: '#fff', fontWeight: '600' },
});
```

```jsx [snack]
import React from 'react';
import { FlatList, View, Text, StyleSheet } from 'react-native';

const categories = [
  { id: '1', name: '전체', color: '#FF6B6B' },
  { id: '2', name: '음식', color: '#4ECDC4' },
  { id: '3', name: '여행', color: '#45B7D1' },
  { id: '4', name: '기술', color: '#96CEB4' },
  { id: '5', name: '스포츠', color: '#FFEAA7' },
  { id: '6', name: '음악', color: '#DDA0DD' },
  { id: '7', name: '영화', color: '#98D8C8' },
];

const HorizontalListExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>카테고리</Text>
      <FlatList
        data={categories}
        horizontal={true}
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={styles.listContent}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => (
          <View style={[styles.categoryChip, { backgroundColor: item.color }]}>
            <Text style={styles.categoryText}>{item.name}</Text>
          </View>
        )}
      />
      <View style={styles.contentArea}>
        <Text style={styles.contentText}>
          카테고리 칩을 수평으로 스크롤할 수 있습니다
        </Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, paddingVertical: 16 },
  title: { fontSize: 18, fontWeight: 'bold', paddingHorizontal: 16, marginBottom: 12 },
  listContent: { paddingHorizontal: 16, gap: 8 },
  categoryChip: {
    paddingHorizontal: 20,
    paddingVertical: 10,
    borderRadius: 20,
  },
  categoryText: { color: '#fff', fontWeight: '600' },
  contentArea: { flex: 1, justifyContent: 'center', alignItems: 'center', padding: 32 },
  contentText: { fontSize: 14, color: '#999', textAlign: 'center' },
});

export default HorizontalListExample;
```

### 예제 5: 그리드 레이아웃 (numColumns)

```tsx
import React from 'react';
import { FlatList, View, Text, Dimensions, StyleSheet } from 'react-native';

const { width: screenWidth } = Dimensions.get('window');
const numColumns = 3;
const itemSize = (screenWidth - 32 - (numColumns - 1) * 8) / numColumns;

interface Photo {
  id: string;
  color: string;
}

const photos: Photo[] = Array.from({ length: 30 }, (_, i) => ({
  id: String(i + 1),
  color: `hsl(${(i * 37) % 360}, 70%, 60%)`,
}));

const GridExample = () => {
  return (
    <FlatList
      data={photos}
      numColumns={numColumns}
      keyExtractor={(item) => item.id}
      contentContainerStyle={styles.gridContainer}
      columnWrapperStyle={styles.row}
      renderItem={({ item }) => (
        <View style={[styles.gridItem, { backgroundColor: item.color }]}>
          <Text style={styles.gridItemText}>{item.id}</Text>
        </View>
      )}
    />
  );
};

// Android 대응:
// RecyclerView with GridLayoutManager(context, 3)

const styles = StyleSheet.create({
  gridContainer: { padding: 16 },
  row: { gap: 8, marginBottom: 8 },
  gridItem: {
    width: itemSize,
    height: itemSize,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  gridItemText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
});
```

```jsx [snack]
import React from 'react';
import { FlatList, View, Text, Dimensions, StyleSheet } from 'react-native';

const { width: screenWidth } = Dimensions.get('window');
const numColumns = 3;
const itemSize = (screenWidth - 32 - (numColumns - 1) * 8) / numColumns;

const photos = Array.from({ length: 30 }, (_, i) => ({
  id: String(i + 1),
  color: 'hsl(' + ((i * 37) % 360) + ', 70%, 60%)',
}));

const GridExample = () => {
  return (
    <FlatList
      data={photos}
      numColumns={numColumns}
      keyExtractor={(item) => item.id}
      contentContainerStyle={styles.gridContainer}
      columnWrapperStyle={styles.row}
      renderItem={({ item }) => (
        <View style={[styles.gridItem, { backgroundColor: item.color }]}>
          <Text style={styles.gridItemText}>{item.id}</Text>
        </View>
      )}
    />
  );
};

const styles = StyleSheet.create({
  gridContainer: { padding: 16 },
  row: { gap: 8, marginBottom: 8 },
  gridItem: {
    width: itemSize,
    height: itemSize,
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  gridItemText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
});

export default GridExample;
```

### 예제 6: 헤더, 푸터, 빈 상태

```tsx
import React, { useState } from 'react';
import { FlatList, View, Text, Pressable, StyleSheet } from 'react-native';

const HeaderFooterEmptyExample = () => {
  const [data, setData] = useState<string[]>([]);

  return (
    <FlatList
      data={data}
      keyExtractor={(item, index) => String(index)}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>{item}</Text>
        </View>
      )}

      // 리스트 상단 헤더
      ListHeaderComponent={
        <View style={styles.header}>
          <Text style={styles.headerTitle}>나의 할 일</Text>
          <Pressable
            style={styles.addButton}
            onPress={() => setData(prev => [...prev, `할 일 ${prev.length + 1}`])}
          >
            <Text style={styles.addButtonText}>+ 추가</Text>
          </Pressable>
        </View>
      }

      // 리스트 하단 푸터
      ListFooterComponent={
        data.length > 0 ? (
          <View style={styles.footer}>
            <Text style={styles.footerText}>총 {data.length}개의 할 일</Text>
          </View>
        ) : null
      }

      // 데이터가 비었을 때
      ListEmptyComponent={
        <View style={styles.emptyContainer}>
          <Text style={styles.emptyIcon}>📋</Text>
          <Text style={styles.emptyTitle}>할 일이 없습니다</Text>
          <Text style={styles.emptySubtitle}>
            상단의 + 추가 버튼을 눌러 할 일을 추가하세요
          </Text>
        </View>
      }
    />
  );
};

const styles = StyleSheet.create({
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  headerTitle: { fontSize: 24, fontWeight: 'bold' },
  addButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
  },
  addButtonText: { color: '#fff', fontWeight: '600' },
  item: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  footer: {
    padding: 16,
    alignItems: 'center',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
  },
  footerText: { color: '#999' },
  emptyContainer: {
    alignItems: 'center',
    paddingVertical: 60,
    paddingHorizontal: 32,
  },
  emptyIcon: { fontSize: 48, marginBottom: 16 },
  emptyTitle: { fontSize: 18, fontWeight: '600', color: '#333', marginBottom: 8 },
  emptySubtitle: { fontSize: 14, color: '#999', textAlign: 'center' },
});
```

```jsx [snack]
import React, { useState } from 'react';
import { FlatList, View, Text, Pressable, StyleSheet } from 'react-native';

const HeaderFooterEmptyExample = () => {
  const [data, setData] = useState([]);

  return (
    <FlatList
      data={data}
      keyExtractor={(item, index) => String(index)}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>{item}</Text>
        </View>
      )}
      ListHeaderComponent={
        <View style={styles.header}>
          <Text style={styles.headerTitle}>나의 할 일</Text>
          <Pressable
            style={styles.addButton}
            onPress={() => setData((prev) => [...prev, '할 일 ' + (prev.length + 1)])}
          >
            <Text style={styles.addButtonText}>+ 추가</Text>
          </Pressable>
        </View>
      }
      ListFooterComponent={
        data.length > 0 ? (
          <View style={styles.footer}>
            <Text style={styles.footerText}>총 {data.length}개의 할 일</Text>
          </View>
        ) : null
      }
      ListEmptyComponent={
        <View style={styles.emptyContainer}>
          <Text style={styles.emptyIcon}>📋</Text>
          <Text style={styles.emptyTitle}>할 일이 없습니다</Text>
          <Text style={styles.emptySubtitle}>
            상단의 + 추가 버튼을 눌러 할 일을 추가하세요
          </Text>
        </View>
      }
    />
  );
};

const styles = StyleSheet.create({
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  headerTitle: { fontSize: 24, fontWeight: 'bold' },
  addButton: {
    backgroundColor: '#2196F3',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
  },
  addButtonText: { color: '#fff', fontWeight: '600' },
  item: {
    padding: 16,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  footer: {
    padding: 16,
    alignItems: 'center',
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
  },
  footerText: { color: '#999' },
  emptyContainer: {
    alignItems: 'center',
    paddingVertical: 60,
    paddingHorizontal: 32,
  },
  emptyIcon: { fontSize: 48, marginBottom: 16 },
  emptyTitle: { fontSize: 18, fontWeight: '600', color: '#333', marginBottom: 8 },
  emptySubtitle: { fontSize: 14, color: '#999', textAlign: 'center' },
});

export default HeaderFooterEmptyExample;
```

### 예제 7: getItemLayout으로 성능 최적화

모든 아이템의 높이가 동일할 때, `getItemLayout`을 제공하면 FlatList가 스크롤 위치를 계산하기 위해 아이템을 렌더링할 필요가 없어집니다.

```tsx
const ITEM_HEIGHT = 72;  // 아이템 높이 고정
const SEPARATOR_HEIGHT = 1;

const OptimizedFlatList = () => {
  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={(item) => item.id}
      getItemLayout={(data, index) => ({
        length: ITEM_HEIGHT,
        offset: (ITEM_HEIGHT + SEPARATOR_HEIGHT) * index,
        index,
      })}
      ItemSeparatorComponent={() => (
        <View style={{ height: SEPARATOR_HEIGHT, backgroundColor: '#f0f0f0' }} />
      )}
      // 아이템 높이를 알고 있으므로 초기 스크롤 위치 점프도 가능
      initialScrollIndex={50}  // 50번째 아이템으로 바로 스크롤
    />
  );
};

// Android 대응:
// recyclerView.setHasFixedSize(true)  // 비슷한 최적화
```

```exercise
type: code-arrange
question: "FlatList의 기본 구조를 조립하세요"
tokens:
  - "<FlatList"
  - "data={items}"
  - "keyExtractor={(item) => item.id}"
  - "renderItem={({ item }) =>"
  - "<Text>{item.title}</Text>}"
  - "/>"
distractors:
  - "adapter={items}"
  - "onBindViewHolder"
  - "ViewHolder"
answer: ["<FlatList", "data={items}", "keyExtractor={(item) => item.id}", "renderItem={({ item }) =>", "<Text>{item.title}</Text>}", "/>"]
hint: "FlatList는 data, keyExtractor, renderItem 3가지 핵심 props가 필요합니다"
xp: 8
```

```exercise
type: categorize
question: "다음을 Android 리스트와 React Native 리스트로 분류하세요"
categories: ["Android", "React Native"]
items:
  - text: "RecyclerView.Adapter"
    category: "Android"
  - text: "renderItem"
    category: "React Native"
  - text: "ViewHolder"
    category: "Android"
  - text: "keyExtractor"
    category: "React Native"
  - text: "DiffUtil"
    category: "Android"
  - text: "getItemLayout"
    category: "React Native"
xp: 6
```

---

## 3. SectionList — 섹션별 리스트

### Android 대응: RecyclerView with 섹션 헤더 (ViewType 분기)

SectionList는 데이터를 섹션으로 그룹화하여 표시합니다. 연락처 앱의 가나다/ABC 그룹처럼요.

### 주요 Props

| Prop | 타입 | 설명 |
|------|------|------|
| `sections` | `{title, data}[]` | 섹션 데이터 배열 |
| `renderItem` | `({item, index, section}) => ReactElement` | 아이템 렌더링 |
| `renderSectionHeader` | `({section}) => ReactElement` | 섹션 헤더 렌더링 |
| `renderSectionFooter` | `({section}) => ReactElement` | 섹션 푸터 렌더링 |
| `stickySectionHeadersEnabled` | `boolean` | 섹션 헤더 고정 (기본 true) |
| `SectionSeparatorComponent` | `ComponentType` | 섹션 간 구분선 |

### 예제: 연락처 리스트

```tsx
import React from 'react';
import { SectionList, View, Text, StyleSheet } from 'react-native';

interface Contact {
  id: string;
  name: string;
  phone: string;
}

interface Section {
  title: string;
  data: Contact[];
}

const sections: Section[] = [
  {
    title: 'ㄱ',
    data: [
      { id: '1', name: '김민수', phone: '010-1234-5678' },
      { id: '2', name: '김영희', phone: '010-2345-6789' },
    ],
  },
  {
    title: 'ㄴ',
    data: [
      { id: '3', name: '나철수', phone: '010-3456-7890' },
    ],
  },
  {
    title: 'ㅂ',
    data: [
      { id: '4', name: '박지영', phone: '010-4567-8901' },
      { id: '5', name: '박서준', phone: '010-5678-9012' },
      { id: '6', name: '배수지', phone: '010-6789-0123' },
    ],
  },
  {
    title: 'ㅇ',
    data: [
      { id: '7', name: '이지은', phone: '010-7890-1234' },
      { id: '8', name: '이민호', phone: '010-8901-2345' },
    ],
  },
];

const ContactListExample = () => {
  return (
    <SectionList
      sections={sections}
      keyExtractor={(item) => item.id}
      stickySectionHeadersEnabled={true}

      renderSectionHeader={({ section }) => (
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>{section.title}</Text>
        </View>
      )}

      renderItem={({ item }) => (
        <View style={styles.contactItem}>
          <View style={styles.contactAvatar}>
            <Text style={styles.contactAvatarText}>{item.name[0]}</Text>
          </View>
          <View style={styles.contactInfo}>
            <Text style={styles.contactName}>{item.name}</Text>
            <Text style={styles.contactPhone}>{item.phone}</Text>
          </View>
        </View>
      )}

      ItemSeparatorComponent={() => (
        <View style={styles.separator} />
      )}

      ListEmptyComponent={
        <View style={styles.empty}>
          <Text>연락처가 없습니다</Text>
        </View>
      }
    />
  );
};

// Android 대응: RecyclerView.Adapter에서 getItemViewType()으로
// TYPE_HEADER와 TYPE_ITEM을 구분하여 처리
// 또는 StickyHeaders 라이브러리 사용

const styles = StyleSheet.create({
  sectionHeader: {
    backgroundColor: '#f0f0f0',
    paddingHorizontal: 16,
    paddingVertical: 8,
  },
  sectionTitle: {
    fontSize: 14,
    fontWeight: 'bold',
    color: '#666',
  },
  contactItem: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
  },
  contactAvatar: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: '#E3F2FD',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  contactAvatarText: { fontSize: 16, fontWeight: 'bold', color: '#1565C0' },
  contactInfo: { flex: 1 },
  contactName: { fontSize: 16, fontWeight: '500' },
  contactPhone: { fontSize: 14, color: '#666', marginTop: 2 },
  separator: { height: 1, backgroundColor: '#f0f0f0', marginLeft: 72 },
  empty: { padding: 32, alignItems: 'center' },
});
```

```jsx [snack]
import React from 'react';
import { SectionList, View, Text, StyleSheet } from 'react-native';

const sections = [
  {
    title: 'ㄱ',
    data: [
      { id: '1', name: '김민수', phone: '010-1234-5678' },
      { id: '2', name: '김영희', phone: '010-2345-6789' },
    ],
  },
  {
    title: 'ㄴ',
    data: [
      { id: '3', name: '나철수', phone: '010-3456-7890' },
    ],
  },
  {
    title: 'ㅂ',
    data: [
      { id: '4', name: '박지영', phone: '010-4567-8901' },
      { id: '5', name: '박서준', phone: '010-5678-9012' },
      { id: '6', name: '배수지', phone: '010-6789-0123' },
    ],
  },
  {
    title: 'ㅇ',
    data: [
      { id: '7', name: '이지은', phone: '010-7890-1234' },
      { id: '8', name: '이민호', phone: '010-8901-2345' },
    ],
  },
];

const ContactListExample = () => {
  return (
    <SectionList
      sections={sections}
      keyExtractor={(item) => item.id}
      stickySectionHeadersEnabled={true}
      renderSectionHeader={({ section }) => (
        <View style={styles.sectionHeader}>
          <Text style={styles.sectionTitle}>{section.title}</Text>
        </View>
      )}
      renderItem={({ item }) => (
        <View style={styles.contactItem}>
          <View style={styles.contactAvatar}>
            <Text style={styles.contactAvatarText}>{item.name[0]}</Text>
          </View>
          <View style={styles.contactInfo}>
            <Text style={styles.contactName}>{item.name}</Text>
            <Text style={styles.contactPhone}>{item.phone}</Text>
          </View>
        </View>
      )}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
    />
  );
};

const styles = StyleSheet.create({
  sectionHeader: {
    backgroundColor: '#f0f0f0',
    paddingHorizontal: 16,
    paddingVertical: 8,
  },
  sectionTitle: {
    fontSize: 14,
    fontWeight: 'bold',
    color: '#666',
  },
  contactItem: {
    flexDirection: 'row',
    alignItems: 'center',
    padding: 12,
    paddingHorizontal: 16,
    backgroundColor: '#fff',
  },
  contactAvatar: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: '#E3F2FD',
    justifyContent: 'center',
    alignItems: 'center',
    marginRight: 12,
  },
  contactAvatarText: { fontSize: 16, fontWeight: 'bold', color: '#1565C0' },
  contactInfo: { flex: 1 },
  contactName: { fontSize: 16, fontWeight: '500' },
  contactPhone: { fontSize: 14, color: '#666', marginTop: 2 },
  separator: { height: 1, backgroundColor: '#f0f0f0', marginLeft: 72 },
});

export default ContactListExample;
```

---

## 4. FlashList — 고성능 대안 (커뮤니티)

Shopify에서 개발한 `@shopify/flash-list`는 FlatList의 드롭인(drop-in) 대체품으로, 더 높은 성능을 제공합니다. RecyclerView의 ViewHolder 재활용 패턴과 더 유사한 방식을 사용합니다.

```tsx
// 설치: npm install @shopify/flash-list

import { FlashList } from "@shopify/flash-list";

const HighPerformanceList = () => {
  return (
    <FlashList
      data={data}
      renderItem={({ item }) => <ItemComponent item={item} />}
      estimatedItemSize={72}    // 예상 아이템 높이 (필수)
      keyExtractor={(item) => item.id}
    />
  );
};
```

| 특성 | FlatList | FlashList |
|------|---------|-----------|
| 가상화 방식 | 마운트/언마운트 | 재활용 (RecyclerView처럼) |
| 빈 셀 깜빡임 | 있을 수 있음 | 거의 없음 |
| 초기 렌더링 속도 | 보통 | 빠름 |
| 메모리 사용 | 높음 | 낮음 |
| API 호환성 | — | FlatList와 거의 동일 |

---

## 5. 성능 최적화 팁

```javascript [playground]
// 🧪 리스트 성능 최적화 개념 실습

// 1) keyExtractor — 올바른 key 생성 패턴
const items = [
  { id: "user-1", name: "홍길동", score: 95 },
  { id: "user-2", name: "김철수", score: 87 },
  { id: "user-3", name: "이영희", score: 92 },
];

// ✅ 고유 ID 기반 key
const keysById = items.map(item => item.id);
console.log("ID key:", keysById);

// ❌ 인덱스 기반 key — 정렬/필터 시 문제
const keysByIndex = items.map((_, i) => i);
console.log("Index key:", keysByIndex);

// 2) 대량 데이터 윈도잉 시뮬레이션
function simulateWindowedList(totalItems, windowSize, scrollPosition) {
  const startIndex = Math.max(0, Math.floor(scrollPosition / 50) - 5);
  const endIndex = Math.min(totalItems, startIndex + windowSize + 10);

  return {
    rendered: endIndex - startIndex,
    total: totalItems,
    range: `${startIndex}~${endIndex}`,
    memoryRatio: `${((endIndex - startIndex) / totalItems * 100).toFixed(1)}%`
  };
}

console.log("\n=== 윈도잉 시뮬레이션 (10,000개 아이템) ===");
[0, 2500, 5000, 7500].forEach(scroll => {
  const result = simulateWindowedList(10000, 20, scroll);
  console.log(`스크롤 ${scroll}px: ${result.rendered}개 렌더 (${result.range}), 메모리 ${result.memoryRatio}`);
});

// 3) getItemLayout 계산 (고정 높이일 때)
function getItemLayout(itemHeight, separatorHeight = 0) {
  return (_, index) => ({
    length: itemHeight,
    offset: (itemHeight + separatorHeight) * index,
    index,
  });
}

const layout = getItemLayout(60, 1);
console.log("\ngetItemLayout (높이 60px + 구분선 1px):");
[0, 5, 99].forEach(i => {
  const l = layout(null, i);
  console.log(`  index=${i}: offset=${l.offset}px`);
});
```

### 팁 1: React.memo로 불필요한 리렌더링 방지

```tsx
import React, { memo } from 'react';
import { View, Text, StyleSheet } from 'react-native';

// memo로 감싸면 props가 변경되지 않는 한 리렌더링되지 않음
const UserItem = memo(({ name, email }: { name: string; email: string }) => {
  console.log(`Rendering: ${name}`);  // props 변경 시에만 호출됨
  return (
    <View style={styles.item}>
      <Text style={styles.name}>{name}</Text>
      <Text style={styles.email}>{email}</Text>
    </View>
  );
});

// Android 대응: RecyclerView는 기본적으로 변경된 아이템만 리바인드
// DiffUtil.ItemCallback의 areContentsTheSame()과 유사

const styles = StyleSheet.create({
  item: { padding: 16 },
  name: { fontSize: 16, fontWeight: '600' },
  email: { fontSize: 14, color: '#666' },
});
```

### 팁 2: useCallback으로 renderItem 안정화

```tsx
import React, { useCallback } from 'react';
import { FlatList } from 'react-native';

const OptimizedList = ({ data }) => {
  // useCallback으로 함수 참조를 안정화
  const renderItem = useCallback(({ item }) => (
    <UserItem name={item.name} email={item.email} />
  ), []);  // 의존성 배열이 비어있으므로 함수가 재생성되지 않음

  const keyExtractor = useCallback((item) => item.id, []);

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
    />
  );
};
```

### 팁 3: 이미지 크기 미리 지정

```tsx
// ❌ 나쁜 예: 이미지 크기 미지정
<Image source={{ uri: imageUrl }} style={{ width: '100%' }} />

// ✅ 좋은 예: 고정 크기 지정
<Image source={{ uri: imageUrl }} style={{ width: 100, height: 100 }} />
```

### 팁 4: windowSize 조절

```tsx
<FlatList
  data={data}
  renderItem={renderItem}
  windowSize={5}        // 기본 21에서 줄임 → 메모리 절약, 스크롤 빈 공간 가능성 증가
  maxToRenderPerBatch={5}  // 배치당 렌더링 수 줄임
  initialNumToRender={10}  // 초기 렌더링 수
  removeClippedSubviews={true}  // 화면 밖 뷰 네이티브에서 제거 (Android)
/>
```

### 팁 5: extraData를 활용한 리렌더링 트리거

```tsx
const [selectedId, setSelectedId] = useState<string | null>(null);

<FlatList
  data={data}
  renderItem={({ item }) => (
    <Pressable
      style={[
        styles.item,
        item.id === selectedId && styles.selectedItem,
      ]}
      onPress={() => setSelectedId(item.id)}
    >
      <Text>{item.name}</Text>
    </Pressable>
  )}
  // data가 변경되지 않아도 selectedId가 변경되면 리렌더링
  extraData={selectedId}
/>
```

> **주의**: `extraData`를 사용하지 않으면 `data` 배열이 변경되지 않는 한 FlatList가 리렌더링되지 않습니다. `selectedId` 같은 외부 상태에 의존하는 경우 반드시 `extraData`에 전달해야 합니다.

### 팁 6: 무거운 아이템은 지연 로드

```tsx
import React, { useState, useEffect } from 'react';
import { View, InteractionManager } from 'react-native';

const HeavyItem = ({ item }) => {
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // 애니메이션이 끝난 후에 무거운 렌더링 수행
    const task = InteractionManager.runAfterInteractions(() => {
      setReady(true);
    });
    return () => task.cancel();
  }, []);

  if (!ready) {
    return <View style={{ height: 200, backgroundColor: '#f0f0f0' }} />;
  }

  return <ExpensiveComponent data={item} />;
};
```

---

## 6. 종합 비교 표: RecyclerView vs FlatList

| 항목 | RecyclerView | FlatList |
|------|-------------|---------|
| **설정 복잡도** | Adapter + ViewHolder + LayoutManager | `data` + `renderItem` + `keyExtractor` |
| **코드량** | ~100줄 (Adapter 클래스) | ~20줄 |
| **ViewType 분기** | `getItemViewType()` + ViewHolder별 분기 | 렌더링 함수 내 조건부 렌더링 |
| **DiffUtil** | `DiffUtil.ItemCallback` 구현 필요 | `keyExtractor`로 자동 처리 |
| **헤더/푸터** | ViewType 분기 또는 ConcatAdapter | `ListHeaderComponent` / `ListFooterComponent` |
| **구분선** | `ItemDecoration` | `ItemSeparatorComponent` |
| **빈 상태** | Adapter에서 직접 구현 | `ListEmptyComponent` |
| **당겨서 새로고침** | SwipeRefreshLayout 래핑 | `refreshing` + `onRefresh` props |
| **무한 스크롤** | OnScrollListener + 수동 계산 | `onEndReached` + `onEndReachedThreshold` |
| **그리드** | `GridLayoutManager` | `numColumns` |
| **수평 리스트** | `LinearLayoutManager(HORIZONTAL)` | `horizontal={true}` |
| **고정 헤더** | 별도 라이브러리 필요 | SectionList의 `stickySectionHeadersEnabled` |
| **스크롤 위치** | `scrollToPosition()` | `scrollToIndex()`, `scrollToOffset()` |

---

> 다음: [04-styling.md](./04-styling.md) — 스타일링 완전 가이드
