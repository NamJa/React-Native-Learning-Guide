# JavaScript 기초 문법 — Kotlin 개발자를 위한 가이드

> Kotlin Android 개발자가 React Native를 시작하기 전에 반드시 알아야 할 JavaScript 기초 문법을
> Kotlin과 1:1 비교하며 설명합니다. 모든 코드는 직접 실행할 수 있습니다.

---

## 1. 변수 선언: var / let / const vs Kotlin val / var

### 1-1. 기본 개념

JavaScript에는 변수를 선언하는 키워드가 **세 가지**(`var`, `let`, `const`)입니다.
Kotlin에는 **두 가지**(`val`, `var`)만 있으므로 처음에 혼란스러울 수 있습니다.

| JavaScript | Kotlin | 재할당 | 스코프 | 호이스팅 |
|------------|--------|--------|--------|----------|
| `var` | — (사용 자제) | O | 함수 스코프 | O (undefined) |
| `let` | `var` | O | 블록 스코프 | O (TDZ) |
| `const` | `val` | X | 블록 스코프 | O (TDZ) |

- **재할당**: 변수에 새 값을 대입할 수 있는지 여부
- **스코프**: 변수가 유효한 범위
- **호이스팅(Hoisting)**: 선언이 코드 실행 전에 끌어올려지는 JavaScript 고유 동작
- **TDZ(Temporal Dead Zone)**: `let`/`const`가 선언 전에 접근하면 ReferenceError를 발생시키는 구간

### 1-2. let — 블록 스코프 변수 (Kotlin var에 해당)

```javascript
// JavaScript
let name = "홍길동";
name = "김철수"; // OK — 재할당 가능

if (true) {
  let age = 25;
  console.log(age); // 25
}
// console.log(age); // ReferenceError — 블록 밖에서 접근 불가
```

```kotlin
// Kotlin
var name = "홍길동"
name = "김철수" // OK — 재할당 가능

if (true) {
    var age = 25
    println(age) // 25
}
// Kotlin에서는 if 블록 안의 var도 블록 밖에서 접근 불가 — 동일한 블록 스코프
```

### 1-3. const — 재할당 불가 (Kotlin val에 해당)

`const`는 **바인딩(binding)**을 변경할 수 없다는 뜻입니다. 객체의 내부 속성은 변경할 수 있습니다.
Kotlin의 `val`도 동일하게 동작합니다.

```javascript
// JavaScript
const PI = 3.14159;
// PI = 3.14; // TypeError: Assignment to constant variable

const user = { name: "홍길동", age: 30 };
user.age = 31;        // OK — 객체 내부 속성 변경은 허용
// user = {};          // TypeError — 변수 자체를 다른 객체로 바꿀 수 없음

const numbers = [1, 2, 3];
numbers.push(4);      // OK — 배열 내용 변경 가능
// numbers = [5, 6];   // TypeError — 변수 자체 재할당 불가
```

```kotlin
// Kotlin — val도 내부 속성 변경 가능
val user = mutableMapOf("name" to "홍길동", "age" to 30)
user["age"] = 31       // OK

val numbers = mutableListOf(1, 2, 3)
numbers.add(4)         // OK
// numbers = mutableListOf(5, 6) // 컴파일 에러 — val 재할당 불가
```

### 1-4. var — 함수 스코프 (사용 자제)

`var`는 **블록 스코프가 아닌 함수 스코프**를 가집니다. 이 때문에 예상치 못한 버그가 발생하기 쉽습니다.
현대 JavaScript에서는 `var` 대신 `let`/`const`를 사용합니다.

```javascript
// JavaScript — var의 함수 스코프 문제
function example() {
  if (true) {
    var message = "안녕하세요"; // var는 함수 스코프
  }
  console.log(message); // "안녕하세요" — 블록 밖에서도 접근 가능!
}
example();

// let으로 바꾸면?
function example2() {
  if (true) {
    let message = "안녕하세요"; // let은 블록 스코프
  }
  // console.log(message); // ReferenceError
}
```

### 1-5. 호이스팅(Hoisting) — Kotlin에는 없는 개념

JavaScript 엔진은 코드를 실행하기 전에 모든 선언을 해당 스코프의 최상단으로 끌어올립니다.
이것을 **호이스팅**이라고 합니다.

