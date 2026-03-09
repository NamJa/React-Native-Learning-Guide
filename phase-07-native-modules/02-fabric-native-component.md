# Fabric Native Component — 커스텀 Android View를 React Native에서 사용하기

## 목차
1. [Fabric Native Component가 필요한 경우](#1-fabric-native-component가-필요한-경우)
2. [Fabric vs Legacy ViewManager](#2-fabric-vs-legacy-viewmanager)
3. [단계별 가이드](#3-단계별-가이드)
4. [예제 1: Custom ProgressBar (Props + 애니메이션)](#4-예제-1-custom-progressbar)
5. [예제 2: Native VideoPlayer (Props + Events + Commands)](#5-예제-2-native-videoplayer)
6. [기존 Android 라이브러리 뷰 래핑](#6-기존-android-라이브러리-뷰-래핑)
7. [성능 고려사항](#7-성능-고려사항)

---

## 1. Fabric Native Component가 필요한 경우

```
┌────────────────────────────────────────────────────────────────┐
│         Fabric Native Component가 필요한 경우                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JS 컴포넌트로 충분한 경우:                                      │
│  ✓ 일반 UI (버튼, 텍스트, 입력 필드)                             │
│  ✓ 레이아웃, 스크롤뷰, FlatList                                  │
│  ✓ 애니메이션 (Reanimated로 대부분 가능)                         │
│  ✓ SVG 그래픽                                                   │
│                                                                 │
│  Native Component가 필요한 경우:                                 │
│  ✗ MapView (Google Maps, Naver Maps)                            │
│  ✗ VideoPlayer (ExoPlayer, MediaPlayer)                         │
│  ✗ Custom Canvas 드로잉                                         │
│  ✗ WebView (네이티브 브라우저 엔진)                              │
│  ✗ Camera Preview                                               │
│  ✗ PDF Viewer                                                   │
│  ✗ AR/VR 뷰                                                    │
│  ✗ 기존 Android 커스텀 View 재사용                               │
│  ✗ 네이티브 광고 SDK 뷰 (AdMob 등)                              │
│                                                                 │
│  판단 기준:                                                     │
│  "Android 네이티브 View 클래스를 직접 사용해야 하는가?"           │
│  YES → Fabric Native Component                                  │
│  NO → React Native JS 컴포넌트                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. Fabric vs Legacy ViewManager

```
┌────────────────────────────────────────────────────────────────┐
│           Legacy ViewManager vs Fabric Component                 │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Legacy ViewManager:                                            │
│  • Bridge 통해 비동기 prop 전달                                  │
│  • @ReactProp 어노테이션으로 prop 등록                           │
│  • SimpleViewManager/ViewGroupManager 상속                      │
│  • JS: requireNativeComponent('ViewName')                       │
│  • 타입 안전성 없음                                              │
│                                                                 │
│  Fabric Component:                                              │
│  • JSI 통해 직접 prop 전달                                       │
│  • Codegen이 타입 안전 인터페이스 생성                            │
│  • TypeScript Spec에서 Props/Events/Commands 정의                │
│  • JS: codegenNativeComponent('ViewName')                       │
│  • 컴파일 타임 타입 검증                                         │
│                                                                 │
├────────────────────────────────────────────────────────────────┤
│  비교표:                                                        │
│  ┌──────────────────┬──────────────┬──────────────────┐        │
│  │ 항목             │ Legacy       │ Fabric           │        │
│  ├──────────────────┼──────────────┼──────────────────┤        │
│  │ Prop 전달        │ Bridge(비동기)│ JSI(동기 가능)    │        │
│  │ Spec 파일        │ 없음         │ TypeScript Spec  │        │
│  │ 코드 생성        │ 없음         │ Codegen          │        │
│  │ 기반 클래스      │ SimpleView   │ SimpleView       │        │
│  │                  │ Manager      │ Manager (동일)   │        │
│  │ 이벤트           │ 수동 등록    │ Spec에 선언      │        │
│  │ Commands         │ 수동 dispatch│ Spec에 선언      │        │
│  │ JS 등록          │ requireNative│ codegenNative    │        │
│  │                  │ Component    │ Component        │        │
│  │ 동시성 렌더링    │ 불가         │ 지원             │        │
│  └──────────────────┴──────────────┴──────────────────┘        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. 단계별 가이드

### Step 1: TypeScript Spec 정의

```typescript
// src/specs/XxxNativeComponent.ts

import type { ViewProps } from 'react-native';
import type {
  Float,
  Int32,
  Double,
  BubblingEventHandler,
  DirectEventHandler,
} from 'react-native/Libraries/Types/CodegenTypes';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';
import codegenNativeCommands from 'react-native/Libraries/Utilities/codegenNativeCommands';

// ═══ Props 정의 ═══
interface NativeProps extends ViewProps {
  // 기본 타입
  title: string;
  count: Int32;
  ratio: Float;
  enabled: boolean;

  // Nullable
  subtitle?: string;

  // 색상 (ColorValue → Android Color int)
  color?: string; // '#RRGGBB' 또는 'red' 등

  // 이미지 소스
  // source?: ImageSource;

  // 배열
  items?: ReadonlyArray<string>;

  // ═══ 이벤트 정의 ═══
  // Direct Event: 이벤트가 해당 뷰에서만 처리 (버블링 X)
  onComplete?: DirectEventHandler<Readonly<{
    success: boolean;
    message: string;
  }>>;

  // Bubbling Event: 부모 뷰로 이벤트 전파
  onSelect?: BubblingEventHandler<Readonly<{
    index: Int32;
    value: string;
  }>>;
}

// ═══ Commands 정의 (JS → Native 명령) ═══
type ComponentType = InstanceType<
  typeof import('react-native').HostComponent<NativeProps>
>;

interface NativeCommands {
  reset: (viewRef: React.ElementRef<ComponentType>) => void;
  scrollTo: (viewRef: React.ElementRef<ComponentType>, x: Float, y: Float) => void;
}

export const Commands = codegenNativeCommands<NativeCommands>({
  supportedCommands: ['reset', 'scrollTo'],
});

// ═══ 컴포넌트 내보내기 ═══
export default codegenNativeComponent<NativeProps>('XxxView');
```

### Step 2: Kotlin ViewManager 구현

```kotlin
// android/app/src/main/java/com/myapp/XxxViewManager.kt

package com.myapp

import android.graphics.Color
import com.facebook.react.bridge.ReadableArray
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.ViewManagerDelegate
import com.facebook.react.uimanager.annotations.ReactProp
import com.facebook.react.viewmanagers.XxxViewManagerDelegate
import com.facebook.react.viewmanagers.XxxViewManagerInterface

@ReactModule(name = XxxViewManager.NAME)
class XxxViewManager : SimpleViewManager<XxxView>(),
    XxxViewManagerInterface<XxxView> {
    // ↑ Codegen이 생성한 인터페이스 구현

    companion object {
        const val NAME = "XxxView"
    }

    private val delegate = XxxViewManagerDelegate(this)

    override fun getName(): String = NAME

    override fun getDelegate(): ViewManagerDelegate<XxxView> = delegate

    override fun createViewInstance(context: ThemedReactContext): XxxView {
        return XxxView(context)
    }

    // ═══ Props 핸들러 ═══

    @ReactProp(name = "title")
    override fun setTitle(view: XxxView, title: String?) {
        view.setTitle(title ?: "")
    }

    @ReactProp(name = "count")
    override fun setCount(view: XxxView, count: Int) {
        view.setCount(count)
    }

    @ReactProp(name = "enabled", defaultBoolean = true)
    override fun setEnabled(view: XxxView, enabled: Boolean) {
        view.isEnabled = enabled
    }

    @ReactProp(name = "color")
    override fun setColor(view: XxxView, color: String?) {
        color?.let { view.setColor(Color.parseColor(it)) }
    }

    // ═══ Commands 핸들러 ═══

    override fun receiveCommand(view: XxxView, commandName: String, args: ReadableArray?) {
        delegate.receiveCommand(view, commandName, args)
    }

    // Codegen 인터페이스의 메서드 구현
    override fun reset(view: XxxView) {
        view.reset()
    }

    override fun scrollTo(view: XxxView, x: Float, y: Float) {
        view.scrollTo(x.toInt(), y.toInt())
    }
}
```

### Step 3: 패키지 등록

```kotlin
// android/app/src/main/java/com/myapp/XxxViewPackage.kt

package com.myapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager

class XxxViewPackage : ReactPackage {

    override fun createNativeModules(reactContext: ReactApplicationContext): List<NativeModule> {
        return emptyList() // ViewManager만 있으므로
    }

    override fun createViewManagers(reactContext: ReactApplicationContext): List<ViewManager<*, *>> {
        return listOf(XxxViewManager())
    }
}

// MainApplication.kt에 추가:
// add(XxxViewPackage())
```

### Step 4: React Native에서 사용

```tsx
// src/components/XxxView.tsx

import React, { useRef, useCallback } from 'react';
import NativeXxxView, { Commands } from '../specs/XxxNativeComponent';

interface XxxViewProps {
  title: string;
  count: number;
  color?: string;
  onComplete?: (event: { success: boolean; message: string }) => void;
  onSelect?: (event: { index: number; value: string }) => void;
  style?: any;
}

export default function XxxView({
  title,
  count,
  color,
  onComplete,
  onSelect,
  style,
}: XxxViewProps) {
  const viewRef = useRef<React.ElementRef<typeof NativeXxxView>>(null);

  const handleComplete = useCallback((event: any) => {
    onComplete?.(event.nativeEvent);
  }, [onComplete]);

  const handleSelect = useCallback((event: any) => {
    onSelect?.(event.nativeEvent);
  }, [onSelect]);

  // 외부에서 호출할 수 있도록 명령 노출
  const reset = useCallback(() => {
    if (viewRef.current) {
      Commands.reset(viewRef.current);
    }
  }, []);

  return (
    <NativeXxxView
      ref={viewRef}
      style={style}
      title={title}
      count={count}
      color={color}
      onComplete={handleComplete}
      onSelect={handleSelect}
    />
  );
}
```

```exercise
type: code-arrange
question: "Fabric Native Component의 TypeScript 스펙을 조립하세요"
tokens:
  - "import type { ViewProps } from 'react-native'"
  - "import codegenNativeComponent from"
  - "'react-native/Libraries/Utilities/codegenNativeComponent'"
  - "interface NativeProps extends ViewProps {"
  - "color?: string"
  - "}"
  - "export default codegenNativeComponent<NativeProps>('MyView')"
distractors:
  - "import { View } from 'react-native'"
  - "class NativeProps"
  - "createViewManager"
answer: ["import type { ViewProps } from 'react-native'", "import codegenNativeComponent from", "'react-native/Libraries/Utilities/codegenNativeComponent'", "interface NativeProps extends ViewProps {", "color?: string", "}", "export default codegenNativeComponent<NativeProps>('MyView')"]
hint: "Fabric 컴포넌트는 ViewProps를 확장하고 codegenNativeComponent로 등록합니다"
xp: 8
```

---

## 4. 예제 1: Custom ProgressBar

그래디언트 색상과 애니메이션을 지원하는 커스텀 ProgressBar.

### Spec 정의

```typescript
// src/specs/GradientProgressBarNativeComponent.ts

import type { ViewProps, ColorValue } from 'react-native';
import type {
  Float,
  DirectEventHandler,
} from 'react-native/Libraries/Types/CodegenTypes';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';
import codegenNativeCommands from 'react-native/Libraries/Utilities/codegenNativeCommands';

interface NativeProps extends ViewProps {
  // 진행률 (0.0 ~ 1.0)
  progress: Float;

  // 시작 색상 (그래디언트 왼쪽)
  startColor?: string;

  // 끝 색상 (그래디언트 오른쪽)
  endColor?: string;

  // 배경 트랙 색상
  trackColor?: string;

  // 바 높이 (dp)
  barHeight?: Float;

  // 모서리 둥글기
  cornerRadius?: Float;

  // 애니메이션 사용 여부
  animated?: boolean;

  // 애니메이션 지속 시간 (ms)
  animationDuration?: Float;

  // 진행 완료 이벤트
  onProgressComplete?: DirectEventHandler<Readonly<{
    finalProgress: Float;
  }>>;
}

type ComponentType = InstanceType<
  typeof import('react-native').HostComponent<NativeProps>
>;

interface NativeCommands {
  setProgressAnimated: (
    viewRef: React.ElementRef<ComponentType>,
    progress: Float,
    duration: Float
  ) => void;
  reset: (viewRef: React.ElementRef<ComponentType>) => void;
}

export const Commands = codegenNativeCommands<NativeCommands>({
  supportedCommands: ['setProgressAnimated', 'reset'],
});

export default codegenNativeComponent<NativeProps>('GradientProgressBar');
```

### Android View 구현

```kotlin
// android/app/src/main/java/com/myapp/GradientProgressBarView.kt

package com.myapp

import android.animation.ValueAnimator
import android.content.Context
import android.graphics.*
import android.util.AttributeSet
import android.view.View
import android.view.animation.AccelerateDecelerateInterpolator
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.ReactContext
import com.facebook.react.uimanager.events.Event
import com.facebook.react.uimanager.events.RCTEventEmitter

class GradientProgressBarView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    // 속성들
    private var progress: Float = 0f
    private var displayedProgress: Float = 0f
    private var startColor: Int = Color.parseColor("#4A90D9")
    private var endColor: Int = Color.parseColor("#67B26F")
    private var trackColor: Int = Color.parseColor("#E0E0E0")
    private var barHeight: Float = 12f * resources.displayMetrics.density
    private var cornerRadius: Float = 6f * resources.displayMetrics.density
    private var animated: Boolean = true
    private var animationDuration: Long = 300L

    // Paint 객체 (재사용)
    private val trackPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val barPaint = Paint(Paint.ANTI_ALIAS_FLAG)
    private val trackRect = RectF()
    private val barRect = RectF()

    private var progressAnimator: ValueAnimator? = null

    fun setProgress(value: Float) {
        val clampedValue = value.coerceIn(0f, 1f)
        progress = clampedValue

        if (animated) {
            animateProgress(clampedValue)
        } else {
            displayedProgress = clampedValue
            invalidate()
        }

        // 완료 이벤트 발송
        if (clampedValue >= 1f) {
            dispatchProgressComplete(clampedValue)
        }
    }

    fun setStartColor(color: Int) {
        startColor = color
        invalidate()
    }

    fun setEndColor(color: Int) {
        endColor = color
        invalidate()
    }

    fun setTrackColor(color: Int) {
        trackColor = color
        invalidate()
    }

    fun setBarHeight(height: Float) {
        barHeight = height * resources.displayMetrics.density
        requestLayout()
        invalidate()
    }

    fun setCornerRadius(radius: Float) {
        cornerRadius = radius * resources.displayMetrics.density
        invalidate()
    }

    fun setAnimated(value: Boolean) {
        animated = value
    }

    fun setAnimationDuration(duration: Float) {
        animationDuration = duration.toLong()
    }

    // ═══ Commands ═══

    fun setProgressAnimated(targetProgress: Float, duration: Float) {
        val clampedTarget = targetProgress.coerceIn(0f, 1f)
        progressAnimator?.cancel()
        progressAnimator = ValueAnimator.ofFloat(displayedProgress, clampedTarget).apply {
            this.duration = duration.toLong()
            interpolator = AccelerateDecelerateInterpolator()
            addUpdateListener { animation ->
                displayedProgress = animation.animatedValue as Float
                invalidate()
            }
            start()
        }
        progress = clampedTarget
    }

    fun reset() {
        progressAnimator?.cancel()
        progress = 0f
        displayedProgress = 0f
        invalidate()
    }

    // ═══ 그리기 ═══

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        val w = width.toFloat()
        val h = height.toFloat()
        val barTop = (h - barHeight) / 2
        val barBottom = barTop + barHeight

        // 배경 트랙
        trackPaint.color = trackColor
        trackRect.set(0f, barTop, w, barBottom)
        canvas.drawRoundRect(trackRect, cornerRadius, cornerRadius, trackPaint)

        // 진행 바 (그래디언트)
        if (displayedProgress > 0f) {
            val barWidth = w * displayedProgress
            barRect.set(0f, barTop, barWidth, barBottom)

            barPaint.shader = LinearGradient(
                0f, 0f, barWidth, 0f,
                startColor, endColor,
                Shader.TileMode.CLAMP
            )

            canvas.drawRoundRect(barRect, cornerRadius, cornerRadius, barPaint)
        }
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val desiredHeight = (barHeight + paddingTop + paddingBottom).toInt()
        val height = resolveSize(desiredHeight, heightMeasureSpec)
        setMeasuredDimension(
            MeasureSpec.getSize(widthMeasureSpec),
            height
        )
    }

    // ═══ 애니메이션 ═══

    private fun animateProgress(target: Float) {
        progressAnimator?.cancel()
        progressAnimator = ValueAnimator.ofFloat(displayedProgress, target).apply {
            duration = animationDuration
            interpolator = AccelerateDecelerateInterpolator()
            addUpdateListener { animation ->
                displayedProgress = animation.animatedValue as Float
                invalidate()
            }
            start()
        }
    }

    // ═══ 이벤트 발송 ═══

    private fun dispatchProgressComplete(finalProgress: Float) {
        val reactContext = context as? ReactContext ?: return
        val params = Arguments.createMap().apply {
            putDouble("finalProgress", finalProgress.toDouble())
        }
        reactContext
            .getJSModule(RCTEventEmitter::class.java)
            .receiveEvent(id, "topProgressComplete", params)
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        progressAnimator?.cancel()
    }
}
```

### ViewManager 구현

```kotlin
// android/app/src/main/java/com/myapp/GradientProgressBarManager.kt

package com.myapp

import android.graphics.Color
import com.facebook.react.bridge.ReadableArray
import com.facebook.react.common.MapBuilder
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.annotations.ReactProp

@ReactModule(name = GradientProgressBarManager.NAME)
class GradientProgressBarManager : SimpleViewManager<GradientProgressBarView>() {

    companion object {
        const val NAME = "GradientProgressBar"
    }

    override fun getName(): String = NAME

    override fun createViewInstance(context: ThemedReactContext): GradientProgressBarView {
        return GradientProgressBarView(context)
    }

    // ═══ Props ═══

    @ReactProp(name = "progress", defaultFloat = 0f)
    fun setProgress(view: GradientProgressBarView, progress: Float) {
        view.setProgress(progress)
    }

    @ReactProp(name = "startColor")
    fun setStartColor(view: GradientProgressBarView, color: String?) {
        color?.let { view.setStartColor(Color.parseColor(it)) }
    }

    @ReactProp(name = "endColor")
    fun setEndColor(view: GradientProgressBarView, color: String?) {
        color?.let { view.setEndColor(Color.parseColor(it)) }
    }

    @ReactProp(name = "trackColor")
    fun setTrackColor(view: GradientProgressBarView, color: String?) {
        color?.let { view.setTrackColor(Color.parseColor(it)) }
    }

    @ReactProp(name = "barHeight", defaultFloat = 12f)
    fun setBarHeight(view: GradientProgressBarView, height: Float) {
        view.setBarHeight(height)
    }

    @ReactProp(name = "cornerRadius", defaultFloat = 6f)
    fun setCornerRadius(view: GradientProgressBarView, radius: Float) {
        view.setCornerRadius(radius)
    }

    @ReactProp(name = "animated", defaultBoolean = true)
    fun setAnimated(view: GradientProgressBarView, animated: Boolean) {
        view.setAnimated(animated)
    }

    @ReactProp(name = "animationDuration", defaultFloat = 300f)
    fun setAnimationDuration(view: GradientProgressBarView, duration: Float) {
        view.setAnimationDuration(duration)
    }

    // ═══ Events 등록 ═══

    override fun getExportedCustomDirectEventTypeConstants(): MutableMap<String, Any>? {
        return MapBuilder.builder<String, Any>()
            .put("topProgressComplete",
                MapBuilder.of("registrationName", "onProgressComplete"))
            .build()
    }

    // ═══ Commands ═══

    override fun getCommandsMap(): MutableMap<String, Int> {
        return MapBuilder.of(
            "setProgressAnimated", 1,
            "reset", 2
        )
    }

    override fun receiveCommand(
        view: GradientProgressBarView,
        commandId: String,
        args: ReadableArray?
    ) {
        when (commandId) {
            "setProgressAnimated" -> {
                val progress = args?.getDouble(0)?.toFloat() ?: 0f
                val duration = args?.getDouble(1)?.toFloat() ?: 300f
                view.setProgressAnimated(progress, duration)
            }
            "reset" -> {
                view.reset()
            }
        }
    }
}
```

### React Native에서 사용

```tsx
// src/components/GradientProgressBar.tsx

import React, { useRef, useCallback, useImperativeHandle, forwardRef } from 'react';
import NativeGradientProgressBar, { Commands }
  from '../specs/GradientProgressBarNativeComponent';

export interface GradientProgressBarRef {
  setProgressAnimated: (progress: number, duration: number) => void;
  reset: () => void;
}

interface Props {
  progress: number;
  startColor?: string;
  endColor?: string;
  trackColor?: string;
  barHeight?: number;
  cornerRadius?: number;
  animated?: boolean;
  animationDuration?: number;
  onProgressComplete?: (finalProgress: number) => void;
  style?: any;
}

const GradientProgressBar = forwardRef<GradientProgressBarRef, Props>(
  function GradientProgressBar(props, ref) {
    const nativeRef = useRef<React.ElementRef<typeof NativeGradientProgressBar>>(null);

    // 외부에서 ref로 접근할 수 있는 메서드
    useImperativeHandle(ref, () => ({
      setProgressAnimated: (progress: number, duration: number) => {
        if (nativeRef.current) {
          Commands.setProgressAnimated(nativeRef.current, progress, duration);
        }
      },
      reset: () => {
        if (nativeRef.current) {
          Commands.reset(nativeRef.current);
        }
      },
    }));

    const handleComplete = useCallback((event: any) => {
      props.onProgressComplete?.(event.nativeEvent.finalProgress);
    }, [props.onProgressComplete]);

    return (
      <NativeGradientProgressBar
        ref={nativeRef}
        style={[{ height: props.barHeight ?? 12 }, props.style]}
        progress={props.progress}
        startColor={props.startColor}
        endColor={props.endColor}
        trackColor={props.trackColor}
        barHeight={props.barHeight}
        cornerRadius={props.cornerRadius}
        animated={props.animated}
        animationDuration={props.animationDuration}
        onProgressComplete={handleComplete}
      />
    );
  }
);

export default GradientProgressBar;
```

```tsx
// src/screens/ProgressDemoScreen.tsx

import React, { useState, useRef } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import GradientProgressBar, { GradientProgressBarRef }
  from '../components/GradientProgressBar';

export default function ProgressDemoScreen() {
  const [progress, setProgress] = useState(0);
  const progressRef = useRef<GradientProgressBarRef>(null);

  return (
    <View style={styles.container}>
      <Text style={styles.header}>ProgressBar 데모</Text>

      {/* 기본 사용 */}
      <Text style={styles.label}>기본 (파란→초록 그래디언트)</Text>
      <GradientProgressBar
        progress={progress}
        animated={true}
        animationDuration={500}
        onProgressComplete={(p) => console.log(`완료: ${p}`)}
        style={styles.progressBar}
      />

      {/* 커스텀 색상 */}
      <Text style={styles.label}>커스텀 색상 (빨강→노랑)</Text>
      <GradientProgressBar
        progress={progress}
        startColor="#FF5722"
        endColor="#FFC107"
        trackColor="#FFE0B2"
        barHeight={20}
        cornerRadius={10}
        style={styles.progressBar}
      />

      {/* ref를 통한 명령 호출 */}
      <Text style={styles.label}>Commands 테스트</Text>
      <GradientProgressBar
        ref={progressRef}
        progress={0}
        startColor="#9C27B0"
        endColor="#E040FB"
        style={styles.progressBar}
      />

      {/* 컨트롤 버튼 */}
      <View style={styles.buttonRow}>
        <Pressable style={styles.btn}
          onPress={() => setProgress(p => Math.min(p + 0.1, 1))}>
          <Text style={styles.btnText}>+10%</Text>
        </Pressable>
        <Pressable style={styles.btn}
          onPress={() => setProgress(p => Math.max(p - 0.1, 0))}>
          <Text style={styles.btnText}>-10%</Text>
        </Pressable>
        <Pressable style={styles.btn}
          onPress={() => setProgress(1)}>
          <Text style={styles.btnText}>100%</Text>
        </Pressable>
        <Pressable style={[styles.btn, { backgroundColor: '#F44336' }]}
          onPress={() => {
            setProgress(0);
            progressRef.current?.reset();
          }}>
          <Text style={styles.btnText}>리셋</Text>
        </Pressable>
      </View>

      <Pressable style={[styles.btn, { backgroundColor: '#9C27B0', marginTop: 8 }]}
        onPress={() => progressRef.current?.setProgressAnimated(0.75, 2000)}>
        <Text style={styles.btnText}>75%로 애니메이션 (2초, Commands)</Text>
      </Pressable>

      <Text style={styles.progressText}>{(progress * 100).toFixed(0)}%</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, padding: 24 },
  header: { fontSize: 24, fontWeight: 'bold', marginBottom: 24 },
  label: { fontSize: 14, color: '#666', marginTop: 16, marginBottom: 8 },
  progressBar: { width: '100%', marginVertical: 4 },
  buttonRow: { flexDirection: 'row', gap: 8, marginTop: 24 },
  btn: { flex: 1, backgroundColor: '#4A90D9', padding: 12, borderRadius: 8,
    alignItems: 'center' },
  btnText: { color: 'white', fontWeight: '600' },
  progressText: { fontSize: 48, fontWeight: 'bold', textAlign: 'center',
    marginTop: 24, color: '#333' },
});
```

---

## 5. 예제 2: Native VideoPlayer

ExoPlayer를 래핑한 비디오 플레이어 컴포넌트. Props, Events, Commands를 모두 활용한다.

### Spec 정의

```typescript
// src/specs/VideoPlayerNativeComponent.ts

import type { ViewProps } from 'react-native';
import type {
  Float,
  Int32,
  DirectEventHandler,
} from 'react-native/Libraries/Types/CodegenTypes';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';
import codegenNativeCommands from 'react-native/Libraries/Utilities/codegenNativeCommands';

interface NativeProps extends ViewProps {
  // 비디오 URL
  source: string;

  // 자동 재생
  autoplay?: boolean;

  // 반복 재생
  loop?: boolean;

  // 볼륨 (0.0 ~ 1.0)
  volume?: Float;

  // 음소거
  muted?: boolean;

  // 크기 조정 모드
  resizeMode?: string; // 'contain' | 'cover' | 'stretch'

  // ═══ 이벤트 ═══
  onReady?: DirectEventHandler<Readonly<{
    duration: Float;      // 전체 길이 (초)
    width: Int32;         // 비디오 너비
    height: Int32;        // 비디오 높이
  }>>;

  onPlay?: DirectEventHandler<Readonly<{}>>;

  onPause?: DirectEventHandler<Readonly<{}>>;

  onProgress?: DirectEventHandler<Readonly<{
    currentTime: Float;   // 현재 위치 (초)
    duration: Float;      // 전체 길이 (초)
    buffered: Float;      // 버퍼 비율 (0~1)
  }>>;

  onEnd?: DirectEventHandler<Readonly<{}>>;

  onError?: DirectEventHandler<Readonly<{
    code: string;
    message: string;
  }>>;
}

// Commands
type ComponentType = InstanceType<
  typeof import('react-native').HostComponent<NativeProps>
>;

interface NativeCommands {
  play: (viewRef: React.ElementRef<ComponentType>) => void;
  pause: (viewRef: React.ElementRef<ComponentType>) => void;
  seek: (viewRef: React.ElementRef<ComponentType>, positionSeconds: Float) => void;
}

export const Commands = codegenNativeCommands<NativeCommands>({
  supportedCommands: ['play', 'pause', 'seek'],
});

export default codegenNativeComponent<NativeProps>('VideoPlayer');
```

### Kotlin View 구현

```kotlin
// android/app/src/main/java/com/myapp/VideoPlayerView.kt

package com.myapp

import android.content.Context
import android.widget.FrameLayout
import androidx.media3.common.MediaItem
import androidx.media3.common.PlaybackException
import androidx.media3.common.Player
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.PlayerView
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.ReactContext
import com.facebook.react.uimanager.events.RCTEventEmitter

class VideoPlayerView(context: Context) : FrameLayout(context) {

    private var player: ExoPlayer? = null
    private val playerView: PlayerView = PlayerView(context)

    private var source: String = ""
    private var autoplay: Boolean = false
    private var loop: Boolean = false
    private var isReady: Boolean = false

    init {
        addView(playerView, LayoutParams(
            LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT
        ))
        playerView.useController = false // React Native 쪽에서 UI 제어
    }

    fun setSource(url: String) {
        if (url == source && player != null) return
        source = url
        initializePlayer()
    }

    fun setAutoplay(value: Boolean) {
        autoplay = value
        player?.playWhenReady = value
    }

    fun setLoop(value: Boolean) {
        loop = value
        player?.repeatMode = if (value) Player.REPEAT_MODE_ALL else Player.REPEAT_MODE_OFF
    }

    fun setVolume(volume: Float) {
        player?.volume = volume.coerceIn(0f, 1f)
    }

    fun setMuted(muted: Boolean) {
        player?.volume = if (muted) 0f else 1f
    }

    fun setResizeMode(mode: String) {
        playerView.resizeMode = when (mode) {
            "cover" -> androidx.media3.ui.AspectRatioFrameLayout.RESIZE_MODE_ZOOM
            "stretch" -> androidx.media3.ui.AspectRatioFrameLayout.RESIZE_MODE_FILL
            else -> androidx.media3.ui.AspectRatioFrameLayout.RESIZE_MODE_FIT
        }
    }

    // ═══ Commands ═══

    fun play() {
        player?.play()
    }

    fun pause() {
        player?.pause()
    }

    fun seekTo(positionSeconds: Float) {
        player?.seekTo((positionSeconds * 1000).toLong())
    }

    // ═══ 내부 구현 ═══

    private fun initializePlayer() {
        releasePlayer()

        player = ExoPlayer.Builder(context).build().apply {
            playerView.player = this

            val mediaItem = MediaItem.fromUri(source)
            setMediaItem(mediaItem)
            playWhenReady = autoplay
            repeatMode = if (loop) Player.REPEAT_MODE_ALL else Player.REPEAT_MODE_OFF

            addListener(object : Player.Listener {
                override fun onPlaybackStateChanged(state: Int) {
                    when (state) {
                        Player.STATE_READY -> {
                            if (!isReady) {
                                isReady = true
                                dispatchOnReady()
                            }
                        }
                        Player.STATE_ENDED -> {
                            dispatchEvent("topEnd", Arguments.createMap())
                        }
                    }
                }

                override fun onIsPlayingChanged(isPlaying: Boolean) {
                    if (isPlaying) {
                        dispatchEvent("topPlay", Arguments.createMap())
                        startProgressUpdates()
                    } else {
                        dispatchEvent("topPause", Arguments.createMap())
                        stopProgressUpdates()
                    }
                }

                override fun onPlayerError(error: PlaybackException) {
                    val params = Arguments.createMap().apply {
                        putString("code", "PLAYBACK_ERROR")
                        putString("message", error.message ?: "Unknown error")
                    }
                    dispatchEvent("topError", params)
                }
            })

            prepare()
        }
    }

    private var progressRunnable: Runnable? = null

    private fun startProgressUpdates() {
        stopProgressUpdates()
        progressRunnable = object : Runnable {
            override fun run() {
                player?.let { p ->
                    val params = Arguments.createMap().apply {
                        putDouble("currentTime",
                            p.currentPosition.toDouble() / 1000)
                        putDouble("duration",
                            p.duration.toDouble() / 1000)
                        putDouble("buffered",
                            p.bufferedPercentage.toDouble() / 100)
                    }
                    dispatchEvent("topProgress", params)
                }
                postDelayed(this, 250) // 250ms 마다 업데이트
            }
        }
        post(progressRunnable)
    }

    private fun stopProgressUpdates() {
        progressRunnable?.let { removeCallbacks(it) }
        progressRunnable = null
    }

    private fun dispatchOnReady() {
        player?.let { p ->
            val videoFormat = p.videoFormat
            val params = Arguments.createMap().apply {
                putDouble("duration", p.duration.toDouble() / 1000)
                putInt("width", videoFormat?.width ?: 0)
                putInt("height", videoFormat?.height ?: 0)
            }
            dispatchEvent("topReady", params)
        }
    }

    private fun dispatchEvent(eventName: String, params: com.facebook.react.bridge.WritableMap) {
        val reactContext = context as? ReactContext ?: return
        reactContext
            .getJSModule(RCTEventEmitter::class.java)
            .receiveEvent(id, eventName, params)
    }

    private fun releasePlayer() {
        stopProgressUpdates()
        player?.release()
        player = null
        isReady = false
    }

    override fun onDetachedFromWindow() {
        super.onDetachedFromWindow()
        releasePlayer()
    }
}
```

### ViewManager 구현

```kotlin
// android/app/src/main/java/com/myapp/VideoPlayerManager.kt

package com.myapp

import com.facebook.react.bridge.ReadableArray
import com.facebook.react.common.MapBuilder
import com.facebook.react.module.annotations.ReactModule
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.annotations.ReactProp

@ReactModule(name = VideoPlayerManager.NAME)
class VideoPlayerManager : SimpleViewManager<VideoPlayerView>() {

    companion object {
        const val NAME = "VideoPlayer"
    }

    override fun getName(): String = NAME

    override fun createViewInstance(context: ThemedReactContext): VideoPlayerView {
        return VideoPlayerView(context)
    }

    @ReactProp(name = "source")
    fun setSource(view: VideoPlayerView, source: String?) {
        source?.let { view.setSource(it) }
    }

    @ReactProp(name = "autoplay", defaultBoolean = false)
    fun setAutoplay(view: VideoPlayerView, autoplay: Boolean) {
        view.setAutoplay(autoplay)
    }

    @ReactProp(name = "loop", defaultBoolean = false)
    fun setLoop(view: VideoPlayerView, loop: Boolean) {
        view.setLoop(loop)
    }

    @ReactProp(name = "volume", defaultFloat = 1f)
    fun setVolume(view: VideoPlayerView, volume: Float) {
        view.setVolume(volume)
    }

    @ReactProp(name = "muted", defaultBoolean = false)
    fun setMuted(view: VideoPlayerView, muted: Boolean) {
        view.setMuted(muted)
    }

    @ReactProp(name = "resizeMode")
    fun setResizeMode(view: VideoPlayerView, mode: String?) {
        view.setResizeMode(mode ?: "contain")
    }

    // Events 등록
    override fun getExportedCustomDirectEventTypeConstants(): MutableMap<String, Any>? {
        return MapBuilder.builder<String, Any>()
            .put("topReady", MapBuilder.of("registrationName", "onReady"))
            .put("topPlay", MapBuilder.of("registrationName", "onPlay"))
            .put("topPause", MapBuilder.of("registrationName", "onPause"))
            .put("topProgress", MapBuilder.of("registrationName", "onProgress"))
            .put("topEnd", MapBuilder.of("registrationName", "onEnd"))
            .put("topError", MapBuilder.of("registrationName", "onError"))
            .build()
    }

    // Commands
    override fun getCommandsMap(): MutableMap<String, Int> {
        return MapBuilder.of(
            "play", 1,
            "pause", 2,
            "seek", 3
        )
    }

    override fun receiveCommand(
        view: VideoPlayerView,
        commandId: String,
        args: ReadableArray?
    ) {
        when (commandId) {
            "play" -> view.play()
            "pause" -> view.pause()
            "seek" -> {
                val position = args?.getDouble(0)?.toFloat() ?: 0f
                view.seekTo(position)
            }
        }
    }
}
```

### React Native 래퍼 및 사용

```tsx
// src/components/VideoPlayer.tsx

import React, { useRef, useImperativeHandle, forwardRef, useCallback } from 'react';
import NativeVideoPlayer, { Commands } from '../specs/VideoPlayerNativeComponent';

export interface VideoPlayerRef {
  play: () => void;
  pause: () => void;
  seek: (positionSeconds: number) => void;
}

interface VideoPlayerProps {
  source: string;
  autoplay?: boolean;
  loop?: boolean;
  volume?: number;
  muted?: boolean;
  resizeMode?: 'contain' | 'cover' | 'stretch';
  onReady?: (info: { duration: number; width: number; height: number }) => void;
  onPlay?: () => void;
  onPause?: () => void;
  onProgress?: (info: { currentTime: number; duration: number; buffered: number }) => void;
  onEnd?: () => void;
  onError?: (error: { code: string; message: string }) => void;
  style?: any;
}

const VideoPlayer = forwardRef<VideoPlayerRef, VideoPlayerProps>(
  function VideoPlayer(props, ref) {
    const nativeRef = useRef<React.ElementRef<typeof NativeVideoPlayer>>(null);

    useImperativeHandle(ref, () => ({
      play: () => nativeRef.current && Commands.play(nativeRef.current),
      pause: () => nativeRef.current && Commands.pause(nativeRef.current),
      seek: (pos) => nativeRef.current && Commands.seek(nativeRef.current, pos),
    }));

    return (
      <NativeVideoPlayer
        ref={nativeRef}
        style={props.style}
        source={props.source}
        autoplay={props.autoplay}
        loop={props.loop}
        volume={props.volume}
        muted={props.muted}
        resizeMode={props.resizeMode}
        onReady={(e: any) => props.onReady?.(e.nativeEvent)}
        onPlay={(e: any) => props.onPlay?.()}
        onPause={(e: any) => props.onPause?.()}
        onProgress={(e: any) => props.onProgress?.(e.nativeEvent)}
        onEnd={(e: any) => props.onEnd?.()}
        onError={(e: any) => props.onError?.(e.nativeEvent)}
      />
    );
  }
);

export default VideoPlayer;
```

```tsx
// src/screens/VideoScreen.tsx

import React, { useState, useRef, useCallback } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';
import VideoPlayer, { VideoPlayerRef } from '../components/VideoPlayer';
import GradientProgressBar from '../components/GradientProgressBar';

export default function VideoScreen() {
  const playerRef = useRef<VideoPlayerRef>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);
  const [currentTime, setCurrentTime] = useState(0);

  const formatTime = (seconds: number) => {
    const min = Math.floor(seconds / 60);
    const sec = Math.floor(seconds % 60);
    return `${min}:${sec.toString().padStart(2, '0')}`;
  };

  return (
    <View style={styles.container}>
      <VideoPlayer
        ref={playerRef}
        source="https://example.com/video.mp4"
        autoplay={false}
        resizeMode="contain"
        style={styles.video}
        onReady={(info) => {
          setDuration(info.duration);
          console.log(`비디오 준비: ${info.width}x${info.height}, ${info.duration}초`);
        }}
        onPlay={() => setIsPlaying(true)}
        onPause={() => setIsPlaying(false)}
        onProgress={(info) => {
          setCurrentTime(info.currentTime);
          setProgress(info.currentTime / info.duration);
        }}
        onEnd={() => {
          setIsPlaying(false);
          setProgress(0);
        }}
        onError={(err) => console.error(`재생 오류: ${err.code} - ${err.message}`)}
      />

      {/* 진행 바 (위에서 만든 GradientProgressBar 재사용!) */}
      <GradientProgressBar
        progress={progress}
        startColor="#E040FB"
        endColor="#536DFE"
        animated={false}
        barHeight={4}
        style={styles.seekBar}
      />

      {/* 시간 표시 */}
      <View style={styles.timeRow}>
        <Text style={styles.time}>{formatTime(currentTime)}</Text>
        <Text style={styles.time}>{formatTime(duration)}</Text>
      </View>

      {/* 컨트롤 */}
      <View style={styles.controls}>
        <Pressable onPress={() => playerRef.current?.seek(currentTime - 10)}>
          <Text style={styles.controlBtn}>-10s</Text>
        </Pressable>

        <Pressable onPress={() =>
          isPlaying ? playerRef.current?.pause() : playerRef.current?.play()
        }>
          <Text style={styles.playBtn}>{isPlaying ? 'II' : '>'}</Text>
        </Pressable>

        <Pressable onPress={() => playerRef.current?.seek(currentTime + 10)}>
          <Text style={styles.controlBtn}>+10s</Text>
        </Pressable>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#000' },
  video: { width: '100%', aspectRatio: 16 / 9, backgroundColor: '#000' },
  seekBar: { marginHorizontal: 16, marginTop: 8 },
  timeRow: { flexDirection: 'row', justifyContent: 'space-between',
    paddingHorizontal: 16, marginTop: 4 },
  time: { color: '#aaa', fontSize: 12 },
  controls: { flexDirection: 'row', justifyContent: 'center',
    alignItems: 'center', gap: 32, marginTop: 16 },
  controlBtn: { color: 'white', fontSize: 16, padding: 8 },
  playBtn: { color: 'white', fontSize: 32, padding: 16 },
});
```

---

## 6. 기존 Android 라이브러리 뷰 래핑

기존 Android 라이브러리의 커스텀 View를 Fabric Component로 감싸는 일반적인 패턴:

```
┌────────────────────────────────────────────────────────────────┐
│          기존 라이브러리 뷰 래핑 패턴                             │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. build.gradle에 라이브러리 의존성 추가                        │
│     implementation("com.github.xxx:library:1.0.0")              │
│                                                                 │
│  2. 래퍼 View 생성 (라이브러리 View를 감싸는 FrameLayout)        │
│     class WrapperView(context: Context) : FrameLayout {         │
│       private val libraryView = LibraryView(context)            │
│       init { addView(libraryView) }                             │
│     }                                                           │
│                                                                 │
│  3. ViewManager에서 래퍼 View 사용                               │
│     override fun createViewInstance(ctx) = WrapperView(ctx)     │
│                                                                 │
│  4. 라이브러리의 메서드/콜백을 Props/Events/Commands로 매핑       │
│     libraryView.setOnXxx { ... } → dispatchEvent(...)          │
│     @ReactProp → libraryView.setXxx(...)                       │
│     Commands → libraryView.doXxx()                              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 7. 성능 고려사항

```
┌────────────────────────────────────────────────────────────────┐
│          Fabric Native Component 성능 가이드                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. onDraw()를 가볍게 유지                                      │
│     • Paint 객체는 생성자에서 만들어 재사용                       │
│     • 매 프레임 객체 생성 금지                                   │
│     • 복잡한 계산은 onMeasure/onLayout에서                       │
│                                                                 │
│  2. invalidate() 최소화                                         │
│     • prop이 실제로 변경된 경우에만 invalidate                   │
│     • 같은 값이면 early return                                   │
│                                                                 │
│  3. 이벤트 디스패치 빈도 조절                                    │
│     • onProgress 같은 연속 이벤트는 throttle                     │
│     • 250ms 간격이 적당 (초당 4회)                               │
│     • 매 프레임(16ms) 이벤트는 JS Thread 부담                    │
│                                                                 │
│  4. 큰 데이터는 비동기로                                         │
│     • 이미지 로딩은 Glide/Coil의 비동기 로더 사용                │
│     • 파일 읽기는 백그라운드 스레드에서                            │
│                                                                 │
│  5. 메모리 누수 방지                                             │
│     • onDetachedFromWindow에서 리소스 해제                       │
│     • Listener 제거, Player 해제, Animation 취소                 │
│                                                                 │
│  6. View Recycling 고려                                          │
│     • FlatList 안에서 사용 시 뷰가 재활용됨                      │
│     • 상태를 prop으로 완전히 관리해야 함                          │
│     • 내부 상태에 의존하면 버그 발생                              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 요약

```
┌────────────────────────────────────────────────────────────────┐
│        Fabric Native Component 핵심 체크리스트                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  구현 순서:                                                     │
│  [ ] 1. TypeScript Spec (codegenNativeComponent)               │
│      - Props, Events, Commands 정의                            │
│  [ ] 2. Android View 클래스 (커스텀 드로잉/로직)                │
│  [ ] 3. ViewManager (Props → View, Commands → View)            │
│  [ ] 4. Events 매핑 (getExportedCustomDirectEventTypeConstants)│
│  [ ] 5. Package 등록 (createViewManagers)                      │
│  [ ] 6. React Native 래퍼 컴포넌트 (TypeScript)                │
│  [ ] 7. MainApplication에 Package 추가                         │
│                                                                 │
│  핵심 개념:                                                     │
│  Props = JS → Native (데이터 전달)                              │
│  Events = Native → JS (이벤트 알림)                             │
│  Commands = JS → Native (명령 실행)                             │
│                                                                 │
│  Android 비유:                                                   │
│  Props ≈ XML 속성 / Compose parameters                          │
│  Events ≈ Listener/Callback                                     │
│  Commands ≈ public 메서드 직접 호출                              │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
