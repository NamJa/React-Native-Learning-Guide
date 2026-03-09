# TanStack Query 완전 가이드 — Retrofit/Repository 개발자를 위한 서버 상태 관리

## 목차
1. [왜 TanStack Query인가?](#1-왜-tanstack-query인가)
2. [설치와 초기 설정](#2-설치와-초기-설정)
3. [useQuery 상세](#3-usequery-상세)
4. [useMutation 상세](#4-usemutation-상세)
5. [useInfiniteQuery: 무한 스크롤](#5-useinfinitequery-무한-스크롤)
6. [쿼리 무효화 전략](#6-쿼리-무효화-전략)
7. [프리페칭](#7-프리페칭)
8. [병렬 쿼리와 의존 쿼리](#8-병렬-쿼리와-의존-쿼리)
9. [에러 처리 패턴](#9-에러-처리-패턴)
10. [Android 패턴과 비교](#10-android-패턴과-비교)
11. [전체 CRUD 예제](#11-전체-crud-예제)

---

## 1. 왜 TanStack Query인가?

Android 개발에서 Retrofit으로 API를 호출하면, 다음 문제들을 직접 해결해야 한다:

1. **캐싱**: 같은 데이터를 여러 화면에서 보여줄 때, 매번 API를 호출하는가?
2. **중복 요청 방지**: 같은 요청이 동시에 여러 번 발생하면?
3. **백그라운드 갱신**: 앱이 포그라운드로 돌아왔을 때 데이터가 오래된 건 아닌가?
4. **로딩/에러 상태**: 매번 sealed class UiState를 만들어야 하는가?
5. **페이지네이션**: Paging3 라이브러리를 설정하는 것이 얼마나 복잡한가?
6. **낙관적 업데이트**: 서버 응답 전에 UI를 먼저 업데이트하려면?
7. **오래된 데이터 처리**: stale-while-revalidate 패턴을 직접 구현하려면?

Android에서는 이를 위해 Repository 패턴 + Room 캐시 + Flow + ViewModel을 조합해야 한다. TanStack Query는 이 모든 것을 **자동으로** 해결한다.

```
Android에서 직접 구현해야 하는 것:          TanStack Query의 자동 제공:
─────────────────────────────────        ─────────────────────────────
Repository 패턴                          내장 (queryFn만 정의)
Room 캐시                                자동 메모리 캐시
StateFlow + sealed UiState               isLoading/error/data 자동 관리
Coroutine viewModelScope                 자동 생명주기 관리
중복 요청 방지 (Flow conflate)            자동 deduplication
Paging3 라이브러리                        useInfiniteQuery
수동 캐시 무효화                          invalidateQueries
stale-while-revalidate                   staleTime 설정 한 줄
```

---

## 2. 설치와 초기 설정

### 2-1. 설치

```bash
npm install @tanstack/react-query
```

### 2-2. QueryClientProvider 설정

Android에서 Hilt의 `@HiltAndroidApp`을 Application 클래스에 설정하는 것처럼, TanStack Query도 앱의 최상위에 Provider를 설정해야 한다.

```tsx
// App.tsx
import React from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// QueryClient 생성 (Android의 OkHttpClient 설정과 유사)
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // 데이터를 "신선한" 것으로 간주하는 시간 (밀리초)
      staleTime: 1000 * 60 * 5,        // 5분 (기본값: 0 = 즉시 stale)

      // 가비지 컬렉션 시간 (비활성 쿼리가 캐시에서 제거되기까지)
      gcTime: 1000 * 60 * 30,          // 30분 (기본값: 5분)

      // 실패 시 재시도 횟수
      retry: 3,                         // 기본값: 3

      // 재시도 간격 (지수 백오프)
      retryDelay: (attemptIndex) =>
        Math.min(1000 * 2 ** attemptIndex, 30000),

      // 앱이 포그라운드로 돌아올 때 자동 갱신
      refetchOnWindowFocus: true,       // 웹에서는 탭 전환, RN에서는 앱 포그라운드

      // 네트워크 재연결 시 자동 갱신
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* 네비게이션, 나머지 앱 구조 */}
      <NavigationContainer>
        <RootNavigator />
      </NavigationContainer>
    </QueryClientProvider>
  );
}

export default App;
```

### 2-3. API 클라이언트 설정

```tsx
// api/client.ts
import axios from 'axios';
import { useAuthStore } from '../stores/authStore';

// Android의 Retrofit + OkHttpClient 설정과 동일
const apiClient = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request Interceptor (OkHttp Interceptor와 동일)
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response Interceptor
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      // 토큰 리프레시 시도
      const refreshed = await useAuthStore.getState().refreshAuth();
      if (refreshed) {
        // 원래 요청 재시도
        error.config.headers.Authorization =
          `Bearer ${useAuthStore.getState().token}`;
        return apiClient(error.config);
      }
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### 2-4. API 함수 정의

```tsx
// api/userApi.ts
// Android의 Retrofit @GET @POST 등과 동일한 역할

import apiClient from './client';

export interface User {
  id: number;
  name: string;
  email: string;
  avatar?: string;
  createdAt: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
  password: string;
}

export interface UpdateUserDto {
  name?: string;
  email?: string;
  avatar?: string;
}

// Retrofit @GET("users")
export const getUsers = async (): Promise<User[]> => {
  const { data } = await apiClient.get('/users');
  return data;
};

// Retrofit @GET("users/{id}")
export const getUser = async (id: number): Promise<User> => {
  const { data } = await apiClient.get(`/users/${id}`);
  return data;
};

// Retrofit @POST("users")
export const createUser = async (dto: CreateUserDto): Promise<User> => {
  const { data } = await apiClient.post('/users', dto);
  return data;
};

// Retrofit @PUT("users/{id}")
export const updateUser = async (id: number, dto: UpdateUserDto): Promise<User> => {
  const { data } = await apiClient.put(`/users/${id}`, dto);
  return data;
};

// Retrofit @DELETE("users/{id}")
export const deleteUser = async (id: number): Promise<void> => {
  await apiClient.delete(`/users/${id}`);
};

// 페이지네이션이 있는 API
export interface PaginatedResponse<T> {
  data: T[];
  nextPage: number | null;
  totalPages: number;
  totalCount: number;
}

export const getUsersPaginated = async (page: number): Promise<PaginatedResponse<User>> => {
  const { data } = await apiClient.get(`/users?page=${page}&limit=20`);
  return data;
};
```

---

## 3. useQuery 상세

`useQuery`는 Retrofit의 `@GET` 메서드 + Repository의 캐싱 로직 + ViewModel의 상태 관리를 한 번에 대체한다.

### 3-1. 기본 사용법

```tsx
import { useQuery } from '@tanstack/react-query';
import { getUsers, User } from '../api/userApi';

function UserListScreen() {
  // Android: viewModel.users.collectAsState()
  const {
    data,          // User[] | undefined — 성공 시 데이터
    isLoading,     // boolean — 최초 로딩 중 (데이터 없음)
    isFetching,    // boolean — 어떤 형태든 요청 중 (백그라운드 포함)
    error,         // Error | null — 에러 발생 시
    isError,       // boolean — 에러 상태인지
    isSuccess,     // boolean — 성공 상태인지
    refetch,       // () => void — 수동 갱신
    isRefetching,  // boolean — 수동 갱신 중
  } = useQuery({
    queryKey: ['users'],           // 캐시 키 (Room의 테이블명과 유사)
    queryFn: getUsers,             // API 호출 함수 (Retrofit 인터페이스 메서드)
  });

  if (isLoading) {
    return <ActivityIndicator size="large" />;
  }

  if (isError) {
    return (
      <View>
        <Text>에러: {error.message}</Text>
        <Button title="재시도" onPress={() => refetch()} />
      </View>
    );
  }

  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id.toString()}
      renderItem={({ item }) => (
        <Text>{item.name} - {item.email}</Text>
      )}
      // 풀-투-리프레시
      refreshing={isRefetching}
      onRefresh={refetch}
    />
  );
}
```

### 3-2. queryKey 설계

queryKey는 캐시의 고유 식별자다. 계층적으로 설계하면 캐시 무효화를 효율적으로 할 수 있다.

```tsx
// 기본 키
useQuery({ queryKey: ['users'], queryFn: getUsers });

// 파라미터가 있는 키 — Retrofit의 @Path/@Query와 매핑
useQuery({
  queryKey: ['users', userId],     // @GET("users/{userId}")
  queryFn: () => getUser(userId),
});

// 필터/정렬 조건이 있는 키
useQuery({
  queryKey: ['users', { role: 'admin', sort: 'name' }],
  queryFn: () => getUsers({ role: 'admin', sort: 'name' }),
});

// 계층적 키 구조
// ['users']                          — 모든 사용자 목록
// ['users', 42]                      — ID 42 사용자 상세
// ['users', 42, 'posts']             — ID 42 사용자의 게시물
// ['users', { role: 'admin' }]       — 관리자 목록

// 키 계층을 활용한 무효화
queryClient.invalidateQueries({ queryKey: ['users'] });
// 위 호출 하나로 ['users'], ['users', 42], ['users', 42, 'posts'] 등
// 'users'로 시작하는 모든 쿼리가 무효화됨
```

### 3-3. 주요 옵션

```tsx
useQuery({
  queryKey: ['users'],
  queryFn: getUsers,

  // === 캐싱 관련 ===

  // staleTime: 데이터를 "신선한" 것으로 간주하는 시간
  // 0(기본값): 항상 stale → 컴포넌트 마운트 시 매번 백그라운드 refetch
  // Infinity: 절대 stale 안 됨 → 수동으로만 갱신
  staleTime: 5 * 60 * 1000,  // 5분간 신선한 데이터

  // gcTime: 비활성 쿼리가 캐시에서 제거되기까지의 시간
  gcTime: 30 * 60 * 1000,    // 30분 (기본값: 5분)

  // === 자동 갱신 관련 ===

  // 앱이 포그라운드로 돌아올 때 갱신
  refetchOnWindowFocus: true, // Android AppLifecycleObserver의 ON_RESUME과 유사

  // 주기적 자동 갱신 (polling)
  refetchInterval: 30000,     // 30초마다 갱신 (실시간 데이터에 유용)
  refetchIntervalInBackground: false, // 백그라운드에서는 폴링 안 함

  // 네트워크 재연결 시 갱신
  refetchOnReconnect: true,

  // === 조건부 실행 ===

  // enabled: false면 쿼리가 실행되지 않음
  // Android의 Flow가 collector가 있을 때만 emit하는 것과 유사
  enabled: !!userId,          // userId가 있을 때만 실행

  // === 데이터 변환 ===

  // select: 서버 데이터를 변환하여 컴포넌트에 전달
  // Android의 Flow.map과 동일
  select: (data) => data.filter((user) => user.isActive),

  // === 초기/플레이스홀더 데이터 ===

  // placeholderData: 로딩 중에 임시로 보여줄 데이터
  // 이전에 가져온 데이터를 보여주면서 새 데이터 로딩
  placeholderData: (previousData) => previousData,

  // initialData: 캐시에 초기 데이터 설정
  // 네비게이션 파라미터로 받은 데이터를 초기값으로 사용
  initialData: () => {
    const cachedUsers = queryClient.getQueryData(['users']);
    return cachedUsers?.find((u) => u.id === userId);
  },
});
```

### 3-4. 커스텀 훅으로 래핑 (권장 패턴)

```tsx
// hooks/useUsers.ts
// Android의 ViewModel 메서드를 커스텀 훅으로 분리하는 패턴

import { useQuery, UseQueryOptions } from '@tanstack/react-query';
import { getUsers, getUser, User } from '../api/userApi';

// 사용자 목록 쿼리
export function useUsers(
  options?: Omit<UseQueryOptions<User[]>, 'queryKey' | 'queryFn'>
) {
  return useQuery({
    queryKey: ['users'],
    queryFn: getUsers,
    staleTime: 5 * 60 * 1000,
    ...options,
  });
}

// 사용자 상세 쿼리
export function useUser(userId: number) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: () => getUser(userId),
    enabled: userId > 0,
    staleTime: 5 * 60 * 1000,
  });
}

// 활성 사용자만 가져오기 (select 활용)
export function useActiveUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: getUsers,
    select: (data) => data.filter((user) => user.isActive),
  });
}
```

사용:
```tsx
function UserListScreen() {
  const { data: users, isLoading, refetch } = useUsers();
  // 깔끔!
}
```

```exercise
type: word-bank
question: "useQuery 호출의 빈칸을 채우세요"
code: |
  const { data, ___, error } = useQuery({
    queryKey: ['users'],
    queryFn: ___,
  });
blanks: ["isLoading", "fetchUsers"]
distractors: ["isPending", "useState", "getUsers", "async"]
hint: "useQuery는 isLoading으로 로딩 상태를, queryFn으로 데이터 fetching 함수를 제공합니다"
xp: 5
```

---

## 4. useMutation 상세

`useMutation`은 Retrofit의 `@POST`, `@PUT`, `@DELETE`에 해당한다. 서버 데이터를 변경하는 모든 요청에 사용한다.

### 4-1. 기본 사용법

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { createUser, CreateUserDto } from '../api/userApi';

function CreateUserScreen() {
  const queryClient = useQueryClient();
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  const createMutation = useMutation({
    // mutationFn: API 호출 함수 (Retrofit @POST)
    mutationFn: (data: CreateUserDto) => createUser(data),

    // 성공 콜백
    onSuccess: (newUser) => {
      console.log('생성 성공:', newUser);
      // 사용자 목록 캐시 무효화 → 자동으로 다시 가져옴
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },

    // 에러 콜백
    onError: (error) => {
      console.error('생성 실패:', error);
      Alert.alert('오류', '사용자 생성에 실패했습니다');
    },

    // 완료 콜백 (성공/실패 무관)
    onSettled: () => {
      console.log('요청 완료');
    },
  });

  const handleSubmit = () => {
    createMutation.mutate({
      name,
      email,
      password: 'defaultPassword123',
    });
  };

  return (
    <View style={{ padding: 20 }}>
      <TextInput
        placeholder="이름"
        value={name}
        onChangeText={setName}
        style={styles.input}
      />
      <TextInput
        placeholder="이메일"
        value={email}
        onChangeText={setEmail}
        style={styles.input}
      />
      <TouchableOpacity
        style={styles.button}
        onPress={handleSubmit}
        disabled={createMutation.isPending}
      >
        <Text style={styles.buttonText}>
          {createMutation.isPending ? '생성 중...' : '사용자 생성'}
        </Text>
      </TouchableOpacity>

      {createMutation.isError && (
        <Text style={styles.error}>
          에러: {createMutation.error.message}
        </Text>
      )}
    </View>
  );
}
```

### 4-2. 낙관적 업데이트 (Optimistic Updates)

서버 응답을 기다리지 않고 UI를 먼저 업데이트하는 패턴. 좋아요/즐겨찾기 등 사용자 경험이 중요한 곳에 사용한다.

```tsx
function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (postId: number) =>
      apiClient.post(`/posts/${postId}/favorite`),

    // 1단계: mutation 시작 전 — 현재 캐시를 저장하고, 낙관적으로 업데이트
    onMutate: async (postId: number) => {
      // 진행 중인 refetch 취소 (낙관적 업데이트를 덮어쓰지 않도록)
      await queryClient.cancelQueries({ queryKey: ['posts'] });

      // 현재 캐시 스냅샷 저장 (롤백용)
      const previousPosts = queryClient.getQueryData(['posts']);

      // 낙관적으로 캐시 업데이트 (UI 즉시 반영)
      queryClient.setQueryData<Post[]>(['posts'], (old) =>
        old?.map((post) =>
          post.id === postId
            ? { ...post, isFavorited: !post.isFavorited }
            : post
        )
      );

      // context로 이전 상태 반환 (롤백용)
      return { previousPosts };
    },

    // 2단계: 에러 발생 시 — 롤백
    onError: (err, postId, context) => {
      // 저장해둔 이전 상태로 복원
      if (context?.previousPosts) {
        queryClient.setQueryData(['posts'], context.previousPosts);
      }
      Alert.alert('오류', '즐겨찾기 변경에 실패했습니다');
    },

    // 3단계: 완료 후 (성공/실패 무관) — 서버 데이터로 동기화
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });
}

// 사용
function PostItem({ post }: { post: Post }) {
  const toggleFavorite = useToggleFavorite();

  return (
    <TouchableOpacity onPress={() => toggleFavorite.mutate(post.id)}>
      <Text>{post.isFavorited ? '★' : '☆'}</Text>
    </TouchableOpacity>
  );
}
```

### 4-3. mutateAsync vs mutate

```tsx
const createMutation = useMutation({
  mutationFn: createUser,
});

// mutate: 콜백 기반 (fire-and-forget)
createMutation.mutate(data, {
  onSuccess: (result) => console.log('성공:', result),
  onError: (error) => console.log('실패:', error),
});

// mutateAsync: Promise 반환 (async/await 사용 가능)
try {
  const result = await createMutation.mutateAsync(data);
  console.log('성공:', result);
  navigation.goBack();
} catch (error) {
  console.log('실패:', error);
}
```

---

## 5. useInfiniteQuery: 무한 스크롤

Android의 `Paging3` 라이브러리에 해당한다. Paging3보다 훨씬 간단하게 무한 스크롤을 구현할 수 있다.

### 5-1. 기본 사용법

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

function useUserInfiniteList() {
  return useInfiniteQuery({
    queryKey: ['users', 'infinite'],

    queryFn: ({ pageParam }) => getUsersPaginated(pageParam),

    // 초기 페이지
    initialPageParam: 1,

    // 다음 페이지 번호 결정
    // Android Paging3의 PagingSource.LoadResult.Page의 nextKey와 동일
    getNextPageParam: (lastPage, allPages) => {
      // lastPage: 마지막으로 가져온 페이지의 응답
      // null을 반환하면 더 이상 페이지 없음
      return lastPage.nextPage ?? undefined;
    },

    // 이전 페이지 (양방향 스크롤 시)
    getPreviousPageParam: (firstPage) => {
      return firstPage.prevPage ?? undefined;
    },
  });
}
```

### 5-2. FlatList에 연결

```tsx
function UserInfiniteListScreen() {
  const {
    data,
    isLoading,
    isFetchingNextPage,  // 다음 페이지 로딩 중
    hasNextPage,          // 다음 페이지 존재 여부
    fetchNextPage,        // 다음 페이지 가져오기
    refetch,
    isRefetching,
  } = useUserInfiniteList();

  // 모든 페이지의 데이터를 하나의 배열로 병합
  // Android Paging3에서 PagingData가 자동으로 해주는 것을 수동으로 처리
  const allUsers = data?.pages.flatMap((page) => page.data) ?? [];

  const loadMore = () => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  };

  if (isLoading) {
    return <ActivityIndicator size="large" />;
  }

  return (
    <FlatList
      data={allUsers}
      keyExtractor={(item) => item.id.toString()}
      renderItem={({ item }) => (
        <View style={styles.userCard}>
          <Text style={styles.userName}>{item.name}</Text>
          <Text style={styles.userEmail}>{item.email}</Text>
        </View>
      )}

      // 무한 스크롤: 끝에 도달하면 다음 페이지 로드
      onEndReached={loadMore}
      onEndReachedThreshold={0.5}  // 50% 지점에서 미리 로드

      // 하단 로딩 인디케이터
      ListFooterComponent={
        isFetchingNextPage ? (
          <View style={styles.footer}>
            <ActivityIndicator size="small" />
            <Text>더 불러오는 중...</Text>
          </View>
        ) : !hasNextPage ? (
          <View style={styles.footer}>
            <Text style={styles.endText}>모든 데이터를 불러왔습니다</Text>
          </View>
        ) : null
      }

      // 풀-투-리프레시
      refreshing={isRefetching}
      onRefresh={refetch}
    />
  );
}
```

### 5-3. Android Paging3 비교

```kotlin
// Android Paging3: 설정이 매우 복잡하다
// 1. PagingSource 구현
class UserPagingSource(private val api: UserApi) : PagingSource<Int, User>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, User> {
        val page = params.key ?: 1
        return try {
            val response = api.getUsers(page, params.loadSize)
            LoadResult.Page(
                data = response.data,
                prevKey = if (page == 1) null else page - 1,
                nextKey = response.nextPage,
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
    override fun getRefreshKey(state: PagingState<Int, User>): Int? =
        state.anchorPosition?.let { state.closestPageToPosition(it)?.prevKey?.plus(1) }
}

// 2. ViewModel에서 Pager 설정
class UserViewModel : ViewModel() {
    val users: Flow<PagingData<User>> = Pager(
        config = PagingConfig(pageSize = 20, prefetchDistance = 5),
        pagingSourceFactory = { UserPagingSource(api) }
    ).flow.cachedIn(viewModelScope)
}

// 3. Compose에서 collectAsLazyPagingItems
@Composable
fun UserList(viewModel: UserViewModel) {
    val users = viewModel.users.collectAsLazyPagingItems()
    LazyColumn {
        items(users.itemCount) { index ->
            users[index]?.let { UserCard(it) }
        }
        // 로딩 상태 처리도 수동으로...
    }
}
```

```tsx
// TanStack Query: 동일 기능이 훨씬 간단
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ['users'],
  queryFn: ({ pageParam }) => api.getUsers(pageParam),
  initialPageParam: 1,
  getNextPageParam: (lastPage) => lastPage.nextPage,
});
// 끝. PagingSource, Pager, PagingConfig, cachedIn 모두 불필요.
```

---

## 6. 쿼리 무효화 전략

### 6-1. 기본 무효화

```tsx
const queryClient = useQueryClient();

// 특정 쿼리 무효화
queryClient.invalidateQueries({ queryKey: ['users'] });

// 계층적 무효화 (users로 시작하는 모든 쿼리)
queryClient.invalidateQueries({ queryKey: ['users'] });
// → ['users'], ['users', 1], ['users', 1, 'posts'] 모두 무효화

// 정확한 키만 무효화
queryClient.invalidateQueries({ queryKey: ['users'], exact: true });
// → ['users']만 무효화, ['users', 1]은 그대로

// 모든 쿼리 무효화
queryClient.invalidateQueries();
```

### 6-2. Mutation 후 무효화 패턴

```tsx
// 패턴 1: 관련 쿼리 무효화 (가장 일반적)
const createUser = useMutation({
  mutationFn: createUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] });
  },
});

