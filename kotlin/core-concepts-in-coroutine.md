# Kotlin Coroutinesの核心：Builder・Scope・Job・Contextの関係を解説

## 本記事から何が得られるのか

Kotlin Coroutinesを利用する上で、「Coroutine Builder」「CoroutineScope」「Job」「CoroutineContext」といった概念は必ず目にするものです。
一方で、それらのつながりや役割について、しっくりくるようなメンタルモデルを作ることに苦労されている方も多いのではないでしょうか。かくいう私がそうでした。

そこで本記事では、Kotlin Coroutinesの基本文法には慣れているものの、その詳細には自信がない方を対象に、「Coroutine Builder」「CoroutineScope」「Job」「CoroutineContext」の関係を紐解きます。
私自身がソースコードを読み解き、また実際にコードを書いて検証した結果をもとにまとめています。

## 🌱 CoroutineScope ー Coroutineの起点

Kotlin Coroutinesを理解することを難しくしている要因の一つとして、「Coroutine Builder」「CoroutineScope」「Job」「CoroutineContext」などの概念が相互に関連し合っていることが挙げられます。ある概念を理解するには、別の概念の理解が前提となることが多く、これが学習の障壁となりがちです。

本記事では、まず「**CoroutineScope**」に着目します。

CoroutineScopeは、端的には**Coroutineの起点であり、かつCoroutineを起動するための情報 (CoroutineContext) を保持するもの**と言えます。
実際に、CoroutineScopeのソースコード ^[CoroutineScopeのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineScope.kt#L76] を見てみると、以下のようにCoroutineContextのみを持つinterfaceです。

```kt
public interface CoroutineScope {
    /**
     * The context of this scope.
     * Context is encapsulated by the scope and used for implementation of coroutine builders that are extensions on the scope.
     * Accessing this property in general code is not recommended for any purposes except accessing the [Job] instance for advanced usages.
     *
     * By convention, should contain an instance of a [job][Job] to enforce structured concurrency.
     */
    public val coroutineContext: CoroutineContext
}
```

**Coroutineは、必ずCoroutineScopeを通じて起動する必要があります**。このとき、CoroutineScopeからCoroutineContextをもとにCoroutineを起動するための関数 (`launch`や`async`など) を「**Coroutine Builder**」と呼びます。
**CoroutineScopeは、そこから起動されたCoroutineの動作を規定するための情報の集まりである「CoroutineContext」を保持します**。

これらの関係を模式図に表すと、以下のようになります。ただし、この時点では未説明のコンセプトが多く含まれるので、現時点で理解する必要はありません。この記事を通して最終的には理解できるよう、説明していきます。

![](https://storage.googleapis.com/zenn-user-upload/88035cc52afb-20250606.png)
*Coroutine Builder・Coroutine Scope・CoroutineContext・Jobの関係*

上記の模式図の状況を、Kotlinコードで表すと、以下のようになります ([Kotlin Playground](https://pl.kotl.in/vgN4Rj856))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope { // `this` is CoroutineScope A
        launch {
            println("Coroutine A-1 launched")
        }
        launch {
            println("Coroutine A-2 launched")
        }
    }
}
```

出力の例:
```sh
Coroutine A-1 launched
Coroutine A-2 launched
```

なお、このコードでは、Coroutine Aおよび Coroutine Bの両方が完了するまで、`coroutineScope { ... }`自体も完了しません。ただ、この`coroutineScope`関数の挙動を理解するには、その背後に存在する「Job」に関する理解も不可欠です。

## 📦 CoroutineContext ー Coroutineの制御に必要な情報を保持

CoroutineContextは、ざっくり言えば、**Coroutineの動作を規定する情報を保持するもの**です。イメージとしては、ユニークなキー (`CoroutineContext.Key`) に対して値(`CoroutineContext.Element`) を保持するハッシュテーブルのような構造をしています。

CoroutineContextが管理する情報の中でも、特に重要なのが「**CoroutineDispatcher**」と「**Job**」の2つです。

## 🎛️ CoroutineDispatcher ー Coroutineの実行スレッドを指定

`CoroutineDispatcher`とは、**Coroutineの実行に使用するスレッド （またはスレッドプール） を指定するためのもの**です。  
Kotlin Coroutinesでは、以下のような定義済みのディスパッチャが提供されており、そのいずれかを指定することが一般的です。

- **`Dispatchers.Default`：**
CPUバウンドな処理向けのスレッドプールで実行されます。
- **`Dispatchers.IO`：**
I/Oバウンドな処理向けのスレッドプールで実行されます。`Dispatchers.IO`はJVMプラットフォームにのみ存在します。
- **`Dispatchers.Main`：**
単一のUIスレッドでの実行されます。Androidなどの特定のプラットフォームにのみ存在します。

例えば、**CoroutineBuilder (launchやasync) にCoroutineDispatcherを渡すことで、起動されたCoroutineが実行されるスレッド/スレッドプールを指定することが可能**です。**launch関数でCoroutineDispatcherを指定しない場合、親CoroutineScopeのDispatcherが引き継がれます**。
なお、親のCoroutineScopeでCoroutineDispatcherが指定されていない場合には、デフォルト値である`Dispatchers.Default`が使用されます。

以下のサンプルコードで、この挙動を確認します ([Kotlin Playground](https://pl.kotl.in/GCQEhwTt_))。CoroutineDispatcherはCoroutineContextに含まれており、ハッシュテーブルのように`this.coroutineContext[CoroutineDispatcher]`で値を参照することが可能です。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        launch {
            // デフォルトではDispatchers.Defaultが使われる（親が明示していない場合）
            assert(this.coroutineContext[CoroutineDispatcher] == Dispatchers.Default)
            println("Coroutine A launched")
        }

        launch(Dispatchers.IO) {
            // 明示的に指定されたDispatchers.IOが使われる
            assert(this.coroutineContext[CoroutineDispatcher] == Dispatchers.IO)
            println("Coroutine B launched")
            
            launch {
                // 子Coroutineは親のDispatcher（この場合はDispatchers.IO）を継承する。
                assert(this.coroutineContext[CoroutineDispatcher] == Dispatchers.IO)
                println("Coroutine C launched")
            }
        }
    }
}
```

## 🧑‍🏭 Job ー ライフサイクル管理とStructured Concurrencyの実現

Kotlin CoroutineにおけるJobは、2つの重要な役割を持ちます。

1. Coroutineのライフサイクルを管理する。
2. Coroutineの親子構造を管理する。すなわち、Structured Concurrency (構造化された並列性) を実現する。

### Jobの1つ目の役割: Coroutineのライフサイクル管理

JobはCoroutineのライフサイクルを管理します。Jobは、以下のような状態を遷移します。

- **New (オプショナルな初期状態):**
Coroutine Builderの`start`パラメータに`CoroutineStart.LAZY`を渡すと、JobはNew状態で作られます。New状態のJob (すなわち未起動のCoroutine) は、startやjoinが呼ばれることでActive状態となります。
- **Active (デフォルトの初期状態):**
通常、Coroutineは作成と同時に実行開始され、JobがデフォルトでActiveな状態となります。Coroutineが実行されている間、JobはActive状態のままです。
- **Completing (過渡的な状態):**
Coroutineの本体 (関数のbody) が終了すると、JobはCompleting状態に入り、未完了の子Coroutineがある場合はその完了を待ちます。子のCoroutineが全て完了したら、最終的に**Completed状態**に移行します。
- **Completed (最終状態):**
Coroutineが正常に完了したことを示す状態です。
- **Cancelling (過渡的な状態):**
Coroutineが実行中に失敗すると、JobがActive状態からCancelling状態になります。子のCoroutineが全て完了またはキャンセルされるのを待ったのち、Jobは**Cancelled状態**へ移行します。
- **Cancelled (最終状態):**
Coroutineが明示的にキャンセルされた、あるいは未処理の例外によって失敗し、キャンセルされたことを示す状態です。

以下に、状態遷移図を示します。

![](https://storage.googleapis.com/zenn-user-upload/5b92e3dc2957-20250605.png)
*Jobの状態遷移図*

実際に、以下のようなコードでJobのライフサイクルの遷移を確認することが可能です ([Kotlin Playground](https://pl.kotl.in/aD6Gzl7aG))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope { // `this` is CoroutineScope.
        // Create custom parent job to complete manually.
        val parentJob = Job(this.coroutineContext.job)
        val job = this.launch(parentJob, start = CoroutineStart.LAZY) {
            // Launches child coroutine which completes after 1000ms.
            this.launch {
                delay(2000)
            }
        }
        println(job) // The `job` is `New`.
        job.start()
        println(job) // The `job` is `Active`.
        parentJob.complete()
        delay(100) // Wait until completion is notified to the child job.
        println(job) // The `job` is `Completing`.
        job.join()
        println(job) // The `job` is `Completed`.
    }

    coroutineScope {
        val job = this.launch {
            // Launches child coroutine which completes after 1000ms.
            this.launch {
                delay(1000)
            }
        }
        println(job) // The `job` is `Active`.
        job.cancel()
        println(job) // The `job` is `Cancelling` since the child job is not completed/cancelled yet.
        job.join()
        println(job) // The `job` is `Cancelled`.
    }
}
```

出力の例:
```sh
"coroutine#1":LazyStandaloneCoroutine{New}@1517365b
"coroutine#1":LazyStandaloneCoroutine{Active}@1517365b
"coroutine#1":LazyStandaloneCoroutine{Completing}@1517365b
"coroutine#1":LazyStandaloneCoroutine{Completed}@1517365b
"coroutine#3":StandaloneCoroutine{Completing}@5530065a
"coroutine#3":StandaloneCoroutine{Cancelling}@5530065a
"coroutine#3":StandaloneCoroutine{Cancelled}@5530065a
```

上のコード例で行っているように、Jobに対して`start()`や`cancel()`、`complete()` (ただし`complete()`は`CompletableJob`を継承したJobのみに対して呼び出し可能) を呼び出すことで、Jobの状態をマニュアルで操作することも可能です。つまり、**Jobは外側からCoroutineのライフサイクルを制御するためのインタフェースとしても機能**します。

なお、`Job.join()`は、そのJobが完了するまで待つ (Coroutineをsuspendする) ためのメソッドです。

### Jobの2つ目の役割: Strucured Concurrencyの実現

Jobは、Kotlin Coroutineの重要な特性であるStructured Concurrency (構造化された並列性) を実現するという役割も担っています。

Structured Concurrencyとは「**タスクをグループ化・階層化することで、キャンセルやエラーハンドリングを安全かつ簡潔に実装可能とする手法**」です。Kotlin Coroutinesに特有のものではなく、Swift Concurrency等の並行処理ライブラリでも採用されている 一般的なアプローチです^[Swift Evolution: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md]。

ここが重要な点ですが、**CoroutineScopeからCoroutine Builder (`launch`や`async`等) を用いてCoroutineを起動すると、起動元のCoroutineScopeのJobを親とする子Jobが作られ、起動後のCoroutineに紐づけられます**。この仕組みによって、Job間にツリー状の構造が形成されます。
実際に、以下のようなコードを実行すると、`launch`によってCoroutineを立ち上げた際に、Jobの親子関係が形成されることが分かります ([Kotlin Playground](https://pl.kotl.in/qqoIJ_7Hm))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val job1 = launch { // Launches Coroutine 1.
            val job1_1 = launch { // Launches Coroutine 1-1.
                val job1_1_1 = launch { // Launches Coroutine 1-1-1.
                    delay(1000)
                }
            	println("job1_1_1.parent (${job1_1_1.parent}) -> job1_1_1 (${job1_1_1})")
                
                val job1_1_2 = launch { // Launches Coroutine 1-1-2.
                    delay(1000)
                }
                println("job1_1_2.parent (${job1_1_2.parent}) -> job1_1_2 (${job1_1_2})")
            }
            println("job1_1.parent (${job1_1.parent}) -> job1_1 (${job1_1})")
        }
        println("job1.parent (${job1.parent}) -> job1 (${job1})")
    }
}
```

出力の例:
```sh
job1.parent (ScopeCoroutine{Active}@3930015a) -> job1 ("coroutine#1":StandaloneCoroutine{Active}@71e7a66b)
job1_1.parent ("coroutine#1":StandaloneCoroutine{Active}@71e7a66b) -> job1_1 ("coroutine#2":StandaloneCoroutine{Active}@140f6bbb)
job1_1_1.parent ("coroutine#2":StandaloneCoroutine{Active}@140f6bbb) -> job1_1_1 ("coroutine#3":StandaloneCoroutine{Active}@55e947e2)
job1_1_2.parent ("coroutine#2":StandaloneCoroutine{Active}@140f6bbb) -> job1_1_2 ("coroutine#4":StandaloneCoroutine{Active}@14d44dd9)
```

Kotlin CoroutinesにおいてStructured Concurrencyが採用されていることには、以下のような複数のメリットがあります。

1. 子Coroutineが完了するまで親Coroutineが待機できる。
2. Coroutineのキャンセルが親から子へと自動的に伝播される。
3. 例外による失敗が親子間で自動的に伝播される。

#### メリット1. 子Coroutineが完了するまで親Coroutineが待機

Jobの親子構造が存在する場合、子のJobが完了 (Completed) となるまでは、親のJobも完了しません。
例えば、以下のコードの場合、`launch`が完了するまで、親のスコープ `coroutineScope` を抜けることはありません。これは`coroutineScope`のJobと、`launch`のJobに親子関係が形成され、`launch`のJobが完了するまでは`coroutineScope`のJobも完了とならないためです ([Kotlin Playground](https://pl.kotl.in/cJ1pY7e5_))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val parentScope = this
        launch {
            delay(1000)
            println("Child Coroutine completing...\n- Child scope: ${this},\n- Parent scope: ${parentScope}")
        }
        println("Parent Coroutine completing...\n- Parent scope: ${parentScope}")
    }
    println("Parent scope completed.")
}
```

出力の例:
```sh
Parent Coroutine completing...
- Parent scope: ScopeCoroutine{Active}@629f0666
Child Coroutine completing...
- Child scope: "coroutine#1":StandaloneCoroutine{Active}@2a0e551a,
- Parent scope: ScopeCoroutine{Completing}@629f0666
Parent scope completed.
```

実際のコードでこのような書き方をすることは稀ですが、`launch`に明示的に独立した`Job()`を渡すと、親のCoroutineScopeのJobとの親子関係は形成されません。
このため、子Coroutineの完了を親Coroutineは待たなくなり、coroutineScopeは子Coroutineの実行中であってもすぐに完了してしまいます。
以下のコードを実行すると、その挙動が確認できます ([Kotlin Playground](https://pl.kotl.in/LJrWvGDzk))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val parentScope = this
        launch(Job()) { // 親子関係を断ち切る
            delay(1000)
            println("Child Coroutine completing...\n- Child scope: ${this},\n- Parent scope: ${parentScope}")
        }
        println("Parent Coroutine completing...\n- Parent scope: ${parentScope}")
    }
    println("Parent scope completed.")
}
```

出力の例:
```sh
Parent Coroutine completing...
- Parent scope: ScopeCoroutine{Active}@282ba1e
Parent scope completed.
```

#### メリット2. Coroutineのキャンセルが親から子へと自動的に伝播

**Coroutineをキャンセルすると、その子のCoroutineも自動的にキャンセルされます**。これもJobの親子構造によって実現されています。

![](https://storage.googleapis.com/zenn-user-upload/a28f860ea1f4-20250606.png =400x)
*Jobのキャンセルが伝播される仕組み*

例えば、以下のコードでは、Job Bをキャンセルすることで、Job Bの子であるJob Dもキャンセルされます。一方、Job Bをキャンセルしても、兄弟であるJob Cや親であるJob Aはキャンセルされません。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val jobA = launch { // Launches Coroutine A.
            val jobB = launch { // Launches Coroutine B.
                val jobD = launch { // Launches Coroutine D.
                    for (i in 0..10) {
                        println("job D still active, i=${i}")
                        delay(50)
                    }
                }
            }
            val jobC = launch { // Launches Coroutine C.
                for (i in 0..10) {
                    println("job C still active, i=${i}")
                    delay(50)
                }
            }
            delay(200)
            jobB.cancel()
            println("job B cancelling, ${jobB}")
            jobB.join()
            println("job B cancelled, ${jobB}")
        }
    }
}
```

出力の例:
```sh
job D still active, i=0
job C still active, i=0
job D still active, i=1
job C still active, i=1
job D still active, i=2
job C still active, i=2
job D still active, i=3
job C still active, i=3
job B cancelling, "coroutine#2":StandaloneCoroutine{Cancelling}@1f192039
job C still active, i=4
job B cancelled, "coroutine#2":StandaloneCoroutine{Cancelled}@1f192039
job C still active, i=5
job C still active, i=6
job C still active, i=7
job C still active, i=8
job C still active, i=9
job C still active, i=10
```

#### メリット3. 例外による失敗が親子間で自動的に伝播

**Coroutine内で例外が発生すると、その例外は親Coroutine (スコープ) に伝播し、さらに親Coroutineを経由して他の子Coroutineも終了します**。つまり、キャンセルは親から子への一方向の伝播ですが、例外発生時には親子構造で繋がったCoroutine全体が強制終了されます。

以下のコードでは、Coroutine Bで例外が発生すると、子のCoroutine Dだけでなく、親のCoroutine Aと、さらにその子のCoroutine Cも終了することが分かります ([Kotlin Playground](https://pl.kotl.in/LQUUrdTqn))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val jobA = launch { // Launches Coroutine A.
            val jobB = launch { // Launches Coroutine B.
                val jobD = launch { // Launches Coroutine D.
                    for (i in 0..10) {
                        delay(50)
                        println("Job D is still active: i=${i}")
                	}
                }
                delay(200)
                throw Exception("Exception in jobB.")
            }
            val jobC = launch { // Launches Coroutine C.
                for (i in 0..10) {
                    delay(50)
                    println("Job C is still active: i=${i}")
                }
            }
        }
        jobA.join()
        println("job A is still active") // Job Aもキャンセル済のため、これは出力されない。
    }
}
```

出力の例:
```sh
Job D is still active: i=0
Job C is still active: i=0
Job D is still active: i=1
Job C is still active: i=1
Job D is still active: i=2
Job C is still active: i=2
Exception in thread "main" java.lang.Exception: Exception in jobB.
 at FileKt$main$2$jobA$1$jobB$1.invokeSuspend (File.kt:14) 
 at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith (ContinuationImpl.kt:33) 
 at kotlinx.coroutines.DispatchedTask.run (DispatchedTask.kt:108) 
```

ただし、SupervisorJobを利用することで、ある子Coroutineの失敗が他の子に影響しないように制御できます ([Kotlin Playground](https://pl.kotl.in/Y45ePF_l4))。SupervisorJobも含めた例外処理の詳細は、別記事で解説予定です。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val jobA = launch { // Launches Coroutine A.
            // supervisorScope (SupervisorJob) により、Job AやJob Cに例外が伝播しない。
            supervisorScope {
                val jobB = launch { // Launches Coroutine B.
                    val jobD = launch { // Launches Coroutine D.
                        for (i in 0..10) {
                            delay(50)
                            println("Job D is still active: i=${i}")
                        }
                    }
                    delay(200)
                    throw Exception("Exception in jobB.")
                }
                val jobC = launch { // Launches Coroutine C.
                    for (i in 0..10) {
                        delay(50)
                        println("Job C is still active: i=${i}")
                    }
                }
            }
        }
        jobA.join()
        println("job A is still active")
    }
}
```

出力の例:
```sh
Job D is still active: i=0
Job C is still active: i=0
Job D is still active: i=1
Job C is still active: i=1
Job D is still active: i=2
Job C is still active: i=2
Job C is still active: i=3
Job C is still active: i=4
Exception in thread "DefaultDispatcher-worker-1 @coroutine#3" java.lang.Exception: Exception in jobB.
	at FileKt$main$2$jobA$1$1$jobB$1.invokeSuspend(File.kt:15)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:108)
	at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:584)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:793)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:697)
	at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:684)
	Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [CoroutineId(2), "coroutine#2":StandaloneCoroutine{Cancelling}@52130ac1, Dispatchers.Default]
