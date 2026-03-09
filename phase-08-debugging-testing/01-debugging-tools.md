# 디버깅 도구 완전 가이드 — Android Studio Debugger 경험을 활용하기

## 목차
1. [Dev Menu (개발자 메뉴)](#1-dev-menu-개발자-메뉴)
2. [React Native DevTools](#2-react-native-devtools)
3. [LogBox 시스템](#3-logbox-시스템)
4. [Console 메서드 활용](#4-console-메서드-활용)
5. [Performance Monitor](#5-performance-monitor)
6. [네이티브 코드 디버깅](#6-네이티브-코드-디버깅)
7. [네트워크 디버깅](#7-네트워크-디버깅)
8. [대체 디버깅 도구](#8-대체-디버깅-도구)
9. [흔한 에러와 해결법](#9-흔한-에러와-해결법)

---

## 1. Dev Menu (개발자 메뉴)

Android Studio에서 앱을 실행하면 Run 탭이나 Logcat으로 디버깅하듯이, React Native에서는 **Dev Menu**가 디버깅의 출발점이다.

### Dev Menu 열기

```
Android 에뮬레이터: Cmd+M (macOS) 또는 Ctrl+M (Windows/Linux)
Android 실기기: 기기를 흔들기 (Shake)
iOS 시뮬레이터: Cmd+D
```

> **Android 개발자 비교**: Android Studio에서 "Apply Changes" 버튼이나 Logcat 필터를 사용하는 것처럼,
> Dev Menu는 React Native 개발 중 가장 자주 접근하는 디버깅 진입점이다.

### Dev Menu 옵션 상세 설명

| 메뉴 옵션 | 기능 | Android Studio 대응 |
|-----------|------|-------------------|
| **Reload** | JS 번들을 다시 로드한다. 네이티브 코드 변경이 없을 때 사용 | Run > Rerun (Shift+F10) |
| **Open DevTools** | Chrome 기반 React Native DevTools를 연다 | Layout Inspector + Logcat |
| **Enable Fast Refresh** | 코드 변경 시 상태를 유지하면서 화면을 업데이트 | Apply Changes (Ctrl+F10) |
| **Disable Fast Refresh** | Fast Refresh를 끈다. 상태 초기화 필요 시 사용 | — |
| **Performance Monitor** | FPS, JS 스레드, UI 스레드 정보를 오버레이로 표시 | GPU Rendering 프로파일링 |
| **Element Inspector** | 화면의 컴포넌트를 탭하여 스타일 정보를 확인 | Layout Inspector |
| **Toggle Inspector** | Element Inspector를 켜고 끔 | — |

### Dev Menu 프로그래밍 방식 열기

```typescript
// 코드에서 Dev Menu를 프로그래밍 방식으로 열기 (디버그 빌드에서만)
import { DevSettings } from 'react-native';

// Fast Refresh 토글
if (__DEV__) {
  DevSettings.addMenuItem('Custom Debug Action', () => {
    console.log('커스텀 디버그 액션 실행됨');
  });
}
```

### Dev Menu에 커스텀 항목 추가

```typescript
// App.tsx에서 개발 모드에서만 커스텀 디버그 메뉴 추가
import { DevSettings } from 'react-native';

if (__DEV__) {
  DevSettings.addMenuItem('Clear AsyncStorage', () => {
    import('@react-native-async-storage/async-storage').then(
      ({ default: AsyncStorage }) => {
        AsyncStorage.clear();
        console.log('AsyncStorage 초기화 완료');
      }
    );
  });

  DevSettings.addMenuItem('Show User Token', () => {
    import('@react-native-async-storage/async-storage').then(
      ({ default: AsyncStorage }) => {
        AsyncStorage.getItem('userToken').then((token) => {
          console.log('현재 토큰:', token);
        });
      }
    );
  });
}
```

---

## 2. React Native DevTools

React Native 0.84에서는 **React Native DevTools**가 기본 디버깅 도구이다. 과거의 Chrome Remote Debugging을 완전히 대체한다.

### DevTools 열기 방법

```bash
# 방법 1: Metro 터미널에서 'j' 키 누르기
# Metro 서버 실행 중 터미널에서:
# j → React Native DevTools가 브라우저에서 열림

# 방법 2: Dev Menu → "Open DevTools" 선택

# 방법 3: 브라우저에서 직접 접근
# http://localhost:8081/debugger-frontend
```

> **Android 개발자 비교**: Android Studio의 Logcat + Layout Inspector + Network Inspector를
> 하나의 브라우저 도구로 합쳐놓은 것이라고 생각하면 된다.

### 2.1 Console 패널

Android Studio의 Logcat에 해당한다. JavaScript 코드의 모든 console 출력이 여기에 표시된다.

```typescript
// 다양한 로그 레벨
console.log('일반 로그');     // Logcat의 Log.d()에 해당
console.warn('경고 메시지');   // Logcat의 Log.w()에 해당
console.error('에러 메시지');  // Logcat의 Log.e()에 해당

// Console 패널에서 직접 표현식 평가 가능 (Logcat에는 없는 기능)
// Console 하단 입력창에서:
// > navigation.getState()
// > AsyncStorage.getAllKeys()
```

**필터링 기능**:
- Log Level 필터: All, Errors, Warnings, Info, Debug
- 텍스트 필터: 특정 키워드가 포함된 로그만 표시
- Regex 필터: 정규 표현식으로 필터링

### 2.2 Components 탭

Android Studio의 **Layout Inspector**에 해당한다. 컴포넌트 트리를 시각적으로 확인할 수 있다.

```
[Components 탭에서 확인 가능한 정보]

컴포넌트 트리:
├── App
│   ├── NavigationContainer
│   │   ├── HomeScreen
│   │   │   ├── View (style: {flex: 1})
│   │   │   │   ├── Header (props: {title: "홈"})
│   │   │   │   ├── FlatList (data: [...100 items])
│   │   │   │   │   ├── ListItem (props: {id: 1, name: "항목1"})
│   │   │   │   │   ├── ListItem (props: {id: 2, name: "항목2"})
│   │   │   │   │   └── ...
```

**컴포넌트 선택 시 확인 가능한 정보**:
- **Props**: 해당 컴포넌트에 전달된 모든 속성값
- **State**: useState로 관리되는 모든 상태값 (실시간 편집 가능!)
- **Hooks**: 사용 중인 모든 훅의 현재 값
- **Source**: 해당 컴포넌트의 소스 코드 위치

```typescript
// 예시: 이 컴포넌트를 Components 탭에서 선택하면
function UserProfile({ userId }: { userId: string }) {
  const [name, setName] = useState('홍길동');  // State에서 확인/편집 가능
  const [age, setAge] = useState(25);           // State에서 확인/편집 가능

  // Components 탭 우측 패널에 표시:
  // Props: { userId: "user-123" }
  // State: { name: "홍길동", age: 25 }
  // Hooks: [useState: "홍길동", useState: 25]

  return (
    <View>
      <Text>{name}</Text>
      <Text>{age}세</Text>
    </View>
  );
}
```

> **Layout Inspector 대비 장점**: Android Studio의 Layout Inspector는 View 속성만 보여주지만,
> Components 탭은 React 상태, props, hooks 값까지 실시간으로 확인하고 편집할 수 있다.

### 2.3 Profiler 탭

React 렌더링 성능을 분석하는 도구다. Android Studio의 **CPU Profiler**에 해당한다.

```
[Profiler 사용 방법]

1. Profiler 탭 클릭
2. "Start profiling" (⏺ 빨간 버튼) 클릭
3. 앱에서 원하는 작업 수행 (화면 이동, 스크롤, 버튼 클릭 등)
4. "Stop profiling" (⏹ 버튼) 클릭
5. 결과 분석

[결과 해석]
- Flamegraph: 각 컴포넌트의 렌더링 시간 시각화
  ┌─────────────────────────────────────────┐
  │ App (2.3ms)                             │
  │ ┌───────────────────────┬──────────────┐│
  │ │ Header (0.5ms)        │ Body (1.6ms) ││
  │ │                       │ ┌──────────┐ ││
  │ │                       │ │List(1.2ms)│ ││
  │ └───────────────────────┴──────────────┘│
  └─────────────────────────────────────────┘

- Ranked: 렌더링 시간이 긴 순서로 정렬
- Timeline: 시간 순서대로 커밋(렌더링) 표시

[중요 지표]
- Render duration: 컴포넌트가 렌더링에 걸린 시간
- Commit duration: React가 DOM에 변경사항을 적용하는 데 걸린 시간
- "Why did this render?" (설정에서 활성화): 리렌더링 원인 표시
```

### 2.4 Network 탭

API 호출을 모니터링한다. Android Studio의 **Network Inspector**에 해당한다.

```
[Network 탭에서 확인 가능한 정보]

요청 목록:
| Method | URL                        | Status | Time  | Size  |
|--------|----------------------------|--------|-------|-------|
| GET    | /api/users                 | 200    | 120ms | 2.3KB |
| POST   | /api/login                 | 200    | 340ms | 0.5KB |
| GET    | /api/posts?page=1          | 200    | 89ms  | 15KB  |
| GET    | /api/images/avatar.png     | 304    | 12ms  | 0B    |
| POST   | /api/posts                 | 422    | 156ms | 0.3KB |

각 요청 클릭 시:
- Headers: Request/Response 헤더 전체
- Payload: Request body (JSON 등)
- Preview: Response body를 포맷팅하여 표시
- Timing: DNS, Connect, TLS, Request, Response 시간 분석
```

---

## 3. LogBox 시스템

React Native의 에러/경고 표시 시스템이다. Android의 `Thread.setDefaultUncaughtExceptionHandler()`와 유사한 역할을 한다.

### 3.1 에러 표시 (Red Screen)

```
┌──────────────────────────────────────────┐
│  ❌ TypeError: undefined is not an       │
│     object (evaluating 'user.name')      │
│                                          │
│  Stack trace:                            │
│  at UserProfile (UserProfile.tsx:15)     │
│  at HomeScreen (HomeScreen.tsx:32)       │
│  at App (App.tsx:8)                      │
│                                          │
│  [Component Stack]                       │
│  [Dismiss]  [Reload]                     │
└──────────────────────────────────────────┘
```

### 3.2 경고 표시 (Yellow Banner)

```
┌──────────────────────────────────────────┐
│  ⚠ Warning: Each child in a list should │
│    have a unique "key" prop.             │
│                                          │
│  Check the render method of `TodoList`.  │
│                     [Dismiss]            │
└──────────────────────────────────────────┘
```

### 3.3 LogBox 제어

```typescript
// index.js 또는 App.tsx 상단에서 설정

import { LogBox } from 'react-native';

// 특정 경고 무시 (문자열 매칭)
LogBox.ignoreLogs([
  'Warning: Each child in a list',  // key 관련 경고 (알고 있는 경우)
  'Require cycle:',                  // 순환 참조 경고
  'Non-serializable values were found in the navigation state',
  'VirtualizedLists should never be nested', // ScrollView 안의 FlatList 경고
]);

// 모든 로그 무시 (데모/발표 시에만 사용!)
// ⚠ 절대 프로덕션 코드에 남기지 말 것
LogBox.ignoreAllLogs();

// 프로그래밍 방식으로 에러 표시
LogBox.install(); // 기본적으로 설치되어 있음
LogBox.uninstall(); // LogBox 비활성화 (권장하지 않음)
```

```typescript
// 개발 환경에서만 특정 경고 무시하는 패턴
if (__DEV__) {
  LogBox.ignoreLogs([
    'Require cycle:', // Metro 번들러의 순환 참조 경고
  ]);
}
```

---

## 4. Console 메서드 활용

Android의 `Log.d()`, `Log.w()`, `Log.e()`에 해당하는 JavaScript console 메서드들이다.

### 기본 메서드

```typescript
// 기본 로그 — Log.d("TAG", "message")에 해당
console.log('사용자 데이터:', userData);

// 경고 — Log.w("TAG", "message")에 해당
console.warn('API 응답이 느립니다:', responseTime);

// 에러 — Log.e("TAG", "message")에 해당
console.error('네트워크 에러:', error);

// 정보 (info 레벨)
console.info('앱 버전:', appVersion);

// 디버그
console.debug('디버그 정보:', debugData);
```

### 고급 메서드

```typescript
// console.table — 객체/배열을 테이블 형태로 표시
const users = [
  { name: '홍길동', age: 25, role: 'admin' },
  { name: '김철수', age: 30, role: 'user' },
  { name: '이영희', age: 28, role: 'editor' },
];
console.table(users);
// ┌─────────┬──────────┬─────┬──────────┐
// │ (index) │  name    │ age │  role    │
// ├─────────┼──────────┼─────┼──────────┤
// │    0    │ '홍길동' │ 25  │ 'admin'  │
// │    1    │ '김철수' │ 30  │ 'user'   │
// │    2    │ '이영희' │ 28  │ 'editor' │
// └─────────┴──────────┴─────┴──────────┘

// console.time / console.timeEnd — 시간 측정
console.time('API 호출');
const response = await fetch('https://api.example.com/data');
const data = await response.json();
console.timeEnd('API 호출');
// 출력: API 호출: 234ms

// console.group / console.groupEnd — 로그 그룹화
console.group('🔐 인증 프로세스');
console.log('1. 토큰 확인');
console.log('2. 토큰 갱신 필요');
console.log('3. 리프레시 토큰으로 갱신 시도');
console.log('4. 새 토큰 발급 완료');
console.groupEnd();

// console.count — 호출 횟수 카운트
function handlePress() {
  console.count('버튼 클릭'); // 버튼 클릭: 1, 버튼 클릭: 2, ...
}

// console.assert — 조건이 false일 때만 출력
console.assert(user.age >= 0, '나이는 음수일 수 없습니다:', user.age);

// console.trace — 스택 트레이스 출력
function deepFunction() {
  console.trace('여기서 호출됨');
}
```

### 구조화된 로깅 패턴

```typescript
// Android의 TAG 패턴을 JavaScript에서 구현
const createLogger = (tag: string) => ({
  d: (message: string, ...args: unknown[]) =>
    console.log(`[${tag}]`, message, ...args),
  w: (message: string, ...args: unknown[]) =>
    console.warn(`[${tag}]`, message, ...args),
  e: (message: string, ...args: unknown[]) =>
    console.error(`[${tag}]`, message, ...args),
  i: (message: string, ...args: unknown[]) =>
    console.info(`[${tag}]`, message, ...args),
});

// 사용법 — Kotlin의 companion object { private const val TAG = "AuthViewModel" } 패턴과 유사
const log = createLogger('AuthService');

log.d('로그인 시도', { email: 'user@example.com' });
log.e('로그인 실패', error);
log.i('토큰 갱신 완료');
```

---

## 5. Performance Monitor

Dev Menu에서 "Performance Monitor"를 활성화하면 화면 상단에 오버레이가 표시된다.

### 표시되는 메트릭

```
┌────────────────────────────────────────┐
│ UI:  60 FPS  │  JS:  58 FPS           │
│ RAM: 125 MB  │  Views: 245 (over 500) │
└────────────────────────────────────────┘

각 메트릭 설명:

1. UI FPS (UI 스레드 프레임 레이트)
   - 네이티브 UI 렌더링의 초당 프레임 수
   - 60 FPS가 이상적 (16.67ms per frame)
   - 떨어지면: 네이티브 뷰 렌더링이 느림
   - Android의 GPU Rendering Profile에서 보는 막대 그래프와 동일 개념

2. JS FPS (JavaScript 스레드 프레임 레이트)
   - JavaScript 코드 실행의 초당 프레임 수
   - React 렌더링, 비즈니스 로직이 여기서 실행
   - 떨어지면: JS 코드 최적화 필요 (과도한 리렌더링, 무거운 계산 등)
   - Android에는 대응하는 것이 없음 (네이티브 Android는 단일 스레드 모델)

3. RAM (메모리 사용량)
   - 앱이 사용 중인 메모리
   - Android Studio의 Memory Profiler와 동일

4. Views (렌더링된 뷰 수)
   - 화면에 마운트된 네이티브 뷰의 총 수
   - 500개 이상이면 경고 (성능 저하 가능)
   - Android의 Hierarchy Viewer에서 보던 뷰 수와 유사
```

### 성능 문제 진단 패턴

```typescript
// JS FPS가 낮을 때 확인할 것들:

// 1. 불필요한 리렌더링 확인
// 리렌더링 횟수를 추적하는 커스텀 훅
function useRenderCount(componentName: string) {
  const renderCount = useRef(0);
  renderCount.current += 1;

  if (__DEV__) {
    console.log(`[Render] ${componentName}: ${renderCount.current}번째 렌더링`);
  }
}

// 사용
function ExpensiveList() {
  useRenderCount('ExpensiveList');
  // ...
}

// 2. 무거운 계산을 useMemo로 최적화
function ProductList({ products }: { products: Product[] }) {
  // ❌ 매 렌더링마다 정렬 수행
  const sorted = products.sort((a, b) => a.price - b.price);

  // ✅ products가 변경될 때만 정렬 수행
  const sorted = useMemo(
    () => [...products].sort((a, b) => a.price - b.price),
    [products]
  );
}
```

---

## 6. 네이티브 코드 디버깅

React Native 앱에서 Kotlin으로 작성된 네이티브 모듈을 디버깅해야 할 때, Android Studio를 함께 사용한다.

### Android Studio에서 RN 프로젝트 열기

```bash
# React Native 프로젝트의 android 디렉토리를 Android Studio에서 열기
# File > Open > [프로젝트경로]/android

# 프로젝트 구조:
# MyApp/
# ├── android/           ← 이 디렉토리를 Android Studio에서 연다
# │   ├── app/
# │   │   ├── src/main/java/com/myapp/
# │   │   │   ├── MainActivity.kt
# │   │   │   └── MainApplication.kt
# │   │   └── build.gradle
# │   └── build.gradle
# ├── src/
# │   └── ... (JS/TS 코드)
# └── package.json
```

### 네이티브 모듈 디버깅

```kotlin
// android/app/src/main/java/com/myapp/NativeCalculator.kt
package com.myapp

import android.util.Log
import com.facebook.react.bridge.*

class NativeCalculatorModule(reactContext: ReactApplicationContext) :
    ReactContextBaseJavaModule(reactContext) {

    override fun getName() = "NativeCalculator"

    @ReactMethod
    fun heavyComputation(input: Double, promise: Promise) {
        // ← Android Studio에서 여기에 브레이크포인트를 설정!
        Log.d("NativeCalculator", "입력값: $input")  // Logcat에서 확인 가능

        try {
            val result = performComputation(input)
            Log.d("NativeCalculator", "결과: $result")
            promise.resolve(result)
        } catch (e: Exception) {
            Log.e("NativeCalculator", "계산 에러", e)
            promise.reject("CALC_ERROR", e.message)
        }
    }
}
```

### 디버거 연결 방법

```
1. Metro 서버 실행: npx react-native start
2. Android Studio에서 android/ 프로젝트 열기
3. 앱이 에뮬레이터/기기에서 실행 중인지 확인
4. Android Studio 메뉴: Run > Attach Debugger to Android Process
5. 프로세스 목록에서 앱 패키지명 선택 (예: com.myapp)
6. 브레이크포인트가 설정된 네이티브 코드에서 실행이 멈춤

[디버깅 가능한 영역]
- Kotlin/Java 네이티브 모듈 코드
- React Native 브릿지 코드
- 서드파티 네이티브 라이브러리 코드

[동시 디버깅]
- Metro 터미널: JS 코드의 console.log 확인
- React Native DevTools: JS 코드 디버깅
- Android Studio: 네이티브 Kotlin 코드 디버깅
- Logcat: 네이티브 로그 확인
→ 네 가지를 동시에 사용하여 전체 스택을 디버깅할 수 있다
```

### Logcat 활용

```bash
# 터미널에서 직접 Logcat 필터링 (Android Studio 없이)
adb logcat *:S ReactNative:V ReactNativeJS:V

# 태그별 필터
adb logcat -s "NativeCalculator"

# React Native JS 로그만 보기
adb logcat -s "ReactNativeJS"
```

```kotlin
// Logcat에서 보기 좋게 네이티브 로그 작성
companion object {
    private const val TAG = "MyModule"
}

Log.d(TAG, "=== 함수 시작 ===")
Log.d(TAG, "파라미터: userId=$userId, action=$action")
Log.d(TAG, "=== 함수 종료 (${System.currentTimeMillis() - startTime}ms) ===")
```

---

## 7. 네트워크 디버깅

### React Native DevTools의 Network 탭 활용

```typescript
// API 호출에 디버깅 로그 추가
const apiClient = {
  async request<T>(url: string, options?: RequestInit): Promise<T> {
    const startTime = Date.now();
    const requestId = Math.random().toString(36).substring(7);

    if (__DEV__) {
      console.group(`🌐 [${requestId}] ${options?.method || 'GET'} ${url}`);
      console.log('Headers:', options?.headers);
      if (options?.body) {
        console.log('Body:', JSON.parse(options.body as string));
      }
    }

    try {
      const response = await fetch(url, options);
      const data = await response.json();

      if (__DEV__) {
        const duration = Date.now() - startTime;
        console.log(`Status: ${response.status} (${duration}ms)`);
        console.log('Response:', data);
        console.groupEnd();
      }

      return data as T;
    } catch (error) {
      if (__DEV__) {
        console.error('Network Error:', error);
        console.groupEnd();
      }
      throw error;
    }
  },
};
```

### 프록시를 사용한 네트워크 디버깅

```typescript
// Charles Proxy 또는 Proxyman 사용 시 설정
// Android 에뮬레이터에서 프록시 설정:
// Settings > Network & internet > Wi-Fi > [네트워크] > Proxy > Manual
// Host: 10.0.2.2 (에뮬레이터에서 호스트 머신)
// Port: 8888 (Charles 기본 포트)

// fetch에 SSL 인증서 무시 설정 (개발 환경에서만!)
// android/app/src/main/res/xml/network_security_config.xml
/*
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="user" />
            <certificates src="system" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
*/
```

---

## 8. 대체 디버깅 도구

### Flipper (참고용)

Flipper는 Meta에서 만든 데스크톱 디버깅 도구였으나, React Native 0.73+ 에서는 공식적으로 **React Native DevTools**로 대체되었다. 새 프로젝트에서는 React Native DevTools를 사용하는 것을 권장한다.

```
[Flipper → React Native DevTools 마이그레이션]

Flipper 기능            → 대체 도구
Layout Inspector       → DevTools Components 탭
Network Inspector      → DevTools Network 탭
Shared Preferences     → react-native-mmkv DevTools 플러그인 또는 console.log
Database Inspector     → expo-sqlite DevTools
React DevTools         → DevTools Components + Profiler 탭
Hermes Debugger        → DevTools Console + Sources 탭
```

### Reactotron (대안 도구)

```bash
# 설치
npm install --save-dev reactotron-react-native

# Reactotron 데스크톱 앱 설치
# https://github.com/infinitered/reactotron 에서 다운로드
```

```typescript
// src/config/reactotron.ts
import Reactotron from 'reactotron-react-native';

if (__DEV__) {
  Reactotron
    .configure({ name: 'MyApp' })
    .useReactNative({
      asyncStorage: true,    // AsyncStorage 모니터링
      networking: true,      // 네트워크 요청 모니터링
      editor: true,          // VSCode에서 파일 열기
      errors: true,          // 에러 추적
      overlay: true,         // 이미지 오버레이
    })
    .connect();

  // console.tron으로 사용
  console.tron = Reactotron;
}

// 사용법
if (__DEV__) {
  console.tron?.log?.('커스텀 로그 메시지');
  console.tron?.display?.({
    name: 'API 응답',
    value: responseData,
    important: true,
  });
}
```

---

## 9. 흔한 에러와 해결법

### 에러 1: "Unable to load script. Make sure you're either running Metro..."

```
원인: Metro 번들러가 실행되지 않았거나, 기기가 Metro에 연결되지 않음
해결:
  1. Metro 실행 확인: npx react-native start
  2. 에뮬레이터에서 Dev Menu > "Settings" > "Debug server host & port"
     → "localhost:8081" 입력
  3. 포트 포워딩: adb reverse tcp:8081 tcp:8081
  4. Metro 캐시 클리어: npx react-native start --reset-cache
```

### 에러 2: "Module not found: Can't resolve './components/Header'"

```
원인: 파일 경로가 잘못되었거나 파일이 존재하지 않음
해결:
  1. 파일명의 대소문자 확인 (macOS는 대소문자 무시하지만 CI/Linux는 구분)
  2. 확장자 확인: .ts vs .tsx vs .js
  3. index 파일 확인: ./components/Header → ./components/Header/index.tsx
  4. 절대 경로 설정 확인: tsconfig.json의 paths 설정
```

### 에러 3: "Invariant Violation: requireNativeComponent: 'RCTView' was not found"

```
원인: 네이티브 모듈이 링크되지 않음
해결:
  1. cd android && ./gradlew clean
  2. npx react-native start --reset-cache
  3. 네이티브 의존성 다시 설치:
     cd android && ./gradlew clean && cd .. && npx react-native run-android
```

### 에러 4: "TypeError: undefined is not an object"

```typescript
// 원인: null/undefined 객체의 프로퍼티에 접근
// ❌ 에러 발생
const userName = user.name; // user가 undefined일 때

// ✅ 해결 1: Optional chaining
const userName = user?.name;

// ✅ 해결 2: 기본값 설정
const userName = user?.name ?? '익명';

// ✅ 해결 3: 조건부 렌더링
return user ? <Text>{user.name}</Text> : <Loading />;
```

### 에러 5: "VirtualizedLists should never be nested inside plain ScrollViews"

```typescript
// 원인: ScrollView 안에 FlatList를 중첩
// ❌ 에러
<ScrollView>
  <FlatList data={items} renderItem={...} />
</ScrollView>

// ✅ 해결: FlatList의 ListHeaderComponent / ListFooterComponent 사용
<FlatList
  data={items}
  renderItem={({ item }) => <ItemCard item={item} />}
  ListHeaderComponent={
    <View>
      <Banner />
      <SearchBar />
    </View>
  }
  ListFooterComponent={<Footer />}
/>
```

### 에러 6: "FAILURE: Build failed with an exception" (Gradle 빌드 실패)

```bash
# 원인: Gradle 캐시 손상 또는 의존성 충돌
# 해결 단계:

# 1. Gradle 캐시 클리어
cd android && ./gradlew clean

# 2. 더 강력한 클리어
cd android && ./gradlew clean
rm -rf android/.gradle
rm -rf android/app/build

# 3. node_modules 재설치
rm -rf node_modules
npm install

# 4. Metro 캐시 클리어 후 다시 빌드
npx react-native start --reset-cache
# 다른 터미널에서:
npx react-native run-android
```

### 에러 7: "Error: EMFILE: too many open files"

```bash
# 원인: 파일 감시자(watchman)의 파일 제한 초과
# macOS에서 해결:
brew install watchman

# 또는 파일 제한 증가
ulimit -n 65536
```

### 에러 8: "React.createElement: type is invalid"

```typescript
// 원인: 컴포넌트가 올바르게 export/import되지 않음
// ❌ 일반적인 실수
// Header.tsx에서:
export const Header = () => { ... }; // named export

// App.tsx에서:
import Header from './Header'; // default import → undefined!

// ✅ 해결:
import { Header } from './Header'; // named import 사용
// 또는 Header.tsx에서 default export 사용:
// export default function Header() { ... }
```

### 에러 9: "Execution failed for task ':app:mergeDebugResources'"

```bash
# 원인: Android 리소스 파일 충돌 (주로 이미지 파일명에 특수문자)
# 해결:
# 1. assets 폴더의 파일명에 대문자, 특수문자, 공백이 없는지 확인
# 2. 다시 빌드:
cd android && ./gradlew clean && cd ..
npx react-native run-android
```

### 에러 10: "Hermes doesn't support 'with' statement"

```typescript
// 원인: Hermes 엔진에서 지원하지 않는 JS 문법 사용
// Hermes는 성능을 위해 일부 오래된 JS 기능을 지원하지 않음

// ❌ 지원하지 않는 문법
with (obj) { ... }  // 'with' 문

// ✅ 해결: 표준적인 문법으로 대체
const { prop1, prop2 } = obj;
```

### 에러 11: "ECONNREFUSED 10.0.2.2:8081" (에뮬레이터에서 Metro 연결 실패)

```bash
# 원인: 에뮬레이터가 호스트의 Metro 서버에 접근하지 못함
# 해결:
adb reverse tcp:8081 tcp:8081

# 여러 기기/에뮬레이터가 연결된 경우:
adb devices  # 기기 목록 확인
adb -s emulator-5554 reverse tcp:8081 tcp:8081
```

---

## 10. 디버깅 워크플로우 총정리

```
[React Native 디버깅 체크리스트]

1. JS 로직 문제
   → Console 로그 + React Native DevTools Console 탭

2. UI/레이아웃 문제
   → Element Inspector (Dev Menu) + DevTools Components 탭

3. 성능 문제
   → Performance Monitor + DevTools Profiler 탭

4. 네트워크 문제
   → DevTools Network 탭 + Console 로그

5. 네이티브 코드 문제
   → Android Studio Debugger + Logcat

6. 빌드 문제
   → Gradle 캐시 클리어 + node_modules 재설치

7. 상태 관리 문제
   → DevTools Components 탭에서 state 확인 + console.log

[Android 개발자를 위한 정리]
Android Studio 도구          → React Native 대응
─────────────────────────────────────────────
Logcat                      → DevTools Console + console.log
Layout Inspector            → DevTools Components 탭
CPU Profiler               → DevTools Profiler 탭
Network Inspector          → DevTools Network 탭
Memory Profiler            → Performance Monitor (RAM)
GPU Rendering Profile      → Performance Monitor (UI FPS)
Build > Clean              → gradlew clean + Metro --reset-cache
Apply Changes              → Fast Refresh (자동)
```
