# TypeScript 핵심 — Kotlin 개발자가 빠르게 익히는 타입 시스템

> TypeScript는 JavaScript에 **정적 타입**을 추가한 언어입니다.
> Kotlin 개발자라면 타입 시스템에 익숙하므로 빠르게 배울 수 있습니다.
> React Native 0.84는 TypeScript를 기본으로 지원하며, 거의 모든 프로젝트가 TypeScript를 사용합니다.

---

## 1. 기본 타입 (Basic Types)

### 1-1. 원시 타입

```typescript
// TypeScript — 기본 타입 선언
// 변수명: 타입 = 값
const name: string = "홍길동";
const age: number = 30;            // 정수, 소수 구분 없음
const isActive: boolean = true;
const nothing: null = null;
const notDefined: undefined = undefined;
const bigNumber: bigint = 9007199254740991n;
const id: symbol = Symbol("id");

// 배열
const numbers: number[] = [1, 2, 3, 4, 5];
const names: Array<string> = ["홍길동", "김철수"]; // 제네릭 문법도 가능
const mixed: (string | number)[] = [1, "two", 3]; // 유니온 타입 배열

// 튜플 (Tuple) — 고정된 길이와 타입의 배열
const pair: [string, number] = ["홍길동", 30];
const rgb: [number, number, number] = [255, 128, 0];
// pair[0]은 string, pair[1]은 number로 타입이 확정됨

// console.log(pair[2]); // 에러: Tuple type '[string, number]' has no element at index '2'
```

```kotlin
// Kotlin — 대응하는 타입
val name: String = "홍길동"
val age: Int = 30             // Kotlin은 정수/소수 구분 (Int, Double)
val isActive: Boolean = true
val nothing: String? = null   // nullable 타입

val numbers: List<Int> = listOf(1, 2, 3, 4, 5)
val pair: Pair<String, Int> = Pair("홍길동", 30) // 튜플 대신 Pair/Triple
```

### 1-2. 특수 타입: any, unknown, void, never

```typescript
// TypeScript — 특수 타입

// any — 모든 타입 허용 (타입 체크를 비활성화)
// ⚠️ 가능하면 사용하지 마세요! TypeScript를 쓰는 의미가 없어집니다.
let anything: any = "hello";
anything = 42;           // OK
anything = true;         // OK
anything.doSomething();  // OK — 타입 체크 안 됨 (런타임에 에러 발생 가능)

// unknown — any보다 안전한 "모든 타입"
// 값을 할당할 수 있지만, 사용하려면 타입을 확인해야 함
let unknownValue: unknown = "hello";
unknownValue = 42;         // OK — 할당은 자유
// unknownValue.toFixed();  // 에러! — unknown 타입을 직접 사용 불가

// 타입 가드(type guard)로 확인 후 사용
if (typeof unknownValue === "number") {
  console.log(unknownValue.toFixed(2)); // OK — number임이 확인됨
}

// void — 반환값이 없는 함수
function logMessage(msg: string): void {
  console.log(msg);
  // return 없음 (또는 return undefined)
}

// never — 절대 발생하지 않는 타입
// 항상 예외를 throw하거나 무한 루프인 함수에 사용
function throwError(message: string): never {
  throw new Error(message);
  // 이 코드에 도달할 수 없으므로 반환 타입이 never
}

function infiniteLoop(): never {
  while (true) {
    // 영원히 반복
  }
}

// never의 실용적 사용 — 도달 불가능한 코드 체크 (exhaustive check)
type Shape = "circle" | "square" | "triangle";
function getArea(shape: Shape): number {
  switch (shape) {
    case "circle": return Math.PI * 10 * 10;
    case "square": return 10 * 10;
    case "triangle": return (10 * 10) / 2;
    default:
      // shape이 never 타입이 됨 — 모든 case를 처리했는지 컴파일러가 확인
      const exhaustiveCheck: never = shape;
      throw new Error(`처리되지 않은 도형: ${exhaustiveCheck}`);
  }
}
```

```kotlin
// Kotlin — 대응
val anything: Any = "hello" // Any는 TypeScript의 unknown에 더 가까움
// Any에서 메서드를 호출하려면 캐스트 필요

fun logMessage(msg: String): Unit { // Unit = void
    println(msg)
}

fun throwError(message: String): Nothing { // Nothing = never
    throw Error(message)
}
```

### 1-3. enum 타입

```typescript
// TypeScript — 숫자 enum
enum Direction {
  Up,      // 0
  Down,    // 1
  Left,    // 2
  Right    // 3
}
const dir: Direction = Direction.Up;
console.log(dir);             // 0
console.log(Direction[0]);    // "Up" — 역방향 매핑

// 문자열 enum — React Native에서 더 실용적
enum Status {
  Loading = "LOADING",
  Success = "SUCCESS",
  Error = "ERROR",
  Idle = "IDLE"
}
const currentStatus: Status = Status.Loading;
console.log(currentStatus); // "LOADING"

// const enum — 컴파일 시 인라인됨 (성능 최적화)
const enum Color {
  Red = "RED",
  Blue = "BLUE",
  Green = "GREEN"
}
// 컴파일 후: const color = "RED" (enum 객체가 생성되지 않음)
const color: Color = Color.Red;
```

