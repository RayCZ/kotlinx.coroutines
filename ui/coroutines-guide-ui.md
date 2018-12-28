# Guide to UI programming with coroutines

Guide to UI programming with coroutines ：使用協程的 UI 編程

This guide assumes familiarity with basic coroutine concepts that are covered in [Guide to kotlinx.coroutines](../docs/coroutines-guide.md) and gives specific examples on how to use coroutines in UI applications. 

這份指導假設您已熟悉在 [kotlinx.coroutines 指導](../docs/coroutines-guide.md)中涵蓋的基本協程觀念，並在 UI 應用程式中如何使用協程的具體範例。

All UI application libraries have one thing in common. They have the single main thread where all state of the UI is confined, and all updates to the UI has to happen in this particular thread. With respect to coroutines, it means that you need an appropriate _coroutine dispatcher context_ that confines the coroutine execution to this main UI thread. 

所有 UI 應用程式函式庫有一個共通點。它們都有單一主線程， UI 所有的狀態被限制於主線程，並且所有 UI 的更新必須發生在特定線程。關於協程，它意味著你需要一個適當的**協程分配器環境**，限制協程執行到這個主 UI 線程。

In particular, `kotlinx.coroutines` has three modules that provide coroutine context for different UI application libraries:

特別是， `kotlinx.coroutines` 有三個模組為不同 UI 應用程式函式庫提供協程環境：

* [kotlinx-coroutines-android](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-android) -- `Dispatchers.Main` context for Android applications.
* [kotlinx-coroutines-javafx](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-javafx) -- `Dispatchers.JavaFx` context for JavaFX UI applications.
* [kotlinx-coroutines-swing](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-swing) -- `Dispatchers.Swing` context for Swing UI applications.

