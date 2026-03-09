# 개발 환경 사전 준비 — 필수 도구 설치 가이드

> React Native 0.84 개발을 위한 환경 구성을 단계별로 안내합니다.
> Android 개발자에게 익숙한 도구(Android Studio, JDK)와 새로 설치해야 하는 도구(Node.js, Watchman, Metro)를 모두 다룹니다.

---

## 1. 시스템 요구 사항

### macOS (권장 개발 환경)

| 항목 | 최소 요구 | 권장 |
|------|----------|------|
| macOS 버전 | macOS 13 (Ventura) | macOS 14 (Sonoma) 이상 |
| RAM | 8GB | 16GB 이상 |
| 디스크 여유 공간 | 20GB | 50GB 이상 (에뮬레이터 포함) |
| CPU | Apple Silicon 또는 Intel | Apple Silicon (M1 이상) |
| Xcode | 15.0 이상 (iOS 빌드 시) | 최신 버전 |

macOS는 Android와 iOS 모두 빌드할 수 있는 유일한 플랫폼이므로 권장됩니다.

### Windows

| 항목 | 최소 요구 | 권장 |
|------|----------|------|
| Windows 버전 | Windows 10 (64-bit) | Windows 11 |
| RAM | 8GB | 16GB 이상 |
| 디스크 여유 공간 | 20GB | 50GB 이상 |
| CPU | x86_64 (HAXM 지원) | AMD with Hyper-V 또는 Intel VT-x |

Windows에서는 Android 앱만 빌드 가능합니다. iOS 빌드를 위해서는 macOS가 필요합니다.

### Linux

| 항목 | 최소 요구 | 권장 |
|------|----------|------|
| 배포판 | Ubuntu 20.04 LTS | Ubuntu 22.04 LTS 이상 |
| RAM | 8GB | 16GB 이상 |
| 디스크 여유 공간 | 20GB | 50GB 이상 |

Linux에서도 Windows와 마찬가지로 Android 빌드만 가능합니다.

---

## 2. Node.js 22.11+ 설치

React Native는 JavaScript 런타임인 Node.js 위에서 동작합니다. Android 개발자에게 비유하자면, Node.js는 JVM(Java Virtual Machine)과 유사한 역할을 합니다 — JavaScript 코드를 실행하는 런타임 환경입니다.

React Native 0.84는 **Node.js 22.11 이상**을 요구합니다.

### 방법 A: nvm을 이용한 설치 (권장)

nvm(Node Version Manager)은 여러 버전의 Node.js를 관리할 수 있는 도구입니다. Android 개발에서 여러 JDK 버전을 관리하는 것과 비슷합니다.

#### macOS / Linux

```bash
# 1단계: nvm 설치 스크립트 실행
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

설치 스크립트가 자동으로 `~/.zshrc` (또는 `~/.bashrc`)에 다음 내용을 추가합니다:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

```bash
# 2단계: 터미널을 재시작하거나 설정 파일을 다시 로드
source ~/.zshrc

# 3단계: nvm 설치 확인
nvm --version
# 예상 출력: 0.40.1
```

```bash
# 4단계: Node.js 22 LTS 설치
nvm install 22

# 예상 출력:
# Downloading and installing node v22.12.0...
# Downloading https://nodejs.org/dist/v22.12.0/node-v22.12.0-darwin-arm64.tar.xz...
# ######################################################################### 100.0%
# Computing checksum with sha256sum
# Checksums matched!
# Now using node v22.12.0 (npm v10.9.0)
```

```bash
# 5단계: 기본 Node.js 버전으로 설정
nvm alias default 22

# 6단계: 설치 확인
node --version
# 예상 출력: v22.12.0 (또는 22.11 이상의 버전)

