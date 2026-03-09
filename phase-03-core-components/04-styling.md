# 스타일링 완전 가이드 — Android styles.xml에서 StyleSheet로

> React Native의 스타일링 시스템을 Android의 스타일/테마 시스템과 비교하며 학습합니다.
> 인라인 스타일, StyleSheet, 동적 스타일링, 반응형 디자인, 다크 모드까지 모두 다룹니다.

---

## 1. React Native 스타일링 기본 개념

React Native에서 스타일은 JavaScript 객체로 작성합니다. CSS와 유사하지만 몇 가지 핵심 차이가 있습니다:

| CSS/웹 | React Native | 차이점 |
|--------|-------------|--------|
| `background-color` (케밥 케이스) | `backgroundColor` (카멜 케이스) | 속성명 표기법 |
| `font-size: 16px` | `fontSize: 16` | 단위 없음 (dp와 유사) |
| 상속 있음 | 텍스트 스타일만 제한적 상속 | `color`, `fontFamily` 등은 자식 Text에 상속 |
| 캐스케이딩 | 없음 | 명시적으로 전달해야 함 |
| 클래스 이름 | style prop | 적용 방식 |
| `display: block` 기본 | `display: flex` 기본 | 레이아웃 기본값 |

### Android와의 비교

```
[Android]
1. res/values/styles.xml에서 스타일 정의
2. XML 레이아웃에서 style="@style/MyStyle" 적용
3. themes.xml에서 앱 전체 테마 설정

[React Native]
1. StyleSheet.create()로 스타일 정의
2. JSX에서 style={styles.myStyle} 적용
3. Context + ThemeProvider로 앱 전체 테마 설정
```

---

## 2. 인라인 스타일 vs StyleSheet.create

### 인라인 스타일

```tsx
import React from 'react';
import { View, Text } from 'react-native';

const InlineStyleExample = () => {
  return (
    <View style={{ flex: 1, padding: 16, backgroundColor: '#fff' }}>
      <Text style={{ fontSize: 24, fontWeight: 'bold', color: '#333' }}>
        인라인 스타일
      </Text>
    </View>
  );
};
```

### StyleSheet.create

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const StyleSheetExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>StyleSheet 스타일</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#333',
  },
});
```

### 비교

| 특성 | 인라인 스타일 | StyleSheet.create |
|------|-------------|-------------------|
| **성능** | 매 렌더링마다 새 객체 생성 | 한 번 생성, 참조로 재사용 |
| **타입 검사** | 약함 | 강함 (자동완성, 오류 감지) |
| **가독성** | JSX 내에 섞여 복잡 | 분리되어 깔끔 |
| **유지보수** | 어려움 | 쉬움 |
| **동적 스타일** | 쉬움 (직접 계산) | 조합 필요 |
| **사용 시기** | 간단한 스타일, 동적 값 | 대부분의 경우 |

> **권장**: 기본적으로 `StyleSheet.create`를 사용하고, 동적으로 변경되는 값만 인라인으로 추가합니다.

### 스타일 조합 (배열 방식)

```tsx
// 여러 스타일을 배열로 조합 (뒤의 스타일이 우선)
<View style={[styles.base, styles.active, { marginTop: 10 }]}>
  <Text style={[styles.text, isError && styles.errorText]}>
    조합된 스타일
  </Text>
</View>

const styles = StyleSheet.create({
  base: { padding: 16, backgroundColor: '#f5f5f5', borderRadius: 8 },
  active: { backgroundColor: '#E3F2FD', borderColor: '#2196F3', borderWidth: 1 },
  text: { fontSize: 16, color: '#333' },
  errorText: { color: '#F44336' },
});
```

Android 비유: `<TextView style="@style/Base.Text" android:textColor="@color/error" />`처럼 기본 스타일에 개별 속성을 오버라이드하는 것과 유사합니다.

```exercise
type: bug-find
question: "이 StyleSheet 코드에는 2개의 버그가 있습니다. 버그가 있는 라인을 클릭하세요."
code: |
  const styles = StyleSheet.create({
    container: {
      flex: 1,
      background-color: '#F5FCFF',
      justifyContent: 'center',
    },
    title: {
      font-size: 20,
      fontWeight: 'bold',
    },
  });
bugLines: [4, 8]
explanations:
  4: "React Native에서는 CSS의 kebab-case가 아닌 camelCase를 사용합니다: backgroundColor"
  8: "fontSize로 작성해야 합니다. React Native 스타일은 camelCase입니다."
