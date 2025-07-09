# Android.kt
## Kotlin Advanced

### **Extension Functions & Properties**
Add new functions or properties to existing classes without inheritance.

```kotlin
fun String.isEmail(): Boolean = contains("@") && contains(".")
val List<Int>.sumOfSquares: Int get() = sumOf { it * it }
```

**Do:**
- Use for utility methods and DSLs.
- Keep extensions focused and non-intrusive.

**Don’t:**
- Overuse for core business logic.
- Break encapsulation or rely on internals.

---

### **Higher-Order Functions & Lambdas**
Functions that take or return other functions.

```kotlin
fun <T> List<T>.customFilter(predicate: (T) -> Boolean): List<T> =
    this.filter(predicate)

val evens = listOf(1,2,3,4).customFilter { it % 2 == 0 }
```

**Do:**
- Use for callbacks, collection operations, and DSLs.
- Prefer inline for small, performance-critical lambdas.

**Don’t:**
- Capture unnecessary references in lambdas (can cause leaks).
- Overcomplicate APIs with too many function parameters.

---

### **Sealed Classes & Exhaustiveness**
Restrict class hierarchies for type-safe state and event handling.

```kotlin
sealed class Result<out T>
data class Success<T>(val data: T) : Result<T>()
data class Error(val exception: Throwable) : Result<Nothing>()
object Loading : Result<Nothing>()

fun handle(result: Result<String>) = when (result) {
    is Success -> println("Data: ${result.data}")
    is Error -> println("Error: ${result.exception}")
    Loading -> println("Loading...")
}
```

**Do:**
- Use sealed classes for state machines, events, and results.
- Always handle all cases in `when` (exhaustiveness).

**Don’t:**
- Use sealed classes for open-ended hierarchies.
- Forget to update `when` branches when adding subclasses.

---

### **Inline, Crossinline, Noinline**
Control how lambdas are inlined for performance and restrictions.

```kotlin
inline fun runTwice(block: () -> Unit) {
    block(); block()
}
```

**Do:**
- Use `inline` for small, performance-sensitive HOFs.
- Use `crossinline` to prevent non-local returns.

**Don’t:**
- Inline large functions (increases bytecode size).
- Use `noinline` unless you need to store lambdas.

---

### **Delegation**
Reuse behavior via `by` keyword.

```kotlin
interface Logger { fun log(msg: String) }
class ConsoleLogger : Logger { override fun log(msg: String) = println(msg) }
class Service(logger: Logger) : Logger by logger
```

**Do:**
- Use property delegation (`by lazy`, `by Delegates.observable`).
- Use interface delegation for composition.

**Don’t:**
- Overuse delegation for trivial cases.

---

### **Coroutines Advanced**
Use structured concurrency, channels, flows, and exception handling.

```kotlin
suspend fun fetchData(): String = withContext(Dispatchers.IO) { /* ... */ }

val channel = Channel<Int>()
GlobalScope.launch { channel.send(42) }
```

**Do:**
- Use `supervisorScope` for independent child coroutines.
- Handle exceptions with `CoroutineExceptionHandler`.

**Don’t:**
- Launch coroutines in `GlobalScope` for app logic.
- Block threads inside coroutines.

---

### **Type Aliases & Inline Classes**
Simplify complex types and add type safety.

```kotlin
typealias UserId = String

@JvmInline
value class Email(val value: String)
```

**Do:**
- Use type aliases for readability.
- Use inline/value classes for lightweight wrappers.

**Don’t:**
- Abuse type aliases for unrelated types.

---

### **Smart Casts & Contracts**
Kotlin smart-casts types after checks; contracts can help the compiler.

```kotlin
fun printLength(obj: Any) {
    if (obj is String) println(obj.length) // smart cast
}
```

**Do:**
- Use smart casts for null and type checks.
- Use contracts for custom control-flow (advanced).

**Don’t:**
- Rely on smart casts after multi-threaded changes.

---

### **Generics & Reified Types**
Use generics for type safety; `reified` for type checks at runtime.

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T =
    fromJson(json, T::class.java)
