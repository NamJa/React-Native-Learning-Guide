# Tab & Drawer Navigation — BottomNavigationView와 DrawerLayout 대체하기

## 목차
1. [Bottom Tab Navigator](#1-bottom-tab-navigator)
2. [Tab Bar 커스터마이징](#2-tab-bar-커스터마이징)
3. [특정 화면에서 탭 바 숨기기](#3-특정-화면에서-탭-바-숨기기)
4. [Material Top Tab Navigator](#4-material-top-tab-navigator)
5. [Drawer Navigator](#5-drawer-navigator)
6. [네비게이터 조합 패턴](#6-네비게이터-조합-패턴)
7. [전체 앱 네비게이션 구조 예제](#7-전체-앱-네비게이션-구조-예제)

---

## 1. Bottom Tab Navigator

Android의 `BottomNavigationView`를 대체하는 네비게이터다. Material Design의 Bottom Navigation과 동일한 역할을 한다.

### 1-1. 설치

```bash
npm install @react-navigation/bottom-tabs
```

### 1-2. 기본 설정

```tsx
// Android의 BottomNavigationView + NavGraph 설정과 비교
//
// Android:
// 1. activity_main.xml에 BottomNavigationView 추가
// 2. menu/bottom_nav_menu.xml에 메뉴 아이템 정의
// 3. NavHostFragment와 연결: NavigationUI.setupWithNavController(bottomNav, navController)
//
// React Navigation:
// 1. createBottomTabNavigator()로 Tab Navigator 생성
// 2. Tab.Screen으로 각 탭 정의 (메뉴 아이템 + NavGraph destination 한 번에)

import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

// 각 탭의 화면 컴포넌트
function HomeScreen() {
  return (
    <View style={styles.center}>
      <Text style={styles.title}>홈</Text>
    </View>
  );
}

function SearchScreen() {
  return (
    <View style={styles.center}>
      <Text style={styles.title}>검색</Text>
    </View>
  );
}

function ProfileScreen() {
  return (
    <View style={styles.center}>
      <Text style={styles.title}>프로필</Text>
    </View>
  );
}

const Tab = createBottomTabNavigator();

function App() {
  return (
    <NavigationContainer>
      <Tab.Navigator
        initialRouteName="Home"
        screenOptions={{
          tabBarActiveTintColor: '#6200EE',      // 선택된 탭 색상
          tabBarInactiveTintColor: '#757575',     // 비선택 탭 색상
          tabBarStyle: {                           // 탭 바 스타일
            backgroundColor: '#FFFFFF',
            borderTopWidth: 1,
            borderTopColor: '#E0E0E0',
            height: 60,
            paddingBottom: 8,
            paddingTop: 8,
          },
          headerStyle: { backgroundColor: '#6200EE' },
          headerTintColor: '#fff',
        }}
      >
        <Tab.Screen
          name="Home"
          component={HomeScreen}
          options={{
            title: '홈',
            tabBarLabel: '홈',
            tabBarIcon: ({ color, size }) => (
              <Text style={{ color, fontSize: size }}>🏠</Text>
              // 실제 앱에서는 react-native-vector-icons 사용
            ),
          }}
        />
        <Tab.Screen
          name="Search"
          component={SearchScreen}
          options={{
            title: '검색',
            tabBarLabel: '검색',
            tabBarIcon: ({ color, size }) => (
              <Text style={{ color, fontSize: size }}>🔍</Text>
            ),
          }}
        />
        <Tab.Screen
          name="Profile"
          component={ProfileScreen}
          options={{
            title: '프로필',
            tabBarLabel: '프로필',
            tabBarIcon: ({ color, size }) => (
              <Text style={{ color, fontSize: size }}>👤</Text>
            ),
          }}
        />
      </Tab.Navigator>
    </NavigationContainer>
  );
}

const styles = StyleSheet.create({
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  title: { fontSize: 24, fontWeight: 'bold' },
});
```

### 1-3. react-native-vector-icons를 사용한 아이콘

```bash
npm install react-native-vector-icons
npm install -D @types/react-native-vector-icons
```

```tsx
import Icon from 'react-native-vector-icons/MaterialIcons';

<Tab.Screen
  name="Home"
  component={HomeScreen}
  options={{
    tabBarIcon: ({ color, size }) => (
      <Icon name="home" color={color} size={size} />
    ),
  }}
/>
<Tab.Screen
  name="Search"
  component={SearchScreen}
  options={{
    tabBarIcon: ({ color, size }) => (
      <Icon name="search" color={color} size={size} />
    ),
  }}
/>
<Tab.Screen
  name="Notifications"
  component={NotificationsScreen}
  options={{
    tabBarIcon: ({ color, size }) => (
      <Icon name="notifications" color={color} size={size} />
    ),
    // 뱃지 표시 (Android BottomNavigationView의 BadgeDrawable과 동일)
    tabBarBadge: 3,
    tabBarBadgeStyle: {
      backgroundColor: '#FF1744',
      color: '#fff',
      fontSize: 10,
    },
  }}
/>
```

```exercise
type: word-bank
question: "Bottom Tab Navigator 설정의 빈칸을 채우세요"
code: |
  const Tab = ___(
    {
      screens: {
        Home: ___,
        Settings: SettingsScreen,
      }
    }
  );
blanks: ["createBottomTabNavigator", "HomeScreen"]
distractors: ["createStackNavigator", "createDrawerNavigator", "TabScreen", "Fragment"]
hint: "Bottom Tab은 createBottomTabNavigator로 생성합니다"
xp: 5
```

---

## 2. Tab Bar 커스터마이징

### 2-1. 탭 바 스타일 옵션

```tsx
<Tab.Navigator
  screenOptions={{
    // 탭 바 전체 스타일
    tabBarStyle: {
      backgroundColor: '#1a1a2e',
      borderTopWidth: 0,
      elevation: 10,                // Android 그림자
      height: 65,
      paddingBottom: 10,
      paddingTop: 5,
    },

    // 활성/비활성 색상
    tabBarActiveTintColor: '#e94560',
    tabBarInactiveTintColor: '#a0a0a0',

    // 라벨 스타일
    tabBarLabelStyle: {
      fontSize: 11,
      fontWeight: '600',
    },

    // 라벨 표시 여부
    tabBarShowLabel: true,          // false로 하면 아이콘만 표시

    // 키보드 열릴 때 탭 바 숨기기
    tabBarHideOnKeyboard: true,     // Android에서 중요!

    // 아이콘과 라벨 동시에 한 줄 표시
    tabBarLabelPosition: 'below-icon', // 'below-icon' | 'beside-icon'
  }}
>
```

### 2-2. 탭별 개별 스타일

```tsx
<Tab.Screen
  name="Home"
  component={HomeScreen}
  options={{
    tabBarIcon: ({ focused, color, size }) => (
      // focused: 현재 선택된 탭인지 여부
      <View style={focused ? styles.activeTab : styles.inactiveTab}>
        <Icon
          name={focused ? 'home' : 'home-outline'}
          color={color}
          size={size}
        />
      </View>
    ),
    // 이 탭을 눌렀을 때의 동작 커스터마이징
    tabBarButton: (props) => (
      <TouchableOpacity {...props} style={styles.customTabButton} />
    ),
  }}
/>
```

### 2-3. 완전한 커스텀 탭 바

Android에서 `BottomNavigationView`를 사용하지 않고 완전히 커스텀한 바텀 네비게이션을 만드는 것처럼, React Navigation에서도 `tabBar` prop으로 완전히 커스텀할 수 있다.

```tsx
import type { BottomTabBarProps } from '@react-navigation/bottom-tabs';

// 커스텀 탭 바 컴포넌트
function CustomTabBar({ state, descriptors, navigation }: BottomTabBarProps) {
  return (
    <View style={customStyles.tabBar}>
      {state.routes.map((route, index) => {
        const { options } = descriptors[route.key];
        const label = options.tabBarLabel ?? options.title ?? route.name;
        const isFocused = state.index === index;

        const onPress = () => {
          const event = navigation.emit({
            type: 'tabPress',
            target: route.key,
            canPreventDefault: true,
          });

          if (!isFocused && !event.defaultPrevented) {
            navigation.navigate(route.name);
          }
        };

        const onLongPress = () => {
          navigation.emit({
            type: 'tabLongPress',
            target: route.key,
          });
        };

        // 탭 아이콘 가져오기
        const icon = options.tabBarIcon?.({
          focused: isFocused,
          color: isFocused ? '#e94560' : '#a0a0a0',
          size: 24,
        });

        return (
          <TouchableOpacity
            key={route.key}
            accessibilityRole="button"
            accessibilityState={isFocused ? { selected: true } : {}}
            accessibilityLabel={options.tabBarAccessibilityLabel}
            onPress={onPress}
            onLongPress={onLongPress}
            style={customStyles.tabItem}
          >
            {icon}
            <Text
              style={[
                customStyles.tabLabel,
                { color: isFocused ? '#e94560' : '#a0a0a0' },
              ]}
            >
              {typeof label === 'string' ? label : route.name}
            </Text>
            {/* 선택 표시 인디케이터 */}
            {isFocused && <View style={customStyles.activeIndicator} />}
          </TouchableOpacity>
        );
      })}
    </View>
  );
}

const customStyles = StyleSheet.create({
  tabBar: {
    flexDirection: 'row',
    backgroundColor: '#1a1a2e',
    height: 70,
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingHorizontal: 10,
    elevation: 15,
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
  },
  tabItem: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingTop: 8,
  },
  tabLabel: {
    fontSize: 10,
    marginTop: 4,
    fontWeight: '600',
  },
  activeIndicator: {
    width: 4,
    height: 4,
    borderRadius: 2,
    backgroundColor: '#e94560',
    marginTop: 4,
  },
});

// 사용법
<Tab.Navigator tabBar={(props) => <CustomTabBar {...props} />}>
  <Tab.Screen name="Home" component={HomeScreen} />
  <Tab.Screen name="Search" component={SearchScreen} />
  <Tab.Screen name="Profile" component={ProfileScreen} />
</Tab.Navigator>
```

---

## 3. 특정 화면에서 탭 바 숨기기

Android에서 특정 Fragment로 이동할 때 BottomNavigationView를 숨기는 것은 흔한 요구사항이다. React Navigation에서는 **네비게이터 구조**로 이를 해결한다.

### 3-1. 잘못된 방법 (비추천)

```tsx
// 비추천: tabBarStyle로 숨기기 (깜빡임 발생)
<Tab.Screen
  name="Detail"
  component={DetailScreen}
  options={{ tabBarStyle: { display: 'none' } }}
/>
```

### 3-2. 올바른 방법: 네비게이터 구조 변경

탭 바를 숨기고 싶은 화면은 Tab Navigator의 밖(상위 Stack)에 배치한다:

```tsx
// 핵심 아이디어: Tab 안에서만 탭 바가 보임
// Tab 바깥의 Stack에 화면을 추가하면 탭 바가 자동으로 가려짐

const HomeStack = createNativeStackNavigator();
function HomeStackScreen() {
  return (
    <HomeStack.Navigator>
      <HomeStack.Screen name="HomeMain" component={HomeScreen} />
      {/* 이 화면은 Tab 안에 있으므로 탭 바가 보임 */}
    </HomeStack.Navigator>
  );
}

const Tab = createBottomTabNavigator();
function MainTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="HomeTab" component={HomeStackScreen}
        options={{ headerShown: false }} />
      <Tab.Screen name="SearchTab" component={SearchScreen} />
    </Tab.Navigator>
  );
}

const RootStack = createNativeStackNavigator();
function App() {
  return (
    <NavigationContainer>
      <RootStack.Navigator>
        <RootStack.Screen
          name="Main"
          component={MainTabs}
          options={{ headerShown: false }}
        />
        {/* 이 화면들은 Tab 바깥이므로 탭 바가 보이지 않음 */}
        <RootStack.Screen name="Detail" component={DetailScreen} />
        <RootStack.Screen name="Settings" component={SettingsScreen} />
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

이 구조는 Android에서 `BottomNavigationView`가 `MainActivity`에 있고, 상세 화면은 별도의 Activity 또는 전체화면 Fragment로 열리는 패턴과 동일하다.

---

## 4. Material Top Tab Navigator

Android의 `ViewPager2 + TabLayout` 조합을 대체한다. 좌우 스와이프로 탭 전환이 가능한 네비게이터다.

### 4-1. 설치

```bash
npm install @react-navigation/material-top-tabs react-native-tab-view react-native-pager-view
```

- **react-native-tab-view**: Material Design 탭 UI 라이브러리
- **react-native-pager-view**: 네이티브 ViewPager (Android의 ViewPager2와 동일)

### 4-2. 기본 설정

```tsx
import { createMaterialTopTabNavigator } from '@react-navigation/material-top-tabs';

const TopTab = createMaterialTopTabNavigator();

// Android의 ViewPager2 + TabLayout과 동일한 역할
function NewsFeedScreen() {
  return (
    <TopTab.Navigator
      initialRouteName="All"
      screenOptions={{
        // TabLayout 스타일
        tabBarStyle: {
          backgroundColor: '#1a1a2e',
          elevation: 0,
        },
        // 인디케이터 (TabLayout의 indicator)
        tabBarIndicatorStyle: {
          backgroundColor: '#e94560',
          height: 3,
          borderRadius: 1.5,
        },
        // 탭 텍스트 색상
        tabBarActiveTintColor: '#e94560',
        tabBarInactiveTintColor: '#a0a0a0',
        // 탭 라벨 스타일
        tabBarLabelStyle: {
          fontSize: 14,
          fontWeight: 'bold',
          textTransform: 'none', // 대문자 변환 비활성화
        },
        // 스와이프 활성화 (ViewPager2의 isUserInputEnabled)
        swipeEnabled: true,
        // 레이지 로딩 (ViewPager2의 offscreenPageLimit)
        lazy: true,
        lazyPlaceholder: () => (
          <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
            <ActivityIndicator size="large" color="#e94560" />
          </View>
        ),
      }}
    >
      <TopTab.Screen name="All" component={AllNewsScreen} options={{ title: '전체' }} />
      <TopTab.Screen name="Tech" component={TechNewsScreen} options={{ title: '기술' }} />
      <TopTab.Screen name="Sports" component={SportsNewsScreen} options={{ title: '스포츠' }} />
      <TopTab.Screen name="Entertainment" component={EntertainmentScreen} options={{ title: '엔터' }} />
    </TopTab.Navigator>
  );
}
```

### 4-3. 스크롤 가능한 탭 (탭이 많을 때)

```tsx
<TopTab.Navigator
  screenOptions={{
    tabBarScrollEnabled: true,  // Android TabLayout의 MODE_SCROLLABLE과 동일
    tabBarItemStyle: {
      width: 100,               // 각 탭의 고정 너비
    },
  }}
>
  <TopTab.Screen name="Tab1" component={Screen1} options={{ title: '탭 1' }} />
  <TopTab.Screen name="Tab2" component={Screen2} options={{ title: '탭 2' }} />
  <TopTab.Screen name="Tab3" component={Screen3} options={{ title: '탭 3' }} />
  <TopTab.Screen name="Tab4" component={Screen4} options={{ title: '탭 4' }} />
  <TopTab.Screen name="Tab5" component={Screen5} options={{ title: '탭 5' }} />
  <TopTab.Screen name="Tab6" component={Screen6} options={{ title: '탭 6' }} />
</TopTab.Navigator>
```

### 4-4. Android ViewPager2 + TabLayout 비교

```kotlin
// Android: ViewPager2 + TabLayout
class NewsFeedFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val viewPager = view.findViewById<ViewPager2>(R.id.viewPager)
        val tabLayout = view.findViewById<TabLayout>(R.id.tabLayout)

        viewPager.adapter = NewsPagerAdapter(this)

        TabLayoutMediator(tabLayout, viewPager) { tab, position ->
            tab.text = when (position) {
                0 -> "전체"
                1 -> "기술"
                2 -> "스포츠"
                else -> "엔터"
            }
        }.attach()
    }
}
```

```tsx
// React Navigation: 동일한 UI를 훨씬 간결하게
function NewsFeedScreen() {
  return (
    <TopTab.Navigator>
      <TopTab.Screen name="All" component={AllNewsScreen} options={{ title: '전체' }} />
      <TopTab.Screen name="Tech" component={TechNewsScreen} options={{ title: '기술' }} />
      <TopTab.Screen name="Sports" component={SportsNewsScreen} options={{ title: '스포츠' }} />
      <TopTab.Screen name="Entertainment" component={EntScreen} options={{ title: '엔터' }} />
    </TopTab.Navigator>
  );
}
```

---

## 5. Drawer Navigator

Android의 `DrawerLayout + NavigationView` 조합을 대체한다.

### 5-1. 설치

```bash
npm install @react-navigation/drawer
npm install react-native-gesture-handler react-native-reanimated
```

`react-native-gesture-handler`는 Drawer의 스와이프 제스처를 처리하고, `react-native-reanimated`는 부드러운 애니메이션을 제공한다.

**중요**: 앱의 진입점(index.js 또는 App.tsx) 최상단에 gesture handler를 import해야 한다:

```tsx
// index.js 또는 App.tsx 최상단 (다른 import보다 먼저!)
import 'react-native-gesture-handler';
```

`babel.config.js`에 reanimated 플러그인 추가:

```javascript
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: ['react-native-reanimated/plugin'], // 반드시 마지막에!
};
```

### 5-2. 기본 설정

```tsx
import { createDrawerNavigator } from '@react-navigation/drawer';

const Drawer = createDrawerNavigator();

// Android DrawerLayout + NavigationView와 동일한 역할
function AppDrawer() {
  return (
    <Drawer.Navigator
      initialRouteName="Home"
      screenOptions={{
        // Drawer 위치 (Android: DrawerLayout의 gravity)
        drawerPosition: 'left',       // 'left' | 'right'

        // Drawer 스타일
        drawerStyle: {
          backgroundColor: '#1a1a2e',
          width: 280,
        },

        // Drawer 아이템 색상
        drawerActiveTintColor: '#e94560',
        drawerInactiveTintColor: '#a0a0a0',
        drawerActiveBackgroundColor: 'rgba(233, 69, 96, 0.1)',

        // Drawer 아이템 라벨 스타일
        drawerLabelStyle: {
          fontSize: 15,
          fontWeight: '500',
          marginLeft: -16,  // 아이콘과 라벨 간격 조정
        },

        // 헤더 스타일
        headerStyle: { backgroundColor: '#1a1a2e' },
        headerTintColor: '#e94560',

        // Drawer 타입
        drawerType: 'front',          // 'front' | 'back' | 'slide' | 'permanent'
        // front: Drawer가 컨텐츠 위에 오버레이 (Android 기본 DrawerLayout)
        // back: Drawer가 컨텐츠 뒤에
        // slide: 컨텐츠가 밀림
        // permanent: 항상 보임 (태블릿/웹)

        // 오버레이 색상 (Drawer 열릴 때 뒤쪽 배경)
        overlayColor: 'rgba(0, 0, 0, 0.5)',

        // 스와이프 활성화
        swipeEnabled: true,
        swipeEdgeWidth: 50,           // 스와이프 감지 영역 너비
      }}
    >
      <Drawer.Screen
        name="Home"
        component={HomeScreen}
        options={{
          title: '홈',
          drawerIcon: ({ color, size }) => (
            <Icon name="home" color={color} size={size} />
          ),
        }}
      />
      <Drawer.Screen
        name="Favorites"
        component={FavoritesScreen}
        options={{
          title: '즐겨찾기',
          drawerIcon: ({ color, size }) => (
            <Icon name="favorite" color={color} size={size} />
          ),
        }}
      />
      <Drawer.Screen
        name="Settings"
        component={SettingsScreen}
        options={{
          title: '설정',
          drawerIcon: ({ color, size }) => (
            <Icon name="settings" color={color} size={size} />
          ),
        }}
      />
    </Drawer.Navigator>
  );
}
```

### 5-3. 프로그래밍 방식으로 Drawer 제어

```tsx
// Android: drawerLayout.openDrawer(GravityCompat.START)
// React Navigation:

function HomeScreen() {
  const navigation = useNavigation();

  return (
    <View>
      {/* Drawer 열기 */}
      <Button
        title="메뉴 열기"
        onPress={() => navigation.openDrawer()}
      />

      {/* Drawer 닫기 */}
      <Button
        title="메뉴 닫기"
        onPress={() => navigation.closeDrawer()}
      />

      {/* Drawer 토글 */}
      <Button
        title="메뉴 토글"
        onPress={() => navigation.toggleDrawer()}
      />
    </View>
  );
}
```

### 5-4. 커스텀 Drawer 콘텐츠

Android의 `NavigationView`에 헤더 뷰와 커스텀 메뉴를 추가하는 것처럼, React Navigation에서도 완전히 커스텀한 Drawer 콘텐츠를 만들 수 있다.

```tsx
import {
  DrawerContentScrollView,
  DrawerItemList,
  DrawerItem,
} from '@react-navigation/drawer';
import type { DrawerContentComponentProps } from '@react-navigation/drawer';

function CustomDrawerContent(props: DrawerContentComponentProps) {
  const { navigation } = props;

  return (
    <View style={{ flex: 1 }}>
      {/* 프로필 헤더 (Android NavigationView의 headerLayout) */}
      <View style={drawerStyles.header}>
        <View style={drawerStyles.avatar}>
          <Text style={drawerStyles.avatarText}>JW</Text>
        </View>
        <Text style={drawerStyles.userName}>김종우</Text>
        <Text style={drawerStyles.userEmail}>jongwoo@example.com</Text>
      </View>

      {/* 기본 메뉴 아이템 목록 */}
      <DrawerContentScrollView {...props}>
        <DrawerItemList {...props} />

        {/* 구분선 */}
        <View style={drawerStyles.divider} />

        {/* 커스텀 메뉴 아이템 */}
        <DrawerItem
          label="도움말"
          icon={({ color, size }) => (
            <Icon name="help-outline" color={color} size={size} />
          )}
          onPress={() => {
            navigation.closeDrawer();
            // 외부 링크 열기 등
          }}
        />
      </DrawerContentScrollView>

      {/* 하단 고정 영역 */}
      <View style={drawerStyles.footer}>
        <DrawerItem
          label="로그아웃"
          icon={({ color, size }) => (
            <Icon name="logout" color={color} size={size} />
          )}
          onPress={handleLogout}
          labelStyle={{ color: '#FF5252' }}
        />
        <Text style={drawerStyles.version}>v1.0.0</Text>
      </View>
    </View>
  );
}

const drawerStyles = StyleSheet.create({
  header: {
    backgroundColor: '#0f3460',
    padding: 20,
    paddingTop: 50,
    marginBottom: 10,
  },
  avatar: {
    width: 60,
    height: 60,
    borderRadius: 30,
    backgroundColor: '#e94560',
    justifyContent: 'center',
    alignItems: 'center',
    marginBottom: 12,
  },
  avatarText: {
    color: '#fff',
    fontSize: 22,
    fontWeight: 'bold',
  },
  userName: {
    color: '#fff',
    fontSize: 18,
    fontWeight: 'bold',
  },
  userEmail: {
    color: '#a0a0a0',
    fontSize: 13,
    marginTop: 4,
  },
  divider: {
    height: 1,
    backgroundColor: '#333',
    marginVertical: 10,
    marginHorizontal: 16,
  },
  footer: {
    borderTopWidth: 1,
    borderTopColor: '#333',
    paddingBottom: 20,
  },
  version: {
    color: '#666',
    fontSize: 12,
    textAlign: 'center',
    marginTop: 4,
  },
});

// 사용법
<Drawer.Navigator
  drawerContent={(props) => <CustomDrawerContent {...props} />}
>
  {/* ... */}
</Drawer.Navigator>
```

### 5-5. Android DrawerLayout + NavigationView 비교

```kotlin
// Android: DrawerLayout + NavigationView
class MainActivity : AppCompatActivity() {
    private lateinit var drawerLayout: DrawerLayout

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        drawerLayout = findViewById(R.id.drawer_layout)
        val navView = findViewById<NavigationView>(R.id.nav_view)
        val navController = findNavController(R.id.nav_host_fragment)

        // 헤더 뷰 설정
        val headerView = navView.getHeaderView(0)
        headerView.findViewById<TextView>(R.id.userName).text = "김종우"

        // 메뉴 아이템 설정
        NavigationUI.setupWithNavController(navView, navController)

        // 햄버거 아이콘
        val toggle = ActionBarDrawerToggle(
            this, drawerLayout, toolbar,
            R.string.nav_open, R.string.nav_close
        )
        drawerLayout.addDrawerListener(toggle)
        toggle.syncState()
    }
}
```

React Navigation의 Drawer는 이 모든 것을 JavaScript 코드 하나로 처리한다. XML 레이아웃, 메뉴 리소스, NavigationUI 설정이 모두 불필요하다.

---

## 6. 네비게이터 조합 패턴

### 6-1. Drawer > Tab > Stack (가장 일반적인 앱 구조)

```
Drawer Navigator
├── MainTabs (Tab Navigator)
│   ├── HomeTab (Stack Navigator)
│   │   ├── HomeMain (Screen)
│   │   └── HomeDetail (Screen)
│   ├── SearchTab (Stack Navigator)
│   │   ├── SearchMain (Screen)
│   │   └── SearchResult (Screen)
│   └── ProfileTab (Stack Navigator)
│       ├── ProfileMain (Screen)
│       └── EditProfile (Screen)
├── Settings (Screen)
└── About (Screen)
```

```tsx
// 1단계: 각 탭의 Stack Navigator
const HomeStack = createNativeStackNavigator();
function HomeStackScreen() {
  return (
    <HomeStack.Navigator screenOptions={{ headerShown: false }}>
      <HomeStack.Screen name="HomeMain" component={HomeScreen} />
      <HomeStack.Screen name="HomeDetail" component={HomeDetailScreen} />
    </HomeStack.Navigator>
  );
}

const SearchStack = createNativeStackNavigator();
function SearchStackScreen() {
  return (
    <SearchStack.Navigator screenOptions={{ headerShown: false }}>
      <SearchStack.Screen name="SearchMain" component={SearchScreen} />
      <SearchStack.Screen name="SearchResult" component={SearchResultScreen} />
    </SearchStack.Navigator>
  );
}

// 2단계: Tab Navigator
const Tab = createBottomTabNavigator();
function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={{
        tabBarActiveTintColor: '#e94560',
        tabBarInactiveTintColor: '#a0a0a0',
        tabBarStyle: { backgroundColor: '#1a1a2e' },
      }}
    >
      <Tab.Screen name="HomeTab" component={HomeStackScreen}
        options={{ title: '홈', headerShown: false }} />
      <Tab.Screen name="SearchTab" component={SearchStackScreen}
        options={{ title: '검색', headerShown: false }} />
      <Tab.Screen name="ProfileTab" component={ProfileScreen}
        options={{ title: '프로필' }} />
    </Tab.Navigator>
  );
}

// 3단계: Drawer Navigator (최상위)
const Drawer = createDrawerNavigator();
function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator
        drawerContent={(props) => <CustomDrawerContent {...props} />}
        screenOptions={{ headerShown: false }}
      >
        <Drawer.Screen name="MainTabs" component={MainTabs} />
        <Drawer.Screen name="Settings" component={SettingsScreen} />
        <Drawer.Screen name="About" component={AboutScreen} />
      </Drawer.Navigator>
    </NavigationContainer>
  );
}
```

### 6-2. Stack > Tab (모달이 탭 위에 오는 구조)

```tsx
// Root Stack이 전체를 감싸고, Tab 위에 모달 화면을 배치
const RootStack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <RootStack.Navigator screenOptions={{ headerShown: false }}>
        {/* 메인 탭 */}
        <RootStack.Screen name="MainTabs" component={MainTabs} />

        {/* 탭 바 위에 표시되는 전체 화면들 */}
        <RootStack.Group screenOptions={{
          presentation: 'modal',
          headerShown: true,
          headerStyle: { backgroundColor: '#1a1a2e' },
          headerTintColor: '#fff',
        }}>
          <RootStack.Screen name="CreatePost" component={CreatePostScreen}
            options={{ title: '새 글 작성' }} />
          <RootStack.Screen name="ImagePicker" component={ImagePickerScreen}
            options={{ title: '사진 선택' }} />
        </RootStack.Group>
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

