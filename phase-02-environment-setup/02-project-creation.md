# 프로젝트 생성과 실행 — 첫 React Native 앱

> 환경 설정이 완료되었다면 이제 실제로 프로젝트를 만들고 실행해봅니다.
> Expo와 React Native CLI 두 가지 방법을 모두 다루며, Android 에뮬레이터와 실제 기기에서 실행하는 방법을 안내합니다.

---

## 1. Expo vs React Native CLI — 언제 무엇을 사용할까?

React Native 프로젝트를 만드는 방법은 크게 두 가지입니다:

1. **Expo** — React Native 위에 구축된 프레임워크. 설정이 간편하고 다양한 네이티브 API를 라이브러리로 제공합니다.
2. **React Native CLI (bare)** — React Native를 직접 사용하는 방식. android/ios 폴더에 완전한 접근 권한을 가집니다.

React Native 0.84 공식 문서에서는 **Expo를 기본 권장**합니다.

### 상세 비교표

| 항목 | Expo | React Native CLI (Bare) |
|------|------|------------------------|
| **초기 설정 난이도** | 매우 쉬움 | 중간 |
| **네이티브 코드 접근** | `expo prebuild`로 가능 | 항상 가능 |
| **네이티브 모듈 사용** | Expo SDK + Config Plugin | 직접 링크 |
| **빌드 방식** | EAS Build (클라우드) 또는 로컬 | 로컬 빌드 |
| **OTA 업데이트** | EAS Update 내장 | CodePush 등 별도 설정 |
| **파일 크기** | 약간 더 큼 (Expo 런타임 포함) | 최소한의 크기 |
| **Android Studio 필요** | 개발 시 불필요 (Expo Go 앱) | 필수 |
| **커스텀 네이티브 코드** | Config Plugin 또는 prebuild | 자유롭게 가능 |
| **Kotlin/Java 코드 추가** | prebuild 후 가능 | 바로 가능 |
| **프로젝트 업그레이드** | `expo upgrade` (비교적 쉬움) | 수동 (어려움) |
| **적합한 경우** | 대부분의 앱, 빠른 프로토타이핑 | 깊은 네이티브 커스터마이징 필요 시 |
| **Android 비유** | Android Studio + Firebase 통합 템플릿 | 빈 Android 프로젝트 |

### 결론: 어떤 걸 선택해야 할까?

- **Expo 권장**: 처음 배우는 경우, 대부분의 일반적인 앱, 빠른 개발이 필요한 경우
- **React Native CLI**: 기존 네이티브 앱에 RN을 통합하는 경우, 매우 특수한 네이티브 모듈이 필요한 경우

> **Android 개발자 참고**: Expo가 "제한적"이라는 인식은 과거의 것입니다. 현재 Expo는 `expo prebuild`를 통해 네이티브 코드에 완전히 접근할 수 있으며, 대부분의 서드파티 네이티브 모듈도 Config Plugin으로 지원됩니다. Expo를 사용하더라도 Kotlin/Java 코드를 작성할 수 있습니다.

---

## 2. Expo로 프로젝트 생성하기

### 2-1. 프로젝트 생성

```bash
# 최신 create-expo-app으로 프로젝트 생성
npx create-expo-app@latest MyFirstApp
```

실행하면 다음과 같은 과정이 진행됩니다:

```
Need to install the following packages:
  create-expo-app@3.2.0
Ok to proceed? (y) y

✔ Downloaded and extracted project files.
✔ Installed dependencies.

✅ Your project is ready!

To run your project, navigate to the directory and run one of the following npm commands.

- cd MyFirstApp
- npm run android
- npm run ios
- npm run web
```

### 2-2. 프로젝트 디렉토리로 이동

```bash
cd MyFirstApp
```

### 2-3. 생성된 프로젝트 구조 (간략)

```
MyFirstApp/
├── app/                    # 화면(라우트) 파일들 (Expo Router 사용)
│   ├── (tabs)/             # 탭 네비게이션 그룹
│   │   ├── _layout.tsx     # 탭 레이아웃 설정
│   │   ├── index.tsx       # 첫 번째 탭 화면
│   │   └── explore.tsx     # 두 번째 탭 화면
│   ├── _layout.tsx         # 루트 레이아웃
│   └── +not-found.tsx      # 404 페이지
├── assets/                 # 이미지, 폰트 등 정적 파일
├── components/             # 재사용 가능한 컴포넌트
├── constants/              # 상수 (색상, 사이즈 등)
├── hooks/                  # 커스텀 훅
├── node_modules/           # 설치된 패키지들 (Git에 포함 안 됨)
├── app.json                # Expo 앱 설정
├── package.json            # 프로젝트 메타데이터 및 의존성
├── tsconfig.json           # TypeScript 설정
└── ...
```

