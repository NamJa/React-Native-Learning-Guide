# Flexbox 레이아웃 완전 정복 — Android 레이아웃에서 전환하기

> Android의 LinearLayout, ConstraintLayout, FrameLayout 등을 React Native의 Flexbox로 대체하는 방법을 학습합니다.
> 모든 Flexbox 속성을 ASCII 다이어그램과 함께 시각적으로 설명합니다.

---

## 1. Flexbox 개념 소개

React Native의 레이아웃 시스템은 CSS Flexbox를 기반으로 합니다. Android에서는 여러 종류의 레이아웃(LinearLayout, ConstraintLayout, FrameLayout, RelativeLayout)을 상황에 따라 선택했지만, React Native에서는 **Flexbox 하나로 모든 레이아웃을 구성**합니다.

### Android 레이아웃 vs React Native Flexbox

| Android | React Native | 비고 |
|---------|-------------|------|
| `LinearLayout` (vertical) | `flexDirection: 'column'` | 기본값 |
| `LinearLayout` (horizontal) | `flexDirection: 'row'` | |
| `FrameLayout` (겹치기) | `position: 'absolute'` | |
| `ConstraintLayout` (비율) | `flex` 비율 | |
| `layout_weight` | `flex` 속성 | 거의 동일한 개념 |
| `layout_gravity` | `alignSelf` | |
| `gravity` | `justifyContent` + `alignItems` | |
| `match_parent` | `flex: 1` 또는 `width: '100%'` | |
| `wrap_content` | 기본 동작 (크기 지정 안 하면) | |
| `dp` | 단위 없는 숫자 (density-independent) | |

### React Native의 핵심 차이점

1. **기본 flexDirection이 `'column'`** (웹 CSS는 `'row'`가 기본)
2. **단위가 없음** — 숫자만 사용 (React Native가 자동으로 density-independent pixel 처리)
3. **모든 View가 기본적으로 flex 컨테이너** — `display: flex`를 따로 지정할 필요 없음

---

## 2. flexDirection — 주축 방향

자식 요소가 배치되는 방향을 결정합니다.

### `flexDirection: 'column'` (기본값)

```
┌─────────────────────────┐
│ ┌─────────────────────┐ │
│ │      Child 1        │ │  ↓ 위에서 아래로
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 2        │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 3        │ │
│ └─────────────────────┘ │
│                         │
└─────────────────────────┘
```

Android 대응: `LinearLayout` with `android:orientation="vertical"`

### `flexDirection: 'row'`

```
┌─────────────────────────────────┐
│ ┌─────────┐ ┌─────────┐ ┌────┐ │
│ │ Child 1 │ │ Child 2 │ │ C3 │ │  → 왼쪽에서 오른쪽으로
│ └─────────┘ └─────────┘ └────┘ │
│                                 │
└─────────────────────────────────┘
```

Android 대응: `LinearLayout` with `android:orientation="horizontal"`

### `flexDirection: 'column-reverse'`

```
┌─────────────────────────┐
│                         │
│ ┌─────────────────────┐ │
│ │      Child 3        │ │  ↑ 아래에서 위로
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 2        │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 1        │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

### `flexDirection: 'row-reverse'`

```
┌─────────────────────────────────┐
│ ┌────┐ ┌─────────┐ ┌─────────┐ │
│ │ C3 │ │ Child 2 │ │ Child 1 │ │  ← 오른쪽에서 왼쪽으로
│ └────┘ └─────────┘ └─────────┘ │
│                                 │
└─────────────────────────────────┘
```

### 코드 예제

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const FlexDirectionExample = () => {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>flexDirection: 'column' (기본)</Text>
      <View style={[styles.box, { flexDirection: 'column' }]}>
        <View style={[styles.child, { backgroundColor: '#FF6B6B' }]}>
          <Text style={styles.childText}>1</Text>
        </View>
        <View style={[styles.child, { backgroundColor: '#4ECDC4' }]}>
          <Text style={styles.childText}>2</Text>
        </View>
        <View style={[styles.child, { backgroundColor: '#45B7D1' }]}>
          <Text style={styles.childText}>3</Text>
        </View>
      </View>

      <Text style={styles.title}>flexDirection: 'row'</Text>
      <View style={[styles.box, { flexDirection: 'row' }]}>
        <View style={[styles.child, { backgroundColor: '#FF6B6B' }]}>
          <Text style={styles.childText}>1</Text>
        </View>
        <View style={[styles.child, { backgroundColor: '#4ECDC4' }]}>
          <Text style={styles.childText}>2</Text>
        </View>
        <View style={[styles.child, { backgroundColor: '#45B7D1' }]}>
          <Text style={styles.childText}>3</Text>
        </View>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  title: { fontSize: 16, fontWeight: 'bold', marginTop: 16, marginBottom: 8 },
  box: {
    backgroundColor: '#f0f0f0',
    padding: 8,
    borderRadius: 8,
  },
  child: {
    width: 60,
    height: 60,
    margin: 4,
    borderRadius: 4,
    justifyContent: 'center',
    alignItems: 'center',
  },
  childText: { color: '#fff', fontWeight: 'bold', fontSize: 18 },
});
```

