# 프로젝트 구조 완전 해부 — Android 프로젝트와 비교

> React Native 프로젝트의 모든 파일과 폴더의 역할을 Android 프로젝트와 비교하며 상세히 설명합니다.
> 이 문서를 통해 "이 파일은 왜 여기 있는 거지?"라는 의문을 해소할 수 있습니다.

---

## 1. 전체 프로젝트 디렉토리 트리 — React Native CLI (Bare) 프로젝트

`npx @react-native-community/cli init MyApp`으로 생성된 프로젝트의 전체 구조입니다:

```
MyApp/
│
├── __tests__/                    # 테스트 파일 디렉토리
│   └── App.test.tsx              # App 컴포넌트 기본 테스트
│
├── android/                      # ★ Android 네이티브 프로젝트 (Gradle)
│   ├── app/
│   │   ├── src/
│   │   │   ├── debug/
│   │   │   │   └── AndroidManifest.xml    # 디버그 빌드용 매니페스트
│   │   │   └── main/
│   │   │       ├── java/com/myapp/
│   │   │       │   ├── MainActivity.kt    # ★ 메인 액티비티
│   │   │       │   └── MainApplication.kt # ★ Application 클래스
│   │   │       ├── res/
│   │   │       │   ├── drawable/          # 스플래시 화면 등
│   │   │       │   ├── mipmap-*/          # 앱 아이콘
│   │   │       │   ├── values/
│   │   │       │   │   ├── colors.xml
│   │   │       │   │   ├── strings.xml
│   │   │       │   │   └── styles.xml
│   │   │       │   └── xml/               # 네트워크 보안 설정 등
│   │   │       └── AndroidManifest.xml    # ★ 메인 매니페스트
│   │   ├── build.gradle(.kts)             # 앱 모듈 빌드 설정
│   │   └── proguard-rules.pro             # ProGuard/R8 난독화 규칙
│   │
│   ├── gradle/
│   │   └── wrapper/
│   │       ├── gradle-wrapper.jar
│   │       └── gradle-wrapper.properties  # Gradle 래퍼 버전
│   │
│   ├── build.gradle(.kts)                 # 루트 빌드 스크립트
│   ├── gradle.properties                  # ★ Gradle 속성 (메모리, 기능 플래그)
│   ├── gradlew                            # Gradle 래퍼 (Unix)
│   ├── gradlew.bat                        # Gradle 래퍼 (Windows)
│   └── settings.gradle(.kts)              # 프로젝트 모듈 설정
│
├── ios/                          # iOS 네이티브 프로젝트 (Xcode)
│   ├── MyApp/
│   │   ├── AppDelegate.mm        # iOS 앱 진입점
│   │   ├── Info.plist             # iOS 앱 설정 (Android의 AndroidManifest.xml)
│   │   └── ...
│   ├── MyApp.xcodeproj/          # Xcode 프로젝트 파일
│   └── Podfile                   # CocoaPods 의존성 (iOS의 build.gradle)
│
├── node_modules/                 # ★ npm 패키지 저장소 (Git에 포함 안 됨)
│
├── .buckconfig                   # Buck 빌드 시스템 설정 (레거시)
├── .eslintrc.js                  # ESLint 코드 린팅 설정
├── .gitignore                    # Git 무시 패턴
├── .prettierrc.js                # Prettier 코드 포맷팅 설정
├── .watchmanconfig               # Watchman 설정
├── App.tsx                       # ★ 메인 앱 컴포넌트 (진입 UI)
├── Gemfile                       # Ruby 의존성 (iOS 빌드용 CocoaPods)
├── app.json                      # 앱 이름 등 기본 설정
├── babel.config.js               # ★ Babel 트랜스파일러 설정
├── index.js                      # ★ 앱 진입점 (registerComponent)
├── jest.config.js                # Jest 테스트 설정
├── metro.config.js               # ★ Metro 번들러 설정
├── package.json                  # ★ 프로젝트 메타데이터 및 의존성
├── package-lock.json             # 의존성 잠금 파일 (정확한 버전 고정)
└── tsconfig.json                 # ★ TypeScript 설정
```

---

## 2. package.json — 프로젝트의 심장