```

**Do:**
- Use `reified` for type-safe reflection.
- Use variance (`out`, `in`) for API design.

**Don’t:**
- Use star-projections (`*`) unless necessary.

---

### **Operator Overloading**
Define custom behavior for operators.

```kotlin
data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point) = Point(x + other.x, y + other.y)
}
```

**Do:**
- Overload operators only when it makes semantic sense.

**Don’t:**
- Overload for unrelated or confusing behaviors.

---

### **DSLs (Domain Specific Languages)**
Build readable APIs using lambdas with receivers.

```kotlin
fun html(block: HtmlBuilder.() -> Unit): Html = HtmlBuilder().apply(block).build()
```

**Do:**
- Use for configuration, builders, and UI.

**Don’t:**
- Overcomplicate with deeply nested DSLs.

---

### **Reflection & Annotations**
Access class metadata and custom behaviors.

```kotlin
@Target(AnnotationTarget.CLASS)
annotation class JsonSerializable

@JsonSerializable
class User(val name: String)
```

**Do:**
- Use annotations for code generation and metadata.
- Use reflection sparingly (runtime cost).

**Don’t:**
- Rely on reflection for performance-critical code.

---

### **Dos and Don’ts Summary**

| Feature                | Do                                                      | Don’t                                 |
|------------------------|---------------------------------------------------------|---------------------------------------|
| Extension Functions    | Utility, DSLs, non-intrusive                            | Core logic, break encapsulation       |
| Higher-Order Functions | Callbacks, collection ops, inline for perf              | Capture unnecessary refs, overuse     |
| Sealed Classes         | State/events, exhaustiveness in `when`                  | Open-ended hierarchies, miss cases    |
| Delegation             | Property/interface reuse                                | Trivial cases                         |
| Coroutines             | Structured concurrency, handle exceptions               | GlobalScope, thread blocking          |
| Generics/Reified       | Type safety, reflection                                 | Star-projections                      |
| Operator Overloading   | Semantic, natural operators                             | Confusing/unrelated behaviors         |
| DSLs                   | Builders, config, UI                                    | Deeply nested, unreadable             |
| Reflection/Annotations | Metadata, codegen                                       | Perf-critical code                    |

--- 
## Core Android Concepts

### **Activities**
The primary UI entry point for an app, implemented as a subclass of `Activity` (typically `AppCompatActivity`). The activity lifecycle (`onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`) dictates UI setup and resource management.

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // Initialization logic
    }
    // onStart, onResume, onPause, etc.
}
```

**Do:**
- Use `savedInstanceState` to restore UI state.
- Keep main-thread work minimal (inflate layouts, set up listeners) to avoid slow startup.

**Don’t:**
- Perform long-running operations on the UI thread (use coroutines, threads, or WorkManager).
- Leak the Activity context via static fields or long-lived references.

---

### **Fragments**
A reusable UI component with its own lifecycle, always hosted by an Activity or another Fragment. Fragments encapsulate portions of UI and can be added, replaced, or removed dynamically.

```kotlin
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Initialize views, set up observers, etc.
    }
}
```

**Do:**
- Use `childFragmentManager` for nested fragments.
- Communicate with other components via interfaces or shared `ViewModel`s.

**Don’t:**
- Add fragments directly to the back stack without user flow.
- Retain references to destroyed views (use `onDestroyView` cleanup or view binding properly).

---

### **Services**
Components for background work without a UI. Services enable your app to keep running tasks even when the user is not interacting with the app.

#### Types of Services
- **Started Service:** Launched via `startService()` or `ContextCompat.startForegroundService()`. Runs until stopped with `stopSelf()` or `stopService()`.
- **Bound Service:** Allows components to bind via `bindService()` and interact with the service through an `IBinder` interface.
- **Foreground Service:** A started service that must display a persistent notification. Required for background tasks on Android 8.0+.
- **IntentService (deprecated):** Use a regular Service with coroutines or WorkManager instead.

#### Service Lifecycle
- `onCreate()`: Called once when the service is first created.
- `onStartCommand(intent, flags, startId)`: Called each time the service is started.
- `onBind(intent)`: Called when a client binds to the service.
- `onUnbind(intent)`: Called when all clients have unbound.
- `onRebind(intent)`: Called when new clients bind after `onUnbind`.
- `onDestroy()`: Called when the service is destroyed.

#### Example: Started Service with Foreground Notification
```kotlin
class MyForegroundService : Service() {
    override fun onCreate() { /* ... */ }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = NotificationCompat.Builder(this, "channel_id")
            .setContentTitle("Service Running")
            .setSmallIcon(R.drawable.ic_service)
            .build()
        startForeground(1, notification)
        CoroutineScope(Dispatchers.IO).launch {
            doBackgroundWork()
            stopSelf()
        }
        return START_NOT_STICKY
    }

    override fun onBind(intent: Intent?): IBinder? = null
    override fun onDestroy() { /* ... */ }
}
```