```kotlin
// Kotlin — enum class
enum class Direction {
    Up, Down, Left, Right
}
val dir: Direction = Direction.Up

enum class Status(val value: String) {
    Loading("LOADING"),
    Success("SUCCESS"),
    Error("ERROR"),
    Idle("IDLE")
}
```

---

## 2. 타입 추론 (Type Inference)

TypeScript와 Kotlin 모두 강력한 타입 추론을 지원합니다.
명시적 타입 선언 없이도 컴파일러가 타입을 자동으로 파악합니다.

```typescript
// TypeScript — 타입 추론
// 대부분의 경우 타입을 명시하지 않아도 됩니다!

const name = "홍길동";        // string으로 추론
const age = 30;               // number로 추론
const isActive = true;        // boolean으로 추론
const numbers = [1, 2, 3];    // number[]로 추론

// 함수 반환 타입 추론
function add(a: number, b: number) {
  return a + b; // 반환 타입 number로 자동 추론
}

// 콜백의 매개변수 타입도 추론
const doubled = numbers.map(n => n * 2); // n은 number로 추론

// 객체 리터럴의 타입 추론
const user = {
  name: "홍길동",
  age: 30
};
// user의 타입: { name: string; age: number; }

// ❗ const와 let의 추론 차이
const x = "hello"; // 타입: "hello" (리터럴 타입)
let y = "hello";   // 타입: string (더 넓은 타입)

// ❗ 타입을 명시해야 하는 경우
// 1) 함수 매개변수 — 추론 불가
function greet(name: string): string {
  return `안녕, ${name}!`;
}

// 2) 빈 배열 — 타입을 알 수 없음
const items: string[] = []; // 명시하지 않으면 never[]로 추론
items.push("항목 1");
```

```kotlin
// Kotlin — 거의 동일한 타입 추론
val name = "홍길동"        // String으로 추론
val age = 30               // Int로 추론
val numbers = listOf(1, 2, 3) // List<Int>로 추론

// 함수 반환 타입 추론
fun add(a: Int, b: Int) = a + b // Int 반환으로 추론

// 빈 리스트 — 타입 명시 필요 (TypeScript와 동일)
val items: MutableList<String> = mutableListOf()
```

**규칙**: 타입 추론이 가능하면 생략하고, 모호한 경우에만 명시합니다.
함수의 매개변수와 공개 API의 반환 타입은 명시하는 것이 좋습니다.

---

## 3. 인터페이스 (Interface) vs 타입 별칭 (Type Alias)

### 3-1. Interface

```typescript
// TypeScript — Interface
interface User {
  id: number;
  name: string;
  email: string;
  age?: number;           // ? = 선택적 속성 (있어도 되고 없어도 됨)
  readonly createdAt: Date; // readonly = 읽기 전용 (수정 불가)
}

const user: User = {
  id: 1,
  name: "홍길동",
  email: "hong@example.com",
  createdAt: new Date()
  // age는 선택적이므로 생략 가능
};

// user.createdAt = new Date(); // 에러! readonly

// 인터페이스 확장 (상속)
interface AdminUser extends User {
  role: "admin" | "superadmin";
  permissions: string[];
}

const admin: AdminUser = {
  id: 2,
  name: "관리자",
  email: "admin@example.com",
  createdAt: new Date(),
  role: "admin",
  permissions: ["read", "write", "delete"]
};

// 인터페이스에 메서드 정의
interface Printable {
  print(): void;
  format(type: string): string;
}

// 여러 인터페이스를 동시에 확장
interface SuperUser extends User, Printable {
  superPower: string;
}

// 인터페이스 병합 (Declaration Merging) — interface만 가능!
interface Config {
  apiUrl: string;
}
interface Config {
  timeout: number;
}
// 자동으로 병합됨:
// interface Config { apiUrl: string; timeout: number; }
const config: Config = { apiUrl: "https://api.example.com", timeout: 5000 };
```

```kotlin
// Kotlin — interface 비교
interface User {
    val id: Int
    val name: String
    val email: String
    val age: Int?     // nullable = TypeScript의 선택적 속성과 유사
    val createdAt: Date
}

// data class로 구현
data class UserImpl(
    override val id: Int,
    override val name: String,
    override val email: String,
    override val age: Int? = null,
    override val createdAt: Date
) : User
```

### 3-2. Type Alias