```javascript
// JavaScript — var 호이스팅
console.log(x); // undefined (에러가 아님!)
var x = 10;
console.log(x); // 10

// 위 코드는 엔진 내부에서 다음과 같이 해석됩니다:
// var x;            // 선언만 끌어올려짐
// console.log(x);   // undefined
// x = 10;           // 할당은 원래 위치에서 실행
// console.log(x);   // 10

// let/const 호이스팅 — TDZ(Temporal Dead Zone)
// console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 20;

// 함수 선언문도 호이스팅됩니다
greet(); // "안녕하세요!" — 선언 전에 호출 가능
function greet() {
  console.log("안녕하세요!");
}
```

```kotlin
// Kotlin — 호이스팅 없음. 선언 전 사용 시 컴파일 에러.
// println(x) // 컴파일 에러: Unresolved reference
val x = 10
println(x) // 10
```

---

## 2. 데이터 타입

### 2-1. 원시 타입(Primitive Types) 7가지

JavaScript에는 **7가지 원시 타입**과 **1가지 참조 타입(Object)**이 있습니다.

| JavaScript | Kotlin 대응 | 설명 |
|------------|-------------|------|
| `string` | `String` | 문자열 |
| `number` | `Int`, `Long`, `Float`, `Double` | 정수·소수 구분 없음 (모두 number) |
| `boolean` | `Boolean` | true / false |
| `null` | `null` | 의도적 빈 값 |
| `undefined` | — (없음) | 값이 할당되지 않은 상태 |
| `symbol` | — (없음) | 고유 식별자 |
| `bigint` | `BigInteger` | 큰 정수 |

```javascript
// JavaScript 원시 타입 예시
const str = "Hello";           // string
const num = 42;                // number (정수)
const float = 3.14;            // number (소수) — JavaScript에서는 정수/소수 구분 없음
const bool = true;             // boolean
const nothing = null;          // null — 의도적으로 "값 없음"을 표현
let notAssigned;               // undefined — 값이 할당되지 않음
const sym = Symbol("id");     // symbol — 고유한 식별자
const big = 9007199254740991n; // bigint — 숫자 뒤에 n

// typeof 연산자로 타입 확인
console.log(typeof str);          // "string"
console.log(typeof num);          // "number"
console.log(typeof bool);         // "boolean"
console.log(typeof nothing);      // "object" (!) — JavaScript의 유명한 버그
console.log(typeof notAssigned);  // "undefined"
console.log(typeof sym);          // "symbol"
console.log(typeof big);          // "bigint"
```

```kotlin
// Kotlin 대응 타입
val str: String = "Hello"
val num: Int = 42
val float: Double = 3.14
val bool: Boolean = true
val nothing: String? = null   // Kotlin에서 null은 nullable 타입으로만 가능
// Kotlin에는 undefined 개념이 없음
// Kotlin에는 Symbol 개념이 없음
val big: java.math.BigInteger = java.math.BigInteger("9007199254740991")
```

### 2-2. null vs undefined — Kotlin 개발자가 가장 헷갈리는 부분

Kotlin에는 `null` 하나만 있지만, JavaScript에는 `null`과 `undefined` 두 가지가 있습니다.

```javascript
// null: 개발자가 명시적으로 "값이 없음"을 설정
let user = null; // "아직 사용자가 없다"는 의미

// undefined: JavaScript 엔진이 자동으로 부여하는 "값이 할당되지 않음"
let name;
console.log(name); // undefined — 선언만 하고 값을 넣지 않음

// 객체에 없는 속성에 접근
const obj = { a: 1 };
console.log(obj.b); // undefined — 존재하지 않는 속성

// 함수에서 return 값이 없는 경우
function doNothing() {}
console.log(doNothing()); // undefined

// null과 undefined 비교
console.log(null == undefined);  // true  — 느슨한 비교에서는 같음
console.log(null === undefined); // false — 엄격한 비교에서는 다름
```

### 2-3. number 타입 — 정수/소수 구분 없음

Kotlin에서는 `Int`, `Long`, `Float`, `Double` 등으로 세분화하지만,
JavaScript `number`는 IEEE 754 64비트 부동소수점 하나로 모두 처리합니다.

