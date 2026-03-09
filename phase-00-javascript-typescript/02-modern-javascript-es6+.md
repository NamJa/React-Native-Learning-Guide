# 모던 JavaScript (ES6+) — 핵심 문법 완전 정리

> ES6(ES2015) 이후 추가된 현대 JavaScript 문법을 Kotlin과 비교하며 상세하게 설명합니다.
> React Native 개발에서 매일 사용하는 문법들이므로 반드시 숙지해야 합니다.

---

## 1. 구조 분해 할당 (Destructuring)

구조 분해 할당은 배열이나 객체의 값을 개별 변수에 간결하게 추출하는 문법입니다.
React Native에서 **가장 빈번하게** 사용하는 문법 중 하나입니다.

### 1-1. 객체 구조 분해 (Object Destructuring)

```javascript
// JavaScript — 기본 객체 구조 분해
const user = {
  name: "홍길동",
  age: 30,
  email: "hong@example.com",
  address: {
    city: "서울",
    zip: "06000"
  }
};

// 기존 방식 — 하나씩 꺼내기
const name1 = user.name;
const age1 = user.age;

// 구조 분해 — 한 번에 여러 개 추출
const { name, age, email } = user;
console.log(name);  // "홍길동"
console.log(age);   // 30
console.log(email); // "hong@example.com"

// 변수 이름 변경 (rename)
const { name: userName, age: userAge } = user;
console.log(userName); // "홍길동"
console.log(userAge);  // 30
// 원래 속성명 name은 변수로 생성되지 않고, userName이라는 이름으로 생성됨

// 기본값 설정 — 속성이 undefined일 때 사용됨
const { name: n, phone = "010-0000-0000" } = user;
console.log(n);     // "홍길동"
console.log(phone); // "010-0000-0000" — user에 phone이 없으므로 기본값 사용

// 중첩 객체 구조 분해
const { address: { city, zip } } = user;
console.log(city); // "서울"
console.log(zip);  // "06000"
// 주의: address 자체는 변수로 생성되지 않음, city와 zip만 생성됨

// 나머지 속성 모으기 (rest)
const { name: nm, ...rest } = user;
console.log(nm);   // "홍길동"
console.log(rest); // { age: 30, email: "hong@example.com", address: { city: "서울", zip: "06000" } }
```

```kotlin
// Kotlin — 구조 분해는 data class의 componentN() 함수를 통해 지원
data class User(val name: String, val age: Int, val email: String)
val user = User("홍길동", 30, "hong@example.com")

// Kotlin 구조 분해 — 순서 기반 (JavaScript는 이름 기반!)
val (name, age, email) = user
println(name)  // "홍길동"
println(age)   // 30

// Kotlin에서는 이름 변경, 기본값, 중첩 구조 분해를 직접 지원하지 않음
// JavaScript의 구조 분해가 훨씬 유연함
```

**핵심 차이**: JavaScript 객체 구조 분해는 **속성 이름** 기반이고, Kotlin은 **순서** 기반입니다.

### 1-2. 배열 구조 분해 (Array Destructuring)

```javascript
// JavaScript — 배열 구조 분해 (위치 기반)
const colors = ["빨강", "파랑", "초록", "노랑", "보라"];

// 기본
const [first, second, third] = colors;
console.log(first);  // "빨강"
console.log(second); // "파랑"
console.log(third);  // "초록"

// 요소 건너뛰기 — 쉼표로 빈 자리 표시
const [, , thirdColor] = colors;
console.log(thirdColor); // "초록"

// 기본값
const [a, b, c, d, e, f = "검정"] = colors;
console.log(f); // "검정" — 6번째 요소가 없으므로 기본값 사용

// 나머지 요소 모으기
const [head, ...tail] = colors;
console.log(head); // "빨강"
console.log(tail); // ["파랑", "초록", "노랑", "보라"]

// 변수 교환 — 임시 변수 없이!
let x = 1;
let y = 2;
[x, y] = [y, x];
console.log(x); // 2
console.log(y); // 1

// React에서의 실제 사용 — useState 훅
// const [count, setCount] = useState(0);
// useState가 [현재값, 설정함수] 배열을 반환하고, 구조 분해로 받음
```

