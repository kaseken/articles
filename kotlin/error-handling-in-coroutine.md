# クイズで理解度を確かめよう！Kotlin Coroutinesの例外処理のフローチャート

本記事では、「**Kotlin Coroutinesの例外処理**」について解説します。
解説に入る前に、現時点での理解度を確かめるために、まずはクイズを解いてみましょう！

## クイズ

クイズは全10問です。
それぞれのサンプルコードを実行した結果、得られる出力を答えて下さい。

### 第1問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        coroutineScope {
            try {
                launch(handler) {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/ZsBLxkOZx)
```
③
```
:::

### 第2問

第1問の`coroutineScope`を、**`supervisorScope`に変えた**ものです。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        supervisorScope {
            try {
                launch(handler) {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Fib3b3ufC)
```
①
```
:::

### 第3問

`GlobalScope`からの`launch`でコルーチンを起動しています。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        val job = GlobalScope.launch(handler) {
            throw SomeException()
        }
        job.join()
    } catch (e: SomeException) {
        println("②")
    }
}

```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/NGUJrJ9tk)
```
①
```
:::

### 第4問

`launch`の代わりに、**`async`を使用**しています。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        coroutineScope {
            val result = async(handler) {
                throw SomeException()
            }
            try {
                result.await()
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/mt-vcurKu)
```
②
③
```
:::

### 第5問

第4問の`coroutineScope`を、**`supervisorScope`に変えた**ものです。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        supervisorScope {
            val result = async(handler) {
                throw SomeException()
            }
            try {
                result.await()
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/8C8C_5peJ)
```
②
```
:::

### 第6問

`GlobalScope`から`async`を起動しています。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        val result = GlobalScope.async(handler) {
            throw SomeException()
        }
        try {
            result.await()
        } catch (e: SomeException) {
            println("②")
        }
    } catch (e: SomeException) {
        println("③")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/LVTmSStR8)
```
②
```
:::

### 第7問

`launch`から、更に`launch`で子コルーチンが起動されています。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            val handler = CoroutineExceptionHandler { _, _ -> println("①") }
            launch(handler) {
                val handler2 = CoroutineExceptionHandler { _, _ -> println("②") }
                launch(handler2) {
                    val handler3 = CoroutineExceptionHandler { _, _ -> println("③") }
                    launch(handler3) {
                        throw SomeException()
                    }
                }
            }
        }  
    } catch (e: SomeException) {
        println("④")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/ZmiQjJiCU)
```
④
```
:::

### 第8問

`async`から、更に`async`で子コルーチンが起動されています。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            val r1 = async {
                val r2 = async {
                    val r3 = async {
                        throw SomeException()
                    }
                    try {
                        r3.await()
                    } catch (e: SomeException) {
                        println("①")
                    }
                }
                try {
                    r2.await()
                } catch (e: SomeException) {
                    println("②")
                }
            }
            try {
                r1.await()
            } catch (e: SomeException) {
                println("③")
            }
        }
    } catch (e: SomeException) {
        println("④")
    }
}
```

:::details 答え
※①②③の出力される順番は非決定的である。
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/LWQbi39IO)
```
①
②
③
④
```
:::

### 第9問

第2問の、**`CoroutineExceptionHandler`を`launch`に与えていない**バージョンです。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        supervisorScope {
            try {
                launch {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("①")
            }
        }  
    } catch (e: SomeException) {
        println("②")
    }
}
```

:::details 答え
※この問題の出力は、実行プラットフォームに依存します。Kotlin Playground上では、以下のような出力となります。
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/DWj0g9jSL)
```
Exception in thread "DefaultDispatcher-worker-1 @coroutine#1" SomeException: 
	at FileKt$main$2$1.invokeSuspend(File.kt:10)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:108)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:584)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:793)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:697)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:684)
	Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [CoroutineId(1), "coroutine#1":StandaloneCoroutine{Cancelling}@435ce98, Dispatchers.Default]
```
:::

### 第10問

**`coroutineScope`内に`coroutineScope`が存在する**ケースです。

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            try {
                coroutineScope {
                    launch {
                        throw SomeException()
                    }
                }  
            } catch (e: SomeException) {
                println("①")
            }
        }  
    } catch (e: SomeException) {
        println("②")
    }
}
```

:::details 答え
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Tpd08ECa6)
```
①
```
:::

---

お疲れ様でした。クイズは以上です。

「全問正解できた」かつ「その理由を説明できる」という方は、既にCoroutinesマスターです。ここで引き返していただいて問題ありません。

一方で、分からない問題があった方は、ぜひ以降の解説を読み、これを機に例外処理に関する理解を深めていただければと思います。

## 例外が投げられる条件