```javascript
// JavaScript
const intVal = 42;       // 정수처럼 보이지만 실제로는 부동소수점
const floatVal = 3.14;   // 소수점
console.log(typeof intVal);   // "number"
console.log(typeof floatVal); // "number" — 둘 다 같은 타입

// 부동소수점 오차 주의!
console.log(0.1 + 0.2);         // 0.30000000000000004 (!)
console.log(0.1 + 0.2 === 0.3); // false (!)

// 안전한 정수 범위
console.log(Number.MAX_SAFE_INTEGER); // 9007199254740991 (2^53 - 1)
console.log(Number.MIN_SAFE_INTEGER); // -9007199254740991

// 특수 값
console.log(1 / 0);          // Infinity
console.log(-1 / 0);         // -Infinity
console.log("abc" * 2);      // NaN (Not a Number)
console.log(NaN === NaN);    // false (!) — NaN은 자기 자신과도 같지 않음
console.log(Number.isNaN(NaN)); // true — NaN 확인은 이 메서드 사용
```

```kotlin
// Kotlin — 타입이 세분화되어 있음
val intVal: Int = 42
val longVal: Long = 42L
val floatVal: Float = 3.14f
val doubleVal: Double = 3.14
// Kotlin에서도 0.1 + 0.2 == 0.3은 false (부동소수점 오차는 동일)
```

---

## 3. 연산자

### 3-1. 동등 비교: == vs === (Kotlin에는 ===가 참조 비교)

JavaScript에서 가장 주의해야 할 연산자입니다.

- `==` (느슨한 비교): **타입 변환** 후 비교
- `===` (엄격한 비교): 타입 변환 없이 **타입과 값 모두** 비교

```javascript
// JavaScript
// == (느슨한 비교) — 타입이 다르면 자동 변환 후 비교
console.log(1 == "1");       // true  — 문자열 "1"이 숫자 1로 변환
console.log(0 == false);     // true  — false가 0으로 변환
console.log("" == false);    // true  — ""가 0으로, false도 0으로 변환
console.log(null == undefined); // true — 특수 규칙

// === (엄격한 비교) — 타입 변환 없이 비교
console.log(1 === "1");      // false — 타입이 다름 (number vs string)
console.log(0 === false);    // false — 타입이 다름 (number vs boolean)
console.log(null === undefined); // false

// 실무에서는 항상 === 사용을 권장합니다!
```

```kotlin
// Kotlin
// == 는 구조적 동등성 (equals) 비교
// === 는 참조적 동등성 (같은 객체인지) 비교
println(1 == 1)       // true — 값 비교
// println(1 == "1")  // 컴파일 에러 — Kotlin은 타입이 다르면 비교 자체가 안 됨

val a = "hello"
val b = "hello"
println(a == b)   // true  — 값이 같음
println(a === b)  // true  — String pool에 의해 같은 참조 (JVM 최적화)
```

### 3-2. Nullish Coalescing (??) vs Kotlin Elvis 연산자 (?:)

```javascript
// JavaScript — ?? (Nullish Coalescing)
// null 또는 undefined일 때만 오른쪽 값을 사용
const value1 = null ?? "기본값";
console.log(value1); // "기본값"

const value2 = undefined ?? "기본값";
console.log(value2); // "기본값"

const value3 = 0 ?? "기본값";
console.log(value3); // 0 — 0은 null/undefined가 아니므로 그대로 사용

const value4 = "" ?? "기본값";
console.log(value4); // "" — 빈 문자열도 null/undefined가 아님

const value5 = false ?? "기본값";
console.log(value5); // false — false도 null/undefined가 아님

// || (OR 연산자)와의 차이 — || 는 falsy 값 전체를 대체
const value6 = 0 || "기본값";
console.log(value6); // "기본값" — 0은 falsy이므로 대체됨!

const value7 = "" || "기본값";
console.log(value7); // "기본값" — ""은 falsy이므로 대체됨!
```

```kotlin
// Kotlin — ?: (Elvis 연산자)
// null일 때만 오른쪽 값 사용 (Kotlin에는 undefined 개념 없음)
val value1 = null ?: "기본값"
println(value1) // "기본값"

val value2: Int? = 0
val result = value2 ?: -1
println(result) // 0 — null이 아니므로 그대로 사용
// JavaScript의 ??와 동일한 동작!
```

### 3-3. Optional Chaining (?.) — Kotlin과 동일한 문법!