```kotlin
// Kotlin — 유사하지만 제한적
val colors = listOf("빨강", "파랑", "초록")
val (first, second, third) = colors
println(first) // "빨강"

// Kotlin에서는 건너뛰기, 나머지 모으기(...) 불가
// val (_, _, third) = colors // _ 로 무시는 가능
```

### 1-3. 함수 매개변수에서의 구조 분해

```javascript
// JavaScript — 함수 인자에서 바로 구조 분해 (React Native에서 매우 자주 사용)
function printUser({ name, age, email = "없음" }) {
  console.log(`이름: ${name}, 나이: ${age}, 이메일: ${email}`);
}

printUser({ name: "홍길동", age: 30 });
// "이름: 홍길동, 나이: 30, 이메일: 없음"

// React Native 컴포넌트에서의 실제 패턴
// function UserCard({ name, age, onPress }) {
//   return (
//     <TouchableOpacity onPress={onPress}>
//       <Text>{name} ({age})</Text>
//     </TouchableOpacity>
//   );
// }
```

```kotlin
// Kotlin — 동일한 패턴이 없으므로 data class를 매개변수로 받음
data class UserInfo(val name: String, val age: Int, val email: String = "없음")

fun printUser(user: UserInfo) {
    println("이름: ${user.name}, 나이: ${user.age}, 이메일: ${user.email}")
}
```

```exercise
type: code-arrange
question: "배열 구조 분해 할당으로 firstName과 lastName을 추출하는 코드를 조립하세요"
tokens:
  - "const"
  - "[firstName, lastName]"
  - "="
  - "['Kim', 'React']"
distractors:
  - "let"
  - "{firstName, lastName}"
  - "new Array"
answer: ["const", "[firstName, lastName]", "=", "['Kim', 'React']"]
hint: "배열 구조 분해는 대괄호[]를 사용합니다"
xp: 8
```

---

## 2. 스프레드 연산자 (Spread Operator: ...)

`...`은 배열이나 객체를 **펼치는** 연산자입니다. 구조 분해의 나머지(`...rest`)와 문법은 같지만 역할이 다릅니다.

### 2-1. 배열 스프레드

```javascript
// JavaScript — 배열 스프레드
const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];

// 배열 합치기
const combined = [...arr1, ...arr2];
console.log(combined); // [1, 2, 3, 4, 5, 6]

// 배열 복사 (얕은 복사)
const copy = [...arr1];
console.log(copy);     // [1, 2, 3]
copy.push(99);
console.log(arr1);     // [1, 2, 3] — 원본 유지
console.log(copy);     // [1, 2, 3, 99]

// 중간에 삽입
const withInsert = [0, ...arr1, 3.5, ...arr2, 7];
console.log(withInsert); // [0, 1, 2, 3, 3.5, 4, 5, 6, 7]

// 문자열을 배열로 분해
const chars = [..."Hello"];
console.log(chars); // ["H", "e", "l", "l", "o"]
```

```kotlin
// Kotlin — spread 연산자 *
val arr1 = intArrayOf(1, 2, 3)
val arr2 = intArrayOf(4, 5, 6)

// Kotlin의 *는 vararg에만 사용 가능
fun sum(vararg numbers: Int) = numbers.sum()
println(sum(*arr1)) // 6

// 배열 합치기는 + 연산자 또는 .plus() 사용
val combined = arr1 + arr2 // [1, 2, 3, 4, 5, 6]

// 리스트 합치기
val list1 = listOf(1, 2, 3)
val list2 = listOf(4, 5, 6)
val combinedList = list1 + list2 // [1, 2, 3, 4, 5, 6]
```

### 2-2. 객체 스프레드 — React Native에서 핵심 패턴