---

## 3. justifyContent — 주축 정렬

주축(Main Axis) 방향으로 자식 요소를 어떻게 배치할지 결정합니다.
- `flexDirection: 'column'`이면 주축은 **세로**
- `flexDirection: 'row'`이면 주축은 **가로**

아래 다이어그램은 `flexDirection: 'column'` (기본) 기준입니다.

### `justifyContent: 'flex-start'` (기본값)

```
┌───────────────┐
│ ┌───────────┐ │
│ │  Child 1  │ │  ← 시작점에 모임
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 2  │ │
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
│               │
│               │
│               │
└───────────────┘
```

### `justifyContent: 'flex-end'`

```
┌───────────────┐
│               │
│               │
│               │
│ ┌───────────┐ │
│ │  Child 1  │ │  ← 끝점에 모임
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 2  │ │
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
└───────────────┘
```

### `justifyContent: 'center'`

```
┌───────────────┐
│               │
│ ┌───────────┐ │
│ │  Child 1  │ │  ← 가운데 모임
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 2  │ │
│ └───────────┘ │
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
│               │
└───────────────┘
```

### `justifyContent: 'space-between'`

```
┌───────────────┐
│ ┌───────────┐ │
│ │  Child 1  │ │  ← 첫 번째와 마지막은 끝에
│ └───────────┘ │
│               │
│               │
│ ┌───────────┐ │
│ │  Child 2  │ │  ← 사이 간격 균등
│ └───────────┘ │
│               │
│               │
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
└───────────────┘
```

### `justifyContent: 'space-around'`

```
┌───────────────┐
│               │  ← 1/2 간격
│ ┌───────────┐ │
│ │  Child 1  │ │
│ └───────────┘ │
│               │  ← 1 간격
│ ┌───────────┐ │
│ │  Child 2  │ │
│ └───────────┘ │
│               │  ← 1 간격
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
│               │  ← 1/2 간격
└───────────────┘
```

### `justifyContent: 'space-evenly'`

```
┌───────────────┐
│               │  ← 동일 간격
│ ┌───────────┐ │
│ │  Child 1  │ │
│ └───────────┘ │
│               │  ← 동일 간격
│ ┌───────────┐ │
│ │  Child 2  │ │
│ └───────────┘ │
│               │  ← 동일 간격
│ ┌───────────┐ │
│ │  Child 3  │ │
│ └───────────┘ │
│               │  ← 동일 간격
└───────────────┘
```

### Android 대응

| React Native | Android |
|-------------|---------|
| `justifyContent: 'flex-start'` | `android:gravity="top"` (vertical) |
| `justifyContent: 'flex-end'` | `android:gravity="bottom"` |
| `justifyContent: 'center'` | `android:gravity="center_vertical"` |
| `justifyContent: 'space-between'` | ConstraintLayout의 Chain (spread) |
| `justifyContent: 'space-around'` | ConstraintLayout의 Chain (spread_inside) + 비율 |
| `justifyContent: 'space-evenly'` | ConstraintLayout의 Chain (spread) + equal weight |

### 코드 예제

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const JustifyContentExample = () => {
  const values = [
    'flex-start', 'flex-end', 'center',
    'space-between', 'space-around', 'space-evenly',
  ] as const;

  return (
    <View style={styles.container}>
      {values.map((value) => (
        <View key={value} style={styles.column}>
          <Text style={styles.label}>{value}</Text>
          <View style={[styles.box, { justifyContent: value }]}>
            <View style={[styles.child, { backgroundColor: '#FF6B6B' }]} />
            <View style={[styles.child, { backgroundColor: '#4ECDC4' }]} />
            <View style={[styles.child, { backgroundColor: '#45B7D1' }]} />
          </View>
        </View>
      ))}
    </View>
  );
};