```typescript
// TypeScript — Type Alias (타입 별칭)
type UserID = number;
type UserName = string;

// 객체 타입
type User = {
  id: UserID;
  name: UserName;
  email: string;
  age?: number;
};

// 유니온 타입 — type으로만 가능!
type Status = "loading" | "success" | "error" | "idle";
type StringOrNumber = string | number;

// 인터섹션 타입 — type으로만 가능!
type Admin = User & {
  role: string;
  permissions: string[];
};

// 함수 타입
type Callback = (data: string) => void;
type AsyncCallback = (data: string) => Promise<void>;
type MathOperation = (a: number, b: number) => number;

const add: MathOperation = (a, b) => a + b;
const subtract: MathOperation = (a, b) => a - b;

// 조건부 타입 — 고급 기능
type IsString<T> = T extends string ? "yes" : "no";
type Result1 = IsString<string>;  // "yes"
type Result2 = IsString<number>;  // "no"

// 매핑된 타입
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};
```

### 3-3. Interface vs Type — 언제 무엇을 사용?

```typescript
// 규칙 1: 객체의 형태(shape)를 정의할 때 → interface 선호
interface User {
  name: string;
  age: number;
}

// 규칙 2: 유니온, 인터섹션, 기본 타입 별칭 → type 필수
type Status = "loading" | "success" | "error";
type ID = string | number;
type Callback = () => void;

// 규칙 3: 확장이 필요한 경우 → interface (선언 병합 가능)
// 규칙 4: React Native 컴포넌트의 Props → 둘 다 가능, 팀 규칙에 따름

// 실무에서 가장 흔한 패턴 (React Native):
interface ButtonProps {
  title: string;
  onPress: () => void;
  disabled?: boolean;
  color?: string;
}
// 또는
type ButtonProps = {
  title: string;
  onPress: () => void;
  disabled?: boolean;
  color?: string;
};
```

```exercise
type: word-bank
question: "TypeScript 타입 정의의 빈칸을 채우세요"
code: |
  type User = {
    id: ___;
    name: ___;
    email?: ___;
  };
blanks: ["number", "string", "string"]
distractors: ["boolean", "any", "void", "undefined"]
hint: "id는 숫자, name과 email은 문자열 타입입니다. email은 선택적(optional) 속성입니다."
xp: 5
```

```exercise
type: categorize
question: "다음 키워드를 Kotlin과 TypeScript로 분류하세요"
categories: ["Kotlin", "TypeScript"]
items:
  - text: "data class"
    category: "Kotlin"
  - text: "interface"
    category: "TypeScript"
  - text: "sealed class"
    category: "Kotlin"
  - text: "union type (|)"
    category: "TypeScript"
  - text: "val / var"
    category: "Kotlin"
  - text: "const / let"
    category: "TypeScript"
xp: 6
```

---

## 4. 유니온 타입과 인터섹션 타입

### 4-1. 유니온 타입 (Union Types): A | B

"A **또는** B" — 여러 타입 중 하나일 수 있음

```typescript
// TypeScript — 유니온 타입
// 기본 유니온
let value: string | number;
value = "hello"; // OK
value = 42;      // OK
// value = true; // 에러: boolean은 허용되지 않음

// 리터럴 유니온 — enum 대신 사용하는 경우가 많음
type Direction = "up" | "down" | "left" | "right";
type HttpStatus = 200 | 201 | 400 | 401 | 404 | 500;

function move(direction: Direction): void {
  console.log(`Moving ${direction}`);
}
move("up");    // OK
// move("diagonal"); // 에러!

// 판별 유니온 (Discriminated Union) — Kotlin의 sealed class에 해당!
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; side: number }
  | { kind: "rectangle"; width: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.side ** 2;
    case "rectangle":
      return shape.width * shape.height;
  }
}

const circle: Shape = { kind: "circle", radius: 10 };
console.log(getArea(circle)); // 314.159...

// React Native에서의 실제 사용
type ApiState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

function renderState(state: ApiState<string[]>) {
  switch (state.status) {
    case "idle":
      return "대기 중";
    case "loading":
      return "로딩 중...";
    case "success":
      return `데이터: ${state.data.join(", ")}`;
      // state.data는 string[]로 타입이 좁혀짐!
    case "error":
      return `에러: ${state.error}`;
      // state.error는 string으로 타입이 좁혀짐!
  }
}
```

```kotlin
// Kotlin — sealed class/interface가 판별 유니온에 해당
sealed class Shape {
    data class Circle(val radius: Double) : Shape()
    data class Square(val side: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
}

fun getArea(shape: Shape): Double = when (shape) {
    is Shape.Circle -> Math.PI * shape.radius * shape.radius
    is Shape.Square -> shape.side * shape.side
    is Shape.Rectangle -> shape.width * shape.height
}

// sealed interface (Kotlin 1.5+)
sealed interface ApiState<out T> {
    data object Idle : ApiState<Nothing>
    data object Loading : ApiState<Nothing>
    data class Success<T>(val data: T) : ApiState<T>
    data class Error(val error: String) : ApiState<Nothing>
}
```

### 4-2. 인터섹션 타입 (Intersection Types): A & B

"A **그리고** B" — 모든 타입의 속성을 가짐