```javascript
// JavaScript — 객체 스프레드
const defaults = {
  theme: "dark",
  fontSize: 16,
  language: "ko"
};

const userPrefs = {
  fontSize: 20,
  showNotifications: true
};

// 객체 병합 — 뒤에 오는 것이 우선 (덮어쓰기)
const settings = { ...defaults, ...userPrefs };
console.log(settings);
// { theme: "dark", fontSize: 20, language: "ko", showNotifications: true }
// fontSize가 16 → 20으로 덮어씌워짐

// 특정 속성만 변경 (불변성 유지 패턴 — React에서 매우 중요!)
const user = { name: "홍길동", age: 30, city: "서울" };
const updatedUser = { ...user, age: 31 };
console.log(user);        // { name: "홍길동", age: 30, city: "서울" } — 원본 유지
console.log(updatedUser); // { name: "홍길동", age: 31, city: "서울" } — 새 객체

// 중첩 객체 주의 — 얕은 복사!
const original = {
  name: "홍길동",
  address: { city: "서울", zip: "06000" }
};
const copied = { ...original };
copied.address.city = "부산"; // 원본도 변경됨! (얕은 복사)
console.log(original.address.city); // "부산" (!)

// 깊은 복사가 필요하면:
const deepCopied = JSON.parse(JSON.stringify(original));
// 또는 structuredClone(original) (최신 환경)
```

```kotlin
// Kotlin — 직접적인 객체 스프레드 연산자는 없음
// data class의 copy() 메서드로 유사하게 구현
data class User(val name: String, val age: Int, val city: String)
val user = User("홍길동", 30, "서울")
val updatedUser = user.copy(age = 31)
println(updatedUser) // User(name=홍길동, age=31, city=서울)

// Map 병합
val defaults = mapOf("theme" to "dark", "fontSize" to 16)
val userPrefs = mapOf("fontSize" to 20)
val settings = defaults + userPrefs // fontSize가 20으로 덮어씌워짐
```

```exercise
type: output-predict
question: "다음 코드의 출력은?"
code: |
  const arr = [1, 2, 3];
  const copy = [...arr, 4, 5];
  console.log(copy.length);
options:
  - "3"
  - "5"
  - "8"
  - "에러 발생"
answer: "5"
explanation: "스프레드 연산자로 arr의 요소를 펼친 뒤 4, 5를 추가하므로 총 5개의 요소가 됩니다."
xp: 6
```

---

## 3. 템플릿 리터럴과 태그드 템플릿

### 3-1. 태그드 템플릿 (Tagged Templates)

일반 템플릿 리터럴(`${}`)은 01편에서 다뤘으므로, 여기서는 **태그드 템플릿**을 설명합니다.
React Native에서 styled-components 등의 라이브러리가 이 문법을 사용합니다.

```javascript
// JavaScript — 태그드 템플릿
// 함수 이름 뒤에 백틱을 바로 붙이면, 해당 함수가 템플릿을 처리
function highlight(strings, ...values) {
  // strings: 문자열 조각 배열, values: ${} 표현식 결과 배열
  let result = "";
  strings.forEach((str, i) => {
    result += str;
    if (i < values.length) {
      result += `【${values[i]}】`; // 값을 강조 표시
    }
  });
  return result;
}

const name = "홍길동";
const age = 30;
const output = highlight`이름은 ${name}이고 나이는 ${age}입니다.`;
console.log(output);
// "이름은 【홍길동】이고 나이는 【30】입니다."

// 실제 사용 예: styled-components (React Native)
// const StyledText = styled.Text`
//   font-size: 16px;
//   color: ${props => props.primary ? 'blue' : 'black'};
// `;
```

```kotlin
// Kotlin — 태그드 템플릿에 해당하는 기능 없음
// buildString 등으로 유사하게 구현은 가능하지만 동일하지 않음
```

---

## 4. 향상된 객체 리터럴 (Enhanced Object Literals)

```javascript
// JavaScript — ES6에서 추가된 간결한 객체 표기법

// 1) 속성 축약 (Property Shorthand)
const name = "홍길동";
const age = 30;

// ES5 방식
const userOld = { name: name, age: age };

// ES6 방식 — 변수명과 속성명이 같으면 축약 가능
const user = { name, age };
console.log(user); // { name: "홍길동", age: 30 }

// 2) 메서드 축약 (Method Shorthand)
const calculator = {
  // ES5 방식
  addOld: function(a, b) { return a + b; },
  // ES6 방식
  add(a, b) { return a + b; },
  subtract(a, b) { return a - b; }
};
console.log(calculator.add(1, 2)); // 3