const styles = StyleSheet.create({
  container: { flex: 1, flexDirection: 'row', flexWrap: 'wrap', padding: 8 },
  column: { width: '33%', padding: 4 },
  label: { fontSize: 10, textAlign: 'center', marginBottom: 4, fontWeight: 'bold' },
  box: {
    height: 200,
    backgroundColor: '#f0f0f0',
    borderRadius: 4,
    padding: 4,
  },
  child: {
    width: '100%',
    height: 30,
    borderRadius: 4,
  },
});
```

---

## 4. alignItems — 교차축 정렬

교차축(Cross Axis) 방향으로 자식 요소를 정렬합니다.
- `flexDirection: 'column'`이면 교차축은 **가로**
- `flexDirection: 'row'`이면 교차축은 **세로**

아래 다이어그램은 `flexDirection: 'column'` 기준입니다.

### `alignItems: 'stretch'` (기본값)

```
┌─────────────────────────┐
│ ┌─────────────────────┐ │
│ │      Child 1        │ │  ← 전체 너비로 늘어남
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 2        │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │      Child 3        │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

> **주의**: `stretch`가 작동하려면 자식의 교차축 크기(여기서는 width)가 설정되지 않아야 합니다.

### `alignItems: 'flex-start'`

```
┌─────────────────────────┐
│ ┌──────┐                │
│ │  C1  │                │  ← 왼쪽 정렬
│ └──────┘                │
│ ┌─────────┐             │
│ │   C2    │             │
│ └─────────┘             │
│ ┌────┐                  │
│ │ C3 │                  │
│ └────┘                  │
└─────────────────────────┘
```

### `alignItems: 'flex-end'`

```
┌─────────────────────────┐
│                ┌──────┐ │
│                │  C1  │ │  ← 오른쪽 정렬
│                └──────┘ │
│             ┌─────────┐ │
│             │   C2    │ │
│             └─────────┘ │
│                  ┌────┐ │
│                  │ C3 │ │
│                  └────┘ │
└─────────────────────────┘
```

### `alignItems: 'center'`

```
┌─────────────────────────┐
│        ┌──────┐         │
│        │  C1  │         │  ← 가운데 정렬
│        └──────┘         │
│      ┌─────────┐        │
│      │   C2    │        │
│      └─────────┘        │
│         ┌────┐          │
│         │ C3 │          │
│         └────┘          │
└─────────────────────────┘
```

### `alignItems: 'baseline'`

텍스트 기준선(baseline)에 맞춰 정렬합니다. `flexDirection: 'row'`에서 주로 사용합니다.

```
flexDirection: 'row', alignItems: 'baseline'

┌──────────────────────────────────┐
│                                  │
│ ┌──────┐                         │
│ │ 큰   │ ┌────┐                  │
│ │ 텍스트│ │작은│ ┌──┐             │  ← 텍스트 기준선 정렬
│ │      │ │텍스│ │보│             │
│ └──────┘ │트  │ │통│             │
│          └────┘ └──┘             │
│                                  │
└──────────────────────────────────┘
```

### Android 대응

| React Native | Android |
|-------------|---------|
| `alignItems: 'stretch'` | `match_parent` (교차축 방향) |
| `alignItems: 'flex-start'` | `android:layout_gravity="start"` |
| `alignItems: 'flex-end'` | `android:layout_gravity="end"` |
| `alignItems: 'center'` | `android:layout_gravity="center"` |
| `alignItems: 'baseline'` | `android:layout_gravity="baseline"` (LinearLayout) |

---

## 5. alignSelf — 개별 아이템 교차축 정렬

부모의 `alignItems`를 무시하고 특정 자식만 다르게 정렬합니다.

```tsx
<View style={{ alignItems: 'center' }}>
  <View style={{ alignSelf: 'flex-start' }}>왼쪽으로</View>
  <View>가운데 (부모 설정 따름)</View>
  <View style={{ alignSelf: 'flex-end' }}>오른쪽으로</View>
</View>
```

```
┌─────────────────────────┐
│ ┌──────┐                │  ← alignSelf: 'flex-start'
│ │왼쪽  │                │
│ └──────┘                │
│        ┌──────┐         │  ← alignItems: 'center' (부모 설정)
│        │가운데│         │
│        └──────┘         │
│                ┌──────┐ │  ← alignSelf: 'flex-end'
│                │오른쪽│ │
│                └──────┘ │
└─────────────────────────┘
```

Android 대응: `android:layout_gravity` (개별 자식의 gravity 설정)

---

## 6. flexWrap — 줄바꿈

### `flexWrap: 'nowrap'` (기본값)

자식 요소가 한 줄에 강제로 배치됩니다. 넘치면 잘립니다.

### `flexWrap: 'wrap'`

공간이 부족하면 다음 줄로 넘어갑니다.

