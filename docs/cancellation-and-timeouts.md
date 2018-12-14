## Table of contents

* [Cancellation and timeouts](#cancellation-and-timeouts)
  * [Cancelling coroutine execution](#cancelling-coroutine-execution)
  * [Cancellation is cooperative](#cancellation-is-cooperative)
  * [Making computation code cancellable](#making-computation-code-cancellable)
  * [Closing resources with finally](#closing-resources-with-finally)
  * [Run non-cancellable block](#run-non-cancellable-block)
  * [Timeout](#timeout)

## Cancellation and timeouts

Cancellation and timeouts ：取消和超時

This section covers coroutine cancellation and timeouts.

本章節涵蓋協程的取消和超時。

### Cancelling coroutine execution

Cancelling coroutine execution ：取消協程執行

In a long-running application you might need fine-grained control on your background coroutines. For example, a user might have closed the page that launched a coroutine and now its result is no longer needed and its operation can be cancelled. The [launch][launch] function returns a [Job][Job] that can be used to cancel running coroutine:

在一個長時間運行的應用程式，在你的背景協程中你可能需要細粒度的控制。例如，一個使用者或許已經關閉已發射協程的畫面，然而現在它的結果不再需要和它的操作可以被取消。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancel() // cancels the job
    job.join() // waits for job's completion 
    println("main: Now I can quit.")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-01.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-01.kt)獲取完整的代碼

It produces the following output:

它產生以下輸出：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

As soon as main invokes `job.cancel`, we don't see any output from the other coroutine because it was cancelled. There is also a [Job][Job.join] extension function [cancelAndJoin][cancelAndJoin]  that combines [cancel][Job.cancel] and [join][Job.join] invocations.

一旦在 main 函數中調用 `job.cancel` ，我們不需要從其他的協程看到任何輸出，因為它被取消了。也有 [Job][Job.join] 擴展函數 [cancelAndJoin][cancelAndJoin] 結合 [cancel][Job.cancel] 和 [join][Job.join] 的調用。

### Cancellation is cooperative

Cancellation is cooperative ：取消是合作的 (cancel + join 操作)

Coroutine cancellation is _cooperative_. A coroutine code has to cooperate to be cancellable. All the suspending functions in `kotlinx.coroutines` are _cancellable_. They check for cancellation of coroutine and throw [CancellationException][CancellationException] when cancelled. However, if a coroutine is working in a computation and does not check for cancellation, then it cannot be cancelled, like the following example shows:

協程取消是合作的。協程代碼必須配合才能取消。在 `kotlinx.coroutines` 中所有的懸掛函數是可取消的。他檢查協程的取消並當已取消時丟出 [CancellationException][CancellationException] 。然而，如果協程正在計算工作並不會檢查取消，接著它不可以被取消，像以下範例所示：


```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        
        // while 當中持續的計算過程，不會因為取消協程而停止， join 會等到執行完才回原本的協程
        while (i < 5) { // computation loop, just wastes CPU
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
//sampleEnd    
}

//ans:
//I'm sleeping 0 ...
//I'm sleeping 1 ...
//I'm sleeping 2 ...
//main: I'm tired of waiting!
//I'm sleeping 3 ...
//I'm sleeping 4 ...
//main: Now I can quit.
```


> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-02.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-02.kt)獲取完整的代碼

Run it to see that it continues to print "I'm sleeping" even after cancellation until the job completes by itself after five iterations.

運行它去看，去看繼續印出 "I'm sleeping" 即使在之後取消，直到在五次遍歷後透過 job 本身完成。

### Making computation code cancellable

Making computation code cancellable ：使計算中的代碼可取消

There are two approaches to making computation code cancellable. The first one is to periodically invoke a suspending function that checks for cancellation. There is a [yield][yield] function that is a good choice for that purpose. The other one is to explicitly check the cancellation status. Let us try the latter approach. 

有兩個方式使計算中的代碼可取消。第一個方式是定期調用懸掛函數檢查取消。對於此目的有一個 [yield][yield] 函數是很好的選擇。另一個方式是明確檢查可取消的狀態。讓我們嘗試後者的方法。

Replace `while (i < 5)` in the previous example with `while (isActive)` and rerun it. 

使用 `while (isActive)` 取代在前一個範例的 `while (i < 5)` 並回傳它。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // cancellable computation loop
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-03.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-03.kt)獲取完整的代碼

As you can see, now this loop is cancelled. [isActive][isActive] is an extension property that is available inside the code of coroutine via [CoroutineScope][CoroutineScope] object.

如你所見，現在這個循環被取消。 [isActive][isActive] 是擴展屬性，經由 [CoroutineScope][CoroutineScope] 物件在協程的代碼內使用。

### Closing resources with finally

Closing resources with finally ：使用 finally 關掉資源

Cancellable suspending functions throw [CancellationException][CancellationException] on cancellation which can be handled in a usual way. For example, `try {...} finally {...}` expression and Kotlin `use` function execute their finalization actions normally when coroutine is cancelled:

可取消的懸掛函數在取消時丟出 [CancellationException][CancellationException] ，在常用的方法中處理取消。例如， `try {...} finally {...}` 表達式，以及當協程被取消時， Kotlin `use` 函數正常執行它們的結束動作。

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("I'm running finally")
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
//sampleEnd    
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-04.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-04.kt)獲取完整的代碼

Both [join][Job.join] and [cancelAndJoin][cancelAndJoin] wait for all the finalization actions to complete, so the example above produces the following output:

[join][Job.join] 和 [cancelAndJoin][cancelAndJoin] 等待所有結束動作完成，所以上面範例生成以下輸出：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
main: Now I can quit.
```

### Run non-cancellable block

Run non-cancellable block ：運行不能取消的區塊

Any attempt to use a suspending function in the `finally` block of the previous example causes [CancellationException][CancellationException], because the coroutine running this code is cancelled. Usually, this is not a problem, since all well-behaving closing operations (closing a file, cancelling a job, or closing any kind of a communication channel) are usually non-blocking and do not involve any suspending functions. However, in the rare case when you need to suspend in the cancelled coroutine you can wrap the corresponding code in `withContext(NonCancellable) {...}` using [withContext][withContext] function and [NonCancellable][NonCancellable] context as the following example shows:

在上一個範例的 `finally` 區塊中，任何嘗試使用懸掛函數都會造成 [CancellationException][CancellationException] ，因為取消正在運行這些代碼的協程。通常，這不是問題，因為所有表示良好的關閉操作 (關閉檔案、取消任務、關閉任何種類的溝通頻道) 通常是非阻塞的並不會涉及任何的懸掛函數。然而，在罕見的情況下 ，當你需要在已取消的協程中懸掛 (暫停) 時，你可以使用 [withContext][withContext] 函數和 [NonCancellable][NonCancellable] 內容如下例所示，包裝對應的代碼在 `withContext(NonCancellable) {...}` 之中：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
//sampleEnd    
}
//ans:
//I'm sleeping 0 ...
//I'm sleeping 1 ...
//I'm sleeping 2 ...
//main: I'm tired of waiting!
//I'm running finally
//And I've just delayed for 1 sec because I'm non-cancellable
//main: Now I can quit.
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-05.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-05.kt)獲取完整的代碼

### Timeout

Timeout ：超時

The most obvious reason to cancel coroutine execution in practice is because its execution time has exceeded some timeout. While you can manually track the reference to the corresponding [Job][Job] and launch a separate coroutine to cancel the tracked one after delay, there is a ready to use [withTimeout][withTimeout] function that does it. Look at the following example:

在實踐中取消協程執行最明顯的原因是因為它的執行時間超出某些超時。當你可以手動追蹤對應的 [Job][Job] 引用並發射單獨協程在延遲後取消追蹤，有一個準備好使用 [withTimeout][withTimeout] 函數做這件事。看看以下範例：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-06.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-06.kt)獲取完整的代碼

It produces the following output:

它產生以下輸出：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms
```

The `TimeoutCancellationException` that is thrown by [withTimeout][withTimeout] is a subclass of [CancellationException][CancellationException]. We have not seen its stack trace printed on the console before. That is because inside a cancelled coroutine `CancellationException` is considered to be a normal reason for coroutine completion. However, in this example we have used `withTimeout` right inside the `main` function.

由 [withTimeout][withTimeout] 丟出 `TimeoutCancellationException` 是 [CancellationException][CancellationException] 的子類別。我們之前在控制台上已經沒看到它的堆疊追蹤印出。這是因為在已取消的協程內 `CancellationException`被視為協程完成的正常原因。然而，在這個例子我們在 `main` 函數內已經正確使用 `withTimeout` 。

Because cancellation is just an exception, all the resources are closed in a usual way. You can wrap the code with timeout in `try {...} catch (e: TimeoutCancellationException) {...}` block if you need to do some additional action specifically on any kind of timeout or use [withTimeoutOrNull][withTimeoutOrNull] function that is similar to [withTimeout][withTimeout], but returns `null` on timeout instead of throwing an exception:

因為取消就是一個異常，在常用的方法中關閉所有資源。如果你需要在各種超時中做一些額外特別的動作，或使用 [withTimeoutOrNull][withTimeoutOrNull] 函數類似於 [withTimeout][withTimeout] ，你可以在 `try {...} catch (e: TimeoutCancellationException) {...}` 區塊中指定超時包裝代碼，但在超時中回傳 `null` 代替丟出一個異常：

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val result = withTimeoutOrNull(1300L) { // 超時沒完全執行也回傳 null
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // will get cancelled before it produces this result
    }
    println("Result is $result")
//sampleEnd
}
```

> You can get full code [here](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-07.kt)
>
> 你可以在[這裡](https://github.com/kotlin/kotlinx.coroutines/blob/master/core/kotlinx-coroutines-core/test/guide/example-cancel-07.kt)獲取完整的代碼

There is no longer an exception when running this code:

當正在運行這些代碼時，不再有異常：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->

[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[cancelAndJoin]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/cancel-and-join.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-non-cancellable.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html

<!--- END -->