# 애니메이션 완전 가이드 — Android Animation에서 Reanimated로

## 목차
1. [React Native 애니메이션 옵션](#1-react-native-애니메이션-옵션)
2. [Animated API (빌트인)](#2-animated-api-빌트인)
3. [react-native-reanimated v3](#3-react-native-reanimated-v3)
4. [제스처 + 애니메이션](#4-제스처--애니메이션)
5. [레이아웃 애니메이션](#5-레이아웃-애니메이션)
6. [실전 애니메이션 예제](#6-실전-애니메이션-예제)
7. [Android 비교 정리](#7-android-비교-정리)
8. [성능 최적화](#8-성능-최적화)

---

## 1. React Native 애니메이션 옵션

```
[옵션 비교]

도구                  난이도   성능     사용 케이스
─────────────────────────────────────────────────────
Animated API          ★☆☆    보통    간단한 애니메이션 (페이드, 이동)
LayoutAnimation       ★☆☆    좋음    레이아웃 변경 전환
react-native-reanimated ★★☆  최고    복잡한 애니메이션, 제스처 기반
Lottie               ★☆☆    좋음    디자이너가 만든 복잡한 벡터 애니메이션

[권장]
- 간단한 애니메이션: Animated API로 시작
- 대부분의 경우: react-native-reanimated (프로덕션 표준)
- 복잡한 벡터 애니메이션: Lottie

[Android 대응]
Animated API         → ValueAnimator / ObjectAnimator
LayoutAnimation      → android:animateLayoutChanges="true"
Reanimated           → MotionLayout / Compose Animation
Lottie               → Lottie (동일)
```

---

## 2. Animated API (빌트인)

React Native에 기본 포함된 애니메이션 API다. Android의 `ValueAnimator`/`ObjectAnimator`에 해당한다.

### 2.1 Animated.Value

```typescript
// Android의 ValueAnimator에 해당
import { Animated } from 'react-native';

function FadeInView() {
  // ValueAnimator animator = ValueAnimator.ofFloat(0f, 1f);
  const fadeAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    // animator.setDuration(1000);
    // animator.start();
    Animated.timing(fadeAnim, {
      toValue: 1,
      duration: 1000,
      useNativeDriver: true, // 네이티브 드라이버 사용 (성능!)
    }).start();
  }, []);

  return (
    <Animated.View style={{ opacity: fadeAnim }}>
      <Text>서서히 나타나는 텍스트</Text>
    </Animated.View>
  );
}
```

### 2.2 애니메이션 드라이버

```typescript
// timing: 일정한 시간 동안 애니메이션 (Duration-based)
// Android의 ValueAnimator.ofFloat()에 해당
Animated.timing(value, {
  toValue: 1,
  duration: 300,         // 밀리초
  easing: Easing.ease,   // 이징 함수
  useNativeDriver: true,
}).start();

// spring: 스프링 물리 기반 애니메이션
// Android의 SpringAnimation에 해당
Animated.spring(value, {
  toValue: 1,
  tension: 40,      // 스프링 강도 (높을수록 빠르게 진동)
  friction: 7,      // 마찰 (높을수록 빨리 멈춤)
  useNativeDriver: true,
}).start();

// decay: 관성 기반 감속 (스크롤 관성과 유사)
// Android의 FlingAnimation에 해당
Animated.decay(value, {
  velocity: 0.5,        // 초기 속도
  deceleration: 0.997,  // 감속 비율
  useNativeDriver: true,
}).start();
```

### 2.3 조합 애니메이션

```typescript
// parallel: 동시 실행 (Android의 AnimatorSet.playTogether())
Animated.parallel([
  Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.spring(scale, { toValue: 1, useNativeDriver: true }),
]).start();

// sequence: 순서대로 실행 (Android의 AnimatorSet.playSequentially())
Animated.sequence([
  Animated.timing(opacity, { toValue: 1, duration: 200, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 300, useNativeDriver: true }),
]).start();

// stagger: 일정 간격으로 순차 실행
// 리스트 아이템이 하나씩 나타나는 효과
Animated.stagger(100, // 100ms 간격
  items.map((_, i) =>
    Animated.timing(itemAnims[i], {
      toValue: 1,
      duration: 300,
      useNativeDriver: true,
    })
  )
).start();
```

### 2.4 Interpolation (보간)

```typescript
// Android의 ArgbEvaluator, PathInterpolator에 해당

const scrollY = useRef(new Animated.Value(0)).current;

// 스크롤 위치에 따라 헤더 높이 변경
const headerHeight = scrollY.interpolate({
  inputRange: [0, 200],       // 스크롤 0~200
  outputRange: [300, 60],     // 헤더 높이 300→60
  extrapolate: 'clamp',       // 범위 밖에서는 고정
});

// 스크롤 위치에 따라 투명도 변경
const headerOpacity = scrollY.interpolate({
  inputRange: [0, 100, 200],
  outputRange: [1, 0.5, 0],   // 여러 구간 설정 가능
});

// 색상 보간 (ArgbEvaluator에 해당)
const backgroundColor = scrollY.interpolate({
  inputRange: [0, 200],
  outputRange: ['rgb(255, 255, 255)', 'rgb(0, 122, 255)'],
});

// 회전 보간
const rotation = scrollY.interpolate({
  inputRange: [0, 360],
  outputRange: ['0deg', '360deg'],
});

return (
  <Animated.View
    style={{
      height: headerHeight,
      opacity: headerOpacity,
      backgroundColor,
      transform: [{ rotate: rotation }],
    }}
  />
);
```

---

## 3. react-native-reanimated v3

프로덕션 앱에서 사용하는 표준 애니메이션 라이브러리다. 핵심 차이점은 **UI 스레드에서 직접 실행**된다는 것이다.

### 3.1 설치 및 설정

```bash
npm install react-native-reanimated
```

```javascript
// babel.config.js — 플러그인은 반드시 마지막에!
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    // 다른 플러그인들...
    'react-native-reanimated/plugin', // 반드시 마지막
  ],
};
```

### 3.2 useSharedValue

Android의 `ValueAnimator` 또는 Compose의 `animateFloatAsState`에 해당한다.

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
} from 'react-native-reanimated';

function AnimatedBox() {
  // useSharedValue: JS 스레드와 UI 스레드가 공유하는 값
  // Android의 ValueAnimator와 달리, UI 스레드에서 직접 접근 가능
  const offset = useSharedValue(0);
  const opacity = useSharedValue(1);

  // useAnimatedStyle: 공유 값에 반응하는 스타일
  // Compose의 animate*AsState + derivedStateOf에 해당
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: offset.value }],
    opacity: opacity.value,
  }));

  const moveRight = () => {
    // withTiming: 시간 기반 애니메이션 (ValueAnimator)
    offset.value = withTiming(200, { duration: 500 });
  };

  const moveLeft = () => {
    // withSpring: 스프링 기반 애니메이션 (SpringAnimation)
    offset.value = withSpring(0, {
      damping: 15,        // 감쇠 (높을수록 빨리 멈춤)
      stiffness: 150,     // 강성 (높을수록 빠르게 이동)
      mass: 1,            // 질량
    });
  };

  return (
    <View>
      <Animated.View style={[styles.box, animatedStyle]} />
      <Button title="오른쪽" onPress={moveRight} />
      <Button title="왼쪽" onPress={moveLeft} />
    </View>
  );
}
```

### 3.3 애니메이션 조합

```typescript
import {
  withSequence,
  withDelay,
  withRepeat,
  withTiming,
  withSpring,
} from 'react-native-reanimated';

// 순차 실행 (AnimatorSet.playSequentially)
const bounce = () => {
  scale.value = withSequence(
    withTiming(1.2, { duration: 150 }),  // 커지기
    withSpring(1, { damping: 4 }),       // 스프링으로 원래 크기
  );
};

// 지연 후 실행 (AnimatorSet.setStartDelay)
const delayedFade = () => {
  opacity.value = withDelay(
    500,  // 500ms 후
    withTiming(1, { duration: 300 }),
  );
};

// 반복 (ValueAnimator.setRepeatCount / setRepeatMode)
const pulse = () => {
  scale.value = withRepeat(
    withSequence(
      withTiming(1.1, { duration: 500 }),
      withTiming(1.0, { duration: 500 }),
    ),
    -1,    // -1 = 무한 반복
    false, // reverse: false = 항상 처음부터
  );
};

// 실전 예: 흔들기 애니메이션 (에러 시 입력 필드 흔들기)
const shake = () => {
  translateX.value = withSequence(
    withTiming(-10, { duration: 50 }),
    withTiming(10, { duration: 50 }),
    withTiming(-10, { duration: 50 }),
    withTiming(10, { duration: 50 }),
    withTiming(0, { duration: 50 }),
  );
};
```

### 3.4 Interpolation

```typescript
import { interpolate, Extrapolation } from 'react-native-reanimated';

function ParallaxHeader() {
  const scrollY = useSharedValue(0);

  const headerStyle = useAnimatedStyle(() => ({
    height: interpolate(
      scrollY.value,
      [0, 200],      // 입력 범위
      [300, 60],     // 출력 범위
      Extrapolation.CLAMP  // 범위 밖: 고정
    ),
    opacity: interpolate(
      scrollY.value,
      [0, 100, 200],
      [1, 0.5, 0],
      Extrapolation.CLAMP
    ),
  }));

  // interpolateColor: 색상 보간
  const backgroundStyle = useAnimatedStyle(() => ({
    backgroundColor: interpolateColor(
      scrollY.value,
      [0, 200],
      ['#FFFFFF', '#007AFF']
    ),
  }));

  return (
    <View>
      <Animated.View style={[styles.header, headerStyle, backgroundStyle]} />
      <Animated.ScrollView
        onScroll={useAnimatedScrollHandler({
          onScroll: (event) => {
            scrollY.value = event.contentOffset.y;
          },
        })}
        scrollEventThrottle={16}
      >
        {/* 콘텐츠 */}
      </Animated.ScrollView>
    </View>
  );
}
```

```javascript [playground]
// 🧪 애니메이션 보간(Interpolation) 실습

// 1) Easing 함수들 — 애니메이션의 가속/감속 곡선
const easing = {
  linear: t => t,
  easeIn: t => t * t,
  easeOut: t => t * (2 - t),
  easeInOut: t => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t,
  bounce: t => {
    if (t < 1/2.75) return 7.5625 * t * t;
    if (t < 2/2.75) return 7.5625 * (t -= 1.5/2.75) * t + 0.75;
    if (t < 2.5/2.75) return 7.5625 * (t -= 2.25/2.75) * t + 0.9375;
    return 7.5625 * (t -= 2.625/2.75) * t + 0.984375;
  },
};

// 2) Interpolation — 값 범위 변환 (Animated.interpolate 핵심)
function interpolate(value, inputRange, outputRange) {
  // 입력 범위에서의 위치 찾기
  let i = 0;
  while (i < inputRange.length - 1 && value > inputRange[i + 1]) i++;
  i = Math.min(i, inputRange.length - 2);

  // 선형 보간
  const ratio = (value - inputRange[i]) / (inputRange[i + 1] - inputRange[i]);
  return outputRange[i] + ratio * (outputRange[i + 1] - outputRange[i]);
}

// 스크롤에 따른 헤더 opacity 변환
console.log("=== 스크롤 → 헤더 투명도 ===");
[0, 50, 100, 150, 200].forEach(scroll => {
  const opacity = interpolate(scroll, [0, 100, 200], [1, 1, 0]);
  console.log(`scroll=${scroll}px → opacity=${opacity.toFixed(2)}`);
});

// 3) 다양한 Easing 비교 (10단계)
console.log("\n=== Easing 비교 (0→1 진행) ===");
const steps = 10;
for (const [name, fn] of Object.entries(easing)) {
  const values = [];
  for (let i = 0; i <= steps; i++) {
    values.push(fn(i / steps).toFixed(2));
  }
  console.log(`${name.padEnd(10)}: ${values.join(' → ')}`);
}

// 4) 스프링 애니메이션 시뮬레이션
console.log("\n=== 스프링 애니메이션 ===");
let position = 0, velocity = 0;
const target = 100, stiffness = 0.1, damping = 0.7;

for (let frame = 0; frame < 20; frame++) {
  const force = (target - position) * stiffness;
  velocity = (velocity + force) * damping;
  position += velocity;
  if (frame % 3 === 0) {
    const bar = '█'.repeat(Math.round(position / 5));
    console.log(`f${String(frame).padStart(2)}: ${position.toFixed(1).padStart(6)} ${bar}`);
  }
}
```

```exercise
type: code-arrange
question: "Reanimated의 useSharedValue + useAnimatedStyle 패턴을 조립하세요"
tokens:
  - "const offset = useSharedValue(0)"
  - "const animatedStyle = useAnimatedStyle(() => ({"
  - "transform: [{ translateX:"
  - "withSpring(offset.value)"
  - "}]"
  - "}))"
distractors:
  - "useState(0)"
  - "Animated.Value"
  - "setNativeProps"
answer: ["const offset = useSharedValue(0)", "const animatedStyle = useAnimatedStyle(() => ({", "transform: [{ translateX:", "withSpring(offset.value)", "}]", "}))"]
hint: "Reanimated는 useSharedValue로 값을 생성하고 useAnimatedStyle로 스타일에 바인딩합니다"
xp: 8
```

---

## 4. 제스처 + 애니메이션

`react-native-gesture-handler`와 `react-native-reanimated`를 결합하면 Android의 `MotionLayout` + `GestureDetector`에 해당하는 인터랙션을 만들 수 있다.

### 4.1 드래그 가능한 카드

```typescript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

function DraggableCard() {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const scale = useSharedValue(1);

  const panGesture = Gesture.Pan()
    .onBegin(() => {
      // 드래그 시작: 카드를 약간 키움
      scale.value = withSpring(1.05);
    })
    .onUpdate((event) => {
      // 드래그 중: 손가락 따라 이동
      translateX.value = event.translationX;
      translateY.value = event.translationY;
    })
    .onEnd(() => {
      // 드래그 종료: 원위치로 복귀
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
      scale.value = withSpring(1);
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { scale: scale.value },
    ],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>드래그해보세요</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 4.2 핀치 투 줌

```typescript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

function PinchableImage({ imageUrl }: { imageUrl: string }) {
  const scale = useSharedValue(1);
  const savedScale = useSharedValue(1);

  const pinchGesture = Gesture.Pinch()
    .onUpdate((event) => {
      scale.value = savedScale.value * event.scale;
    })
    .onEnd(() => {
      // 최소/최대 스케일 제한
      if (scale.value < 1) {
        scale.value = withSpring(1);
        savedScale.value = 1;
      } else if (scale.value > 3) {
        scale.value = withSpring(3);
        savedScale.value = 3;
      } else {
        savedScale.value = scale.value;
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));

  return (
    <GestureDetector gesture={pinchGesture}>
      <Animated.Image
        source={{ uri: imageUrl }}
        style={[{ width: '100%', height: 300 }, animatedStyle]}
      />
    </GestureDetector>
  );
}
```

### 4.3 바텀 시트 (제스처 + 스프링 애니메이션)

```typescript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
  runOnJS,
} from 'react-native-reanimated';
import { Dimensions } from 'react-native';

const { height: SCREEN_HEIGHT } = Dimensions.get('window');
const SNAP_POINTS = {
  TOP: -SCREEN_HEIGHT * 0.7,    // 화면의 70%
  MIDDLE: -SCREEN_HEIGHT * 0.4, // 화면의 40%
  BOTTOM: 0,                     // 숨김 상태
};

function BottomSheet({ children }: { children: React.ReactNode }) {
  const translateY = useSharedValue(0);
  const context = useSharedValue({ y: 0 });

  const panGesture = Gesture.Pan()
    .onStart(() => {
      context.value = { y: translateY.value };
    })
    .onUpdate((event) => {
      translateY.value = Math.max(
        event.translationY + context.value.y,
        SNAP_POINTS.TOP
      );
    })
    .onEnd((event) => {
      // 속도와 위치에 따라 스냅 포인트 결정
      const velocity = event.velocityY;

      if (velocity > 500) {
        // 빠르게 아래로 스와이프: 닫기
        translateY.value = withSpring(SNAP_POINTS.BOTTOM, { damping: 20 });
      } else if (velocity < -500) {
        // 빠르게 위로 스와이프: 최대로 열기
        translateY.value = withSpring(SNAP_POINTS.TOP, { damping: 20 });
      } else {
        // 가장 가까운 스냅 포인트로
        const distances = Object.values(SNAP_POINTS).map(
          (point) => Math.abs(translateY.value - point)
        );
        const closest = Object.values(SNAP_POINTS)[
          distances.indexOf(Math.min(...distances))
        ];
        translateY.value = withSpring(closest, { damping: 20 });
      }
    });

  const sheetStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.bottomSheet, sheetStyle]}>
        {/* 드래그 핸들 */}
        <View style={styles.handle}>
          <View style={styles.handleBar} />
        </View>
        {children}
      </Animated.View>
    </GestureDetector>
  );
}

const styles = StyleSheet.create({
  bottomSheet: {
    position: 'absolute',
    top: SCREEN_HEIGHT,
    left: 0,
    right: 0,
    height: SCREEN_HEIGHT,
    backgroundColor: '#fff',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -4 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 5,
  },
  handle: { alignItems: 'center', paddingVertical: 12 },
  handleBar: {
    width: 40, height: 4, borderRadius: 2, backgroundColor: '#DDD',
  },
});
```

---

## 5. 레이아웃 애니메이션

### 5.1 Entering/Exiting 애니메이션

컴포넌트가 화면에 나타나거나 사라질 때의 애니메이션이다. Android의 `android:animateLayoutChanges="true"` 또는 Compose의 `AnimatedVisibility`에 해당한다.

```typescript
import Animated, {
  FadeIn,
  FadeOut,
  SlideInLeft,
  SlideOutRight,
  BounceIn,
  ZoomIn,
  FlipInXUp,
  FadeInDown,
  FadeOutUp,
  Layout,
} from 'react-native-reanimated';

function AnimatedList() {
  const [items, setItems] = useState<string[]>([]);

  const addItem = () => {
    setItems([...items, `Item ${items.length + 1}`]);
  };

  const removeItem = (index: number) => {
    setItems(items.filter((_, i) => i !== index));
  };

  return (
    <View>
      <Button title="추가" onPress={addItem} />
      {items.map((item, index) => (
        <Animated.View
          key={item}
          // entering: 나타날 때 (AnimatedVisibility enter)
          entering={FadeInDown.delay(index * 100).springify()}
          // exiting: 사라질 때 (AnimatedVisibility exit)
          exiting={FadeOutUp.duration(300)}
          // layout: 위치가 변경될 때 (LayoutAnimation)
          layout={Layout.springify()}
          style={styles.item}
        >
          <Text>{item}</Text>
          <TouchableOpacity onPress={() => removeItem(index)}>
            <Text>삭제</Text>
          </TouchableOpacity>
        </Animated.View>
      ))}
    </View>
  );
}
```

### 사용 가능한 entering/exiting 애니메이션

```typescript
// Entering (나타나기) 프리셋:
FadeIn           // 페이드 인
FadeInDown       // 아래에서 페이드 인
FadeInUp         // 위에서 페이드 인
FadeInLeft       // 왼쪽에서 페이드 인
FadeInRight      // 오른쪽에서 페이드 인
SlideInLeft      // 왼쪽에서 슬라이드
SlideInRight     // 오른쪽에서 슬라이드
SlideInDown      // 위에서 슬라이드
SlideInUp        // 아래에서 슬라이드
BounceIn         // 바운스 효과
ZoomIn           // 확대되며 나타남
FlipInXUp        // X축 뒤집기
FlipInYLeft      // Y축 뒤집기
RotateInDownLeft // 회전하며 나타남
StretchInX       // X축으로 늘어나며 나타남

// 커스텀 설정
FadeInDown
  .delay(200)           // 200ms 지연
  .duration(500)        // 500ms 동안
  .springify()          // 스프링 물리 적용
  .damping(15)          // 스프링 감쇠
  .withInitialValues({ transform: [{ translateY: 100 }] })  // 시작 위치

// Exiting (사라지기) 프리셋: Entering의 반대
FadeOut, FadeOutDown, FadeOutUp, ...
SlideOutLeft, SlideOutRight, ...
BounceOut, ZoomOut, ...
```

### 5.2 Keyframe 애니메이션

Android의 `AnimationSet`과 유사하게, 여러 시간대의 상태를 정의한다.

```typescript
import { Keyframe } from 'react-native-reanimated';

// 복잡한 다단계 애니메이션 정의
const bounceKeyframe = new Keyframe({
  0: {
    transform: [{ scale: 0 }],
    opacity: 0,
  },
  30: {
    transform: [{ scale: 1.2 }],
    opacity: 1,
  },
  50: {
    transform: [{ scale: 0.9 }],
  },
  70: {
    transform: [{ scale: 1.1 }],
  },
  100: {
    transform: [{ scale: 1 }],
  },
}).duration(800);

function AnimatedBadge() {
  return (
    <Animated.View entering={bounceKeyframe} style={styles.badge}>
      <Text style={styles.badgeText}>NEW</Text>
    </Animated.View>
  );
}
```

---

## 6. 실전 애니메이션 예제

### 6.1 버튼 프레스 효과 (Material Ripple 대응)

```typescript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
} from 'react-native-reanimated';

interface AnimatedButtonProps {
  title: string;
  onPress: () => void;
}

function AnimatedButton({ title, onPress }: AnimatedButtonProps) {
  const scale = useSharedValue(1);
  const opacity = useSharedValue(1);

  const tapGesture = Gesture.Tap()
    .onBegin(() => {
      // 눌림 시작: 축소 + 약간 투명
      scale.value = withTiming(0.95, { duration: 100 });
      opacity.value = withTiming(0.8, { duration: 100 });
    })
    .onFinalize(() => {
      // 눌림 종료: 원래 크기로 복원 (스프링)
      scale.value = withSpring(1, { damping: 15, stiffness: 150 });
      opacity.value = withTiming(1, { duration: 100 });
    })
    .onEnd(() => {
      runOnJS(onPress)();
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    opacity: opacity.value,
  }));

  return (
    <GestureDetector gesture={tapGesture}>
      <Animated.View style={[styles.button, animatedStyle]}>
        <Text style={styles.buttonText}>{title}</Text>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 6.2 스와이프 투 딜리트

```typescript
function SwipeToDelete({
  children,
  onDelete,
}: {
  children: React.ReactNode;
  onDelete: () => void;
}) {
  const translateX = useSharedValue(0);
  const itemHeight = useSharedValue(70);
  const marginBottom = useSharedValue(8);
  const opacity = useSharedValue(1);

  const panGesture = Gesture.Pan()
    .activeOffsetX([-20, 20]) // 수평 20px 이상 움직여야 활성화
    .onUpdate((event) => {
      translateX.value = Math.min(0, event.translationX); // 왼쪽으로만
    })
    .onEnd((event) => {
      const shouldDelete = translateX.value < -100;

      if (shouldDelete) {
        // 삭제: 밀려나간 후 높이 축소
        translateX.value = withTiming(-500, { duration: 200 });
        itemHeight.value = withTiming(0, { duration: 300 });
        marginBottom.value = withTiming(0, { duration: 300 });
        opacity.value = withTiming(0, { duration: 200 }, () => {
          runOnJS(onDelete)();
        });
      } else {
        // 복원
        translateX.value = withSpring(0);
      }
    });

  const containerStyle = useAnimatedStyle(() => ({
    height: itemHeight.value,
    marginBottom: marginBottom.value,
    opacity: opacity.value,
  }));

  const contentStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  const deleteIconOpacity = useAnimatedStyle(() => ({
    opacity: interpolate(
      translateX.value,
      [-100, 0],
      [1, 0],
      Extrapolation.CLAMP
    ),
  }));

  return (
    <Animated.View style={containerStyle}>
      {/* 삭제 배경 */}
      <Animated.View style={[styles.deleteBackground, deleteIconOpacity]}>
        <Text style={styles.deleteText}>삭제</Text>
      </Animated.View>
      {/* 콘텐츠 */}
      <GestureDetector gesture={panGesture}>
        <Animated.View style={[styles.swipeContent, contentStyle]}>
          {children}
        </Animated.View>
      </GestureDetector>
    </Animated.View>
  );
}
```

### 6.3 패럴랙스 스크롤 헤더

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  useAnimatedScrollHandler,
  interpolate,
  Extrapolation,
} from 'react-native-reanimated';

const HEADER_HEIGHT = 300;
const COLLAPSED_HEIGHT = 60;

function ParallaxScrollView() {
  const scrollY = useSharedValue(0);

  const scrollHandler = useAnimatedScrollHandler({
    onScroll: (event) => {
      scrollY.value = event.contentOffset.y;
    },
  });

  // 헤더 이미지: 패럴랙스 효과 (0.5배 속도로 스크롤)
  const imageStyle = useAnimatedStyle(() => ({
    transform: [
      {
        translateY: interpolate(
          scrollY.value,
          [-HEADER_HEIGHT, 0, HEADER_HEIGHT],
          [-HEADER_HEIGHT / 2, 0, HEADER_HEIGHT * 0.75],
        ),
      },
      {
        scale: interpolate(
          scrollY.value,
          [-HEADER_HEIGHT, 0],
          [2, 1],
          Extrapolation.CLAMP,
        ),
      },
    ],
  }));

  // 헤더 오버레이: 스크롤 시 어두워짐
  const overlayStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      scrollY.value,
      [0, HEADER_HEIGHT - COLLAPSED_HEIGHT],
      [0, 0.7],
      Extrapolation.CLAMP,
    ),
  }));

  // 헤더 제목: 스크롤 시 나타남
  const titleStyle = useAnimatedStyle(() => ({
    opacity: interpolate(
      scrollY.value,
      [HEADER_HEIGHT - 100, HEADER_HEIGHT - COLLAPSED_HEIGHT],
      [0, 1],
      Extrapolation.CLAMP,
    ),
  }));

  return (
    <View style={{ flex: 1 }}>
      {/* 고정 헤더 이미지 */}
      <Animated.Image
        source={{ uri: 'https://picsum.photos/800/600' }}
        style={[{
          position: 'absolute', top: 0, left: 0, right: 0,
          height: HEADER_HEIGHT,
        }, imageStyle]}
      />

      {/* 어두운 오버레이 */}
      <Animated.View
        style={[{
          position: 'absolute', top: 0, left: 0, right: 0,
          height: COLLAPSED_HEIGHT, backgroundColor: '#000', zIndex: 1,
        }, overlayStyle]}
      />

      {/* 축소된 제목 */}
      <Animated.Text
        style={[{
          position: 'absolute', top: 20, left: 16, zIndex: 2,
          color: '#fff', fontSize: 18, fontWeight: 'bold',
        }, titleStyle]}
      >
        기사 제목
      </Animated.Text>

      {/* 스크롤 콘텐츠 */}
      <Animated.ScrollView
        onScroll={scrollHandler}
        scrollEventThrottle={16}
        contentContainerStyle={{ paddingTop: HEADER_HEIGHT }}
      >
        <View style={{ backgroundColor: '#fff', padding: 16 }}>
          <Text style={{ fontSize: 24, fontWeight: 'bold' }}>기사 제목</Text>
          <Text style={{ fontSize: 16, marginTop: 12, lineHeight: 24 }}>
            기사 본문 내용이 여기에 들어갑니다...
          </Text>
        </View>
      </Animated.ScrollView>
    </View>
  );
}
```

### 6.4 로딩 스켈레톤 (Shimmer 효과)

```typescript
import { useEffect } from 'react';
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withRepeat,
  withTiming,
  interpolate,
} from 'react-native-reanimated';
import { LinearGradient } from 'expo-linear-gradient';

function SkeletonLoader({ width, height }: { width: number; height: number }) {
  const shimmer = useSharedValue(0);

  useEffect(() => {
    shimmer.value = withRepeat(
      withTiming(1, { duration: 1500 }),
      -1,    // 무한 반복
      false, // 되감지 않음
    );
  }, []);

  const shimmerStyle = useAnimatedStyle(() => ({
    transform: [
      {
        translateX: interpolate(
          shimmer.value,
          [0, 1],
          [-width, width],
        ),
      },
    ],
  }));

  return (
    <View
      style={{
        width, height,
        backgroundColor: '#E0E0E0',
        borderRadius: 8,
        overflow: 'hidden',
      }}
    >
      <Animated.View style={[{ width, height }, shimmerStyle]}>
        <LinearGradient
          colors={['#E0E0E0', '#F0F0F0', '#E0E0E0']}
          start={{ x: 0, y: 0 }}
          end={{ x: 1, y: 0 }}
          style={{ width, height }}
        />
      </Animated.View>
    </View>
  );
}

// 사용: 카드 형태의 스켈레톤
function ArticleCardSkeleton() {
  return (
    <View style={{ padding: 16 }}>
      <SkeletonLoader width={300} height={200} />
      <View style={{ marginTop: 12 }}>
        <SkeletonLoader width={250} height={20} />
      </View>
      <View style={{ marginTop: 8 }}>
        <SkeletonLoader width={180} height={16} />
      </View>
    </View>
  );
}
```

### 6.5 Lottie 애니메이션 통합

```bash
npm install lottie-react-native
```

```typescript
// Lottie — Android/iOS 동일 라이브러리
import LottieView from 'lottie-react-native';

function SuccessAnimation({ onFinish }: { onFinish: () => void }) {
  return (
    <LottieView
      source={require('../assets/animations/success.json')}
      autoPlay
      loop={false}
      onAnimationFinish={onFinish}
      style={{ width: 200, height: 200 }}
    />
  );
}

// 로딩 애니메이션
function LoadingAnimation() {
  return (
    <LottieView
      source={require('../assets/animations/loading.json')}
      autoPlay
      loop
      style={{ width: 100, height: 100 }}
    />
  );
}

// 프로그래밍 방식 제어
function ControlledAnimation() {
  const animationRef = useRef<LottieView>(null);

  const play = () => animationRef.current?.play();
  const pause = () => animationRef.current?.pause();
  const reset = () => animationRef.current?.reset();

  return (
    <View>
      <LottieView
        ref={animationRef}
        source={require('../assets/animations/heart.json')}
        style={{ width: 60, height: 60 }}
      />
      <Button title="재생" onPress={play} />
    </View>
  );
}
```

---

## 7. Android 비교 정리

```
[Android Animation → React Native 대응]

Android                              React Native
────────────────────────────────────────────────────────
ValueAnimator                       → useSharedValue + withTiming
ObjectAnimator                      → useAnimatedStyle
PropertyAnimation                   → withTiming / withSpring
SpringAnimation                     → withSpring
FlingAnimation                      → withDecay
AnimatorSet.playTogether()          → 여러 값을 동시에 변경
AnimatorSet.playSequentially()      → withSequence
AnimatorSet.setStartDelay()         → withDelay
ValueAnimator.setRepeatCount()      → withRepeat
AccelerateDecelerateInterpolator    → Easing.inOut(Easing.ease)
OvershootInterpolator              → withSpring (약한 damping)
BounceInterpolator                 → withSpring (매우 약한 damping)
ArgbEvaluator                      → interpolateColor
PathInterpolator                    → interpolate with Extrapolation
MotionLayout                       → Reanimated Layout Transitions
android:animateLayoutChanges       → LayoutAnimation / Layout
AnimatedVisibility (Compose)       → entering / exiting 프리셋
Transition Framework               → Layout animations
GestureDetector + Animation        → react-native-gesture-handler + reanimated
RecyclerView ItemAnimator          → FadeInDown.delay(index * 100)
OvershootInterpolator              → withSpring({ damping: 5 })
Lottie                             → lottie-react-native (동일)
```

---

## 8. 성능 최적화

### UI 스레드에서 실행

```typescript
// Reanimated의 핵심 장점: 애니메이션이 UI 스레드에서 실행됨

// ❌ JS 스레드 애니메이션 (Animated API, useNativeDriver: false)
// JS 스레드가 바쁘면 애니메이션이 끊김

// ✅ UI 스레드 애니메이션 (Reanimated)
// JS 스레드와 독립적으로 60fps 유지
const animatedStyle = useAnimatedStyle(() => {
  // 이 코드는 UI 스레드(worklet)에서 실행됨!
  return {
    transform: [{ translateX: offset.value }],
  };
});
```

### useNativeDriver

```typescript
// Animated API 사용 시 반드시 useNativeDriver: true 사용
// 지원 속성: transform, opacity
// 미지원 속성: width, height, backgroundColor, margin, padding 등

// ✅ 네이티브 드라이버 사용 (빠름)
Animated.timing(translateX, {
  toValue: 100,
  duration: 300,
  useNativeDriver: true,  // transform, opacity만 가능
}).start();

// ❌ JS 드라이버 (느림, 끊김 가능)
Animated.timing(width, {
  toValue: 200,
  duration: 300,
  useNativeDriver: false,  // width는 네이티브 드라이버 미지원
}).start();

// 이런 이유로 Reanimated를 권장:
// Reanimated는 모든 속성을 UI 스레드에서 애니메이션 가능
```

### 애니메이션 최적화 팁

```typescript
// 1. 스크롤 이벤트 처리 시 scrollEventThrottle 설정
<Animated.ScrollView
  scrollEventThrottle={16} // 16ms = 60fps
  onScroll={scrollHandler}
/>

// 2. 불필요한 리렌더링 방지: 애니메이션 컴포넌트를 분리
// ❌ 상태 변경이 애니메이션 컴포넌트까지 리렌더링
function BadComponent() {
  const [text, setText] = useState('');
  const scale = useSharedValue(1);
  // text가 바뀔 때마다 전체 리렌더링 → 애니메이션에도 영향

  return (
    <View>
      <TextInput value={text} onChangeText={setText} />
      <AnimatedBox scale={scale} /> {/* 불필요한 리렌더링 */}
    </View>
  );
}

// ✅ 애니메이션 컴포넌트를 분리하여 격리
function GoodComponent() {
  const [text, setText] = useState('');

  return (
    <View>
      <TextInput value={text} onChangeText={setText} />
      <IsolatedAnimatedBox /> {/* text 변경에 영향 안 받음 */}
    </View>
  );
}

const IsolatedAnimatedBox = React.memo(function () {
  const scale = useSharedValue(1);
  // ...
});

// 3. runOnJS 최소화
// UI 스레드(worklet)에서 JS 함수를 호출하면 오버헤드 발생
// 꼭 필요할 때만 사용
const panGesture = Gesture.Pan()
  .onEnd((event) => {
    if (event.translationX > 100) {
      // runOnJS는 필요할 때만 (네비게이션, 상태 업데이트 등)
      runOnJS(handleSwipeComplete)();
    }
  });
```