// 패턴 2: 캐시 직접 업데이트 (네트워크 절약)
const updateUser = useMutation({
  mutationFn: ({ id, data }) => updateUserApi(id, data),
  onSuccess: (updatedUser) => {
    // 목록 캐시에서 해당 사용자 업데이트
    queryClient.setQueryData<User[]>(['users'], (old) =>
      old?.map((user) =>
        user.id === updatedUser.id ? updatedUser : user
      )
    );
    // 상세 캐시도 업데이트
    queryClient.setQueryData(['users', updatedUser.id], updatedUser);
  },
});

// 패턴 3: 삭제 후 캐시에서도 제거
const deleteUser = useMutation({
  mutationFn: (id: number) => deleteUserApi(id),
  onSuccess: (_, deletedId) => {
    queryClient.setQueryData<User[]>(['users'], (old) =>
      old?.filter((user) => user.id !== deletedId)
    );
    queryClient.removeQueries({ queryKey: ['users', deletedId] });
  },
});
```

---

## 7. 프리페칭

### 7-1. 화면 진입 전 데이터 미리 가져오기

```tsx
// 목록에서 아이템 터치 시, 상세 데이터를 미리 가져오기
function UserListItem({ user }: { user: User }) {
  const queryClient = useQueryClient();
  const navigation = useNavigation();

  const handlePress = () => {
    navigation.navigate('UserDetail', { userId: user.id });
  };

  // 아이템 위에 손가락을 올려놓았을 때(pressIn) 프리페칭
  const handlePressIn = () => {
    queryClient.prefetchQuery({
      queryKey: ['users', user.id],
      queryFn: () => getUser(user.id),
      staleTime: 5 * 60 * 1000,
    });
  };

  return (
    <TouchableOpacity
      onPress={handlePress}
      onPressIn={handlePressIn}  // 터치 시작 시 프리페칭
    >
      <Text>{user.name}</Text>
    </TouchableOpacity>
  );
}
```

### 7-2. 화면 마운트 시 다음 화면 데이터 프리페칭

```tsx
function HomeScreen() {
  const queryClient = useQueryClient();

  useEffect(() => {
    // 홈 화면 진입 시 즐겨찾기, 최근 알림 미리 가져오기
    queryClient.prefetchQuery({
      queryKey: ['favorites'],
      queryFn: getFavorites,
    });
    queryClient.prefetchQuery({
      queryKey: ['notifications'],
      queryFn: getNotifications,
    });
  }, []);

  return <View>{/* ... */}</View>;
}
```

---

## 8. 병렬 쿼리와 의존 쿼리

### 8-1. 병렬 쿼리 (독립적인 여러 데이터)

```tsx
// 방법 1: 여러 useQuery를 나란히 호출 (자동 병렬 실행)
function DashboardScreen() {
  const usersQuery = useQuery({ queryKey: ['users'], queryFn: getUsers });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: getPosts });
  const statsQuery = useQuery({ queryKey: ['stats'], queryFn: getStats });

  const isLoading = usersQuery.isLoading || postsQuery.isLoading || statsQuery.isLoading;

  if (isLoading) return <ActivityIndicator />;

  return (
    <View>
      <Text>사용자: {usersQuery.data?.length}</Text>
      <Text>게시물: {postsQuery.data?.length}</Text>
      <Text>통계: {JSON.stringify(statsQuery.data)}</Text>
    </View>
  );
}

