# 선언형 UI 패러다임 — Android XML/Compose에서 React로

## 목차
1. [명령형(Imperative) vs 선언형(Declarative) UI 개념](#1-명령형-vs-선언형-ui-개념)
2. [Android 전통 방식: XML + findViewById](#2-android-전통-방식-xml--findviewbyid)
3. [Android Jetpack Compose 방식](#3-android-jetpack-compose-방식)
4. [React 방식: Function Component + JSX](#4-react-방식-function-component--jsx)
5. [세 가지 접근법 나란히 비교](#5-세-가지-접근법-나란히-비교)
6. [왜 선언형인가? 이점 분석](#6-왜-선언형인가-이점-분석)
7. [Virtual DOM 개념 — Android 개발자를 위한 설명](#7-virtual-dom-개념)
8. [렌더링 생명주기: 초기 렌더 → 상태 변경 → 리렌더](#8-렌더링-생명주기)

---

## 1. 명령형 vs 선언형 UI 개념

### 핵심 차이를 한 문장으로
- **명령형(Imperative):** "이 버튼을 찾아서, 텍스트를 바꿔라, 색상을 빨간색으로 설정해라" — **방법(How)**을 기술
- **선언형(Declarative):** "카운트가 5 이상이면 빨간 텍스트를 보여줘" — **결과(What)**를 기술

### ASCII 다이어그램으로 이해하기

```
┌──────────────────────────────────────────────────────────────────┐
│                     명령형 UI (Imperative)                        │
│                                                                  │
│   개발자가 직접 DOM/View를 조작                                    │
│                                                                  │
│   [상태 변경] ──→ [View 찾기] ──→ [속성 변경] ──→ [화면 갱신]       │
│                                                                  │
│   counter++                                                      │
│   val tv = findViewById<TextView>(R.id.counterText)              │
│   tv.text = counter.toString()                                   │
│   if (counter >= 5) tv.setTextColor(Color.RED)                   │
│                                                                  │
│   ※ 개발자가 "어떻게" 바꿀지 일일이 지시                            │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     선언형 UI (Declarative)                       │
│                                                                  │
│   개발자는 상태에 따른 UI를 "선언"만 함                              │
│                                                                  │
│   [상태 변경] ──→ [프레임워크가 알아서 diff] ──→ [최소 변경 적용]     │
│                                                                  │
│   Text(                                                          │
│     text = "$counter",                                           │
│     color = if (counter >= 5) Color.Red else Color.Black         │
│   )                                                              │
│                                                                  │
│   ※ 개발자는 "무엇을" 보여줄지만 선언                               │
└──────────────────────────────────────────────────────────────────┘
```

### 비유로 이해하기

명령형은 **요리사에게 레시피를 단계별로 알려주는 것**과 같습니다:
> "냄비에 물 500ml를 넣어라. 불을 켜라. 물이 끓으면 면을 넣어라. 4분 후 불을 꺼라."

선언형은 **레스토랑에서 주문하는 것**과 같습니다:
> "라멘 하나 주세요." (어떻게 만들지는 주방이 알아서 처리)

---

## 2. Android 전통 방식: XML + findViewById

Android 개발에서 가장 오래된 패턴입니다. XML로 레이아웃을 정의하고, Activity/Fragment에서 View를 찾아 직접 조작합니다.

### 예제 1: 간단한 카운터 — XML 레이아웃

```xml
<!-- res/layout/activity_counter.xml -->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="16dp">

    <TextView
        android:id="@+id/tvCounter"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="0"
        android:textSize="48sp"
        android:textColor="@android:color/black" />

    <TextView
        android:id="@+id/tvMessage"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text=""
        android:textSize="16sp"
        android:visibility="gone" />

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_marginTop="16dp">

        <Button
            android:id="@+id/btnDecrement"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="-1" />

        <Button
            android:id="@+id/btnIncrement"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="+1"
            android:layout_marginStart="8dp" />
    </LinearLayout>
</LinearLayout>
```

### 예제 2: 간단한 카운터 — Activity 코드

```kotlin
// CounterActivity.kt
class CounterActivity : AppCompatActivity() {

    private var counter = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_counter)

        val tvCounter = findViewById<TextView>(R.id.tvCounter)
        val tvMessage = findViewById<TextView>(R.id.tvMessage)
        val btnIncrement = findViewById<Button>(R.id.btnIncrement)
        val btnDecrement = findViewById<Button>(R.id.btnDecrement)

        btnIncrement.setOnClickListener {
            counter++
            // ⚠️ 명령형: 개발자가 직접 "어떻게" 변경할지 일일이 지시
            tvCounter.text = counter.toString()

            if (counter >= 5) {
                tvCounter.setTextColor(Color.RED)
                tvMessage.text = "카운트가 높습니다!"
                tvMessage.visibility = View.VISIBLE
            } else {
                tvCounter.setTextColor(Color.BLACK)
                tvMessage.visibility = View.GONE
            }

            // 감소 버튼 활성화/비활성화도 직접 관리
            btnDecrement.isEnabled = counter > 0
        }

        btnDecrement.setOnClickListener {
            if (counter > 0) {
                counter--
                tvCounter.text = counter.toString()

                // ⚠️ 동일한 조건 분기 로직이 또 반복됨!
                if (counter >= 5) {
                    tvCounter.setTextColor(Color.RED)
                    tvMessage.text = "카운트가 높습니다!"
                    tvMessage.visibility = View.VISIBLE
                } else {
                    tvCounter.setTextColor(Color.BLACK)
                    tvMessage.visibility = View.GONE
                }

                btnDecrement.isEnabled = counter > 0
            }
        }
    }
}
```

### 이 방식의 문제점

```
┌─────────────────────────────────────────────────────────┐
│ 명령형 UI의 문제점                                        │
│                                                         │
│ 1. 코드 중복: 증가/감소에서 동일한 UI 업데이트 로직 반복      │
│ 2. 상태 불일치: counter 값과 화면이 동기화되지 않을 위험     │
│ 3. 복잡도 폭발: 상태가 많아지면 조건 분기가 기하급수적       │
│ 4. 테스트 어려움: UI 업데이트 로직이 이벤트 핸들러에 분산    │
│ 5. View 참조 관리: null 참조, 메모리 누수 위험             │
└─────────────────────────────────────────────────────────┘
```

### 예제 3: ViewBinding 개선 버전 (여전히 명령형)

```kotlin
// ViewBinding을 사용해도 핵심 패턴은 동일: 직접 View를 조작
class CounterActivity : AppCompatActivity() {

    private lateinit var binding: ActivityCounterBinding
    private var counter = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityCounterBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.btnIncrement.setOnClickListener {
            counter++
            updateUI()  // 중복 제거를 위해 함수로 추출
        }

        binding.btnDecrement.setOnClickListener {
            if (counter > 0) {
                counter--
                updateUI()
            }
        }
    }

    // ⚠️ 여전히 명령형: "이 View를 이렇게 바꿔라"를 나열
    private fun updateUI() {
        binding.tvCounter.text = counter.toString()
        binding.tvCounter.setTextColor(
            if (counter >= 5) Color.RED else Color.BLACK
        )
        binding.tvMessage.visibility =
            if (counter >= 5) View.VISIBLE else View.GONE
        binding.tvMessage.text =
            if (counter >= 5) "카운트가 높습니다!" else ""
        binding.btnDecrement.isEnabled = counter > 0
    }
}
```

---

## 3. Android Jetpack Compose 방식

Compose는 Android의 **선언형 UI 프레임워크**입니다. React에서 강하게 영감을 받았으며, 개념적으로 매우 유사합니다.

### 예제 4: 같은 카운터를 Compose로

```kotlin
// CounterScreen.kt
@Composable
fun CounterScreen() {
    // 💡 상태 선언 — React의 useState와 거의 동일한 개념
    var counter by remember { mutableStateOf(0) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        // 💡 선언형: "counter가 이 값일 때, 이렇게 보여라"
        Text(
            text = "$counter",
            fontSize = 48.sp,
            color = if (counter >= 5) Color.Red else Color.Black
        )

        // 💡 조건부 렌더링: 상태에 따라 자동으로 보임/숨김
        if (counter >= 5) {
            Text(
                text = "카운트가 높습니다!",
                fontSize = 16.sp
            )
        }

        Row(
            modifier = Modifier.padding(top = 16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Button(
                onClick = { if (counter > 0) counter-- },
                enabled = counter > 0
            ) {
                Text("-1")
            }

            Button(onClick = { counter++ }) {
                Text("+1")
            }
        }
    }
}
```

### Compose에서 주목할 점

```
┌──────────────────────────────────────────────────────────────┐
│ Compose 선언형 UI의 특징                                      │
│                                                              │
│ 1. remember { mutableStateOf(0) }                            │
│    → 상태를 선언하면 변경 시 자동으로 UI가 갱신됨                │
│    → React의 useState(0)과 거의 동일                          │
│                                                              │
│ 2. if (counter >= 5) { Text(...) }                           │
│    → 조건부 렌더링을 그냥 Kotlin의 if문으로 표현                 │
│    → visibility = GONE 같은 명령이 필요 없음                    │
│                                                              │
│ 3. 이벤트 핸들러 안에서는 상태만 변경                            │
│    → UI 업데이트는 프레임워크가 알아서 처리 (Recomposition)       │
│    → React의 Re-render와 동일한 개념                           │
└──────────────────────────────────────────────────────────────┘
```

### 예제 5: Compose 상태 변경 흐름

```kotlin
// 명령형 (전통 Android)
btnIncrement.setOnClickListener {
    counter++
    tvCounter.text = counter.toString()           // 직접 View 업데이트
    tvCounter.setTextColor(Color.RED)             // 직접 View 업데이트
    tvMessage.visibility = View.VISIBLE           // 직접 View 업데이트
}

// 선언형 (Compose)
Button(onClick = { counter++ }) {                 // 상태만 변경!
    Text("+1")
}
// → counter가 변경되면 Compose가 알아서 recomposition하여
//   counter를 읽는 모든 Composable을 다시 실행함
```

---

## 4. React 방식: Function Component + JSX

이제 React에서 **같은 카운터**를 만들어보겠습니다. Compose를 알고 있다면 매우 익숙하게 느껴질 것입니다.

### 예제 6: React 카운터 컴포넌트

```typescript
// Counter.tsx
import React, { useState } from 'react';

function Counter() {
  // 💡 useState — Compose의 remember { mutableStateOf() }와 동일
  const [counter, setCounter] = useState<number>(0);

  return (
    <div style={styles.container}>
      {/* 💡 JSX 안에서 JavaScript 표현식 사용: {} */}
      <span style={{
        fontSize: '48px',
        color: counter >= 5 ? 'red' : 'black'
      }}>
        {counter}
      </span>

      {/* 💡 조건부 렌더링: && 연산자 사용 */}
      {counter >= 5 && (
        <span style={{ fontSize: '16px' }}>
          카운트가 높습니다!
        </span>
      )}

      <div style={styles.buttonRow}>
        <button
          onClick={() => setCounter(prev => prev - 1)}
          disabled={counter <= 0}
        >
          -1
        </button>
        <button onClick={() => setCounter(prev => prev + 1)}>
          +1
        </button>
      </div>
    </div>
  );
}

const styles = {
  container: {
    display: 'flex',
    flexDirection: 'column' as const,
    alignItems: 'center',
    justifyContent: 'center',
    padding: '16px',
    height: '100vh',
  },
  buttonRow: {
    display: 'flex',
    gap: '8px',
    marginTop: '16px',
  },
};

export default Counter;
```

### 예제 7: React Native 버전 (모바일용)

```typescript
// Counter.tsx (React Native)
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';

function Counter() {
  const [counter, setCounter] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={[
        styles.counterText,
        counter >= 5 && styles.counterTextRed
      ]}>
        {counter}
      </Text>

      {counter >= 5 && (
        <Text style={styles.message}>카운트가 높습니다!</Text>
      )}

      <View style={styles.buttonRow}>
        <TouchableOpacity
          style={[styles.button, counter <= 0 && styles.buttonDisabled]}
          onPress={() => setCounter(prev => prev - 1)}
          disabled={counter <= 0}
        >
          <Text style={styles.buttonText}>-1</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={styles.button}
          onPress={() => setCounter(prev => prev + 1)}
        >
          <Text style={styles.buttonText}>+1</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 16,
  },
  counterText: {
    fontSize: 48,
    color: 'black',
  },
  counterTextRed: {
    color: 'red',
  },
  message: {
    fontSize: 16,
  },
  buttonRow: {
    flexDirection: 'row',
    gap: 8,
    marginTop: 16,
  },
  button: {
    backgroundColor: '#007AFF',
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonDisabled: {
    backgroundColor: '#CCCCCC',
  },
  buttonText: {
    color: 'white',
    fontSize: 18,
  },
});

export default Counter;
```

---

## 5. 세 가지 접근법 나란히 비교

### 예제 8: 사용자 프로필 카드 — 세 가지 방식

**시나리오:** 사용자 이름, 이메일, 온라인 상태를 보여주는 프로필 카드

#### 5-1. Android XML + Kotlin

```xml
<!-- res/layout/item_profile_card.xml -->
<androidx.cardview.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="12dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp">

        <View
            android:id="@+id/statusIndicator"
            android:layout_width="12dp"
            android:layout_height="12dp"
            android:background="@drawable/circle_shape" />

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:orientation="vertical"
            android:layout_marginStart="12dp">

            <TextView
                android:id="@+id/tvName"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textSize="18sp"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/tvEmail"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textSize="14sp"
                android:textColor="#666666" />

            <TextView
                android:id="@+id/tvStatus"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textSize="12sp" />
        </LinearLayout>
    </LinearLayout>
</androidx.cardview.widget.CardView>
```

```kotlin
// ProfileCardViewHolder.kt (RecyclerView 기반)
class ProfileCardViewHolder(view: View) : RecyclerView.ViewHolder(view) {
    private val tvName: TextView = view.findViewById(R.id.tvName)
    private val tvEmail: TextView = view.findViewById(R.id.tvEmail)
    private val tvStatus: TextView = view.findViewById(R.id.tvStatus)
    private val statusIndicator: View = view.findViewById(R.id.statusIndicator)

    fun bind(user: User) {
        tvName.text = user.name
        tvEmail.text = user.email

        // ⚠️ 명령형: 조건마다 직접 View 속성을 설정
        if (user.isOnline) {
            tvStatus.text = "온라인"
            tvStatus.setTextColor(Color.GREEN)
            statusIndicator.background.setTint(Color.GREEN)
        } else {
            tvStatus.text = "오프라인"
            tvStatus.setTextColor(Color.GRAY)
            statusIndicator.background.setTint(Color.GRAY)
        }
    }
}
```

#### 5-2. Jetpack Compose

```kotlin
// ProfileCard.kt (Compose)
@Composable
fun ProfileCard(user: User) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        shape = RoundedCornerShape(12.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            // 💡 선언형: 상태에 따른 색상을 직접 표현식으로
            val statusColor = if (user.isOnline) Color.Green else Color.Gray

            Box(
                modifier = Modifier
                    .size(12.dp)
                    .background(statusColor, CircleShape)
            )

            Spacer(modifier = Modifier.width(12.dp))

            Column {
                Text(text = user.name, fontSize = 18.sp, fontWeight = FontWeight.Bold)
                Text(text = user.email, fontSize = 14.sp, color = Color.Gray)
                Text(
                    text = if (user.isOnline) "온라인" else "오프라인",
                    fontSize = 12.sp,
                    color = statusColor
                )
            }
        }
    }
}
```

#### 5-3. React (TypeScript)

```typescript
// ProfileCard.tsx
interface User {
  name: string;
  email: string;
  isOnline: boolean;
}

function ProfileCard({ name, email, isOnline }: User) {
  const statusColor = isOnline ? '#4CAF50' : '#9E9E9E';
  const statusText = isOnline ? '온라인' : '오프라인';

  return (
    <div style={{
      backgroundColor: 'white',
      borderRadius: '12px',
      padding: '16px',
      margin: '8px',
      boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
      display: 'flex',
      alignItems: 'center',
    }}>
      <div style={{
        width: '12px',
        height: '12px',
        borderRadius: '50%',
        backgroundColor: statusColor,
      }} />

      <div style={{ marginLeft: '12px' }}>
        <div style={{ fontSize: '18px', fontWeight: 'bold' }}>{name}</div>
        <div style={{ fontSize: '14px', color: '#666' }}>{email}</div>
        <div style={{ fontSize: '12px', color: statusColor }}>{statusText}</div>
      </div>
    </div>
  );
}
```

### 비교 요약 테이블

| 특성 | Android XML | Jetpack Compose | React |
|------|------------|----------------|-------|
| UI 정의 | XML 파일 (별도) | Kotlin 코드 내 | JSX (TypeScript 내) |
| 상태 관리 | 수동 (변수 + updateUI) | `mutableStateOf` | `useState` |
| View 참조 | `findViewById`/ViewBinding | 불필요 | 불필요 |
| 조건부 표시 | `visibility = GONE` | Kotlin `if` 문 | JSX `{&&}` / 삼항 |
| 리스트 표시 | RecyclerView + Adapter | `LazyColumn` | `map()` |
| 패러다임 | 명령형 | 선언형 | 선언형 |
| 상태→UI 동기화 | 수동 | 자동 (Recomposition) | 자동 (Re-render) |

---

## 6. 왜 선언형인가? 이점 분석

### 6-1. 예측 가능성 (Predictability)

```
┌──────────────────────────────────────────────────────────┐
│ 명령형에서의 상태 불일치 문제                                │
│                                                          │
│   상태: counter = 3                                       │
│   화면: "5" (← 이전 업데이트 로직에서 버그 발생)             │
│                                                          │
│   상태와 화면이 일치하지 않을 수 있음!                       │
│   → View를 직접 조작하므로, 어느 시점에서든 꼬일 수 있음     │
├──────────────────────────────────────────────────────────┤
│ 선언형에서는 불가능                                        │
│                                                          │
│   상태: counter = 3                                       │
│   화면: 반드시 "3" (← 상태가 곧 UI)                        │
│                                                          │
│   UI = f(state) ← 이 공식이 항상 성립                      │
│   → 같은 상태이면 항상 같은 화면                            │
└──────────────────────────────────────────────────────────┘
```

### 예제 9: UI = f(state) 원칙

```typescript
// React에서 UI는 상태의 "순수 함수"
// 같은 입력(state/props)이면 항상 같은 출력(UI)

function Greeting({ name, hour }: { name: string; hour: number }) {
  // 이 함수는 순수 함수처럼 동작:
  // name="김철수", hour=14 → 항상 "안녕하세요, 김철수님"
  // name="김철수", hour=22 → 항상 "좋은 밤이에요, 김철수님"

  const greeting = hour < 12 ? '좋은 아침이에요' :
                   hour < 18 ? '안녕하세요' :
                               '좋은 밤이에요';

  return <h1>{greeting}, {name}님</h1>;
}
```

### 6-2. 테스트 용이성 (Testability)

```typescript
// 예제 10: 선언형 컴포넌트는 테스트가 단순
// input → output을 확인하면 됨

// 컴포넌트
function StatusBadge({ isActive }: { isActive: boolean }) {
  return (
    <span style={{ color: isActive ? 'green' : 'gray' }}>
      {isActive ? '활성' : '비활성'}
    </span>
  );
}

// 테스트 (개념적 pseudocode)
// render(<StatusBadge isActive={true} />)
// expect(screen.getByText('활성')).toHaveStyle({ color: 'green' })
//
// render(<StatusBadge isActive={false} />)
// expect(screen.getByText('비활성')).toHaveStyle({ color: 'gray' })
```

### 6-3. 재사용성 (Reusability)

### 예제 11: 컴포넌트 재사용

```typescript
// 한 번 정의하면 어디서든 다른 props로 재사용
function App() {
  return (
    <div>
      {/* 같은 ProfileCard를 다른 데이터로 여러 번 사용 */}
      <ProfileCard name="김철수" email="cs@mail.com" isOnline={true} />
      <ProfileCard name="이영희" email="yh@mail.com" isOnline={false} />
      <ProfileCard name="박민수" email="ms@mail.com" isOnline={true} />
    </div>
  );
}

// Android XML에서는?
// → 같은 레이아웃을 재사용하려면 Custom View나 include 태그를 써야 하고,
//   데이터 바인딩을 위한 별도의 코드가 필요
```

### 6-4. 복잡한 UI 상태 관리

### 예제 12: 여러 상태가 얽힌 폼 — 선언형의 장점

```typescript
function RegistrationForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [agreed, setAgreed] = useState(false);

  // 모든 조건이 만족되었는지 자동 계산
  const isValid = name.length > 0
    && email.includes('@')
    && password.length >= 8
    && agreed;

  // 비밀번호 강도도 자동 계산
  const passwordStrength =
    password.length === 0 ? 'none' :
    password.length < 6 ? 'weak' :
    password.length < 10 ? 'medium' : 'strong';

  return (
    <form>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
      />

      {/* 비밀번호 강도 표시: 상태에서 자동 파생 */}
      {passwordStrength !== 'none' && (
        <div style={{ color:
          passwordStrength === 'weak' ? 'red' :
          passwordStrength === 'medium' ? 'orange' : 'green'
        }}>
          비밀번호 강도: {passwordStrength}
        </div>
      )}

      <label>
        <input
          type="checkbox"
          checked={agreed}
          onChange={e => setAgreed(e.target.checked)}
        />
        약관에 동의합니다
      </label>

      {/* 유효성에 따라 버튼 자동 활성화/비활성화 */}
      <button disabled={!isValid}>가입하기</button>
    </form>
  );
}

// ⬆️ 명령형이었다면?
// → name이 바뀔 때 isValid 재계산 + 버튼 enabled 업데이트
// → email이 바뀔 때 isValid 재계산 + 버튼 enabled 업데이트
// → password가 바뀔 때 isValid 재계산 + 버튼 enabled 업데이트
//   + passwordStrength 재계산 + 강도 텍스트 업데이트 + 색상 업데이트
// → agreed가 바뀔 때 isValid 재계산 + 버튼 enabled 업데이트
// 총 10개 이상의 수동 업데이트 코드가 필요!
```

---

## 7. Virtual DOM 개념

### Compose의 Recomposition과 React의 Re-render 비교

React의 핵심 메커니즘인 **Virtual DOM**은 Compose의 **Recomposition**과 목적이 같습니다: **실제 화면에 최소한의 변경만 적용하기.**

```
┌────────────────────────────────────────────────────────────────┐
│                    React의 Virtual DOM 동작 과정                │
│                                                                │
│  1단계: 컴포넌트 함수 실행 → Virtual DOM 트리 생성               │
│                                                                │
│         function App() {                                       │
│           const [count, setCount] = useState(0);               │
│           return (                                             │
│             <div>                                              │
│               <h1>카운터</h1>                                   │
│               <p>{count}</p>      ← count=0일 때 "0"           │
│               <button>+1</button>                              │
│             </div>                                             │
│           );                                                   │
│         }                                                      │
│                                                                │
│         Virtual DOM (v1):                                      │
│         div                                                    │
│          ├── h1: "카운터"                                       │
│          ├── p: "0"                                            │
│          └── button: "+1"                                      │
│                                                                │
│  2단계: setCount(1) 호출 → 함수 다시 실행 → 새 Virtual DOM      │
│                                                                │
│         Virtual DOM (v2):                                      │
│         div                                                    │
│          ├── h1: "카운터"        ← 변경 없음                    │
│          ├── p: "1"             ← 변경됨!                      │
│          └── button: "+1"       ← 변경 없음                    │
│                                                                │
│  3단계: Diff 비교 (Reconciliation)                              │
│                                                                │
│         v1 vs v2:                                              │
│          ├── h1: "카운터" = "카운터"  → 스킵                    │
│          ├── p: "0" ≠ "1"           → ⚡ 업데이트 필요!         │
│          └── button: "+1" = "+1"    → 스킵                     │
│                                                                │
│  4단계: 실제 DOM에 최소 변경만 적용                               │
│                                                                │
│         실제 DOM: p 요소의 텍스트만 "0" → "1"로 변경              │
│         (나머지 요소는 건드리지 않음)                              │
└────────────────────────────────────────────────────────────────┘
```

### 예제 13: Compose Recomposition vs React Re-render

```kotlin
// Compose: Recomposition
// counter가 변경되면 counter를 "읽는" Composable만 다시 실행
@Composable
fun CounterScreen() {
    var counter by remember { mutableStateOf(0) }

    Column {
        Header()            // ← counter를 읽지 않음 → recomposition 안 됨
        CounterDisplay(counter)  // ← counter를 읽음 → recomposition 됨!
        IncrementButton { counter++ }  // ← counter를 읽지 않음 → 스킵
    }
}
```

```typescript
// React: Virtual DOM + Re-render
// 부모 컴포넌트가 re-render되면 자식도 기본적으로 re-render
// (하지만 Virtual DOM diff에서 변경이 없으면 실제 DOM은 안 건드림)
function CounterScreen() {
  const [counter, setCounter] = useState(0);

  return (
    <div>
      <Header />                    {/* re-render되지만 DOM 변경 없으면 스킵 */}
      <CounterDisplay count={counter} />  {/* props 변경 → DOM 업데이트 */}
      <IncrementButton onIncrement={() => setCounter(c => c + 1)} />
    </div>
  );
}
```

### 핵심 차이점

```
┌───────────────────────────────┬──────────────────────────────────┐
│     Jetpack Compose           │          React                   │
├───────────────────────────────┼──────────────────────────────────┤
│ Recomposition                 │ Re-render + Virtual DOM Diff     │
│                               │                                  │
│ 상태를 읽는 Composable만      │ 부모가 re-render되면 자식도       │
│ 다시 실행됨 (정밀한 추적)      │ 함수 재실행 (넓은 범위)           │
│                               │                                  │
│ Compose 컴파일러가             │ React.memo()로 최적화 가능       │
│ 자동으로 최적화               │ (수동으로 지정해야 함)             │
│                               │                                  │
│ Snapshot System으로            │ Virtual DOM diff로               │
│ 변경 감지                     │ 최소 DOM 업데이트 계산            │
└───────────────────────────────┴──────────────────────────────────┘
```

### 예제 14: React.memo — Compose의 스마트 Recomposition 흉내

```typescript
// React.memo를 사용하면 props가 변경되지 않은 경우 re-render를 건너뜀
// Compose에서는 이것이 기본 동작이지만, React에서는 명시적으로 해야 함

import React, { memo, useState } from 'react';

// memo로 감싸면: props가 같으면 re-render 스킵
const ExpensiveHeader = memo(function Header() {
  console.log('Header 렌더링'); // memo가 없으면 매 re-render마다 실행됨
  return <h1>복잡한 헤더 컴포넌트</h1>;
});

const CounterDisplay = memo(function CounterDisplay({ count }: { count: number }) {
  console.log('CounterDisplay 렌더링'); // count가 바뀔 때만 실행
  return <p style={{ fontSize: '48px' }}>{count}</p>;
});

function App() {
  const [counter, setCounter] = useState(0);

  return (
    <div>
      <ExpensiveHeader />                      {/* counter가 바뀌어도 스킵 */}
      <CounterDisplay count={counter} />        {/* counter가 바뀔 때만 렌더 */}
      <button onClick={() => setCounter(c => c + 1)}>+1</button>
    </div>
  );
}
```

---

## 8. 렌더링 생명주기

### 초기 렌더 → 상태 변경 → 리렌더 흐름

```
┌──────────────────────────────────────────────────────────────────┐
│                    React 렌더링 생명주기                          │
│                                                                  │
│  ① 초기 렌더 (Mount)                                             │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ 컴포넌트 함수 최초 실행                                    │     │
│  │ → useState(0) → counter = 0으로 초기화                    │     │
│  │ → JSX 반환 → Virtual DOM 생성                            │     │
│  │ → Virtual DOM → 실제 DOM에 반영                          │     │
│  │ → 화면에 표시                                             │     │
│  │ → useEffect 실행 (있는 경우)                              │     │
│  └─────────────────────────────────────────────────────────┘     │
│                          │                                       │
│                          ▼                                       │
│  ② 이벤트 발생 (사용자가 버튼 클릭)                               │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ onClick 핸들러 실행                                       │     │
│  │ → setCounter(prev => prev + 1) 호출                      │     │
│  │ → React가 "상태가 변경됨"을 인지                          │     │
│  │ → 리렌더 예약 (즉시가 아니라 배치 처리)                    │     │
│  └─────────────────────────────────────────────────────────┘     │
│                          │                                       │
│                          ▼                                       │
│  ③ 리렌더 (Re-render / Update)                                   │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ 컴포넌트 함수 다시 실행                                    │     │
│  │ → useState() → counter = 1 (업데이트된 값)                │     │
│  │ → JSX 반환 → 새 Virtual DOM 생성                         │     │
│  │ → 이전 Virtual DOM과 비교 (Diffing)                       │     │
│  │ → 변경된 부분만 실제 DOM에 반영                           │     │
│  │ → useEffect 정리 + 재실행 (의존성 변경 시)                │     │
│  └─────────────────────────────────────────────────────────┘     │
│                          │                                       │
│                   ②←─────┘ (반복)                                │
│                                                                  │
│  ④ 언마운트 (Unmount)                                            │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ 컴포넌트가 화면에서 제거됨                                 │     │
│  │ → useEffect의 cleanup 함수 실행                           │     │
│  │ → 메모리 정리                                             │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 예제 15: 전체 생명주기를 보여주는 컴포넌트

```typescript
import React, { useState, useEffect } from 'react';

function LifecycleDemo() {
  console.log('🔄 컴포넌트 함수 실행 (렌더)');

  const [count, setCount] = useState(() => {
    console.log('1️⃣ useState 초기화 (최초 1회만)');
    return 0;
  });

  const [visible, setVisible] = useState(true);

  useEffect(() => {
    console.log('3️⃣ useEffect 실행 (마운트 또는 count 변경)');
    console.log(`   현재 count: ${count}`);

    return () => {
      console.log('🧹 useEffect cleanup (다음 effect 전 또는 언마운트)');
    };
  }, [count]);

  console.log('2️⃣ JSX 반환 직전');

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => {
        console.log('👆 버튼 클릭 → setCount 호출');
        setCount(c => c + 1);
      }}>
        +1
      </button>
    </div>
  );
}

// 콘솔 출력 순서 (초기 마운트):
// 🔄 컴포넌트 함수 실행 (렌더)
// 1️⃣ useState 초기화 (최초 1회만)
// 2️⃣ JSX 반환 직전
// 3️⃣ useEffect 실행 (마운트 또는 count 변경)
//    현재 count: 0

// 콘솔 출력 순서 (버튼 클릭 후):
// 👆 버튼 클릭 → setCount 호출
// 🔄 컴포넌트 함수 실행 (렌더)        ← 함수 전체가 다시 실행!
// 2️⃣ JSX 반환 직전                   ← useState 초기화는 스킵됨
// 🧹 useEffect cleanup                ← 이전 effect 정리
// 3️⃣ useEffect 실행                  ← 새 effect 실행
//    현재 count: 1
```

### Android Lifecycle과의 매핑

```
┌──────────────────────────────┬──────────────────────────────────┐
│     Android Lifecycle        │     React Lifecycle               │
├──────────────────────────────┼──────────────────────────────────┤
│ onCreate()                   │ 컴포넌트 함수 최초 실행            │
│                              │ + useEffect(fn, [])              │
│                              │                                  │
│ onStart() / onResume()       │ (해당 없음 — 웹은 항상 "보임")     │
│                              │                                  │
│ onPause() / onStop()         │ (해당 없음)                       │
│                              │                                  │
│ onDestroy()                  │ useEffect cleanup 함수           │
│                              │                                  │
│ onSaveInstanceState()        │ (별도 상태 관리 라이브러리 사용)    │
│                              │                                  │
│ 화면 회전 → 재생성            │ key 변경 → 언마운트 + 재마운트     │
│                              │                                  │
│ LiveData.observe()           │ useEffect(fn, [dependency])      │
│ → 데이터 변경 감지             │ → 의존성 변경 시 재실행           │
│                              │                                  │
│ ViewModel 유지               │ 부모 컴포넌트의 state 유지         │
│ (설정 변경 시에도)             │ (언마운트 안 되면 유지)            │
└──────────────────────────────┴──────────────────────────────────┘
```

### 예제 16: 컴포넌트 마운트/언마운트 시각화

```typescript
function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    // 마운트: 타이머 시작 (Android의 onCreate에서 시작하는 것과 유사)
    const interval = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);

    // 언마운트: 타이머 정리 (Android의 onDestroy에서 정리하는 것과 유사)
    return () => {
      clearInterval(interval);
    };
  }, []); // 빈 배열 = 마운트 시 1회만 실행

  return <p>{seconds}초 경과</p>;
}

// 부모에서 조건부로 마운트/언마운트
function App() {
  const [showTimer, setShowTimer] = useState(true);

  return (
    <div>
      <button onClick={() => setShowTimer(!showTimer)}>
        타이머 {showTimer ? '숨기기' : '보이기'}
      </button>

      {/* showTimer가 false가 되면 Timer가 언마운트 → cleanup 실행 */}
      {showTimer && <Timer />}
    </div>
  );
}
```

---

## 핵심 요약

```
┌──────────────────────────────────────────────────────────────────┐
│                    기억해야 할 핵심 개념                          │
│                                                                  │
│  1. UI = f(state)                                                │
│     → UI는 상태의 함수. 같은 상태 = 같은 화면.                    │
│                                                                  │
│  2. 상태만 바꾸면 UI는 자동으로 따라옴                              │
│     → findViewById → setText 같은 명령형 코드 불필요               │
│                                                                  │
│  3. Virtual DOM이 효율적인 업데이트를 보장                         │
│     → 전체를 다시 그리는 것처럼 코드를 작성하지만,                  │
│       실제로는 변경된 부분만 업데이트                               │
│                                                                  │
│  4. React ≈ Compose                                              │
│     → useState ≈ mutableStateOf                                  │
│     → Re-render ≈ Recomposition                                  │
│     → JSX ≈ @Composable 함수의 반환값                             │
│     → Compose를 안다면 React는 80% 이해한 것!                     │
│                                                                  │
│  5. 선언형의 핵심 이점                                            │
│     → 예측 가능성, 테스트 용이성, 재사용성, 유지보수성              │
└──────────────────────────────────────────────────────────────────┘
```

---

> **다음 문서:** [02-components-and-jsx.md](./02-components-and-jsx.md) — 컴포넌트와 JSX 완전 가이드
