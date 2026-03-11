# 비동기 프로그래밍 완전 정복 — Kotlin Coroutine 개발자를 위한 가이드

> JavaScript의 비동기 프로그래밍을 Kotlin Coroutine과 비교하며 상세하게 설명합니다.
> React Native에서 API 호출, 파일 처리, 타이머 등 거의 모든 작업이 비동기로 이루어지므로
> 이 내용은 매우 중요합니다.

---

## 1. 이벤트 루프 (Event Loop) — JavaScript의 동작 원리

### 1-1. 싱글 스레드라는 의미

JavaScript는 **싱글 스레드(Single Thread)**입니다. 한 번에 하나의 작업만 실행할 수 있습니다.
그런데 어떻게 네트워크 요청, 타이머, 파일 읽기 등을 **동시에** 처리할 수 있을까요?
바로 **이벤트 루프(Event Loop)** 덕분입니다.

### 1-2. 이벤트 루프 구조

```
┌─────────────────────────────────────────┐
│           Call Stack (호출 스택)          │  ← 현재 실행 중인 함수
│  (한 번에 하나의 함수만 실행)              │
└────────────────────┬────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────┐
│           Web APIs / Node APIs           │  ← 비동기 작업 처리 (별도 스레드)
│  (setTimeout, fetch, DOM events 등)      │     타이머, 네트워크, I/O 등
└────────────────────┬────────────────────┘
                     │  완료되면 콜백을
                     │  큐에 추가
                     ▼
┌─────────────────────────────────────────┐
│        Callback Queue (콜백 큐)          │
│  ┌─────────────────────────────────┐    │
│  │ Microtask Queue (마이크로태스크)  │    │  ← Promise, queueMicrotask
│  │ (우선순위 높음)                   │    │     매 태스크 후 전부 처리
│  └─────────────────────────────────┘    │
│  ┌─────────────────────────────────┐    │
│  │ Macrotask Queue (매크로태스크)    │    │  ← setTimeout, setInterval, I/O
│  │ (우선순위 낮음)                   │    │     한 번에 하나씩 처리
│  └─────────────────────────────────┘    │
└────────────────────┬────────────────────┘
                     │
        ◄── Event Loop가 Call Stack이
            비었을 때 큐에서 꺼내서
            Call Stack에 넣음
```

### 1-3. 실행 순서 이해 — 매우 중요!

```javascript
// JavaScript — 이벤트 루프 실행 순서 예제
console.log("1. 동기 코드 시작");  // (1) 바로 실행

setTimeout(() => {
  console.log("4. setTimeout 콜백"); // (4) 매크로태스크 큐 → 마지막에 실행
}, 0); // 0ms여도 바로 실행되지 않음!

Promise.resolve().then(() => {
  console.log("3. Promise 콜백");   // (3) 마이크로태스크 큐 → setTimeout보다 먼저
});

console.log("2. 동기 코드 끝");    // (2) 바로 실행

// 출력 순서:
// 1. 동기 코드 시작
// 2. 동기 코드 끝
// 3. Promise 콜백      ← setTimeout(0)보다 먼저!
// 4. setTimeout 콜백
```

**실행 흐름 분석**:
1. `console.log("1...")` → Call Stack에서 즉시 실행
2. `setTimeout(callback, 0)` → Web API에 등록, 0ms 후 **매크로태스크 큐**에 콜백 추가
3. `Promise.resolve().then(callback)` → **마이크로태스크 큐**에 콜백 추가
4. `console.log("2...")` → Call Stack에서 즉시 실행
5. Call Stack이 비었으므로 이벤트 루프가 **마이크로태스크 큐**를 먼저 확인 → `"3. Promise 콜백"` 실행
6. 마이크로태스크 큐가 비었으므로 **매크로태스크 큐** 확인 → `"4. setTimeout 콜백"` 실행

```javascript [playground]
// 🧪 이벤트 루프 실행 순서 실습

console.log("1. 동기 코드 시작");

setTimeout(() => {
  console.log("4. setTimeout 콜백 (매크로태스크)");
}, 0);

Promise.resolve().then(() => {
  console.log("3. Promise 콜백 (마이크로태스크)");
});

console.log("2. 동기 코드 끝");

// 출력 순서를 예측해보세요!
// 마이크로태스크(Promise)가 매크로태스크(setTimeout)보다 먼저 실행됩니다.
```

### 1-4. Kotlin Coroutine과의 비교

```kotlin
// Kotlin — Coroutine은 멀티 스레드 기반
// Dispatcher가 스레드를 관리

import kotlinx.coroutines.*

fun main() = runBlocking {
    println("1. 코루틴 시작") // 메인 스레드

    // Dispatchers.IO — I/O 작업용 스레드 풀
    launch(Dispatchers.IO) {
        println("2. IO 스레드에서 실행: ${Thread.currentThread().name}")
    }

    // Dispatchers.Default — CPU 집약적 작업용
    launch(Dispatchers.Default) {
        println("3. Default 스레드에서 실행: ${Thread.currentThread().name}")
    }

    // Dispatchers.Main — UI 스레드 (Android)
    // launch(Dispatchers.Main) { updateUI() }
}
```