xp: 10
```

---

## 3. 모든 스타일 속성 상세 가이드

### 3-1. 레이아웃 (Layout) 속성

| 속성 | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `width` | `number \| string` | 너비 | `android:layout_width` |
| `height` | `number \| string` | 높이 | `android:layout_height` |
| `minWidth` | `number` | 최소 너비 | `android:minWidth` |
| `maxWidth` | `number` | 최대 너비 | `android:maxWidth` |
| `minHeight` | `number` | 최소 높이 | `android:minHeight` |
| `maxHeight` | `number` | 최대 높이 | `android:maxHeight` |
| `padding` | `number` | 내부 여백 (전체) | `android:padding` |
| `paddingHorizontal` | `number` | 좌우 내부 여백 | `android:paddingHorizontal` |
| `paddingVertical` | `number` | 상하 내부 여백 | `android:paddingVertical` |
| `paddingTop` | `number` | 상단 내부 여백 | `android:paddingTop` |
| `paddingBottom` | `number` | 하단 내부 여백 | `android:paddingBottom` |
| `paddingLeft` | `number` | 좌측 내부 여백 | `android:paddingLeft` |
| `paddingRight` | `number` | 우측 내부 여백 | `android:paddingRight` |
| `paddingStart` | `number` | RTL 지원 시작 여백 | `android:paddingStart` |
| `paddingEnd` | `number` | RTL 지원 끝 여백 | `android:paddingEnd` |
| `margin` | `number` | 외부 여백 (전체) | `android:layout_margin` |
| `marginHorizontal` | `number` | 좌우 외부 여백 | `android:layout_marginHorizontal` |
| `marginVertical` | `number` | 상하 외부 여백 | `android:layout_marginVertical` |
| `marginTop` | `number` | 상단 외부 여백 | `android:layout_marginTop` |
| `marginBottom` | `number` | 하단 외부 여백 | `android:layout_marginBottom` |
| `marginLeft` | `number` | 좌측 외부 여백 | `android:layout_marginLeft` |
| `marginRight` | `number` | 우측 외부 여백 | `android:layout_marginRight` |
| `aspectRatio` | `number` | 가로세로 비율 | `ConstraintLayout`의 `dimensionRatio` |
| `overflow` | `'visible' \| 'hidden' \| 'scroll'` | 넘침 처리 | `android:clipChildren` |

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const LayoutExample = () => {
  return (
    <View style={styles.container}>
      {/* 퍼센트 크기 */}
      <View style={styles.percentBox}>
        <Text style={styles.text}>width: '80%'</Text>
      </View>

      {/* aspectRatio */}
      <View style={styles.ratioBox}>
        <Text style={styles.text}>16:9 비율</Text>
      </View>

      {/* 고정 + 최대 크기 */}
      <View style={styles.constrainedBox}>
        <Text style={styles.text}>maxWidth: 300</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, gap: 12 },
  percentBox: {
    width: '80%',
    padding: 16,
    backgroundColor: '#E3F2FD',
    borderRadius: 8,
  },
  ratioBox: {
    width: '100%',
    aspectRatio: 16 / 9,     // 너비 대비 높이 = 16:9
    backgroundColor: '#FFF3E0',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  constrainedBox: {
    width: '100%',
    maxWidth: 300,
    padding: 16,
    backgroundColor: '#E8F5E9',
    borderRadius: 8,
    alignSelf: 'center',
  },
  text: { fontSize: 14, textAlign: 'center' },
});
```

### 3-2. 텍스트 (Typography) 속성

| 속성 | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `fontSize` | `number` | 글자 크기 | `android:textSize` (sp) |
| `fontWeight` | `string` | 글자 두께 | `android:textStyle` |
| `fontFamily` | `string` | 글꼴 | `android:fontFamily` |
| `fontStyle` | `'normal' \| 'italic'` | 이탤릭 | `android:textStyle="italic"` |
| `color` | `string` | 글자 색상 | `android:textColor` |
| `textAlign` | `string` | 텍스트 정렬 | `android:textAlignment` |
| `textAlignVertical` | `string` | 세로 텍스트 정렬 (Android) | `android:gravity` |
| `lineHeight` | `number` | 줄 높이 | `android:lineHeight` |
| `letterSpacing` | `number` | 자간 | `android:letterSpacing` |
| `textDecorationLine` | `string` | 밑줄/취소선 | `Paint.UNDERLINE_TEXT_FLAG` |
| `textDecorationColor` | `string` | 데코레이션 색상 | — |
| `textDecorationStyle` | `string` | 데코레이션 스타일 | — |
| `textTransform` | `string` | 대소문자 변환 | `android:textAllCaps` |
| `textShadowColor` | `string` | 텍스트 그림자 색상 | `android:shadowColor` |
| `textShadowOffset` | `{width, height}` | 그림자 오프셋 | `android:shadowDx/Dy` |
| `textShadowRadius` | `number` | 그림자 반경 | `android:shadowRadius` |
| `includeFontPadding` | `boolean` | 폰트 패딩 포함 (Android) | `android:includeFontPadding` |
| `writingDirection` | `string` | 쓰기 방향 (RTL 지원) | `android:layoutDirection` |