```
flexDirection: 'row', flexWrap: 'wrap'

┌─────────────────────────────────┐
│ ┌──────┐ ┌──────┐ ┌──────┐     │
│ │  C1  │ │  C2  │ │  C3  │     │  ← 첫 번째 줄
│ └──────┘ └──────┘ └──────┘     │
│ ┌──────┐ ┌──────┐              │
│ │  C4  │ │  C5  │              │  ← 두 번째 줄 (넘침)
│ └──────┘ └──────┘              │
└─────────────────────────────────┘
```

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const FlexWrapExample = () => {
  const items = Array.from({ length: 8 }, (_, i) => i + 1);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>flexWrap: 'wrap'</Text>
      <View style={styles.wrapContainer}>
        {items.map((item) => (
          <View key={item} style={styles.chip}>
            <Text style={styles.chipText}>태그 {item}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16 },
  title: { fontSize: 16, fontWeight: 'bold', marginBottom: 8 },
  wrapContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 8,
  },
  chip: {
    backgroundColor: '#E3F2FD',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: '#90CAF9',
  },
  chipText: { color: '#1565C0', fontSize: 14 },
});
```

---

## 7. flex — 유연한 크기 배분

`flex` 속성은 Android의 `layout_weight`와 거의 동일한 개념입니다.

### flex 기본 동작

```tsx
<View style={{ flex: 1, flexDirection: 'column' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />     {/* 1/3 */}
  <View style={{ flex: 1, backgroundColor: 'green' }} />   {/* 1/3 */}
  <View style={{ flex: 1, backgroundColor: 'blue' }} />    {/* 1/3 */}
</View>
```

```
┌─────────────────────────┐
│                         │
│      flex: 1 (빨강)     │  ← 전체의 1/3
│                         │
├─────────────────────────┤
│                         │
│      flex: 1 (초록)     │  ← 전체의 1/3
│                         │
├─────────────────────────┤
│                         │
│      flex: 1 (파랑)     │  ← 전체의 1/3
│                         │
└─────────────────────────┘
```

### 비율이 다른 경우

```tsx
<View style={{ flex: 1, flexDirection: 'column' }}>
  <View style={{ flex: 1, backgroundColor: 'red' }} />    {/* 1/4 */}
  <View style={{ flex: 2, backgroundColor: 'green' }} />  {/* 2/4 */}
  <View style={{ flex: 1, backgroundColor: 'blue' }} />   {/* 1/4 */}
</View>
```

```
┌─────────────────────────┐
│   flex: 1 (빨강)        │  ← 전체의 1/4
├─────────────────────────┤
│                         │
│   flex: 2 (초록)        │  ← 전체의 2/4
│                         │
├─────────────────────────┤
│   flex: 1 (파랑)        │  ← 전체의 1/4
└─────────────────────────┘
```

Android 대응:
```xml
<LinearLayout android:orientation="vertical" ...>
    <View android:layout_weight="1" android:layout_height="0dp" />
    <View android:layout_weight="2" android:layout_height="0dp" />
    <View android:layout_weight="1" android:layout_height="0dp" />
</LinearLayout>
```

### flex: 0 vs 양수 vs 음수

| 값 | 동작 |
|-----|------|
| `flex: 0` | 기본 크기 사용 (grow도 shrink도 안 함) |
| `flex: 1` (양수) | 남은 공간을 비율대로 차지 |
| `flex: -1` (음수) | 기본 크기 사용, 공간 부족 시 minWidth/minHeight까지 축소 |

### flexGrow, flexShrink, flexBasis

`flex`는 사실 세 가지 속성의 축약입니다:

| 속성 | 설명 | 기본값 |
|------|------|--------|
| `flexGrow` | 남은 공간을 얼마나 차지할지 | 0 |
| `flexShrink` | 공간이 부족할 때 얼마나 줄어들지 | 1 |
| `flexBasis` | 기본 크기 | `auto` |

```tsx
// flex: 1은 아래와 같음
{ flexGrow: 1, flexShrink: 1, flexBasis: 0 }

// 고정 크기 + 나머지 공간 채우기
<View style={{ flexDirection: 'row' }}>
  <View style={{ width: 80, backgroundColor: 'red' }} />          {/* 고정 80 */}
  <View style={{ flexGrow: 1, backgroundColor: 'green' }} />      {/* 나머지 공간 */}
  <View style={{ width: 80, backgroundColor: 'blue' }} />         {/* 고정 80 */}
</View>
```

```
┌──────┬────────────────────────────┬──────┐
│  80  │       flexGrow: 1          │  80  │
│(빨강)│         (초록)              │(파랑)│
└──────┴────────────────────────────┴──────┘
```

---

## 8. gap, rowGap, columnGap — 간격

React Native 0.71부터 `gap` 속성을 지원합니다. 자식 요소 사이의 간격을 설정합니다.

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const GapExample = () => {
  return (
    <View style={styles.container}>
      {/* gap: 모든 방향 간격 */}
      <Text style={styles.title}>gap: 12</Text>
      <View style={styles.gapContainer}>
        {[1, 2, 3, 4, 5, 6].map((i) => (
          <View key={i} style={styles.item}>
            <Text style={styles.itemText}>{i}</Text>
          </View>
        ))}
      </View>

      {/* rowGap + columnGap: 각각 다른 간격 */}
      <Text style={styles.title}>rowGap: 16, columnGap: 8</Text>
      <View style={[styles.gapContainer, { rowGap: 16, columnGap: 8, gap: undefined }]}>
        {[1, 2, 3, 4, 5, 6].map((i) => (
          <View key={i} style={styles.item}>
            <Text style={styles.itemText}>{i}</Text>
          </View>
        ))}
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: { padding: 16 },
  title: { fontSize: 16, fontWeight: 'bold', marginTop: 16, marginBottom: 8 },
  gapContainer: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 12,
    backgroundColor: '#f0f0f0',
    padding: 8,
    borderRadius: 8,
  },
  item: {
    width: 80,
    height: 60,
    backgroundColor: '#4ECDC4',
    borderRadius: 4,
    justifyContent: 'center',
    alignItems: 'center',
  },
  itemText: { color: '#fff', fontWeight: 'bold' },
});
```

> **이전 방식**: `gap`이 지원되기 전에는 각 자식에 `margin`을 개별적으로 설정해야 했습니다. `gap`은 훨씬 깔끔합니다.

---

## 9. position — 위치 지정

### `position: 'relative'` (기본값)

정상적인 문서 흐름에 따라 배치된 후, `top`, `left`, `right`, `bottom`으로 상대적 이동.

### `position: 'absolute'`

부모를 기준으로 절대 위치에 배치됩니다. Android의 `FrameLayout`에서 자식을 겹치는 것과 같습니다.

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const AbsolutePositionExample = () => {
  return (
    <View style={styles.container}>
      {/* 일반 콘텐츠 */}
      <View style={styles.content}>
        <Text>일반 콘텐츠</Text>
      </View>

      {/* 우측 하단에 FAB (Floating Action Button) */}
      <View style={styles.fab}>
        <Text style={styles.fabText}>+</Text>
      </View>

      {/* 좌측 상단에 배지 */}
      <View style={styles.badge}>
        <Text style={styles.badgeText}>3</Text>
      </View>
    </View>
  );
};

// Android 대응:
// <FrameLayout>
//     <ContentView ... />
//     <FloatingActionButton
//         android:layout_gravity="bottom|end"
//         android:layout_margin="16dp" />
// </FrameLayout>

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#f5f5f5',
  },
  content: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  fab: {
    position: 'absolute',
    bottom: 24,
    right: 24,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: '#FF4081',
    justifyContent: 'center',
    alignItems: 'center',
    elevation: 6,
  },
  fabText: {
    color: '#fff',
    fontSize: 24,
    fontWeight: 'bold',
  },
  badge: {
    position: 'absolute',
    top: 12,
    left: 12,
    width: 24,
    height: 24,
    borderRadius: 12,
    backgroundColor: '#F44336',
    justifyContent: 'center',
    alignItems: 'center',
  },
  badgeText: {
    color: '#fff',
    fontSize: 12,
    fontWeight: 'bold',
  },
});
```

---

## 10. zIndex — 겹침 순서

`zIndex`가 높은 요소가 위에 표시됩니다. Android의 `View.setTranslationZ()` 또는 `elevation`과 유사합니다.

```tsx
<View style={{ position: 'relative', height: 200 }}>
  <View style={{
    position: 'absolute', top: 0, left: 0,
    width: 100, height: 100, backgroundColor: 'red',
    zIndex: 1,
  }} />
  <View style={{
    position: 'absolute', top: 30, left: 30,
    width: 100, height: 100, backgroundColor: 'green',
    zIndex: 3,     // 가장 위에 표시됨
  }} />
  <View style={{
    position: 'absolute', top: 60, left: 60,
    width: 100, height: 100, backgroundColor: 'blue',
    zIndex: 2,
  }} />
