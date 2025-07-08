# Android.kt

## Core Android Concepts

**Activities:** The primary UI entry point for an app, each implemented as a subclass of `Activity` (typically `AppCompatActivity`). The activity lifecycle (`onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`) dictates UI setup and resource management. For example:

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

* **Do:** Use `savedInstanceState` to restore UI state.
* **Do:** Keep the main-thread work minimal (inflate layouts, set up listeners) to avoid slow startup.
* **Don’t:** Perform long-running operations on the UI thread (use coroutines, threads, or WorkManager instead).
* **Don’t:** Leak the Activity context via static fields or long-lived references.

**Fragments:** A reusable UI component with its own lifecycle, always hosted by an Activity or another Fragment. Fragments encapsulate portions of UI and can be added, replaced, or removed dynamically. For example:

```kotlin
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Initialize views, set up observers, etc.
    }
}
```

* **Do:** Use `childFragmentManager` for nested fragments.
* **Do:** Communicate with other components via interfaces or shared `ViewModel`s.
* **Don’t:** Add fragments directly to the back stack without user flow.
* **Don’t:** Retain references to destroyed views (use `onDestroyView` cleanup or view binding properly).

**Services:** Components for background work without UI. A `Service` runs on the main thread by default (use a separate thread within it if needed). Android provides started services (`startService`/`stopService`) and bound services (`bindService`). For example:

```kotlin
class MyService : Service() {
    override fun onBind(intent: Intent?): IBinder? {
        return null  // Return binder for bound service, or null for started service
    }
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Perform background work, e.g. start a coroutine or thread
        return START_NOT_STICKY
    }
}
```

* **Do:** Call `stopSelf()` or `stopService()` when done.
* **Don’t:** Run heavy work on the main thread; use `CoroutineScope` or `Thread`.
* **Don’t:** Forget to declare services in `AndroidManifest.xml`.

**Broadcast Receivers:** Components that listen for broadcasted `Intent`s (from the system or other apps). They have no UI and must be registered (statically in the manifest or dynamically in code). Example:

```kotlin
class NetworkChangeReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val isOnline = /* check network state from intent */
        // Handle connectivity change
    }
}
```

* **Do:** Unregister dynamic receivers in `onPause`/`onDestroy` to avoid leaks.
* **Don’t:** Do heavy processing in `onReceive` (it times out if too slow) – start a Service or use WorkManager instead.
* **Don’t:** Assume order of broadcast delivery or guaranteed delivery – code defensively.

**Content Providers:** Components that manage structured app data and expose it via a content URI. Often backed by a database, they allow other apps to query or modify the data with proper permissions. Example stub:

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

* **Do:** Declare the provider in `AndroidManifest.xml` with `<provider>`, and use `content://` URIs to share data.
* **Don’t:** Expose sensitive data without permissions.
* **Don’t:** Block the calling process; use asynchronous operations (e.g. return a cursor quickly or use `CursorLoader`).

## Jetpack Components

**Lifecycle (AndroidX):** The Lifecycle library provides `LifecycleOwner` (e.g. `Fragment`/`ComponentActivity`) and `LifecycleObserver`/`DefaultLifecycleObserver` to react to lifecycle changes without leaking resources. Use it to tie resources to lifecycle events. For example:

```kotlin
class LoggerObserver : DefaultLifecycleObserver {
    override fun onPause(owner: LifecycleOwner) {
        Log.d("Lifecycle", "Paused!")
    }
}
// In an Activity or Fragment:
lifecycle.addObserver(LoggerObserver())
```

* **Do:** Use `onCreateView`/`onDestroyView` in Fragments for view-related setup/teardown.
* **Do:** Remove observers when no longer needed (though Lifecycle handles this for `DefaultLifecycleObserver`).
* **Don’t:** Leak references to the `LifecycleOwner` (avoid static observers holding the owner).

**ViewModel:** Holds UI-related data that survives configuration changes. It separates UI logic from Activities/Fragments. Example ViewModel:

```kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData<Int>(0)
    val count: LiveData<Int> = _count

    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}
```

* **Do:** Access shared ViewModels via `viewModels()` or `activityViewModels()` delegates.
* **Do:** Use `viewModelScope` to launch coroutines tied to ViewModel's lifecycle.
* **Don’t:** Put UI references (Context, Views) in a ViewModel – store UI state instead.
* **Don’t:** Try to replace ViewModel with Singleton or global state (it’s scoped to UI controllers).

**LiveData:** An observable, lifecycle-aware data holder. Observers (Activities/Fragments) get updates only when in active states, preventing leaks. Example usage:

```kotlin
viewModel.count.observe(viewLifecycleOwner) { value ->
    textView.text = "Count: $value"
}
```

* **Do:** Use `MutableLiveData` privately and expose immutable `LiveData`.
* **Don’t:** Observe LiveData with non-lifecycle owners or after `onDestroy` (will auto-unregister anyway).
* **Don’t:** Perform long tasks in `setValue`; use coroutines to update LiveData.

**Navigation Component:** Simplifies in-app navigation and back-stack management. It uses a `NavHostFragment` or `NavHost` in Compose with a navigation graph (`nav_graph.xml`). Example in Compose:

```kotlin
NavHost(navController, startDestination = "home") {
    composable("home") { HomeScreen(navController) }
    composable("details/{id}") { backStackEntry ->
        val id = backStackEntry.arguments?.getString("id")
        DetailsScreen(id)
    }
}
```

* **Do:** Use Safe Args to pass type-safe arguments between destinations.
* **Do:** Handle Up (back) navigation with `navController.navigateUp()`.
* **Don’t:** Mix manual `FragmentTransaction` with Navigation – let the NavController manage the stack.
* **Don’t:** Hardcode deep link URIs; configure them in the nav graph.

**Room:** The Jetpack persistence library that abstracts SQLite with compile-time safety. Define `@Entity` data classes, `@Dao` interfaces for queries, and a `RoomDatabase`. Example:

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

* **Do:** Expose `Flow` or `LiveData` in DAO for reactive queries.
* **Do:** Use migrations or `fallbackToDestructiveMigration()` for schema changes.
* **Don’t:** Run database operations on the main thread (Room enforces this by default).
* **Don’t:** Forget to close `Cursor`s or resources if using raw queries.

**WorkManager:** The recommended library for deferrable, guaranteed background work. Define `Worker` classes and enqueue requests:

```kotlin
class SyncWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        // Background sync logic
        return Result.success()
    }
}

// Enqueue work
val workRequest = OneTimeWorkRequestBuilder<SyncWorker>().build()
WorkManager.getInstance(context).enqueue(workRequest)
```

* **Do:** Use `WorkManager` for tasks that must run even if the app or device restarts.
* **Do:** Set constraints (e.g. network, charging) on `WorkRequest` as needed.
* **Don’t:** Use `WorkManager` for immediate tasks; for real-time work prefer coroutines or threads.
* **Don’t:** Forget to cancel or chain long-running work to avoid redundant tasks.

## Jetpack Compose

Jetpack Compose is Android’s modern UI toolkit using **@Composable** functions. It uses declarative code and automatically handles UI updates via recomposition.

* **Basic Usage:** Define composable functions annotated with `@Composable`. They can call other composables and must only be called from a composable context. Example:

  ```kotlin
  @Composable
  fun Greeting(name: String) {
      Text(text = "Hello, $name!")
  }
  ```

* **State & Recomposition:** Hold local UI state with `remember { mutableStateOf(...) }`. Compose automatically re-runs (recomposes) functions when state changes. Example:

  ```kotlin
  @Composable
  fun Counter() {
      var count by remember { mutableStateOf(0) }
      Button(onClick = { count++ }) {
          Text("Clicked $count times")
      }
  }
  ```

  The `remember` and `mutableStateOf` track state; changing `count` causes the composable to update. Compose encourages **state hoisting**: keep composables stateless and pass state/data from parents.