#### fontWeight 값

| fontWeight | 시각적 | Android 대응 |
|-----------|--------|-------------|
| `'100'` | Thin | `android:fontWeight="100"` |
| `'200'` | Extra Light | `android:fontWeight="200"` |
| `'300'` | Light | `android:fontWeight="300"` |
| `'400'` / `'normal'` | Regular (기본) | `android:fontWeight="400"` |
| `'500'` | Medium | `android:fontWeight="500"` |
| `'600'` | Semi Bold | `android:fontWeight="600"` |
| `'700'` / `'bold'` | Bold | `android:textStyle="bold"` |
| `'800'` | Extra Bold | `android:fontWeight="800"` |
| `'900'` | Black | `android:fontWeight="900"` |

#### textAlign 값

| textAlign | 동작 | Android 대응 |
|-----------|------|-------------|
| `'auto'` | 기본 (RTL 시 자동) | `android:textAlignment="gravity"` |
| `'left'` | 왼쪽 정렬 | `android:textAlignment="textStart"` |
| `'right'` | 오른쪽 정렬 | `android:textAlignment="textEnd"` |
| `'center'` | 가운데 정렬 | `android:textAlignment="center"` |
| `'justify'` | 양쪽 정렬 | `android:justificationMode="inter_word"` |

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const TypographyExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.h1}>Heading 1 (28px Bold)</Text>
      <Text style={styles.h2}>Heading 2 (22px Semi Bold)</Text>
      <Text style={styles.h3}>Heading 3 (18px Medium)</Text>
      <Text style={styles.body}>
        본문 텍스트 (16px Regular). lineHeight와 letterSpacing으로 가독성을 높입니다.
        여러 줄의 텍스트에서 적절한 줄 간격은 읽기 편안함을 제공합니다.
      </Text>
      <Text style={styles.caption}>캡션 텍스트 (12px, 회색)</Text>
      <Text style={styles.underline}>밑줄 텍스트</Text>
      <Text style={styles.strikethrough}>취소선 텍스트</Text>
      <Text style={styles.uppercase}>uppercase 변환</Text>
      <Text style={styles.shadow}>그림자 텍스트</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
  h1: { fontSize: 28, fontWeight: 'bold', color: '#1a1a1a' },
  h2: { fontSize: 22, fontWeight: '600', color: '#333' },
  h3: { fontSize: 18, fontWeight: '500', color: '#444' },
  body: {
    fontSize: 16,
    fontWeight: '400',
    color: '#555',
    lineHeight: 24,
    letterSpacing: 0.3,
  },
  caption: { fontSize: 12, color: '#999' },
  underline: {
    fontSize: 16,
    textDecorationLine: 'underline',
    textDecorationColor: '#2196F3',
  },
  strikethrough: {
    fontSize: 16,
    textDecorationLine: 'line-through',
    color: '#999',
  },
  uppercase: {
    fontSize: 14,
    textTransform: 'uppercase',
    letterSpacing: 1.5,
    fontWeight: '600',
    color: '#666',
  },
  shadow: {
    fontSize: 20,
    fontWeight: 'bold',
    textShadowColor: 'rgba(0,0,0,0.3)',
    textShadowOffset: { width: 1, height: 1 },
    textShadowRadius: 3,
  },
});
```

### 3-3. 배경 및 투명도 속성

| 속성 | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `backgroundColor` | `string` | 배경색 | `android:background` |
| `opacity` | `number` (0~1) | 전체 투명도 | `view.setAlpha()` |

```tsx
const OpacityExample = () => {
  return (
    <View style={{ padding: 16, gap: 8 }}>
      <View style={{ backgroundColor: '#2196F3', padding: 16, opacity: 1.0 }}>
        <Text style={{ color: '#fff' }}>opacity: 1.0 (불투명)</Text>
      </View>
      <View style={{ backgroundColor: '#2196F3', padding: 16, opacity: 0.7 }}>
        <Text style={{ color: '#fff' }}>opacity: 0.7</Text>
      </View>
      <View style={{ backgroundColor: '#2196F3', padding: 16, opacity: 0.4 }}>
        <Text style={{ color: '#fff' }}>opacity: 0.4</Text>
      </View>
      <View style={{ backgroundColor: '#2196F3', padding: 16, opacity: 0.1 }}>
        <Text style={{ color: '#fff' }}>opacity: 0.1 (거의 투명)</Text>
      </View>

      {/* rgba로 배경만 반투명 (자식은 영향 없음) */}
      <View style={{ backgroundColor: 'rgba(33, 150, 243, 0.3)', padding: 16 }}>
        <Text>배경만 반투명 (텍스트는 온전)</Text>
      </View>
    </View>
  );
};
```

> **핵심 차이**: `opacity`는 View와 모든 자식의 투명도를 함께 변경합니다. 배경만 반투명하게 하려면 `rgba()` 색상을 사용하세요.

### 3-4. 테두리 (Border) 속성

| 속성 | 타입 | 설명 | Android 대응 |
|------|------|------|-------------|
| `borderWidth` | `number` | 테두리 두께 | `android:background` (shape drawable) |
| `borderColor` | `string` | 테두리 색상 | shape의 `<stroke>` |
| `borderRadius` | `number` | 모서리 둥글기 | shape의 `<corners>` |
| `borderStyle` | `'solid' \| 'dotted' \| 'dashed'` | 테두리 스타일 | — |
| `borderTopWidth` | `number` | 상단 테두리만 | — |
| `borderBottomWidth` | `number` | 하단 테두리만 | — |
| `borderLeftWidth` | `number` | 좌측 테두리만 | — |
| `borderRightWidth` | `number` | 우측 테두리만 | — |
| `borderTopLeftRadius` | `number` | 좌상단 모서리 | `topLeftRadius` |
| `borderTopRightRadius` | `number` | 우상단 모서리 | `topRightRadius` |
| `borderBottomLeftRadius` | `number` | 좌하단 모서리 | `bottomLeftRadius` |
| `borderBottomRightRadius` | `number` | 우하단 모서리 | `bottomRightRadius` |

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const BorderExample = () => {
  return (
    <View style={styles.container}>
      {/* 기본 테두리 */}
      <View style={styles.solidBorder}>
        <Text>solid 테두리</Text>
      </View>

      {/* 점선 테두리 */}
      <View style={styles.dashedBorder}>
        <Text>dashed 테두리</Text>
      </View>

      {/* 원형 */}
      <View style={styles.circle}>
        <Text style={styles.circleText}>원</Text>
      </View>

      {/* 부분 모서리 둥글기 */}
      <View style={styles.partialRadius}>
        <Text>상단만 둥글게</Text>
      </View>

      {/* 하단 테두리만 */}
      <View style={styles.bottomBorderOnly}>
        <Text>하단 테두리만</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 12 },
  solidBorder: {
    borderWidth: 2,
    borderColor: '#2196F3',
    borderRadius: 8,
    padding: 16,
  },
  dashedBorder: {
    borderWidth: 2,
    borderColor: '#FF9800',
    borderStyle: 'dashed',
    borderRadius: 8,
    padding: 16,
  },
  circle: {
    width: 80,
    height: 80,
    borderRadius: 40,
    borderWidth: 3,
    borderColor: '#4CAF50',
    justifyContent: 'center',
    alignItems: 'center',
    alignSelf: 'center',
  },
  circleText: { fontWeight: 'bold', color: '#4CAF50' },
  partialRadius: {
    borderTopLeftRadius: 16,
    borderTopRightRadius: 16,
    borderBottomLeftRadius: 0,
    borderBottomRightRadius: 0,
    backgroundColor: '#E3F2FD',
    padding: 16,
  },
  bottomBorderOnly: {
    borderBottomWidth: 2,
    borderBottomColor: '#e0e0e0',
    paddingBottom: 12,
    paddingTop: 12,
  },
});
```