#### Example: Bound Service
```kotlin
class MyBoundService : Service() {
    private val binder = LocalBinder()
    inner class LocalBinder : Binder() {
        fun getService(): MyBoundService = this@MyBoundService
    }
    override fun onBind(intent: Intent?): IBinder = binder
    fun performAction() { /* ... */ }
}
```
**Client:**
```kotlin
val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName, service: IBinder) {
        val myService = (service as MyBoundService.LocalBinder).getService()
        myService.performAction()
    }
    override fun onServiceDisconnected(name: ComponentName) {}
}
bindService(Intent(this, MyBoundService::class.java), connection, Context.BIND_AUTO_CREATE)
```

#### Foreground Service Requirements (Android 8.0+)
- Must call `startForeground()` with a notification within 5 seconds of starting.
- Declare `android:foregroundServiceType` in the manifest for specific use cases.

#### Best Practices
**Do:**
- Offload heavy work from the main thread using coroutines, threads, or executors.
- Use `WorkManager` for deferrable, guaranteed background work.
- Use foreground services for user-visible, long-running tasks.
- Handle service restarts and process death gracefully.
- Declare all services in `AndroidManifest.xml`.
- Request runtime permissions if needed.
- Stop the service with `stopSelf()` or `stopService()` when work is complete.

**Don’t:**
- Run long or blocking operations on the main thread.
- Forget to release resources in `onDestroy()`.
- Use services for tasks that can be handled by `JobIntentService` or `WorkManager`.
- Leak memory by holding references to activities or contexts.
- Start background services from the background on Android 8.0+.

#### Manifest Declaration Example
```xml
<service
    android:name=".MyForegroundService"
    android:exported="false"
    android:foregroundServiceType="location" />
```

#### Security
- Set `android:exported="false"` unless you intend for other apps to bind/start your service.
- Use permissions (`android:permission`) to restrict access if needed.

#### Alternatives
- For scheduled or deferrable work, prefer `WorkManager`.
- For short-lived background tasks, use coroutines or `ExecutorService` in ViewModel or Repository layers.

#### Summary Table

| Service Type      | Use Case                        | Threading         | Lifecycle Control         | UI?   |
|-------------------|---------------------------------|-------------------|--------------------------|-------|
| Started Service   | Ongoing background work         | Main (default)    | App or self stops        | No    |
| Foreground Service| User-visible, long-running task | Main (default)    | App or self stops        | No    |
| Bound Service     | Client-server communication     | Main (default)    | Unbinds when no clients  | No    |

---

### **Broadcast Receivers**
Components that listen for broadcasted `Intent`s (from the system or other apps). They have no UI and must be registered (statically in the manifest or dynamically in code).

```kotlin
class NetworkChangeReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val isOnline = /* check network state from intent */
        // Handle connectivity change
    }
}
```

**Do:**
- Unregister dynamic receivers in `onPause`/`onDestroy` to avoid leaks.

**Don’t:**
- Do heavy processing in `onReceive` (it times out if too slow).
- Assume order of broadcast delivery or guaranteed delivery.

---

### **Content Providers**
Components that manage structured app data and expose it via a content URI. Often backed by a database.

```kotlin
class MyProvider : ContentProvider() {
    override fun onCreate(): Boolean = true
    override fun query(
        uri: Uri, projection: Array<String>?, selection: String?, 
        selectionArgs: Array<String>?, sortOrder: String?
    ): Cursor? {
        // Perform database query and return Cursor
    }
    // Implement insert(), update(), delete(), getType()...
}
```

**Do:**
- Declare the provider in `AndroidManifest.xml` with `<provider>`, and use `content://` URIs to share data.

**Don’t:**
- Expose sensitive data without permissions.
- Block the calling process; use asynchronous operations.

---

## Jetpack Components

### **Lifecycle (AndroidX)**
Provides `LifecycleOwner` and `LifecycleObserver`/`DefaultLifecycleObserver` to react to lifecycle changes.

```kotlin
class LoggerObserver : DefaultLifecycleObserver {
    override fun onPause(owner: LifecycleOwner) {
        Log.d("Lifecycle", "Paused!")
    }
}
// In an Activity or Fragment:
lifecycle.addObserver(LoggerObserver())
```

**Do:**
- Use `onCreateView`/`onDestroyView` in Fragments for view-related setup/teardown.
- Remove observers when no longer needed.