// 방법 2: useQueries (동적 개수의 쿼리)
import { useQueries } from '@tanstack/react-query';

function UserDetailsScreen({ userIds }: { userIds: number[] }) {
  const userQueries = useQueries({
    queries: userIds.map((id) => ({
      queryKey: ['users', id],
      queryFn: () => getUser(id),
    })),
  });

  const isLoading = userQueries.some((q) => q.isLoading);
  const allUsers = userQueries
    .filter((q) => q.isSuccess)
    .map((q) => q.data!);

  return (
    <FlatList
      data={allUsers}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}
```

### 8-2. 의존 쿼리 (순차적 실행)

```tsx
// 사용자 정보를 먼저 가져온 후, 해당 사용자의 게시물 가져오기
// Android: viewModelScope.launch 안에서 순차 suspend 호출
function UserPostsScreen({ userId }: { userId: number }) {
  // 1단계: 사용자 정보 가져오기
  const userQuery = useQuery({
    queryKey: ['users', userId],
    queryFn: () => getUser(userId),
  });

  // 2단계: 사용자의 게시물 가져오기 (사용자 정보가 있을 때만)
  const postsQuery = useQuery({
    queryKey: ['users', userId, 'posts'],
    queryFn: () => getUserPosts(userId),
    enabled: !!userQuery.data,  // userQuery가 성공해야 실행
  });

  // 3단계: 게시물의 댓글 수 가져오기
  const commentsCountQuery = useQuery({
    queryKey: ['users', userId, 'posts', 'commentCounts'],
    queryFn: () => getPostsCommentCounts(postsQuery.data!.map((p) => p.id)),
    enabled: !!postsQuery.data && postsQuery.data.length > 0,
  });

  if (userQuery.isLoading) return <ActivityIndicator />;
  if (userQuery.isError) return <Text>에러: {userQuery.error.message}</Text>;

  return (
    <View>
      <Text style={styles.header}>{userQuery.data.name}의 게시물</Text>
      {postsQuery.isLoading ? (
        <ActivityIndicator />
      ) : (
        <FlatList
          data={postsQuery.data}
          renderItem={({ item }) => <PostCard post={item} />}
        />
      )}
    </View>
  );
}
```

---

## 9. 에러 처리 패턴

### 9-1. 쿼리별 에러 처리

```tsx
function UserScreen() {
  const { data, error, isError, refetch } = useQuery({
    queryKey: ['users'],
    queryFn: getUsers,
    retry: 3,              // 3번 재시도 후 실패
    retryDelay: 1000,      // 1초 간격
  });

  if (isError) {
    return (
      <View style={styles.errorContainer}>
        <Text style={styles.errorTitle}>데이터를 불러올 수 없습니다</Text>
        <Text style={styles.errorMessage}>{error.message}</Text>
        <Button title="재시도" onPress={() => refetch()} />
      </View>
    );
  }
}
```

### 9-2. 전역 에러 처리

```tsx
// QueryClient에서 전역 에러 핸들러 설정
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error: any) => {
        // 401, 403은 재시도하지 않음
        if (error?.response?.status === 401 || error?.response?.status === 403) {
          return false;
        }
        return failureCount < 3;
      },
    },
    mutations: {
      onError: (error: any) => {
        // 모든 mutation 에러에 대한 전역 토스트
        const message = error?.response?.data?.message || '요청에 실패했습니다';
        showToast(message);
      },
    },
  },
});
```

### 9-3. Error Boundary와 Suspense (선택)

```tsx
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
  return (
    <View style={styles.errorContainer}>
      <Text>문제가 발생했습니다</Text>
      <Text>{error.message}</Text>
      <Button title="재시도" onPress={resetErrorBoundary} />
    </View>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <React.Suspense fallback={<ActivityIndicator />}>
        <UserScreen />
      </React.Suspense>
    </ErrorBoundary>
  );
}