`package.json`은 React Native 프로젝트의 핵심 설정 파일입니다. Android의 `build.gradle`(앱 수준)에 해당합니다.

### 전체 구조 분석

```json
{
  "name": "MyApp",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "lint": "eslint .",
    "start": "react-native start",
    "test": "jest"
  },
  "dependencies": {
    "react": "19.0.0",
    "react-native": "0.84.0"
  },
  "devDependencies": {
    "@babel/core": "^7.25.0",
    "@babel/preset-env": "^7.25.0",
    "@babel/runtime": "^7.25.0",
    "@react-native-community/cli": "18.0.0",
    "@react-native/babel-preset": "0.84.0",
    "@react-native/eslint-config": "0.84.0",
    "@react-native/metro-config": "0.84.0",
    "@react-native/typescript-config": "0.84.0",
    "@types/react": "^19.0.0",
    "@types/react-test-renderer": "^19.0.0",
    "babel-jest": "^29.7.0",
    "eslint": "^8.57.0",
    "jest": "^29.7.0",
    "prettier": "^3.3.0",
    "react-test-renderer": "19.0.0",
    "typescript": "~5.6.0"
  },
  "engines": {
    "node": ">=22.11"
  }
}
```

### 각 섹션 상세 설명

#### `name`, `version`, `private`

```json
{
  "name": "MyApp",          // 프로젝트 이름 (npm 패키지 이름 규칙: 소문자, 하이픈)
  "version": "0.0.1",       // 시맨틱 버전 (major.minor.patch)
  "private": true            // npm 레지스트리에 실수로 퍼블리시하는 것을 방지
}
```

Android 비유:
- `name` → `applicationId` (com.example.myapp)
- `version` → `versionName`
- `private` → 이 앱은 라이브러리가 아님을 표시

#### `scripts` — 실행 명령어

```json
{
  "scripts": {
    "android": "react-native run-android",   // npm run android → Android 빌드 + 실행
    "ios": "react-native run-ios",           // npm run ios → iOS 빌드 + 실행
    "lint": "eslint .",                       // npm run lint → 코드 린트 검사
    "start": "react-native start",           // npm start → Metro 서버 시작
    "test": "jest"                            // npm test → 테스트 실행
  }
}
```

Android 비유: `build.gradle`의 tasks와 유사합니다.
- `npm run android` ≈ Android Studio의 Run 버튼 (▶)
- `npm start` ≈ 개발 서버 시작 (Android에는 직접적인 대응 없음)
- `npm test` ≈ `./gradlew test`
- `npm run lint` ≈ `./gradlew lint`

커스텀 스크립트를 추가할 수도 있습니다:

```json
{
  "scripts": {
    "clean": "cd android && ./gradlew clean && cd ..",
    "build:release": "cd android && ./gradlew assembleRelease",
    "type-check": "tsc --noEmit"
  }
}
```

#### `dependencies` — 프로덕션 의존성

```json
{
  "dependencies": {
    "react": "19.0.0",           // React 라이브러리 (UI 렌더링 엔진)
    "react-native": "0.84.0"     // React Native 프레임워크
  }
}
```

빌드된 앱에 포함되는 라이브러리입니다. Android의 `implementation` 의존성에 해당합니다.

```groovy
// Android build.gradle 대응
dependencies {
    implementation "com.facebook.react:react-android:0.84.0"  // react-native
}
```

의존성 추가 방법:
```bash
# 패키지 설치 (dependencies에 추가됨)
npm install @react-navigation/native

# 특정 버전 설치
npm install react-native-reanimated@3.16.0
```

#### `devDependencies` — 개발 전용 의존성

```json
{
  "devDependencies": {
    "@babel/core": "^7.25.0",                    // Babel 코어 (트랜스파일러)
    "@react-native/babel-preset": "0.84.0",       // RN 전용 Babel 프리셋
    "@react-native/metro-config": "0.84.0",       // Metro 기본 설정
    "@types/react": "^19.0.0",                   // React TypeScript 타입 정의
    "typescript": "~5.6.0",                       // TypeScript 컴파일러
    "eslint": "^8.57.0",                          // 코드 린트 도구
    "prettier": "^3.3.0",                         // 코드 포맷터
    "jest": "^29.7.0"                             // 테스트 프레임워크
  }
}
```

