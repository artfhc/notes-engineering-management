# Observer Pattern in Kotlin

## Basic Implementation

```kotlin
interface Observer {
    fun notify(event: Event)
}

class ObserverImpl: Observer {
    override fun notify(event: Event) {
        when (event) {
            is Event.ClickEvent -> println("Click event ${event.id}")
            is Event.SelectEvent -> println("Select event ${event.id}")
        }
    }
}

class Orchestrator {
    private val observers = mutableListOf<Observer>()

    fun addObserver(observer: Observer) {
        observers.add(observer)
    }

    fun removeObserver(observer: Observer) {
        observers.remove(observer)
    }

    fun notifyAll(event: Event) {
        observers.forEach { it.notify(event) }
    }
}
```

---

## Making Orchestrator Thread-Safe

To make your Orchestrator thread-safe in Kotlin, you need to guard concurrent access to the observers list. There are two main approaches:

### Option 1: Use CopyOnWriteArrayList (Recommended for Read-Mostly Use)

This is great if `notifyAll` is called frequently and `addObserver`/`removeObserver` are rare.

```kotlin
import java.util.concurrent.CopyOnWriteArrayList

class Orchestrator {
    private val observers = CopyOnWriteArrayList<Observer>()

    fun addObserver(observer: Observer) {
        observers.add(observer)
    }

    fun removeObserver(observer: Observer) {
        observers.remove(observer)
    }

    fun notifyAll(event: Event) {
        observers.forEach {
            it.notify(event)
        }
    }
}
```

`CopyOnWriteArrayList` handles concurrent reads and writes without manual synchronization.

### Option 2: Use synchronized Manually (Fine-Grained Control)

```kotlin
class Orchestrator {
    private val observers = mutableListOf<Observer>()
    private val lock = Any()

    fun addObserver(observer: Observer) {
        synchronized(lock) {
            observers.add(observer)
        }
    }

    fun removeObserver(observer: Observer) {
        synchronized(lock) {
            observers.remove(observer)
        }
    }

    fun notifyAll(event: Event) {
        val snapshot: List<Observer>
        synchronized(lock) {
            snapshot = observers.toList() // Avoid holding lock during callbacks
        }
        snapshot.forEach {
            it.notify(event)
        }
    }
}
```

This version avoids holding a lock while notifying, which prevents potential deadlocks or performance issues if observers are slow or misbehaving.

### Summary

| Strategy | Thread-Safety | Performance | Use Case |
|----------|---------------|-------------|----------|
| CopyOnWriteArrayList | ✅ | Great for reads | Notify often, modify rarely |
| synchronized + copy | ✅ | Balanced | More control, avoids locking notify |

---

## Thread-Safe Observer Pattern with Coroutines

To utilize coroutines for thread safety in your observer pattern, the best approach is to use a **Mutex** or **actor-based concurrency** to protect access to the observer list.

Here are two coroutine-friendly implementations:

### Option 1: Use Mutex for Synchronization

```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.*

class Orchestrator {
    private val observers = mutableListOf<Observer>()
    private val mutex = Mutex()

    suspend fun addObserver(observer: Observer) {
        mutex.withLock {
            observers.add(observer)
        }
    }

    suspend fun removeObserver(observer: Observer) {
        mutex.withLock {
            observers.remove(observer)
        }
    }

    suspend fun notifyAll(event: Event) {
        val snapshot = mutex.withLock { observers.toList() }
        snapshot.forEach {
            it.notify(event) // Assuming notify is fast and doesn't need to be suspended
        }
    }
}
```

If `notify()` itself is suspendable, you can call it in parallel using `coroutineScope` and `launch`.

### Option 2: Use Actor for Single-threaded Access (Channel-based Messaging)

Using an actor turns your Orchestrator into a coroutine that serializes all operations.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.actor

sealed class OrchestratorMsg
data class Add(val observer: Observer) : OrchestratorMsg()
data class Remove(val observer: Observer) : OrchestratorMsg()
data class Notify(val event: Event) : OrchestratorMsg()

fun CoroutineScope.orchestratorActor() = actor<OrchestratorMsg> {
    val observers = mutableListOf<Observer>()
    for (msg in channel) {
        when (msg) {
            is Add -> observers.add(msg.observer)
            is Remove -> observers.remove(msg.observer)
            is Notify -> observers.forEach { it.notify(msg.event) }
        }
    }
}
```

**Usage:**

```kotlin
val orchestrator = scope.orchestratorActor()

orchestrator.send(Add(observer))
orchestrator.send(Notify(Event.ClickEvent(123)))
```

### Summary

| Approach | Pros | When to Use |
|----------|------|-------------|
| Mutex | Simple and coroutine-safe | Easy migration from synchronized code |
| Actor | Fully serializes access, avoids manual locking | When you want strong guarantees and isolation |

---

## Thread-Safe Event Bus Implementation

**Question:** I have an event bus implementation. Can you make it thread safe?

```kotlin
sealed class Event {
    data class ClickEvent(val id: String) : Event()
    data class SelectEvent(val id: Int) : Event()
}

object EventBus {
    private val _events = MutableSharedFlow<Event>(replay = 0)
    val events: SharedFlow<Event> = _events.asSharedFlow()

    suspend fun emit(event: Event) {
        _events.emit(event)
    }

    suspend fun collectEvents(action: suspend (Event) -> Unit) {
        events.collect {
            action(it)
        }
    }
}
```

**Answer:** Your `EventBus` implementation is already thread-safe! `MutableSharedFlow` is designed to be thread-safe by default:

- Multiple coroutines can call `emit()` concurrently without additional synchronization
- Multiple collectors can call `collect()` safely
- The internal implementation handles concurrent access

However, if you want to ensure strict ordering of emissions from multiple coroutines, you can use a `Mutex`:

```kotlin
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

object EventBus {
    private val _events = MutableSharedFlow<Event>(replay = 0)
    val events: SharedFlow<Event> = _events.asSharedFlow()
    private val mutex = Mutex()

    suspend fun emit(event: Event) {
        mutex.withLock {
            _events.emit(event)
        }
    }

    suspend fun collectEvents(action: suspend (Event) -> Unit) {
        events.collect {
            action(it)
        }
    }
}
```