Job C is still active: i=5
Job C is still active: i=6
Job C is still active: i=7
Job C is still active: i=8
Job C is still active: i=9
Job C is still active: i=10
job A is still active
```

## 🚀 Coroutine Builder ー Coroutineを起動する関数

CoroutineContextやJobについて解説できたところで、これまでのサンプルコードには度々登場してきたCoroutine Builderの説明に移ります。

Coroutine Builderとは、**CoroutineScopeを起点として、CoroutineContextをもとにCoroutineを起動する関数**です。
以下では、代表的なCoroutine Builderである、`launch`と`async`の2つについて解説します。**これらはどちらもCoroutineScopeの拡張関数として定義されており、CoroutineScopeに対して呼び出す必要があります**。この制約によって、単独でCoroutineを起動することが基本的にはできなくなる、すなわちStructured Concurrencyに則ってCoroutineが構造化されることとなります。

なお、例外として、`runBlocking`というCoroutine Builderを用いることで、CoroutineScopeを用いずにCoroutineを起動することができます。しかし、実際のコードで`runBlocking`は基本的に使われないため、ここでは解説しません。

### CoroutineScope.launch

`launch`のソースコードは以下のようになっており、CoroutineScopeの拡張関数として定義されています^[CoroutineScope.launchのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L44]。

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

パラメータとして`context: CoroutineContext`を受け取ることができますが、それがそのままContextとして使われるわけではなく、`newCoroutineContext`関数によって**起動元の`CoroutineScope`が既に持つContextに対してマージされた上で、新たに作成されたCoroutine用の`CoroutineContext`が作られます**。また、起動元のCoroutineScopeとは別の、**新たなCoroutineScopeが作られ、それが新規Coroutine用の`CoroutineContext`を保持**します。

なお、パラメータ`start: CoroutineStart`については、Jobのセクションでも説明しましたが、Jobの初期状態をデフォルトの`Active` (Coroutine作成と同時に実行開始) ではなく`New` (作成後に未実行で待機) に変えるためのものです。`CoroutineStart.LAZY`を渡すことで`New`とすることが可能です。

ここで冒頭で示した模式図を再掲します。ここまで読み通してきた方であれば、Coroutine Builder・CoroutineScope・Coroutine Context・Jobの関係性が腑に落ちているのではないでしょうか。

![](https://storage.googleapis.com/zenn-user-upload/ae16ea64f827-20250606.png)
*Coroutine Builder・CoroutineScope・CoroutineContext・Jobの関係*

#### launch時のCoroutineContextの継承に関する注意点

CoroutineContextの継承に関して、少し補足します。
CoroutineContextに含まれる要素のうち、Jobだけやや挙動が異なるという注意点があります。

- **launchにパラメータを渡さない場合:**
  - Job: 起動元のCoroutineScopeを親とするJobが作られる。
  - Job以外 (CoroutineDispatcher): 起動元のCoroutineScopeと同じものが使われる。
- **launchにパラメータを渡した場合:**
  - Job: 渡されたJobを親とするJobが作られる。
  - Job以外 (CoroutineDispatcher): 渡されたパラメータがそのまま使われる。例えばDispatchers.IOが渡された場合、Dispatchers.IOが使われる。

つまり、**Jobに関しては、パラメータの有無に関わらず新しいJobが作られます**。
実際に、以下のようなソースコードで、この挙動を確かめることが可能です ([Kotlin Playground](https://pl.kotl.in/wFFKLD-Cw))。なお、launchの返り値として起動されたCoroutineに紐づくJobが返されることも確認しています。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        launch {
            val jobA = this.coroutineContext.job
            val dispatcherA = this.coroutineContext[CoroutineDispatcher]
            println("CoroutineScope A - job: ${jobA}, dispatcher: ${dispatcherA}")
            
            val jobB = launch(Dispatchers.IO) {
                val jobB = this.coroutineContext.job
                val dispatcherB = this.coroutineContext[CoroutineDispatcher]
                println("CoroutineScope B - job: ${jobB}, parentJob: ${jobB.parent}, dispatcher: ${dispatcherB}")
            }
            
            val jobC = launch(jobA) {
                val jobC = this.coroutineContext.job
                val dispatcherC = this.coroutineContext[CoroutineDispatcher]
                println("CoroutineScope C - job: ${jobC}, parentJob: ${jobC.parent}, dispatcher: ${dispatcherC}")
            }
            
            delay(100)
            println("JobB: ${jobB}")
            println("JobC: ${jobC}")
        }
    }
}
```