**핵심 차이점**:

| 특성 | JavaScript | Kotlin Coroutine |
|------|-----------|-----------------|
| 스레드 | 싱글 스레드 | 멀티 스레드 |
| 동시성 모델 | 이벤트 루프 | Dispatcher + 스레드 풀 |
| 컨텍스트 전환 | 큐(마이크로/매크로태스크) | Dispatcher 전환 |
| 블로킹 | 절대 불가 (UI 멈춤) | 가능 (별도 스레드) |
| 취소 | AbortController (수동) | structured concurrency (자동) |

---

## 2. 콜백 (Callback)

### 2-1. 콜백 패턴 기초

콜백은 JavaScript에서 가장 오래된 비동기 처리 패턴입니다.
"작업이 끝나면 이 함수를 호출해줘"라고 전달하는 것입니다.

```javascript
// JavaScript — 콜백 패턴
// 파일 읽기를 시뮬레이션
function readFile(path, callback) {
  // 실제로는 비동기 작업 (1초 후 완료된다고 가정)
  setTimeout(() => {
    if (path === "error.txt") {
      callback(new Error("파일을 찾을 수 없습니다"), null);
    } else {
      callback(null, `${path}의 내용입니다.`);
    }
  }, 1000);
}

// 사용
readFile("hello.txt", (error, data) => {
  if (error) {
    console.error("에러:", error.message);
    return;
  }
  console.log("데이터:", data); // "데이터: hello.txt의 내용입니다."
});

console.log("파일 읽기 요청 완료 (아직 결과 안 옴)");
// 출력 순서:
// "파일 읽기 요청 완료 (아직 결과 안 옴)" ← 즉시
// (1초 후) "데이터: hello.txt의 내용입니다."
```

### 2-2. 콜백 지옥 (Callback Hell)

비동기 작업을 순차적으로 실행하려면 콜백 안에 콜백을 넣어야 하는데,
이것이 깊어지면 **콜백 지옥**이 됩니다.

```javascript
// JavaScript — 콜백 지옥의 예
// 사용자 정보 조회 → 주문 목록 조회 → 첫 번째 주문 상세 조회
getUser(userId, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }
  getOrders(user.id, (error, orders) => {
    if (error) {
      console.error(error);
      return;
    }
    getOrderDetail(orders[0].id, (error, detail) => {
      if (error) {
        console.error(error);
        return;
      }
      getShippingInfo(detail.shippingId, (error, shipping) => {
        if (error) {
          console.error(error);
          return;
        }
        console.log("배송 정보:", shipping);
        // 더 깊어질 수 있음...
      });
    });
  });
});
// 들여쓰기가 점점 깊어지면서 코드를 읽고 유지보수하기 어려워짐
// → Promise와 async/await로 해결!
```

```javascript [playground]
// 🧪 콜백 패턴 실습

// 비동기 작업을 시뮬레이션하는 함수
function fetchData(id, callback) {
  setTimeout(() => {
    if (id > 0) {
      callback(null, { id, name: `사용자${id}` });
    } else {
      callback(new Error("유효하지 않은 ID"), null);
    }
  }, 100);
}

// 성공 케이스
fetchData(1, (error, data) => {
  if (error) {
    console.error("에러:", error.message);
    return;
  }
  console.log("성공:", JSON.stringify(data));
});

// 실패 케이스
fetchData(-1, (error, data) => {
  if (error) {
    console.error("에러:", error.message);
    return;
  }
  console.log("데이터:", data);
});

console.log("요청 전송 완료 (결과는 아직 안 옴)");
```

---

## 3. Promise — 비동기의 핵심

### 3-1. Promise란?

Promise는 **"미래에 완료될 비동기 작업의 결과를 나타내는 객체"**입니다.

세 가지 상태를 가집니다:
- **Pending (대기)**: 아직 결과가 나오지 않은 초기 상태
- **Fulfilled (이행)**: 작업이 성공적으로 완료됨 → `resolve(값)` 호출
- **Rejected (거부)**: 작업이 실패함 → `reject(에러)` 호출

```
         ┌──── resolve(value) ───→ Fulfilled (이행)
Pending ─┤
         └──── reject(error)  ───→ Rejected (거부)
```

### 3-2. Promise 생성

```javascript
// JavaScript — Promise 생성
// new Promise((resolve, reject) => { ... })

// 성공하는 Promise
const successPromise = new Promise((resolve, reject) => {
  // 비동기 작업 시뮬레이션 (1초 후 완료)
  setTimeout(() => {
    resolve("작업 성공!"); // Fulfilled 상태로 전환
  }, 1000);
});

// 실패하는 Promise
const failPromise = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject(new Error("작업 실패!")); // Rejected 상태로 전환
  }, 1000);
});

// 즉시 이행/거부되는 Promise (테스트에 유용)
const immediate = Promise.resolve("즉시 완료");
const immediateError = Promise.reject(new Error("즉시 실패"));

// 실용적인 예: API 호출을 Promise로 감싸기
function fetchUser(userId) {
  return new Promise((resolve, reject) => {
    // 가상의 비동기 작업
    setTimeout(() => {
      if (userId > 0) {
        resolve({ id: userId, name: "홍길동", age: 30 });
      } else {
        reject(new Error("유효하지 않은 사용자 ID"));
      }
    }, 500);
  });
}
```

