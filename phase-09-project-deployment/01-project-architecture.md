# 실전 프로젝트 아키텍처 — Android Clean Architecture에서 React Native로

## 목차
1. [아키텍처 패턴 비교](#1-아키텍처-패턴-비교)
2. [권장 폴더 구조 (Feature-based)](#2-권장-폴더-구조-feature-based)
3. [API 레이어 설정](#3-api-레이어-설정)
4. [환경 설정 (Environment Configuration)](#4-환경-설정-environment-configuration)
5. [에러 바운더리](#5-에러-바운더리)
6. [앱 초기화 패턴](#6-앱-초기화-패턴)
7. [실전 프로젝트: News Reader App](#7-실전-프로젝트-news-reader-app)

---

## 1. 아키텍처 패턴 비교

### Android Clean Architecture 복습

```
[Android Clean Architecture]

presentation/         domain/              data/
├── ui/              ├── model/           ├── repository/
│   ├── Activity     │   ├── User.kt     │   ├── UserRepositoryImpl.kt
│   ├── Fragment     │   └── Post.kt     │   └── PostRepositoryImpl.kt
│   └── Adapter      ├── repository/     ├── remote/
├── viewmodel/       │   ├── UserRepo    │   ├── ApiService.kt
│   ├── UserVM       │   └── PostRepo    │   └── dto/
│   └── PostVM       └── usecase/        ├── local/
└── di/                  ├── GetUser     │   ├── AppDatabase.kt
    └── Module           └── GetPosts    │   └── dao/
                                         └── di/

[의존성 방향]: presentation → domain ← data
```

### React Native에서의 아키텍처 옵션

```
[Option 1: Feature-based (기능 기반) — 권장!]

장점: 관련 코드가 한 곳에 모여 있어 작업 효율이 높음
     기능을 추가/삭제할 때 영향 범위가 명확함

src/
├── app/              ← 앱 진입점, 프로바이더 설정
├── features/         ← 기능별 모듈 (가장 중요!)
│   ├── auth/         ← 인증 기능 전체
│   ├── home/         ← 홈 기능 전체
│   └── profile/      ← 프로필 기능 전체
├── shared/           ← 공유 컴포넌트/훅/유틸
├── services/         ← API 클라이언트, 스토리지
└── types/            ← 전역 타입

[Option 2: Layer-based (계층 기반)]

Android Clean Architecture와 가장 유사하지만,
기능이 여러 디렉토리에 흩어져 있어 개발 시 불편

src/
├── screens/          ← presentation (Activity/Fragment에 해당)
├── components/       ← presentation (커스텀 View에 해당)
├── hooks/            ← domain/presentation (ViewModel에 해당)
├── stores/           ← domain (UseCase + State에 해당)
├── services/         ← data (Repository에 해당)
├── types/            ← domain/model
└── utils/            ← domain (utility functions)
```

---

## 2. 권장 폴더 구조 (Feature-based)

```
src/
├── app/                          ← 앱 설정
│   ├── App.tsx                   ← 루트 컴포넌트
│   ├── navigation/
│   │   ├── RootNavigator.tsx     ← 최상위 네비게이터
│   │   ├── AuthNavigator.tsx     ← 인증 플로우 네비게이터
│   │   ├── MainTabNavigator.tsx  ← 메인 탭 네비게이터
│   │   └── types.ts              ← 네비게이션 타입
│   └── providers/
│       └── AppProviders.tsx      ← 모든 프로바이더 래핑
│
├── features/                     ← 기능별 모듈
│   ├── auth/                     ← 인증 기능
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── SocialLoginButtons.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts        ← 인증 로직 (ViewModel 역할)
│   │   │   └── useLoginForm.ts   ← 폼 로직
│   │   ├── screens/
│   │   │   ├── LoginScreen.tsx
│   │   │   └── RegisterScreen.tsx
│   │   ├── stores/
│   │   │   └── authStore.ts      ← Zustand 스토어
│   │   ├── services/
│   │   │   └── authApi.ts        ← 인증 API 호출
│   │   └── types.ts              ← 인증 관련 타입
│   │
│   ├── home/                     ← 홈 기능
│   │   ├── components/
│   │   │   ├── ArticleCard.tsx
│   │   │   ├── CategoryFilter.tsx
│   │   │   └── FeaturedBanner.tsx
│   │   ├── hooks/
│   │   │   ├── useArticles.ts    ← TanStack Query 훅
│   │   │   └── useCategories.ts
│   │   ├── screens/
│   │   │   └── HomeScreen.tsx
│   │   └── types.ts
│   │
│   ├── article/                  ← 게시글 상세
│   │   ├── components/
│   │   │   ├── ArticleContent.tsx
│   │   │   ├── CommentList.tsx
│   │   │   └── CommentInput.tsx
│   │   ├── hooks/
│   │   │   ├── useArticleDetail.ts
│   │   │   └── useComments.ts
│   │   ├── screens/
│   │   │   └── ArticleDetailScreen.tsx
│   │   └── types.ts
│   │
│   ├── bookmarks/                ← 북마크 기능
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── screens/
│   │   ├── stores/
│   │   │   └── bookmarkStore.ts
│   │   └── types.ts
│   │
│   └── settings/                 ← 설정
│       ├── components/
│       ├── screens/
│       └── stores/
│
├── shared/                       ← 공유 코드
│   ├── components/               ← 재사용 UI 컴포넌트
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Input.tsx
│   │   ├── Loading.tsx
│   │   ├── ErrorView.tsx
│   │   └── EmptyState.tsx
│   ├── hooks/                    ← 공유 커스텀 훅
│   │   ├── useDebounce.ts
│   │   ├── useKeyboard.ts
│   │   └── useNetworkStatus.ts
│   ├── utils/                    ← 유틸리티 함수
│   │   ├── format.ts             ← 날짜/숫자 포맷팅
│   │   ├── validation.ts         ← 입력값 검증
│   │   └── storage.ts            ← MMKV 래퍼
│   └── constants/
│       ├── colors.ts
│       ├── spacing.ts
│       └── typography.ts
│
├── services/                     ← 외부 서비스 연동
│   ├── api/
│   │   ├── client.ts             ← fetch 래퍼 (OkHttp 대응)
│   │   ├── interceptors.ts       ← 요청/응답 인터셉터
│   │   └── endpoints.ts          ← API 엔드포인트 상수
│   └── storage/
│       └── secureStorage.ts      ← 보안 저장소
│
└── types/                        ← 전역 타입 정의
    ├── global.d.ts
    ├── navigation.ts
    └── api.ts
```

### Android Clean Architecture와 매핑

```
Android                         React Native (Feature-based)
──────────────────────────────────────────────────────────────
Activity/Fragment              → screens/*.tsx
ViewModel                      → hooks/use*.ts
UseCase                        → hooks/use*.ts (훅 안에 통합)
Repository (interface)         → services/*Api.ts
Repository (impl)              → services/*Api.ts (인터페이스와 구현 통합)
Model/Entity                   → types.ts
DI Module                      → app/providers/AppProviders.tsx
Adapter/ViewHolder             → components/*.tsx
res/layout                     → 컴포넌트의 JSX (XML 대신)
res/values/strings             → shared/constants/ 또는 i18n
build.gradle                   → package.json
```

---

## 3. API 레이어 설정

### Fetch 래퍼 (OkHttp 인터셉터와 유사)

```typescript
// src/services/api/client.ts

import { useAuthStore } from '@/features/auth/stores/authStore';

// Android의 OkHttp + Retrofit에 해당하는 fetch 래퍼

interface ApiConfig {
  baseUrl: string;
  timeout: number;
}

interface RequestConfig extends RequestInit {
  params?: Record<string, string>;
  timeout?: number;
}

interface ApiError {
  status: number;
  message: string;
  code?: string;
}

const config: ApiConfig = {
  baseUrl: process.env.API_BASE_URL || 'https://api.example.com',
  timeout: 30000,
};

// 요청 인터셉터 (OkHttp의 Interceptor.intercept()에 해당)
async function requestInterceptor(
  url: string,
  options: RequestConfig
): Promise<[string, RequestConfig]> {
  const token = useAuthStore.getState().token;

  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    ...(options.headers as Record<string, string>),
  };

  // Authorization 헤더 추가 (OkHttp의 addHeader("Authorization", ...)와 동일)
  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }

  // 쿼리 파라미터 처리
  let fullUrl = `${config.baseUrl}${url}`;
  if (options.params) {
    const queryString = new URLSearchParams(options.params).toString();
    fullUrl += `?${queryString}`;
  }

  return [fullUrl, { ...options, headers }];
}

// 응답 인터셉터
async function responseInterceptor(response: Response): Promise<Response> {
  // 401: 토큰 만료 → 리프레시 시도
  if (response.status === 401) {
    const refreshed = await refreshToken();
    if (refreshed) {
      // 원래 요청을 새 토큰으로 재시도
      const newToken = useAuthStore.getState().token;
      const retryResponse = await fetch(response.url, {
        ...response,
        headers: {
          ...Object.fromEntries(response.headers.entries()),
          'Authorization': `Bearer ${newToken}`,
        },
      });
      return retryResponse;
    }

    // 리프레시 실패: 로그아웃
    useAuthStore.getState().logout();
    throw new ApiError(401, '인증이 만료되었습니다. 다시 로그인해주세요.');
  }

  return response;
}

// 토큰 리프레시
async function refreshToken(): Promise<boolean> {
  const refreshToken = useAuthStore.getState().refreshToken;
  if (!refreshToken) return false;

  try {
    const response = await fetch(`${config.baseUrl}/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refreshToken }),
    });

    if (response.ok) {
      const data = await response.json();
      useAuthStore.getState().setTokens(data.accessToken, data.refreshToken);
      return true;
    }
    return false;
  } catch {
    return false;
  }
}

// ApiError 클래스
class ApiError extends Error {
  status: number;
  code?: string;

  constructor(status: number, message: string, code?: string) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.code = code;
  }
}

