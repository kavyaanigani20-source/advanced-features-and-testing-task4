# Advanced Features and Testing - Android Kotlin Code

This sample adds:

- User authentication with saved login session
- Data persistence using Room
- Local push-style notifications with WorkManager
- Third-party libraries: Room, Coroutines, WorkManager, Hilt, Mockito
- Unit and integration test examples
- LinkedIn post draft

## 1. Gradle Dependencies

Add these to `app/build.gradle`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("com.google.android.material:material:1.12.0")

    // Room database
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    kapt("androidx.room:room-compiler:2.6.1")

    // Lifecycle + ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.4")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.4")

    // WorkManager notifications/background work
    implementation("androidx.work:work-runtime-ktx:2.9.1")

    // Hilt dependency injection
    implementation("com.google.dagger:hilt-android:2.51.1")
    kapt("com.google.dagger:hilt-android-compiler:2.51.1")

    // Testing
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.mockito:mockito-core:5.12.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")

    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")
    androidTestImplementation("androidx.room:room-testing:2.6.1")
}
```

## 2. User Entity

Create `User.kt`:

```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val name: String,
    val email: String,
    val password: String
)
```

## 3. Room DAO

Create `UserDao.kt`:

```kotlin
import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query

@Dao
interface UserDao {

    @Insert(onConflict = OnConflictStrategy.ABORT)
    suspend fun registerUser(user: User)

    @Query("SELECT * FROM users WHERE email = :email AND password = :password LIMIT 1")
    suspend fun loginUser(email: String, password: String): User?

    @Query("SELECT * FROM users WHERE email = :email LIMIT 1")
    suspend fun findByEmail(email: String): User?
}
```

## 4. Room Database

Create `AppDatabase.kt`:

```kotlin
import androidx.room.Database
import androidx.room.RoomDatabase