**Don’t:**
- Leak references to the `LifecycleOwner`.

---

### **ViewModel**
Holds UI-related data that survives configuration changes.

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData<Int>(0)
    val count: LiveData<Int> = _count
    fun increment() { _count.value = (_count.value ?: 0) + 1 }
}
```

**Do:**
- Access shared ViewModels via `viewModels()` or `activityViewModels()`.
- Use `viewModelScope` for coroutines.

**Don’t:**
- Put UI references (Context, Views) in a ViewModel.
- Replace ViewModel with Singleton or global state.

---

### **LiveData**
An observable, lifecycle-aware data holder.

```kotlin
viewModel.count.observe(viewLifecycleOwner) { value ->
    textView.text = "Count: $value"
}
```

**Do:**
- Use `MutableLiveData` privately and expose immutable `LiveData`.

**Don’t:**
- Observe LiveData with non-lifecycle owners.
- Perform long tasks in `setValue`.

---

### **Navigation Component**
Simplifies in-app navigation and back-stack management.

```kotlin
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen(navController) }
    composable("details/{id}") { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id")
        DetailsScreen(id)
    }
}
```

**Do:**
- Use Safe Args to pass type-safe arguments.
- Handle Up (back) navigation with `navController.navigateUp()`.

**Don’t:**
- Mix manual `FragmentTransaction` with Navigation.
- Hardcode deep link URIs.

---

### **Room**
Jetpack persistence library that abstracts SQLite.

```kotlin
@Entity
data class User(@PrimaryKey val id: Int, val name: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM user")
    fun getAllUsers(): Flow<List<User>>
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User)
}

@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract val userDao: UserDao
}
```

**Do:**
- Expose `Flow` or `LiveData` in DAO for reactive queries.
- Use migrations or `fallbackToDestructiveMigration()` for schema changes.

**Don’t:**
- Run database operations on the main thread.
- Forget to close `Cursor`s or resources if using raw queries.

---

### **WorkManager**
Recommended library for deferrable, guaranteed background work.

```kotlin
class SyncWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        // Background sync logic
        return Result.success()
    }
}
val workRequest = OneTimeWorkRequestBuilder<SyncWorker>().build()
WorkManager.getInstance(context).enqueue(workRequest)
```

**Do:**
- Use `WorkManager` for tasks that must run even if the app or device restarts.
- Set constraints on `WorkRequest` as needed.

**Don’t:**
- Use `WorkManager` for immediate tasks.
- Forget to cancel or chain long-running work.

---

## Jetpack Compose

Jetpack Compose is Android’s modern UI toolkit using **@Composable** functions.

**Basic Usage:**
```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

**State & Recomposition:**
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Clicked $count times")
    }
}
```

**Side-Effects:**
```kotlin
@Composable
fun LoadDataEffect(viewModel: MyViewModel) {
    LaunchedEffect(Unit) {
        viewModel.loadData()
    }
}
```

**UI Architecture:**
```kotlin
class MyViewModel : ViewModel() {
    private val _text = MutableStateFlow("Hello")
    val text: StateFlow<String> = _text
}

@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    val text by viewModel.text.collectAsState()
    Text(text = text)
}
```

**Compose Best Practices:**
- Keep composables small and stateless.
- Lift state to ViewModels or caller.
- Avoid heavy logic in the UI layer.

**Dos and Don’ts:**
- **Do:** Use `Modifier` for layout, styling, and click handlers.
- **Do:** Use `@Preview` to rapidly iterate on UI.
- **Do:** Use `rememberSaveable` for state that should survive process death.
- **Don’t:** Perform side-effects or I/O directly in composable bodies.
- **Don’t:** Over-nest composables excessively.

---

## Coroutines and Concurrency

Kotlin **Coroutines** provide asynchronous programming with structured concurrency.

**Structured Concurrency:**  
Coroutines are launched in a `CoroutineScope`, and child coroutines are automatically canceled when the scope ends.

**Coroutine Scopes:**  
Common scopes include `MainScope()`, `viewModelScope`, `lifecycleScope`, `IO` or `Default` (via `Dispatchers`).

**Best Practice:**  
Avoid `GlobalScope`. Inject and manage scopes via architecture.

**Flow:**  
A cold asynchronous stream of values.

```kotlin
val flow: Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)
    }
}
viewModelScope.launch {
    flow.collect { value ->
        println("Received $value")
    }
}
```

**Channels:**  
A coroutine-safe queue for passing values between coroutines.

```kotlin
val channel = Channel<Int>()
launch { for (i in 1..5) channel.send(i); channel.close() }
launch { for (value in channel) println("Got $value") }
```

**Best Practices:**
- Inject `CoroutineDispatcher`.
- Make public APIs main-safe.
- Only use `runBlocking` in tests or main functions.
- Handle exceptions with structured coroutine builders.

---

## Dependency Injection (Hilt & Dagger)

**Dagger:**  
A compile-time DI framework using `@Module`, `@Component`, and `@Inject`.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder().build()
}
```

