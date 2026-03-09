# 로컬 데이터 저장 — SharedPreferences/Room에서 MMKV/AsyncStorage로

## 목차
1. [AsyncStorage](#1-asyncstorage)
2. [react-native-mmkv](#2-react-native-mmkv)
3. [Zustand persist 미들웨어](#3-zustand-persist-미들웨어)
4. [SQLite](#4-sqlite)
5. [보안 저장소 (Secure Storage)](#5-보안-저장소)
6. [파일 시스템](#6-파일-시스템)
7. [어떤 것을 사용할까](#7-어떤-것을-사용할까)

---

## 1. AsyncStorage

AsyncStorage는 Android의 `SharedPreferences`에 가장 가까운 React Native 솔루션이다. 키-값(key-value) 형태로 문자열을 비동기적으로 저장하고 읽는다.

### 1-1. 설치

```bash
npm install @react-native-async-storage/async-storage
```

### 1-2. 기본 API

```tsx
import AsyncStorage from '@react-native-async-storage/async-storage';

// ===== 저장 =====
// Android: SharedPreferences.edit().putString("key", "value").apply()
await AsyncStorage.setItem('username', 'jongwoo');

// 객체 저장 (JSON 직렬화 필요)
// Android: SharedPreferences.edit().putString("user", gson.toJson(user)).apply()
const user = { id: 1, name: '종우', email: 'jw@test.com' };
await AsyncStorage.setItem('user', JSON.stringify(user));

// ===== 읽기 =====
// Android: SharedPreferences.getString("username", null)
const username = await AsyncStorage.getItem('username');
// username: string | null

// 객체 읽기
const userJson = await AsyncStorage.getItem('user');
const user = userJson ? JSON.parse(userJson) : null;

// ===== 삭제 =====
// Android: SharedPreferences.edit().remove("username").apply()
await AsyncStorage.removeItem('username');

// ===== 전체 삭제 =====
// Android: SharedPreferences.edit().clear().apply()
await AsyncStorage.clear();

// ===== 모든 키 조회 =====
const allKeys = await AsyncStorage.getAllKeys();
// ['username', 'user', 'settings', ...]
```

### 1-3. 배치 작업

```tsx
// 여러 값 한번에 저장
// Android: SharedPreferences.edit().putString(k1,v1).putString(k2,v2).apply()
await AsyncStorage.multiSet([
  ['theme', 'dark'],
  ['language', 'ko'],
  ['fontSize', '16'],
]);

// 여러 값 한번에 읽기
const values = await AsyncStorage.multiGet(['theme', 'language', 'fontSize']);
// values: [['theme', 'dark'], ['language', 'ko'], ['fontSize', '16']]

// Map으로 변환
const settingsMap = Object.fromEntries(values);
// { theme: 'dark', language: 'ko', fontSize: '16' }

// 여러 값 한번에 삭제
await AsyncStorage.multiRemove(['theme', 'language']);

// 기존 값에 병합 (JSON merge)
await AsyncStorage.mergeItem('user', JSON.stringify({ name: '새이름' }));
// 기존 user 객체에 name만 업데이트
```

### 1-4. 타입 안전한 래퍼

```tsx
// utils/storage.ts
// Android의 SharedPreferences 래퍼 클래스와 동일한 패턴

import AsyncStorage from '@react-native-async-storage/async-storage';

// 저장할 데이터의 타입 정의
interface StorageSchema {
  'auth.token': string;
  'auth.refreshToken': string;
  'settings.theme': 'light' | 'dark';
  'settings.language': 'ko' | 'en' | 'ja';
  'settings.fontSize': number;
  'user.profile': {
    id: string;
    name: string;
    email: string;
  };
  'onboarding.completed': boolean;
}

// 타입 안전한 get/set 함수
export async function getStorageItem<K extends keyof StorageSchema>(
  key: K
): Promise<StorageSchema[K] | null> {
  try {
    const value = await AsyncStorage.getItem(key);
    if (value === null) return null;
    return JSON.parse(value) as StorageSchema[K];
  } catch {
    return null;
  }
}

export async function setStorageItem<K extends keyof StorageSchema>(
  key: K,
  value: StorageSchema[K]
): Promise<void> {
  try {
    await AsyncStorage.setItem(key, JSON.stringify(value));
  } catch (error) {
    console.error('Storage set error:', error);
  }
}

export async function removeStorageItem<K extends keyof StorageSchema>(
  key: K
): Promise<void> {
  try {
    await AsyncStorage.removeItem(key);
  } catch (error) {
    console.error('Storage remove error:', error);
  }
}
```

사용:

```tsx
// 타입 자동 추론!
await setStorageItem('settings.theme', 'dark');     // OK
await setStorageItem('settings.theme', 'blue');     // TypeScript Error!

const theme = await getStorageItem('settings.theme');
// theme: 'light' | 'dark' | null — 타입 자동 추론
```

### 1-5. AsyncStorage의 한계

| 한계 | 설명 |
|------|------|
| 비동기 전용 | 모든 작업이 `await` 필요 (동기적 읽기 불가) |
| 문자열 전용 | 숫자, 불리언 등도 JSON.stringify 필요 |
| 성능 | 대용량 데이터에 느림 (SQLite 기반이지만 오버헤드) |
| 암호화 없음 | 데이터가 평문으로 저장됨 |
| 크기 제한 | Android에서 기본 6MB (변경 가능하지만 권장하지 않음) |

이런 한계 때문에 **MMKV가 대안으로 부상**했다.

---

## 2. react-native-mmkv

MMKV는 Tencent의 WeChat에서 개발한 초고속 키-값 저장소로, Android의 `SharedPreferences`보다 **30배 빠르다**. C++ JSI를 통해 네이티브 코드와 직접 통신하므로, **동기적(synchronous)** 읽기/쓰기가 가능하다.

### 2-1. 설치

```bash
npm install react-native-mmkv
```

Expo를 사용하는 경우:
```bash
npx expo install react-native-mmkv
```

### 2-2. 기본 설정

```tsx
// storage/mmkv.ts
import { MMKV } from 'react-native-mmkv';

// 기본 인스턴스 생성
// Android SharedPreferences의 getSharedPreferences("default", MODE_PRIVATE)와 유사
export const storage = new MMKV();

// 커스텀 인스턴스 (별도 파일에 저장)
export const userStorage = new MMKV({
  id: 'user-storage',       // 저장소 이름 (Android의 SharedPreferences 파일명)
  // encryptionKey: 'secret', // 선택: AES 256 암호화
});

// 암호화된 인스턴스
export const secureStorage = new MMKV({
  id: 'secure-storage',
  encryptionKey: 'my-encryption-key-256bit', // AES-256 키
});
```

### 2-3. CRUD 작업

```tsx
import { storage } from './storage/mmkv';

// ===== 저장 (동기!) =====
// Android SharedPreferences는 비동기(apply) 또는 동기(commit)
// MMKV는 항상 동기적이면서도 빠름

// 문자열
storage.set('username', 'jongwoo');

// 숫자 (JSON.stringify 불필요!)
storage.set('loginCount', 42);

// 불리언
storage.set('isLoggedIn', true);

// 객체 (JSON.stringify 필요)
const user = { id: 1, name: '종우' };
storage.set('user', JSON.stringify(user));

// ===== 읽기 (동기!) =====
const username = storage.getString('username');         // string | undefined
const loginCount = storage.getNumber('loginCount');     // number | undefined
const isLoggedIn = storage.getBoolean('isLoggedIn');   // boolean | undefined

// 객체 읽기
const userJson = storage.getString('user');
const user = userJson ? JSON.parse(userJson) : null;

// ===== 키 존재 확인 =====
const exists = storage.contains('username');            // boolean

// ===== 삭제 =====
storage.delete('username');

// ===== 전체 삭제 =====
storage.clearAll();

// ===== 모든 키 조회 =====
const allKeys = storage.getAllKeys();                   // string[]
```

### 2-4. AsyncStorage vs MMKV 비교

```tsx
// AsyncStorage: 비동기, await 필요
const value = await AsyncStorage.getItem('key');  // Promise<string | null>

// MMKV: 동기, 즉시 반환
const value = storage.getString('key');            // string | undefined
```

| 특성 | AsyncStorage | MMKV |
|------|-------------|------|
| 동기/비동기 | 비동기 only | 동기 (+ 비동기 불필요) |
| 타입 지원 | string only | string, number, boolean, Buffer |
| 성능 | 느림 | ~30x 빠름 (C++ JSI) |
| 암호화 | 없음 | AES-256 내장 |
| 번들 크기 | 작음 | 약간 큼 (네이티브 모듈) |
| Expo 지원 | O | O (expo-dev-client 필요) |

### 2-5. 타입 안전한 MMKV 래퍼

```tsx
// storage/typedStorage.ts
import { MMKV } from 'react-native-mmkv';

const mmkv = new MMKV();

// Android SharedPreferences처럼 타입별 get/set 메서드 제공
class TypedStorage {
  // 문자열
  getString(key: string): string | undefined {
    return mmkv.getString(key);
  }

  setString(key: string, value: string): void {
    mmkv.set(key, value);
  }

  // 숫자
  getNumber(key: string): number | undefined {
    return mmkv.getNumber(key);
  }

  setNumber(key: string, value: number): void {
    mmkv.set(key, value);
  }

  // 불리언
  getBoolean(key: string): boolean | undefined {
    return mmkv.getBoolean(key);
  }

  setBoolean(key: string, value: boolean): void {
    mmkv.set(key, value);
  }

  // 객체 (타입 안전)
  getObject<T>(key: string): T | undefined {
    const json = mmkv.getString(key);
    if (!json) return undefined;
    try {
      return JSON.parse(json) as T;
    } catch {
      return undefined;
    }
  }

  setObject<T>(key: string, value: T): void {
    mmkv.set(key, JSON.stringify(value));
  }

  // 삭제
  delete(key: string): void {
    mmkv.delete(key);
  }

  // 키 존재 확인
  contains(key: string): boolean {
    return mmkv.contains(key);
  }

  // 전체 삭제
  clearAll(): void {
    mmkv.clearAll();
  }
}

export const typedStorage = new TypedStorage();
```

사용:

```tsx
import { typedStorage } from './storage/typedStorage';

// 객체 저장/읽기가 깔끔
interface UserProfile {
  id: string;
  name: string;
  email: string;
}

typedStorage.setObject<UserProfile>('profile', {
  id: '1',
  name: '종우',
  email: 'jw@test.com',
});

const profile = typedStorage.getObject<UserProfile>('profile');
// profile: UserProfile | undefined — 타입 추론
```

### 2-6. MMKV 암호화 (EncryptedSharedPreferences 대응)

```tsx
// Android EncryptedSharedPreferences와 동일한 역할
import { MMKV } from 'react-native-mmkv';

// 암호화된 저장소 생성
const encryptedStorage = new MMKV({
  id: 'encrypted-storage',
  encryptionKey: 'your-256-bit-encryption-key',
});

// 사용법은 일반 MMKV와 동일
encryptedStorage.set('sensitive_token', 'eyJhbGciOiJIUzI1NiIs...');
const token = encryptedStorage.getString('sensitive_token');

// 암호화 키 변경 (리키잉)
encryptedStorage.recrypt('new-encryption-key');

// 암호화 제거
encryptedStorage.recrypt(undefined);
```

---

## 3. Zustand persist 미들웨어

Zustand의 persist 미들웨어를 사용하면 Store 상태가 자동으로 로컬 저장소에 영속화된다. Android에서 ViewModel의 상태를 DataStore에 자동 저장하는 것과 비슷하다.

### 3-1. AsyncStorage와 함께 사용

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

interface AppState {
  theme: 'light' | 'dark';
  language: string;
  onboardingDone: boolean;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (lang: string) => void;
  completeOnboarding: () => void;
}

const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'ko',
      onboardingDone: false,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      completeOnboarding: () => set({ onboardingDone: true }),
    }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => AsyncStorage),
    }
  )
);
```

### 3-2. MMKV와 함께 사용 (권장)

```tsx
import { create } from 'zustand';
import { persist, createJSONStorage, StateStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

// MMKV를 Zustand의 StateStorage 인터페이스에 맞춤
const mmkvStateStorage: StateStorage = {
  getItem: (name) => {
    const value = storage.getString(name);
    return value ?? null;
  },
  setItem: (name, value) => {
    storage.set(name, value);
  },
  removeItem: (name) => {
    storage.delete(name);
  },
};

const useAppStore = create<AppState>()(
  persist(
    (set) => ({
      theme: 'light',
      language: 'ko',
      onboardingDone: false,
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
      completeOnboarding: () => set({ onboardingDone: true }),
    }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => mmkvStateStorage),
      // 함수는 영속화하지 않음
      partialize: (state) => ({
        theme: state.theme,
        language: state.language,
        onboardingDone: state.onboardingDone,
      }),
    }
  )
);
```

### 3-3. 하이드레이션 상태 확인

```tsx
// persist 미들웨어는 앱 시작 시 저장된 상태를 비동기로 복원(하이드레이션)한다.
// 복원 완료 전에 기본값이 표시될 수 있으므로, 하이드레이션 완료를 확인해야 한다.

function App() {
  // MMKV는 동기적이므로 하이드레이션이 즉시 완료됨
  // AsyncStorage 사용 시에는 아래 패턴이 중요
  const hasHydrated = useAppStore.persist.hasHydrated();

  if (!hasHydrated) {
    return <SplashScreen />;
  }

  return <MainApp />;
}

// 또는 onRehydrateStorage 콜백 사용
const useAppStore = create<AppState>()(
  persist(
    (set) => ({ /* ... */ }),
    {
      name: 'app-settings',
      storage: createJSONStorage(() => mmkvStateStorage),
      onRehydrateStorage: (state) => {
        console.log('하이드레이션 시작');
        return (state, error) => {
          if (error) {
            console.error('하이드레이션 실패:', error);
          } else {
            console.log('하이드레이션 완료');
          }
        };
      },
    }
  )
);
```

---

## 4. SQLite

관계형 데이터가 필요할 때 Android의 Room에 해당하는 솔루션이다.

### 4-1. expo-sqlite (Expo 프로젝트)

```bash
npx expo install expo-sqlite
```

```tsx
// database/db.ts
import * as SQLite from 'expo-sqlite';

// Room의 @Database 어노테이션과 유사
const db = SQLite.openDatabaseSync('myapp.db');

// 테이블 생성 (Room의 @Entity와 유사)
export function initDatabase() {
  db.execSync(`
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      email TEXT NOT NULL UNIQUE,
      created_at TEXT DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS posts (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER NOT NULL,
      title TEXT NOT NULL,
      content TEXT,
      created_at TEXT DEFAULT (datetime('now')),
      FOREIGN KEY (user_id) REFERENCES users(id)
    );
  `);
}

// ===== CRUD 함수 (Room의 @Dao와 유사) =====

// Room @Insert
export function insertUser(name: string, email: string) {
  const result = db.runSync(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    [name, email]
  );
  return result.lastInsertRowId;
}

// Room @Query("SELECT * FROM users")
export function getAllUsers() {
  return db.getAllSync<{
    id: number;
    name: string;
    email: string;
    created_at: string;
  }>('SELECT * FROM users ORDER BY created_at DESC');
}

// Room @Query("SELECT * FROM users WHERE id = :id")
export function getUserById(id: number) {
  return db.getFirstSync<{
    id: number;
    name: string;
    email: string;
    created_at: string;
  }>('SELECT * FROM users WHERE id = ?', [id]);
}

// Room @Update
export function updateUser(id: number, name: string, email: string) {
  db.runSync(
    'UPDATE users SET name = ?, email = ? WHERE id = ?',
    [name, email, id]
  );
}

// Room @Delete
export function deleteUser(id: number) {
  db.runSync('DELETE FROM users WHERE id = ?', [id]);
}

// JOIN 쿼리 (Room의 @Relation과 유사)
export function getUserWithPosts(userId: number) {
  const user = getUserById(userId);
  const posts = db.getAllSync<{
    id: number;
    title: string;
    content: string;
    created_at: string;
  }>(
    'SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC',
    [userId]
  );
  return { ...user, posts };
}
```

### 4-2. TanStack Query와 함께 사용

```tsx
// SQLite를 로컬 캐시로 사용하고 TanStack Query로 래핑
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import * as db from '../database/db';

export function useLocalUsers() {
  return useQuery({
    queryKey: ['local-users'],
    queryFn: () => db.getAllUsers(),
    staleTime: Infinity,  // 로컬 데이터는 항상 신선
  });
}

export function useCreateLocalUser() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ name, email }: { name: string; email: string }) =>
      Promise.resolve(db.insertUser(name, email)),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['local-users'] });
    },
  });
}
```

### 4-3. Android Room과의 비교

```kotlin
// Android Room: 상당한 보일러플레이트가 필요
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val name: String,
    val email: String,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