```kotlin
// Kotlin — Deferred가 Promise에 해당
import kotlinx.coroutines.*

val deferred: Deferred<String> = GlobalScope.async {
    delay(1000)
    "작업 성공!" // 반환값
}
// deferred.await() // 결과를 기다림
```

### 3-3. Promise 체이닝 (then / catch / finally)

```javascript
// JavaScript — then/catch/finally로 Promise 결과 처리
fetchUser(1)
  .then(user => {
    // 성공 시 실행 — user는 resolve()에 전달한 값
    console.log("사용자:", user); // { id: 1, name: "홍길동", age: 30 }
    return user.id; // 다음 then에 전달할 값을 return
  })
  .then(userId => {
    // 이전 then의 반환값을 받음
    console.log("사용자 ID:", userId); // 1
    return fetchOrders(userId); // Promise를 반환하면 자동으로 언래핑
  })
  .then(orders => {
    console.log("주문 목록:", orders);
  })
  .catch(error => {
    // 체인 어디에서든 에러가 발생하면 여기로 옴
    console.error("에러 발생:", error.message);
  })
  .finally(() => {
    // 성공/실패 관계없이 항상 실행 (정리 작업에 유용)
    console.log("로딩 완료");
  });

// 콜백 지옥이 평탄한 체인으로 바뀜!
// getUser → getOrders → getOrderDetail → getShippingInfo
fetchUser(1)
  .then(user => fetchOrders(user.id))
  .then(orders => fetchOrderDetail(orders[0].id))
  .then(detail => fetchShippingInfo(detail.shippingId))
  .then(shipping => console.log("배송 정보:", shipping))
  .catch(error => console.error("에러:", error.message));
```

```javascript [playground]
// 🧪 Promise 체이닝 실습

// API 호출 시뮬레이션
function fetchUser(id) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (id > 0) resolve({ id, name: `사용자${id}`, role: "개발자" });
      else reject(new Error("유효하지 않은 ID"));
    }, 100);
  });
}

// 1) 기본 체이닝
fetchUser(1)
  .then(user => {
    console.log("사용자:", user.name);
    return user.role; // 다음 then에 전달
  })
  .then(role => {
    console.log("역할:", role);
  })
  .catch(error => {
    console.error("에러:", error.message);
  })
  .finally(() => {
    console.log("요청 완료!");
  });

// 2) 에러 처리
fetchUser(-1)
  .then(user => console.log(user))
  .catch(error => console.error("실패:", error.message));
```

### 3-4. Promise.all — 병렬 실행 후 모든 결과 기다리기

```javascript
// JavaScript — Promise.all([p1, p2, p3])
// 모든 Promise가 이행되면 결과 배열을 반환
// 하나라도 거부되면 즉시 catch로 이동

function fetchUser(id) {
  return new Promise(resolve => {
    setTimeout(() => resolve({ id, name: `사용자${id}` }), 1000);
  });
}

// 3개의 API를 동시에 호출 (병렬 실행)
const startTime = Date.now();

Promise.all([
  fetchUser(1),
  fetchUser(2),
  fetchUser(3)
])
  .then(([user1, user2, user3]) => {
    // 구조 분해로 결과를 받음
    console.log(user1); // { id: 1, name: "사용자1" }
    console.log(user2); // { id: 2, name: "사용자2" }
    console.log(user3); // { id: 3, name: "사용자3" }
    console.log(`소요 시간: ${Date.now() - startTime}ms`);
    // 약 1000ms — 순차 실행했으면 3000ms 걸렸을 것
  })
  .catch(error => {
    // 하나라도 실패하면 여기로 — 나머지 결과는 무시됨
    console.error("하나 이상 실패:", error.message);
  });
```

```javascript [playground]
// 🧪 Promise.all 병렬 실행 실습

function fetchItem(name, delay) {
  return new Promise(resolve => {
    setTimeout(() => resolve(`${name} 완료`), delay);
  });
}

const start = Date.now();

// 병렬 실행 — 모든 Promise가 완료될 때까지 기다림
Promise.all([
  fetchItem("사용자 조회", 100),
  fetchItem("주문 조회", 200),
  fetchItem("알림 조회", 150)
]).then(results => {
  const elapsed = Date.now() - start;
  console.log("결과:", results);
  console.log(`소요 시간: 약 ${elapsed}ms (순차면 450ms)`);
});
```

```kotlin
// Kotlin — awaitAll()이 Promise.all에 해당
import kotlinx.coroutines.*

suspend fun fetchUser(id: Int): Map<String, Any> {
    delay(1000)
    return mapOf("id" to id, "name" to "사용자$id")
}

runBlocking {
    val deferreds = listOf(
        async { fetchUser(1) },
        async { fetchUser(2) },
        async { fetchUser(3) }
    )
    val results = deferreds.awaitAll()
    // results: List<Map<String, Any>>
}
```