npm --version
# 예상 출력: 10.9.0 (또는 그 이상)
```

#### Windows

Windows에서는 nvm-windows를 사용합니다:

1. https://github.com/coreybutler/nvm-windows/releases 에서 `nvm-setup.exe` 다운로드
2. 설치 프로그램 실행
3. PowerShell을 관리자 권한으로 열고:

```powershell
nvm install 22
nvm use 22
node --version
# 예상 출력: v22.12.0
```

### 방법 B: Homebrew를 이용한 설치 (macOS)

```bash
# Homebrew가 설치되어 있지 않다면 먼저 설치
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Node.js 22 설치
brew install node@22

# PATH에 추가 (brew가 안내하는 대로)
echo 'export PATH="/opt/homebrew/opt/node@22/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# 확인
node --version
# 예상 출력: v22.x.x
```

> **nvm vs Homebrew**: nvm은 프로젝트마다 다른 Node.js 버전을 사용해야 할 때 유용합니다. 하나의 버전만 사용한다면 Homebrew도 충분합니다. 다만 여러 프로젝트를 다룰 계획이라면 nvm을 권장합니다.

### npm과 npx 이해하기

Node.js를 설치하면 **npm**(Node Package Manager)과 **npx**가 함께 설치됩니다.

| 도구 | 역할 | Android 비유 |
|------|------|-------------|
| npm | 패키지(라이브러리) 관리자 | Gradle의 dependencies 관리 |
| npx | 패키지를 설치하지 않고 바로 실행 | — |
| node | JavaScript 런타임 | JVM |

---

## 3. Watchman 설치

Watchman은 Facebook이 만든 파일 변경 감시 도구입니다. React Native에서 소스 코드가 변경되면 이를 감지하여 자동으로 앱을 다시 빌드하는 데 사용됩니다.

Android 개발의 **Instant Run** / **Apply Changes**가 파일 변경을 감지하는 방식과 유사하지만, Watchman은 더 효율적으로 대규모 파일 시스템의 변경을 추적합니다.

### macOS 설치

```bash
brew install watchman

# 설치 확인
watchman --version
# 예상 출력: 2024.11.04.00 (또는 그 이상)
```

### Linux 설치

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y watchman
```

빌드가 필요한 경우:
```bash
git clone https://github.com/facebook/watchman.git
cd watchman
git checkout v2024.11.04.00
sudo mkdir -p /usr/local/var/run/watchman
sudo ./install-system-packages.sh
./autogen.sh
./configure
make
sudo make install
```

### Windows

Windows에서는 Watchman이 필수가 아닙니다. React Native CLI가 자체적으로 파일 변경을 감지합니다. 하지만 성능 향상을 위해 설치를 권장합니다.

https://github.com/nicedoc/watchman-bin 에서 Windows용 바이너리를 받을 수 있습니다.

### Watchman이 하는 일

```
[파일 변경 감지 흐름]

개발자가 App.tsx 수정 → Watchman이 변경 감지 → Metro Bundler에 알림
→ 변경된 모듈만 다시 번들링 → 앱에 Hot Module Replacement 전송
→ 앱이 새로고침 없이 업데이트
```

이 과정은 Android Studio의 Apply Changes보다 훨씬 빠릅니다 (보통 1초 이내).

---

## 4. JDK 17 (Azul Zulu) 설치

React Native 0.84의 Android 빌드에는 **JDK 17**이 필요합니다. React Native 공식 문서에서는 **Azul Zulu OpenJDK**를 권장합니다.

> **왜 Azul Zulu인가?** Azul Zulu는 Oracle JDK와 100% 호환되는 무료 OpenJDK 배포판이며, Apple Silicon(M1/M2/M3)을 네이티브로 지원합니다. Android 개발자라면 Android Studio에 내장된 JDK를 사용해왔겠지만, React Native의 Gradle 빌드에서는 시스템 JDK를 참조하므로 별도 설치가 필요합니다.

### macOS 설치

```bash
# Homebrew 탭 추가
brew tap homebrew/cask-versions 2>/dev/null; true

# Azul Zulu JDK 17 설치
brew install --cask zulu@17

# 설치 확인
java -version
# 예상 출력:
# openjdk version "17.0.13" 2024-10-15 LTS
# OpenJDK Runtime Environment Zulu17.54+21-CA (build 17.0.13+11-LTS)
# OpenJDK 64-Bit Server VM Zulu17.54+21-CA (build 17.0.13+11-LTS, mixed mode, sharing)
```

