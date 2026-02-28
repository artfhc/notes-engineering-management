# Android Basics: Activity vs Fragment

In Android, **Activity** and **Fragment** are both key components used to build UIs, but they have different purposes and characteristics.

## Activity vs Fragment

| Feature | Activity | Fragment |
|---------|----------|----------|
| **Definition** | A single, focused thing the user can do | A modular section of an activity |
| **Lifecycle** | Has its own lifecycle | Lifecycle is tied to the hosting Activity |
| **UI Management** | Manages its own UI | Can be reused in multiple Activities |
| **Navigation** | Starts other Activities via Intent | Usually replaced/swapped via FragmentManager |
| **Reuse** | Harder to reuse | Easier to reuse across Activities |
| **Back Stack** | Managed by system | Must be managed manually via FragmentManager |
| **Independent?** | Yes | No – depends on a host Activity |

---

## Lifecycle Comparison

### Activity Lifecycle

```
onCreate()
   ↓
onStart()
   ↓
onResume()
   ↓
(Running)
   ↓
onPause()
   ↓
onStop()
   ↓
onDestroy()
```

### Fragment Lifecycle

```
onAttach()
   ↓
onCreate()
   ↓
onCreateView()
   ↓
onViewCreated()
   ↓
onStart()
   ↓
onResume()
   ↓
(Running)
   ↓
onPause()
   ↓
onStop()
   ↓
onDestroyView()
   ↓
onDestroy()
   ↓
onDetach()
```

**Note:** Fragment's lifecycle includes view-specific methods like `onCreateView()` and `onDestroyView()` since views can be created/destroyed independently of the Fragment itself.

---

## When to Use

### Use Activity when:

- You're building a standalone screen or UI task
- It needs to be launched independently (e.g., via launcher or deep link)

### Use Fragment when:

- You want reusable UI blocks inside an Activity
- You're using a ViewPager, BottomNavigationView, or Navigation Component
- You're designing a responsive UI for phones/tablets

---

## Why Fragment Views Can Be Created/Destroyed Independently

A Fragment's view can be created and destroyed independently of the Fragment itself because a Fragment is designed to outlive its view hierarchy, especially during UI transitions like configuration changes or Fragment transactions.

### Why This Happens

**1. Fragment ≠ View**

- A Fragment is a controller that holds logic and state
- The View (`onCreateView()`) is just the UI layer attached to it
- The system may destroy the Fragment's View to save memory or because it's no longer visible, but the Fragment instance might still remain in memory for reattachment later

**2. Back Stack and Navigation**

- When you detach or hide a Fragment (e.g., using `FragmentTransaction.detach()`), the system destroys its view (`onDestroyView()`), but not the Fragment itself
- The Fragment remains alive in the back stack, so when it comes back, `onCreateView()` is called again to rebuild the UI

**3. Configuration Changes**

- On screen rotation, the Activity is recreated, but you might choose to retain the Fragment instance (via `setRetainInstance(true)` in old APIs or ViewModel now), so the Fragment remains alive, but the view must be recreated

### Lifecycle Separation Example

```kotlin
override fun onCreateView(...) {
    // Inflate layout
}

override fun onDestroyView() {
    // Clean up bindings or UI references
}
```

You clean up views in `onDestroyView()` because the Fragment is still alive, but the View is gone.

### Why It's Useful

- **Efficient memory usage:** Views can be destroyed when not visible
- **State preservation:** Business logic/state survives even if view doesn't
- **Navigation transitions:** Allows smoother UI transitions and reuse of fragments

---

## How Lifecycle Methods Are Triggered

In Android, the lifecycle methods of Activity and Fragment are called by the **Android Framework** (AMS – Activity Manager Service), not by your code.

### 1. Activity Lifecycle

- When the system starts an Activity (e.g. via `startActivity()`), the Activity Manager Service (AMS) sends a message to the main thread (UI thread)
- This is handled by an internal class called `ActivityThread`, which:
  - Instantiates the Activity
  - Calls lifecycle methods in order: `onCreate()`, `onStart()`, `onResume()`, etc.

**Under the hood:**
```
startActivity() → AMS → ActivityThread → Instrumentation.callActivityOnCreate()
```

### 2. Fragment Lifecycle