---

## 7. 전체 앱 네비게이션 구조 예제

실제 프로덕션 앱에서 사용할 수 있는 완전한 네비게이션 구조 예제다. SNS 앱을 기준으로 작성했다.

```tsx
// navigation/AppNavigator.tsx — 완전한 앱 네비게이션 구조
import React from 'react';
import { View, Text, StyleSheet, TouchableOpacity, ActivityIndicator } from 'react-native';
import { NavigationContainer, useNavigation } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createDrawerNavigator, DrawerContentScrollView, DrawerItemList } from '@react-navigation/drawer';

// ========== 타입 정의 ==========
type RootStackParamList = {
  // 인증 화면
  Login: undefined;
  Register: undefined;
  // 메인 앱
  MainDrawer: undefined;
  // 전체 화면 (탭 바 위에 표시)
  PostDetail: { postId: number };
  UserProfile: { userId: string };
  CreatePost: undefined;
  Comments: { postId: number };
  PhotoViewer: { imageUrl: string };
};

type MainTabParamList = {
  FeedTab: undefined;
  ExploreTab: undefined;
  NotificationsTab: undefined;
  MyProfileTab: undefined;
};

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}

// ========== 화면 컴포넌트 (간략화) ==========
const makeScreen = (name: string, color: string) => () => (
  <View style={[screenStyles.container, { backgroundColor: color }]}>
    <Text style={screenStyles.title}>{name}</Text>
  </View>
);

const FeedScreen = makeScreen('피드', '#16213e');
const ExploreScreen = makeScreen('탐색', '#1a1a2e');
const NotificationsScreen = makeScreen('알림', '#0f3460');
const MyProfileScreen = makeScreen('내 프로필', '#16213e');
const PostDetailScreen = makeScreen('게시물 상세', '#1a1a2e');
const UserProfileScreen = makeScreen('유저 프로필', '#0f3460');
const CreatePostScreen = makeScreen('새 게시물', '#16213e');
const LoginScreen = makeScreen('로그인', '#1a1a2e');
const RegisterScreen = makeScreen('회원가입', '#0f3460');
const SettingsScreen = makeScreen('설정', '#16213e');
const BookmarksScreen = makeScreen('북마크', '#1a1a2e');

const screenStyles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  title: { fontSize: 24, fontWeight: 'bold', color: '#fff' },
});

// ========== 1. Bottom Tab Navigator ==========
const Tab = createBottomTabNavigator<MainTabParamList>();

function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={{
        headerShown: false,
        tabBarStyle: {
          backgroundColor: '#1a1a2e',
          borderTopColor: '#333',
          height: 60,
          paddingBottom: 8,
        },
        tabBarActiveTintColor: '#e94560',
        tabBarInactiveTintColor: '#666',
      }}
    >
      <Tab.Screen
        name="FeedTab"
        component={FeedScreen}
        options={{
          title: '피드',
          tabBarIcon: ({ color, size }) => (
            <Text style={{ color, fontSize: size - 4 }}>H</Text>
          ),
        }}
      />
      <Tab.Screen
        name="ExploreTab"
        component={ExploreScreen}
        options={{
          title: '탐색',
          tabBarIcon: ({ color, size }) => (
            <Text style={{ color, fontSize: size - 4 }}>S</Text>
          ),
        }}
      />
      <Tab.Screen
        name="NotificationsTab"
        component={NotificationsScreen}
        options={{
          title: '알림',
          tabBarBadge: 5,
          tabBarIcon: ({ color, size }) => (
            <Text style={{ color, fontSize: size - 4 }}>N</Text>
          ),
        }}
      />
      <Tab.Screen
        name="MyProfileTab"
        component={MyProfileScreen}
        options={{
          title: '프로필',
          tabBarIcon: ({ color, size }) => (
            <Text style={{ color, fontSize: size - 4 }}>P</Text>
          ),
        }}
      />
    </Tab.Navigator>
  );
}

// ========== 2. Drawer Navigator ==========
const Drawer = createDrawerNavigator();

function DrawerContent(props: any) {
  return (
    <View style={{ flex: 1, backgroundColor: '#1a1a2e' }}>
      <View style={{ padding: 20, paddingTop: 50, backgroundColor: '#0f3460' }}>
        <Text style={{ color: '#fff', fontSize: 20, fontWeight: 'bold' }}>김종우</Text>
        <Text style={{ color: '#a0a0a0', marginTop: 4 }}>@jongwoo</Text>
      </View>
      <DrawerContentScrollView {...props}>
        <DrawerItemList {...props} />
      </DrawerContentScrollView>
    </View>
  );
}

function MainDrawer() {
  return (
    <Drawer.Navigator
      drawerContent={(props) => <DrawerContent {...props} />}
      screenOptions={{
        headerStyle: { backgroundColor: '#1a1a2e' },
        headerTintColor: '#e94560',
        drawerStyle: { backgroundColor: '#1a1a2e', width: 260 },
        drawerActiveTintColor: '#e94560',
        drawerInactiveTintColor: '#a0a0a0',
      }}
    >
      <Drawer.Screen
        name="Tabs"
        component={MainTabs}
        options={{ title: 'SNS App' }}
      />
      <Drawer.Screen
        name="Settings"
        component={SettingsScreen}
        options={{ title: '설정' }}
      />
      <Drawer.Screen
        name="Bookmarks"
        component={BookmarksScreen}
        options={{ title: '북마크' }}
      />
    </Drawer.Navigator>
  );
}

// ========== 3. Root Stack Navigator (최상위) ==========
const RootStack = createNativeStackNavigator<RootStackParamList>();

export default function App() {
  const isLoggedIn = true; // 실제로는 AuthContext 사용

  return (
    <NavigationContainer>
      <RootStack.Navigator screenOptions={{ headerShown: false }}>
        {isLoggedIn ? (
          <>
            {/* 메인 앱 (Drawer > Tab) */}
            <RootStack.Screen name="MainDrawer" component={MainDrawer} />

            {/* 전체 화면 — 탭 바 없이 표시 */}
            <RootStack.Group
              screenOptions={{
                headerShown: true,
                headerStyle: { backgroundColor: '#1a1a2e' },
                headerTintColor: '#e94560',
                animation: 'slide_from_right',
              }}
            >
              <RootStack.Screen name="PostDetail" component={PostDetailScreen}
                options={{ title: '게시물' }} />
              <RootStack.Screen name="UserProfile" component={UserProfileScreen}
                options={{ title: '프로필' }} />
              <RootStack.Screen name="Comments" component={PostDetailScreen}
                options={{ title: '댓글' }} />
            </RootStack.Group>

            {/* 모달 화면 */}
            <RootStack.Group
              screenOptions={{
                presentation: 'modal',
                headerShown: true,
                headerStyle: { backgroundColor: '#1a1a2e' },
                headerTintColor: '#fff',
                animation: 'slide_from_bottom',
              }}
            >
              <RootStack.Screen name="CreatePost" component={CreatePostScreen}
                options={{ title: '새 게시물' }} />
              <RootStack.Screen name="PhotoViewer" component={PostDetailScreen}
                options={{ title: '사진', presentation: 'fullScreenModal' }} />
            </RootStack.Group>
          </>
        ) : (
          <>
            <RootStack.Screen name="Login" component={LoginScreen} />
            <RootStack.Screen name="Register" component={RegisterScreen} />
          </>
        )}
      </RootStack.Navigator>
    </NavigationContainer>
  );
}
```