**Hilt:**  
A Jetpack library built on Dagger that reduces boilerplate.

```kotlin
@HiltAndroidApp
class MyApp : Application()

@AndroidEntryPoint
class MainActivity : AppCompatActivity() { ... }

class Repo @Inject constructor(private val api: ApiService) { ... }
```

**Do:**
- Prefer Hilt for Android projects.
- Use constructor injection.

**Don’t:**
- Create your own singletons manually.
- Mix Hilt and manual Dagger components without careful design.

---

## Architecture Patterns

### **MVVM (Model-View-ViewModel)**
Separates UI (View) from business logic (ViewModel) and data (Model).

- **View:** Activities/Fragments observing `LiveData`/`StateFlow`.
- **ViewModel:** Holds UI state and calls use-cases or repository methods.
- **Model (Repository):** Manages data (network, database).

---

### **MVI (Model-View-Intent)**
Emphasizes unidirectional data flow.

```kotlin
sealed class MyIntent { object LoadData : MyIntent() }
data class MyState(val loading: Boolean = false, val items: List<String> = emptyList())

class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(MyState())
    val state: StateFlow<MyState> = _state

    fun processIntent(intent: MyIntent) {
        when (intent) {
            is MyIntent.LoadData -> loadData()
        }
    }

    private fun loadData() = viewModelScope.launch {
        _state.value = MyState(loading = true)
        val result = repository.getItems()
        _state.value = MyState(items = result)
    }
}
```

**Do:**
- Keep state immutable and emit new state on each update.
- Render UI purely from the state.

**Don’t:**
- Have multiple independent sources of truth.

---

### **Clean Architecture**
Organizes code into layers with strict dependencies.

```
Presentation ---> Domain ---> (Interfaces) ---> Data
```

**Example:**
```kotlin
// Domain layer
class FetchUsersUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(): List<User> = repo.getAllUsers()
}

// Presentation layer (ViewModel)
class UserViewModel(private val fetchUsers: FetchUsersUseCase): ViewModel() {
    val users = liveData { emit(fetchUsers()) }
}

// Data layer (implements UserRepository)
class UserRepositoryImpl : UserRepository {
    override suspend fun getAllUsers() = userDao.getAll()
}
```

**Do:**
- Keep domain layer free of Android or framework libraries.
- Use interfaces for repository contracts.

**Don’t:**
- Allow data or presentation to access Android APIs in the domain layer.
- Bypass use-cases.

---

## Modularization

Modularize large codebases into Gradle modules (features, libraries) to improve build performance and code organization.

**Setup:**  
Use “New Module” to create `:core`, `:feature:login`, etc.

**Dynamic Feature Modules:**  
With Android App Bundles, you can deliver modules on-demand.

```xml
<dist:module
    dist:instant="false"
    dist:title="@string/title_profile">
    <dist:delivery>
        <dist:onDemand />
    </dist:delivery>
</dist:module>
```

**Do:**
- Enforce encapsulation.
- Use Gradle composite builds or version catalogs.

**Don’t:**
- Over-modularize.
- Duplicate code across modules.

---

## Performance

### **Memory Management**
Use the Android Studio **Memory Profiler** to track allocations and leaks.

```kotlin
override fun onTrimMemory(level: Int) {
    if (level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN) {
        // Release UI-related memory
    }
    if (level >= ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
        // Release background processing memory
    }
}
```

**Do:**
- Use `ApplicationContext` where appropriate.
- Utilize tools like LeakCanary.

**Don’t:**
- Cache large objects or bitmaps without clearing them.
- Use too much static in Android.

---

### **UI Rendering**
Diagnose slow UI frames with **Profile GPU Rendering** and **Layout Inspector**.

**Tips:**
- Use flatter view hierarchies.
- Offload complex rendering to the GPU.
- Ensure `android:hardwareAccelerated="true"`.

---