* **Side-Effects:** Use Compose effect APIs for non-UI work. For example, `LaunchedEffect(key1) { ... }` runs a suspend block when keys change; `SideEffect { ... }` runs after every successful recomposition. These allow integrating coroutines or external state changes safely.

  ```kotlin
  @Composable
  fun LoadDataEffect(viewModel: MyViewModel) {
      LaunchedEffect(Unit) {
          viewModel.loadData()
      }
  }
  ```

* **UI Architecture:** In Compose, follow a unidirectional data flow. ViewModels expose `StateFlow` or `LiveData`, which can be observed via `collectAsState()` or `observeAsState()`. Example using `StateFlow`:

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

* **Compose Best Practices:** Keep composables small and stateless, lift state to ViewModels or caller, and avoid heavy logic in the UI layer. Compose handles recomposition efficiently, but unnecessary recompositions can be avoided by keying state and using `remember` appropriately.

* **Dos and Don’ts:**

  * **Do:** Use `Modifier` for layout, styling, and click handlers.
  * **Do:** Use `@Preview` to rapidly iterate on UI.
  * **Do:** Use `rememberSaveable` for state that should survive process death (ViewModel handles most other state).
  * **Don’t:** Perform side-effects or I/O directly in composable bodies (use effects or ViewModel).
  * **Don’t:** Over-nest composables excessively; break into reusable pieces to optimize performance.

## Coroutines and Concurrency

Kotlin **Coroutines** provide asynchronous programming with structured concurrency. Key concepts:

* **Structured Concurrency:** Coroutines are launched in a `CoroutineScope`, and child coroutines are automatically canceled when the scope ends. This prevents leaks and lost coroutines. For example, `viewModelScope.launch { ... }` ties work to the ViewModel lifecycle. Use `coroutineScope { ... }` or `supervisorScope { ... }` in suspend functions to run parallel tasks safely.

* **Coroutine Scopes:** Common scopes include `MainScope()`, `GlobalScope` (discouraged), `viewModelScope`, `lifecycleScope`, `IO` or `Default` (via `Dispatchers`).

  * **Best Practice:** Avoid `GlobalScope`. Instead, inject and manage scopes via architecture (e.g. use `viewModelScope` in ViewModels).
  * **Don’t:** Hardcode dispatchers in library code; instead, inject `CoroutineDispatcher` (see Kotlin docs).
  * **Don’t:** Block the main thread; use `withContext(Dispatchers.IO) { ... }` for blocking I/O.

* **Flow:** A cold asynchronous stream of values (part of Kotlinx.coroutines). Use `flow { emit(...) }` to create flows and `collect` to consume. Flows are sequential and suspend between emissions. Example:

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

  Use `StateFlow`/`SharedFlow` for hot, stateful streams. They can be collected in Composables or LiveData.

* **Channels:** A channel is a coroutine-safe queue for passing values between coroutines. One coroutine can send values and another can receive. Example:

  ```kotlin
  val channel = Channel<Int>()
  // Producer
  launch {
      for (i in 1..5) channel.send(i)
      channel.close()
  }
  // Consumer
  launch {
      for (value in channel) {
          println("Got $value")
      }
  }
  ```

  Use channels for explicit producer-consumer patterns or ConflatedBroadcastChannel/SharedFlow for broadcasts.

* **Best Practices:** Follow official guidelines for coroutine use:

  * Inject `CoroutineDispatcher` (avoid `Dispatchers.IO` hardcoded in libraries).
  * Make public APIs **main-safe**: if they do blocking work, use `withContext(Dispatchers.IO)` internally.
  * Only use `runBlocking` in tests or main functions, not in production code.
  * Handle exceptions with structured coroutine builders (`supervisorScope` or `CoroutineExceptionHandler`).

## Dependency Injection (Hilt & Dagger)