```javascript
// JavaScript — ?. (Optional Chaining)
const user = {
  name: "홍길동",
  address: {
    city: "서울"
  }
};

console.log(user.address?.city);    // "서울"
console.log(user.phone?.number);    // undefined — phone이 없으므로

// 중첩 사용
const company = null;
console.log(company?.departments?.engineering?.lead); // undefined

// 메서드 호출에도 사용 가능
const arr = [1, 2, 3];
console.log(arr.find?.(x => x > 2)); // 3

// 없는 메서드
const obj = {};
console.log(obj.someMethod?.());     // undefined
```

```kotlin
// Kotlin — ?. (Safe Call) — 문법이 동일!
data class Address(val city: String)
data class User(val name: String, val address: Address?)

val user = User("홍길동", Address("서울"))
println(user.address?.city) // "서울"

val user2 = User("홍길동", null)
println(user2.address?.city) // null (JavaScript의 undefined에 해당)
```

---

## 4. 문자열 템플릿

### 4-1. 백틱 템플릿 리터럴 vs Kotlin 문자열 템플릿

```javascript
// JavaScript — 백틱(`) 사용, ${} 안에 표현식
const name = "홍길동";
const age = 30;

// 기본 사용
const greeting = `안녕하세요, ${name}님! 나이는 ${age}세입니다.`;
console.log(greeting); // "안녕하세요, 홍길동님! 나이는 30세입니다."

// 표현식 사용
console.log(`내년 나이: ${age + 1}`);  // "내년 나이: 31"
console.log(`성인?: ${age >= 18 ? "예" : "아니오"}`); // "성인?: 예"

// 여러 줄 문자열 — 백틱은 줄바꿈을 그대로 유지
const multiLine = `
  첫 번째 줄
  두 번째 줄
  세 번째 줄
`;
console.log(multiLine);

// 일반 따옴표로는 줄바꿈 불가
// const fail = "첫 번째 줄
//   두 번째 줄"; // SyntaxError
```

```kotlin
// Kotlin — $ 또는 ${} 사용, 큰따옴표 또는 삼중따옴표
val name = "홍길동"
val age = 30

// 기본 사용 — 단순 변수는 $ 만으로 가능
val greeting = "안녕하세요, ${name}님! 나이는 ${age}세입니다."
// 또는: "안녕하세요, $name 님! 나이는 $age 세입니다."

// 표현식은 ${} 필수
println("내년 나이: ${age + 1}")
println("성인?: ${if (age >= 18) "예" else "아니오"}")

// 여러 줄 문자열 — 삼중따옴표 사용
val multiLine = """
    첫 번째 줄
    두 번째 줄
    세 번째 줄
""".trimIndent()
```

**핵심 차이점**:
- JavaScript: 백틱(`` ` ``) + `${}`
- Kotlin: 큰따옴표(`"`) + `$` 또는 `${}`
- JavaScript의 일반 따옴표(`'` 또는 `"`)는 템플릿을 지원하지 않음

---

## 5. 제어 흐름 (Control Flow)

### 5-1. if / else — 거의 동일하지만 결정적 차이

```javascript
// JavaScript — if/else는 **문(statement)**
const score = 85;

if (score >= 90) {
  console.log("A");
} else if (score >= 80) {
  console.log("B");
} else {
  console.log("C");
}

// JavaScript에서는 if가 값을 반환하지 않으므로 삼항 연산자 사용
const grade = score >= 90 ? "A" : score >= 80 ? "B" : "C";
console.log(grade); // "B"
```

```kotlin
// Kotlin — if/else는 **식(expression)** — 값을 반환할 수 있음
val score = 85

val grade = if (score >= 90) "A"
            else if (score >= 80) "B"
            else "C"
println(grade) // "B"
// Kotlin에도 삼항 연산자가 없음 — if/else 자체가 식이므로 불필요
```

### 5-2. switch vs when

```javascript
// JavaScript — switch/case
const fruit = "사과";

switch (fruit) {
  case "사과":
    console.log("빨간색");
    break; // break를 빠뜨리면 다음 case로 떨어짐(fall-through)!
  case "바나나":
    console.log("노란색");
    break;
  case "포도":
  case "블루베리": // 여러 값을 묶을 때
    console.log("보라색/파란색");
    break;
  default:
    console.log("알 수 없음");
}

// switch는 값을 반환하지 않음 — 변수에 할당하려면 별도 처리 필요
let color;
switch (fruit) {
  case "사과": color = "빨간색"; break;
  case "바나나": color = "노란색"; break;
  default: color = "알 수 없음";
}
```