```typescript
// TypeScript — 인터섹션 타입
type HasName = {
  name: string;
};

type HasAge = {
  age: number;
};

type HasEmail = {
  email: string;
};

// 세 타입의 속성을 모두 가져야 함
type Person = HasName & HasAge & HasEmail;

const person: Person = {
  name: "홍길동",
  age: 30,
  email: "hong@example.com"
  // 세 가지 모두 있어야 함 — 하나라도 빠지면 에러
};

// 실용적 사용 — 기존 타입에 속성 추가
type WithTimestamp<T> = T & {
  createdAt: Date;
  updatedAt: Date;
};

type User = {
  id: number;
  name: string;
};

type UserWithTimestamp = WithTimestamp<User>;
// { id: number; name: string; createdAt: Date; updatedAt: Date; }
```

```kotlin
// Kotlin — 인터페이스 다중 구현이 인터섹션에 해당
interface HasName { val name: String }
interface HasAge { val age: Int }
interface HasEmail { val email: String }

data class Person(
    override val name: String,
    override val age: Int,
    override val email: String
) : HasName, HasAge, HasEmail
```

---

## 5. 제네릭 (Generics)

### 5-1. 함수 제네릭

```typescript
// TypeScript — 제네릭 함수
// <T>: 타입 매개변수 (호출 시 구체적 타입으로 대체)
function identity<T>(value: T): T {
  return value;
}

const str = identity<string>("hello"); // 명시적 타입 지정
const num = identity(42);              // number로 자동 추론

// 여러 타입 매개변수
function pair<A, B>(first: A, second: B): [A, B] {
  return [first, second];
}
const result = pair("hello", 42); // [string, number]

// 배열 관련 제네릭 함수
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
const firstNum = first([1, 2, 3]);       // number | undefined
const firstStr = first(["a", "b", "c"]); // string | undefined

// React Native에서의 실제 사용 — API 응답 래퍼
async function fetchData<T>(url: string): Promise<T> {
  const response = await fetch(url);
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json() as Promise<T>;
}

// 사용
interface User { id: number; name: string; }
const user = await fetchData<User>("/api/users/1");
// user는 User 타입
```

```kotlin
// Kotlin — 거의 동일한 문법
fun <T> identity(value: T): T = value
val str = identity("hello") // String으로 추론

fun <A, B> pair(first: A, second: B): Pair<A, B> = Pair(first, second)

fun <T> first(list: List<T>): T? = list.firstOrNull()
```

### 5-2. 인터페이스/타입 제네릭

```typescript
// TypeScript — 제네릭 인터페이스
interface ApiResponse<T> {
  status: number;
  message: string;
  data: T;
  timestamp: Date;
}

// 다양한 데이터 타입으로 사용
const userResponse: ApiResponse<{ name: string; age: number }> = {
  status: 200,
  message: "성공",
  data: { name: "홍길동", age: 30 },
  timestamp: new Date()
};

const listResponse: ApiResponse<string[]> = {
  status: 200,
  message: "성공",
  data: ["항목1", "항목2", "항목3"],
  timestamp: new Date()
};

// 제네릭 타입 별칭
type Nullable<T> = T | null;
type Optional<T> = T | undefined;
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

// 기본 타입 매개변수 (default generic)
interface PaginatedResponse<T, M = {}> {
  data: T[];
  page: number;
  totalPages: number;
  meta: M;
}
// M을 지정하지 않으면 {}가 기본값
```

```kotlin
// Kotlin — 동일한 패턴
data class ApiResponse<T>(
    val status: Int,
    val message: String,
    val data: T,
    val timestamp: Date
)

// 사용
val userResponse = ApiResponse(200, "성공", User("홍길동", 30), Date())
```

### 5-3. 제네릭 제약 (Constraints)

```typescript
// TypeScript — extends로 제약 조건 설정
// T는 반드시 { id: number } 속성을 가져야 함
function findById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);
}

interface User { id: number; name: string; }
interface Product { id: number; price: number; }

const users: User[] = [{ id: 1, name: "홍길동" }];
const products: Product[] = [{ id: 1, price: 10000 }];

findById(users, 1);    // OK — User는 { id: number }를 가짐
findById(products, 1); // OK — Product도 { id: number }를 가짐
// findById([{ name: "test" }], 1); // 에러! — { id: number }가 없음

// keyof를 사용한 제약
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: "홍길동", age: 30, email: "hong@example.com" };
getProperty(user, "name");  // OK — 반환 타입: string
getProperty(user, "age");   // OK — 반환 타입: number
// getProperty(user, "phone"); // 에러! — "phone"은 user의 키가 아님
```

```kotlin
// Kotlin — where절 또는 : 로 제약
fun <T : Comparable<T>> findMax(items: List<T>): T? = items.maxOrNull()

// 여러 제약
fun <T> processItem(item: T) where T : Serializable, T : Comparable<T> {
    // T는 Serializable이면서 Comparable이어야 함
}
```

---

## 6. 유틸리티 타입 (Utility Types)

