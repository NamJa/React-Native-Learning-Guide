# Stack Navigation 완전 가이드 — Android Fragment Transaction에서 전환하기

## 목차
1. [NativeStack vs JS Stack](#1-nativestack-vs-js-stack)
2. [화면 생성과 등록](#2-화면-생성과-등록)
3. [화면 간 이동: navigate, push, goBack, popTo, replace](#3-화면-간-이동)
4. [파라미터 전달과 TypeScript 타입](#4-파라미터-전달과-typescript-타입)
5. [Screen Options: 헤더 커스터마이징](#5-screen-options-헤더-커스터마이징)
6. [커스텀 헤더 컴포넌트](#6-커스텀-헤더-컴포넌트)
7. [화면 전환 애니메이션](#7-화면-전환-애니메이션)
8. [중첩 네비게이터](#8-중첩-네비게이터)
9. [모달 프레젠테이션 패턴](#9-모달-프레젠테이션-패턴)
10. [인증 플로우 패턴](#10-인증-플로우-패턴)
11. [딥링크 설정](#11-딥링크-설정)
12. [전체 동작 예제](#12-전체-동작-예제)

---

## 1. NativeStack vs JS Stack

React Navigation은 두 가지 Stack Navigator를 제공한다. Android 개발자 입장에서 이 차이를 이해하는 것이 중요하다.

### createNativeStackNavigator (권장)

```bash
npm install @react-navigation/native-stack
```

```tsx
import { createNativeStackNavigator } from '@react-navigation/native-stack';
const Stack = createNativeStackNavigator();
```

**내부 동작**: Android에서는 `Fragment`와 `FragmentTransaction`을 실제로 사용한다. iOS에서는 `UINavigationController`를 사용한다. 즉, 네이티브 플랫폼의 네비게이션 시스템을 그대로 활용한다.

**장점**:
- 네이티브 성능 (60fps 전환 애니메이션)
- 메모리 효율적 (비활성 화면은 네이티브 레벨에서 최적화)
- 플랫폼 기본 제스처 지원 (iOS 스와이프 백, Android 시스템 백)
- 네이티브 헤더 바 (Android Toolbar와 동일)

**단점**:
- 커스터마이징에 일부 제한 (네이티브 API의 한계)
- 헤더 커스터마이징이 JS Stack보다 제한적

### createStackNavigator (JS 기반)

```bash
npm install @react-navigation/stack
npm install react-native-gesture-handler
```

```tsx
import { createStackNavigator } from '@react-navigation/stack';
const Stack = createStackNavigator();
```

**내부 동작**: 모든 화면을 JavaScript(React)로 렌더링하고, 전환 애니메이션도 JS/Reanimated로 처리한다. Android 관점에서 보면, 하나의 View 위에서 모든 화면을 직접 관리하는 것과 같다.

**장점**:
- 완전한 커스터마이징 가능 (커스텀 전환 애니메이션, 헤더 등)
- 공유 요소 전환(Shared Element Transition) 지원이 더 유연

**단점**:
- JS 스레드에서 애니메이션 처리하므로 성능이 약간 떨어질 수 있음
- 메모리 사용량이 더 높음 (모든 화면이 메모리에 상주)

### 어떤 것을 사용할까?

```
90%의 경우: createNativeStackNavigator (성능, 네이티브 경험)
10%의 경우: createStackNavigator (고도로 커스텀된 전환 애니메이션 필요 시)
```

이 문서에서는 `createNativeStackNavigator`를 기준으로 설명한다.

---

## 2. 화면 생성과 등록

### 2-1. 기본 화면 컴포넌트

Android에서 Fragment를 만들고 NavGraph에 등록하는 것처럼, React Navigation에서는 컴포넌트를 만들고 Navigator에 Screen으로 등록한다.

```tsx
// screens/HomeScreen.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

// Android Fragment의 onCreateView()에서 반환하는 View와 동일한 개념
function HomeScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>홈 화면</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  title: { fontSize: 24, fontWeight: 'bold' },
});

export default HomeScreen;
```

### 2-2. Navigator에 화면 등록

```tsx
// navigation/AppNavigator.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import HomeScreen from '../screens/HomeScreen';
import DetailScreen from '../screens/DetailScreen';
import ProfileScreen from '../screens/ProfileScreen';

type RootStackParamList = {
  Home: undefined;
  Detail: { id: number };
  Profile: { userId: string };
};

const Stack = createNativeStackNavigator<RootStackParamList>();

function AppNavigator() {
  return (
    <Stack.Navigator
      initialRouteName="Home"  // Android nav_graph의 app:startDestination
      screenOptions={{
        // 모든 화면에 적용되는 기본 옵션
        headerStyle: { backgroundColor: '#6200EE' },
        headerTintColor: '#fff',
      }}
    >
      {/* 각 Screen이 NavGraph의 <fragment> 태그에 해당 */}
      <Stack.Screen
        name="Home"           // 고유 식별자 (Android의 destination id)
        component={HomeScreen} // 표시할 컴포넌트 (Android의 Fragment 클래스)
        options={{ title: '홈' }} // 화면별 옵션
      />
      <Stack.Screen
        name="Detail"
        component={DetailScreen}
        options={{ title: '상세' }}
      />
      <Stack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: '프로필' }}
      />
    </Stack.Navigator>
  );
}

export default AppNavigator;
```

### 2-3. Screen에 인라인 컴포넌트 사용 (비추천)

```tsx
// 비추천: 렌더링 시마다 새 컴포넌트가 생성됨
<Stack.Screen name="Home">
  {() => <Text>Home</Text>}
</Stack.Screen>

// 추천: 별도 컴포넌트로 분리
<Stack.Screen name="Home" component={HomeScreen} />
```

인라인 컴포넌트는 부모가 리렌더링될 때마다 새로운 함수가 생성되어 불필요한 마운트/언마운트가 발생한다. Android에서 Fragment를 매번 new로 생성하는 것과 비슷한 안티패턴이다.

---

## 3. 화면 간 이동

Android의 NavController가 제공하는 다양한 네비게이션 메서드에 대응하는 React Navigation 메서드들이 있다.

### 3-1. navigation.navigate()

```tsx
// Android: findNavController().navigate(R.id.detailFragment)
// React Navigation:
function HomeScreen() {
  const navigation = useNavigation();

  const goToDetail = () => {
    // 스택에 이미 Detail이 있으면 그 화면으로 돌아감
    // Android의 popUpTo + navigate 조합과 유사
    navigation.navigate('Detail', { id: 1 });
  };

  return <Button title="상세 보기" onPress={goToDetail} />;
}
```

**핵심 동작**: `navigate()`는 대상 화면이 이미 스택에 있으면 해당 화면까지 팝(pop)한 후 파라미터를 업데이트한다. 스택에 없으면 새로 푸시한다. 이는 Android의 `popUpTo` + `navigate` 조합과 유사한 동작이다.

### 3-2. navigation.push()

```tsx
// Android: 중복 destination을 스택에 쌓는 navigate()
// React Navigation:
function DetailScreen() {
  const navigation = useNavigation();
  const route = useRoute();

  const goToNextDetail = () => {
    // 항상 새 화면을 스택에 쌓음 (같은 화면이라도)
    navigation.push('Detail', { id: route.params.id + 1 });
  };

  return <Button title="다음 아이템" onPress={goToNextDetail} />;
}
```

**navigate vs push 비교**:
```
초기 스택: [Home]

navigate('Detail', {id:1}) → [Home, Detail(1)]
navigate('Detail', {id:2}) → [Home, Detail(2)]  ← Detail(1)이 업데이트됨

push('Detail', {id:1}) → [Home, Detail(1)]
push('Detail', {id:2}) → [Home, Detail(1), Detail(2)]  ← 새로 쌓임
```

### 3-3. navigation.goBack()

```tsx
// Android: findNavController().popBackStack()
function DetailScreen() {
  const navigation = useNavigation();

  return (
    <Button
      title="뒤로 가기"
      onPress={() => navigation.goBack()}
    />
  );
}
```

`goBack()`은 현재 화면을 스택에서 제거하고 이전 화면으로 돌아간다. 스택에 화면이 하나뿐이면 아무 동작도 하지 않는다(앱이 종료되지 않음). `navigation.canGoBack()`으로 뒤로 갈 수 있는지 확인할 수 있다.

### 3-4. navigation.popTo() (v7 신규)

```tsx
// Android: findNavController().popBackStack(R.id.homeFragment, false)
function SettingsScreen() {
  const navigation = useNavigation();

  return (
    <Button
      title="홈으로 돌아가기"
      onPress={() => {
        // Home 화면까지 모든 중간 화면을 팝
        navigation.popTo('Home');
      }}
    />
  );
}
```

`popTo()`는 v7에서 새로 도입된 메서드로, 지정된 화면까지 스택을 되감는다. Android의 `popBackStack(destinationId, inclusive=false)`와 동일한 동작이다.

```
스택: [Home, List, Detail, Settings]
popTo('Home') → [Home]
popTo('List') → [Home, List]
```

### 3-5. navigation.pop(n)

```tsx
// Stack Navigator에서만 사용 가능
function DeepScreen() {
  const navigation = useNavigation();

  return (
    <View>
      <Button
        title="1개 뒤로"
        onPress={() => navigation.pop(1)}
      />
      <Button
        title="3개 뒤로"
        onPress={() => navigation.pop(3)}
      />
    </View>
  );
}
```

### 3-6. navigation.replace()

```tsx
// Android: navigate() with popUpTo(self, inclusive=true) + launchSingleTop
function LoginScreen() {
  const navigation = useNavigation();

  const onLoginSuccess = () => {
    // 현재 화면(Login)을 Home으로 교체
    // 뒤로 가기 시 Login으로 돌아가지 않음
    navigation.replace('Home');
  };

  return <Button title="로그인" onPress={onLoginSuccess} />;
}
```

`replace()`는 현재 화면을 새 화면으로 교체한다. 백스택에 현재 화면이 남지 않는다. 로그인 성공 후 홈 화면으로 이동할 때 유용하다.

### 3-7. navigation.reset()

```tsx
// Android: NavController.navigate() with popUpTo(graph, inclusive=true)
function LogoutButton() {
  const navigation = useNavigation();

  const onLogout = () => {
    // 전체 네비게이션 스택을 초기화하고 Login 화면만 남김
    navigation.reset({
      index: 0,
      routes: [{ name: 'Login' }],
    });
  };

  return <Button title="로그아웃" onPress={onLogout} />;
}
```

`reset()`은 전체 네비게이션 상태를 새로 설정한다. 로그아웃 시 모든 화면을 제거하고 로그인 화면만 남기는 패턴에 사용한다.

```exercise
type: code-arrange
question: "Stack Navigator에서 화면 전환 코드를 조립하세요"
tokens:
  - "navigation"
  - ".navigate"
  - "('Detail'"
  - ","
  - "{ id: 42 }"
  - ")"
distractors:
  - ".push"
  - "startActivity"
  - "Intent"
answer: ["navigation", ".navigate", "('Detail'", ",", "{ id: 42 }", ")"]
hint: "navigation.navigate()에 화면 이름과 params 객체를 전달합니다"
xp: 8
```

---

## 4. 파라미터 전달과 TypeScript 타입

### 4-1. 타입 정의

```tsx
// navigation/types.ts
export type RootStackParamList = {
  Home: undefined;
  ProductList: { category: string };
  ProductDetail: {
    productId: number;
    productName: string;
    price?: number;        // 선택적 파라미터
  };
  Cart: undefined;
  Checkout: { totalAmount: number };
};

// 글로벌 타입 등록 — 이후 모든 곳에서 타입 체크
declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

### 4-2. 파라미터 보내기

```tsx
function ProductListScreen() {
  const navigation = useNavigation();

  const products = [
    { id: 1, name: '노트북', price: 1500000 },
    { id: 2, name: '키보드', price: 150000 },
  ];

  return (
    <FlatList
      data={products}
      renderItem={({ item }) => (
        <TouchableOpacity
          onPress={() =>
            // TypeScript가 필수/선택 파라미터를 검증
            navigation.navigate('ProductDetail', {
              productId: item.id,
              productName: item.name,
              price: item.price, // 선택적이므로 생략 가능
            })
          }
        >
          <Text>{item.name}</Text>
        </TouchableOpacity>
      )}
    />
  );
}
```

### 4-3. 파라미터 받기

```tsx
import { useRoute, RouteProp } from '@react-navigation/native';

// 방법 1: useRoute 훅 사용 (함수 컴포넌트)
function ProductDetailScreen() {
  const route = useRoute<RouteProp<RootStackParamList, 'ProductDetail'>>();

  const { productId, productName, price } = route.params;
  // productId: number (자동 타입 추론)
  // productName: string
  // price: number | undefined

  return (
    <View>
      <Text>상품명: {productName}</Text>
      <Text>가격: {price ? `${price.toLocaleString()}원` : '가격 미정'}</Text>
    </View>
  );
}

// 방법 2: props에서 직접 받기
type DetailProps = {
  route: RouteProp<RootStackParamList, 'ProductDetail'>;
  navigation: NativeStackNavigationProp<RootStackParamList, 'ProductDetail'>;
};

function ProductDetailScreen({ route, navigation }: DetailProps) {
  const { productId, productName } = route.params;
  // ...
}
```

### 4-4. 파라미터 업데이트 (이전 화면으로 결과 전달)

Android에서는 `setFragmentResult()` / `setFragmentResultListener()`를 사용하지만, React Navigation에서는 `navigation.navigate()`로 이전 화면의 파라미터를 업데이트한다.

```tsx
// 화면 A: 결과를 기다리는 화면
function ScreenA() {
  const route = useRoute();
  const navigation = useNavigation();

  // 화면 B에서 돌아올 때 파라미터가 업데이트됨
  React.useEffect(() => {
    if (route.params?.selectedItem) {
      console.log('선택된 아이템:', route.params.selectedItem);
    }
  }, [route.params?.selectedItem]);

  return (
    <Button
      title="아이템 선택하기"
      onPress={() => navigation.navigate('ScreenB')}
    />
  );
}

// 화면 B: 결과를 반환하는 화면
function ScreenB() {
  const navigation = useNavigation();

  const selectItem = (item: string) => {
    // 이전 화면으로 돌아가면서 결과 전달
    navigation.navigate('ScreenA', { selectedItem: item });
  };

  return (
    <View>
      <Button title="사과 선택" onPress={() => selectItem('사과')} />
      <Button title="바나나 선택" onPress={() => selectItem('바나나')} />
    </View>
  );
}
```

### 4-5. 초기 파라미터 설정

```tsx
<Stack.Screen
  name="ProductList"
  component={ProductListScreen}
  initialParams={{ category: 'all' }}  // 기본값 설정
/>
```

---

## 5. Screen Options: 헤더 커스터마이징

Android의 Toolbar/ActionBar 커스터마이징에 해당하는 부분이다.

### 5-1. 기본 옵션

```tsx
<Stack.Screen
  name="Home"
  component={HomeScreen}
  options={{
    // 기본 설정
    title: '홈',                    // Android: toolbar.title
    headerShown: true,              // 헤더 표시 여부 (false로 숨김)

    // 헤더 스타일
    headerStyle: {
      backgroundColor: '#6200EE',   // Android: toolbar background
    },
    headerTintColor: '#FFFFFF',     // 뒤로가기 버튼, 타이틀 색상
    headerTitleStyle: {
      fontWeight: 'bold',
      fontSize: 18,
    },

    // 타이틀 정렬
    headerTitleAlign: 'center',     // 'left' | 'center'
    // Android는 기본 'left', iOS는 기본 'center'

    // 뒤로가기 버튼
    headerBackTitle: '뒤로',         // iOS 전용: 뒤로가기 텍스트
    headerBackVisible: true,         // 뒤로가기 버튼 표시 여부

    // 상태바
    statusBarStyle: 'light',         // 'dark' | 'light'
    statusBarColor: '#6200EE',       // Android 전용
  }}
/>
```

### 5-2. 동적 옵션 (route params 기반)

```tsx
// Android에서 Fragment에서 toolbar 타이틀을 동적으로 바꾸는 것과 동일
<Stack.Screen
  name="ProductDetail"
  component={ProductDetailScreen}
  options={({ route, navigation }) => ({
    title: route.params.productName,
    headerRight: () => (
      <TouchableOpacity onPress={() => navigation.navigate('Cart')}>
        <Text style={{ color: '#fff', fontSize: 16 }}>장바구니</Text>
      </TouchableOpacity>
    ),
  })}
/>
```

### 5-3. 화면 내부에서 옵션 변경

```tsx
// Android: requireActivity().supportActionBar?.title = "새 타이틀"
function ProductDetailScreen() {
  const navigation = useNavigation();

  // 마운트 시 헤더 옵션 변경
  React.useLayoutEffect(() => {
    navigation.setOptions({
      title: '동적으로 변경된 타이틀',
      headerRight: () => (
        <TouchableOpacity onPress={handleShare}>
          <Text style={{ color: '#fff' }}>공유</Text>
        </TouchableOpacity>
      ),
    });
  }, [navigation]);

  return <Text>상품 상세</Text>;
}
```

### 5-4. headerRight / headerLeft

```tsx
<Stack.Screen
  name="Home"
  component={HomeScreen}
  options={{
    title: '홈',
    // Android: toolbar의 menu item과 유사
    headerRight: () => (
      <View style={{ flexDirection: 'row', gap: 15 }}>
        <TouchableOpacity onPress={handleSearch}>
          <Text>검색</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={handleMore}>
          <Text>더보기</Text>
        </TouchableOpacity>
      </View>
    ),
    // Android: toolbar의 navigation icon (햄버거 메뉴 등)
    headerLeft: () => (
      <TouchableOpacity onPress={openDrawer}>
        <Text>메뉴</Text>
      </TouchableOpacity>
    ),
  }}
/>
```

### 5-5. 전역 기본 옵션 (screenOptions)

```tsx
// 모든 화면에 적용 — Android의 Theme/Style과 유사
<Stack.Navigator
  screenOptions={{
    headerStyle: { backgroundColor: '#6200EE' },
    headerTintColor: '#fff',
    headerTitleStyle: { fontWeight: 'bold' },
    animation: 'slide_from_right',
    headerShadowVisible: true,   // 헤더 하단 그림자
    contentStyle: {               // 화면 컨텐츠 배경
      backgroundColor: '#F5F5F5',
    },
  }}
>
  {/* 이 Navigator의 모든 Screen에 적용 */}
</Stack.Navigator>
```

### 5-6. 헤더 숨기기

```tsx
// 특정 화면에서 헤더 숨기기 (전체화면)
<Stack.Screen
  name="FullScreenVideo"
  component={VideoScreen}
  options={{ headerShown: false }}
/>

// 전체 Navigator에서 헤더 숨기기
<Stack.Navigator screenOptions={{ headerShown: false }}>
  {/* 모든 화면에 헤더 없음 */}
</Stack.Navigator>
```

---

## 6. 커스텀 헤더 컴포넌트

Android에서 `setSupportActionBar()`로 커스텀 Toolbar를 설정하는 것처럼, React Navigation에서도 완전히 커스텀한 헤더를 만들 수 있다.

```tsx
// components/CustomHeader.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import type { NativeStackHeaderProps } from '@react-navigation/native-stack';

function CustomHeader({ navigation, route, options, back }: NativeStackHeaderProps) {
  const title = options.title ?? route.name;

  return (
    <View style={styles.header}>
      {/* 뒤로가기 버튼: back이 있으면 이전 화면 존재 */}
      {back && (
        <TouchableOpacity
          style={styles.backButton}
          onPress={navigation.goBack}
        >
          <Text style={styles.backText}>← 뒤로</Text>
        </TouchableOpacity>
      )}

      {/* 타이틀 */}
      <Text style={styles.title}>{title}</Text>

      {/* 우측 액션 */}
      <View style={styles.rightActions}>
        {options.headerRight?.({ canGoBack: !!back })}
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  header: {
    height: 60,
    backgroundColor: '#6200EE',
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    elevation: 4,                    // Android 그림자
    shadowColor: '#000',             // iOS 그림자
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
  },
  backButton: {
    marginRight: 16,
  },
  backText: {
    color: '#fff',
    fontSize: 16,
  },
  title: {
    flex: 1,
    color: '#fff',
    fontSize: 18,
    fontWeight: 'bold',
  },
  rightActions: {
    flexDirection: 'row',
  },
});

export default CustomHeader;
```

적용:

```tsx
<Stack.Navigator
  screenOptions={{
    header: (props) => <CustomHeader {...props} />,
  }}
>
  <Stack.Screen name="Home" component={HomeScreen} />
</Stack.Navigator>
```

---

## 7. 화면 전환 애니메이션

Android의 `anim/` 리소스 XML이나 Compose Navigation의 `AnimatedContentTransitionScope`에 해당하는 부분이다.

### 7-1. 기본 제공 애니메이션

```tsx
<Stack.Screen
  name="Detail"
  component={DetailScreen}
  options={{
    // animation 속성으로 전환 애니메이션 지정
    animation: 'slide_from_right',   // 오른쪽에서 슬라이드 (기본값)
  }}
/>
```

사용 가능한 `animation` 값:

| 값 | 설명 | Android 대응 |
|---|------|-------------|
| `'slide_from_right'` | 오른쪽에서 슬라이드 (기본) | `R.anim.slide_in_right` |
| `'slide_from_left'` | 왼쪽에서 슬라이드 | `R.anim.slide_in_left` |
| `'slide_from_bottom'` | 아래에서 슬라이드 | 모달 전환 |
| `'fade'` | 페이드 인/아웃 | `R.anim.fade_in` |
| `'fade_from_bottom'` | 아래에서 페이드 | Material 전환 |
| `'flip'` | 플립 | - |
| `'simple_push'` | 단순 푸시 | 기본 Fragment 전환 |
| `'none'` | 애니메이션 없음 | - |
| `'default'` | 플랫폼 기본값 | - |
| `'ios_from_right'` | iOS 스타일 | - |

### 7-2. 프레젠테이션 모드

```tsx
<Stack.Screen
  name="Modal"
  component={ModalScreen}
  options={{
    // presentation 속성으로 화면 표시 방식 지정
    presentation: 'modal',           // 모달로 표시
  }}
/>
```

사용 가능한 `presentation` 값:

| 값 | 설명 |
|---|------|
| `'card'` | 일반 카드형 (기본값) |
| `'modal'` | 모달 (아래에서 올라옴) |
| `'transparentModal'` | 투명 배경 모달 |
| `'containedModal'` | 컨테이너 안 모달 |
| `'containedTransparentModal'` | 컨테이너 안 투명 모달 |
| `'fullScreenModal'` | 전체화면 모달 |
| `'formSheet'` | 폼 시트 (iPad) |

### 7-3. 애니메이션 조합

```tsx
<Stack.Screen
  name="Settings"
  component={SettingsScreen}
  options={{
    animation: 'slide_from_bottom',
    presentation: 'modal',
    // 제스처 방향
    gestureDirection: 'vertical',    // 세로 스와이프로 닫기
    // 제스처 활성화
    gestureEnabled: true,
    // 전체화면 제스처 (iOS)
    fullScreenGestureEnabled: true,
  }}
/>
```

### 7-4. Screen Group으로 공통 옵션 적용

```tsx
<Stack.Navigator>
  {/* 일반 화면 그룹 */}
  <Stack.Group screenOptions={{ animation: 'slide_from_right' }}>
    <Stack.Screen name="Home" component={HomeScreen} />
    <Stack.Screen name="Detail" component={DetailScreen} />
  </Stack.Group>

  {/* 모달 그룹 */}
  <Stack.Group
    screenOptions={{
      presentation: 'modal',
      animation: 'slide_from_bottom',
    }}
  >
    <Stack.Screen name="CreatePost" component={CreatePostScreen} />
    <Stack.Screen name="Filter" component={FilterScreen} />
  </Stack.Group>
</Stack.Navigator>
```

---

## 8. 중첩 네비게이터

Android에서 BottomNavigationView의 각 탭이 별도의 NavGraph를 갖는 것처럼, React Navigation에서도 Navigator를 중첩할 수 있다.

### 8-1. Tab 안에 Stack (가장 흔한 패턴)

```tsx
// 각 탭이 독립적인 Stack을 가짐
// Android: BottomNavigationView + 각 탭별 NavGraph

// 홈 탭의 Stack
const HomeStack = createNativeStackNavigator();
function HomeStackScreen() {
  return (
    <HomeStack.Navigator>
      <HomeStack.Screen name="HomeMain" component={HomeScreen} />
      <HomeStack.Screen name="HomeDetail" component={DetailScreen} />
    </HomeStack.Navigator>
  );
}

// 프로필 탭의 Stack
const ProfileStack = createNativeStackNavigator();
function ProfileStackScreen() {
  return (
    <ProfileStack.Navigator>
      <ProfileStack.Screen name="ProfileMain" component={ProfileScreen} />
      <ProfileStack.Screen name="EditProfile" component={EditProfileScreen} />
    </ProfileStack.Navigator>
  );
}

// 메인 Tab Navigator
const Tab = createBottomTabNavigator();
function MainTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen
        name="HomeTab"
        component={HomeStackScreen}
        options={{ headerShown: false }}  // Stack이 자체 헤더를 가지므로
      />
      <Tab.Screen
        name="ProfileTab"
        component={ProfileStackScreen}
        options={{ headerShown: false }}
      />
    </Tab.Navigator>
  );
}
```

### 8-2. Stack 안에 Tab

```tsx
// 전체 앱을 감싸는 Root Stack
// Android: Activity의 NavHostFragment > BottomNav > 각 탭 NavGraph

const RootStack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <RootStack.Navigator>
        {/* Tab Navigator를 하나의 Screen으로 등록 */}
        <RootStack.Screen
          name="MainTabs"
          component={MainTabs}
          options={{ headerShown: false }}
        />
        {/* 탭 위에 표시되는 전체 화면 (모달 등) */}
        <RootStack.Screen
          name="PostCreate"
          component={PostCreateScreen}
          options={{ presentation: 'modal' }}
        />
        <RootStack.Screen
          name="Settings"
          component={SettingsScreen}
        />
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### 8-3. 중첩 네비게이터 간 이동

```tsx
// 깊은 중첩 화면으로 이동
function SomeScreen() {
  const navigation = useNavigation();

  // 다른 탭의 특정 화면으로 이동
  navigation.navigate('HomeTab', {
    screen: 'HomeDetail',
    params: { id: 42 },
  });

  // 또는 중첩이 더 깊은 경우
  navigation.navigate('MainTabs', {
    screen: 'HomeTab',
    params: {
      screen: 'HomeDetail',
      params: { id: 42 },
    },
  });
}
```

**주의**: 중첩이 깊어지면 navigate 호출이 복잡해진다. 이는 Android에서 여러 NavGraph 간 이동이 복잡해지는 것과 같다. 가능하면 중첩을 2단계 이내로 유지하는 것이 좋다.

---

## 9. 모달 프레젠테이션 패턴

Android의 `DialogFragment`나 `BottomSheetDialogFragment`에 해당하는 패턴이다.

### 9-1. 기본 모달

```tsx
const RootStack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <RootStack.Navigator>
        {/* 일반 화면 */}
        <RootStack.Screen name="Home" component={HomeScreen} />

        {/* 모달 화면 — 아래에서 올라옴 */}
        <RootStack.Screen
          name="CreateModal"
          component={CreateScreen}
          options={{
            presentation: 'modal',
            animation: 'slide_from_bottom',
          }}
        />
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### 9-2. 투명 모달 (오버레이)

```tsx
// Android의 DialogFragment와 유사
<RootStack.Screen
  name="Alert"
  component={AlertScreen}
  options={{
    presentation: 'transparentModal',
    animation: 'fade',
    headerShown: false,
  }}
/>

// AlertScreen.tsx
function AlertScreen() {
  const navigation = useNavigation();

  return (
    <View style={styles.overlay}>
      <View style={styles.dialog}>
        <Text style={styles.dialogTitle}>확인</Text>
        <Text>정말 삭제하시겠습니까?</Text>
        <View style={styles.buttonRow}>
          <Button title="취소" onPress={() => navigation.goBack()} />
          <Button title="삭제" onPress={handleDelete} />
        </View>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)', // 반투명 배경
    justifyContent: 'center',
    alignItems: 'center',
  },
  dialog: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 24,
    width: '80%',
    elevation: 8,
  },
  dialogTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 12,
  },
  buttonRow: {
    flexDirection: 'row',
    justifyContent: 'flex-end',
    gap: 12,
    marginTop: 20,
  },
});
```

---

## 10. 인증 플로우 패턴

Android에서 로그인 상태에 따라 NavGraph의 `startDestination`을 바꾸는 것과 동일한 패턴이다. React Navigation에서는 **조건부 렌더링**으로 구현한다.

### 10-1. Dynamic API로 인증 플로우

```tsx
// contexts/AuthContext.tsx
import React, { createContext, useContext, useState } from 'react';

type AuthContextType = {
  isLoggedIn: boolean;
  token: string | null;
  login: (token: string) => void;
  logout: () => void;
};

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [token, setToken] = useState<string | null>(null);

  const login = (newToken: string) => setToken(newToken);
  const logout = () => setToken(null);

  return (
    <AuthContext.Provider
      value={{ isLoggedIn: !!token, token, login, logout }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

```tsx
// App.tsx
function AppNavigator() {
  const { isLoggedIn } = useAuth();

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {isLoggedIn ? (
        // 로그인 상태: 메인 앱 화면
        <>
          <Stack.Screen name="MainTabs" component={MainTabs} />
          <Stack.Screen name="Settings" component={SettingsScreen} />
        </>
      ) : (
        // 비로그인 상태: 인증 화면
        <>
          <Stack.Screen name="Login" component={LoginScreen} />
          <Stack.Screen name="Register" component={RegisterScreen} />
          <Stack.Screen name="ForgotPassword" component={ForgotPasswordScreen} />
        </>
      )}
    </Stack.Navigator>
  );
}

function App() {
  return (
    <AuthProvider>
      <NavigationContainer>
        <AppNavigator />
      </NavigationContainer>
    </AuthProvider>
  );
}
```

**핵심 포인트**: `isLoggedIn`이 변경되면 React Navigation이 자동으로 적절한 화면으로 전환한다. 직접 `navigation.reset()`을 호출할 필요가 없다. 이는 Android에서 `MutableStateFlow<Boolean>`을 observe하다가 NavGraph를 교체하는 것과 유사하지만, 훨씬 간결하다.

### 10-2. Static API로 인증 플로우

```tsx
// 커스텀 훅
function useIsSignedIn() {
  const { isLoggedIn } = useAuth();
  return isLoggedIn;
}

function useIsSignedOut() {
  const { isLoggedIn } = useAuth();
  return !isLoggedIn;
}

const RootStack = createNativeStackNavigator({
  screens: {
    // 항상 존재하는 화면
    MainTabs: {
      screen: MainTabsScreen,
      options: { headerShown: false },
    },
  },
  groups: {
    // 로그아웃 상태에서만 보이는 그룹
    Auth: {
      if: useIsSignedOut,
      screenOptions: { headerShown: false },
      screens: {
        Login: LoginScreen,
        Register: RegisterScreen,
      },
    },
    // 로그인 상태에서만 보이는 그룹
    User: {
      if: useIsSignedIn,
      screens: {
        Settings: SettingsScreen,
        EditProfile: EditProfileScreen,
      },
    },
  },
});
```

---

## 11. 딥링크 설정

Android의 `AndroidManifest.xml` intent-filter에 해당하는 딥링크 설정이다.

### 11-1. 기본 딥링크 설정

```tsx
// Android에서의 딥링크:
// <intent-filter>
//   <action android:name="android.intent.action.VIEW" />
//   <category android:name="android.intent.category.DEFAULT" />
//   <data android:scheme="myapp" android:host="product" />
// </intent-filter>

// React Navigation에서의 딥링크:
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],

  config: {
    screens: {
      Home: 'home',
      ProductDetail: 'product/:productId',
      Profile: 'user/:userId',

      // 중첩 네비게이터
      MainTabs: {
        screens: {
          HomeTab: {
            screens: {
              HomeMain: 'feed',
              HomeDetail: 'feed/:id',
            },
          },
        },
      },
    },
  },
};

function App() {
  return (
    <NavigationContainer linking={linking} fallback={<LoadingScreen />}>
      <RootStack.Navigator>
        {/* ... */}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### 11-2. 딥링크 테스트

```bash
# Android에서 딥링크 테스트
adb shell am start -W -a android.intent.action.VIEW -d "myapp://product/42" com.myapp

# iOS에서 딥링크 테스트
npx uri-scheme open "myapp://product/42" --ios

# React Navigation 디버깅
npx react-native start
# Metro 콘솔에서 딥링크 URL 확인 가능
```

---

## 12. 전체 동작 예제

3개 화면, 파라미터 전달, 커스텀 헤더를 포함한 완전한 예제:

```tsx
// App.tsx — 전체 동작 예제
import React from 'react';
import {
  View, Text, FlatList, TouchableOpacity, Button, StyleSheet, Image,
} from 'react-native';
import { NavigationContainer, useNavigation, useRoute } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import type { RouteProp } from '@react-navigation/native';

// ========================================
// 타입 정의
// ========================================
type Product = {
  id: number;
  name: string;
  price: number;
  description: string;
  category: string;
};

type RootStackParamList = {
  ProductList: undefined;
  ProductDetail: { product: Product };
  Cart: { items: Product[] };
};

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// ========================================
// 더미 데이터
// ========================================
const PRODUCTS: Product[] = [
  { id: 1, name: '맥북 프로 14"', price: 2390000, description: 'M3 Pro 칩 탑재', category: '노트북' },
  { id: 2, name: '에어팟 프로 2', price: 359000, description: 'USB-C 충전 케이스', category: '이어폰' },
  { id: 3, name: '아이패드 에어', price: 929000, description: 'M2 칩 탑재', category: '태블릿' },
  { id: 4, name: '매직 키보드', price: 149000, description: 'Touch ID 포함', category: '악세서리' },
  { id: 5, name: '스튜디오 디스플레이', price: 2299000, description: '5K Retina', category: '모니터' },
];

// ========================================
// 화면 1: 상품 목록
// ========================================
function ProductListScreen() {
  const navigation = useNavigation();

  const renderItem = ({ item }: { item: Product }) => (
    <TouchableOpacity
      style={styles.productCard}
      onPress={() => navigation.navigate('ProductDetail', { product: item })}
      activeOpacity={0.7}
    >
      <View style={styles.productInfo}>
        <Text style={styles.productName}>{item.name}</Text>
        <Text style={styles.productCategory}>{item.category}</Text>
        <Text style={styles.productPrice}>
          {item.price.toLocaleString()}원
        </Text>
      </View>
      <Text style={styles.arrow}>→</Text>
    </TouchableOpacity>
  );

  return (
    <FlatList
      data={PRODUCTS}
      renderItem={renderItem}
      keyExtractor={(item) => item.id.toString()}
      contentContainerStyle={styles.listContainer}
    />
  );
}

// ========================================
// 화면 2: 상품 상세
// ========================================
function ProductDetailScreen() {
  const navigation = useNavigation();
  const route = useRoute<RouteProp<RootStackParamList, 'ProductDetail'>>();
  const { product } = route.params;

  return (
    <View style={styles.detailContainer}>
      <View style={styles.detailCard}>
        <Text style={styles.detailName}>{product.name}</Text>
        <Text style={styles.detailCategory}>{product.category}</Text>
        <Text style={styles.detailDescription}>{product.description}</Text>
        <Text style={styles.detailPrice}>
          {product.price.toLocaleString()}원
        </Text>
      </View>

      <View style={styles.buttonGroup}>
        <TouchableOpacity
          style={styles.primaryButton}
          onPress={() => navigation.navigate('Cart', { items: [product] })}
        >
          <Text style={styles.primaryButtonText}>장바구니에 추가</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={styles.secondaryButton}
          onPress={() => navigation.goBack()}
        >
          <Text style={styles.secondaryButtonText}>목록으로</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

// ========================================
// 화면 3: 장바구니
// ========================================
function CartScreen() {
  const navigation = useNavigation();
  const route = useRoute<RouteProp<RootStackParamList, 'Cart'>>();
  const { items } = route.params;

  const total = items.reduce((sum, item) => sum + item.price, 0);

  return (
    <View style={styles.cartContainer}>
      <Text style={styles.cartTitle}>장바구니</Text>

      {items.map((item) => (
        <View key={item.id} style={styles.cartItem}>
          <Text style={styles.cartItemName}>{item.name}</Text>
          <Text style={styles.cartItemPrice}>
            {item.price.toLocaleString()}원
          </Text>
        </View>
      ))}

      <View style={styles.cartTotal}>
        <Text style={styles.cartTotalLabel}>합계</Text>
        <Text style={styles.cartTotalPrice}>
          {total.toLocaleString()}원
        </Text>
      </View>

      <TouchableOpacity
        style={styles.primaryButton}
        onPress={() => navigation.popTo('ProductList')}
      >
        <Text style={styles.primaryButtonText}>계속 쇼핑하기</Text>
      </TouchableOpacity>
    </View>
  );
}

// ========================================
// Navigator & App
// ========================================
const Stack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="ProductList"
        screenOptions={{
          headerStyle: { backgroundColor: '#1a1a2e' },
          headerTintColor: '#e94560',
          headerTitleStyle: { fontWeight: 'bold', fontSize: 18 },
          contentStyle: { backgroundColor: '#16213e' },
          animation: 'slide_from_right',
        }}
      >
        <Stack.Screen
          name="ProductList"
          component={ProductListScreen}
          options={{
            title: '상품 목록',
            headerRight: () => (
              <TouchableOpacity>
                <Text style={{ color: '#e94560', fontSize: 16 }}>필터</Text>
              </TouchableOpacity>
            ),
          }}
        />
        <Stack.Screen
          name="ProductDetail"
          component={ProductDetailScreen}
          options={({ route }) => ({
            title: route.params.product.name,
          })}
        />
        <Stack.Screen
          name="Cart"
          component={CartScreen}
          options={{
            title: '장바구니',
            presentation: 'modal',
            animation: 'slide_from_bottom',
          }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// ========================================
// 스타일
// ========================================
const styles = StyleSheet.create({
  listContainer: { padding: 16 },
  productCard: {
    backgroundColor: '#0f3460',
    borderRadius: 12,
    padding: 16,
    marginBottom: 12,
    flexDirection: 'row',
    alignItems: 'center',
    elevation: 3,
  },
  productInfo: { flex: 1 },
  productName: { fontSize: 16, fontWeight: 'bold', color: '#fff', marginBottom: 4 },
  productCategory: { fontSize: 12, color: '#a0a0a0', marginBottom: 8 },
  productPrice: { fontSize: 18, fontWeight: 'bold', color: '#e94560' },
  arrow: { fontSize: 24, color: '#e94560' },
  detailContainer: { flex: 1, padding: 20 },
  detailCard: {
    backgroundColor: '#0f3460',
    borderRadius: 16,
    padding: 24,
    marginBottom: 24,
  },
  detailName: { fontSize: 24, fontWeight: 'bold', color: '#fff', marginBottom: 8 },
  detailCategory: { fontSize: 14, color: '#a0a0a0', marginBottom: 16 },
  detailDescription: { fontSize: 16, color: '#ccc', marginBottom: 20 },
  detailPrice: { fontSize: 28, fontWeight: 'bold', color: '#e94560' },
  buttonGroup: { gap: 12 },
  primaryButton: {
    backgroundColor: '#e94560',
    borderRadius: 12,
    padding: 16,
    alignItems: 'center',
  },
  primaryButtonText: { color: '#fff', fontSize: 16, fontWeight: 'bold' },
  secondaryButton: {
    borderColor: '#e94560',
    borderWidth: 2,
    borderRadius: 12,
    padding: 16,
    alignItems: 'center',
  },
  secondaryButtonText: { color: '#e94560', fontSize: 16, fontWeight: 'bold' },
  cartContainer: { flex: 1, padding: 20 },
  cartTitle: { fontSize: 24, fontWeight: 'bold', color: '#fff', marginBottom: 20 },
  cartItem: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    backgroundColor: '#0f3460',
    borderRadius: 8,
    padding: 16,
    marginBottom: 8,
  },
  cartItemName: { fontSize: 16, color: '#fff' },
  cartItemPrice: { fontSize: 16, fontWeight: 'bold', color: '#e94560' },
  cartTotal: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    borderTopWidth: 2,
    borderTopColor: '#e94560',
    paddingTop: 16,
    marginTop: 16,
    marginBottom: 24,
  },
  cartTotalLabel: { fontSize: 20, fontWeight: 'bold', color: '#fff' },
  cartTotalPrice: { fontSize: 20, fontWeight: 'bold', color: '#e94560' },
});
```

---

## 요약: Android Fragment Transaction vs React Navigation Stack

```
Android Fragment                    React Navigation Stack
─────────────────────────────      ─────────────────────────────
fragmentTransaction.replace()      navigation.replace()
fragmentTransaction.add()          navigation.push()
fragmentTransaction.addToBackStack navigation.navigate() (자동)
popBackStack()                     goBack() / pop()
popBackStack(name, 0)              popTo('name')
popBackStack(name, INCLUSIVE)      popTo('name') + navigate('name')
setCustomAnimations()              options.animation
DialogFragment                     presentation: 'modal'
savedInstanceState Bundle          route.params
```

다음 문서에서는 Tab과 Drawer Navigation을 다루며, BottomNavigationView와 DrawerLayout을 대체하는 방법을 설명한다.