```kotlin
// Kotlin — when (switch보다 훨씬 강력)
val fruit = "사과"

// when은 식(expression)이므로 값을 반환
val color = when (fruit) {
    "사과" -> "빨간색"
    "바나나" -> "노란색"
    "포도", "블루베리" -> "보라색/파란색" // 여러 값
    else -> "알 수 없음"
}
println(color) // "빨간색"
// break 불필요, fall-through 없음
```

### 5-3. 반복문 비교

JavaScript에는 여러 종류의 반복문이 있습니다.

```javascript
// === for (전통적 for) ===
for (let i = 0; i < 5; i++) {
  console.log(i); // 0, 1, 2, 3, 4
}

// === for...of (Kotlin의 for-in에 해당) — 값을 순회 ===
const fruits = ["사과", "바나나", "포도"];
for (const fruit of fruits) {
  console.log(fruit); // "사과", "바나나", "포도"
}

// === for...in — 객체의 키(속성 이름)를 순회 ===
const person = { name: "홍길동", age: 30, city: "서울" };
for (const key in person) {
  console.log(`${key}: ${person[key]}`);
  // "name: 홍길동", "age: 30", "city: 서울"
}
// 주의: for-in은 배열에 사용하지 마세요! (인덱스가 문자열로 나옴)

// === forEach — 배열 메서드 ===
fruits.forEach((fruit, index) => {
  console.log(`${index}: ${fruit}`);
});
// "0: 사과", "1: 바나나", "2: 포도"

// === while / do-while — Kotlin과 동일 ===
let count = 0;
while (count < 3) {
  console.log(count); // 0, 1, 2
  count++;
}
```

```kotlin
// Kotlin 반복문 비교
// 전통적 for — Kotlin에는 C 스타일 for 없음, 대신 범위 사용
for (i in 0 until 5) {
    println(i) // 0, 1, 2, 3, 4
}

// for-in — JavaScript의 for-of에 해당
val fruits = listOf("사과", "바나나", "포도")
for (fruit in fruits) {
    println(fruit)
}

// forEachIndexed — JavaScript의 forEach(item, index)에 해당
fruits.forEachIndexed { index, fruit ->
    println("$index: $fruit")
}

// 맵 순회 — JavaScript의 for-in에 해당
val person = mapOf("name" to "홍길동", "age" to 30)
for ((key, value) in person) {
    println("$key: $value")
}
```

### 5-4. Falsy와 Truthy — Kotlin에는 없는 개념

JavaScript에서는 boolean이 아닌 값도 조건문에서 `true` 또는 `false`로 평가됩니다.

```javascript
// JavaScript — Falsy 값 (false로 평가되는 값들)
// false, 0, -0, 0n, "", null, undefined, NaN — 이 8개가 전부

if (0) console.log("실행 안 됨");
if ("") console.log("실행 안 됨");
if (null) console.log("실행 안 됨");
if (undefined) console.log("실행 안 됨");
if (NaN) console.log("실행 안 됨");

// Truthy 값 — 위 8개를 제외한 모든 값
if (1) console.log("실행됨");
if ("hello") console.log("실행됨");
if ([]) console.log("실행됨");    // 빈 배열도 truthy!
if ({}) console.log("실행됨");    // 빈 객체도 truthy!

// 실무에서 자주 쓰는 패턴
const name = "";
if (name) {
  console.log("이름이 있습니다");
} else {
  console.log("이름이 비어있습니다"); // 이것이 출력됨
}

// Boolean()으로 명시적 변환
console.log(Boolean(0));       // false
console.log(Boolean("hello")); // true
console.log(Boolean([]));      // true — 주의!
```

```kotlin
// Kotlin — if 조건에는 반드시 Boolean만 가능
// if (0) println("...") // 컴파일 에러!
// if ("") println("...") // 컴파일 에러!
val name = ""
if (name.isNotEmpty()) {
    println("이름이 있습니다")
}
```

---

## 6. 함수

### 6-1. 함수 선언(Function Declaration)

