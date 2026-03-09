# Codegen과 TurboModules — 타입 안전한 네이티브 인터페이스

## 목차
1. [Codegen](#1-codegen)
2. [TurboModules](#2-turbomodules)
3. [TurboModule 만들기: 단계별 가이드](#3-turbomodule-만들기-단계별-가이드)
4. [완전한 실전 예제: Calendar TurboModule](#4-완전한-실전-예제-calendar-turbomodule)

---

## 1. Codegen

### 1.1 Codegen의 목적

Codegen은 **TypeScript(또는 Flow) Spec 파일로부터 네이티브 인터페이스 코드를 자동 생성**하는 도구이다. 이것으로 JS ↔ Native 간의 타입 안전성을 보장한다.

```
┌────────────────────────────────────────────────────────────────┐
│                 Codegen의 역할                                   │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  문제 (Legacy):                                                 │
│  JS: NativeModules.MyModule.greet("Kim", 30)                   │
│  Native: @ReactMethod fun greet(name: String, age: Int)        │
│                                                                 │
│  → JS에서 greet(30, "Kim")으로 잘못 호출해도                    │
│    컴파일 에러 없음! 런타임에 크래시 발생                         │
│                                                                 │
│  해결 (Codegen):                                                │
│  TypeScript Spec에서 타입을 정의하면                             │
│  Codegen이 C++/Kotlin 인터페이스를 자동 생성                     │
│  → 잘못된 타입은 컴파일 시점에 에러                              │
│                                                                 │
│  Android 비유:                                                   │
│  Legacy = Retrofit 없이 OkHttp 직접 사용                        │
│    (JSON 파싱 실수 → 런타임 크래시)                              │
│  Codegen = Retrofit + Moshi/Gson 어노테이션                     │
│    (인터페이스 정의 → 자동 코드 생성 → 타입 안전)                 │
│                                                                 │
│  또는:                                                          │
│  Legacy = 수동 SQL 쿼리 (타입 보장 없음)                        │
│  Codegen = Room @Dao (컴파일 타임 검증)                         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 Codegen 동작 흐름

```
┌────────────────────────────────────────────────────────────────┐
│               Codegen 전체 흐름                                  │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: 개발자가 TypeScript Spec 작성                          │
│  ┌──────────────────────────────────────┐                      │
│  │ // NativeDeviceInfo.ts               │                      │
│  │ import type { TurboModule } from     │                      │
│  │   'react-native';                    │                      │
│  │ import { TurboModuleRegistry } from  │                      │
│  │   'react-native';                    │                      │
│  │                                      │                      │
│  │ export interface Spec extends        │                      │
│  │   TurboModule {                      │                      │
│  │   getBatteryLevel(): Promise<number>;│                      │
│  │   getModel(): string;                │                      │
│  │ }                                    │                      │
│  │                                      │                      │
│  │ export default TurboModuleRegistry   │                      │
│  │   .getEnforcing<Spec>('DeviceInfo'); │                      │
│  └──────────────┬───────────────────────┘                      │
│                 │                                                │
│  Step 2: Codegen 실행 (빌드 시 자동)                            │
│                 │                                                │
│                 ▼                                                │
│  ┌──────────────────────────────────────┐                      │
│  │ react-native-codegen 도구가          │                      │
│  │ TypeScript AST를 분석하여            │                      │
│  │ 네이티브 코드 생성                    │                      │
│  └──────────────┬───────────────────────┘                      │
│                 │                                                │
│  Step 3: 생성되는 파일들                                        │
│                 │                                                │
│         ┌───────┴──────────────────────┐                       │
│         ▼                              ▼                        │
│  C++ Header                     Kotlin Abstract Class           │
│  ┌──────────────────┐          ┌──────────────────┐            │
│  │ NativeDeviceInfo │          │ NativeDeviceInfo │            │
│  │ Spec.h           │          │ Spec.kt          │            │
│  │                  │          │                  │            │
│  │ virtual double   │          │ abstract fun     │            │
│  │ getBatteryLevel()│          │ getBatteryLevel()│            │
│  │ = 0;             │          │ : Double         │            │
│  │                  │          │                  │            │
│  │ virtual string   │          │ abstract fun     │            │
│  │ getModel() = 0;  │          │ getModel()       │            │
│  │                  │          │ : String         │            │
│  └──────────────────┘          └──────────────────┘            │
│                                                                 │
│  Step 4: 개발자가 구현 클래스 작성                               │
│  ┌──────────────────────────────────────┐                      │
│  │ class DeviceInfoModule(ctx)          │                      │
│  │   : NativeDeviceInfoSpec(ctx) {      │                      │
│  │   override fun getBatteryLevel()     │                      │
│  │     : Double { ... }                 │                      │
│  │   override fun getModel()            │                      │
│  │     : String { ... }                 │                      │
│  │ }                                    │                      │
│  │ // 잘못된 반환 타입 → 컴파일 에러!    │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 Codegen이 지원하는 타입

```typescript
// Codegen이 지원하는 모든 타입 일람

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // ═══ 기본 타입 (Primitives) ═══
  getString(): string;                    // → Kotlin: String
  getNumber(): number;                    // → Kotlin: Double
  getBoolean(): boolean;                  // → Kotlin: Boolean
  getDouble(): Double;                    // → Kotlin: Double (명시적)
  getFloat(): Float;                      // → Kotlin: Float
  getInt32(): Int32;                      // → Kotlin: Int

  // ═══ Nullable 타입 ═══
  getNullableString(): string | null;     // → Kotlin: String?
  getNullableNumber(): number | null;     // → Kotlin: Double?

  // ═══ 배열 타입 ═══
  getStringArray(): string[];             // → Kotlin: ReadableArray
  getNumberArray(): number[];             // → Kotlin: ReadableArray
  getObjectArray(): Object[];             // → Kotlin: ReadableArray

  // ═══ 객체 타입 ═══
  getObject(): Object;                    // → Kotlin: ReadableMap
  getTypedObject(): {                     // → Kotlin: ReadableMap
    name: string;                         //   (키-값 접근)
    age: number;
  };

  // ═══ Promise (비동기) ═══
  fetchData(): Promise<string>;           // → Kotlin: fun fetchData(promise: Promise)
  fetchUser(): Promise<{                  // → Kotlin: Promise 사용
    name: string;
    email: string;
  }>;

  // ═══ Callback ═══
  onEvent(callback: (value: string) => void): void;
  // → Kotlin: fun onEvent(callback: Callback)

  // ═══ Enum (문자열 유니온) ═══
  getStatus(): 'active' | 'inactive';    // → Kotlin: String

  // ═══ Constants ═══
  getConstants(): {                       // → Kotlin: getTypedExportedConstants()
    VERSION: string;
    PLATFORM: string;
  };

  // ═══ void 반환 ═══
  doSomething(input: string): void;       // → Kotlin: Unit
}
```

### 1.4 Spec 파일 명명 규칙과 위치

```
프로젝트 구조에서 Spec 파일 위치:

my-app/
├── src/
│   └── specs/                          ← Spec 파일 모음 폴더
│       ├── NativeDeviceInfo.ts         ← "Native" 접두사 필수!
│       ├── NativeCalendar.ts
│       └── NativeStorage.ts
├── android/
│   └── app/
│       └── src/main/java/com/myapp/
│           ├── DeviceInfoModule.kt      ← Kotlin 구현
│           ├── CalendarModule.kt
│           └── StorageModule.kt
└── package.json
```

**명명 규칙 (매우 중요)**:

```
┌────────────────────────────────────────────────────────────────┐
│              Spec 파일 명명 규칙                                 │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 파일명: "Native" + 모듈명 + ".ts"                           │
│     ✓ NativeDeviceInfo.ts                                      │
│     ✓ NativeCalendar.ts                                        │
│     ✗ DeviceInfoSpec.ts (Native 접두사 없음)                    │
│     ✗ NativeDeviceInfo.tsx (확장자 .tsx 아닌 .ts)               │
│                                                                 │
│  2. TurboModuleRegistry.getEnforcing의 이름 =                   │
│     Kotlin 모듈의 getName() 반환값                              │
│                                                                 │
│     // TypeScript:                                              │
│     export default TurboModuleRegistry                          │
│       .getEnforcing<Spec>('DeviceInfo');                        │
│     // ↕ 반드시 일치!                                           │
│     // Kotlin:                                                  │
│     override fun getName() = "DeviceInfo"                       │
│                                                                 │
│  3. Spec 인터페이스는 반드시 TurboModule을 상속                  │
│     export interface Spec extends TurboModule { ... }           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.5 Codegen 실행: 수동 vs 자동

```bash
# ═══ 자동 실행 (권장) ═══
# Android 빌드 시 자동으로 Codegen이 실행됨
cd android && ./gradlew assembleDebug
# → 빌드 과정에서 Codegen이 자동 실행
# → android/app/build/generated/source/codegen/ 에 생성됨

# ═══ 수동 실행 (디버깅/확인용) ═══
# Codegen만 별도로 실행
npx react-native codegen

# 생성된 코드 위치 확인
ls android/app/build/generated/source/codegen/java/
# → com/facebook/fbreact/specs/NativeDeviceInfoSpec.java
```

### 1.6 package.json에서 Codegen 설정

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "codegenConfig": {
    "name": "MyAppSpecs",
    "type": "modules",
    "jsSrcsDir": "src/specs",
    "android": {
      "javaPackageName": "com.myapp.specs"
    }
  }
}
```

```
설정 필드 설명:
┌──────────────────┬─────────────────────────────────────────┐
│ 필드             │ 설명                                     │
├──────────────────┼─────────────────────────────────────────┤
│ name             │ 생성될 라이브러리 이름                    │
│ type             │ "modules" (TurboModule용)               │
│                  │ "components" (Fabric Component용)        │
│                  │ "all" (둘 다)                            │
│ jsSrcsDir        │ Spec 파일이 있는 디렉토리                 │
│ android.         │ 생성될 Java/Kotlin 코드의 패키지명        │
│ javaPackageName  │                                         │
└──────────────────┴─────────────────────────────────────────┘
```

```exercise
type: word-bank
question: "TurboModule TypeScript 스펙의 빈칸을 채우세요"
code: |
  import type { ___ } from 'react-native';
  import { ___ } from 'react-native';

  export interface Spec extends TurboModule {
    getDeviceModel(): string;
  }

  export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
blanks: ["TurboModule", "TurboModuleRegistry"]
distractors: ["NativeModule", "ReactModule", "BridgeModule", "FabricModule"]
hint: "TurboModule 스펙은 TurboModule 타입과 TurboModuleRegistry를 import합니다"
xp: 5
```

---

## 2. TurboModules

### 2.1 Legacy Native Modules vs TurboModules

```
┌────────────────────────────────────────────────────────────────┐
│          Legacy Native Modules vs TurboModules 비교              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌── Legacy Native Module ──────────────────────────────────┐  │
│  │                                                           │  │
│  │  // Kotlin 구현                                           │  │
│  │  class MyModule(ctx: ReactApplicationContext)              │  │
│  │    : ReactContextBaseJavaModule(ctx) {                    │  │
│  │                                                           │  │
│  │    override fun getName() = "MyModule"                    │  │
│  │                                                           │  │
│  │    @ReactMethod                                           │  │
│  │    fun greet(name: String, promise: Promise) {            │  │
│  │      promise.resolve("Hello, $name!")                     │  │
│  │    }                                                      │  │
│  │  }                                                        │  │
│  │                                                           │  │
│  │  // 패키지 등록                                            │  │
│  │  class MyPackage : ReactPackage {                         │  │
│  │    override fun createNativeModules(ctx) =                │  │
│  │      listOf(MyModule(ctx))  // ← 앱 시작 시 즉시 생성!    │  │
│  │    override fun createViewManagers(ctx) = emptyList()     │  │
│  │  }                                                        │  │
│  │                                                           │  │
│  │  // JS 사용                                               │  │
│  │  const { MyModule } = NativeModules;                      │  │
│  │  const result = await MyModule.greet("Kim");              │  │
│  │  // → Bridge를 통해 비동기 전달                            │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌── TurboModule ────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  // TypeScript Spec (타입 정의)                            │  │
│  │  export interface Spec extends TurboModule {              │  │
│  │    greet(name: string): Promise<string>;                  │  │
│  │  }                                                        │  │
│  │  export default TurboModuleRegistry                       │  │
│  │    .getEnforcing<Spec>('MyModule');                       │  │
│  │                                                           │  │
│  │  // Kotlin 구현 (Codegen 생성 클래스 상속)                 │  │
│  │  class MyModule(ctx: ReactApplicationContext)              │  │
│  │    : NativeMyModuleSpec(ctx) {                            │  │
│  │    // ↑ Codegen이 생성한 abstract class 상속              │  │
│  │                                                           │  │
│  │    override fun getName() = "MyModule"                    │  │
│  │                                                           │  │
│  │    override fun greet(name: String, promise: Promise) {   │  │
│  │      promise.resolve("Hello, $name!")                     │  │
│  │    }                                                      │  │
│  │  }                                                        │  │
│  │                                                           │  │
│  │  // 패키지 등록 (TurboReactPackage)                       │  │
│  │  class MyPackage : TurboReactPackage() {                  │  │
│  │    override fun getModule(name: String, ctx) =            │  │
│  │      if (name == "MyModule") MyModule(ctx)                │  │
│  │      else null  // ← 요청 시에만 생성! (Lazy)             │  │
│  │    override fun getReactModuleInfoProvider() = ...        │  │
│  │  }                                                        │  │
│  │                                                           │  │
│  │  // JS 사용 (타입 안전!)                                   │  │
│  │  import NativeMyModule from './NativeMyModule';           │  │
│  │  const result = await NativeMyModule.greet("Kim");        │  │
│  │  // → JSI를 통해 직접 호출                                 │  │
│  │  // → TypeScript가 타입 검증                               │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 TurboModule의 핵심 차이점

| 항목 | Legacy NativeModule | TurboModule |
|------|-------------------|-------------|
| 통신 방식 | Bridge (JSON) | JSI (직접 참조) |
| 초기화 | 앱 시작 시 전부 | 사용 시 지연 로드 |
| 동기 메서드 | 불가 | 가능 |
| 타입 안전 | 없음 | Codegen 보장 |
| 기반 클래스 | `ReactContextBaseJavaModule` | `NativeXxxSpec` (Codegen 생성) |
| 패키지 | `ReactPackage` | `TurboReactPackage` |
| JS 접근 | `NativeModules.Xxx` | `TurboModuleRegistry.getEnforcing` |
| 메서드 등록 | `@ReactMethod` | abstract override (Codegen 기반) |

### 2.3 TurboModule의 생명주기

```
┌────────────────────────────────────────────────────────────────┐
│              TurboModule 생명주기                                │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  앱 시작                                                        │
│  ┌─────────────────────┐                                       │
│  │ React Native 초기화  │                                       │
│  │ TurboModule Registry │                                       │
│  │ 에 모듈 이름만 등록   │  ← 인스턴스는 아직 없음!              │
│  └──────────┬──────────┘                                       │
│             │                                                   │
│  ... (시간이 지남) ...                                           │
│             │                                                   │
│  JS에서 처음 접근 시                                             │
│  ┌──────────▼──────────┐                                       │
│  │ TurboModuleRegistry │                                       │
│  │ .getEnforcing('Xxx')│ ← 이 시점에 인스턴스 생성!             │
│  └──────────┬──────────┘                                       │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                       │
│  │ Module 초기화        │                                       │
│  │ • 생성자 실행        │                                       │
│  │ • 리소스 할당        │                                       │
│  │ • JSI 바인딩 설정    │                                       │
│  └──────────┬──────────┘                                       │
│             │                                                   │
│             ▼                                                   │
│  ┌─────────────────────┐                                       │
│  │ 메서드 호출 가능     │                                       │
│  │ JSI를 통해 직접 호출 │                                       │
│  │ 동기/비동기 모두     │                                       │
│  └──────────┬──────────┘                                       │
│             │                                                   │
│  앱 종료 시                                                     │
│  ┌──────────▼──────────┐                                       │
│  │ invalidate() 호출   │                                       │
│  │ • 리소스 해제        │                                       │
│  │ • 리스너 제거        │                                       │
│  └─────────────────────┘                                       │
│                                                                 │
│  Android 비유:                                                   │
│  • Hilt @Singleton + Lazy 주입과 유사                           │
│  • 처음 @Inject 접근 시에만 인스턴스 생성                        │
│  • 앱 전체 수명과 동일한 Scope                                   │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.4 성능 비교

```
┌────────────────────────────────────────────────────────────────┐
│          Legacy vs TurboModule 성능 벤치마크                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  테스트 환경: Pixel 7, Android 14, Release 빌드                 │
│                                                                 │
│  1. 앱 시작 시간 (10개 네이티브 모듈)                            │
│     Legacy:  420ms  ████████████████████                        │
│     Turbo:   280ms  █████████████                               │
│     개선:    -33%                                               │
│                                                                 │
│  2. 단순 메서드 호출 (1회, string 반환)                          │
│     Legacy:  2.1ms  ██████████                                  │
│     Turbo:   0.3ms  █                                           │
│     개선:    -86%                                               │
│                                                                 │
│  3. 큰 데이터 전송 (100KB JSON 객체)                             │
│     Legacy:  15ms   ███████████████████████████████             │
│     Turbo:   2ms    ████                                        │
│     개선:    -87%                                               │
│                                                                 │
│  4. 연속 호출 (1000회 반복)                                      │
│     Legacy:  850ms  █████████████████████████████████████████   │
│     Turbo:   120ms  ██████                                      │
│     개선:    -86%                                               │
│                                                                 │
│  5. 메모리 사용량 (10개 모듈, 3개만 사용)                        │
│     Legacy:  +12MB  ████████████████████████                    │
│     Turbo:   +4MB   ████████                                    │
│     개선:    -67%   (사용 안 하는 7개 모듈 초기화 안 함)          │
│                                                                 │
│  핵심: Bridge JSON 직렬화 제거 + Lazy 로딩이                    │
│        모든 지표에서 압도적 개선                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. TurboModule 만들기: 단계별 가이드

### Step 1: TypeScript Spec 파일 작성

모든 지원 타입을 보여주는 완전한 Spec 파일:

```typescript
// src/specs/NativeDeviceInfo.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // ═══ 동기 메서드 (즉시 반환) ═══
  // JSI 덕분에 동기 호출이 가능해짐
  getModel(): string;
  getOSVersion(): string;
  getApiLevel(): number;
  isEmulator(): boolean;

  // ═══ 비동기 메서드 (Promise 반환) ═══
  // 시간이 걸리는 작업은 Promise 사용
  getBatteryLevel(): Promise<number>;
  getFreeDiskSpace(): Promise<number>;
  getIPAddress(): Promise<string>;

  // ═══ 객체 반환 ═══
  getScreenInfo(): {
    width: number;
    height: number;
    density: number;
  };

  // ═══ 배열 반환 ═══
  getAvailableLocales(): string[];

  // ═══ Nullable 반환 ═══
  getCarrierName(): string | null;

  // ═══ 파라미터 있는 메서드 ═══
  setClipboardText(text: string): void;
  getClipboardText(): Promise<string>;

  // ═══ Callback (이벤트 수신) ═══
  addBatteryListener(callback: (level: number) => void): void;

  // ═══ 상수 (앱 시작 시 한 번만 읽힘) ═══
  getConstants(): {
    BRAND: string;
    MANUFACTURER: string;
    IS_TABLET: boolean;
  };
}

export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
```

### Step 2: Kotlin 구현 (전체 코드)

```kotlin
// android/app/src/main/java/com/myapp/DeviceInfoModule.kt

package com.myapp

import android.content.ClipData
import android.content.ClipboardManager
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.os.BatteryManager
import android.os.Build
import android.os.Environment
import android.os.StatFs
import android.telephony.TelephonyManager
import android.util.DisplayMetrics
import android.view.WindowManager
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Callback
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.ReadableMap
import com.facebook.react.bridge.WritableMap
import com.facebook.react.module.annotations.ReactModule
// Codegen이 생성한 Spec 클래스를 import
import com.myapp.specs.NativeDeviceInfoSpec
import java.net.Inet4Address
import java.net.NetworkInterface

@ReactModule(name = DeviceInfoModule.NAME)
class DeviceInfoModule(
    private val reactContext: ReactApplicationContext
) : NativeDeviceInfoSpec(reactContext) {
    // ↑ Codegen이 생성한 abstract 클래스를 상속
    //   모든 Spec 메서드를 override 해야 컴파일됨

    companion object {
        const val NAME = "DeviceInfo"
    }

    override fun getName(): String = NAME

    // ═══ 동기 메서드 구현 ═══

    override fun getModel(): String {
        return Build.MODEL  // 예: "Pixel 7"
    }

    override fun getOSVersion(): String {
        return Build.VERSION.RELEASE  // 예: "14"
    }

    override fun getApiLevel(): Double {
        // Codegen은 number를 Double로 매핑
        return Build.VERSION.SDK_INT.toDouble()  // 예: 34.0
    }

    override fun isEmulator(): Boolean {
        return (Build.FINGERPRINT.startsWith("generic")
                || Build.FINGERPRINT.startsWith("unknown")
                || Build.MODEL.contains("Emulator")
                || Build.MODEL.contains("Android SDK"))
    }

    // ═══ 비동기 메서드 구현 ═══

    override fun getBatteryLevel(promise: Promise) {
        try {
            val intentFilter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
            val batteryStatus = reactContext.registerReceiver(null, intentFilter)
            val level = batteryStatus?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
            val scale = batteryStatus?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1

            if (level == -1 || scale == -1) {
                promise.reject("BATTERY_ERROR", "배터리 정보를 가져올 수 없습니다")
                return
            }

            val batteryPct = level.toDouble() / scale.toDouble()
            promise.resolve(batteryPct)
        } catch (e: Exception) {
            promise.reject("BATTERY_ERROR", e.message, e)
        }
    }

    override fun getFreeDiskSpace(promise: Promise) {
        try {
            val stat = StatFs(Environment.getExternalStorageDirectory().path)
            val freeBytes = stat.availableBlocksLong * stat.blockSizeLong
            promise.resolve(freeBytes.toDouble())
        } catch (e: Exception) {
            promise.reject("DISK_ERROR", e.message, e)
        }
    }

    override fun getIPAddress(promise: Promise) {
        try {
            val interfaces = NetworkInterface.getNetworkInterfaces()
            while (interfaces.hasMoreElements()) {
                val networkInterface = interfaces.nextElement()
                val addresses = networkInterface.inetAddresses
                while (addresses.hasMoreElements()) {
                    val address = addresses.nextElement()
                    if (!address.isLoopbackAddress && address is Inet4Address) {
                        promise.resolve(address.hostAddress)
                        return
                    }
                }
            }
            promise.resolve("0.0.0.0")
        } catch (e: Exception) {
            promise.reject("NETWORK_ERROR", e.message, e)
        }
    }

    // ═══ 객체 반환 ═══

    override fun getScreenInfo(): WritableMap {
        val windowManager = reactContext.getSystemService(Context.WINDOW_SERVICE)
            as WindowManager
        val metrics = DisplayMetrics()
        windowManager.defaultDisplay.getMetrics(metrics)

        val map = Arguments.createMap()
        map.putDouble("width", metrics.widthPixels.toDouble())
        map.putDouble("height", metrics.heightPixels.toDouble())
        map.putDouble("density", metrics.density.toDouble())
        return map
    }

    // ═══ 배열 반환 ═══

    override fun getAvailableLocales(): WritableArray {
        val array = Arguments.createArray()
        java.util.Locale.getAvailableLocales().forEach { locale ->
            array.pushString(locale.toString())
        }
        return array
    }

    // ═══ Nullable 반환 ═══

    override fun getCarrierName(): String? {
        val telephonyManager = reactContext.getSystemService(
            Context.TELEPHONY_SERVICE
        ) as? TelephonyManager
        return telephonyManager?.networkOperatorName
    }

    // ═══ 파라미터 있는 메서드 ═══

    override fun setClipboardText(text: String) {
        val clipboard = reactContext.getSystemService(Context.CLIPBOARD_SERVICE)
            as ClipboardManager
        val clip = ClipData.newPlainText("ReactNative", text)
        clipboard.setPrimaryClip(clip)
    }

    override fun getClipboardText(promise: Promise) {
        try {
            val clipboard = reactContext.getSystemService(Context.CLIPBOARD_SERVICE)
                as ClipboardManager
            val text = clipboard.primaryClip?.getItemAt(0)?.text?.toString() ?: ""
            promise.resolve(text)
        } catch (e: Exception) {
            promise.reject("CLIPBOARD_ERROR", e.message, e)
        }
    }

    // ═══ Callback ═══

    override fun addBatteryListener(callback: Callback) {
        val intentFilter = IntentFilter(Intent.ACTION_BATTERY_CHANGED)
        val batteryStatus = reactContext.registerReceiver(null, intentFilter)
        val level = batteryStatus?.getIntExtra(BatteryManager.EXTRA_LEVEL, -1) ?: -1
        val scale = batteryStatus?.getIntExtra(BatteryManager.EXTRA_SCALE, -1) ?: -1
        val batteryPct = level.toDouble() / scale.toDouble()
        callback.invoke(batteryPct)
    }

    // ═══ 상수 ═══

    override fun getTypedExportedConstants(): MutableMap<String, Any> {
        return mutableMapOf(
            "BRAND" to Build.BRAND,
            "MANUFACTURER" to Build.MANUFACTURER,
            "IS_TABLET" to (reactContext.resources.configuration
                .screenLayout and android.content.res.Configuration
                .SCREENLAYOUT_SIZE_MASK >=
                android.content.res.Configuration.SCREENLAYOUT_SIZE_LARGE)
        )
    }
}
```

### Step 3: 패키지 등록

```kotlin
// android/app/src/main/java/com/myapp/DeviceInfoPackage.kt

package com.myapp

import com.facebook.react.TurboReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider

class DeviceInfoPackage : TurboReactPackage() {

    override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? {
        return when (name) {
            DeviceInfoModule.NAME -> DeviceInfoModule(reactContext)
            else -> null
        }
    }

    override fun getReactModuleInfoProvider(): ReactModuleInfoProvider {
        return ReactModuleInfoProvider {
            mapOf(
                DeviceInfoModule.NAME to ReactModuleInfo(
                    DeviceInfoModule.NAME,   // name
                    DeviceInfoModule.NAME,   // className
                    false,                    // canOverrideExistingModule
                    false,                    // needsEagerInit (false = Lazy!)
                    false,                    // isCxxModule
                    true                      // isTurboModule ← 중요!
                )
            )
        }
    }
}
```

```kotlin
// android/app/src/main/java/com/myapp/MainApplication.kt

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

class MainApplication : Application(), ReactApplication {

    override val reactNativeHost: ReactNativeHost =
        object : DefaultReactNativeHost(this) {
            override fun getPackages(): List<ReactPackage> =
                PackageList(this).packages.apply {
                    // 여기에 커스텀 패키지 추가!
                    add(DeviceInfoPackage())
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
        if (BuildConfig.IS_NEW_ARCHITECTURE_ENABLED) {
            load()
        }
    }
}
```

### Step 4: JavaScript/TypeScript에서 사용

```typescript
// src/screens/DeviceInfoScreen.tsx

import React, { useEffect, useState } from 'react';
import { View, Text, StyleSheet, ScrollView, Button } from 'react-native';
import NativeDeviceInfo from '../specs/NativeDeviceInfo';

function DeviceInfoScreen() {
  const [batteryLevel, setBatteryLevel] = useState<number | null>(null);
  const [diskSpace, setDiskSpace] = useState<number | null>(null);
  const [ipAddress, setIpAddress] = useState<string>('');

  // 동기 메서드 — 즉시 값 반환 (Bridge 시절에는 불가능했음!)
  const model = NativeDeviceInfo.getModel();
  const osVersion = NativeDeviceInfo.getOSVersion();
  const apiLevel = NativeDeviceInfo.getApiLevel();
  const isEmulator = NativeDeviceInfo.isEmulator();
  const screenInfo = NativeDeviceInfo.getScreenInfo();
  const carrierName = NativeDeviceInfo.getCarrierName();

  // 상수 — 앱 시작 시 한 번만 읽힘
  const constants = NativeDeviceInfo.getConstants();

  useEffect(() => {
    // 비동기 메서드 — Promise 반환
    async function loadAsyncData() {
      try {
        const battery = await NativeDeviceInfo.getBatteryLevel();
        setBatteryLevel(battery);

        const disk = await NativeDeviceInfo.getFreeDiskSpace();
        setDiskSpace(disk);

        const ip = await NativeDeviceInfo.getIPAddress();
        setIpAddress(ip);
      } catch (error) {
        console.error('디바이스 정보 로딩 실패:', error);
      }
    }

    loadAsyncData();
  }, []);

  const handleCopyModel = () => {
    // 동기 호출 — void 반환
    NativeDeviceInfo.setClipboardText(model);
  };

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.header}>디바이스 정보</Text>

      <InfoRow label="모델" value={model} />
      <InfoRow label="OS 버전" value={osVersion} />
      <InfoRow label="API 레벨" value={String(apiLevel)} />
      <InfoRow label="에뮬레이터" value={isEmulator ? '예' : '아니오'} />
      <InfoRow label="통신사" value={carrierName ?? '없음'} />
      <InfoRow label="브랜드" value={constants.BRAND} />
      <InfoRow label="제조사" value={constants.MANUFACTURER} />
      <InfoRow label="태블릿" value={constants.IS_TABLET ? '예' : '아니오'} />
      <InfoRow label="해상도"
        value={`${screenInfo.width}x${screenInfo.height} (${screenInfo.density}x)`} />
      <InfoRow label="배터리"
        value={batteryLevel ? `${(batteryLevel * 100).toFixed(1)}%` : '로딩 중...'} />
      <InfoRow label="디스크 여유"
        value={diskSpace ? `${(diskSpace / 1024 / 1024 / 1024).toFixed(2)} GB` : '로딩 중...'} />
      <InfoRow label="IP 주소" value={ipAddress || '로딩 중...'} />

      <Button title="모델명 복사" onPress={handleCopyModel} />
    </ScrollView>
  );
}

function InfoRow({ label, value }: { label: string; value: string }) {
  return (
    <View style={styles.row}>
      <Text style={styles.label}>{label}</Text>
      <Text style={styles.value}>{value}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  header: { fontSize: 24, fontWeight: 'bold', marginBottom: 16 },
  row: { flexDirection: 'row', justifyContent: 'space-between', paddingVertical: 8,
    borderBottomWidth: 1, borderBottomColor: '#eee' },
  label: { fontSize: 16, color: '#666' },
  value: { fontSize: 16, fontWeight: '500' },
});

export default DeviceInfoScreen;
```

---

## 4. 완전한 실전 예제: Calendar TurboModule

### 4.1 Spec 정의

```typescript
// src/specs/NativeCalendar.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

type CalendarEvent = {
  id: string;
  title: string;
  description: string;
  startTime: number;   // Unix timestamp (ms)
  endTime: number;
  location: string;
  isAllDay: boolean;
};

type CreateEventParams = {
  title: string;
  description: string;
  startTime: number;
  endTime: number;
  location: string;
  isAllDay: boolean;
};

export interface Spec extends TurboModule {
  // 캘린더 접근 권한 확인
  checkPermission(): Promise<string>;  // 'granted' | 'denied' | 'never_asked'

  // 캘린더 접근 권한 요청
  requestPermission(): Promise<boolean>;

  // 특정 날짜 범위의 이벤트 조회
  getEvents(startDate: number, endDate: number): Promise<Object[]>;

  // 이벤트 생성
  createEvent(params: Object): Promise<string>;  // 생성된 이벤트 ID 반환

  // 이벤트 삭제
  deleteEvent(eventId: string): Promise<boolean>;

  // 캘린더 목록 (동기)
  getCalendarCount(): number;
}

export default TurboModuleRegistry.getEnforcing<Spec>('Calendar');
```

### 4.2 Kotlin 구현

```kotlin
// android/app/src/main/java/com/myapp/CalendarModule.kt

package com.myapp

import android.Manifest
import android.content.ContentUris
import android.content.ContentValues
import android.content.pm.PackageManager
import android.provider.CalendarContract
import androidx.core.content.ContextCompat
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.ReadableMap
import com.facebook.react.bridge.WritableArray
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeCalendarSpec
import java.util.TimeZone

@ReactModule(name = CalendarModule.NAME)
class CalendarModule(
    private val reactContext: ReactApplicationContext
) : NativeCalendarSpec(reactContext) {

    companion object {
        const val NAME = "Calendar"
    }

    override fun getName(): String = NAME

    override fun checkPermission(promise: Promise) {
        val readPermission = ContextCompat.checkPermission(
            reactContext,
            Manifest.permission.READ_CALENDAR
        )
        val writePermission = ContextCompat.checkPermission(
            reactContext,
            Manifest.permission.WRITE_CALENDAR
        )

        val result = when {
            readPermission == PackageManager.PERMISSION_GRANTED &&
                writePermission == PackageManager.PERMISSION_GRANTED -> "granted"
            else -> "denied"
        }
        promise.resolve(result)
    }

    override fun requestPermission(promise: Promise) {
        // React Native에서는 PermissionsAndroid JS API를 사용하는 것이 일반적
        // 여기서는 현재 상태만 확인
        val granted = ContextCompat.checkPermission(
            reactContext,
            Manifest.permission.READ_CALENDAR
        ) == PackageManager.PERMISSION_GRANTED
        promise.resolve(granted)
    }

    override fun getEvents(startDate: Double, endDate: Double, promise: Promise) {
        try {
            val projection = arrayOf(
                CalendarContract.Events._ID,
                CalendarContract.Events.TITLE,
                CalendarContract.Events.DESCRIPTION,
                CalendarContract.Events.DTSTART,
                CalendarContract.Events.DTEND,
                CalendarContract.Events.EVENT_LOCATION,
                CalendarContract.Events.ALL_DAY,
            )

            val selection = "(${CalendarContract.Events.DTSTART} >= ?) AND " +
                    "(${CalendarContract.Events.DTSTART} <= ?)"
            val selectionArgs = arrayOf(
                startDate.toLong().toString(),
                endDate.toLong().toString()
            )

            val cursor = reactContext.contentResolver.query(
                CalendarContract.Events.CONTENT_URI,
                projection,
                selection,
                selectionArgs,
                "${CalendarContract.Events.DTSTART} ASC"
            )

            val events: WritableArray = Arguments.createArray()

            cursor?.use {
                while (it.moveToNext()) {
                    val event = Arguments.createMap().apply {
                        putString("id", it.getLong(0).toString())
                        putString("title", it.getString(1) ?: "")
                        putString("description", it.getString(2) ?: "")
                        putDouble("startTime", it.getLong(3).toDouble())
                        putDouble("endTime", (it.getLong(4)).toDouble())
                        putString("location", it.getString(5) ?: "")
                        putBoolean("isAllDay", it.getInt(6) == 1)
                    }
                    events.pushMap(event)
                }
            }

            promise.resolve(events)
        } catch (e: SecurityException) {
            promise.reject("PERMISSION_ERROR", "캘린더 접근 권한이 없습니다", e)
        } catch (e: Exception) {
            promise.reject("CALENDAR_ERROR", "이벤트 조회 실패: ${e.message}", e)
        }
    }

    override fun createEvent(params: ReadableMap, promise: Promise) {
        try {
            val values = ContentValues().apply {
                put(CalendarContract.Events.CALENDAR_ID, 1) // 기본 캘린더
                put(CalendarContract.Events.TITLE, params.getString("title"))
                put(CalendarContract.Events.DESCRIPTION,
                    params.getString("description") ?: "")
                put(CalendarContract.Events.DTSTART,
                    params.getDouble("startTime").toLong())
                put(CalendarContract.Events.DTEND,
                    params.getDouble("endTime").toLong())
                put(CalendarContract.Events.EVENT_LOCATION,
                    params.getString("location") ?: "")
                put(CalendarContract.Events.ALL_DAY,
                    if (params.getBoolean("isAllDay")) 1 else 0)
                put(CalendarContract.Events.EVENT_TIMEZONE,
                    TimeZone.getDefault().id)
            }

            val uri = reactContext.contentResolver.insert(
                CalendarContract.Events.CONTENT_URI, values
            )

            if (uri != null) {
                val eventId = ContentUris.parseId(uri)
                promise.resolve(eventId.toString())
            } else {
                promise.reject("CREATE_ERROR", "이벤트 생성에 실패했습니다")
            }
        } catch (e: SecurityException) {
            promise.reject("PERMISSION_ERROR", "캘린더 쓰기 권한이 없습니다", e)
        } catch (e: Exception) {
            promise.reject("CREATE_ERROR", "이벤트 생성 실패: ${e.message}", e)
        }
    }

    override fun deleteEvent(eventId: String, promise: Promise) {
        try {
            val uri = ContentUris.withAppendedId(
                CalendarContract.Events.CONTENT_URI, eventId.toLong()
            )
            val rowsDeleted = reactContext.contentResolver.delete(uri, null, null)
            promise.resolve(rowsDeleted > 0)
        } catch (e: Exception) {
            promise.reject("DELETE_ERROR", "이벤트 삭제 실패: ${e.message}", e)
        }
    }

    override fun getCalendarCount(): Double {
        return try {
            val cursor = reactContext.contentResolver.query(
                CalendarContract.Calendars.CONTENT_URI,
                arrayOf(CalendarContract.Calendars._ID),
                null, null, null
            )
            val count = cursor?.count ?: 0
            cursor?.close()
            count.toDouble()
        } catch (e: Exception) {
            0.0
        }
    }
}
```

### 4.3 JavaScript에서 사용

```typescript
// src/screens/CalendarScreen.tsx

import React, { useEffect, useState } from 'react';
import {
  View, Text, FlatList, StyleSheet, Alert,
  Pressable, PermissionsAndroid, Platform
} from 'react-native';
import NativeCalendar from '../specs/NativeCalendar';

type CalendarEvent = {
  id: string;
  title: string;
  description: string;
  startTime: number;
  endTime: number;
  location: string;
  isAllDay: boolean;
};

function CalendarScreen() {
  const [events, setEvents] = useState<CalendarEvent[]>([]);
  const [hasPermission, setHasPermission] = useState(false);

  // 동기 메서드 — 즉시 결과
  const calendarCount = NativeCalendar.getCalendarCount();

  useEffect(() => {
    checkAndRequestPermission();
  }, []);

  async function checkAndRequestPermission() {
    if (Platform.OS === 'android') {
      const granted = await PermissionsAndroid.request(
        PermissionsAndroid.PERMISSIONS.READ_CALENDAR,
        {
          title: '캘린더 접근 권한',
          message: '일정을 표시하려면 캘린더 접근이 필요합니다.',
          buttonPositive: '허용',
          buttonNegative: '거부',
        }
      );
      setHasPermission(granted === PermissionsAndroid.RESULTS.GRANTED);
      if (granted === PermissionsAndroid.RESULTS.GRANTED) {
        loadEvents();
      }
    }
  }

  async function loadEvents() {
    try {
      const now = Date.now();
      const oneWeekLater = now + 7 * 24 * 60 * 60 * 1000;
      const result = await NativeCalendar.getEvents(now, oneWeekLater);
      setEvents(result as CalendarEvent[]);
    } catch (error) {
      Alert.alert('오류', '일정을 불러올 수 없습니다.');
    }
  }

  async function createTestEvent() {
    try {
      const startTime = Date.now() + 60 * 60 * 1000; // 1시간 후
      const endTime = startTime + 60 * 60 * 1000;     // 2시간 후

      const eventId = await NativeCalendar.createEvent({
        title: 'React Native 공부',
        description: 'TurboModule 실습',
        startTime,
        endTime,
        location: '홈 오피스',
        isAllDay: false,
      });

      Alert.alert('성공', `이벤트 생성됨 (ID: ${eventId})`);
      loadEvents(); // 목록 새로고침
    } catch (error) {
      Alert.alert('오류', '이벤트 생성에 실패했습니다.');
    }
  }

  async function handleDeleteEvent(eventId: string) {
    try {
      const deleted = await NativeCalendar.deleteEvent(eventId);
      if (deleted) {
        setEvents(prev => prev.filter(e => e.id !== eventId));
      }
    } catch (error) {
      Alert.alert('오류', '이벤트 삭제에 실패했습니다.');
    }
  }

  const formatTime = (timestamp: number) => {
    return new Date(timestamp).toLocaleString('ko-KR');
  };

  return (
    <View style={styles.container}>
      <Text style={styles.header}>캘린더 (총 {calendarCount}개)</Text>

      <Pressable style={styles.addButton} onPress={createTestEvent}>
        <Text style={styles.addButtonText}>+ 테스트 이벤트 추가</Text>
      </Pressable>

      <FlatList
        data={events}
        keyExtractor={item => item.id}
        renderItem={({ item }) => (
          <View style={styles.eventCard}>
            <Text style={styles.eventTitle}>{item.title}</Text>
            <Text style={styles.eventTime}>
              {formatTime(item.startTime)} ~ {formatTime(item.endTime)}
            </Text>
            {item.location ? (
              <Text style={styles.eventLocation}>{item.location}</Text>
            ) : null}
            <Pressable
              style={styles.deleteButton}
              onPress={() => handleDeleteEvent(item.id)}
            >
              <Text style={styles.deleteText}>삭제</Text>
            </Pressable>
          </View>
        )}
        ListEmptyComponent={
          <Text style={styles.empty}>
            {hasPermission ? '이번 주 일정이 없습니다' : '권한이 필요합니다'}
          </Text>
        }
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, backgroundColor: '#f5f5f5' },
  header: { fontSize: 24, fontWeight: 'bold', marginBottom: 16 },
  addButton: { backgroundColor: '#4A90D9', padding: 12, borderRadius: 8,
    alignItems: 'center', marginBottom: 16 },
  addButtonText: { color: 'white', fontSize: 16, fontWeight: '600' },
  eventCard: { backgroundColor: 'white', padding: 16, borderRadius: 8,
    marginBottom: 8, elevation: 2 },
  eventTitle: { fontSize: 18, fontWeight: '600' },
  eventTime: { fontSize: 14, color: '#666', marginTop: 4 },
  eventLocation: { fontSize: 14, color: '#888', marginTop: 2 },
  deleteButton: { position: 'absolute', top: 16, right: 16 },
  deleteText: { color: 'red', fontSize: 14 },
  empty: { textAlign: 'center', color: '#999', marginTop: 32, fontSize: 16 },
});

export default CalendarScreen;
```

---

## 요약

```
┌────────────────────────────────────────────────────────────────┐
│              Codegen + TurboModule 핵심 정리                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Codegen:                                                       │
│  • TypeScript Spec → C++/Kotlin 인터페이스 자동 생성             │
│  • 타입 안전성 보장 (컴파일 타임 검증)                            │
│  • Android의 kapt/KSP + Room @Dao와 유사한 개념                  │
│  • 파일명: Native*.ts, 빌드 시 자동 실행                         │
│                                                                 │
│  TurboModules:                                                  │
│  • JSI 기반 직접 통신 (Bridge 제거)                              │
│  • Lazy Loading (사용 시에만 초기화)                              │
│  • 동기/비동기 메서드 모두 지원                                   │
│  • Codegen 생성 클래스를 상속하여 구현                            │
│                                                                 │
│  구현 순서:                                                      │
│  1. TypeScript Spec 작성 (NativeXxx.ts)                         │
│  2. package.json에 codegenConfig 설정                           │
│  3. Kotlin 모듈 구현 (NativeXxxSpec 상속)                       │
│  4. TurboReactPackage로 패키지 등록                              │
│  5. MainApplication에 패키지 추가                                │
│  6. JS/TS에서 import하여 사용                                    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