Also, UI dispatcher is available via `Dispatchers.Main` from `kotlinx-coroutines-core` and corresponding implementation (Android, JavaFx or Swing) is discovered by [`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) API. For example, if you are writing JavaFx application, you can use either `Dispatchers.Main` or `Dispachers.JavaFx` extension, it will be the same object.

此外， UI 分配器從 `kotlinx-coroutines-core` 透過 `Dispatchers.Main` 可用，並透過 [`ServiceLoader`](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html) API 發現對應的實作 (Android 、 JavaFx 、 Swing) 。例如，如果你正在寫 JavaFx 應用程式，你可以使用 `Dispatchers.Main` 或 `Dispachers.JavaFx` 擴展，它將是同一個物件。

This guide covers all UI libraries simultaneously, because each of these modules consists of just one object definition that is a couple of pages long. You can use any of them as an example to write the corresponding context object for your favourite UI library, even if it is not included out of the box here.

這分指導同時涵蓋所有 UI 函式庫，因為這些模組每個只包含一個物件定義，每個定義幾頁的長度。你可以使用它們中的任何一個作為範例，為你喜歡的 UI 函式庫寫對應的環境物件，即使它在這裡不包括開箱即刻可用。

## Table of contents

* [Setup](#setup)
  * [JavaFx](#javafx)
  * [Android](#android)
* [Basic UI coroutines](#basic-ui-coroutines)
  * [Launch UI coroutine](#launch-ui-coroutine)
  * [Cancel UI coroutine](#cancel-ui-coroutine)
* [Using actors within UI context](#using-actors-within-ui-context)
  * [Extensions for coroutines](#extensions-for-coroutines)
  * [At most one concurrent job](#at-most-one-concurrent-job)
  * [Event conflation](#event-conflation)
* [Blocking operations](#blocking-operations)
  * [The problem of UI freezes](#the-problem-of-ui-freezes)
  * [Structured concurrency, lifecycle and coroutine parent-child hierarchy](#structured-concurrency-lifecycle-and-coroutine-parent-child-hierarchy)
  * [Blocking operations](#blocking-operations)
* [Advanced topics](#advanced-topics)
  * [Starting coroutine in UI event handlers without dispatch](#starting-coroutine-in-ui-event-handlers-without-dispatch)

## Setup

Setup ：設置

The runnable examples in this guide are presented for JavaFx. The advantage is that all the examples can be directly started on any OS without the need for emulators or anything like that and they are fully self-contained (each example is in one file). There are separate notes on what changes need to be made (if any) to reproduce them on Android. 

在這指導提供 JavaFX 中可運行的範例。優點是所有範例都可使直接開始使用在任何作業系統，不需要模擬器或任何類似的東西，並且它們完全自主獨立的 (每個範例都一個檔案中) 。在 Android 重現它們需要製作當有什麼改變時單獨的說明筆記。

### JavaFx

The basic example application for JavaFx consists of a window with a text label named `hello` that initially contains "Hello World!" string and a pinkish circle in the bottom-right corner named `fab` (floating action button).

JavaFx 基本範例應用程式包含使用命名 `Hello` 文字標籤的視窗，初始化包含 "Hello World!" 字串，並且在右下角的粉紅字圈圈命名 `fab` (浮動按鈕) 。

![UI example for JavaFx](ui-example-javafx.png)

The `start` function of JavaFX application invokes `setup` function, passing it reference to `hello` and `fab` nodes. That is where various code is placed in the rest of this guide:

JavaFx 應用程式的 `start` 函數調用 `setup` 函數，傳遞它引用 `hello` 和 `fab` 的節點。這裡的各種代碼被放置到這份指導的其他部份 (地方) ：

```kotlin
fun setup(hello: Text, fab: Circle) {
    // placeholder
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-01.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-01.kt)獲取完整的代碼

You can clone [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) project from GitHub onto your workstation and open the project in IDE. All the examples from this guide are in the test folder of [`ui/kotlinx-coroutines-javafx`](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-javafx) module. This way you'll be able to run and see how each example works and to experiment with them by making changes.

你可以從 Github 複製 [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 到你的工作站，並有 IDE 下打開專案。從這份指導的所有範例在 [`ui/kotlinx-coroutines-javafx`](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-javafx) 模組的 test 目錄。這種方式你將能執行並看到每個範例如何運行，以及透過修改程式碼試驗它們。

### Android

Follow the guide on [Getting Started With Android and Kotlin](https://kotlinlang.org/docs/tutorials/kotlin-android.html) to create Kotlin project in Android Studio. You are also encouraged to add [Kotlin Android Extensions](https://kotlinlang.org/docs/tutorials/android-plugin.html) to your application.

在 [Getting Started With Android and Kotlin](https://kotlinlang.org/docs/tutorials/kotlin-android.html) 中遵循這份指導，在 Android Studio 去創建  Kotlin 傳案 。也鼓勵你添加 [Kotlin Android Extensions](https://kotlinlang.org/docs/tutorials/android-plugin.html) 到你的應用程式。

In Android Studio 2.3 you'll get an application that looks similarly to the one shown below:

在 Android Studio 2.3 中，你將可以獲得一個類似於下圖所示的應用程式：

![UI example for Android](ui-example-android.png)

Go to the `context_main.xml` of your application and assign an ID of "hello" to the text view with "Hello World!" string, so that it is available in your application as `hello` with Kotlin Android extensions. The pinkish floating action button is already named `fab` in the project template that gets created.

進到你的應用程式 `context_main.xml` 並分配一個 "hello" ID 到 TextView 使用 "Hello World!" 字串，以便它在你的應用程式作用 Kotlin Android 擴展的 `hello` 元件可用。粉紅色浮動按鈕在已創建的專案模版中已經命名 `fab` 。

In the `MainActivity.kt` of your application remove the block `fab.setOnClickListener { ... }` and instead add `setup(hello, fab)` invocation as the last line of `onCreate` function. Create a placeholder `setup` function at the end of the file. That is where various code is placed in the rest of this guide:

在你的應用程式 `MainActivity.kt` 中刪除 `fab.setOnClickListener { ... }` 區域，並取代添加 `setup(hello, fab)` 調用作為 `onCreate` 函數的最後一行。有檔案的尾端中創建一個佔位符 `setup` 。 這裡的各種代碼被放置到這份指導的其他部份 (地方) ：

```kotlin
fun setup(hello: TextView, fab: FloatingActionButton) {
    // placeholder
}
```

Add dependencies on `kotlinx-coroutines-android` module to the `dependencies { ... }` section of `app/build.gradle` file:

添加 `kotlinx-coroutines-android` 模組依賴到 `app/build.gradle` 檔案的 `dependencies { ... }` 部份。

```groovy
implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.0.1"
```

You can clone [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) project from GitHub onto your workstation. The resulting template project for Android resides in [`ui/kotlinx-coroutines-android/example-app`](kotlinx-coroutines-android/example-app) directory. You can load it in Android Studio to follow this guide on Android.

你可以從 Github 複製 [kotlinx.coroutines](https://github.com/Kotlin/kotlinx.coroutines) 到你的工作站。 Android 生成的模版專案位於 [`ui/kotlinx-coroutines-android/example-app`](kotlinx-coroutines-android/example-app) 目錄。你可以在 Android Studio 中載入它，並在 Android 上遵循這份指導。	

## Basic UI coroutines

Basic UI coroutines ：基本 UI 協程

This section shows basic usage of coroutines in UI applications.

這個章節在 UI 應用程式上展示基本協程的用法。

### Launch UI coroutine

Launch UI coroutine ：發射 UI 協程

The `kotlinx-coroutines-javafx` module contains [Dispatchers.JavaFx][kotlinx.coroutines.Dispatchers.JavaFx] dispatcher that dispatches coroutine execution to
the JavaFx application thread. We import it as `Main` to make all the presented examples easily portable to Android:

`kotlinx-coroutines-javafx` 模組包含 [Dispatchers.JavaFx][kotlinx.coroutines.Dispatchers.JavaFx] 分配器，分配協程執行到 JavaFx 應用程式線程。我們匯入它作為 Main 別名，使所有展示的範例輕鬆得移植到 Android ：

```kotlin
import kotlinx.coroutines.javafx.JavaFx as Main
```

Coroutines confined to the main UI thread can freely update anything in UI and suspend without blocking the main thread. For example, we can perform animations by coding them in imperative style. The following code updates the text with a 10 to 1 countdown twice a second, using [launch][launch] coroutine builder:

協程已限制在主要 UI 線程可以在 UI 中自由的更新任何事情，並懸掛無阻塞主要線程。例名，我們可以透過命令式風格編碼它們執行動畫。以下代碼使用 10 到 1 每秒兩次倒數的計數器更新文字，使用 [launch][launch] 協程建造者：

```kotlin
fun setup(hello: Text, fab: Circle) {
    GlobalScope.launch(Dispatchers.Main) { // launch coroutine in the main thread
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-02.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-02.kt)獲取完整的代碼

So, what happens here? Because we are launching coroutine in the main UI context, we can freely update UI from inside this coroutine and invoke _suspending functions_ like [delay][delay] at the same time. UI is not frozen while `delay` waits, because it does not block the UI thread -- it just suspends the coroutine.

所以，這裡發生什麼事？因為我們在主 UI 環境中發射協程，我們可以從 (在) 這個協程內自由的更新 UI，並在同時調用懸掛函數像是 [delay][delay] 。 當 `delay` 等待時， UI 不會凍結，因為它不會阻塞 UI 線程 -- 它只是懸掛 (暫停) 協程。

> The corresponding code for Android application is the same. You just need to copy the body of `setup` function into the corresponding function of Android project. 
>
> Android 應用程式對應的代碼是相同方式。你只需要複製 `setup` 函數的內文到 Android 專案的對應函數。

### Cancel UI coroutine

Cancel UI coroutine ：取消 UI 協程

We can keep a reference to the [Job][Job] object that `launch` function returns and use it to cancel coroutine when we want to stop it. Let us cancel the coroutine when pinkish circle is clicked:

我們可以持有 [Job][Job] 物件的參照，參照由 `launch` 函數回傳，並在我們想要停止它時，使用它去取消協程。當粉紅圈圈被點擊時，讓我們取消協程。

```kotlin
fun setup(hello: Text, fab: Circle) {
    val job = GlobalScope.launch(Dispatchers.Main) { // launch coroutine in the main thread
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
    fab.onMouseClicked = EventHandler { job.cancel() } // cancel coroutine on click
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-03.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-basic-03.kt)獲取完整的代碼

Now, if the circle is clicked while countdown is still running, the countdown stops. Note, that [Job.cancel][Job.cancel] is completely thread-safe and non-blocking. It just signals the coroutine to cancel its job, without waiting for it to actually terminate. It can be invoked from anywhere. Invoking it on a coroutine that was already cancelled or has completed does nothing. 

現在，如果當倒數還在執行時，點擊圈圈，倒數停止。注意 [Job.cancel][Job.cancel] 是完全線程安全並無阻塞。它只是示意協程去取消它的工作，而不是為它實際終止等待 (類似於通知)。可以從任何地方調用它。在一個已經被取消或已經完成的協程上調用它不會做任何事。

> The corresponding line for Android is shown below: 
>
> Andrroid 的對應行數顯示如下：

```kotlin
fab.setOnClickListener { job.cancel() }  // cancel coroutine on click
```

## Using actors within UI context

Using actors within UI context ：在 UI 環境內使用執行者模式

In this section we show how UI applications can use actors within their UI context make sure that there is no unbounded growth in the number of launched coroutines.

在這個章節中，我們展示 UI 應用程式在它們的 UI 環境內如何使用執行者模式，確保在已發射協程的數量沒有無限的增長。

### Extensions for coroutines

Extensions for coroutines ：協程的擴展

Our goal is to write an extension _coroutine builder_ function named `onClick`, so that we can perform countdown animation every time when the circle is clicked with this simple code:

我們的目的是寫一個命名為 `onClick` 的擴展協程建造者函數，以便於使用這些簡單代碼點擊圈圈時，我們可以每次執行倒數動畫。

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onClick { // start coroutine when the circle is clicked
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
}
```

Our first implementation for `onClick` just launches a new coroutine on each mouse event and passes the corresponding mouse event into the supplied action (just in case we need it):

我們對 `onClick` 的第一個實作，只是在每次滑鼠事件中發射一個新的協程，並且傳遞對應的滑鼠事件到提供的動作 (在這個例子我們需要它) ：

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    onMouseClicked = EventHandler { event ->
        GlobalScope.launch(Dispatchers.Main) { 
            action(event)
        }
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-01.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-01.kt)獲取完整的代碼

Note, that each time the circle is clicked, it starts a new coroutine and they all compete to update the text. Try it. It does not look very good. We'll fix it later.

注意，每次點擊圈圈，它啟動一個新協程並它們都競爭的去更新文字。嘗試它，它看起來不太好。稍後我們將修正它。

> On Android, the corresponding extension can be written for `View` class, so that the code in `setup` function that is shown above can be used without changes. There is no `MouseEvent` used in OnClickListener on Android, so it is omitted.
>
> 在 Android 上，可以為 `View` 類別寫擴展函數，以便沒有改變的使用上述的 `setup` 函數中的代碼。在 Android 上的 OnClickListener 中沒有使用 `MouseEvent` ，所以它可以被省略。

```kotlin
fun View.onClick(action: suspend () -> Unit) {
    setOnClickListener { 
        GlobalScope.launch(Dispatchers.Main) {
            action()
        }
    }
}
```

### At most one concurrent job

At most one concurrent job ：最多一個並發 Job

We can cancel an active job before starting a new one to ensure that at most one coroutine is animating the countdown. However, it is generally not the best idea. The [cancel][Job.cancel] function serves only as a signal to abort a coroutine. Cancellation is cooperative and a coroutine may, at the moment, be doing something non-cancellable or otherwise ignore a cancellation signal. A better solution is to use an [actor][actor] for tasks that should not be performed concurrently. Let us change `onClick` extension implementation:

我們可以在啟動一個新的協程前，取消活動中的 Job，確保最多一個協程顯示動畫倒數。然而，這通常不是最好的主意。 [cancel][Job.cancel] 函數只當作退出協程的訊號。取消是協作的，並且此刻的協程或許正在做些不可取消的事情或相反的忽略一個取消訊號。更好的解決方式是對任務使用 [actor][actor] 模式，不應該並發的執行。讓我們改變  `onClick` 擴展函數的實作。

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // launch one actor to handle all events on this node
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main) {
        for (event in channel) action(event) // pass event to action
    }
    // install a listener to offer events to this actor
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-02.kt)
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-02.kt)獲取完整的代碼

The key idea that underlies an integration of an actor coroutine and a regular event handler is that there is an [offer][SendChannel.offer] function on [SendChannel][SendChannel] that does not wait. It sends an element to the actor immediately, if it is possible, or discards an element otherwise. An `offer` actually returns a `Boolean` result which we ignore here.

關鍵想法，作為 actor 協程和常規事件處理器的整合基礎，在 [SendChannel][SendChannel] 中有一個 [offer][SendChannel.offer] 函數不會等待。如果可能，它立即發送一個元素到 actor ，否則丟棄元素。 `offer` 實際上回傳一個 `Boolean` 結果，我們在這裡忽略結果。

Try clicking repeatedly on a circle in this version of the code. The clicks are just ignored while the countdown animation is running. This happens because the actor is busy with an animation and does not receive from its channel. By default, an actor's mailbox is backed by `RendezvousChannel`, whose `offer` operation succeeds only when the `receive` is active. 

嘗試在這個版本的代碼中反覆點擊。當倒數動畫正在運行時，就忽略點擊。發生這種情況，因為 actor 忙於動畫以及沒有從它的通道接收。預設下，透過 `RendezvousChannel` 支援執行者的信箱，只當 `receive` 是活動的時，它的 `offer` 操作才會成功。

> On Android, there is `View` sent in OnClickListener, so we send the `View` to the actor as a signal. The corresponding extension for `View` class looks like this:
>
> 在 Android 上，在 OnClickListener 中有 `View` 參數發送，所以我們發送 `View` 到執行者當作訊號。為 `View` 類別對應的擴展函數看起來像這樣：

```kotlin
fun View.onClick(action: suspend (View) -> Unit) {
    // launch one actor
    val eventActor = GlobalScope.actor<View>(Dispatchers.Main) {
        // 通道拿出來的每個事件 (訊號) 剛好就是一個 View
        for (event in channel) action(event)
    }
    
    // it 代表的是 View 是 setOnClickListener 的參數
    // install a listener to activate this actor
    setOnClickListener { 
        // 負責發送 View 當訊號到通道
        eventActor.offer(it)
    }
}
```

### Event conflation

Event conflation ：事件合併

Sometimes it is more appropriate to process the most recent event, instead of just ignoring events while we were busy processing the previous one.  The [actor][actor] coroutine builder accepts an optional `capacity` parameter that  controls the implementation of the channel that this actor is using for its mailbox. The description of all the available choices is given in documentation of the [`Channel()`][Channel] factory function.

有時處理最近的事件更適合的方式，而不是在我們忙於處理前一個事件，就忽略目前事件。 [actor][actor] 協程建造者接受一個可選的 `capacity` 參數，參數控制這個 actor 用於它的信箱通道實作演算法。 

Let us change the code to use `ConflatedChannel` by passing [Channel.CONFLATED][Channel.CONFLATED] capacity value. The change is only to the line that creates an actor:

讓我們透過傳遞 [Channel.CONFLATED][Channel.CONFLATED] 容量值改變代碼去使用 `ConflatedChannel` 。這個變動只在創建 actor 那行。

```kotlin
fun Node.onClick(action: suspend (MouseEvent) -> Unit) {
    // launch one actor to handle all events on this node
    val eventActor = GlobalScope.actor<MouseEvent>(Dispatchers.Main, capacity = Channel.CONFLATED) { // <--- Changed here
        for (event in channel) action(event) // pass event to action
    }
    // install a listener to offer events to this actor
    onMouseClicked = EventHandler { event ->
        eventActor.offer(event)
    }
}
```

> You can get full JavaFx code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-03.kt). On Android you need to update `val eventActor = ...` line from the previous example. 
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-actor-03.kt)獲取完整 JavaFx 的代碼。在 Android 上你需要從上一個範例更新 `val eventActor = ...` 行。

Now, if a circle is clicked while the animation is running, it restarts animation after the end of it. Just once. Repeated clicks while the animation is running are _conflated_ and only the most recent event gets to be processed. 

現在，如果在動畫在運行時，點擊圈圈，在它結束後，它重新啟動動畫。就一次。當動畫在運行時重覆的點擊被合併，並且只有最近的事件被處理。

This is also a desired behaviour for UI applications that have to react to incoming high-frequency event streams by updating their UI based on the most recently received update. A coroutine that is using `ConflatedChannel` avoids delays that are usually introduced by buffering of events.

這也是 UI 應用程式期望的行為，透過根據最近收到的更新來更新它們的 UI，必須傳入高頻率事件串流做反應。正在使用 `ConflatedChannel` 的一個協程，來避免時常由多事件緩衝引入的延遲。

You can experiment with `capacity` parameter in the above line to see how it affects the behaviour of the code. Setting `capacity = Channel.UNLIMITED` creates a coroutine with `LinkedListChannel` mailbox that buffers all events. In this case, the animation runs as many times as the circle is clicked.

你可以在上面的行中試驗使用 `capacity` 參數，看它如何影響代碼的行為。設置 `capacity = Channel.UNLIMITED` 使用 `LinkedListChannel` 信箱來創建一個協程緩衝所有事件。有這種情況下，動畫運行和點擊圈圈一樣多次。

## Blocking operations

Blocking operations ：阻塞操作

This section explains how to use UI coroutines with thread-blocking operations.

這個章節解釋如何使用線程阻塞操作的 UI 協程

### The problem of UI freezes 

It would have been great if all APIs out there were written as suspending functions that never blocks an execution thread. However, it is quite often not the case. Sometimes you need to do a CPU-consuming computation or just need to invoke some 3rd party APIs for network access, for example, that blocks the invoker thread. You cannot do that from the main UI thread nor from the UI-confined coroutine directly, because that would block the main UI thread and cause the freeze up of the UI.

如果那裡所有的 API 被寫作永遠不會阻塞執行線程的懸掛函數，那就太棒了。然而，通常不是如此。有時我們需要執行 CPU 消耗的計算，或只需調用一些網路存取的第三方 API ，例如，阻塞調用者線程。你不可以直接的從主要 UI 線程，也不可從 UI 限制的協程執行，因為這樣可能阻塞主要 UI 線程，並造成 UI 凍結。

The following example illustrates the problem. We are going to use `onClick` extension with UI-confined event-conflating actor from the last section to process the last click in the main UI thread. For this example, we are going to perform naive computation of [Fibonacci numbers](https://en.wikipedia.org/wiki/Fibonacci_number):

以下範例解釋這個問題。我們將使用 `onClick` 擴展函數，從上一個章節 UI 限制的事件合併 actor 來處理在主要 UI 線程中最後一次點擊。對於這個例子，我們將執行 [Fibonacci numbers](https://en.wikipedia.org/wiki/Fibonacci_number) 的天真計算：

```kotlin
fun fib(x: Int): Int =
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
```

We'll be computing larger and larger Fibonacci number each time the circle is clicked. To make the UI freeze more obvious, there is also a fast counting animation that is always running and is constantly updating the text in the main UI dispatcher:

我們將在每次點擊圈圈時計算越來越大的 Fibonacci number 。為了使 UI 凍結更明顯，也有一個快速計算總在執行中的動畫，並不斷的在主 UI 分配器協程中更新文字。

```kotlin
fun setup(hello: Text, fab: Circle) {
    var result = "none" // the last result
    // counting animation 
    GlobalScope.launch(Dispatchers.Main) {
        var counter = 0
        while (true) {
            hello.text = "${++counter}: $result"
            delay(100) // update the text every 100ms
        }
    }
    // compute the next fibonacci number of each click
    var x = 1
    fab.onClick {
        result = "fib($x) = ${fib(x)}"
        x++
    }
}
```

> You can get full JavaFx code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-01.kt). You can just copy the `fib` function and the body of the `setup` function to your Android project.
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-01.kt)獲取完整 JavaFx 的代碼。你可以只是複製 `fib` 函數和 `setup` 的內文到你的 Android 專案。

Try clicking on the circle in this example. After around 30-40th click our naive computation is going to become quite slow and you would immediately see how the main UI thread freezes, because the animation stops running during UI freeze.

嘗試點擊在這個範例中的圈圈。在大約 30-40 秒之後點擊我們天真的計算將變得相當慢，並且你可能立即看到主要 UI 線程如何凍結，因為動畫在 UI 凍結期間停止運行。

### Structured concurrency, lifecycle and coroutine parent-child hierarchy

Structured concurrency, lifecycle and coroutine parent-child hierarchy ：結構化並發、生命週期和協程父子層級

A typical UI application has a number of elements with a lifecycle. Windows, UI controls, activities, views, fragments and other visual elements are created and destroyed. A long-running coroutine, performing some IO or a background computation, can retain references to the corresponding UI elements for longer than it is needed, preventing garbage collection of the whole trees of UI objects that were already destroyed and will not be displayed anymore.

典型的 UI 應用程式有許多生命週期的元素。 Windows 、 UI 控制、 Activity 、 View 、 Fragment 和其他的視覺元素被創建和銷毀。在執行某些 IO 或背景運算的長時間運行協程可以保留對應的 UI 元素引用超過它所需要的時間，防止整個 UI 物件樹狀的垃圾集合，這些 UI 物件已經被被銷毀，並且將不再被顯示。

The natural solution to this problem is to associate a [Job][Job] object with each UI object that has a lifecycle and create all the coroutines in the context of this job. But passing associated job object to every coroutine builder is error-prone, it is easy to forget it. For this purpose, [CoroutineScope][CoroutineScope] interface should be implemented by UI owner, and then every coroutine builder defined as an extension on [CoroutineScope][CoroutineScope] inherits UI job without explicitly mentioning it.

這個問題的自然解決方式是使用 [Job][Job] 物件聯繫每個有生命週期的 UI 物件，並在這個 job 物件的環境中創建所有協程。但是傳遞聯繫的 job 物件給每個協程建造者容易出錯，很容易忘記它。對於這個目的，應該透過 UI 擁有者實作[CoroutineScope][CoroutineScope] 介面，並接著在 [CoroutineScope][CoroutineScope] 中每個協程建造者定義為擴展函數，繼承 UI job 無明確的提及它。

For example, in Android application an `Activity` is initially _created_ and is _destroyed_ when it is no longer needed and when its memory must be released. A natural solution is to attach an instance of a `Job` to an instance of an `Activity`:

例如，在 Android 應用程式中最初創建一個 `Activity`，並當不再需要它並且必須釋放它的記憶體時銷毀它。一個自然的解決方式是附加一個 `Job` 的實例到 `Activity` 的實例：

```kotlin
abstract class ScopedAppActivity: AppCompatActivity(), CoroutineScope {
    protected lateinit var job: Job
    override val coroutineContext: CoroutineContext 
        get() = job + Dispatchers.Main
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        job = Job()
    }
        
    override fun onDestroy() {
        super.onDestroy()
        job.cancel()
    } 
}
```

Now, an activity that is associated with a job has to extend ScopedAppActivity

現在， Activity 與 job 聯繫必須繼承 ScopedAppActivity

```kotlin
class MainActivity : ScopedAppActivity() {

    fun asyncShowData() = launch { // Is invoked in UI context with Activity's job as a parent
        // actual implementation
    }
    
    suspend fun showIOData() {
        val deferred = async(Dispatchers.IO) {
            // impl      
        }
        withContext(Dispatchers.Main) {
          val data = deferred.await()
          // Show data in UI
        }
    }
}
```

Every coroutine launched from within a `MainActivity` has its job as a parent and is immediately cancelled when activity is destroyed.

從 `Mainactivity` 內發射每個協程，因父類別有它的 job 實例並在 Activity 被銷毀時立刻取消。

To propagate activity scope to its views and presenters, multiple techniques can be used:

傳播 Activity 的協程範圍到它的 View 、 Presenter ，可以使用多種技術：

- [coroutineScope][coroutineScope] builder to provide a nested scope
  提供內嵌協程範圍的 [coroutineScope][coroutineScope] 建造者
- Receive [CoroutineScope][CoroutineScope] in presenter method parameters
  在 Presenter 方法參數中接收 [CoroutineScope][CoroutineScope]
- Make method extension on [CoroutineScope] (applicable only for top-level methods)
  在 [CoroutineScope][CoroutineScope] 中製作方法擴展 (只適用於最高層的方法)

```kotlin
class ActivityWithPresenters: ScopedAppActivity() {
    fun init() {
        val presenter = Presenter()
        val presenter2 = ScopedPresenter(this)
    }
}

class Presenter {
    suspend fun loadData() = coroutineScope {
        // Nested scope of outer activity
    }
    
    suspend fun loadData(uiScope: CoroutineScope) = uiScope.launch {
      // Invoked in the uiScope
    }
}

class ScopedPresenter(scope: CoroutineScope): CoroutineScope by scope {
    fun loadData() = launch { // Extension on ActivityWithPresenters's scope
    }
}

suspend fun CoroutineScope.launchInIO() = launch(Dispatchers.IO) {
   // Launched in the scope of the caller, but with IO dispatcher
}
```

Parent-child relation between jobs forms a hierarchy. A coroutine that performs some background job on behalf of the view and in its context can create further children coroutines. The whole tree of coroutines gets cancelled when the parent job is cancelled. An example of that is shown in the ["Children of a coroutine"](../docs/coroutine-context-and-dispatchers.md#children-of-a-coroutine) section of the guide to coroutines.

在 job 之間的父子關係形成層級。協程代表 View 執行某些背景工作，並且在它的環境中可以創建進一步的子協程。在父協程的 job 被取消時，整個協程的樹狀聯繫 job 都被取消。

### Blocking operations

Blocking operations ：阻塞操作

The fix for the blocking operations on the main UI thread is quite straightforward with coroutines. We'll convert our "blocking" `fib` function to a non-blocking suspending function that runs the computation in the background thread by using [withContext][withContext] function to change its execution context to [Dispatchers.Default][Dispatchers.Default] that is backed by the background pool of threads. Notice, that `fib` function is now marked with `suspend` modifier. It does not block the coroutine that it is invoked from anymore, but suspends its execution when the computation in the background thread is working:

修正在主要 UI 線程的阻塞操作，使用協程相當直接了當。我們將我們的 "阻塞操作" `fib` 函數轉換到非阻塞的懸掛函數，此函數在背景線程中運行計算工作，透過使用 [withContext][withContext] 函數去改變它執行的環境到 [Dispatchers.Default][Dispatchers.Default] ，由背景線程池支援的 [Dispatchers.Default][Dispatchers.Default] 。注意，該 `fib` 函數現在使用 `suspend` 修飾符標記。它不會阻塞再次調協的協程，但是在背景線程中計算正在執行時，懸掛 (暫停) 它的執行

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    if (x <= 1) x else fib(x - 1) + fib(x - 2)
}
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-02.kt).
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-02.kt)獲取完整的代碼

You can run this code and verify that UI is not frozen while large Fibonacci numbers are being computed. However, this code computes `fib` somewhat slower, because every recursive call to `fib` goes via `withContext`. This is not a big problem in practice, because `withContext` is smart enough to check that the coroutine is already running in the required context and avoids overhead of dispatching coroutine to a different thread again. It is an overhead nonetheless, which is visible on this primitive code that does nothing else, but only adds integers in between invocations to `withContext`. For some more substantial code, the overhead of an extra `withContext` invocation is not going to be significant.

你可以運行此代碼，並驗證在計算大量 Fibonacci numbers 時不會凍結 UI ，然而，這些代碼計算 `fib` 有些慢，因為每個遞迴透過 `withContext` 調用 `fib`。在實踐中這不是大問題，因為 `withContext` 足夠聰明，去檢查協程已經在所需的環境中運行，並避免再次分配協程到不同的線程開銷。儘管如此，它是一個開銷，在這些原生代碼中顯示的，它不做其它的事情，而只在調 `withContext` 之間添加整數。對於一些更實質的代碼，額外的 `withContext` 調用開銷將不重要。

Still, this particular `fib` implementation can be made to run as fast as before, but in the background thread, by renaming the original `fib` function to `fibBlocking` and defining `fib` with `withContext` wrapper on top of `fibBlocking`:

仍然，這個特定的 `fib` 實作可以像以前一樣快的運行，但是在背景線程中，透過原始的 `fib` 函數重新命名為 `fibBlocking` 並在 `fibBlocking` 的上面使用 `withContext` 包裝來定義 `fib` 函數：

```kotlin
suspend fun fib(x: Int): Int = withContext(Dispatchers.Default) {
    fibBlocking(x)
}

fun fibBlocking(x: Int): Int = 
    if (x <= 1) x else fibBlocking(x - 1) + fibBlocking(x - 2)
```

> You can get full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-03.kt).
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-blocking-03.kt)獲取完整的代碼

You can now enjoy full-speed naive Fibonacci computation without blocking the main UI thread. All we need is `withContext(Dispatchers.Default)`.

你現在可以在無阻塞主要 UI 線程全速天真的 Fibonacci 運算。我們只需要 `withContext(Dispatchers.Default)` 。

Note, that because the `fib` function is invoked from the single actor in our code, there is at most one concurrent computation of it at any given time, so this code has a natural limit on the resource utilization. It can saturate at most one CPU core.

注意，那是因為在我們代碼中從單一執行者調用 `fib` 函數，在任何已給的時間點最多只有一個它的並發計算，所以這些代碼在資源利用上有自然的限制。它最多可以飽合一個 CPU 核心。

## Advanced topics

Advanced topics ：進階主題

This section covers various advanced topics. 

這個章節涵蓋各種進階主題。

### Starting coroutine in UI event handlers without dispatch

Starting coroutine in UI event handlers without dispatch ：

Let us write the following code in `setup` to visualize the order of execution when coroutine is launched from the UI thread:

我我們在 `setup` 中寫以下的代碼，當從 UI 線程發射協程時，視覺化執行的順序：

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main) {
            println("Inside coroutine")
            delay(100)
            println("After delay")
        } 
        println("After launch")
    }
}
```

> You can get full JavaFx code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-advanced-01.kt).
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-advanced-01.kt)獲取完整的代碼

When we start this code and click on a pinkish circle, the following messages are printed to the console:

當我們啟動這個代碼並在粉紅圈圈點擊時，印出以下訊息到控制台：

```text
Before launch
After launch
Inside coroutine
After delay
```

As you can see, execution immediately continues after [launch][launch], while the coroutine gets posted onto the main UI thread for execution later. All UI dispatchers in `kotlinx.coroutines` are implemented this way. Why so? 

如你所見，在 "After [launch][launch]" 之後協程立即執行，而協程發怖到主要 UI 線程而稍後執行。在 `kotlinx.coroutines` 所有 UI 分配器以這種方式實作。為何這樣？

Basically, the choice here is between "JS-style" asynchronous approach (async actions are always postponed to be executed later in the even dispatch thread) and "C#-style" approach (async actions are executed in the invoker thread until the first suspension point). While, C# approach seems to be more efficient, it ends up with recommendations like "use `yield` if you need to ....". This is error-prone. JS-style approach is more consistent and does not require programmers to think about whether they need to yield or not.

基本上，這裡的選擇是在 "JS 風格" 異步方式之間 (異步動作總是被推遲的，在實際分配的線程中稍後執行) ，而 "C# 風格" 方式 (異步動作在調用者線程中執行，直到第一個懸掛點) 。雖然， C# 方式看起來更有效率，它最終的建議像是 "如果你需要.....，使用 `yield`" 。這是易於出錯的。 JS 風格方式是更一致並不會需要程式設計師去思考它們是否需要 `yield` 或不用。

However, in this particular case when coroutine is started from an event handler and there is no other code around it, this extra dispatch does indeed add an extra overhead without bringing any additional value. In this case an optional [CoroutineStart][CoroutineStart] parameter to [launch][launch], [async][async] and [actor][actor] coroutine builders can be used for performance optimization. Setting it to the value of [CoroutineStart.UNDISPATCHED][CoroutineStart.UNDISPATCHED] has the effect of starting to execute coroutine immediately until its first suspension point as the following example shows:

然而，在這種特殊情況下，當從事件處理者 `EventHandler` 中啟動協程並沒有其他的代碼在它週圍時，這個額外分配確實增加額外開銷沒有帶來任何額外的價值。在這種情況可選的 [CoroutineStart][CoroutineStart] 參數給 [launch][launch] 、 [async][async] 、 [actor][actor] 協程建造者可以用作效能優化。設置 [CoroutineStart.UNDISPATCHED][CoroutineStart.UNDISPATCHED] 的值有立即啟動執行協程的效果，直到它第一個懸掛點，如以下範例所示：

```kotlin
fun setup(hello: Text, fab: Circle) {
    fab.onMouseClicked = EventHandler {
        println("Before launch")
        GlobalScope.launch(Dispatchers.Main, CoroutineStart.UNDISPATCHED) { // <--- Notice this change
            println("Inside coroutine")
            delay(100)                            // <--- And this is where coroutine suspends      
            println("After delay")
        }
        println("After launch")
    }
}
```

> You can get full JavaFx code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-advanced-02.kt).
>
> 你可以在[這裡](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/kotlinx-coroutines-javafx/test/guide/example-ui-advanced-02.kt)獲取完整的代碼

It prints the following messages on click, confirming that code in the coroutine starts to execute immediately:

在點擊時它印出以下訊息，確認在協程中的代碼立即的啟動執行。

```text
Before launch
Inside coroutine
After launch
After delay
```

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[delay]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/delay.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[coroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[Dispatchers.Default]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html
[CoroutineStart]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/index.html
[async]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html
[CoroutineStart.UNDISPATCHED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-start/-u-n-d-i-s-p-a-t-c-h-e-d.html
<!--- INDEX kotlinx.coroutines.channels -->
[actor]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/actor.html
[SendChannel.offer]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/offer.html
[SendChannel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/index.html
[Channel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/index.html
[Channel.CONFLATED]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-channel/-c-o-n-f-l-a-t-e-d.html
<!--- MODULE kotlinx-coroutines-javafx -->
<!--- INDEX kotlinx.coroutines.javafx -->
[kotlinx.coroutines.Dispatchers.JavaFx]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-javafx/kotlinx.coroutines.javafx/kotlinx.coroutines.-dispatchers/-java-fx.html
<!--- MODULE kotlinx-coroutines-android -->
<!--- INDEX kotlinx.coroutines.android -->
<!--- END -->