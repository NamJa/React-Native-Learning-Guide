# 단위 테스트 & 컴포넌트 테스트 — JUnit/Espresso에서 Jest/RNTL로

## 목차
1. [테스트 철학: Testing Trophy](#1-테스트-철학-testing-trophy)
2. [Jest 설정과 구성](#2-jest-설정과-구성)
3. [Jest 기초](#3-jest-기초)
4. [Matchers (단언문)](#4-matchers-단언문)
5. [Setup/Teardown](#5-setupteardown)
6. [Mocking (모킹)](#6-mocking-모킹)
7. [비동기 테스트](#7-비동기-테스트)
8. [타이머 모킹](#8-타이머-모킹)
9. [스냅샷 테스트](#9-스냅샷-테스트)
10. [React Native Testing Library (RNTL)](#10-react-native-testing-library-rntl)
11. [실전 테스트 예제](#11-실전-테스트-예제)
12. [코드 커버리지](#12-코드-커버리지)

---

## 1. 테스트 철학: Testing Trophy

Android에서는 Google이 제안하는 **Test Pyramid** (단위 70% / 통합 20% / E2E 10%)를 따르지만, React Native 생태계에서는 Kent C. Dodds가 제안한 **Testing Trophy**를 더 많이 따른다.

```
Testing Trophy (React Native)    vs    Test Pyramid (Android)

     ╱ E2E ╲                              /\
    ╱ (적음) ╲                            /E2E\
   ╱──────────╲                          /____\
  ╱ Integration ╲    ← 가장 많이!       / Integ \
 ╱ (가장 많이!)   ╲                    /________\
╱──────────────────╲                  /   Unit    \
      Unit                           /____________\
   (필요한 만큼)                       (가장 많이)
    Static
   (TypeScript)

[핵심 차이]
- Android: 단위 테스트를 가장 많이 작성
- React Native: 통합 테스트(컴포넌트 테스트)를 가장 많이 작성
- 이유: 컴포넌트가 렌더링 + 로직 + 상호작용을 모두 포함하므로,
        컴포넌트 단위의 통합 테스트가 가장 효율적
```

---

## 2. Jest 설정과 구성

React Native 프로젝트를 생성하면 Jest가 기본으로 포함되어 있다.

### jest.config.js 옵션

```javascript
// jest.config.js
/** @type {import('jest').Config} */
module.exports = {
  // React Native 프리셋 (필수)
  preset: 'react-native',

  // 테스트 파일 패턴
  testMatch: [
    '**/__tests__/**/*.{ts,tsx}',      // __tests__/ 디렉토리 내 모든 파일
    '**/*.{test,spec}.{ts,tsx}',       // *.test.ts, *.spec.tsx 등
  ],

  // TypeScript 변환 설정
  transform: {
    '^.+\\.(ts|tsx)$': 'babel-jest',
  },

  // 모듈 경로 별칭 (tsconfig paths와 동기화)
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/shared/components/$1',
    '^@hooks/(.*)$': '<rootDir>/src/shared/hooks/$1',
    // 이미지/에셋 파일 모킹
    '\\.(jpg|jpeg|png|gif|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },

  // 테스트 환경 설정 파일
  setupFiles: [
    './jest.setup.js',
  ],

  // 테스트 환경 설정 (RNTL afterEach 자동 정리 등)
  setupFilesAfterFramework: [
    '@testing-library/react-native/cleanup-after-each',
  ],

  // 무시할 경로
  transformIgnorePatterns: [
    'node_modules/(?!(react-native|@react-native|@react-navigation|react-native-reanimated)/)',
  ],

  // 커버리지 설정
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/types.ts',
    '!src/**/__tests__/**',
  ],

  // 커버리지 임계값
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 80,
      statements: 80,
    },
  },
};
```

> **Android 개발자 비교**: `jest.config.js`는 `build.gradle`의 `testOptions`와
> `app/build.gradle`의 `android { testOptions { ... } }` 블록에 해당한다.

### 테스트 파일 명명 규칙과 배치

```
src/
├── features/
│   └── auth/
│       ├── components/
│       │   ├── LoginForm.tsx
│       │   └── __tests__/
│       │       └── LoginForm.test.tsx      ← 방법 1: __tests__ 디렉토리
│       ├── hooks/
│       │   ├── useAuth.ts
│       │   └── useAuth.test.ts             ← 방법 2: 같은 디렉토리에 .test
│       └── utils/
│           ├── validation.ts
│           └── validation.spec.ts          ← 방법 3: .spec 확장자
```

### 테스트 실행 명령어

```bash
# 모든 테스트 실행 (JUnit의 "Run All Tests"에 해당)
npx jest

# 워치 모드 (파일 변경 시 자동 재실행)
npx jest --watch

# 특정 파일만 실행
npx jest LoginForm.test.tsx

# 특정 테스트 이름으로 필터링 (-t 플래그)
npx jest -t "로그인 버튼을 클릭하면"

# 커버리지 리포트 생성
npx jest --coverage

# 변경된 파일과 관련된 테스트만 실행
npx jest --changedSince=main

# package.json에 스크립트 추가
// package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --reporters=default --reporters=jest-junit"
  }
}
```

---

## 3. Jest 기초

### describe, it/test, expect

```typescript
// JUnit 비교
// JUnit:
// @RunWith(JUnit4::class)
// class CalculatorTest {
//     @Test
//     fun `addition should return correct sum`() {
//         assertEquals(4, Calculator.add(2, 2))
//     }
// }

// Jest:
describe('Calculator', () => {
  // describe = JUnit의 테스트 클래스
  // 중첩 가능하다

  describe('add', () => {
    it('두 양수를 더하면 정확한 합을 반환한다', () => {
      // it (또는 test) = JUnit의 @Test 메서드
      expect(add(2, 2)).toBe(4);
      // expect().toBe() = JUnit의 assertEquals()
    });

    it('음수를 더해도 정확한 결과를 반환한다', () => {
      expect(add(-1, 1)).toBe(0);
    });

    test('0을 더하면 원래 값을 반환한다', () => {
      // test와 it은 완전히 동일하다
      expect(add(5, 0)).toBe(5);
    });
  });

  describe('divide', () => {
    it('0으로 나누면 에러를 던진다', () => {
      expect(() => divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

### 테스트 건너뛰기 및 포커스

```typescript
// 특정 테스트만 실행 (JUnit의 @Ignore 반대)
describe.only('이 그룹만 실행', () => {
  it.only('이 테스트만 실행', () => {
    // ...
  });
});

// 특정 테스트 건너뛰기 (JUnit의 @Ignore에 해당)
describe.skip('이 그룹 건너뛰기', () => {
  it.skip('이 테스트 건너뛰기', () => {
    // ...
  });
});

// TODO로 표시 (나중에 작성할 테스트)
it.todo('삭제 기능을 테스트해야 한다');
```

---

## 4. Matchers (단언문)

```typescript
// === 동등성 비교 ===

// toBe: 원시값 비교 (===) — JUnit의 assertEquals
expect(2 + 2).toBe(4);
expect('hello').toBe('hello');

// toEqual: 객체/배열 깊은 비교 — JUnit의 assertEquals (객체)
expect({ name: '홍길동', age: 25 }).toEqual({ name: '홍길동', age: 25 });
expect([1, 2, 3]).toEqual([1, 2, 3]);

// toStrictEqual: toEqual + undefined 프로퍼티도 확인
expect({ a: 1 }).not.toStrictEqual({ a: 1, b: undefined });

// === 참/거짓 비교 ===

// toBeTruthy: truthy 값인지 (JUnit의 assertTrue)
expect('non-empty').toBeTruthy();
expect(1).toBeTruthy();

// toBeFalsy: falsy 값인지 (JUnit의 assertFalse)
expect('').toBeFalsy();
expect(0).toBeFalsy();
expect(null).toBeFalsy();

// toBeNull: null인지
expect(null).toBeNull();

// toBeUndefined / toBeDefined
expect(undefined).toBeUndefined();
expect('value').toBeDefined();

// === 숫자 비교 ===

expect(10).toBeGreaterThan(5);
expect(10).toBeGreaterThanOrEqual(10);
expect(5).toBeLessThan(10);
expect(5).toBeLessThanOrEqual(5);

// 부동소수점 비교 (JUnit의 assertEquals(expected, actual, delta))
expect(0.1 + 0.2).toBeCloseTo(0.3, 5); // 5자리까지 비교

// === 문자열 비교 ===

expect('Hello World').toContain('World');
expect('Hello World').toMatch(/hello/i); // 정규 표현식

// === 배열/이터러블 비교 ===

expect([1, 2, 3]).toContain(2);
expect([{ id: 1 }, { id: 2 }]).toContainEqual({ id: 1 });
expect([1, 2, 3]).toHaveLength(3);

// === 객체 비교 ===

expect({ name: '홍길동', age: 25, city: '서울' }).toMatchObject({
  name: '홍길동',
  age: 25,
  // city는 확인하지 않음 — 부분 매칭
});

expect({ name: '홍길동' }).toHaveProperty('name');
expect({ user: { name: '홍길동' } }).toHaveProperty('user.name', '홍길동');

// === 예외 비교 ===

expect(() => { throw new Error('에러!'); }).toThrow();
expect(() => { throw new Error('에러!'); }).toThrow('에러!');
expect(() => { throw new Error('에러!'); }).toThrow(/에러/);

// === 함수 호출 비교 ===

const mockFn = jest.fn();
mockFn('hello');
mockFn('world');

expect(mockFn).toHaveBeenCalled();           // 호출되었는지
expect(mockFn).toHaveBeenCalledTimes(2);     // 2번 호출되었는지
expect(mockFn).toHaveBeenCalledWith('hello'); // 'hello'로 호출되었는지
expect(mockFn).toHaveBeenLastCalledWith('world'); // 마지막 호출 인자

// === 부정 ===

expect(1).not.toBe(2);
expect([1, 2]).not.toContain(3);
```

---

## 5. Setup/Teardown

```typescript
// JUnit 비교:
// @Before → beforeEach
// @After → afterEach
// @BeforeClass → beforeAll
// @AfterClass → afterAll

describe('UserService', () => {
  let db: Database;
  let service: UserService;

  // 모든 테스트 시작 전 1번 실행 (JUnit의 @BeforeClass)
  beforeAll(() => {
    db = new Database(':memory:');
    console.log('DB 연결');
  });

  // 모든 테스트 종료 후 1번 실행 (JUnit의 @AfterClass)
  afterAll(() => {
    db.close();
    console.log('DB 연결 종료');
  });

  // 각 테스트 시작 전 실행 (JUnit의 @Before)
  beforeEach(() => {
    service = new UserService(db);
    db.clear(); // 테스트 간 데이터 격리
    console.log('테스트 시작: 데이터 초기화');
  });

  // 각 테스트 종료 후 실행 (JUnit의 @After)
  afterEach(() => {
    jest.restoreAllMocks(); // 모든 모킹 복원
    console.log('테스트 종료: 모킹 복원');
  });

  it('사용자를 생성할 수 있다', () => {
    const user = service.create({ name: '홍길동' });
    expect(user.id).toBeDefined();
  });

  it('사용자를 조회할 수 있다', () => {
    service.create({ name: '홍길동' });
    const users = service.getAll();
    expect(users).toHaveLength(1);
  });
});
```

실행 순서:
```
beforeAll
  beforeEach → 테스트 1 → afterEach
  beforeEach → 테스트 2 → afterEach
afterAll
```

---

## 6. Mocking (모킹)

Android에서 Mockito를 사용하는 것처럼, Jest에는 내장 모킹 시스템이 있다.

### jest.fn() — Mockito.mock()에 해당

```typescript
// Mockito:
// val callback = mock<(String) -> Unit>()
// verify(callback).invoke("hello")

// Jest:
const callback = jest.fn();

// 함수 호출
callback('hello');
callback('world');

// 검증
expect(callback).toHaveBeenCalledTimes(2);
expect(callback).toHaveBeenCalledWith('hello');

// 반환값 설정 (Mockito의 `when...thenReturn`)
const mockGetUser = jest.fn()
  .mockReturnValue({ id: 1, name: '홍길동' });          // 항상 같은 값
  // .mockReturnValueOnce({ id: 1, name: '홍길동' })    // 첫 번째 호출만
  // .mockImplementation((id) => ({ id, name: '유저' })) // 커스텀 구현

expect(mockGetUser()).toEqual({ id: 1, name: '홍길동' });
```

### jest.mock() — 모듈 전체 모킹

```typescript
// 모듈 전체를 모킹 (Mockito의 @Mock + @InjectMocks 패턴과 유사)

// api.ts
export const fetchUsers = async (): Promise<User[]> => {
  const response = await fetch('/api/users');
  return response.json();
};

// __tests__/UserList.test.tsx
import { fetchUsers } from '../api';

// 모듈 전체를 모킹
jest.mock('../api', () => ({
  fetchUsers: jest.fn(),
}));

// 타입 안전한 모킹
const mockFetchUsers = fetchUsers as jest.MockedFunction<typeof fetchUsers>;

describe('UserList', () => {
  beforeEach(() => {
    mockFetchUsers.mockResolvedValue([
      { id: 1, name: '홍길동' },
      { id: 2, name: '김철수' },
    ]);
  });

  it('사용자 목록을 표시한다', async () => {
    // ... 테스트 코드
    expect(mockFetchUsers).toHaveBeenCalledTimes(1);
  });
});
```

### jest.spyOn() — Mockito.spy()에 해당

```typescript
// 실제 구현을 유지하면서 호출을 감시
// Mockito:
// val spy = spy(realObject)
// verify(spy).someMethod()

// Jest:
import * as mathUtils from '../utils/math';

describe('Calculator', () => {
  it('add 함수 호출을 감시한다', () => {
    // 실제 구현을 호출하면서 감시
    const spy = jest.spyOn(mathUtils, 'add');

    const result = mathUtils.add(2, 3);

    expect(result).toBe(5);           // 실제 결과
    expect(spy).toHaveBeenCalledWith(2, 3); // 호출 검증

    spy.mockRestore(); // 원래대로 복원
  });

  it('add 함수를 모킹할 수도 있다', () => {
    const spy = jest.spyOn(mathUtils, 'add').mockReturnValue(999);

    const result = mathUtils.add(2, 3);

    expect(result).toBe(999);  // 모킹된 값
    spy.mockRestore();
  });
});
```

### React Native 모듈 모킹

```typescript
// react-native-mmkv 모킹
jest.mock('react-native-mmkv', () => ({
  MMKV: jest.fn().mockImplementation(() => ({
    getString: jest.fn(),
    set: jest.fn(),
    delete: jest.fn(),
    contains: jest.fn(),
    getAllKeys: jest.fn().mockReturnValue([]),
  })),
}));

// @react-navigation/native 모킹
const mockNavigate = jest.fn();
jest.mock('@react-navigation/native', () => ({
  ...jest.requireActual('@react-navigation/native'),
  useNavigation: () => ({
    navigate: mockNavigate,
    goBack: jest.fn(),
  }),
  useRoute: () => ({
    params: { userId: '123' },
  }),
}));

// fetch 모킹
global.fetch = jest.fn().mockResolvedValue({
  ok: true,
  json: () => Promise.resolve({ data: 'test' }),
  status: 200,
});
```

---

## 7. 비동기 테스트

```typescript
// Kotlin의 runTest { } 또는 runBlocking { }에 해당

// 방법 1: async/await (가장 권장)
it('사용자 데이터를 가져온다', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('홍길동');
});

// 방법 2: resolves/rejects
it('사용자 데이터를 가져온다', async () => {
  await expect(fetchUser(1)).resolves.toEqual({ id: 1, name: '홍길동' });
});

it('존재하지 않는 사용자는 에러를 던진다', async () => {
  await expect(fetchUser(999)).rejects.toThrow('User not found');
});

// 방법 3: done 콜백 (레거시, 권장하지 않음)
it('콜백을 테스트한다', (done) => {
  fetchUserWithCallback(1, (user) => {
    expect(user.name).toBe('홍길동');
    done(); // 이것을 호출해야 테스트가 완료됨
  });
});
```

---

## 8. 타이머 모킹

```typescript
// setTimeout, setInterval 등을 제어
// Android의 TestCoroutineDispatcher와 유사한 개념

describe('Debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers(); // 가짜 타이머 활성화
  });

  afterEach(() => {
    jest.useRealTimers(); // 진짜 타이머로 복원
  });

  it('300ms 후에 콜백을 실행한다', () => {
    const callback = jest.fn();
    debounce(callback, 300)();

    // 아직 실행되지 않음
    expect(callback).not.toHaveBeenCalled();

    // 300ms 전진
    jest.advanceTimersByTime(300);

    // 이제 실행됨
    expect(callback).toHaveBeenCalledTimes(1);
  });

  it('모든 타이머를 즉시 실행한다', () => {
    const callback = jest.fn();

    setTimeout(callback, 1000);
    setTimeout(callback, 2000);
    setTimeout(callback, 3000);

    // 모든 타이머를 즉시 실행
    jest.runAllTimers();

    expect(callback).toHaveBeenCalledTimes(3);
  });
});
```

---

## 9. 스냅샷 테스트

컴포넌트의 렌더링 결과를 "스냅샷"으로 저장하고, 이후 변경 여부를 확인한다.

```typescript
import { render } from '@testing-library/react-native';
import { Button } from '../Button';

// 기본 스냅샷 테스트
it('Button이 올바르게 렌더링된다', () => {
  const tree = render(<Button title="클릭" onPress={() => {}} />);

  // 첫 실행: __snapshots__/Button.test.tsx.snap 파일 생성
  // 이후 실행: 스냅샷과 비교하여 변경 감지
  expect(tree.toJSON()).toMatchSnapshot();
});

// 인라인 스냅샷 (별도 파일 없이 테스트 파일 내에 저장)
it('Button의 텍스트가 올바르다', () => {
  const { getByText } = render(<Button title="확인" onPress={() => {}} />);

  expect(getByText('확인').props).toMatchInlineSnapshot(`
    {
      "children": "확인",
      "style": {
        "color": "#FFFFFF",
        "fontSize": 16,
      },
    }
  `);
});

// 스냅샷 업데이트 (UI를 의도적으로 변경한 경우)
// npx jest --updateSnapshot  또는  npx jest -u
```

> **주의**: 스냅샷 테스트는 보조적으로만 사용하라. 스냅샷이 너무 크면 변경 사항을 리뷰하기 어렵다.
> 핵심 동작은 반드시 명시적 assertion으로 테스트해야 한다.

---

## 10. React Native Testing Library (RNTL)

RNTL은 React Native 컴포넌트의 통합 테스트를 위한 라이브러리다. Android의 **Espresso**에 해당한다.

### 설치

```bash
npm install --save-dev @testing-library/react-native @testing-library/jest-native
```

### jest.setup.js 설정

```javascript
// jest.setup.js
import '@testing-library/react-native/extend-expect';
// 이제 toBeOnTheScreen(), toHaveTextContent() 등의 매처를 사용할 수 있다
```

### 기본 사용법: render

```typescript
import { render, screen } from '@testing-library/react-native';
import { Greeting } from '../Greeting';

it('이름을 표시한다', () => {
  // Espresso: ActivityScenarioRule로 액티비티 실행
  // RNTL: render로 컴포넌트 렌더링
  render(<Greeting name="홍길동" />);

  // 화면에 텍스트가 표시되는지 확인
  expect(screen.getByText('안녕하세요, 홍길동님!')).toBeOnTheScreen();
});
```

### 쿼리 메서드 (요소 찾기)

```typescript
// Espresso: onView(withId(R.id.username))
// RNTL: 다양한 쿼리 메서드 제공

// === getBy* — 요소가 반드시 있어야 함 (없으면 에러) ===

// 텍스트로 찾기
screen.getByText('로그인');
screen.getByText(/로그인/); // 정규 표현식도 가능

// testID로 찾기 (Android의 viewId에 해당)
// <View testID="login-button" />
screen.getByTestId('login-button');

// placeholder로 찾기
// <TextInput placeholder="이메일 입력" />
screen.getByPlaceholderText('이메일 입력');

// 접근성 라벨로 찾기
// <TouchableOpacity accessibilityLabel="로그인 버튼" />
screen.getByLabelText('로그인 버튼');

// 접근성 역할로 찾기
screen.getByRole('button', { name: '로그인' });

// displayValue로 찾기 (TextInput의 현재 값)
screen.getByDisplayValue('user@email.com');

// === queryBy* — 요소가 없으면 null 반환 (에러 아님) ===
// 요소가 화면에 없는지 확인할 때 사용

const errorMessage = screen.queryByText('에러 발생');
expect(errorMessage).toBeNull(); // 에러 메시지가 없음을 확인

// === findBy* — 비동기, 요소가 나타날 때까지 기다림 ===
// await 필수! (Espresso의 IdlingResource와 유사)

it('로딩 후 데이터가 표시된다', async () => {
  render(<UserProfile userId="1" />);

  // 비동기 데이터 로딩 후 요소가 나타날 때까지 기다림
  const userName = await screen.findByText('홍길동');
  expect(userName).toBeOnTheScreen();
});

// === *AllBy* — 여러 요소 찾기 ===
const listItems = screen.getAllByTestId(/list-item/);
expect(listItems).toHaveLength(5);
```

### 쿼리 선택 가이드

```
우선순위 (사용자 관점에서 가장 자연스러운 것부터):

1. getByRole        ← 접근성 역할 (button, heading 등)
2. getByText        ← 화면에 보이는 텍스트
3. getByLabelText   ← 접근성 라벨
4. getByPlaceholderText ← 입력 필드의 placeholder
5. getByDisplayValue    ← 입력 필드의 현재 값
6. getByTestId      ← 최후의 수단 (testID 속성)

[원칙]: 사용자가 보는 것, 사용자가 하는 것으로 테스트
         → 구현 세부사항이 아닌 동작을 테스트
```

### 사용자 이벤트 (fireEvent)

```typescript
import { render, screen, fireEvent } from '@testing-library/react-native';

// Espresso: onView(...).perform(click())
// RNTL: fireEvent.press(element)

it('버튼 클릭 시 카운터가 증가한다', () => {
  render(<Counter />);

  const button = screen.getByText('증가');
  const count = screen.getByTestId('count-display');

  expect(count).toHaveTextContent('0');

  // 버튼 클릭
  fireEvent.press(button);
  expect(count).toHaveTextContent('1');

  // 여러 번 클릭
  fireEvent.press(button);
  fireEvent.press(button);
  expect(count).toHaveTextContent('3');
});

// 텍스트 입력
// Espresso: onView(...).perform(typeText("hello"))
it('텍스트를 입력할 수 있다', () => {
  render(<SearchBar />);

  const input = screen.getByPlaceholderText('검색어를 입력하세요');

  fireEvent.changeText(input, '리액트 네이티브');
  expect(input).toHaveDisplayValue('리액트 네이티브');
});

// 스크롤
it('스크롤이 동작한다', () => {
  render(<InfiniteList />);

  const list = screen.getByTestId('item-list');
  fireEvent.scroll(list, {
    nativeEvent: {
      contentOffset: { y: 500 },
      contentSize: { height: 1000, width: 400 },
      layoutMeasurement: { height: 800, width: 400 },
    },
  });
});

// Switch 토글
it('스위치를 토글할 수 있다', () => {
  render(<SettingsScreen />);

  const toggle = screen.getByTestId('dark-mode-switch');
  fireEvent(toggle, 'valueChange', true);
});
```

### waitFor — 비동기 assertion

```typescript
import { render, screen, waitFor, fireEvent } from '@testing-library/react-native';

// Espresso의 IdlingResource에 해당
it('폼 제출 후 성공 메시지가 표시된다', async () => {
  render(<ContactForm />);

  fireEvent.changeText(screen.getByPlaceholderText('이름'), '홍길동');
  fireEvent.changeText(screen.getByPlaceholderText('이메일'), 'test@test.com');
  fireEvent.press(screen.getByText('제출'));

  // 비동기 작업 완료 후 UI가 업데이트될 때까지 기다림
  await waitFor(() => {
    expect(screen.getByText('제출 완료!')).toBeOnTheScreen();
  });

  // 또는 findByText 사용 (내부적으로 waitFor 사용)
  const successMessage = await screen.findByText('제출 완료!');
  expect(successMessage).toBeOnTheScreen();
});
```

---

## 11. 실전 테스트 예제

### 예제 1: 간단한 컴포넌트 테스트

```typescript
// src/features/home/components/UserCard.tsx
import React from 'react';
import { View, Text, Image, StyleSheet } from 'react-native';

interface UserCardProps {
  name: string;
  email: string;
  avatarUrl: string;
  isOnline: boolean;
}

export function UserCard({ name, email, avatarUrl, isOnline }: UserCardProps) {
  return (
    <View style={styles.container} testID="user-card">
      <Image source={{ uri: avatarUrl }} style={styles.avatar} testID="user-avatar" />
      <View>
        <Text style={styles.name}>{name}</Text>
        <Text style={styles.email}>{email}</Text>
      </View>
      {isOnline && (
        <View style={styles.onlineBadge} testID="online-badge">
          <Text style={styles.onlineText}>온라인</Text>
        </View>
      )}
    </View>
  );
}
```

```typescript
// src/features/home/components/__tests__/UserCard.test.tsx
import { render, screen } from '@testing-library/react-native';
import { UserCard } from '../UserCard';

describe('UserCard', () => {
  const defaultProps = {
    name: '홍길동',
    email: 'hong@example.com',
    avatarUrl: 'https://example.com/avatar.jpg',
    isOnline: false,
  };

  it('사용자 이름과 이메일을 표시한다', () => {
    render(<UserCard {...defaultProps} />);

    expect(screen.getByText('홍길동')).toBeOnTheScreen();
    expect(screen.getByText('hong@example.com')).toBeOnTheScreen();
  });

  it('아바타 이미지를 표시한다', () => {
    render(<UserCard {...defaultProps} />);

    const avatar = screen.getByTestId('user-avatar');
    expect(avatar.props.source.uri).toBe('https://example.com/avatar.jpg');
  });

  it('온라인일 때 온라인 배지를 표시한다', () => {
    render(<UserCard {...defaultProps} isOnline={true} />);

    expect(screen.getByTestId('online-badge')).toBeOnTheScreen();
    expect(screen.getByText('온라인')).toBeOnTheScreen();
  });

  it('오프라인일 때 온라인 배지를 표시하지 않는다', () => {
    render(<UserCard {...defaultProps} isOnline={false} />);

    expect(screen.queryByTestId('online-badge')).toBeNull();
  });
});
```

### 예제 2: 사용자 상호작용 테스트

```typescript
// src/features/auth/components/LoginForm.tsx
import React, { useState } from 'react';
import { View, Text, TextInput, TouchableOpacity, ActivityIndicator } from 'react-native';

interface LoginFormProps {
  onLogin: (email: string, password: string) => Promise<void>;
}

export function LoginForm({ onLogin }: LoginFormProps) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async () => {
    if (!email || !password) {
      setError('이메일과 비밀번호를 입력해주세요');
      return;
    }
    setError('');
    setLoading(true);
    try {
      await onLogin(email, password);
    } catch (e) {
      setError('로그인에 실패했습니다');
    } finally {
      setLoading(false);
    }
  };

  return (
    <View>
      <TextInput
        placeholder="이메일"
        value={email}
        onChangeText={setEmail}
        testID="email-input"
      />
      <TextInput
        placeholder="비밀번호"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        testID="password-input"
      />
      {error ? <Text testID="error-message">{error}</Text> : null}
      <TouchableOpacity onPress={handleSubmit} testID="login-button" disabled={loading}>
        {loading ? <ActivityIndicator /> : <Text>로그인</Text>}
      </TouchableOpacity>
    </View>
  );
}
```

```typescript
// src/features/auth/components/__tests__/LoginForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react-native';
import { LoginForm } from '../LoginForm';

describe('LoginForm', () => {
  const mockOnLogin = jest.fn();

  beforeEach(() => {
    mockOnLogin.mockReset();
  });

  it('이메일과 비밀번호 입력 필드를 표시한다', () => {
    render(<LoginForm onLogin={mockOnLogin} />);

    expect(screen.getByPlaceholderText('이메일')).toBeOnTheScreen();
    expect(screen.getByPlaceholderText('비밀번호')).toBeOnTheScreen();
    expect(screen.getByText('로그인')).toBeOnTheScreen();
  });

  it('빈 필드로 제출하면 에러 메시지를 표시한다', () => {
    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.press(screen.getByTestId('login-button'));

    expect(screen.getByTestId('error-message')).toHaveTextContent(
      '이메일과 비밀번호를 입력해주세요'
    );
    expect(mockOnLogin).not.toHaveBeenCalled();
  });

  it('유효한 입력으로 제출하면 onLogin을 호출한다', async () => {
    mockOnLogin.mockResolvedValue(undefined);
    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.changeText(screen.getByTestId('email-input'), 'test@test.com');
    fireEvent.changeText(screen.getByTestId('password-input'), 'password123');
    fireEvent.press(screen.getByTestId('login-button'));

    await waitFor(() => {
      expect(mockOnLogin).toHaveBeenCalledWith('test@test.com', 'password123');
    });
  });

  it('로그인 실패 시 에러 메시지를 표시한다', async () => {
    mockOnLogin.mockRejectedValue(new Error('Invalid credentials'));
    render(<LoginForm onLogin={mockOnLogin} />);

    fireEvent.changeText(screen.getByTestId('email-input'), 'test@test.com');
    fireEvent.changeText(screen.getByTestId('password-input'), 'wrong');
    fireEvent.press(screen.getByTestId('login-button'));

    await waitFor(() => {
      expect(screen.getByTestId('error-message')).toHaveTextContent(
        '로그인에 실패했습니다'
      );
    });
  });
});
```

### 예제 3: 네비게이션 테스트

```typescript
// src/features/home/screens/__tests__/HomeScreen.test.tsx
import { render, screen, fireEvent } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { HomeScreen } from '../HomeScreen';
import { DetailScreen } from '../../detail/screens/DetailScreen';

const Stack = createNativeStackNavigator();

function TestNavigator() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Detail" component={DetailScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

describe('HomeScreen 네비게이션', () => {
  it('항목을 탭하면 상세 화면으로 이동한다', async () => {
    render(<TestNavigator />);

    // 항목 클릭
    fireEvent.press(screen.getByText('항목 1'));

    // 상세 화면이 표시되는지 확인
    const detailTitle = await screen.findByText('항목 1 상세');
    expect(detailTitle).toBeOnTheScreen();
  });
});
```

### 예제 4: API 호출 모킹

```typescript
// src/features/users/hooks/__tests__/useUsers.test.tsx
import { renderHook, waitFor } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useUsers } from '../useUsers';

// fetch를 모킹
global.fetch = jest.fn();

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });

  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

describe('useUsers', () => {
  beforeEach(() => {
    (global.fetch as jest.Mock).mockReset();
  });

  it('사용자 목록을 가져온다', async () => {
    const mockUsers = [
      { id: 1, name: '홍길동' },
      { id: 2, name: '김철수' },
    ];

    (global.fetch as jest.Mock).mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(mockUsers),
    });

    const { result } = renderHook(() => useUsers(), {
      wrapper: createWrapper(),
    });

    // 초기 로딩 상태
    expect(result.current.isLoading).toBe(true);

    // 데이터 로드 완료 대기
    await waitFor(() => {
      expect(result.current.isLoading).toBe(false);
    });

    expect(result.current.data).toEqual(mockUsers);
    expect(global.fetch).toHaveBeenCalledWith('/api/users');
  });

  it('에러 시 에러 상태를 반환한다', async () => {
    (global.fetch as jest.Mock).mockRejectedValueOnce(new Error('Network error'));

    const { result } = renderHook(() => useUsers(), {
      wrapper: createWrapper(),
    });

    await waitFor(() => {
      expect(result.current.isError).toBe(true);
    });

    expect(result.current.error?.message).toBe('Network error');
  });
});
```

### 예제 5: Zustand 스토어 테스트

```typescript
// src/features/cart/stores/cartStore.ts
import { create } from 'zustand';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  updateQuantity: (id: string, quantity: number) => void;
  clearCart: () => void;
  totalPrice: () => number;
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  addItem: (item) =>
    set((state) => {
      const existing = state.items.find((i) => i.id === item.id);
      if (existing) {
        return {
          items: state.items.map((i) =>
            i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        };
      }
      return { items: [...state.items, { ...item, quantity: 1 }] };
    }),
  removeItem: (id) =>
    set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
  updateQuantity: (id, quantity) =>
    set((state) => ({
      items: state.items.map((i) => (i.id === id ? { ...i, quantity } : i)),
    })),
  clearCart: () => set({ items: [] }),
  totalPrice: () =>
    get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),
}));
```

```typescript
// src/features/cart/stores/__tests__/cartStore.test.ts
import { useCartStore } from '../cartStore';

