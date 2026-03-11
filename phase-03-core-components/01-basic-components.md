# 기본 컴포넌트 완전 가이드 — Android View를 React Native로

> React Native의 핵심 컴포넌트를 Android 네이티브 컴포넌트와 1:1 비교하며 학습합니다.
> 각 컴포넌트의 모든 주요 props를 다루고, 실전 예제를 제공합니다.

---

## 1. View — 모든 UI의 기본 블록

### Android 대응: `ViewGroup` / `FrameLayout` / `LinearLayout`

`View`는 React Native에서 가장 기본적인 UI 빌딩 블록입니다. Android의 `ViewGroup`처럼 다른 컴포넌트를 담는 컨테이너 역할을 하며, 스타일링과 레이아웃의 기초가 됩니다.

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `style` | `ViewStyle` | 스타일 객체 | XML attributes + styles.xml |
| `onLayout` | `(event) => void` | 레이아웃 계산 완료 시 콜백 | `View.OnLayoutChangeListener` |
| `accessible` | `boolean` | 접근성 요소로 표시 | `importantForAccessibility` |
| `accessibilityLabel` | `string` | 접근성 레이블 | `contentDescription` |
| `accessibilityRole` | `string` | 접근성 역할 (button, header 등) | `AccessibilityNodeInfo.setRoleDescription` |
| `testID` | `string` | 테스트 식별자 | `view.tag` 또는 Espresso `withTagValue` |
| `pointerEvents` | `string` | 터치 이벤트 처리 방식 | `touchDelegate` / `clickable` |
| `hitSlop` | `object` | 터치 영역 확장 | `TouchDelegate` |
| `nativeID` | `string` | 네이티브 뷰 참조 ID | `android:id` |
| `collapsable` | `boolean` | 최적화 시 View 제거 여부 | — |
| `needsOffscreenAlphaCompositing` | `boolean` | 오프스크린 렌더링 | `View.setLayerType` |

### `pointerEvents` 옵션

| 값 | 동작 | 사용 시나리오 |
|-----|------|-------------|
| `'auto'` | 기본값. 정상적으로 터치 이벤트 처리 | — |
| `'none'` | 이 View와 자식들이 터치 이벤트를 받지 않음 | 오버레이 통과 |
| `'box-none'` | 이 View는 터치를 받지 않지만 자식은 받음 | 투명한 컨테이너 |
| `'box-only'` | 이 View만 터치를 받고 자식은 받지 않음 | 터치 가로채기 |

### 예제 1: 기본 View 컨테이너

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const BasicViewExample = () => {
  return (
    <View style={styles.container}>
      <View style={styles.box}>
        <Text>상자 1</Text>
      </View>
      <View style={styles.box}>
        <Text>상자 2</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#ffffff',
  },
  box: {
    padding: 20,
    marginBottom: 10,
    backgroundColor: '#e0e0e0',
    borderRadius: 8,
  },
});

export default BasicViewExample;
```

Android 대응 코드:
```xml
<!-- Android XML -->
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    android:background="#ffffff">

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="20dp"
        android:layout_marginBottom="10dp"
        android:background="@drawable/rounded_bg">
        <TextView android:text="상자 1" />
    </FrameLayout>

    <FrameLayout ...>
        <TextView android:text="상자 2" />
    </FrameLayout>
</LinearLayout>
```

### 예제 2: onLayout으로 View 크기 얻기

```tsx
import React, { useState } from 'react';
import { View, Text, StyleSheet, LayoutChangeEvent } from 'react-native';

const LayoutExample = () => {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  const handleLayout = (event: LayoutChangeEvent) => {
    const { width, height } = event.nativeEvent.layout;
    setDimensions({ width, height });
  };

  return (
    <View style={styles.container} onLayout={handleLayout}>
      <Text>
        이 View의 크기: {dimensions.width.toFixed(0)} x {dimensions.height.toFixed(0)}
      </Text>
    </View>
  );
};

// Android 대응: view.addOnLayoutChangeListener { _, _, _, _, _, _, _, _ ->
//   val width = view.width
//   val height = view.height
// }

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

### 예제 3: 접근성 설정

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const AccessibleView = () => {
  return (
    <View
      accessible={true}
      accessibilityLabel="사용자 프로필 카드"
      accessibilityRole="button"
      accessibilityHint="두 번 탭하면 프로필 상세 화면으로 이동합니다"
      style={styles.card}
    >
      <Text style={styles.name}>홍길동</Text>
      <Text style={styles.email}>hong@example.com</Text>
    </View>
  );
};

// Android 대응:
// android:contentDescription="사용자 프로필 카드"
// android:importantForAccessibility="yes"

