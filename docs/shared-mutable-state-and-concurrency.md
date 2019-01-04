**Table of contents**

* [Shared mutable state and concurrency](#shared-mutable-state-and-concurrency)
  * [The problem](#the-problem)
  * [Volatiles are of no help](#volatiles-are-of-no-help)
  * [Thread-safe data structures](#thread-safe-data-structures)
  * [Thread confinement fine-grained](#thread-confinement-fine-grained)
  * [Thread confinement coarse-grained](#thread-confinement-coarse-grained)
  * [Mutual exclusion](#mutual-exclusion)
  * [Actors](#actors)

## Shared mutable state and concurrency

Shared mutable state and concurrency ：共享的可變狀態與並發

Coroutines can be executed concurrently using a multi-threaded dispatcher like the [Dispatchers.Default][Dispatchers.Default]. It presents all the usual concurrency problems. The main problem being synchronization of access to **shared mutable state**. Some solutions to this problem in the land of coroutines are similar to the solutions in the multi-threaded world, but others are unique.

使用多線程的分配器像是 [Dispatchers.Default][Dispatchers.Default] 去並發 (同時) 執行協程。它提出所有常見並發的問題。主要問題是存取**共享的可變狀態**的同步。在協程的領土 (領域) 中，這問題的一些解決方式類似於在多線程的世界中的解決方式，但其他的解決方式是唯一的 (獨一無二) 。

### The problem

The problem ：問題

Let us launch a hundred coroutines all doing the same action thousand times. We'll also measure their completion time for further comparisons:

讓我們發射 100 個協程，每個協程做相同的動作 1000 次。我們也將衡量它們的完成次數用於進一步的比較。

```kotlin
suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}
```

We start with a very simple action that increments a shared mutable variable using multi-threaded [Dispatchers.Default][Dispatchers.Default]  that is used in [GlobalScope][GlobalScope]. 

我們從一個非常簡單的動作開始，在 [GlobalScope][GlobalScope] 中使用多線程的 [Dispatchers.Default][Dispatchers.Default] 增量一個共享可變的變數 `counter++` 。

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*    

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}

//ans:
//Completed 100000 actions in 50 ms
//Counter = 99207 (這個不一定，因為本身就是個錯誤)
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01.kt)獲取完整的代碼

What does it print at the end? It is highly unlikely to ever print "Counter = 100000", because a thousand coroutines increment the `counter` concurrently from multiple threads without any synchronization.

最後印出什麼？它不太可能印出 "Counter = 100000" ，因為 1000 個協程從多個線程同時增量 `counter` 沒有任何同步。 

> Note: if you have an old system with 2 or fewer CPUs, then you _will_ consistently see 100000, because the thread pool is running in only one thread in this case. To reproduce the problem you'll need to make the following change:
>
> 注意：如果你有舊系統使用 2 或 更少的 CPU ，接著你將始終看到 100000 ，因為在這種情況下，線程池只在一個線程中執行。重現問題，你將需要做以下修改：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 自定義線程池
val mtContext = newFixedThreadPoolContext(2, "mtPool") // explicitly define context with two threads
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 使用自定義的線程池 mtContext
    CoroutineScope(mtContext).massiveRun { // use it instead of Dispatchers.Default in this sample and below 
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}

//ans:
//Completed 100000 actions in 63 ms
//Counter = 99764 (這個不一定，因為本身就是個錯誤)
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01b.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-01b.kt)獲取完整的代碼

### Volatiles are of no help

Volatiles are of no help ： Volatiles 功能沒有任何幫助

There is common misconception that making a variable `volatile` solves concurrency problem. Let us try it:

有常見的誤解，使變數為 `volatile` 解決並發問題。讓我們試試它：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 使用 Volatile 功能
@Volatile // in Kotlin `volatile` is an annotation 
var counter = 0

fun main() = runBlocking<Unit> {
    GlobalScope.massiveRun {
        counter++
    }
    println("Counter = $counter")
}

//ans:
//Completed 100000 actions in 46 ms
//Counter = 99370 (這個不一定，因為本身就是個錯誤)
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-02.kt)獲取完整的代碼

This code works slower, but we still don't get "Counter = 100000" at the end, because volatile variables guarantee linearizable (this is a technical term for "atomic") reads and writes to the corresponding variable, but do not provide atomicity of larger actions (increment in our case).

這些代碼運作很慢，但我們在最終還是不會獲取 "Counter = 100000" ，因為 volatile 變數保證可線性 (這是 "原子" 的技術術語) 讀跟寫對應的變數，但不會提供大量動作的原子性 (在我們例子中的增量) 。

### Thread-safe data structures

Thread-safe data structures ：線程安全的資料結構

The general solution that works both for threads and for coroutines is to use a thread-safe (aka synchronized, linearizable, or atomic) data structure that provides all the necessarily synchronization for the corresponding operations that needs to be performed on a shared state. In the case of a simple counter we can use `AtomicInteger` class which has atomic `incrementAndGet` operations:

適用於多線程或多協程的通用解決方式是使用一個線程安全的 (或稱同步、線性、或原子) 資料結構，在共享狀態中，執行需要的操作，為相對應的操作提供所有必須的同步化。在簡單計數器的情況下，我們可以使用 `AtomicInteger` 類別，該類別有原子 `incrementAndGet` 操作。

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.atomic.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 使用線程安全的類別
var counter = AtomicInteger()

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        
        // 原子性的操作
        counter.incrementAndGet()
    }
    println("Counter = ${counter.get()}")
//sampleEnd    
}
//ans:
//Completed 100000 actions in 77 ms
//Counter = 100000
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-03.kt)獲取完整的代碼

This is the fastest solution for this particular problem. It works for plain counters, collections, queues and other standard data structures and basic operations on them. However, it does not easily scale to complex state or to complex operations that do not have ready-to-use thread-safe implementations. 

這是為了這特定的問題最快的解決方式。它適用於原始數值的計數器、集合、佇列和其他標準的資料結構，以及在它們之中的基本操作。然而，它不容易衡量複雜的狀態或複雜的操作，所以不會有隨時即用的線程安全實作。

### Thread confinement fine-grained

Thread confinement fine-grained ：線程限制的「細」粒度方式，細粒度是為每個細節的操作增加線程安全的方式

_Thread confinement_ is an approach to the problem of shared mutable state where all access to the particular shared state is confined to a single thread. It is typically used in UI applications, where all UI state is confined to the single event-dispatch/application thread. It is easy to apply with coroutines by using a single-threaded context. 

線程限制是共享的可變狀態問題的解決辦法，所有存取特定共享狀態被限制到單一線程。它在 UI 應用程式中是典型的使用，所有 UI 狀態被限制到單一事件 - 分配 / 應用程式線程。透過使用一個單一線程環境應應用協程是容易的。

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 建立單一線程的環境
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun { // run each coroutine with DefaultDispathcer
        
        // 細粒度的限制，每次的動作使用單一線程的環境來運行增量
        withContext(counterContext) { // but confine each increment to the single-threaded context
            counter++
        }
    }
    println("Counter = $counter")
//sampleEnd    
}
//ans:
//Completed 100000 actions in 2007 ms
//Counter = 100000
```
> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-04.kt)獲取完整的代碼

This code works very slowly, because it does _fine-grained_ thread-confinement. Each individual increment switches from multi-threaded [Dispatchers.Default][Dispatchers.Default] context to the single-threaded context using [withContext][withContext] block.

這些代碼非常慢的運作，因為它進行細粒度的線程限制。每個個別增量操作使用 [withContext][withContext] 區塊從多線程 [Dispatchers.Default][Dispatchers.Default] 環境轉換到單線程環境。

### Thread confinement coarse-grained

Thread confinement coarse-grained ：線程限制的「粗」粒度方式，，粗粒度是為每個細節操作的環境增加線程安全的方式

In practice, thread confinement is performed in large chunks, e.g. big pieces of state-updating business logic are confined to the single thread. The following example does it like that, running each coroutine in the single-threaded context to start with. Here we use [CoroutineScope()][CoroutineScope()] function to convert coroutine context reference to [CoroutineScope][CoroutineScope]:

在實踐中，在大區塊中執行線程限制，例如：大部分的狀態更新業務邏輯被限制到單一線程。以下範例像這樣，在單一線程的環境中運行每個協程為開始。這裡我們使用 [CoroutineScope()][CoroutineScope()] 函數，讓協程環境轉換到 [CoroutineScope][CoroutineScope] 引用：

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    CoroutineScope(counterContext).massiveRun { // run each coroutine in the single-threaded context
        counter++
    }
    println("Counter = $counter")
//sampleEnd    
}
//ans:
//Completed 100000 actions in 45 ms
//Counter = 100000
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-05.kt)獲取完整的代碼

This now works much faster and produces correct result.

這是現在更快運行的方式以並產生正確的結果。

### Mutual exclusion

Mutual exclusion ：互斥排除，當只有一開一關的情況時才讓你寫入。

Mutual exclusion solution to the problem is to protect all modifications of the shared state with a _critical section_ that is never executed concurrently. In a blocking world you'd typically use `synchronized` or `ReentrantLock` for that. Coroutine's alternative is called [Mutex][Mutex]. It has [lock][Mutex.lock] and [unlock][Mutex.unlock] functions to delimit a critical section. The key difference is that `Mutex.lock()` is a suspending function. It does not block a thread.

問題的互斥排除解決方式是永不同時執行關鍵部分的程式，保護共享狀態的所有修改。在阻塞的世界中，你可能會使用 Java 典型的 `synchronized` 或  `ReentrantLock` API 用於阻塞。多協程的替代品是稱 [Mutex][Mutex] 。它有 [lock][Mutex.lock] 和 [unlock][Mutex.unlock] 函數去劃分某一關鍵區塊。關鍵的區別是 `Mutex.lock()` 是懸掛函數。它不會阻塞線程。

There is also [withLock][withLock] extension function that conveniently represents `mutex.lock(); try { ... } finally { mutex.unlock() }` pattern: 

也有 [withLock][withLock] 擴展函數，方便表示 `mutex.lock(); try { ... } finally { mutex.unlock() }` 模式。

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}

// 互斥的 API
val mutex = Mutex()
var counter = 0

fun main() = runBlocking<Unit> {
//sampleStart
    GlobalScope.massiveRun {
        
        // 便於互斥的情況
        mutex.withLock {
            counter++        
        }
    }
    println("Counter = $counter")
//sampleEnd    
}
//ans:
//Completed 100000 actions in 1157 ms
//Counter = 100000
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-06.kt)獲取完整的代碼

The locking in this example is fine-grained, so it pays the price. However, it is a good choice for some situations where you absolutely must modify some shared state periodically, but there is no natural thread that this state is confined to.

在這個範例中的鎖是細粒度的 (每個操作都執行) ，所以它付出了代價。而且，對於你絕對必須定期的修改某些共享的狀態某些情況是一個好的選擇，但是沒有這種限制狀態的自然線程。

### Actors

Actors ：執行者，通道類型，與 produce 相反，函數內負責消費，回傳類型負責生產

An [actor](https://en.wikipedia.org/wiki/Actor_model) is an entity made up of a combination of a coroutine, the state that is confined and encapsulated into this coroutine, and a channel to communicate with other coroutines. A simple actor can be written as a function, but an actor with a complex state is better suited for a class. 

[actor](https://en.wikipedia.org/wiki/Actor_model) 是由協程組合而成的實例，共享的狀態被限制並被封裝到這個協程，並且使用一個通道與其他的協程溝通。一個簡單的 actor 被寫為函數，而一個複雜狀態的適合為一個類別讓 actor 判斷使用。

There is an [actor][actor] coroutine builder that conveniently combines actor's mailbox channel into its scope to receive messages from and combines the send channel into the resulting job object, so that a single reference to the actor can be carried around as its handle.

有一個 [actor][actor] 協程建造者，將 actor 的接收通道結合到 [ActorScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-actor-scope/index.html) 去接收訊息 `actory<CounterMsg> { ... for (msg in channel) ...}`，並將發送通道結合到結果的工作物件 `val counter = counterActor()`，以便於 actor 的單一引用 channel 可以被作為它的句柄攜帶。

The first step of using an actor is to define a class of messages that an actor is going to process. Kotlin's [sealed classes](https://github.com/RayCZ/kotlin-web-site/blob/ray/pages/docs/reference/sealed-classes.md) are well suited for that purpose. We define `CounterMsg` sealed class with `IncCounter` message to increment a counter and `GetCounter` message to get its value. The later needs to send a response. A [CompletableDeferred][CompletableDeferred] communication primitive, that represents a single value that will be known (communicated) in the future, is used here for that purpose.

使用 actor 的第一步是定義 actor 將要處理的訊息類別。 Kotlin 的[密封類別](https://github.com/RayCZ/kotlin-web-site/blob/ray/pages/docs/reference/sealed-classes.md)非常適合這個目的。我們使用 `IncCounter` 訊息類別去增量計數器，並 `GetCounter` 訊息類別去獲取它的結果值，具體的定義 `CounterMsg` 密封類別。後者 (`GetCounter`) 需要發送一個回應 `msg.response.complete(counter) ` 。 [CompletableDeferred][CompletableDeferred] 類型通訊原始資料，表示在將來已知 (通訊) 的單一值，此處用於此目的。

```kotlin
// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply
```

Then we define a function that launches an actor using an [actor][actor] coroutine builder:

接著我們定義一個函數，該函數使用一個 [actor][actor] 協程建造者發射一個 actor。 

```kotlin
// This function launches a new counter actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor state
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