- Managed by the `FragmentManager` inside an Activity
- When you call `add()`, `replace()`, or navigate via `NavController`:
  - `FragmentManager` coordinates the changes
  - It invokes methods like `onAttach()`, `onCreate()`, `onCreateView()`, etc., on the main thread
  - The Fragment's view is inflated using `LayoutInflater`

**Example:**

```kotlin
supportFragmentManager.beginTransaction()
    .replace(R.id.container, MyFragment())
    .commit()
```

This results in the system:
- Creating the Fragment instance
- Attaching it to the Activity
- Calling each method in the lifecycle based on the current state

### 3. Event-Driven Model

These lifecycle calls are reactive, based on events such as:

- App launched or closed
- Screen rotation
- Fragment added/removed/hidden
- App sent to background

All changes are funneled through Android's main thread looper, which dispatches messages from the system.

### Summary

| Trigger | Who Calls Lifecycle |
|---------|---------------------|
| App launch | Android system (AMS → ActivityThread) |
| Fragment added | FragmentManager |
| Configuration change | Android system restarts the Activity |
| Navigation | FragmentManager or NavController |
| Going to background | AMS triggers `onPause()` and `onStop()` |

---

## onCreate(), onStart(), and onResume()

These are key lifecycle methods in Android. They represent different stages of an Activity or Fragment becoming visible and interactive.

### Lifecycle Flow

```
onCreate() → onStart() → onResume()
```

Each method serves a different purpose.

### Breakdown

| Method | Purpose | Called When |
|--------|---------|-------------|
| `onCreate()` | Initialize non-UI logic and inflate UI | First time the component is created (e.g., new Activity or Fragment) |
| `onStart()` | Prepare the component to become visible | Every time the component is coming to the screen (including after pause) |
| `onResume()` | Final step; the component is now interactive | Just before user can start interacting (e.g., touch, typing) |

### Practical Example

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Setup dependencies, restore saved state, setContentView()
}

override fun onStart() {
    super.onStart()
    // Start observing LiveData, register BroadcastReceiver
}

override fun onResume() {
    super.onResume()
    // Start animations, resume video, acquire camera
}
```

### Comparison Summary

| Feature | onCreate() | onStart() | onResume() |
|---------|------------|-----------|------------|
| Called once? | Yes (unless recreated) | No | No |
| Safe to access UI? | Yes | Yes | Yes |
| Interactive? | Not yet | Not yet | ✅ Yes |
| Common actions | Init, ViewModel, view | Register listeners | Start sensors, playback |

---

## Kotlin Flow: Thread Safety

**Yes** — Kotlin Flow emits values one by one in a thread-safe way by default.

### Thread Safety of Flow

Flow is **cold** and **sequential** by design:

- Emissions happen one at a time
- Collector code and emitter code run sequentially on the same coroutine

**Example:**

```kotlin
flow {
    emit(1)   // will suspend until collected
    emit(2)
}
```

`emit(2)` won't run until `emit(1)` is fully collected.

This ensures no concurrent emissions → thread safe.

### But Not Always...

If you use multiple coroutines or concurrent emitters, like with `SharedFlow` or `StateFlow`, you need to manage thread safety explicitly.

**Example:**

```kotlin
val flow = MutableSharedFlow<Int>()

coroutineScope {
    launch { flow.emit(1) }   // concurrently
    launch { flow.emit(2) }
}
```

This can emit from multiple coroutines → still safe internally, but the timing can be interleaved. You may need to coordinate access if you're doing mutation alongside.

### To Ensure Thread Safety with Shared Flow

Use `Mutex`, or ensure all emits happen from the same coroutine context.

```kotlin
val mutex = Mutex()

suspend fun safeEmit(flow: MutableSharedFlow<Int>, value: Int) {
    mutex.withLock {
        flow.emit(value)
    }
}
```

### Summary

| Flow Type | Emits One-by-One | Thread Safe by Default | Notes |
|-----------|------------------|------------------------|-------|
| `flow {}` | ✅ Yes | ✅ Yes | Cold, sequential, suspending |
| `SharedFlow` | ❌ Not enforced | ✅ Internally safe | May emit concurrently |
| `StateFlow` | ❌ Overwrites | ✅ Internally safe | Only keeps the latest |

---

## Cold Flow vs Hot Flow

| Aspect | Cold Flow | Hot Flow |
|--------|-----------|----------|
| **Starts Emitting?** | Only when collected | Starts emitting regardless of collectors |
| **Per Collector** | Each collector gets its own stream | All collectors share the same stream |
| **Examples** | `flow {}` | `SharedFlow`, `StateFlow`, `Channel` |

### Cold Flow

- **Lazy:** doesn't emit anything until `collect()` is called
- Each collector gets all emissions from the beginning
- Think of it like a recipe — nothing happens until you start cooking

**Example:**

```kotlin
val coldFlow = flow {
    println("Flow started")
    emit(1)
    emit(2)
}