const styles = StyleSheet.create({
  card: {
    padding: 16,
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
    margin: 16,
  },
  name: { fontSize: 18, fontWeight: 'bold' },
  email: { fontSize: 14, color: '#666' },
});
```

---

## 2. Text — 텍스트 표시

### Android 대응: `TextView`

React Native에서 텍스트는 반드시 `<Text>` 컴포넌트 안에 있어야 합니다. Android와 달리 `<View>` 안에 직접 문자열을 넣을 수 없습니다.

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `numberOfLines` | `number` | 최대 줄 수 (초과 시 `...` 처리) | `android:maxLines` + `android:ellipsize="end"` |
| `selectable` | `boolean` | 텍스트 선택/복사 가능 | `android:textIsSelectable` |
| `onPress` | `() => void` | 탭 이벤트 | `OnClickListener` |
| `onLongPress` | `() => void` | 길게 누르기 이벤트 | `OnLongClickListener` |
| `ellipsizeMode` | `'head' \| 'middle' \| 'tail' \| 'clip'` | 말줄임 위치 | `android:ellipsize` |
| `adjustsFontSizeToFit` | `boolean` | 텍스트 크기 자동 조절 | `android:autoSizeTextType` |
| `allowFontScaling` | `boolean` | 시스템 폰트 크기 설정 적용 | `android:textSize` (sp 단위) |
| `accessibilityRole` | `string` | 접근성 역할 | `AccessibilityNodeInfo` |
| `style` | `TextStyle` | 텍스트 스타일 | XML attributes |

### 텍스트 스타일 속성

| 스타일 속성 | 타입 | Android 대응 |
|-----------|------|-------------|
| `fontSize` | `number` | `android:textSize` (기본 단위: sp에 가까움) |
| `fontWeight` | `string` | `android:textStyle` ('bold', '100'~'900') |
| `fontFamily` | `string` | `android:fontFamily` |
| `color` | `string` | `android:textColor` |
| `textAlign` | `string` | `android:textAlignment` / `android:gravity` |
| `lineHeight` | `number` | `android:lineHeight` |
| `letterSpacing` | `number` | `android:letterSpacing` |
| `textDecorationLine` | `string` | — (`underline`, `line-through`) |
| `textTransform` | `string` | — (`uppercase`, `lowercase`, `capitalize`) |
| `fontStyle` | `string` | `android:textStyle` ('italic', 'normal') |
| `textShadowColor` | `string` | `android:shadowColor` |
| `textShadowOffset` | `object` | `android:shadowDx`, `android:shadowDy` |
| `textShadowRadius` | `number` | `android:shadowRadius` |
| `includeFontPadding` | `boolean` | `android:includeFontPadding` |

### 예제 1: 기본 텍스트 스타일링

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const TextStyleExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>제목 텍스트</Text>
      <Text style={styles.subtitle}>부제목 텍스트</Text>
      <Text style={styles.body}>
        본문 텍스트입니다. 일반적인 읽기용 텍스트에 적합한 크기와 색상입니다.
      </Text>
      <Text style={styles.caption}>캡션 텍스트 (작은 설명)</Text>
      <Text style={styles.link}>링크 텍스트</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16 },
  title: {
    fontSize: 28,
    fontWeight: 'bold',
    color: '#1a1a1a',
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 20,
    fontWeight: '600',
    color: '#333333',
    marginBottom: 8,
  },
  body: {
    fontSize: 16,
    fontWeight: '400',
    color: '#555555',
    lineHeight: 24,
    marginBottom: 8,
  },
  caption: {
    fontSize: 12,
    color: '#999999',
    marginBottom: 8,
  },
  link: {
    fontSize: 16,
    color: '#2196F3',
    textDecorationLine: 'underline',
  },
});
```

### 예제 2: 중첩 Text (SpannableString 대응)

Android에서 하나의 TextView에 여러 스타일을 적용하려면 `SpannableString`을 사용해야 합니다. React Native에서는 `<Text>`를 중첩하면 됩니다.

```tsx
import React from 'react';
import { Text, StyleSheet } from 'react-native';

const NestedTextExample = () => {
  return (
    <Text style={styles.base}>
      일반 텍스트와{' '}
      <Text style={styles.bold}>굵은 텍스트</Text>
      , 그리고{' '}
      <Text style={styles.italic}>기울임 텍스트</Text>
      를 한 줄에 표시합니다.{' '}
      <Text style={styles.colored}>색상도 변경</Text>
      할 수 있습니다.
    </Text>
  );
};

// Android 대응:
// val spannable = SpannableStringBuilder().apply {
//   append("일반 텍스트와 ")
//   append("굵은 텍스트", StyleSpan(Typeface.BOLD), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
//   append(", 그리고 ")
//   append("기울임 텍스트", StyleSpan(Typeface.ITALIC), Spanned.SPAN_EXCLUSIVE_EXCLUSIVE)
//   // ...
// }
// textView.text = spannable

const styles = StyleSheet.create({
  base: { fontSize: 16, lineHeight: 24, color: '#333' },
  bold: { fontWeight: 'bold' },
  italic: { fontStyle: 'italic' },
  colored: { color: '#e91e63' },
});
```

### 예제 3: numberOfLines와 ellipsizeMode

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const EllipsisExample = () => {
  const longText = '이것은 매우 긴 텍스트입니다. 화면에 다 표시할 수 없을 만큼 긴 내용을 가지고 있으며, 말줄임 처리를 보여주기 위한 예제입니다. React Native에서는 numberOfLines와 ellipsizeMode를 사용합니다.';

  return (
    <View style={styles.container}>
      <Text>말줄임 없음:</Text>
      <Text style={styles.text}>{longText}</Text>

      <Text>2줄 제한 (tail):</Text>
      <Text style={styles.text} numberOfLines={2} ellipsizeMode="tail">
        {longText}
      </Text>

      <Text>1줄 제한 (middle):</Text>
      <Text style={styles.text} numberOfLines={1} ellipsizeMode="middle">
        {longText}
      </Text>

      <Text>1줄 제한 (head):</Text>
      <Text style={styles.text} numberOfLines={1} ellipsizeMode="head">
        {longText}
      </Text>
    </View>
  );
};

// Android 대응:
// android:maxLines="2"
// android:ellipsize="end"  (tail)
// android:ellipsize="middle"
// android:ellipsize="start"  (head)

const styles = StyleSheet.create({
  container: { padding: 16 },
  text: {
    fontSize: 14,
    marginBottom: 16,
    backgroundColor: '#f0f0f0',
    padding: 8,
    borderRadius: 4,
  },
});
```

### 예제 4: 탭 가능한 Text

```tsx
import React from 'react';
import { Text, Alert, StyleSheet } from 'react-native';

const TappableTextExample = () => {
  return (
    <Text style={styles.paragraph}>
      이용약관에 동의합니다.{' '}
      <Text
        style={styles.link}
        onPress={() => Alert.alert('이용약관', '이용약관 내용...')}
      >
        이용약관 보기
      </Text>
      {' '}및{' '}
      <Text
        style={styles.link}
        onPress={() => Alert.alert('개인정보처리방침', '개인정보처리방침 내용...')}
      >
        개인정보처리방침
      </Text>
    </Text>
  );
};

