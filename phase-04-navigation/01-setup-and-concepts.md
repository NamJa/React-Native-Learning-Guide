# React Navigation 설치와 핵심 개념 — Android Navigation Component와 비교

## 목차
1. [왜 React Navigation인가?](#1-왜-react-navigation인가)
2. [대안 라이브러리 비교](#2-대안-라이브러리-비교)
3. [설치 가이드](#3-설치-가이드)
4. [NavigationContainer 설정](#4-navigationcontainer-설정)
5. [Navigator 타입 개요](#5-navigator-타입-개요)
6. [Android Navigation Component와 핵심 개념 매핑](#6-android-navigation-component와-핵심-개념-매핑)
7. [Static vs Dynamic Navigation API (v7)](#7-static-vs-dynamic-navigation-api-v7)
8. [TypeScript 타입 안전성](#8-typescript-타입-안전성)
9. [완전한 초기 설정 코드](#9-완전한-초기-설정-코드)

---

## 1. 왜 React Navigation인가?

Android 개발자라면 Navigation Component(Jetpack Navigation)를 사용하여 Fragment 간 전환, 딥링크, Safe Args를 통한 타입 안전한 인자 전달 등을 경험했을 것이다. React Native에서는 **React Navigation**이 사실상 표준(de facto standard) 네비게이션 라이브러리다.

React Navigation을 선택해야 하는 이유:

- **공식 추천**: React Native 공식 문서에서 React Navigation을 권장한다
- **커뮤니티**: npm 주간 다운로드 200만 이상, 활발한 유지보수
- **Expo 호환**: Expo와 완벽 호환되며, Expo Router의 기반이기도 하다
- **플랫폼 네이티브 경험**: iOS와 Android 각각의 네이티브 전환 애니메이션 지원
- **TypeScript 지원**: v7부터 더욱 강화된 타입 안전성
- **유연성**: Stack, Tab, Drawer 등 다양한 네비게이터 조합 가능

Android Navigation Component와의 핵심 차이점:

| 항목 | Android Navigation | React Navigation |
|------|-------------------|-----------------|
| 설정 방식 | XML nav_graph + NavHostFragment | JavaScript/TypeScript 코드 |
| 화면 단위 | Fragment | React Component (Screen) |
| 인자 전달 | Bundle / Safe Args | route.params (TypeScript 타입) |
| 백스택 관리 | FragmentManager 자동 관리 | Navigator 내부 상태 관리 |
| 딥링크 | AndroidManifest intent-filter | linking config 객체 |
| 전환 애니메이션 | anim/animator XML | JavaScript 설정 또는 네이티브 전환 |

---

## 2. 대안 라이브러리 비교

### React Navigation (추천)
```
장점: 가장 넓은 커뮤니티, Expo 완벽 지원, 유연한 커스터마이징
단점: JS 기반 전환 시 성능 이슈(NativeStack으로 해결)
설치: npm install @react-navigation/native
```

### react-native-navigation (Wix)
```
장점: 완전한 네이티브 네비게이션 (각 화면이 진짜 네이티브 Activity/Fragment)
단점: Expo 미지원, 설정 복잡, 커뮤니티 축소 추세
설치: npm install react-native-navigation
```

Wix의 react-native-navigation은 Android 관점에서 보면 각 화면이 실제 별도의 Activity처럼 동작한다. 이는 네이티브 성능을 보장하지만, 설정이 복잡하고 Expo를 사용할 수 없다는 큰 단점이 있다. 2024년 이후로 업데이트 빈도가 줄어들고 있어 새 프로젝트에서는 권장하지 않는다.

### Expo Router
```
장점: 파일 기반 라우팅 (Next.js 스타일), React Navigation 기반
단점: Expo 프로젝트 전용, 파일 구조에 따른 제약
설치: npx expo install expo-router
```

Expo Router는 React Navigation 위에 구축된 파일 기반 라우팅 시스템이다. `app/` 디렉토리 안의 파일 구조가 곧 라우트 구조가 된다. Android의 Navigation Component XML이 파일 시스템으로 대체된 것이라 생각하면 된다. Expo 프로젝트라면 Expo Router를 고려할 만하지만, React Navigation의 개념을 먼저 이해하는 것이 필수다.

**결론**: 이 학습 문서에서는 React Navigation v7을 기준으로 설명한다. Expo Router를 사용하더라도 내부적으로 React Navigation이므로, 여기서 배우는 모든 개념이 그대로 적용된다.

---

## 3. 설치 가이드

### 3-1. 핵심 패키지 설치

```bash
# 1단계: 코어 라이브러리 설치
npm install @react-navigation/native

# 2단계: peer dependencies 설치 (React Native CLI 프로젝트)
npm install react-native-screens react-native-safe-area-context

# 2단계 대안: Expo 프로젝트의 경우
npx expo install react-native-screens react-native-safe-area-context

# 3단계: Native Stack Navigator 설치 (가장 많이 사용)
npm install @react-navigation/native-stack
```

각 패키지의 역할:

- **@react-navigation/native**: React Navigation의 코어. NavigationContainer, 훅(useNavigation, useRoute 등) 제공
- **react-native-screens**: 네이티브 화면 최적화. Android에서는 Fragment를 사용하여 메모리 효율적인 화면 관리를 한다. 이 라이브러리가 없으면 모든 화면이 메모리에 상주한다
- **react-native-safe-area-context**: 노치, 상태바 등 안전 영역 처리. Android의 `WindowInsetsCompat`과 유사한 역할
- **@react-navigation/native-stack**: 네이티브 스택 네비게이터. Android의 Fragment Transaction을 내부적으로 사용

### 3-2. Android 네이티브 설정 (React Native CLI만 해당)

`android/app/src/main/java/.../MainActivity.kt` 파일을 수정한다:

```kotlin
// Android 개발자에게 익숙한 Kotlin 코드
import android.os.Bundle

class MainActivity : ReactActivity() {
    // react-native-screens가 제대로 작동하려면 필요
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(null) // savedInstanceState 대신 null 전달
    }
}
```

왜 `null`을 전달하는가? react-native-screens는 자체적으로 Fragment 상태를 관리하는데, Android의 기본 savedInstanceState 복원과 충돌할 수 있다. 이는 Android Navigation Component에서도 유사한 이슈가 있던 것과 같은 맥락이다.

### 3-3. iOS 설정 (참고)

```bash
cd ios && pod install && cd ..
```

### 3-4. 추가 네비게이터 설치 (필요시)

```bash
# Bottom Tab Navigator
npm install @react-navigation/bottom-tabs

# Drawer Navigator
npm install @react-navigation/drawer
npm install react-native-gesture-handler react-native-reanimated

# Material Top Tabs
npm install @react-navigation/material-top-tabs react-native-tab-view react-native-pager-view
```

---

## 4. NavigationContainer 설정

`NavigationContainer`는 Android의 `NavHostFragment`에 해당한다. 앱의 최상위에서 네비게이션 상태를 관리하고, 딥링크를 처리하고, 화면 전환을 조율한다.

### 4-1. 기본 설정

```tsx
// App.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { View, Text } from 'react-native';

// Android의 Fragment에 해당하는 Screen 컴포넌트
function HomeScreen() {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>Home Screen</Text>
    </View>
  );
}

const Stack = createNativeStackNavigator();

// Android에서 NavHostFragment + nav_graph를 설정하는 것과 동일
function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

export default App;
```

### 4-2. Android 비교: NavHostFragment 설정

Android에서의 동일한 개념:

```xml
<!-- Android: activity_main.xml -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    app:navGraph="@navigation/nav_graph"
    app:defaultNavHost="true" />
```

```xml
<!-- Android: nav_graph.xml -->
<navigation
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">
    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home" />
</navigation>
```

React Navigation에서는 이 모든 XML이 JavaScript 코드로 대체된다. NavigationContainer가 NavHostFragment, Stack.Navigator가 navigation XML, Stack.Screen이 각 fragment 태그에 해당한다.

### 4-3. NavigationContainer의 주요 props

```tsx
<NavigationContainer
  // 네비게이션 상태 변경 추적 (Analytics에 유용)
  onStateChange={(state) => {
    console.log('Navigation state:', state);
  }}

  // 네비게이션 준비 완료 콜백
  onReady={() => {
    console.log('Navigation is ready');
    // 스플래시 스크린 숨기기에 적합
  }}

  // 딥링크 설정 (나중에 자세히 다룸)
  linking={linkingConfig}

  // 초기 상태 복원 (앱 재시작 시)
  initialState={restoredState}

  // 테마 설정
  theme={MyTheme}

  // ref로 외부에서 네비게이션 제어
  ref={navigationRef}
>
  {/* Navigator들 */}
</NavigationContainer>
```

---

## 5. Navigator 타입 개요

React Navigation은 여러 종류의 Navigator를 제공한다. Android 개발자에게 익숙한 개념과 매핑하면 다음과 같다:

### 5-1. Stack Navigator

```
Android: Fragment Transaction (push/pop), NavController navigate/popBackStack
React Navigation: createNativeStackNavigator()

동작: 화면을 스택처럼 쌓고 뒤로가기 시 이전 화면으로 돌아감
사용처: 메인 앱 플로우, 상세 화면 이동, 모달
```

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function AppStack() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
      <Stack.Screen name="Settings" component={SettingsScreen} />
    </Stack.Navigator>
  );
}
```

### 5-2. Bottom Tab Navigator

```
Android: BottomNavigationView + NavGraph (각 탭이 별도 NavGraph)
React Navigation: createBottomTabNavigator()

동작: 화면 하단에 탭 바, 각 탭은 독립적인 화면 스택 보유 가능
사용처: 메인 탭 UI (홈, 검색, 프로필 등)
```

```tsx
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MainTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Search" component={SearchScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

### 5-3. Drawer Navigator

```
Android: DrawerLayout + NavigationView
React Navigation: createDrawerNavigator()

동작: 좌측(또는 우측)에서 슬라이드되는 메뉴 패널
사용처: 설정, 프로필 메뉴 등 보조 네비게이션
```

```tsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

function AppDrawer() {
  return (
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={HomeScreen} />
      <Drawer.Screen name="Settings" component={SettingsScreen} />
    </Drawer.Navigator>
  );
}
```

### 5-4. Material Top Tab Navigator

```
Android: ViewPager2 + TabLayout
React Navigation: createMaterialTopTabNavigator()

동작: 화면 상단에 탭, 좌우 스와이프로 전환
사용처: 카테고리별 콘텐츠 (예: 뉴스 카테고리, 채팅방 종류)
```

```tsx
import { createMaterialTopTabNavigator } from '@react-navigation/material-top-tabs';

const TopTab = createMaterialTopTabNavigator();

function CategoryTabs() {
  return (
    <TopTab.Navigator>
      <TopTab.Screen name="All" component={AllScreen} />
      <TopTab.Screen name="Popular" component={PopularScreen} />
      <TopTab.Screen name="Recent" component={RecentScreen} />
    </TopTab.Navigator>
  );
}
```

---

## 6. Android Navigation Component와 핵심 개념 매핑

### 6-1. Navigator ↔ NavGraph

Android의 NavGraph는 화면들의 연결 관계를 정의하는 XML이다. React Navigation의 Navigator가 동일한 역할을 한다.

```kotlin
// Android: NavGraph 정의 (Kotlin DSL)
val navController = findNavController(R.id.nav_host_fragment)
navController.graph = navController.createGraph(
    startDestination = "home"
) {
    fragment<HomeFragment>("home")
    fragment<DetailFragment>("detail/{id}")
}
```

```tsx
// React Navigation: 동일한 구조
const Stack = createNativeStackNavigator();

function AppNavigator() {
  return (
    <Stack.Navigator initialRouteName="Home">
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Detail" component={DetailScreen} />
    </Stack.Navigator>
  );
}
```

### 6-2. Screen ↔ Fragment/Destination

Android에서 각 화면은 Fragment이고 NavGraph의 destination이다. React Navigation에서는 React 컴포넌트가 Screen으로 등록된다.

```kotlin
// Android: Fragment
class HomeFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setContent { HomeContent() }
        }
    }
}
```

```tsx
// React Navigation: Screen 컴포넌트
// Fragment의 onCreateView가 함수 컴포넌트의 return에 해당
function HomeScreen({ navigation, route }) {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text>Home</Text>
      <Button
        title="Go to Detail"
        onPress={() => navigation.navigate('Detail', { id: 42 })}
      />
    </View>
  );
}
```

### 6-3. navigation.navigate() ↔ NavController.navigate()

```kotlin
// Android: 화면 이동
findNavController().navigate(R.id.detailFragment, bundleOf("id" to 42))
// 또는 Safe Args 사용
findNavController().navigate(HomeFragmentDirections.actionToDetail(id = 42))
```

```tsx
// React Navigation: 화면 이동
navigation.navigate('Detail', { id: 42 });
```

**중요한 차이**: Android의 `navigate()`는 같은 destination을 여러 번 호출해도 스택에 쌓이지만, React Navigation의 `navigate()`는 이미 스택에 있는 화면으로 이동 시 해당 화면으로 돌아간다(스택에 새로 쌓지 않음). Android의 `popUpTo`와 유사한 동작이다. 스택에 새로 쌓고 싶으면 `push()`를 사용한다.

```tsx
// navigate: 이미 스택에 있으면 그 화면으로 돌아감 (Android popUpTo와 유사)
navigation.navigate('Detail', { id: 42 });

// push: 항상 스택에 새로 쌓음 (Android navigate()와 유사)
navigation.push('Detail', { id: 42 });
```

### 6-4. route.params ↔ Bundle/SafeArgs

```kotlin
// Android: Safe Args로 인자 받기
class DetailFragment : Fragment() {
    private val args: DetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val id = args.id // 타입 안전
    }
}
```

```tsx
// React Navigation: route.params로 인자 받기
type DetailParams = {
  id: number;
};

function DetailScreen({ route }: { route: RouteProp<RootStackParamList, 'Detail'> }) {
  const { id } = route.params; // TypeScript로 타입 안전
  return <Text>Detail for item {id}</Text>;
}
```

### 6-5. Back Stack 관리

```kotlin
// Android: 백스택 관리
findNavController().popBackStack()              // 하나 뒤로
findNavController().popBackStack(R.id.home, false) // 특정 화면까지 pop
```

```tsx
// React Navigation: 백스택 관리
navigation.goBack();                    // 하나 뒤로
navigation.popTo('Home');               // 특정 화면까지 pop (v7)
navigation.pop(2);                      // 2개 뒤로 (Stack Navigator에서만)
```

### 6-6. navigation.goBack() ↔ NavController.popBackStack()

두 메서드 모두 스택에서 현재 화면을 제거하고 이전 화면으로 돌아간다. Android에서 시스템 뒤로가기 버튼이 자동으로 popBackStack을 호출하는 것처럼, React Navigation에서도 Android의 하드웨어 뒤로가기 버튼이 자동으로 goBack을 호출한다.

---

## 7. Static vs Dynamic Navigation API (v7)

React Navigation v7에서 도입된 중요한 변화가 **Static API**다. Android 개발자 관점에서, Static API는 Compose Navigation의 `NavHost { composable("route") {} }` 선언적 패턴과 더 비슷하다.

### 7-1. Dynamic API (기존 방식, 여전히 유효)

```tsx
// Dynamic API: 런타임에 화면 구성 가능
const Stack = createNativeStackNavigator();

function App() {
  const isLoggedIn = useAuth();

  return (
    <NavigationContainer>
      <Stack.Navigator>
        {isLoggedIn ? (
          <>
            <Stack.Screen name="Home" component={HomeScreen} />
            <Stack.Screen name="Profile" component={ProfileScreen} />
          </>
        ) : (
          <>
            <Stack.Screen name="Login" component={LoginScreen} />
            <Stack.Screen name="Register" component={RegisterScreen} />
          </>
        )}
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

### 7-2. Static API (v7 신규)

```tsx
// Static API: 빌드 타임에 화면 구성 확정
// 타입 추론이 자동으로 이루어져 TypeScript와 더 잘 맞음
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const RootStack = createNativeStackNavigator({
  initialRouteName: 'Home',
  screens: {
    Home: {
      screen: HomeScreen,
      options: {
        title: '홈',
      },
    },
    Detail: {
      screen: DetailScreen,
      options: {
        title: '상세',
      },
    },
    Settings: {
      screen: SettingsScreen,
      options: {
        title: '설정',
      },
    },
  },
});

// NavigationContainer 대신 createStaticNavigation 사용
import { createStaticNavigation } from '@react-navigation/native';

const Navigation = createStaticNavigation(RootStack);

function App() {
  return <Navigation />;
}
```

### 7-3. 어떤 API를 사용할까?

| 기준 | Static API | Dynamic API |
|------|-----------|-------------|
| TypeScript 타입 추론 | 자동 (권장) | 수동으로 ParamList 정의 필요 |
| 조건부 화면 | 제한적 (if 사용) | 완전 자유 |
| 딥링크 설정 | 화면 정의에 통합 | 별도 linking config |
| 러닝 커브 | 낮음 | 약간 높음 |
| 유연성 | 보통 | 높음 |
| 공식 권장 (v7) | 새 프로젝트에 권장 | 기존 프로젝트 유지 |

Static API에서의 조건부 화면 (인증 플로우):

```tsx
const RootStack = createNativeStackNavigator({
  screens: {
    Home: HomeScreen,
    Profile: ProfileScreen,
  },
  groups: {
    // 로그인하지 않은 사용자에게만 보이는 화면 그룹
    Auth: {
      if: useIsSignedOut, // boolean 반환하는 훅
      screens: {
        Login: LoginScreen,
        Register: RegisterScreen,
      },
    },
  },
});
```

---

## 8. TypeScript 타입 안전성

Android의 Safe Args가 네비게이션 인자의 타입 안전성을 보장하듯, React Navigation에서는 TypeScript를 활용하여 동일한 안전성을 확보한다.

### 8-1. ParamList 타입 정의 (Dynamic API)

```tsx
// 모든 화면의 파라미터 타입을 하나의 타입으로 정의
// Android Safe Args의 nav_graph.xml에 정의하는 argument와 동일한 역할
type RootStackParamList = {
  Home: undefined;                        // 파라미터 없음
  Detail: { id: number; title: string };  // 필수 파라미터
  Profile: { userId: string } | undefined; // 선택적 파라미터
  Settings: undefined;
};

// Navigator 생성 시 타입 전달
const Stack = createNativeStackNavigator<RootStackParamList>();
```

### 8-2. useNavigation 훅에 타입 적용

```tsx
import { useNavigation } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';

// 방법 1: 직접 타입 지정
type HomeNavigationProp = NativeStackNavigationProp<RootStackParamList, 'Home'>;

function HomeScreen() {
  const navigation = useNavigation<HomeNavigationProp>();

  // 타입 안전! 'Detail'에는 id와 title이 필수
  navigation.navigate('Detail', { id: 1, title: '아이템 1' });

  // 타입 에러! title이 빠졌음
  // navigation.navigate('Detail', { id: 1 }); // TS Error

  // 타입 에러! 존재하지 않는 화면
  // navigation.navigate('NonExistent'); // TS Error
}
```

### 8-3. useRoute 훅에 타입 적용

```tsx
import { useRoute, RouteProp } from '@react-navigation/native';

type DetailRouteProp = RouteProp<RootStackParamList, 'Detail'>;

function DetailScreen() {
  const route = useRoute<DetailRouteProp>();

  // 타입 안전! id는 number, title은 string
  const { id, title } = route.params;

  return (
    <View>
      <Text>ID: {id}</Text>
      <Text>Title: {title}</Text>
    </View>
  );
}
```

### 8-4. 글로벌 타입 선언 (권장 패턴)

```tsx
// navigation/types.ts
// 전역으로 타입을 선언하면 모든 곳에서 자동 적용
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

export type RootStackParamList = {
  Home: undefined;
  Detail: { id: number; title: string };
  Profile: { userId: string };
  Settings: undefined;
};
```

이렇게 글로벌 타입을 선언하면, `useNavigation()`을 호출할 때 제네릭 타입을 매번 지정하지 않아도 자동으로 타입이 추론된다:

```tsx
function AnyScreen() {
  // 글로벌 타입 선언 덕분에 제네릭 없이도 타입 안전
  const navigation = useNavigation();
  navigation.navigate('Detail', { id: 1, title: 'test' }); // 자동 완성 동작!
}
```

---

## 9. 완전한 초기 설정 코드

아래는 처음부터 끝까지 완전히 동작하는 App.tsx 예제다. Android의 Single Activity + Navigation Component 패턴과 동일한 구조를 React Navigation으로 구현한 것이다.

```tsx
// App.tsx - 완전한 초기 설정 (Dynamic API 사용)
import React from 'react';
import {
  View,
  Text,
  Button,
  StyleSheet,
  TouchableOpacity,
} from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import {
  createNativeStackNavigator,
  NativeStackNavigationProp,
} from '@react-navigation/native-stack';
import { RouteProp, useNavigation, useRoute } from '@react-navigation/native';

// ========================================
// 1. 타입 정의 (Android Safe Args 대체)
// ========================================
type RootStackParamList = {
  Home: undefined;
  Detail: { itemId: number; itemTitle: string };
  Settings: undefined;
};

// 글로벌 타입 등록
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// ========================================
// 2. 화면 컴포넌트 (Android Fragment 대체)
// ========================================

// Home 화면
function HomeScreen() {
  const navigation = useNavigation();

  return (
    <View style={styles.container}>
      <Text style={styles.title}>홈 화면</Text>
      <Text style={styles.subtitle}>Android Navigation에서 온 개발자를 환영합니다!</Text>

      <View style={styles.buttonContainer}>
        <Button
          title="아이템 상세 보기"
          onPress={() =>
            navigation.navigate('Detail', {
              itemId: 42,
              itemTitle: 'React Navigation 학습',
            })
          }
        />
      </View>

      <View style={styles.buttonContainer}>
        <Button
          title="설정"
          onPress={() => navigation.navigate('Settings')}
        />
      </View>
    </View>
  );
}

// Detail 화면 - 파라미터를 받는 화면
function DetailScreen() {
  const navigation = useNavigation();
  const route = useRoute<RouteProp<RootStackParamList, 'Detail'>>();

  // Android의 args.itemId와 동일
  const { itemId, itemTitle } = route.params;

  return (
    <View style={styles.container}>
      <Text style={styles.title}>상세 화면</Text>
      <Text style={styles.body}>아이템 ID: {itemId}</Text>
      <Text style={styles.body}>아이템 제목: {itemTitle}</Text>

      <View style={styles.buttonContainer}>
        <Button
          title="다른 상세 페이지로 (push)"
          onPress={() =>
            navigation.push('Detail', {
              itemId: itemId + 1,
              itemTitle: `아이템 ${itemId + 1}`,
            })
          }
        />
      </View>

      <View style={styles.buttonContainer}>
        <Button title="뒤로 가기" onPress={() => navigation.goBack()} />
      </View>
    </View>
  );
}

// Settings 화면
function SettingsScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>설정 화면</Text>
      <Text style={styles.body}>앱 설정을 여기서 관리합니다.</Text>
    </View>
  );
}

// ========================================
// 3. Navigator 생성 (Android NavGraph 대체)
// ========================================
const Stack = createNativeStackNavigator<RootStackParamList>();

// ========================================
// 4. App 컴포넌트 (Android MainActivity 대체)
// ========================================
function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="Home"
        screenOptions={{
          // 모든 화면에 적용되는 기본 옵션
          headerStyle: { backgroundColor: '#6200EE' },  // Android Primary Color
          headerTintColor: '#FFFFFF',                     // 헤더 텍스트/아이콘 색상
          headerTitleStyle: { fontWeight: 'bold' },
          animation: 'slide_from_right',                 // Android 기본 전환과 유사
        }}
      >
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: '홈' }}
        />
        <Stack.Screen
          name="Detail"
          component={DetailScreen}
          options={({ route }) => ({
            title: route.params.itemTitle, // 동적 타이틀
          })}
        />
        <Stack.Screen
          name="Settings"
          component={SettingsScreen}
          options={{ title: '설정' }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// ========================================
// 5. 스타일 (Android XML layout 대체)
// ========================================
const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
    backgroundColor: '#F5F5F5',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 10,
    color: '#212121',
  },
  subtitle: {
    fontSize: 16,
    color: '#757575',
    marginBottom: 30,
    textAlign: 'center',
  },
  body: {
    fontSize: 16,
    color: '#424242',
    marginBottom: 10,
  },
  buttonContainer: {
    marginTop: 10,
    width: '80%',
  },
});

export default App;
```

### Static API 버전

```tsx
// App.tsx - Static API 사용 (v7 권장)
import React from 'react';
import { View, Text, Button, StyleSheet } from 'react-native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import {
  createStaticNavigation,
  useNavigation,
  type StaticParamList,
} from '@react-navigation/native';