```javascript
// JavaScript — 함수 선언문 (호이스팅됨)
function add(a, b) {
  return a + b;
}
console.log(add(1, 2)); // 3

// 호이스팅 덕분에 선언 전에 호출 가능
console.log(multiply(3, 4)); // 12
function multiply(a, b) {
  return a * b;
}
```

```kotlin
// Kotlin — fun 키워드
fun add(a: Int, b: Int): Int {
    return a + b
}
println(add(1, 2)) // 3
// Kotlin에서는 선언 전에 호출 가능 (클래스 내 메서드)
```

### 6-2. 함수 표현식(Function Expression)

```javascript
// JavaScript — 변수에 함수를 할당
const subtract = function(a, b) {
  return a - b;
};
console.log(subtract(5, 3)); // 2

// 함수 표현식은 호이스팅되지 않음
// console.log(divide(10, 2)); // TypeError: divide is not a function
const divide = function(a, b) {
  return a / b;
};
```

```kotlin
// Kotlin — 람다를 변수에 할당
val subtract: (Int, Int) -> Int = { a, b -> a - b }
println(subtract(5, 3)) // 2
```

### 6-3. 화살표 함수(Arrow Function) — React Native에서 가장 많이 사용

```javascript
// JavaScript — 화살표 함수 (Arrow Function)
// 기본 형태
const add = (a, b) => {
  return a + b;
};

// 본문이 한 줄이면 중괄호와 return 생략 가능 (암묵적 반환)
const addShort = (a, b) => a + b;

// 매개변수가 하나면 괄호도 생략 가능
const double = x => x * 2;

// 매개변수가 없으면 빈 괄호 필수
const greet = () => "안녕하세요!";

// 객체를 반환할 때는 소괄호로 감싸야 함
const makeUser = (name, age) => ({ name, age });
// ({ name, age })가 아니라 { name, age }로 쓰면 코드 블록으로 해석됨!

console.log(add(1, 2));       // 3
console.log(double(5));       // 10
console.log(greet());         // "안녕하세요!"
console.log(makeUser("홍길동", 30)); // { name: "홍길동", age: 30 }
```

```kotlin
// Kotlin — 람다 표현식
val add: (Int, Int) -> Int = { a, b -> a + b }
val double: (Int) -> Int = { x -> x * 2 }
// 매개변수가 하나면 it 사용 가능
val doubleShorter: (Int) -> Int = { it * 2 }
val greet: () -> String = { "안녕하세요!" }
```

**화살표 함수 vs 일반 함수의 핵심 차이: `this` 바인딩**

```javascript
// 화살표 함수는 자신만의 this를 가지지 않고, 외부의 this를 그대로 사용합니다.
// 이것이 React Native에서 화살표 함수를 선호하는 이유입니다.

const obj = {
  name: "홍길동",
  // 일반 함수: this는 obj를 가리킴
  greetRegular: function() {
    console.log(`안녕, ${this.name}`); // "안녕, 홍길동"
  },
  // 화살표 함수: this는 외부 스코프를 가리킴
  greetArrow: () => {
    console.log(`안녕, ${this.name}`); // "안녕, undefined" (this가 obj가 아님!)
  },
  // 일반 함수 내부의 화살표 함수: 유용한 패턴
  delayedGreet: function() {
    setTimeout(() => {
      console.log(`안녕, ${this.name}`); // "안녕, 홍길동" — 화살표 함수가 외부 this(obj) 사용
    }, 1000);
  }
};
```

### 6-4. 기본 매개변수(Default Parameters)

```javascript
// JavaScript
function greet(name = "손님", greeting = "안녕하세요") {
  return `${greeting}, ${name}님!`;
}
console.log(greet());                 // "안녕하세요, 손님님!"
console.log(greet("홍길동"));          // "안녕하세요, 홍길동님!"
console.log(greet("홍길동", "반갑습니다")); // "반갑습니다, 홍길동님!"

// 중간 매개변수를 건너뛰려면 undefined를 전달
console.log(greet(undefined, "하이"));  // "하이, 손님님!"
```

