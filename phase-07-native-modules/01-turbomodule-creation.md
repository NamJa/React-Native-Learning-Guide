# TurboModule 만들기 — Kotlin으로 네이티브 기능 구현하기

## 목차
1. [네이티브 모듈이 필요한 경우](#1-네이티브-모듈이-필요한-경우)
2. [TurboModule 생성 단계별 가이드](#2-turbomodule-생성-단계별-가이드)
3. [완전한 예제 1: Device Info 모듈 (동기 메서드)](#3-예제-1-device-info-모듈)
4. [완전한 예제 2: Biometric Auth 모듈 (비동기 + Promise)](#4-예제-2-biometric-auth-모듈)
5. [완전한 예제 3: Native Toast/Snackbar 모듈 (이벤트)](#5-예제-3-native-toast-snackbar-모듈)
6. [에러 처리 패턴](#6-에러-처리-패턴)
7. [테스트와 디버깅](#7-테스트와-디버깅)

---

## 1. 네이티브 모듈이 필요한 경우

React Native에서 JavaScript만으로는 할 수 없는 작업이 있다. 플랫폼 고유의 API에 접근해야 할 때 네이티브 모듈을 만든다.

```
┌────────────────────────────────────────────────────────────────┐
│              네이티브 모듈이 필요한 대표 사례                      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  하드웨어 접근:                                                  │
│  • 카메라 (CameraX)                                             │
│  • 블루투스 (BLE 통신)                                          │
│  • 센서 (가속도계, 자이로스코프, 근접 센서)                       │
│  • NFC                                                          │
│  • 생체 인증 (지문, 얼굴)                                       │
│                                                                 │
│  플랫폼 API:                                                    │
│  • 알림 채널 관리                                                │
│  • 앱 설정 화면 열기                                             │
│  • Intent로 다른 앱 실행                                         │
│  • ContentProvider 접근 (연락처, 캘린더)                         │
│  • WorkManager (백그라운드 작업)                                  │
│                                                                 │
│  성능:                                                          │
│  • 이미지 처리 (네이티브 속도 필요)                               │
│  • 암호화/복호화                                                 │
│  • 대용량 파일 처리                                              │
│  • SQLite 직접 접근 (Room DB)                                    │
│                                                                 │
│  기존 코드 재사용:                                               │
│  • 기존 Android 앱의 Kotlin 클래스                               │
│  • 기존 Android 라이브러리                                       │
│  • 회사 내부 SDK                                                 │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. TurboModule 생성 단계별 가이드

### Step 1: TypeScript Spec 파일 작성

Spec 파일은 네이티브 모듈의 "계약서(Contract)"이다. JS와 Native가 어떤 타입으로 통신할지 정의한다.

```typescript
// src/specs/NativeXxx.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // ═══ 지원되는 모든 타입 ═══

  // — 동기 메서드 (즉시 반환) —
  syncMethod(param: string): string;
  getNumber(): number;
  getBoolean(): boolean;

  // — 비동기 메서드 (Promise) —
  asyncMethod(param: string): Promise<string>;

  // — void 메서드 (반환값 없음) —
  fireAndForget(message: string): void;

  // — 콜백 메서드 —
  withCallback(
    onSuccess: (result: string) => void,
    onError: (error: string) => void
  ): void;

  // — 상수 (앱 시작 시 한 번만 읽힘) —
  getConstants(): {
    KEY1: string;
    KEY2: number;
  };

  // — 이벤트 발송 (Native → JS) —
  // 이벤트는 Spec에 직접 정의하지 않고
  // EventEmitter 패턴으로 별도 처리
  addListener(eventName: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('ModuleName');
```

### Step 2: package.json에 Codegen 설정

```json
{
  "name": "MyApp",
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

### Step 3: Kotlin 모듈 클래스 작성

```kotlin
// android/app/src/main/java/com/myapp/XxxModule.kt

package com.myapp

import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.ReactMethod
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.modules.core.DeviceEventManagerModule
// Codegen이 생성한 abstract 클래스 import
import com.myapp.specs.NativeXxxSpec

@ReactModule(name = XxxModule.NAME)
class XxxModule(
    private val reactContext: ReactApplicationContext
) : NativeXxxSpec(reactContext) {

    companion object {
        const val NAME = "Xxx"
    }

    override fun getName(): String = NAME

    // 각 Spec 메서드를 override하여 구현
    // 컴파일러가 타입을 강제 검증!

    // 이벤트 발송 헬퍼
    private fun sendEvent(eventName: String, params: Any?) {
        reactContext
            .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
            .emit(eventName, params)
    }
}
```

### Step 4: 패키지 클래스 생성 및 등록

```kotlin
// android/app/src/main/java/com/myapp/XxxPackage.kt

package com.myapp

import com.facebook.react.TurboReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider

class XxxPackage : TurboReactPackage() {

    override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? {
        return when (name) {
            XxxModule.NAME -> XxxModule(reactContext)
            else -> null
        }
    }

    override fun getReactModuleInfoProvider(): ReactModuleInfoProvider {
        return ReactModuleInfoProvider {
            mapOf(
                XxxModule.NAME to ReactModuleInfo(
                    XxxModule.NAME,
                    XxxModule.NAME,
                    false,   // canOverrideExistingModule
                    false,   // needsEagerInit
                    false,   // isCxxModule
                    true     // isTurboModule
                )
            )
        }
    }
}
```

```kotlin
// MainApplication.kt에 패키지 추가
override fun getPackages(): List<ReactPackage> =
    PackageList(this).packages.apply {
        add(XxxPackage())
    }
```

### Step 5: JavaScript/TypeScript에서 사용

```typescript
// src/hooks/useXxx.ts (또는 직접 import)

import NativeXxx from '../specs/NativeXxx';

// 동기 메서드
const value = NativeXxx.syncMethod('param');

// 비동기 메서드
const result = await NativeXxx.asyncMethod('param');

// 이벤트 리스너
import { NativeEventEmitter } from 'react-native';
const emitter = new NativeEventEmitter(NativeXxx);
const subscription = emitter.addListener('eventName', (data) => {
  console.log(data);
});
// 해제: subscription.remove();
```

---

## 3. 예제 1: Device Info 모듈

동기 메서드만 사용하는 간단한 모듈. 디바이스 정보를 즉시 반환한다.

### Spec 정의

```typescript
// src/specs/NativeDeviceInfo.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // 동기 메서드들 — JSI 덕분에 즉시 반환 가능
  getDeviceModel(): string;
  getAndroidVersion(): string;
  getApiLevel(): number;
  getTotalMemory(): number;
  getAvailableMemory(): number;
  isTablet(): boolean;
  getLanguage(): string;
  getTimezone(): string;

  // 상수
  getConstants(): {
    BRAND: string;
    MANUFACTURER: string;
    DEVICE_NAME: string;
    BUILD_NUMBER: string;
    APP_VERSION: string;
  };
}

export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
```

### Kotlin 구현

```kotlin
// android/app/src/main/java/com/myapp/DeviceInfoModule.kt

package com.myapp

import android.app.ActivityManager
import android.content.Context
import android.content.pm.PackageManager
import android.content.res.Configuration
import android.os.Build
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeDeviceInfoSpec
import java.util.Locale
import java.util.TimeZone

@ReactModule(name = DeviceInfoModule.NAME)
class DeviceInfoModule(
    private val reactContext: ReactApplicationContext
) : NativeDeviceInfoSpec(reactContext) {

    companion object {
        const val NAME = "DeviceInfo"
    }

    override fun getName(): String = NAME

    override fun getDeviceModel(): String {
        return Build.MODEL  // "Pixel 7", "Galaxy S24" 등
    }

    override fun getAndroidVersion(): String {
        return Build.VERSION.RELEASE  // "14", "13" 등
    }

    override fun getApiLevel(): Double {
        return Build.VERSION.SDK_INT.toDouble()  // 34.0 등
    }

    override fun getTotalMemory(): Double {
        val activityManager = reactContext.getSystemService(
            Context.ACTIVITY_SERVICE
        ) as ActivityManager
        val memInfo = ActivityManager.MemoryInfo()
        activityManager.getMemoryInfo(memInfo)
        return memInfo.totalMem.toDouble()  // 바이트 단위
    }

    override fun getAvailableMemory(): Double {
        val activityManager = reactContext.getSystemService(
            Context.ACTIVITY_SERVICE
        ) as ActivityManager
        val memInfo = ActivityManager.MemoryInfo()
        activityManager.getMemoryInfo(memInfo)
        return memInfo.availMem.toDouble()
    }

    override fun isTablet(): Boolean {
        val screenLayout = reactContext.resources.configuration.screenLayout
        val screenSize = screenLayout and Configuration.SCREENLAYOUT_SIZE_MASK
        return screenSize >= Configuration.SCREENLAYOUT_SIZE_LARGE
    }

    override fun getLanguage(): String {
        return Locale.getDefault().toLanguageTag()  // "ko-KR", "en-US" 등
    }

    override fun getTimezone(): String {
        return TimeZone.getDefault().id  // "Asia/Seoul" 등
    }

    override fun getTypedExportedConstants(): MutableMap<String, Any> {
        val packageInfo = try {
            reactContext.packageManager.getPackageInfo(
                reactContext.packageName, 0
            )
        } catch (e: PackageManager.NameNotFoundException) {
            null
        }

        return mutableMapOf(
            "BRAND" to Build.BRAND,
            "MANUFACTURER" to Build.MANUFACTURER,
            "DEVICE_NAME" to Build.DEVICE,
            "BUILD_NUMBER" to (packageInfo?.let {
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                    it.longVersionCode.toString()
                } else {
                    @Suppress("DEPRECATION")
                    it.versionCode.toString()
                }
            } ?: "unknown"),
            "APP_VERSION" to (packageInfo?.versionName ?: "unknown")
        )
    }
}
```

### 패키지 등록

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
                    DeviceInfoModule.NAME,
                    DeviceInfoModule.NAME,
                    false, false, false, true
                )
            )
        }
    }
}
```

### JavaScript 사용

```typescript
// src/hooks/useDeviceInfo.ts

import NativeDeviceInfo from '../specs/NativeDeviceInfo';

export function useDeviceInfo() {
  // 모든 동기 메서드 — 즉시 반환!
  const info = {
    model: NativeDeviceInfo.getDeviceModel(),
    androidVersion: NativeDeviceInfo.getAndroidVersion(),
    apiLevel: NativeDeviceInfo.getApiLevel(),
    totalMemoryGB: (NativeDeviceInfo.getTotalMemory() / 1024 / 1024 / 1024).toFixed(1),
    availableMemoryGB: (NativeDeviceInfo.getAvailableMemory() / 1024 / 1024 / 1024).toFixed(1),
    isTablet: NativeDeviceInfo.isTablet(),
    language: NativeDeviceInfo.getLanguage(),
    timezone: NativeDeviceInfo.getTimezone(),
    ...NativeDeviceInfo.getConstants(),
  };

  return info;
}
```

```tsx
// src/screens/DeviceInfoScreen.tsx

import React from 'react';
import { View, Text, ScrollView, StyleSheet } from 'react-native';
import { useDeviceInfo } from '../hooks/useDeviceInfo';

export default function DeviceInfoScreen() {
  const info = useDeviceInfo();

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.header}>디바이스 정보</Text>
      <Row label="모델" value={info.model} />
      <Row label="Android 버전" value={info.androidVersion} />
      <Row label="API 레벨" value={String(info.apiLevel)} />
      <Row label="전체 메모리" value={`${info.totalMemoryGB} GB`} />
      <Row label="사용 가능 메모리" value={`${info.availableMemoryGB} GB`} />
      <Row label="태블릿" value={info.isTablet ? '예' : '아니오'} />
      <Row label="언어" value={info.language} />
      <Row label="시간대" value={info.timezone} />
      <Row label="브랜드" value={info.BRAND} />
      <Row label="제조사" value={info.MANUFACTURER} />
      <Row label="앱 버전" value={info.APP_VERSION} />
      <Row label="빌드 번호" value={info.BUILD_NUMBER} />
    </ScrollView>
  );
}

function Row({ label, value }: { label: string; value: string }) {
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
  row: { flexDirection: 'row', justifyContent: 'space-between',
    paddingVertical: 12, borderBottomWidth: 1, borderBottomColor: '#eee' },
  label: { fontSize: 15, color: '#555' },
  value: { fontSize: 15, fontWeight: '600' },
});
```

---

## 4. 예제 2: Biometric Auth 모듈

생체 인증(지문/얼굴)을 처리하는 비동기 모듈. Promise를 적극 활용한다.

### Spec 정의

```typescript
// src/specs/NativeBiometricAuth.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // 생체 인증 지원 여부 확인 (동기)
  isBiometricAvailable(): boolean;

  // 지원되는 인증 타입 (동기)
  getSupportedBiometricType(): string; // 'fingerprint' | 'face' | 'iris' | 'none'

  // 생체 인증 실행 (비동기 — 사용자 상호작용 필요)
  authenticate(title: string, subtitle: string, description: string): Promise<boolean>;

  // 생체 인증으로 데이터 암호화/복호화 (비동기)
  encryptWithBiometric(key: string, data: string): Promise<string>;
  decryptWithBiometric(key: string, encryptedData: string): Promise<string>;

  // 등록된 생체 정보 변경 감지
  hasNewBiometricEnrolled(): Promise<boolean>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('BiometricAuth');
```

### Kotlin 구현

```kotlin
// android/app/src/main/java/com/myapp/BiometricAuthModule.kt

package com.myapp

import android.os.Build
import androidx.biometric.BiometricManager
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.UiThreadUtil
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeBiometricAuthSpec

@ReactModule(name = BiometricAuthModule.NAME)
class BiometricAuthModule(
    private val reactContext: ReactApplicationContext
) : NativeBiometricAuthSpec(reactContext) {

    companion object {
        const val NAME = "BiometricAuth"

        // 에러 코드 상수
        const val ERROR_NOT_AVAILABLE = "BIOMETRIC_NOT_AVAILABLE"
        const val ERROR_NOT_ENROLLED = "BIOMETRIC_NOT_ENROLLED"
        const val ERROR_USER_CANCEL = "USER_CANCEL"
        const val ERROR_LOCKOUT = "BIOMETRIC_LOCKOUT"
        const val ERROR_UNKNOWN = "BIOMETRIC_UNKNOWN"
    }

    override fun getName(): String = NAME

    private val biometricManager: BiometricManager by lazy {
        BiometricManager.from(reactContext)
    }

    override fun isBiometricAvailable(): Boolean {
        val result = biometricManager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG or
            BiometricManager.Authenticators.BIOMETRIC_WEAK
        )
        return result == BiometricManager.BIOMETRIC_SUCCESS
    }

    override fun getSupportedBiometricType(): String {
        // Android는 구체적인 생체 인증 타입을 API로 구분하기 어려움
        // PackageManager feature로 확인
        val pm = reactContext.packageManager
        return when {
            pm.hasSystemFeature("android.hardware.fingerprint") -> "fingerprint"
            pm.hasSystemFeature("android.hardware.biometrics.face") -> "face"
            pm.hasSystemFeature("android.hardware.biometrics.iris") -> "iris"
            else -> "none"
        }
    }

    override fun authenticate(
        title: String,
        subtitle: String,
        description: String,
        promise: Promise
    ) {
        // BiometricPrompt은 Main Thread에서 실행해야 함
        UiThreadUtil.runOnUiThread {
            try {
                val activity = reactContext.currentActivity as? FragmentActivity
                if (activity == null) {
                    promise.reject(ERROR_UNKNOWN, "Activity를 찾을 수 없습니다")
                    return@runOnUiThread
                }

                val executor = ContextCompat.getMainExecutor(reactContext)

                val callback = object : BiometricPrompt.AuthenticationCallback() {
                    override fun onAuthenticationSucceeded(
                        result: BiometricPrompt.AuthenticationResult
                    ) {
                        promise.resolve(true)
                    }

                    override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                        val code = when (errorCode) {
                            BiometricPrompt.ERROR_USER_CANCELED,
                            BiometricPrompt.ERROR_NEGATIVE_BUTTON -> ERROR_USER_CANCEL
                            BiometricPrompt.ERROR_LOCKOUT,
                            BiometricPrompt.ERROR_LOCKOUT_PERMANENT -> ERROR_LOCKOUT
                            BiometricPrompt.ERROR_NO_BIOMETRICS -> ERROR_NOT_ENROLLED
                            BiometricPrompt.ERROR_HW_NOT_PRESENT,
                            BiometricPrompt.ERROR_HW_UNAVAILABLE -> ERROR_NOT_AVAILABLE
                            else -> ERROR_UNKNOWN
                        }
                        promise.reject(code, errString.toString())
                    }

                    override fun onAuthenticationFailed() {
                        // 인증 실패 (지문 불일치 등) — 아직 재시도 가능
                        // Promise를 reject하지 않음 (시스템이 재시도 허용)
                    }
                }

                val biometricPrompt = BiometricPrompt(activity, executor, callback)

                val promptInfo = BiometricPrompt.PromptInfo.Builder()
                    .setTitle(title)
                    .setSubtitle(subtitle)
                    .setDescription(description)
                    .setNegativeButtonText("취소")
                    .setAllowedAuthenticators(
                        BiometricManager.Authenticators.BIOMETRIC_STRONG or
                        BiometricManager.Authenticators.BIOMETRIC_WEAK
                    )
                    .build()

                biometricPrompt.authenticate(promptInfo)
            } catch (e: Exception) {
                promise.reject(ERROR_UNKNOWN, e.message, e)
            }
        }
    }

    override fun encryptWithBiometric(key: String, data: String, promise: Promise) {
        // 실제 구현: Android Keystore + BiometricPrompt.CryptoObject 사용
        // 여기서는 간단한 예시
        try {
            // TODO: Android Keystore로 암호화 구현
            promise.resolve("encrypted_$data")
        } catch (e: Exception) {
            promise.reject(ERROR_UNKNOWN, "암호화 실패: ${e.message}", e)
        }
    }

    override fun decryptWithBiometric(
        key: String,
        encryptedData: String,
        promise: Promise
    ) {
        try {
            // TODO: Android Keystore로 복호화 구현
            promise.resolve(encryptedData.removePrefix("encrypted_"))
        } catch (e: Exception) {
            promise.reject(ERROR_UNKNOWN, "복호화 실패: ${e.message}", e)
        }
    }

    override fun hasNewBiometricEnrolled(promise: Promise) {
        // Android 키스토어의 setInvalidatedByBiometricEnrollment 활용
        // 새 생체 등록 시 키가 무효화되는지 확인
        try {
            // TODO: 실제 키스토어 검증 구현
            promise.resolve(false)
        } catch (e: Exception) {
            promise.reject(ERROR_UNKNOWN, e.message, e)
        }
    }
}
```

### 패키지 등록

```kotlin
// android/app/src/main/java/com/myapp/BiometricAuthPackage.kt

