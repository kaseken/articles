# Android開発におけるJVMバージョン設定

Androidアプリのビルドには、Java Development Kit (JDK) のバージョンに関する種々の設定――例えば、Toolchain、Target Compatibility、Source Compatibility、Kotlin  jvmTarget、JAVA_HOME など――が関係します。
これらの設定の役割・違いは、Androidの公式ドキュメント ^[Java versions in Android builds:  https://developer.android.com/build/jdks]では、以下のような図として整理されています。しかし、少なくとも私にとっては、初見で簡単に理解できるものではありませんでした。

![](https://storage.googleapis.com/zenn-user-upload/44976ff236b0-20250529.png)
*図. AndroidビルドにおけるJDKの関係 (Androidの公式ドキュメントより引用)*

そこで本記事では、Android開発におけるJVMバージョンに関する設定の役割・違いを噛み砕いて解説します。

## Androidビルドにおける「2つのJDK」

まず押さえておきたいのは、Androidアプリのビルドにおいて使用されるJDKには、**「Gradleの実行に使用されるJDK」と「Java/Kotlinコードのコンパイルに使用されるJDK」の2種類が存在する**という点です。
再びAndroid公式ドキュメントの図を参照すると、青枠で囲まれた部分が前者 (Gradle実行用のJDK)、赤枠が後者 (コンパイル用のJDK) に対応しています。

![](https://storage.googleapis.com/zenn-user-upload/0b3fe5470e87-20250529.png)
*図. 2つのJDK*

この2つのJDKは、**基本的には独立している**と考えて構いません。たとえば、Gradleの実行にはJDK 17を使用しつつ、Java/KotlinコードのコンパイルにはJDK 11を使う、といった構成も一般的です。

ただし、上の図にもあるように、Toolchain JDKにはGradle JDKがデフォルトで用いられます。つまり、**Toolchainで明示的に設定しない限り、Gradle用のJDKがコンパイル用のJDKとしても使われる**という依存関係が実際には存在します。これが混乱を招く一つの要因であると思うのですが、後述のToolchainに関するセクションで詳しく解説します。

---

2つのJDKのうち、より単純と思われる「Gradle実行用のJDK」の解説から始めます。

## Gradle実行用のJDKのバージョン設定

「Gradle実行用のJDK」とは、Androidアプリのビルド、テスト、Lintの実行、端末やエミュレータへのインストールなど、さまざまなGradleタスクを実行する際に使われるJDKのことを指します。つまり、GradleやGradle PluginといったビルドツールもJVM言語 (Java, Kotlin, Groovy) で書かれているため、その実行に用いられるJDKです。

Gradleタスクの実行方法には、主に以下の2つがあります。
- **Android Studio (IDE) 上での実行**
- **ターミナル (CLI) からの実行**

以下、それぞれの場合に使用されるJDKがどのように決定されるかを説明します。

#### IDE （Android Studio） で実行する場合

Android Studioの設定画面 (`Settings > Build, Execution, Deployment > Build Tools > Gradle`)にて指定された「Gradle JDK」が使用されます。
^[Gradle JDK configuration in Android Studio: https://developer.android.com/build/jdks#jdk-config-in-studio]

#### CLI （ターミナル） から実行する場合

CLIでGradleタスクを実行する場合、以下の優先順位でJDKが選ばれます：

1. `gradle.properties`に`org.gradle.java.home=/path/to/jdk`が設定されている場合、そのJDKが使用されます。
2. 1.が設定されていない場合、環境変数`JAVA_HOME`に指定されたJDKが使用されます。
3. `JAVA_HOME`も未設定の場合、`java`コマンドのパス (`which java`で確認可能) が使われます。

Andrid公式ドキュメントでは、少なくとも`JAVA_HOME`の設定を行うことが推奨されています。
^[How do I choose which JDK runs my Gradle builds?: https://developer.android.com/build/jdks#jdk-gradle]

---

続いて、2つ目のJDKである「Java/Kotlinコードのコンパイル用のJDK」について解説します。

## Java/Kotlinコードのコンパイル用のJDKのバージョン設定

「Java/Kotlinコードのコンパイルに使用されるJDK」とは、AndroidアプリのJava/Kotlinコードのコンパイルに使用されるJDKです。

このJDKのバージョン設定を理解する上で、一旦Toolchainに関しては脇に置き、`sourceCompatibility`, `targetCompatibility`, `jvmTarget`の3つの設定について説明します。これらは、Androidアプリの`build.gradle(.kts)`で以下のように設定されます。

```build.gradle.kts
android {
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    kotlinOptions {
        jvmTarget = "11"
    }
}
``` 

#### sourceCompatibility

`sourceCompatibility`は、**Javaコード中で、どのバージョンのJavaの構文を使用可能か**を決めます^[sourceCompatibility: https://developer.android.com/reference/tools/gradle-api/8.3/null/com/android/build/api/dsl/CompileOptions#sourceCompatibility(kotlin.Any)]。例えば、`sourceCompatibility = JavaVersion.VERSION_17`とされている場合、Java 17までの構文が利用可能です (ただし、Androidにおいては、compileSdkによっても利用可能なJavaライブラリが変わるため、一概にそうとは言えません)。この設定は、Kotlinコードには関係しません。

`sourceCompatibility`は、実際にコンパイルに使用されるJDKのバージョンを決定しません。ただ、`sourceCompatibility ≦ 実際にコンパイルに使用されるJDKのバージョン`である必要があります。

#### targetCompatibility

`targetCompatibility`は、**Javaコードのコンパイル後の、Javaバイトコードの形式のバージョン**を決めます^[targetCompatibility: https://developer.android.com/reference/tools/gradle-api/8.3/null/com/android/build/api/dsl/CompileOptions#targetCompatibility(kotlin.Any)]。
Androidにおいては、Javaバイトコードは一般的なJVMで実行される訳ではなく、あらかじめD8^[D8: https://developer.android.com/tools/d8]によってDEXバイトコードへとAhead-of-timeコンパイルされた上で、Android Runtime (ART) 上で実行されます。そのため、targetCompatibilityはD8と互換性のある値に設定する必要があります。
D8はAndroid Gradle Plugin (AGP) に内包されるため、AGPのバージョンに応じて利用可能なtargetCompatibilityが決まります。例えば、AGP 7.0ではtargetCompatibilityに`JavaVersion.VERSION_11`を設定することが可能になりました ^[AGP 7.0 Release note: https://developer.android.com/build/releases/past-releases/agp-7-0-0-release-notes#java-11]。

`sourceCompatibility`と同様、`targetCompatibility`も、実際にコンパイルに使用されるJDKのバージョンを決定しません。ただし、`sourceCompatibility ≦ targetCompatibility (≦ 実際にコンパイルに使用されるJDKのバージョン)`である必要があります。これは、新しいJavaの構文を使用したコードを、それよりも古い形式のバイトコードへと変換することはできないためです。

#### jvmTarget

`jvmTarget`は、**Kotlinコードのコンパイル後の、Javaバイトコードの形式のバージョン**を決めます。つまり`targetCompatibility`のKotlin版です。
こちらも、実際にコンパイルに使用されるJDKのバージョンを決定しませんが、`jvmTarget ≦ 実際にコンパイルに使用されるJDKのバージョン`である必要はあります。

---

基本的には、`sourceCompatibility`・`targetCompatibility`・`jvmTarget`は同一のバージョンに設定することが一般的です^[Which Java binary features can be used when I compile my Kotlin or Java source?: https://developer.android.com/build/jdks#jdk-gradle]。

--- 

さて、これら3つの設定のいずれも、実際にコンパイルに使われるJDKのバージョンを決めるものではありませんでした。

GradleにToolchainが追加されたのはバージョン6.7.1でしたが^[Toolchain support for JVM projects: https://docs.gradle.org/6.7.1/release-notes.html#jvm-toolchains]、**Toolchainが導入される以前は「実際にコンパイルに使われるJDKのバージョン」として「Gradle実行用のJDKのバージョン設定」が使われていた**ことになります。つまり、本記事のはじめに、これら2つのJDKは機能上は独立していると述べましたが、実際には同一のJDKを使わざるを得ない状態でした。

この問題は、Toolchainの導入によって解決されます。

## Toolchainとは何か

Toolchainには複数の役割がありますが、Androidビルドにおいては以下の3つの役割が重要です。

1. `sourceCompatibility`・`targetCompatibility`・`jvmTarget`のデフォルト値を提供する。
2. Java/Kotlinコードのコンパイルに使用されるJDKのバージョンを明示的に指定する。
3. コンパイルに使用するJDKをローカル環境から自動で検出し、必要に応じてインストールする。

#### 1. `sourceCompatibility`・`targetCompatibility`・`jvmTarget`のデフォルト値を提供

Toolchainを利用することで、`sourceCompatibility`・`targetCompatibility`・`jvmTarget`のデフォルト値を設定できます。これにより、個別に指定する必要がなくなります。
例えば、`build.gradle.kts`において、以下のように置き換えることが可能です。

```kt
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(11)
    }
}

//従来の設定
//android {
//    compileOptions {
//        sourceCompatibility = JavaVersion.VERSION_11
//        targetCompatibility = JavaVersion.VERSION_11
//    }
//    kotlinOptions {
//        jvmTarget = "11"
//    }
//}
```

#### 2. Java/Kotlinコードのコンパイルに利用されるJDKのバージョンの決定

`sourceCompatibility`・`targetCompatibility`・`jvmTarget`は、いずれも実際にJava/Kotlinコードのコンパイルに使用されるJDKバージョンを指定するものではありませんでしたが、Toolchainによって明示的に指定することが可能となりました。

例えば、従来は`targetCompatibility = JavaVersion.VERSION_11`としてもJDK 11がコンパイルに使用されるとは限りませんでした (ただしJDK 11以上である必要はある)。
一方、Toolchainを使用すると、`languageVersion = JavaLanguageVersion.of(11)`のように記述することで、コンパイルに使用されるJDKのバージョンを確実に指定可能になります。
これにより、Gradle自体はJDK 17で実行しつつ、コンパイルにはJDK 11を使用するといった構成も簡単に実現できます。

#### 3. Java/Kotlinコードのコンパイルに利用するためのJDKの解決・インストール

従来は「コンパイル用のJDK」として、JAVA_HOMEやIDEで指定されたパスのJDKが利用されていました。では、Toolchainを利用する場合においては、どのようにJDKのパスが決定されるのでしょうか。

まず、**Toolchainには、ローカルマシン上にインストールされたJDKを自動的に探して選択する仕組み (Auto-detection) があります**^[Auto-detection of installed toolchains: https://docs.gradle.org/current/userguide/toolchains.html#sec:auto_detection]。
Toolchainによって自動検出されるJDKは、`./gradlew -q javaToolchains`コマンドによって一覧表示することが可能です^[Viewing and debugging toolchains: https://docs.gradle.org/current/userguide/toolchains.html#sub:viewing_toolchains]。

さらに、ローカルマシン上に合致するバージョンのJDKが存在しない場合に、自動インストール (Auto-provisioning) させることも可能です^[Auto-provisioning: https://docs.gradle.org/current/userguide/toolchains.html#sec:provisioning]。ただし、Auto-provisioningを有効にするには、Foojay Toolchains Plugin^[Foojay Toolchains Plugin: https://github.com/gradle/foojay-toolchains]のようなToolchain Resolver Plugins^[Toolchain Resolver Plugins: https://docs.gradle.org/current/userguide/toolchain_plugins.html]を導入する必要があります。

---

以上、JDKのバージョンに関する主要な設定――Toolchain、Target Compatibility、Source Compatibility、Kotlin  jvmTarget、JAVA_HOMEについて、一通り解説しました。

本記事の内容に関して、ご不明点やご指摘がありましたら、ぜひコメントにてお知らせください。

## 本記事のCHANGELOG

- 2025/5/30: 初稿執筆・公開