// Suspense 모드에서는 useSuspenseQuery 사용
function UserScreen() {
  // isLoading 체크 불필요 — Suspense가 처리
  // 에러도 ErrorBoundary가 처리
  const { data } = useSuspenseQuery({
    queryKey: ['users'],
    queryFn: getUsers,
  });

  // data는 항상 존재 (undefined 아님)
  return <FlatList data={data} /* ... */ />;
}
```

---

## 10. Android 패턴과 비교

### 10-1. Retrofit @GET → useQuery

```kotlin
// Android
interface UserApi {
    @GET("users")
    suspend fun getUsers(): List<User>
}

class UserRepository(private val api: UserApi) {
    fun getUsers() = flow {
        emit(api.getUsers())
    }
}

@HiltViewModel
class UserViewModel @Inject constructor(
    private val repo: UserRepository
) : ViewModel() {
    val users = repo.getUsers()
        .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
}
```

```tsx
// React Native — 위 전체를 이것 하나로 대체
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: () => apiClient.get('/users').then(r => r.data),
  });
}
```

### 10-2. Retrofit @POST → useMutation

```kotlin
// Android
interface UserApi {
    @POST("users")
    suspend fun createUser(@Body dto: CreateUserDto): User
}
```

```tsx
// React Native
function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (dto: CreateUserDto) =>
      apiClient.post('/users', dto).then(r => r.data),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['users'] }),
  });
}
```

### 10-3. Repository 캐싱 → 내장 쿼리 캐시

```kotlin
// Android: 수동 캐싱
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    fun getUsers() = flow {
        val cached = dao.getAll()
        if (cached.isNotEmpty()) emit(cached)
        try {
            val fresh = api.getUsers()
            dao.insertAll(fresh)
            emit(fresh)
        } catch (e: Exception) { /* ... */ }
    }
}
```

```tsx
// React Native: staleTime 설정으로 자동 캐싱
useQuery({
  queryKey: ['users'],
  queryFn: getUsers,
  staleTime: 5 * 60 * 1000,  // 5분간 캐시 사용, 이후 백그라운드 갱신
  // Room + Retrofit의 캐시 전략을 이 한 줄이 대체함
});
```

### 10-4. Paging3 → useInfiniteQuery

이 비교는 5장에서 이미 다루었다. TanStack Query의 useInfiniteQuery가 Paging3의 PagingSource + Pager + PagingConfig + cachedIn을 모두 대체한다.

---

## 11. 전체 CRUD 예제

사용자 관리 화면의 전체 CRUD를 구현한다.

### 11-1. API 및 훅 정의

```tsx
// hooks/useUserQueries.ts
import {
  useQuery,
  useMutation,
  useQueryClient,
  useInfiniteQuery,
} from '@tanstack/react-query';
import * as userApi from '../api/userApi';

