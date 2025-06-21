# Kotlin Coroutinesの制御・キャンセル・例外処理を支える技術 ー Structured Concurrencyの内部実装

本記事は、Kotlin Coroutinesの基礎をすでに理解している中級者以上の開発者を対象とし、さらに理解を深めることを目的としたシリーズの一部です。
今回は、Kotlin Coroutinesの制御・キャンセル・例外処理を支える重要な概念である、**Structured Concurrency**について解説します。

:::message
本記事は入門者向けではありません。Kotlin Coroutinesの基礎は理解しているが、その内部実装についてはブラックボックス化しているという読者を主な対象としています。

Coroutine Builder・CoroutineScope・Job・CoroutineContext等の用語に馴染みがない方は、公式ドキュメント^[Coroutines: https://kotlinlang.org/docs/coroutines-overview.html]や私の書いた別の記事^[Kotlin Coroutinesの核心：Builder・CoroutineScope・Job・CoroutineContextの関係: https://zenn.dev/kaseken/articles/99d92a128cbc9a]を先に参照されることをお勧めします。
:::

## Structured Concurrencyの歴史と設計思想

まずは、Structured Concurrencyが、どのような背景のもと、どのような課題を解決するために生まれたのか、を説明します。
これは、**なぜKotlin Coroutinesの仕様を理解する上で、Structured Concurrencyの設計思想を理解することが不可欠である**ためです。

### Martin Sústrikによる創出 (2016年)

"Structured Concurrency"という用語は、2016年にMartin Sústrik^[sustrik: https://github.com/sustrik]が生み出したとされています^[JEP 505 Structured Concurrency: https://openjdk.org/jeps/505]。Martin氏は、C言語でStructured Concurrencyを実現するためのライブラリであるlibdill^[libdill: https://github.com/sustrik/libdill]を公開しました。

libdillのドキュメント上には、Martin氏の考えるStructured Concurrencyのコンセプトが説明されています^[Explanation of Structured Concurrency in libdill documentation: https://libdill.org/structured-concurrency.html]

このドキュメントでは、**親コルーチンが子コルーチンを起動した場合、子コルーチンが完了するまで親コルーチンが完了しないこと**がStructured Concurrencyの本質とされています。

![](https://storage.googleapis.com/zenn-user-upload/2b474802f92e-20250620.png)
*Structured Concurrencyに従った子コルーチンの起動*

一方で、**親コルーチンの完了後に、子コルーチンが残存するような場合は、Structured Concurrencyを違反します**。例えば、Javaでスレッドをforkした後、joinせずに起動元のスコープを抜けた場合には、このような状況になり得ます。

![](https://storage.googleapis.com/zenn-user-upload/0d3372e8921d-20250620.png)
*Structured Concurrencyに反する子コルーチンの起動*

Structured Concurrencyにおいては、親コルーチンの完了時には、子コルーチンも必ず完了していることから、**リソースのリークを防止できる**という大きなメリットがあります。

### Nathaniel J. Smithによる普及 (2018年)

その後、2018にNathaniel J. Smith^[njsmith: https://github.com/njsmith]は、PythonでStructured Concurrencyを実現するためのライブラリであるTrio^[Trio: https://github.com/python-trio/trio]を作成するとともに、Structured Concurrencyの有益性を主張するブログ記事 "Notes on structured concurrency, or: Go statement considered harmful" ^[Notes on structured concurrency, or: Go statement considered harmful: https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/]を公開しました。彼のブログ記事は、Structured Concurrencyの設計思想を広めることに貢献しました。

Nathanielは、構造化プログラミング (Structured Programming) とのアナロジーで、Structured Concurrencyを導入すべき理由を説明しています。

1960年台に遡ると、Edsger W. Dijkstraは構造化プログラミングを提唱し、goto構文を否定する有名な論文 "Go To statement considered harmful" を公表しました^[Edsger W. Dijkstra: https://en.wikipedia.org/wiki/Edsger_W._Dijkstra]。
構造化プログラミングとは、**「順次 (順番に処理を進める)」「選択 (分岐)」「反復 (ループ)」の3つの制御構造のみからプログラムを構成する手法**です^[構造化プログラミング: https://ja.wikipedia.org/wiki/%E6%A7%8B%E9%80%A0%E5%8C%96%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0]。

構造化プログラミングは今やスタンダードな考え方となっており、現代的なプログラミング言語でgoto構文がサポートされる自体が稀でしょう。
しかしながら、**並行処理に目を向けると、goto構文に相当するような、制御フローを壊しうる設計パターンが今もなお広く使われている**、というのがNathaniel氏の主張です。このような危険なパターンの代表例として、Go言語の`go`文が挙げられています。
結論として、**Structured Concurrencyが普及することで、構造化プログラミングによってプログラムの明確さ・安全性が得られたのと同様のパラダイムシフトとなる**とされています。

### Kotlin Coroutinesへの実装 (2018年)

Kotlinは、メジャーなプログラミング言語としては、**Structured Concurrencyを初めて公式の標準ライブラリでサポートした言語**と言えます。
Kotlin Coroutinesにおいては、**CoroutineScopeという概念によって、Structured Concurrencyを実現**しています。
CoroutineScope内でコルーチンが起動されると、**そのスコープ内で起動された全ての子コルーチンが完了するまで、CoroutineScope自体 (= 親コルーチン) は完了しません**。

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Ius3K8EON)

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope { // CoroutineScopeを作成
        launch { // CoroutineScope内で、コルーチンを起動
            delay(100)
            println("Delay finished.")
        }
    }
    println("All finished.")
}
```

**出力:**
```sh
Delay finished.
All finished.
```

また、コルーチンを起動するための関数 (= Coroutine Builder) は、CoroutineScopeをレシーバとする拡張関数として定義されています。すなわち、**すべてのコルーチンはCoroutineScopeを起点として起動する**必要があります。
この制約によって、Kotlin Coroutinesの利用者は、Structured Concurrencyの本質である「**子コルーチンが全て完了するまで、親コルーチンが完了しない**」という仕様に、半ば強制的に従うこととなります。

:::message
厳密には、GlobalScopeを使う、あるいは親のCoroutineScopeに紐づかないJobをCoroutine Builderの引数とするなど、Structured Concurrencyに反する書き方も可能ではあります。
ただ、プログラムを不明瞭にする、あるいはリソースがリークする危険性があるため、推奨されていません^[The reason to avoid GlobalScope: https://elizarov.medium.com/the-reason-to-avoid-globalscope-835337445abc]。
:::

---

以上、Structured Concurrencyの設計思想と、その背景を振り返りました。

Structured Concurrencyは、Kotlin Coroutinesにおいて、**並行処理を明瞭かつ安全に記述すること**に貢献しています。
一方で、十分に理解しきれていない人にとっては、むしろ複雑あるいは分かりにくいと感じさせる挙動も生み出しています。例えば、キャンセル処理・例外処理の流れを難しいと感じている方は少なくないのではないでしょうか。

そこで以降の章では、Kotlin Coroutinesにおいて、**Structured Concurrency ー すなわち子コルーチンの完了を待つという仕様 ー**が、どのように実現されているかを解明していきます。

## 子コルーチンの完了を待つ仕組み

Kotlin Coroutinesが、「**子コルーチンが全て完了するまで、親コルーチンが完了しない**」というStrucured Concurrencyの本質をどのように実現しているかを、内部実装から解明していきます。

この仕様を実現してする上で重要なのが「CoroutineScope」です。
**CoroutineScopeは、内部で起動された子コルーチンの完了を待機してから、CoroutineScope自体のスコープを抜けます。**

以下のサンプルコードを実行して確かめてみましょう。
ここで使用されている`coroutineScope`はCoroutineScopeを作成するためのメソッドです。また、`launch`はCoroutineScope内で子コルーチンを起動するためのメソッドです。
コードを実行すると、**外側の`coroutineScope`のスコープが、内部で`launch`された子コルーチンの完了を待つ**ため、"All finished."というメッセージが最後にprintされます。

▶️ [Kotlin Playgroundで試す](https://pl.kotl.in/Ius3K8EON)

```kt
import kotlinx.coroutines.*

suspend fun main() {
    coroutineScope { // CoroutineScopeを作成
        launch { // CoroutineScope内で、コルーチンを起動
            delay(100)
            println("Delay finished.")
        }
    }
    println("All finished.")
}
```

**出力:**
```sh
Delay finished.
All finished.
```

### `coroutineScope`が子コルーチンの完了を待つ仕組み

では、CoroutineScopeを作成している`coroutineScope`メソッドの内部実装を追うことで、**「子コルーチンの完了を待つ」仕組みがどのように実現されているのか**を解明していきましょう。

`coroutineScope`のソースコードを以下に抜粋します^[`coroutineScope`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/CoroutineScope.kt#L280]。
なお、今回の説明に関係しないコードは、省略しています。

```kt
public suspend fun <R> coroutineScope(block: suspend CoroutineScope.() -> R): R {
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        val coroutine = ScopeCoroutine(uCont.context, uCont)
        coroutine.startUndispatchedOrReturn(coroutine, block)
    }
}
```

`coroutineScope`内では、以下の**3つのステップ**で処理を行っています。

1. `suspendCoroutineUninterceptedOrReturn`で、**呼び出し元の`Continuation`を取得**する。
2. CoroutineScopeに紐づくコルーチンである、**`ScopeCoroutine`を作成**する。
3. `ScopeCoroutine`を`startUndispatchedOrReturn`で起動し、**CoroutineScope内の処理を実行**する。

1.のステップで登場する`Continuation`や`suspendCoroutineUninterceptedOrReturn`について、初めて聞いた読者も多いかもしれません。

これらの用語を簡単に説明します。

- `Continuation`とは、**コルーチン (suspend関数) の、中断・再開の状態を管理するインスタンス**。`Continuation`に対して`resumeWith`メソッドを呼び出すことで、呼び出し元の処理を再開する (**今回の場合は`coroutineScope`のスコープを抜ける**)。
- `suspendCoroutineUninterceptedOrReturn`とは、**"呼び出し元" (今回の場合`coroutineScope`の呼び出し元) の`Continuation`を取得する**ためのメソッド。

suspend関数がbytecodeへとコンパイルされると、**その引数には呼び出し元の`Continuation`が追加**されます。つまり、コンパイル後は、以下のようなbytecodeとなります。

```kt
public suspend fun <R> coroutineScope(
    block: suspend CoroutineScope.() -> R,
    continuation: Continuation<*>): R {
    /* 省略 */
}
```

この**コンパイル後に追加される`Continuation`を、コンパイル前のソースコード上で参照するために用いられるのが、`suspendCoroutineUninterceptedOrReturn`メソッド**です。

:::message
`Continuation`および`suspendCoroutineUninterceptedOrReturn`に関しては、それ単体で非常に複雑なトピックです。別の記事でも解説しているので、そちらもご一読いただくことをお勧めします^[内部実装から理解するKotlin Coroutines：suspend関数・Continuation編: https://zenn.dev/kaseken/articles/a50fd3f5e6e2ba]
:::

https://zenn.dev/kaseken/articles/a50fd3f5e6e2ba

---

`coroutineScope`メソッドの説明に戻ります。
`coroutineScope`内では、`suspendCoroutineUninterceptedOrReturn`によって**呼び出し元の`Continuation`を取得**し、**その`Continuation`が持つ呼び出し元の情報 ( `CoroutineContext`) を使って、`ScopeCoroutine`を作成・起動**しています。

```kt
    // 呼び出し元の`Continuation`を取得
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        // 呼び出し元の`CoroutineContext`と`Continuation`から、`ScopeCoroutine`を作成
        val coroutine = ScopeCoroutine(uCont.context, uCont)
        // `ScopeCoroutine`を起動し、`block`を実行
        coroutine.startUndispatchedOrReturn(coroutine, block)
    }
```

ここまでの流れを、図にまとめます。

![](https://storage.googleapis.com/zenn-user-upload/4a0aa178e605-20250620.png)
*`coroutineScope`メソッドの処理*

では、ここで作成されている`ScopeCoroutine`とは、一体何でしょうか？

#### `ScopeCoroutine`とは

`ScopeCoroutine`は、**CoroutineScopeに紐づくコルーチン**です。
一般的には、Kotlin Coroutinesにおいて、コルーチンは`launch`や`async`、`runBlocking`のようなCoroutine Builderから起動されると言われます。ただ、内部的には、**CoroutineScopeもそれ自体のコルーチンを作ります**。

:::message
ただし、`coroutineScope`内でコルーチンが作られるとはいえ、**`launch`や`async`のようなCoroutine Builderでコルーチンを起動する場合とは大きな違いがあります。**

先述した`coroutineScope`メソッドのソースコードを見ると、`ScopeCoroutine`は、**`startUndispatchedOrReturn`メソッドによって起動されています**。
このため、`coroutineScope`に渡された処理は、非同期的に実行 (`dispatch`) されるわけではなく、**コルーチンが起動されたスレッド上で同期的に実行**されます。
そのため、一般的な意味においては、「CoroutineScopeがコルーチンを起動している」とは言い難いです。
:::

---

続いて、`ScopeCoroutine`の初期化時の引数として渡されている、`CoroutineContext`と`Continuation`という2つのパラメータの役割を確認します。
これら2つは、**「子コルーチンの完了を待つ」というStructured Concurrencyの定義を実現する上で不可欠**な役割を果たしています。

#### `ScopeCoroutine`に渡された`CoroutineContext`の役割

まず、`CoroutineContext`の役割を見ていきます。結論から述べると、これは「**コルーチンの親子構造の形成**」という重要な役割を持ちます。

`ScopeCoroutine`のソースコードを以下に抜粋します^[`ScopeCoroutine`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/Scopes.kt#L11]。

```kt
/**
 * This is a coroutine instance that is created by [coroutineScope] builder.
 */
internal open class ScopeCoroutine<in T>(
    context: CoroutineContext,
    @JvmField val uCont: Continuation<T>
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame {
  /* 実装は省略 */
}
```

`ScopeCoroutine`の初期化時に渡された`context`パラメータは、**基底クラスである`AbstractCoroutine`の初期化パラメータ**として使用されています。

`AbstractCoroutine`のソースコードから、特に関係する部分を抜粋します^[`AbstractCoroutine`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/AbstractCoroutine.kt#L36]

```kt
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {

    init {
        /*
         * Setup parent-child relationship between the parent in the context and the current coroutine.
         * It may cause this coroutine to become _cancelling_ if the parent is already cancelled.
         * It is dangerous to install parent-child relationship here if the coroutine class
         * operates its state from within onCancelled or onCancelling
         * (with exceptions for rx integrations that can't have any parent)
         */
        if (initParentJob) initParentJob(parentContext[Job])
    }
```

`init`内で、`JobSupport` (基底クラス) の`initParentJob`メソッドが呼ばれています。
`initParentJob`内で、**呼び出し元 (親コルーチン) と、新たに作られた`ScopeCoroutine`との間に、親子構造が形成**されます。

#### Jobの親子構造の形成

ここで登場する`Job`とは、**コルーチンのライフサイクルやコルーチン間の親子構造を管理するもの**です。`Job`は、Structured Concurrencyを実現する上で、非常に重要なパーツです。

ソースコード上は、`Job`はinterfaceであり、具体的な実装は`JobSupport`クラスに存在します。
Structured Concurrencyに関わる多くの処理 (状態管理・キャンセル処理・例外処理など) が、`JobSupport`クラス内で実装されているため、このクラスは以降頻出します。

では、Jobの親子構造の形成を担っている、`JobSupport`クラスの`initParentJob`の内部実装を見ていきます。

`JobSupport`の`initParentJob`メソッドのソースコードを以下に示します^[`JobSupport`の`initParentJob`メソッドのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L141]。

```kt
    protected fun initParentJob(parent: Job?) {
        assert { parentHandle == null }
        if (parent == null) {
            parentHandle = NonDisposableHandle
            return
        }
        parent.start() // make sure the parent is started
        // ここでJobの親子関係を形成している。
        val handle = parent.attachChild(this)
        parentHandle = handle
        if (isCompleted) {
            handle.dispose()
            parentHandle = NonDisposableHandle // release it just in case, to aid GC
        }
    }
```

着目して頂きたいのは、**`parent.attachChild(this)`の部分**です。
ここで、**親の`Job`に対して、このコルーチンに紐づく`Job`をattachする (取り付ける) ことで、親子構造を形成**しています。

また、`attachChild`メソッドからは`ChildHandle`^[`ChildHandle`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Job.kt#L461]というオブジェクトが返されます。このオブジェクトは、子`Job`の`parentHandle`というフィールドに保持されます。
この`ChildHandle`とは、**子コルーチンのJobから、親コルーチンのJobを参照するためのハンドラ**です。
後のセクションで触れますが、子コルーチンからキャンセル処理・例外処理を親コルーチンに伝える際には、この`ChildHandle`が使用されます。

:::messages
`initParentJob`によって親子構造を形成する処理は、あらゆるコルーチンの基底クラスである`AbstractCoroutine`から呼ばれています。そのため、`ScopeCoroutine`に限らず、あらゆるコルーチンの初期化時に、同様のことが行われます。

例えば、後ほど`launch`によってコルーチンが起動される際の内部実装も説明しますが、その際にも「起動元のCoroutineScopeのコルーチン」と「`launch`で起動された子コルーチン」の間の親子関係の形成するために、ここで説明した`JobSupport`の`initParentJob`が使われています。
:::

---

ここまでで、`ScopeCoroutine`に渡された`CoroutineContext`の役割を見てきました。
その重要な役割の一つに、**呼び出し元の親コルーチンと、新しく作られた`Job`の親子構造を形成すること**があると分かりました。

![](https://storage.googleapis.com/zenn-user-upload/f76ce4fb1c63-20250620.png)
*`Job`の親子構造の形成*

---

続いて、もう一方の、`ScopeCoroutine`に渡された`Continuation`の役割を見ていきましょう。

結論から述べると、`CoroutineContext`は「親子構造の形成」を担ったのに対し、`Continuation`は**その親子構造に依存して「子コルーチンの実行を待つ」という役割**を担います。

#### `ScopeCoroutine`に渡された`Continuation`の役割

`ScopeCoroutine`の初期化時に渡された`Continuation`は、`uCont`フィールドとして、`ScopeCoroutine`内に保持されます。
`ScopeCoroutine`のソースコード^[`ScopeCoroutine`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/internal/Scopes.kt#L11]から、`Continuation`に関わる部分を抜粋します。

```kt
internal open class ScopeCoroutine<in T>(
    context: CoroutineContext,
    @JvmField val uCont: Continuation<T> // unintercepted continuation
) : AbstractCoroutine<T>(context, true, true), CoroutineStackFrame {

    override fun afterCompletion(state: Any?) {
        uCont.intercepted().resumeCancellableWith(recoverResult(state, uCont))
    }

    override fun afterResume(state: Any?) {
        uCont.resumeWith(recoverResult(state, uCont))
    }
}
```

渡された`Continuation`は、`afterCompletion`と`afterResume`の2箇所で使用されており、それぞれ`resumeCancellableWith`あるいは`resumeWith`が呼ばれています。
これらのメソッドは、挙動に違いはあるものの、**呼び出し元の`Continuation`に制御を戻す、すなわち`coroutineScope`自体のスコープを抜けるタイミングで呼ばれる**、という点では共通しています。

つまり、これらのメソッドが呼ばれるまでの経路を知ることで、**Structured Concurrencyの本質である「子コルーチンの完了を待つ」ことの仕組み**を理解することに繋がります。

#### `ScopeCoroutine`の起動

先述した`afterCompletion`あるいは`afterResume`は、`ScopeCoroutine`の実行の最終段階で呼ばれるものと考えられます。
そこで、**CoroutineScopeに紐づく`ScopeCoroutine`の起動から完了までの流れ**を追うことで、どのような経路で`afterCompletion`あるいは`afterResume`の呼び出しに繋がるのかを解明しましょう。

`coroutineScope`のソースコードを、以下に再掲します。

```kt
public suspend fun <R> coroutineScope(block: suspend CoroutineScope.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        val coroutine = ScopeCoroutine(uCont.context, uCont)
        // `ScopeCoroutine`の起動
        coroutine.startUndispatchedOrReturn(coroutine, block)
    }
}
```

CoroutineScopeに紐づく`ScopeCoroutine`は、**`startUndispatchedOrReturn`メソッドによって起動**されます。

`startUndispatchedOrReturn`ならびに`startUndispatched`のソースコードを以下に示します^[`startUndispatchedOrReturn`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/intrinsics/Undispatched.kt#L41]。

```kt
internal fun <T, R> ScopeCoroutine<T>.startUndispatchedOrReturn(
    receiver: R, block: suspend R.() -> T
): Any? = startUndispatched(alwaysRethrow = true, receiver, block)

private fun <T, R> ScopeCoroutine<T>.startUndispatched(
    alwaysRethrow: Boolean,
    receiver: R, block: suspend R.() -> T
): Any? {
    val result = try {
        block.startCoroutineUninterceptedOrReturn(receiver, this)
    } catch (e: DispatchException) {
        dispatchExceptionAndMakeCompleting(e)
    } catch (e: Throwable) {
        CompletedExceptionally(e)
    }

    if (result === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
    val state = makeCompletingOnce(result)
    if (state === COMPLETING_WAITING_CHILDREN) return COROUTINE_SUSPENDED
    afterCompletionUndispatched()
    return if (state is CompletedExceptionally) {
        when {
            alwaysRethrow || notOwnTimeout(state.cause) -> throw recoverStackTrace(state.cause, uCont)
            result is CompletedExceptionally -> throw recoverStackTrace(result.cause, uCont)
            else -> result
        }
    } else {
        state.unboxState()
    }
}
```

まず、、**引数として渡された`block` (これは`coroutineScope`の内部の処理) を、実行スレッド上でそのまま実行**しています。その実行結果 (`result`変数) に応じて、異なるハンドリングを行っています。

`block`の実行結果は、3パターンに分かれます。それぞれどのようにハンドリングされるかを見ていきます。

#### パターン1. `coroutineScope`に渡された`block`が中断された場合

パターン1に相当するコードを示します。

```kt
coroutineScope {
    delay(100) // suspension point: ここで実行が一時中断される。
    launch { delay(1000) }
}
```

このように`coroutineScope`内にsuspension pointが存在する場合 (ここではsuspend関数である`delay`が存在する)、ここで処理が中断されます。

この場合、`block`の実行は中断されているため、`result`は`COROUTINE_SUSPENDED`となります。その結果、`startUndispatched`では、`coroutineScope`の`block`を実行するのみで早期returnすることとなります。 

```kt
    val result = try {
        block.startCoroutineUninterceptedOrReturn(receiver, this)
    } catch (e: DispatchException) {
        dispatchExceptionAndMakeCompleting(e)
    } catch (e: Throwable) {
        CompletedExceptionally(e)
    }

    if (result === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
```

その後、`coroutineScope`の`block`が完了したタイミングで、coroutineScopeに紐づくコルーチン (`ScopeCoroutine`) に対して、非同期的に`resumeWith`が呼ばれます。

`resumeWith`のソースコードを以下に示します^[`AbstractCoroutine`の`resumeWith`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/AbstractCoroutine.kt#L98]。これは、基底クラスの`AbstractCoroutine`に存在します。

```kt
    public final override fun resumeWith(result: Result<T>) {
        val state = makeCompletingOnce(result.toState())
        if (state === COMPLETING_WAITING_CHILDREN) return
        afterResume(state)
    }
```

ここで登場している`makeCompletingOnce`は、**コルーチンの完了処理を進めるためのメソッド**です。パターン2でも`makeCompletingOnce`は利用されるため、後ほど詳細に解説します。

`makeCompletingOnce`の呼び出し後の流れは2パターンに分かれます。

**`coroutineScope`内の`block`が完了した際に、未完了の子コルーチンが残っている場合:**
`makeCompletingOnce`でコルーチンの完了処理を進めた結果、**まだ未完了の子コルーチンが存在する場合 (`COMPLETING_WAITING_CHILDREN`が返された場合) は、子コルーチンの完了を待機するために、何もせずreturn**します。
後ほど説明しますが、こちらの場合には、全ての子コルーチンが完了した時点で`afterCompletion`が呼ばれます。

**`coroutineScope`内の`block`が完了した際に、未完了の子コルーチンがない場合:**
`makeCompletingOnce`でコルーチンが完了した場合 (= 未完了の子コルーチンが存在しない場合) には、**`afterResume`を呼び出します**。
**`ScopeCoroutine`の`afterResume`が呼ばれ、CoroutineScopeのスコープを抜ける**こととなります。

言葉だけでは難しいので、サンプルコードを図示して説明します。これは`makeCompletingOnce`の後に、**未完了のコルーチンが存在するパターン**に相当します。

```kt
coroutineScope {
    delay(100) // suspension point: ここで実行が一時中断される。
    launch { delay(1000) }
}
```

![](https://storage.googleapis.com/zenn-user-upload/1004459eb522-20250620.png)
*パターン1の流れ: `coroutineScope`の`block`終了時に、未完了の子コルーチンが存在するケース*

一方、以下のようなサンプルコードは、`coroutineScope`の`block`終了時に未完了の子コルーチンが存在しないパターンに相当します。

```kt
coroutineScope {
    launch { delay(100) }
    delay(1000) // delayの間にlaunchされた子コルーチンが完了する。
}
```

![](https://storage.googleapis.com/zenn-user-upload/8835acc4b55a-20250620.png)
*パターン1の流れ: `coroutineScope`の`block`終了時に、未完了の子コルーチンが存在しないケース*

いずれのケースにおいても、**「子コルーチンが完了するまで、親コルーチンが完了しない」というStructured Concurrencyの原則が満たされている**ことが分かります。

#### パターン2. `coroutineScope`に渡された`block`は完了するものの、未完了の子コルーチンが存在する場合

パターン2に相当するコードを、以下に示します。

```kt
coroutineScope {
    launch { delay(1000) } // `block`内の処理が終わった直後は未完了
}
```

`launch`自体はsuspend関数ではないので`block`内のコードは中断されません。
しかし、`block`内の処理が完了した時点では、`launch`によって起動された子コルーチンが未完了です。

以下に`startUndispatched`のコードを抜粋します。

```kt
    val result = try {
        block.startCoroutineUninterceptedOrReturn(receiver, this)
    } catch (e: DispatchException) {
        dispatchExceptionAndMakeCompleting(e)
    } catch (e: Throwable) {
        CompletedExceptionally(e)
    }

    if (result === COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
    val state = makeCompletingOnce(result)
    if (state === COMPLETING_WAITING_CHILDREN) return COROUTINE_SUSPENDED
```

パターン2では、`coroutineScope`内の`block`自体は、中断することなく完了します。
そのため、`result`には`COROUTINE_SUSPENDED`ではなく実行結果が入り、一つ目のif文は通過します。そして、先ほども登場した`makeCompletingOnce`が呼ばれます。

`makeCompletingOnce`はコルーチンの完了処理を進めるためのメソッドです。
パターン2は、`makeCompletingOnce`を実行した結果、`COMPLETING_WAITING_CHILDREN`が返される、すなわち**未完了の子コルーチンが残っている**場合です (一方で、残っていない場合は、後述するパターン3となります)。

`makeCompletingOnce`が実行されると、**全ての子コルーチンが完了したタイミングで、`ScopeCoroutine`に対して`afterCompletion`が呼ばれます**。
これによって、`coroutineScope`の呼び出し元の`Continuation`の`resumewWith`が呼ばれ、`coroutineScope`のスコープを抜けることとなります。

以下のコードを例に、パターン2における処理の流れを図示します。

```kt
coroutineScope {
    launch { delay(1000) } // `block`内の処理が終わった直後は未完了
}
```

![](https://storage.googleapis.com/zenn-user-upload/81260a90e7ca-20250620.png)
*パターン2の処理の流れ*


#### パターン3. `coroutineScope`に渡された`block`が完了し、未完了の子コルーチンも存在しない場合

パターン3に相当するコードを、以下に示します。

```kt
coroutineScope {
    println("Hello, World!")
}
```

このケースは、`coroutineScope`に渡された`block`が中断せず、かつ`block`の完了時点で未完了の子コルーチンも存在しないケースです。実際の使い方で、このようなケースが生じることは稀かもしれません。

このパターンでは、`block`が同期的に完了し、即座に`coroutineScope`のスコープを抜けることになります。

![](https://storage.googleapis.com/zenn-user-upload/1a4b110d3ceb-20250620.png)
*パターン3の処理の流れ*

#### `makeCompletingOnce`の内部実装

パターン1・パターン2で使用されていた、`makeCompletingOnce`の内部実装を解説します。

`makeCompletingOnce`は、**コルーチンの完了を進めるためのメソッド**です。
`makeCompletingOnce`は`tryMakeCompleting`メソッドを呼び、子コルーチンが未完了な場合には、そこから更に`tryMakeCompletingSlowPath`メソッドが呼ばれます。

`tryMakeCompletingSlowPath`から、特に重要な部分を抜粋して以下に示します^[`tryMakeCompletingSlowPath`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L902]。

```kt
  private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
        // 子コルーチンの一覧を取得する。
        val list = getOrPromoteCancellingList(state) ?: return

        // 子コルーチンを1つ取得する。
        val child = list.nextChild()
        // 未完了のコルーチンが存在する場合には、`COMPLETING_WAITING_CHILDREN`を返す。
        if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
            return COMPLETING_WAITING_CHILDREN

        return finalizeFinishingState(finishing, proposedUpdate)
    }
```

子コルーチンに対して`tryWaitForChild`メソッドを呼び出すことで、子コルーチンが完了するまで待機しています。

`tryWaitForChild`のソースコードを以下に示します^[`tryWaitForChild`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L954]。

```kt
    private tailrec fun tryWaitForChild(state: Finishing, child: ChildHandleNode, proposedUpdate: Any?): Boolean {
        val handle = child.childJob.invokeOnCompletion(
            invokeImmediately = false,
            handler = ChildCompletion(this, state, child, proposedUpdate)
        )
        // 子コルーチンが未完了であれば、returnして完了を待つ。
        if (handle !== NonDisposableHandle) return true
        // 他に子コルーチンがあれば、それに対して再帰的に`tryWaitForChild`を呼び出す。
        val nextChild = child.nextChild() ?: return false
        return tryWaitForChild(state, nextChild, proposedUpdate)
    }
```

子コルーチンのJobに対して、`invokeOnCompletion`メソッドが呼ばれています。
これはコールバック関数に等しく、子コルーチンの完了時に、ハンドラ (`ChildCompletion`) が呼ばれるようになります。

また、この子コルーチン自体はすでに完了している場合には、別の子コルーチンに対して`tryWaitForChild`を再帰的に呼び出しています。
これにより、**未完了の子コルーチンが一つでも存在する場合には、`tryWaitForChild`メソッドはtrueを返す**ことになります。

続いて、子コルーチンに渡されているハンドラである、`ChildCompletion`のソースコードを以下に示します^[`ChildCompletion`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L1260]。

```kt
    private class ChildCompletion(
        private val parent: JobSupport,
        private val state: Finishing,
        private val child: ChildHandleNode,
        private val proposedUpdate: Any?
    ) : JobNode() {
        override val onCancelling get() = false
        override fun invoke(cause: Throwable?) {
            parent.continueCompleting(state, child, proposedUpdate)
        }
    }
```

子コルーチンの完了時に、`continueCompleting`が呼ばれることが分かります。
そこで、`continueCompleting`メソッドのソースコードから、特に重要な部分を抜粋して、以下に示します。^[`continueCompleting`メソッドのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L965]

```kt
    private fun continueCompleting(state: Finishing, lastChild: ChildHandleNode, proposedUpdate: Any?) {
        val waitChild = lastChild.nextChild()
        if (waitChild != null && tryWaitForChild(state, waitChild, proposedUpdate)) return

        // 全ての子コルーチンが完了している場合、`afterCompletion`を呼び出す。
        val finalState = finalizeFinishingState(state, proposedUpdate)
        afterCompletion(finalState)
    }
```

`tryMakeCompletingSlowPath`と同様に、`tryWaitForChild`が呼ばれています。
また、`tryWaitForChild`がfalseを返した場合、すなわち**全ての子コルーチンが完了している場合には、`afterCompletion`が呼ばれます**。
`ScopeCoroutine`の`afterCompletion`が呼ばれることで、`coroutineScope`の呼び出し元の`Continuation`に対して`resumeCancellableWith`が呼ばれ、`coroutineScope`のスコープから抜けることになります。

`makeCompletingOnce`が呼ばれてからの流れを、図に整理します。

![](https://storage.googleapis.com/zenn-user-upload/d2c54fb71da4-20250620.png)
*`makeCompletingOnce`が呼ばれてからの流れ*

全ての子コルーチンが完了するまで、「`tryWaitForChild` -> ハンドラを子コルーチンのJobに登録 -> 子コルーチンの完了時に`continueCompletion`が呼ばれる -> `tryWaitForChild`」というループが継続することが分かります。

以上のようにして、CoroutineScopeは「全ての子コルーチンの完了を待つ」というStructured Concurrencyの原則を実現しています。

### `launch`が`CoroutineScope`との親子構造を形成する仕組み

`CoroutineScope`が、内部で起動された子コルーチンの完了を待つには、子コルーチンとの親子構造が形成される必要があります。
そこで、**`coroutineScope`の内部でコルーチンを`launch`した際に、「`CoroutineScope`のコルーチン」と「`launch`された子コルーチン」の間に親子構造が形成される仕組み**を確認します。

`launch`のソースコードを以下に示します^[`launch`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Builders.common.kt#L44]。

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

Jobの親子構造の形成には、最初の2つのステップが関わっています。

1. **`newCoroutineContext`の作成**
2. **`StandaloneCoroutine`の作成**

それぞれの実装を追っていきます。

#### `newCoroutineContext`の作成

`newCoroutineContext`はプラットフォームごとに実装が定義されています。
ここではJVM版のソースコードを示します^[`newCoroutineContext`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/jvm/src/CoroutineContext.kt#L14]。

```kt
public actual fun CoroutineScope.newCoroutineContext(context: CoroutineContext): CoroutineContext {
    val combined = foldCopies(coroutineContext, context, true)
    val debug = if (DEBUG) combined + CoroutineId(COROUTINE_ID.incrementAndGet()) else combined
    return if (combined !== Dispatchers.Default && combined[ContinuationInterceptor] == null)
        debug + Dispatchers.Default else debug
}
```

重要なのは、1行目の`foldCopies(coroutineContext, context, true)`です。
ここで、親である`CoroutineScope`の持つ`CoroutineContext`と、`launch`のパラメータとして渡された`CoroutineContext`をマージしています。
この結果として、`newCoroutineContext`が返す`CoroutineContext`には、(基本的には) 親の`Job`が含まれることになります。

:::message
ここで"基本的には"と書いた理由は、`launch(Job())`のように、`launch`の引数に`Job`を明示的に渡すと、この`Job`によって上書きされ、親の`Job`が含まれないようになるためです。
しかし、これは`Job`の親子構造の形成を妨げる、すなわちStructured Concurrencyが機能しない状態になるため、通常行うべきではありません。
:::

#### `StandaloneCoroutine`の作成

続いて、`launch`内の以下の部分で、`StandaloneCoroutine`あるいは`LazyStandaloneCoroutine`が作られます。

```kt
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
```

`StandaloneCoroutine`あるいは`LazyStandaloneCoroutine`のいずれになるかは、コルーチンの起動直後の状態を左右します。後者は、明示的に`start`されるまでコルーチンが起動しません。
ただ、今回調査している`Job`の親子構造の形成プロセスにおいては、両者の基底クラスである`AbstractCoroutine`が関係しているため、いずれであっても違いはありません。

`StandaloneCoroutine`あるいは`LazyStandaloneCoroutine`の初期化パラメータとして、前ステップで作成された`newContext` (呼び出し元の`CoroutineContext`と`launch`のパラメータの`CoroutineContext`をマージしたもの) が渡されています。
これにより、基底クラスの`AbstractCoroutine`へと、この`CoroutineContext`が渡されることになります。

`AbstractCoroutine`の初期化時の流れに関しては、既に`ScopeCoroutine`のパートでも触れました。再び`AbstractCoroutine`のソースコードを抜粋します。

```kt
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
    init {
        if (initParentJob) initParentJob(parentContext[Job])
    }
```

`init`内の`initParentJob`メソッドにより、**「呼び出し元のコルーチンのJob」と「`launch`されたコルーチンのJob」の間に親子構造**が形成されます。

---

ここまでは、子コルーチンが正常に完了することを想定して、どのように親の`CoroutineScope`が子コルーチンの完了を待つか、を追ってきました。

一方で、**子コルーチンが完了に至らない、すなわち途中でキャンセルされるケース**も存在します。
そういったケースにおいても、「**子コルーチンが全て完了するまで、親コルーチンが完了しない**」というStructured Concurrencyの原則は守られます。そこで次のセクションでは、キャンセル処理に関して解説します。

## キャンセル処理でStructured Concurrencyが守られる仕組み

コルーチンのキャンセルには、「**正常なキャンセル**」と「(**`CancellationException`以外の) 例外発生による失敗**」の2種類が存在します。
ただ、いずれのパターンでも、キャンセルの伝播に共通の機構を用いているため、併せて解説していきます。

### キャンセル・例外処理の基本的な仕様

まずは簡単に、Kotlin Coroutinesにおけるキャンセル・例外処理の基本的な仕様をおさらいします。

1つ目の「正常なキャンセル」においては、**あるコルーチンを正常にキャンセルすると、その子孫のコルーチンも正常にキャンセル**されます。一方、親コルーチンへは、キャンセルが伝播されません。

![](https://storage.googleapis.com/zenn-user-upload/d2e5811e2ba2-20250620.png)
*正常なキャンセルの伝播*

`Job`に対して`cancel`メソッドが明示的に呼ばれた場合、あるいはコルーチン内で`CancellationException`が投げられた場合などに、正常なキャンセルが開始されます。

---

一方で、「例外発生による失敗」においては、**あるコルーチンが失敗すると、その子孫のコルーチンだけでなく、その親のコルーチン (そしてその子孫のコルーチン) もキャンセル**されます。
コルーチン内で`CancellationException`以外の例外が投げられた場合には、失敗の伝播が開始されます。

![](https://storage.googleapis.com/zenn-user-upload/fa9736b9753d-20250620.png)
*例外発生による失敗の伝播*

---

いずれのケースにおいても、子孫のコルーチンのキャンセルが完了してから、親コルーチンのキャンセルも完了します。
この仕様により、**キャンセル時のコルーチンの生存期間も、Structured Concurrencyに従う**こととなります。

### 例外が投げられた場合のキャンセル伝播の流れ

`launch`内で例外が発生すると、**`launch`で起動されたコルーチンに紐づく`Continuation`に対して、`resumeWith(Exception)`が呼び出され、`launch`が終了**することとなります。

`resumeWith`が呼ばれると、以下のような流れで、最終的に`JobSupport`の`tryMakeCompletingSlowPath`メソッドが呼ばれます。

1. `resumeWith`から`JobSupport.makeCompletingOnce`が呼ばれる。
2. `JobSupport.makeCompletingOnce`から`JobSupport.tryMakeCompleting`が呼ばれる。
3. `JobSupport.tryMakeCompleting`から`JobSupport.tryMakeCompletingSlowPath`が呼ばれる。

`JobSupport`の`tryMakeCompletingSlowPath`から、例外処理に関連するコードを抜粋して、以下に示します^[`tryMakeCompletingSlowPath`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L902]。

```kt
    private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
        // `list`は子コルーチンの一覧
        val list = getOrPromoteCancellingList(state) ?: return 
        val finishing = state as? Finishing ?: Finishing(list, false, null)
        val notifyRootCause: Throwable?
        synchronized(finishing) {
            val wasCancelling = finishing.isCancelling
            (proposedUpdate as? CompletedExceptionally)?.let { finishing.addExceptionLocked(it.cause) }
            // まだキャンセル中でない場合、キャンセル処理を開始する。
            // 失敗の原因となった例外を`notifyRootCause`にセットする。
            notifyRootCause = finishing.rootCause.takeIf { !wasCancelling }
        }
        // 例外によって終了していた場合、子コルーチンにキャンセルを伝播する。
        notifyRootCause?.let { notifyCancelling(list, it) }
        // `tryWaitForChild`を呼び出し、全ての子コルーチンが実行完了あるいはキャンセル完了するまで待機する。
        val child = list.nextChild()
        if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
            return COMPLETING_WAITING_CHILDREN
        return finalizeFinishingState(finishing, proposedUpdate)
    }
```

`tryMakeCompletingSlowPath`内で、以下のような流れで例外処理が行われます。

1. 子コルーチンの一覧を`list`変数にセットする。
2. **キャンセル・例外発生によって終了していた場合、その例外を`notifyRootCause`にセット**する。
3. キャンセル・例外発生によって終了していた場合、**子コルーチンにキャンセルを伝播**する。
具体的には、`list`と`notifyRootCause`を引数として、`notifyCancelling`が呼ばれる。
4. **`tryWaitForChild`メソッドにより、全ての子コルーチンが完了 (正常終了もしくはキャンセル完了) するまで待機**する。

4つ目のステップで利用されている`tryWaitForChild`は、子コルーチンが正常終了する際にも使われていたものです。
例外処理においても、**正常系と同様の仕組みで、子コルーチンの完了が待機されている**ことが分かります。

---

続いて、`notifyCancelling`メソッドのソースコード^[`notifyCancelling`メソッドのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L320]を以下に示します。

```kt
    private fun notifyCancelling(list: NodeList, cause: Throwable) {
        // first cancel our own children
        onCancelling(cause)
        list.close(LIST_CANCELLATION_PERMISSION)
        notifyHandlers(list, cause) { it.onCancelling }
        // then cancel parent
        cancelParent(cause) // tentative cancellation -- does not matter if there is no parent
    }
```

`notifyCancelling`内で行われていることとして、以下の2つが特に重要です。

1. `notifyHandlers`メソッドによって、**子コルーチンにキャンセルを伝播**する。
2. `cancelParent`メソッドによって、**親コルーチンにキャンセルを伝播**する。

#### 子コルーチンへのキャンセルの伝播

`notifyHandlers`メソッドのソースコード^[`notifyHandlers`メソッドのソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L360]を以下に示します。

```kt
    private inline fun notifyHandlers(list: NodeList, cause: Throwable?, predicate: (JobNode) -> Boolean) {
        var exception: Throwable? = null
        list.forEach { node ->
            if (node is JobNode && predicate(node)) {
                try {
                    node.invoke(cause)
                } catch (ex: Throwable) {
                    exception?.apply { addSuppressed(ex) } ?: run {
                        exception = CompletionHandlerException("Exception in completion handler $node for $this", ex)
                    }
                }
            }
        }
        exception?.let { handleOnCompletionException(it) }
    }
```

`node.invoke(cause)`の部分で、`ChildHandleNode`の`invoke`メソッドが呼ばれています。
`ChildHandleNode`のソースコード^[`ChildHandleNode`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L1575]を見ると、`invoke`メソッドでは、**`ChildJob`に対して`parentCancelled`が呼ばれています**。

```kt
private class ChildHandleNode(
    @JvmField val childJob: ChildJob
) : JobNode(), ChildHandle {
    override val parent: Job get() = job
    override val onCancelling: Boolean get() = true
    override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
    override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
}
```

`JobSupport`にある`parentCancelled`メソッドの実装を、以下に示します^[`parentCancelled`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L667]。

```kt
    public final override fun parentCancelled(parentJob: ParentJob) {
        cancelImpl(parentJob)
    }
```

`cancelImpl`メソッドが呼ばれています。詳細な解説は省略しますが、`cancelImpl`メソッドを通じて、再び「そのコルーチンのキャンセル」と「親・子コルーチンへのキャンセル伝播」が行われます。
このようにして、**子孫コルーチンへとキャンセルが伝播**していきます。

#### 親コルーチンへのキャンセルの伝播

続いて、`notifyHandlers` (子コルーチンへのキャンセル伝播) の後に呼ばれる、`cancelParent`のソースコード^[`cancelParent`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L336]を以下に示します。

```kt
    private fun cancelParent(cause: Throwable): Boolean {
        if (isScopedCoroutine) return true

        val isCancellation = cause is CancellationException
        val parent = parentHandle
        if (parent === null || parent === NonDisposableHandle) {
            return isCancellation
        }

        return parent.childCancelled(cause) || isCancellation
    }
```

メソッド名からも明らかなように、これは**親コルーチンをキャンセルするためのメソッド**です。
重要なのは`parent.childCancelled`の部分で、ここで親コルーチンに対してキャンセルが伝播されます。
`JobSupport`内にある`childCancelled`のソースコードを以下に示します^[`childCancelled`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/JobSupport.kt#L680]。

```kt
    public open fun childCancelled(cause: Throwable): Boolean {
        if (cause is CancellationException) return true
        return cancelImpl(cause) && handlesException
    }
```

まず、**原因が`CancellationException`だった場合には、親のキャンセルがスキップ**されています。これによって「**正常なキャンセルだった場合、子孫コルーチンのみに伝播する**」という仕様が実現されています。

一方、**原因が`CancellationException`以外の例外だった場合には、親コルーチンに対して、`cancelImpl`が呼び出されます**。
`cancelImpl`は子コルーチンへのキャンセル伝播の仕組みを追った際にも登場しました。
これは「**そのコルーチンのキャンセル**」と「**親・子孫コルーチンへのキャンセル伝播**」を行うメソッドです。
このようなメカニズムで、**`CancellationException`以外の例外発生時には、親コルーチンもキャンセルされる**ことになります。

#### `supervisorScope`を使うとなぜキャンセルが伝播しないのか

`coroutineScope`の代わりに`supervisorScope`を利用すると、**`CancellationException`以外の例外が発生した場合でも、そのコルーチンの親・子孫に対してキャンセルが伝播しません**。

![](https://storage.googleapis.com/zenn-user-upload/b6f9305783fe-20250621.png)
*`supervisorScope`利用時の失敗の伝播*

`supervisorScope`を使ったコードの例を、以下に示します。

▶️ Kotlin Playgroundで試す

```kt
import kotlinx.coroutines.*

suspend fun main() {
    supervisorScope {
        launch { // Launch Coroutine A.
            throw Exception("Some error message.")
        }
        .invokeOnCompletion { cause -> println("Completed Child Coroutine A, cause: $cause") }
        launch { // Launch Coroutine B.
            delay(100)
        }
        .invokeOnCompletion { cause -> println("Completed Child Coroutine B, cause: $cause") }
    }
    println("supervisorScope completed.")
}
```

**出力:**
```kt
Exception in thread "DefaultDispatcher-worker-2 @coroutine#1"
Completed Child Coroutine A, cause: java.lang.Exception: Some error message.
Completed Child Coroutine B, cause: null
supervisorScope completed.
```

`launch`されたコルーチンで例外が発生しても、**親である`supervisorScope`や、同じ`supervisorScope`内で`launch`された他のコルーチンはキャンセルされません** (試しに`coroutineScope`に変えてみると、違いがよく分かるかと思います)。

`supervisorScope`のソースコード^[`supervisorScope`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Supervisor.kt#L50]から、上記の挙動となる理由が分かります。

```kt
public suspend fun <R> supervisorScope(block: suspend CoroutineScope.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return suspendCoroutineUninterceptedOrReturn { uCont ->
        val coroutine = SupervisorCoroutine(uCont.context, uCont)
        coroutine.startUndispatchedOrReturn(coroutine, block)
    }
}
```

`coroutineScope`と異なり、`ScopeCoroutine`の代わりに`SupervisorCoroutine`が起動されています。
`SupervisorCoroutine`のソースコード^[`SupervisorCoroutine`のソースコード: https://github.com/Kotlin/kotlinx.coroutines/blob/f4f519b36734238ec686dfaec1e174086691781e/kotlinx-coroutines-core/common/src/Supervisor.kt#L64]は以下の通りで、`ScopeCoroutine`を継承しています。

```kt
private class SupervisorCoroutine<in T>(
    context: CoroutineContext,
    uCont: Continuation<T>
) : ScopeCoroutine<T>(context, uCont) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

`SupervisorCoroutine`では、**`childCancelled`がオーバーライドされており、デフォルトの挙動と異なりキャンセル処理を行わない** (`cancelImpl`を呼ばない) ようになっています。
そのため、子コルーチンの`cancelParent`メソッドから`childCancelled`が呼ばれても、`supervisorScope`に紐づくコルーチンがキャンセルされません。

---

以上、キャンセルがどのように親・子孫コルーチンへと伝播されるのか、そしてどのように子コルーチンのキャンセル完了を待つのかを、ソースコードから明らかにしました。
なお、実際の例外処理のソースコードはもっと複雑で、説明しきれていない部分も多いことをご承知おきください。

## まとめ

本記事では、**Structured Concurrencyとは何か、そしてKotlin CoroutinesにおいてどのようにStructured Concurrencyが実現されているのか**を解説しました。

私自身、CoroutineScopeが内部のコルーチンの完了を待つ仕組みなど、「なぜこう動くのか分からない」という部分が多くありました。
ただ、今回の調査を通じて、そういった「ブラックボックス」に光を当てたことで、より自信を持ってKotlin Coroutinesの実装やデバッグ、レビューができるようになったと思っています。

読者の皆様にも、同じような価値を提供できれば嬉しいです。また、同じくKotlin Coroutines自体のソースコードを読んでみたいという方への指針になればと思います。