개발 시에만 사용되고 최종 앱에는 포함되지 않는 도구입니다. Android의 `testImplementation`, `lintChecks`에 해당합니다.

#### `engines`

```json
{
  "engines": {
    "node": ">=22.11"    // Node.js 22.11 이상 필요
  }
}
```

Android의 `minSdk`와 유사한 개념이지만, 앱 사용자가 아닌 개발 환경의 요구 사항입니다.

### 버전 표기법

| 표기 | 의미 | 예시 |
|------|------|------|
| `"0.84.0"` | 정확히 이 버전 | 0.84.0만 허용 |
| `"^7.25.0"` | 메이저 버전 고정, 마이너/패치 업데이트 허용 | 7.25.0 ~ 7.x.x |
| `"~5.6.0"` | 메이저+마이너 고정, 패치만 업데이트 허용 | 5.6.0 ~ 5.6.x |
| `">=22.11"` | 이 버전 이상 | 22.11 이상 모두 |

### build.gradle와 package.json 비교 표

| package.json | build.gradle | 설명 |
|-------------|-------------|------|
| `name` | `namespace` | 프로젝트 식별자 |
| `version` | `versionName` | 앱 버전 |
| `scripts` | Gradle tasks | 빌드/실행 명령어 |
| `dependencies` | `implementation` | 프로덕션 의존성 |
| `devDependencies` | `testImplementation` | 개발/테스트 의존성 |
| `package-lock.json` | `gradle.lockfile` | 버전 잠금 파일 |
| `npm install` | Gradle sync | 의존성 다운로드 |
| `node_modules/` | `~/.gradle/caches/` | 다운로드된 패키지 저장소 |

---

## 3. tsconfig.json — TypeScript 설정

```json
{
  "extends": "@react-native/typescript-config/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "jsx": "react-native",
    "target": "esnext",
    "module": "esnext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "lib": ["es2023"],
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules", "babel.config.js", "metro.config.js", "jest.config.js"]
}
```

### 주요 옵션 설명

| 옵션 | 값 | 설명 |
|------|-----|------|
| `strict` | `true` | 엄격한 타입 검사 (Kotlin의 null safety와 유사) |
| `jsx` | `"react-native"` | JSX를 React Native 방식으로 처리 |
| `target` | `"esnext"` | 최신 JavaScript 문법 사용 (Metro가 추가로 변환) |
| `module` | `"esnext"` | ES 모듈 시스템 사용 (import/export) |
| `moduleResolution` | `"bundler"` | Metro 번들러에 맞는 모듈 해석 방식 |
| `noEmit` | `true` | TS 컴파일러는 타입 검사만, 실제 변환은 Babel이 담당 |
| `resolveJsonModule` | `true` | JSON 파일 import 가능 |
| `isolatedModules` | `true` | 파일 단위 컴파일 호환 (Babel과의 호환성) |
| `skipLibCheck` | `true` | .d.ts 파일의 타입 검사 건너뛰기 (빌드 속도 향상) |
| `baseUrl` / `paths` | `"."` / `"@/*"` | 절대 경로 별칭 설정 |

### Android와의 비교

TypeScript는 Kotlin과 비슷한 위치에 있습니다:

| TypeScript 설정 | Kotlin/Android 설정 | 역할 |
|-----------------|---------------------|------|
| `tsconfig.json` | `build.gradle`의 kotlinOptions | 컴파일러 옵션 |
| `strict: true` | Kotlin의 기본 null safety | 엄격한 타입 검사 |
| `target` | `jvmTarget` | 출력 대상 버전 |
| `paths` (별칭) | — (Kotlin에는 없음) | import 경로 별칭 |
| `include/exclude` | `sourceSets` | 컴파일 대상 파일 |

---

## 4. metro.config.js — Metro 번들러 설정

```javascript
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

/**
 * Metro 설정
 * @see https://reactnative.dev/docs/metro
 */
const config = {};

module.exports = mergeConfig(getDefaultConfig(__dirname), config);
```