coldFlow.collect { println(it) }
// Output:
// Flow started
// 1
// 2
```

If two collectors collect `coldFlow`, both will re-run the whole block independently.

### Hot Flow

- **Always active** (once started), independent of collectors
- Emissions happen even if no one is collecting
- Collectors may miss past values if they weren't subscribed yet
- Like a radio broadcast — if you tune in late, you miss the beginning

**Example with SharedFlow:**

```kotlin
val sharedFlow = MutableSharedFlow<Int>()

launch {
    sharedFlow.emit(1) // happens even if no one is collecting
}

launch {
    sharedFlow.collect { println(it) } // might miss "1"
}
```

### When to Use What

| Use Case | Recommended Flow |
|----------|------------------|
| Lazy computation, recomputed per use | `flow {}` |
| Event broadcasting | `SharedFlow` |
| Observing latest state | `StateFlow` |
| Buffered, multi-value stream | `Channel` |

---

## ViewModel

ViewModel is an Android Architecture Component that holds and manages UI-related data in a lifecycle-conscious way. Its key property: it **survives configuration changes** (like screen rotation) while the Activity or Fragment is destroyed and recreated.

**Why it survives rotation:**
- Activities are destroyed on rotation, but they share a `ViewModelStore`
- The `ViewModelStore` is retained across configuration changes by the system
- A new Activity instance calls `ViewModelProvider`, which finds the existing ViewModel in the store

**Key properties:**
- Scoped to an Activity or Fragment's lifecycle (not the View)
- Cleared only when the owner is permanently finished (back pressed, not rotated)
- Exposes state via `StateFlow` or `LiveData` — never holds references to Views

**`viewModelScope`:**
- A `CoroutineScope` tied to the ViewModel's lifecycle
- Automatically cancelled when `onCleared()` is called
- Use for all coroutines launched from a ViewModel

**`onCleared()`:**
- Called when the ViewModel is about to be destroyed (owner permanently gone)
- Use to cancel non-coroutine resources (e.g., custom thread pools, listeners)
- Note: NOT called on process death — use `onSaveInstanceState()` for that

**Obtaining a ViewModel:**

```kotlin
// In a Fragment
private val viewModel: MyViewModel by viewModels()

// Shared ViewModel between Fragments (scoped to Activity)
private val sharedViewModel: SharedViewModel by activityViewModels()
```

**ViewModel with StateFlow example:**

```kotlin
class CounterViewModel : ViewModel() {

    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() {
        viewModelScope.launch {
            _count.update { it + 1 }
        }
    }

    fun loadData() {
        viewModelScope.launch {
            val result = repository.fetchData() // suspend fun
            _count.value = result.size
        }
    }

    override fun onCleared() {
        super.onCleared()
        // cancel non-coroutine resources here
    }
}
```

---

## ViewModel Lifecycle

### Compact Summary

```
Activity onCreate ──► Activity onStart ──► Activity onResume
      │
      ▼
 ViewModel created (first time only)
      │
      ▼
[User rotates screen]
      │
      ├─ Activity onPause → onStop → onDestroy  (Activity destroyed)
      │
      ▼
 ViewModel SURVIVES (ViewModelStore retained)
      │
      ├─ New Activity onCreate → onStart → onResume
      │
      ▼
 Same ViewModel instance reused
      │
[User presses Back]
      │
      ├─ Activity onPause → onStop → onDestroy (isFinishing = true)
      │
      ▼
 ViewModel.onCleared() called ── ViewModel destroyed
```

### Step-by-step Lifecycle Timeline

```
Event                    Activity State        ViewModel State
─────────────────────────────────────────────────────────────────
App launches             onCreate()            ← ViewModel CREATED
                         onStart()             [count = 0]
                         onResume()