// 3) 계산된 속성명 (Computed Property Names)
const key = "email";
const obj = {
  [key]: "hong@example.com",           // email: "hong@example.com"
  [`${key}Verified`]: true,            // emailVerified: true
  [`get${key[0].toUpperCase() + key.slice(1)}`]() { // getEmail()
    return this[key];
  }
};
console.log(obj.email);           // "hong@example.com"
console.log(obj.emailVerified);   // true
console.log(obj.getEmail());      // "hong@example.com"
```

```kotlin
// Kotlin — 객체 리터럴 축약은 없음
// mapOf를 사용하거나, data class를 정의해야 함
val name = "홍길동"
val age = 30
val user = mapOf("name" to name, "age" to age)
```

---

## 5. Optional Chaining (?.)과 Nullish Coalescing (??)

01편에서 기본을 다뤘으므로, 여기서는 **고급 사용법**을 다룹니다.

```javascript
// JavaScript — Optional Chaining 고급 사용법

const users = [
  { name: "홍길동", scores: [90, 85, 95] },
  { name: "김철수" },
  null
];

// 배열 인덱스 접근에도 사용
console.log(users[0]?.scores?.[2]); // 95
console.log(users[1]?.scores?.[0]); // undefined — scores가 없음
console.log(users[2]?.name);        // undefined — null

// 메서드 호출에 사용
const response = {
  data: {
    items: [1, 2, 3],
    getTotal() { return this.items.length; }
  }
};
console.log(response.data?.getTotal?.()); // 3
console.log(response.meta?.getTotal?.()); // undefined

// ?? 와 ?. 조합 — 매우 실용적인 패턴
const config = {
  api: {
    timeout: 0 // 0은 유효한 값!
  }
};

// || 를 사용하면 0이 falsy라 기본값으로 대체됨 (버그!)
const timeout1 = config.api?.timeout || 5000;
console.log(timeout1); // 5000 — 잘못된 결과!

// ?? 를 사용하면 null/undefined일 때만 대체
const timeout2 = config.api?.timeout ?? 5000;
console.log(timeout2); // 0 — 올바른 결과!

// 깊은 속성 접근에서의 기본값 지정
const userName = response?.data?.user?.name ?? "익명";
console.log(userName); // "익명"
```

---

## 6. 모듈 시스템 (Modules: import / export)

### 6-1. Named Export / Import

```javascript
// === math.js ===
// 이름 있는 내보내기 (여러 개 가능)
export const PI = 3.14159;
export const E = 2.71828;

export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

// 또는 한꺼번에 내보내기
const multiply = (a, b) => a * b;
const divide = (a, b) => a / b;
export { multiply, divide };
```

```javascript
// === app.js ===
// 이름 있는 가져오기 — 중괄호 사용, 이름이 일치해야 함
import { PI, add, subtract } from './math.js';
console.log(PI);          // 3.14159
console.log(add(1, 2));   // 3

// 이름 변경하여 가져오기
import { add as sum, subtract as minus } from './math.js';
console.log(sum(1, 2)); // 3

// 전체 가져오기 — 네임스페이스로 사용
import * as MathUtils from './math.js';
console.log(MathUtils.PI);        // 3.14159
console.log(MathUtils.add(1, 2)); // 3
```

### 6-2. Default Export / Import

```javascript
// === UserCard.js ===
// 기본 내보내기 (파일당 1개만 가능)
export default function UserCard({ name, age }) {
  // React Native 컴포넌트
  return `${name} (${age})`;
}

// 또는
function UserCard({ name, age }) {
  return `${name} (${age})`;
}
export default UserCard;
```

```javascript
// === app.js ===
// 기본 가져오기 — 중괄호 없음, 아무 이름으로 가져올 수 있음
import UserCard from './UserCard.js';
import MyCard from './UserCard.js'; // 다른 이름으로도 가져올 수 있음

// 기본 + 이름 있는 내보내기를 동시에 사용
// === api.js ===
// export default class ApiClient { ... }
// export const BASE_URL = "https://api.example.com";
// export function handleError(err) { ... }