### 3-5. 그림자 (Shadow) 속성 — iOS vs Android

React Native에서 그림자는 **플랫폼마다 다른 속성**을 사용합니다. 이것은 React Native의 대표적인 플랫폼 차이점입니다.

#### iOS 그림자 속성

| 속성 | 타입 | 설명 |
|------|------|------|
| `shadowColor` | `string` | 그림자 색상 |
| `shadowOffset` | `{width, height}` | 그림자 오프셋 |
| `shadowOpacity` | `number` (0~1) | 그림자 투명도 |
| `shadowRadius` | `number` | 그림자 블러 반경 |

#### Android 그림자 속성

| 속성 | 타입 | 설명 |
|------|------|------|
| `elevation` | `number` | Material Design 고도 (그림자 자동 생성) |

```tsx
import React from 'react';
import { View, Text, Platform, StyleSheet } from 'react-native';

const ShadowExample = () => {
  return (
    <View style={styles.container}>
      {/* elevation 단계별 비교 */}
      {[0, 1, 2, 4, 8, 12, 16, 24].map((elev) => (
        <View key={elev} style={[styles.card, createShadow(elev)]}>
          <Text style={styles.text}>elevation: {elev}</Text>
        </View>
      ))}
    </View>
  );
};

// 크로스 플랫폼 그림자 유틸리티 함수
const createShadow = (elevation: number) => {
  if (Platform.OS === 'android') {
    return { elevation };
  }

  // iOS: elevation 값을 기반으로 iOS 그림자 속성 계산
  return {
    shadowColor: '#000',
    shadowOffset: {
      width: 0,
      height: Math.round(elevation / 2),
    },
    shadowOpacity: elevation > 0 ? 0.05 + elevation * 0.02 : 0,
    shadowRadius: elevation * 0.7,
  };
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 16, backgroundColor: '#f5f5f5' },
  card: {
    padding: 16,
    backgroundColor: '#fff',
    borderRadius: 8,
  },
  text: { fontSize: 14, color: '#333' },
});
```