Metro는 React Native의 JavaScript 번들러입니다. Android 개발에서 Gradle이 Java/Kotlin 소스를 컴파일하고 리소스를 처리하여 APK를 만드는 역할을 하듯, Metro는 JavaScript/TypeScript 소스를 번들링합니다.

### 커스텀 설정 예시

```javascript
const { getDefaultConfig, mergeConfig } = require('@react-native/metro-config');

const defaultConfig = getDefaultConfig(__dirname);

const config = {
  resolver: {
    // 지원하는 파일 확장자 추가
    sourceExts: [...defaultConfig.resolver.sourceExts, 'cjs', 'svg'],

    // 에셋 확장자 추가
    assetExts: defaultConfig.resolver.assetExts.filter(ext => ext !== 'svg'),

    // 모듈 해석 시 무시할 패턴
    blockList: [/some-module\/.*/],
  },

  transformer: {
    // Babel 트랜스포머 설정
    babelTransformerPath: require.resolve('react-native-svg-transformer'),

    // 인라인 require 최적화 (성능 향상)
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true,
      },
    }),
  },

  // 감시할 디렉토리 (모노레포에서 유용)
  watchFolders: [],
};

module.exports = mergeConfig(defaultConfig, config);
```

### Metro vs Gradle 비교

| Metro 설정 | Gradle 대응 | 역할 |
|-----------|------------|------|
| `resolver.sourceExts` | `sourceSets` | 처리할 파일 확장자 |
| `resolver.assetExts` | `aaptOptions` | 에셋 파일 확장자 |
| `transformer` | 컴파일러 설정 | 소스 코드 변환 방법 |
| `watchFolders` | `include` 프로젝트 | 감시할 추가 디렉토리 |
| `inlineRequires` | ProGuard 최적화 | 성능 최적화 |

---

## 5. babel.config.js — Babel 트랜스파일러 설정

```javascript
module.exports = {
  presets: ['module:@react-native/babel-preset'],
};
```

Babel은 JavaScript 트랜스파일러입니다. 최신 JavaScript(ES2023+) 및 TypeScript 코드를 React Native의 Hermes 엔진이 이해할 수 있는 코드로 변환합니다.

### Babel의 역할

```
[Babel 변환 과정]

TypeScript + JSX 코드:
  const App: React.FC = () => {
    const [count, setCount] = useState<number>(0);
    return <View><Text>{count}</Text></View>;
  };
          ↓ Babel 변환
일반 JavaScript 코드:
  const App = () => {
    const [count, setCount] = useState(0);
    return React.createElement(View, null,
      React.createElement(Text, null, count)
    );
  };
```

### Android와의 비교

| Babel | Android | 역할 |
|-------|---------|------|
| `babel.config.js` | `build.gradle`의 compileOptions | 컴파일러 설정 |
| `@react-native/babel-preset` | AGP(Android Gradle Plugin) | 기본 변환 규칙 세트 |
| Babel 플러그인 | 어노테이션 프로세서 (kapt/ksp) | 코드 변환 확장 |
| JSX → createElement 변환 | XML → View 인플레이션 | UI 코드 변환 |

### 주요 Babel 플러그인

```javascript
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    // react-native-reanimated 사용 시 (항상 마지막에 추가)
    'react-native-reanimated/plugin',

    // 경로 별칭 설정
    ['module-resolver', {
      root: ['./src'],
      alias: {
        '@components': './src/components',
        '@screens': './src/screens',
        '@utils': './src/utils',
      },
    }],

    // 선택적 체이닝, Nullish 병합 등은 이미 preset에 포함
  ],
};
```

---

## 6. app.json / app.config.js — 앱 메타데이터 설정

### React Native CLI의 app.json

```json
{
  "name": "MyApp",
  "displayName": "My App"
}
```

CLI 프로젝트에서 `app.json`은 최소한의 정보만 포함합니다. 대부분의 설정은 `android/app/build.gradle`과 `AndroidManifest.xml`에서 관리합니다.

### Expo의 app.json (훨씬 풍부)

