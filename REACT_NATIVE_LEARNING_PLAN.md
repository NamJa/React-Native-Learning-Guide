# React Native 0.84 학습 로드맵

> **대상**: Kotlin 기반 Android 네이티브 개발자
> **목표**: React Native 0.84를 활용한 크로스 플랫폼 앱 개발 역량 확보
> **기준**: 2026년 최신 Tech Spec
> **공식 문서**: https://reactnative.dev/

---

## 기술 스택 Overview (2026 기준)

| 영역 | 기술 | 비고 |
|------|------|------|
| Runtime | Node.js 22.11.0+ | RN 0.84 최소 요구사항 |
| Language | TypeScript 5.x | RN 기본 언어 (JS/Flow도 지원) |
| JS Engine | Hermes V1 | RN 0.84에서 기본 엔진으로 승격 |
| UI Framework | React 19.2 | Concurrent Rendering 지원 |
| Architecture | New Architecture (Fabric + TurboModules) | RN 0.82부터 완전 전환, Legacy 제거 |
| Navigation | React Navigation 7.x | 공식 권장 네비게이션 라이브러리 |
| State Management | Zustand / TanStack Query | 2026 커뮤니티 주류 |
| Build Tool | Metro Bundler | RN 기본 번들러 |
| Package Manager | npm / yarn / bun | 프로젝트에 따라 선택 |
| Testing | Jest + React Native Testing Library | 공식 권장 |

---

## Phase 0: 사전 준비 — JavaScript/TypeScript 기초

> **기간**: 1~2주
> **Android 개발자를 위한 핵심**: Kotlin과 문법적 유사점이 많으므로 빠르게 습득 가능

### 0-1. JavaScript 핵심 문법

Kotlin 개발자가 집중해야 할 JS 차이점:

| Kotlin | JavaScript/TypeScript | 비고 |
|--------|----------------------|------|
| `val` / `var` | `const` / `let` | `var`(JS)는 사용하지 않음 |
| `fun` | `function` / Arrow `=>` | Arrow function이 주류 |
| `data class` | `interface` / `type` | TS의 타입 시스템 |
| `null safety (?.)` | Optional Chaining `?.` | 동일 문법 |
| `when` | `switch` / Pattern matching | |
| `coroutine` | `async/await` + `Promise` | 비동기 처리의 핵심 |
| `List/Map` | `Array` / `Object` / `Map` | 구조 분해 할당 중요 |
| `Lambda` | Arrow Function | `(x) => x * 2` |

**학습 자료**:
- [MDN JavaScript Guide](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide)
- 구조 분해 할당 (Destructuring)
- 스프레드 연산자 (`...`)
- `map`, `filter`, `reduce` 고차 함수
- `Promise`, `async/await` 비동기 패턴

### 0-2. TypeScript 핵심

```typescript
// Kotlin의 data class와 유사
type User = {
  id: number;
  name: string;
  email?: string; // optional (Kotlin의 String?과 유사)
};

// 제네릭 — Kotlin과 거의 동일
function identity<T>(arg: T): T {
  return arg;
}

// Union Type — Kotlin의 sealed class와 유사한 역할
type Status = 'loading' | 'success' | 'error';
```

