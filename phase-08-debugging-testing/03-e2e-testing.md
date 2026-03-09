# E2E 테스트 — UI Automator/Espresso에서 Detox/Maestro로

## 목차
1. [E2E 테스트 도구 비교](#1-e2e-테스트-도구-비교)
2. [Maestro — 간단하고 강력한 E2E 테스트](#2-maestro--간단하고-강력한-e2e-테스트)
3. [Detox — 가장 강력한 React Native E2E 프레임워크](#3-detox--가장-강력한-react-native-e2e-프레임워크)
4. [테스트 전략](#4-테스트-전략)
5. [시각적 회귀 테스트](#5-시각적-회귀-테스트)
6. [접근성 테스트](#6-접근성-테스트)

---

## 1. E2E 테스트 도구 비교

Android에서는 Espresso(UI 테스트)와 UI Automator(시스템 UI 테스트)를 사용한다. React Native에서는 여러 옵션이 있다.

```
[E2E 테스트 도구 비교표]

도구        | 난이도  | 언어     | 특징                    | Android 대응
------------|---------|----------|------------------------|----------------
Maestro     | ★☆☆    | YAML     | 가장 쉬움, 빠른 시작      | UI Automator
Detox       | ★★☆    | JS/TS   | RN에 특화, 자동 동기화    | Espresso
Appium      | ★★★    | 다양     | 범용, 크로스 플랫폼       | Appium (동일)

[권장]
- 빠른 시작 & 간단한 흐름 테스트: Maestro
- 복잡한 테스트 & CI 통합: Detox
- 크로스 플랫폼 (iOS + Android + Web): Appium
```

```exercise
type: categorize
question: "다음 테스트 도구를 테스트 유형별로 분류하세요"
categories: ["단위 테스트", "E2E 테스트"]
items:
  - text: "Jest"
    category: "단위 테스트"
  - text: "Detox"
    category: "E2E 테스트"
  - text: "React Native Testing Library"
    category: "단위 테스트"
  - text: "Maestro"
    category: "E2E 테스트"
  - text: "Jest Snapshot"
    category: "단위 테스트"
  - text: "Appium"
    category: "E2E 테스트"
xp: 6
```

---

## 2. Maestro — 간단하고 강력한 E2E 테스트

Maestro는 **YAML 파일**로 E2E 테스트를 작성하는 도구다. 코드를 작성할 필요 없이 직관적인 YAML 문법으로 사용자 흐름을 정의한다.

### 2.1 설치

```bash
# macOS
curl -Ls "https://get.maestro.mobile.dev" | bash

# 설치 확인
maestro --version

# Android 에뮬레이터에서 사용하기 위한 전제 조건:
# 1. Android SDK가 설치되어 있어야 함
# 2. 에뮬레이터가 실행 중이거나 기기가 연결되어 있어야 함
# 3. adb가 PATH에 있어야 함

# 에뮬레이터 확인
adb devices
```

### 2.2 YAML 플로우 작성

```yaml
# e2e/flows/login.yaml
appId: com.myapp  # Android 패키지 이름

---
# 앱 실행
- launchApp

# 로그인 화면 확인
- assertVisible: "로그인"

# 이메일 입력
- tapOn: "이메일"
- inputText: "test@example.com"

# 비밀번호 입력
- tapOn: "비밀번호"
- inputText: "password123"

# 로그인 버튼 클릭
- tapOn: "로그인"

# 홈 화면으로 이동 확인
- assertVisible: "홈"
- assertVisible: "환영합니다"
```

> **Espresso 비교**: Espresso에서 10줄 이상의 Java/Kotlin 코드가 필요한 테스트를
> Maestro에서는 10줄의 YAML로 작성할 수 있다.

```yaml
# e2e/flows/search-and-navigate.yaml
appId: com.myapp

---
- launchApp

# 검색 기능 테스트
- tapOn: "검색"
- inputText: "React Native"
- pressKey: Enter

# 검색 결과 확인
- assertVisible: "검색 결과"
- assertVisible: "React Native 가이드"

# 결과 항목 클릭
- tapOn: "React Native 가이드"

# 상세 페이지 확인
- assertVisible: "React Native 가이드"
- scrollDown

# 뒤로 가기
- pressKey: back
- assertVisible: "검색 결과"
```

### 2.3 고급 Maestro 기능

```yaml
# e2e/flows/full-user-journey.yaml
appId: com.myapp

---
- launchApp

# 조건부 실행: 로그인 화면이면 로그인 수행
- runFlow:
    when:
      visible: "로그인"
    file: login.yaml

# 리스트 스크롤
- scrollDown:
    timeout: 3000

# 특정 요소가 보일 때까지 스크롤
- scrollUntilVisible:
    element: "마지막 항목"
    direction: DOWN
    timeout: 10000

# 스와이프
- swipe:
    direction: LEFT
    duration: 500

# 스크린샷 캡처
- takeScreenshot: "home_screen"

# 텍스트 추출 및 검증
- assertVisible:
    text: "총 \\d+개"  # 정규 표현식 지원

# 반복 실행
- repeat:
    times: 3
    commands:
      - tapOn: "좋아요"
      - assertVisible: "좋아요 완료"
      - tapOn: "취소"
```

```yaml
# e2e/flows/form-validation.yaml
appId: com.myapp

---
- launchApp
- tapOn: "회원가입"

# 빈 폼 제출 시 에러 확인
- tapOn: "가입하기"
- assertVisible: "이름을 입력해주세요"
- assertVisible: "이메일을 입력해주세요"

# 잘못된 이메일 형식
- tapOn: "이메일"
- inputText: "invalid-email"
- tapOn: "가입하기"
- assertVisible: "올바른 이메일을 입력해주세요"

# 올바른 입력
- clearText
- inputText: "valid@email.com"
- tapOn: "이름"
- inputText: "홍길동"
- tapOn: "비밀번호"
- inputText: "SecurePass123!"

# 제출
- tapOn: "가입하기"
- assertVisible: "가입이 완료되었습니다"
```

### 2.4 Maestro 실행

```bash
# 단일 플로우 실행
maestro test e2e/flows/login.yaml

# 모든 플로우 실행
maestro test e2e/flows/

# 연속 모드 (파일 변경 시 자동 재실행)
maestro test --continuous e2e/flows/login.yaml

# Maestro Studio (GUI로 테스트 작성)
maestro studio
# 브라우저에서 http://localhost:9999 열림
# 화면의 요소를 클릭하여 자동으로 YAML 명령 생성

# 특정 기기에서 실행
maestro test --device emulator-5554 e2e/flows/login.yaml
```

### 2.5 CI/CD 통합

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on: [push]

jobs:
  maestro-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      - name: Start Android Emulator
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 34
          script: |
            # Maestro 설치
            curl -Ls "https://get.maestro.mobile.dev" | bash
            export PATH="$HOME/.maestro/bin:$PATH"

            # 앱 빌드 및 설치
            npx react-native run-android --mode release

            # E2E 테스트 실행
            maestro test e2e/flows/ --format junit --output e2e-results.xml

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-results
          path: e2e-results.xml
```

---

## 3. Detox — 가장 강력한 React Native E2E 프레임워크

Detox는 Wix에서 만든 React Native 전용 E2E 테스트 프레임워크다. JavaScript/TypeScript로 테스트를 작성하며, **자동 동기화** 기능이 핵심이다.

> **Espresso 비교**: Detox의 자동 동기화는 Espresso의 **IdlingResource**와 동일한 개념이다.
> 네트워크 요청, 애니메이션, 타이머 등이 완료될 때까지 자동으로 기다린다.

### 3.1 설치 및 설정

```bash
# Detox CLI 전역 설치
npm install -g detox-cli

# 프로젝트에 Detox 설치
npm install --save-dev detox jest-circus

# Android 빌드 도구 확인
# ANDROID_SDK_ROOT 환경 변수가 설정되어 있어야 함
echo $ANDROID_SDK_ROOT
```

### 3.2 설정 파일

```javascript
// .detoxrc.js
/** @type {import('detox').DetoxConfig} */
module.exports = {
  logger: {
    level: process.env.CI ? 'debug' : 'trace',
  },

  testRunner: {
    args: {
      config: 'e2e/jest.config.js',
      maxWorkers: process.env.CI ? 2 : 1,
      _: ['e2e'],
    },
    jest: {
      setupTimeout: 120000,
    },
  },

  apps: {
    // Android 디버그 빌드
    'android.debug': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/debug/app-debug.apk',
      build: 'cd android && ./gradlew assembleDebug assembleAndroidTest -DtestBuildType=debug',
      reversePorts: [8081],
    },
    // Android 릴리스 빌드
    'android.release': {
      type: 'android.apk',
      binaryPath: 'android/app/build/outputs/apk/release/app-release.apk',
      build: 'cd android && ./gradlew assembleRelease assembleAndroidTest -DtestBuildType=release',
    },
  },

  devices: {
    // Android 에뮬레이터
    emulator: {
      type: 'android.emulator',
      device: {
        avdName: 'Pixel_7_API_34', // avdmanager list avd로 확인
      },
    },
    // 연결된 실기기
    attached: {
      type: 'android.attached',
      device: {
        adbName: '.*', // 아무 기기나
      },
    },
  },

  configurations: {
    'android.emu.debug': {
      device: 'emulator',
      app: 'android.debug',
    },
    'android.emu.release': {
      device: 'emulator',
      app: 'android.release',
    },
    'android.att.debug': {
      device: 'attached',
      app: 'android.debug',
    },
  },
};
```

```javascript
// e2e/jest.config.js
/** @type {import('@jest/types').Config.InitialOptions} */
module.exports = {
  rootDir: '..',
  testMatch: ['<rootDir>/e2e/**/*.test.ts'],
  testTimeout: 120000,
  maxWorkers: 1,
  globalSetup: 'detox/runners/jest/globalSetup',
  globalTeardown: 'detox/runners/jest/globalTeardown',
  reporters: ['detox/runners/jest/reporter'],
  testEnvironment: 'detox/runners/jest/testEnvironment',
  verbose: true,
};
```

### 3.3 Detox 테스트 작성

```typescript
// e2e/login.test.ts
import { by, device, element, expect } from 'detox';

describe('로그인 플로우', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  beforeEach(async () => {
    await device.reloadReactNative();
  });

  it('로그인 화면이 표시된다', async () => {
    // Espresso: onView(withText("로그인")).check(matches(isDisplayed()))
    await expect(element(by.text('로그인'))).toBeVisible();
    await expect(element(by.id('email-input'))).toBeVisible();
    await expect(element(by.id('password-input'))).toBeVisible();
  });

  it('이메일과 비밀번호를 입력하고 로그인한다', async () => {
    // Espresso: onView(withId(R.id.emailInput)).perform(typeText("..."))
    await element(by.id('email-input')).typeText('test@example.com');
    await element(by.id('password-input')).typeText('password123');

    // Espresso: onView(withId(R.id.loginButton)).perform(click())
    await element(by.id('login-button')).tap();

    // 홈 화면 확인 (Detox가 자동으로 네트워크 요청 완료를 기다림)
    await expect(element(by.text('환영합니다'))).toBeVisible();
  });

  it('잘못된 비밀번호로 로그인하면 에러 메시지가 표시된다', async () => {
    await element(by.id('email-input')).typeText('test@example.com');
    await element(by.id('password-input')).typeText('wrong');
    await element(by.id('login-button')).tap();

    await expect(element(by.text('로그인에 실패했습니다'))).toBeVisible();
  });
});
```

### 3.4 Detox 요소 선택자 (Matchers)

```typescript
// === by.id — testID로 찾기 (Espresso의 withId) ===
// <View testID="user-profile" />
element(by.id('user-profile'));

// === by.text — 텍스트로 찾기 (Espresso의 withText) ===
element(by.text('로그인'));

// === by.label — 접근성 라벨로 찾기 ===
// <Image accessibilityLabel="프로필 사진" />
element(by.label('프로필 사진'));

// === by.type — 네이티브 뷰 타입으로 찾기 ===
element(by.type('android.widget.ScrollView'));

// === by.traits — 접근성 특성으로 찾기 ===
element(by.traits(['button']));

// === 결합 매처 (AND 조건) ===
element(by.id('submit-button').and(by.text('제출')));

// === 자식 요소 찾기 ===
element(by.id('user-list')).atIndex(0); // 리스트의 첫 번째 항목

// === 스크롤 뷰 내 요소 찾기 ===
// Espresso의 scrollTo와 유사
await element(by.id('bottom-button')).scrollTo('bottom');
```

### 3.5 Detox 액션 (Actions)

```typescript
// === 탭 (Espresso의 click()) ===
await element(by.id('button')).tap();
await element(by.id('button')).longPress();
await element(by.id('button')).multiTap(2); // 더블 탭

// === 텍스트 입력 (Espresso의 typeText()) ===
await element(by.id('input')).typeText('Hello');
await element(by.id('input')).replaceText('New text'); // 기존 텍스트 교체
await element(by.id('input')).clearText();

// === 스크롤 (Espresso의 scrollTo()) ===
await element(by.id('scrollView')).scroll(200, 'down');
await element(by.id('scrollView')).scroll(100, 'up');
await element(by.id('scrollView')).scrollTo('bottom');
await element(by.id('scrollView')).scrollTo('top');

// === 스와이프 (Espresso의 swipeLeft/Right) ===
await element(by.id('card')).swipe('left');
await element(by.id('card')).swipe('right', 'fast');
await element(by.id('card')).swipe('up', 'slow', 0.75); // 75% 거리

// === 키보드 ===
await element(by.id('input')).tapReturnKey();
await element(by.id('input')).tapBackspaceKey();

// === 기기 제어 ===
await device.pressBack(); // Android 뒤로 가기
await device.reloadReactNative(); // JS 리로드
await device.openURL({ url: 'myapp://profile/123' }); // 딥 링크
await device.setOrientation('landscape'); // 화면 회전
await device.shake(); // 기기 흔들기
```

### 3.6 Detox 단언 (Assertions)

```typescript
// === 가시성 (Espresso의 matches(isDisplayed())) ===
await expect(element(by.id('title'))).toBeVisible();
await expect(element(by.id('hidden'))).not.toBeVisible();

// === 존재 여부 ===
await expect(element(by.id('element'))).toExist();
await expect(element(by.id('removed'))).not.toExist();

// === 텍스트 (Espresso의 matches(withText())) ===
await expect(element(by.id('label'))).toHaveText('안녕하세요');

// === ID 확인 ===
await expect(element(by.text('로그인'))).toHaveId('login-button');

// === 포커스 ===
await expect(element(by.id('input'))).toBeFocused();

// === 토글 상태 ===
await expect(element(by.id('switch'))).toHaveToggleValue(true);

// === 스크롤 위치 ===
await expect(element(by.id('scrollView'))).toHaveSliderPosition(0.5);
```

### 3.7 waitFor — 동기화 (Espresso IdlingResource 대응)

```typescript
// Detox는 기본적으로 자동 동기화를 수행하지만,
// 수동으로 기다려야 하는 경우:

// 요소가 나타날 때까지 기다림
await waitFor(element(by.text('데이터 로드 완료')))
  .toBeVisible()
  .withTimeout(10000); // 10초 타임아웃

// 요소가 사라질 때까지 기다림
await waitFor(element(by.id('loading-spinner')))
  .not.toBeVisible()
  .withTimeout(5000);

// 요소가 나타날 때까지 스크롤하며 기다림
await waitFor(element(by.text('항목 50')))
  .toBeVisible()
  .whileElement(by.id('item-list'))
  .scroll(200, 'down');
```

### 3.8 실전 Detox 테스트: 전체 사용자 여정

```typescript
// e2e/userJourney.test.ts
import { by, device, element, expect, waitFor } from 'detox';

describe('전체 사용자 여정', () => {
  beforeAll(async () => {
    await device.launchApp({
      newInstance: true,
      permissions: { notifications: 'YES', location: 'always' },
    });
  });

  it('앱을 실행하면 스플래시 후 로그인 화면이 표시된다', async () => {
    await waitFor(element(by.id('login-screen')))
      .toBeVisible()
      .withTimeout(5000);
  });

  it('로그인한다', async () => {
    await element(by.id('email-input')).typeText('user@test.com');
    await element(by.id('password-input')).typeText('pass123');
    await element(by.id('login-button')).tap();

    await waitFor(element(by.id('home-screen')))
      .toBeVisible()
      .withTimeout(10000);
  });

  it('게시글 목록을 스크롤하여 더 많은 항목을 로드한다', async () => {
    // 아래로 스크롤하여 무한 스크롤 테스트
    await element(by.id('post-list')).scroll(500, 'down');
    await waitFor(element(by.text('로딩 중...')))
      .not.toBeVisible()
      .withTimeout(5000);
  });

  it('게시글을 작성한다', async () => {
    await element(by.id('create-post-fab')).tap();
    await element(by.id('post-title-input')).typeText('새 게시글');
    await element(by.id('post-body-input')).typeText('게시글 내용입니다.');
    await element(by.id('submit-post-button')).tap();

    await waitFor(element(by.text('게시 완료!')))
      .toBeVisible()
      .withTimeout(5000);
  });

  it('프로필을 확인하고 로그아웃한다', async () => {
    await element(by.id('profile-tab')).tap();
    await expect(element(by.text('user@test.com'))).toBeVisible();

    await element(by.id('logout-button')).tap();
    await element(by.text('확인')).tap(); // 확인 다이얼로그

    await waitFor(element(by.id('login-screen')))
      .toBeVisible()
      .withTimeout(5000);
  });
});
```

### 3.9 Detox 실행 명령

```bash
# 앱 빌드 (한 번만 실행)
detox build --configuration android.emu.debug

# 테스트 실행
detox test --configuration android.emu.debug

# 특정 테스트 파일만 실행
detox test --configuration android.emu.debug e2e/login.test.ts

# 릴리스 빌드 테스트 (CI에서 권장)
detox build --configuration android.emu.release
detox test --configuration android.emu.release

# 재사용 모드 (빌드 건너뛰기, 테스트만 재실행)
detox test --configuration android.emu.debug --reuse

# 헤드리스 모드 (CI에서 사용)
detox test --configuration android.emu.release --headless

# 아티팩트 생성 (스크린샷, 로그)
detox test --configuration android.emu.debug --artifacts-location e2e/artifacts
```

---

## 4. 테스트 전략

### 4.1 반드시 E2E 테스트해야 하는 사용자 흐름

```
[Critical User Flows — 우선순위 순]

1. 앱 첫 실행 → 온보딩 → 회원가입/로그인
   - 가장 중요한 흐름. 이것이 깨지면 모든 사용자가 차단됨

2. 핵심 비즈니스 로직
   - 쇼핑몰: 상품 검색 → 장바구니 → 결제
   - SNS: 피드 로드 → 게시글 작성 → 좋아요/댓글
   - 뱅킹: 잔액 확인 → 송금

3. 인증 관련
   - 로그인/로그아웃
   - 토큰 갱신 (장시간 사용 후 토큰 만료 시)
   - 비밀번호 변경

4. 에러 복구
   - 네트워크 끊김 → 재연결 후 정상 동작
   - 백그라운드 → 포그라운드 복귀
```

### 4.2 Mock API 서버 설정

```typescript
// e2e/mockServer.ts
// json-server 또는 msw를 사용한 목 서버

// 방법 1: json-server (간단)
// package.json
// "scripts": {
//   "mock-server": "json-server --watch e2e/db.json --port 3001"
// }
```

```json
// e2e/db.json
{
  "users": [
    { "id": 1, "email": "test@example.com", "name": "테스트 사용자" }
  ],
  "posts": [
    { "id": 1, "title": "첫 번째 글", "body": "내용", "userId": 1 },
    { "id": 2, "title": "두 번째 글", "body": "내용", "userId": 1 }
  ]
}
```

```typescript
// 방법 2: MSW (Mock Service Worker) — 더 유연함
// e2e/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('/api/login', async ({ request }) => {
    const body = await request.json() as { email: string; password: string };

    if (body.email === 'test@example.com' && body.password === 'password123') {
      return HttpResponse.json({
        token: 'mock-jwt-token',
        user: { id: 1, name: '테스트 사용자' },
      });
    }

    return HttpResponse.json(
      { error: '이메일 또는 비밀번호가 잘못되었습니다' },
      { status: 401 }
    );
  }),

  http.get('/api/posts', () => {
    return HttpResponse.json([
      { id: 1, title: '첫 번째 글', body: '내용' },
      { id: 2, title: '두 번째 글', body: '내용' },
    ]);
  }),
];
```

---

## 5. 시각적 회귀 테스트

UI가 의도치 않게 변경되었는지 스크린샷을 비교하여 확인한다.

### Maestro로 시각적 테스트

```yaml
# e2e/flows/visual-test.yaml
appId: com.myapp

---
- launchApp

# 홈 화면 스크린샷
- assertVisible: "홈"
- takeScreenshot: "home_screen_light"

# 다크 모드 전환
- tapOn: "설정"
- tapOn: "다크 모드"
- pressKey: back

# 다크 모드 스크린샷
- takeScreenshot: "home_screen_dark"
```

### Detox로 시각적 테스트 (jest-image-snapshot)

```bash
npm install --save-dev jest-image-snapshot
```

```typescript
// e2e/visual.test.ts
import { by, device, element } from 'detox';
import { toMatchImageSnapshot } from 'jest-image-snapshot';

expect.extend({ toMatchImageSnapshot });

describe('시각적 회귀 테스트', () => {
  it('홈 화면이 이전과 동일하게 보인다', async () => {
    await device.launchApp({ newInstance: true });

    // 스크린샷 캡처
    const screenshot = await device.takeScreenshot('home-screen');

    // 이전 스냅샷과 비교 (허용 오차: 0.1%)
    expect(screenshot).toMatchImageSnapshot({
      failureThreshold: 0.001,
      failureThresholdType: 'percent',
    });
  });

  it('로그인 폼이 이전과 동일하게 보인다', async () => {
    await device.launchApp({ newInstance: true });
    const screenshot = await device.takeScreenshot('login-form');

    expect(screenshot).toMatchImageSnapshot({
      failureThreshold: 0.001,
      failureThresholdType: 'percent',
    });
  });
});
```

---

## 6. 접근성 테스트

### Maestro로 접근성 확인

```yaml
# e2e/flows/accessibility.yaml
appId: com.myapp

---
- launchApp

# 접근성 라벨이 있는 요소 확인
- assertVisible:
    label: "로그인 버튼"

- assertVisible:
    label: "이메일 입력 필드"

# TalkBack이 읽을 수 있는 텍스트 확인
- assertVisible:
    text: "홈 화면"
```

### Detox로 접근성 테스트

```typescript
// e2e/accessibility.test.ts
import { by, device, element, expect } from 'detox';

describe('접근성 테스트', () => {
  beforeAll(async () => {
    await device.launchApp({ newInstance: true });
  });

  it('모든 버튼에 접근성 라벨이 있다', async () => {
    // 접근성 라벨로 요소를 찾을 수 있는지 확인
    await expect(element(by.label('로그인'))).toBeVisible();
    await expect(element(by.label('회원가입'))).toBeVisible();
  });

  it('폼 필드에 적절한 힌트가 있다', async () => {
    // accessibilityHint로 설정한 텍스트 확인
    // <TextInput accessibilityLabel="이메일" accessibilityHint="이메일 주소를 입력하세요" />
    await expect(element(by.label('이메일'))).toBeVisible();
  });

  it('터치 대상이 최소 크기를 충족한다', async () => {
    // Android: 최소 48dp x 48dp
    // 이것은 네이티브 접근성 스캐너로 확인하는 것이 더 적합함
    // adb shell을 통해 접근성 스캐너 실행:
    // adb shell am start -n com.google.android.apps.accessibility.auditor/.ui.MainActivitykk
  });

  it('색상 대비가 충분하다', async () => {
    // 시각적 테스트와 결합하여 확인
    const screenshot = await device.takeScreenshot('contrast-check');
    // 별도의 색상 대비 분석 도구로 확인
  });
});
```

### 접근성 테스트를 위한 컴포넌트 작성 가이드

```typescript
// ✅ 올바른 접근성 설정
import { TouchableOpacity, Text, Image, View } from 'react-native';

function AccessibleButton() {
  return (
    <TouchableOpacity
      accessible={true}                    // TalkBack이 하나의 요소로 인식
      accessibilityLabel="장바구니에 추가"     // TalkBack이 읽는 텍스트
      accessibilityHint="이 상품을 장바구니에 추가합니다"  // 추가 설명
      accessibilityRole="button"            // 역할 (Android의 className)
      accessibilityState={{ disabled: false }} // 상태
    >
      <Image source={cartIcon} />
      <Text>장바구니</Text>
    </TouchableOpacity>
  );
}

// Android 비교:
// accessible → importantForAccessibility="yes"
// accessibilityLabel → contentDescription
// accessibilityRole → className
// accessibilityHint → 없음 (RN이 더 풍부한 접근성 API를 제공)
```

---

## 7. 종합 정리: E2E 도구 선택 가이드

```
[프로젝트 규모별 권장 도구]

소규모 프로젝트 (1-2명 개발):
  → Maestro만으로 충분
  → YAML로 빠르게 작성, 유지보수 비용 낮음

중규모 프로젝트 (3-5명 개발):
  → Maestro (주요 흐름) + RNTL (컴포넌트 테스트)
  → E2E는 핵심 흐름만, 나머지는 컴포넌트 테스트로 커버

대규모 프로젝트 (5명 이상):
  → Detox (E2E) + RNTL (컴포넌트) + Jest (단위)
  → Detox의 자동 동기화와 JS 기반 테스트가 CI에서 안정적

[공통 원칙]
1. E2E 테스트는 핵심 사용자 흐름만 (10-20개)
2. 나머지는 RNTL 컴포넌트 테스트로 커버 (50-100개)
3. 순수 로직은 Jest 단위 테스트 (필요한 만큼)
4. TypeScript가 정적 분석을 담당 (기본)
```