Use **Dependency Injection** to manage complex app dependencies:

* **Dagger:** A compile-time DI framework using `@Module`, `@Component`, and `@Inject`. You declare how to provide objects in `@Module` classes, e.g.:

  ```kotlin
  @Module
  @InstallIn(SingletonComponent::class)
  object NetworkModule {
      @Provides fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder().build()
  }
  ```

  Dagger generates code to assemble these dependencies.

* **Hilt:** A Jetpack library built on Dagger that reduces boilerplate. Annotate your `Application` with `@HiltAndroidApp`, Activities/Fragments with `@AndroidEntryPoint`, and use `@Inject` in ViewModels or classes. Hilt automatically creates Dagger components for Android classes:

  ```kotlin
  @HiltAndroidApp
  class MyApp : Application()

  @AndroidEntryPoint
  class MainActivity : AppCompatActivity() { ... }

  class Repo @Inject constructor(private val api: ApiService) { ... }
  ```

  * **Do:** Prefer Hilt for Android projects for simpler setup and lifecycle-aware injection.
  * **Do:** Use constructor injection (`@Inject constructor(...)`) whenever possible.
  * **Don’t:** Create your own singletons manually; let Hilt manage scope.
  * **Don’t:** Mix Hilt and manual Dagger components in the same codebase without careful design – Hilt expects control of the DI graph.

## Architecture Patterns

**MVVM (Model-View-ViewModel):** Separates UI (View) from business logic (ViewModel) and data (Model).

* **View:** Activities/Fragments (stateless UI) observing `LiveData`/`StateFlow` from ViewModel.
* **ViewModel:** Holds UI state and calls use-cases or repository methods.
* **Model (Repository):** Manages data (network, database) and domain models.

Example flow: ViewModel calls a repository, updates `LiveData`, UI observes and updates.

**MVI (Model-View-Intent):** Emphasizes unidirectional data flow. Entire UI state is represented as an immutable data class. User actions are sent as “intents” to the ViewModel, which produces a new state. Example skeleton:

```kotlin
// Intent (user action)
sealed class MyIntent { object LoadData : MyIntent() }

// State
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

* **Do:** Keep state immutable and emit new state on each update.
* **Do:** Render UI purely from the state (`collectAsState()` in Compose, `observe` in XML).
* **Don’t:** Have multiple independent sources of truth; keep one authoritative state object.

**Clean Architecture:** Organizes code into layers with strict dependencies. Typically:

* **Domain Layer:** Pure Kotlin module (no Android deps) containing business logic (entities, use-cases). Does **not** depend on data or presentation.
* **Data Layer:** Implements repositories and data sources (network, DB), depends on domain interfaces.
* **Presentation Layer:** UI (Activities/Fragments/Compose) and ViewModels, depends on domain (use-cases) but not data layer.

```
Presentation ---> Domain ---> (Interfaces) ---> Data
```

Example (using a use-case):

```kotlin
// Domain layer
class FetchUsersUseCase(private val repo: UserRepository) {
    suspend operator fun invoke(): List<User> = repo.getAllUsers()
}

// Presentation layer (ViewModel)
class UserViewModel(private val fetchUsers: FetchUsersUseCase): ViewModel() {
    val users = liveData {
        emit(fetchUsers())
    }
}