```json
{
  "expo": {
    "name": "My App",
    "slug": "my-app",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/icon.png",
    "userInterfaceStyle": "light",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.mycompany.myapp"
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#ffffff"
      },
      "package": "com.mycompany.myapp",
      "versionCode": 1,
      "permissions": ["CAMERA", "ACCESS_FINE_LOCATION"]
    },
    "web": {
      "favicon": "./assets/favicon.png"
    },
    "plugins": [
      "expo-router",
      ["expo-camera", { "cameraPermission": "카메라 접근이 필요합니다." }]
    ]
  }
}
```

### Expo app.json 주요 항목과 Android 대응

| app.json | Android 대응 | 설명 |
|----------|-------------|------|
| `name` | `android:label` (strings.xml) | 앱 표시 이름 |
| `version` | `versionName` | 앱 버전 |
| `android.package` | `applicationId` | 패키지 이름 |
| `android.versionCode` | `versionCode` | 빌드 번호 |
| `android.permissions` | `<uses-permission>` | 권한 선언 |
| `android.adaptiveIcon` | `mipmap/ic_launcher` | 앱 아이콘 |
| `splash` | SplashScreen API / themes.xml | 스플래시 화면 |
| `orientation` | `android:screenOrientation` | 화면 방향 |
| `plugins` | Gradle 플러그인 | 네이티브 설정 플러그인 |

### app.config.js (동적 설정)

`app.json` 대신 `app.config.js`를 사용하면 동적으로 설정을 생성할 수 있습니다:

```javascript
// app.config.js
export default ({ config }) => {
  const isProduction = process.env.APP_ENV === 'production';

  return {
    ...config,
    name: isProduction ? 'My App' : 'My App (Dev)',
    slug: 'my-app',
    version: '1.0.0',
    android: {
      package: isProduction
        ? 'com.mycompany.myapp'
        : 'com.mycompany.myapp.dev',
    },
  };
};
```

이것은 Android의 `productFlavors`와 유사한 역할을 합니다:

```groovy
// Android의 productFlavors 대응
android {
    productFlavors {
        dev {
            applicationId "com.mycompany.myapp.dev"
            resValue "string", "app_name", "My App (Dev)"
        }
        production {
            applicationId "com.mycompany.myapp"
            resValue "string", "app_name", "My App"
        }
    }
}
```

---

## 7. android/ 디렉토리 — React Native와 Android 네이티브의 통합

Android 개발자에게 가장 익숙한 부분입니다. React Native 프로젝트의 `android/` 디렉토리는 표준 Android 프로젝트와 거의 동일한 구조를 가집니다.

### android/app/src/main/java/com/myapp/MainApplication.kt

```kotlin
package com.myapp

import android.app.Application
import com.facebook.react.PackageList
import com.facebook.react.ReactApplication
import com.facebook.react.ReactHost
import com.facebook.react.ReactNativeHost
import com.facebook.react.ReactPackage
import com.facebook.react.defaults.DefaultNewArchitectureEntryPoint.load
import com.facebook.react.defaults.DefaultReactHost.getDefaultReactHost
import com.facebook.react.defaults.DefaultReactNativeHost
import com.facebook.soloader.SoLoader

class MainApplication : Application(), ReactApplication {

    override val reactNativeHost: ReactNativeHost =
        object : DefaultReactNativeHost(this) {
            override fun getPackages(): List<ReactPackage> =
                PackageList(this).packages.apply {
                    // 수동으로 추가할 네이티브 패키지가 있다면 여기에
                }

            override fun getJSMainModuleName(): String = "index"

            override fun getUseDeveloperSupport(): Boolean = BuildConfig.DEBUG

            override val isNewArchEnabled: Boolean = BuildConfig.IS_NEW_ARCHITECTURE_ENABLED
            override val isHermesEnabled: Boolean = BuildConfig.IS_HERMES_ENABLED
        }

    override val reactHost: ReactHost
        get() = getDefaultReactHost(applicationContext, reactNativeHost)

    override fun onCreate() {
        super.onCreate()
        SoLoader.init(this, false)
        if (BuildConfig.IS_NEW_ARCHITECTURE_ENABLED) {
            load()
        }
    }
}
```

**핵심 포인트:**
- `ReactApplication` 인터페이스를 구현합니다
- `getPackages()`: 네이티브 모듈 패키지 목록. 대부분 자동 링크(autolinking)로 처리됩니다
- `getJSMainModuleName()`: JavaScript 진입점 파일 (index.js)
- `isHermesEnabled`: Hermes JavaScript 엔진 사용 여부 (0.84에서 기본 true)
- `isNewArchEnabled`: 새 아키텍처(Fabric + TurboModules) 사용 여부