// 동시에 가져오기
// import ApiClient, { BASE_URL, handleError } from './api.js';
```

### 6-3. Re-export (재내보내기)

```javascript
// === index.js (배럴 파일) ===
// 여러 모듈을 하나로 모아서 재내보내기
export { default as UserCard } from './UserCard.js';
export { default as ProductCard } from './ProductCard.js';
export { formatDate, formatCurrency } from './utils.js';
export * from './constants.js'; // 모든 named export를 재내보내기

// 사용하는 쪽에서:
// import { UserCard, ProductCard, formatDate } from './components';
// (index.js는 생략 가능)
```

```kotlin
// Kotlin — import 비교
// Kotlin은 패키지 기반 import
import com.example.math.add
import com.example.math.PI
import com.example.math.add as sum // 이름 변경 가능
import com.example.math.* // 와일드카드 import

// Kotlin에는 default export 개념이 없음
// Kotlin에는 re-export 개념이 없음
```

**핵심 차이**:
- JavaScript: 파일 단위로 export/import, default export 가능
- Kotlin: 패키지 단위로 import, default export 없음

---

## 7. 배열 메서드 — React Native 개발의 핵심

배열 메서드는 React Native에서 데이터를 가공하여 UI에 표시할 때 **매일 사용**합니다.

### 7-1. map() — 변환

```javascript
// JavaScript — 배열의 각 요소를 변환하여 새 배열 반환
const numbers = [1, 2, 3, 4, 5];
const doubled = numbers.map(n => n * 2);
console.log(doubled); // [2, 4, 6, 8, 10]
// 원본 유지: console.log(numbers); // [1, 2, 3, 4, 5]

// 객체 배열 변환 — React Native에서 가장 흔한 패턴
const users = [
  { id: 1, firstName: "길동", lastName: "홍" },
  { id: 2, firstName: "철수", lastName: "김" },
];

const fullNames = users.map(user => ({
  id: user.id,
  fullName: `${user.lastName}${user.firstName}`
}));
console.log(fullNames);
// [{ id: 1, fullName: "홍길동" }, { id: 2, fullName: "김철수" }]

// 인덱스 사용
const indexed = numbers.map((n, index) => `${index}: ${n}`);
console.log(indexed); // ["0: 1", "1: 2", "2: 3", "3: 4", "4: 5"]
```

```kotlin
// Kotlin
val numbers = listOf(1, 2, 3, 4, 5)
val doubled = numbers.map { it * 2 } // [2, 4, 6, 8, 10]

val indexed = numbers.mapIndexed { index, n -> "$index: $n" }
```

### 7-2. filter() — 필터링

```javascript
// JavaScript — 조건을 만족하는 요소만 모은 새 배열 반환
const numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

const evens = numbers.filter(n => n % 2 === 0);
console.log(evens); // [2, 4, 6, 8, 10]

const bigNumbers = numbers.filter(n => n > 5);
console.log(bigNumbers); // [6, 7, 8, 9, 10]

// 객체 배열 필터링
const products = [
  { name: "노트북", price: 1500000, inStock: true },
  { name: "키보드", price: 80000, inStock: false },
  { name: "마우스", price: 50000, inStock: true },
  { name: "모니터", price: 500000, inStock: true },
];

const affordable = products.filter(p => p.price < 100000 && p.inStock);
console.log(affordable);
// [{ name: "마우스", price: 50000, inStock: true }]
```

```kotlin
// Kotlin
val evens = numbers.filter { it % 2 == 0 }
```

### 7-3. reduce() — 누적

```javascript
// JavaScript — 배열을 단일 값으로 축소
// reduce(콜백(누적값, 현재값, 인덱스, 배열), 초기값)

const numbers = [1, 2, 3, 4, 5];

// 합계
const sum = numbers.reduce((acc, curr) => acc + curr, 0);
console.log(sum); // 15
// 실행 과정: 0+1=1, 1+2=3, 3+3=6, 6+4=10, 10+5=15

// 최대값 찾기
const max = numbers.reduce((acc, curr) => Math.max(acc, curr), -Infinity);
console.log(max); // 5

