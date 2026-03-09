# 2026 추천 라이브러리 가이드 — Android 라이브러리와 1:1 매핑

## 목차
1. [네비게이션](#1-네비게이션)
2. [상태 관리](#2-상태-관리)
3. [서버 상태 관리](#3-서버-상태-관리)
4. [로컬 저장소](#4-로컬-저장소)
5. [스타일링 (NativeWind)](#5-스타일링-nativewind)
6. [폼 관리](#6-폼-관리)
7. [애니메이션](#7-애니메이션)
8. [제스처](#8-제스처)
9. [이미지 로딩](#9-이미지-로딩)
10. [아이콘](#10-아이콘)
11. [카메라](#11-카메라)
12. [위치 정보](#12-위치-정보)
13. [푸시 알림](#13-푸시-알림)
14. [스플래시 스크린](#14-스플래시-스크린)
15. [SVG](#15-svg)
16. [WebView](#16-webview)
17. [날짜 처리](#17-날짜-처리)
18. [다국어 (i18n)](#18-다국어-i18n)
19. [애널리틱스](#19-애널리틱스)
20. [크래시 리포팅](#20-크래시-리포팅)

---

## 1. 네비게이션

**Android**: Navigation Component + SafeArgs
**React Native**: `@react-navigation/native` v7

```bash
# 설치
npm install @react-navigation/native @react-navigation/native-stack @react-navigation/bottom-tabs
npm install react-native-screens react-native-safe-area-context
```

```typescript
// 기본 설정 — Android의 nav_graph.xml에 해당
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

// 타입 안전한 네비게이션 (SafeArgs 대응)
type RootStackParamList = {
  Home: undefined;
  Detail: { itemId: string };
  Settings: undefined;
};

const Stack = createNativeStackNavigator<RootStackParamList>();

function AppNavigator() {
  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: '홈' }}
        />
        <Stack.Screen
          name="Detail"
          component={DetailScreen}
          options={{ title: '상세' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// 화면에서 네비게이션 사용 (SafeArgs의 navigate()에 해당)
import { useNavigation, useRoute } from '@react-navigation/native';
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';

function HomeScreen() {
  const navigation = useNavigation<NativeStackNavigationProp<RootStackParamList>>();

  return (
    <TouchableOpacity onPress={() => navigation.navigate('Detail', { itemId: '123' })}>
      <Text>상세 보기</Text>
    </TouchableOpacity>
  );
}

function DetailScreen() {
  const route = useRoute<RouteProp<RootStackParamList, 'Detail'>>();
  const { itemId } = route.params; // 타입 안전!

  return <Text>아이템 ID: {itemId}</Text>;
}
```

---

## 2. 상태 관리

**Android**: ViewModel + StateFlow / Hilt
**React Native**: `zustand`

```bash
npm install zustand
```

```typescript
// Zustand 스토어 — Android ViewModel + StateFlow에 해당
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// MMKV를 Zustand persist 저장소로 사용
const mmkvStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};

interface AuthState {
  token: string | null;
  user: User | null;
  isAuthenticated: boolean;
  login: (token: string, user: User) => void;
  logout: () => void;
}

// Android ViewModel 비교:
// class AuthViewModel @Inject constructor(...) : ViewModel() {
//   private val _state = MutableStateFlow(AuthState())
//   val state: StateFlow<AuthState> = _state.asStateFlow()
// }

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      token: null,
      user: null,
      isAuthenticated: false,

      login: (token, user) => set({
        token,
        user,
        isAuthenticated: true,
      }),

      logout: () => set({
        token: null,
        user: null,
        isAuthenticated: false,
      }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => mmkvStorage),
    }
  )
);

// 컴포넌트에서 사용 (collectAsState()에 해당)
function ProfileScreen() {
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);

  return (
    <View>
      <Text>{user?.name}</Text>
      <Button title="로그아웃" onPress={logout} />
    </View>
  );
}
```

---

## 3. 서버 상태 관리

**Android**: Retrofit + Coroutine + Flow
**React Native**: `@tanstack/react-query` v5

```bash
npm install @tanstack/react-query
```

```typescript
// TanStack Query 설정 — Retrofit + Repository 패턴 대응
import { QueryClient, QueryClientProvider, useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5분간 fresh
      gcTime: 1000 * 60 * 30,   // 30분간 캐시 유지
      retry: 2,
    },
  },
});

// Provider 래핑 (Hilt Module 제공과 유사)
function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RootNavigator />
    </QueryClientProvider>
  );
}

// 데이터 조회 훅 (Repository.getUsers() + ViewModel에 해당)
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('/api/users');
      if (!response.ok) throw new Error('Failed to fetch users');
      return response.json() as Promise<User[]>;
    },
  });
}

// 데이터 변경 훅 (Repository.createUser()에 해당)
function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newUser: CreateUserRequest) => {
      const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newUser),
      });
      return response.json() as Promise<User>;
    },
    // 성공 시 목록 새로고침
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

// 컴포넌트에서 사용
function UserList() {
  const { data: users, isLoading, error } = useUsers();
  const createUser = useCreateUser();

  if (isLoading) return <Loading />;
  if (error) return <ErrorView message={error.message} />;

  return (
    <FlatList
      data={users}
      renderItem={({ item }) => <UserCard user={item} />}
    />
  );
}
```

---

## 4. 로컬 저장소

**Android**: SharedPreferences / DataStore / Room
**React Native**: `react-native-mmkv`

```bash
npm install react-native-mmkv
```

```typescript
// MMKV — SharedPreferences/DataStore에 해당 (하지만 10배 빠름)
import { MMKV } from 'react-native-mmkv';

// 인스턴스 생성 (SharedPreferences의 getSharedPreferences()에 해당)
const storage = new MMKV();
const secureStorage = new MMKV({ id: 'secure', encryptionKey: 'my-encryption-key' });

// 저장/읽기 (SharedPreferences.Editor에 해당)
// Android: editor.putString("username", "홍길동").apply()
storage.set('username', '홍길동');
storage.set('age', 25);
storage.set('isPremium', true);
storage.set('settings', JSON.stringify({ theme: 'dark', language: 'ko' }));

// Android: prefs.getString("username", null)
const username = storage.getString('username');   // '홍길동'
const age = storage.getNumber('age');             // 25
const isPremium = storage.getBoolean('isPremium'); // true
const settings = JSON.parse(storage.getString('settings') || '{}');

// 삭제
storage.delete('username');
storage.clearAll(); // 모든 데이터 삭제

// 키 목록
const allKeys = storage.getAllKeys(); // ['age', 'isPremium', 'settings']

// 존재 확인
const hasUsername = storage.contains('username'); // false

// 타입 안전한 래퍼
export const appStorage = {
  getUser(): User | null {
    const json = storage.getString('user');
    return json ? JSON.parse(json) : null;
  },
  setUser(user: User) {
    storage.set('user', JSON.stringify(user));
  },
  getTheme(): 'light' | 'dark' {
    return (storage.getString('theme') as 'light' | 'dark') || 'light';
  },
  setTheme(theme: 'light' | 'dark') {
    storage.set('theme', theme);
  },
};
```

---

## 5. 스타일링 (NativeWind)

**Android**: Jetpack Compose Material Theme
**React Native**: `nativewind` v4 (Tailwind CSS for React Native)

```bash
# 설치
npm install nativewind tailwindcss
npx tailwindcss init
```

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './App.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        primary: '#007AFF',
        secondary: '#5856D6',
      },
    },
  },
  plugins: [],
};
```

```javascript
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset', 'nativewind/babel'],
};
```

```javascript
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = mergeConfig(getDefaultConfig(__dirname), {});
module.exports = withNativeWind(config, { input: './global.css' });
```

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

```typescript
// 사용법 — Compose의 Modifier 체이닝과 유사
import { View, Text, TouchableOpacity } from 'react-native';

// Compose: Column(modifier = Modifier.fillMaxWidth().padding(16.dp).background(Color.White))
// NativeWind:
function Card() {
  return (
    <View className="w-full p-4 bg-white rounded-lg shadow-md">
      <Text className="text-lg font-bold text-gray-900">제목</Text>
      <Text className="text-sm text-gray-500 mt-2">설명 텍스트</Text>
      <TouchableOpacity className="mt-4 bg-primary py-3 px-6 rounded-lg">
        <Text className="text-white text-center font-semibold">버튼</Text>
      </TouchableOpacity>
    </View>
  );
}

// 다크 모드
function ThemeCard() {
  return (
    <View className="bg-white dark:bg-gray-800 p-4 rounded-lg">
      <Text className="text-gray-900 dark:text-white">다크 모드 지원</Text>
    </View>
  );
}

// 반응형 (기기 크기별)
function ResponsiveLayout() {
  return (
    <View className="flex-1 p-4 md:p-8 lg:p-12">
      <Text className="text-base md:text-lg lg:text-xl">반응형 텍스트</Text>
    </View>
  );
}
```

---

## 6. 폼 관리

**Android**: EditText + TextWatcher / Compose TextField + ViewModel
**React Native**: `react-hook-form` + `zod`

```bash
npm install react-hook-form zod @hookform/resolvers
```

```typescript
// Zod 스키마 정의 — Android에서 직접 검증 로직 작성하는 것을 대체
import { z } from 'zod';

export const registerSchema = z.object({
  name: z.string()
    .min(2, '이름은 2자 이상이어야 합니다')
    .max(50, '이름은 50자 이하여야 합니다'),
  email: z.string()
    .email('올바른 이메일 형식이 아닙니다'),
  password: z.string()
    .min(8, '비밀번호는 8자 이상이어야 합니다')
    .regex(/[A-Z]/, '대문자를 포함해야 합니다')
    .regex(/[0-9]/, '숫자를 포함해야 합니다')
    .regex(/[!@#$%^&*]/, '특수문자를 포함해야 합니다'),
  confirmPassword: z.string(),
  agreeToTerms: z.boolean().refine(val => val === true, {
    message: '약관에 동의해야 합니다',
  }),
}).refine((data) => data.password === data.confirmPassword, {
  message: '비밀번호가 일치하지 않습니다',
  path: ['confirmPassword'],
});

type RegisterFormData = z.infer<typeof registerSchema>;
```

```typescript
// 폼 컴포넌트 — 전체 회원가입 폼 구현
import React from 'react';
import { View, Text, TextInput, TouchableOpacity, Switch, StyleSheet } from 'react-native';
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { registerSchema, type RegisterFormData } from '../schemas';

export function RegisterForm() {
  const {
    control,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
    defaultValues: {
      name: '',
      email: '',
      password: '',
      confirmPassword: '',
      agreeToTerms: false,
    },
  });

  const onSubmit = async (data: RegisterFormData) => {
    console.log('폼 데이터:', data);
    // API 호출
  };

  return (
    <View style={styles.container}>
      {/* 이름 필드 */}
      <Controller
        control={control}
        name="name"
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.field}>
            <Text style={styles.label}>이름</Text>
            <TextInput
              style={[styles.input, errors.name && styles.inputError]}
              placeholder="이름을 입력하세요"
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
            />
            {errors.name && (
              <Text style={styles.error}>{errors.name.message}</Text>
            )}
          </View>
        )}
      />

      {/* 이메일 필드 */}
      <Controller
        control={control}
        name="email"
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.field}>
            <Text style={styles.label}>이메일</Text>
            <TextInput
              style={[styles.input, errors.email && styles.inputError]}
              placeholder="이메일을 입력하세요"
              keyboardType="email-address"
              autoCapitalize="none"
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
            />
            {errors.email && (
              <Text style={styles.error}>{errors.email.message}</Text>
            )}
          </View>
        )}
      />

      {/* 비밀번호 필드 */}
      <Controller
        control={control}
        name="password"
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.field}>
            <Text style={styles.label}>비밀번호</Text>
            <TextInput
              style={[styles.input, errors.password && styles.inputError]}
              placeholder="비밀번호를 입력하세요"
              secureTextEntry
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
            />
            {errors.password && (
              <Text style={styles.error}>{errors.password.message}</Text>
            )}
          </View>
        )}
      />

      {/* 비밀번호 확인 필드 */}
      <Controller
        control={control}
        name="confirmPassword"
        render={({ field: { onChange, onBlur, value } }) => (
          <View style={styles.field}>
            <Text style={styles.label}>비밀번호 확인</Text>
            <TextInput
              style={[styles.input, errors.confirmPassword && styles.inputError]}
              placeholder="비밀번호를 다시 입력하세요"
              secureTextEntry
              value={value}
              onChangeText={onChange}
              onBlur={onBlur}
            />
            {errors.confirmPassword && (
              <Text style={styles.error}>{errors.confirmPassword.message}</Text>
            )}
          </View>
        )}
      />

      {/* 약관 동의 */}
      <Controller
        control={control}
        name="agreeToTerms"
        render={({ field: { onChange, value } }) => (
          <View style={styles.switchRow}>
            <Switch value={value} onValueChange={onChange} />
            <Text style={styles.switchLabel}>서비스 약관에 동의합니다</Text>
          </View>
        )}
      />
      {errors.agreeToTerms && (
        <Text style={styles.error}>{errors.agreeToTerms.message}</Text>
      )}

      {/* 제출 버튼 */}
      <TouchableOpacity
        style={[styles.button, isSubmitting && styles.buttonDisabled]}
        onPress={handleSubmit(onSubmit)}
        disabled={isSubmitting}
      >
        <Text style={styles.buttonText}>
          {isSubmitting ? '처리 중...' : '회원가입'}
        </Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { padding: 16 },
  field: { marginBottom: 16 },
  label: { fontSize: 14, fontWeight: '600', marginBottom: 4, color: '#333' },
  input: {
    borderWidth: 1, borderColor: '#ddd', borderRadius: 8,
    padding: 12, fontSize: 16, backgroundColor: '#fff',
  },
  inputError: { borderColor: '#ff3b30' },
  error: { color: '#ff3b30', fontSize: 12, marginTop: 4 },
  switchRow: { flexDirection: 'row', alignItems: 'center', marginBottom: 16 },
  switchLabel: { marginLeft: 8, fontSize: 14 },
  button: {
    backgroundColor: '#007AFF', padding: 16, borderRadius: 8,
    alignItems: 'center',
  },
  buttonDisabled: { opacity: 0.6 },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

## 7. 애니메이션

**Android**: MotionLayout / Property Animation / Compose Animation
**React Native**: `react-native-reanimated` v3

```bash
npm install react-native-reanimated
```

```javascript
// babel.config.js — reanimated 플러그인은 반드시 마지막에!
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: ['react-native-reanimated/plugin'], // 반드시 마지막
};
```

```typescript
// 기본 예제: 페이드 인/아웃 애니메이션
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
} from 'react-native-reanimated';

function FadeInBox() {
  // Android의 ValueAnimator에 해당
  const opacity = useSharedValue(0);
  const scale = useSharedValue(0.5);

  // Android의 addUpdateListener에 해당
  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [{ scale: scale.value }],
  }));

  const show = () => {
    // withTiming = ValueAnimator (선형/커브)
    opacity.value = withTiming(1, { duration: 300 });
    // withSpring = SpringAnimation
    scale.value = withSpring(1, { damping: 15, stiffness: 150 });
  };

  return (
    <View>
      <Animated.View style={[styles.box, animatedStyle]} />
      <Button title="보이기" onPress={show} />
    </View>
  );
}
```

---

## 8. 제스처

**Android**: GestureDetector / Compose Modifier.pointerInput
**React Native**: `react-native-gesture-handler`

```bash
npm install react-native-gesture-handler
```

```typescript
// 스와이프 투 딜리트 예제
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';

function SwipeableCard({ onDelete }: { onDelete: () => void }) {
  const translateX = useSharedValue(0);

  const panGesture = Gesture.Pan()
    .onUpdate((event) => {
      // 왼쪽으로만 스와이프 허용
      translateX.value = Math.min(0, event.translationX);
    })
    .onEnd((event) => {
      if (event.translationX < -100) {
        // 충분히 스와이프했으면 삭제
        translateX.value = withSpring(-300);
        runOnJS(onDelete)();
      } else {
        // 원위치로 복귀
        translateX.value = withSpring(0);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>스와이프하여 삭제</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

---

## 9. 이미지 로딩

**Android**: Glide / Coil
**React Native**: `expo-image`

```bash
npx expo install expo-image
```

```typescript
// expo-image — Glide/Coil에 해당
import { Image } from 'expo-image';

// Glide/Coil의 placeholder, error, crossfade에 해당하는 기능 모두 제공
function UserAvatar({ url }: { url: string }) {
  return (
    <Image
      source={{ uri: url }}
      style={{ width: 80, height: 80, borderRadius: 40 }}
      placeholder={require('../assets/placeholder-avatar.png')} // Glide placeholder
      contentFit="cover"       // Glide의 centerCrop
      transition={200}         // Glide의 crossFade
      cachePolicy="memory-disk" // Glide의 diskCacheStrategy
      recyclingKey={url}       // RecyclerView에서 재사용 최적화
    />
  );
}

// contentFit 옵션:
// 'cover'   → Android의 centerCrop (기본값, 가장 많이 사용)
// 'contain' → Android의 fitCenter
// 'fill'    → Android의 fitXY
// 'none'    → Android의 center

// 블러 해시 (저해상도 프리뷰)
function ArticleImage({ url, blurhash }: { url: string; blurhash: string }) {
  return (
    <Image
      source={{ uri: url }}
      placeholder={{ blurhash }}  // 로딩 중 블러 이미지 표시
      style={{ width: '100%', height: 200 }}
      contentFit="cover"
      transition={300}
    />
  );
}
```

---

## 10. 아이콘

**Android**: Material Icons / Vector Drawable
**React Native**: `@expo/vector-icons`

```bash
npx expo install @expo/vector-icons
```

```typescript
// 6000+ 아이콘 제공 (MaterialIcons, FontAwesome, Ionicons 등 포함)
import { Ionicons, MaterialIcons, FontAwesome } from '@expo/vector-icons';

function TabBar() {
  return (
    <View style={styles.tabBar}>
      {/* Ionicons — iOS 스타일 아이콘 */}
      <Ionicons name="home" size={24} color="#007AFF" />

      {/* MaterialIcons — Android Material 아이콘 (Android 개발자에게 익숙) */}
      <MaterialIcons name="search" size={24} color="#333" />

      {/* FontAwesome */}
      <FontAwesome name="bookmark" size={24} color="#333" />

      {/* MaterialIcons의 outlined 변형 */}
      <MaterialIcons name="settings" size={24} color="#333" />
    </View>
  );
}

// 아이콘을 버튼으로 사용
function IconButton({ name, onPress }: { name: string; onPress: () => void }) {
  return (
    <TouchableOpacity onPress={onPress} style={styles.iconButton}>
      <Ionicons name={name as any} size={24} color="#007AFF" />
    </TouchableOpacity>
  );
}
```

---

## 11. 카메라

**Android**: CameraX
**React Native**: `expo-camera`

```bash
npx expo install expo-camera
```

```typescript
// 카메라 — CameraX에 해당
import { CameraView, useCameraPermissions } from 'expo-camera';
import { useRef, useState } from 'react';

function CameraScreen() {
  const [permission, requestPermission] = useCameraPermissions();
  const [facing, setFacing] = useState<'back' | 'front'>('back');
  const cameraRef = useRef<CameraView>(null);

  if (!permission) return <View />;

  if (!permission.granted) {
    return (
      <View style={styles.container}>
        <Text>카메라 권한이 필요합니다</Text>
        <Button title="권한 요청" onPress={requestPermission} />
      </View>
    );
  }

  const takePicture = async () => {
    if (cameraRef.current) {
      // CameraX의 takePicture()에 해당
      const photo = await cameraRef.current.takePictureAsync({
        quality: 0.8,
        base64: false,
      });
      console.log('사진 URI:', photo?.uri);
    }
  };

  return (
    <View style={{ flex: 1 }}>
      <CameraView
        ref={cameraRef}
        style={{ flex: 1 }}
        facing={facing}
      >
        <View style={styles.controls}>
          <TouchableOpacity onPress={() =>
            setFacing(f => f === 'back' ? 'front' : 'back')
          }>
            <Text style={styles.controlText}>전환</Text>
          </TouchableOpacity>
          <TouchableOpacity onPress={takePicture}>
            <View style={styles.captureButton} />
          </TouchableOpacity>
        </View>
      </CameraView>
    </View>
  );
}
```

---

## 12. 위치 정보

**Android**: FusedLocationProviderClient
**React Native**: `expo-location`

```bash
npx expo install expo-location
```

```typescript
// 위치 정보 — FusedLocationProviderClient에 해당
import * as Location from 'expo-location';

function useCurrentLocation() {
  const [location, setLocation] = useState<Location.LocationObject | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    (async () => {
      // 권한 요청 (Android Manifest + 런타임 권한에 해당)
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        setError('위치 권한이 거부되었습니다');
        return;
      }

      // 현재 위치 가져오기 (getCurrentLocation()에 해당)
      const currentLocation = await Location.getCurrentPositionAsync({
        accuracy: Location.Accuracy.High, // GPS 정확도
      });
      setLocation(currentLocation);
    })();
  }, []);

  return { location, error };
}

// 위치 추적 (requestLocationUpdates()에 해당)
async function startTracking() {
  const subscription = await Location.watchPositionAsync(
    {
      accuracy: Location.Accuracy.High,
      timeInterval: 5000,    // 5초마다
      distanceInterval: 10,  // 10미터마다
    },
    (newLocation) => {
      console.log('새 위치:', newLocation.coords);
    }
  );

  // 추적 중지
  // subscription.remove();
}
```

---

## 13. 푸시 알림

**Android**: Firebase Cloud Messaging (FCM)
**React Native**: `expo-notifications`

```bash
npx expo install expo-notifications expo-device expo-constants
```

```typescript
// 푸시 알림 — FCM에 해당
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { useEffect, useRef, useState } from 'react';

// 알림 핸들러 설정 (앱이 포그라운드일 때)
// Android의 FirebaseMessagingService.onMessageReceived()에 해당
Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,   // 앱 포그라운드에서도 알림 표시
    shouldPlaySound: true,
    shouldSetBadge: true,
  }),
});

// 푸시 토큰 등록
async function registerForPushNotifications(): Promise<string | null> {
  if (!Device.isDevice) {
    console.warn('에뮬레이터에서는 푸시 알림을 사용할 수 없습니다');
    return null;
  }

  // 권한 요청 (Android 13+ POST_NOTIFICATIONS에 해당)
  const { status } = await Notifications.requestPermissionsAsync();
  if (status !== 'granted') {
    console.warn('알림 권한이 거부되었습니다');
    return null;
  }

  // Expo Push Token 가져오기 (FCM 토큰에 해당)
  const token = await Notifications.getExpoPushTokenAsync({
    projectId: Constants.expoConfig?.extra?.eas?.projectId,
  });

  // Android 알림 채널 설정 (NotificationChannel에 해당)
  await Notifications.setNotificationChannelAsync('default', {
    name: '기본 알림',
    importance: Notifications.AndroidImportance.HIGH,
    vibrationPattern: [0, 250, 250, 250],
    lightColor: '#FF231F7C',
  });

  return token.data;
}

// 알림 리스너 훅
function useNotifications() {
  const [expoPushToken, setExpoPushToken] = useState<string | null>(null);
  const notificationListener = useRef<Notifications.EventSubscription>();
  const responseListener = useRef<Notifications.EventSubscription>();

  useEffect(() => {
    registerForPushNotifications().then(setExpoPushToken);

    // 알림 수신 리스너 (포그라운드)
    notificationListener.current = Notifications.addNotificationReceivedListener(
      (notification) => {
        console.log('알림 수신:', notification.request.content);
      }
    );

    // 알림 탭 리스너 (사용자가 알림을 탭했을 때)
    responseListener.current = Notifications.addNotificationResponseReceivedListener(
      (response) => {
        const data = response.notification.request.content.data;
        console.log('알림 탭:', data);
        // 딥 링크 처리: navigation.navigate(data.screen, data.params)
      }
    );

    return () => {
      notificationListener.current?.remove();
      responseListener.current?.remove();
    };
  }, []);

  return { expoPushToken };
}

// 로컬 알림 예약 (AlarmManager + NotificationCompat에 해당)
async function scheduleLocalNotification() {
  await Notifications.scheduleNotificationAsync({
    content: {
      title: '리마인더',
      body: '오늘의 뉴스를 확인하세요!',
      data: { screen: 'Home' },
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.TIME_INTERVAL,
      seconds: 60, // 60초 후
    },
  });
}
```

---

## 14. 스플래시 스크린

**Android**: SplashScreen API (Android 12+)
**React Native**: `expo-splash-screen`

```bash
npx expo install expo-splash-screen
```

```typescript
// 스플래시 스크린 — Android SplashScreen API에 해당
import * as SplashScreen from 'expo-splash-screen';
import { useEffect, useState } from 'react';

// 자동 숨김 방지 (데이터 로딩이 완료될 때까지 유지)
SplashScreen.preventAutoHideAsync();

export default function App() {
  const [isReady, setIsReady] = useState(false);

  useEffect(() => {
    async function prepare() {
      try {
        // 초기 데이터 로딩 (토큰 확인, 폰트 로딩 등)
        await loadFonts();
        await checkAuthToken();
        await preloadImages();
      } catch (error) {
        console.error('초기화 실패:', error);
      } finally {
        setIsReady(true);
      }
    }
    prepare();
  }, []);

  useEffect(() => {
    if (isReady) {
      // 준비 완료 시 스플래시 스크린 숨김
      SplashScreen.hideAsync();
    }
  }, [isReady]);

  if (!isReady) return null;

  return <AppProviders><RootNavigator /></AppProviders>;
}
```

---

## 15. SVG

**Android**: VectorDrawable / SVG 라이브러리
**React Native**: `react-native-svg`

```bash
npm install react-native-svg
```

```typescript
// SVG 컴포넌트
import Svg, { Circle, Rect, Path, G, Text as SvgText } from 'react-native-svg';

function Logo() {
  return (
    <Svg width={100} height={100} viewBox="0 0 100 100">
      <Circle cx={50} cy={50} r={45} fill="#007AFF" />
      <SvgText
        x={50} y={55}
        textAnchor="middle"
        fill="white"
        fontSize={24}
        fontWeight="bold"
      >
        RN
      </SvgText>
    </Svg>
  );
}

// SVG 파일을 컴포넌트로 사용 (react-native-svg-transformer 필요)
// npm install --save-dev react-native-svg-transformer
// metro.config.js에 transformer 설정 추가 후:
import Logo from '../assets/logo.svg';

function Header() {
  return <Logo width={32} height={32} fill="#007AFF" />;
}
```

---

## 16. WebView

**Android**: WebView / AndroidX WebView
**React Native**: `react-native-webview`

```bash
npm install react-native-webview
```

```typescript
// WebView — Android WebView에 해당
import { WebView } from 'react-native-webview';

function WebViewScreen({ url }: { url: string }) {
  return (
    <WebView
      source={{ uri: url }}
      // JavaScript 실행 허용 (Android의 settings.javaScriptEnabled)
      javaScriptEnabled={true}
      // DOM Storage 허용 (Android의 settings.domStorageEnabled)
      domStorageEnabled={true}
      // 시작 시 로딩 인디케이터
      startInLoadingState={true}
      // JS에서 RN으로 메시지 전달 (Android의 addJavascriptInterface)
      onMessage={(event) => {
        const data = JSON.parse(event.nativeEvent.data);
        console.log('WebView 메시지:', data);
      }}
      // RN에서 JS 실행 (Android의 evaluateJavascript)
      injectedJavaScript={`
        window.ReactNativeWebView.postMessage(JSON.stringify({
          height: document.body.scrollHeight
        }));
      `}
    />
  );
}
```

---

## 17. 날짜 처리

**Android**: java.time (LocalDate, ZonedDateTime, DateTimeFormatter)
**React Native**: `date-fns` 또는 `dayjs`

```bash
npm install date-fns
# 또는
npm install dayjs
```

```typescript
// date-fns — java.time에 해당 (함수형 스타일)
import { format, formatDistanceToNow, parseISO, isAfter, addDays } from 'date-fns';
import { ko } from 'date-fns/locale';

// 포맷팅 (DateTimeFormatter에 해당)
const date = new Date();
format(date, 'yyyy년 MM월 dd일', { locale: ko }); // "2026년 3월 9일"
format(date, 'a h:mm', { locale: ko });            // "오후 3:30"

// 상대적 시간 표시
formatDistanceToNow(parseISO('2026-03-08T10:00:00Z'), {
  addSuffix: true,
  locale: ko,
}); // "약 1일 전"

// 날짜 비교
isAfter(new Date(), parseISO('2026-01-01')); // true

// 날짜 연산
const nextWeek = addDays(new Date(), 7);
```

```typescript
// dayjs — 더 가벼운 대안 (moment.js API 호환)
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
import 'dayjs/locale/ko';

dayjs.extend(relativeTime);
dayjs.locale('ko');

dayjs().format('YYYY년 MM월 DD일');        // "2026년 03월 09일"
dayjs('2026-03-08').fromNow();             // "1일 전"
dayjs().add(7, 'day').format('MM/DD');     // "03/16"
```

---

## 18. 다국어 (i18n)

**Android**: strings.xml + Locale
**React Native**: `i18next` + `react-i18next`

```bash
npm install i18next react-i18next
```

```typescript
// src/i18n/index.ts — strings.xml에 해당
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

const resources = {
  ko: {
    translation: {
      common: {
        confirm: '확인',
        cancel: '취소',
        save: '저장',
        delete: '삭제',
        loading: '로딩 중...',
      },
      auth: {
        login: '로그인',
        logout: '로그아웃',
        email: '이메일',
        password: '비밀번호',
        welcomeMessage: '{{name}}님, 환영합니다!', // 변수 삽입
      },
      home: {
        title: '홈',
        articleCount: '총 {{count}}개의 기사',
      },
    },
  },
  en: {
    translation: {
      common: {
        confirm: 'Confirm',
        cancel: 'Cancel',
        save: 'Save',
        delete: 'Delete',
        loading: 'Loading...',
      },
      auth: {
        login: 'Login',
        logout: 'Logout',
        email: 'Email',
        password: 'Password',
        welcomeMessage: 'Welcome, {{name}}!',
      },
      home: {
        title: 'Home',
        articleCount: '{{count}} articles',
      },
    },
  },
};

i18n.use(initReactI18next).init({
  resources,
  lng: 'ko', // 기본 언어
  fallbackLng: 'en',
  interpolation: {
    escapeValue: false,
  },
});

export default i18n;
```

```typescript
// 컴포넌트에서 사용
import { useTranslation } from 'react-i18next';

function HomeScreen() {
  const { t, i18n } = useTranslation();

  // strings.xml의 getString(R.string.title)에 해당
  return (
    <View>
      <Text>{t('home.title')}</Text>
      <Text>{t('auth.welcomeMessage', { name: '홍길동' })}</Text>
      <Text>{t('home.articleCount', { count: 42 })}</Text>

      {/* 언어 변경 */}
      <Button title="English" onPress={() => i18n.changeLanguage('en')} />
      <Button title="한국어" onPress={() => i18n.changeLanguage('ko')} />
    </View>
  );
}
```

---

## 19. 애널리틱스

**Android**: Firebase Analytics
**React Native**: `@react-native-firebase/analytics`

```bash
npm install @react-native-firebase/app @react-native-firebase/analytics
```

```typescript
// Firebase Analytics — Android Firebase Analytics와 동일 API
import analytics from '@react-native-firebase/analytics';

// 화면 추적 (setCurrentScreen에 해당)
async function logScreenView(screenName: string) {
  await analytics().logScreenView({
    screen_name: screenName,
    screen_class: screenName,
  });
}

// 이벤트 로깅 (logEvent에 해당)
async function logAddToCart(item: { id: string; name: string; price: number }) {
  await analytics().logEvent('add_to_cart', {
    item_id: item.id,
    item_name: item.name,
    value: item.price,
    currency: 'KRW',
  });
}

// 사용자 속성 (setUserProperty에 해당)
await analytics().setUserProperty('preferred_language', 'ko');
await analytics().setUserId('user-123');

// 네비게이션과 통합: 화면 전환 시 자동 추적
import { NavigationContainer } from '@react-navigation/native';

function App() {
  const routeNameRef = useRef<string>();
  const navigationRef = useNavigationContainerRef();

  return (
    <NavigationContainer
      ref={navigationRef}
      onStateChange={() => {
        const currentRouteName = navigationRef.getCurrentRoute()?.name;
        if (currentRouteName && currentRouteName !== routeNameRef.current) {
          logScreenView(currentRouteName);
          routeNameRef.current = currentRouteName;
        }
      }}
    >
      <RootNavigator />
    </NavigationContainer>
  );
}
```

---

## 20. 크래시 리포팅

**Android**: Firebase Crashlytics / Sentry
**React Native**: `@sentry/react-native` 또는 `@react-native-firebase/crashlytics`

```bash
# Sentry (권장 — 더 상세한 리포트)
npm install @sentry/react-native
npx @sentry/wizard@latest -i reactNative

# 또는 Firebase Crashlytics
npm install @react-native-firebase/app @react-native-firebase/crashlytics
```

```typescript
// Sentry 설정 — Firebase Crashlytics에 해당
import * as Sentry from '@sentry/react-native';

Sentry.init({
  dsn: 'https://your-dsn@sentry.io/project-id',
  tracesSampleRate: 1.0, // 프로덕션에서는 0.1~0.2로 낮추기
  debug: __DEV__,
  environment: __DEV__ ? 'development' : 'production',
});

// App을 Sentry로 래핑
export default Sentry.wrap(App);

// 수동 에러 보고 (Crashlytics.recordException에 해당)
try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error);
}

