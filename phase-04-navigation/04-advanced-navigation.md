# 고급 네비게이션 패턴 — 실전 앱 네비게이션 설계

## 목차
1. [딥링크 설정과 테스트](#1-딥링크-설정과-테스트)
2. [네비게이션 상태 영속화](#2-네비게이션-상태-영속화)
3. [화면 추적과 애널리틱스](#3-화면-추적과-애널리틱스)
4. [인증 플로우 패턴](#4-인증-플로우-패턴)
5. [네비게이션 이벤트](#5-네비게이션-이벤트)
6. [뒤로가기 방지 (저장하지 않은 변경)](#6-뒤로가기-방지)
7. [커스텀 전환과 Shared Element Transition](#7-커스텀-전환과-shared-element-transition)
8. [컴포넌트 바깥에서 네비게이션 (navigationRef)](#8-컴포넌트-바깥에서-네비게이션)
9. [URL 기반 네비게이션 (Linking Configuration)](#9-url-기반-네비게이션)
10. [성능 최적화](#10-성능-최적화)
11. [프로덕션 레디 네비게이션 아키텍처](#11-프로덕션-레디-네비게이션-아키텍처)

---

## 1. 딥링크 설정과 테스트

### 1-1. 딥링크 개념

Android에서 딥링크는 `AndroidManifest.xml`의 `intent-filter`로 설정한다. React Navigation에서는 JavaScript 설정 객체로 동일한 기능을 구현한다.

```xml
<!-- Android: AndroidManifest.xml -->
<activity android:name=".MainActivity">
  <!-- 커스텀 스킴 딥링크 -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>

  <!-- 유니버설 링크 (App Links) -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="myapp.com" />
  </intent-filter>
</activity>
```

React Navigation에서의 동일 설정:

```tsx
// linking.ts
import { LinkingOptions } from '@react-navigation/native';

const linking: LinkingOptions<RootStackParamList> = {
  // URI 스킴 프리픽스 (Android의 intent-filter data 태그에 해당)
  prefixes: [
    'myapp://',                    // 커스텀 스킴
    'https://myapp.com',           // 유니버설 링크
    'https://www.myapp.com',       // www 포함
  ],

  // 화면별 URL 매핑 (Android NavGraph의 deepLink 태그에 해당)
  config: {
    // 초기 화면
    initialRouteName: 'MainTabs',

    screens: {
      // 단순 매핑: myapp://home
      Home: 'home',

      // 파라미터가 있는 매핑: myapp://product/42
      ProductDetail: {
        path: 'product/:productId',
        parse: {
          productId: (id: string) => parseInt(id, 10), // 문자열 → 숫자
        },
        stringify: {
          productId: (id: number) => id.toString(),     // 숫자 → 문자열
        },
      },

      // 쿼리 파라미터: myapp://search?q=react&category=tech
      Search: {
        path: 'search',
        // 쿼리 파라미터는 자동으로 route.params에 매핑됨
      },

      // 유저 프로필: myapp://user/john123
      UserProfile: 'user/:userId',

      // 중첩 네비게이터 딥링크
      MainTabs: {
        screens: {
          FeedTab: {
            screens: {
              FeedMain: 'feed',
              FeedDetail: 'feed/:postId',
            },
          },
          ExploreTab: 'explore',
          NotificationsTab: 'notifications',
        },
      },

      // 인증이 필요없는 화면
      Login: 'login',
      Register: 'register',

      // 와일드카드: 매칭되지 않는 URL
      NotFound: '*',
    },
  },
};
```

### 1-2. Android 네이티브 딥링크 설정

React Navigation의 linking만으로는 부족하다. Android OS가 앱으로 URL을 전달하려면 `AndroidManifest.xml`에도 설정이 필요하다.

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<activity
  android:name=".MainActivity"
  android:launchMode="singleTask">

  <!-- 커스텀 스킴 (myapp://...) -->
  <intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="myapp" />
  </intent-filter>

  <!-- HTTPS 딥링크 (https://myapp.com/...) -->
  <intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="https" android:host="myapp.com" />
  </intent-filter>
</activity>
```

**`launchMode="singleTask"`가 중요한 이유**: 딥링크로 앱이 이미 실행 중일 때 새 Activity를 생성하지 않고 기존 Activity에서 처리하기 위해서다. Android의 `onNewIntent()`에서 처리하는 것과 같다.

### 1-3. 딥링크 테스트

```bash
# Android 에뮬레이터/기기에서 테스트
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://product/42" \
  com.myapp.packagename

# HTTPS 딥링크 테스트
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://myapp.com/product/42" \
  com.myapp.packagename

# 쿼리 파라미터 테스트
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://search?q=react&category=tech" \
  com.myapp.packagename

# 중첩 네비게이터 딥링크 테스트
adb shell am start -W -a android.intent.action.VIEW \
  -d "myapp://feed/123" \
  com.myapp.packagename
```

### 1-4. NavigationContainer에 적용

```tsx
import { NavigationContainer } from '@react-navigation/native';
import { linking } from './linking';

function App() {
  return (
    <NavigationContainer
      linking={linking}
      fallback={
        // 딥링크 해석 중 표시되는 로딩 화면
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <ActivityIndicator size="large" color="#6200EE" />
          <Text>로딩 중...</Text>
        </View>
      }
    >
      <RootStack.Navigator>
        {/* ... */}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

```exercise
type: categorize
question: "다음을 Android Navigation과 React Navigation으로 분류하세요"
categories: ["Android Navigation", "React Navigation"]
items:
  - text: "NavGraph"
    category: "Android Navigation"
  - text: "Navigator"
    category: "React Navigation"
  - text: "SafeArgs"
    category: "Android Navigation"
  - text: "route.params"
    category: "React Navigation"
  - text: "NavController"
    category: "Android Navigation"
  - text: "useNavigation()"
    category: "React Navigation"
xp: 6
```

```javascript [playground]
// 🧪 딥링크 URL 파싱 실습

// React Navigation에서 딥링크 URL을 화면+파라미터로 변환하는 로직
function parsePath(url, config) {
  // URL에서 경로 추출
  const path = url.replace(/^.*:\/\//, '').replace(/\?.*$/, '');
  const queryString = url.includes('?') ? url.split('?')[1] : '';
  const segments = path.split('/').filter(Boolean);

  // 쿼리 파라미터 파싱
  const params = {};
  if (queryString) {
    queryString.split('&').forEach(pair => {
      const [key, value] = pair.split('=');
      params[decodeURIComponent(key)] = decodeURIComponent(value || '');
    });
  }

  // config에서 매칭되는 화면 찾기
  for (const [screen, pattern] of Object.entries(config)) {
    const patternSegments = pattern.split('/').filter(Boolean);
    if (patternSegments.length !== segments.length) continue;

    const routeParams = {};
    let match = true;

    for (let i = 0; i < patternSegments.length; i++) {
      if (patternSegments[i].startsWith(':')) {
        routeParams[patternSegments[i].slice(1)] = segments[i];
      } else if (patternSegments[i] !== segments[i]) {
        match = false;
        break;
      }
    }

    if (match) {
      return { screen, params: { ...routeParams, ...params } };
    }
  }
  return null;
}

// 딥링크 설정 (React Navigation linking config와 동일한 구조)
const linkingConfig = {
  'Home': '',
  'ProductList': 'products',
  'ProductDetail': 'products/:id',
  'UserProfile': 'users/:userId',
  'Settings': 'settings',
};

// 테스트
const urls = [
  "myapp://products",
  "myapp://products/42?ref=home",
  "myapp://users/hong123",
  "https://myapp.com/settings",
];

urls.forEach(url => {
  const result = parsePath(url, linkingConfig);
  if (result) {
    console.log(`${url}\n  → 화면: ${result.screen}, 파라미터: ${JSON.stringify(result.params)}\n`);
  } else {
    console.log(`${url}\n  → 매칭 없음\n`);
  }
});
```

---

## 2. 네비게이션 상태 영속화

앱이 백그라운드에서 종료되었다가 다시 열릴 때 이전 네비게이션 상태를 복원하는 기능이다. Android에서 `savedInstanceState`로 Fragment 상태를 복원하는 것과 유사하다.

```tsx
import React, { useCallback, useEffect, useState } from 'react';
import { Linking } from 'react-native';
import { NavigationContainer, NavigationState } from '@react-navigation/native';
import AsyncStorage from '@react-native-async-storage/async-storage';

const NAVIGATION_STATE_KEY = 'NAVIGATION_STATE';

function App() {
  const [isReady, setIsReady] = useState(false);
  const [initialState, setInitialState] = useState<NavigationState | undefined>();

  // 앱 시작 시 저장된 상태 복원
  useEffect(() => {
    const restoreState = async () => {
      try {
        // 딥링크로 열렸으면 저장된 상태 무시
        const initialUrl = await Linking.getInitialURL();
        if (initialUrl != null) {
          setIsReady(true);
          return;
        }

        // AsyncStorage에서 저장된 상태 읽기
        const savedStateString = await AsyncStorage.getItem(NAVIGATION_STATE_KEY);
        if (savedStateString) {
          const state = JSON.parse(savedStateString);
          setInitialState(state);
        }
      } catch (e) {
        console.warn('Navigation state 복원 실패:', e);
      } finally {
        setIsReady(true);
      }
    };

    restoreState();
  }, []);

  // 상태 변경 시 저장
  const onStateChange = useCallback((state: NavigationState | undefined) => {
    if (state) {
      AsyncStorage.setItem(NAVIGATION_STATE_KEY, JSON.stringify(state));
    }
  }, []);

  if (!isReady) {
    // 스플래시 스크린 또는 로딩 표시
    return <SplashScreen />;
  }

  return (
    <NavigationContainer
      initialState={initialState}
      onStateChange={onStateChange}
    >
      <RootStack.Navigator>
        {/* ... */}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

**주의**: 프로덕션에서는 상태 복원을 개발 모드에서만 활성화하거나, 상태 버전 관리를 추가하는 것이 좋다. 앱 업데이트로 화면 구조가 변경되면 이전 상태가 호환되지 않을 수 있다.

---

## 3. 화면 추적과 애널리틱스

Android에서 Fragment의 `onResume()`에서 애널리틱스 이벤트를 보내는 것처럼, React Navigation에서도 화면 전환을 추적할 수 있다.

### 3-1. NavigationContainer의 onStateChange 활용

```tsx
import analytics from '@react-native-firebase/analytics';

function App() {
  const navigationRef = useNavigationContainerRef();
  const routeNameRef = React.useRef<string | undefined>();

  return (
    <NavigationContainer
      ref={navigationRef}
      onReady={() => {
        // 초기 화면 이름 저장
        routeNameRef.current = navigationRef.getCurrentRoute()?.name;
      }}
      onStateChange={async () => {
        const previousRouteName = routeNameRef.current;
        const currentRouteName = navigationRef.getCurrentRoute()?.name;

        if (previousRouteName !== currentRouteName && currentRouteName) {
          // 화면 전환 이벤트 전송
          await analytics().logScreenView({
            screen_name: currentRouteName,
            screen_class: currentRouteName,
          });

          console.log(`Screen: ${previousRouteName} → ${currentRouteName}`);
        }

        // 현재 화면 이름 저장
        routeNameRef.current = currentRouteName;
      }}
    >
      {/* ... */}
    </NavigationContainer>
  );
}
```

### 3-2. 개별 화면에서 focus 이벤트 활용

```tsx
import { useFocusEffect } from '@react-navigation/native';

function ProductDetailScreen() {
  const route = useRoute();

  // Android의 onResume()에 해당
  useFocusEffect(
    React.useCallback(() => {
      // 화면에 포커스가 올 때
      analytics().logEvent('view_product', {
        product_id: route.params.productId,
      });

      return () => {
        // 화면에서 포커스가 떠날 때 (onPause에 해당)
        analytics().logEvent('leave_product', {
          product_id: route.params.productId,
        });
      };
    }, [route.params.productId])
  );

  return <Text>상품 상세</Text>;
}
```

---

## 4. 인증 플로우 패턴

### 4-1. 완전한 인증 플로우 구현

```tsx
// stores/authStore.ts (Zustand 사용)
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

type AuthState = {
  token: string | null;
  user: { id: string; name: string; email: string } | null;
  isLoading: boolean;
  isLoggedIn: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  checkAuth: () => Promise<void>;
};

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      token: null,
      user: null,
      isLoading: true,
      isLoggedIn: false,

      login: async (email, password) => {
        try {
          // API 호출 (실제로는 Retrofit의 @POST에 해당)
          const response = await fetch('https://api.example.com/login', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ email, password }),
          });
          const data = await response.json();

          set({
            token: data.token,
            user: data.user,
            isLoggedIn: true,
          });
        } catch (error) {
          throw error;
        }
      },

      logout: () => {
        set({ token: null, user: null, isLoggedIn: false });
      },

      checkAuth: async () => {
        const { token } = get();
        if (!token) {
          set({ isLoading: false });
          return;
        }
        try {
          const response = await fetch('https://api.example.com/me', {
            headers: { Authorization: `Bearer ${token}` },
          });
          if (response.ok) {
            const user = await response.json();
            set({ user, isLoggedIn: true, isLoading: false });
          } else {
            set({ token: null, user: null, isLoggedIn: false, isLoading: false });
          }
        } catch {
          set({ isLoading: false });
        }
      },
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => AsyncStorage),
      partialize: (state) => ({ token: state.token }), // token만 영속화
    }
  )
);
```

```tsx
// App.tsx — 인증 기반 네비게이션
function AppNavigator() {
  const { isLoggedIn, isLoading, checkAuth } = useAuthStore();

  useEffect(() => {
    checkAuth();
  }, []);

  if (isLoading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" />
        <Text>인증 확인 중...</Text>
      </View>
    );
  }

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isLoggedIn ? (
        // 로그인 상태: 메인 앱
        <>
          <Stack.Screen name="MainTabs" component={MainTabs} />
          <Stack.Screen name="Settings" component={SettingsScreen}
            options={{ headerShown: true }} />
        </>
      ) : (
        // 비로그인 상태: 인증 플로우
        // animationTypeForReplace: 로그아웃 시 자연스러운 전환
        <>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
          <Stack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}
```

### 4-2. 로그인 화면

```tsx
function LoginScreen() {
  const navigation = useNavigation();
  const login = useAuthStore((state) => state.login);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    setLoading(true);
    setError('');
    try {
      await login(email, password);
      // 로그인 성공 시 자동으로 MainTabs로 전환됨
      // (isLoggedIn이 true가 되면서 조건부 렌더링)
      // navigate()를 호출할 필요 없음!
    } catch (e) {
      setError('로그인에 실패했습니다');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>로그인</Text>
      {error ? <Text style={styles.error}>{error}</Text> : null}
      <TextInput
        style={styles.input}
        placeholder="이메일"
        value={email}
        onChangeText={setEmail}
        autoCapitalize="none"
        keyboardType="email-address"
      />
      <TextInput
        style={styles.input}
        placeholder="비밀번호"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
      />
      <TouchableOpacity
        style={styles.button}
        onPress={handleLogin}
        disabled={loading}
      >
        <Text style={styles.buttonText}>
          {loading ? '로그인 중...' : '로그인'}
        </Text>
      </TouchableOpacity>
      <TouchableOpacity
        onPress={() => navigation.navigate('Register')}
      >
        <Text style={styles.linkText}>계정이 없으신가요? 회원가입</Text>
      </TouchableOpacity>
    </View>
  );
}
```

---

## 5. 네비게이션 이벤트

Android Fragment의 생명주기(onResume, onPause, onDestroy 등)에 대응하는 React Navigation 이벤트들이다.

### 5-1. focus / blur 이벤트

```tsx
// Android의 onResume / onPause에 해당
function ChatScreen() {
  const navigation = useNavigation();

  useEffect(() => {
    // focus: 화면이 보이게 될 때 (onResume)
    const unsubscribeFocus = navigation.addListener('focus', () => {
      console.log('Chat 화면 활성화');
      // WebSocket 연결, 타이머 시작 등
      connectWebSocket();
    });

    // blur: 화면이 안 보이게 될 때 (onPause)
    const unsubscribeBlur = navigation.addListener('blur', () => {
      console.log('Chat 화면 비활성화');
      // WebSocket 해제, 타이머 정지 등
      disconnectWebSocket();
    });

    return () => {
      unsubscribeFocus();
      unsubscribeBlur();
    };
  }, [navigation]);

  return <Text>채팅</Text>;
}
```

### 5-2. useFocusEffect 훅 (권장)

```tsx
import { useFocusEffect } from '@react-navigation/native';

function ChatScreen() {
  // useEffect + focus/blur 리스너 조합을 간편하게 대체
  useFocusEffect(
    React.useCallback(() => {
      // 화면 포커스 시 실행 (onResume)
      const connection = connectWebSocket();

      return () => {
        // 화면 블러 시 실행 (onPause)
        connection.close();
      };
    }, [])
  );

  return <Text>채팅</Text>;
}
```

### 5-3. useIsFocused 훅

```tsx
import { useIsFocused } from '@react-navigation/native';

function CameraScreen() {
  const isFocused = useIsFocused();

  // 화면이 보일 때만 카메라 활성화
  // Android의 Lifecycle.State.RESUMED 체크와 유사
  return (
    <View style={{ flex: 1 }}>
      {isFocused ? (
        <Camera style={{ flex: 1 }} />
      ) : (
        <View style={{ flex: 1, backgroundColor: '#000' }} />
      )}
    </View>
  );
}
```

### 5-4. beforeRemove 이벤트

```tsx
// 화면이 스택에서 제거되기 직전에 발생
// Android의 Fragment onDestroy + addOnBackPressedCallback 조합
function FormScreen() {
  const navigation = useNavigation();
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false);

  useEffect(() => {
    const unsubscribe = navigation.addListener('beforeRemove', (e) => {
      if (!hasUnsavedChanges) return; // 변경사항 없으면 통과

      // 뒤로가기 방지
      e.preventDefault();

      // 확인 다이얼로그 표시
      Alert.alert(
        '저장하지 않은 변경',
        '변경사항을 저장하지 않고 나가시겠습니까?',
        [
          { text: '머무르기', style: 'cancel' },
          {
            text: '나가기',
            style: 'destructive',
            onPress: () => navigation.dispatch(e.data.action),
          },
        ]
      );
    });

    return unsubscribe;
  }, [navigation, hasUnsavedChanges]);

  return (
    <TextInput
      placeholder="내용을 입력하세요"
      onChangeText={() => setHasUnsavedChanges(true)}
    />
  );
}
```

---

## 6. 뒤로가기 방지

Android에서 `OnBackPressedCallback`으로 뒤로가기를 가로채는 것과 동일한 패턴이다.

### 6-1. Android OnBackPressedCallback 비교

```kotlin
// Android: OnBackPressedCallback
class EditFragment : Fragment() {
    private var hasUnsavedChanges = false

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        requireActivity().onBackPressedDispatcher.addCallback(
            viewLifecycleOwner,
            object : OnBackPressedCallback(true) {
                override fun handleOnBackPressed() {
                    if (hasUnsavedChanges) {
                        showExitConfirmDialog()
                    } else {
                        isEnabled = false
                        requireActivity().onBackPressedDispatcher.onBackPressed()
                    }
                }
            }
        )
    }
}
```

```tsx
// React Navigation: beforeRemove 이벤트
function EditScreen() {
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false);
  const navigation = useNavigation();

  // usePreventRemove 훅 사용 (v7)
  // beforeRemove 리스너를 더 간편하게 사용
  React.useEffect(() => {
    const unsubscribe = navigation.addListener('beforeRemove', (e) => {
      if (!hasUnsavedChanges) return;
      e.preventDefault();

      Alert.alert(
        '변경사항이 있습니다',
        '저장하지 않은 변경사항이 있습니다. 정말 나가시겠습니까?',
        [
          { text: '취소', style: 'cancel' },
          {
            text: '저장하고 나가기',
            onPress: async () => {
              await saveChanges();
              navigation.dispatch(e.data.action);
            },
          },
          {
            text: '저장 안 함',
            style: 'destructive',
            onPress: () => navigation.dispatch(e.data.action),
          },
        ]
      );
    });

    return unsubscribe;
  }, [navigation, hasUnsavedChanges]);

  return (
    <View style={{ flex: 1, padding: 20 }}>
      <TextInput
        style={{
          borderWidth: 1,
          borderColor: '#ccc',
          padding: 12,
          borderRadius: 8,
          minHeight: 200,
          textAlignVertical: 'top',
        }}
        multiline
        placeholder="내용을 입력하세요..."
        onChangeText={(text) => {
          setHasUnsavedChanges(text.length > 0);
        }}
      />
    </View>
  );
}
```

---

## 7. 커스텀 전환과 Shared Element Transition

### 7-1. NativeStack 기본 전환 옵션

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    animation: 'slide_from_right',
    animationDuration: 350,
    // 커스텀 전환은 NativeStack에서는 제한적
    // 고도의 커스터마이징이 필요하면 JS Stack 사용
  }}
/>
```

### 7-2. 화면별 동적 전환

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={({ route }) => ({
    // 파라미터에 따라 다른 전환 적용
    animation: route.params.fromModal ? 'slide_from_bottom' : 'slide_from_right',
  })}
/>
```

### 7-3. Shared Element Transition (react-native-shared-element)

Android의 `SharedElementTransition`에 해당한다. 두 화면 간에 공유 요소가 자연스럽게 전환되는 효과다.

```bash
npm install react-native-shared-element react-navigation-shared-element
```

```tsx
import { SharedElement } from 'react-native-shared-element';
import { createSharedElementStackNavigator } from 'react-navigation-shared-element';

const Stack = createSharedElementStackNavigator();

// 목록 화면
function ListScreen() {
  const navigation = useNavigation();

  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => navigation.push('Detail', { item })}>
          {/* SharedElement로 감싸기 — Android의 transitionName과 유사 */}
          <SharedElement id={`item.${item.id}.photo`}>
            <Image source={{ uri: item.imageUrl }} style={styles.thumbnail} />
          </SharedElement>
          <SharedElement id={`item.${item.id}.title`}>
            <Text style={styles.itemTitle}>{item.title}</Text>
          </SharedElement>
        </TouchableOpacity>
      )}
    />
  );
}

// 상세 화면
function DetailScreen({ route }) {
  const { item } = route.params;

  return (
    <View>
      {/* 동일한 id로 SharedElement 매칭 */}
      <SharedElement id={`item.${item.id}.photo`}>
        <Image source={{ uri: item.imageUrl }} style={styles.heroImage} />
      </SharedElement>
      <SharedElement id={`item.${item.id}.title`}>
        <Text style={styles.detailTitle}>{item.title}</Text>
      </SharedElement>
      <Text>{item.description}</Text>
    </View>
  );
}

// sharedElements 설정
DetailScreen.sharedElements = (route) => {
  const { item } = route.params;
  return [
    { id: `item.${item.id}.photo`, animation: 'move', resize: 'clip' },
    { id: `item.${item.id}.title`, animation: 'fade', resize: 'clip' },
  ];
};

// Navigator
function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="List" component={ListScreen} />
        <Stack.Screen name="Detail" component={DetailScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

---

## 8. 컴포넌트 바깥에서 네비게이션

Android에서 Service나 BroadcastReceiver에서 Activity를 시작하는 것처럼, React 컴포넌트가 아닌 곳(API 인터셉터, 푸시 알림 핸들러 등)에서 네비게이션을 수행해야 할 때가 있다.

### 8-1. navigationRef 설정

```tsx
// navigation/navigationRef.ts
import { createNavigationContainerRef } from '@react-navigation/native';

export const navigationRef = createNavigationContainerRef<RootStackParamList>();

// 타입 안전한 네비게이션 함수
export function navigate(name: keyof RootStackParamList, params?: any) {
  if (navigationRef.isReady()) {
    navigationRef.navigate(name, params);
  } else {
    // 네비게이션 준비 안됨 — 대기열에 추가하거나 무시
    console.warn('Navigation is not ready yet');
  }
}

export function goBack() {
  if (navigationRef.isReady() && navigationRef.canGoBack()) {
    navigationRef.goBack();
  }
}

export function resetToLogin() {
  if (navigationRef.isReady()) {
    navigationRef.reset({
      index: 0,
      routes: [{ name: 'Login' }],
    });
  }
}
```

```tsx
// App.tsx
import { navigationRef } from './navigation/navigationRef';

function App() {
  return (
    <NavigationContainer ref={navigationRef}>
      {/* ... */}
    </NavigationContainer>
  );
}
```

### 8-2. API 인터셉터에서 사용

```tsx
// api/client.ts — Axios 인터셉터에서 401 처리
import axios from 'axios';
import { resetToLogin } from '../navigation/navigationRef';
import { useAuthStore } from '../stores/authStore';

const apiClient = axios.create({
  baseURL: 'https://api.example.com',
});

// 요청 인터셉터 (Android의 OkHttp Interceptor와 동일)
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 응답 인터셉터
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 토큰 만료 → 로그아웃 → 로그인 화면으로 이동
      useAuthStore.getState().logout();
      resetToLogin(); // 컴포넌트 바깥에서 네비게이션!
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### 8-3. 푸시 알림 핸들러에서 사용

```tsx
// services/pushNotification.ts
import messaging from '@react-native-firebase/messaging';
import { navigate } from '../navigation/navigationRef';

// 백그라운드 메시지 핸들러 (컴포넌트 바깥)
messaging().setBackgroundMessageHandler(async (remoteMessage) => {
  console.log('Background message:', remoteMessage);
});

// 알림 클릭 핸들러
messaging().onNotificationOpenedApp((remoteMessage) => {
  const { type, id } = remoteMessage.data || {};

  switch (type) {
    case 'chat':
      navigate('ChatDetail', { chatId: id });
      break;
    case 'post':
      navigate('PostDetail', { postId: parseInt(id, 10) });
      break;
    case 'user':
      navigate('UserProfile', { userId: id });
      break;
  }
});

// 앱이 종료된 상태에서 알림으로 열렸을 때
messaging()
  .getInitialNotification()
  .then((remoteMessage) => {
    if (remoteMessage) {
      const { type, id } = remoteMessage.data || {};
      // 초기 라우트 설정 또는 navigate
    }
  });
```

---

## 9. URL 기반 네비게이션 (Linking Configuration)

### 9-1. 고급 Linking 설정

```tsx
const linking: LinkingOptions<RootStackParamList> = {
  prefixes: ['myapp://', 'https://myapp.com'],

  // 커스텀 URL 파싱 (선택사항)
  getInitialURL: async () => {
    // 푸시 알림에서 온 URL 확인
    const message = await messaging().getInitialNotification();
    if (message?.data?.url) {
      return message.data.url as string;
    }

    // 기본 딥링크 URL 확인
    const url = await Linking.getInitialURL();
    return url;
  },

  // URL 변경 구독 (앱이 이미 실행 중일 때)
  subscribe: (listener) => {
    // 일반 딥링크
    const linkingSubscription = Linking.addEventListener('url', ({ url }) => {
      listener(url);
    });

    // 푸시 알림으로 인한 URL
    const unsubscribeNotification = messaging().onNotificationOpenedApp(
      (message) => {
        const url = message.data?.url;
        if (url) {
          listener(url as string);
        }
      }
    );

    return () => {
      linkingSubscription.remove();
      unsubscribeNotification();
    };
  },

  config: {
    screens: {
      MainTabs: {
        screens: {
          HomeTab: 'home',
          SearchTab: 'search',
        },
      },
      ProductDetail: 'product/:id',
      NotFound: '*',
    },
  },
};
```

### 9-2. 404 Not Found 화면

```tsx
function NotFoundScreen() {
  const navigation = useNavigation();

  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text style={{ fontSize: 48, marginBottom: 20 }}>404</Text>
      <Text style={{ fontSize: 18, color: '#666', marginBottom: 30 }}>
        페이지를 찾을 수 없습니다
      </Text>
      <Button
        title="홈으로 돌아가기"
        onPress={() => navigation.navigate('MainTabs')}
      />
    </View>
  );
}

// Navigator에 등록
<Stack.Screen
  name="NotFound"
  component={NotFoundScreen}
  options={{ title: '페이지 없음' }}
/>
```

---

## 10. 성능 최적화

### 10-1. React.lazy와 동적 import

```tsx
// 화면을 필요할 때만 로드 (Android의 Dynamic Feature Module과 유사한 개념)
const SettingsScreen = React.lazy(() => import('../screens/SettingsScreen'));
const HeavyChartScreen = React.lazy(() => import('../screens/HeavyChartScreen'));

// Suspense로 로딩 표시
<Stack.Screen name="Settings">
  {() => (
    <React.Suspense
      fallback={
        <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
          <ActivityIndicator size="large" />
        </View>
      }
    >
      <SettingsScreen />
    </React.Suspense>
  )}
</Stack.Screen>
```

### 10-2. Tab Navigator의 lazy 로딩

```tsx
<Tab.Navigator
  screenOptions={{
    // 탭을 처음 선택할 때만 화면 렌더링 (기본값: true)
    lazy: true,

    // 레이지 로딩 중 표시할 플레이스홀더
    lazyPlaceholder: () => (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <ActivityIndicator size="large" color="#6200EE" />
      </View>
    ),
  }}
>
```

### 10-3. 불필요한 리렌더링 방지

```tsx
// 비추천: navigation/route 객체를 직접 props로 전달
// 이 객체들은 매번 새로운 참조를 가짐
function ParentScreen({ navigation, route }) {
  return <ChildComponent navigation={navigation} />; // 매번 리렌더링
}

// 추천: 필요한 값만 전달하거나 훅 사용
function ParentScreen() {
  return <ChildComponent />;
}

function ChildComponent() {
  const navigation = useNavigation(); // 자체적으로 훅 사용
}
```

### 10-4. react-native-screens 최적화

```tsx
// react-native-screens를 활성화하면 비활성 화면이 네이티브 레벨에서 최적화됨
// (기본적으로 활성화되어 있음)
import { enableScreens } from 'react-native-screens';
enableScreens(true); // index.js에서 호출

// 특정 화면에서 detach 방지 (예: 비디오 재생 화면)
<Stack.Screen
  name="Video"
  component={VideoScreen}
  options={{
    freezeOnBlur: false, // 이 화면은 블러 시에도 활성 상태 유지
  }}
/>
```

### 10-5. 큰 목록 화면의 메모리 관리

```tsx
function HeavyListScreen() {
  const [data, setData] = useState<Item[]>([]);

  // useFocusEffect로 화면 포커스/블러 시 데이터 관리
  useFocusEffect(
    React.useCallback(() => {
      // 포커스 시 데이터 로드
      loadData().then(setData);

      return () => {
        // 블러 시 대용량 데이터 해제 (메모리 최적화)
        // 주의: 필요한 경우에만 사용
        // setData([]);
      };
    }, [])
  );

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      // FlatList 자체 최적화 옵션
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={5}
      initialNumToRender={10}
      getItemLayout={(data, index) => ({
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}
    />
  );
}
```

---

## 11. 프로덕션 레디 네비게이션 아키텍처

실제 프로덕션 앱에서 사용할 수 있는 전체 네비게이션 아키텍처 예제다.

### 11-1. 폴더 구조

```
src/
├── navigation/
│   ├── types.ts              # 모든 ParamList 타입
│   ├── navigationRef.ts      # 외부 네비게이션 ref
│   ├── linking.ts            # 딥링크 설정
│   ├── RootNavigator.tsx     # 최상위 네비게이터 (인증 분기)
│   ├── AppNavigator.tsx      # 로그인 후 메인 네비게이터
│   ├── AuthNavigator.tsx     # 로그인 전 인증 네비게이터
│   └── MainTabNavigator.tsx  # 바텀 탭 네비게이터
├── screens/
│   ├── auth/
│   │   ├── LoginScreen.tsx
│   │   └── RegisterScreen.tsx
│   ├── home/
│   │   ├── HomeScreen.tsx
│   │   └── HomeDetailScreen.tsx
│   ├── search/
│   │   └── SearchScreen.tsx
│   └── profile/
│       ├── ProfileScreen.tsx
│       └── EditProfileScreen.tsx
└── App.tsx
```

### 11-2. 타입 파일

```tsx
// navigation/types.ts
export type AuthStackParamList = {
  Login: undefined;
  Register: undefined;
  ForgotPassword: { email?: string };
};

export type MainTabParamList = {
  HomeTab: undefined;
  SearchTab: undefined;
  NotificationsTab: undefined;
  ProfileTab: undefined;
};

export type AppStackParamList = {
  MainTabs: undefined;
  PostDetail: { postId: number };
  UserProfile: { userId: string };
  CreatePost: undefined;
  Settings: undefined;
  EditProfile: undefined;
};

export type RootStackParamList = {
  Auth: undefined;
  App: undefined;
  Onboarding: undefined;
};

// 글로벌 타입
declare global {
  namespace ReactNavigation {
    interface RootParamList extends AppStackParamList {}
  }
}
```

### 11-3. 최상위 네비게이터

```tsx
// navigation/RootNavigator.tsx
import React, { useEffect } from 'react';
import { ActivityIndicator, View } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { navigationRef } from './navigationRef';
import { linking } from './linking';
import { useAuthStore } from '../stores/authStore';
import AuthNavigator from './AuthNavigator';
import AppNavigator from './AppNavigator';

const RootStack = createNativeStackNavigator();

export default function RootNavigator() {
  const { isLoggedIn, isLoading, checkAuth } = useAuthStore();

  useEffect(() => {
    checkAuth();
  }, []);

  if (isLoading) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center', backgroundColor: '#1a1a2e' }}>
        <ActivityIndicator size="large" color="#e94560" />
      </View>
    );
  }

  return (
    <NavigationContainer
      ref={navigationRef}
      linking={linking}
      onStateChange={(state) => {
        // 화면 추적
        const currentRoute = navigationRef.getCurrentRoute();
        if (currentRoute) {
          console.log('Current screen:', currentRoute.name);
        }
      }}
    >
      <RootStack.Navigator screenOptions={{ headerShown: false }}>
        {isLoggedIn ? (
          <RootStack.Screen name="App" component={AppNavigator} />
        ) : (
          <RootStack.Screen
            name="Auth"
            component={AuthNavigator}
            options={{ animationTypeForReplace: 'pop' }}
          />
        )}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### 11-4. App.tsx

```tsx
// App.tsx
import React from 'react';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import RootNavigator from './navigation/RootNavigator';

const queryClient = new QueryClient();

export default function App() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <SafeAreaProvider>
        <QueryClientProvider client={queryClient}>
          <RootNavigator />
        </QueryClientProvider>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}
```

---

## 요약: 고급 패턴 비교표

```
Android 패턴                          React Navigation
─────────────────────────────────    ─────────────────────────────
intent-filter (딥링크)                linking config + AndroidManifest
savedInstanceState 복원               initialState + onStateChange
onResume / onPause                   focus / blur 이벤트 (useFocusEffect)
OnBackPressedCallback                beforeRemove 이벤트
SharedElementTransition              react-native-shared-element
startActivity (Service에서)           navigationRef.navigate()
OkHttp Interceptor → 화면 이동        Axios interceptor + navigationRef
FirebaseMessagingService             messaging().onNotificationOpenedApp
Dynamic Feature Module               React.lazy + dynamic import
Fragment lifecycle                   focus/blur/beforeRemove 이벤트
```

이것으로 React Navigation의 기본부터 고급까지 모든 내용을 다루었다. 다음 phase에서는 상태 관리와 네트워킹을 학습한다.