クイズの解説に入る前に、例外処理の流れを解説します。

コルーチン内で例外が投げられると、以下のようなルールに沿って、例外が処理されます。

1. `launch`と`async`の両方で、**親コルーチン (`launch`や`async`) が存在する場合、例外は親に伝播**する。この際、**`CoroutineExceptionHandler`は使われない**。
2. `launch`と`async`の両方で、親が`coroutineScope`の場合には、**`coroutineScope`から例外が投げられる**。
3. `launch`のみ、「**親が`supervisorScope`**」あるいは「**親が存在しない場合**」には、**`CoroutineExceptionHandler`によって例外が処理**される。
4. `async`のみ、**`async`の返り値 (`Deferred`インスタンス) に対して`await`を呼び出した際に例外が投げられる**。

これらのルールをフローチャートに整理すると、以下のようになります。

![](https://storage.googleapis.com/zenn-user-upload/0be95df572a6-20250624.png)
*例外が投げられる条件のフローチャート*

## クイズの解説

各問題を、先述したルール・フローチャートに従って解説していきます。

### 第1問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        coroutineScope {
            try {
                launch(handler) {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/ZsBLxkOZx)
```
③
```

#### 解説

**親が`coroutineScope`なので、`coroutineScope`から例外が投げられます。**
`launch`自体からは例外は投げられません。
また、**例外を親に伝播できる場合 (`coroutineScope`や`launch`、`async`が親として存在する場合) には、`launch`に渡された`CoroutineExceptionHandler`は使用されません**。

![](https://storage.googleapis.com/zenn-user-upload/10dfcc6b13ad-20250625.png)

親コルーチンが存在する場合に`CoroutineExceptionHandler`が使われない理由を、内部実装からも確認してみましょう。
`JobSupport`の`finalizeFinishingState`^[`JobSupport`の`finalizeFinishingState`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L222]に、以下のようなコードがあります。
 
```kt
val handled = cancelParent(finalException) || handleJobException(finalException)
```

`cancelParent`メソッドは、親コルーチンが例外処理の責務を持つ場合に`true`を返します。
また、`handleJobException`内で、`CoroutineExceptionHandler`を使った例外処理が行われます。つまり、**親コルーチンに例外処理が委譲可能な場合、`CoroutineExceptionHandler`の使用がスキップされる**ことが分かります。

### 第2問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        supervisorScope {
            try {
                launch(handler) {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Fib3b3ufC)
```
①
```

#### 解説

**親が`supervisorScope`なので、親コルーチンに例外処理を委譲できず、`CoroutineExceptionHandler`が使われます**。

![](https://storage.googleapis.com/zenn-user-upload/62051bc84519-20250625.png)

`supervisorScope`で`CoroutineExceptionHandler`が使われる理由を、内部実装からも確認します。

第1問の解説で述べたように、`cancelParent`がtrueを返すと、親コルーチンに例外処理が委譲され、`CoroutineExceptionHandler`の利用がスキップされることとなります。

`JobSupport`の`cancelParent`のソースコード^[`JobSupport`の`cancelParent`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L222]を以下に示します。

```kt
    private fun cancelParent(cause: Throwable): Boolean {
        if (isScopedCoroutine) return true

        val isCancellation = cause is CancellationException
        val parent = parentHandle
        // No parent -- ignore CE, report other exceptions.
        if (parent === null || parent === NonDisposableHandle) {
            return isCancellation
        }

        return parent.childCancelled(cause) || isCancellation
    }
```

親コルーチン (の`Job`) に対して`parent.childCancelled`が呼ばれています。

`supervisorScope`で使用される`SupervisorCoroutine`を見ると、以下のように`childCancelled`メソッドがoverrideされており、falseを返すのみとなっています^[`SupervisorCoroutine`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Supervisor.kt#L64]。そのため、`cancelParent`がfalseを返し、`CoroutineExceptionHandler`を使った例外処理が行われることとなります。

```kt
private class SupervisorCoroutine<in T>(
    context: CoroutineContext,
    uCont: Continuation<T>
) : ScopeCoroutine<T>(context, uCont) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

### 第3問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, _ -> println("①") }
    try {
        val job = GlobalScope.launch(handler) {
            throw SomeException()
        }
        job.join()
    } catch (e: SomeException) {
        println("②")
    }
}

```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/NGUJrJ9tk)
```
①
```

#### 解説

`GlobalScope`からコルーチンを起動すると親コルーチン (親の`Job`) が存在しない状態となります。そのため、`CoroutineExceptionHandler`が使われます。

![](https://storage.googleapis.com/zenn-user-upload/c5ddad91cada-20250625.png)

:::message alert
**実際のコードでは、`GlobalScope`は使うべきではありません**^[The reason to avoid GlobalScope: https://elizarov.medium.com/the-reason-to-avoid-globalscope-835337445abc]

基本的に、コルーチンは`CoroutineScope`を通じて起動されるもので、「親のコルーチンが存在しない」という状況が生じることは滅多にないと思われます。

一方で、Kotlinの公式ドキュメント^[Coroutine exceptions handling: https://kotlinlang.org/docs/exception-handling.html]では、このレアケースである`GlobalScope`を使った状況から説明されており、個人的には分かりにくさを生んでいると思います。
:::

### 第4問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        coroutineScope {
            val result = async(handler) {
                throw SomeException()
            }
            try {
                result.await()
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/mt-vcurKu)
```
②
③
```

#### 解説

`await`の場合、返り値の`Deferred`に対して`await`した際に例外が投げられます。
一方で、親コルーチンへの例外の伝播も起こるため、`coroutineScope`からも例外が投げられます。

![](https://storage.googleapis.com/zenn-user-upload/3ce6e726318f-20250625.png)

### 第5問


```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        supervisorScope {
            val result = async(handler) {
                throw SomeException()
            }
            try {
                result.await()
            } catch (e: SomeException) {
                println("②")
            }
        }  
    } catch (e: SomeException) {
        println("③")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/8C8C_5peJ)
```
②
```

#### 解説

第4問と同様に、`await`した際に例外が投げられます。
一方で、親が`supervisorScope`のため、`supervisorScope`へは例外が伝播せず、そこからは例外が投げられません。
また、**`launch`との大きな違いとして、`CoroutineExceptionHandler`は呼ばれません**。

![](https://storage.googleapis.com/zenn-user-upload/628261cccdc8-20250625.png)

`async`で`CoroutineExceptionHandler`が使われない理由を、内部実装からも確認してみます。
`launch`では`StandaloneCoroutine`が用いられ、以下のように`handleJobException`メソッドがoverrideされています^[`StandaloneCoroutine`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L187]。

```kt
private open class StandaloneCoroutine(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<Unit>(parentContext, initParentJob = true, active = active) {
    override fun handleJobException(exception: Throwable): Boolean {
        handleCoroutineException(context, exception)
        return true
    }
}
```

この`handleJobException`は、親コルーチンに例外処理を委譲できなかった場合 (`cancelParent`がfalseを返した場合) に呼ばれるメソッドです。このメソッド内の`handleCoroutineException`で、`CoroutineExceptionHandler`を使った例外処理が行われます。

他方、`async`では、`DeferredCoroutine`が使われます。

```kt
private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T> {
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = onAwaitInternal as SelectClause1<T>
}
```

こちらは`handleJobException`の実装が存在せず、基底クラスの`JobSupport`にある実装 (falseを返すのみ) が使われることとなります。
そのため、**`async`では`CoroutineExceptionHandler`が使用されません**。
`await`時に例外を捕捉できるため、そちらで例外処理をすべきという設計上の意図と思われます。

### 第6問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, e -> println("①") }
    try {
        val result = GlobalScope.async(handler) {
            throw SomeException()
        }
        try {
            result.await()
        } catch (e: SomeException) {
            println("②")
        }
    } catch (e: SomeException) {
        println("③")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/LVTmSStR8)
```
②
```

#### 解説

`GlobalScope`から起動されたコルーチンには、親コルーチンが存在しないため、親へと例外が伝播せず、単に`await`時に例外が投げられるのみとなります。

![](https://storage.googleapis.com/zenn-user-upload/32f74f8356ef-20250625.png)

### 第7問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            val handler = CoroutineExceptionHandler { _, _ -> println("①") }
            launch(handler) {
                val handler2 = CoroutineExceptionHandler { _, _ -> println("②") }
                launch(handler2) {
                    val handler3 = CoroutineExceptionHandler { _, _ -> println("③") }
                    launch(handler3) {
                        throw SomeException()
                    }
                }
            }
        }  
    } catch (e: SomeException) {
        println("④")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/ZmiQjJiCU)
```
④
```

#### 解説

親コルーチンへと例外が伝播したのち、最終的に`coroutineScope`から例外が投げられます。

![](https://storage.googleapis.com/zenn-user-upload/18e84634adab-20250625.png)

### 第8問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            val r1 = async {
                val r2 = async {
                    val r3 = async {
                        throw SomeException()
                    }
                    try {
                        r3.await()
                    } catch (e: SomeException) {
                        println("①")
                    }
                }
                try {
                    r2.await()
                } catch (e: SomeException) {
                    println("②")
                }
            }
            try {
                r1.await()
            } catch (e: SomeException) {
                println("③")
            }
        }
    } catch (e: SomeException) {
        println("④")
    }
}
```

#### 答え

※①②③の出力される順番は非決定的である。
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/LWQbi39IO)
```
①
②
③
④
```

#### 解説

第7問の`launch`の場合と同様に、親コルーチンへと例外が伝播したのち、最終的に`coroutineScope`から例外が投げられます。
また、それぞれの`await`の結果を`await`したタイミングでも、例外が投げられます。
たとえ`await`で例外を捕捉しても、親コルーチンへの例外の伝播は生じることは覚えておきましょう。

![](https://storage.googleapis.com/zenn-user-upload/2ce8f343fbaa-20250625.png)

### 第9問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        supervisorScope {
            try {
                launch {
                    throw SomeException()
                }
            } catch (e: SomeException) {
                println("①")
            }
        }  
    } catch (e: SomeException) {
        println("②")
    }
}
```

#### 答え

※この問題の出力は、実行プラットフォームに依存します。Kotlin Playground上では、以下のような出力となります。
▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/DWj0g9jSL)
```
Exception in thread "DefaultDispatcher-worker-1 @coroutine#1" SomeException: 
	at FileKt$main$2$1.invokeSuspend(File.kt:10)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:108)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:584)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:793)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:697)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:684)
	Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [CoroutineId(1), "coroutine#1":StandaloneCoroutine{Cancelling}@435ce98, Dispatchers.Default]
```

#### 解説

`CoroutineExceptionHandler`が使われるべき条件で、明示的に`CoroutineExceptionHandler`が設定されていない場合、デフォルトのハンドラが使用されます。

![](https://storage.googleapis.com/zenn-user-upload/7494a805d7eb-20250625.png)

内部実装からも、この挙動を説明してみます。
`handleCoroutineException`のソースコード^[`handleCoroutineException`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineExceptionHandler.kt#L18]を見ると、`CoroutineContext`に`CoroutineExceptionHandler`が存在する場合にはそれが使用され、存在しない場合には`handleUncaughtCoroutineException`メソッドにフォールバックすることが分かります。

```kt
public fun handleCoroutineException(context: CoroutineContext, exception: Throwable) {
    val reportException = if (exception is DispatchException) exception.cause else exception
    // Invoke an exception handler from the context if present
    try {
        context[CoroutineExceptionHandler]?.let {
            it.handleException(context, reportException)
            return
        }
    } catch (t: Throwable) {
        handleUncaughtCoroutineException(context, handlerException(reportException, t))
        return
    }
    // If a handler is not present in the context or an exception was thrown, fallback to the global handler
    handleUncaughtCoroutineException(context, reportException)
}
```

`handleUncaughtCoroutineException`のソースコードを以下に示します^[`handleUncaughtCoroutineException`: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/CoroutineExceptionHandlerImpl.common.kt#L30]。

```kt
internal fun handleUncaughtCoroutineException(context: CoroutineContext, exception: Throwable) {
    // use additional extension handlers
    for (handler in platformExceptionHandlers) {
        try {
            handler.handleException(context, exception)
        } catch (_: ExceptionSuccessfullyProcessed) {
            return
        } catch (t: Throwable) {
            propagateExceptionFinalResort(handlerException(exception, t))
        }
    }

    try {
        exception.addSuppressed(DiagnosticCoroutineContextException(context))
    } catch (e: Throwable) {
        // addSuppressed is never user-defined and cannot normally throw with the only exception being OOM
        // we do ignore that just in case to definitely deliver the exception
    }
    propagateExceptionFinalResort(exception)
}
```

`handleUncaughtCoroutineException`内の処理は、実行プラットフォームに依存します。
プラットフォームごとに定義された`platformExceptionHandlers`に対して`handleException`が呼び出され、例外処理が行われます。

### 第10問

```kt
import kotlinx.coroutines.*

private class SomeException(): Exception("")

suspend fun main() {
    try {
        coroutineScope {
            try {
                coroutineScope {
                    launch {
                        throw SomeException()
                    }
                }  
            } catch (e: SomeException) {
                println("①")
            }
        }  
    } catch (e: SomeException) {
        println("②")
    }
}
```

#### 答え

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Tpd08ECa6)
```
①
```

#### 解説

フローチャートには含まれないパターンです。
内部の`coroutineScope`から外部`coroutineScope`への、コルーチン (`Job`) の親子構造を通じた例外の伝播は行われません。そのため、内部の`coroutineScope`から投げられた例外を補足すれば、外部の`coroutineScope`へは例外が伝播しません。
仮に内部の`coroutineScope`から投げられた例外を補足しなければ、その側の`coroutineScope`から再び投げられることになります。