TypeScript가 기본 제공하는 강력한 타입 변환 도구들입니다.
Kotlin에는 대응하는 기능이 없으며, TypeScript만의 큰 장점입니다.

### 6-1. Partial\<T\> — 모든 속성을 선택적으로

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
}

// Partial<User> = { id?: number; name?: string; email?: string; age?: number; }
// 모든 속성이 선택적(optional)이 됨

function updateUser(id: number, updates: Partial<User>): void {
  // updates는 User의 일부 속성만 포함할 수 있음
  console.log(`Updating user ${id} with`, updates);
}

updateUser(1, { name: "새이름" });              // OK
updateUser(1, { age: 31, email: "new@test.com" }); // OK
updateUser(1, {});                               // OK — 아무것도 안 바꿔도 됨
```

### 6-2. Required\<T\> — 모든 속성을 필수로

```typescript
interface Config {
  apiUrl?: string;
  timeout?: number;
  retries?: number;
}

// Required<Config> = { apiUrl: string; timeout: number; retries: number; }
// 모든 선택적 속성이 필수가 됨

const fullConfig: Required<Config> = {
  apiUrl: "https://api.example.com",
  timeout: 5000,
  retries: 3
  // 하나라도 빠지면 에러!
};
```

### 6-3. Pick\<T, K\> — 특정 속성만 선택

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  address: string;
}

// Pick<User, "id" | "name"> = { id: number; name: string; }
type UserPreview = Pick<User, "id" | "name">;

const preview: UserPreview = {
  id: 1,
  name: "홍길동"
  // email, age, address는 포함하지 않음
};
```

### 6-4. Omit\<T, K\> — 특정 속성만 제외

```typescript
// Omit<User, "address"> = { id: number; name: string; email: string; age: number; }
type UserWithoutAddress = Omit<User, "address">;

// 여러 속성 제외
type UserBasic = Omit<User, "address" | "age">;
// { id: number; name: string; email: string; }

// 실무에서 자주 쓰는 패턴 — 생성 요청에서 id 제외
type CreateUserRequest = Omit<User, "id">;
// id는 서버에서 자동 생성하므로 요청에 포함하지 않음
```

### 6-5. Record\<K, V\> — 키-값 맵 타입

```typescript
// Record<Keys, ValueType> — 키 타입과 값 타입을 지정한 객체
type Fruit = "apple" | "banana" | "grape";
type FruitPrice = Record<Fruit, number>;

const prices: FruitPrice = {
  apple: 3000,
  banana: 2000,
  grape: 5000
  // 모든 과일이 있어야 함 — 하나라도 빠지면 에러
};

// 동적 키
type UserScores = Record<string, number>;
const scores: UserScores = {
  math: 90,
  english: 85,
  science: 95
};

// 실무 패턴 — 상태별 스타일 매핑
type StatusStyle = Record<Status, { color: string; icon: string }>;
const statusStyles: StatusStyle = {
  loading: { color: "gray", icon: "spinner" },
  success: { color: "green", icon: "check" },
  error: { color: "red", icon: "x" },
  idle: { color: "blue", icon: "minus" }
};
```

### 6-6. Readonly\<T\>

```typescript
interface User {
  id: number;
  name: string;
}

const user: Readonly<User> = { id: 1, name: "홍길동" };
// user.name = "김철수"; // 에러! — 모든 속성이 readonly

// 깊은 Readonly는 직접 만들어야 함
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
```

### 6-7. ReturnType\<T\> / Parameters\<T\>

```typescript
// ReturnType — 함수의 반환 타입을 추출
function getUser() {
  return { id: 1, name: "홍길동", age: 30 };
}
type UserType = ReturnType<typeof getUser>;
// { id: number; name: string; age: number; }

// Parameters — 함수의 매개변수 타입을 튜플로 추출
function createUser(name: string, age: number, email: string): void { }
type CreateUserParams = Parameters<typeof createUser>;
// [string, number, string]

// Awaited — Promise를 언래핑
type PromiseResult = Awaited<Promise<string>>; // string
type NestedPromise = Awaited<Promise<Promise<number>>>; // number

// 실무 사용 — API 함수의 반환 타입 재사용
async function fetchUsers() {
  const response = await fetch("/api/users");
  return response.json() as Promise<{ id: number; name: string }[]>;
}
type Users = Awaited<ReturnType<typeof fetchUsers>>;
// { id: number; name: string; }[]
```

---

## 7. 타입 좁히기 (Type Narrowing)

### 7-1. typeof 가드

```typescript
// TypeScript — typeof로 타입 좁히기
function formatValue(value: string | number | boolean): string {
  if (typeof value === "string") {
    // 이 블록 안에서 value는 string 타입
    return value.toUpperCase(); // string 메서드 사용 가능
  }
  if (typeof value === "number") {
    // 이 블록 안에서 value는 number 타입
    return value.toFixed(2); // number 메서드 사용 가능
  }
  // 여기서 value는 boolean 타입
  return value ? "참" : "거짓";
}
```