The main code is straightforward:

主要代碼直截了當 (簡單、坦率的)：

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlin.system.*

suspend fun CoroutineScope.massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        val jobs = List(n) {
            launch {
                repeat(k) { action() }
            }
        }
        jobs.forEach { it.join() }
    }
    println("Completed ${n * k} actions in $time ms")    
}


// 定義訊息類別
// Message types for counterActor
sealed class CounterMsg
object IncCounter : CounterMsg() // one-way message to increment counter
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // a request with reply

// actor 函數回傳 SendChannel 類型，所以可以讓別的協程負責送資料，而 actor 負責消費處理
// This function launches a new counter actor
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // actor state
    
    // 當 channel 有東西進來時，判斷類型做處理
    for (msg in channel) { // iterate over incoming messages
        when (msg) {
            
            // 增量處理
            is IncCounter -> counter++ 
            
            // CompletableDeferred 類型送出結果，await 才能取值
            is GetCounter -> msg.response.complete(counter) 
        }
    }
}

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 回傳一個通道讓別的協程可以送資料
    val counter = counterActor() // create the actor
    
    // 利用通道送出大量的送 IncCounter 訊息類
    // 在 actor 程式內判斷類型是 IncCounter 就做數字增量
    GlobalScope.massiveRun {
        counter.send(IncCounter)
    }
    
    // CompletableDeferred 類型讓別的協程呼叫 complete 帶上參數結果，讓 await 取值
    // send a message to get a counter value from an actor
    val response = CompletableDeferred<Int>()
    
    // 利用通道送出 GetCounter 訊息類 (包含 CompletableDeferred 類型)
    // 在 actor 程式內判斷類型是 GetCounter 就做結果處理
    counter.send(GetCounter(response))
    
    // 在 main 協程持續的等待結果， CompletableDeferred 必須呼叫 complete() 才會有結果
    println("Counter = ${response.await()}")
    counter.close() // shutdown the actor