### 3-5. Promise.race — 가장 먼저 완료되는 것만

```javascript
// JavaScript — Promise.race([p1, p2, p3])
// 가장 먼저 이행 또는 거부되는 Promise의 결과를 반환

// 타임아웃 구현에 유용!
function fetchWithTimeout(url, timeoutMs) {
  const fetchPromise = fetch(url);
  const timeoutPromise = new Promise((_, reject) => {
    setTimeout(() => reject(new Error("타임아웃!")), timeoutMs);
  });

  return Promise.race([fetchPromise, timeoutPromise]);
}

// 3초 안에 응답이 오지 않으면 타임아웃
fetchWithTimeout("https://api.example.com/data", 3000)
  .then(response => console.log("성공:", response))
  .catch(error => console.error("에러:", error.message));
```

### 3-6. Promise.allSettled — 모든 결과 수집 (실패해도 계속)

```javascript
// JavaScript — Promise.allSettled([p1, p2, p3])
// 모든 Promise가 완료될 때까지 기다림 (성공이든 실패든)
// Promise.all과 달리 하나가 실패해도 나머지를 기다림

const promises = [
  Promise.resolve("성공 1"),
  Promise.reject(new Error("실패 2")),
  Promise.resolve("성공 3"),
];

Promise.allSettled(promises).then(results => {
  console.log(results);
  // [
  //   { status: "fulfilled", value: "성공 1" },
  //   { status: "rejected", reason: Error("실패 2") },
  //   { status: "fulfilled", value: "성공 3" }
  // ]

  // 성공한 것만 추출
  const successes = results
    .filter(r => r.status === "fulfilled")
    .map(r => r.value);
  console.log(successes); // ["성공 1", "성공 3"]

  // 실패한 것만 추출
  const failures = results
    .filter(r => r.status === "rejected")
    .map(r => r.reason.message);
  console.log(failures); // ["실패 2"]
});
```

### 3-7. Promise.any — 하나라도 성공하면

```javascript
// JavaScript — Promise.any([p1, p2, p3])
// 하나라도 이행되면 그 결과를 반환
// 모두 거부되면 AggregateError 발생

// 여러 서버에서 가장 빠른 응답 사용
const mirrors = [
  fetch("https://mirror1.example.com/data"),
  fetch("https://mirror2.example.com/data"),
  fetch("https://mirror3.example.com/data"),
];

Promise.any(mirrors)
  .then(response => {
    // 가장 먼저 성공한 응답 사용
    console.log("가장 빠른 응답:", response);
  })
  .catch(error => {
    // 모든 요청이 실패한 경우에만 여기로 옴
    console.error("모든 미러 실패:", error); // AggregateError
  });
```

---

## 4. async / await — Promise의 문법적 설탕

### 4-1. 기본 문법

`async`/`await`는 Promise를 더 읽기 쉽게 사용할 수 있게 해주는 문법입니다.
내부적으로는 Promise와 완전히 동일하게 동작합니다.

```javascript
// JavaScript — async/await 기본
// async 함수는 항상 Promise를 반환합니다
async function fetchUser(id) {
  // await는 Promise가 이행될 때까지 기다립니다
  // (실제로는 이벤트 루프를 블로킹하지 않음 — 다른 작업은 계속 진행)
  const response = await fetch(`https://api.example.com/users/${id}`);
  const user = await response.json(); // json() 도 Promise를 반환
  return user; // 자동으로 Promise.resolve(user)로 감싸짐
}

// 사용
fetchUser(1).then(user => console.log(user));

// 또는 다른 async 함수 안에서 await로 사용
async function main() {
  const user = await fetchUser(1);
  console.log(user);
}
main();
```

```javascript [playground]
// 🧪 async/await 기본 실습

function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetchUser(id) {
  await delay(100); // API 호출 시뮬레이션
  return { id, name: `사용자${id}`, level: id * 10 };
}

// async 함수 사용
async function main() {
  console.log("시작...");

  // 순차 실행
  const user1 = await fetchUser(1);
  console.log("user1:", JSON.stringify(user1));

  // 병렬 실행 (Promise.all + await)
  const [user2, user3] = await Promise.all([
    fetchUser(2),
    fetchUser(3)
  ]);
  console.log("user2:", JSON.stringify(user2));
  console.log("user3:", JSON.stringify(user3));

  console.log("완료!");
}

main();
```

```kotlin
// Kotlin — suspend 함수와 비교
// suspend fun은 JavaScript의 async function에 해당
suspend fun fetchUser(id: Int): User {
    // withContext로 디스패처 전환
    return withContext(Dispatchers.IO) {
        val response = apiClient.get("users/$id")
        response.body<User>()
    }
}