package com.myapp

import com.facebook.react.TurboReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider

class BiometricAuthPackage : TurboReactPackage() {

    override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? {
        return when (name) {
            BiometricAuthModule.NAME -> BiometricAuthModule(reactContext)
            else -> null
        }
    }

    override fun getReactModuleInfoProvider(): ReactModuleInfoProvider {
        return ReactModuleInfoProvider {
            mapOf(
                BiometricAuthModule.NAME to ReactModuleInfo(
                    BiometricAuthModule.NAME,
                    BiometricAuthModule.NAME,
                    false, false, false, true
                )
            )
        }
    }
}
```

### JavaScript 사용

```tsx
// src/screens/BiometricScreen.tsx

import React, { useState } from 'react';
import { View, Text, Pressable, Alert, StyleSheet } from 'react-native';
import NativeBiometricAuth from '../specs/NativeBiometricAuth';

export default function BiometricScreen() {
  const [authResult, setAuthResult] = useState<string>('인증 전');

  // 동기 메서드 — 즉시 값 반환
  const isAvailable = NativeBiometricAuth.isBiometricAvailable();
  const biometricType = NativeBiometricAuth.getSupportedBiometricType();

  const handleAuthenticate = async () => {
    try {
      setAuthResult('인증 중...');
      const success = await NativeBiometricAuth.authenticate(
        '본인 인증',
        '생체 정보로 확인합니다',
        '결제를 위해 본인 인증이 필요합니다'
      );

      if (success) {
        setAuthResult('인증 성공!');
      }
    } catch (error: any) {
      switch (error.code) {
        case 'USER_CANCEL':
          setAuthResult('사용자가 취소했습니다');
          break;
        case 'BIOMETRIC_LOCKOUT':
          setAuthResult('너무 많이 시도했습니다. 잠시 후 다시 시도하세요.');
          break;
        case 'BIOMETRIC_NOT_ENROLLED':
          Alert.alert('생체 정보 없음', '설정에서 생체 인증을 등록해주세요.');
          setAuthResult('생체 정보 미등록');
          break;
        default:
          setAuthResult(`인증 실패: ${error.message}`);
      }
    }
  };

  return (
    <View style={styles.container}>
      <Text style={styles.header}>생체 인증</Text>

      <View style={styles.infoSection}>
        <Text style={styles.infoText}>
          지원 여부: {isAvailable ? '지원됨' : '미지원'}
        </Text>
        <Text style={styles.infoText}>
          인증 타입: {biometricType === 'fingerprint' ? '지문'
            : biometricType === 'face' ? '얼굴'
            : biometricType === 'iris' ? '홍채' : '없음'}
        </Text>
        <Text style={styles.infoText}>
          상태: {authResult}
        </Text>
      </View>

      <Pressable
        style={[styles.authButton, !isAvailable && styles.disabledButton]}
        onPress={handleAuthenticate}
        disabled={!isAvailable}
      >
        <Text style={styles.authButtonText}>
          {isAvailable ? '생체 인증 실행' : '생체 인증 미지원'}
        </Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, justifyContent: 'center' },
  header: { fontSize: 28, fontWeight: 'bold', textAlign: 'center', marginBottom: 32 },
  infoSection: { backgroundColor: '#f5f5f5', padding: 20, borderRadius: 12,
    marginBottom: 32 },
  infoText: { fontSize: 16, marginBottom: 8, color: '#333' },
  authButton: { backgroundColor: '#4A90D9', paddingVertical: 16, borderRadius: 12,
    alignItems: 'center' },
  disabledButton: { backgroundColor: '#ccc' },
  authButtonText: { color: 'white', fontSize: 18, fontWeight: '600' },
});
```

---

## 5. 예제 3: Native Toast/Snackbar 모듈

Android 네이티브 Toast와 Snackbar를 보여주고, 이벤트를 JS로 전달하는 모듈이다.

### Spec 정의

```typescript
// src/specs/NativeToast.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // Toast 표시 (fire-and-forget — 반환값 없음)
  showToast(message: string, duration: string): void;
  // duration: 'short' | 'long'

  // Snackbar 표시 (액션 버튼 포함)
  showSnackbar(
    message: string,
    actionLabel: string,
    duration: string
  ): void;
  // duration: 'short' | 'long' | 'indefinite'

  // Snackbar 숨기기
  dismissSnackbar(): void;

  // 이벤트 지원 (Native → JS)
  addListener(eventName: string): void;
  removeListeners(count: number): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('Toast');