// Data layer (implements UserRepository)
class UserRepositoryImpl : UserRepository {
    override suspend fun getAllUsers() = userDao.getAll()
}
```

* **Do:** Keep domain layer free of Android or framework libraries.
* **Do:** Use interfaces for repository contracts.
* **Don’t:** Allow data or presentation to access Android APIs in the domain layer.
* **Don’t:** Bypass use-cases; keep logic out of Activities/Fragments.

## Modularization

Modularize large codebases into Gradle modules (features, libraries) to improve build performance and code organization. Each module is a self-contained Gradle project with its own code and dependencies. Benefits include improved **reusability**, strict dependency control, and parallel builds.

* **Setup:** In Android Studio, use “New Module” to create `:core`, `:feature:login`, etc. Move related code (UI, domain, data) into feature modules. Example `settings` module could be an Android library or dynamic feature.
* **Dynamic Feature Modules:** With Android App Bundles, you can deliver modules on-demand. Mark feature modules with `dist:module` in `AndroidManifest.xml`. This lets you split the APK so rarely-used features don’t bloat the initial install.

  ```xml
  <dist:module
      dist:instant="false"
      dist:title="@string/title_profile">
      <dist:delivery>
          <dist:onDemand />
      </dist:delivery>
  </dist:module>
  ```
* **Do:** Enforce encapsulation: make module APIs minimal and mark classes `internal` if they shouldn’t be public.
* **Do:** Use Gradle composite builds or version catalogs for cross-module dependencies management.
* **Don’t:** Over-modularize (avoid hundreds of tiny modules). Balance granularity – too fine increases build complexity.
* **Don’t:** Duplicate code across modules; shared logic should live in a common library module.

## Performance

**Memory Management:** Use the Android Studio **Memory Profiler** to track allocations and leaks. Avoid memory leaks (e.g. static references to `Context` or `Activity`). Implement `onTrimMemory()` in Activities to release heavy resources when the app goes background:

```kotlin
override fun onTrimMemory(level: Int) {
    if (level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN) {
        // Release UI-related memory (bitmaps, caches)
    }
    if (level >= ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
        // Release background processing memory (close DB, large data structures)
    }
}
```

* **Do:** Use `ApplicationContext` where appropriate (e.g. `applicationContext` vs `activity`).
* **Do:** Utilize tools like LeakCanary to catch leaks in development.
* **Don’t:** Cache large objects or bitmaps without clearing them.
* **Don’t:** Use too much static in Android (it ties objects to app lifetime).

**UI Rendering:** Diagnose slow UI frames with tools: **Profile GPU Rendering** and **Layout Inspector**. Overdraw (drawing the same pixel multiple times) wastes GPU time. Use the Debug GPU Overdraw visualization in Developer Options to spot it. Optimization tips:

* Use flatter view hierarchies (ConstraintLayout, Compose) and remove unnecessary backgrounds.
* Offload complex rendering (animations, bitmaps) to the GPU or use `RenderScript`/`GPU` where applicable.
* **Do:** Ensure `android:hardwareAccelerated="true"` (default) for smooth drawing.
* **Don’t:** Overuse `layout_weight` or deep nested layouts.

**Startup Time:** Cold start time (to first frame) should be minimized.  Loading/inflating views in `Activity.onCreate()` is the heaviest operation, so keep it lean. Use `Application.onCreate()` only for essential init. Tools:

* Android Studio **Startup Profiler** or **Perfetto** to trace the startup path.
* Enable `reportFullyDrawn()` to measure time to full UI.
* **Do:** Delay non-critical work (e.g. analytics init) to after launch.
* **Don’t:** Do disk or network I/O synchronously at startup.
* **Don’t:** Load huge resources or layouts on main thread without optimization.

## Custom Views and Animations

When you need a unique UI component or performance beyond XML views, implement a custom `View` by overriding drawing methods. Key steps:

* Override `onDraw(canvas: Canvas)` and use `Paint` and `Canvas` to draw primitives (circles, text, paths).
* Pre-create expensive objects (e.g. `Paint`, `Path`) in initialization to avoid recreating them on every draw.

Example custom view drawing a circle:

```kotlin
class CircleView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : View(context, attrs) {
    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLUE
    }
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        val radius = min(width, height) / 2f
        canvas.drawCircle(width / 2f, height / 2f, radius, paint)
    }
}
```

* **Do:** Handle padding and `onMeasure` if needed for `wrap_content`.
* **Do:** Use `invalidate()` to request redraw when state changes.
* **Don’t:** Allocate new objects (e.g. `new Paint()`, `new Path()`) inside `onDraw` each frame – reuse them.
* **Don’t:** Block the UI in custom view drawing; keep `onDraw` efficient.

**Animations:** Use Android’s property animation APIs (`ObjectAnimator`, `ValueAnimator`, `ViewPropertyAnimator`) for view animations. Example fade-out:

```kotlin
ObjectAnimator.ofFloat(myView, "alpha", 1f, 0f).apply {
    duration = 500L
    start()
}
```

For layout transitions, consider `MotionLayout` (in ConstraintLayout) for rich animations via XML. In Compose, use `animate*AsState` or `Transition`s.

* **Do:** Use hardware-accelerated properties (`translation`, `alpha`, `scale`) for smooth animations.
* **Do:** Use `AnimatorListener` or `doOnEnd` to trigger logic after animation.
* **Don’t:** Animate layout params on the main thread in a loop; use animator APIs instead.
* **Don’t:** Forget to `.cancel()` animators in activities/fragments to avoid leaks.

## System Internals

**App Process & Lifecycle:** Android apps run in a Linux process. The `Application` object is created on app launch, then the main thread (UI thread) calls `onCreate()` of the initial `Activity`. Understand cold vs warm vs hot starts (cold includes process launch and full init, warm may reuse process, hot just resumes). Use `onCreate()`, `onStart()`, `onResume()`, etc. callbacks to manage state transitions.

**Looper & Handler:** Each thread can have a `Looper` with a message queue. The main thread has a `Looper` by default. A `Handler` is bound to a `Looper` and can post `Message`s or `Runnable`s to it. Example:

```kotlin
val handler = Handler(Looper.getMainLooper())
handler.postDelayed({ textView.text = "Hello" }, 1000L)
```

This posts a task to run on the main thread’s queue after a delay. Internally, `Looper.loop()` processes the queue and dispatches to `Handler.handleMessage()`.

* **Do:** Use `HandlerThread` if you need a background thread with a Looper.
* **Don’t:** Touch views from a non-main thread (always update UI via main `Looper` or `runOnUiThread`).
* **Don’t:** Block the main thread’s `Looper` (avoid long loops or sleep).

**Binder & AIDL:** Android’s IPC mechanism is Binder. If your app uses a bound service accessible to other processes, define an AIDL interface. AIDL generates an `IBinder` stub to handle cross-process calls. Key points:

* AIDL is necessary only for remote IPC (different apps) or for multi-threaded service.
* Calls via AIDL come in on a Binder thread pool; you must make your implementation thread-safe.
* Use the `oneway` keyword for asynchronous (no-block) remote calls.

**Memory & Threads:** Android uses Garbage Collection (ART) for memory. Monitor heap usage and handle low-memory (`onTrimMemory` above). Use `Thread` pools (e.g. `Executors` or `CoroutineDispatcher`) for background tasks. Android system also has a pool of Binder threads for each process to handle IPC.

* **Do:** Stick to 2–4 threads for intensive work; oversizing thread pools can cause contention.
* **Do:** Use `Executors.newSingleThreadExecutor()` or `Dispatchers.IO` for disk/network tasks.
* **Don’t:** Spawn unbounded threads (`Thread()`) without managing their lifecycle.
* **Don’t:** Assume thread-affinity for objects; always lock/synchronize shared data or confine via coroutines.

## Networking (Retrofit & OkHttp)

**Retrofit:** A type-safe HTTP client. Define service interfaces with annotations:

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

* **Do:** Use `suspend` functions (Coroutine support) or RxJava adapters.
* **Do:** Always call network APIs off the main thread (Retrofit `suspend` does this by default).
* **Don’t:** Forget to check `Response.isSuccessful` and handle HTTP errors (e.g. 4xx, 5xx).
* **Don’t:** Keep API keys or secrets in code (use NDK or secure storage if needed).

**OkHttp:** The HTTP client used by Retrofit. Configure via `OkHttpClient.Builder`. Use interceptors for logging or auth:

```kotlin
val logging = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY
}
val client = OkHttpClient.Builder()
    .addInterceptor(logging)
    .readTimeout(30, TimeUnit.SECONDS)
    .build()