### **Startup Time**
Minimize cold start time.

**Tools:**
- Android Studio **Startup Profiler** or **Perfetto**.
- Enable `reportFullyDrawn()`.

**Do:**
- Delay non-critical work to after launch.

**Don’t:**
- Do disk or network I/O synchronously at startup.

---

## Custom Views and Animations

### **Custom Views**
Override drawing methods for unique UI components.

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : View(context, attrs) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply { color = Color.BLUE }
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        val radius = min(width, height) / 2f
        canvas.drawCircle(width / 2f, height / 2f, radius, paint)
    }
}
```

**Do:**
- Handle padding and `onMeasure` if needed.
- Use `invalidate()` to request redraw.

**Don’t:**
- Allocate new objects inside `onDraw`.
- Block the UI in custom view drawing.

---

### **Animations**
Use property animation APIs for view animations.

```kotlin
ObjectAnimator.ofFloat(myView, "alpha", 1f, 0f).apply {
    duration = 500L
    start()
}
```

**Do:**
- Use hardware-accelerated properties.
- Use `AnimatorListener` or `doOnEnd` to trigger logic after animation.

**Don’t:**
- Animate layout params on the main thread in a loop.
- Forget to `.cancel()` animators.

---

## System Internals

### **App Process & Lifecycle**
- **Process Model:**  
    - Each Android app runs in its own Linux process, isolated from other apps for security and stability.
    - The process is started by the system when any component (Activity, Service, BroadcastReceiver, ContentProvider) is needed.
    - By default, all app components run in the main process, but you can specify a different process in the manifest using `android:process`.
- **Application Object:**  
    - The `Application` class is instantiated before any other class when the process starts.
    - Override `onCreate()` for global initialization (dependency injection, logging, analytics).
    - Only one `Application` instance exists per process.
- **Component Lifecycle:**  
    - Activities, Services, etc., are created and destroyed by the system as needed.
    - The process may be killed at any time to reclaim resources; always save state in `onSaveInstanceState` or persistent storage.
- **Process Death:**  
    - The system may kill the process to free memory (especially in the background).
    - Use persistent storage (SharedPreferences, database) for critical data.
    - Avoid holding onto static references that may leak memory across process restarts.

---

### **Looper & Handler**
- **Main Thread (UI Thread):**  
    - All UI operations must occur on the main thread.
    - The main thread has a `Looper` and a message queue that processes messages and runnables sequentially.
- **Looper:**  
    - A `Looper` manages a message queue for a thread.
    - The main thread’s `Looper` is prepared automatically; for background threads, call `Looper.prepare()` and `Looper.loop()`.
- **Handler:**  
    - A `Handler` posts messages and runnables to a thread’s message queue.
    - Use `Handler(Looper.getMainLooper())` to post work to the main thread from any thread.
    - `post()`, `postDelayed()`, `sendMessage()` are common methods.
- **HandlerThread:**  
    - A `HandlerThread` is a thread with its own `Looper`.
    - Useful for background work that needs to process messages sequentially.
    - Always quit the `HandlerThread` when done to avoid leaks.

```kotlin
val handler = Handler(Looper.getMainLooper())
handler.postDelayed({ textView.text = "Hello" }, 1000L)
```

**Do:**
- Use `HandlerThread` for background threads that need a message queue.
- Use `Handler` to schedule work on the main thread or a background thread with a Looper.
- Remove callbacks and messages (`handler.removeCallbacksAndMessages(null)`) when no longer needed to avoid leaks.

**Don’t:**
- Touch or update views from a non-main thread (causes exceptions and undefined behavior).
- Block the main thread’s `Looper` (e.g., with long-running operations or infinite loops).
- Forget to quit background `Looper`s when done.

---

### **Binder & AIDL**
- **Binder:**  
    - The core IPC (Inter-Process Communication) mechanism in Android.
    - Enables communication between different processes (e.g., app and system services, or two apps).
    - Used internally for system services (e.g., ActivityManager, PackageManager).
- **AIDL (Android Interface Definition Language):**  
    - Used to define the interface for remote services.
    - Generates code to marshal/unmarshal data across process boundaries.
    - Only needed for IPC between different processes; for in-process communication, use interfaces or direct method calls.
- **How Binder Works:**  
    - Calls are marshaled into parcels and sent to the target process.
    - The receiving process handles the call on a thread from the Binder thread pool (not the main thread).
    - Be careful with thread safety and avoid blocking Binder threads.
- **Security:**  
    - Use permissions in the manifest to restrict which apps can bind to your service.
    - Validate all incoming data in AIDL methods.

**Key Points:**
- Use AIDL only when you need to communicate across processes.
- All AIDL calls are asynchronous and may come in on any thread (typically a Binder thread).
- Avoid heavy work in AIDL methods; offload to background threads if needed.

---

### **Memory & Threads**
- **Memory Management:**  
    - Android uses the ART (Android Runtime) garbage collector to manage memory.
    - Memory is limited per app; exceeding limits causes `OutOfMemoryError`.
    - Use tools like Android Studio Profiler and LeakCanary to detect leaks.
- **Threading:**  
    - The main thread handles UI and must not be blocked.
    - Use background threads for disk, network, or CPU-intensive work.
    - Options: `Thread`, `HandlerThread`, `Executors`, Kotlin coroutines (`Dispatchers.IO`, `Dispatchers.Default`), RxJava.
- **Thread Pooling:**  
    - Use `Executors.newFixedThreadPool()` or coroutines to limit concurrency.
    - 2–4 threads are usually sufficient for intensive work; too many threads cause context switching overhead and memory pressure.
- **Best Practices:**  
    - Use `Executors.newSingleThreadExecutor()` for serialized background tasks.
    - Use `Dispatchers.IO` for disk/network I/O in coroutines.
    - Always shut down executors and threads when done.
    - Avoid unbounded thread creation (can exhaust system resources).
    - Do not assume objects are tied to a specific thread unless documented.
    - Use synchronization or thread-safe data structures when sharing data across threads.

**Do:**
- Limit the number of concurrent threads for background work.
- Use thread pools or coroutines for efficient concurrency.
- Profile memory and thread usage regularly.

**Don’t:**
- Spawn unbounded or excessive threads.
- Block the main thread with heavy work.
- Assume thread-affinity for objects unless explicitly required (e.g., Views must be accessed on the main thread).

---

---

## Networking (Retrofit & OkHttp)

### **Retrofit**
A type-safe HTTP client.

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): Response<User>
}

val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(MoshiConverterFactory.create())
    .build()

val api = retrofit.create(ApiService::class.java)
```