@Database(
    entities = [User::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

## 5. Repository

Create `AuthRepository.kt`:

```kotlin
class AuthRepository(
    private val userDao: UserDao
) {
    suspend fun register(name: String, email: String, password: String): Result<Unit> {
        if (name.isBlank() || email.isBlank() || password.isBlank()) {
            return Result.failure(IllegalArgumentException("All fields are required"))
        }

        if (userDao.findByEmail(email) != null) {
            return Result.failure(IllegalArgumentException("Email already registered"))
        }

        userDao.registerUser(
            User(
                name = name.trim(),
                email = email.trim(),
                password = password
            )
        )

        return Result.success(Unit)
    }

    suspend fun login(email: String, password: String): Result<User> {
        val user = userDao.loginUser(email.trim(), password)

        return if (user != null) {
            Result.success(user)
        } else {
            Result.failure(IllegalArgumentException("Invalid email or password"))
        }
    }
}
```

## 6. Login Session With SharedPreferences

Create `SessionManager.kt`:

```kotlin
import android.content.Context

class SessionManager(context: Context) {

    private val prefs = context.getSharedPreferences("user_session", Context.MODE_PRIVATE)

    fun saveLogin(userId: Int, email: String) {
        prefs.edit()
            .putBoolean("is_logged_in", true)
            .putInt("user_id", userId)
            .putString("email", email)
            .apply()
    }

    fun isLoggedIn(): Boolean {
        return prefs.getBoolean("is_logged_in", false)
    }

    fun logout() {
        prefs.edit().clear().apply()
    }
}
```

## 7. ViewModel

Create `AuthViewModel.kt`:

```kotlin
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

data class AuthUiState(
    val isLoading: Boolean = false,
    val isLoggedIn: Boolean = false,
    val message: String? = null
)

class AuthViewModel(
    private val repository: AuthRepository,
    private val sessionManager: SessionManager
) : ViewModel() {

    private val _uiState = MutableStateFlow(AuthUiState())
    val uiState: StateFlow<AuthUiState> = _uiState

    fun register(name: String, email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = AuthUiState(isLoading = true)

            val result = repository.register(name, email, password)

            _uiState.value = result.fold(
                onSuccess = {
                    AuthUiState(message = "Registration successful")
                },
                onFailure = {
                    AuthUiState(message = it.message)
                }
            )
        }
    }

    fun login(email: String, password: String) {
        viewModelScope.launch {
            _uiState.value = AuthUiState(isLoading = true)

            val result = repository.login(email, password)

            _uiState.value = result.fold(
                onSuccess = { user ->
                    sessionManager.saveLogin(user.id, user.email)
                    AuthUiState(isLoggedIn = true, message = "Login successful")
                },
                onFailure = {
                    AuthUiState(message = it.message)
                }
            )
        }
    }
}
```

## 8. Hilt App Module

Create `AppModule.kt`:

```kotlin
import android.content.Context
import androidx.room.Room
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "advanced_app_db"
        ).build()
    }

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao()
    }

    @Provides
    fun provideAuthRepository(userDao: UserDao): AuthRepository {
        return AuthRepository(userDao)
    }

    @Provides
    fun provideSessionManager(@ApplicationContext context: Context): SessionManager {
        return SessionManager(context)
    }
}
```

## 9. Notification Worker

Create `ReminderWorker.kt`:

```kotlin
import android.Manifest
import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import androidx.core.app.ActivityCompat
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.work.Worker
import androidx.work.WorkerParameters

class ReminderWorker(
    private val context: Context,
    workerParams: WorkerParameters
) : Worker(context, workerParams) {

    override fun doWork(): Result {
        createNotificationChannel()

        if (
            Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU &&
            ActivityCompat.checkSelfPermission(
                context,
                Manifest.permission.POST_NOTIFICATIONS
            ) != PackageManager.PERMISSION_GRANTED
        ) {
            return Result.success()
        }

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(android.R.drawable.ic_dialog_info)
            .setContentTitle("Advanced App")
            .setContentText("Your app data is saved and ready.")
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .build()

        NotificationManagerCompat.from(context).notify(1001, notification)
        return Result.success()
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "App reminders",
                NotificationManager.IMPORTANCE_DEFAULT
            )

            val manager = context.getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(channel)
        }
    }

    companion object {
        private const val CHANNEL_ID = "advanced_app_notifications"
    }
}
```

Schedule it from an Activity or ViewModel layer:

```kotlin
import android.content.Context
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import java.util.concurrent.TimeUnit

fun scheduleReminder(context: Context) {
    val request = OneTimeWorkRequestBuilder<ReminderWorker>()
        .setInitialDelay(10, TimeUnit.SECONDS)
        .build()

    WorkManager.getInstance(context).enqueue(request)
}
```

## 10. Unit Test For Repository

Create `AuthRepositoryTest.kt`:

```kotlin
import kotlinx.coroutines.test.runTest
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Before
import org.junit.Test
import org.mockito.Mockito.mock
import org.mockito.Mockito.`when`

class AuthRepositoryTest {

    private lateinit var userDao: UserDao
    private lateinit var repository: AuthRepository

    @Before
    fun setup() {
        userDao = mock(UserDao::class.java)
        repository = AuthRepository(userDao)
    }

    @Test
    fun loginReturnsUserWhenCredentialsAreValid() = runTest {
        val expectedUser = User(id = 1, name = "Test User", email = "test@mail.com", password = "123456")

        `when`(userDao.loginUser("test@mail.com", "123456")).thenReturn(expectedUser)

        val result = repository.login("test@mail.com", "123456")

        assertTrue(result.isSuccess)
        assertEquals(expectedUser, result.getOrNull())
    }

    @Test
    fun loginFailsWhenCredentialsAreInvalid() = runTest {
        `when`(userDao.loginUser("wrong@mail.com", "bad")).thenReturn(null)

        val result = repository.login("wrong@mail.com", "bad")

        assertTrue(result.isFailure)
    }
}
```

## 11. Room Integration Test

Create `UserDaoTest.kt` inside `androidTest`:

```kotlin
import android.content.Context
import androidx.room.Room
import androidx.test.core.app.ApplicationProvider
import androidx.test.ext.junit.runners.AndroidJUnit4
import kotlinx.coroutines.runBlocking
import org.junit.After
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNotNull
import org.junit.Before
import org.junit.Test
import org.junit.runner.RunWith

@RunWith(AndroidJUnit4::class)
class UserDaoTest {

    private lateinit var database: AppDatabase
    private lateinit var userDao: UserDao

    @Before
    fun setup() {
        val context = ApplicationProvider.getApplicationContext<Context>()

        database = Room.inMemoryDatabaseBuilder(
            context,
            AppDatabase::class.java
        ).allowMainThreadQueries().build()

        userDao = database.userDao()
    }

    @After
    fun tearDown() {
        database.close()
    }

    @Test
    fun registerAndLoginUser() = runBlocking {
        val user = User(name = "Demo User", email = "demo@mail.com", password = "123456")

        userDao.registerUser(user)
        val loggedInUser = userDao.loginUser("demo@mail.com", "123456")

        assertNotNull(loggedInUser)
        assertEquals("Demo User", loggedInUser?.name)
    }
}
```

## 12. Manual Testing Checklist

- Register with valid details
- Try registering with an existing email
- Login with correct credentials
- Login with wrong credentials
- Close and reopen app to confirm session remains active
- Logout and confirm session clears
- Add database records and verify data remains after app restart
- Trigger notification permission flow on Android 13+
- Verify notification appears after scheduled delay
- Test app responsiveness on a low-end emulator
- Rotate screen and check state behavior

## 13. Optimization Checklist

- Keep database calls off the main thread
- Use `StateFlow` or `LiveData` instead of manual UI polling
- Avoid memory leaks by keeping `Context` out of long-lived UI classes
- Use pagination if lists become large
- Keep images compressed and load them with Coil or Glide
- Remove unused dependencies
- Run Android Studio Profiler for CPU and memory checks

## 14. LinkedIn Post Draft

I completed Task 4 of my Android development roadmap: Advanced Features and Testing.

In this phase, I implemented user authentication, local data persistence using Room, login session handling, and app notifications with WorkManager. I also explored third-party libraries such as Room, Hilt, Coroutines, WorkManager, Mockito, and Espresso to improve structure, reliability, and maintainability.

For testing, I wrote unit tests for authentication logic and integration tests for Room database operations. I also performed manual testing for registration, login, session persistence, notifications, and app responsiveness.

This task helped me understand how production-ready Android apps combine features, testing, optimization, and clean architecture.

#AndroidDevelopment #Kotlin #RoomDatabase #MobileAppDevelopment #Testing #WorkManager #LearningJourney