### 네비게이션 구조 시각화

```
RootStack (Native Stack)
├── [Auth Group] (isLoggedIn === false)
│   ├── Login
│   └── Register
│
├── [Main Group] (isLoggedIn === true)
│   ├── MainDrawer (Drawer)
│   │   ├── Tabs (Bottom Tab)
│   │   │   ├── FeedTab
│   │   │   ├── ExploreTab
│   │   │   ├── NotificationsTab
│   │   │   └── MyProfileTab
│   │   ├── Settings
│   │   └── Bookmarks
│   │
│   ├── [Full Screen Group] — 탭 바 없이 표시
│   │   ├── PostDetail
│   │   ├── UserProfile
│   │   └── Comments
│   │
│   └── [Modal Group] — 모달로 표시
│       ├── CreatePost
│       └── PhotoViewer
```

---

## 요약: Android Navigation 컴포넌트 vs React Navigation (Tab & Drawer)

```
Android                              React Navigation
─────────────────────────────────    ─────────────────────────────
BottomNavigationView                 createBottomTabNavigator
menu/bottom_nav_menu.xml             Tab.Screen 컴포넌트 배열
NavigationUI.setupWithNavController  자동 (Navigator가 처리)
BadgeDrawable                        tabBarBadge 옵션
ViewPager2 + TabLayout               createMaterialTopTabNavigator
TabLayoutMediator                    자동 (Navigator가 처리)
DrawerLayout + NavigationView        createDrawerNavigator
ActionBarDrawerToggle                headerLeft / 자동 햄버거 아이콘
nav_view.headerLayout                drawerContent의 커스텀 헤더
drawerLayout.openDrawer()            navigation.openDrawer()
drawerLayout.closeDrawer()           navigation.closeDrawer()
```

다음 문서에서는 딥링크, 인증 플로우, 화면 전환 애니메이션 등 고급 네비게이션 패턴을 다룬다.
