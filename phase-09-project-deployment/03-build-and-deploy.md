# 빌드와 배포 — Android 앱 서명부터 Play Store 배포까지

## 목차
1. [Debug vs Release 빌드](#1-debug-vs-release-빌드)
2. [Android 앱 서명](#2-android-앱-서명)
3. [릴리스 빌드 생성](#3-릴리스-빌드-생성)
4. [ProGuard/R8 설정](#4-proguardr8-설정)
5. [앱 크기 최적화](#5-앱-크기-최적화)
6. [Play Store 배포](#6-play-store-배포)
7. [OTA (Over-the-Air) 업데이트](#7-ota-over-the-air-업데이트)
8. [CI/CD 파이프라인](#8-cicd-파이프라인)
9. [버저닝 전략](#9-버저닝-전략)

---

## 1. Debug vs Release 빌드

```
[Debug vs Release 빌드 차이점]

항목                  Debug                    Release
──────────────────────────────────────────────────────────
Metro 번들러          연결됨 (Hot Reload)       내장 (오프라인)
Dev Menu              활성화                    비활성화
Hermes 바이트코드     인터프리트                 사전 컴파일 (AOT)
소스맵                포함                      제외 (선택적)
ProGuard/R8           비활성화                  활성화
앱 서명               debug.keystore            release.keystore
성능                  느림 (디버그 오버헤드)     빠름 (최적화됨)
로깅                  console.log 동작          console.log 제거 가능
__DEV__               true                     false
```

```typescript
// 코드에서 빌드 타입 확인
if (__DEV__) {
  // 디버그 빌드에서만 실행
  console.log('디버그 모드');
} else {
  // 릴리스 빌드에서만 실행
  // Sentry 초기화, 애널리틱스 등
}
```

---

## 2. Android 앱 서명

React Native의 Android 앱 서명 과정은 네이티브 Android와 완전히 동일하다.

### 키스토어 생성

```bash
# 키스토어 생성 (네이티브 Android와 동일한 keytool 명령)
keytool -genkeypair -v \
  -storetype PKCS12 \
  -keystore my-release-key.keystore \
  -alias my-key-alias \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000

# 프롬프트:
# Enter keystore password: ********
# Re-enter new password: ********
# What is your first and last name? [홍길동]
# What is your name of your organizational unit? [개발팀]
# What is the name of your organization? [MyCompany]
# What is the name of your City or Locality? [서울]
# What is the name of your State or Province? [서울특별시]
# What is the two-letter country code for this unit? [KR]
# Is CN=홍길동, OU=개발팀, O=MyCompany, L=서울, ST=서울특별시, C=KR correct? [yes]

# 키스토어를 android/app 디렉토리로 이동
mv my-release-key.keystore android/app/
```

### Gradle 서명 설정

```groovy
// android/gradle.properties — 키스토어 정보 (절대 Git에 커밋하지 말 것!)
MYAPP_RELEASE_STORE_FILE=my-release-key.keystore
MYAPP_RELEASE_KEY_ALIAS=my-key-alias
MYAPP_RELEASE_STORE_PASSWORD=your-store-password
MYAPP_RELEASE_KEY_PASSWORD=your-key-password
```

```groovy
// android/app/build.gradle
android {
    ...
    defaultConfig {
        applicationId "com.myapp"
        minSdkVersion rootProject.ext.minSdkVersion
        targetSdkVersion rootProject.ext.targetSdkVersion
        versionCode 1
        versionName "1.0.0"
    }

    signingConfigs {
        debug {
            // debug.keystore는 자동 생성됨
            storeFile file('debug.keystore')
            storePassword 'android'
            keyAlias 'androiddebugkey'
            keyPassword 'android'
        }
        release {
            if (project.hasProperty('MYAPP_RELEASE_STORE_FILE')) {
                storeFile file(MYAPP_RELEASE_STORE_FILE)
                storePassword MYAPP_RELEASE_STORE_PASSWORD
                keyAlias MYAPP_RELEASE_KEY_ALIAS
                keyPassword MYAPP_RELEASE_KEY_PASSWORD
            }
        }
    }

    buildTypes {
        debug {
            signingConfig signingConfigs.debug
        }
        release {
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds  // ProGuard/R8
            shrinkResources false  // RN에서는 false 권장
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
        }
    }
}
```

### 키스토어 보안 관리

```bash
# .gitignore에 추가
# android/app/my-release-key.keystore
# android/gradle.properties (민감 정보 포함)

# CI/CD에서는 환경 변수로 관리
# GitHub Secrets:
# KEYSTORE_BASE64 — base64로 인코딩된 keystore 파일
# KEYSTORE_PASSWORD
# KEY_ALIAS
# KEY_PASSWORD

# 키스토어를 base64로 인코딩 (CI에서 복원용)
base64 -i android/app/my-release-key.keystore -o keystore-base64.txt
```

---

## 3. 릴리스 빌드 생성

### APK 빌드

```bash
# APK 빌드 (테스트/사이드로딩용)
cd android && ./gradlew assembleRelease

# 출력 위치:
# android/app/build/outputs/apk/release/app-release.apk

# 빌드 확인 (파일 크기, 서명 확인)
ls -lh android/app/build/outputs/apk/release/app-release.apk
# 서명 확인
jarsigner -verify android/app/build/outputs/apk/release/app-release.apk
```

### AAB 빌드 (Play Store 권장)

```bash
# AAB (Android App Bundle) 빌드 — Play Store에 올릴 때 사용
cd android && ./gradlew bundleRelease

# 출력 위치:
# android/app/build/outputs/bundle/release/app-release.aab

# AAB 파일 크기 확인
ls -lh android/app/build/outputs/bundle/release/app-release.aab
```

> **AAB vs APK**: Play Store는 2021년 8월부터 AAB를 필수로 요구한다.
> AAB는 기기별로 최적화된 APK를 생성하여 다운로드 크기를 줄인다.

### 릴리스 빌드 테스트

```bash
# 릴리스 빌드를 에뮬레이터/기기에 직접 설치하여 테스트
cd android && ./gradlew assembleRelease

# APK 직접 설치
adb install android/app/build/outputs/apk/release/app-release.apk

# 또는 react-native CLI로 릴리스 모드 실행
npx react-native run-android --mode release

# 릴리스 빌드 테스트 체크리스트:
# 1. 앱이 Metro 없이 오프라인으로 동작하는가?
# 2. 모든 화면이 정상 표시되는가?
# 3. API 호출이 정상 동작하는가?
# 4. 이미지/폰트가 정상 로드되는가?
# 5. 네비게이션이 정상 동작하는가?
# 6. 딥 링크가 동작하는가?
# 7. 푸시 알림이 동작하는가?
# 8. 성능이 디버그보다 향상되었는가?
```

```exercise
type: code-arrange
question: "Android Release 빌드 명령어를 조립하세요"
tokens:
  - "cd android"
  - "&&"
  - "./gradlew"
  - "bundleRelease"
distractors:
  - "assembleDebug"
  - "npm run build"
  - "expo build"
answer: ["cd android", "&&", "./gradlew", "bundleRelease"]
hint: "Android 빌드는 android 디렉토리에서 gradlew를 사용합니다"
xp: 8
```

---

## 4. ProGuard/R8 설정

React Native 앱에서 ProGuard/R8은 네이티브 Java/Kotlin 코드에만 적용된다. JavaScript 코드는 Hermes 바이트코드로 별도 처리된다.

```groovy
// android/app/build.gradle
def enableProguardInReleaseBuilds = true  // true로 설정

android {
    buildTypes {
        release {
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"),
                         "proguard-rules.pro"
        }
    }
}
```

```proguard
# android/app/proguard-rules.pro

# React Native 기본 규칙
-keep class com.facebook.hermes.unicode.** { *; }
-keep class com.facebook.jni.** { *; }

# React Native 모듈 보존
-keep,allowobfuscation @interface com.facebook.proguard.annotations.DoNotStrip
-keep,allowobfuscation @interface com.facebook.proguard.annotations.KeepGettersAndSetters
-keep @com.facebook.proguard.annotations.DoNotStrip class *
-keepclassmembers class * {
    @com.facebook.proguard.annotations.DoNotStrip *;
}

# Hermes 엔진
-keep class com.facebook.hermes.** { *; }
-dontwarn com.facebook.hermes.**

# react-native-reanimated
-keep class com.swmansion.reanimated.** { *; }

# react-native-gesture-handler
-keep class com.swmansion.gesturehandler.** { *; }

# OkHttp (네트워크 라이브러리가 사용하는 경우)
-dontwarn okhttp3.**
-dontwarn okio.**
-keep class okhttp3.** { *; }

# Gson (JSON 파싱 라이브러리가 사용하는 경우)
-keep class com.google.gson.** { *; }
-keepattributes Signature
-keepattributes *Annotation*

# 커스텀 네이티브 모듈
-keep class com.myapp.** { *; }
```

---

## 5. 앱 크기 최적화

### Hermes 바이트코드 사전 컴파일

```groovy
// android/app/build.gradle
// React Native 0.84에서는 Hermes가 기본으로 활성화됨

project.ext.react = [
    enableHermes: true,  // 기본값: true
]

// Hermes의 장점:
// 1. 앱 시작 시간 단축 (바이트코드 사전 컴파일)
// 2. 메모리 사용량 감소
// 3. 앱 크기 최적화
```

### ABI 분리 빌드

```groovy
// android/app/build.gradle
android {
    splits {
        abi {
            reset()
            enable true  // ABI별 분리 빌드 활성화
            universalApk false  // 통합 APK 생성 안 함
            include "armeabi-v7a", "arm64-v8a", "x86", "x86_64"
        }
    }
}

// 결과: ABI별로 별도 APK 생성
// app-armeabi-v7a-release.apk (32bit ARM — 구형 기기)
// app-arm64-v8a-release.apk   (64bit ARM — 대부분의 현대 기기)
// app-x86-release.apk         (x86 에뮬레이터)
// app-x86_64-release.apk      (x86_64 에뮬레이터)

// AAB를 사용하면 이 설정이 필요 없음 (Play Store가 자동으로 분리)
```

### 이미지 최적화

```bash
# PNG 최적화
# npx react-native-image-optimizer --quality 80 --output optimized/

# WebP 사용 (30-40% 크기 감소)
# cwebp input.png -q 80 -o output.webp
```

```typescript
// 이미지를 적절한 크기로 요청 (CDN 사용 시)
import { PixelRatio } from 'react-native';

function getOptimizedImageUrl(baseUrl: string, width: number): string {
  const pixelWidth = PixelRatio.getPixelSizeForLayoutSize(width);
  return `${baseUrl}?w=${pixelWidth}&q=80&fm=webp`;
}

// 사용
<Image
  source={{ uri: getOptimizedImageUrl('https://cdn.example.com/image.jpg', 200) }}
  style={{ width: 200, height: 200 }}
/>
```

### 미사용 코드 제거

```typescript
// 미사용 import 자동 제거: ESLint 설정
// .eslintrc.js
module.exports = {
  rules: {
    'no-unused-vars': 'error',
    '@typescript-eslint/no-unused-imports': 'error',
  },
};

// 번들 크기 분석
// npx react-native-bundle-visualizer
// → 어떤 라이브러리가 번들 크기를 많이 차지하는지 시각화
```

---

## 6. Play Store 배포

### 내부 테스트 트랙

```
[Play Console 배포 흐름]

1. Google Play Console 접속 (https://play.google.com/console)
2. 앱 만들기 (최초 1회)
   - 앱 이름, 기본 언어, 앱 유형 설정
3. 스토어 등록정보 작성
   - 제목, 설명, 스크린샷, 기능 그래픽
   - 개인정보 처리방침 URL
4. 앱 콘텐츠 설정
   - 콘텐츠 등급 설문
   - 대상 연령
   - 광고 포함 여부

[배포 트랙]
Internal testing → Closed testing → Open testing → Production

5. 내부 테스트 (Internal Testing)
   - 최대 100명의 테스터
   - 이메일로 테스터 초대
   - 리뷰 없이 즉시 배포 (몇 분 내)

6. 비공개 테스트 (Closed Testing)
   - 이메일 목록 또는 Google 그룹으로 테스터 관리
   - 리뷰 없이 배포

7. 공개 테스트 (Open Testing)
   - 누구나 참여 가능
   - Play Store에 "앞서 해보기" 표시

8. 프로덕션 (Production)
   - 모든 사용자에게 공개
   - 리뷰 필요 (최초 리뷰는 며칠 소요)
```

### AAB 업로드

```bash
# 1. AAB 빌드
cd android && ./gradlew bundleRelease

# 2. 빌드 출력 확인
ls -lh android/app/build/outputs/bundle/release/app-release.aab

# 3. Play Console에 AAB 업로드
#    - Play Console → 앱 → 테스트 → 내부 테스트
#    - "새 버전 만들기" 클릭
#    - AAB 파일 드래그 & 드롭
#    - 버전 이름, 출시 노트 작성
#    - "검토 시작" 클릭
```

---

## 7. OTA (Over-the-Air) 업데이트

OTA 업데이트는 Play Store 심사 없이 JavaScript 번들을 업데이트하는 기술이다. Android 네이티브에는 없는 React Native만의 강력한 기능이다.

### EAS Update (Expo)

```bash
# EAS CLI 설치
npm install -g eas-cli

# 프로젝트에 EAS Update 설정
npx expo install expo-updates
eas update:configure

# 업데이트 배포
eas update --branch production --message "버그 수정: 로그인 화면 크래시"
```

```json
// app.json (또는 app.config.js)
{
  "expo": {
    "updates": {
      "url": "https://u.expo.dev/your-project-id",
      "fallbackToCacheTimeout": 0
    },
    "runtimeVersion": {
      "policy": "sdkVersion"
    }
  }
}
```

```typescript
// 앱에서 업데이트 확인 및 적용
import * as Updates from 'expo-updates';

async function checkForUpdates() {
  try {
    const update = await Updates.checkForUpdateAsync();

    if (update.isAvailable) {
      // 업데이트 다운로드
      await Updates.fetchUpdateAsync();

      // 사용자에게 재시작 알림
      Alert.alert(
        '업데이트 가능',
        '새 버전이 다운로드되었습니다. 앱을 재시작하시겠습니까?',
        [
          { text: '나중에', style: 'cancel' },
          {
            text: '재시작',
            onPress: () => Updates.reloadAsync(),
          },
        ]
      );
    }
  } catch (error) {
    console.error('업데이트 확인 실패:', error);
  }
}
```

### OTA 업데이트로 변경할 수 있는 것 / 없는 것

```
[OTA로 업데이트 가능 (Play Store 심사 불필요)]
- JavaScript/TypeScript 코드 변경
- 스타일 변경 (StyleSheet, NativeWind)
- 이미지/에셋 변경 (JS 번들에 포함된 것)
- 텍스트 변경
- 비즈니스 로직 변경
- 새 화면 추가 (네이티브 모듈이 필요 없는 경우)
- API 엔드포인트 변경
- 버그 수정

[OTA로 업데이트 불가 (Play Store 재배포 필요)]
- 네이티브 모듈 추가/변경 (Kotlin/Java 코드)
- 네이티브 라이브러리 추가/업데이트
- Android 매니페스트 변경 (권한, 딥 링크 등)
- Gradle 빌드 설정 변경
- Hermes 엔진 업데이트
- React Native 버전 업그레이드
- 앱 아이콘/스플래시 스크린 변경
```

---

## 8. CI/CD 파이프라인

### GitHub Actions 워크플로우

```yaml
# .github/workflows/android-build.yml
name: Android Build & Deploy

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  # 1단계: 테스트
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: TypeScript 타입 체크
        run: npx tsc --noEmit

      - name: ESLint
        run: npx eslint src/ --ext .ts,.tsx

      - name: Unit Tests
        run: npx jest --coverage --ci

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  # 2단계: Android 빌드
  build-android:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: 'zulu'
          java-version: '17'

      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      - name: Install dependencies
        run: npm ci

      - name: Decode keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > android/app/my-release-key.keystore

      - name: Build AAB
        env:
          MYAPP_RELEASE_STORE_FILE: my-release-key.keystore
          MYAPP_RELEASE_STORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          MYAPP_RELEASE_KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          MYAPP_RELEASE_KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
        run: |
          cd android && ./gradlew bundleRelease

      - name: Upload AAB artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-release
          path: android/app/build/outputs/bundle/release/app-release.aab

  # 3단계: Play Store 배포 (태그 푸시 시에만)
  deploy-play-store:
    needs: build-android
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')

    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-release

      - name: Deploy to Internal Testing
        uses: r0adkll/upload-google-play@v1
        with:
          serviceAccountJsonPlainText: ${{ secrets.PLAY_STORE_SERVICE_ACCOUNT }}
          packageName: com.myapp
          releaseFiles: app-release.aab
          track: internal
          whatsNewDirectory: whatsnew/
```

### EAS Build (클라우드 빌드)

```bash
# EAS Build 설정
eas build:configure

# 빌드 실행 (클라우드에서 빌드)
eas build --platform android --profile production

# 빌드 후 자동 제출
eas build --platform android --profile production --auto-submit
```

```json
// eas.json
{
  "cli": {
    "version": ">= 12.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "android": {
        "buildType": "app-bundle"
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./google-services-key.json",
        "track": "internal"
      }
    }
  }
}
```

### Fastlane 설정

```ruby
# android/fastlane/Fastfile
default_platform(:android)

platform :android do
  desc "내부 테스트 트랙에 배포"
  lane :internal do
    # 버전 코드 자동 증가
    increment_version_code(
      gradle_file_path: "app/build.gradle"
    )

    # AAB 빌드
    gradle(
      task: "bundle",
      build_type: "Release",
      properties: {
        "android.injected.signing.store.file" => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASSWORD"],
        "android.injected.signing.key.alias" => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password" => ENV["KEY_PASSWORD"],
      }
    )

    # Play Store 내부 테스트 트랙에 업로드
    upload_to_play_store(
      track: "internal",
      aab: "app/build/outputs/bundle/release/app-release.aab",
      json_key: ENV["PLAY_STORE_JSON_KEY"],
      skip_upload_metadata: true,
      skip_upload_changelogs: false,
      skip_upload_images: true,
      skip_upload_screenshots: true,
    )
  end

  desc "프로덕션 배포"
  lane :production do
    # 내부 테스트에서 프로덕션으로 승격
    upload_to_play_store(
      track: "internal",
      track_promote_to: "production",
      json_key: ENV["PLAY_STORE_JSON_KEY"],
      skip_upload_changelogs: false,
    )
  end
end
```

```bash
# Fastlane 실행
cd android
bundle exec fastlane internal  # 내부 테스트 배포
bundle exec fastlane production  # 프로덕션 승격
```

---

## 9. 버저닝 전략

### Semantic Versioning (SemVer)

```
[버전 구조]
MAJOR.MINOR.PATCH (예: 2.3.1)

MAJOR: 호환되지 않는 API 변경 (대규모 UI 변경, 마이그레이션 필요)
MINOR: 하위 호환되는 기능 추가 (새 화면, 새 기능)
PATCH: 하위 호환되는 버그 수정 (버그 수정, 텍스트 변경)

[Android 버전 코드와 매핑]
versionName: "2.3.1"     → 사용자에게 표시되는 버전 (Play Store)
versionCode: 20030010    → Play Store 내부 정렬용 (항상 증가해야 함)

versionCode 계산법:
MAJOR * 1000000 + MINOR * 10000 + PATCH * 10 + BUILD
2 * 1000000 + 3 * 10000 + 1 * 10 + 0 = 2030010
```

```groovy
// android/app/build.gradle — 자동 버전 관리
def getVersionCode = { ->
    // package.json에서 버전 읽기
    def packageJson = new groovy.json.JsonSlurper().parse(
        new File("${project.rootDir}/../package.json")
    )
    def version = packageJson.version.split("\\.")
    def major = version[0].toInteger()
    def minor = version[1].toInteger()
    def patch = version[2].toInteger()
    return major * 1000000 + minor * 10000 + patch * 10
}

def getVersionName = { ->
    def packageJson = new groovy.json.JsonSlurper().parse(
        new File("${project.rootDir}/../package.json")
    )
    return packageJson.version
}

android {
    defaultConfig {
        versionCode getVersionCode()
        versionName getVersionName()
    }
}
```

```json
// package.json — 버전의 단일 소스
{
  "name": "my-app",
  "version": "2.3.1",
  "scripts": {
    "version:patch": "npm version patch",
    "version:minor": "npm version minor",
    "version:major": "npm version major"
  }
}
```

```bash
# 버전 업데이트 워크플로우
# 1. 버전 올리기 (package.json 자동 업데이트 + git tag 생성)
npm version patch  # 2.3.1 → 2.3.2
npm version minor  # 2.3.2 → 2.4.0
npm version major  # 2.4.0 → 3.0.0

# 2. 태그 푸시 (CI가 자동 빌드 & 배포)
git push origin main --tags
```

---

## 10. 배포 전 체크리스트

```
[릴리스 전 체크리스트]

빌드 설정:
[ ] Hermes 엔진 활성화 확인
[ ] ProGuard/R8 활성화 확인
[ ] 릴리스 키스토어 서명 확인
[ ] .env.production 환경 변수 확인
[ ] versionCode 증가 확인
[ ] versionName 업데이트 확인

기능 테스트:
[ ] 모든 핵심 사용자 흐름 수동 테스트
[ ] 릴리스 빌드에서 테스트 (디버그 빌드 아님)
[ ] 네트워크 끊김 상태에서 테스트
[ ] 다양한 화면 크기에서 테스트
[ ] Android 버전별 테스트 (최소 API 레벨 ~ 최신)

성능:
[ ] 앱 시작 시간 확인
[ ] 메모리 누수 없음 확인
[ ] 스크롤 성능 (60 FPS) 확인
[ ] APK/AAB 크기 확인

보안:
[ ] API 키가 하드코딩되지 않았는지 확인
[ ] console.log가 제거되었는지 확인 (__DEV__ 가드)
[ ] SSL 핀닝 설정 확인
[ ] 키스토어가 Git에 포함되지 않았는지 확인

Play Store:
[ ] 스토어 설명 업데이트
[ ] 스크린샷 업데이트 (필요 시)
[ ] 릴리스 노트 작성
[ ] 콘텐츠 등급 설문 업데이트 (새 기능 추가 시)
```