### 2-4. 템플릿 옵션

`create-expo-app`은 기본적으로 탭 네비게이션이 포함된 템플릿을 생성합니다. 빈 프로젝트로 시작하려면:

```bash
# 빈 템플릿으로 생성
npx create-expo-app@latest MyBlankApp --template blank-typescript
```

주요 템플릿:
| 템플릿 | 설명 |
|--------|------|
| (기본, 인자 없음) | 탭 네비게이션 포함, Expo Router 사용 |
| `blank` | JavaScript 최소 프로젝트 |
| `blank-typescript` | TypeScript 최소 프로젝트 |
| `tabs` | 기본과 동일 (탭 네비게이션) |

### 2-5. Expo 프로젝트 실행

```bash
# Android에서 실행
npx expo start
```

실행하면 터미널에 QR 코드와 함께 메뉴가 표시됩니다:

```
Starting Metro Bundler

▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
█ ▄▄▄▄▄ █▄██ █ ▄▄▄▄▄ █
█ █   █ █  ▄ █ █   █ █
█ █▄▄▄█ █ ██ █ █▄▄▄█ █
█▄▄▄▄▄▄▄█ █ █▄▄▄▄▄▄▄█
▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀

› Metro waiting on exp://192.168.0.10:8081
› Scan the QR code above with Expo Go (Android) or the Camera app (iOS)

› Using Expo Go
› Press a │ open Android
› Press i │ open iOS simulator
› Press w │ open web

› Press j │ open debugger
› Press r │ reload app
› Press m │ toggle menu
› Press o │ open project code in your editor
› Press ? │ show all commands
```

**키보드 단축키:**
- `a` — Android 에뮬레이터에서 앱 열기
- `i` — iOS 시뮬레이터에서 앱 열기 (macOS만)
- `w` — 웹 브라우저에서 열기
- `r` — 앱 리로드
- `m` — 개발자 메뉴 토글
- `j` — JavaScript 디버거 열기

### 2-6. Expo Go vs Development Build

| 방식 | 설명 | 사용 시기 |
|------|------|----------|
| **Expo Go** | Expo가 제공하는 미리 빌드된 앱. 설치만 하면 바로 개발 가능 | 빠른 프로토타이핑, Expo SDK 내 기능만 사용 |
| **Development Build** | 커스텀 네이티브 코드가 포함된 자체 빌드 | 서드파티 네이티브 모듈 사용 시 |

Expo Go로 시작하되, 서드파티 네이티브 모듈이 필요해지면 Development Build로 전환합니다:

```bash
# Development Build 생성
npx expo install expo-dev-client
npx expo prebuild
npx expo run:android
```

---

## 3. React Native CLI로 프로젝트 생성하기

### 3-1. 프로젝트 생성

```bash
npx @react-native-community/cli init MyBareApp
```

> **주의**: 과거에 사용하던 `react-native init`은 더 이상 권장되지 않습니다. `@react-native-community/cli`를 사용하세요.

실행 과정:

```
               ╭───────────────────────────────────────╮
               │                                       │
               │   Welcome to React Native 0.84!       │
               │                                       │
               ╰───────────────────────────────────────╯

✔ Downloading template
✔ Copying template
✔ Processing template
✔ Installing dependencies
✔ Do you want to install CocoaPods now? (iOS) › No

              ╭───────────────────────────────────────╮
              │                                       │
              │   Run instructions for Android:       │
              │                                       │
              │   cd MyBareApp                        │
              │   npx react-native run-android        │
              │                                       │
              ╰───────────────────────────────────────╯
```

### 3-2. 생성된 프로젝트 구조 (간략)