//sampleEnd    
}

//ans:
//Completed 100000 actions in 1031 ms
//Counter = 100000
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-07.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-sync-07.kt)獲取完整的代碼

It does not matter (for correctness) what context the actor itself is executed in. An actor is a coroutine and a coroutine is executed sequentially, so confinement of the state to the specific coroutine works as a solution to the problem of shared mutable state. Indeed, actors may modify their own private state, but can only affect each other through messages (avoiding the need for any locks).

actor 本身執行在什麼環境不重要。 actor 是一個協程並且一個協程被依序執行 (因為是 Channel 負責處理訊息)，所以狀態限制到特定協程適用共享可變狀態問題的解決方式。實際上 ，actor 可以修改它們擁有的私有狀態，只能透過訊息互相影響 (避免任何鎖)。

**actor 本身是一個 Channel 類型，已經採用依序性的執行方式。**

Actor is more efficient than locking under load, because in this case it always has work to do and it does not have to switch to a different context at all.

Actor 在負載下比鎖的方式更有效，因為在這種情況下它總是有工作去做，並且它根本不必切換到不同的協程環境。

> Note, that an [actor][actor] coroutine builder is a dual of [produce][produce] coroutine builder. An actor is associated with the channel that it receives messages from, while a producer is associated with the channel that it  sends elements to.
>
> 注意： 一個 [actor][actor] 協程建造者是一個 [produce][produce] 協程建造者的雙重。一個執行者與通道連繫，執行者從通道接收訊息，而一個生產者與通道連繫，生產者發送元素到通道。
>
> **actor 函數內負責消費 / 消耗，回傳類型負責生產 / 發送**
>
> **produce 函數內負責生產 / 發送，回傳類型負責消費 / 消耗**

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->

[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[CoroutineScope()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[CompletableDeferred]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/index.html
<!--- INDEX kotlinx.coroutines.sync -->
[Mutex]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/index.html
[Mutex.lock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/lock.html
[Mutex.unlock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/-mutex/unlock.html
[withLock]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.sync/with-lock.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
<!--- END -->