```

### Kotlin 구현

```kotlin
// android/app/src/main/java/com/myapp/ToastModule.kt

package com.myapp

import android.view.View
import android.widget.Toast
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.UiThreadUtil
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.modules.core.DeviceEventManagerModule
import com.google.android.material.snackbar.Snackbar
import com.myapp.specs.NativeToastSpec

@ReactModule(name = ToastModule.NAME)
class ToastModule(
    private val reactContext: ReactApplicationContext
) : NativeToastSpec(reactContext) {

    companion object {
        const val NAME = "Toast"

        // JS로 보내는 이벤트 이름 상수
        const val EVENT_SNACKBAR_ACTION = "onSnackbarAction"
        const val EVENT_SNACKBAR_DISMISSED = "onSnackbarDismissed"
    }

    override fun getName(): String = NAME

    private var currentSnackbar: Snackbar? = null
    private var listenerCount = 0

    override fun showToast(message: String, duration: String) {
        val toastDuration = when (duration) {
            "long" -> Toast.LENGTH_LONG
            else -> Toast.LENGTH_SHORT
        }

        UiThreadUtil.runOnUiThread {
            Toast.makeText(reactContext, message, toastDuration).show()
        }
    }

    override fun showSnackbar(
        message: String,
        actionLabel: String,
        duration: String
    ) {
        UiThreadUtil.runOnUiThread {
            val activity = reactContext.currentActivity ?: return@runOnUiThread
            val rootView = activity.findViewById<View>(android.R.id.content)
                ?: return@runOnUiThread

            val snackbarDuration = when (duration) {
                "long" -> Snackbar.LENGTH_LONG
                "indefinite" -> Snackbar.LENGTH_INDEFINITE
                else -> Snackbar.LENGTH_SHORT
            }

            currentSnackbar = Snackbar.make(rootView, message, snackbarDuration).apply {
                if (actionLabel.isNotEmpty()) {
                    setAction(actionLabel) {
                        // 액션 버튼 클릭 → JS로 이벤트 발송
                        sendEvent(EVENT_SNACKBAR_ACTION, Arguments.createMap().apply {
                            putString("action", actionLabel)
                        })
                    }
                }

                addCallback(object : Snackbar.Callback() {
                    override fun onDismissed(transientBottomBar: Snackbar?, event: Int) {
                        val reason = when (event) {
                            DISMISS_EVENT_ACTION -> "action"
                            DISMISS_EVENT_TIMEOUT -> "timeout"
                            DISMISS_EVENT_MANUAL -> "manual"
                            DISMISS_EVENT_SWIPE -> "swipe"
                            else -> "unknown"
                        }
                        sendEvent(EVENT_SNACKBAR_DISMISSED, Arguments.createMap().apply {
                            putString("reason", reason)
                        })
                    }
                })

                show()
            }
        }
    }

    override fun dismissSnackbar() {
        UiThreadUtil.runOnUiThread {
            currentSnackbar?.dismiss()
            currentSnackbar = null
        }
    }

    // ═══ 이벤트 지원 메서드 ═══

    override fun addListener(eventName: String) {
        listenerCount++
    }

    override fun removeListeners(count: Double) {
        listenerCount -= count.toInt()
        if (listenerCount < 0) listenerCount = 0
    }

    private fun sendEvent(eventName: String, params: Any?) {
        if (listenerCount > 0) {
            reactContext
                .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter::class.java)
                .emit(eventName, params)
        }
    }
}
```

### 패키지 등록

```kotlin
// android/app/src/main/java/com/myapp/ToastPackage.kt