</View>
```

---

## 11. Android 레이아웃 → React Native 변환 가이드

### LinearLayout (vertical) → flexDirection: 'column'

```xml
<!-- Android -->
<LinearLayout
    android:orientation="vertical"
    android:gravity="center_horizontal"
    android:padding="16dp">
    <TextView android:text="제목" android:textSize="24sp" />
    <TextView android:text="내용" android:textSize="16sp" android:layout_marginTop="8dp" />
    <Button android:text="확인" android:layout_marginTop="16dp" />
</LinearLayout>
```

```tsx
// React Native
<View style={{ padding: 16, alignItems: 'center' }}>
  <Text style={{ fontSize: 24 }}>제목</Text>
  <Text style={{ fontSize: 16, marginTop: 8 }}>내용</Text>
  <Pressable style={{ marginTop: 16 }}>
    <Text>확인</Text>
  </Pressable>
</View>
```

### ConstraintLayout 비율 → flex 비율

```xml
<!-- Android: 3:7 비율 -->
<androidx.constraintlayout.widget.ConstraintLayout>
    <View
        app:layout_constraintWidth_percent="0.3"
        app:layout_constraintStart_toStartOf="parent" />
    <View
        app:layout_constraintWidth_percent="0.7"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

```tsx
// React Native: 3:7 비율
<View style={{ flexDirection: 'row' }}>
  <View style={{ flex: 3, backgroundColor: '#eee' }} />
  <View style={{ flex: 7, backgroundColor: '#ddd' }} />
</View>
```