```
MyBareApp/
├── __tests__/              # 테스트 파일
├── android/                # Android 네이티브 프로젝트 (Gradle)
│   ├── app/
│   │   ├── src/main/
│   │   │   ├── java/       # Java/Kotlin 소스
│   │   │   ├── res/        # Android 리소스
│   │   │   └── AndroidManifest.xml
│   │   └── build.gradle
│   ├── build.gradle
│   ├── gradle.properties
│   └── settings.gradle
├── ios/                    # iOS 네이티브 프로젝트
├── node_modules/           # npm 패키지들
├── .eslintrc.js            # ESLint 설정
├── .prettierrc.js          # Prettier 설정
├── App.tsx                 # 메인 앱 컴포넌트
├── app.json                # 앱 이름 등 기본 설정
├── babel.config.js         # Babel 트랜스파일러 설정
├── index.js                # 앱 진입점
├── metro.config.js         # Metro 번들러 설정
├── package.json            # 프로젝트 설정 및 의존성
└── tsconfig.json           # TypeScript 설정
```

### 3-3. CLI 프로젝트 실행

```bash
cd MyBareApp

# Android 에뮬레이터에서 실행
npx react-native run-android
```

첫 실행 시 Gradle 빌드가 진행되므로 시간이 걸립니다 (5-15분):

```
info Running jetifier to migrate libraries to AndroidX.
info Starting JS server...
info Launching emulator...
info Building the app...

> Task :app:bundleDebugJsAndAssets
> Task :app:mergeDebugAssets
> Task :app:processDebugResources
> Task :app:compileDebugKotlin
...

> Task :app:installDebug
info Successfully installed the app on the device.
```

---

## 4. Android 에뮬레이터에서 실행하기

### 4-1. AVD(Android Virtual Device) 생성

Android Studio를 통해 에뮬레이터를 생성합니다:

1. Android Studio 열기
2. More Actions (또는 Tools) → Device Manager
3. "Create Virtual Device" 클릭
4. 디바이스 선택: **Pixel 7** (또는 원하는 기기) → Next
5. 시스템 이미지 선택: **API 35** (VanillaIceCream) → Next
   - Apple Silicon Mac: `arm64-v8a` 이미지
   - Intel Mac/Windows: `x86_64` 이미지 (HAXM 필요)
6. AVD 이름 확인 후 → Finish

또는 커맨드라인으로:

```bash
# 사용 가능한 시스템 이미지 확인
sdkmanager --list | grep "system-images;android-35"

# 시스템 이미지 설치 (Apple Silicon)
sdkmanager "system-images;android-35;google_apis;arm64-v8a"

# AVD 생성
avdmanager create avd -n Pixel7_API35 -k "system-images;android-35;google_apis;arm64-v8a" -d pixel_7
```

### 4-2. 에뮬레이터 실행

```bash
# 사용 가능한 AVD 목록 확인
emulator -list-avds
# 출력: Pixel7_API35

# 에뮬레이터 실행
emulator -avd Pixel7_API35
```

또는 Android Studio의 Device Manager에서 ▶ 버튼으로 실행합니다.

### 4-3. 앱 실행

에뮬레이터가 실행된 상태에서:

```bash
# Expo 프로젝트
npx expo start
# 그 후 'a' 키 입력

# 또는 직접 Android로 실행
npx expo run:android

# React Native CLI 프로젝트
npx react-native run-android
```

### 4-4. 연결 확인

```bash
adb devices
# 출력:
# List of devices attached
# emulator-5554   device     ← 에뮬레이터가 연결됨
```

---

## 5. 실제 Android 기기에서 실행하기

### 5-1. 개발자 옵션 활성화

1. **설정** → **휴대전화 정보** (About phone)
2. **빌드 번호** (Build number)를 **7번 연속 탭**
3. "개발자가 되었습니다!" 메시지 확인

### 5-2. USB 디버깅 활성화

1. **설정** → **개발자 옵션** (Developer options)
2. **USB 디버깅** (USB debugging) → ON
3. USB 케이블로 컴퓨터에 연결
4. 기기에 "USB 디버깅을 허용하시겠습니까?" 팝업 → **허용** (항상 허용 체크 권장)

### 5-3. 연결 확인

```bash
adb devices
# 출력:
# List of devices attached
# ABCD1234     device     ← 물리 기기가 연결됨
```

`unauthorized`가 표시되면 기기에서 USB 디버깅 허용 팝업을 확인하세요.

### 5-4. 앱 실행

에뮬레이터 실행과 동일한 명령어를 사용합니다:

```bash
# 물리 기기가 연결된 상태에서
npx react-native run-android
# 또는
npx expo run:android
```

여러 기기/에뮬레이터가 연결된 경우:

```bash
# 특정 기기 지정
npx react-native run-android --deviceId ABCD1234

# 연결된 기기 목록 확인
adb devices
```