const styles = StyleSheet.create({
  paragraph: { fontSize: 14, color: '#333', padding: 16 },
  link: { color: '#2196F3', textDecorationLine: 'underline' },
});
```

---

## 3. Image — 이미지 표시

### Android 대응: `ImageView` + Glide/Coil

React Native의 `Image`는 로컬 이미지와 네트워크 이미지를 모두 표시할 수 있습니다. Android에서 `ImageView`와 Glide/Coil을 합친 역할입니다.

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `source` | `ImageSource` | 이미지 소스 (로컬 또는 URL) | `android:src` / Glide.load(url) |
| `resizeMode` | `string` | 이미지 크기 조절 방식 | `android:scaleType` |
| `style` | `ImageStyle` | 스타일 (width, height 필수) | XML attributes |
| `onLoad` | `() => void` | 로드 완료 콜백 | Glide `.listener()` |
| `onError` | `(error) => void` | 로드 실패 콜백 | Glide `.error()` |
| `onLoadStart` | `() => void` | 로드 시작 콜백 | — |
| `onLoadEnd` | `() => void` | 로드 완료/실패 후 콜백 | — |
| `defaultSource` | `ImageSource` | 로드 중 표시할 플레이스홀더 (iOS) | Glide `.placeholder()` |
| `blurRadius` | `number` | 블러 효과 | RenderEffect (API 31+) |
| `fadeDuration` | `number` | 페이드인 시간 (Android, ms) | Glide `.transition()` |
| `tintColor` | `string` | 틴트 색상 | `android:tint` |
| `accessible` | `boolean` | 접근성 | `importantForAccessibility` |
| `accessibilityLabel` | `string` | 접근성 레이블 | `contentDescription` |

### resizeMode 비교

| React Native `resizeMode` | Android `scaleType` | 동작 |
|---------------------------|---------------------|------|
| `'cover'` | `CENTER_CROP` | 비율 유지, 영역 채움 (잘릴 수 있음) |
| `'contain'` | `FIT_CENTER` | 비율 유지, 전체 이미지 표시 |
| `'stretch'` | `FIT_XY` | 비율 무시, 영역에 맞춤 |
| `'repeat'` | `REPEAT` (tile) | 이미지 반복 |
| `'center'` | `CENTER` | 원본 크기로 중앙 배치 |

### 예제 1: 로컬 이미지 로드

```tsx
import React from 'react';
import { View, Image, StyleSheet } from 'react-native';

const LocalImageExample = () => {
  return (
    <View style={styles.container}>
      {/* require()로 로컬 이미지 로드 */}
      <Image
        source={require('./assets/logo.png')}
        style={styles.logo}
        resizeMode="contain"
      />
    </View>
  );
};

// Android 대응:
// <ImageView
//     android:src="@drawable/logo"
//     android:scaleType="fitCenter" />

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  logo: {
    width: 200,
    height: 100,
  },
});
```

> **중요**: 로컬 이미지는 `require()`를 사용합니다. Metro 번들러가 빌드 시점에 이미지를 처리합니다. 동적 경로(변수)는 사용할 수 없습니다.

```tsx
// ✅ 올바른 사용법
const image = require('./assets/logo.png');

// ❌ 잘못된 사용법 — 동적 require는 작동하지 않음
const imageName = 'logo';
const image = require(`./assets/${imageName}.png`); // 오류!
```

### 예제 2: 네트워크 이미지 로드

```tsx
import React, { useState } from 'react';
import { View, Image, Text, ActivityIndicator, StyleSheet } from 'react-native';

const NetworkImageExample = () => {
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(false);

  return (
    <View style={styles.container}>
      {loading && (
        <ActivityIndicator style={styles.loader} size="large" color="#2196F3" />
      )}

      {error ? (
        <View style={styles.errorContainer}>
          <Text style={styles.errorText}>이미지를 불러올 수 없습니다</Text>
        </View>
      ) : (
        <Image
          source={{
            uri: 'https://picsum.photos/400/300',
            // headers: { Authorization: 'Bearer token' },  // 인증 헤더 추가 가능
            // cache: 'force-cache',  // 캐시 정책 (iOS)
          }}
          style={styles.image}
          resizeMode="cover"
          onLoadStart={() => setLoading(true)}
          onLoadEnd={() => setLoading(false)}
          onError={(e) => {
            setError(true);
            setLoading(false);
            console.error('Image load error:', e.nativeEvent.error);
          }}
          fadeDuration={300}   // Android: 300ms 페이드인
        />
      )}
    </View>
  );
};

// Android/Glide 대응:
// Glide.with(context)
//     .load("https://picsum.photos/400/300")
//     .placeholder(R.drawable.placeholder)
//     .error(R.drawable.error)
//     .transition(DrawableTransitionOptions.withCrossFade(300))
//     .listener(object : RequestListener<Drawable> { ... })
//     .into(imageView)

const styles = StyleSheet.create({
  container: { alignItems: 'center', padding: 16 },
  image: { width: 400, height: 300, borderRadius: 8 },
  loader: { position: 'absolute', top: 150 },
  errorContainer: {
    width: 400, height: 300, borderRadius: 8,
    backgroundColor: '#f0f0f0',
    justifyContent: 'center', alignItems: 'center',
  },
  errorText: { color: '#999' },
});
```

### 예제 3: 원형 프로필 이미지

```tsx
import React from 'react';
import { View, Image, StyleSheet } from 'react-native';

const CircleImageExample = () => {
  return (
    <View style={styles.container}>
      <Image
        source={{ uri: 'https://i.pravatar.cc/200' }}
        style={styles.avatar}
      />
    </View>
  );
};

// Android 대응:
// ShapeableImageView + ShapeAppearanceOverlay (circle)
// 또는 Glide의 .circleCrop()