```kotlin
// Kotlin — 거의 동일한 문법
fun greet(name: String = "손님", greeting: String = "안녕하세요"): String {
    return "$greeting, ${name}님!"
}
println(greet())                          // "안녕하세요, 손님님!"
println(greet("홍길동"))                   // "안녕하세요, 홍길동님!"
// Kotlin에서는 이름 있는 인자(named argument)로 중간 매개변수 건너뛰기 가능
println(greet(greeting = "하이"))          // "하이, 손님님!"
```

### 6-5. 나머지 매개변수(Rest Parameters)

```javascript
// JavaScript — ... (나머지 매개변수)
function sum(...numbers) {
  return numbers.reduce((total, n) => total + n, 0);
}
console.log(sum(1, 2, 3));     // 6
console.log(sum(1, 2, 3, 4, 5)); // 15

// 첫 번째 인자는 따로, 나머지는 배열로
function log(level, ...messages) {
  console.log(`[${level}]`, ...messages);
}
log("INFO", "서버 시작", "포트: 3000");
// [INFO] 서버 시작 포트: 3000
```

```kotlin
// Kotlin — vararg
fun sum(vararg numbers: Int): Int {
    return numbers.sum()
}
println(sum(1, 2, 3))     // 6
println(sum(1, 2, 3, 4, 5)) // 15
```

### 6-6. 콜백 함수 — React Native에서 매우 중요

```javascript
// JavaScript — 함수를 다른 함수의 인자로 전달
function processData(data, callback) {
  const result = data.toUpperCase();
  callback(result);
}

processData("hello", (result) => {
  console.log(result); // "HELLO"
});

// 배열 메서드에서의 콜백 — 가장 흔한 사용 패턴
const numbers = [1, 2, 3, 4, 5];

const doubled = numbers.map(n => n * 2);
console.log(doubled); // [2, 4, 6, 8, 10]

const evens = numbers.filter(n => n % 2 === 0);
console.log(evens); // [2, 4]

const sum = numbers.reduce((acc, n) => acc + n, 0);
console.log(sum); // 15
```

```kotlin
// Kotlin — 고차 함수와 람다
fun processData(data: String, callback: (String) -> Unit) {
    val result = data.uppercase()
    callback(result)
}
processData("hello") { result ->
    println(result) // "HELLO"
}

val numbers = listOf(1, 2, 3, 4, 5)
val doubled = numbers.map { it * 2 }       // [2, 4, 6, 8, 10]
val evens = numbers.filter { it % 2 == 0 } // [2, 4]
val sum = numbers.reduce { acc, n -> acc + n } // 15
```

---

## 7. 객체(Object) — Kotlin의 Map/Data Class에 해당

### 7-1. 객체 리터럴

```javascript
// JavaScript — 가장 기본적인 데이터 구조
const user = {
  name: "홍길동",
  age: 30,
  isActive: true,
  address: {
    city: "서울",
    zip: "06000"
  },
  // 메서드 (축약 문법)
  greet() {
    return `안녕하세요, ${this.name}입니다.`;
  }
};

// 접근 방법 1: 점 표기법
console.log(user.name);         // "홍길동"
console.log(user.address.city); // "서울"

// 접근 방법 2: 대괄호 표기법 — 동적 키에 유용
const key = "name";
console.log(user[key]);         // "홍길동"

// 속성 추가/수정/삭제 — 매우 동적!
user.email = "hong@example.com"; // 추가
user.age = 31;                   // 수정
delete user.isActive;            // 삭제
```

```kotlin
// Kotlin — data class 사용 (정적 타입)
data class Address(val city: String, val zip: String)
data class User(
    val name: String,
    var age: Int,
    val isActive: Boolean,
    val address: Address
) {
    fun greet(): String = "안녕하세요, ${name}입니다."
}

val user = User("홍길동", 30, true, Address("서울", "06000"))
println(user.name)           // "홍길동"
println(user.address.city)   // "서울"

// Kotlin은 런타임에 속성을 동적으로 추가/삭제할 수 없음
// user.email = "hong@example.com" // 컴파일 에러
```

### 7-2. 배열(Array) — Kotlin의 List에 해당