```

* **Interceptors:** Powerful for modifying requests/responses or adding headers. Application interceptors (`addInterceptor`) run once, network interceptors (`addNetworkInterceptor`) see redirects. You **must** call `chain.proceed(request)` exactly once per interceptor to continue the call.

  * Example logging interceptor (from OkHttp docs): it logs request and response details.
* **Do:** Use connection pooling, caching (`.cache(...)`), and timeouts to optimize network.
* **Do:** Handle exceptions (`IOException` for network, `HttpException` for HTTP errors).
* **Don’t:** Perform retries without backoff or condition.
* **Don’t:** Ignore SSL issues; use HTTPS and pin certificates if possible.

## Testing

**Unit Testing:** Test business logic with JUnit and mocking (MockK or Mockito for Kotlin). Use `runBlockingTest`/`runTest` from `kotlinx-coroutines-test` to test suspending code. Example with MockK:

```kotlin
@Test
fun `loadUsers updates live data`() = runTest {
    coEvery { repository.getUsers() } returns listOf(User(1, "Alice"))
    val viewModel = UserViewModel(repository)
    viewModel.loadUsers()
    assertEquals(listOf(User(1, "Alice")), viewModel.users.value)
}
```

* **Do:** Mock dependencies (e.g. repositories) so unit tests are fast and deterministic.
* **Do:** Use `@Before` for setup and `@After` for cleanup.
* **Don’t:** Hit the real database or network in unit tests.

**Instrumentation (Android) Tests:** Use AndroidX Test with Espresso for UI. Example Espresso usage:

```kotlin
@Test
fun clickButton_opensDetail() {
    onView(withId(R.id.buttonNext)).perform(click())
    onView(withText("Details Page")).check(matches(isDisplayed()))
}
```

* **Do:** Run Espresso tests on emulators or devices, isolate them (clean state).
* **Don’t:** Use `Thread.sleep`; use `IdlingResource` or Espresso’s sync mechanisms.

**Compose UI Tests:** Use `createComposeRule()` and `onNode` assertions. Example:

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

**Flow Testing:** Use `runTest` and collect Flows. The Turbine library simplifies this:

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

For `StateFlow`, either assert on its `value` or use Turbine.

* **Do:** Test coroutine code with `UnconfinedTestDispatcher` or `StandardTestDispatcher` to control execution.
* **Do:** Use `Turbine` or `toList()` in `runTest` for collecting Flows.
* **Don’t:** Rely on timing; make tests deterministic by controlling dispatchers.
* **Don’t:** Test private methods directly; test via public interfaces (like ViewModel behavior).

## Security

**Secure Storage:** Do **not** store sensitive data (tokens, passwords) in plaintext `SharedPreferences` or files. Use Android Keystore and AndroidX Security libraries:

* **Android Keystore:** Store cryptographic keys so they are non-exportable. Example: generate an AES key in the Keystore and use it to encrypt data.
* **EncryptedSharedPreferences:** Part of `androidx.security:security-crypto`, uses keystore-backed keys to encrypt preferences.

**Obfuscation:** Enable R8/ProGuard for release builds to obfuscate code and strip unused classes. This makes reverse-engineering harder (but not impossible). Example in `release` buildType:

```gradle
buildTypes {
    release {
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
    }
}
```

**NDK (Native Code):** If using the NDK (C/C++), remember native code can also be reverse-engineered.

* **Do:** Avoid storing secrets in native code or resources; consider server-side key management.
* **Don’t:** Rely on NDK alone for security; combine with signatures and checks.

**Network Security:** Always use HTTPS/TLS. Use **Network Security Config** to enforce security policies. With an XML file, you can:

* Disable cleartext traffic (`<base-config cleartextTrafficPermitted="false">`) to ensure only HTTPS.
* Pin certificates or custom CAs to prevent MITM.
* Use TLS 1.2+ on older Android (enable via security config if needed).

Example `network_security_config.xml` snippet:

```xml
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
    <domain-config>
        <domain includeSubdomains="true">api.example.com</domain>
        <!-- Optionally pin certs here -->
    </domain-config>