```sh
CoroutineScope A - job: "coroutine#1":StandaloneCoroutine{Active}@36ecf294, dispatcher: Dispatchers.Default
CoroutineScope B - job: "coroutine#2":StandaloneCoroutine{Active}@7726f986, parentJob: "coroutine#1":StandaloneCoroutine{Active}@36ecf294, dispatcher: Dispatchers.IO
CoroutineScope C - job: "coroutine#3":StandaloneCoroutine{Active}@3854f842, parentJob: "coroutine#1":StandaloneCoroutine{Active}@36ecf294, dispatcher: Dispatchers.Default
JobB: "coroutine#2":StandaloneCoroutine{Completed}@7726f986
JobC: "coroutine#3":StandaloneCoroutine{Completed}@3854f842
```

### CoroutineScope.async

`async`も、`launch`と同様にCoroutineを起動するのに使用され、CoroutineContextを継承する点も同様です。以下にソースコードを引用しますが、`launch`と非常に類似していることが分かります ^[CoroutineScope.asyncのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L79]。

```kt
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

一番の違いは、**`async`はJobではなく`Deferred<T>`を返す**点にあります。
`Defered<T>`は、他のプログラミング言語でいうところの`Future<T>`や`Promise<T>`のようなものです。つまり、`async`を使うことで、そのCoroutineの実行結果を返すことができます。
なお、DeferredはJobを継承しているため、通常のJobに存在するメソッド (`cancel()`等) を呼び出すことも可能です。

Deferredに対して`await()`を呼びことで、結果が返されるまで、スレッドをブロックせずに待機することができます ([Kotlin Playground](https://pl.kotl.in/ED4JLNWGn))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val result = async {
            delay(100)
            123
        }
        println(result.await())
    }
}
```