```javascript
// JavaScript — 배열은 동적이고 어떤 타입이든 섞을 수 있음
const mixed = [1, "two", true, null, { name: "test" }]; // 다양한 타입 혼합 가능
const numbers = [1, 2, 3, 4, 5];

// 인덱스 접근
console.log(numbers[0]);     // 1
console.log(numbers.length); // 5

// 추가/제거
numbers.push(6);       // 끝에 추가 → [1, 2, 3, 4, 5, 6]
numbers.pop();         // 끝에서 제거 → [1, 2, 3, 4, 5]
numbers.unshift(0);    // 앞에 추가 → [0, 1, 2, 3, 4, 5]
numbers.shift();       // 앞에서 제거 → [1, 2, 3, 4, 5]

// 배열 확인
console.log(Array.isArray(numbers)); // true
console.log(typeof numbers);        // "object" — typeof로는 배열 판별 불가!

// 유용한 메서드들
console.log(numbers.includes(3));    // true — 포함 여부
console.log(numbers.indexOf(3));     // 2 — 인덱스 찾기
console.log(numbers.join(", "));     // "1, 2, 3, 4, 5" — 문자열로 합치기
console.log(numbers.slice(1, 3));    // [2, 3] — 잘라내기 (원본 유지)
numbers.splice(1, 2);               // [2, 3] 제거 → numbers는 [1, 4, 5] (원본 변경!)
```

```kotlin
// Kotlin
val numbers = mutableListOf(1, 2, 3, 4, 5)
println(numbers[0])      // 1
println(numbers.size)    // 5 (JavaScript의 length에 해당)

numbers.add(6)           // push에 해당
numbers.removeAt(numbers.size - 1) // pop에 해당
numbers.contains(3)      // includes에 해당
numbers.indexOf(3)       // indexOf에 해당
numbers.joinToString(", ") // join에 해당
numbers.subList(1, 3)    // slice에 해당
```

---

## 8. 타입 변환과 체크

```javascript
// JavaScript — 타입 변환 (자주 필요!)
// 문자열 → 숫자
const num1 = Number("42");       // 42
const num2 = parseInt("42.5");   // 42 (정수로)
const num3 = parseFloat("42.5"); // 42.5
const num4 = +"42";              // 42 (단항 + 연산자)

// 숫자 → 문자열
const str1 = String(42);     // "42"
const str2 = (42).toString(); // "42"
const str3 = `${42}`;        // "42" (템플릿 리터럴)

// 변환 실패 시
console.log(Number("abc"));  // NaN
console.log(parseInt("abc")); // NaN

// 타입 체크
console.log(typeof "hello");   // "string"
console.log(typeof 42);       // "number"
console.log(typeof true);     // "boolean"
console.log(typeof undefined); // "undefined"
console.log(typeof null);     // "object" (!)
console.log(typeof []);       // "object" (!)
console.log(typeof {});       // "object"
console.log(typeof function(){}); // "function"

// 배열인지 확인: Array.isArray()
console.log(Array.isArray([]));  // true
console.log(Array.isArray({}));  // false
```

```kotlin
// Kotlin — 타입 변환
val num = "42".toInt()
val float = "42.5".toDouble()
val str = 42.toString()

// 안전한 변환
val safeNum = "abc".toIntOrNull() // null

// 타입 체크 — is 연산자
println("hello" is String) // true
println(42 is Int)         // true
```

---

## 핵심 정리: JavaScript vs Kotlin 빠른 비교표

| 개념 | JavaScript | Kotlin |
|------|-----------|--------|
| 불변 변수 | `const` | `val` |
| 가변 변수 | `let` | `var` |
| 문자열 템플릿 | `` `${expr}` `` | `"${expr}"` 또는 `"$var"` |
| null 안전 접근 | `?.` | `?.` |
| null 대체 값 | `??` | `?:` |
| 엄격한 비교 | `===` | `==` |
| 타입 체크 | `typeof x` | `x is Type` |
| 배열 | `[]` (Array) | `listOf()` |
| 객체/맵 | `{}` (Object) | `mapOf()` / data class |
| 람다/화살표 함수 | `(x) => x * 2` | `{ x -> x * 2 }` |
| 반복(배열) | `for (const x of arr)` | `for (x in list)` |
| 반복(객체) | `for (const k in obj)` | `for ((k, v) in map)` |
| falsy 체크 | `if (value)` | `if (value != null)` |
| 없는 값 | `null` / `undefined` | `null` |

---

> **다음 문서**: [02-modern-javascript-es6+.md](./02-modern-javascript-es6+.md) — 모던 JavaScript (ES6+) 핵심 문법 완전 정리