@Dao
interface UserDao {
    @Insert
    suspend fun insert(user: UserEntity): Long

    @Query("SELECT * FROM users ORDER BY created_at DESC")
    fun getAll(): Flow<List<UserEntity>>

    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getById(id: Int): UserEntity?

    @Update
    suspend fun update(user: UserEntity)

    @Delete
    suspend fun delete(user: UserEntity)
}

@Database(entities = [UserEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

```tsx
// React Native expo-sqlite: 훨씬 간결
// Entity, Dao, Database 클래스 모두 불필요
// SQL 문자열로 직접 쿼리 (유연하지만 타입 안전성은 떨어짐)
function getAllUsers() {
  return db.getAllSync('SELECT * FROM users ORDER BY created_at DESC');
}
```

Room의 장점인 컴파일 타임 쿼리 검증과 Flow 지원은 expo-sqlite에 없다. 하지만 React Native에서는 TanStack Query가 Flow의 역할을 대신하며, 대부분의 앱에서 SQLite가 필요한 경우는 드물다 (MMKV + TanStack Query로 충분).

---

## 5. 보안 저장소 (Secure Storage)

Android의 `Keystore` / `EncryptedSharedPreferences`에 해당하는 솔루션이다.

### 5-1. expo-secure-store

```bash
npx expo install expo-secure-store
```

```tsx
import * as SecureStore from 'expo-secure-store';

// Android에서는 내부적으로 Android Keystore + EncryptedSharedPreferences 사용
// iOS에서는 Keychain 사용

// 저장 (Android Keystore에 안전하게 저장)
await SecureStore.setItemAsync('auth_token', 'eyJhbGciOiJIUzI1NiIs...');
await SecureStore.setItemAsync('refresh_token', 'dGhpcyBpcyBhIHJlZnJl...');

// 읽기
const token = await SecureStore.getItemAsync('auth_token');
// token: string | null

// 삭제
await SecureStore.deleteItemAsync('auth_token');

// 옵션
await SecureStore.setItemAsync('biometric_key', 'secret-value', {
  // 생체 인증이 필요한 저장 (Android BiometricPrompt과 유사)
  requireAuthentication: true,
  authenticationPrompt: '본인 확인이 필요합니다',
  // Android KeyGenParameterSpec의 키 보호 수준에 해당
  keychainAccessible: SecureStore.WHEN_UNLOCKED,
});
```

### 5-2. Android Keystore 비교

```kotlin
// Android: EncryptedSharedPreferences
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val securePrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

securePrefs.edit().putString("token", "eyJ...").apply()
val token = securePrefs.getString("token", null)
```

```tsx
// React Native: expo-secure-store (내부적으로 위 Android 코드와 동일)
await SecureStore.setItemAsync('token', 'eyJ...');
const token = await SecureStore.getItemAsync('token');
```

### 5-3. 보안 저장소 사용 지침

```
저장해야 할 데이터           사용할 저장소
────────────────────────   ─────────────────
API 토큰, 리프레시 토큰      SecureStore / MMKV (암호화)
비밀번호, PIN               SecureStore
신용카드 정보               절대 로컬에 저장 X
사용자 설정 (테마, 언어)     MMKV / AsyncStorage
캐시 데이터                 MMKV / AsyncStorage
관계형 데이터               SQLite
대용량 파일                 FileSystem
```

---

## 6. 파일 시스템

Android의 `FileProvider`, `Context.filesDir`, `Context.cacheDir`에 해당한다.

### 6-1. expo-file-system

```bash
npx expo install expo-file-system
```

```tsx
import * as FileSystem from 'expo-file-system';

// ===== 디렉토리 경로 =====
// Android Context.filesDir에 해당
const docsDir = FileSystem.documentDirectory;
// 예: file:///data/user/0/com.myapp/files/

// Android Context.cacheDir에 해당
const cacheDir = FileSystem.cacheDirectory;
// 예: file:///data/user/0/com.myapp/cache/

// ===== 파일 읽기/쓰기 =====

// 텍스트 파일 쓰기
await FileSystem.writeAsStringAsync(
  docsDir + 'notes.txt',
  '안녕하세요! 파일 시스템 테스트입니다.',
  { encoding: FileSystem.EncodingType.UTF8 }
);

// 텍스트 파일 읽기
const content = await FileSystem.readAsStringAsync(
  docsDir + 'notes.txt',
  { encoding: FileSystem.EncodingType.UTF8 }
);

// JSON 파일 쓰기/읽기
const data = { users: [{ id: 1, name: '종우' }] };
await FileSystem.writeAsStringAsync(
  docsDir + 'data.json',
  JSON.stringify(data)
);
const jsonStr = await FileSystem.readAsStringAsync(docsDir + 'data.json');
const parsedData = JSON.parse(jsonStr);

// ===== 파일 정보 =====
const fileInfo = await FileSystem.getInfoAsync(docsDir + 'notes.txt');
// { exists: true, size: 42, modificationTime: 1703..., isDirectory: false, uri: '...' }

// ===== 파일 삭제 =====
await FileSystem.deleteAsync(docsDir + 'notes.txt');

// ===== 디렉토리 작업 =====
// 디렉토리 생성
await FileSystem.makeDirectoryAsync(docsDir + 'images/', {
  intermediates: true, // 중간 디렉토리도 생성 (mkdir -p)
});

// 디렉토리 내용 목록
const files = await FileSystem.readDirectoryAsync(docsDir + 'images/');
// ['photo1.jpg', 'photo2.jpg']

// ===== 파일 다운로드 =====
const downloadResult = await FileSystem.downloadAsync(
  'https://example.com/image.jpg',
  docsDir + 'images/downloaded.jpg'
);
// { uri: 'file:///...', status: 200, headers: {...} }

// 다운로드 진행률 추적
const downloadResumable = FileSystem.createDownloadResumable(
  'https://example.com/large-file.zip',
  docsDir + 'large-file.zip',
  {},
  (downloadProgress) => {
    const progress =
      downloadProgress.totalBytesWritten /
      downloadProgress.totalBytesExpectedToWrite;
    console.log(`다운로드 진행: ${(progress * 100).toFixed(1)}%`);
  }
);

const result = await downloadResumable.downloadAsync();

// ===== 파일 복사/이동 =====
await FileSystem.copyAsync({
  from: docsDir + 'data.json',
  to: cacheDir + 'data-backup.json',
});

await FileSystem.moveAsync({
  from: cacheDir + 'temp-file.txt',
  to: docsDir + 'permanent-file.txt',
});
```

### 6-2. Android FileProvider 비교

```kotlin
// Android: 파일 저장
val file = File(context.filesDir, "notes.txt")
file.writeText("안녕하세요!")

// 파일 읽기
val content = File(context.filesDir, "notes.txt").readText()

// 파일 공유 (FileProvider 필요)
val uri = FileProvider.getUriForFile(
    context, "${context.packageName}.fileprovider", file
)
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```

---

## 7. 어떤 것을 사용할까

### 결정 가이드

```
데이터 특성                      추천 솔루션              Android 대응
──────────────────────────      ─────────────────       ──────────────────
단순 키-값 (설정, 플래그)         MMKV                    SharedPreferences
민감한 키-값 (토큰, 비밀번호)     SecureStore 또는         EncryptedShared
                                MMKV (암호화)            Preferences
전역 앱 상태 (영속)              Zustand + persist       ViewModel + DataStore
                                + MMKV
서버 캐시 데이터                  TanStack Query          Repository + Room
관계형 데이터 (오프라인 우선)      expo-sqlite             Room
대용량 파일 (이미지, 문서)        expo-file-system        FileProvider + filesDir
임시 캐시 파일                   expo-file-system        cacheDir
                                (cacheDirectory)
```

### 성능 비교

```
작업: 1000개 키-값 쌍 읽기/쓰기

AsyncStorage:     ~500ms (비동기, SQLite 기반)
MMKV:             ~15ms  (동기, mmap 기반, C++ JSI)
SharedPreferences: ~100ms (Android 네이티브)
MMKV (Android):   ~3ms   (Android 네이티브 MMKV)
```

### 실전 조합 예시

```
일반적인 앱의 저장소 구성:

1. MMKV                    ← 앱 설정, 사용자 선호도, 간단한 캐시
2. SecureStore             ← 인증 토큰, 민감한 데이터
3. Zustand + persist + MMKV ← 전역 상태 자동 영속화
4. TanStack Query           ← 서버 데이터 캐싱 (메모리)
5. expo-file-system         ← 다운로드한 파일, 이미지 캐시
6. expo-sqlite (선택)       ← 복잡한 오프라인 데이터 (필요시만)
```

### 완전한 저장소 설정 예제

```tsx
// storage/index.ts — 앱에서 사용하는 모든 저장소를 한 곳에서 관리
import { MMKV } from 'react-native-mmkv';
import * as SecureStore from 'expo-secure-store';

// 1. 일반 저장소 (MMKV)
export const storage = new MMKV({ id: 'app-storage' });

// 2. 암호화된 저장소 (MMKV)
export const encryptedStorage = new MMKV({
  id: 'encrypted-storage',
  encryptionKey: 'app-encryption-key-2024',
});

// 3. 보안 저장소 (SecureStore) 래퍼
export const secureStorage = {
  async getToken(): Promise<string | null> {
    return SecureStore.getItemAsync('auth_token');
  },
  async setToken(token: string): Promise<void> {
    await SecureStore.setItemAsync('auth_token', token);
  },
  async removeToken(): Promise<void> {
    await SecureStore.deleteItemAsync('auth_token');
  },
  async getRefreshToken(): Promise<string | null> {
    return SecureStore.getItemAsync('refresh_token');
  },
  async setRefreshToken(token: string): Promise<void> {
    await SecureStore.setItemAsync('refresh_token', token);
  },
  async removeRefreshToken(): Promise<void> {
    await SecureStore.deleteItemAsync('refresh_token');
  },
  async clearAll(): Promise<void> {
    await SecureStore.deleteItemAsync('auth_token');
    await SecureStore.deleteItemAsync('refresh_token');
  },
};

// 4. Zustand용 MMKV StateStorage 어댑터
import { StateStorage } from 'zustand/middleware';

export const zustandMMKVStorage: StateStorage = {
  getItem: (name) => storage.getString(name) ?? null,
  setItem: (name, value) => storage.set(name, value),
  removeItem: (name) => storage.delete(name),
};

// 5. 앱 설정 헬퍼
export const appSettings = {
  getTheme: () => storage.getString('theme') as 'light' | 'dark' | undefined,
  setTheme: (theme: 'light' | 'dark') => storage.set('theme', theme),

  getLanguage: () => storage.getString('language') ?? 'ko',
  setLanguage: (lang: string) => storage.set('language', lang),

  isOnboardingDone: () => storage.getBoolean('onboarding_done') ?? false,
  setOnboardingDone: () => storage.set('onboarding_done', true),

  getLastSyncTime: () => storage.getNumber('last_sync') ?? 0,
  setLastSyncTime: () => storage.set('last_sync', Date.now()),
};
```

---

## 요약: Android 로컬 저장소 vs React Native

```
Android                              React Native
─────────────────────────────────   ─────────────────────────────
SharedPreferences                    MMKV (추천) / AsyncStorage
EncryptedSharedPreferences           SecureStore / MMKV (암호화)
Android Keystore                     SecureStore
DataStore (Proto/Preferences)        Zustand persist + MMKV
Room (SQLite ORM)                    expo-sqlite (직접 SQL)
Context.filesDir                     FileSystem.documentDirectory
Context.cacheDir                     FileSystem.cacheDirectory
FileProvider                         expo-file-system + expo-sharing
ContentProvider                      해당 없음 (앱 간 데이터 공유 제한적)
```

이것으로 Phase 5: 상태 관리와 네트워킹 학습을 마친다. React Native에서의 상태 관리(Zustand), 서버 데이터(TanStack Query), 로컬 저장소(MMKV/SQLite)를 모두 다루었다.
