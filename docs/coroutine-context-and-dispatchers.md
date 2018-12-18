## Table of contents

* [Coroutine context and dispatchers](#coroutine-context-and-dispatchers)
  * [Dispatchers and threads](#dispatchers-and-threads)
  * [Unconfined vs confined dispatcher](#unconfined-vs-confined-dispatcher)
  * [Debugging coroutines and threads](#debugging-coroutines-and-threads)
  * [Jumping between threads](#jumping-between-threads)
  * [Job in the context](#job-in-the-context)
  * [Children of a coroutine](#children-of-a-coroutine)
  * [Parental responsibilities](#parental-responsibilities)
  * [Naming coroutines for debugging](#naming-coroutines-for-debugging)
  * [Combining context elements](#combining-context-elements)
  * [Cancellation via explicit job](#cancellation-via-explicit-job)
  * [Thread-local data](#thread-local-data)

## Coroutine context and dispatchers

Coroutine context and dispatchers ：協程執行環境與分配器

Coroutines always execute in some context which is represented by the value of [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) type, defined in the Kotlin standard library.

協程總是在某種環境下執行，執行環境由 Kotlin 標準函式庫中定義的 [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 類型值表示。

The coroutine context is a set of various elements. The main elements are the [Job][Job] of the coroutine, which we've seen before, and its dispatcher, which is covered in this section.

協程執行環境是一組各種元素。主要元素是我們之前看過的協程 [Job][Job] 及它的分配器，在本章節中涵蓋。 

### Dispatchers and threads

Dispatchers and threads 分配器和線程

Coroutine context includes a _coroutine dispatcher_ (see [CoroutineDispatcher][CoroutineDispatcher]) that determines what thread or threads the corresponding coroutine uses for its execution. Coroutine dispatcher can confine coroutine execution to a specific thread, dispatch it to a thread pool, or let it run unconfined. 

協程執行環境包括**協程分配器** (參閱 [CoroutineDispatcher][CoroutineDispatcher]) 決定協程相對應的單線程或多線程用於它的執行。協程分配器可以限制協程執行於特定線程，分配它到線程池，或讓它無限制的執行。

All coroutine builders like [launch][launch] and [async][async] accept an optional [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) parameter that can be used to explicitly specify the dispatcher for new coroutine and other context elements. 

所有協程建造者像 [launch][launch] 或 [async][async] 接受一個可選的 [CoroutineContext](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/) 參數 ，參數被用來指定分配器給新的協程以及其他的執行環境元素。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // context(協程執行環境) 依上層的元素決定 (runBlocking) 是在主線程運行
    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    
    // context(協程執行環境) 沒有限制值接在主線程運行
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    
    // context(協程執行環境) 指定預設分配器中的線程運行
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    
    // context(協程執行環境) 指定一個自建的線程來運行
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-01.kt)獲取完整的代碼

It produces the following output (maybe in different order):

它產生以下輸出 (也許是不同順序) ：

```text
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

When `launch { ... }` is used without parameters, it inherits the context (and thus dispatcher) from the [CoroutineScope][CoroutineScope] that it is being launched from. In this case, it inherits the context of the main `runBlocking` coroutine which runs in the `main` thread. 

當無參數的使用`launch { ... }` 時，它從被正在發射的 [CoroutineScope][CoroutineScope] 中繼承執行環境 (以及分配器)。在這樣的情況下，它繼承在 `main` 線程中執行的主要 `runBlocking` 協程的執行環境。

[Dispatchers.Unconfined][Dispatchers.Unconfined] is a special dispatcher that also appears to run in the `main` thread, but it is, in fact, a different mechanism that is explained later.

[Dispatchers.Unconfined][Dispatchers.Unconfined]  是特別的分配器，也似乎在 `main` 線程中運行，但它實際上是一種不同的機制，稍後會解釋。

The default dispatcher, that is used when coroutines are launched in [GlobalScope][GlobalScope], is represented by [Dispatchers.Default][Dispatchers.Default] and uses shared background pool of threads, so `launch(Dispatchers.Default) { ... }` uses the same dispatcher as `GlobalScope.launch { ... }`.

當在 [GlobalScope][GlobalScope] 中發射協程時，使用預設的分配器，由 [Dispatchers.Default][Dispatchers.Default] 表示並使用共享背景程序的線程池，所以 `launch(Dispatchers.Default) { ... }` 使用與 `GlobalScope.launch { ... }` 相同的分配器。

[newSingleThreadContext][newSingleThreadContext] creates a new thread for the coroutine to run. A dedicated thread is a very expensive resource. In a real application it must be either released, when no longer needed, using [close][ExecutorCoroutineDispatcher.close] function, or stored in a top-level variable and reused throughout the application.  

[newSingleThreadContext][newSingleThreadContext]  創建一個新的線程用於協程運行。一個專用的線程是非常昂貴的資源。在實際應用程式中，當不再需要運行時，必須釋放，使用 [close][ExecutorCoroutineDispatcher.close] 函數，或儲存在最高層級中，並遍布整個應用程式重覆使用。

### Unconfined vs confined dispatcher

Unconfined vs confined dispatcher ：不受限制的 (不受拘束的、自由的) vs 受限制的 分配器

The [Dispatchers.Unconfined][Dispatchers.Unconfined] coroutine dispatcher starts coroutine in the caller thread, but only until the first suspension point. After suspension it resumes in the thread that is fully determined by the suspending function that was invoked. Unconfined dispatcher is appropriate when coroutine does not consume CPU time nor updates any shared data (like UI) that is confined to a specific thread. 

在調用者的線程中 [Dispatchers.Unconfined][Dispatchers.Unconfined] 協程分配器啟動協程，但只在第一個懸掛點之前。懸掛之後，它在線程中恢復，由被調用的懸掛函數完全決定線程。不受限制的分配器適用於，當協程不消耗 CPU 時間也不更新任何共享資料 (像是 UI) ，它被受限制在特定的線程時。

On the other side, by default, a dispatcher for the outer [CoroutineScope][CoroutineScope] is inherited. The default dispatcher for [runBlocking][runBlocking] coroutine, in particular, is confined to the invoker thread, so inheriting it has the effect of confining execution to this thread with a predictable FIFO scheduling.

另一方面，根據預設，對於繼承外部的 [CoroutineScope][CoroutineScope] 分配器。特別是， 對於 [runBlocking][runBlocking] 協程的預設分配器只受限於調用者線程，因此繼承它有受限制的執行效果，使用可預測的先進先出排程的線程。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        
        // 不受限制的協程，一開始在主線程
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        
        // 後來在懸掛點後，在預設的執行者線程
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }
    launch { // context of the parent, main runBlocking coroutine
        
        // 受限制的協程，都一直在主線程
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-02.kt)獲取完整的代碼

Produces the output: 

產生輸出：

```text
Unconfined      : I'm working in thread main
main runBlocking: I'm working in thread main
Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
main runBlocking: After delay in thread main
```

So, the coroutine that had inherited context of `runBlocking {...}` continues to execute in the `main` thread, while the unconfined one had resumed in the default executor thread that [delay][delay] function is using.

所以，協程已繼承 `runBlocking {...}` 協程的執行環境在主線程中執行，而不受限制的協程在 [delay][delay] 函數使用時的預設執行者線程已恢復。

> Unconfined dispatcher is an advanced mechanism that can be helpful in certain corner cases where dispatching of coroutine for its execution later is not needed or produces undesirable side-effects, because some operation in a coroutine must be performed right away. Unconfined dispatcher should not be used in general code. 
>
> 不受限制的分配器是一種進階的機制，它在某些極端情況下可能有幫助，在極端情況下，分配它執行的協程是稍後不被需要的或產生不良的副作用，因為在協程中某些操作必須立即被執行。不受限制的分配器在一般的代碼中不應該使用。

### Debugging coroutines and threads

Debugging coroutines and threads ：執行協程和線程的除錯

Coroutines can suspend on one thread and resume on another thread. Even with a single-threaded dispatcher it might be hard to figure out what coroutine was doing, where, and when. The common approach to debugging applications with threads is to print the thread name in the log file on each log statement. This feature is universally supported by logging frameworks. When using coroutines, the thread name alone does not give much of a context, so `kotlinx.coroutines` includes debugging facilities to make it easier. 

協程可以懸掛一個線程並恢復其他的線程。即使使用單線程的分配器，它可能很難指出協程正在做什麼，那時和那處。使用線程除錯應用程式常用的方式是在日誌檔中的每個日誌敘述印出線程名稱。這個功能透過日誌框架普遍支援。當使用協程時，單獨的線程名稱不會給出很多的內容，因此 `kotlinx.coroutines` 包括除錯工具，使它更容易使用。

Run the following code with `-Dkotlinx.coroutines.debug=on/off` JVM option:

使用 `-Dkotlinx.coroutines.debug=on/off` JVM 選項執行以下代碼：

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
//sampleStart
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-03.kt)獲取完整的代碼

There are three coroutines. The main coroutine (#1) -- `runBlocking` one, and two coroutines computing deferred values `a` (#2) and `b` (#3). They are all executing in the context of `runBlocking` and are confined to the main thread. The output of this code is:

有三個協程。主協程 (#1) -- `runBlocking` 協程，以及兩個協程正在計算推遲的值 `a` (#2) 和 `b` (#3) 。他們在 `runBlocking` 的執行環境下執行，並受限於主線程。這些代碼輸出是：

```text
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

The `log` function prints the name of the thread in square brackets and you can see, that it is the `main` thread, but the identifier of the currently executing coroutine is appended to it. This identifier is consecutively assigned to all created coroutines when debugging mode is turned on.

`log`函數在方括號中印出線程的名稱，並且你可以看到它是 `main` 線程，而且目前正在執行的協程識別附加到 `main` 線程，當除錯模式被打開時，這個識別是連續被分配到所有已創建的協程。

You can read more about debugging facilities in the documentation for [newCoroutineContext][newCoroutineContext] function.

你可以在 [newCoroutineContext][newCoroutineContext] 函數的文件中閱讀多更多有關除錯工具。

### Jumping between threads

Jumping between threads ：在協程之間跳轉：

Run the following code with `-Dkotlinx.coroutines.debug=on/off` JVM option (see [debug](#debugging-coroutines-and-threads)):

使用 `-Dkotlinx.coroutines.debug=on/off` JVM 選項執行以下代碼 (看 [debug](#debugging-coroutines-and-threads))：

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() {
//sampleStart
    
    // 建立一個線程並指定 Ctx1 名稱
    newSingleThreadContext("Ctx1").use { ctx1 ->
                                        
        // 建立一個線程並指定 Ctx2 名稱
        newSingleThreadContext("Ctx2").use { ctx2 ->
                                            
            // 使用 ctx1 的環境來執行協程
            runBlocking(ctx1) {
                log("Started in ctx1")
                
                // 使用 ctx2 的環境來執行協程
                withContext(ctx2) {
                    log("Working in ctx2")
                }
                log("Back to ctx1")
            }
        }
    }
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-04.kt)獲取完整的代碼

It demonstrates several new techniques. One is using [runBlocking][runBlocking] with an explicitly specified context, and the other one is using [withContext][withContext] function to change a context of a coroutine while still staying in the same coroutine as you can see in the output below:

它展示幾種新技術。一個是使用 [runBlocking][runBlocking] 明確指定執行環境 ，而另一個是使用 [withContext][withContext] 函數改變協程的執行環境，還是停留在與你在下面輸出可以看到的協程相同：

```text
[Ctx1 @coroutine#1] Started in ctx1
[Ctx2 @coroutine#1] Working in ctx2
[Ctx1 @coroutine#1] Back to ctx1
```

Note, that this example also uses `use` function from the Kotlin standard library to release threads that are created with [newSingleThreadContext][newSingleThreadContext]  when they are no longer needed. 

注意：這個範例也使用從 Kotlin 標準函式庫中的  `use` 函數，當你不再需要時，來釋放使用 [newSingleThreadContext][newSingleThreadContext] 創建的線程。

### Job in the context

Job in the context ：在執行環境中的 Job

The coroutine's [Job][Job] is part of its context. The coroutine can retrieve it from its own context using `coroutineContext[Job]` expression:

協程的 [Job][Job] 是它執行環境中的一部分。協程可以使用 `coroutineContext[Job]` 表達式從它擁有的環境中獲取它。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    println("My job is ${coroutineContext[Job]}")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-05.kt)獲取完整的代碼

It produces something like that when running in [debug mode](#debugging-coroutines-and-threads):

在[除錯模式](#debugging-coroutines-and-threads)運行時，它產生某些像是：

```
My job is "coroutine#1":BlockingCoroutine{Active}@6d311334
```

Note, that [isActive][isActive] in [CoroutineScope][CoroutineScope] is just a convenient shortcut for `coroutineContext[Job]?.isActive == true`.

注意： [CoroutineScope][CoroutineScope] 中的 [isActive][isActive] 只是 `coroutineContext[Job]?.isActive == true` 的方便快捷方式。

### Children of a coroutine

Children of a coroutine ：協程的後代

When a coroutine is launched in the [CoroutineScope][CoroutineScope] of another coroutine, it inherits its context via [CoroutineScope.coroutineContext][CoroutineScope.coroutineContext] and the [Job][Job] of the new coroutine becomes a _child_ of the parent coroutine's job. When the parent coroutine is cancelled, all its children are recursively cancelled, too. 

當在另一個協程的 [CoroutineScope][CoroutineScope] 中發射一個協程時，它通過 [CoroutineScope.coroutineContext][CoroutineScope.coroutineContext] 繼承它的環境，並且新協程的 [Job][Job] 成為父協程 job 的後代。當父協程被取消，所有它的後代也會遞迴式的被取消。

However, when [GlobalScope][GlobalScope] is used to launch a coroutine, it is not tied to the scope it was launched from and operates independently.

無論如何，當 [GlobalScope][GlobalScope] 被用於發射協程時，從它發射的範圍沒有聯繫而且獨立的操作。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        
        // 利用 GlobalScope 的環境與範圍發射，不會依上層而被取消，是獨立的範圍
        // it spawns two other jobs, one with GlobalScope
        GlobalScope.launch {
            println("job1: I run in GlobalScope and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        
        // 依上層的環境與範圍發射，所以上層取消這裡也會取消
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled")
        }
    }
    
    // 延遲 0.5 秒，然後取消協程
    delay(500)
    request.cancel() // cancel processing of the request
    delay(1000) // delay a second to see what happens
    println("main: Who has survived request cancellation?")
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-06.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出是：

```text
job1: I run in GlobalScope and execute independently!
job2: I am a child of the request coroutine
job1: I am not affected by cancellation of the request
main: Who has survived request cancellation?
```

### Parental responsibilities 

Parental responsibilities  ：父協程的責任

A parent coroutine always waits for completion of all its children. Parent does not have to explicitly track all the children it launches and it does not have to use [Job.join][Job.join] to wait for them at the end:

父協程總是等待所有它的子協程完成。父協程不必明確追蹤它發射的所有子協程，也不必使用 [Job.join][Job.join] 在結束時等待它們：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 父協程
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        
        // 重覆發射三個子協程
        repeat(3) { i -> // launch a few children jobs
            launch  {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        
        // 不用等待子協程
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    
    // 等待子協程全部完成
    request.join() // wait for completion of the request, including all its children
    println("Now processing of the request is complete")
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-07.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-07.kt)獲取完整的代碼

The result is going to be:

結果將是：

```text
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```

### Naming coroutines for debugging

Naming coroutines for debugging ：命名協程用於除錯

Automatically assigned ids are good when coroutines log often and you just need to correlate log records coming from the same coroutine. However, when coroutine is tied to the processing of a specific request or doing some specific background task, it is better to name it explicitly for debugging purposes. [CoroutineName][CoroutineName] context element serves the same function as a thread name. It'll get displayed in the thread name that is executing this coroutine when [debugging mode](#debugging-coroutines-and-threads) is turned on.

當協程經常記錄時，自動分配 ID 很好，並且你只需要關聯來自相同協程的日誌記錄。然而，當協程與特定請求的處理有聯繫時，或做某些特定背景程序任務時，對於除錯目的明確命名它是很好的。 [CoroutineName][CoroutineName] 環境元素提供與線程名稱相同函數。當 [debugging mode](#debugging-coroutines-and-threads) 開啟時，它將在顯示正在執行這個協程的線程名稱。

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

// 為 runBlocking 命名稱 main
fun main() = runBlocking(CoroutineName("main")) {
//sampleStart
    log("Started main coroutine")
    
    // 命名第一個子協程 v1coroutine
    // run two background value computations
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    
    // 命名第二個子協程 v2coroutine
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-08.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-08.kt)獲取完整的代碼

The output it produces with `-Dkotlinx.coroutines.debug` JVM option is similar to:

使用 `-Dkotlinx.coroutines.debug` JVM 選項它產生輸出類似於：

```text
[main @main#1] Started main coroutine
[main @v1coroutine#2] Computing v1
[main @v2coroutine#3] Computing v2
[main @main#1] The answer for v1 / v2 = 42
```

### Combining context elements

Combining context elements ：結合環境元素

Sometimes we need to define multiple elements for coroutine context. We can use `+` operator for that. For example, we can launch a coroutine with an explicitly specified dispatcher and an explicitly specified name at the same time: 

有時我們需要為協程環境定義多個元素。我們可以使用 `+` 運算符。例如，我們可以同時明確指定分配器和明確指定名稱發射一個協程：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 指定協程分配器和命稱
    launch(Dispatchers.Default + CoroutineName("test")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-09.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-09.kt)獲取完整的代碼

The output of this code  with `-Dkotlinx.coroutines.debug` JVM option is: 

這代碼使用 `-Dkotlinx.coroutines.debug` JVM 選項的輸出是：

```text
I'm working in thread DefaultDispatcher-worker-1 @test#2
```

### Cancellation via explicit job

Cancellation via explicit job ：經由明確的 job 物件取消

Let us put our knowledge about contexts, children and jobs together. Assume that our application has an object with a lifecycle, but that object is not a coroutine. For example, we are writing an Android application and launch various coroutines in the context of an Android activity to perform asynchronous operations to fetch and update data, do animations, etc. All of these coroutines must be cancelled when activity is destroyed to avoid memory leaks. 

讓我們把有關環境、子協程、[Job][Job] 的知識放一起。假設我們的應用程式有一個生命週期的物件，但該物件不是協程。例如，我們正在寫 Android 應用程式並在 Android Activity 的環境中發射各種協程，去執行異步的操作去獲取或更新資料、做動畫等等。當 Activity 被銷毀時，這些所有的協程必須被取消，來避免記憶體洩漏。

We manage a lifecycle of our coroutines by creating an instance of [Job][Job] that is tied to the lifecycle of our activity. A job instance is created using [Job()][Job()] factory function when activity is created and it is cancelled when an activity is destroyed like this:

我們透過創建與我們 Activity 生命週期相關的 [Job][Job] 實例，管理我們協程的生命週期。 當 `Activity` 被創建時，使用 [Job()][Job()] 工廠函數創建 job 實例，當 `Activity` 被銷毀時，取消 job，像這樣：

```kotlin
class Activity : CoroutineScope {
    lateinit var job: Job

    // 在 Activity 生命週期創建時，建立 job
    fun create() {
        job = Job()
    }

    // 在 Activity 生命週期創建時，取消 job
    fun destroy() {
        job.cancel()
    }
    // to be continued ...
```

We also implement [CoroutineScope][CoroutineScope] interface in this `Actvity` class. We only need to provide an override for its [CoroutineScope.coroutineContext][CoroutineScope.coroutineContext] property to specify the context for coroutines launched in its scope. We combine the desired dispatcher (we used [Dispatchers.Default][Dispatchers.Default] in this example) and a job:

我們在這個 `Activity` 類別中也實作 [CoroutineScope][CoroutineScope] 介面。我們只需要為它的 [CoroutineScope.coroutineContext][CoroutineScope.coroutineContext] 屬性提供 override (覆寫) ，去指定在它範圍中發射的協程環境。我們組合需要的分配器 (我們在範例中使用 [Dispatchers.Default][Dispatchers.Default]) 和 job ：

```kotlin
    // class Activity continues
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job
    // to be continued ...
```

Now, we can launch coroutines in the scope of this `Activity` without having to explicitly specify their context. For the demo, we launch ten coroutines that delay for a different time:

現在，我們可以在這個 `Activity` 的範圍中發射協程，不必明確指定它的環境。如示範，我們不同時間的延遲發射十個協程：

```kotlin
    // class Activity continues
    fun doSomething() {
        
        // 重覆發射十個協程並延遲
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends
```

In our main function we create activity, call our test `doSomething` function, and destroy activity after 500ms. This cancels all the coroutines that were launched which we can confirm by noting that it does not print onto the screen anymore if we wait: 

我們的 `main` 函數，我們創建 activity 實例，調用我們測試 `doSomething` 函數，並延遲 0.5 秒後銷毀 activity ，這取消所有被發射的協程，我們可以透過提示，如果我們等待它不會再印出到畫面上來確定：

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class Activity : CoroutineScope {
    lateinit var job: Job

    fun create() {
        job = Job()
    }

    fun destroy() {
        job.cancel()
    }
    // to be continued ...

    // class Activity continues
    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Default + job
    // to be continued ...

    // class Activity continues
    fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 建立 activity 實例，呼叫生命週期、做事情、銷毀
    val activity = Activity()
    activity.create() // create an activity
    activity.doSomething() // run test function
    println("Launched coroutines")
    delay(500L) // delay for half a second
    println("Destroying activity!")
    activity.destroy() // cancels all coroutines
    delay(1000) // visually confirm that they don't work
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-10.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-10.kt)獲取完整的代碼

The output of this example is:

這個範例的輸出是：

```text
Launched coroutines
Coroutine 0 is done
Coroutine 1 is done
Destroying activity!
```

As you can see, only the first two coroutines had printed a message and the others were cancelled by a single invocation of `job.cancel()` in `Activity.destroy()`.

如你可以看到的。只有首要兩個協程已印出訊息，而在 `Activity.destroy()` 中單獨調用 `job.cancel()` 取消其他的協程。

### Thread-local data

Thread-local data ：線程 - 區域資料，每個線程獨立的儲存區

Sometimes it is convenient to have an ability to pass some thread-local data, but, for coroutines, which are not bound to any particular thread, it is hard to achieve it manually without writing a lot of boilerplate.

有時能夠傳遞一些線程 - 區域資料是方便的。但對於沒有受約束於特定線程的協程，不寫很多模版程式，很難去手動的獲取它。

For [`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html), [asContextElement][asContextElement] extension function is here for the rescue. It creates an additional context element, which keeps the value of the given `ThreadLocal` and restores it every time the coroutine switches its context.

對於 [`ThreadLocal`](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html) 類型， [asContextElement][asContextElement] 擴展函數在這裡用於救援。它創建額外的環境元素，它保留已給的 `ThreadLocal` 值，並協程每次切換它的環境時儲存它。

It is easy to demonstrate it in action:

在實踐中，容易展示它：

```kotlin
import kotlinx.coroutines.*

val threadLocal = ThreadLocal<String?>() // declare thread-local variable

fun main() = runBlocking<Unit> {
//sampleStart
    
    // 設置線程區域值
    threadLocal.set("main")
    println("Pre-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    
    // 發射協程，使用 asContextElement 擴展函數創建環境元素並切換線程時保留上一個 ThreadLocal 值
    val job = launch(Dispatchers.Default + threadLocal.asContextElement(value = "launch")) {
       println("Launch start, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
        
        // 讓出 (切換) 線程，，把保留 ThreadLocal 值回存回去
        yield()
        println("After yield, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
    }
    job.join()
    println("Post-main, current thread: ${Thread.currentThread()}, thread local value: '${threadLocal.get()}'")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-11.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-context-11.kt)獲取完整的代碼

In this example we launch new coroutine in a background thread pool using [Dispatchers.Default][Dispatchers.Default], so it works on a different threads from a thread pool, but it still has the value of thread local variable, that we've specified using `threadLocal.asContextElement(value = "launch")`, no matter on what thread the coroutine is executed.Thus, output (with [debug](#debugging-coroutines-and-threads)) is:

有這個範例中，我們在背景線程池中使用 [Dispatchers.Default][Dispatchers.Default] 發射新的協程，所以它適用於不同線程池的線程，但它還是有線程區域變數的值，我們已指定使用 `threadLocal.asContextElement(value = "launch")` ，無論執行協程在什麼線程。因此，輸出 (使用 [debug](#debugging-coroutines-and-threads)) 是：

```text
Pre-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
Launch start, current thread: Thread[DefaultDispatcher-worker-1 @coroutine#2,5,main], thread local value: 'launch'
After yield, current thread: Thread[DefaultDispatcher-worker-2 @coroutine#2,5,main], thread local value: 'launch'
Post-main, current thread: Thread[main @coroutine#1,5,main], thread local value: 'main'
```

`ThreadLocal` has first-class support and can be used with any primitive `kotlinx.coroutines` provides. It has one key limitation: when thread-local is mutated, a new value is not propagated to the coroutine caller (as context element cannot track all `ThreadLocal` object accesses) and updated value is lost on the next suspension. Use [withContext][withContext] to update the value of the thread-local in a coroutine, see [asContextElement][asContextElement] for more details. 

`ThreadLocal` 已經支援頭等函數，並可以與任何原生提供的 `kotlinx.coroutines` 一起使用。它有一個關鍵的限制：當線程區域是可變的，新值不會傳播到協程調用者 (因為環境元素不可以追蹤所有 `ThreadLocal` 物件存取) 並且更新值在下一個懸掛 (暫停) 時丟失。使用 [withContext][withContext] 去更新在協程中的線程區域值，更多詳細請參閱 [asContextElement][asContextElement]。

Alternatively, a value can be stored in a mutable box like `class Counter(var i: Int)`, which is, in turn, stored in a thread-local variable. However, in this case you are fully responsible to synchronize potentially concurrent modifications to the variable in this mutable box.

另外，值可以被儲存在可變的裝箱中像 `class Counter(var i: Int)`，反過來，它存儲在線程區域變數。然而，在這樣的情況下，你有完全的責任在這樣可變的裝箱中，同步潛在的並發(同時)修改變數。

For advanced usage, for example for integration with logging MDC, transactional contexts or any other libraries which internally use thread-locals for passing data, see documentation for [ThreadContextElement][ThreadContextElement] interface that should be implemented. 

對於進階的用法，例如，使用 MDC 函式庫日誌，交易環境或內部使用線程區域用於傳遞資料的任何其他函式庫整合， 參閱 [ThreadContextElement][ThreadContextElement] 介面應該被實作的文件。

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->

[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[CoroutineDispatcher]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[Dispatchers.Unconfined]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-unconfined.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[newSingleThreadContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-single-thread-context.html
[ExecutorCoroutineDispatcher.close]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-executor-coroutine-dispatcher/close.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[newCoroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/new-coroutine-context.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope.coroutineContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/coroutine-context.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CoroutineName]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-name/index.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[asContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/java.lang.-thread-local/as-context-element.html
[ThreadContextElement]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-thread-context-element/index.html
<!--- END -->