const styles = StyleSheet.create({
  container: { alignItems: 'center', padding: 16 },
  avatar: {
    width: 100,
    height: 100,
    borderRadius: 50,          // width/2 = 원형
    borderWidth: 2,
    borderColor: '#2196F3',
  },
});
```

---

## 4. TextInput — 텍스트 입력

### Android 대응: `EditText` / `TextInputEditText` (Material)

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `value` | `string` | 현재 값 (Controlled) | `editText.text` |
| `onChangeText` | `(text: string) => void` | 텍스트 변경 콜백 | `TextWatcher.onTextChanged` |
| `placeholder` | `string` | 플레이스홀더 텍스트 | `android:hint` |
| `placeholderTextColor` | `string` | 플레이스홀더 색상 | `android:textColorHint` |
| `keyboardType` | `string` | 키보드 타입 | `android:inputType` |
| `secureTextEntry` | `boolean` | 비밀번호 입력 (마스킹) | `android:inputType="textPassword"` |
| `multiline` | `boolean` | 여러 줄 입력 | `android:inputType="textMultiLine"` |
| `maxLength` | `number` | 최대 문자 수 | `android:maxLength` |
| `editable` | `boolean` | 편집 가능 여부 | `android:enabled` |
| `autoCapitalize` | `string` | 자동 대문자 | `android:inputType` 플래그 |
| `autoCorrect` | `boolean` | 자동 교정 | `android:inputType` 플래그 |
| `autoFocus` | `boolean` | 자동 포커스 | `requestFocus()` |
| `returnKeyType` | `string` | 키보드 완료 버튼 타입 | `android:imeOptions` |
| `onSubmitEditing` | `() => void` | 완료 버튼 콜백 | `setOnEditorActionListener` |
| `onFocus` | `() => void` | 포커스 획득 | `OnFocusChangeListener` (true) |
| `onBlur` | `() => void` | 포커스 해제 | `OnFocusChangeListener` (false) |
| `textContentType` | `string` | 자동완성 타입 (iOS) | `android:autofillHints` |
| `autoComplete` | `string` | 자동완성 힌트 (Android) | `android:autofillHints` |
| `cursorColor` | `string` | 커서 색상 | `android:textCursorDrawable` |
| `selectionColor` | `string` | 선택 영역 색상 | `android:textColorHighlight` |

### keyboardType 옵션

| React Native | Android `inputType` | 설명 |
|-------------|---------------------|------|
| `'default'` | `TYPE_CLASS_TEXT` | 일반 텍스트 |
| `'email-address'` | `TYPE_TEXT_VARIATION_EMAIL_ADDRESS` | 이메일 |
| `'numeric'` | `TYPE_CLASS_NUMBER` | 숫자만 |
| `'phone-pad'` | `TYPE_CLASS_PHONE` | 전화번호 |
| `'decimal-pad'` | `TYPE_CLASS_NUMBER \| TYPE_NUMBER_FLAG_DECIMAL` | 소수점 포함 숫자 |
| `'number-pad'` | `TYPE_CLASS_NUMBER` | 숫자 패드 |
| `'url'` | `TYPE_TEXT_VARIATION_URI` | URL |

### returnKeyType 옵션

| React Native | Android `imeOptions` | 키보드 버튼 |
|-------------|---------------------|-----------|
| `'done'` | `IME_ACTION_DONE` | 완료 |
| `'go'` | `IME_ACTION_GO` | 이동 |
| `'next'` | `IME_ACTION_NEXT` | 다음 |
| `'search'` | `IME_ACTION_SEARCH` | 검색 |
| `'send'` | `IME_ACTION_SEND` | 보내기 |

### 예제 1: 기본 텍스트 입력

```tsx
import React, { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';

const BasicInputExample = () => {
  const [text, setText] = useState('');

  return (
    <View style={styles.container}>
      <Text style={styles.label}>이름</Text>
      <TextInput
        style={styles.input}
        value={text}
        onChangeText={setText}
        placeholder="이름을 입력하세요"
        placeholderTextColor="#999"
        autoCapitalize="words"
        returnKeyType="done"
      />
      <Text style={styles.preview}>입력한 이름: {text}</Text>
    </View>
  );
};

// Android 대응:
// val editText = findViewById<EditText>(R.id.editText)
// editText.addTextChangedListener(object : TextWatcher {
//     override fun onTextChanged(s: CharSequence?, ...) {
//         textView.text = "입력한 이름: $s"
//     }
// })

const styles = StyleSheet.create({
  container: { padding: 16 },
  label: { fontSize: 16, fontWeight: '600', marginBottom: 8 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    backgroundColor: '#fafafa',
  },
  preview: { marginTop: 16, fontSize: 14, color: '#666' },
});
```

### 예제 2: 로그인 폼 (다중 입력)

```tsx
import React, { useState, useRef } from 'react';
import { View, TextInput, Text, Pressable, StyleSheet, TextInput as RNTextInput } from 'react-native';

const LoginFormExample = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const passwordRef = useRef<RNTextInput>(null);

  const handleLogin = () => {
    console.log('로그인:', email, password);
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>로그인</Text>

      <TextInput
        style={styles.input}
        value={email}
        onChangeText={setEmail}
        placeholder="이메일"
        placeholderTextColor="#999"
        keyboardType="email-address"
        autoCapitalize="none"
        autoComplete="email"
        autoCorrect={false}
        returnKeyType="next"
        onSubmitEditing={() => passwordRef.current?.focus()}
      />

      <TextInput
        ref={passwordRef}
        style={styles.input}
        value={password}
        onChangeText={setPassword}
        placeholder="비밀번호"
        placeholderTextColor="#999"
        secureTextEntry={true}
        autoComplete="password"
        returnKeyType="done"
        onSubmitEditing={handleLogin}
      />

      <Pressable style={styles.button} onPress={handleLogin}>
        <Text style={styles.buttonText}>로그인</Text>
      </Pressable>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 24 },
  title: { fontSize: 28, fontWeight: 'bold', marginBottom: 24 },
  input: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 14,
    fontSize: 16,
    marginBottom: 12,
    backgroundColor: '#fafafa',
  },
  button: {
    backgroundColor: '#2196F3',
    padding: 16,
    borderRadius: 8,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

### 예제 3: 여러 줄 입력 (메모)

```tsx
import React, { useState } from 'react';
import { View, TextInput, Text, StyleSheet } from 'react-native';

const MultilineExample = () => {
  const [memo, setMemo] = useState('');

  return (
    <View style={styles.container}>
      <Text style={styles.label}>메모</Text>
      <TextInput
        style={styles.textArea}
        value={memo}
        onChangeText={setMemo}
        placeholder="메모를 입력하세요..."
        placeholderTextColor="#999"
        multiline={true}
        numberOfLines={6}
        maxLength={500}
        textAlignVertical="top"   // Android에서 텍스트를 상단 정렬
      />
      <Text style={styles.counter}>{memo.length}/500</Text>
    </View>
  );
};

// Android 대응:
// <EditText
//     android:inputType="textMultiLine"
//     android:minLines="6"
//     android:maxLength="500"
//     android:gravity="top" />

const styles = StyleSheet.create({
  container: { padding: 16 },
  label: { fontSize: 16, fontWeight: '600', marginBottom: 8 },
  textArea: {
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 8,
    padding: 12,
    fontSize: 16,
    minHeight: 150,
    backgroundColor: '#fafafa',
  },
  counter: {
    textAlign: 'right',
    marginTop: 4,
    fontSize: 12,
    color: '#999',
  },
});
```

---

## 5. Pressable — 터치 상호작용

### Android 대응: `View.OnClickListener` + Ripple 효과

`Pressable`은 React Native 0.63부터 도입된 핵심 터치 처리 컴포넌트입니다. 기존의 `TouchableOpacity`, `TouchableHighlight`를 대체합니다.

### 주요 Props

| Prop | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `onPress` | `() => void` | 탭 이벤트 | `OnClickListener` |
| `onLongPress` | `() => void` | 길게 누르기 | `OnLongClickListener` |
| `onPressIn` | `() => void` | 누르기 시작 | `MotionEvent.ACTION_DOWN` |
| `onPressOut` | `() => void` | 누르기 해제 | `MotionEvent.ACTION_UP` |
| `delayLongPress` | `number` | 롱프레스 지연시간 (ms) | `ViewConfiguration.getLongPressTimeout()` |
| `disabled` | `boolean` | 비활성화 | `android:enabled="false"` |
| `hitSlop` | `number \| Insets` | 터치 영역 확장 | `TouchDelegate` |
| `pressRetentionOffset` | `Insets` | 눌린 상태 유지 영역 | — |
| `android_ripple` | `RippleConfig` | Android 리플 효과 | `?android:attr/selectableItemBackground` |
| `style` | `ViewStyle \| (state) => ViewStyle` | 스타일 (함수로 동적 스타일 가능) | — |
| `children` | `ReactNode \| (state) => ReactNode` | 자식 (함수로 동적 렌더링 가능) | — |

### android_ripple 옵션

| 속성 | 타입 | 설명 |
|------|------|------|
| `color` | `string` | 리플 색상 |
| `borderless` | `boolean` | 테두리 밖까지 리플 확장 |
| `radius` | `number` | 리플 반경 |
| `foreground` | `boolean` | 전경/배경 리플 |

### 예제 1: 기본 버튼 (리플 효과)

```tsx
import React from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';

const PressableButtonExample = () => {
  return (
    <View style={styles.container}>
      {/* Android 리플 효과가 있는 버튼 */}
      <Pressable
        style={({ pressed }) => [
          styles.button,
          pressed && styles.buttonPressed,
        ]}
        android_ripple={{ color: 'rgba(255,255,255,0.3)' }}
        onPress={() => console.log('버튼 클릭!')}
      >
        {({ pressed }) => (
          <Text style={[styles.buttonText, pressed && styles.buttonTextPressed]}>
            {pressed ? '누르는 중...' : '버튼을 눌러보세요'}
          </Text>
        )}
      </Pressable>
    </View>
  );
};

// Android 대응:
// <Button
//     android:background="?android:attr/selectableItemBackground"
//     android:onClick="onButtonClick" />

const styles = StyleSheet.create({
  container: { padding: 16 },
  button: {
    backgroundColor: '#2196F3',
    paddingVertical: 14,
    paddingHorizontal: 24,
    borderRadius: 8,
    alignItems: 'center',
    elevation: 2,
  },
  buttonPressed: {
    backgroundColor: '#1976D2',
    elevation: 0,
  },
  buttonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  buttonTextPressed: {
    opacity: 0.8,
  },
});
```

### 예제 2: 다양한 버튼 스타일

```tsx
import React from 'react';
import { View, Pressable, Text, StyleSheet } from 'react-native';

const ButtonVariantsExample = () => {
  return (
    <View style={styles.container}>
      {/* Primary 버튼 */}
      <Pressable
        style={({ pressed }) => [
          styles.buttonBase,
          styles.primaryButton,
          pressed && { opacity: 0.8 },
        ]}
        android_ripple={{ color: 'rgba(255,255,255,0.3)' }}
        onPress={() => {}}
      >
        <Text style={[styles.buttonText, styles.primaryText]}>Primary</Text>
      </Pressable>

      {/* Secondary 버튼 (Outlined) */}
      <Pressable
        style={({ pressed }) => [
          styles.buttonBase,
          styles.secondaryButton,
          pressed && { backgroundColor: '#e3f2fd' },
        ]}
        android_ripple={{ color: 'rgba(33,150,243,0.2)' }}
        onPress={() => {}}
      >
        <Text style={[styles.buttonText, styles.secondaryText]}>Secondary</Text>
      </Pressable>

      {/* Text 버튼 */}
      <Pressable
        style={({ pressed }) => [
          styles.buttonBase,
          styles.textButton,
          pressed && { backgroundColor: '#f5f5f5' },
        ]}
        onPress={() => {}}
      >
        <Text style={[styles.buttonText, styles.textButtonText]}>Text Button</Text>
      </Pressable>

      {/* 비활성화 버튼 */}
      <Pressable
        style={[styles.buttonBase, styles.disabledButton]}
        disabled={true}
        onPress={() => {}}
      >
        <Text style={[styles.buttonText, styles.disabledText]}>Disabled</Text>
      </Pressable>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
  buttonBase: {
    paddingVertical: 14,
    paddingHorizontal: 24,
    borderRadius: 8,
    alignItems: 'center',
  },
  primaryButton: { backgroundColor: '#2196F3', elevation: 2 },
  primaryText: { color: '#fff' },
  secondaryButton: {
    backgroundColor: 'transparent',
    borderWidth: 1.5,
    borderColor: '#2196F3',
  },
  secondaryText: { color: '#2196F3' },
  textButton: { backgroundColor: 'transparent' },
  textButtonText: { color: '#2196F3' },
  disabledButton: { backgroundColor: '#e0e0e0' },
  disabledText: { color: '#9e9e9e' },
  buttonText: { fontSize: 16, fontWeight: '600' },
});
```

### 예제 3: 카드 형태의 Pressable

```tsx
import React from 'react';
import { View, Pressable, Text, Image, StyleSheet } from 'react-native';

const PressableCardExample = () => {
  return (
    <View style={styles.container}>
      <Pressable
        style={({ pressed }) => [
          styles.card,
          pressed && styles.cardPressed,
        ]}
        android_ripple={{ color: 'rgba(0,0,0,0.1)' }}
        onPress={() => console.log('카드 클릭')}
        onLongPress={() => console.log('카드 길게 누름')}
      >
        <Image
          source={{ uri: 'https://picsum.photos/300/150' }}
          style={styles.cardImage}
        />
        <View style={styles.cardContent}>
          <Text style={styles.cardTitle}>카드 제목</Text>
          <Text style={styles.cardDescription}>
            카드 설명 텍스트입니다. 이 카드를 누르면 상세 화면으로 이동합니다.
          </Text>
        </View>
      </Pressable>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16 },
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    overflow: 'hidden',
  },
  cardPressed: {
    elevation: 1,
    transform: [{ scale: 0.98 }],
  },
  cardImage: { width: '100%', height: 150 },
  cardContent: { padding: 16 },
  cardTitle: { fontSize: 18, fontWeight: 'bold', marginBottom: 8 },
  cardDescription: { fontSize: 14, color: '#666', lineHeight: 20 },
});
```

---

## 6. TouchableOpacity — 레거시 터치 컴포넌트

### Pressable과의 비교

`TouchableOpacity`는 React Native 초기부터 있던 컴포넌트입니다. 현재는 `Pressable`이 권장되지만, 기존 코드와 많은 라이브러리에서 여전히 사용됩니다.

| 특성 | Pressable | TouchableOpacity |
|------|-----------|-----------------|
| **도입 시기** | 0.63+ (현재 권장) | 초기부터 |
| **pressed 상태 접근** | `style={({pressed}) => ...}` | 불가 |
| **Android 리플** | `android_ripple` prop | 별도 설정 필요 |
| **유연성** | 높음 (함수형 style/children) | 낮음 (opacity만 변경) |
| **기본 효과** | 없음 (직접 구현) | opacity 감소 |
| **커스터마이징** | 완전한 커스터마이징 | 제한적 |

```tsx
import React from 'react';
import { View, TouchableOpacity, Pressable, Text, StyleSheet } from 'react-native';

const ComparisonExample = () => {
  return (
    <View style={styles.container}>
      {/* TouchableOpacity — 누르면 자동으로 투명도 감소 */}
      <TouchableOpacity
        style={styles.button}
        activeOpacity={0.7}  // 누를 때 투명도 (기본: 0.2)
        onPress={() => console.log('TouchableOpacity')}
      >
        <Text style={styles.text}>TouchableOpacity</Text>
      </TouchableOpacity>

      {/* Pressable — 동일한 효과를 수동으로 구현 */}
      <Pressable
        style={({ pressed }) => [
          styles.button,
          { opacity: pressed ? 0.7 : 1 },
        ]}
        onPress={() => console.log('Pressable')}
      >
        <Text style={styles.text}>Pressable (같은 효과)</Text>
      </Pressable>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
  button: {
    backgroundColor: '#4CAF50',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
  },
  text: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

## 7. SafeAreaView — 안전 영역 관리

노치(Notch), 상태바, 홈 인디케이터 등에 의해 UI가 가려지지 않도록 합니다.

```tsx
import React from 'react';
import { SafeAreaView, View, Text, StyleSheet } from 'react-native';

const SafeAreaExample = () => {
  return (
    // SafeAreaView는 최상위에서 사용
    <SafeAreaView style={styles.safeArea}>
      <View style={styles.content}>
        <Text>이 내용은 노치/상태바에 가려지지 않습니다</Text>
      </View>
    </SafeAreaView>
  );
};

// Android 대응:
// WindowCompat.setDecorFitsSystemWindows(window, false)
// ViewCompat.setOnApplyWindowInsetsListener(view) { ... }
// 또는 android:fitsSystemWindows="true"

const styles = StyleSheet.create({
  safeArea: {
    flex: 1,
    backgroundColor: '#fff',
  },
  content: {
    flex: 1,
    padding: 16,
  },
});
```

> **참고**: 더 정밀한 safe area 제어를 위해 `react-native-safe-area-context` 라이브러리 사용을 권장합니다. 이 라이브러리는 `useSafeAreaInsets()` 훅을 제공합니다.

---

## 8. StatusBar — 상태바 제어

```tsx
import React from 'react';
import { View, StatusBar, Text, StyleSheet } from 'react-native';

const StatusBarExample = () => {
  return (
    <View style={styles.container}>
      <StatusBar
        barStyle="dark-content"        // 상태바 아이콘 색상: 'dark-content' | 'light-content'
        backgroundColor="#ffffff"       // Android 전용: 상태바 배경색
        translucent={false}            // Android 전용: 투명 상태바 여부
        animated={true}                // 변경 시 애니메이션
        hidden={false}                 // 상태바 숨김
      />
      <Text>상태바가 설정되었습니다</Text>
    </View>
  );
};

// Android 대응:
// window.statusBarColor = Color.WHITE
// WindowInsetsControllerCompat(window, decorView).isAppearanceLightStatusBars = true

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
});
```

---

## 9. KeyboardAvoidingView — 키보드 회피

Android 개발에서 키보드가 나타날 때 입력 필드가 가려지는 문제는 매우 흔합니다. `KeyboardAvoidingView`는 이 문제를 해결합니다.

```tsx
import React, { useState } from 'react';
import {
  KeyboardAvoidingView,
  Platform,
  View,
  TextInput,
  Text,
  Pressable,
  StyleSheet,
} from 'react-native';