// ===== READ =====

// 사용자 목록
export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: userApi.getUsers,
    staleTime: 5 * 60 * 1000,
  });
}

// 사용자 상세
export function useUser(id: number) {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => userApi.getUser(id),
    enabled: id > 0,
  });
}

// 사용자 목록 (무한 스크롤)
export function useUsersInfinite() {
  return useInfiniteQuery({
    queryKey: ['users', 'infinite'],
    queryFn: ({ pageParam }) => userApi.getUsersPaginated(pageParam),
    initialPageParam: 1,
    getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined,
  });
}

// ===== CREATE =====

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: userApi.createUser,
    onSuccess: (newUser) => {
      // 목록 캐시에 새 사용자 추가
      queryClient.setQueryData<userApi.User[]>(['users'], (old) =>
        old ? [...old, newUser] : [newUser]
      );
      // 상세 캐시도 설정
      queryClient.setQueryData(['users', newUser.id], newUser);
    },
  });
}

// ===== UPDATE =====

export function useUpdateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: number; data: userApi.UpdateUserDto }) =>
      userApi.updateUser(id, data),

    // 낙관적 업데이트
    onMutate: async ({ id, data }) => {
      await queryClient.cancelQueries({ queryKey: ['users'] });
      const previousUsers = queryClient.getQueryData<userApi.User[]>(['users']);
      const previousUser = queryClient.getQueryData<userApi.User>(['users', id]);

      // 목록 캐시 업데이트
      queryClient.setQueryData<userApi.User[]>(['users'], (old) =>
        old?.map((user) => (user.id === id ? { ...user, ...data } : user))
      );

      // 상세 캐시 업데이트
      if (previousUser) {
        queryClient.setQueryData(['users', id], { ...previousUser, ...data });
      }

      return { previousUsers, previousUser };
    },

    onError: (err, { id }, context) => {
      if (context?.previousUsers) {
        queryClient.setQueryData(['users'], context.previousUsers);
      }
      if (context?.previousUser) {
        queryClient.setQueryData(['users', id], context.previousUser);
      }
    },

    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
      queryClient.invalidateQueries({ queryKey: ['users', id] });
    },
  });
}