### JAVA_HOME 환경 변수 설정

```bash
# Zulu JDK 17의 경로 확인
/usr/libexec/java_home -v 17
# 예상 출력 (Apple Silicon): /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
# 예상 출력 (Intel):         /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

`~/.zshrc`에 추가:
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
```

```bash
# 설정 적용
source ~/.zshrc

# 확인
echo $JAVA_HOME
# 예상 출력: /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

### Windows 설치

1. https://www.azul.com/downloads/?version=java-17-lts&package=jdk 에서 Windows용 `.msi` 파일 다운로드
2. 설치 프로그램 실행 (기본 옵션으로 진행)
3. 시스템 환경 변수 설정:
   - 시스템 속성 → 고급 → 환경 변수
   - 새 시스템 변수: `JAVA_HOME` = `C:\Program Files\Zulu\zulu-17`
   - Path에 `%JAVA_HOME%\bin` 추가

```powershell
# PowerShell에서 확인
java -version
echo $env:JAVA_HOME
```

### Linux 설치

```bash
sudo apt-get install -y zulu17-jdk

# 또는 수동 설치
# https://www.azul.com/downloads/ 에서 .deb 또는 .tar.gz 다운로드

export JAVA_HOME=/usr/lib/jvm/zulu-17
echo 'export JAVA_HOME=/usr/lib/jvm/zulu-17' >> ~/.bashrc
source ~/.bashrc
```

---

## 5. Android Studio 및 Android SDK 설정

Android 개발자라면 이미 Android Studio가 설치되어 있을 것입니다. React Native는 Android Studio 자체보다는 Android Studio에 포함된 **Android SDK**와 빌드 도구를 사용합니다.

### Android Studio 설치 (처음 설치하는 경우)

1. https://developer.android.com/studio 에서 다운로드
2. 설치 프로그램 실행
3. 설치 유형: **Custom** 선택 (필요한 컴포넌트만 설치하기 위해)

### SDK Manager 설정

Android Studio → Settings (Preferences) → Languages & Frameworks → Android SDK

#### SDK Platforms 탭

다음 항목을 체크하여 설치합니다:

| 항목 | 설명 |
|------|------|
| **Android 15 (VanillaIceCream)** — API Level 35 | React Native 0.84의 타겟 SDK |
| → Android SDK Platform 35 | 플랫폼 API |
| → Google APIs Intel x86_64 Atom System Image | Intel Mac 에뮬레이터용 |
| → Google APIs ARM 64 v8a System Image | Apple Silicon 에뮬레이터용 |

> **참고**: React Native 0.84는 `compileSdkVersion 35`, `targetSdkVersion 35`를 기본으로 사용합니다.

#### SDK Tools 탭

다음 항목을 체크하여 설치합니다:

| 항목 | 버전 | 설명 |
|------|------|------|
| Android SDK Build-Tools | **36.0.0** | APK 빌드에 필요한 도구 (aapt, dex 등) |
| Android SDK Command-line Tools (latest) | 최신 | `sdkmanager`, `avdmanager` 등 CLI 도구 |
| Android Emulator | 최신 | 안드로이드 에뮬레이터 |
| Android SDK Platform-Tools | 최신 | `adb`, `fastboot` 등 |
| NDK (Side by side) | 최신 | 네이티브 코드 빌드용 (일부 라이브러리에서 필요) |
| CMake | 최신 | 네이티브 빌드 도구 |

"Show Package Details"를 체크하면 세부 버전을 선택할 수 있습니다.

```
[SDK Manager 설정 화면 구조]

SDK Platforms 탭:
  ☑ Android 15.0 ("VanillaIceCream") — API Level 35
    ☑ Android SDK Platform 35
    ☑ Sources for Android 35
    ☑ Google APIs ARM 64 v8a System Image  (Apple Silicon Mac)
    ☑ Google APIs Intel x86_64 Atom System Image  (Intel Mac / Windows)