Android의 `elevation`은 Material Design 사양을 따릅니다:

| elevation | 사용처 | 예시 |
|-----------|--------|------|
| 0 | 배경 | 기본 View |
| 1 | 카드, 검색바 | Card |
| 2 | 떠있는 버튼 (resting) | FAB (눌리지 않은 상태) |
| 4 | 앱바 | Toolbar |
| 6 | FAB (눌린 상태) | — |
| 8 | 메뉴, 사이드 시트 | BottomSheet |
| 12 | FAB (눌린 상태) | — |
| 16 | 네비게이션 드로어 | DrawerLayout |
| 24 | 다이얼로그 | AlertDialog |

### 3-6. 변환 (Transform) 속성

| 속성 | 사용법 | 설명 | Android 대응 |
|------|--------|------|-------------|
| `translateX` | `transform: [{translateX: 50}]` | X축 이동 | `view.translationX = 50f` |
| `translateY` | `transform: [{translateY: 30}]` | Y축 이동 | `view.translationY = 30f` |
| `scale` | `transform: [{scale: 1.5}]` | 균일 확대/축소 | `view.scaleX = 1.5f; view.scaleY = 1.5f` |
| `scaleX` | `transform: [{scaleX: 2}]` | X축 확대/축소 | `view.scaleX = 2f` |
| `scaleY` | `transform: [{scaleY: 2}]` | Y축 확대/축소 | `view.scaleY = 2f` |
| `rotate` | `transform: [{rotate: '45deg'}]` | 회전 | `view.rotation = 45f` |
| `rotateX` | `transform: [{rotateX: '45deg'}]` | X축 3D 회전 | `view.rotationX = 45f` |
| `rotateY` | `transform: [{rotateY: '45deg'}]` | Y축 3D 회전 | `view.rotationY = 45f` |

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const TransformExample = () => {
  return (
    <View style={styles.container}>
      <View style={[styles.box, { transform: [{ rotate: '15deg' }] }]}>
        <Text style={styles.text}>15도 회전</Text>
      </View>

      <View style={[styles.box, { transform: [{ scale: 0.8 }] }]}>
        <Text style={styles.text}>0.8x 축소</Text>
      </View>

      <View style={[styles.box, {
        transform: [
          { translateX: 30 },
          { rotate: '-10deg' },
          { scale: 1.1 },
        ]
      }]}>
        <Text style={styles.text}>복합 변환</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    gap: 40,
  },
  box: {
    width: 120,
    height: 80,
    backgroundColor: '#2196F3',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  text: { color: '#fff', fontWeight: 'bold' },
});
```

---

## 4. 동적 스타일링

### 조건부 스타일

```tsx
import React, { useState } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

const ConditionalStyleExample = () => {
  const [isActive, setIsActive] = useState(false);
  const [isError, setIsError] = useState(false);

  return (
    <View style={styles.container}>
      {/* 방법 1: 배열 + 조건부 */}
      <Pressable
        style={[
          styles.button,
          isActive && styles.activeButton,
          isError && styles.errorButton,
        ]}
        onPress={() => setIsActive(!isActive)}
      >
        <Text style={[
          styles.buttonText,
          isActive && styles.activeButtonText,
        ]}>
          {isActive ? '활성' : '비활성'}
        </Text>
      </Pressable>

      {/* 방법 2: 계산된 스타일 객체 */}
      <View style={[styles.box, { backgroundColor: isActive ? '#4CAF50' : '#e0e0e0' }]}>
        <Text>동적 배경색</Text>
      </View>

      {/* 방법 3: Pressable의 함수형 스타일 */}
      <Pressable
        style={({ pressed }) => [
          styles.button,
          {
            backgroundColor: pressed ? '#1565C0' : '#2196F3',
            transform: [{ scale: pressed ? 0.95 : 1 }],
          },
        ]}
        onPress={() => {}}
      >
        <Text style={styles.buttonText}>누르면 변화</Text>
      </Pressable>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16, gap: 16 },
  button: {
    backgroundColor: '#e0e0e0',
    padding: 14,
    borderRadius: 8,
    alignItems: 'center',
  },
  activeButton: { backgroundColor: '#2196F3' },
  errorButton: { backgroundColor: '#F44336' },
  buttonText: { fontSize: 16, fontWeight: '600', color: '#333' },
  activeButtonText: { color: '#fff' },
  box: { padding: 16, borderRadius: 8 },
});
```

### 계산된 스타일

```tsx
const DynamicSizeExample = ({ progress }: { progress: number }) => {
  // progress: 0 ~ 1
  return (
    <View style={styles.progressBarContainer}>
      <View
        style={[
          styles.progressBarFill,
          {
            width: `${progress * 100}%`,
            backgroundColor: progress > 0.7 ? '#4CAF50' : progress > 0.3 ? '#FF9800' : '#F44336',
          },
        ]}
      />
      <Text style={styles.progressText}>{Math.round(progress * 100)}%</Text>
    </View>
  );
};