// ===== DELETE =====

export function useDeleteUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: userApi.deleteUser,

    onMutate: async (deletedId: number) => {
      await queryClient.cancelQueries({ queryKey: ['users'] });
      const previousUsers = queryClient.getQueryData<userApi.User[]>(['users']);

      queryClient.setQueryData<userApi.User[]>(['users'], (old) =>
        old?.filter((user) => user.id !== deletedId)
      );

      return { previousUsers };
    },

    onError: (err, deletedId, context) => {
      if (context?.previousUsers) {
        queryClient.setQueryData(['users'], context.previousUsers);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}
```

### 11-2. 화면 구현

```tsx
// screens/UserListScreen.tsx — 목록 + 삭제
import React from 'react';
import { View, Text, FlatList, TouchableOpacity, Alert, StyleSheet, ActivityIndicator } from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { useUsers, useDeleteUser } from '../hooks/useUserQueries';

function UserListScreen() {
  const navigation = useNavigation();
  const { data: users, isLoading, isError, error, refetch, isRefetching } = useUsers();
  const deleteMutation = useDeleteUser();

  const handleDelete = (user: User) => {
    Alert.alert(
      '삭제 확인',
      `${user.name}을(를) 삭제하시겠습니까?`,
      [
        { text: '취소', style: 'cancel' },
        {
          text: '삭제',
          style: 'destructive',
          onPress: () => deleteMutation.mutate(user.id),
        },
      ]
    );
  };

  if (isLoading) return <ActivityIndicator size="large" style={styles.center} />;
  if (isError) return <Text style={styles.error}>{error.message}</Text>;

  return (
    <View style={styles.container}>
      <FlatList
        data={users}
        keyExtractor={(item) => item.id.toString()}
        renderItem={({ item }) => (
          <TouchableOpacity
            style={styles.card}
            onPress={() => navigation.navigate('UserDetail', { userId: item.id })}
            onLongPress={() => handleDelete(item)}
          >
            <View style={styles.cardContent}>
              <Text style={styles.name}>{item.name}</Text>
              <Text style={styles.email}>{item.email}</Text>
            </View>
            <TouchableOpacity
              style={styles.editButton}
              onPress={() => navigation.navigate('UserEdit', { userId: item.id })}
            >
              <Text style={styles.editText}>편집</Text>
            </TouchableOpacity>
          </TouchableOpacity>
        )}
        refreshing={isRefetching}
        onRefresh={refetch}
      />

      <TouchableOpacity
        style={styles.fab}
        onPress={() => navigation.navigate('UserCreate')}
      >
        <Text style={styles.fabText}>+</Text>
      </TouchableOpacity>
    </View>
  );
}

// screens/UserDetailScreen.tsx — 상세 조회
function UserDetailScreen() {
  const route = useRoute();
  const { userId } = route.params as { userId: number };
  const { data: user, isLoading, isError, error } = useUser(userId);

  if (isLoading) return <ActivityIndicator size="large" />;
  if (isError) return <Text>에러: {error.message}</Text>;
  if (!user) return <Text>사용자를 찾을 수 없습니다</Text>;

  return (
    <View style={styles.detailContainer}>
      <Text style={styles.detailName}>{user.name}</Text>
      <Text style={styles.detailEmail}>{user.email}</Text>
      <Text style={styles.detailDate}>
        가입일: {new Date(user.createdAt).toLocaleDateString('ko-KR')}
      </Text>
    </View>
  );
}

// screens/UserCreateScreen.tsx — 생성
function UserCreateScreen() {
  const navigation = useNavigation();
  const createMutation = useCreateUser();
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  const handleCreate = async () => {
    try {
      await createMutation.mutateAsync({
        name,
        email,
        password: 'tempPassword123',
      });
      navigation.goBack();
    } catch (error: any) {
      Alert.alert('오류', error.message || '생성 실패');
    }
  };

  return (
    <View style={styles.formContainer}>
      <TextInput
        style={styles.input}
        placeholder="이름"
        value={name}
        onChangeText={setName}
      />
      <TextInput
        style={styles.input}
        placeholder="이메일"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TouchableOpacity
        style={[styles.submitButton, createMutation.isPending && styles.disabled]}
        onPress={handleCreate}
        disabled={createMutation.isPending}
      >
        <Text style={styles.submitText}>
          {createMutation.isPending ? '생성 중...' : '사용자 생성'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}

// screens/UserEditScreen.tsx — 수정
function UserEditScreen() {
  const navigation = useNavigation();
  const route = useRoute();
  const { userId } = route.params as { userId: number };
  const { data: user, isLoading } = useUser(userId);
  const updateMutation = useUpdateUser();

  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  // 유저 데이터 로드 후 폼 초기화
  useEffect(() => {
    if (user) {
      setName(user.name);
      setEmail(user.email);
    }
  }, [user]);

  const handleUpdate = async () => {
    try {
      await updateMutation.mutateAsync({
        id: userId,
        data: { name, email },
      });
      navigation.goBack();
    } catch (error: any) {
      Alert.alert('오류', error.message || '수정 실패');
    }
  };

  if (isLoading) return <ActivityIndicator size="large" />;

  return (
    <View style={styles.formContainer}>
      <TextInput
        style={styles.input}
        placeholder="이름"
        value={name}
        onChangeText={setName}
      />
      <TextInput
        style={styles.input}
        placeholder="이메일"
        value={email}
        onChangeText={setEmail}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TouchableOpacity
        style={[styles.submitButton, updateMutation.isPending && styles.disabled]}
        onPress={handleUpdate}
        disabled={updateMutation.isPending}
      >
        <Text style={styles.submitText}>
          {updateMutation.isPending ? '수정 중...' : '사용자 수정'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  error: { color: 'red', padding: 20, textAlign: 'center' },
  card: {
    flexDirection: 'row', backgroundColor: '#fff', padding: 16,
    marginHorizontal: 16, marginTop: 8, borderRadius: 12, elevation: 2,
    alignItems: 'center',
  },
  cardContent: { flex: 1 },
  name: { fontSize: 16, fontWeight: 'bold', color: '#333' },
  email: { fontSize: 14, color: '#666', marginTop: 4 },
  editButton: { padding: 8, backgroundColor: '#e3f2fd', borderRadius: 8 },
  editText: { color: '#1976d2', fontWeight: '600' },
  fab: {
    position: 'absolute', bottom: 20, right: 20, width: 56, height: 56,
    borderRadius: 28, backgroundColor: '#6200EE', justifyContent: 'center',
    alignItems: 'center', elevation: 8,
  },
  fabText: { color: '#fff', fontSize: 28, lineHeight: 30 },
  detailContainer: { flex: 1, padding: 20, backgroundColor: '#fff' },
  detailName: { fontSize: 28, fontWeight: 'bold', color: '#333', marginBottom: 8 },
  detailEmail: { fontSize: 18, color: '#666', marginBottom: 16 },
  detailDate: { fontSize: 14, color: '#999' },
  formContainer: { flex: 1, padding: 20, backgroundColor: '#fff' },
  input: {
    borderWidth: 1, borderColor: '#ddd', borderRadius: 8, padding: 12,
    fontSize: 16, marginBottom: 16,
  },
  submitButton: {
    backgroundColor: '#6200EE', padding: 16, borderRadius: 8, alignItems: 'center',
  },
  disabled: { opacity: 0.5 },
  submitText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
});
```

---

## 요약: Android 네트워킹 스택 vs TanStack Query

```
Android                              React Native (TanStack Query)
─────────────────────────────────   ─────────────────────────────
Retrofit interface                   API 함수 (fetch/axios)
@GET annotation                      useQuery
@POST / @PUT / @DELETE               useMutation
Paging3 (PagingSource+Pager)         useInfiniteQuery
Repository 캐싱                      내장 메모리 캐시 (staleTime/gcTime)
Flow + sealed class UiState          isLoading / isError / data 자동
viewModelScope.launch                자동 생명주기 관리
Flow.distinctUntilChanged()          Selector + structural sharing
OkHttp Interceptor                   axios interceptor (동일)
Room 로컬 캐시                       persist middleware (선택)
```

다음 문서에서는 로컬 데이터 저장을 다룬다: AsyncStorage, MMKV, SQLite 등 Android의 SharedPreferences/Room 대체 솔루션을 학습한다.