### 5-5. 무선 디버깅 (Wi-Fi)

USB 케이블 없이도 디버깅할 수 있습니다:

```bash
# 1. USB로 먼저 연결한 상태에서
adb tcpip 5555

# 2. 기기의 IP 주소 확인 (설정 → Wi-Fi → 연결된 네트워크 → IP 주소)
# 또는
adb shell ip addr show wlan0 | grep "inet "

# 3. USB 케이블 분리 후 Wi-Fi로 연결
adb connect 192.168.0.100:5555

# 4. 확인
adb devices
# 출력:
# 192.168.0.100:5555   device
```

> **Android 11 이상**: 개발자 옵션 → 무선 디버깅(Wireless debugging)을 직접 사용할 수도 있습니다.

---

## 6. Hot Reload / Fast Refresh — Android Apply Changes와 비교

### Fast Refresh란?

Fast Refresh는 React Native의 핵심 개발 기능으로, 코드를 수정하면 **앱을 재시작하지 않고** 변경 사항이 즉시 반영됩니다.

```
[Fast Refresh 동작 흐름]

1. App.tsx 파일 수정 (예: 텍스트 변경)
2. Watchman이 파일 변경 감지 (수 ms)
3. Metro Bundler가 변경된 모듈만 재번들링 (~100ms)
4. 앱에 변경 사항 전송 (Hot Module Replacement)
5. React가 컴포넌트를 다시 렌더링
6. 화면에 변경 사항 표시

총 소요 시간: 보통 1초 이내
```

### Android Apply Changes와 비교

| 특성 | Android Apply Changes | React Native Fast Refresh |
|------|----------------------|--------------------------|
| **속도** | 수 초 ~ 수십 초 | 1초 이내 |
| **상태 유지** | 제한적 (Activity 재생성 가능) | 컴포넌트 state 유지 |
| **지원 범위** | 메서드 본문 변경만 | 모든 변경 (구조 변경 포함) |
| **실패 시** | 앱 재시작 필요 | 자동 Full Reload |
| **빌드 필요** | 부분 빌드 필요 | 빌드 불필요 (JS 번들만 갱신) |
| **신뢰도** | 때때로 실패 | 거의 항상 성공 |

### Fast Refresh의 동작 규칙

1. **함수형 컴포넌트 수정** → State 유지, UI만 업데이트
2. **스타일만 수정** → State 유지, 스타일만 업데이트
3. **컴포넌트가 아닌 파일 수정** (유틸리티 함수 등) → 해당 파일을 import하는 모든 컴포넌트 재실행
4. **구문 오류** → 오류 오버레이 표시, 수정 후 자동 복구
5. **Fast Refresh 실패 시** → 자동으로 Full Reload (앱 재시작)

### Fast Refresh 상태 초기화

State를 유지하고 싶지 않은 경우, 파일 상단에 다음 주석을 추가합니다:

```typescript
// @refresh reset
```

이 주석이 있는 파일은 수정 시 항상 컴포넌트의 state가 초기화됩니다.

---

## 7. Dev Menu — 개발자 메뉴

React Native 앱에는 개발 빌드에서만 사용 가능한 개발자 메뉴가 내장되어 있습니다.

### 개발자 메뉴 열기

| 환경 | 방법 |
|------|------|
| Android 에뮬레이터 | `Cmd + M` (macOS) 또는 `Ctrl + M` (Windows/Linux) |
| 물리 기기 | 기기를 흔들기 (Shake) |
| 터미널 | `d` 키 입력 (Metro 서버가 실행 중일 때) |
| adb 명령어 | `adb shell input keyevent 82` |

### 개발자 메뉴 항목

```
┌─────────────────────────────────┐
│         Dev Menu                │
├─────────────────────────────────┤
│ 🔄 Reload                      │  ← 앱 전체 리로드
│ 🐛 Open Debugger               │  ← Chrome DevTools 또는 Flipper로 디버깅
│ 📊 Open React DevTools          │  ← React 컴포넌트 트리 검사
│ 🔧 Toggle Inspector            │  ← UI 요소 검사 (Android의 Layout Inspector와 유사)
│ 📐 Toggle Perf Monitor         │  ← FPS, RAM, JS 스레드 성능 모니터
│ ⚙️ Settings                     │  ← Fast Refresh 토글, JS 엔진 정보 등
│ 🔌 Toggle Element Inspector    │  ← 터치한 요소의 컴포넌트 정보 표시
└─────────────────────────────────┘
```