describe('CartStore', () => {
  beforeEach(() => {
    // 매 테스트 전 스토어 초기화
    useCartStore.setState({ items: [] });
  });

  it('아이템을 추가할 수 있다', () => {
    const { addItem } = useCartStore.getState();

    addItem({ id: '1', name: '노트북', price: 1000000 });

    const { items } = useCartStore.getState();
    expect(items).toHaveLength(1);
    expect(items[0]).toEqual({
      id: '1',
      name: '노트북',
      price: 1000000,
      quantity: 1,
    });
  });

  it('같은 아이템을 추가하면 수량이 증가한다', () => {
    const { addItem } = useCartStore.getState();

    addItem({ id: '1', name: '노트북', price: 1000000 });
    addItem({ id: '1', name: '노트북', price: 1000000 });

    const { items } = useCartStore.getState();
    expect(items).toHaveLength(1);
    expect(items[0].quantity).toBe(2);
  });

  it('아이템을 삭제할 수 있다', () => {
    const { addItem, removeItem } = useCartStore.getState();

    addItem({ id: '1', name: '노트북', price: 1000000 });
    addItem({ id: '2', name: '마우스', price: 50000 });

    removeItem('1');

    const { items } = useCartStore.getState();
    expect(items).toHaveLength(1);
    expect(items[0].name).toBe('마우스');
  });

  it('총 가격을 계산한다', () => {
    const { addItem, totalPrice } = useCartStore.getState();

    addItem({ id: '1', name: '노트북', price: 1000000 });
    addItem({ id: '2', name: '마우스', price: 50000 });
    addItem({ id: '1', name: '노트북', price: 1000000 }); // 수량 2

    expect(useCartStore.getState().totalPrice()).toBe(2050000);
  });

  it('장바구니를 비울 수 있다', () => {
    const { addItem, clearCart } = useCartStore.getState();

    addItem({ id: '1', name: '노트북', price: 1000000 });
    addItem({ id: '2', name: '마우스', price: 50000 });

    clearCart();

    expect(useCartStore.getState().items).toHaveLength(0);
  });
});
```

### 예제 6: TanStack Query 훅 테스트

```typescript
// src/features/posts/hooks/__tests__/usePosts.test.tsx
import { renderHook, waitFor, act } from '@testing-library/react-native';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { usePosts, useCreatePost } from '../usePosts';