---

## 12. 실전 레이아웃 패턴 (10가지)

### 패턴 1: Header / Content / Footer

```tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';

const HeaderContentFooter = () => {
  return (
    <View style={styles.container}>
      <View style={styles.header}>
        <Text style={styles.headerText}>헤더</Text>
      </View>
      <View style={styles.content}>
        <Text>콘텐츠 영역 (flex: 1로 남은 공간 차지)</Text>
      </View>
      <View style={styles.footer}>
        <Text style={styles.footerText}>푸터</Text>
      </View>
    </View>
  );
};

// ┌─────────────────────────┐
// │        Header           │  ← 고정 높이
// ├─────────────────────────┤
// │                         │
// │        Content          │  ← flex: 1 (나머지 차지)
// │                         │
// ├─────────────────────────┤
// │        Footer           │  ← 고정 높이
// └─────────────────────────┘

const styles = StyleSheet.create({
  container: { flex: 1 },
  header: {
    height: 60,
    backgroundColor: '#2196F3',
    justifyContent: 'center',
    alignItems: 'center',
  },
  headerText: { color: '#fff', fontSize: 18, fontWeight: 'bold' },
  content: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f5f5f5',
  },
  footer: {
    height: 50,
    backgroundColor: '#333',
    justifyContent: 'center',
    alignItems: 'center',
  },
  footerText: { color: '#fff' },
});
```

### 패턴 2: 카드 레이아웃

```tsx
const Card = () => {
  return (
    <View style={cardStyles.card}>
      <View style={cardStyles.imageContainer}>
        <View style={cardStyles.imagePlaceholder}>
          <Text style={{ color: '#999' }}>이미지</Text>
        </View>
      </View>
      <View style={cardStyles.body}>
        <Text style={cardStyles.title}>카드 제목</Text>
        <Text style={cardStyles.description}>
          카드 설명 텍스트입니다. 간략한 내용을 표시합니다.
        </Text>
      </View>
      <View style={cardStyles.actions}>
        <Pressable style={cardStyles.actionButton}>
          <Text style={cardStyles.actionText}>더 보기</Text>
        </Pressable>
        <Pressable style={cardStyles.actionButton}>
          <Text style={cardStyles.actionText}>공유</Text>
        </Pressable>
      </View>
    </View>
  );
};

// ┌─────────────────────────┐
// │        [이미지]         │
// ├─────────────────────────┤
// │ 카드 제목               │
// │ 카드 설명 텍스트...     │
// ├─────────────────────────┤
// │ [더 보기]    [공유]     │
// └─────────────────────────┘

const cardStyles = StyleSheet.create({
  card: {
    backgroundColor: '#fff',
    borderRadius: 12,
    margin: 16,
    elevation: 3,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    overflow: 'hidden',
  },
  imageContainer: { height: 180, backgroundColor: '#e0e0e0' },
  imagePlaceholder: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  body: { padding: 16 },
  title: { fontSize: 18, fontWeight: 'bold', marginBottom: 8 },
  description: { fontSize: 14, color: '#666', lineHeight: 20 },
  actions: {
    flexDirection: 'row',
    justifyContent: 'flex-end',
    padding: 8,
    borderTopWidth: 1,
    borderTopColor: '#f0f0f0',
    gap: 8,
  },
  actionButton: { padding: 8 },
  actionText: { color: '#2196F3', fontWeight: '600' },
});
```

### 패턴 3: 그리드 레이아웃

```tsx
const GridLayout = () => {
  const items = Array.from({ length: 6 }, (_, i) => i + 1);

  return (
    <View style={gridStyles.container}>
      {items.map((item) => (
        <View key={item} style={gridStyles.gridItem}>
          <Text style={gridStyles.gridText}>{item}</Text>
        </View>
      ))}
    </View>
  );
};

// ┌──────┐ ┌──────┐ ┌──────┐
// │  1   │ │  2   │ │  3   │
// └──────┘ └──────┘ └──────┘
// ┌──────┐ ┌──────┐ ┌──────┐
// │  4   │ │  5   │ │  6   │
// └──────┘ └──────┘ └──────┘

const gridStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    padding: 8,
    gap: 8,
  },
  gridItem: {
    width: '31%',       // (100% - gap) / 3 ≈ 31%
    aspectRatio: 1,     // 정사각형
    backgroundColor: '#4ECDC4',
    borderRadius: 8,
    justifyContent: 'center',
    alignItems: 'center',
  },
  gridText: { color: '#fff', fontSize: 24, fontWeight: 'bold' },
});
```