#### 각 메뉴 설명

**Reload**: 앱 전체를 처음부터 다시 로드합니다. Fast Refresh가 안 될 때 사용합니다. Android Studio의 "Run"을 다시 누르는 것과 비슷하지만 네이티브 빌드는 하지 않습니다.

**Open Debugger**: JavaScript 디버거를 엽니다. React Native 0.84에서는 Chrome DevTools 기반의 새로운 디버거를 사용합니다. `console.log`, 브레이크포인트, 네트워크 요청 등을 확인할 수 있습니다.

**Toggle Inspector**: 화면의 UI 요소를 터치하면 해당 요소의 스타일, 크기, 마진/패딩 정보를 표시합니다. Android Studio의 Layout Inspector와 유사합니다.

**Toggle Perf Monitor**: 화면 상단에 성능 모니터를 표시합니다:
- JS FPS: JavaScript 스레드의 프레임레이트 (60이 이상적)
- UI FPS: 네이티브 UI 스레드의 프레임레이트 (60이 이상적)
- RAM 사용량
- 뷰 수

---

## 8. Metro Bundler — JavaScript 번들러

### Metro란?

Metro는 React Native의 JavaScript 번들러입니다. Android 개발에서 Gradle이 Java/Kotlin 코드를 컴파일하고 APK를 만드는 것처럼, Metro는 JavaScript/TypeScript 코드를 하나의 번들로 합칩니다.

```
[Metro Bundler의 역할]

소스 코드 (.tsx, .ts, .js)
       ↓
  [Babel 트랜스파일]     ← TypeScript → JavaScript 변환, JSX 변환
       ↓
  [모듈 해석]            ← import/require 관계 파악
       ↓
  [번들링]               ← 모든 모듈을 하나의 파일로 합침
       ↓
  [최적화]               ← 미사용 코드 제거, 압축
       ↓
  index.bundle.js        ← 최종 JavaScript 번들
       ↓
  [앱에 로드]            ← Hermes 엔진이 실행
```

### Android 빌드와의 비교

| 과정 | Android (Gradle) | React Native (Metro) |
|------|------------------|---------------------|
| 코드 변환 | kotlinc/javac 컴파일 | Babel 트랜스파일 |
| 의존성 해석 | Gradle dependency resolution | Metro module resolution |
| 번들링 | DEX 생성 | JS 번들 생성 |
| 출력물 | APK/AAB | index.bundle.js (+ native shell) |
| 개발 서버 | 없음 (매번 빌드) | Metro Dev Server (핫 리로드) |

### Metro 서버 시작

```bash
# Expo 프로젝트
npx expo start

# React Native CLI 프로젝트
npx react-native start
```

### Metro 서버 명령어

Metro 서버가 실행 중일 때 터미널에서 사용할 수 있는 키:

| 키 | 동작 |
|----|------|
| `r` | 앱 리로드 |
| `d` | 개발자 메뉴 열기 |
| `i` | iOS 시뮬레이터에서 열기 |
| `a` | Android 에뮬레이터에서 열기 |
| `j` | JavaScript 디버거 열기 |
| `?` | 모든 명령어 보기 |

### Metro 캐시 초기화

빌드 문제가 발생할 때 캐시를 초기화하면 해결되는 경우가 많습니다:

```bash
# Expo
npx expo start -c

# React Native CLI
npx react-native start --reset-cache
```

### Metro 설정 파일 (metro.config.js)

```javascript
// metro.config.js
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);

const config = {
  // 커스텀 설정을 여기에 추가
  resolver: {
    // 추가 파일 확장자 지원
    sourceExts: [...defaultConfig.resolver.sourceExts, 'svg'],
  },
  transformer: {
    // 트랜스포머 설정
  },
};

module.exports = mergeConfig(defaultConfig, config);
```

---

## 9. 자주 발생하는 첫 실행 오류와 해결 방법

### 오류 1: `Could not connect to development server`

```
Could not connect to development server.
Ensure the following:
- Metro is running
- Your device/emulator is connected to the same network
```

**원인**: 앱이 Metro 서버에 연결할 수 없음.

**해결**:
```bash
# Metro 서버가 실행 중인지 확인
# 별도의 터미널에서 Metro를 실행해야 합니다

# 에뮬레이터의 경우 포트 리버스 설정
adb reverse tcp:8081 tcp:8081

# Metro 서버 재시작 (캐시 초기화)
npx react-native start --reset-cache
```