**Do:**
- Use `suspend` functions.
- Always call network APIs off the main thread.

**Don’t:**
- Forget to check `Response.isSuccessful`.
- Keep API keys or secrets in code.

---

### **OkHttp**
The HTTP client used by Retrofit.

```kotlin
val logging = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY
}
val client = OkHttpClient.Builder()
    .addInterceptor(logging)
    .readTimeout(30, TimeUnit.SECONDS)
    .build()
```

**Do:**
- Use connection pooling, caching, and timeouts.
- Handle exceptions.

**Don’t:**
- Perform retries without backoff.
- Ignore SSL issues.

---

## Testing

### **Unit Testing**
Test business logic with JUnit and mocking.

```kotlin
@Test
fun `loadUsers updates live data`() = runTest {
    coEvery { repository.getUsers() } returns listOf(User(1, "Alice"))
    val viewModel = UserViewModel(repository)
    viewModel.loadUsers()
    assertEquals(listOf(User(1, "Alice")), viewModel.users.value)
}
```

**Do:**
- Mock dependencies.
- Use `@Before` for setup and `@After` for cleanup.

**Don’t:**
- Hit the real database or network in unit tests.

---

### **Instrumentation (Android) Tests**
Use AndroidX Test with Espresso for UI.

```kotlin
@Test
fun clickButton_opensDetail() {
    onView(withId(R.id.buttonNext)).perform(click())
    onView(withText("Details Page")).check(matches(isDisplayed()))
}
```

---

### **Compose UI Tests**
Use `createComposeRule()` and `onNode` assertions.

```kotlin
@get:Rule val composeTestRule = createComposeRule()

@Test
fun testGreetingDisplay() {
    composeTestRule.setContent {
        Greeting("Android")
    }
    composeTestRule.onNodeWithText("Hello, Android!").assertIsDisplayed()
}
```

---

### **Flow Testing**
Use `runTest` and collect Flows. The Turbine library simplifies this.

```kotlin
@Test
fun testFlowEmitsValues() = runTest {
    val fakeRepo = FakeRepository()
    val viewModel = MyViewModel(fakeRepo)
    viewModel.scores().test {
        fakeRepo.emit(1)
        assertEquals(10, awaitItem())
        fakeRepo.emit(2)
        assertEquals(20, awaitItem())
        cancelAndIgnoreRemainingEvents()
    }
}
```