const KeyboardAvoidingExample = () => {
  const [message, setMessage] = useState('');

  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      // behavior 옵션:
      // 'padding' — 하단에 패딩 추가 (iOS 권장)
      // 'height' — 높이 조절 (Android에서 주로 사용)
      // 'position' — 위치 이동
      keyboardVerticalOffset={Platform.OS === 'ios' ? 64 : 0}
      // 네비게이션 바 높이만큼 추가 오프셋
    >
      <View style={styles.content}>
        <Text style={styles.title}>채팅</Text>
        <View style={styles.messagesArea}>
          <Text>메시지 영역</Text>
        </View>
      </View>
      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          value={message}
          onChangeText={setMessage}
          placeholder="메시지를 입력하세요..."
          multiline
        />
        <Pressable style={styles.sendButton} onPress={() => {}}>
          <Text style={styles.sendText}>전송</Text>
        </Pressable>
      </View>
    </KeyboardAvoidingView>
  );
};

// Android 대응:
// AndroidManifest.xml에서:
// android:windowSoftInputMode="adjustResize"
// 또는
// android:windowSoftInputMode="adjustPan"

const styles = StyleSheet.create({
  container: { flex: 1 },
  content: { flex: 1, padding: 16 },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 16 },
  messagesArea: { flex: 1, backgroundColor: '#f5f5f5', borderRadius: 8, padding: 16 },
  inputContainer: {
    flexDirection: 'row',
    padding: 8,
    borderTopWidth: 1,
    borderTopColor: '#e0e0e0',
    alignItems: 'flex-end',
  },
  input: {
    flex: 1,
    borderWidth: 1,
    borderColor: '#ddd',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    maxHeight: 100,
    fontSize: 16,
  },
  sendButton: {
    backgroundColor: '#2196F3',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 10,
    marginLeft: 8,
  },
  sendText: { color: '#fff', fontWeight: '600' },
});
```

---

## 10. Platform API — 플랫폼별 코드 작성

### Platform.OS

```tsx
import { Platform } from 'react-native';