// 배열을 객체로 변환 — 매우 유용한 패턴
const fruits = ["사과", "바나나", "사과", "포도", "바나나", "사과"];
const count = fruits.reduce((acc, fruit) => {
  acc[fruit] = (acc[fruit] || 0) + 1;
  return acc;
}, {});
console.log(count); // { 사과: 3, 바나나: 2, 포도: 1 }

// 중첩 배열 평탄화 (flat 대체)
const nested = [[1, 2], [3, 4], [5, 6]];
const flat = nested.reduce((acc, arr) => [...acc, ...arr], []);
console.log(flat); // [1, 2, 3, 4, 5, 6]

// 그룹화
const people = [
  { name: "홍길동", city: "서울" },
  { name: "김철수", city: "부산" },
  { name: "이영희", city: "서울" },
];
const grouped = people.reduce((acc, person) => {
  const key = person.city;
  if (!acc[key]) acc[key] = [];
  acc[key].push(person);
  return acc;
}, {});
console.log(grouped);
// { 서울: [{name:"홍길동",...}, {name:"이영희",...}], 부산: [{name:"김철수",...}] }
```

```kotlin
// Kotlin
val sum = listOf(1, 2, 3, 4, 5).reduce { acc, curr -> acc + curr } // 15
// 또는 fold (초기값 지정 가능 — JavaScript reduce의 초기값과 동일)
val sum2 = listOf(1, 2, 3, 4, 5).fold(0) { acc, curr -> acc + curr }

// 그룹화 — Kotlin은 내장 함수 제공
val grouped = people.groupBy { it.city }
```

### 7-4. find() / findIndex()

```javascript
// JavaScript
const users = [
  { id: 1, name: "홍길동", age: 30 },
  { id: 2, name: "김철수", age: 25 },
  { id: 3, name: "이영희", age: 35 },
];

// find — 조건을 만족하는 첫 번째 요소 반환 (없으면 undefined)
const user = users.find(u => u.id === 2);
console.log(user); // { id: 2, name: "김철수", age: 25 }

const notFound = users.find(u => u.id === 99);
console.log(notFound); // undefined

// findIndex — 조건을 만족하는 첫 번째 요소의 인덱스 반환 (없으면 -1)
const index = users.findIndex(u => u.name === "이영희");
console.log(index); // 2

const notFoundIndex = users.findIndex(u => u.name === "박민수");
console.log(notFoundIndex); // -1
```

```kotlin
// Kotlin
val user = users.find { it.id == 2 }        // 동일
val index = users.indexOfFirst { it.name == "이영희" } // findIndex에 해당
```

### 7-5. some() / every()

```javascript
// JavaScript
const numbers = [2, 4, 6, 8, 10];

// some — 하나라도 조건을 만족하면 true
const hasOdd = numbers.some(n => n % 2 !== 0);
console.log(hasOdd); // false

const hasEven = [1, 3, 4, 7].some(n => n % 2 === 0);
console.log(hasEven); // true

// every — 모든 요소가 조건을 만족하면 true
const allEven = numbers.every(n => n % 2 === 0);
console.log(allEven); // true

const allPositive = [-1, 2, 3].every(n => n > 0);
console.log(allPositive); // false
```

```kotlin
// Kotlin — any()와 all()
val hasOdd = numbers.any { it % 2 != 0 }  // some에 해당
val allEven = numbers.all { it % 2 == 0 } // every에 해당
```

### 7-6. flat() / flatMap()

```javascript
// JavaScript
// flat — 중첩 배열을 평탄화
const nested = [1, [2, 3], [4, [5, 6]]];
console.log(nested.flat());     // [1, 2, 3, 4, [5, 6]] — 1단계만
console.log(nested.flat(2));    // [1, 2, 3, 4, 5, 6]   — 2단계
console.log(nested.flat(Infinity)); // [1, 2, 3, 4, 5, 6] — 완전 평탄화

// flatMap — map + flat(1)
const sentences = ["Hello World", "Good Morning"];
const words = sentences.flatMap(s => s.split(" "));
console.log(words); // ["Hello", "World", "Good", "Morning"]