// 코루틴 스코프 안에서 호출
lifecycleScope.launch {
    val user = fetchUser(1) // await 키워드 없이 바로 사용
    println(user)
}
```

**핵심 비교**:

| JavaScript | Kotlin |
|-----------|--------|
| `async function` | `suspend fun` |
| `await promise` | 자동 (suspend 함수 호출 자체가 await) |
| `Promise` | `Deferred` |
| 최상위에서 await 불가 (async 함수 안에서만) | 코루틴 스코프 안에서만 suspend 호출 가능 |

### 4-2. 에러 처리 — try/catch

```javascript
// JavaScript — async/await에서의 에러 처리
async function fetchUserSafe(id) {
  try {
    const response = await fetch(`https://api.example.com/users/${id}`);

    // HTTP 에러 체크 (fetch는 404, 500 등에서도 reject하지 않음!)
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const user = await response.json();
    return user;
  } catch (error) {
    // 네트워크 에러, HTTP 에러, JSON 파싱 에러 등 모두 여기서 처리
    console.error("사용자 조회 실패:", error.message);
    return null; // 기본값 반환 또는 에러를 다시 throw
  } finally {
    // 항상 실행 — 로딩 상태 해제 등
    console.log("요청 완료");
  }
}

// 사용
async function main() {
  const user = await fetchUserSafe(1);
  if (user) {
    console.log("사용자:", user);
  } else {
    console.log("사용자를 찾을 수 없습니다");
  }
}
```

```javascript [playground]
// 🧪 async/await 에러 처리 실습

async function riskyOperation(shouldFail) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (shouldFail) reject(new Error("작업 실패!"));
      else resolve("작업 성공!");
    }, 100);
  });
}

async function main() {
  // 1) try/catch로 에러 처리
  try {
    const result = await riskyOperation(false);
    console.log("성공:", result);

    const result2 = await riskyOperation(true); // 여기서 에러!
    console.log("이 줄은 실행 안 됨");
  } catch (error) {
    console.error("잡힌 에러:", error.message);
  } finally {
    console.log("항상 실행되는 finally");
  }

  // 2) [data, error] 패턴
  async function to(promise) {
    try {
      return [await promise, null];
    } catch (error) {
      return [null, error];
    }
  }

  const [data, error] = await to(riskyOperation(true));
  if (error) console.log("[data, error] 패턴:", error.message);
}

main();
```

```kotlin
// Kotlin — 동일한 try/catch 패턴
suspend fun fetchUserSafe(id: Int): User? {
    return try {
        withContext(Dispatchers.IO) {
            apiClient.get("users/$id").body<User>()
        }
    } catch (e: Exception) {
        println("사용자 조회 실패: ${e.message}")
        null
    } finally {
        println("요청 완료")
    }
}
```

### 4-3. 병렬 실행 — async/await에서의 주의점

```javascript
// JavaScript — 잘못된 방법 (순차 실행 — 느림!)
async function fetchAllSequential() {
  // 각 await가 이전 작업이 끝날 때까지 기다림
  const user1 = await fetchUser(1); // 1초 대기
  const user2 = await fetchUser(2); // 1초 대기 (총 2초)
  const user3 = await fetchUser(3); // 1초 대기 (총 3초!)
  return [user1, user2, user3];
}

// 올바른 방법 1: Promise.all 사용
async function fetchAllParallel() {
  // 모든 Promise를 먼저 생성하고, 한꺼번에 기다림
  const [user1, user2, user3] = await Promise.all([
    fetchUser(1),
    fetchUser(2),
    fetchUser(3)
  ]);
  // 총 1초만 소요!
  return [user1, user2, user3];
}

// 올바른 방법 2: Promise를 먼저 생성
async function fetchAllParallel2() {
  // Promise는 생성 즉시 실행 시작
  const promise1 = fetchUser(1); // 바로 시작
  const promise2 = fetchUser(2); // 바로 시작
  const promise3 = fetchUser(3); // 바로 시작

  // 각각 await — 이미 실행 중이므로 병렬 효과
  const user1 = await promise1;
  const user2 = await promise2;
  const user3 = await promise3;
  return [user1, user2, user3];
}
```

```kotlin
// Kotlin — async/await 병렬 실행
suspend fun fetchAllParallel() = coroutineScope {
    // async는 코루틴을 즉시 시작 (JavaScript의 Promise 생성과 동일)
    val deferred1 = async { fetchUser(1) }
    val deferred2 = async { fetchUser(2) }
    val deferred3 = async { fetchUser(3) }

    // await로 결과 수집
    listOf(deferred1.await(), deferred2.await(), deferred3.await())
}

// 또는 awaitAll 사용 (Promise.all에 해당)
suspend fun fetchAllParallel2() = coroutineScope {
    val deferreds = (1..3).map { id ->
        async { fetchUser(id) }
    }
    deferreds.awaitAll()
}
```

### 4-4. for 루프에서의 async/await

```javascript
// JavaScript — 순차적으로 처리해야 하는 경우
async function processItemsSequential(items) {
  const results = [];
  // for...of 루프에서 await를 사용하면 순차 실행
  for (const item of items) {
    const result = await processItem(item); // 하나씩 처리
    results.push(result);
    console.log(`${item} 처리 완료`);
  }
  return results;
}