出力の例:

```sh
123
```

なお、Coroutineがキャンセルされた場合には、`await()`時にCancellationExceptionが投げられます ([Kotlin Playground](https://pl.kotl.in/sW1bCgH3C))。

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope {
        val cancelled = async {
            delay(100)
            123
        }
        cancelled.cancel()
        try {
            cancelled.await()
        } catch (e: Throwable) {
            println(e)
        }
    }
}
```

出力の例:

```sh
kotlinx.coroutines.JobCancellationException: DeferredCoroutine was cancelled; job="coroutine#1":DeferredCoroutine{Cancelled}@ebc26be
```

## 🌳 CoroutineScopeへの再訪

CoroutineContext・Job・Coroutine Builderの解説を踏まえた上で、CoroutineScopeに再訪し、補足説明を加えます。

### `coroutineScope`関数

ここまでのサンプルコードでは、`coroutineScope`関数を使ってきました。
`coroutineScope`は、Coroutine BuilderのようにCoroutineを起動することなく、新たにCoroutineScopeを作成するための関数です。
launchやasyncと同様に、CoroutineContextは親のCoroutineScopeから引き継がれます。

以下に、coroutineScopeを使用したサンプルコードを示します ([Kotlin Playground](https://pl.kotl.in/HwLFFXYTe))。`coroutineScope`のブロック内のsuspend関数が全て完了すると、そのブロック自体も完了することが分かります。これはJobの親子構造、すなわちStructured Concurrencyによって実現されています。

```kt
import kotlinx.coroutines.*

suspend fun someSuspendFunction() {
    coroutineScope {
        val job = this.coroutineContext.job
        val dispatcher = this.coroutineContext[CoroutineDispatcher]
        println("Child CoroutineScope in someSuspendFunction - job: ${job}, parentJob: ${job.parent}, dispatcher: ${dispatcher}")
    }
}

suspend fun main() {
    coroutineScope {
        val jobA = this.coroutineContext.job
        val dispatcherA = this.coroutineContext[CoroutineDispatcher]
        println("CoroutineScope A - job: ${jobA}, dispatcher: ${dispatcherA}")
        
        coroutineScope {
            val job = this.coroutineContext.job
            val dispatcher = this.coroutineContext[CoroutineDispatcher]
            println("Child CoroutineScope in main - job: ${job}, parentJob: ${job.parent}, dispatcher: ${dispatcher}")
        }
        
        someSuspendFunction()
    }

    println("coroutineScope finished.")
}
```

出力の例:
```sh
CoroutineScope A - job: ScopeCoroutine{Active}@3d012ddd, dispatcher: null
Child CoroutineScope in main - job: ScopeCoroutine{Active}@762efe5d, parentJob: ScopeCoroutine{Active}@3d012ddd, dispatcher: null
Child CoroutineScope in someSuspendFunction - job: ScopeCoroutine{Active}@123772c4, parentJob: ScopeCoroutine{Active}@3d012ddd, dispatcher: null
```

### `withContext`関数

`withContext`は`coroutineScope`と同様に、新たにCoroutineScopeを作成します。ただし、`coroutineScope`と異なり、CoroutineContextを更新することが可能です。Coroutine Builder (launchやasync) と同様に、Jobを渡した場合には、そのJobを親とする新たなJobが作成されます。
suspend関数内でDispatch先のスレッドプールを切り替えたい時、例えばIOバウンドな処理をDispatchers.IOに移したい場合などに、頻繁に使用されるメソッドです。

`withContext`の挙動は、以下のようなサンプルコードで確認できます ([Kotlin Playground](https://pl.kotl.in/SRaELtS8c))。

```kt
import kotlinx.coroutines.*

suspend fun someSuspendFunction() {
    withContext(Dispatchers.IO) {
        val job = this.coroutineContext.job
        val dispatcher = this.coroutineContext[CoroutineDispatcher]
        println("Child CoroutineScope in someSuspendFunction - job: ${job}, parentJob: ${job.parent}, dispatcher: ${dispatcher}")
    }
}

suspend fun main() {
    withContext(Dispatchers.Default) {
        val jobA = this.coroutineContext.job
        val dispatcherA = this.coroutineContext[CoroutineDispatcher]
        println("CoroutineScope A - job: ${jobA}, dispatcher: ${dispatcherA}")
        
        withContext(Dispatchers.IO + jobA) {
            val job = this.coroutineContext.job
            val dispatcher = this.coroutineContext[CoroutineDispatcher]
            println("Child CoroutineScope in main - job: ${job}, parentJob: ${job.parent}, dispatcher: ${dispatcher}")
        }
        
        someSuspendFunction()
    }
}
```

出力の例:
```sh
CoroutineScope A - job: DispatchedCoroutine{Active}@756b50c6, dispatcher: Dispatchers.Default
Child CoroutineScope in main - job: DispatchedCoroutine{Active}@1ef2554c, parentJob: DispatchedCoroutine{Active}@756b50c6, dispatcher: Dispatchers.IO
Child CoroutineScope in someSuspendFunction - job: DispatchedCoroutine{Active}@43ca4ad6, parentJob: DispatchedCoroutine{Active}@756b50c6, dispatcher: Dispatchers.IO
```

## 🗺️ 全体像の整理

最後に、Coroutine Builder・CoroutineScope・CoroutineContext・Jobの目的・関係を整理します。**これら4つのコア要素が相互に関係しあうことで、Kotlin Coroutinesは「Structured Concurrency」という並行処理モデルを実現**しています。

- **CoroutineScope:**
Coroutineの起点であり、かつCoroutineを起動するための情報 (CoroutineContext) を保持するもの。
- **CoroutineContext:**
Coroutineの動作を規定する情報を保持するもの。
- **Job:**
CoroutineContextの一要素であり、「Coroutineのライフサイクル管理」と「Structured Concurrency」を実現するもの。
- **CoroutineBuilder:**
CoroutineScopeを起点として、CoroutineContextをもとにCoroutineを起動する関数。

![](https://storage.googleapis.com/zenn-user-upload/88035cc52afb-20250606.png)
*Coroutine Builder・CoroutineScope・CoroutineContext・Jobの関係*

## ✏️ 本記事のCHANGELOG

- 2025/6/6: 初版執筆
