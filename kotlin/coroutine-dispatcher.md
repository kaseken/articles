# 内部実装から理解するKotlin Coroutines：CoroutineDispatcher編

## 本記事の目的・対象者

本記事は、Kotlin Coroutines の基礎をすでに理解している中級者以上の開発者を対象に、さらに理解を深めることを目的としたシリーズの一部です。
今回は、**CoroutineDispatcher**に焦点を当て、その内部的な仕組みについて解説していきます。

Kotlin Coroutinesの初歩は、すでに理解していることを前提としています。
公式ドキュメントの説明^[Coroutines guide: https://kotlinlang.org/docs/coroutines-guide.html]や、私が以前執筆した別の記事^["Kotlin Coroutinesの核心：Builder・CoroutineScope・Job・CoroutineContextの関係" https://zenn.dev/kaseken/articles/99d92a128cbc9a]を先に参照されることをおすすめします。

https://zenn.dev/kaseken/articles/99d92a128cbc9a

## CoroutineDispatcherはどこで利用されるのか

CoroutineDispatcherとは、**コルーチンがどのスレッドまたはスレッドプールで実行されるかを指定する、CoroutineContextの一要素**です。

コンピュータサイエンスの分野において「dispatch」とは、「命令・イベント・関数呼び出しなどをハンドラに割り当てる」という意味で使われる用語です。CoroutineDispatcherも同様に、**処理をスレッドあるいはスレッドプールへ割り当てる**という役割を担っています。

CoroutineDispatcherは、以下の2つのタイミングで利用されます。

1. **Coroutineの起動時**
2. **Coroutineの再開時**

以下に示すコード例を用いて、具体的に見ていきましょう ([Kotlin Playground](https://pl.kotl.in/-IyrS7D5v))。

まず、「**1. Coroutineの起動時**」は、Coroutine BuilderがCoroutineを作成・起動したタイミングです。以下のサンプルコードでは、`launch`の部分に当たります。

```kt
import kotlinx.coroutines.*

suspend fun someSuspendFunction(functionId: Int) {
    val firstThread = Thread.currentThread().id
    delay(100) // 2. Coroutineの再開時 (= delayの完了時)
    val secondThread = Thread.currentThread().id
    println("Thread switched from $firstThread to $secondThread.")
}

suspend fun main() {
    coroutineScope {
        for (i in 0..<5) {
            launch(Dispatchers.IO) { // 1. Coroutineの起動時
                someSuspendFunction(i)
            }
        }
    }
}
```

出力:
```sh
Thread switched from 15 to 13.
Thread switched from 13 to 17.
Thread switched from 14 to 15.
Thread switched from 12 to 15.
Thread switched from 17 to 17.
```

suspend関数内では、**処理が一時停止・再開するポイント**があります。このポイントのことを**suspension point**と呼びます。上記のサンプルコードにおいては、`delay(100)`がsuspension pointに相当します。
`delay(100)`の処理が完了し、後続の処理が再開するタイミングが「**2. Coroutineの再開時**」に相当します。

suspension pointに達すると、suspend関数内の処理が一時停止するとともに、実行スレッドが解放されます。そして、再開時には、停止以前と同じスレッドで再開するとは限りません。
上記のサンプルコードにおいては、CoroutineDispatcherとして`Dispatchers.IO`を使用しています。出力からも分かる通り、スレッドプール内のスレッドで処理が実行されるため、**suspension pointの前後で実行スレッドが切り替わる**ことがあります。

## CoroutineDispatcherの内部実装 ー `dispatch`が呼ばれるまで

前項で見たように、CoroutineDispatcherは、**「Coroutineの起動時」と「Coroutineの再開時」に処理を実行するスレッドを決定する役割**を担っています。
ここからは、どのようにして所定のスレッドへと処理が割り当てられるのか、具体的にソースコードを追っていきましょう。

まずは、Coroutine BuilderがCoroutineを起動する際のCoroutineDispatcherの関与を追ってみましょう。
代表的なCoroutine Builderである`launch`のソースコードを以下に示します^[`launch`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L44]。

```kt
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

Coroutineを起動するための`coroutines.start`が呼ばれていることが分かります。なお、同じく代表的なCoroutine Builderである`async`メソッドでも、同様に`coroutines.start`が呼ばれています^[`async`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L79]。

`start`メソッドは、`AbstractCoroutine`に対して定義されています^[AbstractCoroutine.start: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/AbstractCoroutine.kt#L133]。
コメントからも、Coroutineを起動するためのメソッドであることが分かります。

```kt
    /**
     * Starts this coroutine with the given code [block] and [start] strategy.
     * This function shall be invoked at most once on this coroutine.
     * 
     * - [DEFAULT] uses [startCoroutineCancellable].
     * - [ATOMIC] uses [startCoroutine].
     * - [UNDISPATCHED] uses [startCoroutineUndispatched].
     * - [LAZY] does nothing.
     */
    public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }
```

`AbstractCoroutine.start`の内部の`start(block, receiver, this)`では、`CoroutineStart.invoke`^[CoroutineStart.invoke: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineStart.kt#L356]が呼ばれます。

```kt
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit // will start lazily
        }
```

基本的には、CoroutineStartは`DEFAULT`となり、`block.startCoroutineCancellable(receiver, completion)`が呼ばれます。

```kt
public fun <T> (suspend () -> T).startCoroutineCancellable(completion: Continuation<T>): Unit = runSafely(completion) {
    createCoroutineUnintercepted(completion).intercepted().resumeCancellableWith(Result.success(Unit))
}
```

`startCoroutineCancellable`の内部では、`createCoroutineUnintercepted`・`intercepted`・`resume`という3つのメソッドが続けて呼ばれています。それぞれの役割を見ていきましょう。

#### 1. createCoroutineUninterceptedの役割

`createCoroutineUnintercepted`は、**Coroutineの動作の要である「Continuation」を初期化**します。Continuationは、Coroutineの状態を管理するためのインスタンスです。**suspend関数内で処理を一時停止できる仕組みは、Continuationによって実現**されています。
Continuationについては、それ自体が大きなトピックのため、別の記事で解説予定です。

CoroutineDispatcherは、作成されたContinuationの`context: CoroutineContext`フィールド内に保持されます^[Continuation.context: https://github.com/JetBrains/kotlin/blob/6ef387e3e9ee07d24821c276ddf11a984a38c5d8/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L20]。

```kt
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

#### 2. interceptedの役割

`intercepted`メソッドは、**前のステップで初期化されたContinuationを、CoroutineDispatcherに対して送信可能とします**。より厳密に言えば、CoroutineContextに登録されたContinuationInterceptorを用いて、Continuationに変換を加えます。
「intercept」とは傍受するという意味の単語ですが、Coroutineの実行を傍受して、変換を加えるというイメージです。

JVMプラットフォームにおけるinterceptedのソースコードを以下に示します ^[intercepted: https://github.com/JetBrains/kotlin/blob/15b95af041cb7068f1a721278f7a5c0057fbfa4e/libraries/stdlib/jvm/src/kotlin/coroutines/jvm/internal/ContinuationImpl.kt#L111]。

```kt
    public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
```

このように、`ContinuationInterceptor`が`CoroutineContext`に存在する場合、`interceptContinuation`が呼び出されてContinuationが変換されます。
そして、**`CoroutineDispatcher`は`ContinuationInterceptor`を継承している**ため、DispatcherがContextに含まれていれば、そのDispatcherの`interceptContinuation`が呼ばれます。

`interceptContinuation`のソースコードを以下に示します ^[interceptContinuation: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineDispatcher.kt#L240]。

```kt
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)
```

このメソッドは、渡された`Continuation`を`DispatchedContinuation`にラップして返します。

`DispatchedContinuation`は、**元の`Continuation`をCoroutineDispatcherによってスケジューリング可能な形に変換するラッパー**です。以下にソースコードの一部を抜粋します^[DispatchedContinuation: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/DispatchedContinuation.kt#L12]。

```kt
internal class DispatchedContinuation<in T>(
    @JvmField internal val dispatcher: CoroutineDispatcher,
    @JvmField val continuation: Continuation<T>
) : DispatchedTask<T>(MODE_UNINITIALIZED), CoroutineStackFrame, Continuation<T> by continuation {
}
```

このように、`DispatchedContinuation`は`dispatcher`を内部に保持し、Coroutineの起動・再開処理をDispatcherに委ねることができます。

#### 3. resumeの役割

前のステップで返された`DispatchedContinuation`に対して、`resume`が呼び出されます。
`resume`は、Coroutineを再開させるためのメソッドです。ここから分かるように、`resume`というメソッド名であるものの、実は初回起動時にも`resume`メソッドが呼ばれます。

`resume`のソースコードを以下に示します ^[Coroutine.resume: https://github.com/JetBrains/kotlin/blob/6ef387e3e9ee07d24821c276ddf11a984a38c5d8/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L44]。

```kt
/**
 * Resumes the execution of the corresponding coroutine passing [value] as the return value of the last suspension point.
 */
@SinceKotlin("1.3")
@InlineOnly
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))
```

`resume`は、`DispatchedContinuation.resumeWith`を呼び出します。`resumeWith`のソースコードを以下に示します^[`DispatchedContinuation.resumeWith`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/DispatchedContinuation.kt#L188]。

```kt
    override fun resumeWith(result: Result<T>) {
        val state = result.toState()
        if (dispatcher.safeIsDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_ATOMIC
            dispatcher.safeDispatch(context, this)
        } else {
            executeUnconfined(state, MODE_ATOMIC) {
                withCoroutineContext(context, countOrElement) {
                    continuation.resumeWith(result)
                }
            }
        }
    }
```

`resumeWith`は、まず`safeIsDispatchNeeded`メソッドで、`CoroutineDispatcher`による`dispatch` (スレッドあるいはスレッドプールへの割り当て) が必要かをチェックします。
そして、`dispatch`が必要であれば、`dispatcher.safeDispatch`を呼び出し、CoroutineDispatcherに処理を送信します。
`safeDispatch`メソッド内では、以下のように`dispatch`が呼ばれます^[CoroutineDispatcher.safeDispatch: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/DispatchedContinuation.kt#L252]。

```kt
internal fun CoroutineDispatcher.safeDispatch(context: CoroutineContext, runnable: Runnable) {
    try {
        dispatch(context, runnable)
    } catch (e: Throwable) {
        throw DispatchException(e, this, context)
    }
}
```

基本的には`safeIsDispatchNeeded`はtrueとなる、すなわちCoroutineDispatcherの`dispatch`が呼ばれるケースが大半です。
例外として、「`Dispatchers.Unconfined`を使用している」あるいは「`Dispatchers.Main.immediate`を使用している、かつ既にメインスレッドで実行されている」の場合のように、`dispatch`がスキップされるケースもあります。

ここまでで、Coroutine Builderの`launch`メソッドを出発点として、CoroutineDispatcherのdispatchが呼ばれるまでの流れを追うことができました。

他方、**Coroutineの再開時**には、Continuationに対して`resume`が呼び出されます。そのため、同様に`resumeWith`内で`CoroutineDispatcher.dispatch`が呼ばれることになります。

## CoroutineDispatcherの内部実装 ー `dispatch`が呼ばれた後

続いて、`CoroutineDispatcher.dispatch`が呼ばれた後の動作を追っていきます。
`dispatch`は、CoroutineDispatcherに対して抽象メソッドとして定義されています ^[`CoroutineDispatcher.dispatch`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineDispatcher.kt#L216]。

```kt
public abstract class CoroutineDispatcher {
    public abstract fun dispatch(context: CoroutineContext, block: Runnable)
}
```

具体的な実装はCoroutineDispatcherの各サブクラスで行われています。すなわち、**dispatchの具体的な内部実装は、「CoroutineDispatcherの種類」ならびに「動作プラットフォーム」によって異なります。**

そこで、以降のセクションでは、実行環境として最も典型的である「JVM/Androidプラットフォーム」を想定した上で、各CoroutineDispatcherごとの内部実装を追っていきます。

### CoroutineDispatcherの種類

JVM/Androidプラットフォームでは、以下の4種類のCoroutineDispatcherがあらかじめ定義されています。

- **Dispatchers.Default**
- **Dispatchers.IO**
- **Dispatchers.Main**
- **Dispatchers.Unconfined**

### Dispatchers.Default

`Dispatchers.Default`では、CPUのコア数と同数のスレッドを持つスレッドプールが、処理の実行に用いられます^[`Dispatchers.Default`: https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-dispatchers/-default.html]。スレッド数に上限が存在するため、CPU-boundな処理に向いています。
`Dispatchers.Default`は、その名の通り、CoroutineDispatcherがCoroutineContextで指定されていない場合にデフォルトで使用されるCoroutineDispatcherです。

では、Android/JVMプラットフォームにおける`Dispatchers.Default`の具体的な実装を見ていきましょう。

#### Android/JVMプラットフォーム上の`Dispatchers.Default`の　実装

Dispatchers.Defaultの実装は、`Dispatchers.kt`にあります^[`Dispatchers.Default`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/Dispatchers.kt#L16]。

```kt
public actual object Dispatchers {
    @JvmStatic
    public actual val Default: CoroutineDispatcher = DefaultScheduler
}
```

ここで用いられている`DefaultScheduler`は、`SchedulerCoroutineDispatcher`を継承しており、`dispatch`メソッドはそちらで定義されています^[`DefaultScheduler`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/Dispatcher.kt#L9]。

以下に、`SchedulerCoroutineDispatcher`の`dispatch`メソッドを抜粋して示します^[`SchedulerCoroutineDispatcher`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/Dispatcher.kt#L114]。

```kt
internal open class SchedulerCoroutineDispatcher(
    private val corePoolSize: Int = CORE_POOL_SIZE,
    private val maxPoolSize: Int = MAX_POOL_SIZE,
    private val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    private val schedulerName: String = "CoroutineScheduler",
) : ExecutorCoroutineDispatcher() {
    private var coroutineScheduler = createScheduler()

    private fun createScheduler() =
        CoroutineScheduler(corePoolSize, maxPoolSize, idleWorkerKeepAliveNs, schedulerName)

    override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block)
}
```

`dispatch`メソッドが呼び出されると、`CoroutineScheduler`の`dispatch`メソッドが呼ばれることが分かります。
この**`CoroutineScheluler`こそが、`CoroutineDispatcher`の内部実装の中核**といえます。

`CoroutineScheduler`は、その名の通り、実行タスクのスケジューリングを行います。
具体的には、「**タスクのスケジューリング (スレッドへのタスクの割り当てや実行順の制御)**」ならびに「**スレッドプール内のスレッド数の調整**」を行っています。

`CoroutineScheduler`は、**`Dispatchers.Default`と`Dispatchers.IO`の両方で利用するための共通のスレッドプール**を管理しています。
ただし、スレッドプールは共通であるものの、「**CPUバウンドな処理を実行するスレッド**」と「**IOバウンドなブロッキングタスクを実行するスレッド**」は、別々にカウントされています。`Dispatchers.Default`では、前者のスレッドが利用されており、こちらのスレッド数は上限に収まるようになっています。以下に模式図を示します。

![](https://storage.googleapis.com/zenn-user-upload/6ecb138fd4c5-20250613.png)

これら2種類のスレッドは、以下のような違いが存在します。

- **CPUバウンドな処理を実行するスレッド**
ワーカースレッドの数は、`corePoolSize`パラメータで決まる。基本的には、**CPUのコア数と同一**である (ただし、最低でも2以上の値を取り、例外的にシングルコアのCPUでは2となる)。**`Dispatchers.Default`では、こちらが利用される。**
- **ブロッキング処理を実行するスレッド**
ワーカースレッドは必要に応じて作成され、最大で`maxPoolSize`パラメータで指定された個数までスレッドが作成される。デフォルト値のMAX_POOL_SIZEは`1 << 21 = 2,097,152`で、実質的には無制限にブロッキングタスク用のバックグラウンドスレッドが作られる。**`Dispatchers.IO`では、こちらが利用される。**

以下のサンプルコードで、`Dispatchers.Default`と`Dispatchers.IO`で使用されるスレッド数の上限を確認してみましょう ([Kotlin Playground](https://pl.kotl.in/65reXK84Q))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    println("コア数: ${Runtime.getRuntime().availableProcessors()}")
    coroutineScope {
        for (i in 0..<8) {
            launch(Dispatchers.Default) {
                println("Current thread ID: ${Thread.currentThread().id}")
                Thread.sleep(10)
            }
        }
    }
    println("`Dispatchers.Default` finished.")
    coroutineScope {
        for (i in 0..<8) {
            launch(Dispatchers.IO) {
                println("Current thread ID: ${Thread.currentThread().id}")
                Thread.sleep(10)
            }
        }
    }
    println("`Dispatchers.IO` finished.")
}
```

出力の例:
```sh
コア数: 2
Current thread ID: 12
Current thread ID: 13
Current thread ID: 13
Current thread ID: 12
Current thread ID: 13
Current thread ID: 12
Current thread ID: 13
Current thread ID: 12
`Dispatchers.Default` finished.
Current thread ID: 13
Current thread ID: 14
Current thread ID: 16
Current thread ID: 15
Current thread ID: 17
Current thread ID: 18
Current thread ID: 20
Current thread ID: 12
`Dispatchers.IO` finished.
```

`Dispatchers.Default`の場合には、スレッド数に上限が存在しており、かつその上限が、コア数 (= 2) と合致していることが分かります。
他方、`Dispatchers.IO`では、スレッド数に上限がない (正確には上限が非常に大きい) ことが分かるます。また、`Dispatchers.Default`と`Dispatchers.IO`で同じスレッドが使用されており、共通のスレッドプールに基づいていることも示されています。

---

`CoroutineDispatcher`の全体像は、タスクスケジューリングの機構など、非常に複雑です。ここでは`dispatch`メソッドで行われていることを理解するに留めます。

以下に`CoroutineScheduler.dispatch`のソースコードを抜粋し、簡略化したものを示します^[`CoroutineScheduler.dispatch`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt#L394]。
補足として、`Dispatchers.IO`でもこのメソッドが使用されますが、説明のために、ここでは`Dispatchers.IO`に対応するためのコードを省いています。

```kt
    fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, fair: Boolean = false) {
        val task = createTask(block, taskContext)
        val currentWorker = currentWorker()
        val notAdded = currentWorker.submitToLocalQueue(task, fair)
        if (notAdded != null) {
            addToGlobalQueue(notAdded)
        }
        signalCpuWork()
    }
```

`CoroutineScheduler.dispatch`内には、主に3つのことが行われています。。

1. 処理 (`block: Runnable`) を、**スケジュール可能な形 (`Task`) へと変換**する。
2. **タスクキューにタスクを追加**する。
なお、タスクキューには、「スレッドごとのローカルキュー」と「全スレッドが参照できるグローバルキュー」の2種類が存在する。まずは、優先的にローカルキューへのプッシュし、所定のスレッドに対して処理を割り当てる。それに失敗した場合には、グローバルキューに追加する。
3. **`signalCpuWork`を実行**する。これにより、スレッドにタスクが追加されたことが通知され、処理の実行が促される。なお、タスクは同期的に実行されるわけではなく、各スレッドが非同期的に実行する。

`signalCpuWork`のコードは以下のようになっています。
`tryUnpark`は、idle状態 (Parked状態) のスレッドがあれば、それを呼び起こす処理です。
`tryCreateWorker`は、idle状態のスレッドが存在せず、かつスレッド数が未だ上限に達していなかった場合に、新たにスレッドを作成する処理です。

```kt
    fun signalCpuWork() {
        if (tryUnpark()) return
        if (tryCreateWorker()) return
        tryUnpark()
    }
```

このようにして、**スレッドが空いている、あるいはスレッドの追加作成の余地がある場合には、できるだけ速やかに処理が実行される**ようになっています。

なお、スレッドは必要に応じて追加されるだけではなく、スレッドが一定時間idle状態であった場合には、`tryTerminateWorker`メソッドが呼び出され、停止するようになっています ^[`CoroutineScheduler.tryTerminateWorker`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt#L867]。

### Dispatchers.IO

`Dispatchers.IO`は、IO-boundなブロッキングタスクの実行に適したCoroutineDispatcherです。
裏側では、`Dispatchers.Default`と共通のスレッドプール上で処理を実行します。
ただし、前述したように、`Dispatchers.Default`とは別でスレッド数の上限が管理されており、より多くのスレッドを起動することが可能です。

では、上記の仕様が、どのように実現されているのかを内部実装から探っていきましょう。

JVMにおいては、`Dispatchers.IO`として`DefaultIoScheduler`が使用されます。

```kt
public actual object Dispatchers {
    @JvmStatic
    public val IO: CoroutineDispatcher get() = DefaultIoScheduler
}
```

`DefaultIoScheduler.dispatch`のソースコードを以下に抜粋します。

```kt
internal object DefaultIoScheduler : ExecutorCoroutineDispatcher(), Executor {

    private val default = UnlimitedIoScheduler.limitedParallelism(
        systemProp(
            IO_PARALLELISM_PROPERTY_NAME,
            64.coerceAtLeast(AVAILABLE_PROCESSORS)
        )
    )

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        default.dispatch(context, block)
    }
}
```

`DefaultIoScheduler.dispatch`内では、`UnlimitedIoScheduler.limitedParallelism`から得られる`LimitedDispatcher` (`CoroutineDispatcher`のサブクラス) に対して`dispatch`が呼ばれています。

なお、ここで`UnlimitedIoScheduler.limitedParallelism`の`parallelism`パラメータに`64.coerceAtLeast(AVAILABLE_PROCESSORS)`が渡されています。**`Dispatchers.IO`が同時並行に実行できるスレッド数の上限は、デフォルトで64に設定されている**ことが分かります。

`LimitedDispatcher`とは、同時並行に実行されるスレッド数を制限するための仕組みを持った`CoroutineDispatcher`です。`LimitedDispatcher`の内部実装に関しては、後ほど別のセクションで詳細に見るため、ここでは詳細な説明は省かせていただきます。

`LimitedDispatcher.dispatch`を経由して、`UnlimitedIoScheduler`の`dispatch`メソッドが呼ばれます ^[`LimitedDispatcher.dispatch`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/LimitedDispatcher.kt#L45]。

`UnlimitedIoScheduler`の`dispatch`メソッドを以下に示します ^[`UnlimitedIoScheduler.dispatch`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/Dispatcher.kt#L43]。

```kt
private object UnlimitedIoScheduler : CoroutineDispatcher() {
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        DefaultScheduler.dispatchWithContext(block, BlockingContext, false)
    }
}
```

`DefaultScheduler.dispatchWithContext`が呼ばれています。
**注目すべき点として、2つ目の引数に`BlockingContext`が渡されています**。この意味については後ほど解説します。

これは、基底クラスである`SchedulerCoroutineDispatcher`の`dispatchWithContext`メソッドを呼び出し、最終的に`SchedulerCoroutineDispatcher.dispatch`が呼ばれます^[`SchedulerCoroutineDispatcher.dispatchWithContext`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/Dispatcher.kt#L129]。
`SchedulerCoroutineDispatcher.dispatch`は、`Dispatchers.Defalult`においてもタスクスケジューリングの根本を担っていたメソッドです。

```kt
internal open class SchedulerCoroutineDispatcher() {
    internal fun dispatchWithContext(block: Runnable, context: TaskContext, fair: Boolean) {
        coroutineScheduler.dispatch(block, context, fair)
    }
```

ただし、`Dispatchers.Default`と`Dispatchers.IO`は同じメソッドを利用しているものの、先述した`BlockingContext`が渡されている点が大きな違いを生じます。

以下に、`Dispatchers.IO`から呼ばれる場合の`SchedulerCoroutineDispatcher.dispatch`のコードを簡略化して示します^[`SchedulerCoroutineDispatcher.dispatch`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt#L393]。

```kt
    fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, fair: Boolean = false) {
        val task = createTask(block, taskContext)
        val stateSnapshot = incrementBlockingTasks()
        val currentWorker = currentWorker()
        val notAdded = currentWorker.submitToLocalQueue(task, fair)
        if (notAdded != null) {
            addToGlobalQueue(notAdded)
        }
        signalBlockingWork(stateSnapshot)
    }
```

`Dispatchers.Default`との違いとして、**「`incrementBlockingTasks()`を呼ぶことで、ブロッキングタスクの個数をインクリメントしていること」および「`signalCpuWork`の代わりに`signalBlockingWork`を呼んでいること」**の2点があります。

`signalBlockingWork`メソッドのソースコードを以下に示します^[`SchedulerCoroutineDispatcher.signalBlockingWork`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt#L430]。

```kt
    private fun signalBlockingWork(stateSnapshot: Long) {
        if (tryUnpark()) return
        if (tryCreateWorker(stateSnapshot)) return
        tryUnpark()
    }
```

`tryUnpark`は、`Dispatchers.Default`のセクションでも説明したように、idle状態のスレッドがあれば、それを確保するためのものです。

`tryCreateWorker`は、idle状態のスレッドがない場合に、新たにスレッドを追加します。`tryCreateWorker`のソースコードを以下に示します^[`SchedulerCoroutineDispatcher.tryCreateWorker`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/scheduling/CoroutineScheduler.kt#L443]

```kt
    private fun tryCreateWorker(state: Long = controlState.value): Boolean {
        val created = createdWorkers(state)
        val blocking = blockingTasks(state)
        val cpuWorkers = (created - blocking).coerceAtLeast(0)
        if (cpuWorkers < corePoolSize) {
            val newCpuWorkers = createNewWorker()
            if (newCpuWorkers == 1 && corePoolSize > 1) createNewWorker()
            if (newCpuWorkers > 0) return true
        }
        return false
    }
```

`tryCreateWorker`のソースコードから、**「全スレッド数 - ブロッキングタスク用のスレッド数」が、CPUタスク用のスレッド数の上限 (基本的にはCPUのコア数) を下回る場合には、新たに共通スレッドプールに対してスレッドが追加される**ことが分かります。`Dispatchers.IO`経由でdispatchを実行する際には、「ブロッキングタスクの個数」が加算されるため、`tryCreateWorker`が呼ばれた際には**新しいスレッドが1つ作られる**ことになります。

このようなメカニズムによって、「CPUバウンドな処理用のスレッド (上限はCPUのコア数)」と「IOバウンドなブロッキングタスク用のスレッド」のスレッド数を、裏側では共通スレッドプールを用いつつ、別々に管理することが可能となっています。

### Dispatchers.Main

`Dispatchers.Main`は、メインスレッド (Androidの場合はUIスレッド) で実行するための`CoroutineDispatcher`です。

以下に、JVMプラットフォームにおける`Dispatchers.Main`のソースコードを示します^[`Dispatchers.Main`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/Dispatchers.kt#L19]。

```
public actual object Dispatchers {
    @JvmStatic
    public actual val Main: MainCoroutineDispatcher get() = MainDispatcherLoader.dispatcher
```

`MainDispatcherLoader.dispatcher`が使用されています。
そこで、`MainDispatcherLoader`のソースコードを以下に示します^[`MainDispatcherLoader`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/internal/MainDispatchers.kt#L13]。

```kt
internal object MainDispatcherLoader {

    private val FAST_SERVICE_LOADER_ENABLED = systemProp(FAST_SERVICE_LOADER_PROPERTY_NAME, true)

    @JvmField
    val dispatcher: MainCoroutineDispatcher = loadMainDispatcher()

    private fun loadMainDispatcher(): MainCoroutineDispatcher {
        return try {
            val factories = if (FAST_SERVICE_LOADER_ENABLED) {
                FastServiceLoader.loadMainDispatcherFactory()
            } else {
                // We are explicitly using the
                // `ServiceLoader.load(MyClass::class.java, MyClass::class.java.classLoader).iterator()`
                // form of the ServiceLoader call to enable R8 optimization when compiled on Android.
                ServiceLoader.load(
                        MainDispatcherFactory::class.java,
                        MainDispatcherFactory::class.java.classLoader
                ).iterator().asSequence().toList()
            }
            @Suppress("ConstantConditionIf")
            factories.maxByOrNull { it.loadPriority }?.tryCreateDispatcher(factories)
                ?: createMissingDispatcher()
        } catch (e: Throwable) {
            // Service loader can throw an exception as well
            createMissingDispatcher(e)
        }
    }
}
```

`MainDispatcherLoader.dispatcher`が、`MainDispatcherFactory`で作られることが分かります。
**`MainDispatcherFactory`の実装は、実行環境ごとに定義**されています。例えば、kotlinx-coroutines-android`内の`AndroidDispatcherFactory`、`kotlinx-coroutines-javafx`内の`JavaFxDispatcherFactory`、`kotlinx-coroutines-swing`内の`SwingDispatcherFactory`などが存在します。

`AndroidDispatcherFactory`をソースコードの一部を抜粋します^[`AndroidDispatcherFactory`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/ui/kotlinx-coroutines-android/src/HandlerDispatcher.kt#L48]。

```kt
internal class AndroidDispatcherFactory : MainDispatcherFactory {
    override fun createDispatcher(allFactories: List<MainDispatcherFactory>): MainCoroutineDispclass Handler(atcher {
        val mainLooper = Looper.getMainLooper() ?: throw IllegalStateException("The main looper is not available")
        return HandlerContext(mainLooper.asHandler(async = true))
    }
}
```

`Looper.getMainLooper()`^[Looper: https://developer.android.com/reference/android/os/Looper]から作成された`Handler`^[Handler: https://developer.android.com/reference/android/os/Handler]を使って、`HandlerContext`が作られています。
補足すると、`Looper`とは、Android SDKで定義されているクラスで、**スレッド上でメッセージループを実行する**ためのオブジェクトです。また、`Handler`もAndroid SDKで定義されているクラスで、**メッセージループのキューに対して処理 (Runnable) を渡すためのオブジェクト**です。

`HandlerContext`の一部を以下に示します^[`HandlerContext`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/ui/kotlinx-coroutines-android/src/HandlerDispatcher.kt#L110]。

```kt
internal class HandlerContext private constructor(
    private val handler: Handler,
    private val name: String?,
    private val invokeImmediately: Boolean
) : HandlerDispatcher(), Delay {
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        if (!handler.post(block)) {
            cancelOnRejection(context, block)
        }
    }
}
```

`HandlerContext`は、**Kotlin Coroutineから`Handler`に対して処理を送るためのWrapper**です。つまり、`HandlerContext`を経由して、AndroidのUIスレッドに対して、処理の実行を依頼することが可能です。

`dispatch`が呼ばれると、`handler.post`に`block`が渡されることが分かります。Handlerのpostメソッドは、UIスレッド用の実行キューに対して処理を渡します。

このようにして、`Dispatchers.Main`は実行環境に応じたメインスレッドに対して、処理の依頼を可能としています。

#### `Dispatchers.Main.immediate`について

`Dispatchers.Main`の代わりに、`Dispatchers.Main.immediate`と指定することもできます。
この場合、すでに実行スレッドがUIスレッドだった場合には、`dispatch`をスキップして、そのままUIスレッド上で処理が実行されます。これにより、`dispatch`にかかるオーバーヘッドを削減し、パフォーマンスを改善することができます。

`Dispatchers.Main.immediate`がどのように実現されているのを、`HandlerContext`の内部実装から確かめてみましょう。

```kt
internal class HandlerContext private constructor(
    private val handler: Handler,
    private val name: String?,
    private val invokeImmediately: Boolean
) : HandlerDispatcher(), Delay {
    override val immediate: HandlerContext = if (invokeImmediately) this else
        HandlerContext(handler, name, true)

    override fun isDispatchNeeded(context: CoroutineContext): Boolean {
        return !invokeImmediately || Looper.myLooper() != handler.looper
    }

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        if (!handler.post(block)) {
            cancelOnRejection(context, block)
        }
    }
}
```

`Dispatchers.Main.immediate`の場合には、`invokeImmediately`がtrueとなります。
`isDispatchNeeded`メソッドを見ると、`Looper.myLooper() != handler.looper`により、現在の実行スレッドがUIスレッドだった場合にはfalseが返され、`dispatch`がスキップされることが分かります。

### Dispatchers.Unconfined

`Dispatchers.Unconfined`は、特殊なDispatcherで、Coroutineが実行されるスレッドを指定しません ^[Unconfined: https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html#unconfined-vs-confined-dispatcher]。

Coroutineの起動後は、そのCoroutineが起動されたスレッド上で、そのまま処理が実行されます。内部の動作としては、`isDispatchNeeded`がfalseとなるため、`dispatch`がスキップされます。
その後、suspension pointで処理が再開された場合には、そのsuspend関数が実行されていたスレッド上で、そのまま処理が再開されます。例えば、`delay`でsuspendされた場合には、`delay`メソッドが使用していたスレッドでそのまま再開後の処理も継続されます。

以下のサンプルコードで動作を確認してみましょう ([Kotlin Playground](https://pl.kotl.in/OiUZy12Wa))

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        // ID=1のスレッドで実行
        println("0: Current thread ID: ${Thread.currentThread().id}")
        launch(Dispatchers.Unconfined) {
            // dispatchされず、ID=1のスレッドでそのまま実行
            println("1: Current thread ID: ${Thread.currentThread().id}")
            delay(100) // delayはバックグラウンドスレッドの1つ (ID=12のスレッド) で実行
            // dispatchされず、ID=12のスレッドでそのまま実行
            println("2: Current thread ID: ${Thread.currentThread().id}")
            launch(Dispatchers.Unconfined) {
                // dispatchされず、ID=12のスレッドでそのまま実行
                println("3: Current thread ID: ${Thread.currentThread().id}")
            }
        }
    }
})。
```

出力の例:
```sh
0: Current thread ID: 1
1: Current thread ID: 1
2: Current thread ID: 12
3: Current thread ID: 12
```

`Dispatchers.Unconfined`の利用場面としては、不要な`dispatch`を無くすことでパフォーマンス上のオーバーヘッドを削減したいケース、あるいは`dispatch`が望ましくない副作用を引き起こすことを避けたいケースなどが考えられます。
ただ、一般的な利用の範囲内では、あえて`Dispatchers.Unconfined`を利用すべきケースはほとんどありません。そのため、`Dispatchers.Unconfined`に関しては、内部実装の説明はスキップさせていただきます。

## limitedParallelismの動作・仕組み

`CoroutineDispatcher`のインタフェースには、`limitedParallelism`というメソッドが定義されています。これは、`Dispatchers.Default`や`Dispatchers.IO`などのスレッドプールに対して処理を送信する`CoroutineDispatcher`において、**同時並行で実行されるスレッド数を制限する**ための仕組みです。

`limitedParallelism`の動作を、サンプルコードを用いて実際に確認してみましょう ([Kotlin Playground](https://pl.kotl.in/PcuWUpg46))。
`Dispatchers.IO`を使用して、同時並行に6個のCoroutineを立ち上げてみます。各Coroutine内では、スレッドを1秒間スリープさせます。すると、ID 12からID 17までの6個のスレッドが起動し、同時並行で実行が進むことが分かります。同時並行で処理が進むので、合計所要時間も1秒程度です。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    val start = System.currentTimeMillis()
    val dispatcher = Dispatchers.IO
    coroutineScope {
        for (i in 0..5) {
            launch(dispatcher) {
                println("i=$i, Current thread ID: ${Thread.currentThread().id}")
                Thread.sleep(1000)
            }
        }
    }
    val end = System.currentTimeMillis()
    println("Duration milliseconds=${end - start}")
}
```

出力:
```sh
i=2, Current thread ID: 14
i=0, Current thread ID: 12
i=4, Current thread ID: 16
i=1, Current thread ID: 13
i=3, Current thread ID: 15
i=5, Current thread ID: 17
Duration milliseconds=1117
```

続いて、`Dispatchers.IO.limitedParallelism(2)`に変えてみましょう ([Kotlin Playground](https://pl.kotl.in/phGi0n3OO))。
サンプルコードを実行すると、その`CoroutineDispatcher`を利用しているスコープにおいて、同時に実行されるコルーチンの個数が2つに制限されることが分かります。並列数が2になるため、合計実行時間は3秒程度になっています。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    val start = System.currentTimeMillis()
    val dispatcher = Dispatchers.IO.limitedParallelism(2)
    coroutineScope {
        for (i in 0..5) {
            launch(dispatcher) {
                println("i=$i, Current thread ID: ${Thread.currentThread().id}")
                Thread.sleep(1000)
            }
        }
    }
    val end = System.currentTimeMillis()
    println("Duration milliseconds=${end - start}")
}
```

出力の例:
```sh
i=0, Current thread ID: 12
i=1, Current thread ID: 13
i=2, Current thread ID: 13
i=3, Current thread ID: 12
i=5, Current thread ID: 12
i=4, Current thread ID: 13
Duration milliseconds=3103
```

ただし、誤解を与えないよう補足すると、`limitedParallelism`は専用のスレッドプールを作るわけではありません。`Dispatchers.Default`や`Dispatchers.IO`と共通のスレッドプールを利用します。
また、あくまで同時に実行されるスレッドの数が指定されるのみで、どのスレッドで実行されるかが固定されることもありません。上の例では、ID 12, ID 13の2つのスレッドで実行されましたが、あるタイムスパンでは別の組み合わせ (例えば、ID 12, ID 14の組み合わせ) で実行されることも考えられます。

### `limitedParallelism`の内部実装

では、`limitedParallelism`がどのように機能を実現しているか、内部実装を見ながら解説します。
`limitedParallelism`は、基底クラスの`CoroutineDispatcher`に対して定義されています [CoroutineDispatcher.limitedParallelismのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineDispatcher.kt#L176]。

```kt
public abstract class CoroutineDispatcher {
    public open fun limitedParallelism(parallelism: Int, name: String? = null): CoroutineDispatcher {
        parallelism.checkParallelism()
        return LimitedDispatcher(this, parallelism, name)
    }
}
```

`LimitedDispatcher`のインスタンスが返されます。これは先述した`Dispatchers.IO`の解説でも登場しました。再び`LimitedDispatcher`の主要部分を抜粋して示します。

```kt
internal class LimitedDispatcher(
    private val dispatcher: CoroutineDispatcher,
    private val parallelism: Int,
    private val name: String?
) : CoroutineDispatcher(), Delay by (dispatcher as? Delay ?: DefaultDelay) {
    private val runningWorkers = atomic(0)
    private val queue = LockFreeTaskQueue<Runnable>(singleConsumer = false)

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        dispatchInternal(block) { worker ->
            dispatcher.safeDispatch(this, worker)
        }
    }

    // 渡された`block`を`dispatch`する。Workerが不足する場合には新たに起動する。
    private inline fun dispatchInternal(block: Runnable, startWorker: (Worker) -> Unit) {
        // タスクキューにタスクを追加する。
        queue.addLast(block)
        // 実行中のWorker数が`parallelism`に達している場合にはreturnする。
        if (runningWorkers.value >= parallelism) return
        // Worker数を増やす。
        if (!tryAllocateWorker()) return
        val task = obtainTaskOrDeallocateWorker() ?: return
        try {
            // 増やしたWorkerでタスクを実行開始する。
            startWorker(Worker(task))
        } catch (e: Throwable) {
            runningWorkers.decrementAndGet()
            throw e
        }
    }

    private inner class Worker(private var currentTask: Runnable) : Runnable {
        override fun run() {
            try {
                while (true) {
                    try {
                        // 初期タスクを実行する。
                        currentTask.run()
                    } catch (e: Throwable) {
                        handleCoroutineException(EmptyCoroutineContext, e)
                    }
                    // キューにまだタスクが残っている場合には、拾って実行する。
                    currentTask = obtainTaskOrDeallocateWorker() ?: return
                }
            } catch (e: Throwable) {
                synchronized(workerAllocationLock) {
                    runningWorkers.decrementAndGet()
                }
                throw e
            }
        }
    }
}
```

`LimitedDispatcher`は、主に以下の3つの構成要素から成ります。

1. タスクを実行する (`CoroutineDispatcher`の`dispatcher`を呼ぶ) ためのWorker
2. 並列実行中のWorker数を管理・調整するためのAtomic Integerの変数 (`runningWorkers`)
3. タスクを保持・バッファリングするためのタスクキュー (`queue`)

`LimitedDispatcher`の`dispatch`メソッドが呼び出されると、まずはタスクキューにタスクが追加されます。その後、実行中の`Worker`の数が上限に達していない場合 (`runningWorkers < `parallelism`の場合) には、新たな`Worker`を追加し、起動します。この時`runningWorkers`がインクリメントされます。
実行中の`Worker`は、タスクキューが空になるまで、タスクを拾って実行し続けます。`Worker`がidle状態になると解放され、`runningWorkers`がデクリメントされます。

このようにして、並列実行中のタスク数を制限する仕組みを実現しています。スレッドプールのサイズ設定に関わらず、インメモリの変数で並列度を制御していることが分かります。