// 주의: forEach에서는 await가 제대로 동작하지 않음!
async function wrongWay(items) {
  items.forEach(async (item) => {
    await processItem(item); // 각 콜백은 독립적인 async 함수
    // forEach는 이 await를 기다리지 않음!
  });
  console.log("완료?"); // 모든 처리가 끝나기 전에 실행됨!
}

// 병렬로 처리하고 싶으면 map + Promise.all
async function processItemsParallel(items) {
  const results = await Promise.all(
    items.map(item => processItem(item))
  );
  return results;
}
```

### 4-5. 에러 처리 고급 패턴

```javascript
// JavaScript — 여러 비동기 작업의 에러를 개별 처리
async function fetchDashboard() {
  // 각 요청의 에러를 독립적으로 처리
  const [userResult, ordersResult, notificationsResult] =
    await Promise.allSettled([
      fetchUser(1),
      fetchOrders(1),
      fetchNotifications(1)
    ]);

  return {
    // 성공 시 값을 사용, 실패 시 기본값
    user: userResult.status === "fulfilled"
      ? userResult.value
      : { name: "알 수 없음" },
    orders: ordersResult.status === "fulfilled"
      ? ordersResult.value
      : [],
    notifications: notificationsResult.status === "fulfilled"
      ? notificationsResult.value
      : [],
  };
}

// React Native에서의 실제 패턴 — 유틸리티 함수
// 에러를 throw하지 않고 [data, error] 튜플로 반환
async function to(promise) {
  try {
    const data = await promise;
    return [data, null];
  } catch (error) {
    return [null, error];
  }
}

// 사용
async function main() {
  const [user, error] = await to(fetchUser(1));
  if (error) {
    console.error("에러:", error.message);
    return;
  }
  console.log("사용자:", user);
}
```

---

## 5. Kotlin과의 상세 비교

### 5-1. launch vs Promise

```kotlin
// Kotlin — launch는 결과를 반환하지 않는 코루틴 (fire and forget)
lifecycleScope.launch {
    val data = fetchData()
    updateUI(data)
}
// launch의 반환값은 Job (취소 등 제어용)
```

```javascript
// JavaScript — 가장 유사한 패턴
// Promise를 생성하고 결과를 기다리지 않음 (fire and forget)
fetchData()
  .then(data => updateUI(data))
  .catch(error => console.error(error));
// 또는
async function run() {
  try {
    const data = await fetchData();
    updateUI(data);
  } catch (error) {
    console.error(error);
  }
}
run(); // await 없이 호출 — fire and forget
```

### 5-2. Deferred vs Promise

```kotlin
// Kotlin — Deferred는 결과를 반환하는 코루틴
val deferred: Deferred<User> = scope.async {
    fetchUser(1)
}
val user = deferred.await() // 결과를 기다림
```

```javascript
// JavaScript — Promise가 Deferred에 해당
const promise = fetchUser(1); // Promise<User>
const user = await promise;   // 결과를 기다림
```

### 5-3. Flow vs AsyncIterator / Observable

```kotlin
// Kotlin — Flow (비동기 데이터 스트림)
fun getMessages(): Flow<Message> = flow {
    while (true) {
        val message = fetchLatestMessage()
        emit(message) // 값을 방출
        delay(1000)
    }
}

// 수집
lifecycleScope.launch {
    getMessages().collect { message ->
        println("새 메시지: $message")
    }
}
```

```javascript
// JavaScript — AsyncGenerator (Flow와 가장 유사)
async function* getMessages() {
  while (true) {
    const message = await fetchLatestMessage();
    yield message; // 값을 방출 (Kotlin의 emit에 해당)
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}

// 수집
for await (const message of getMessages()) {
  console.log("새 메시지:", message);
}

// 실제 React Native에서는 WebSocket이나 EventEmitter를 더 많이 사용
// 또는 RxJS, Observable 패턴을 사용
```

### 5-4. 취소(Cancellation) 비교

```kotlin
// Kotlin — structured concurrency로 자동 취소
val job = lifecycleScope.launch {
    val data = fetchData() // lifecycleScope가 취소되면 자동 취소
    updateUI(data)
}
job.cancel() // 수동 취소도 가능
```

```javascript
// JavaScript — AbortController로 수동 취소
const controller = new AbortController();
const signal = controller.signal;

// fetch에 signal 전달
fetch("https://api.example.com/data", { signal })
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => {
    if (error.name === "AbortError") {
      console.log("요청이 취소되었습니다");
    } else {
      console.error("에러:", error);
    }
  });

// 필요 시 취소
controller.abort(); // "요청이 취소되었습니다" 출력

// React Native에서의 실제 사용 — useEffect 정리
// useEffect(() => {
//   const controller = new AbortController();
//
//   fetchData(controller.signal)
//     .then(data => setData(data));
//
//   return () => controller.abort(); // 컴포넌트 언마운트 시 취소
// }, []);
```

---

## 6. fetch API — 네트워크 요청

### 6-1. GET 요청

```javascript
// JavaScript — fetch API
// fetch()는 Promise<Response>를 반환