// 현재 플랫폼 확인
console.log(Platform.OS);
// 'android' | 'ios' | 'web' | 'windows' | 'macos'

// 조건부 로직
if (Platform.OS === 'android') {
  console.log('Android에서 실행 중');
}
```

### Platform.select()

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  shadow: Platform.select({
    ios: {
      shadowColor: '#000',
      shadowOffset: { width: 0, height: 2 },
      shadowOpacity: 0.25,
      shadowRadius: 4,
    },
    android: {
      elevation: 4,
    },
    default: {
      // web 등 기타 플랫폼
      boxShadow: '0 2px 4px rgba(0,0,0,0.25)',
    },
  }),
});

// 값으로도 사용 가능
const statusBarHeight = Platform.select({
  ios: 44,
  android: 0,
  default: 0,
});
```

### Platform.Version

```tsx
import { Platform } from 'react-native';

// Android API 레벨 확인
if (Platform.OS === 'android') {
  console.log(Platform.Version); // 예: 35 (API 레벨)

  if (Platform.Version >= 33) {
    // Android 13+ 전용 기능
    console.log('POST_NOTIFICATIONS 권한 필요');
  }
}
```

### 플랫폼별 파일 확장자

파일 확장자를 통해 플랫폼별 코드를 완전히 분리할 수 있습니다:

```
components/
├── Button.tsx           # 공통 (기본)
├── Button.android.tsx   # Android 전용
└── Button.ios.tsx       # iOS 전용
```

```tsx
// 사용 시 — Metro가 자동으로 플랫폼에 맞는 파일을 선택
import Button from './components/Button';
// Android에서는 Button.android.tsx가 로드됨
// iOS에서는 Button.ios.tsx가 로드됨
// 둘 다 없으면 Button.tsx가 로드됨
```

이것은 Android의 리소스 한정자(resource qualifiers)와 유사한 개념입니다:
```
res/
├── layout/             # 기본
├── layout-land/        # 가로 모드
└── layout-sw600dp/     # 태블릿
```

### 예제: 플랫폼별 그림자 유틸리티

```tsx
import { Platform, ViewStyle } from 'react-native';

export const createShadow = (elevation: number): ViewStyle => {
  if (Platform.OS === 'android') {
    return { elevation };
  }

  // iOS 그림자
  return {
    shadowColor: '#000',
    shadowOffset: {
      width: 0,
      height: Math.round(elevation / 2),
    },
    shadowOpacity: 0.05 + elevation * 0.03,
    shadowRadius: elevation * 0.8,
  };
};

// 사용법:
// const styles = StyleSheet.create({
//   card: {
//     ...createShadow(4),
//     backgroundColor: '#fff',
//     borderRadius: 8,
//   },
// });
```

---

## 11. ActivityIndicator — 로딩 인디케이터

```tsx
import React, { useState, useEffect } from 'react';
import { View, ActivityIndicator, Text, StyleSheet } from 'react-native';

const LoadingExample = () => {
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 3초 후 로딩 완료
    const timer = setTimeout(() => setLoading(false), 3000);
    return () => clearTimeout(timer);
  }, []);

  if (loading) {
    return (
      <View style={styles.centered}>
        <ActivityIndicator size="large" color="#2196F3" />
        <Text style={styles.loadingText}>데이터를 불러오는 중...</Text>
      </View>
    );
  }

  return (
    <View style={styles.centered}>
      <Text>데이터 로드 완료!</Text>
    </View>
  );
};

// Android 대응: ProgressBar (indeterminate)
// <ProgressBar
//     android:indeterminate="true"
//     style="?android:attr/progressBarStyleLarge" />

const styles = StyleSheet.create({
  centered: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  loadingText: { marginTop: 16, fontSize: 16, color: '#666' },
});
```

---

## 12. Alert — 알림 다이얼로그

```tsx
import React from 'react';
import { View, Alert, Pressable, Text, StyleSheet } from 'react-native';

const AlertExample = () => {
  // 기본 알림
  const showBasicAlert = () => {
    Alert.alert('알림', '기본 알림 메시지입니다.');
  };

  // 확인/취소 버튼
  const showConfirmAlert = () => {
    Alert.alert(
      '삭제 확인',
      '정말 삭제하시겠습니까? 이 작업은 취소할 수 없습니다.',
      [
        {
          text: '취소',
          style: 'cancel',
          onPress: () => console.log('취소됨'),
        },
        {
          text: '삭제',
          style: 'destructive',
          onPress: () => console.log('삭제됨'),
        },
      ]
    );
  };

  // 3개 버튼
  const showThreeButtonAlert = () => {
    Alert.alert(
      '변경 사항 저장',
      '변경 사항을 저장하시겠습니까?',
      [
        { text: '저장하지 않음', style: 'destructive', onPress: () => {} },
        { text: '취소', style: 'cancel' },
        { text: '저장', onPress: () => {} },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <Pressable style={styles.button} onPress={showBasicAlert}>
        <Text style={styles.text}>기본 알림</Text>
      </Pressable>
      <Pressable style={styles.button} onPress={showConfirmAlert}>
        <Text style={styles.text}>확인 다이얼로그</Text>
      </Pressable>
      <Pressable style={styles.button} onPress={showThreeButtonAlert}>
        <Text style={styles.text}>3버튼 다이얼로그</Text>
      </Pressable>
    </View>
  );
};

// Android 대응:
// AlertDialog.Builder(context)
//     .setTitle("삭제 확인")
//     .setMessage("정말 삭제하시겠습니까?")
//     .setPositiveButton("삭제") { _, _ -> }
//     .setNegativeButton("취소") { _, _ -> }
//     .show()

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
  button: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
  },
  text: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

## 13. Switch — 토글 스위치

```tsx
import React, { useState } from 'react';
import { View, Switch, Text, StyleSheet } from 'react-native';

const SwitchExample = () => {
  const [darkMode, setDarkMode] = useState(false);
  const [notifications, setNotifications] = useState(true);

  return (
    <View style={styles.container}>
      <View style={styles.row}>
        <Text style={styles.label}>다크 모드</Text>
        <Switch
          value={darkMode}
          onValueChange={setDarkMode}
          trackColor={{ false: '#e0e0e0', true: '#81b0ff' }}
          thumbColor={darkMode ? '#2196F3' : '#f4f3f4'}
        />
      </View>
      <View style={styles.row}>
        <Text style={styles.label}>알림</Text>
        <Switch
          value={notifications}
          onValueChange={setNotifications}
          trackColor={{ false: '#e0e0e0', true: '#81C784' }}
          thumbColor={notifications ? '#4CAF50' : '#f4f3f4'}
        />
      </View>
    </View>
  );
};

