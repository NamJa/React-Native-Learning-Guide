# 심화 학습 토픽 — React Native 전문가로의 여정

## 목차
1. [Expo Router: 파일 기반 라우팅](#1-expo-router-파일-기반-라우팅)
2. [React Native for Web](#2-react-native-for-web)
3. [모노레포 설정](#3-모노레포-설정)
4. [접근성 (Accessibility)](#4-접근성-accessibility)
5. [다국어 지원 (i18n)](#5-다국어-지원-i18n)
6. [보안 (Security)](#6-보안-security)
7. [오프라인 퍼스트 아키텍처](#7-오프라인-퍼스트-아키텍처)
8. [푸시 알림](#8-푸시-알림)
9. [백그라운드 작업](#9-백그라운드-작업)

---

## 1. Expo Router: 파일 기반 라우팅

Expo Router는 Next.js의 파일 기반 라우팅을 React Native에 적용한 것이다. 폴더 구조만으로 네비게이션이 자동 생성된다.

> **Android 비교**: Navigation Component에서 nav_graph.xml에 수동으로 경로를 정의하는 대신,
> 파일 시스템이 곧 라우팅이 된다. Android Navigation Component의 auto-generated Directions와 유사한 개념이다.

### 설치 및 설정

```bash
# Expo 프로젝트에서
npx expo install expo-router expo-linking expo-constants expo-status-bar
```

```json
// package.json
{
  "main": "expo-router/entry"
}
```

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "web": {
      "bundler": "metro"
    }
  }
}
```

### 파일 구조 = 라우팅

```
app/                              URL 경로
├── _layout.tsx                   → 루트 레이아웃 (모든 화면 공통)
├── index.tsx                     → /  (홈 화면)
├── about.tsx                     → /about
├── (tabs)/                       → 탭 네비게이터 그룹
│   ├── _layout.tsx               → 탭 레이아웃 정의
│   ├── index.tsx                 → / (홈 탭)
│   ├── search.tsx                → /search (검색 탭)
│   └── profile.tsx               → /profile (프로필 탭)
├── (auth)/                       → 인증 관련 그룹 (URL에 미포함)
│   ├── _layout.tsx               → 인증 레이아웃
│   ├── login.tsx                 → /login
│   └── register.tsx              → /register
├── article/
│   ├── [id].tsx                  → /article/123  (동적 라우트)
│   └── [id]/
│       └── comments.tsx          → /article/123/comments
├── settings/
│   ├── _layout.tsx               → 설정 레이아웃 (네스트 네비게이터)
│   ├── index.tsx                 → /settings
│   ├── notifications.tsx         → /settings/notifications
│   └── privacy.tsx               → /settings/privacy
└── +not-found.tsx                → 404 페이지
```

### 레이아웃 파일

```typescript
// app/_layout.tsx — 루트 레이아웃 (Android의 MainActivity에 해당)
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="(auth)" options={{ headerShown: false }} />
      <Stack.Screen
        name="article/[id]"
        options={{ title: '기사 상세' }}
      />
    </Stack>
  );
}
```

```typescript
// app/(tabs)/_layout.tsx — 탭 레이아웃
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#007AFF' }}>
      <Tabs.Screen
        name="index"
        options={{
          title: '홈',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: '검색',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: '프로필',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

### 동적 라우트 (Dynamic Route)

```typescript
// app/article/[id].tsx — 동적 파라미터
// Android Navigation의 SafeArgs와 유사: ArticleDetailFragmentArgs
import { useLocalSearchParams, router } from 'expo-router';

export default function ArticleDetail() {
  // URL에서 파라미터 추출 (SafeArgs의 args.articleId에 해당)
  const { id } = useLocalSearchParams<{ id: string }>();

  return (
    <View style={{ flex: 1, padding: 16 }}>
      <Text>기사 ID: {id}</Text>
      <Button
        title="댓글 보기"
        onPress={() => router.push(`/article/${id}/comments`)}
      />
    </View>
  );
}
```

### 네비게이션 명령

```typescript
import { router, Link } from 'expo-router';

// 프로그래밍 방식 네비게이션
router.push('/article/123');           // 새 화면 추가 (navigate)
router.replace('/home');               // 현재 화면 교체 (replace)
router.back();                         // 뒤로 가기
router.canGoBack();                    // 뒤로 갈 수 있는지 확인
router.dismiss();                      // 모달 닫기
router.dismissAll();                   // 모든 모달 닫기

// 선언적 네비게이션 (Link 컴포넌트)
function HomeScreen() {
  return (
    <View>
      <Link href="/article/123">
        <Text>기사 보기</Text>
      </Link>

      <Link href="/settings" asChild>
        <TouchableOpacity>
          <Text>설정</Text>
        </TouchableOpacity>
      </Link>
    </View>
  );
}
```

### 전체 앱 예시

```typescript
// app/(tabs)/index.tsx — 홈 화면
import { View, FlatList, TouchableOpacity, Text } from 'react-native';
import { router } from 'expo-router';

export default function HomeScreen() {
  const articles = useArticles();

  return (
    <FlatList
      data={articles.data}
      renderItem={({ item }) => (
        <TouchableOpacity
          onPress={() => router.push(`/article/${item.id}`)}
        >
          <Text>{item.title}</Text>
        </TouchableOpacity>
      )}
    />
  );
}
```

---

## 2. React Native for Web

### react-strict-dom (Meta의 차세대 솔루션)

```typescript
// react-strict-dom은 React Native와 Web에서 동일한 API를 사용하여
// 단일 코드베이스로 모바일 + 웹을 지원하는 Meta의 공식 프로젝트

// 핵심 개념:
// - CSS 기반 스타일링 (StyleX)
// - 웹과 네이티브에서 동일하게 동작하는 HTML-like 컴포넌트
// - React Native의 View, Text 대신 html.div, html.span 사용

import { css, html } from 'react-strict-dom';

const styles = css.create({
  container: {
    display: 'flex',
    flexDirection: 'column',
    padding: 16,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
  },
});

function MyComponent() {
  return (
    <html.div style={styles.container}>
      <html.span style={styles.title}>제목</html.span>
    </html.div>
  );
}
// 이 코드가 Android, iOS, Web에서 모두 동작함
```

### 플랫폼별 코드 분리 전략

```typescript
// 방법 1: Platform.select
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    padding: Platform.select({
      ios: 20,
      android: 16,
      web: 24,
    }),
  },
});

// 방법 2: 플랫폼별 파일
// Button.tsx        → 공통
// Button.ios.tsx    → iOS 전용
// Button.android.tsx → Android 전용
// Button.web.tsx    → 웹 전용
// Metro 번들러가 자동으로 플랫폼에 맞는 파일을 선택

// 방법 3: Platform.OS로 조건부 렌더링
function DatePicker() {
  if (Platform.OS === 'web') {
    return <input type="date" />;
  }
  return <NativeDatePicker />;
}
```

---

## 3. 모노레포 설정

### Turborepo로 React Native 모노레포

```
[모노레포 구조]

my-project/
├── apps/
│   ├── mobile/          ← React Native 앱
│   │   ├── app/         ← Expo Router
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/             ← Next.js 웹 앱
│       ├── pages/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── ui/              ← 공유 UI 컴포넌트
│   │   ├── src/
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   └── index.ts
│   │   └── package.json
│   ├── api-client/      ← 공유 API 클라이언트
│   │   ├── src/
│   │   └── package.json
│   └── types/           ← 공유 타입 정의
│       ├── src/
│       └── package.json
├── turbo.json
├── package.json
└── pnpm-workspace.yaml
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".expo/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {},
    "test": {
      "dependsOn": ["^build"]
    }
  }
}
```

```yaml
# pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"
```

```json
// packages/ui/package.json
{
  "name": "@myapp/ui",
  "version": "0.0.1",
  "main": "src/index.ts",
  "dependencies": {
    "react": "*",
    "react-native": "*"
  }
}
```

```typescript
// packages/ui/src/Button.tsx — 모바일/웹에서 공유
import { TouchableOpacity, Text, StyleSheet } from 'react-native';

interface ButtonProps {
  title: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary';
}

export function Button({ title, onPress, variant = 'primary' }: ButtonProps) {
  return (
    <TouchableOpacity
      style={[styles.button, styles[variant]]}
      onPress={onPress}
    >
      <Text style={[styles.text, styles[`${variant}Text`]]}>{title}</Text>
    </TouchableOpacity>
  );
}

// apps/mobile에서 사용:
// import { Button } from '@myapp/ui';
```

---

## 4. 접근성 (Accessibility)

Android에서 `contentDescription`, `importantForAccessibility`를 사용하는 것처럼, React Native에서도 접근성 속성을 설정한다.

### 기본 접근성 속성

```typescript
// Android contentDescription → accessibilityLabel
<TouchableOpacity
  accessible={true}                           // 접근성 요소로 인식
  accessibilityLabel="장바구니에 추가"          // TalkBack이 읽는 텍스트
  accessibilityHint="이 상품을 장바구니에 추가합니다"  // 추가 설명
  accessibilityRole="button"                  // 역할 (버튼, 링크, 이미지 등)
  accessibilityState={{
    disabled: false,                          // 비활성화 상태
    selected: false,                          // 선택 상태
    checked: undefined,                       // 체크박스 상태 (true/false/'mixed')
    busy: false,                              // 로딩 중
    expanded: undefined,                      // 확장 가능한 요소의 상태
  }}
  accessibilityValue={{
    min: 0,                                   // 슬라이더 등의 최소값
    max: 100,                                 // 최대값
    now: 50,                                  // 현재값
    text: '50%',                              // 값의 텍스트 표현
  }}
>
  <Image source={cartIcon} />
  <Text>추가</Text>
</TouchableOpacity>
```

### 접근성 역할 (Role)

```typescript
// Android의 className / Compose의 semantics에 해당

accessibilityRole 옵션:
- "none"          → 특별한 역할 없음
- "button"        → 버튼 (Android Button)
- "link"          → 링크
- "search"        → 검색 필드
- "image"         → 이미지 (ImageView)
- "text"          → 텍스트 (TextView)
- "adjustable"    → 슬라이더 (SeekBar)
- "header"        → 섹션 헤더
- "summary"       → 요약 정보
- "alert"         → 알림 메시지
- "checkbox"      → 체크박스
- "switch"        → 스위치 (Switch)
- "progressbar"   → 프로그레스 바
- "tab"           → 탭
- "tabbar"        → 탭 바
- "menu"          → 메뉴
- "menuitem"      → 메뉴 아이템
```

### 실전 접근성 예제

```typescript
// 접근성이 잘 적용된 카드 컴포넌트
function ArticleCard({ article }: { article: Article }) {
  return (
    <TouchableOpacity
      accessible={true}
      accessibilityLabel={`${article.title}, ${article.author}의 기사`}
      accessibilityHint="기사 상세 화면으로 이동합니다"
      accessibilityRole="button"
      onPress={() => router.push(`/article/${article.id}`)}
    >
      <Image
        source={{ uri: article.imageUrl }}
        accessibilityLabel={`${article.title} 대표 이미지`}
        accessibilityRole="image"
      />
      <View>
        <Text
          accessibilityRole="header"
          style={styles.title}
        >
          {article.title}
        </Text>
        <Text accessibilityLabel={`작성자: ${article.author}`}>
          {article.author}
        </Text>
        <Text accessibilityLabel={`${article.readTime}분 소요`}>
          {article.readTime}분 읽기
        </Text>
      </View>
    </TouchableOpacity>
  );
}

// 접근성 좋은 폼
function AccessibleForm() {
  const [email, setEmail] = useState('');
  const [error, setError] = useState('');

  return (
    <View>
      <Text nativeID="email-label">이메일</Text>
      <TextInput
        value={email}
        onChangeText={setEmail}
        accessibilityLabel="이메일 입력"
        accessibilityLabelledBy="email-label"  // 라벨과 연결
        accessibilityErrorMessage={error || undefined}  // 에러 메시지 연결
        keyboardType="email-address"
        autoComplete="email"
      />
      {error && (
        <Text
          accessibilityRole="alert"  // 스크린 리더가 즉시 읽음
          accessibilityLiveRegion="assertive"
          style={styles.error}
        >
          {error}
        </Text>
      )}
    </View>
  );
}
```

### 스크린 리더 테스트

```bash
# Android TalkBack 테스트
# 1. Settings > Accessibility > TalkBack 활성화
# 2. 앱에서 스와이프하여 요소 탐색
# 3. 각 요소가 올바르게 읽히는지 확인

# adb로 TalkBack 활성화/비활성화
adb shell settings put secure enabled_accessibility_services com.google.android.marvin.talkback/com.google.android.marvin.talkback.TalkBackService
adb shell settings put secure accessibility_enabled 1
```

---

## 5. 다국어 지원 (i18n)

### RTL (Right-to-Left) 레이아웃 지원

```typescript
// 아랍어, 히브리어 등 RTL 언어 지원
import { I18nManager } from 'react-native';

// RTL 활성화/비활성화
function setRTL(isRTL: boolean) {
  I18nManager.forceRTL(isRTL);
  I18nManager.allowRTL(isRTL);
  // 앱 재시작 필요
}

// RTL을 고려한 스타일
const styles = StyleSheet.create({
  container: {
    // flexDirection: 'row'는 RTL에서 자동으로 오른쪽→왼쪽이 됨
    flexDirection: 'row',
    // 명시적으로 방향을 지정하려면:
    // flexDirection: I18nManager.isRTL ? 'row-reverse' : 'row',
  },
  text: {
    // textAlign: 'left'는 RTL에서도 왼쪽 정렬
    // 자동으로 방향을 따르려면:
    writingDirection: 'auto', // RTL에서 자동으로 오른쪽 정렬
  },
  icon: {
    // RTL에서 아이콘 뒤집기
    transform: [{ scaleX: I18nManager.isRTL ? -1 : 1 }],
  },
});
```

### 날짜/숫자 포맷팅

```typescript
// 로케일별 날짜/숫자 포맷팅
import { format } from 'date-fns';
import { ko, en, ar } from 'date-fns/locale';

const locales: Record<string, Locale> = { ko, en, ar };

function formatDate(date: Date, locale: string): string {
  return format(date, 'PPP', {
    locale: locales[locale] || ko,
  });
  // ko: "2026년 3월 9일"
  // en: "March 9, 2026"
  // ar: "٩ مارس ٢٠٢٦"
}

// 통화 포맷팅
function formatCurrency(amount: number, locale: string, currency: string): string {
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(amount);
  // ko, KRW: "₩1,000,000"
  // en, USD: "$1,000,000.00"
  // ja, JPY: "¥1,000,000"
}
```

---

## 6. 보안 (Security)

### 보안 저장소

```typescript
// 민감한 데이터 저장: expo-secure-store
// Android의 EncryptedSharedPreferences / Keystore에 해당
import * as SecureStore from 'expo-secure-store';

// 토큰 저장
async function saveToken(token: string) {
  await SecureStore.setItemAsync('authToken', token);
  // Android: Android Keystore System으로 암호화
  // iOS: Keychain Services로 암호화
}

// 토큰 읽기
async function getToken(): Promise<string | null> {
  return await SecureStore.getItemAsync('authToken');
}

// 토큰 삭제
async function deleteToken() {
  await SecureStore.deleteItemAsync('authToken');
}

// MMKV 암호화 (대안)
import { MMKV } from 'react-native-mmkv';

const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'your-encryption-key', // 하드코딩하지 말 것!
});
```

### SSL 핀닝

```typescript
// 네트워크 보안: SSL 핀닝
// Android의 network_security_config.xml + CertificatePinner에 해당

// react-native-ssl-pinning 사용
import { fetch as sslFetch } from 'react-native-ssl-pinning';

async function secureApiCall(url: string) {
  const response = await sslFetch(url, {
    method: 'GET',
    sslPinning: {
      certs: ['server-cert'], // res/raw/server-cert.cer
    },
    headers: {
      'Content-Type': 'application/json',
    },
  });

  return response.json();
}
```

### 코드 난독화

```groovy
// android/app/build.gradle — ProGuard/R8로 네이티브 코드 난독화
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources false
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"),
                         "proguard-rules.pro"
        }
    }
}

// JavaScript 코드는 Hermes 바이트코드로 컴파일되어
// 소스 코드 직접 노출은 방지됨 (하지만 완전한 난독화는 아님)
```

### 보안 체크리스트

```
[React Native 보안 체크리스트]

데이터 저장:
[ ] 토큰/비밀번호는 SecureStore 또는 MMKV(암호화)에 저장
[ ] 민감 데이터는 절대 AsyncStorage에 저장하지 않음
[ ] .env 파일은 .gitignore에 추가

네트워크:
[ ] HTTPS만 사용 (HTTP 금지)
[ ] SSL 핀닝 적용 (MITM 공격 방지)
[ ] API 키는 환경 변수로 관리
[ ] 서버에서 토큰 유효성 검증

코드:
[ ] console.log에 민감 정보 출력하지 않음
[ ] __DEV__ 가드로 디버그 코드 격리
[ ] ProGuard/R8 활성화
[ ] Hermes 바이트코드 컴파일 활성화

인증:
[ ] JWT 토큰 만료 시 자동 갱신
[ ] 리프레시 토큰 로테이션
[ ] 비활성 시간 후 자동 로그아웃
[ ] 생체 인증 (Fingerprint/FaceID) 지원
```

---

## 7. 오프라인 퍼스트 아키텍처

### 네트워크 상태 감지

```typescript
// Android의 ConnectivityManager에 해당
import NetInfo from '@react-native-community/netinfo';

function useNetworkStatus() {
  const [isConnected, setIsConnected] = useState(true);
  const [connectionType, setConnectionType] = useState<string>('unknown');

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener((state) => {
      setIsConnected(state.isConnected ?? true);
      setConnectionType(state.type); // wifi, cellular, none, etc.
    });

    return () => unsubscribe();
  }, []);

  return { isConnected, connectionType };
}

// 오프라인 배너 표시
function OfflineBanner() {
  const { isConnected } = useNetworkStatus();

  if (isConnected) return null;

  return (
    <View style={styles.banner}>
      <Text style={styles.bannerText}>
        인터넷에 연결되어 있지 않습니다. 일부 기능이 제한될 수 있습니다.
      </Text>
    </View>
  );
}
```

### TanStack Query + 오프라인 지원

```typescript
// TanStack Query는 기본적으로 오프라인 지원을 제공
import { QueryClient, onlineManager } from '@tanstack/react-query';
import NetInfo from '@react-native-community/netinfo';

// 네트워크 상태를 TanStack Query에 연결
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});

// 오프라인 시 mutation 큐잉
const queryClient = new QueryClient({
  defaultOptions: {
    mutations: {
      // 오프라인 시 mutation을 큐에 저장
      // 온라인 복귀 시 자동으로 재시도
      networkMode: 'offlineFirst',
    },
    queries: {
      networkMode: 'offlineFirst',
      staleTime: 1000 * 60 * 5,  // 5분간 캐시 유효
    },
  },
});
```

### 낙관적 UI 업데이트 (Optimistic Update)

```typescript
// 서버 응답을 기다리지 않고 UI를 먼저 업데이트
// Android에서도 Kotlin Flow + Repository 패턴으로 구현하는 패턴

function useToggleLike(articleId: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: () => api.post(`/articles/${articleId}/like`),

    // 낙관적 업데이트: 서버 요청 전에 UI를 먼저 변경
    onMutate: async () => {
      // 진행 중인 쿼리 취소 (충돌 방지)
      await queryClient.cancelQueries({ queryKey: ['article', articleId] });

      // 현재 데이터 스냅샷 저장
      const previousArticle = queryClient.getQueryData(['article', articleId]);

      // 캐시를 낙관적으로 업데이트
      queryClient.setQueryData(['article', articleId], (old: Article) => ({
        ...old,
        isLiked: !old.isLiked,
        likeCount: old.isLiked ? old.likeCount - 1 : old.likeCount + 1,
      }));

      return { previousArticle };
    },

    // 에러 시 롤백
    onError: (err, variables, context) => {
      queryClient.setQueryData(
        ['article', articleId],
        context?.previousArticle
      );
    },

    // 성공/실패 후 서버 데이터로 동기화
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['article', articleId] });
    },
  });
}
```

---

## 8. 푸시 알림

### expo-notifications 상세 설정

```typescript
// src/services/notifications.ts
import * as Notifications from 'expo-notifications';
import * as Device from 'expo-device';
import Constants from 'expo-constants';
import { Platform } from 'react-native';

// 포그라운드 알림 핸들러
Notifications.setNotificationHandler({
  handleNotification: async (notification) => {
    // 알림 내용에 따라 표시 여부 결정
    const data = notification.request.content.data;

    return {
      shouldShowAlert: true,
      shouldPlaySound: data.playSound !== false,
      shouldSetBadge: true,
    };
  },
});

// Android 알림 채널 설정 (NotificationChannel에 해당)
// Android 8.0+ (API 26+)에서 필수
async function setupNotificationChannels() {
  if (Platform.OS === 'android') {
    // 기본 채널
    await Notifications.setNotificationChannelAsync('default', {
      name: '기본 알림',
      importance: Notifications.AndroidImportance.HIGH,
      vibrationPattern: [0, 250, 250, 250],
      lightColor: '#007AFF',
      lockscreenVisibility: Notifications.AndroidNotificationVisibility.PUBLIC,
      sound: 'default',
    });

    // 채팅 채널
    await Notifications.setNotificationChannelAsync('chat', {
      name: '채팅 알림',
      importance: Notifications.AndroidImportance.MAX,
      vibrationPattern: [0, 100, 100, 100],
      sound: 'chat_notification.wav', // android/app/src/main/res/raw/
    });

    // 마케팅 채널
    await Notifications.setNotificationChannelAsync('marketing', {
      name: '마케팅/이벤트',
      importance: Notifications.AndroidImportance.DEFAULT,
      sound: null, // 소리 없음
    });
  }
}

// 푸시 토큰 등록
async function registerForPushNotifications(): Promise<string | undefined> {
  if (!Device.isDevice) {
    console.warn('에뮬레이터에서는 푸시 알림이 지원되지 않습니다');
    return undefined;
  }

  const { status: existingStatus } = await Notifications.getPermissionsAsync();
  let finalStatus = existingStatus;

  if (existingStatus !== 'granted') {
    const { status } = await Notifications.requestPermissionsAsync();
    finalStatus = status;
  }

  if (finalStatus !== 'granted') {
    console.warn('알림 권한이 거부되었습니다');
    return undefined;
  }

  const token = await Notifications.getExpoPushTokenAsync({
    projectId: Constants.expoConfig?.extra?.eas?.projectId,
  });

  return token.data;
}

// 알림 리스너 설정
function useNotificationListeners() {
  const lastNotificationResponse = Notifications.useLastNotificationResponse();

  useEffect(() => {
    if (lastNotificationResponse) {
      const data = lastNotificationResponse.notification.request.content.data;

      // 알림 탭 시 딥 링크 처리
      if (data.screen) {
        router.push(data.screen as string);
      }
    }
  }, [lastNotificationResponse]);
}
```

```typescript
// 로컬 알림 스케줄링 (AlarmManager + NotificationCompat에 해당)
async function scheduleReminder(title: string, body: string, date: Date) {
  const id = await Notifications.scheduleNotificationAsync({
    content: {
      title,
      body,
      data: { screen: '/reminders' },
      sound: 'default',
      // Android 전용
      channelId: 'default',
      color: '#007AFF',
      priority: Notifications.AndroidNotificationPriority.HIGH,
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.DATE,
      date,
    },
  });

  return id;
}

// 반복 알림
async function scheduleDailyReminder() {
  await Notifications.scheduleNotificationAsync({
    content: {
      title: '오늘의 뉴스',
      body: '오늘의 주요 뉴스를 확인하세요!',
    },
    trigger: {
      type: Notifications.SchedulableTriggerInputTypes.DAILY,
      hour: 9,
      minute: 0,
    },
  });
}

// 예약된 알림 취소
async function cancelNotification(id: string) {
  await Notifications.cancelScheduledNotificationAsync(id);
}

// 모든 예약 취소
async function cancelAllNotifications() {
  await Notifications.cancelAllScheduledNotificationsAsync();
}
```

---

## 9. 백그라운드 작업

### Headless JS (Android 전용)

```typescript
// Android의 Service에 해당
// 앱이 백그라운드에 있을 때도 JS 코드 실행 가능

// index.js
import { AppRegistry } from 'react-native';

// 백그라운드 작업 등록
AppRegistry.registerHeadlessTask('SyncTask', () => require('./SyncTask'));

// SyncTask.ts
export default async function syncTask(taskData: { userId: string }) {
  console.log('백그라운드 동기화 시작:', taskData.userId);

  try {
    // API 호출, 데이터 동기화 등
    await syncLocalData(taskData.userId);
    console.log('동기화 완료');
  } catch (error) {
    console.error('동기화 실패:', error);
  }
}
```

### expo-task-manager (WorkManager 대응)

```bash
npx expo install expo-task-manager expo-background-fetch
```

```typescript
// Android의 WorkManager에 해당
import * as TaskManager from 'expo-task-manager';
import * as BackgroundFetch from 'expo-background-fetch';

const BACKGROUND_SYNC_TASK = 'background-sync';

// 백그라운드 작업 정의
TaskManager.defineTask(BACKGROUND_SYNC_TASK, async () => {
  try {
    console.log('백그라운드 동기화 실행');
    const now = Date.now();

    // 서버에서 새 데이터 확인
    const response = await fetch('https://api.example.com/check-updates');
    const data = await response.json();

    if (data.hasUpdates) {
      // 로컬 데이터 업데이트
      await syncLocalData(data);

      // 로컬 알림 표시
      await Notifications.scheduleNotificationAsync({
        content: {
          title: '새 콘텐츠',
          body: `${data.count}개의 새 기사가 있습니다`,
        },
        trigger: null, // 즉시
      });
    }

    // BackgroundFetch.Result를 반환해야 함
    return data.hasUpdates
      ? BackgroundFetch.BackgroundFetchResult.NewData
      : BackgroundFetch.BackgroundFetchResult.NoData;
  } catch (error) {
    console.error('백그라운드 동기화 실패:', error);
    return BackgroundFetch.BackgroundFetchResult.Failed;
  }
});

// 백그라운드 작업 등록
async function registerBackgroundSync() {
  const status = await BackgroundFetch.getStatusAsync();

  if (status === BackgroundFetch.BackgroundFetchStatus.Available) {
    await BackgroundFetch.registerTaskAsync(BACKGROUND_SYNC_TASK, {
      minimumInterval: 15 * 60,  // 최소 15분 간격
      stopOnTerminate: false,    // 앱 종료 후에도 실행
      startOnBoot: true,         // 기기 재부팅 후 자동 시작
    });
    console.log('백그라운드 동기화 등록 완료');
  } else {
    console.warn('백그라운드 가져오기가 사용 불가능합니다');
  }
}

// 등록 해제
async function unregisterBackgroundSync() {
  await BackgroundFetch.unregisterTaskAsync(BACKGROUND_SYNC_TASK);
}
```

### 위치 추적 (백그라운드)

```typescript
// Android의 Foreground Service + FusedLocationProviderClient에 해당
import * as Location from 'expo-location';
import * as TaskManager from 'expo-task-manager';

const LOCATION_TRACKING = 'location-tracking';

// 백그라운드 위치 추적 작업 정의
TaskManager.defineTask(LOCATION_TRACKING, async ({ data, error }) => {
  if (error) {
    console.error('위치 추적 에러:', error);
    return;
  }

  if (data) {
    const { locations } = data as { locations: Location.LocationObject[] };
    const location = locations[0];

    console.log('백그라운드 위치:', {
      lat: location.coords.latitude,
      lng: location.coords.longitude,
      timestamp: location.timestamp,
    });

    // 서버에 위치 전송
    await fetch('https://api.example.com/location', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        latitude: location.coords.latitude,
        longitude: location.coords.longitude,
      }),
    });
  }
});

// 백그라운드 위치 추적 시작
async function startLocationTracking() {
  // 백그라운드 위치 권한 요청 (Android: ACCESS_BACKGROUND_LOCATION)
  const { status: foregroundStatus } =
    await Location.requestForegroundPermissionsAsync();
  if (foregroundStatus !== 'granted') return;

  const { status: backgroundStatus } =
    await Location.requestBackgroundPermissionsAsync();
  if (backgroundStatus !== 'granted') return;

  await Location.startLocationUpdatesAsync(LOCATION_TRACKING, {
    accuracy: Location.Accuracy.High,
    timeInterval: 10000,     // 10초마다
    distanceInterval: 50,    // 50미터마다
    foregroundService: {     // Android Foreground Service 설정
      notificationTitle: '위치 추적 중',
      notificationBody: '앱이 위치를 추적하고 있습니다',
      notificationColor: '#007AFF',
    },
    showsBackgroundLocationIndicator: true, // iOS 상태 바 표시
  });
}

// 추적 중지
async function stopLocationTracking() {
  const isTracking = await Location.hasStartedLocationUpdatesAsync(LOCATION_TRACKING);
  if (isTracking) {
    await Location.stopLocationUpdatesAsync(LOCATION_TRACKING);
  }
}
```

### Android 비교 정리

```
[백그라운드 작업 비교]

Android                          React Native
────────────────────────────────────────────────
Service                         → Headless JS (Android만)
WorkManager                     → expo-task-manager + BackgroundFetch
Foreground Service              → expo-location foregroundService 옵션
JobScheduler                    → expo-task-manager
AlarmManager                    → expo-notifications (예약 알림)
BroadcastReceiver               → expo-task-manager defineTask
ContentProvider                 → 해당 없음 (앱 간 데이터 공유 제한적)

[제약 사항]
- React Native의 백그라운드 작업은 Android 네이티브보다 제한적
- 복잡한 백그라운드 작업이 필요하면 네이티브 모듈로 구현 권장
- iOS는 Android보다 백그라운드 실행에 더 엄격한 제한이 있음
- expo-task-manager는 Android WorkManager와 유사한 수준의 기능 제공
```

```exercise
type: categorize
question: "다음을 빌드 도구와 테스트/배포 도구로 분류하세요"
categories: ["빌드 도구", "테스트/배포 도구"]
items:
  - text: "Metro Bundler"
    category: "빌드 도구"
  - text: "EAS Build"
    category: "테스트/배포 도구"
  - text: "Hermes"
    category: "빌드 도구"
  - text: "Fastlane"
    category: "테스트/배포 도구"
  - text: "Babel"
    category: "빌드 도구"
  - text: "CodePush"
    category: "테스트/배포 도구"
xp: 6
```

```javascript [playground]
// 🧪 고급 패턴 실습 — 이벤트 시스템 구현

// EventEmitter 구현 (React Native에서 모듈 간 통신에 사용)
class EventEmitter {
  constructor() {
    this.listeners = {};
  }

  on(event, callback) {
    if (!this.listeners[event]) this.listeners[event] = [];
    this.listeners[event].push(callback);
    // unsubscribe 함수 반환
    return () => {
      this.listeners[event] = this.listeners[event].filter(cb => cb !== callback);
    };
  }

  emit(event, data) {
    const callbacks = this.listeners[event] || [];
    callbacks.forEach(cb => cb(data));
    console.log(`[emit] ${event} → ${callbacks.length}개 리스너`);
  }

  once(event, callback) {
    const unsub = this.on(event, (data) => {
      callback(data);
      unsub();
    });
  }
}

// 사용 예: 앱 이벤트 시스템
const appEvents = new EventEmitter();

// 리스너 등록
const unsub1 = appEvents.on('user-login', (user) => {
  console.log(`환영합니다, ${user.name}님!`);
});

appEvents.on('user-login', (user) => {
  console.log(`마지막 로그인: ${new Date().toLocaleTimeString()}`);
});

appEvents.once('user-login', () => {
  console.log("첫 로그인 보너스 지급! (once — 한 번만 실행)");
});

// 이벤트 발생
console.log("=== 첫 번째 로그인 ===");
appEvents.emit('user-login', { name: '홍길동', id: 1 });

console.log("\n=== 두 번째 로그인 ===");
appEvents.emit('user-login', { name: '홍길동', id: 1 });

// 구독 해제
unsub1();
console.log("\n=== 구독 해제 후 ===");
appEvents.emit('user-login', { name: '홍길동', id: 1 });
```

---

## 10. 학습 로드맵 요약

```
[React Native 전문가가 되기 위한 단계]

1단계: 기초 (1-2주)
  ✅ React 기본 (JSX, 컴포넌트, props, state, hooks)
  ✅ TypeScript 기본
  ✅ React Native 핵심 컴포넌트 (View, Text, Image, FlatList)
  ✅ StyleSheet, Flexbox

2단계: 핵심 라이브러리 (2-3주)
  ✅ React Navigation
  ✅ Zustand (상태 관리)
  ✅ TanStack Query (서버 상태)
  ✅ react-native-mmkv (로컬 저장소)

3단계: 중급 (2-3주)
  ✅ react-native-reanimated (애니메이션)
  ✅ react-native-gesture-handler (제스처)
  ✅ react-hook-form + zod (폼)
  ✅ 테스팅 (Jest + RNTL)

4단계: 고급 (3-4주)
  ✅ 성능 최적화
  ✅ 네이티브 모듈 작성 (Kotlin)
  ✅ CI/CD 파이프라인
  ✅ Play Store 배포

5단계: 전문가 (지속적)
  ✅ Expo Router
  ✅ 오프라인 퍼스트 아키텍처
  ✅ 보안 강화
  ✅ 접근성
  ✅ 국제화
  ✅ 모노레포
  ✅ 크로스 플랫폼 (Web)

[Android 개발자의 강점]
- 네이티브 모듈 작성 시 Kotlin 활용 가능
- Android 빌드 시스템 (Gradle) 이해
- Play Store 배포 경험
- 성능 최적화 마인드셋 (60fps 목표)
- 접근성, 보안, 국제화 경험
```