**학습 자료**:
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [React + TypeScript Cheatsheet](https://github.com/typescript-cheatsheets/react-typescript-cheatsheet)

### 0-3. 체크리스트

- [ ] `const`, `let`, Arrow Function 사용에 익숙한가?
- [ ] 구조 분해 할당, 스프레드 연산자를 이해하는가?
- [ ] `Promise`, `async/await`로 비동기 코드를 작성할 수 있는가?
- [ ] TypeScript의 `type`, `interface`, 제네릭을 사용할 수 있는가?

---

## Phase 1: React 기초 — UI 패러다임 전환 이해

> **기간**: 1~2주
> **핵심**: Android의 명령형(Imperative) UI → React의 선언형(Declarative) UI 사고방식 전환

### 1-1. Android vs React 패러다임 비교

```
[Android 명령형 방식]
val textView = findViewById<TextView>(R.id.title)
textView.text = "Hello"        // 직접 View를 찾아서 조작
textView.visibility = View.GONE

[React 선언형 방식]
function Title({ visible }: { visible: boolean }) {
  if (!visible) return null;   // 상태에 따라 UI를 "선언"
  return <Text>Hello</Text>;
}
```

### 1-2. React 핵심 개념

| 개념 | Android 대응 | 설명 |
|------|-------------|------|
| **Component** | `View` / `Fragment` | UI의 기본 빌딩 블록 |
| **JSX** | XML Layout | JS 안에서 UI를 선언하는 문법 |
| **Props** | Constructor 파라미터 | 부모 → 자식 데이터 전달 (읽기 전용) |
| **State** | `MutableStateFlow` | 컴포넌트 내부의 변경 가능한 데이터 |
| **Hook** | Lifecycle callback | `useState`, `useEffect` 등 |

### 1-3. 필수 Hooks

```typescript
import { useState, useEffect, useCallback, useMemo } from 'react';

// useState — Kotlin의 MutableState와 유사
const [count, setCount] = useState(0);

// useEffect — Lifecycle(onCreate, onResume 등)과 유사
useEffect(() => {
  // 마운트 시 실행 (onCreate)
  const subscription = api.subscribe();

  return () => {
    // 언마운트 시 실행 (onDestroy)
    subscription.unsubscribe();
  };
}, []); // 빈 배열 = 최초 1회만 실행

// useCallback — 함수 메모이제이션
const handlePress = useCallback(() => {
  setCount(prev => prev + 1);
}, []);

// useMemo — 값 메모이제이션
const expensiveValue = useMemo(() => computeExpensive(data), [data]);
```

### 1-4. 학습 자료

- [React 공식 문서 — Learn React](https://react.dev/learn)
- [RN 공식 — React Fundamentals](https://reactnative.dev/docs/intro-react)

### 1-5. 체크리스트

- [ ] 함수형 컴포넌트를 작성할 수 있는가?
- [ ] Props와 State의 차이를 이해하는가?
- [ ] `useState`, `useEffect`를 올바르게 사용할 수 있는가?
- [ ] 조건부 렌더링과 리스트 렌더링을 구현할 수 있는가?

---

## Phase 2: 개발 환경 구축

> **기간**: 반나절~1일
> **참고**: https://reactnative.dev/docs/set-up-your-environment

### 2-1. 필수 도구 설치 (macOS 기준)

```bash
# 1. Node.js & Watchman
brew install node        # 22.11.0 이상
brew install watchman     # 파일 시스템 변경 감지

# 2. JDK 17 (이미 Android 개발 중이라면 설치되어 있을 수 있음)
brew install --cask zulu@17

# 3. 환경 변수 설정 (~/.zshrc에 추가)
export JAVA_HOME=/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

### 2-2. Android SDK 설정

Android Studio → SDK Manager에서 확인:

- **SDK Platforms**: Android 15 (VanillaIceCream) — API Level 35
- **SDK Build-Tools**: 36.0.0
- **SDK Command-line Tools**: latest

> **Android 개발자 장점**: Android Studio, SDK, 에뮬레이터가 이미 설치되어 있으므로 Node.js만 추가 설치하면 됨

### 2-3. 프로젝트 생성

**방법 A: Expo 사용 (권장 — 빠른 시작)**
```bash
npx create-expo-app@latest MyFirstRNApp
cd MyFirstRNApp
npx expo start
```

**방법 B: React Native CLI (네이티브 코드 직접 제어 필요 시)**
```bash
npx @react-native-community/cli init MyFirstRNApp
cd MyFirstRNApp
npx react-native run-android
```

### 2-4. 프로젝트 구조 이해

```
MyFirstRNApp/
├── android/              ← 네이티브 Android 프로젝트 (익숙한 영역!)
│   ├── app/
│   │   ├── build.gradle
│   │   └── src/main/
│   └── build.gradle
├── ios/                  ← 네이티브 iOS 프로젝트
├── App.tsx               ← 앱 진입점 컴포넌트 (루트에 위치)
├── app/                  ← Expo Router 사용 시 화면 파일 디렉토리
├── components/           ← 재사용 컴포넌트
├── package.json          ← build.gradle의 dependencies 역할
├── tsconfig.json         ← TypeScript 설정
├── metro.config.js       ← Metro 번들러 설정
└── babel.config.js       ← Babel 트랜스파일러 설정
```

### 2-5. 체크리스트

- [ ] `npx react-native doctor` 또는 `npx expo doctor`로 환경 검증 통과
- [ ] 에뮬레이터에서 샘플 앱이 실행되는가?
- [ ] Hot Reload가 정상 동작하는가?

---

## Phase 3: React Native 핵심 컴포넌트

> **기간**: 1~2주
> **참고**: https://reactnative.dev/docs/intro-react-native-components

### 3-1. Core Components — Android View 매핑

| React Native | Android | 용도 |
|-------------|---------|------|
| `<View>` | `ViewGroup` / `FrameLayout` | 컨테이너 (Flexbox 레이아웃) |
| `<Text>` | `TextView` | 텍스트 표시 |
| `<Image>` | `ImageView` | 이미지 표시 |
| `<TextInput>` | `EditText` | 텍스트 입력 |
| `<ScrollView>` | `ScrollView` | 스크롤 가능한 컨테이너 |
| `<FlatList>` | `RecyclerView` | 대량 데이터 리스트 (가상화) |
| `<SectionList>` | Grouped `RecyclerView` | 섹션별 리스트 |
| `<Pressable>` | `View` + `OnClickListener` | 터치 이벤트 처리 |
| `<Modal>` | `DialogFragment` | 모달 다이얼로그 |
| `<ActivityIndicator>` | `ProgressBar` | 로딩 인디케이터 |
| `<Switch>` | `SwitchCompat` | 토글 스위치 |

### 3-2. Flexbox 레이아웃 — ConstraintLayout과의 비교

```typescript
// Android의 LinearLayout(vertical) + weight와 유사
<View style={{ flex: 1, flexDirection: 'column' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />    {/* weight=1 */}
  <View style={{ flex: 2, backgroundColor: 'blue' }} />   {/* weight=2 */}
  <View style={{ flex: 1, backgroundColor: 'green' }} />  {/* weight=1 */}
</View>
```

| Android XML | React Native Style | 설명 |
|------------|-------------------|------|
| `orientation="vertical"` | `flexDirection: 'column'` | 세로 배치 (기본값) |
| `orientation="horizontal"` | `flexDirection: 'row'` | 가로 배치 |
| `gravity="center"` | `justifyContent` + `alignItems` | 주축/교차축 정렬 (gravity는 양축 모두 제어) |
| `layout_gravity="center"` | `alignSelf: 'center'` | 개별 자식의 교차축 위치 지정 |
| `layout_weight` | `flex: N` | 비율 배분 |
| `padding` | `padding` | 내부 여백 |
| `margin` | `margin` | 외부 여백 |

### 3-3. StyleSheet — Android의 styles.xml 대체

```typescript
import { StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  title: {
    fontSize: 20,
    fontWeight: 'bold',
    color: '#333',
  },
});
```

### 3-4. FlatList — RecyclerView 대체

```typescript
import { FlatList, Text, View } from 'react-native';

type Item = { id: string; title: string };

function MyList({ data }: { data: Item[] }) {
  return (
    <FlatList
      data={data}
      keyExtractor={(item) => item.id}         // DiffUtil의 areItemsTheSame
      renderItem={({ item }) => (              // onBindViewHolder
        <View style={styles.item}>
          <Text>{item.title}</Text>
        </View>
      )}
      ItemSeparatorComponent={() => <View style={styles.separator} />}
      ListEmptyComponent={() => <Text>데이터가 없습니다</Text>}
    />
  );
}
```

### 3-5. 실습 과제

1. **프로필 카드 UI**: `View`, `Text`, `Image`, `StyleSheet`으로 프로필 카드 만들기
2. **할 일 목록**: `FlatList`, `TextInput`, `Pressable`로 TODO 리스트 만들기
3. **플랫폼별 UI 분기**: `Platform.OS`를 사용하여 Android/iOS 다른 스타일 적용

### 3-6. 체크리스트

- [ ] 핵심 컴포넌트를 조합하여 화면을 구성할 수 있는가?
- [ ] Flexbox로 원하는 레이아웃을 만들 수 있는가?
- [ ] `FlatList`로 리스트를 효율적으로 렌더링할 수 있는가?

---

## Phase 4: 네비게이션

> **기간**: 1주
> **참고**: https://reactnative.dev/docs/navigation

### 4-1. React Navigation 설치

```bash
# Core
npm install @react-navigation/native @react-navigation/native-stack

# Dependencies
npm install react-native-screens react-native-safe-area-context
```

### 4-2. Android 개발자를 위한 네비게이션 개념 매핑

| Android | React Navigation | 설명 |
|---------|-----------------|------|
| `NavGraph` | `Navigator` 정의 | 화면 구조 선언 |
| `NavController.navigate()` | `navigation.navigate()` | 화면 전환 |
| `Bundle` / `SafeArgs` | `route.params` | 화면 간 데이터 전달 |
| `Activity` + `Fragment` | `Screen` | 개별 화면 단위 |
| `BottomNavigationView` | `createBottomTabNavigator` | 하단 탭 네비게이션 |
| `DrawerLayout` | `createDrawerNavigator` | 드로어 네비게이션 |
| `Back Stack` | Navigation Stack | 화면 스택 관리 |

### 4-3. 기본 Stack Navigation

```typescript
import { createStaticNavigation } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

// Android의 NavGraph 정의와 유사
const RootStack = createNativeStackNavigator({
  screens: {
    Home: {
      screen: HomeScreen,
      options: { title: '홈' },
    },
    Detail: {
      screen: DetailScreen,
    },
  },
});

const Navigation = createStaticNavigation(RootStack);

export default function App() {
  return <Navigation />;
}

// 화면 전환 — startActivity(intent)와 유사
function HomeScreen() {
  const navigation = useNavigation();
  return (
    <Pressable onPress={() => navigation.navigate('Detail', { id: 42 })}>
      <Text>상세 보기</Text>
    </Pressable>
  );
}

// 파라미터 수신 — intent.extras와 유사
function DetailScreen({ route }) {
  const { id } = route.params;
  return <Text>Item ID: {id}</Text>;
}
```

### 4-4. 실습 과제

1. Stack Navigation으로 3개 이상의 화면 전환 구현
2. Bottom Tab Navigation 추가
3. TypeScript로 Navigation 파라미터 타입 안전하게 관리

### 4-5. 체크리스트

- [ ] Stack, Tab, Drawer Navigation을 구현할 수 있는가?
- [ ] 화면 간 파라미터를 타입 안전하게 전달할 수 있는가?
- [ ] Deep Linking을 설정할 수 있는가?

---

## Phase 5: 상태 관리 & 네트워크

> **기간**: 1~2주

### 5-1. 상태 관리 — Android 아키텍처와의 비교

| Android | React Native | 역할 |
|---------|-------------|------|
| `ViewModel` + `StateFlow` | `useState` + Context / Zustand | UI 상태 관리 |
| `Repository` + `Retrofit` | TanStack Query + `fetch` | 서버 데이터 관리 |
| `Room DB` | AsyncStorage / MMKV | 로컬 데이터 저장 |
| `Hilt` / `Koin` | React Context / Props | 의존성 주입 |

### 5-2. Zustand — 간결한 전역 상태 관리

```typescript
import { create } from 'zustand';

// Kotlin의 ViewModel과 유사한 역할
interface AuthStore {
  user: User | null;
  isLoggedIn: boolean;
  login: (user: User) => void;
  logout: () => void;
}

const useAuthStore = create<AuthStore>((set) => ({
  user: null,
  isLoggedIn: false,
  login: (user) => set({ user, isLoggedIn: true }),
  logout: () => set({ user: null, isLoggedIn: false }),
}));

// 컴포넌트에서 사용 — collectAsState()와 유사
function ProfileScreen() {
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);

  return (
    <View>
      <Text>{user?.name}</Text>
      <Pressable onPress={logout}><Text>로그아웃</Text></Pressable>
    </View>
  );
}
```

### 5-3. TanStack Query — 서버 상태 관리 (Retrofit 대체)

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Retrofit의 @GET과 유사
function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('https://api.example.com/users');
      return response.json() as Promise<User[]>;
    },
    staleTime: 5 * 60 * 1000, // 5분간 캐시 유지
  });
}

// Retrofit의 @POST와 유사
function useCreateUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (newUser: CreateUserDto) => {
      const response = await fetch('https://api.example.com/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newUser),
      });
      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] }); // 캐시 무효화
    },
  });
}

// 컴포넌트에서 사용
function UserListScreen() {
  const { data: users, isLoading, error } = useUsers();

  if (isLoading) return <ActivityIndicator />;
  if (error) return <Text>에러 발생</Text>;

  return (
    <FlatList
      data={users}
      keyExtractor={(item) => String(item.id)}
      renderItem={({ item }) => <Text>{item.name}</Text>}
    />
  );
}
```

### 5-4. 체크리스트

- [ ] Zustand로 전역 상태를 관리할 수 있는가?
- [ ] TanStack Query로 API 호출 & 캐싱을 구현할 수 있는가?
- [ ] 로딩/에러/성공 상태를 UI에 반영할 수 있는가?

---

## Phase 6: New Architecture 이해

> **기간**: 1주
> **참고**: https://reactnative.dev/docs/the-new-architecture/landing-page

### 6-1. New Architecture 구성 요소

```
┌─────────────────────────────────────────────────┐
│                  JavaScript                      │
│              (React 19.2 + Hermes V1)            │
├─────────────────────────────────────────────────┤
│                    JSI                           │  ← Bridge 제거, C++ 직접 바인딩
│          (JavaScript Interface)                  │
├──────────────────┬──────────────────────────────┤
│    Fabric        │      TurboModules            │
│  (새 렌더러)      │   (새 네이티브 모듈 시스템)     │
├──────────────────┴──────────────────────────────┤
│              Codegen                             │  ← TypeScript → 네이티브 타입 자동 생성
├─────────────────────────────────────────────────┤
│          Native Platform (Android / iOS)         │
└─────────────────────────────────────────────────┘
```

### 6-2. Android 개발자가 알아야 할 핵심 변화

| 항목 | Legacy Architecture | New Architecture (0.84) |
|------|-------------------|------------------------|
| JS-Native 통신 | 비동기 Bridge (JSON 직렬화) | JSI (C++ 직접 호출, 동기/비동기) |
| 렌더러 | Legacy Renderer | Fabric (동기 레이아웃) |
| Native Module | `@ReactMethod` (Bridge) | TurboModules (JSI 기반) |
| 타입 안전성 | 런타임 타입 체크 | Codegen으로 컴파일 타임 보장 |
| React 기능 | React 18까지 (Concurrent 제한적) | React 19.2 (Concurrent Rendering 완전 지원) |

### 6-3. Concurrent Rendering 활용

```typescript
import { useState, useTransition } from 'react';

function SearchScreen() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Item[]>([]);
  const [isPending, startTransition] = useTransition();

  const handleSearch = (text: string) => {
    setQuery(text); // 즉시 반영 (높은 우선순위)

    startTransition(() => {
      // 검색 결과 업데이트 (낮은 우선순위, 인터럽트 가능)
      setResults(filterItems(text));
    });
  };

  return (
    <View>
      <TextInput value={query} onChangeText={handleSearch} />
      {isPending && <ActivityIndicator />}
      <FlatList data={results} renderItem={/* ... */} />
    </View>
  );
}
```

### 6-4. 체크리스트

- [ ] JSI, Fabric, TurboModules의 역할을 설명할 수 있는가?
- [ ] Legacy Architecture와의 차이를 이해하는가?
- [ ] Concurrent Rendering의 이점을 이해하는가?

---

## Phase 7: 네이티브 모듈 연동 (Android/Kotlin)

> **기간**: 1~2주
> **Android 개발자의 최대 장점을 활용하는 단계**

### 7-1. TurboModule 생성 (Kotlin)

TypeScript 스펙 정의 → Codegen → Kotlin 구현의 흐름:

**Step 1: TypeScript Spec 정의**
```typescript
// specs/NativeDeviceInfo.ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getBatteryLevel(): Promise<number>;
  getDeviceModel(): string; // 동기 호출 가능 (JSI의 장점)
}

export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
```

**Step 2: Kotlin 구현**
```kotlin
// android/app/src/main/java/com/myapp/DeviceInfoModule.kt
package com.myapp

import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.Promise

class DeviceInfoModule(reactContext: ReactApplicationContext) :
    NativeDeviceInfoSpec(reactContext) {

    override fun getName() = "DeviceInfo"

    override fun getBatteryLevel(promise: Promise) {
        // Android 네이티브 코드를 그대로 활용!
        val batteryManager = reactApplicationContext
            .getSystemService(Context.BATTERY_SERVICE) as BatteryManager
        val level = batteryManager.getIntProperty(
            BatteryManager.BATTERY_PROPERTY_CAPACITY
        )
        promise.resolve(level.toDouble())
    }

    override fun getDeviceModel(): String {
        return android.os.Build.MODEL
    }
}
```

**Step 3: JS에서 사용**
```typescript
import DeviceInfo from './specs/NativeDeviceInfo';

const model = DeviceInfo.getDeviceModel(); // 동기 호출
const battery = await DeviceInfo.getBatteryLevel(); // 비동기 호출
```

### 7-2. Fabric Native Component (커스텀 네이티브 뷰)

Android의 커스텀 View를 React Native에서 사용하기:

**Step 1: TypeScript Spec 정의 (Codegen 입력)**
```typescript
// specs/MyCustomViewNativeComponent.ts
import type { ViewProps } from 'react-native';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

export interface NativeProps extends ViewProps {
  color?: string;
}

export default codegenNativeComponent<NativeProps>('MyCustomView') as HostComponent<NativeProps>;
```

**Step 2: Kotlin 구현 (Codegen이 생성한 인터페이스 구현)**
```kotlin
// android/app/src/main/java/com/myapp/MyCustomViewManager.kt
class MyCustomViewManager : SimpleViewManager<MyCustomView>() {
    override fun getName() = "MyCustomView"

    override fun createViewInstance(context: ThemedReactContext): MyCustomView {
        return MyCustomView(context)
    }

    @ReactProp(name = "color")
    fun setColor(view: MyCustomView, color: String) {
        view.setColor(Color.parseColor(color))
    }
}
```

> **참고**: `SimpleViewManager`는 New Architecture에서도 호환 레이어를 통해 동작합니다.
> 순수 Fabric 방식은 Codegen이 생성한 `ViewManagerDelegate`를 구현하는 형태이며,
> 대부분의 경우 위 방식으로 충분합니다.
```

### 7-3. 체크리스트

- [ ] TurboModule을 생성하고 JS에서 호출할 수 있는가?
- [ ] 기존 Kotlin 코드를 React Native에서 재사용할 수 있는가?
- [ ] Fabric Native Component를 만들 수 있는가?

---

## Phase 8: 디버깅 & 테스팅

> **기간**: 1주
> **참고**: https://reactnative.dev/docs/debugging

### 8-1. 디버깅 도구

| 도구 | Android 대응 | 용도 |
|------|-------------|------|
| React Native DevTools | Android Studio Debugger | JS 코드 디버깅, 컴포넌트 트리 검사 |
| Dev Menu (`Cmd+M`) | — | 개발 메뉴 (Reload, DevTools 등) |
| LogBox | Logcat | 인앱 에러/경고 표시 |
| Performance Monitor | Android Profiler | 성능 모니터링 |
| React DevTools (Profiler) | Layout Inspector | 컴포넌트 트리 & 렌더링 분석 |
| Android Studio | — | 네이티브 코드 디버깅 (기존 도구 활용) |

### 8-2. 테스팅

```typescript
// Jest + React Native Testing Library
import { render, screen, fireEvent } from '@testing-library/react-native';
import { Counter } from './Counter';

test('increments counter on press', () => {
  render(<Counter />);

  const button = screen.getByText('증가');
  fireEvent.press(button);

  expect(screen.getByText('1')).toBeTruthy();
});
```

| 테스트 종류 | 도구 | Android 대응 |
|-----------|------|-------------|
| Unit Test | Jest | JUnit |
| Component Test | React Native Testing Library | Espresso |
| E2E Test | Detox / Maestro | UI Automator / Espresso |
| Snapshot Test | Jest Snapshot | — |

### 8-3. 체크리스트

- [ ] React Native DevTools로 컴포넌트를 검사할 수 있는가?
- [ ] Jest로 단위 테스트를 작성할 수 있는가?
- [ ] 네이티브 코드는 Android Studio에서 디버깅할 수 있는가?

---

## Phase 9: 실전 프로젝트 & 배포

> **기간**: 2~3주

### 9-1. 실전 프로젝트 제안

**프로젝트: "뉴스 리더 앱"**

구현할 기능:
1. 뉴스 목록 (FlatList + TanStack Query)
2. 뉴스 상세 (Stack Navigation + 파라미터 전달)
3. 카테고리 탭 (Bottom Tab Navigation)
4. 북마크 기능 (Zustand + MMKV 로컬 저장)
5. 다크 모드 (React Context + `useColorScheme`)
6. 당겨서 새로고침 (Pull to Refresh)
7. 네이티브 모듈: 공유 기능 (Android Share Intent를 TurboModule로 구현)

### 9-2. 추천 라이브러리 (2026 기준)

| 카테고리 | 라이브러리 | 설명 |
|---------|----------|------|
| Navigation | `@react-navigation/native` v7 | 공식 권장 |
| State | `zustand` | 간결한 전역 상태 |
| Server State | `@tanstack/react-query` v5 | API 캐싱/동기화 |
| Storage | `react-native-mmkv` | 고속 KV 저장소 (SharedPreferences 대체) |
| Styling | `nativewind` v4 (Tailwind) | Tailwind CSS 기반 스타일링 |
| Forms | `react-hook-form` + `zod` | 폼 관리 + 스키마 검증 |
| Animation | `react-native-reanimated` v3 | 네이티브 드리븐 애니메이션 |
| Image | `expo-image` | 고성능 이미지 컴포넌트 |
| Icons | `@expo/vector-icons` | 아이콘 라이브러리 |

### 9-3. 배포

```bash
# Android AAB 빌드 (기존 Android 배포 프로세스와 동일)
cd android
./gradlew bundleRelease

# 빌드 결과: android/app/build/outputs/bundle/release/app-release.aab
# Google Play Console에 업로드
```

### 9-4. 체크리스트

- [ ] 실전 프로젝트를 완성했는가?
- [ ] Release 빌드를 생성하고 디바이스에서 테스트했는가?
- [ ] Play Store 배포 프로세스를 이해하는가?

---

## Phase 10: 심화 학습

> **기간**: 지속적

### 10-1. 성능 최적화

- `React.memo`, `useMemo`, `useCallback`으로 불필요한 리렌더링 방지
- `FlatList` 최적화: `getItemLayout`, `windowSize`, `maxToRenderPerBatch`
- Hermes V1 바이트코드 프리컴파일 활용
- React DevTools Profiler로 병목 구간 분석

### 10-2. 애니메이션

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

function AnimatedBox() {
  const offset = useSharedValue(0);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: withSpring(offset.value) }],
  }));

  return (
    <Animated.View style={[styles.box, animatedStyle]}>
      <Pressable onPress={() => { offset.value = offset.value + 50; }}>
        <Text>이동</Text>
      </Pressable>
    </Animated.View>
  );
}
```

### 10-3. 추가 학습 토픽

- [ ] CodePush를 이용한 OTA 업데이트
- [ ] CI/CD 파이프라인 구축 (EAS Build / Fastlane)
- [ ] Expo Router (파일 기반 라우팅)
- [ ] React Native for Web (react-strict-dom)
- [ ] Monorepo 구성 (Turborepo)

---

## 학습 타임라인 요약

| Phase | 주제 | 예상 기간 | 우선순위 |
|-------|------|----------|---------|
| 0 | JavaScript/TypeScript 기초 | 1~2주 | 🔴 필수 |
| 1 | React 기초 | 1~2주 | 🔴 필수 |
| 2 | 개발 환경 구축 | 0.5~1일 | 🔴 필수 |
| 3 | 핵심 컴포넌트 & 레이아웃 | 1~2주 | 🔴 필수 |
| 4 | 네비게이션 | 1주 | 🔴 필수 |
| 5 | 상태 관리 & 네트워크 | 1~2주 | 🔴 필수 |
| 6 | New Architecture 이해 | 1주 | 🟡 권장 |
| 7 | 네이티브 모듈 연동 (Kotlin) | 1~2주 | 🟡 권장 |
| 8 | 디버깅 & 테스팅 | 1주 | 🟡 권장 |
| 9 | 실전 프로젝트 & 배포 | 2~3주 | 🔴 필수 |
| 10 | 심화 학습 | 지속적 | 🟢 선택 |

> **총 예상 기간: 약 10~16주 (2.5~4개월)**

---

## 참고 자료

- [React Native 공식 문서](https://reactnative.dev/)
- [React 공식 문서](https://react.dev/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/)
- [React Navigation 문서](https://reactnavigation.org/)
- [Expo 문서](https://docs.expo.dev/)
- [React Native Directory (커뮤니티 라이브러리)](https://reactnative.directory/)