function HomeScreen() {
  const navigation = useNavigation();
  return (
    <View style={styles.container}>
      <Text style={styles.title}>홈</Text>
      <Button
        title="상세 보기"
        onPress={() => navigation.navigate('Detail', { id: 1, title: '테스트' })}
      />
    </View>
  );
}

function DetailScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>상세</Text>
    </View>
  );
}

// Static API로 Navigator 생성 — 타입이 자동 추론됨
const RootStack = createNativeStackNavigator({
  initialRouteName: 'Home',
  screenOptions: {
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#fff',
  },
  screens: {
    Home: {
      screen: HomeScreen,
      options: { title: '홈' },
    },
    Detail: {
      screen: DetailScreen,
      options: { title: '상세' },
    },
  },
});

// 타입 자동 생성
type RootStackParamList = StaticParamList<typeof RootStack>;

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// Static Navigation 생성
const Navigation = createStaticNavigation(RootStack);

export default function App() {
  return <Navigation />;
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  title: { fontSize: 24, fontWeight: 'bold' },
});
```

---

## 요약: Android 개발자가 기억해야 할 핵심 매핑

```
Android                          React Navigation
─────────────────────────────    ─────────────────────────────
NavHostFragment                  NavigationContainer
NavGraph (XML/Kotlin DSL)        Stack.Navigator / Tab.Navigator 등
Fragment (destination)           Screen 컴포넌트
NavController.navigate()         navigation.navigate() / push()
NavController.popBackStack()     navigation.goBack() / pop()
Bundle / Safe Args               route.params + TypeScript
intent-filter (딥링크)            linking config
FragmentTransaction 애니메이션    animation 옵션
OnBackPressedCallback            beforeRemove 이벤트
Activity                         NavigationContainer (단 하나)
BottomNavigationView             createBottomTabNavigator
DrawerLayout + NavigationView    createDrawerNavigator
ViewPager2 + TabLayout           createMaterialTopTabNavigator
```

다음 문서에서는 Stack Navigation을 깊이 있게 다루며, Fragment Transaction에서 전환하는 방법을 상세히 설명한다.