const mockFetch = jest.fn();
global.fetch = mockFetch;

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false, gcTime: 0 },
      mutations: { retry: false },
    },
  });
}

function createWrapper() {
  const client = createTestQueryClient();
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={client}>{children}</QueryClientProvider>
  );
}

describe('usePosts', () => {
  beforeEach(() => {
    mockFetch.mockReset();
  });

  it('게시글 목록을 가져온다', async () => {
    const posts = [
      { id: 1, title: '첫 번째 글', body: '내용1' },
      { id: 2, title: '두 번째 글', body: '내용2' },
    ];

    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(posts),
    });

    const { result } = renderHook(() => usePosts(), { wrapper: createWrapper() });

    await waitFor(() => expect(result.current.isSuccess).toBe(true));

    expect(result.current.data).toEqual(posts);
  });
});

describe('useCreatePost', () => {
  beforeEach(() => {
    mockFetch.mockReset();
  });

  it('새 게시글을 생성한다', async () => {
    const newPost = { title: '새 글', body: '새 내용' };
    const createdPost = { id: 3, ...newPost };

    mockFetch.mockResolvedValueOnce({
      ok: true,
      json: () => Promise.resolve(createdPost),
    });

    const { result } = renderHook(() => useCreatePost(), {
      wrapper: createWrapper(),
    });

    await act(async () => {
      result.current.mutate(newPost);
    });

    await waitFor(() => expect(result.current.isSuccess).toBe(true));
    expect(result.current.data).toEqual(createdPost);
  });
});
```

---

## 12. 코드 커버리지

```bash
# 커버리지 리포트 생성
npx jest --coverage