const styles = StyleSheet.create({
  progressBarContainer: {
    height: 32,
    backgroundColor: '#e0e0e0',
    borderRadius: 16,
    overflow: 'hidden',
    justifyContent: 'center',
  },
  progressBarFill: {
    position: 'absolute',
    left: 0,
    top: 0,
    bottom: 0,
    borderRadius: 16,
  },
  progressText: {
    textAlign: 'center',
    fontWeight: 'bold',
    fontSize: 12,
  },
});
```

---

## 5. 플랫폼별 스타일링

### Platform.select()

```tsx
import { Platform, StyleSheet } from 'react-native';

const styles = StyleSheet.create({
  container: {
    ...Platform.select({
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 4,
      },
      android: {
        elevation: 4,
      },
    }),
    backgroundColor: '#fff',
    borderRadius: 8,
    padding: 16,
  },

  text: {
    fontSize: 16,
    fontFamily: Platform.select({
      ios: 'San Francisco',
      android: 'Roboto',
      default: 'System',
    }),
  },
});
```

### 플랫폼별 파일

```
components/
├── Card.tsx            # 공통
├── Card.android.tsx    # Android 전용 스타일
└── Card.ios.tsx        # iOS 전용 스타일
```

---

## 6. 반응형 디자인

### Dimensions API

```tsx
import { Dimensions } from 'react-native';

const { width, height } = Dimensions.get('window');

// 화면 크기 기반 스타일
const styles = StyleSheet.create({
  card: {
    width: width * 0.9,       // 화면 너비의 90%
    height: height * 0.3,      // 화면 높이의 30%
  },
});
```

### useWindowDimensions Hook (권장)

`Dimensions`와 달리 화면 회전 시 자동으로 업데이트됩니다.

```tsx
import React from 'react';
import { View, Text, useWindowDimensions, StyleSheet } from 'react-native';

const ResponsiveExample = () => {
  const { width, height } = useWindowDimensions();
  const isLandscape = width > height;
  const isTablet = width >= 768;

  return (
    <View style={[
      styles.container,
      { flexDirection: isLandscape ? 'row' : 'column' },
    ]}>
      <View style={[
        styles.sidebar,
        {
          width: isLandscape ? 300 : '100%',
          height: isLandscape ? '100%' : 200,
        },
      ]}>
        <Text>사이드바 / 헤더</Text>
      </View>
      <View style={styles.content}>
        <Text>메인 콘텐츠</Text>
        <Text>화면 크기: {width} x {height}</Text>
        <Text>{isTablet ? '태블릿' : '폰'} / {isLandscape ? '가로' : '세로'}</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1 },
  sidebar: { backgroundColor: '#E3F2FD', padding: 16, justifyContent: 'center' },
  content: { flex: 1, padding: 16, justifyContent: 'center' },
});
```

### 퍼센트 값

```tsx
const styles = StyleSheet.create({
  halfWidth: { width: '50%' },
  fullWidth: { width: '100%' },
  thirdHeight: { height: '33.33%' },
});
```

> **주의**: `margin`과 `padding`에서 퍼센트는 **부모의 너비(width)**를 기준으로 합니다. `paddingTop: '10%'`는 부모 너비의 10%입니다 (높이가 아님). 이것은 CSS와 동일한 동작입니다.

---

## 7. 다크 모드

### useColorScheme Hook

```tsx
import React from 'react';
import { View, Text, useColorScheme, StyleSheet } from 'react-native';

const DarkModeExample = () => {
  const colorScheme = useColorScheme();
  // 'light' | 'dark' | null

  const isDark = colorScheme === 'dark';

  return (
    <View style={[
      styles.container,
      { backgroundColor: isDark ? '#121212' : '#ffffff' },
    ]}>
      <Text style={[
        styles.title,
        { color: isDark ? '#ffffff' : '#1a1a1a' },
      ]}>
        현재 모드: {isDark ? '다크' : '라이트'}
      </Text>
      <Text style={[
        styles.body,
        { color: isDark ? '#b0b0b0' : '#666666' },
      ]}>
        시스템 설정에 따라 자동으로 변경됩니다.
      </Text>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16, justifyContent: 'center' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 8 },
  body: { fontSize: 16, lineHeight: 24 },
});
```

### 테마 시스템 만들기

```tsx
// theme/colors.ts
export const LightTheme = {
  background: '#ffffff',
  surface: '#f5f5f5',
  primary: '#2196F3',
  primaryVariant: '#1565C0',
  secondary: '#FF4081',
  text: '#1a1a1a',
  textSecondary: '#666666',
  textDisabled: '#999999',
  border: '#e0e0e0',
  error: '#F44336',
  success: '#4CAF50',
  warning: '#FF9800',
};