User rotates             onPause()             [count = 5] ← RETAINED
                         onStop()
                         onDestroy()           ← Activity destroyed
                                               ViewModel still alive

New Activity created     onCreate()            Same ViewModel instance
                         onStart()             [count = 5] ← available
                         onResume()

User presses Back        onPause()             [count = 5]
                         onStop()
                         onDestroy()           ← onCleared() called
                                               ViewModel destroyed
```

### Comparison Table: What Survives?

| Event | Activity | ViewModel | `savedStateHandle` |
|-------|----------|-----------|-------------------|
| Screen rotates | Destroyed + recreated | **Survives** | Survives |
| Back pressed | Destroyed (finished) | **Destroyed** (`onCleared`) | Cleared |
| Process killed (OOM) | Destroyed | **Destroyed** (no `onCleared`) | Survives (restored from bundle) |
| Fragment replaced (backstack) | Alive | Survives (if scoped to Activity) | Survives |
| Fragment replaced (no backstack) | Alive | **Destroyed** (`onCleared`) | Cleared |

**Key interview callout:** `onCleared()` is NOT called on process death. If you need to persist state across process death (e.g., a draft form), use `SavedStateHandle` (injected into ViewModel via Hilt or `SavedStateViewModelFactory`). `onSaveInstanceState()` in the Activity/Fragment is the underlying mechanism.

```kotlin
class FormViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Survives both rotation AND process death
    var draftText: String
        get() = savedStateHandle["draft"] ?: ""
        set(value) { savedStateHandle["draft"] = value }
}
```

---

## LiveData vs StateFlow

### Comparison Table

| Dimension | `LiveData` | `StateFlow` |
|-----------|-----------|------------|
| Lifecycle awareness | Built-in (auto-stops on STOPPED) | Manual — use `repeatOnLifecycle` |
| Initial value required | No (starts null) | Yes (must provide initial value) |
| Replay to new collectors | Latest value replayed | Latest value replayed |
| Backpressure | None (drops intermediates) | Conflates by default (latest wins) |
| Thread safety | Must post from background (`postValue`) | Thread-safe (`value` from any thread) |
| Jetpack / Java interop | First-class (works in Java, DataBinding) | Kotlin-only; needs `asLiveData()` for DataBinding |
| Transformation operators | `Transformations.map`, `switchMap` | Full Kotlin Flow operators (`map`, `filter`, `combine`, etc.) |

### StateFlow inside ViewModel

```kotlin
class SearchViewModel : ViewModel() {

    private val _query = MutableStateFlow("")
    val query: StateFlow<String> = _query.asStateFlow()

    private val _results = MutableStateFlow<List<String>>(emptyList())
    val results: StateFlow<List<String>> = _results.asStateFlow()

    fun onQueryChanged(q: String) {
        _query.value = q
        viewModelScope.launch {
            _results.value = repository.search(q)
        }
    }
}
```

### Safe collection in a Fragment with `repeatOnLifecycle`

```kotlin
class SearchFragment : Fragment() {

    private val viewModel: SearchViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // repeatOnLifecycle restarts the block each time lifecycle reaches STARTED
        // and cancels it when it drops below STARTED (e.g., app goes to background)
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.results.collect { results ->
                    adapter.submitList(results)
                }
            }
        }
    }
}
```

**Anti-pattern — `launchWhenStarted`:**

```kotlin
// ❌ DO NOT USE — launchWhenStarted suspends the coroutine when app backgrounds
// but does NOT cancel it. The coroutine keeps the Flow subscription alive,
// leaking resources and potentially processing stale updates on resume.
lifecycleScope.launchWhenStarted {
    viewModel.results.collect { ... }
}

// ✅ Use repeatOnLifecycle instead — it cancels and restarts cleanly
```

### When to Use Each

| Scenario | Use `LiveData` | Use `StateFlow` |
|----------|---------------|----------------|
| Legacy Java codebase | ✓ | |
| XML DataBinding with two-way binding | ✓ | |
| Simple observable with no transformation | ✓ | |
| Existing codebase already uses LiveData | ✓ | |
| New Kotlin-first codebase | | ✓ |
| Complex async transformations (`combine`, `flatMapLatest`) | | ✓ |
| Sharing state across multiple collectors | | ✓ |
| Testing with Turbine or Flow test utilities | | ✓ |