SDK Tools 탭:
  ☑ Android SDK Build-Tools 36.0.0
  ☑ Android SDK Command-line Tools (latest)
  ☑ Android Emulator
  ☑ Android SDK Platform-Tools
  ☑ NDK (Side by side) — 최신
  ☑ CMake — 최신
```

### Android SDK 경로 확인

SDK의 기본 설치 경로:

| OS | 기본 경로 |
|----|----------|
| macOS | `~/Library/Android/sdk` |
| Windows | `%LOCALAPPDATA%\Android\Sdk` (보통 `C:\Users\사용자명\AppData\Local\Android\Sdk`) |
| Linux | `~/Android/Sdk` |

Android Studio → Settings → Languages & Frameworks → Android SDK 에서 "Android SDK Location"으로 확인할 수 있습니다.

---

## 6. ANDROID_HOME 환경 변수 설정

React Native CLI가 Android SDK를 찾을 수 있도록 환경 변수를 설정해야 합니다.

### macOS (zsh) — 전체 ~/.zshrc 환경 변수 내용

`~/.zshrc` 파일을 편집하여 다음 내용을 추가합니다:

```bash
# ===========================================
# React Native 개발 환경 변수
# ===========================================

# Java (Azul Zulu JDK 17)
export JAVA_HOME=$(/usr/libexec/java_home -v 17)

# Android SDK
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
export PATH=$PATH:$ANDROID_HOME/build-tools/36.0.0

# Node Version Manager (nvm 사용 시)
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

설정 적용:
```bash
source ~/.zshrc
```

### macOS (bash) — ~/.bash_profile 또는 ~/.bashrc

```bash
# React Native 개발 환경 변수 (bash용)
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
export PATH=$PATH:$ANDROID_HOME/build-tools/36.0.0
```

### Windows — 시스템 환경 변수

1. 시스템 속성 → 고급 → 환경 변수
2. 새 사용자 변수 추가:
   - 변수 이름: `ANDROID_HOME`
   - 변수 값: `C:\Users\사용자명\AppData\Local\Android\Sdk`
3. Path 변수에 다음 추가:
   - `%ANDROID_HOME%\emulator`
   - `%ANDROID_HOME%\platform-tools`
   - `%ANDROID_HOME%\tools`
   - `%ANDROID_HOME%\tools\bin`
   - `%ANDROID_HOME%\cmdline-tools\latest\bin`

### Linux — ~/.bashrc

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin
```

### 각 PATH 항목의 역할

| PATH 항목 | 포함된 주요 명령어 | 용도 |
|-----------|------------------|------|
| `emulator/` | `emulator` | Android 에뮬레이터 실행 |
| `platform-tools/` | `adb`, `fastboot` | 디바이스 통신, 앱 설치 |
| `tools/` | 레거시 도구들 | 하위 호환성 |
| `tools/bin/` | `sdkmanager`, `avdmanager` (레거시) | SDK/AVD 관리 |
| `cmdline-tools/latest/bin/` | `sdkmanager`, `avdmanager` (최신) | SDK/AVD 관리 (권장) |
| `build-tools/36.0.0/` | `aapt2`, `d8`, `zipalign` | APK 빌드 도구 |

---

## 7. 모든 도구 설치 확인

모든 설치가 완료된 후 다음 명령어를 순서대로 실행하여 확인합니다:

```bash
# Node.js 확인
node --version
# ✅ 예상: v22.11.0 이상

# npm 확인
npm --version
# ✅ 예상: 10.x.x

# Watchman 확인 (macOS/Linux)
watchman --version
# ✅ 예상: 2024.x.x

# Java 확인
java -version
# ✅ 예상: openjdk version "17.0.x"

# JAVA_HOME 확인
echo $JAVA_HOME
# ✅ 예상: /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home

# ANDROID_HOME 확인
echo $ANDROID_HOME
# ✅ 예상: /Users/사용자명/Library/Android/sdk