package com.myapp

import com.facebook.react.TurboReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.model.ReactModuleInfo
import com.facebook.react.module.model.ReactModuleInfoProvider

class ToastPackage : TurboReactPackage() {

    override fun getModule(name: String, reactContext: ReactApplicationContext): NativeModule? {
        return when (name) {
            ToastModule.NAME -> ToastModule(reactContext)
            else -> null
        }
    }

    override fun getReactModuleInfoProvider(): ReactModuleInfoProvider {
        return ReactModuleInfoProvider {
            mapOf(
                ToastModule.NAME to ReactModuleInfo(
                    ToastModule.NAME,
                    ToastModule.NAME,
                    false, false, false, true
                )
            )
        }
    }
}
```

### JavaScript 사용

```tsx
// src/hooks/useNativeToast.ts

import { useEffect, useRef, useCallback } from 'react';
import { NativeEventEmitter } from 'react-native';
import NativeToast from '../specs/NativeToast';

type SnackbarDismissReason = 'action' | 'timeout' | 'manual' | 'swipe' | 'unknown';

interface UseNativeToastOptions {
  onSnackbarAction?: (action: string) => void;
  onSnackbarDismissed?: (reason: SnackbarDismissReason) => void;
}

export function useNativeToast(options?: UseNativeToastOptions) {
  const emitterRef = useRef<NativeEventEmitter | null>(null);

  useEffect(() => {
    const emitter = new NativeEventEmitter(NativeToast);
    emitterRef.current = emitter;

    const subscriptions = [
      emitter.addListener('onSnackbarAction', (event) => {
        options?.onSnackbarAction?.(event.action);
      }),
      emitter.addListener('onSnackbarDismissed', (event) => {
        options?.onSnackbarDismissed?.(event.reason);
      }),
    ];

    return () => {
      subscriptions.forEach(sub => sub.remove());
    };
  }, [options?.onSnackbarAction, options?.onSnackbarDismissed]);

  const showToast = useCallback((message: string, duration: 'short' | 'long' = 'short') => {
    NativeToast.showToast(message, duration);
  }, []);

  const showSnackbar = useCallback((
    message: string,
    actionLabel: string = '',
    duration: 'short' | 'long' | 'indefinite' = 'short'
  ) => {
    NativeToast.showSnackbar(message, actionLabel, duration);
  }, []);

  const dismissSnackbar = useCallback(() => {
    NativeToast.dismissSnackbar();
  }, []);

  return { showToast, showSnackbar, dismissSnackbar };
}
```

```tsx
// src/screens/ToastDemoScreen.tsx