export const DarkTheme = {
  background: '#121212',
  surface: '#1e1e1e',
  primary: '#90CAF9',
  primaryVariant: '#42A5F5',
  secondary: '#FF80AB',
  text: '#ffffff',
  textSecondary: '#b0b0b0',
  textDisabled: '#666666',
  border: '#333333',
  error: '#EF5350',
  success: '#66BB6A',
  warning: '#FFA726',
};

export type Theme = typeof LightTheme;
```

```tsx
// theme/ThemeContext.tsx
import React, { createContext, useContext, useMemo } from 'react';
import { useColorScheme } from 'react-native';
import { LightTheme, DarkTheme, Theme } from './colors';

const ThemeContext = createContext<Theme>(LightTheme);

export const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const colorScheme = useColorScheme();
  const theme = useMemo(
    () => (colorScheme === 'dark' ? DarkTheme : LightTheme),
    [colorScheme]
  );

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => useContext(ThemeContext);
```

```tsx
// 사용법
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { useTheme } from '../theme/ThemeContext';

const ThemedScreen = () => {
  const theme = useTheme();

  return (
    <View style={[styles.container, { backgroundColor: theme.background }]}>
      <Text style={[styles.title, { color: theme.text }]}>테마 적용된 화면</Text>
      <Text style={[styles.body, { color: theme.textSecondary }]}>
        시스템 다크 모드에 따라 자동으로 테마가 변경됩니다.
      </Text>
      <View style={[styles.card, {
        backgroundColor: theme.surface,
        borderColor: theme.border,
      }]}>
        <Text style={{ color: theme.text }}>테마 카드</Text>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 8 },
  body: { fontSize: 16, lineHeight: 24, marginBottom: 16 },
  card: {
    padding: 16,
    borderRadius: 8,
    borderWidth: 1,
  },
});
```

Android 대응:
```kotlin
// Android에서의 다크 모드 처리
// 1. res/values/themes.xml — 라이트 테마
// 2. res/values-night/themes.xml — 다크 테마
// 3. AppCompatDelegate.setDefaultNightMode()
```

---

## 8. 스타일 패턴: Styled Components

재사용 가능한 스타일 컴포넌트 패턴입니다.

```tsx
import React from 'react';
import { View, Text, Pressable, ViewStyle, TextStyle, StyleSheet } from 'react-native';

// 스타일된 버튼 컴포넌트
interface ButtonProps {
  title: string;
  variant?: 'primary' | 'secondary' | 'outline';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
  onPress: () => void;
}

const Button = ({ title, variant = 'primary', size = 'medium', disabled = false, onPress }: ButtonProps) => {
  return (
    <Pressable
      style={({ pressed }) => [
        buttonStyles.base,
        buttonStyles[variant],
        buttonStyles[size],
        disabled && buttonStyles.disabled,
        pressed && !disabled && buttonStyles.pressed,
      ]}
      disabled={disabled}
      onPress={onPress}
    >
      <Text style={[
        buttonStyles.text,
        buttonStyles[`${variant}Text` as keyof typeof buttonStyles],
        buttonStyles[`${size}Text` as keyof typeof buttonStyles],
        disabled && buttonStyles.disabledText,
      ]}>
        {title}
      </Text>
    </Pressable>
  );
};

const buttonStyles = StyleSheet.create({
  base: {
    borderRadius: 8,
    alignItems: 'center',
    justifyContent: 'center',
  },
  // 변형 (Variant)
  primary: { backgroundColor: '#2196F3' },
  secondary: { backgroundColor: '#FF4081' },
  outline: { backgroundColor: 'transparent', borderWidth: 1.5, borderColor: '#2196F3' },
  // 크기 (Size)
  small: { paddingVertical: 8, paddingHorizontal: 16 },
  medium: { paddingVertical: 12, paddingHorizontal: 24 },
  large: { paddingVertical: 16, paddingHorizontal: 32 },
  // 상태
  disabled: { backgroundColor: '#e0e0e0' },
  pressed: { opacity: 0.8 },
  // 텍스트 스타일
  text: { fontWeight: '600' },
  primaryText: { color: '#fff', fontSize: 16 },
  secondaryText: { color: '#fff', fontSize: 16 },
  outlineText: { color: '#2196F3', fontSize: 16 },
  smallText: { fontSize: 14 },
  mediumText: { fontSize: 16 },
  largeText: { fontSize: 18 },
  disabledText: { color: '#999' },
});