// 필터 + 변환을 동시에 (null/빈 배열 반환으로 필터링)
const users = [
  { name: "홍길동", hobbies: ["독서", "영화"] },
  { name: "김철수", hobbies: ["게임"] },
];
const allHobbies = users.flatMap(u => u.hobbies);
console.log(allHobbies); // ["독서", "영화", "게임"]
```

```kotlin
// Kotlin
val nested = listOf(listOf(1, 2), listOf(3, 4))
val flat = nested.flatten() // [1, 2, 3, 4]
val allHobbies = users.flatMap { it.hobbies } // 동일
```

### 7-7. 메서드 체이닝 — 여러 메서드를 연결

```javascript
// JavaScript — map, filter, reduce 등을 연결하여 데이터 파이프라인 구성
const orders = [
  { product: "노트북", price: 1500000, quantity: 1 },
  { product: "키보드", price: 80000, quantity: 3 },
  { product: "마우스", price: 50000, quantity: 2 },
  { product: "모니터", price: 500000, quantity: 1 },
];

// 10만원 이상인 주문의 총 금액
const totalExpensive = orders
  .filter(order => order.price >= 100000)          // 비싼 주문만
  .map(order => order.price * order.quantity)       // 총 가격 계산
  .reduce((sum, price) => sum + price, 0);          // 합산

console.log(totalExpensive); // 2000000 (1500000 + 500000)
```

```kotlin
// Kotlin — 동일한 체이닝 가능
val totalExpensive = orders
    .filter { it.price >= 100000 }
    .map { it.price * it.quantity }
    .reduce { acc, price -> acc + price }
```

---

## 8. Object 정적 메서드

### 8-1. Object.keys() / values() / entries()

```javascript
// JavaScript
const user = { name: "홍길동", age: 30, city: "서울" };

// 키 배열
console.log(Object.keys(user));   // ["name", "age", "city"]

// 값 배열
console.log(Object.values(user)); // ["홍길동", 30, "서울"]

// [키, 값] 쌍의 배열
console.log(Object.entries(user));
// [["name", "홍길동"], ["age", 30], ["city", "서울"]]

// Object.entries()로 순회 — 구조 분해와 결합
for (const [key, value] of Object.entries(user)) {
  console.log(`${key}: ${value}`);
}
// "name: 홍길동", "age: 30", "city: 서울"

// entries를 다시 객체로 변환
const entries = [["a", 1], ["b", 2], ["c", 3]];
const obj = Object.fromEntries(entries);
console.log(obj); // { a: 1, b: 2, c: 3 }
```

```kotlin
// Kotlin — Map
val user = mapOf("name" to "홍길동", "age" to 30, "city" to "서울")
println(user.keys)    // [name, age, city]
println(user.values)  // [홍길동, 30, 서울]
println(user.entries) // [name=홍길동, age=30, city=서울]
```

### 8-2. Object.assign() / Object.freeze()

```javascript
// JavaScript
// Object.assign — 객체 병합 (스프레드로 대체 가능)
const target = { a: 1, b: 2 };
const source = { b: 3, c: 4 };
const merged = Object.assign({}, target, source);
console.log(merged); // { a: 1, b: 3, c: 4 }
// 스프레드와 동일: const merged = { ...target, ...source };

// Object.freeze — 객체 동결 (수정, 추가, 삭제 불가)
const config = Object.freeze({
  API_URL: "https://api.example.com",
  TIMEOUT: 5000
});
config.API_URL = "https://other.com"; // 무시됨 (strict mode에서는 TypeError)
console.log(config.API_URL); // "https://api.example.com" — 변경 안 됨

// 주의: freeze는 얕은 동결!
const nested = Object.freeze({
  a: 1,
  b: { c: 2 }
});
nested.b.c = 99; // 중첩 객체는 변경 가능!
console.log(nested.b.c); // 99
```

---

## 9. 클래스 (Class) — 간략 정리

React Native는 주로 **함수형 컴포넌트**를 사용하므로 클래스를 자주 쓰지는 않지만,
라이브러리 코드를 읽을 때 필요합니다.

```javascript
// JavaScript — ES6 Class
class Animal {
  // 생성자
  constructor(name, sound) {
    this.name = name;
    this.sound = sound;
  }