import React from 'react';
import { View, Text, Pressable, StyleSheet, Alert } from 'react-native';
import { useNativeToast } from '../hooks/useNativeToast';

export default function ToastDemoScreen() {
  const { showToast, showSnackbar, dismissSnackbar } = useNativeToast({
    onSnackbarAction: (action) => {
      Alert.alert('Snackbar 액션', `"${action}" 버튼이 클릭되었습니다`);
    },
    onSnackbarDismissed: (reason) => {
      console.log('Snackbar 닫힘:', reason);
    },
  });

  return (
    <View style={styles.container}>
      <Text style={styles.header}>Toast & Snackbar</Text>

      <Pressable
        style={styles.button}
        onPress={() => showToast('안녕하세요! (짧은 Toast)', 'short')}
      >
        <Text style={styles.buttonText}>짧은 Toast</Text>
      </Pressable>

      <Pressable
        style={styles.button}
        onPress={() => showToast('긴 Toast 메시지입니다!', 'long')}
      >
        <Text style={styles.buttonText}>긴 Toast</Text>
      </Pressable>

      <Pressable
        style={[styles.button, { backgroundColor: '#FF9800' }]}
        onPress={() => showSnackbar(
          '항목이 삭제되었습니다',
          '실행 취소',
          'long'
        )}
      >
        <Text style={styles.buttonText}>Snackbar (액션 포함)</Text>
      </Pressable>

      <Pressable
        style={[styles.button, { backgroundColor: '#F44336' }]}
        onPress={() => showSnackbar(
          '네트워크 연결이 끊어졌습니다',
          '재시도',
          'indefinite'
        )}
      >
        <Text style={styles.buttonText}>무기한 Snackbar</Text>
      </Pressable>

      <Pressable
        style={[styles.button, { backgroundColor: '#9E9E9E' }]}
        onPress={dismissSnackbar}
      >
        <Text style={styles.buttonText}>Snackbar 닫기</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24, justifyContent: 'center' },
  header: { fontSize: 24, fontWeight: 'bold', textAlign: 'center', marginBottom: 32 },
  button: { backgroundColor: '#4A90D9', padding: 16, borderRadius: 12,
    marginBottom: 12, alignItems: 'center' },
  buttonText: { color: 'white', fontSize: 16, fontWeight: '600' },
});
```

---

## 6. 에러 처리 패턴

### 6.1 Promise 에러 코드 체계

```kotlin
// 에러 코드를 상수로 정의하여 JS와 공유
companion object {
    // 일반 에러
    const val ERROR_UNKNOWN = "E_UNKNOWN"
    const val ERROR_INVALID_PARAM = "E_INVALID_PARAM"

    // 권한 에러
    const val ERROR_PERMISSION_DENIED = "E_PERMISSION_DENIED"
    const val ERROR_PERMISSION_NOT_ASKED = "E_PERMISSION_NOT_ASKED"

    // 하드웨어 에러
    const val ERROR_NOT_AVAILABLE = "E_NOT_AVAILABLE"
    const val ERROR_HARDWARE_BUSY = "E_HARDWARE_BUSY"

    // 네트워크 에러
    const val ERROR_NETWORK = "E_NETWORK"
    const val ERROR_TIMEOUT = "E_TIMEOUT"
}