# adb 확인
adb --version
# ✅ 예상: Android Debug Bridge version 1.0.41

# emulator 확인
emulator -list-avds
# ✅ 예상: 설치된 AVD 목록 (없으면 빈 출력)

# sdkmanager 확인
sdkmanager --version
# ✅ 예상: 12.0 이상
```

### 한 번에 확인하는 스크립트

```bash
echo "=== React Native 개발 환경 확인 ==="
echo ""
echo "Node.js: $(node --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "npm:     $(npm --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Watchman: $(watchman --version 2>/dev/null || echo 'NOT INSTALLED')"
echo "Java:    $(java -version 2>&1 | head -1)"
echo "JAVA_HOME: ${JAVA_HOME:-'NOT SET'}"
echo "ANDROID_HOME: ${ANDROID_HOME:-'NOT SET'}"
echo "adb:     $(adb --version 2>/dev/null | head -1 || echo 'NOT INSTALLED')"
echo ""
echo "=== 확인 완료 ==="
```

예상 출력:
```
=== React Native 개발 환경 확인 ===

Node.js: v22.12.0
npm:     10.9.0
Watchman: 2024.11.04.00
Java:    openjdk version "17.0.13" 2024-10-15 LTS
JAVA_HOME: /Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
ANDROID_HOME: /Users/jongwoo/Library/Android/sdk
adb:     Android Debug Bridge version 1.0.41

=== 확인 완료 ===
```

```exercise
type: categorize
question: "다음 도구를 React Native 개발에 필수인 것과 선택인 것으로 분류하세요"
categories: ["필수", "선택"]
items:
  - text: "Node.js"
    category: "필수"
  - text: "Watchman"
    category: "필수"
  - text: "JDK 17"
    category: "필수"
  - text: "Yarn"
    category: "선택"
  - text: "Expo CLI"
    category: "선택"
  - text: "Android Studio"
    category: "필수"
xp: 6
```

---

## 8. 자주 발생하는 설치 오류와 해결 방법

### 오류 1: `JAVA_HOME is not set` 또는 잘못된 JDK 버전

```
ERROR: JAVA_HOME is set to an invalid directory
```

**원인**: `JAVA_HOME`이 설정되지 않았거나, JDK 17이 아닌 다른 버전을 가리키고 있음.

**해결**:
```bash
# 현재 설정 확인
echo $JAVA_HOME
java -version

# 설치된 JDK 목록 확인 (macOS)
/usr/libexec/java_home -V
# 출력 예시:
# Matching Java Virtual Machines (2):
#     21.0.1 (arm64) "Oracle Corporation"
#     17.0.13 (arm64) "Azul Systems, Inc."

# JAVA_HOME을 JDK 17로 설정
export JAVA_HOME=$(/usr/libexec/java_home -v 17)

# ~/.zshrc에 영구적으로 추가했는지 확인
```

Android Studio에서 설정한 JDK와 터미널의 JDK가 다를 수 있습니다. React Native는 터미널의 `JAVA_HOME`을 사용합니다.

### 오류 2: `SDK location not found` — ANDROID_HOME 미설정

```
SDK location not found. Define a location with sdk.dir in the local.properties file
or with an ANDROID_HOME environment variable.
```

**원인**: `ANDROID_HOME` 환경 변수가 설정되지 않았거나, 잘못된 경로를 가리킴.

**해결**:
```bash
# ANDROID_HOME 확인
echo $ANDROID_HOME

# Android SDK가 실제로 존재하는지 확인
ls $ANDROID_HOME
# 예상 출력: build-tools  cmdline-tools  emulator  licenses  platforms  platform-tools ...

# 만약 경로가 다르다면 Android Studio에서 확인:
# Settings → Languages & Frameworks → Android SDK → Android SDK Location

# 또는 프로젝트의 android/local.properties 파일에 직접 설정
echo "sdk.dir=$HOME/Library/Android/sdk" > android/local.properties
```

### 오류 3: `watchman: command not found`

```
bash: watchman: command not found
```

**원인**: Watchman이 설치되지 않았거나, PATH에 추가되지 않음.

**해결**:
```bash
# macOS에서 Homebrew로 설치
brew install watchman

# 설치 후에도 안 된다면 Homebrew 경로 확인
brew --prefix
# /opt/homebrew  (Apple Silicon)
# /usr/local     (Intel)

# PATH에 Homebrew가 포함되어 있는지 확인
echo $PATH | tr ':' '\n' | grep brew
```

Watchman 없이도 React Native는 동작하지만, 파일 변경 감지 성능이 떨어질 수 있습니다.

### 오류 4: `Command 'adb' not found` — platform-tools PATH 미설정

```
adb: command not found
```

**원인**: `$ANDROID_HOME/platform-tools`가 PATH에 포함되지 않음.

**해결**:
```bash
# adb가 실제로 존재하는지 확인
ls $ANDROID_HOME/platform-tools/adb
# 파일이 존재하면 PATH 문제

# PATH에 추가
export PATH=$PATH:$ANDROID_HOME/platform-tools

# ~/.zshrc에 영구적으로 추가했는지 확인
# 그 후 터미널 재시작 또는:
source ~/.zshrc
```

### 오류 5: `error: no devices/emulators found` — 에뮬레이터/기기 미연결

```
error: no devices/emulators found
```

**원인**: 에뮬레이터가 실행되지 않았거나, USB 디버깅이 활성화되지 않은 물리 기기를 연결함.

**해결**:
```bash
# 연결된 디바이스 확인
adb devices
# 출력 예:
# List of devices attached
# (아무것도 없음 = 연결된 기기 없음)

# 에뮬레이터 실행 (AVD 이름 확인 후)
emulator -list-avds
# 출력: Pixel_7_API_35
emulator -avd Pixel_7_API_35

# 물리 기기의 경우 USB 디버깅 활성화 확인:
# 설정 → 개발자 옵션 → USB 디버깅 ON
```

### 오류 6: nvm 설치 후 `node: command not found`

```
zsh: command not found: node
```

**원인**: 새 터미널을 열었을 때 nvm이 자동 로드되지 않음.

**해결**:
```bash
# ~/.zshrc에 nvm 초기화 코드가 있는지 확인
cat ~/.zshrc | grep NVM

# 없다면 추가:
cat >> ~/.zshrc << 'EOF'
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
EOF

source ~/.zshrc
nvm use default
node --version
```

### 오류 7: Gradle 빌드 시 메모리 부족

```
GC overhead limit exceeded
또는
java.lang.OutOfMemoryError: Java heap space
```

**원인**: Gradle 빌드에 할당된 메모리가 부족.

**해결**: 프로젝트의 `android/gradle.properties` 파일에서 메모리 설정을 조정합니다:

```properties
# 기본값을 늘림
org.gradle.jvmargs=-Xmx4096m -XX:MaxMetaspaceSize=1024m
```

---

## 9. Android 개발자를 위한 환경 비교 요약

| 항목 | Android 네이티브 | React Native |
|------|-----------------|-------------|
| 런타임 | JVM (Dalvik/ART) | JavaScript Engine (Hermes) |
| SDK | Android SDK | Android SDK + Node.js |
| 빌드 도구 | Gradle | Metro Bundler + Gradle |
| 패키지 관리 | Gradle (Maven/Google repo) | npm / yarn |
| IDE | Android Studio | VS Code (+ Android Studio for native) |
| JDK 버전 | JDK 17 | JDK 17 (동일) |
| 핫 리로드 | Apply Changes (제한적) | Fast Refresh (거의 즉시) |
| 환경 변수 | ANDROID_HOME, JAVA_HOME | ANDROID_HOME, JAVA_HOME + Node.js PATH |

---

## 10. 다음 단계

환경 설정이 완료되었다면, 다음 문서에서 실제로 React Native 프로젝트를 생성하고 실행하는 방법을 다룹니다.

> 다음: [02-project-creation.md](./02-project-creation.md) — 프로젝트 생성과 실행