### 7-2. instanceof 가드

```typescript
// TypeScript — instanceof로 클래스 인스턴스 체크
class Dog {
  bark() { return "멍멍!"; }
}

class Cat {
  meow() { return "야옹!"; }
}

function makeSound(animal: Dog | Cat): string {
  if (animal instanceof Dog) {
    return animal.bark(); // Dog 메서드 사용 가능
  }
  return animal.meow(); // Cat 메서드 사용 가능
}
```

```kotlin
// Kotlin — is 연산자로 스마트 캐스트
fun makeSound(animal: Any): String = when (animal) {
    is Dog -> animal.bark()  // 스마트 캐스트
    is Cat -> animal.meow()  // 스마트 캐스트
    else -> "알 수 없는 동물"
}
```

### 7-3. in 연산자로 속성 존재 여부 체크

```typescript
// TypeScript — in 연산자
interface Fish {
  swim: () => void;
}
interface Bird {
  fly: () => void;
}

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    // animal은 Fish 타입으로 좁혀짐
    animal.swim();
  } else {
    // animal은 Bird 타입으로 좁혀짐
    animal.fly();
  }
}
```

### 7-4. 판별 유니온 (Discriminated Union) — 가장 실용적

```typescript
// TypeScript — kind/type 필드로 구분하는 패턴
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function handleResult(result: Result<string[]>) {
  if (result.success) {
    // result.data 사용 가능 (string[])
    console.log("데이터:", result.data.join(", "));
  } else {
    // result.error 사용 가능 (string)
    console.error("에러:", result.error);
  }
}

// 사용자 정의 타입 가드 (custom type guard)
interface User { type: "user"; name: string; }
interface Admin { type: "admin"; name: string; permissions: string[]; }

// 반환 타입에 'is'를 사용하여 타입 가드 선언
function isAdmin(person: User | Admin): person is Admin {
  return person.type === "admin";
}

function greet(person: User | Admin) {
  if (isAdmin(person)) {
    // person은 Admin으로 좁혀짐
    console.log(`관리자 ${person.name}, 권한: ${person.permissions.join(", ")}`);
  } else {
    console.log(`사용자 ${person.name}`);
  }
}
```

```kotlin
// Kotlin — sealed class로 동일한 패턴
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: String) : Result<Nothing>()
}

fun handleResult(result: Result<List<String>>) {
    when (result) {
        is Result.Success -> println("데이터: ${result.data.joinToString(", ")}")
        is Result.Failure -> println("에러: ${result.error}")
    }
}
```

---

## 8. 타입 단언 (Type Assertions)

### 8-1. as 키워드

```typescript
// TypeScript — as (타입 단언)
// "이 값은 이 타입이다"라고 컴파일러에게 알려줌
// ⚠️ 런타임 체크가 아님! 잘못 사용하면 런타임 에러 발생 가능

// DOM 요소 접근 시 자주 사용
const input = document.getElementById("myInput") as HTMLInputElement;
input.value = "hello"; // HTMLInputElement로 단언했으므로 .value 접근 가능

// API 응답 처리
const data = JSON.parse(jsonString) as User;

// 또는 꺾쇠 괄호 문법 (JSX에서는 사용 불가!)
const input2 = <HTMLInputElement>document.getElementById("myInput");

// as 대신 unknown을 거치는 이중 단언 (위험!)
const value = "hello" as unknown as number;
// ⚠️ 이렇게 하면 안 됩니다. 타입 시스템을 완전히 우회합니다.

// const assertions — 리터럴 타입으로 좁히기
const colors = ["red", "green", "blue"] as const;
// 타입: readonly ["red", "green", "blue"]
// 일반적으로는: string[]

const config = {
  endpoint: "/api/v1",
  timeout: 5000
} as const;
// 타입: { readonly endpoint: "/api/v1"; readonly timeout: 5000; }
// as const 없이는: { endpoint: string; timeout: number; }
```

```kotlin
// Kotlin — as (캐스트)
// Kotlin의 as는 실제 런타임 캐스트! TypeScript의 as와 다름

val obj: Any = "hello"
val str = obj as String  // 성공 — 실제로 String이므로
// val num = obj as Int  // ClassCastException — 런타임 에러!

// 안전한 캐스트
val num = obj as? Int    // null 반환 (에러 대신)
```

**핵심 차이**:
- TypeScript `as`: 컴파일 타임에만 존재, 런타임에는 아무 효과 없음
- Kotlin `as`: 런타임에 실제로 타입을 확인하고 캐스트

### 8-2. Non-null Assertion (!)

```typescript
// TypeScript — ! (Non-null Assertion)
// null/undefined가 아니라고 단언

function processUser(name: string | null) {
  // name!는 "name은 절대 null이 아니다"라고 선언
  const upperName = name!.toUpperCase();
  // ⚠️ 만약 name이 실제로 null이면 런타임 에러!
}

// 더 안전한 방법
function processUserSafe(name: string | null) {
  if (name === null) {
    throw new Error("name은 필수입니다");
  }
  // 여기서 name은 자동으로 string 타입 (타입 좁히기)
  const upperName = name.toUpperCase(); // ! 불필요
}
```