// Promise reject 시 일관된 형식
override fun riskyOperation(promise: Promise) {
    try {
        // 작업 수행
        val result = performOperation()
        promise.resolve(result)
    } catch (e: SecurityException) {
        promise.reject(ERROR_PERMISSION_DENIED, "권한이 거부되었습니다: ${e.message}", e)
    } catch (e: IllegalArgumentException) {
        promise.reject(ERROR_INVALID_PARAM, "잘못된 파라미터: ${e.message}", e)
    } catch (e: java.net.SocketTimeoutException) {
        promise.reject(ERROR_TIMEOUT, "요청 시간 초과", e)
    } catch (e: Exception) {
        promise.reject(ERROR_UNKNOWN, "알 수 없는 오류: ${e.message}", e)
    }
}
```

### 6.2 JavaScript 에러 처리

```typescript
// src/utils/nativeErrorHandler.ts

interface NativeError {
  code: string;
  message: string;
  nativeStackAndroid?: string;
}

function isNativeError(error: unknown): error is NativeError {
  return (
    typeof error === 'object' &&
    error !== null &&
    'code' in error &&
    'message' in error
  );
}

export async function safeNativeCall<T>(
  operation: () => Promise<T>,
  errorHandlers?: Partial<Record<string, (error: NativeError) => void>>
): Promise<T | null> {
  try {
    return await operation();
  } catch (error) {
    if (isNativeError(error)) {
      const handler = errorHandlers?.[error.code];
      if (handler) {
        handler(error);
      } else {
        console.error(`Native error [${error.code}]: ${error.message}`);
      }
    } else {
      console.error('Unexpected error:', error);
    }
    return null;
  }
}