### android/app/src/main/java/com/myapp/MainActivity.kt

```kotlin
package com.myapp

import com.facebook.react.ReactActivity
import com.facebook.react.ReactActivityDelegate
import com.facebook.react.defaults.DefaultNewArchitectureEntryPoint.fabricEnabled
import com.facebook.react.defaults.DefaultReactActivityDelegate

class MainActivity : ReactActivity() {

    /**
     * JavaScript에서 등록된 메인 컴포넌트 이름을 반환합니다.
     * index.js의 AppRegistry.registerComponent()에서 사용한 이름과 일치해야 합니다.
     */
    override fun getMainComponentName(): String = "MyApp"

    override fun createReactActivityDelegate(): ReactActivityDelegate =
        DefaultReactActivityDelegate(this, mainComponentName, fabricEnabled)
}
```

**핵심 포인트:**
- `ReactActivity`를 상속합니다 (일반적인 `AppCompatActivity`가 아님)
- `getMainComponentName()`: JavaScript에서 `AppRegistry.registerComponent('MyApp', ...)`으로 등록한 이름
- 이 Activity 하나가 전체 React Native 앱의 호스트 역할을 합니다
- 일반 Android 앱에서는 화면마다 Activity/Fragment를 사용하지만, React Native에서는 단일 Activity 위에서 JavaScript가 화면을 관리합니다

### android/app/build.gradle(.kts)

```groovy
apply plugin: "com.android.application"
apply plugin: "org.jetbrains.kotlin.android"
apply plugin: "com.facebook.react"

react {
    /* React Native 빌드 설정 */
    // JS 번들의 진입점
    entryFile = file("../../index.js")

    // Hermes 엔진 활성화
    enableHermes = true

    // 새 아키텍처 활성화
    // hermesCommand = "$rootDir/../node_modules/react-native/sdks/hermesc/%OS-BIN%/hermesc"
}

android {
    namespace "com.myapp"
    compileSdk rootProject.ext.compileSdkVersion   // 35

    defaultConfig {
        applicationId "com.myapp"
        minSdk rootProject.ext.minSdkVersion        // 24
        targetSdk rootProject.ext.targetSdkVersion   // 35
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }
        release {
            minifyEnabled true    // ProGuard/R8 활성화
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
            signingConfig signingConfigs.debug   // 실제 배포 시 변경 필요
        }
    }
}

dependencies {
    implementation("com.facebook.react:react-android")

    if (hermesEnabled.toBoolean()) {
        implementation("com.facebook.react:hermes-android")
    } else {
        implementation jscFlavor
    }
}
```

### android/gradle.properties

```properties
# 프로젝트 속성
android.useAndroidX=true
android.enableJetifier=true

# React Native 새 아키텍처 활성화 여부
newArchEnabled=true

# Hermes JS 엔진 사용 여부
hermesEnabled=true

# Gradle 메모리 설정
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m

# 빌드 속도 최적화
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true
```

### Android 네이티브 코드 추가하기

React Native에서 필요한 네이티브 기능이 있으면 `android/` 디렉토리에 직접 Kotlin/Java 코드를 추가할 수 있습니다:

```
android/app/src/main/java/com/myapp/
├── MainActivity.kt          # RN이 생성한 기본 파일
├── MainApplication.kt       # RN이 생성한 기본 파일
├── MyNativeModule.kt         # 직접 추가한 네이티브 모듈
└── MyNativePackage.kt        # 네이티브 모듈 패키지
```

---

## 8. index.js — 앱 진입점

```javascript
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => App);
```

이 파일은 React Native 앱의 진입점입니다.

### 코드 분석

| 코드 | 역할 | Android 대응 |
|------|------|-------------|
| `AppRegistry.registerComponent` | 루트 컴포넌트 등록 | `setContentView(R.layout.activity_main)` |
| `appName` | 앱 이름 (네이티브에서 참조) | `getMainComponentName()` 반환값 |
| `App` | 루트 React 컴포넌트 | activity_main.xml의 최상위 ViewGroup |