### 패턴 4: 정가운데 배치

```tsx
const CenteredContent = () => {
  return (
    <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
      <Text style={{ fontSize: 24 }}>정확히 가운데</Text>
    </View>
  );
};

// ┌─────────────────────────┐
// │                         │
// │                         │
// │     정확히 가운데       │  ← 가로세로 모두 중앙
// │                         │
// │                         │
// └─────────────────────────┘
```

### 패턴 5: 사이드바 레이아웃

```tsx
const SidebarLayout = () => {
  return (
    <View style={{ flex: 1, flexDirection: 'row' }}>
      <View style={{
        width: 80,
        backgroundColor: '#1a1a2e',
        alignItems: 'center',
        paddingTop: 20,
        gap: 20,
      }}>
        <View style={{ width: 40, height: 40, borderRadius: 20, backgroundColor: '#e94560' }} />
        <View style={{ width: 40, height: 40, borderRadius: 20, backgroundColor: '#0f3460' }} />
        <View style={{ width: 40, height: 40, borderRadius: 20, backgroundColor: '#16213e' }} />
      </View>
      <View style={{ flex: 1, backgroundColor: '#f5f5f5', padding: 16 }}>
        <Text style={{ fontSize: 20, fontWeight: 'bold' }}>메인 콘텐츠</Text>
      </View>
    </View>
  );
};

// ┌──────┬──────────────────┐
// │ [●]  │                  │
// │ [●]  │   메인 콘텐츠     │
// │ [●]  │                  │
// │      │                  │
// │ 80px │   flex: 1        │
// └──────┴──────────────────┘
```

### 패턴 6: 오버레이 / 모달 위치

```tsx
const OverlayLayout = () => {
  return (
    <View style={{ flex: 1 }}>
      {/* 배경 콘텐츠 */}
      <View style={{ flex: 1, backgroundColor: '#f5f5f5' }}>
        <Text style={{ padding: 16 }}>배경 콘텐츠</Text>
      </View>

      {/* 오버레이 (전체 화면 덮기) */}
      <View style={{
        position: 'absolute',
        top: 0, left: 0, right: 0, bottom: 0,
        backgroundColor: 'rgba(0,0,0,0.5)',
        justifyContent: 'center',
        alignItems: 'center',
      }}>
        <View style={{
          width: '80%',
          backgroundColor: '#fff',
          borderRadius: 12,
          padding: 24,
        }}>
          <Text style={{ fontSize: 18, fontWeight: 'bold' }}>모달 내용</Text>
        </View>
      </View>
    </View>
  );
};
```

### 패턴 7: 균등 간격 (아이콘 바)

```tsx
const EqualSpacingBar = () => {
  const icons = ['홈', '검색', '알림', '설정'];

  return (
    <View style={{
      flexDirection: 'row',
      justifyContent: 'space-around',  // 또는 'space-evenly'
      paddingVertical: 12,
      backgroundColor: '#fff',
      borderTopWidth: 1,
      borderTopColor: '#e0e0e0',
    }}>
      {icons.map((icon) => (
        <Pressable key={icon} style={{ alignItems: 'center', padding: 8 }}>
          <Text style={{ fontSize: 20 }}>■</Text>
          <Text style={{ fontSize: 12, marginTop: 4, color: '#666' }}>{icon}</Text>
        </Pressable>
      ))}
    </View>
  );
};

// ┌──────────────────────────────────┐
// │   ■      ■       ■       ■      │
// │  홈     검색     알림    설정    │
// └──────────────────────────────────┘
```

### 패턴 8: 반응형 비율 레이아웃

```tsx
const ProportionalLayout = () => {
  return (
    <View style={{ flex: 1 }}>
      <View style={{ flex: 3, backgroundColor: '#2196F3', justifyContent: 'center', alignItems: 'center' }}>
        <Text style={{ color: '#fff', fontSize: 20 }}>메인 영역 (3/5)</Text>
      </View>
      <View style={{ flex: 2, backgroundColor: '#4CAF50', justifyContent: 'center', alignItems: 'center' }}>
        <Text style={{ color: '#fff', fontSize: 20 }}>하단 영역 (2/5)</Text>
      </View>
    </View>
  );
};

// ┌─────────────────────────┐
// │                         │
// │    메인 영역 (3/5)      │  ← flex: 3
// │                         │
// ├─────────────────────────┤
// │    하단 영역 (2/5)      │  ← flex: 2
// │                         │
// └─────────────────────────┘
```