### 오류 2: `INSTALL_FAILED_INSUFFICIENT_STORAGE` — 에뮬레이터 저장 공간 부족

```
INSTALL_FAILED_INSUFFICIENT_STORAGE
```

**원인**: 에뮬레이터의 내부 저장 공간이 부족.

**해결**:
```bash
# 에뮬레이터의 데이터 초기화
# Android Studio → Device Manager → ▼ (드롭다운) → Wipe Data

# 또는 에뮬레이터를 더 큰 저장 공간으로 재생성
# Advanced Settings → Internal Storage: 4096 MB
```

### 오류 3: `BUILD FAILED — Execution failed for task ':app:compileDebugKotlin'`

```
Execution failed for task ':app:compileDebugKotlin'.
> A failure occurred while executing org.jetbrains.kotlin.compilerRunner.GradleCompilerRunnerWithWorkers$GradleKotlinCompilerWorkAction
```

**원인**: JDK 버전 불일치 또는 Gradle 캐시 문제.

**해결**:
```bash
# JDK 버전 확인
java -version
# JDK 17이어야 합니다

# Gradle 캐시 정리
cd android
./gradlew clean
cd ..

# 재빌드
npx react-native run-android
```

### 오류 4: `error: bundling failed — Unable to resolve module`

```
error: bundling failed: Error: Unable to resolve module `./App` from `/index.js`
```

**원인**: import 경로가 잘못되었거나, node_modules가 손상됨.

**해결**:
```bash
# node_modules 재설치
rm -rf node_modules
npm install

# Metro 캐시 초기화
npx react-native start --reset-cache
```

### 오류 5: `Watchman crawl failed` — Watchman 문제

```
Watchman crawl failed. Make sure watchman is running for this project.
```

**원인**: Watchman의 상태가 비정상적임.

**해결**:
```bash
# Watchman 상태 초기화
watchman watch-del-all
watchman shutdown-server

# Metro 캐시와 함께 재시작
npx react-native start --reset-cache
```

### 오류 6: `adb: error: failed to get feature set: device offline`

```
error: device offline
```

**원인**: adb 데몬과 기기 간의 연결이 끊어짐.

**해결**:
```bash
# adb 서버 재시작
adb kill-server
adb start-server

# 기기 재연결 확인
adb devices
```

### 오류 7: `The development server returned response error code: 500`

```
The development server returned response error code: 500
```

**원인**: Metro Bundler에서 JavaScript 번들을 생성하는 중 오류가 발생.

**해결**:
```bash
# Metro 서버 터미널에서 상세한 오류 메시지를 확인합니다
# 보통 구문 오류(syntax error)나 import 오류입니다

# Metro 캐시 초기화 후 재시작
npx react-native start --reset-cache
```

---

## 10. 개발 워크플로우 정리

### 일상적인 개발 흐름

```
[React Native 일상 개발 워크플로우]

1. 터미널에서 Metro 서버 시작
   $ npx expo start          (Expo)
   $ npx react-native start  (CLI)

2. 에뮬레이터 또는 물리 기기에서 앱 실행
   $ a (Expo 메뉴에서)
   $ npx react-native run-android (CLI, 첫 실행 또는 네이티브 변경 시)

3. VS Code에서 코드 수정
   → Fast Refresh로 자동 반영 (1초 이내)

4. 반복: 수정 → 확인 → 수정 → 확인
   (네이티브 코드 변경 시에만 재빌드 필요)
```

### Android 개발 워크플로우와 비교

```
[Android 네이티브 개발 워크플로우]

1. Android Studio에서 프로젝트 열기
2. Run (▶) 버튼으로 빌드 + 설치 (30초 ~ 수 분)
3. 코드 수정
4. 다시 Run 또는 Apply Changes (10초 ~ 1분)
5. 반복

[React Native 개발 워크플로우]

1. Metro 서버 시작 (1회)
2. 앱 실행 (1회, 네이티브 빌드 필요 시에만 재실행)
3. 코드 수정
4. Fast Refresh로 자동 반영 (~1초)
5. 반복
```

핵심 차이: React Native에서는 **JavaScript 코드 변경 시 네이티브 빌드가 필요 없습니다**. 이것이 개발 속도를 극적으로 향상시키는 핵심 요소입니다.

---

> 다음: [03-project-structure.md](./03-project-structure.md) — 프로젝트 구조 완전 해부
