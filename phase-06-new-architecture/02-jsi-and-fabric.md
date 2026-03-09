# JSI와 Fabric 렌더러 — 새로운 렌더링 파이프라인 이해하기

## 목차
1. [JSI (JavaScript Interface)](#1-jsi-javascript-interface)
2. [Fabric 렌더러](#2-fabric-렌더러)
3. [Hermes V1](#3-hermes-v1)

---

## 1. JSI (JavaScript Interface)

### 1.1 JSI가 대체하는 것: Bridge

Legacy Architecture에서 JS와 Native는 JSON 문자열을 주고받는 Bridge를 통해 통신했다. JSI는 이 Bridge를 완전히 대체하여, C++ 레벨에서 JS와 Native가 직접 소통하게 만든다.

```
┌────────────────────────────────────────────────────────────────┐
│                    Bridge (Legacy) 통신                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JS                          Native                             │
│  ┌──────┐    JSON 문자열     ┌──────┐                          │
│  │ obj  │ ──serialize──────►│parse │──► native obj             │
│  │      │                   │      │                           │
│  │ obj  │ ◄──deserialize───│build │◄── native obj             │
│  └──────┘                   └──────┘                           │
│                                                                 │
│  특징:                                                          │
│  • 모든 통신 비동기                                              │
│  • 매번 JSON.stringify + JSON.parse                             │
│  • 큐에 쌓여서 배치 처리                                         │
│  • 메모리 공유 불가                                              │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│                    JSI (New) 통신                                │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JS                          Native                             │
│  ┌──────┐    C++ 포인터      ┌──────┐                          │
│  │ obj  │ ══ 직접 참조 ════►│ obj  │  (같은 메모리!)           │
│  │      │                   │      │                           │
│  │ func │ ══ 직접 호출 ════►│ func │  (즉시 실행!)             │
│  └──────┘                   └──────┘                           │
│                                                                 │
│  특징:                                                          │
│  • 동기/비동기 모두 가능                                         │
│  • 직렬화 없음 — 직접 메모리 접근                                 │
│  • 즉시 실행 (큐 대기 없음)                                      │
│  • 메모리 공유 가능                                              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.2 JSI의 동작 원리: C++ Host Objects

JSI의 핵심 개념은 **Host Object**이다. C++로 작성된 객체를 JavaScript 런타임에 직접 노출시키는 방식이다.

```
┌────────────────────────────────────────────────────────────────┐
│                    Host Object 개념도                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JavaScript Runtime (Hermes)                                    │
│  ┌──────────────────────────────────────┐                      │
│  │                                      │                      │
│  │  const nativeCalc = global.__calc;   │                      │
│  │  const result = nativeCalc.add(3,5); │ ← 동기적 호출!       │
│  │  console.log(result); // 8           │ ← 즉시 결과 반환!    │
│  │                                      │                      │
│  └───────────────┬──────────────────────┘                      │
│                  │ JSI binding (C++ 포인터)                     │
│                  ▼                                              │
│  C++ Layer                                                      │
│  ┌──────────────────────────────────────┐                      │
│  │  class CalcHostObject               │                      │
│  │    : public jsi::HostObject {       │                      │
│  │    jsi::Value get(Runtime& rt,      │                      │
│  │      const PropNameID& name) {      │                      │
│  │      if (name == "add") {           │                      │
│  │        return Function::create(     │                      │
│  │          rt, "add", 2,              │                      │
│  │          [](Runtime& rt,            │                      │
│  │             const Value& thisVal,   │                      │
│  │             const Value* args,      │                      │
│  │             size_t count) {         │                      │
│  │            double a = args[0]       │                      │
│  │              .asNumber();           │                      │
│  │            double b = args[1]       │                      │
│  │              .asNumber();           │                      │
│  │            return Value(a + b);     │                      │
│  │          });                        │                      │
│  │      }                              │                      │
│  │    }                                │                      │
│  │  };                                 │                      │
│  └──────────────────────────────────────┘                      │
│                  │                                              │
│                  ▼ (필요 시 네이티브 호출)                       │
│  Native (Kotlin/Java or Objective-C)                            │
│  ┌──────────────────────────────────────┐                      │
│  │  실제 플랫폼 API 접근                 │                      │
│  │  Camera, Bluetooth, FileSystem 등    │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 JSI의 핵심 기능들

**기능 1: 동기적 함수 호출**

```javascript
// Legacy Bridge: 항상 비동기, Promise 필수
const result = await NativeModules.Calculator.add(3, 5);

// JSI: 동기 호출 가능! — 일반 함수 호출처럼 즉시 반환
const result = global.__calculator.add(3, 5);
// result = 8 (즉시)
```

**기능 2: 직렬화 없는 데이터 전달**

```javascript
// Legacy Bridge: 복잡한 객체를 JSON으로 변환해야 함
// JS: { users: [{name:"Kim", age:30}, ...10000개] }
// → JSON.stringify (비용 발생)
// → 문자열 전송
// → JSON.parse (비용 발생)

// JSI: 직접 참조로 전달, 직렬화 없음
// C++ 객체가 JS에서 직접 접근 가능
// 10,000개 객체를 복사하지 않고 참조로 접근
```

**기능 3: 공유 메모리 (ArrayBuffer)**

```javascript
// JSI를 통해 Native와 JS가 같은 메모리 블록을 공유
// 이미지 처리, 오디오 데이터 등에 매우 유용

// C++에서 ArrayBuffer 생성 → JS에서 직접 접근
const imageBuffer = global.__imageProcessor.getBuffer();
// imageBuffer는 Native 메모리를 직접 가리킴
// 복사 없이 바로 조작 가능!
```

**기능 4: JavaScript 런타임 교체 가능**

```
┌─────────────────────────────────────────────────┐
│          JSI의 런타임 추상화                       │
├─────────────────────────────────────────────────┤
│                                                  │
│  JSI Interface (추상 레이어)                      │
│  ┌──────────────────────────────────┐           │
│  │  class Runtime {                 │           │
│  │    virtual Value evaluateJS(...) │           │
│  │    virtual Object createObject() │           │
│  │    virtual String createString() │           │
│  │  }                               │           │
│  └────────┬──────────┬──────────────┘           │
│           │          │                           │
│     ┌─────▼────┐ ┌──▼────────┐                  │
│     │  Hermes  │ │    V8     │  ← 교체 가능!    │
│     │  (기본)  │ │ (선택적)  │                   │
│     └──────────┘ └───────────┘                  │
│                                                  │
│  Android 비유:                                   │
│  Retrofit의 Converter 교체와 유사                 │
│  Gson ↔ Moshi ↔ Jackson 교체하듯                 │
│  Hermes ↔ V8 ↔ JSC 교체 가능                    │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 1.4 Android JNI와 JSI 비교

Android 개발자라면 JNI(Java Native Interface)를 알 것이다. JSI는 개념적으로 매우 유사하다.

```
┌──────────────────────────┬──────────────────────────┐
│     Android JNI           │     React Native JSI      │
├──────────────────────────┼──────────────────────────┤
│ Java/Kotlin ↔ C++         │ JavaScript ↔ C++          │
│ JNI_OnLoad에서 등록       │ 런타임 초기화 시 등록      │
│ native 키워드로 선언      │ HostObject으로 노출       │
│ 동기 호출 가능            │ 동기 호출 가능             │
│ 직접 메모리 접근 가능     │ 직접 메모리 접근 가능      │
│ 타입 시그니처 필요        │ Codegen이 타입 생성       │
│ System.loadLibrary()     │ JSI 바인딩 자동 등록       │
└──────────────────────────┴──────────────────────────┘
```

**JNI 예시 (Android)**:

```kotlin
// Kotlin 선언
class NativeLib {
    external fun add(a: Int, b: Int): Int

    companion object {
        init { System.loadLibrary("native-lib") }
    }
}
```

```cpp
// C++ 구현 (JNI)
extern "C" JNIEXPORT jint JNICALL
Java_com_example_NativeLib_add(JNIEnv *env, jobject thiz, jint a, jint b) {
    return a + b;
}
```

**JSI 예시 (React Native)**:

```cpp
// C++ Host Object (JSI)
class Calculator : public jsi::HostObject {
public:
  jsi::Value get(jsi::Runtime& rt, const jsi::PropNameID& propName) override {
    auto name = propName.utf8(rt);
    if (name == "add") {
      return jsi::Function::createFromHostFunction(
        rt, propName, 2,
        [](jsi::Runtime& rt, const jsi::Value& thisVal,
           const jsi::Value* args, size_t count) -> jsi::Value {
          double a = args[0].asNumber();
          double b = args[1].asNumber();
          return jsi::Value(a + b);
        });
    }
    return jsi::Value::undefined();
  }
};

// 런타임에 등록
void install(jsi::Runtime& rt) {
  auto calc = std::make_shared<Calculator>();
  auto obj = jsi::Object::createFromHostObject(rt, calc);
  rt.global().setProperty(rt, "__calculator", std::move(obj));
}
```

```javascript
// JavaScript에서 사용
const result = global.__calculator.add(3, 5); // 8 — 동기, 즉시 반환
```

### 1.5 JSI 실전 활용 사례

**사례 1: 고성능 데이터 처리 라이브러리**

```
JSI를 활용하는 대표 라이브러리들:

react-native-reanimated:
  애니메이션 코드가 JS Thread가 아닌 UI Thread에서 직접 실행
  JSI로 JS 함수를 UI Thread에서 호출 → 60fps 보장

react-native-mmkv:
  Key-Value 저장소 (SharedPreferences 대체)
  JSI로 C++ MMKV에 직접 접근 → AsyncStorage보다 30배 빠름

  // AsyncStorage (Bridge 기반):
  const value = await AsyncStorage.getItem('key'); // 비동기, 느림

  // MMKV (JSI 기반):
  const value = storage.getString('key'); // 동기, 즉시

react-native-quick-sqlite:
  SQLite를 JSI로 직접 접근
  Bridge 오버헤드 없이 쿼리 실행 → Room DB와 비슷한 성능
```

**사례 2: 동기적 레이아웃 측정**

```javascript
// Legacy: 비동기로만 측정 가능, 결과가 다음 프레임에 옴
UIManager.measure(viewRef, (x, y, width, height) => {
  // 여기서 이미 한 프레임 지남 → 깜빡임 가능
});

// JSI + Fabric: 동기적 측정 가능
// useLayoutEffect 안에서 측정하면 같은 프레임에서 처리
useLayoutEffect(() => {
  // Fabric이 JSI를 통해 동기적으로 레이아웃 정보 반환
  // 화면에 그리기 전에 측정 완료 → 깜빡임 없음
}, []);
```

```exercise
type: categorize
question: "다음을 Legacy Architecture와 New Architecture로 분류하세요"
categories: ["Legacy Architecture", "New Architecture"]
items:
  - text: "Bridge (JSON 직렬화)"
    category: "Legacy Architecture"
  - text: "JSI (C++ 직접 바인딩)"
    category: "New Architecture"
  - text: "비동기 통신만 가능"
    category: "Legacy Architecture"
  - text: "동기/비동기 모두 가능"
    category: "New Architecture"
  - text: "Legacy Renderer"
    category: "Legacy Architecture"
  - text: "Fabric"
    category: "New Architecture"
xp: 6
```

---

## 2. Fabric 렌더러

### 2.1 Fabric이 대체하는 것: Legacy Renderer

Legacy Renderer는 Bridge를 통해 비동기적으로 뷰를 조작했다. Fabric은 JSI 기반으로 렌더링 파이프라인을 완전히 재설계했다.

```
┌────────────────────────────────────────────────────────────────┐
│            Legacy Renderer vs Fabric 비교                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Legacy Renderer:                                               │
│  JS Thread                Shadow Thread           UI Thread     │
│  ┌─────────┐             ┌──────────┐           ┌─────────┐   │
│  │ React   │  Bridge     │  Shadow  │  Bridge   │ Native  │   │
│  │ render  │ ──(async)──►│  Tree +  │──(async)─►│ Views   │   │
│  │         │             │  Yoga    │           │         │   │
│  └─────────┘             └──────────┘           └─────────┘   │
│  세 스레드가 Bridge 큐를 통해 비동기 통신                        │
│  → 프레임 드롭, UI 지연                                        │
│                                                                 │
│  Fabric:                                                        │
│  JS Thread                C++ Layer              UI Thread      │
│  ┌─────────┐             ┌──────────┐           ┌─────────┐   │
│  │ React   │  JSI 직접   │  Shadow  │  직접     │ Native  │   │
│  │ render  │ ══(sync)══►│  Tree +  │══(sync)═►│ Views   │   │
│  │         │             │  Yoga    │           │         │   │
│  └─────────┘             └──────────┘           └─────────┘   │
│  C++ 레이어에서 Shadow Tree를 직접 관리                         │
│  → 동기 통신 가능, 프레임 드롭 최소화                            │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 Fabric 렌더링 파이프라인: 3단계

Fabric의 렌더링은 **Render → Commit → Mount** 세 단계로 나뉜다.

```
┌────────────────────────────────────────────────────────────────┐
│              Fabric 렌더링 3단계 파이프라인                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ╔═══════════════════════════════════════════╗                  │
│  ║  Phase 1: RENDER (JS Thread)              ║                  │
│  ╠═══════════════════════════════════════════╣                  │
│  ║                                           ║                  │
│  ║  React가 컴포넌트 함수를 실행하여           ║                  │
│  ║  React Element Tree 생성                   ║                  │
│  ║                                           ║                  │
│  ║  function App() {                         ║                  │
│  ║    return (                               ║                  │
│  ║      <View style={{padding: 16}}>         ║                  │
│  ║        <Text style={{fontSize: 20}}>      ║                  │
│  ║          Hello                            ║                  │
│  ║        </Text>                            ║                  │
│  ║        <Image source={logo} />            ║                  │
│  ║      </View>                              ║                  │
│  ║    );                                     ║                  │
│  ║  }                                        ║                  │
│  ║                                           ║                  │
│  ║  결과: React Element Tree                  ║                  │
│  ║  { type: 'View', props: {padding:16},     ║                  │
│  ║    children: [                            ║                  │
│  ║      {type:'Text', props:{fontSize:20}},  ║                  │
│  ║      {type:'Image', props:{source:logo}}  ║                  │
│  ║    ]                                      ║                  │
│  ║  }                                        ║                  │
│  ║                                           ║                  │
│  ╚═════════════════════╦═════════════════════╝                  │
│                        ║ JSI (C++ 직접 호출)                     │
│                        ▼                                        │
│  ╔═══════════════════════════════════════════╗                  │
│  ║  Phase 2: COMMIT (C++ Layer, 어떤 스레드든)║                  │
│  ╠═══════════════════════════════════════════╣                  │
│  ║                                           ║                  │
│  ║  2a. Shadow Tree 생성                      ║                  │
│  ║  C++ ShadowNode 트리 구축                  ║                  │
│  ║                                           ║                  │
│  ║  ShadowNode(View, padding:16)             ║                  │
│  ║    ├─ ShadowNode(Text, fontSize:20)       ║                  │
│  ║    │   └─ RawText("Hello")                ║                  │
│  ║    └─ ShadowNode(Image, source:logo)      ║                  │
│  ║                                           ║                  │
│  ║  2b. Tree Diffing                          ║                  │
│  ║  이전 Shadow Tree와 비교하여 변경 사항 계산   ║                  │
│  ║  (새 트리 vs 기존 트리의 차이점)              ║                  │
│  ║                                           ║                  │
│  ║  2c. Yoga 레이아웃 계산                     ║                  │
│  ║  Flexbox 스타일 → 절대 좌표/크기 변환        ║                  │
│  ║                                           ║                  │
│  ║  View:  {x:0,  y:0,  w:1080, h:2340}     ║                  │
│  ║  Text:  {x:16, y:16, w:200,  h:28}       ║                  │
│  ║  Image: {x:16, y:60, w:100,  h:100}      ║                  │
│  ║                                           ║                  │
│  ╚═════════════════════╦═════════════════════╝                  │
│                        ║                                        │
│                        ▼                                        │
│  ╔═══════════════════════════════════════════╗                  │
│  ║  Phase 3: MOUNT (UI Thread)               ║                  │
│  ╠═══════════════════════════════════════════╣                  │
│  ║                                           ║                  │
│  ║  변경 사항만 네이티브 뷰에 적용             ║                  │
│  ║                                           ║                  │
│  ║  • CREATE: 새 뷰 생성                     ║                  │
│  ║    ViewGroup container = new ViewGroup(); ║                  │
│  ║    container.setPadding(16,16,16,16);     ║                  │
│  ║                                           ║                  │
│  ║  • INSERT: 자식 뷰 추가                   ║                  │
│  ║    container.addView(textView);           ║                  │
│  ║    container.addView(imageView);          ║                  │
│  ║                                           ║                  │
│  ║  • UPDATE: 기존 뷰 속성 변경              ║                  │
│  ║    textView.setText("Hello");             ║                  │
│  ║    textView.setTextSize(20);              ║                  │
│  ║                                           ║                  │
│  ║  • DELETE: 불필요한 뷰 제거               ║                  │
│  ║    parent.removeView(oldView);            ║                  │
│  ║                                           ║                  │
│  ╚═══════════════════════════════════════════╝                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.3 Yoga 엔진: Flexbox 레이아웃 계산

Yoga는 Facebook이 만든 크로스 플랫폼 Flexbox 레이아웃 엔진이다 (C++로 작성). CSS Flexbox를 네이티브 뷰의 절대 좌표로 변환한다.

```
┌────────────────────────────────────────────────────────────────┐
│                    Yoga 레이아웃 계산 과정                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  입력 (Flexbox 스타일):                                         │
│  ┌──────────────────────────────────────┐                      │
│  │ <View style={{                       │                      │
│  │   flexDirection: 'column',           │                      │
│  │   justifyContent: 'center',          │                      │
│  │   padding: 16,                       │                      │
│  │   width: '100%',                     │                      │
│  │   height: 200                        │                      │
│  │ }}>                                  │                      │
│  │   <Text style={{fontSize: 20}}>      │                      │
│  │     Hello                            │                      │
│  │   </Text>                            │                      │
│  │   <Text style={{fontSize: 14}}>      │                      │
│  │     World                            │                      │
│  │   </Text>                            │                      │
│  │ </View>                              │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
│  Yoga 계산 과정:                                                │
│  ┌──────────────────────────────────────┐                      │
│  │ 1. 루트 노드 크기 결정: 1080 x 200   │                      │
│  │ 2. padding 적용: 내부 영역 1048 x 168 │                      │
│  │ 3. 자식 노드 크기 계산:               │                      │
│  │    Text1: 폭 자동, 높이 28 (fontSize) │                      │
│  │    Text2: 폭 자동, 높이 20 (fontSize) │                      │
│  │ 4. justifyContent:center 적용        │                      │
│  │    총 자식 높이: 48                   │                      │
│  │    남는 공간: 168 - 48 = 120          │                      │
│  │    위 여백: 60                        │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
│  출력 (절대 좌표):                                               │
│  ┌────────────────────────── 1080 ──────────────────────────┐  │
│  │ View (0, 0, 1080, 200)                                    │  │
│  │ ┌────────────────────── 1048 ────────────────────────┐    │  │
│  │ │ padding 영역                                        │    │  │
│  │ │                                                     │    │  │
│  │ │   (여백 60px)                                       │    │  │
│  │ │                                                     │    │  │
│  │ │ ┌─────────────────────────────────────────────┐     │    │  │
│  │ │ │ Text1 (16, 76, 80, 28) "Hello"              │     │    │  │
│  │ │ └─────────────────────────────────────────────┘     │    │  │
│  │ │ ┌─────────────────────────────────────────────┐     │    │  │
│  │ │ │ Text2 (16, 104, 60, 20) "World"             │     │    │  │
│  │ │ └─────────────────────────────────────────────┘     │    │  │
│  │ │                                                     │    │  │
│  │ │   (여백 60px)                                       │    │  │
│  │ │                                                     │    │  │
│  │ └─────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 2.4 Android View 레이아웃과 Yoga 비교

| Android (XML/Compose) | React Native (Yoga/Flexbox) | 설명 |
|----------------------|---------------------------|------|
| `LinearLayout(vertical)` | `flexDirection: 'column'` | 세로 배치 |
| `LinearLayout(horizontal)` | `flexDirection: 'row'` | 가로 배치 |
| `gravity: center` | `justifyContent: 'center'` + `alignItems: 'center'` | 중앙 정렬 |
| `layout_weight: 1` | `flex: 1` | 남는 공간 채우기 |
| `match_parent` | `width: '100%'` 또는 `alignSelf: 'stretch'` | 부모 크기 맞춤 |
| `wrap_content` | 기본값 (자동) | 내용 크기 |
| `padding: 16dp` | `padding: 16` | 안쪽 여백 (dp 단위) |
| `margin: 8dp` | `margin: 8` | 바깥 여백 |
| `ConstraintLayout` | `position: 'absolute'` + 좌표 | 절대 위치 |
| `View.measure()` | Yoga 내부 계산 | 크기 측정 |
| `View.layout()` | Mount 단계에서 적용 | 위치 배치 |

**Yoga 레이아웃의 Android 코드 매핑**:

```kotlin
// Android에서 동일한 레이아웃을 코드로 구현하면:
val container = LinearLayout(context).apply {
    orientation = LinearLayout.VERTICAL
    gravity = Gravity.CENTER
    setPadding(16.dp, 16.dp, 16.dp, 16.dp)
    layoutParams = ViewGroup.LayoutParams(
        ViewGroup.LayoutParams.MATCH_PARENT, 200.dp
    )
}

val text1 = TextView(context).apply {
    text = "Hello"
    textSize = 20f
}

val text2 = TextView(context).apply {
    text = "World"
    textSize = 14f
}

container.addView(text1)
container.addView(text2)
```

```tsx
// React Native에서 동일한 레이아웃:
<View style={{
  flexDirection: 'column',
  justifyContent: 'center',
  padding: 16,
  width: '100%',
  height: 200
}}>
  <Text style={{ fontSize: 20 }}>Hello</Text>
  <Text style={{ fontSize: 14 }}>World</Text>
</View>

// Yoga가 내부적으로 위의 Android 코드와 동일한 결과를 계산
```

### 2.5 useLayoutEffect: 깜빡임 없는 측정과 업데이트

Fabric에서는 `useLayoutEffect`가 동기적으로 실행되어, 레이아웃 측정 후 화면에 그리기 전에 업데이트할 수 있다.

```tsx
// 문제 상황: 동적 높이에 따라 위치를 조정해야 할 때

// BAD — useEffect는 화면에 그린 후 실행 → 깜빡임
function BadTooltip({ text, targetRef }) {
  const [position, setPosition] = useState({ top: 0 });

  useEffect(() => {
    // 화면에 이미 그려진 후 실행됨
    // 위치가 0 → 계산된 값으로 변경 → 깜빡임!
    targetRef.current.measure((x, y, width, height, pageX, pageY) => {
      setPosition({ top: pageY + height });
    });
  }, []);

  return <View style={{ position: 'absolute', top: position.top }}>
    <Text>{text}</Text>
  </View>;
}

// GOOD — useLayoutEffect는 화면에 그리기 전 동기적 실행 → 깜빡임 없음
function GoodTooltip({ text, targetRef }) {
  const [position, setPosition] = useState({ top: 0 });

  useLayoutEffect(() => {
    // Fabric + JSI: 동기적으로 측정 → 같은 프레임에서 위치 설정
    // 화면에는 최종 위치만 표시 → 깜빡임 없음!
    targetRef.current.measure((x, y, width, height, pageX, pageY) => {
      setPosition({ top: pageY + height });
    });
  }, []);

  return <View style={{ position: 'absolute', top: position.top }}>
    <Text>{text}</Text>
  </View>;
}
```

```
┌────────────────────────────────────────────────────────────────┐
│              useEffect vs useLayoutEffect 타이밍                 │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  useEffect:                                                     │
│  Render → Commit → Mount → 화면표시 → useEffect 실행            │
│                                  ↑              │               │
│                             사용자가 봄      다시 업데이트       │
│                             (깜빡임!)        → 두 번째 렌더     │
│                                                                 │
│  useLayoutEffect:                                               │
│  Render → Commit → Mount → useLayoutEffect 실행 → 화면표시     │
│                                   │                    ↑        │
│                              동기 업데이트        사용자가 봄    │
│                              (같은 프레임)       (최종 결과만)   │
│                                                                 │
│  Android 비유:                                                   │
│  useEffect ≈ View.post { } (다음 프레임에서 실행)                │
│  useLayoutEffect ≈ View.doOnLayout { } (레이아웃 직후 동기 실행) │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.6 Fabric의 동시성(Concurrent) 렌더링 지원

Fabric은 React 19의 동시성 기능을 네이티브 렌더링에서 지원한다.

```
┌────────────────────────────────────────────────────────────────┐
│          Fabric + Concurrent Rendering                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Legacy:                                                        │
│  렌더 시작 ═══════════════════════════════ 렌더 완료 → Mount    │
│  (중단 불가 — 시작하면 끝까지 가야 함)                            │
│                                                                 │
│  Fabric + Concurrent:                                           │
│  렌더 시작 ════╗                                                │
│                ║ 긴급 업데이트 발생!                              │
│                ╠═ 현재 렌더 중단                                 │
│                ║                                                 │
│                ╠═ 긴급 렌더 처리 ══► Mount (즉시 화면 반영)       │
│                ║                                                 │
│                ╠═ 이전 렌더 재개 또는 폐기                        │
│                ╚═════════════════════════► Mount                 │
│                                                                 │
│  Android Compose 비유:                                          │
│  Compose도 recomposition을 중단하고                              │
│  더 긴급한 recomposition을 먼저 처리할 수 있다                    │
│  → 동일한 개념!                                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**실전 예시 — 탭 전환과 타이핑을 동시에**:

```tsx
import { useTransition, useState, Suspense } from 'react';

function TabContainer() {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();

  const switchTab = (tab: string) => {
    // 탭 전환은 transition으로 — 긴급하지 않은 업데이트
    startTransition(() => {
      setActiveTab(tab);
    });
  };

  return (
    <View>
      {/* 탭 버튼들 — 즉시 반응 */}
      <View style={styles.tabBar}>
        <Pressable onPress={() => switchTab('home')}>
          <Text>Home</Text>
        </Pressable>
        <Pressable onPress={() => switchTab('search')}>
          <Text>Search</Text>
        </Pressable>
      </View>

      {/* isPending이면 로딩 표시 */}
      <View style={{ opacity: isPending ? 0.7 : 1.0 }}>
        <Suspense fallback={<ActivityIndicator size="large" />}>
          {activeTab === 'home' && <HomeScreen />}
          {activeTab === 'search' && <SearchScreen />}
        </Suspense>
      </View>
    </View>
  );
}
```

### 2.7 Fabric과 Android RecyclerView 비교

```
┌────────────────────────────────────────────────────────────────┐
│        RecyclerView와 Fabric의 유사 개념 비교                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  RecyclerView                    Fabric                         │
│  ─────────────                   ─────────                      │
│  ViewHolder 재활용               ShadowNode 재활용              │
│  DiffUtil (변경사항 계산)         Tree Diffing (C++)             │
│  LayoutManager (위치 결정)        Yoga (Flexbox 레이아웃)        │
│  pre-layout (애니메이션 준비)     pre-rendering (off-screen)     │
│  notifyItemChanged (부분 업데이트) 개별 ShadowNode 업데이트       │
│  Adapter.onBindViewHolder        Mount: 뷰 속성 설정             │
│  Adapter.onCreateViewHolder      Mount: 뷰 생성                  │
│  ItemAnimator                    LayoutAnimation / Reanimated    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.8 Fabric 컴포넌트와 Legacy 컴포넌트의 차이

```tsx
// Legacy View Manager (old)
// Java에서 SimpleViewManager 상속

// Fabric Native Component (new)
// TypeScript spec → Codegen → Kotlin 구현

// TypeScript Spec 예시
import type { ViewProps } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';

interface NativeProps extends ViewProps {
  color: string;
  size: number;
}

export default codegenNativeComponent<NativeProps>('CustomBadge');
```

```kotlin
// Kotlin ViewManager 구현 (Fabric)
class CustomBadgeManager : SimpleViewManager<CustomBadgeView>() {
    override fun getName() = "CustomBadge"

    override fun createViewInstance(context: ThemedReactContext): CustomBadgeView {
        return CustomBadgeView(context)
    }

    @ReactProp(name = "color")
    fun setColor(view: CustomBadgeView, color: String) {
        view.setBadgeColor(Color.parseColor(color))
    }

    @ReactProp(name = "size")
    fun setSize(view: CustomBadgeView, size: Int) {
        view.setBadgeSize(size)
    }
}
```

---

## 3. Hermes V1

### 3.1 Hermes란?

Hermes는 Facebook(Meta)이 React Native를 위해 만든 JavaScript 엔진이다. Chrome의 V8이나 Safari의 JavaScriptCore(JSC)와 같은 역할을 하지만, 모바일 환경에 최적화되어 있다.

```
┌────────────────────────────────────────────────────────────────┐
│                JavaScript 엔진 비교                              │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  엔진     │ 사용처            │ 특징                            │
│  ─────────┼──────────────────┼────────────────────────────     │
│  V8       │ Chrome, Node.js  │ JIT 컴파일, 최고 실행 속도       │
│  JSC      │ Safari, iOS      │ JIT 컴파일, iOS 최적화           │
│  Hermes   │ React Native     │ AOT 컴파일, 모바일 최적화        │
│  SpiderM. │ Firefox          │ JIT 컴파일, 웹 표준 선도         │
│                                                                 │
│  Hermes의 핵심 전략:                                            │
│  "실행 속도를 약간 양보하고,                                     │
│   시작 속도와 메모리 효율을 극대화한다"                            │
│                                                                 │
│  모바일에서는:                                                   │
│  • 앱 시작 시간이 사용자 경험에 직접적                            │
│  • 메모리가 제한적 (특히 저사양 기기)                             │
│  • 장시간 실행보다 빠른 초기 로딩이 중요                          │
│  → Hermes가 최적의 선택                                         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 3.2 바이트코드 사전 컴파일 (AOT)

Hermes의 가장 큰 특징은 **빌드 시 JavaScript를 바이트코드로 미리 컴파일**하는 것이다.

```
┌────────────────────────────────────────────────────────────────┐
│               V8 (JIT) vs Hermes (AOT) 실행 흐름                │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  V8 (Chrome/Node):                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ JS 소스   │→│  파싱    │→│ AST 생성  │→│ 바이트코드│      │
│  │ (텍스트)  │  │ (런타임) │  │ (런타임)  │  │ (런타임)  │      │
│  └──────────┘  └──────────┘  └──────────┘  └────┬─────┘      │
│                                                  │             │
│          앱 시작 시 이 모든 과정이 실행됨          │             │
│          (시작 시간에 포함)                        ▼             │
│                                           ┌──────────┐        │
│                                           │  JIT 컴파일│        │
│                                           │ (핫 경로)  │        │
│                                           └──────────┘        │
│                                                                 │
│  Hermes (AOT):                                                  │
│  빌드 시:                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ JS 소스   │→│  파싱    │→│ AST 생성  │→│ 바이트코드│      │
│  │ (텍스트)  │  │ (빌드시) │  │ (빌드시)  │  │ (.hbc 파일)│     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                  │             │
│  실행 시:                                         │             │
│  ┌──────────┐                                    │             │
│  │ .hbc 파일 │ ◄─────────────────────────────────┘             │
│  │ 바로 실행 │ → 파싱/컴파일 단계 건너뜀!                        │
│  └──────────┘                                                  │
│                                                                 │
│  결과: 앱 시작 시간 대폭 단축                                    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 3.3 Android ART/Dalvik과 Hermes 비교

```
┌──────────────────────────┬──────────────────────────────┐
│   Android ART             │   React Native Hermes        │
├──────────────────────────┼──────────────────────────────┤
│ .java → .class → .dex    │ .js → .hbc (Hermes Bytecode) │
│ (Java 바이트코드)         │ (Hermes 바이트코드)           │
│                           │                              │
│ AOT 컴파일 (설치 시)      │ AOT 컴파일 (빌드 시)         │
│ .dex → native code        │ .js → .hbc                   │
│ (oat 파일)               │                              │
│                           │                              │
│ 프로파일 기반 최적화       │ 바이트코드 최적화            │
│ (PGO: 자주 쓰는 코드 최적)│                              │
│                           │                              │
│ GC: Concurrent Copying    │ GC: GenGC (세대별 GC)        │
│ (동시 GC, 짧은 pause)     │ (Young/Old 세대 분리)        │
│                           │                              │
│ Memory-mapped .oat 파일   │ Memory-mapped .hbc 파일      │
│ (필요한 부분만 로드)      │ (필요한 부분만 로드)          │
└──────────────────────────┴──────────────────────────────┘
```

### 3.4 Hermes V1의 주요 개선사항 (React Native 0.84)

```
┌────────────────────────────────────────────────────────────────┐
│               Hermes V1 주요 개선사항                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 시작 시간 (Cold Start)                                      │
│     ─────────────────────                                      │
│     Hermes V0: ~800ms (중간 규모 앱)                            │
│     Hermes V1: ~500ms (약 37% 개선)                             │
│                                                                 │
│     이유: 바이트코드 포맷 최적화, 지연 파싱 개선                   │
│                                                                 │
│  2. 메모리 사용량                                                │
│     ─────────────────                                           │
│     Hermes V0: ~180MB (중간 규모 앱)                            │
│     Hermes V1: ~140MB (약 22% 감소)                             │
│                                                                 │
│     이유: GC 개선, 객체 표현 최적화                               │
│                                                                 │
│  3. 실행 속도                                                    │
│     ──────────                                                  │
│     인터프리터 성능 15~20% 향상                                   │
│     자주 실행되는 함수 최적화                                     │
│                                                                 │
│  4. ES2022+ 지원 확대                                            │
│     ─────────────────                                           │
│     • class fields (public/private)                             │
│     • WeakRef, FinalizationRegistry                             │
│     • Array.at(), Object.hasOwn()                               │
│     • Error.cause                                               │
│     • top-level await (모듈 환경)                                │
│                                                                 │
│  5. Intl (국제화) API 지원 개선                                  │
│     ─────────────────────────                                   │
│     Intl.NumberFormat 완전 지원                                  │
│     Intl.DateTimeFormat 완전 지원                                │
│     → 별도 polyfill 불필요                                      │
│                                                                 │
│  6. 디버깅 지원 개선                                             │
│     ────────────────                                            │
│     Chrome DevTools Protocol 호환                                │
│     소스맵 통합 개선                                             │
│     프로파일링 정확도 향상                                       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 3.5 Hermes 바이트코드 확인하기

```bash
# 앱을 빌드하면 JS 번들이 .hbc로 변환됨
# Android APK 안에서 확인:
# assets/index.android.bundle → Hermes 바이트코드 (.hbc)

# 바이트코드 디스어셈블 (디버깅용)
npx hermes --dump-bytecode assets/index.android.bundle

# 결과 예시:
# Function<global>(1 params, 13 registers, 0 symbols):
#   LoadConstUndefined r0
#   Call1            r0, r1, r0
#   Ret              r0
```

### 3.6 Hermes 설정 확인 (React Native 0.84)

React Native 0.84에서는 Hermes가 기본 엔진이다. 별도 설정 없이 자동으로 사용된다.

```groovy
// android/app/build.gradle
// React Native 0.84에서는 Hermes가 기본값이므로 별도 설정 불필요
// 만약 확인하고 싶다면:
android {
    buildTypes {
        release {
            // Hermes는 기본적으로 활성화됨
            // 프로파일 빌드 (Flipper 디버깅용):
            // debuggableVariants = ["debug"]
        }
    }
}

// react-native.config.js에서 확인 가능:
// Hermes는 0.82+에서 유일한 지원 엔진
```

```javascript
// 런타임에 Hermes 사용 중인지 확인
const isHermes = () => !!global.HermesInternal;
console.log('Hermes 사용 중:', isHermes()); // true
```

### 3.7 Hermes의 GC (Garbage Collection) 이해

```
┌────────────────────────────────────────────────────────────────┐
│                Hermes GenGC 동작 방식                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  힙 메모리 구조:                                                │
│  ┌────────────────────────────────────────────────┐            │
│  │  Young Generation (YG)     │  Old Generation   │            │
│  │  ┌──────────────────────┐  │  ┌──────────────┐ │            │
│  │  │ 새로 생성된 객체     │  │  │ 오래 살아남은│ │            │
│  │  │ 대부분 금방 GC됨    │  │  │ 객체          │ │            │
│  │  │ (짧은 수명)         │  │  │ (긴 수명)     │ │            │
│  │  └──────────────────────┘  │  └──────────────┘ │            │
│  └────────────────────────────────────────────────┘            │
│                                                                 │
│  GC 전략:                                                       │
│  • Young GC: 자주, 빠르게 (1~2ms)                              │
│    → 새 객체 중 참조 없는 것 즉시 수거                           │
│  • Old GC: 가끔, 좀 더 오래 (5~10ms)                           │
│    → 오래된 객체 중 참조 없는 것 수거                             │
│                                                                 │
│  Android ART GC와 비교:                                         │
│  ART도 세대별 GC 사용 (Nursery + Main space)                    │
│  Hermes GenGC는 ART의 설계를 참고하여 만들어짐                   │
│                                                                 │
│  개발자 팁:                                                     │
│  • 큰 배열/객체를 반복 생성하지 마라 → GC 부담                   │
│  • 이벤트 핸들러에서 객체 리터럴 반복 생성 피하기                  │
│  • useMemo/useCallback으로 불필요한 객체 생성 방지                │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

```tsx
// BAD: 매 렌더마다 새 객체 생성 → GC 부담
function BadComponent({ data }) {
  return (
    <View style={{ padding: 16, margin: 8 }}>  {/* 매번 새 객체! */}
      <FlatList
        data={data}
        renderItem={({ item }) => (
          <Text style={{ color: 'blue', fontSize: 16 }}>{item.name}</Text>
          /* 매 아이템마다 새 스타일 객체! */
        )}
      />
    </View>
  );
}

// GOOD: StyleSheet.create로 객체 재사용 → GC 부담 최소화
const styles = StyleSheet.create({
  container: { padding: 16, margin: 8 },
  itemText: { color: 'blue', fontSize: 16 },
});

function GoodComponent({ data }) {
  const renderItem = useCallback(({ item }) => (
    <Text style={styles.itemText}>{item.name}</Text>
  ), []);

  return (
    <View style={styles.container}>
      <FlatList data={data} renderItem={renderItem} />
    </View>
  );
}
```

---

## 요약: JSI + Fabric + Hermes의 시너지

```
┌────────────────────────────────────────────────────────────────┐
│              3가지 기술의 시너지 효과                             │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Hermes V1                                                      │
│  ┌──────────────────────┐                                      │
│  │ JS를 빠르게 실행     │                                      │
│  │ 바이트코드 사전 컴파일│                                      │
│  │ 효율적 메모리 관리   │                                      │
│  └──────────┬───────────┘                                      │
│             │ JSI (C++ 직접 통신)                                │
│             ▼                                                   │
│  ┌──────────────────────┐                                      │
│  │ JSI Layer            │                                      │
│  │ 직렬화 없이 직접 통신│                                      │
│  │ 동기 호출 가능       │                                      │
│  │ 메모리 공유          │                                      │
│  └──────────┬───────────┘                                      │
│             │                                                   │
│             ▼                                                   │
│  ┌──────────────────────┐                                      │
│  │ Fabric Renderer      │                                      │
│  │ 효율적 트리 비교     │                                      │
│  │ Yoga 레이아웃        │                                      │
│  │ 동시성 렌더링        │                                      │
│  │ 최소한의 네이티브 조작│                                      │
│  └──────────────────────┘                                      │
│                                                                 │
│  결합 효과:                                                     │
│  • 앱 시작: Hermes AOT로 빠른 부팅                              │
│  • 인터랙션: JSI로 즉각 반응                                    │
│  • 렌더링: Fabric으로 최소 업데이트                              │
│  • 전체: 네이티브 앱에 근접한 성능                               │
│                                                                 │
│  Android 개발자 관점 요약:                                       │
│  Hermes = ART (런타임 엔진)                                     │
│  JSI = JNI (네이티브 인터페이스)                                 │
│  Fabric = Compose Renderer (선언적 UI 렌더링 엔진)              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