**Do:**
- Test coroutine code with test dispatchers.
- Use Turbine or `toList()` in `runTest` for collecting Flows.

**Don’t:**
- Rely on timing; make tests deterministic.
- Test private methods directly.

---

## Security

### **Secure Storage**
Do **not** store sensitive data in plaintext `SharedPreferences` or files. Use Android Keystore and AndroidX Security libraries.

- **Android Keystore:** Store cryptographic keys so they are non-exportable.
- **EncryptedSharedPreferences:** Uses keystore-backed keys to encrypt preferences.

---

### **Obfuscation**

Obfuscation helps protect your app from reverse engineering by renaming classes, methods, and fields, removing unused code, and optimizing bytecode. Android uses **R8** (default) or **ProGuard** for code shrinking and obfuscation.

#### Enabling R8/ProGuard

Enable R8/ProGuard for release builds in your `build.gradle`:

```gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}
```

#### ProGuard Rules File (`proguard-rules.pro`)

Customize obfuscation and shrinking with rules in `proguard-rules.pro`.

**Common Rules:**

- **Keep Application Classes:**
    ```proguard
    -keep class com.example.app.** { *; }
    ```

- **Keep All Activities, Services, BroadcastReceivers, ContentProviders:**
    ```proguard
    -keep public class * extends android.app.Activity
    -keep public class * extends android.app.Service
    -keep public class * extends android.content.BroadcastReceiver
    -keep public class * extends android.content.ContentProvider
    ```

- **Keep Custom Views (used in XML):**
    ```proguard
    -keep public class * extends android.view.View { public <init>(android.content.Context, android.util.AttributeSet); }
    ```

- **Keep Serializable/Parcelable Models:**
    ```proguard
    -keepclassmembers class * implements java.io.Serializable {
        static final long serialVersionUID;
        private static final java.io.ObjectStreamField[] serialPersistentFields;
        private void writeObject(java.io.ObjectOutputStream);
        private void readObject(java.io.ObjectInputStream);
        java.lang.Object writeReplace();
        java.lang.Object readResolve();
    }
    -keep class * implements android.os.Parcelable { *;

---

### **NDK (Native Code)**
Native code can also be reverse-engineered.

**Do:**
- Avoid storing secrets in native code or resources.

---

### **Network Security**
Always use HTTPS/TLS. Use **Network Security Config** to enforce security policies.

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
    </domain-config>
</network-security-config>
```

**Do:**
- Update `android:networkSecurityConfig` in the Manifest.
- Use certificate pinning for extra security.

**Don’t:**
- Trust user-added CAs or allow cleartext.

---

## Gradle Build System

### **Build Optimization**
Follow Android’s official tips for faster builds.

- Enable Gradle Daemon and parallel builds in `gradle.properties`.
- Use KSP instead of Kapt.
- Avoid unnecessary resources.
- Keep Android Gradle Plugin and dependencies up to date.

---

### **Custom Gradle Tasks**
Write tasks in Kotlin DSL.

```kotlin
tasks.register("printVersion") {
    doLast {
        println("App Version: ${project.properties["VERSION_NAME"]}")
    }
}
```

---

### **Build Variants**
Define `buildTypes` and `productFlavors` in Gradle.

```gradle
android {
    defaultConfig { applicationId "com.example.app" }
    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
        debug { }
    }
    flavorDimensions "tier"
    productFlavors {
        demo { dimension "tier"; applicationIdSuffix ".demo" }
        full { dimension "tier" }
    }
}
```

**Do:**
- Use `applicationIdSuffix` or `versionNameSuffix` to distinguish flavors.

---

## Google Play Store

### **Publishing**
Build an **Android App Bundle (AAB)** for release. Use **Play App Signing**.

**Signing:**  
Upload a cryptographic key to Play Console or enroll in Google Play App Signing.

**Versioning:**  
Follow semantic versioning with `versionCode` and `versionName` in Gradle.

---

### **Play Integrity API**
Integrate the Play Integrity API to ensure your app is genuine.

---

### **Feature Delivery**
Use **Dynamic Feature Modules** for on-demand or conditional delivery.

---

**Do:** Test your release build and signing process.  
**Don’t:** Release debug builds or un-optimized APKs.  
**Do:** Monitor **Android Vitals** in Play Console.  
**Don’t:** Violate Play policies.

---

**Sources:** Android official documentation and developer guides.
