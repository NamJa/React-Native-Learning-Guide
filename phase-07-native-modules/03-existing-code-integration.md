# 기존 Android 코드 재사용 — Kotlin 자산을 React Native에서 활용하기

## 목차
1. [코드 재사용 전략](#1-코드-재사용-전략)
2. [기존 Kotlin 클래스를 TurboModule로 래핑](#2-기존-kotlin-클래스를-turbomodule로-래핑)
3. [Android 라이브러리 통합](#3-android-라이브러리-통합)
4. [Brownfield: 기존 Android 앱에 React Native 추가](#4-brownfield-기존-android-앱에-react-native-추가)
5. [Greenfield: React Native 앱에서 Kotlin 전문성 활용](#5-greenfield-react-native-앱에서-kotlin-전문성-활용)
6. [Gradle 설정과 ProGuard/R8](#6-gradle-설정과-proguardr8)

---

## 1. 코드 재사용 전략

```
┌────────────────────────────────────────────────────────────────┐
│              코드 재사용 3가지 전략                               │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  전략 1: Kotlin 클래스 → TurboModule 래핑                       │
│  ┌────────────────────────────────────────────┐                │
│  │ 기존 Kotlin 비즈니스 로직 클래스            │                │
│  │ (네트워크, DB, 암호화 등)                   │                │
│  │         │                                   │                │
│  │         ▼ TurboModule로 감싸기              │                │
│  │ class MyTurboModule : NativeSpec {          │                │
│  │   private val existingClass = ExistingClass()│               │
│  │   override fun doWork() =                   │                │
│  │     existingClass.doWork()                  │                │
│  │ }                                           │                │
│  └────────────────────────────────────────────┘                │
│  적합한 경우: 비UI 로직, 유틸리티, 서비스 레이어                  │
│                                                                 │
│  전략 2: 커스텀 View → Fabric Component 래핑                    │
│  ┌────────────────────────────────────────────┐                │
│  │ 기존 커스텀 Android View                    │                │
│  │ (ChartView, MapView 등)                    │                │
│  │         │                                   │                │
│  │         ▼ ViewManager로 감싸기              │                │
│  │ class MyViewManager : SimpleViewManager {   │                │
│  │   override fun createViewInstance() =       │                │
│  │     ExistingCustomView(context)             │                │
│  │ }                                           │                │
│  └────────────────────────────────────────────┘                │
│  적합한 경우: 커스텀 드로잉, 네이티브 UI 컴포넌트                 │
│                                                                 │
│  전략 3: 공유 Kotlin 라이브러리                                  │
│  ┌────────────────────────────────────────────┐                │
│  │ Kotlin 모듈 (.aar / maven dependency)      │                │
│  │ (회사 내부 SDK, 공통 유틸 등)               │                │
│  │         │                                   │                │
│  │         ▼ build.gradle 의존성 추가          │                │
│  │ implementation(project(":shared-module"))    │                │
│  │ 또는                                        │                │
│  │ implementation("com.company:sdk:1.0.0")     │                │
│  └────────────────────────────────────────────┘                │
│  적합한 경우: 기존 앱과 RN 앱이 공통 로직을 공유할 때             │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 2. 기존 Kotlin 클래스를 TurboModule로 래핑

### 2.1 예시: 기존 인증 서비스 래핑

기존 Android 앱에 이미 잘 작동하는 인증 관련 Kotlin 클래스가 있다고 가정한다:

```kotlin
// 기존 코드 — 이미 잘 동작하는 Kotlin 클래스
// shared/src/main/java/com/company/auth/AuthService.kt

package com.company.auth

import android.content.Context
import android.content.SharedPreferences
import java.security.MessageDigest
import java.util.UUID

class AuthService(private val context: Context) {

    private val prefs: SharedPreferences =
        context.getSharedPreferences("auth", Context.MODE_PRIVATE)

    fun isLoggedIn(): Boolean {
        return prefs.getString("access_token", null) != null
    }

    fun getAccessToken(): String? {
        return prefs.getString("access_token", null)
    }

    fun getUserId(): String? {
        return prefs.getString("user_id", null)
    }

    suspend fun login(email: String, password: String): AuthResult {
        // 실제로는 서버 API 호출
        val hashedPassword = hashPassword(password)
        // ... API 호출 로직 ...
        return AuthResult(
            success = true,
            userId = UUID.randomUUID().toString(),
            accessToken = "token_${System.currentTimeMillis()}",
            refreshToken = "refresh_${System.currentTimeMillis()}"
        )
    }

    fun logout() {
        prefs.edit().clear().apply()
    }

    private fun hashPassword(password: String): String {
        val digest = MessageDigest.getInstance("SHA-256")
        val hash = digest.digest(password.toByteArray())
        return hash.joinToString("") { "%02x".format(it) }
    }
}

data class AuthResult(
    val success: Boolean,
    val userId: String,
    val accessToken: String,
    val refreshToken: String,
    val errorMessage: String? = null
)
```

이제 이 기존 클래스를 TurboModule로 래핑한다:

```typescript
// src/specs/NativeAuthService.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  // 기존 AuthService의 메서드들을 그대로 노출
  isLoggedIn(): boolean;          // 동기 — 즉시 확인
  getAccessToken(): string | null;  // 동기
  getUserId(): string | null;       // 동기

  login(email: string, password: string): Promise<{
    success: boolean;
    userId: string;
    accessToken: string;
    refreshToken: string;
    errorMessage: string | null;
  }>;

  logout(): void;
}

export default TurboModuleRegistry.getEnforcing<Spec>('AuthService');
```

```kotlin
// android/app/src/main/java/com/myapp/AuthServiceModule.kt

package com.myapp

import com.company.auth.AuthService  // 기존 클래스 import!
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeAuthServiceSpec
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

@ReactModule(name = AuthServiceModule.NAME)
class AuthServiceModule(
    private val reactContext: ReactApplicationContext
) : NativeAuthServiceSpec(reactContext) {

    companion object {
        const val NAME = "AuthService"
    }

    // 기존 클래스를 그대로 사용!
    private val authService = AuthService(reactContext)
    private val scope = CoroutineScope(Dispatchers.IO)

    override fun getName(): String = NAME

    // 기존 메서드를 단순히 위임 (delegate)
    override fun isLoggedIn(): Boolean {
        return authService.isLoggedIn()
    }

    override fun getAccessToken(): String? {
        return authService.getAccessToken()
    }

    override fun getUserId(): String? {
        return authService.getUserId()
    }

    override fun login(email: String, password: String, promise: Promise) {
        scope.launch {
            try {
                // 기존 suspend 함수 그대로 호출!
                val result = authService.login(email, password)

                val map = Arguments.createMap().apply {
                    putBoolean("success", result.success)
                    putString("userId", result.userId)
                    putString("accessToken", result.accessToken)
                    putString("refreshToken", result.refreshToken)
                    if (result.errorMessage != null) {
                        putString("errorMessage", result.errorMessage)
                    } else {
                        putNull("errorMessage")
                    }
                }
                promise.resolve(map)
            } catch (e: Exception) {
                promise.reject("LOGIN_ERROR", e.message, e)
            }
        }
    }

    override fun logout() {
        authService.logout()
    }
}
```

```
┌────────────────────────────────────────────────────────────────┐
│              래핑 패턴 요약                                      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JS (TypeScript)                                                │
│  └── NativeAuthService.login(email, pw)                        │
│       │                                                         │
│       ▼ JSI 통해 직접 호출                                      │
│  TurboModule (AuthServiceModule.kt)                             │
│  └── override fun login(email, pw, promise)                    │
│       │                                                         │
│       ▼ 단순 위임 (delegate)                                    │
│  기존 코드 (AuthService.kt)                                     │
│  └── suspend fun login(email, password): AuthResult            │
│       │                                                         │
│       ▼ 기존 로직 그대로 실행                                    │
│  결과 → Promise.resolve() → JS로 반환                           │
│                                                                 │
│  핵심: 기존 코드를 수정하지 않고 TurboModule 레이어만 추가        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 3. Android 라이브러리 통합

### 3.1 Retrofit/OkHttp 래핑

```kotlin
// 기존 네트워크 레이어가 Retrofit으로 구현되어 있다면:

// 기존 코드
// shared/src/main/java/com/company/network/ApiService.kt
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") userId: String): UserResponse

    @POST("orders")
    suspend fun createOrder(@Body order: CreateOrderRequest): OrderResponse
}

// 기존 코드
// shared/src/main/java/com/company/network/NetworkClient.kt
object NetworkClient {
    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(AuthInterceptor())
        .addInterceptor(HttpLoggingInterceptor())
        .connectTimeout(30, TimeUnit.SECONDS)
        .build()

    val apiService: ApiService = Retrofit.Builder()
        .baseUrl("https://api.example.com/")
        .client(okHttpClient)
        .addConverterFactory(MoshiConverterFactory.create())
        .build()
        .create(ApiService::class.java)
}
```

```kotlin
// TurboModule로 래핑
// android/app/src/main/java/com/myapp/NetworkModule.kt

package com.myapp

import com.company.network.NetworkClient
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.ReadableMap
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeNetworkSpec
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

@ReactModule(name = NetworkModule.NAME)
class NetworkModule(
    reactContext: ReactApplicationContext
) : NativeNetworkSpec(reactContext) {

    companion object {
        const val NAME = "Network"
    }

    // 기존 NetworkClient를 그대로 사용!
    private val api = NetworkClient.apiService
    private val scope = CoroutineScope(Dispatchers.IO)

    override fun getName(): String = NAME

    override fun getUser(userId: String, promise: Promise) {
        scope.launch {
            try {
                val user = api.getUser(userId)

                val map = Arguments.createMap().apply {
                    putString("id", user.id)
                    putString("name", user.name)
                    putString("email", user.email)
                    putString("avatarUrl", user.avatarUrl)
                }
                promise.resolve(map)
            } catch (e: Exception) {
                promise.reject("NETWORK_ERROR", e.message, e)
            }
        }
    }

    override fun createOrder(orderData: ReadableMap, promise: Promise) {
        scope.launch {
            try {
                val request = CreateOrderRequest(
                    productId = orderData.getString("productId") ?: "",
                    quantity = orderData.getInt("quantity"),
                    address = orderData.getString("address") ?: ""
                )

                val response = api.createOrder(request)

                val map = Arguments.createMap().apply {
                    putString("orderId", response.orderId)
                    putString("status", response.status)
                    putDouble("totalPrice", response.totalPrice)
                }
                promise.resolve(map)
            } catch (e: Exception) {
                promise.reject("ORDER_ERROR", e.message, e)
            }
        }
    }
}
```

### 3.2 Room DB 래핑

```kotlin
// 기존 Room DB 코드

// entities
@Entity(tableName = "notes")
data class NoteEntity(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

// DAO
@Dao
interface NoteDao {
    @Query("SELECT * FROM notes ORDER BY updatedAt DESC")
    suspend fun getAll(): List<NoteEntity>

    @Query("SELECT * FROM notes WHERE id = :id")
    suspend fun getById(id: Long): NoteEntity?

    @Insert
    suspend fun insert(note: NoteEntity): Long

    @Update
    suspend fun update(note: NoteEntity)

    @Query("DELETE FROM notes WHERE id = :id")
    suspend fun deleteById(id: Long)

    @Query("SELECT * FROM notes WHERE title LIKE :query OR content LIKE :query")
    suspend fun search(query: String): List<NoteEntity>
}

// Database
@Database(entities = [NoteEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun noteDao(): NoteDao
}
```

```typescript
// src/specs/NativeNoteDB.ts

import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getAllNotes(): Promise<Object[]>;
  getNoteById(id: number): Promise<Object | null>;
  createNote(title: string, content: string): Promise<number>; // 생성된 ID
  updateNote(id: number, title: string, content: string): Promise<boolean>;
  deleteNote(id: number): Promise<boolean>;
  searchNotes(query: string): Promise<Object[]>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NoteDB');
```

```kotlin
// android/app/src/main/java/com/myapp/NoteDBModule.kt

package com.myapp

import androidx.room.Room
import com.company.db.AppDatabase  // 기존 DB 클래스!
import com.company.db.NoteEntity
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.bridge.WritableArray
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeNoteDBSpec
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

@ReactModule(name = NoteDBModule.NAME)
class NoteDBModule(
    private val reactContext: ReactApplicationContext
) : NativeNoteDBSpec(reactContext) {

    companion object {
        const val NAME = "NoteDB"
    }

    // 기존 Room DB 인스턴스 사용
    private val db: AppDatabase by lazy {
        Room.databaseBuilder(
            reactContext,
            AppDatabase::class.java,
            "app-database"
        ).build()
    }

    private val noteDao by lazy { db.noteDao() }
    private val scope = CoroutineScope(Dispatchers.IO)

    override fun getName(): String = NAME

    override fun getAllNotes(promise: Promise) {
        scope.launch {
            try {
                val notes = noteDao.getAll()
                promise.resolve(notesToWritableArray(notes))
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    override fun getNoteById(id: Double, promise: Promise) {
        scope.launch {
            try {
                val note = noteDao.getById(id.toLong())
                if (note != null) {
                    promise.resolve(noteToWritableMap(note))
                } else {
                    promise.resolve(null)
                }
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    override fun createNote(title: String, content: String, promise: Promise) {
        scope.launch {
            try {
                val entity = NoteEntity(title = title, content = content)
                val id = noteDao.insert(entity)
                promise.resolve(id.toDouble())
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    override fun updateNote(
        id: Double,
        title: String,
        content: String,
        promise: Promise
    ) {
        scope.launch {
            try {
                val existing = noteDao.getById(id.toLong())
                if (existing != null) {
                    noteDao.update(
                        existing.copy(
                            title = title,
                            content = content,
                            updatedAt = System.currentTimeMillis()
                        )
                    )
                    promise.resolve(true)
                } else {
                    promise.resolve(false)
                }
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    override fun deleteNote(id: Double, promise: Promise) {
        scope.launch {
            try {
                noteDao.deleteById(id.toLong())
                promise.resolve(true)
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    override fun searchNotes(query: String, promise: Promise) {
        scope.launch {
            try {
                val notes = noteDao.search("%$query%")
                promise.resolve(notesToWritableArray(notes))
            } catch (e: Exception) {
                promise.reject("DB_ERROR", e.message, e)
            }
        }
    }

    // ═══ 변환 헬퍼 ═══

    private fun noteToWritableMap(note: NoteEntity) = Arguments.createMap().apply {
        putDouble("id", note.id.toDouble())
        putString("title", note.title)
        putString("content", note.content)
        putDouble("createdAt", note.createdAt.toDouble())
        putDouble("updatedAt", note.updatedAt.toDouble())
    }

    private fun notesToWritableArray(notes: List<NoteEntity>): WritableArray {
        return Arguments.createArray().apply {
            notes.forEach { pushMap(noteToWritableMap(it)) }
        }
    }
}
```

### 3.3 WorkManager 래핑

```kotlin
// 기존 WorkManager 작업

// shared/src/main/java/com/company/workers/SyncWorker.kt
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        // 서버와 데이터 동기화
        val syncResult = SyncManager.sync()
        return if (syncResult.success) Result.success() else Result.retry()
    }
}
```

```kotlin
// TurboModule로 래핑
// android/app/src/main/java/com/myapp/BackgroundTaskModule.kt

package com.myapp

import androidx.work.*
import com.company.workers.SyncWorker
import com.facebook.react.bridge.Arguments
import com.facebook.react.bridge.Promise
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.module.annotations.ReactModule
import com.myapp.specs.NativeBackgroundTaskSpec
import java.util.concurrent.TimeUnit

@ReactModule(name = BackgroundTaskModule.NAME)
class BackgroundTaskModule(
    private val reactContext: ReactApplicationContext
) : NativeBackgroundTaskSpec(reactContext) {

    companion object {
        const val NAME = "BackgroundTask"
    }

    override fun getName(): String = NAME

    override fun scheduleSync(intervalMinutes: Double, promise: Promise) {
        try {
            val constraints = Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .setRequiresBatteryNotLow(true)
                .build()

            val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
                intervalMinutes.toLong(), TimeUnit.MINUTES
            )
                .setConstraints(constraints)
                .addTag("sync")
                .build()

            WorkManager.getInstance(reactContext)
                .enqueueUniquePeriodicWork(
                    "periodic_sync",
                    ExistingPeriodicWorkPolicy.UPDATE,
                    syncRequest
                )

            promise.resolve(syncRequest.id.toString())
        } catch (e: Exception) {
            promise.reject("WORK_ERROR", e.message, e)
        }
    }

    override fun cancelSync(promise: Promise) {
        try {
            WorkManager.getInstance(reactContext)
                .cancelUniqueWork("periodic_sync")
            promise.resolve(true)
        } catch (e: Exception) {
            promise.reject("WORK_ERROR", e.message, e)
        }
    }

    override fun getSyncStatus(promise: Promise) {
        try {
            val workInfos = WorkManager.getInstance(reactContext)
                .getWorkInfosForUniqueWork("periodic_sync")
                .get()

            val info = workInfos.firstOrNull()
            val map = Arguments.createMap().apply {
                putString("state", info?.state?.name ?: "UNKNOWN")
                putBoolean("isRunning", info?.state == WorkInfo.State.RUNNING)
                putBoolean("isEnqueued", info?.state == WorkInfo.State.ENQUEUED)
            }
            promise.resolve(map)
        } catch (e: Exception) {
            promise.reject("WORK_ERROR", e.message, e)
        }
    }

    override fun runSyncNow(promise: Promise) {
        try {
            val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>()
                .addTag("sync_now")
                .build()

            WorkManager.getInstance(reactContext)
                .enqueue(syncRequest)

            promise.resolve(syncRequest.id.toString())
        } catch (e: Exception) {
            promise.reject("WORK_ERROR", e.message, e)
        }
    }
}
```

---

## 4. Brownfield: 기존 Android 앱에 React Native 추가

### 4.1 개요

Brownfield란 **기존에 운영 중인 네이티브 Android 앱에 React Native 화면을 추가**하는 방식이다.

```
┌────────────────────────────────────────────────────────────────┐
│              Brownfield 접근 방식                                │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  기존 Android 앱                                                │
│  ┌──────────────────────────────────────────┐                  │
│  │ MainActivity (Kotlin)                     │                  │
│  │  └── HomeFragment (Kotlin)                │                  │
│  │       ├── OrdersFragment (Kotlin)         │                  │
│  │       ├── ProfileFragment (Kotlin)        │                  │
│  │       └── SettingsFragment (RN!)  ← 새로 추가               │
│  │                                           │                  │
│  │ CheckoutActivity (RN!)  ← 새로 추가       │                  │
│  │                                           │                  │
│  │ 나머지는 기존 Kotlin 코드 그대로 유지       │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
│  장점:                                                          │
│  • 점진적 마이그레이션 가능                                      │
│  • 기존 앱 안정성 유지                                           │
│  • 새 기능만 React Native로 개발                                 │
│  • 팀이 점차 React Native에 적응                                 │
│                                                                 │
│  단점:                                                          │
│  • 빌드 시간 증가 (두 시스템 모두 빌드)                           │
│  • 앱 크기 증가 (Hermes + RN 런타임 포함)                        │
│  • 네이티브 ↔ RN 간 데이터 공유 복잡                             │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 단계별 가이드

**Step 1: 프로젝트 구조 설정**

```
my-android-app/                 ← 기존 Android 프로젝트 루트
├── app/
│   ├── src/main/java/com/myapp/
│   │   ├── MainActivity.kt      ← 기존 코드
│   │   ├── ReactActivity.kt     ← 새로 추가: RN 호스트 Activity
│   │   └── ...
│   └── build.gradle
├── rn/                          ← React Native 코드 (새 디렉토리)
│   ├── index.js                 ← RN 엔트리 포인트
│   ├── package.json
│   ├── src/
│   │   └── screens/
│   │       └── SettingsScreen.tsx
│   └── node_modules/
├── settings.gradle
└── build.gradle (project level)
```

**Step 2: settings.gradle 수정**

```groovy
// settings.gradle

// React Native 자동 링킹
apply from: file("rn/node_modules/@react-native-community/cli-platform-android/native_modules.gradle")
applyNativeModulesSettingsGradle(settings, "rn")

include ':app'
```

**Step 3: app/build.gradle 수정**

```groovy
// app/build.gradle

apply plugin: "com.android.application"
apply plugin: "org.jetbrains.kotlin.android"
// React Native 설정 적용
apply from: file("../rn/node_modules/react-native/react.gradle")

android {
    compileSdk 34

    defaultConfig {
        applicationId "com.myapp"
        minSdk 24
        targetSdk 34
    }

    buildTypes {
        debug {
            // Metro 번들러 연결 허용
        }
        release {
            // Hermes 바이트코드 번들 포함
        }
    }
}

dependencies {
    // 기존 의존성들...
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("com.google.android.material:material:1.11.0")

    // React Native 추가
    implementation("com.facebook.react:react-android")
    implementation("com.facebook.react:hermes-android")
}

// React Native 네이티브 모듈 자동 링킹
apply from: file("../rn/node_modules/@react-native-community/cli-platform-android/native_modules.gradle")
applyNativeModulesAppBuildGradle(project, "../rn")
```

**Step 4: React Native 호스트 Activity 생성**

```kotlin
// app/src/main/java/com/myapp/ReactNativeActivity.kt

package com.myapp

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import com.facebook.react.ReactInstanceManager
import com.facebook.react.ReactRootView
import com.facebook.react.common.LifecycleState
import com.facebook.react.modules.core.DefaultHardwareBackBtnHandler
import com.facebook.react.shell.MainReactPackage

class ReactNativeActivity : AppCompatActivity(), DefaultHardwareBackBtnHandler {

    private lateinit var reactRootView: ReactRootView
    private lateinit var reactInstanceManager: ReactInstanceManager

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // React Native 인스턴스 매니저 생성
        reactInstanceManager = ReactInstanceManager.builder()
            .setApplication(application)
            .setCurrentActivity(this)
            .setBundleAssetName("index.android.bundle")
            .setJSMainModulePath("index")
            .addPackage(MainReactPackage())
            // 커스텀 패키지 추가
            // .addPackage(MyCustomPackage())
            .setUseDeveloperSupport(BuildConfig.DEBUG)
            .setInitialLifecycleState(LifecycleState.RESUMED)
            .build()

        // React Root View 생성
        reactRootView = ReactRootView(this)

        // 시작할 RN 컴포넌트 이름과 초기 props
        val initialProps = Bundle().apply {
            // 네이티브에서 RN으로 데이터 전달
            putString("userId", intent.getStringExtra("userId"))
            putString("screen", intent.getStringExtra("screen") ?: "settings")
        }

        reactRootView.startReactApplication(
            reactInstanceManager,
            "MyRNApp",       // AppRegistry.registerComponent 이름
            initialProps     // 초기 props (JS에서 받을 수 있음)
        )

        setContentView(reactRootView)
    }

    override fun invokeDefaultOnBackPressed() {
        super.onBackPressed()
    }

    override fun onPause() {
        super.onPause()
        reactInstanceManager.onHostPause(this)
    }

    override fun onResume() {
        super.onResume()
        reactInstanceManager.onHostResume(this, this)
    }

    override fun onDestroy() {
        super.onDestroy()
        reactInstanceManager.onHostDestroy(this)
        reactRootView.unmountReactApplication()
    }

    @Deprecated("Deprecated in Java")
    override fun onBackPressed() {
        reactInstanceManager.onBackPressed()
    }
}
```

**Step 5: 기존 네이티브 화면에서 RN 화면으로 이동**

```kotlin
// 기존 Kotlin 코드에서 RN 화면 열기

// MainActivity.kt 또는 다른 Activity에서:
class MainActivity : AppCompatActivity() {

    // ... 기존 코드 ...

    private fun openReactNativeSettings() {
        val intent = Intent(this, ReactNativeActivity::class.java).apply {
            putExtra("userId", currentUser.id)
            putExtra("screen", "settings")
        }
        startActivity(intent)
    }

    private fun openReactNativeCheckout(orderId: String) {
        val intent = Intent(this, ReactNativeActivity::class.java).apply {
            putExtra("screen", "checkout")
            putExtra("orderId", orderId)
        }
        startActivityForResult(intent, REQUEST_CODE_CHECKOUT)
    }

    // RN에서 결과 받기
    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if (requestCode == REQUEST_CODE_CHECKOUT && resultCode == RESULT_OK) {
            val success = data?.getBooleanExtra("success", false) ?: false
            if (success) {
                // 결제 성공 처리
            }
        }
    }
}
```

**Step 6: React Native에서 초기 props 받기**

```tsx
// rn/index.js

import { AppRegistry } from 'react-native';
import App from './src/App';

AppRegistry.registerComponent('MyRNApp', () => App);
```

```tsx
// rn/src/App.tsx

import React from 'react';
import SettingsScreen from './screens/SettingsScreen';
import CheckoutScreen from './screens/CheckoutScreen';

interface AppProps {
  userId?: string;
  screen?: string;
  orderId?: string;
}

// 네이티브에서 전달한 initialProps를 받음
export default function App(props: AppProps) {
  switch (props.screen) {
    case 'checkout':
      return <CheckoutScreen orderId={props.orderId} />;
    case 'settings':
    default:
      return <SettingsScreen userId={props.userId} />;
  }
}
```

### 4.3 네이티브 ↔ RN 간 데이터 공유

```
┌────────────────────────────────────────────────────────────────┐
│           데이터 공유 방법들                                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Initial Props (네이티브 → RN, 시작 시 1회)                  │
│     Intent extras → Bundle → RN initialProperties              │
│     간단하지만 단방향, 시작 시에만                                │
│                                                                 │
│  2. TurboModule (양방향, 실시간)                                 │
│     RN → Native: TurboModule 메서드 호출                        │
│     Native → RN: 이벤트 발송 (DeviceEventEmitter)               │
│     가장 유연한 방법                                             │
│                                                                 │
│  3. SharedPreferences (양방향, 비실시간)                         │
│     네이티브와 RN이 같은 SharedPreferences 접근                  │
│     간단한 설정값 공유에 적합                                    │
│                                                                 │
│  4. Room DB (양방향, 비실시간)                                   │
│     네이티브와 RN이 같은 SQLite DB 접근                         │
│     구조화된 데이터 공유에 적합                                   │
│                                                                 │
│  5. Activity Result (RN → 네이티브, 종료 시)                    │
│     RN 화면 닫으면서 결과 전달                                   │
│     setResult(RESULT_OK, intent) 패턴                           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 5. Greenfield: React Native 앱에서 Kotlin 전문성 활용

Greenfield는 **처음부터 React Native로 새 앱을 시작**하는 방식이다. Kotlin 경험을 다음과 같이 활용할 수 있다:

```
┌────────────────────────────────────────────────────────────────┐
│          Greenfield에서 Kotlin 전문성 활용                       │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 성능 민감 로직 → TurboModule (Kotlin)                       │
│     • 이미지 처리, 암호화, 복잡한 계산                           │
│     • JS보다 Kotlin이 월등히 빠른 작업                           │
│                                                                 │
│  2. 플랫폼 API → TurboModule (Kotlin)                           │
│     • 기존에 잘 알고 있는 Android API들                          │
│     • Camera, Bluetooth, NFC, Sensor 등                         │
│                                                                 │
│  3. 커스텀 UI → Fabric Component (Kotlin)                       │
│     • Canvas 드로잉이 필요한 복잡한 위젯                         │
│     • 기존 Android 라이브러리 뷰 활용                            │
│                                                                 │
│  4. build.gradle 설정                                           │
│     • Android 빌드 시스템에 대한 깊은 이해                       │
│     • 의존성 관리, ProGuard, 멀티모듈 설정                       │
│                                                                 │
│  5. 디버깅 & 프로파일링                                          │
│     • Android Studio로 네이티브 레이어 디버깅                    │
│     • Layout Inspector, Memory Profiler 활용                     │
│                                                                 │
│  핵심: UI는 React Native로, 플랫폼 기능은 Kotlin으로             │
│  양쪽의 장점을 최대한 활용                                       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

**실전 아키텍처 예시**:

```
┌────────────────────────────────────────────────────────────────┐
│                Greenfield 프로젝트 구조                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  React Native Layer (TypeScript)                                │
│  ┌──────────────────────────────────────────┐                  │
│  │ UI 컴포넌트 (View, Text, FlatList...)    │                  │
│  │ 화면 구성 (Navigation, Screens)           │                  │
│  │ 상태 관리 (Zustand/Redux)                 │                  │
│  │ API 호출 (TanStack Query + fetch)         │                  │
│  │ 폼 처리 (React Hook Form)                │                  │
│  └────────────────┬─────────────────────────┘                  │
│                   │ TurboModules (JSI)                          │
│                   ▼                                             │
│  Native Layer (Kotlin)                                          │
│  ┌──────────────────────────────────────────┐                  │
│  │ BiometricAuth Module (지문/얼굴 인증)     │                  │
│  │ SecureStorage Module (KeyStore 암호화)    │                  │
│  │ PushNotification Module (FCM)             │                  │
│  │ FileManager Module (파일 다운로드/압축)    │                  │
│  │ NativeVideoPlayer Component (ExoPlayer)   │                  │
│  │ NativeMapView Component (Google Maps)     │                  │
│  └──────────────────────────────────────────┘                  │
│                                                                 │
│  비율 (일반적인 앱):                                            │
│  TypeScript: 80~90% (UI + 비즈니스 로직)                        │
│  Kotlin: 10~20% (플랫폼 기능 + 성능 민감 로직)                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 6. Gradle 설정과 ProGuard/R8

### 6.1 android/app/build.gradle 설정

```groovy
// android/app/build.gradle

plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    // React Native New Architecture
    id("com.facebook.react")
}

react {
    // Codegen 설정
    jsRootDir = file("../../src")
    libraryName = "MyAppSpecs"
    codegenJavaPackageName = "com.myapp.specs"
}

android {
    namespace = "com.myapp"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.myapp"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0.0"
    }

    buildTypes {
        debug {
            // 디버그 빌드 설정
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    // React Native (자동 관리)
    implementation("com.facebook.react:react-android")
    implementation("com.facebook.react:hermes-android")

    // ═══ 네이티브 모듈에 필요한 의존성 추가 ═══

    // 생체 인증
    implementation("androidx.biometric:biometric:1.1.0")

    // Material Design (Snackbar 등)
    implementation("com.google.android.material:material:1.11.0")

    // 비디오 플레이어 (ExoPlayer / Media3)
    implementation("androidx.media3:media3-exoplayer:1.2.1")
    implementation("androidx.media3:media3-ui:1.2.1")

    // Room DB
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // 네트워크 (기존 코드 재사용 시)
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-moshi:2.9.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // WorkManager
    implementation("androidx.work:work-runtime-ktx:2.9.0")

    // CameraX
    implementation("androidx.camera:camera-camera2:1.3.1")
    implementation("androidx.camera:camera-lifecycle:1.3.1")
    implementation("androidx.camera:camera-view:1.3.1")

    // Google Maps
    implementation("com.google.android.gms:play-services-maps:18.2.0")
}
```

### 6.2 ProGuard/R8 설정

R8(ProGuard 후속)은 릴리스 빌드에서 코드를 난독화하고 축소한다. React Native와 네이티브 모듈에 필요한 규칙을 추가해야 한다.

```
# android/app/proguard-rules.pro

# ═══ React Native 기본 규칙 ═══
# (대부분 react-native 라이브러리가 자동으로 포함하지만 확인용)

-keep class com.facebook.react.** { *; }
-keep class com.facebook.hermes.** { *; }
-keep class com.facebook.jni.** { *; }

# JSI 관련
-keep class com.facebook.react.bridge.** { *; }
-keep class com.facebook.react.turbomodule.** { *; }

# ═══ 커스텀 TurboModule 규칙 ═══
# @ReactModule 어노테이션이 붙은 클래스 보호
-keep @com.facebook.react.module.annotations.ReactModule class * { *; }

# TurboModule Spec (Codegen 생성 클래스) 보호
-keep class com.myapp.specs.** { *; }

# 커스텀 모듈 보호
-keep class com.myapp.*Module { *; }
-keep class com.myapp.*Package { *; }
-keep class com.myapp.*Manager { *; }

# ═══ 사용하는 라이브러리별 규칙 ═══

# Retrofit
-keepattributes Signature, InnerClasses, EnclosingMethod
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations
-keepclassmembers,allowshrinking,allowobfuscation interface * {
    @retrofit2.http.* <methods>;
}

# Moshi
-keepclasseswithmembers class * {
    @com.squareup.moshi.* <methods>;
}
-keep @com.squareup.moshi.JsonQualifier interface *

# Room
-keep class * extends androidx.room.RoomDatabase { *; }
-keep @androidx.room.Entity class * { *; }
-keep @androidx.room.Dao interface * { *; }

# ExoPlayer / Media3
-keep class androidx.media3.** { *; }

# ═══ 디버그 정보 ═══
# 크래시 리포트를 위해 소스 파일 정보 유지
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile
```

### 6.3 멀티모듈 프로젝트 구성 (대규모 프로젝트)

```
┌────────────────────────────────────────────────────────────────┐
│          멀티모듈 구성 (선택사항, 대규모 프로젝트용)               │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  프로젝트 구조:                                                  │
│  my-app/                                                        │
│  ├── android/                                                   │
│  │   ├── app/              (React Native 앱 모듈)               │
│  │   ├── native-modules/   (TurboModule 모음)                   │
│  │   │   ├── auth/                                              │
│  │   │   ├── storage/                                           │
│  │   │   └── media/                                             │
│  │   └── shared/           (기존 Kotlin 코드, 공통 로직)         │
│  │       ├── network/                                           │
│  │       ├── database/                                          │
│  │       └── utils/                                             │
│  └── src/                  (React Native TypeScript 코드)       │
│                                                                 │
│  settings.gradle:                                               │
│  include ':app'                                                 │
│  include ':native-modules:auth'                                 │
│  include ':native-modules:storage'                              │
│  include ':native-modules:media'                                │
│  include ':shared:network'                                      │
│  include ':shared:database'                                     │
│                                                                 │
│  app/build.gradle:                                              │
│  dependencies {                                                 │
│    implementation(project(":native-modules:auth"))              │
│    implementation(project(":native-modules:storage"))           │
│    implementation(project(":native-modules:media"))             │
│  }                                                              │
│                                                                 │
│  native-modules/auth/build.gradle:                              │
│  dependencies {                                                 │
│    implementation(project(":shared:network"))                   │
│    implementation(project(":shared:database"))                  │
│  }                                                              │
│                                                                 │
│  장점:                                                          │
│  • 모듈별 독립 빌드 가능 → 빌드 캐시 효율                       │
│  • 관심사 분리 명확                                              │
│  • 기존 Kotlin 모듈을 그대로 의존성으로 추가                     │
│  • 다른 프로젝트에서도 모듈 재사용 가능                           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 요약

```
┌────────────────────────────────────────────────────────────────┐
│           기존 Android 코드 재사용 핵심 정리                      │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Kotlin 클래스 재사용:                                           │
│  • 기존 클래스를 수정하지 않고 TurboModule 래퍼 추가             │
│  • private val existing = ExistingClass(context)                │
│  • override fun method() = existing.method()                   │
│                                                                 │
│  라이브러리 통합:                                                │
│  • Retrofit → TurboModule에서 API 호출 위임                    │
│  • Room DB → TurboModule에서 DAO 메서드 위임                   │
│  • WorkManager → TurboModule에서 작업 스케줄링                  │
│  • build.gradle에 implementation 추가만 하면 됨                 │
│                                                                 │
│  Brownfield (기존 앱 + RN):                                     │
│  • ReactNativeActivity로 RN 화면 호스팅                         │
│  • Intent extras → initialProperties로 데이터 전달              │
│  • 점진적 마이그레이션에 적합                                    │
│                                                                 │
│  Greenfield (새 RN 앱):                                         │
│  • UI는 React Native (80~90%)                                   │
│  • 플랫폼 기능은 Kotlin TurboModule (10~20%)                    │
│  • Kotlin 전문성 = 강력한 네이티브 모듈 개발 능력                │
│                                                                 │
│  Gradle/ProGuard:                                               │
│  • build.gradle에서 네이티브 의존성 관리                         │
│  • proguard-rules.pro에서 RN + 커스텀 모듈 보호 규칙            │
│  • 대규모 프로젝트는 멀티모듈 구성 고려                          │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```
