# 内部実装から理解するKotlin Coroutines：suspend関数・Continuation編

本記事は、Kotlin Coroutinesの基礎をすでに理解している中級者以上の開発者を対象とし、さらに理解を深めることを目的としたシリーズの一部です。

今回は、**suspend関数**と、それを支える概念である**Continuation**について解説します。
suspend関数は、Kotlin開発者であればほとんどの方が利用したことがあるでしょう。しかし、その仕組みがどのように実現されているかを正しく説明できる方は多くないのではないでしょうか。

本記事を通じて、**読者がsuspend関数の内部実装をイメージできるようになり、より効率的に実装・デバッグ・レビューを行えるようになること**を目指しています。

:::message
本記事は入門者向けではありません。**Kotlin Coroutinesの基礎は理解しているが、その内部実装についてはブラックボックス化している**という読者を主な対象としています。
Kotlin Coroutinesの入門記事をお探しの場合は、まずは公式ドキュメント^[Coroutines: https://kotlinlang.org/docs/coroutines-overview.html]などで基礎的な仕様を学ぶことをおすすめします。
:::

## suspend関数とは何か

suspend関数は、**Coroutineあるいは他のsuspend関数から呼び出すことができ、中断・再開可能な関数**です。

以下にsuspend関数の一例を示します。

▶️ [Kotlin Playgroundで実行](https://pl.kotl.in/u2MTumVid)

```kt
import kotlinx.coroutines.*

suspend fun helloWorld() {
    println("Hello")
    delay(1000)
    println("World!")
}
```

出力:
```sh
Hello
World!
```

これは、"Hello"と出力し、1秒後に"World!"と出力するsuspend関数です。

`delay(1000)`は1秒間待機する処理ですが、似た挙動を示す`Thread.sleep(1000)`とは異なり、**実行スレッドをブロックしません**。**中断されている間は、実行スレッドを別の処理に明け渡す**ことができます。また、**再開時には、中断前のスレッドで再開されるとは限りません**。

## suspend関数の中断・再開を可能とする仕組み

では、先述したsuspend関数の仕組みがどのように実現されているのかを見ていきましょう。

suspend関数を中断・再開可能にしているのは、**Continuation**^[Continuation - Wikipedia: https://en.wikipedia.org/wiki/Continuation]と呼ばれるインスタンスです。
Kotlinコンパイラは、**suspend関数をJava bytecodeへとコンパイルする際に、Continuationを用いた関数へと変換**します。

**冒頭に示した以下のコードを`suspend`を使わない形に変えていくこと**によって、Continuationとは何かを解説していきます。

```kt
import kotlinx.coroutines.*

suspend fun helloWorld() {
    println("Hello")
    delay(1000)
    println("World!")
}
```

手始めに、`Continuation`を使わず、多くの方々に馴染み深いであろうCallback関数を使って、上記のサンプルコードを書き換えてみます。

▶️ [Kotlin Playgroundで実行](https://pl.kotl.in/lEVrDomkP)

```kt
import java.util.Timer
import java.util.concurrent.CountDownLatch
import kotlin.concurrent.schedule

fun helloWorld(onCompleted: () -> Unit) {
    println("Hello")
    delay(millis = 1000) {
        println("World!")
        onCompleted()
    }
}

private fun delay(millis: Long, onCompleted: () -> Unit) {
    Timer().schedule(delay = millis) { onCompleted() }
}

fun main() {
    val latch = CountDownLatch(1)
    helloWorld() {
        latch.countDown()
    }
    latch.await() // `latch.countDown()`が呼ばれるまでブロックする。
}
```

出力:
```sh
Hello
World!
```

Callback関数である`onCompleted`を、非同期処理を行う下位の関数に渡すことで、呼び出し元での処理の中断・再開が可能となっています。この場合でも、suspend関数を使ったコードと同様の出力が得られています。
なお、上記のサンプルコードでは便宜上、"World!"が出力される前に`main`関数が終了してしまわないよう、`CountDownLatch`を使って待機しています。

`Continuation`を使った方式も「**下位の関数にオブジェクトを渡し、それに対する呼び出しを元に処理を再開する**」という点において、Callback関数を使う方式と本質的には類似しています。

以下に、Continuationを用いた実装を示します。
ここでは、**Callback関数の代わりに、`Continuation`のインスタンスを下位の関数に渡しています**。
なお、このサンプルコードは、説明を分かりやすくするために極限まで簡略化した、いわば擬似コードです。実際のKotlinコンパイラが生成するコードはもっと複雑であることをご留意ください。

▶️ [Kotlin Playgroundで実行](https://pl.kotl.in/M-3eyWp13)

```kt
import java.util.Timer
import java.util.concurrent.CountDownLatch
import kotlin.concurrent.schedule
import kotlin.coroutines.Continuation
import kotlin.coroutines.CoroutineContext
import kotlin.coroutines.EmptyCoroutineContext

const val COROUTINE_SUSPENDED = -1

fun helloWorld(continuation: Continuation<Unit>): Any {
    return HelloWorldContinuation(continuation).resumeWith(Result.success(Unit))
}

private class HelloWorldContinuation(
    private val completion: Continuation<Unit>
) : Continuation<Unit> {
    private var label = 0

    override val context: CoroutineContext
        get() = completion.context

    override fun resumeWith(result: Result<Unit>) {
        when (label) {
            0 -> {
                println("Hello")
                label = 1
                delay(millis = 1000, continuation = this)
            }
            1 -> {
                println("World!")
                completion.resumeWith(Result.success(Unit))
            }
            else -> {
                throw IllegalStateException()
            }
        }
    }
}

private fun delay(millis: Long, continuation: Continuation<Unit>): Any {
    Timer().schedule(delay = millis) {
        continuation.resumeWith(Result.success(Unit))
    }
    return COROUTINE_SUSPENDED
}

fun main() {
    val latch = CountDownLatch(1)
    helloWorld(object : Continuation<Unit> {
        override val context: CoroutineContext
            get() = EmptyCoroutineContext

        override fun resumeWith(result: Result<Unit>) {
            latch.countDown()
        }
    })
    latch.await() // `latch.countDown()`が呼ばれるまでブロックする。
}
```

出力:
```sh
Hello
World!
```

元のsuspend関数、あるいはCallback関数を使ったケースと比べると、同様の出力は得られているものの、複雑なコードになっています。次のセクションで、このコードが実際に何をしているのか、そしてContinuationとは何であるのかを解説していきます。

### Continuationを利用した中断・再開の流れ

上記のContinuationを使ったコードが何をおこなっているのか、順を追って解説していきましょう。

#### 1. 関数が呼び出される

はじめに、`helloWorld`関数が呼び出されます。
元のsuspend関数と異なるのは、**引数に`Continuation`が足されている**ことです。
実際のKotlinコンパイラも、このようにsuspend関数の引数に`Continuation`を追加します。

```kt
fun helloWorld(continuation: Continuation<Unit>): Any {
    return HelloWorldContinuation(continuation).resumeWith(Result.success(Unit))
}
```

まずは、関数定義の`helloWorld(continuation: Continuation<Unit>)`の部分に着目します。

`Continuation`とは、端的に言えば「**suspend関数の状態マシンを保持するインスタンス**」と言えます。suspend関数は、中断・再開可能であるため、どこまで処理が進んだのか・次にどの処理に進むべきか、といった状態を管理する必要があります。「現在の状態」や「どのように状態遷移すべきか」を保持するのが、`Continuation`です。
Kotlinは、言語レベルで`Continuation`を定義しています^[Continuationのインタフェース: https://github.com/JetBrains/kotlin/blob/bc0801e5de83f80756816a6428d66a802e2f9f18/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L16]

ここで、`helloWorld`関数に渡されている`Continuation`は、**呼び出し元のCoroutineあるいはsuspend関数に紐づくContinuation**です。
`helloWorld`関数が中断あるいは完了したタイミングで、この`Continuation`に対してメソッドを呼び出すことで、呼び出し元のCoroutineあるいはsuspend関数を中断・完了することが可能となります。

#### 2. 関数に紐づく`Continuation`オブジェクトが作られる

続いて、bodyの`HelloWorldContinuation(continuation)`の部分に着目すると、`HelloWorldContinuation`というオブジェクトが作られています。

**suspend関数がbytecodeへとコンパイルされる際には、関数に紐づく`Continuation`クラスが定義されます**。今回のサンプルコードでは、このクラスを`HelloWorldContinuation`と名づけています。

```kt
private class HelloWorldContinuation(
    private val completion: Continuation<Unit>
) : Continuation<Unit> {
}
```

#### 3. 関数に紐づく`Continuation`インスタンスが起動される

続いて、作成された`HelloWorldContinuation`に対して`resumeWith`メソッドが呼ばれています。これは**suspend関数の実行を開始・再開**するためのメソッドです。

先ほども触れたように、`Continuation`とは、**suspend関数の状態マシンを保持するインスタンス**です。状態マシンを管理する上で要となっているのが、**`label`フィールド**と、**`resumeWith`メソッド内のswitch文**の2つです。

```kt
private class HelloWorldContinuation(
    private val completion: Continuation<Unit>
) : Continuation<Unit> {
    private var label = 0

    override fun resumeWith(result: Result<Unit>) {
        when (label) {
            0 -> {
                println("Hello")
                label = 1
                delay(millis = 1000, continuation = this)
            }
            1 -> {
                println("World!")
                completion.resumeWith(Result.success(Unit))
            }
            else -> {
                throw IllegalStateException()
            }
        }
    }
}
```

デフォルトでは、`label`は0となっています。そのため、初回の`resumeWith`メソッドの呼び出し時には、以下の範囲のコードのみが実行されます。つまり、**`delay`を呼び出すところまで実行した後、`helloWorld`メソッドは中断される**ことが分かります。

また、**label=0に紐づく範囲の処理が完了すると、`label`が1になる**ということも、処理の再開時のキーポイントとなるため、頭の片隅に置いておいてください。

`delay`が内部で何をしているのかは、次のステップで解説します。

```kt
    override fun resumeWith(result: Result<Unit>) {
        when (label) {
            0 -> {
                println("Hello")
                label = 1
                delay(millis = 1000, continuation = this)
            }
        }
    }
```

#### 4. 別のsuspend関数を呼び出す

`delay`もsuspend関数の一種であるため、`Continuation`を引数とする形で、bytecodeへとコンパイルされます。呼び出し元である、helloWorld関数に紐づく`Continuation` (`HelloWorldContinuation`) が、ここでは`Continuation`として渡されています。

今回のサンプルコードでは、説明のために簡略化した`delay`メソッドを定義しました。

```kt
private fun delay(millis: Long, continuation: Continuation<Unit>): Any {
    Timer().schedule(delay = millis) {
        continuation.resumeWith(Result.success(Unit))
    }
    return COROUTINE_SUSPENDED
}
```
:::message
実際の`delay`メソッドはもっと複雑です。
:::

`Timer().schedule`で、所定時間後に`Continuation`の`resumeWith`メソッドが呼ばれるよう、スケジューリングしています。
その後、即座に`COROUTINE_SUSPENDED`を返しています。つまり、3.のステップで`delay`を呼び出した際には、返り値として`COROUTINE_SUSPENDED`が返されることになります。

ここまで実行された後、一度`helloWorld`関数は中断されます。
その後、`delay`メソッド内で、`Continuation`の`resumeWith`メソッドが呼ばれることで、再び`helloWorld`関数の実行が再開されます。

#### 5. 関数に紐づく`Continuation`インスタンスが再開される

前ステップで、`delay`メソッド内から`Continuation`の`resumeWith`メソッドが呼ばれました。

着目すべき点として、**初回実行時とは異なり、今回は`label`が1に変わった状態で`resumeWith`が呼ばれます。**
そのため、`label=1`に紐づく以下の範囲のコードが実行されることとなります。

```kt
    override fun resumeWith(result: Result<Unit>) {
        when (label) {
            1 -> {
                println("World!")
                completion.resumeWith(Result.success(Unit))
            }
        }
    }
}
```

ここでは、"World!"を出力したのち、これは呼び出し元の`Continuation`である`completion`に対して、`resumeWith`を呼び出しています。
これによって、`helloWorld`メソッドの呼び出し元 (今回のサンプルコードではmain関数) に対して、再開を促すことができます。

---

以上の5つのステップで、`helloWorld`関数の実行が完了します。
この処理の流れを、模式図を用いて整理してみましょう。

![](https://storage.googleapis.com/zenn-user-upload/2be02294e507-20250616.png)
*`helloWorld`関数の実行の流れ*

この模式図を見ると、では大元のmain関数のContinuationはどのように作られるのか、という疑問が湧きます。これに関しては、後のセクションで解説します。

---

以上、suspend関数として定義されていた`helloWorld`が、Kotlinコンパイラによってどのような関数へと変わるのか、そしてContinuationを用いることによってどのように中断・再開処理が実現されているのかを説明しました。要点を整理します。

- Kotlinコンパイラによって、**suspend関数は、Continuationを引数に取る形へと変換**される。
- **suspend関数に紐づくContinutationが、Kotlinコンパイラによって生成**される。この**Continuationは、suspend関数の状態マシンを保持**する。
- **`Continuation`は、`label`フィールドによって、そのsuspend関数の処理がどこまで進んだかを管理**する。
- 下位の関数へとContinuationが渡され、その**Continuationの`resumeWith`メソッドが呼ばれることで、呼び出し元での処理が再開**される。

### suspend関数は実際にどのようなbytecodeへとコンパイルされるのか

前のセクションでは、擬似コードを使ってCSPの動作原理を解説しました。ただ、実際にKotlinコンパイラが生成するbytecodeは、もっと複雑なものです。

IntelliJ IDEAには、Kotlinコードから生成されるbytecodeをプレビュー表示し、さらにそれをJavaコードへとdecompileする機能があります。
そこで、Kotlinコンパイラによって、suspend関数が実際にはどのようなbytecodeへと変換されるのかも確認してみましょう

以下のKotlinコードをbytecodeに変換し、さらにdecompileしてJavaコード化します。

```kt
import kotlinx.coroutines.delay

suspend fun helloWorld() {
    println("Hello")
    delay(1000)
    println("World!")
}
```

すると、以下のようなJavaコードが得られます。

```java
public final class HelloWorldKt {
   @Nullable
   public static final Object helloWorld(@NotNull Continuation $completion) {
      Continuation $continuation;
      label20: {
         if ($completion instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)$completion;
            if (($continuation.label & Integer.MIN_VALUE) != 0) {
               $continuation.label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl($completion) {
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return HelloWorldKt.helloWorld((Continuation)this);
            }
         };
      }

      Object $result = $continuation.result;
      Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch ($continuation.label) {
         case 0:
            ResultKt.throwOnFailure($result);
            System.out.println("Hello");
            $continuation.label = 1;
            if (DelayKt.delay(1000L, $continuation) == var3) {
               return var3;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      System.out.println("World!");
      return Unit.INSTANCE;
   }
}
```

深く説明しませんが、「suspend関数の引数に`Continuation`が足されていること」「`label`フィールドで処理をswitchしていること」「`delay`メソッドに、`helloWorld`関数に紐づく`Continuation`を引数として渡されていること」など、以前に説明した要素が実際に含まれていることが分かります。

## `suspendCoroutine`の内部実装

Kotlin Coroutineに馴染みがある方であれば、Callbackベースの非同期処理をsuspend関数へと変換するために、`suspendCoroutine`あるいは`suspendCancellableCoroutine`を使えることはご存知でしょう。

例えば、以下のような、Callbackベースの非同期処理が存在するとします。

```kt
private fun <T> someAsyncFunc(onCompleted: (result: Result<T>) -> Unit) {
    // implementation goes here
}
```

`suspendCoroutine`を用いることで、以下のようにsuspend関数としてインタフェースを公開することができます。

```kt
private fun <T> someAsyncFunc(onCompleted: (result: Result<T>) -> Unit) {
    // implementation goes here
}

suspend fun <T> someAsyncFunc(): T = suspendCoroutine { cont ->
    someAsyncFunc<T> { result ->
        result.onSuccess { value ->
            cont.resume(value)
        }.onFailure { exception ->
            cont.resumeWithException(exception)
        }
    }
}
```

:::message
実際には、`suspendCoroutine`ではなく、Coroutineのキャンセル処理に対応した`suspendCancellableCoroutine`が利用されるケースが大半です。
:::

他の例として、`kotlinx.coroutines`内で定義されている`delay`関数も、内部的には`suspendCancellableCoroutine`を使っています。`delay`関数のソースコードを以下に示します^[`delay`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Delay.kt#L121]。

```kt
public suspend fun delay(timeMillis: Long) {
    if (timeMillis <= 0) return // don't delay
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
        // if timeMillis == Long.MAX_VALUE then just wait forever like awaitCancellation, don't schedule.
        if (timeMillis < Long.MAX_VALUE) {
            cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
        }
    }
}
```

このように、`suspendCoroutine`あるいは`suspendCancellableCoroutine`は、suspend関数に関わる重要なパーツです。
このセクションでは、suspend関数への理解を更に深めるべく、`suspendCoroutine`の内部実装を追っていきます。

---

`suspendCoroutine`のソースコードを以下に示します^[`suspendCoroutine`のソースコード: https://github.com/JetBrains/kotlin/blob/2d95ece640699335b06372d76eae1eb31a357c7b/libraries/stdlib/src/kotlin/coroutines/Continuation.kt#L142]。

```kt
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
}
```

内部では`suspendCoroutineUninterceptedOrReturn`を利用して、`Continuation`を取得していることが分かります。

この`suspendCoroutineUninterceptedOrReturn`のソースコードを見ると、` NotImplementedError`が投げられており、ソースコード上には実装が存在しません^[`suspendCoroutineUninterceptedOrReturn`のソースコード: https://github.com/JetBrains/kotlin/blob/2d95ece640699335b06372d76eae1eb31a357c7b/libraries/stdlib/src/kotlin/coroutines/intrinsics/Intrinsics.kt#L41]

```kt
public suspend inline fun <T> suspendCoroutineUninterceptedOrReturn(crossinline block: (Continuation<T>) -> Any?): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    throw NotImplementedError("Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic")
}
```

例外のメッセージには`Implementation of suspendCoroutineUninterceptedOrReturn is intrinsic`とあり、Kotlinコンパイラがビルド時に変換を加えるものと考えられます。

---

ソースコードからは`suspendCoroutine`の内部実装が判明しなかったため、アプローチを変えましょう。
代替案として、リバースエンジニアリング的に、`suspendCoroutine`を使った関数をbytecodeに変換し、そのbytecodeをdecompileしたコードを見てみることとします。

以下の`suspendCoroutine`を使ったサンプルコードで実験します。
コールバック関数を取る非同期処理 (ただし、簡略化のため実際は同期処理) をwrapして、suspend関数へと変換しているという想定です。

```kt
import kotlin.coroutines.resume
import kotlin.coroutines.suspendCoroutine

private fun someAsyncFunc(onCompleted: () -> Unit) {
    onCompleted()
}

suspend fun mySuspendFunc() {
    suspendCoroutine { cont ->
        someAsyncFunc { cont.resume(Unit) }
    }
}
```

IntelliJ IDEAの機能を利用して、上記のサンプルコードをbytecodeに変換し、更にそれをJavaコードへとdecompileしたところ、以下のようなコードが得られました。

```java
public final class MySuspendFuncKt {
   private static final void someAsyncFunc(Function0 onCompleted) {
      onCompleted.invoke();
   }

   @Nullable
   public static final Object mySuspendFunc(@NotNull Continuation $completion) {
      SafeContinuation var2 = new SafeContinuation(IntrinsicsKt.intercepted($completion));
      final Continuation cont = (Continuation)var2;
      int var4 = 0;
      someAsyncFunc(new Function0() {
         public final void invoke() {
            Result.Companion var10001 = Result.Companion;
            cont.resumeWith(Result.constructor-impl(Unit.INSTANCE));
         }

         public Object invoke() {
            this.invoke();
            return Unit.INSTANCE;
         }
      });
      Object var10000 = var2.getOrThrow();
      if (var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED()) {
         DebugProbesKt.probeCoroutineSuspended($completion);
      }

      return var10000 == IntrinsicsKt.getCOROUTINE_SUSPENDED() ? var10000 : Unit.INSTANCE;
   }
}
```

以前説明したように、suspend関数がコンパイルされると、引数に`Continuation`が追加されます。
今回の`mySuspendFunc`の引数にも、`Continuation`が追加されていることが分かります。
また、`suspendCoroutine`のソースコードと見比べても、`suspendCoroutineUninterceptedOrReturn`に特有のロジックは見当たりません。

これらのことから、`suspendCoroutineUninterceptedOrReturn`は、**本来はコンパイル時に追加される`Continuation`引数を、コンパイル前のソースコード上で参照できるようにしているもの**、と推察することができます。

### 大元のContinuationはどのように作られるのか

ここまでの説明で「**コンパイル時にsuspend関数の引数に`Continuation`が足されること**」ならびに「**呼び出し元の`Continuation`が渡されること**」を述べました。

では、大元の`Continuation`は、どのように作られるのでしょうか。
例えば、以下のようなコードでは、どこで`someSuspendFunc`に渡される`Continuation`が作られるのでしょうか。

▶️ [Kotlin Playgroundで実行](https://pl.kotl.in/ukcggsSPZ)
```kt
import kotlin.coroutines.*
import kotlinx.coroutines.*

suspend fun someSuspendFunc(/* コンパイル後には、Continuationを受け取る */) {
    suspendCoroutine { cont ->
        println(cont) // 受け取ったContinuationを出力する
        cont.resume(Unit)
    }
}

fun main() {
    runBlocking {
        launch {
            someSuspendFunc(/* コンパイル後には、Continuationが渡される */)
        }
    }
}
```

先に答えを述べると、`runBlocking`や`launch`のような**Coroutine Builder**で大元の`Continuation`が作成されます。Coroutine Builderとは、**Coroutineを起動する関数**です。

:::message
Coroutine Builderについて詳しく知りたい方は、こちらの記事^[Kotlin Coroutinesの核心：Builder・CoroutineScope・Job・CoroutineContextの関係: https://zenn.dev/kaseken/articles/99d92a128cbc9a]もご参照ください。
https://zenn.dev/kaseken/articles/99d92a128cbc9a
:::

一例として、`launch`のソースコードを以下に示します^[`launch`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L44]。

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

`LazyStandaloneCoroutine`あるいは`StandaloneCoroutine` (いずれも`AbstractCoroutine`を継承する) に対して、`start`メソッドが呼び出されていることが分かります。

その後、細部は省略しますが、以下のような流れで、最終的には`createCoroutineUnintercepted`メソッドが呼ばれます。

`AbstractCoroutine.start`^[`AbstractCoroutine.start`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/AbstractCoroutine.kt#L133] -> `CoroutineStart.invoke`^[`CoroutineStart.invoke`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineStart.kt#L356] -> `startCoroutineCancellable`^[`startCoroutineCancellable`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/intrinsics/Cancellable.kt#L15] -> `createCoroutineUnintercepted`^[`createCoroutineUnintercepted`のソースコード: https://github.com/JetBrains/kotlin/blob/211397c931b69d467248d53fcaafe76d155f3ed4/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt#L157]

`createCoroutineUnintercepted`のソースコードを以下に示します。

```kt
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted(
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion)
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
```

`createCoroutineUnintercepted`は、返り値として`Continuation<Unit>`を返しています。
`createCoroutineUnintercepted`のDocumentationコメントを見ると、以下のようにあります。

> To start executing the created coroutine, invoke `resume(Unit)` on the returned [Continuation] instance.
> The [completion] continuation is invoked when coroutine completes with result or exception.

このコメントから、`createCoroutineUnintercepted`で返される`Continuation`が、`launch`が起動するCoroutineに紐づく`Continuation`であると考えられます。
すなわち、**`createCoroutineUnintercepted`で返された`Continuation`が、そのCoroutine内で呼ばれたsuspend関数に渡される、大元の`Continuation`である**と推測されます。

## 次の学習ステップ

本記事では、suspend関数がどのように中断・再開可能となっているのかを、その内部実装から明らかにしました。

一方で、suspend関数に関わるトピックとして、本記事では触れていないものもあります。
例えば、suspend関数の中断・再開時には、何らかのスレッドあるいはスレッドプールへの割り当てが行われており、それは`CoroutineContext`や`CoroutineDispatcher`といった概念に支えられています。
これらについて更に詳しく学びたい方は、以下の記事も一読されることをお勧めします^[内部実装から理解するKotlin Coroutines：CoroutineDispatcher編: https://zenn.dev/kaseken/articles/7d5531a8eb1eae]。

https://zenn.dev/kaseken/articles/7d5531a8eb1eae