// 기본 GET 요청
async function getUsers() {
  try {
    const response = await fetch("https://jsonplaceholder.typicode.com/users");

    // response.ok: 200~299 상태 코드인지 확인
    if (!response.ok) {
      throw new Error(`HTTP 에러: ${response.status}`);
    }

    // response.json()은 Promise를 반환 — await 필요!
    const users = await response.json();
    console.log(users);
    return users;
  } catch (error) {
    console.error("API 호출 실패:", error.message);
    throw error; // 에러를 다시 throw하여 호출자가 처리하게 함
  }
}

// 쿼리 파라미터 포함
async function searchUsers(query, page = 1) {
  const params = new URLSearchParams({
    q: query,
    page: String(page),
    limit: "10"
  });
  const response = await fetch(
    `https://api.example.com/users?${params}`
  );
  return response.json();
}
```

```kotlin
// Kotlin — Retrofit 사용 시
interface ApiService {
    @GET("users")
    suspend fun getUsers(): List<User>

    @GET("users")
    suspend fun searchUsers(
        @Query("q") query: String,
        @Query("page") page: Int = 1
    ): List<User>
}

// OkHttp 직접 사용 시
val client = OkHttpClient()
val request = Request.Builder()
    .url("https://api.example.com/users")
    .build()

client.newCall(request).enqueue(object : Callback {
    override fun onResponse(call: Call, response: Response) {
        val body = response.body?.string()
    }
    override fun onFailure(call: Call, e: IOException) {
        println("실패: ${e.message}")
    }
})
```

### 6-2. POST 요청

```javascript
// JavaScript — POST 요청
async function createUser(userData) {
  try {
    const response = await fetch("https://api.example.com/users", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer eyJhbGci...", // 인증 토큰
      },
      body: JSON.stringify(userData), // 객체를 JSON 문자열로 변환
    });

    if (!response.ok) {
      // 서버에서 보낸 에러 메시지 파싱
      const errorData = await response.json();
      throw new Error(errorData.message || `HTTP ${response.status}`);
    }

    const createdUser = await response.json();
    console.log("생성된 사용자:", createdUser);
    return createdUser;
  } catch (error) {
    console.error("사용자 생성 실패:", error.message);
    throw error;
  }
}

// 사용
createUser({
  name: "홍길동",
  email: "hong@example.com",
  age: 30
});
```

```kotlin
// Kotlin — Retrofit POST
interface ApiService {
    @POST("users")
    suspend fun createUser(@Body user: CreateUserRequest): User
}

// 사용
val user = apiService.createUser(
    CreateUserRequest(name = "홍길동", email = "hong@example.com", age = 30)
)
```

### 6-3. 다양한 HTTP 메서드와 에러 처리

```javascript
// JavaScript — PUT, PATCH, DELETE
async function updateUser(id, updates) {
  const response = await fetch(`https://api.example.com/users/${id}`, {
    method: "PUT", // 또는 "PATCH"
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(updates),
  });
  return response.json();
}

async function deleteUser(id) {
  const response = await fetch(`https://api.example.com/users/${id}`, {
    method: "DELETE",
  });
  if (!response.ok) {
    throw new Error(`삭제 실패: ${response.status}`);
  }
  return true;
}

// 범용 API 호출 래퍼 (React Native 프로젝트에서 자주 만드는 패턴)
const API_BASE = "https://api.example.com";

async function apiCall(endpoint, options = {}) {
  const {
    method = "GET",
    body = null,
    headers = {},
    timeout = 10000, // 10초 타임아웃
  } = options;

  // 타임아웃 처리
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeout);

  try {
    const response = await fetch(`${API_BASE}${endpoint}`, {
      method,
      headers: {
        "Content-Type": "application/json",
        ...headers, // 추가 헤더 병합
      },
      body: body ? JSON.stringify(body) : null,
      signal: controller.signal,
    });

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      throw new Error(errorData.message || `HTTP ${response.status}`);
    }

    // 204 No Content인 경우 body가 없음
    if (response.status === 204) return null;

    return await response.json();
  } catch (error) {
    if (error.name === "AbortError") {
      throw new Error("요청 시간이 초과되었습니다");
    }
    throw error;
  } finally {
    clearTimeout(timeoutId);
  }
}

// 사용 예시
async function main() {
  // GET
  const users = await apiCall("/users");

  // POST
  const newUser = await apiCall("/users", {
    method: "POST",
    body: { name: "홍길동", email: "hong@example.com" }
  });

  // DELETE
  await apiCall(`/users/${newUser.id}`, { method: "DELETE" });
}
```

---

## 7. 실전 패턴: React Native에서 자주 쓰는 비동기 패턴

### 7-1. 로딩/에러/데이터 상태 관리

```javascript
// React Native에서의 비동기 데이터 로딩 패턴 (미리보기)
// 이 패턴은 Phase 03에서 자세히 다루지만, 비동기 관점에서 이해해두세요