### 동작 흐름

```
[앱 시작 흐름]

1. Android가 MainActivity를 시작
2. MainActivity.getMainComponentName() → "MyApp" 반환
3. ReactNativeHost가 Hermes 엔진 초기화
4. Metro에서 JS 번들(index.bundle.js) 로드
5. index.js 실행 → AppRegistry.registerComponent("MyApp", () => App)
6. "MyApp"이라는 이름으로 등록된 App 컴포넌트를 렌더링
7. App 컴포넌트의 JSX → 네이티브 View로 변환 → 화면에 표시
```

### Expo 프로젝트에서의 진입점

Expo 프로젝트(Expo Router 사용 시)에서는 `index.js` 대신 `app/_layout.tsx`가 실질적인 진입점이 됩니다:

```typescript
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return <Stack />;
}
```

내부적으로 Expo는 `expo-router/entry`를 진입점으로 사용합니다 (`package.json`의 `"main"` 필드 참조).

---

## 9. Android 프로젝트 구조 vs React Native 프로젝트 구조 — Side-by-Side 비교

### 디렉토리 매핑

```
Android 프로젝트                    React Native 프로젝트
================                   ====================

app/                               (프로젝트 루트)
├── build.gradle          ←→       package.json + android/app/build.gradle
├── src/main/
│   ├── java/
│   │   └── com/myapp/
│   │       ├── MainActivity.kt    ← App.tsx (메인 화면 UI)
│   │       ├── ui/
│   │       │   ├── HomeScreen.kt  ← screens/HomeScreen.tsx
│   │       │   ├── ProfileScreen  ← screens/ProfileScreen.tsx
│   │       │   └── components/
│   │       │       ├── Header.kt  ← components/Header.tsx
│   │       │       └── Card.kt   ← components/Card.tsx
│   │       ├── data/
│   │       │   ├── api/          ← api/ 또는 services/
│   │       │   └── repository/   ← hooks/ (custom hooks)
│   │       ├── domain/
│   │       │   └── model/        ← types/ 또는 models/
│   │       └── utils/            ← utils/
│   │
│   ├── res/
│   │   ├── layout/               ← JSX (코드 내에서 직접 정의)
│   │   ├── drawable/             ← assets/images/
│   │   ├── values/
│   │   │   ├── colors.xml        ← constants/Colors.ts
│   │   │   ├── strings.xml       ← i18n/ (국제화 라이브러리)
│   │   │   ├── dimens.xml        ← constants/Layout.ts
│   │   │   └── styles.xml        ← StyleSheet 또는 theme/
│   │   └── font/                 ← assets/fonts/
│   │
│   └── AndroidManifest.xml       ← app.json + AndroidManifest.xml
│
├── test/                         ← __tests__/
│
build.gradle (root)               ← package.json (scripts)
settings.gradle                   ← metro.config.js
gradle.properties                 ← .env / app.config.js
```

### 핵심 개념 매핑

| Android 개념 | React Native 대응 | 상세 설명 |
|-------------|-------------------|----------|
| **Activity** | **Screen (컴포넌트)** | 하나의 화면 단위. RN에서는 컴포넌트가 화면 역할 |
| **Fragment** | **컴포넌트** | 재사용 가능한 UI 블록 |
| **XML Layout** | **JSX** | UI를 선언적으로 정의하는 방법 |
| **View/ViewGroup** | **View** | 기본 컨테이너 |
| **TextView** | **Text** | 텍스트 표시 |
| **ImageView** | **Image** | 이미지 표시 |
| **RecyclerView** | **FlatList** | 효율적인 리스트 |
| **Adapter** | **renderItem** | 리스트 아이템 렌더링 |
| **ViewModel** | **Custom Hook** | 비즈니스 로직 및 상태 관리 |
| **LiveData/StateFlow** | **useState/useReducer** | 반응형 상태 |
| **Repository** | **Custom Hook / API 모듈** | 데이터 접근 계층 |
| **Gradle** | **Metro + npm** | 빌드 및 의존성 관리 |
| **ProGuard/R8** | **Metro 최적화 + Hermes 바이트코드** | 코드 최적화 |
| **strings.xml** | **i18n 라이브러리** | 다국어 지원 |
| **styles.xml** | **StyleSheet.create** | 스타일 정의 |
| **themes.xml** | **Context + ThemeProvider** | 테마 시스템 |
| **SharedPreferences** | **AsyncStorage** | 간단한 키-값 저장소 |
| **Room/SQLite** | **expo-sqlite / WatermelonDB** | 로컬 DB |
| **Retrofit/OkHttp** | **fetch / axios** | HTTP 클라이언트 |
| **Dagger/Hilt** | **Context API / Zustand** | 의존성 주입 / 상태 관리 |
| **Navigation Component** | **React Navigation / Expo Router** | 화면 전환 |