```kotlin
// Kotlin — !! (Non-null Assertion)
// 동일한 개념!
fun processUser(name: String?) {
    val upperName = name!!.uppercase()
    // name이 null이면 NullPointerException 발생
}

// 안전한 방법
fun processUserSafe(name: String?) {
    val upperName = name?.uppercase() ?: throw IllegalArgumentException("name 필수")
}
```

---

## 9. Null 처리: strictNullChecks

### 9-1. strictNullChecks 옵션

```typescript
// tsconfig.json에서 "strictNullChecks": true (기본값: strict: true에 포함)
// 이 옵션이 켜지면 null/undefined를 명시적으로 처리해야 함

// strictNullChecks: true (권장)
let name: string = "홍길동";
// name = null; // 에러! string 타입에 null 할당 불가

let nullableName: string | null = "홍길동";
nullableName = null; // OK — | null 을 명시했으므로

// 선택적 속성
interface User {
  name: string;
  age?: number;  // number | undefined와 동일
}

const user: User = { name: "홍길동" };
// user.age는 number | undefined 타입

// 안전하게 사용하는 방법들
// 1) Optional chaining
console.log(user.age?.toFixed(2)); // undefined (에러 아님)

// 2) Nullish coalescing
const age = user.age ?? 0; // number 타입 확정

// 3) 타입 가드
if (user.age !== undefined) {
  console.log(user.age.toFixed(2)); // number로 좁혀짐
}

// 4) Non-null assertion (가능하면 피하세요)
// console.log(user.age!.toFixed(2)); // 위험!
```

```kotlin
// Kotlin — 널 안전성은 언어 차원에서 강제
val name: String = "홍길동"
// name = null // 컴파일 에러

val nullableName: String? = "홍길동"
// nullableName.length // 컴파일 에러 — ?. 또는 !! 필요
nullableName?.length   // OK
nullableName!!.length  // OK (위험)
```

**비교 요약**:
- TypeScript: `strictNullChecks`가 있어야 null 안전, `?`로 선택적 표시, `| null`로 nullable
- Kotlin: 항상 null 안전, `?`로 nullable 표시

---

## 10. 선언 파일 (.d.ts)

### 10-1. .d.ts 파일이란?

`.d.ts` 파일은 JavaScript 라이브러리의 **타입 정보만** 담고 있는 파일입니다.
실행 코드는 없고, TypeScript 컴파일러에게 타입 정보를 제공하는 역할입니다.

```typescript
// === types.d.ts ===
// 타입 선언만 포함 (구현 없음)

// 전역 변수 선언
declare const API_URL: string;
declare const IS_DEV: boolean;

// 전역 함수 선언
declare function formatDate(date: Date): string;

// 모듈 선언 — 타입이 없는 JavaScript 라이브러리에 타입 추가
declare module "some-untyped-library" {
  export function doSomething(input: string): number;
  export const VERSION: string;
}

// 이미지 파일 등 비-JS 모듈의 타입 선언
declare module "*.png" {
  const value: number; // React Native에서 이미지는 number(리소스 ID)
  export default value;
}

declare module "*.svg" {
  import { SvgProps } from "react-native-svg";
  const content: React.FC<SvgProps>;
  export default content;
}
```

### 10-2. DefinitelyTyped와 @types 패키지

```bash
# JavaScript 라이브러리에 TypeScript 타입 추가
# @types/xxx 패키지를 설치하면 자동으로 타입이 적용됨

npm install lodash                  # JavaScript 라이브러리
npm install --save-dev @types/lodash # 타입 정의

# React Native 관련
npm install --save-dev @types/react @types/react-native
```

```kotlin
// Kotlin에서는 모든 라이브러리가 타입 정보를 포함하고 있으므로
// 별도의 타입 정의 패키지가 필요 없음
// 이것은 TypeScript만의 특수한 상황
```

---

## 11. tsconfig.json 핵심 옵션