// Android 대응: SwitchMaterial / MaterialSwitch

const styles = StyleSheet.create({
  container: { padding: 16 },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 12,
    borderBottomWidth: 1,
    borderBottomColor: '#f0f0f0',
  },
  label: { fontSize: 16 },
});
```

---

## 14. Modal — 모달 다이얼로그

```tsx
import React, { useState } from 'react';
import { View, Modal, Text, Pressable, StyleSheet } from 'react-native';

const ModalExample = () => {
  const [visible, setVisible] = useState(false);

  return (
    <View style={styles.container}>
      <Pressable style={styles.openButton} onPress={() => setVisible(true)}>
        <Text style={styles.buttonText}>모달 열기</Text>
      </Pressable>

      <Modal
        visible={visible}
        transparent={true}           // 배경 투명
        animationType="fade"         // 'none' | 'slide' | 'fade'
        onRequestClose={() => setVisible(false)}  // Android 뒤로가기 처리
        statusBarTranslucent={true}  // Android: 상태바 위로 확장
      >
        <View style={styles.overlay}>
          <View style={styles.modalContent}>
            <Text style={styles.modalTitle}>모달 제목</Text>
            <Text style={styles.modalBody}>
              모달 내용입니다. Android의 DialogFragment와 유사합니다.
            </Text>
            <Pressable
              style={styles.closeButton}
              onPress={() => setVisible(false)}
            >
              <Text style={styles.closeButtonText}>닫기</Text>
            </Pressable>
          </View>
        </View>
      </Modal>
    </View>
  );
};

// Android 대응: DialogFragment 또는 BottomSheetDialogFragment

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  openButton: {
    backgroundColor: '#2196F3',
    padding: 14,
    borderRadius: 8,
  },
  buttonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
    justifyContent: 'center',
    alignItems: 'center',
  },
  modalContent: {
    backgroundColor: '#fff',
    borderRadius: 12,
    padding: 24,
    width: '85%',
    elevation: 5,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
  },
  modalTitle: {
    fontSize: 20,
    fontWeight: 'bold',
    marginBottom: 12,
  },
  modalBody: {
    fontSize: 16,
    color: '#666',
    lineHeight: 24,
    marginBottom: 20,
  },
  closeButton: {
    backgroundColor: '#2196F3',
    padding: 12,
    borderRadius: 8,
    alignItems: 'center',
  },
  closeButtonText: { color: '#fff', fontSize: 16, fontWeight: '600' },
});
```

---

### 데이터 처리 실습

```javascript [playground]
// 🧪 기본 컴포넌트 데이터 처리 실습

// View/Text에서 사용하는 조건부 렌더링 로직
function formatUserInfo(user) {
  const parts = [];
  parts.push(user.name);
  if (user.isVip) parts.push("⭐ VIP");
  if (user.age) parts.push(`${user.age}세`);
  return parts.join(" · ");
}

const users = [
  { name: "홍길동", age: 30, isVip: true },
  { name: "김철수", age: 25, isVip: false },
  { name: "이영희", age: null, isVip: true },
];

users.forEach(u => console.log(formatUserInfo(u)));

// TextInput 검증 로직
function validateInput(type, value) {
  const rules = {
    email: { pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/, message: "유효한 이메일을 입력하세요" },
    phone: { pattern: /^01[0-9]-\d{3,4}-\d{4}$/, message: "010-1234-5678 형식" },
    password: { pattern: /^(?=.*[A-Za-z])(?=.*\d).{8,}$/, message: "영문+숫자 8자 이상" },
  };
  const rule = rules[type];
  if (!rule) return { valid: true };
  return {
    valid: rule.pattern.test(value),
    message: rule.pattern.test(value) ? "✅ 유효" : `❌ ${rule.message}`
  };
}

console.log("\n입력 검증:");
console.log("이메일:", validateInput("email", "test@mail.com").message);
console.log("이메일:", validateInput("email", "invalid").message);
console.log("전화:", validateInput("phone", "010-1234-5678").message);
console.log("비번:", validateInput("password", "abc12345").message);
console.log("비번:", validateInput("password", "short").message);
```

---

## 15. 컴포넌트 종합 매핑 표

| React Native | Android | 용도 |
|-------------|---------|------|
| `View` | `ViewGroup` / `FrameLayout` | 컨테이너 |
| `Text` | `TextView` | 텍스트 표시 |
| `Image` | `ImageView` | 이미지 표시 |
| `TextInput` | `EditText` | 텍스트 입력 |
| `Pressable` | `View + OnClickListener + ripple` | 터치 처리 |
| `TouchableOpacity` | `View + OnClickListener (opacity)` | 터치 처리 (레거시) |
| `ScrollView` | `ScrollView` | 스크롤 컨테이너 |
| `FlatList` | `RecyclerView` | 효율적인 리스트 |
| `SectionList` | `RecyclerView + Section headers` | 섹션별 리스트 |
| `SafeAreaView` | `fitsSystemWindows` | 안전 영역 |
| `StatusBar` | `Window.statusBarColor` | 상태바 제어 |
| `KeyboardAvoidingView` | `windowSoftInputMode` | 키보드 회피 |
| `ActivityIndicator` | `ProgressBar` | 로딩 인디케이터 |
| `Switch` | `SwitchMaterial` | 토글 스위치 |
| `Modal` | `DialogFragment` | 모달 다이얼로그 |
| `Alert` | `AlertDialog` | 알림 다이얼로그 |
| `Platform` | `Build.VERSION.SDK_INT` | 플랫폼 정보 |

---

## ✅ 학습 확인 퀴즈

```quiz
type: match
question: "Android 컴포넌트와 React Native 대응을 연결하세요"
pairs:
  - ["ViewGroup / FrameLayout", "View"]
  - ["TextView", "Text"]
  - ["ImageView", "Image"]
  - ["EditText", "TextInput"]
  - ["RecyclerView", "FlatList"]
```

```quiz
type: mcq
question: "React Native에서 터치 이벤트를 처리하기 위해 권장되는 컴포넌트는?"
options:
  - "TouchableOpacity"
  - "Button"
  - "Pressable"
  - "TouchableHighlight"
answer: "Pressable"
explanation: "Pressable은 React Native에서 공식 권장하는 터치 처리 컴포넌트로, press 상태에 따른 세밀한 스타일 제어가 가능합니다."
```

> 다음: [02-flexbox-layout.md](./02-flexbox-layout.md) — Flexbox 레이아웃 완전 정복