### 파일 확장자 비교

| Android | React Native | 용도 |
|---------|-------------|------|
| `.kt` / `.java` | `.tsx` / `.ts` | 로직 + UI (RN은 하나에 통합) |
| `.xml` (layout) | `.tsx` (JSX 부분) | UI 정의 |
| `.xml` (values) | `.ts` | 상수/리소스 |
| `.gradle` / `.gradle.kts` | `package.json` | 빌드 설정 |
| `.png` / `.jpg` | `.png` / `.jpg` | 이미지 에셋 |

---

## 10. 권장 프로젝트 구조 (실무)

React Native에는 공식적인 프로젝트 구조 가이드라인이 없지만, Android 개발자에게 친숙한 클린 아키텍처 기반의 구조를 추천합니다:

```
src/
├── api/                    # API 클라이언트 및 엔드포인트
│   ├── client.ts           # Axios/fetch 설정 (Retrofit 인스턴스 역할)
│   └── userApi.ts          # 사용자 관련 API 호출
│
├── components/             # 재사용 가능한 UI 컴포넌트
│   ├── common/             # 범용 컴포넌트 (Button, Card, Modal)
│   │   ├── Button.tsx
│   │   └── Card.tsx
│   └── user/               # 도메인별 컴포넌트
│       └── UserAvatar.tsx
│
├── hooks/                  # 커스텀 훅 (ViewModel 역할)
│   ├── useAuth.ts
│   └── useUsers.ts
│
├── navigation/             # 네비게이션 설정
│   ├── AppNavigator.tsx
│   └── types.ts
│
├── screens/                # 화면 컴포넌트 (Activity/Fragment 역할)
│   ├── HomeScreen.tsx
│   ├── ProfileScreen.tsx
│   └── SettingsScreen.tsx
│
├── stores/                 # 전역 상태 관리 (Zustand/Redux)
│   └── authStore.ts
│
├── theme/                  # 테마 및 스타일
│   ├── colors.ts           # colors.xml 역할
│   ├── spacing.ts          # dimens.xml 역할
│   └── typography.ts       # TextAppearance 역할
│
├── types/                  # TypeScript 타입 정의
│   ├── user.ts             # data class User에 해당
│   └── navigation.ts
│
└── utils/                  # 유틸리티 함수
    ├── format.ts
    └── validation.ts
```

---

## 11. 핵심 요약

| 파일/폴더 | 한 줄 설명 | Android 대응 |
|----------|----------|-------------|
| `package.json` | 프로젝트 설정, 의존성, 스크립트 | `build.gradle` |
| `tsconfig.json` | TypeScript 컴파일러 옵션 | `kotlinOptions` |
| `metro.config.js` | JS 번들러 설정 | Gradle 빌드 설정 |
| `babel.config.js` | JS 트랜스파일러 설정 | 컴파일러 설정 |
| `app.json` | 앱 메타데이터 | `AndroidManifest.xml` + `build.gradle` |
| `index.js` | 앱 진입점 | `MainActivity.onCreate` |
| `App.tsx` | 루트 UI 컴포넌트 | `activity_main.xml` |
| `android/` | Android 네이티브 프로젝트 | 그 자체 |
| `node_modules/` | 설치된 패키지 | `.gradle/caches/` |
| `package-lock.json` | 의존성 버전 잠금 | `gradle.lockfile` |

> 다음: [phase-03-core-components/01-basic-components.md](../phase-03-core-components/01-basic-components.md) — 기본 컴포넌트 완전 가이드