```jsonc
// tsconfig.json — React Native 프로젝트에서 사용하는 주요 옵션
{
  "compilerOptions": {
    // === 기본 설정 ===
    "target": "esnext",
    // 컴파일 대상 JavaScript 버전 (React Native는 Hermes 엔진이 변환하므로 esnext)

    "module": "commonjs",
    // 모듈 시스템 (React Native는 Metro 번들러가 처리하므로 commonjs)

    "lib": ["es2022"],
    // 사용할 표준 라이브러리 타입 (Promise, Map, Set 등의 타입 포함 여부)

    "jsx": "react-native",
    // JSX 처리 방식 (react-native: JSX를 그대로 유지)

    // === 엄격한 타입 체크 (모두 켜는 것을 강력 권장) ===
    "strict": true,
    // 아래 옵션들을 한꺼번에 활성화:
    // "strictNullChecks": true     — null/undefined 엄격 체크
    // "strictFunctionTypes": true  — 함수 타입 엄격 체크
    // "strictBindCallApply": true  — bind/call/apply 타입 체크
    // "noImplicitAny": true        — 암묵적 any 금지
    // "noImplicitThis": true       — 암묵적 this 금지

    // === 모듈 해석 ===
    "moduleResolution": "node",
    // import 경로를 node_modules에서 찾는 방식

    "resolveJsonModule": true,
    // .json 파일 import 허용

    "allowSyntheticDefaultImports": true,
    // default export가 없는 모듈에서 import X from "..." 허용

    "esModuleInterop": true,
    // CommonJS/ES Module 호환성 개선

    // === 경로 설정 ===
    "baseUrl": ".",
    // import 기준 경로

    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@screens/*": ["src/screens/*"]
    },
    // 경로 별칭 (import Button from "@components/Button")

    // === 기타 유용한 옵션 ===
    "skipLibCheck": true,
    // .d.ts 파일의 타입 체크 건너뛰기 (빌드 속도 향상)

    "forceConsistentCasingInFileNames": true,
    // 파일명 대소문자 일관성 강제

    "noEmit": true,
    // TypeScript 컴파일러가 JavaScript를 출력하지 않음
    // (Metro 번들러가 대신 처리)

    "isolatedModules": true,
    // 파일 단위로 트랜스파일 가능하도록 강제 (Babel/Metro 호환)

    "noUnusedLocals": true,
    // 사용하지 않는 지역 변수 에러

    "noUnusedParameters": true
    // 사용하지 않는 매개변수 에러
  },

  "include": ["src/**/*"],
  // 타입 체크할 파일 경로

  "exclude": ["node_modules", "babel.config.js", "metro.config.js"]
  // 제외할 파일
}
```

```kotlin
// Kotlin에서의 대응:
// build.gradle.kts에서 컴파일 옵션 설정
// kotlinOptions {
//     jvmTarget = "17"
//     freeCompilerArgs += listOf("-Xjsr305=strict")
// }
// TypeScript의 tsconfig.json은 Kotlin의 build.gradle에 해당한다고 볼 수 있음
```

---

## 핵심 요약: TypeScript vs Kotlin 타입 시스템

| 개념 | TypeScript | Kotlin |
|------|-----------|--------|
| 타입 선언 | `name: string` | `name: String` |
| 타입 추론 | `const x = 1` → number | `val x = 1` → Int |
| Nullable | `string \| null` | `String?` |
| 안전 호출 | `obj?.prop` | `obj?.prop` |
| Non-null 단언 | `value!` | `value!!` |
| 타입 가드 | `typeof`, `instanceof`, `in` | `is` (스마트 캐스트) |
| 타입 캐스트 | `value as Type` (컴파일만) | `value as Type` (런타임) |
| 안전 캐스트 | — (unknown 거쳐서) | `value as? Type` |
| 인터페이스 | `interface` | `interface` |
| 타입 별칭 | `type` | `typealias` |
| 유니온 | `A \| B` | `sealed class/interface` |
| 인터섹션 | `A & B` | 다중 인터페이스 구현 |
| 제네릭 | `<T>`, `<T extends X>` | `<T>`, `<T : X>` |
| enum | `enum` (숫자/문자열) | `enum class` |
| 유틸리티 타입 | `Partial<T>`, `Pick<T,K>` | — (없음) |
| 선언 파일 | `.d.ts` | — (불필요) |
| 설정 파일 | `tsconfig.json` | `build.gradle.kts` |

---

## 자주 쓰는 TypeScript 패턴 모음

```typescript
// 1. 컴포넌트 Props 타입 정의
interface ButtonProps {
  title: string;
  onPress: () => void;
  disabled?: boolean;
  style?: object;
}

// 2. API 응답 타입
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
}

// 3. 상태 유니온
type LoadingState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

// 4. 이벤트 핸들러 타입
type EventHandler<T = void> = (event: T) => void;

// 5. Record로 맵 타입
type RouteParams = Record<string, string | undefined>;

// 6. 객체 키를 타입으로 추출
const COLORS = { primary: "#007AFF", secondary: "#5856D6" } as const;
type ColorKey = keyof typeof COLORS;     // "primary" | "secondary"
type ColorValue = typeof COLORS[ColorKey]; // "#007AFF" | "#5856D6"

// 7. 함수 오버로드 시그니처
function createElement(type: "button"): HTMLButtonElement;
function createElement(type: "input"): HTMLInputElement;
function createElement(type: string): HTMLElement;
function createElement(type: string): HTMLElement {
  return document.createElement(type);
}
```

---

> **Phase 00 완료!** 이제 React 기초(Phase 01)로 넘어갈 준비가 되었습니다.
> JavaScript/TypeScript 문법이 헷갈릴 때마다 이 문서들을 참고하세요.