# 출력 예시:
# ----------------------|---------|----------|---------|---------|
# File                  | % Stmts | % Branch | % Funcs | % Lines |
# ----------------------|---------|----------|---------|---------|
# All files             |   85.32 |    78.45 |   82.14 |   86.21 |
#  features/auth/       |   92.11 |    85.71 |   88.89 |   93.33 |
#   LoginForm.tsx       |     100 |      100 |     100 |     100 |
#   useAuth.ts          |   84.21 |    71.43 |   77.78 |   86.67 |
#  features/cart/       |   78.57 |    71.43 |   75.00 |   79.31 |
# ----------------------|---------|----------|---------|---------|

# HTML 리포트 확인 (브라우저에서 열기)
# coverage/lcov-report/index.html
open coverage/lcov-report/index.html
```

```
[커버리지 지표 설명]

Stmts (Statements): 코드의 각 문(statement)이 실행되었는지
Branch: 조건문(if/else, switch, 삼항연산자)의 각 분기가 실행되었는지
Funcs (Functions): 각 함수가 호출되었는지
Lines: 각 코드 줄이 실행되었는지

[Android 비교]
- JaCoCo의 Instruction Coverage ≈ Stmts
- JaCoCo의 Branch Coverage ≈ Branch
- JaCoCo의 Method Coverage ≈ Funcs
- JaCoCo의 Line Coverage ≈ Lines
```

### 커버리지 임계값 설정

```javascript
// jest.config.js
module.exports = {
  coverageThreshold: {
    // 전역 기준
    global: {
      branches: 70,
      functions: 70,
      lines: 80,
      statements: 80,
    },
    // 특정 경로별 기준
    './src/features/auth/': {
      branches: 90,
      functions: 90,
      lines: 95,
      statements: 95,
    },
  },
};
```

### CI에서 커버리지 활용

```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci
      - run: npx jest --coverage --ci

      # 커버리지 리포트 업로드
      - uses: codecov/codecov-action@v4
        with:
          file: ./coverage/lcov.info
```