</network-security-config>
```

* **Do:** Update `android:networkSecurityConfig` in the Manifest to point to this file.
* **Do:** Use certificate pinning (in OkHttp or config) for extra security.
* **Don’t:** Trust user-added CAs or allow cleartext (unless absolutely needed with a debug override).

## Gradle Build System

**Build Optimization:** Follow Android’s official tips for faster builds. Key practices:

* **Enable Gradle Daemon and parallel builds:** In `gradle.properties`:

  ```
  org.gradle.daemon=true
  org.gradle.parallel=true
  ```
* **Use KSP instead of Kapt:** KSP is faster for annotation processing (e.g. use Room KSP artifact).
* **Avoid unnecessary resources:** Limit `resourceConfigurations` and `resConfigs` for debug builds to speed up packaging.
* **Do:** Keep Android Gradle Plugin and dependencies up to date.
* **Don’t:** Apply irrelevant plugins or tasks on all modules; scope them only where needed.

**Custom Gradle Tasks:** You can write tasks in Kotlin DSL. For example, to print version:

```kotlin
tasks.register("printVersion") {
    doLast {
        println("App Version: ${project.properties["VERSION_NAME"]}")
    }
}
```

* **Do:** Reuse tasks and configuration across modules via convention plugins or Gradle scripts.
* **Don’t:** Hardcode values; use `project.properties` or build config fields.

**Build Variants:** Define `buildTypes` (e.g., `debug`, `release`) and `productFlavors` (e.g., `free`, `paid`) in `module:app` Gradle:

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

* Each combination (e.g. `demoDebug`, `fullRelease`) is a variant. You can customize source sets (`src/full/`, `src/demo/`).
* **Do:** Use `applicationIdSuffix` or `versionNameSuffix` to distinguish flavors.
* **Do:** Avoid duplicating code; share code in `src/main`.
* **Don’t:** Have mismatched minSdk or compileSdk in different variants.

## Google Play Store

**Publishing:** Build an **Android App Bundle (AAB)** for release. Use **Play App Signing** to let Google manage your signing key. This enhances security and allows Google to optimize APK splits for devices:

* **Signing:** Upload a cryptographic key to Play Console or enroll in Google Play App Signing. Google then generates optimized APKs from your AAB.
* **Versioning:** Follow semantic versioning with `versionCode` and `versionName` in Gradle.

**Play Integrity API:** Integrate the Play Integrity API to ensure your app is genuine. Call it at critical points to verify the app’s authenticity:

> “Call the Integrity API at important moments in your app to check that user actions and requests are coming from your unmodified app binary, installed by Google Play… Your backend can decide what to do next to prevent abuse”.

It helps protect against rooting, tampering, and fraud.

**Feature Delivery:** Use **Dynamic Feature Modules** with the Android App Bundle for on-demand or conditional delivery (as mentioned in Modularization). In Play Console:

* **On-demand modules:** Specify `installTime`, `onDemand`, or `conditional` (e.g. device features, user country) in the manifest.
* **In-app updates and reviews:** Implement the Play In-App Updates API to prompt users to update, and the Review API for in-app feedback.

**Do:** Test your release build and signing process (use `play` Gradle plugin or CLI).
**Don’t:** Release debug builds or un-optimized APKs.
**Do:** Monitor **Android Vitals** in Play Console for crashes and excessive startup time. Vitals consider 5s+ cold start (TTID) as bad.
**Don’t:** Violate Play policies; follow guidelines (e.g. privacy, content) to avoid app rejection.

---

**Sources:** Android official documentation and developer guides.