async function loadData() {
  // 상태: { loading: true, error: null, data: null }
  try {
    const response = await fetch("https://api.example.com/data");
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    // 상태: { loading: false, error: null, data: data }
    return data;
  } catch (error) {
    // 상태: { loading: false, error: error.message, data: null }
    throw error;
  }
}
```

### 7-2. 디바운스 (Debounce) — 검색 입력 최적화

```javascript
// JavaScript — 디바운스 함수 구현
function debounce(func, delay) {
  let timeoutId;
  return function(...args) {
    // 이전 타이머가 있으면 취소
    clearTimeout(timeoutId);
    // 새 타이머 설정 — delay ms 후에 함수 실행
    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// 사용 — 사용자가 타이핑을 멈춘 후 300ms 후에 검색
const searchAPI = debounce(async (query) => {
  const response = await fetch(`/api/search?q=${query}`);
  const results = await response.json();
  console.log("검색 결과:", results);
}, 300);

// 사용자가 "hello"를 빠르게 입력하면:
searchAPI("h");       // 타이머 시작
searchAPI("he");      // 이전 타이머 취소, 새 타이머
searchAPI("hel");     // 이전 타이머 취소, 새 타이머
searchAPI("hell");    // 이전 타이머 취소, 새 타이머
searchAPI("hello");   // 이전 타이머 취소, 새 타이머
// 300ms 후 "hello"로 한 번만 API 호출!
```

```javascript [playground]
// 🧪 디바운스 실습

function debounce(func, delay) {
  let timeoutId;
  return function(...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => func(...args), delay);
  };
}

// 검색 시뮬레이션
let callCount = 0;
const search = debounce((query) => {
  callCount++;
  console.log(`API 호출 #${callCount}: "${query}" 검색`);
}, 300);

// 빠르게 입력하면 마지막 입력만 실행됨
search("h");
search("he");
search("hel");
search("hell");
search("hello");

// 300ms 후 "hello"로 한 번만 호출됨!
setTimeout(() => {
  console.log(`총 API 호출 횟수: ${callCount} (5번 입력 중 1번만 호출)`);
}, 500);
```

### 7-3. 재시도 (Retry) 패턴

```javascript
// JavaScript — 실패 시 자동 재시도
async function fetchWithRetry(url, options = {}, maxRetries = 3) {
  let lastError;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await fetch(url, options);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return await response.json();
    } catch (error) {
      lastError = error;
      console.warn(`시도 ${attempt}/${maxRetries} 실패: ${error.message}`);

      if (attempt < maxRetries) {
        // 지수 백오프(exponential backoff): 1초, 2초, 4초...
        const delay = Math.pow(2, attempt - 1) * 1000;
        console.log(`${delay}ms 후 재시도...`);
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
  }

  throw new Error(`${maxRetries}번 시도 후 실패: ${lastError.message}`);
}

// 사용
try {
  const data = await fetchWithRetry("https://api.example.com/data");
  console.log("성공:", data);
} catch (error) {
  console.error("최종 실패:", error.message);
}
```

---

## 핵심 요약: JavaScript 비동기 vs Kotlin Coroutine

| 개념 | JavaScript | Kotlin Coroutine |
|------|-----------|-----------------|
| 비동기 함수 선언 | `async function` | `suspend fun` |
| 결과 대기 | `await promise` | suspend 호출 자체 (await 불필요) |
| 결과 객체 | `Promise<T>` | `Deferred<T>` |
| 병렬 실행 | `Promise.all([p1, p2])` | `awaitAll()` / `async { }` |
| 화재 후 망각 | `promise.then()` / 그냥 호출 | `launch { }` |
| 데이터 스트림 | `AsyncGenerator` / Observable | `Flow<T>` |
| 에러 처리 | `try/catch` 또는 `.catch()` | `try/catch` |
| 취소 | `AbortController` | `Job.cancel()` / structured |
| 컨텍스트 전환 | N/A (싱글 스레드) | `withContext(Dispatchers.IO)` |
| 타임아웃 | `Promise.race` + `setTimeout` | `withTimeout(ms)` |
| 스코프 | N/A | `coroutineScope`, `viewModelScope` |

---

## 🎮 인터랙티브 연습

```javascript [playground]
// async/await 연습
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function fetchUserData() {
  console.log('데이터 요청 시작...');
  await delay(100);
  console.log('사용자 데이터 수신 완료');
  return { id: 1, name: 'Kim', role: 'Android Developer' };
}

async function main() {
  try {
    const user = await fetchUserData();
    console.log('사용자:', JSON.stringify(user));
  } catch (error) {
    console.error('에러 발생:', error.message);
  }
}

main();
```

## ✅ 학습 확인 퀴즈

```quiz
type: mcq
question: "Kotlin의 coroutine과 가장 유사한 JavaScript 비동기 패턴은?"
options:
  - "callback"
  - "Promise.then()"
  - "async/await"
  - "setTimeout"
answer: "async/await"
explanation: "async/await는 Kotlin의 coroutine처럼 비동기 코드를 동기적으로 읽히도록 작성할 수 있게 해줍니다."
```

---

> **다음 문서**: [04-typescript-essentials.md](./04-typescript-essentials.md) — TypeScript 핵심: Kotlin 개발자가 빠르게 익히는 타입 시스템