  // 메서드
  speak() {
    return `${this.name}이(가) ${this.sound} 소리를 냅니다.`;
  }

  // 정적 메서드
  static create(name, sound) {
    return new Animal(name, sound);
  }

  // getter
  get info() {
    return `${this.name} (${this.sound})`;
  }

  // setter
  set nickname(value) {
    this._nickname = value;
  }
}

// 상속
class Dog extends Animal {
  constructor(name) {
    super(name, "멍멍"); // 부모 생성자 호출
    this.tricks = [];
  }

  addTrick(trick) {
    this.tricks.push(trick);
  }

  // 메서드 오버라이드
  speak() {
    return `${super.speak()} 꼬리도 흔듭니다!`;
  }
}

const dog = new Dog("바둑이");
console.log(dog.speak()); // "바둑이이(가) 멍멍 소리를 냅니다. 꼬리도 흔듭니다!"
dog.addTrick("앉아");

const cat = Animal.create("나비", "야옹");
console.log(cat.info); // "나비 (야옹)"
```

```kotlin
// Kotlin — 거의 동일한 구조
open class Animal(val name: String, val sound: String) {
    open fun speak(): String = "${name}이(가) $sound 소리를 냅니다."

    companion object {
        fun create(name: String, sound: String) = Animal(name, sound)
    }

    val info: String get() = "$name ($sound)"
}

class Dog(name: String) : Animal(name, "멍멍") {
    val tricks = mutableListOf<String>()

    fun addTrick(trick: String) { tricks.add(trick) }

    override fun speak(): String = "${super.speak()} 꼬리도 흔듭니다!"
}
```

---

## 10. 이터레이터와 제너레이터 (간략)

```javascript
// JavaScript — 제너레이터 (Generator)
// function* 키워드와 yield로 값을 하나씩 생성
function* numberGenerator(start, end) {
  for (let i = start; i <= end; i++) {
    yield i; // 값을 하나 반환하고 일시 정지
  }
}

const gen = numberGenerator(1, 5);
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }

// for-of로 순회 가능
for (const num of numberGenerator(1, 3)) {
  console.log(num); // 1, 2, 3
}

// 무한 시퀀스
function* fibonacci() {
  let a = 0, b = 1;
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

const fib = fibonacci();
console.log(fib.next().value); // 0
console.log(fib.next().value); // 1
console.log(fib.next().value); // 1
console.log(fib.next().value); // 2
console.log(fib.next().value); // 3
```

```kotlin
// Kotlin — sequence로 유사 기능 제공
val fib = sequence {
    var a = 0
    var b = 1
    while (true) {
        yield(a)
        val temp = a
        a = b
        b = temp + b
    }
}
println(fib.take(6).toList()) // [0, 1, 1, 2, 3, 5]
```

---

## 핵심 요약

| ES6+ 기능 | JavaScript 문법 | Kotlin 대응 |
|-----------|-----------------|-------------|
| 객체 구조 분해 | `const { a, b } = obj` | `val (a, b) = dataClass` (순서 기반) |
| 배열 구조 분해 | `const [a, b] = arr` | `val (a, b) = list` |
| 스프레드(배열) | `[...arr1, ...arr2]` | `arr1 + arr2` |
| 스프레드(객체) | `{ ...obj, key: val }` | `obj.copy(key = val)` |
| Optional Chaining | `obj?.prop?.method?.()` | `obj?.prop?.method()` |
| Nullish Coalescing | `value ?? default` | `value ?: default` |
| Named Export | `export { func }` | 패키지 수준 함수 |
| Default Export | `export default func` | — (없음) |
| map/filter/reduce | `arr.map(fn)` | `list.map { }` |
| 클래스 상속 | `class B extends A` | `class B : A()` |
| 제너레이터 | `function* gen() { yield }` | `sequence { yield() }` |

---

> **다음 문서**: [03-async-programming.md](./03-async-programming.md) — 비동기 프로그래밍 완전 정복