// 메인 API 클라이언트
export const api = {
  async request<T>(url: string, options: RequestConfig = {}): Promise<T> {
    const [fullUrl, finalOptions] = await requestInterceptor(url, options);

    // 타임아웃 처리 (OkHttp의 callTimeout과 동일)
    const controller = new AbortController();
    const timeoutId = setTimeout(
      () => controller.abort(),
      options.timeout || config.timeout
    );

    try {
      const response = await fetch(fullUrl, {
        ...finalOptions,
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      // 응답 인터셉터 적용
      const interceptedResponse = await responseInterceptor(response);

      if (!interceptedResponse.ok) {
        const errorBody = await interceptedResponse.json().catch(() => ({}));
        throw new ApiError(
          interceptedResponse.status,
          errorBody.message || `HTTP ${interceptedResponse.status} 에러`,
          errorBody.code
        );
      }

      // 204 No Content
      if (interceptedResponse.status === 204) {
        return {} as T;
      }

      return interceptedResponse.json() as Promise<T>;
    } catch (error) {
      clearTimeout(timeoutId);

      if (error instanceof ApiError) throw error;

      if (error instanceof DOMException && error.name === 'AbortError') {
        throw new ApiError(0, '요청 시간이 초과되었습니다.');
      }

      throw new ApiError(0, '네트워크 연결을 확인해주세요.');
    }
  },

  get<T>(url: string, params?: Record<string, string>) {
    return this.request<T>(url, { method: 'GET', params });
  },

  post<T>(url: string, body?: unknown) {
    return this.request<T>(url, {
      method: 'POST',
      body: body ? JSON.stringify(body) : undefined,
    });
  },

  put<T>(url: string, body?: unknown) {
    return this.request<T>(url, {
      method: 'PUT',
      body: body ? JSON.stringify(body) : undefined,
    });
  },

  patch<T>(url: string, body?: unknown) {
    return this.request<T>(url, {
      method: 'PATCH',
      body: body ? JSON.stringify(body) : undefined,
    });
  },

  delete<T>(url: string) {
    return this.request<T>(url, { method: 'DELETE' });
  },
};
```

### 기능별 API 서비스

```typescript
// src/features/auth/services/authApi.ts
import { api } from '@/services/api/client';
import type { LoginRequest, LoginResponse, User } from '../types';

// Android의 Retrofit @POST("auth/login")에 해당
export const authApi = {
  login(data: LoginRequest): Promise<LoginResponse> {
    return api.post('/auth/login', data);
  },

  register(data: RegisterRequest): Promise<LoginResponse> {
    return api.post('/auth/register', data);
  },

  getProfile(): Promise<User> {
    return api.get('/auth/profile');
  },

  logout(): Promise<void> {
    return api.post('/auth/logout');
  },
};

// src/features/home/services/articleApi.ts
import { api } from '@/services/api/client';
import type { Article, ArticlesResponse } from '../types';

// Android의 Retrofit @GET("articles")에 해당
export const articleApi = {
  getArticles(page: number = 1, category?: string): Promise<ArticlesResponse> {
    return api.get('/articles', {
      page: String(page),
      ...(category && { category }),
    });
  },

  getArticleById(id: string): Promise<Article> {
    return api.get(`/articles/${id}`);
  },

  bookmarkArticle(id: string): Promise<void> {
    return api.post(`/articles/${id}/bookmark`);
  },
};
```

---

## 4. 환경 설정 (Environment Configuration)

### react-native-config 사용

```bash
npm install react-native-config
```

```bash
# .env (기본 - 개발)
API_BASE_URL=https://dev-api.example.com
API_KEY=dev-api-key-12345
APP_NAME=MyApp (Dev)
ENABLE_LOGGING=true

# .env.staging
API_BASE_URL=https://staging-api.example.com
API_KEY=staging-api-key-12345
APP_NAME=MyApp (Staging)
ENABLE_LOGGING=true

# .env.production
API_BASE_URL=https://api.example.com
API_KEY=prod-api-key-12345
APP_NAME=MyApp
ENABLE_LOGGING=false
```

```typescript
// src/config/env.ts
import Config from 'react-native-config';

export const env = {
  apiBaseUrl: Config.API_BASE_URL || 'https://api.example.com',
  apiKey: Config.API_KEY || '',
  appName: Config.APP_NAME || 'MyApp',
  enableLogging: Config.ENABLE_LOGGING === 'true',
  isDev: __DEV__,
} as const;

// 사용법
import { env } from '@/config/env';
console.log(env.apiBaseUrl); // https://dev-api.example.com
```

```groovy
// android/app/build.gradle
// react-native-config가 자동으로 .env 파일을 읽어서 BuildConfig에 추가

// 환경별 빌드:
// ENVFILE=.env.staging npx react-native run-android
// ENVFILE=.env.production npx react-native run-android

// 또는 package.json 스크립트:
// "android:staging": "ENVFILE=.env.staging npx react-native run-android",
// "android:prod": "ENVFILE=.env.production npx react-native run-android --mode release"
```

> **주의**: `.env` 파일을 `.gitignore`에 추가하되, `.env.example`은 커밋하여 필요한 환경 변수를 문서화한다.

---

## 5. 에러 바운더리

Android에서 `Thread.setDefaultUncaughtExceptionHandler()`로 크래시를 처리하는 것처럼, React Native에서는 **Error Boundary**를 사용한다.

```typescript
// src/shared/components/ErrorBoundary.tsx
import React, { Component, type ErrorInfo, type ReactNode } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

// Error Boundary는 반드시 클래스 컴포넌트로 작성해야 한다
// (getDerivedStateFromError, componentDidCatch는 클래스 전용)
export class ErrorBoundary extends Component<Props, State> {
  state: State = {
    hasError: false,
    error: null,
  };

  // 에러 발생 시 상태 업데이트
  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  // 에러 로깅 (Crashlytics 등에 보고)
  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('ErrorBoundary caught:', error, errorInfo);

    // Sentry나 Crashlytics에 보고
    // Sentry.captureException(error, { extra: { componentStack: errorInfo.componentStack } });

    this.props.onError?.(error, errorInfo);
  }

  handleReset = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) {
        return this.props.fallback;
      }

      return (
        <View style={styles.container}>
          <Text style={styles.emoji}>!</Text>
          <Text style={styles.title}>문제가 발생했습니다</Text>
          <Text style={styles.message}>
            {this.state.error?.message || '알 수 없는 오류가 발생했습니다'}
          </Text>
          <TouchableOpacity style={styles.button} onPress={this.handleReset}>
            <Text style={styles.buttonText}>다시 시도</Text>
          </TouchableOpacity>
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
    backgroundColor: '#fff',
  },
  emoji: { fontSize: 48, marginBottom: 16 },
  title: { fontSize: 20, fontWeight: 'bold', marginBottom: 8 },
  message: { fontSize: 14, color: '#666', textAlign: 'center', marginBottom: 24 },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

```typescript
// 사용법: App.tsx에서 전체 앱을 감싸기
import { ErrorBoundary } from '@/shared/components/ErrorBoundary';

export default function App() {
  return (
    <ErrorBoundary
      onError={(error, info) => {
        // Sentry.captureException(error);
        console.error('App error:', error);
      }}
    >
      <AppProviders>
        <RootNavigator />
      </AppProviders>
    </ErrorBoundary>
  );
}

// 기능별로도 개별 Error Boundary 적용 가능
function HomeScreen() {
  return (
    <View style={{ flex: 1 }}>
      <ErrorBoundary fallback={<Text>피드 로드 실패</Text>}>
        <ArticleFeed />
      </ErrorBoundary>
      <ErrorBoundary fallback={<Text>추천 로드 실패</Text>}>
        <Recommendations />
      </ErrorBoundary>
    </View>
  );
}
```

---

## 6. 앱 초기화 패턴

Android에서 `Application.onCreate()` → `SplashActivity` → `MainActivity` 순서로 초기화하는 것처럼, React Native에서도 유사한 패턴을 사용한다.

```typescript
// src/app/App.tsx
import React from 'react';
import { AppProviders } from './providers/AppProviders';
import { RootNavigator } from './navigation/RootNavigator';
import { ErrorBoundary } from '@/shared/components/ErrorBoundary';

export default function App() {
  return (
    <ErrorBoundary>
      <AppProviders>
        <RootNavigator />
      </AppProviders>
    </ErrorBoundary>
  );
}
```

```typescript
// src/app/providers/AppProviders.tsx
import React, { type ReactNode } from 'react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { NavigationContainer } from '@react-navigation/native';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5분
      gcTime: 1000 * 60 * 30,    // 30분
      retry: 2,
      refetchOnWindowFocus: false,
    },
  },
});

// Android의 Application 클래스에서 Hilt/Koin 모듈을 초기화하는 것에 해당
export function AppProviders({ children }: { children: ReactNode }) {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <SafeAreaProvider>
        <QueryClientProvider client={queryClient}>
          <NavigationContainer>
            {children}
          </NavigationContainer>
        </QueryClientProvider>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}
```

```typescript
// src/app/navigation/RootNavigator.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useAuth } from '@/features/auth/hooks/useAuth';
import { AuthNavigator } from './AuthNavigator';
import { MainTabNavigator } from './MainTabNavigator';
import { SplashScreen } from '@/features/auth/screens/SplashScreen';

const Stack = createNativeStackNavigator();

// Android의 Navigation Graph 최상위와 동일
export function RootNavigator() {
  const { isAuthenticated, isLoading } = useAuth();

  // 앱 초기화 중: 스플래시 화면
  if (isLoading) {
    return <SplashScreen />;
  }

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isAuthenticated ? (
        <Stack.Screen name="Main" component={MainTabNavigator} />
      ) : (
        <Stack.Screen name="Auth" component={AuthNavigator} />
      )}
    </Stack.Navigator>
  );
}
```

```typescript
// src/features/auth/hooks/useAuth.ts
import { useEffect, useState } from 'react';
import { useAuthStore } from '../stores/authStore';

// Android의 SplashActivity에서 토큰 확인 → MainActivity 또는 LoginActivity 이동 패턴
export function useAuth() {
  const { token, user, setUser, logout } = useAuthStore();
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    async function checkAuth() {
      try {
        if (token) {
          // 토큰이 있으면 유효성 확인
          const response = await fetch('/api/auth/profile', {
            headers: { Authorization: `Bearer ${token}` },
          });

          if (response.ok) {
            const userData = await response.json();
            setUser(userData);
          } else {
            logout(); // 토큰 만료
          }
        }
      } catch (error) {
        console.error('Auth check failed:', error);
      } finally {
        setIsLoading(false);
      }
    }

    checkAuth();
  }, [token]);

  return {
    isAuthenticated: !!token && !!user,
    isLoading,
    user,
  };
}
```

---

## 7. 실전 프로젝트: News Reader App

### 프로젝트 개요

```
[News Reader App — 기능 목록]

1. 인증: 로그인/회원가입/로그아웃
2. 홈: 카테고리별 뉴스 기사 목록 (무한 스크롤)
3. 기사 상세: 기사 전문, 댓글
4. 북마크: 관심 기사 저장/관리
5. 검색: 키워드 검색
6. 설정: 테마, 알림, 프로필

[기술 스택]
- Navigation: @react-navigation/native v7
- State: zustand (인증, 설정)
- Server State: @tanstack/react-query v5
- Storage: react-native-mmkv (토큰, 설정)
- Styling: StyleSheet (또는 NativeWind)
- API: fetch wrapper (위에서 만든 것)
```

### 타입 정의

```typescript
// src/features/home/types.ts

export interface Article {
  id: string;
  title: string;
  summary: string;
  content: string;
  imageUrl: string;
  category: Category;
  author: Author;
  publishedAt: string;
  readTime: number; // minutes
  commentCount: number;
  isBookmarked: boolean;
}

export interface Author {
  id: string;
  name: string;
  avatarUrl: string;
}

export type Category =
  | 'technology'
  | 'business'
  | 'science'
  | 'health'
  | 'sports'
  | 'entertainment';

export interface ArticlesResponse {
  articles: Article[];
  page: number;
  totalPages: number;
  totalCount: number;
}

// Android의 sealed class와 유사한 패턴으로 카테고리를 관리
export const CATEGORIES: { key: Category; label: string }[] = [
  { key: 'technology', label: '기술' },
  { key: 'business', label: '비즈니스' },
  { key: 'science', label: '과학' },
  { key: 'health', label: '건강' },
  { key: 'sports', label: '스포츠' },
  { key: 'entertainment', label: '엔터테인먼트' },
];
```

### TanStack Query 훅

```typescript
// src/features/home/hooks/useArticles.ts
import { useInfiniteQuery } from '@tanstack/react-query';
import { articleApi } from '../services/articleApi';
import type { Category } from '../types';

// Android의 Paging 3 라이브러리에 해당
export function useArticles(category?: Category) {
  return useInfiniteQuery({
    queryKey: ['articles', category],
    queryFn: ({ pageParam }) =>
      articleApi.getArticles(pageParam, category),
    initialPageParam: 1,
    getNextPageParam: (lastPage) =>
      lastPage.page < lastPage.totalPages ? lastPage.page + 1 : undefined,
    select: (data) => ({
      articles: data.pages.flatMap((page) => page.articles),
      totalCount: data.pages[0]?.totalCount ?? 0,
    }),
  });
}
```

### 홈 화면

```typescript
// src/features/home/screens/HomeScreen.tsx
import React, { useState, useCallback } from 'react';
import { View, FlatList, RefreshControl, StyleSheet } from 'react-native';
import { useArticles } from '../hooks/useArticles';
import { ArticleCard } from '../components/ArticleCard';
import { CategoryFilter } from '../components/CategoryFilter';
import { Loading } from '@/shared/components/Loading';
import { ErrorView } from '@/shared/components/ErrorView';
import type { Category, Article } from '../types';

export function HomeScreen() {
  const [selectedCategory, setSelectedCategory] = useState<Category | undefined>();
  const {
    data,
    isLoading,
    isError,
    error,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    refetch,
    isRefetching,
  } = useArticles(selectedCategory);

  const handleEndReached = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) {
      fetchNextPage();
    }
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  const renderItem = useCallback(({ item }: { item: Article }) => (
    <ArticleCard article={item} />
  ), []);

  const keyExtractor = useCallback((item: Article) => item.id, []);

  if (isLoading) return <Loading />;
  if (isError) return <ErrorView message={error.message} onRetry={refetch} />;

  return (
    <View style={styles.container}>
      <FlatList
        data={data?.articles}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
        ListHeaderComponent={
          <CategoryFilter
            selected={selectedCategory}
            onSelect={setSelectedCategory}
          />
        }
        onEndReached={handleEndReached}
        onEndReachedThreshold={0.5}
        ListFooterComponent={isFetchingNextPage ? <Loading /> : null}
        refreshControl={
          <RefreshControl refreshing={isRefetching} onRefresh={refetch} />
        }
        // 성능 최적화 (RecyclerView의 setHasFixedSize와 유사)
        removeClippedSubviews={true}
        maxToRenderPerBatch={10}
        windowSize={5}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f5f5f5' },
});
```

### 공유 컴포넌트 예시

```typescript
// src/shared/components/ErrorView.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

interface ErrorViewProps {
  message?: string;
  onRetry?: () => void;
}

export function ErrorView({
  message = '문제가 발생했습니다',
  onRetry,
}: ErrorViewProps) {
  return (
    <View style={styles.container}>
      <Text style={styles.message}>{message}</Text>
      {onRetry && (
        <TouchableOpacity style={styles.retryButton} onPress={onRetry}>
          <Text style={styles.retryText}>다시 시도</Text>
        </TouchableOpacity>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 24,
  },
  message: {
    fontSize: 16,
    color: '#666',
    textAlign: 'center',
    marginBottom: 16,
  },
  retryButton: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  retryText: {
    color: '#fff',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

### 네비게이션 타입 정의

```typescript
// src/app/navigation/types.ts
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import type { BottomTabScreenProps } from '@react-navigation/bottom-tabs';
import type { CompositeScreenProps, NavigatorScreenParams } from '@react-navigation/native';

// Android의 Navigation SafeArgs에 해당하는 타입 안전 네비게이션

// 인증 플로우 파라미터
export type AuthStackParamList = {
  Login: undefined;
  Register: undefined;
  ForgotPassword: { email?: string };
};

// 메인 탭 파라미터
export type MainTabParamList = {
  Home: NavigatorScreenParams<HomeStackParamList>;
  Search: undefined;
  Bookmarks: undefined;
  Settings: undefined;
};

// 홈 스택 파라미터
export type HomeStackParamList = {
  HomeList: undefined;
  ArticleDetail: { articleId: string };
  AuthorProfile: { authorId: string };
};

// 화면별 Props 타입
export type HomeScreenProps = CompositeScreenProps<
  NativeStackScreenProps<HomeStackParamList, 'HomeList'>,
  BottomTabScreenProps<MainTabParamList>
>;

export type ArticleDetailProps = NativeStackScreenProps<
  HomeStackParamList,
  'ArticleDetail'
>;
```