### 패턴 9: 하단 고정 버튼

```tsx
const BottomButton = () => {
  return (
    <View style={{ flex: 1 }}>
      {/* 스크롤 가능한 콘텐츠 */}
      <View style={{ flex: 1, padding: 16 }}>
        <Text style={{ fontSize: 20, fontWeight: 'bold', marginBottom: 16 }}>
          가입 정보
        </Text>
        <Text>폼 내용이 여기에...</Text>
      </View>

      {/* 하단 고정 버튼 */}
      <View style={{
        padding: 16,
        borderTopWidth: 1,
        borderTopColor: '#e0e0e0',
        backgroundColor: '#fff',
      }}>
        <Pressable style={{
          backgroundColor: '#2196F3',
          padding: 16,
          borderRadius: 8,
          alignItems: 'center',
        }}>
          <Text style={{ color: '#fff', fontSize: 16, fontWeight: 'bold' }}>
            다음 단계
          </Text>
        </Pressable>
      </View>
    </View>
  );
};

// ┌─────────────────────────┐
// │ 가입 정보               │
// │                         │
// │ 폼 내용이 여기에...     │  ← flex: 1 (스크롤 가능)
// │                         │
// │                         │
// ├─────────────────────────┤
// │   [ 다음 단계 버튼 ]    │  ← 고정
// └─────────────────────────┘
```

### 패턴 10: 수평 스크롤 행

```tsx
import { ScrollView } from 'react-native';

const HorizontalScrollRow = () => {
  const items = Array.from({ length: 10 }, (_, i) => i + 1);

  return (
    <View style={{ padding: 16 }}>
      <Text style={{ fontSize: 18, fontWeight: 'bold', marginBottom: 12 }}>
        추천 항목
      </Text>
      <ScrollView
        horizontal={true}
        showsHorizontalScrollIndicator={false}
        contentContainerStyle={{ gap: 12 }}
      >
        {items.map((item) => (
          <View key={item} style={{
            width: 120,
            height: 160,
            backgroundColor: '#E3F2FD',
            borderRadius: 8,
            justifyContent: 'center',
            alignItems: 'center',
          }}>
            <Text style={{ fontSize: 24, fontWeight: 'bold', color: '#1565C0' }}>
              {item}
            </Text>
          </View>
        ))}
      </ScrollView>
    </View>
  );
};

// 추천 항목
// ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──  →  스크롤
// │      │ │      │ │      │ │      │ │
// │  1   │ │  2   │ │  3   │ │  4   │ │ 5
// │      │ │      │ │      │ │      │ │
// └──────┘ └──────┘ └──────┘ └──────┘ └──
```

---

## 13. 핵심 요약 표

| Flexbox 속성 | 기본값 | Android 대응 | 역할 |
|-------------|--------|-------------|------|
| `flexDirection` | `'column'` | `orientation` | 주축 방향 |
| `justifyContent` | `'flex-start'` | `gravity` (주축) | 주축 정렬 |
| `alignItems` | `'stretch'` | `gravity` (교차축) | 교차축 정렬 |
| `alignSelf` | `'auto'` | `layout_gravity` | 개별 교차축 정렬 |
| `flexWrap` | `'nowrap'` | — | 줄바꿈 |
| `flex` | `0` | `layout_weight` | 유연한 크기 |
| `gap` | `0` | — (margin으로 대체) | 자식 간격 |
| `position` | `'relative'` | — | 위치 모드 |
| `zIndex` | `0` | `elevation` / `translationZ` | 겹침 순서 |

## 🎮 Flexbox 인터랙티브 Playground

아래에서 Flexbox 속성을 직접 조작하고 결과를 확인하세요:

<div class="flexbox-playground-mount"></div>

## ✅ 학습 확인 퀴즈

```quiz
type: mcq
question: "React Native의 기본 flexDirection 값은?"
options:
  - "row"
  - "column"
  - "row-reverse"
  - "column-reverse"
answer: "column"
explanation: "React Native는 웹과 달리 기본 flexDirection이 'column'입니다. 웹 CSS의 기본값은 'row'이므로 주의하세요."
```

```quiz
type: mcq
question: "Android의 layout_weight와 가장 유사한 React Native 스타일 속성은?"
options:
  - "width"
  - "flex"
  - "flexGrow"
  - "alignSelf"
answer: "flex"
explanation: "flex 속성은 Android의 layout_weight처럼 사용 가능한 공간을 비율로 분배합니다."
```

> 다음: [03-lists-and-scrolling.md](./03-lists-and-scrolling.md) — 리스트와 스크롤