// 사용 예시:
const result = await safeNativeCall(
  () => NativeBiometricAuth.authenticate('제목', '부제', '설명'),
  {
    'USER_CANCEL': () => console.log('사용자 취소'),
    'BIOMETRIC_LOCKOUT': (e) => Alert.alert('잠금', e.message),
    'BIOMETRIC_NOT_AVAILABLE': () => Alert.alert('미지원', '이 기기는 생체 인증을 지원하지 않습니다'),
  }
);
```

---

## 7. 테스트와 디버깅

### 7.1 Android Studio에서 TurboModule 디버깅

```
┌────────────────────────────────────────────────────────────────┐
│           TurboModule 디버깅 가이드                              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Android Studio에서 프로젝트 열기:                            │
│     android/ 폴더를 Android Studio로 열기                       │
│                                                                 │
│  2. 브레이크포인트 설정:                                         │
│     Kotlin 파일에서 원하는 라인에 브레이크포인트                  │
│                                                                 │
│  3. 디버거 붙이기:                                               │
│     Run → Attach Debugger to Android Process                    │
│     → 앱 프로세스 선택                                           │
│                                                                 │
│  4. JS에서 네이티브 메서드 호출                                   │
│     → 브레이크포인트에서 멈춤                                     │
│     → 변수 검사, 스텝 실행 가능                                   │
│                                                                 │
│  주의: Release 빌드에서는 디버거가 붙지 않음                      │
│        Debug 빌드에서만 가능                                     │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 7.2 로그 확인