// 사용법
const ButtonShowcase = () => {
  return (
    <View style={{ padding: 16, gap: 12 }}>
      <Button title="Primary Large" variant="primary" size="large" onPress={() => {}} />
      <Button title="Secondary Medium" variant="secondary" size="medium" onPress={() => {}} />
      <Button title="Outline Small" variant="outline" size="small" onPress={() => {}} />
      <Button title="Disabled" variant="primary" disabled={true} onPress={() => {}} />
    </View>
  );
};
```

---

## 9. Android styles.xml → StyleSheet 변환 가이드

### Android styles.xml 예시

```xml
<!-- res/values/styles.xml -->
<resources>
    <style name="AppTheme" parent="Theme.Material3.DayNight.NoActionBar">
        <item name="colorPrimary">@color/primary</item>
        <item name="colorOnPrimary">@color/white</item>
    </style>

    <style name="Widget.Card" parent="">
        <item name="android:layout_margin">16dp</item>
        <item name="android:padding">16dp</item>
        <item name="android:background">@drawable/card_bg</item>
        <item name="android:elevation">4dp</item>
    </style>

    <style name="TextAppearance.Title">
        <item name="android:textSize">24sp</item>
        <item name="android:textColor">@color/text_primary</item>
        <item name="android:fontFamily">sans-serif-medium</item>
    </style>

    <style name="TextAppearance.Body">
        <item name="android:textSize">16sp</item>
        <item name="android:textColor">@color/text_secondary</item>
        <item name="android:lineHeight">24sp</item>
    </style>
</resources>
```

### React Native StyleSheet 대응

```tsx
// constants/Colors.ts  (colors.xml 대응)
export const Colors = {
  primary: '#2196F3',
  onPrimary: '#ffffff',
  textPrimary: '#1a1a1a',
  textSecondary: '#666666',
  background: '#ffffff',
  surface: '#f5f5f5',
};

// theme/Typography.ts  (TextAppearance 대응)
import { StyleSheet } from 'react-native';
import { Colors } from '../constants/Colors';

export const Typography = StyleSheet.create({
  title: {
    fontSize: 24,
    color: Colors.textPrimary,
    fontWeight: '500',     // sans-serif-medium에 대응
  },
  body: {
    fontSize: 16,
    color: Colors.textSecondary,
    lineHeight: 24,
  },
  caption: {
    fontSize: 12,
    color: Colors.textSecondary,
  },
});

// theme/Components.ts  (Widget 스타일 대응)
import { Platform, StyleSheet } from 'react-native';

export const Components = StyleSheet.create({
  card: {
    margin: 16,
    padding: 16,
    backgroundColor: '#ffffff',
    borderRadius: 12,
    ...Platform.select({
      android: { elevation: 4 },
      ios: {
        shadowColor: '#000',
        shadowOffset: { width: 0, height: 2 },
        shadowOpacity: 0.1,
        shadowRadius: 4,
      },
    }),
  },
});
```

```tsx
// 사용법
import React from 'react';
import { View, Text } from 'react-native';
import { Typography } from '../theme/Typography';
import { Components } from '../theme/Components';

const ExampleScreen = () => {
  return (
    <View style={Components.card}>
      <Text style={Typography.title}>카드 제목</Text>
      <Text style={Typography.body}>카드 본문 내용입니다.</Text>
    </View>
  );
};
```

---

## 10. 핵심 비교 요약

| Android | React Native | 비고 |
|---------|-------------|------|
| `res/values/styles.xml` | `StyleSheet.create()` | 스타일 정의 |
| `style="@style/..."` | `style={styles.xxx}` | 스타일 적용 |
| `themes.xml` | Context + ThemeProvider | 전역 테마 |
| `values-night/` | `useColorScheme()` | 다크 모드 |
| `dimens.xml` | 상수 파일 + `useWindowDimensions` | 반응형 크기 |
| `colors.xml` | Colors 상수 파일 | 색상 팔레트 |
| `shape drawable` (XML) | `borderRadius`, `borderWidth` 등 | 도형/테두리 |
| `elevation` | `elevation` (Android) / `shadow*` (iOS) | 그림자 |
| `sp` (폰트 단위) | 숫자 (시스템 스케일 자동 적용) | 접근성 폰트 크기 |
| `dp` (레이아웃 단위) | 숫자 (density-independent) | 레이아웃 크기 |
| `?attr/selectableItemBackground` | `android_ripple` (Pressable) | 터치 피드백 |
| `MaterialComponents` 스타일 | 직접 구현 또는 UI 라이브러리 | Material Design |

> 이것으로 Phase 03의 핵심 컴포넌트와 스타일링 학습이 완료되었습니다.
> 다음 Phase에서는 상태 관리, 네비게이션, 네트워크 통신 등 앱 아키텍처를 다룹니다.
