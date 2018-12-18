## Table of contents

* [Exception handling](#exception-handling)
  * [Exception propagation](#exception-propagation)
  * [CoroutineExceptionHandler](#coroutineexceptionhandler)
  * [Cancellation and exceptions](#cancellation-and-exceptions)
  * [Exceptions aggregation](#exceptions-aggregation)
* [Supervision](#supervision)
  * [Supervision job](#supervision-job)
  * [Supervision scope](#supervision-scope)
  * [Exceptions in supervised coroutines](#exceptions-in-supervised-coroutines)

## Exception handling

Exception handling ：異常處理

This section covers exception handling and cancellation on exceptions. We already know that cancelled coroutine throws [CancellationException][CancellationException] in suspension points and that it is ignored by coroutines machinery. But what happens if an exception is thrown during cancellation or multiple children of the same coroutine throw an exception?

這個章節涵蓋異常處理和在異常中取消。我們已經知道在懸掛點中取消協程會丟出 [CancellationException][CancellationException] 並且被協程機制忽略。但是如果異常在取消期間丟出異常，或相同協程的多個子協程丟出異常會發生什麼事？

### Exception propagation

Exception propagation ：異常傳播 (丟出、拋出)

Coroutine builders come in two flavors: propagating exceptions automatically ([launch][launch] and [actor][actor]) or exposing them to users ([async][async] and [produce][produce]). The former treat exceptions as unhandled, similar to Java's `Thread.uncaughtExceptionHandler`, while the latter are relying on the user to consume the final exception, for example via [await][Deferred.await] or [receive][ReceiveChannel.receive] ([produce][produce] and [receive][ReceiveChannel.receive] are covered later in [Channels](channels.md) section).

協程建造者有兩種風格：自動的傳播 (丟出) 異常 ([launch][launch] 和 [actor][actor]) 或揭露它們給使用者 ([async][async] 和 [produce][produce]) 。前者視異常未處理，類似於 Java 的 `Thread.uncaughtExceptionHandler` ，而後者依賴使用者去消耗處理最終的異常，例如：透過 [await][Deferred.await] 或 [receive][ReceiveChannel.receive] ([produce][produce] 和 [receive][ReceiveChannel.receive] 在 [Channels](channels.md) 章節中稍後涵蓋) 函數去捕獲異常。

It can be demonstrated by a simple example that creates new coroutines in [GlobalScope][GlobalScope]:

可以透過在 [GlobalScope][GlobalScope] 中創建新的協程的簡單例子展示它：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    
    // GlobalScope 發射協程並在之中丟出異常
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Will be printed to the console by Thread.defaultUncaughtExceptionHandler
    }
    
    // 等待 lanch 處理完回到協程
    job.join()
    println("Joined failed job")
    
    // GlobalScope 發射異步協程，等待使用者調用 await 時捕獲異常
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException() // Nothing is printed, relying on user to call await
    }
    
    // async 需要 await() ，處理最終的異常
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-01.kt)獲取完整的代碼

The output of this code is (with [debug](coroutine-context-and-dispatchers.md#debugging-coroutines-and-threads)):

這些代碼的輸出是 (使用 [debug](coroutine-context-and-dispatchers.md#debugging-coroutines-and-threads) 模式)：

```text
Throwing exception from launch
Exception in thread "DefaultDispatcher-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
Joined failed job
Throwing exception from async
Caught ArithmeticException
```

### CoroutineExceptionHandler

CoroutineExceptionHandler ：協程異常處理器

But what if one does not want to print all exceptions to the console? [CoroutineExceptionHandler][CoroutineExceptionHandler] context element is used as generic `catch` block of coroutine where custom logging or exception handling may take place. It is similar to using [`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)).

但是如果一個人不想要印出所有的異常到控制台？ [CoroutineExceptionHandler][CoroutineExceptionHandler] 環境元素用作協程的通用 `catch` 區域，區域中帶入可以自定義日誌記錄或異常處理。它類似於使用 [`Thread.uncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) 。

On JVM it is possible to redefine global exception handler for all coroutines by registering [CoroutineExceptionHandler][CoroutineExceptionHandler] via [`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html). Global exception handler is similar to [`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) which is used when no more specific handlers are registered. On Android, `uncaughtExceptionPreHandler` is installed as a global coroutine exception handler.

在 JVM 上，經由 [`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) 註冊 [CoroutineExceptionHandler][CoroutineExceptionHandler] 能重新定義所有協程的全域異常處理器。全域異常處理器類似於 [`Thread.defaultUncaughtExceptionHandler`](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#setDefaultUncaughtExceptionHandler(java.lang.Thread.UncaughtExceptionHandler)) ，它在沒有註冊更多特定處理器時使用。在 Android 上， `uncaughtExceptionPreHandler` 被安裝作為全域協程異常處理器。

[CoroutineExceptionHandler][CoroutineExceptionHandler] is invoked only on exceptions which are not expected to be handled by the user, so registering it in [async][async] builder and the like of it has no effect.

[CoroutineExceptionHandler][CoroutineExceptionHandler] 只在由使用者處理非預期的異常上調用，所以在 [async][async] 建造者註冊它並且同樣於它沒有效果。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    
    // 創建 CoroutineExceptionHandler 實例
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    
    // 協程發射時順便帶入
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    
    // 協程發射時順便帶入
    val deferred = GlobalScope.async(handler) {
        throw ArithmeticException() // Nothing will be printed, relying on user to call deferred.await()
    }
    joinAll(job, deferred)
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-02.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出是：

```text
Caught java.lang.AssertionError
```

### Cancellation and exceptions

Cancellation and exceptions ：取消和異常

Cancellation is tightly bound with exceptions. Coroutines internally use `CancellationException` for cancellation, these exceptions are ignored by all handlers, so they should be used only as the source of additional debug information, which can be obtained by `catch` block. When a coroutine is cancelled using [Job.cancel][Job.cancel] without a cause, it terminates, but it does not cancel its parent. Cancelling without cause is a mechanism for parent to cancel its children without cancelling itself. 

取消與異常緊密相關。協程內部使用 `CancellationException` 用於取消，所有處理器忽略這些異常，所以它們應用只能被用為額外除錯資訊的來源，透過 `catch` 區域獲取這些資訊。當沒有理由使用 [Job.cancel][Job.cancel] 取消協程時，它被終止，但它不會取消它的父協程。沒有理由取消是一種機制，用於父協程去取它的子協程，父協程無法取消本身。

```kotlin
import kotlinx.coroutines.*

// 第一個協程 runBlocking
fun main() = runBlocking {
//sampleStart
    
    // 第二個協程
    val job = launch {
        
        // 3.第三個協程持續的等待
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Child is cancelled")
            }
        }
        
        // 2.第二協程讓出協程執行權，讓第三協程先執行待等
        yield()
        println("Cancelling child")
        
        // 4.第二協程拿回執行權，取消第三協程持續等待，並發生 CancellationException
        child.cancel()
        
        // 5.等待第三協程處理完訊息才繼續執行
        child.join()
        
        // 6.第二協程讓出執行權，但沒有人執行，又繼續執行
        yield()
        println("Parent is not cancelled")
    }
    
    // 1.在第一個協程中，等待第二協程完成才繼續執行
    job.join()
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-03.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出是：

```text
Cancelling child
Child is cancelled
Parent is not cancelled
```

If a coroutine encounters exception other than `CancellationException`, it cancels its parent with that exception. This behaviour cannot be overridden and is used to provide stable coroutines hierarchies for [structured concurrency](composing-suspending-functions.md#structured-concurrency-with-async) which do not depend on [CoroutineExceptionHandler][CoroutineExceptionHandler] implementation. The original exception is handled by the parent when all its children terminate.

如果協程遇到除了 `CancellationException` 之外的異常，它取消有異常的父協程。這個行為無法被覆寫並且用於[結構化並發](composing-suspending-functions.md#structured-concurrency-with-async)提供穩定的協程階層。它不依賴 [CoroutineExceptionHandler][CoroutineExceptionHandler] 實作。當所有它的子協程中止時，父協程處理原始異常。

> This also a reason why, in these examples, [CoroutineExceptionHandler][CoroutineExceptionHandler] is always installed to a coroutine that is created in [GlobalScope][GlobalScope]. It does not make sense to install an exception handler to a coroutine that is launched in the scope of the main [runBlocking][runBlocking], since the main coroutine is going to be always cancelled when its child completes with exception despite the installed handler. 
>
> 這也是為什麼在這些例子中，[CoroutineExceptionHandler][CoroutineExceptionHandler] 總是被設置到在 [GlobalScope][GlobalScope] 中創建的協程。設置一個異常處理器給在主要 [runBlocking][runBlocking] 協程範圍中發射的協程是無意義的，當它的子協程有異常完成時，儘管已設置處理器，主要協程將總是被取消。

```kotlin
import kotlinx.coroutines.*

// 第一個協程
fun main() = runBlocking {
//sampleStart
    
    //
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    
    // 第二個協程
    val job = GlobalScope.launch(handler) {
        
        // 2.第三個協程，持續的等待
        launch { // the first child
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        
        // 3.第四個協程，丟出 ArithmeticException 異常，而不是 CancellationException 會使整個協程中止
        launch { // the second child
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    
    // 1.在第一個協程中，等待第二協程完成才繼續執行
    job.join()
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-04.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出是：

```text
Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
Caught java.lang.ArithmeticException
```

### Exceptions aggregation

Exceptions aggregation ：異常的聚集，表示當發生多異常會以那個異常處理

What happens if multiple children of a coroutine throw an exception? The general rule is "the first exception wins", so the first thrown exception is exposed to the handler. But that may cause lost exceptions, for example if coroutine throws an exception in its `finally` block. So, additional exceptions are suppressed. 

如果一個協程的多個子協程丟出異常會發生什麼事？一般規則是 "第一個異常獲勝" ，所以第一個丟出異常是揭露給處理器。但這樣可能導致遺失異常，例如，如果協程在它的 `final` 區域中丟出異常。所以，額外異常被抑制了。

> One of the solutions would have been to report each exception separately, but then [Deferred.await][Deferred.await] should have had the same mechanism to avoid behavioural inconsistency and this would cause implementation details of a coroutines (whether it had delegated parts of its work to its children or not) to leak to its exception handler.
>
> 已有的解決方式之一，是單獨報告每個異常，而接著 [Deferred.await][Deferred.await] 應該有相同的機制來避免行為不一致，並且這會導致協程的實作細節 (是否已委外它部分的工作給它的子協程或否) 會洩漏給它的異常處理器。

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    
    // 設置協程處理器
    val job = GlobalScope.launch(handler) {
        
        // 2.第一個協程先等待，並讓第二個協程丟出錯誤，中止整個協程
        launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                throw ArithmeticException()
            }
        }
        
        // 1.第二個協程先丟出 IOException ，造成第一個協程中止也丟出 throw ArithmeticException()
        launch {
            delay(100)
            throw IOException()
        }
        delay(Long.MAX_VALUE)
    }
    
    // 等待子協程完成
    job.join()  
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-05.kt)獲取完整的代碼

> Note: This above code will work properly only on JDK7+ that supports `suppressed` exceptions
>
> 注意：以上代碼只適用於支援 `suppressed` 異常的 JDK7+ 。

The output of this code is:

這些代碼的輸出是：

```text
Caught java.io.IOException with suppressed [java.lang.ArithmeticException]
```

> Note, this mechanism currently works only on Java version 1.7+. Limitation on JS and Native is temporary and will be fixed in the future.
>
> 注意：目前這種機制只適用於 Java 版本 1.7+ 。對於 JS 和 Native 版本暫時限制，並且將在未來修正。

Cancellation exceptions are transparent and unwrapped by default:

取消異常是透明的並且根據預設解開：

```kotlin
import kotlinx.coroutines.*
import java.io.*

fun main() = runBlocking {
//sampleStart
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught original $exception")
    }
    
    // 造成外部中止
    val job = GlobalScope.launch(handler) {
        
        // 以內層方式創建協程，但發生異常，還是會一層一層的丟出來 (解開)
        // 造成外部中止
        val inner = launch {
            
            // 造成外部中止
            launch {
                
                // 內部丟出異常
                launch {
                    throw IOException()
                }
            }
        }
        try {
            inner.join()
        } catch (e: CancellationException) {
            println("Rethrowing CancellationException with original cause")
            
            // 重丟異常，但取得的還是 IOException ，而不是 CancellationException
            throw e
        }
    }
    job.join()
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-exceptions-06.kt)獲取完整的代碼

The output of this code is:

這些代碼的輸出是：

```text
Rethrowing CancellationException with original cause
Caught original java.io.IOException
```

## Supervision

As we have studied before, cancellation is a bidirectional relationship propagating through the whole
coroutines hierarchy. But what if unidirectional cancellation is required? 

Good example of such requirement can be a UI component with the job defined in its scope. If any of UI's child task
has failed, it is not always necessary to cancel (effectively kill) the whole UI component,
but if UI component is destroyed (and its job is cancelled), then it is necessary to fail all children jobs as their result is no longer required.

Another example is a server process that spawns several children jobs and needs to _supervise_
their execution, tracking their failures and restarting just those children jobs that had failed.

### Supervision job

For these purposes [SupervisorJob][SupervisorJob()] can be used. It is similar to a regular [Job][Job()] with the only exception that cancellation is propagated
only downwards. It is easy to demonstrate with an example:

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // launch the first child -- its exception is ignored for this example (don't do this in practice!)
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("First child is failing")
            throw AssertionError("First child is cancelled")
        }
        // launch the second child
        val secondChild = launch {
            firstChild.join()
            // Cancellation of the first child is not propagated to the second child
            println("First child is cancelled: ${firstChild.isCancelled}, but second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // But cancellation of the supervisor is propagated
                println("Second child is cancelled because supervisor is cancelled")
            }
        }
        // wait until the first child fails & completes
        firstChild.join()
        println("Cancelling supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-supervision-01.kt)

The output of this code is:

```text
First child is failing
First child is cancelled: true, but second one is still active
Cancelling supervisor
Second child is cancelled because supervisor is cancelled
```
<!--- TEST-->


### Supervision scope

For *scoped* concurrency [supervisorScope] can be used instead of [coroutineScope] for the same purpose. It propagates cancellation 
only in one direction and cancels all children only if it has failed itself. It also waits for all children before completion
just like [coroutineScope] does.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("Child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("Child is cancelled")
                }
            }
            // Give our child a chance to execute and print using yield 
            yield()
            println("Throwing exception from scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught assertion error")
    }
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-supervision-02.kt)

The output of this code is:

```text
Child is sleeping
Throwing exception from scope
Child is cancelled
Caught assertion error
```
<!--- TEST-->

### Exceptions in supervised coroutines

Another crucial difference between regular and supervisor jobs is exception handling.
Every child should handle its exceptions by itself via exception handling mechanisms.
This difference comes from the fact that child's failure is not propagated to the parent.

<div class="sample" markdown="1" theme="idea" data-highlight-only>

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception -> 
        println("Caught $exception") 
    }
    supervisorScope {
        val child = launch(handler) {
            println("Child throws an exception")
            throw AssertionError()
        }
        println("Scope is completing")
    }
    println("Scope is completed")
}
```

</div>

> You can get full code [here](../core/kotlinx-coroutines-core/test/guide/example-supervision-03.kt)

The output of this code is:

```text
Scope is completing
Child throws an exception
Caught java.lang.AssertionError
Scope is completed
```
<!--- TEST-->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[Deferred.await]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/await.html
[GlobalScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html
[CoroutineExceptionHandler]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[runBlocking]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html
[SupervisorJob()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html
[Job()]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job.html
[supervisorScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[produce]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/produce.html
[ReceiveChannel.receive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-receive-channel/receive.html
<!--- END -->