// 사용자 정보 설정 (Crashlytics.setUserId에 해당)
Sentry.setUser({ id: 'user-123', email: 'user@example.com' });

// 커스텀 태그 (Crashlytics.setCustomKey에 해당)
Sentry.setTag('screen', 'HomeScreen');

// 브레드크럼 (이벤트 전 발생한 액션 추적)
Sentry.addBreadcrumb({
  category: 'navigation',
  message: 'HomeScreen → DetailScreen',
  level: 'info',
});
```

```typescript
// Firebase Crashlytics 방식
import crashlytics from '@react-native-firebase/crashlytics';

// 크래시 보고
try {
  await riskyOperation();
} catch (error) {
  crashlytics().recordError(error as Error);
}

// 사용자 ID 설정
crashlytics().setUserId('user-123');

// 커스텀 키
crashlytics().setAttribute('screen', 'HomeScreen');
crashlytics().setAttributes({ role: 'admin', plan: 'premium' });

// 로그 메시지 (크래시 발생 전 로그)
crashlytics().log('장바구니에 아이템 추가');

// 강제 크래시 (테스트용)
crashlytics().crash();
```

---

## 라이브러리 전체 매핑 요약

```
Android 라이브러리              React Native 대응
───────────────────────────────────────────────────
Navigation Component       → @react-navigation/native v7
ViewModel + StateFlow      → zustand
Retrofit + Coroutine       → @tanstack/react-query v5
SharedPreferences/DataStore → react-native-mmkv
Jetpack Compose Theme      → nativewind v4
EditText + TextWatcher     → react-hook-form + zod
Property Animation         → react-native-reanimated v3
GestureDetector            → react-native-gesture-handler
Glide / Coil               → expo-image
Material Icons             → @expo/vector-icons
CameraX                    → expo-camera
FusedLocationProvider      → expo-location
FCM                        → expo-notifications
SplashScreen API           → expo-splash-screen
VectorDrawable             → react-native-svg
WebView                    → react-native-webview
java.time                  → date-fns / dayjs
strings.xml + Locale       → i18next + react-i18next
Firebase Analytics         → @react-native-firebase/analytics
Firebase Crashlytics       → @sentry/react-native
Room Database              → expo-sqlite / WatermelonDB
WorkManager                → expo-task-manager
Hilt/Koin (DI)             → React Context / zustand (DI 불필요)
Paging 3                   → @tanstack/react-query useInfiniteQuery
```

## ✅ 학습 확인 퀴즈

```quiz
type: mcq
question: "React Native에서 Android의 SharedPreferences를 대체하는 고속 저장소 라이브러리는?"
options:
  - "AsyncStorage"
  - "react-native-mmkv"
  - "SQLite"
  - "Realm"
answer: "react-native-mmkv"
explanation: "MMKV는 WeChat이 개발한 고성능 key-value 저장소로, AsyncStorage보다 30배 이상 빠릅니다. Android의 SharedPreferences를 대체합니다."
```

```quiz
type: mcq
question: "2026년 기준 React Native 애니메이션에 권장되는 라이브러리는?"
options:
  - "Animated API (내장)"
  - "react-native-reanimated"
  - "react-spring"
  - "Lottie"
answer: "react-native-reanimated"
explanation: "react-native-reanimated v3는 네이티브 드리븐 애니메이션을 지원하여 JS 스레드를 차단하지 않고 60fps 애니메이션을 구현할 수 있습니다."
```