```kotlin
// Kotlin에서 로그 남기기
import android.util.Log

class MyModule(ctx: ReactApplicationContext) : NativeMyModuleSpec(ctx) {
    private val TAG = "MyModule"

    override fun someMethod(param: String, promise: Promise) {
        Log.d(TAG, "someMethod 호출됨: param=$param")

        try {
            val result = doWork(param)
            Log.d(TAG, "someMethod 성공: result=$result")
            promise.resolve(result)
        } catch (e: Exception) {
            Log.e(TAG, "someMethod 실패", e)
            promise.reject("ERROR", e.message, e)
        }
    }
}
```

```bash
# Logcat으로 로그 확인
adb logcat -s MyModule:D

# React Native 콘솔 로그도 함께 보기
adb logcat -s ReactNative:D MyModule:D
```

### 7.3 JavaScript에서 모의(Mock) 테스트

```typescript
// __mocks__/specs/NativeDeviceInfo.ts
// Jest에서 자동으로 사용됨

const NativeDeviceInfo = {
  getDeviceModel: jest.fn(() => 'MockDevice'),
  getAndroidVersion: jest.fn(() => '14'),
  getApiLevel: jest.fn(() => 34),
  getTotalMemory: jest.fn(() => 8589934592), // 8GB
  getAvailableMemory: jest.fn(() => 4294967296), // 4GB
  isTablet: jest.fn(() => false),
  getLanguage: jest.fn(() => 'ko-KR'),
  getTimezone: jest.fn(() => 'Asia/Seoul'),
  getConstants: jest.fn(() => ({
    BRAND: 'MockBrand',
    MANUFACTURER: 'MockManufacturer',
    DEVICE_NAME: 'mock_device',
    BUILD_NUMBER: '100',
    APP_VERSION: '1.0.0',
  })),
};

export default NativeDeviceInfo;
```

```typescript
// __tests__/DeviceInfoScreen.test.tsx

import React from 'react';
import { render, screen } from '@testing-library/react-native';
import DeviceInfoScreen from '../src/screens/DeviceInfoScreen';

jest.mock('../src/specs/NativeDeviceInfo');

describe('DeviceInfoScreen', () => {
  it('디바이스 정보를 올바르게 표시한다', () => {
    render(<DeviceInfoScreen />);

    expect(screen.getByText('MockDevice')).toBeTruthy();
    expect(screen.getByText('14')).toBeTruthy();
    expect(screen.getByText('ko-KR')).toBeTruthy();
    expect(screen.getByText('Asia/Seoul')).toBeTruthy();
  });
});
```

---

## 요약

```
┌────────────────────────────────────────────────────────────────┐
│           TurboModule 생성 핵심 체크리스트                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [ ] 1. TypeScript Spec (NativeXxx.ts)                         │
│      - TurboModule 상속                                        │
│      - 모든 메서드 타입 정의                                     │
│      - TurboModuleRegistry.getEnforcing 내보내기                │
│                                                                 │
│  [ ] 2. package.json codegenConfig                              │
│      - name, type, jsSrcsDir, javaPackageName                  │
│                                                                 │
│  [ ] 3. Kotlin Module (XxxModule.kt)                           │
│      - NativeXxxSpec 상속                                       │
│      - getName() = Spec의 Registry 이름과 동일                  │
│      - 모든 Spec 메서드 override                                │
│                                                                 │
│  [ ] 4. Kotlin Package (XxxPackage.kt)                         │
│      - TurboReactPackage 상속                                   │
│      - getModule에서 인스턴스 생성                               │
│      - isTurboModule = true                                     │
│                                                                 │
│  [ ] 5. MainApplication에 패키지 등록                           │
│      - getPackages()에 add(XxxPackage())                       │
│                                                                 │
│  [ ] 6. build.gradle 의존성 추가 (필요시)                       │
│      - implementation("...biometric...")                        │
│      - implementation("...material...")                         │
│                                                                 │
│  [ ] 7. AndroidManifest.xml 권한 추가 (필요시)                  │
│      - <uses-permission android:name="..." />                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
