---
title: "Gradleでマルチプロジェクトを作成する"
emoji: "🐘"
type: "tech"
topics: ["gradle"]
published: false
---

# はじめに

モノリポ構成でGradleプロジェクトを作成したい機会があり、その実現方法を調べました。
本稿ではGradleマルチプロジェクトを作成する方法をご紹介します。

## multi-project と composite build

Gradleでは、ソフトウェアの構造を分割する仕組みとして `multi-project` と `composite build` の2種類の方法があります。

- multi-project
  - 1つのソフトウェアコンポーネントで構成され、内部的に分割する方法
  - ルートプロジェクトと複数のサブプロジェクトで構成される
- composite build
  - 複数のソフトウェアコンポーネントで構成する方法
  - 各コンポーネントで別々にビルドされる

違いについては、実際のコードを読み進めてもらったほうがわかりやすいと思います。
なお、今回は `multi-project` のみ紹介をし、`composite build` については別の記事で紹介する予定です。

# 実際に試してみる

## 目標

今回はSpring Bootアプリケーションとドメインモデルを分割したプロジェクトを構成したいと思います。
大まかな構成は以下の通りです。`app` は `product` と `user` に依存している想定です。

```bash
root
├─ app # Spring Bootアプリケーション
└─ domain
    ├─ product # 商品を表すドメインモデル
    └─ user # ユーザーを表すドメインモデル
```

## ルートプロジェクトを作成する

rootディレクトリとなる `gradle-multi-project` を作成し、その中で `gradle init` を実行しbasicでベースを作成します。
`build.gradle.kts` は不要なので削除しておきます。

```
gradle-multi-project
└─ settings.gradle.kts
```

## サブプロジェクトを追加する

### Spring Bootサブプロジェクトの追加

[Spring Initializr](https://start.spring.io/) を使ってSpring Bootアプリケーションを作成します。
作成したプロジェクト内の `settings.gradle.kts` と `gradlew` と `gradle` ディレクトリは不要なので削除しておきます。
そしてルートプロジェクトに配置しておきます。

```
gradle-multi-project
├─ app
│   ├── build.gradle.kts
│   └── src
└─ settings.gradle.kts
```

ルートディレクトリの `settings.gradle.kts` に `include` を使って `app` をサブプロジェクトとして追加します。

```kotlin:settings.gradle.kts
rootProject.name = "gradle-multi-project"
include("app")
```

これでサブプロジェクトと認識されたため、サブプロジェクトのタスクを実行してみます。
ルートプロジェクトから `gradle :{サブプロジェクト名}:{タスク名}` の形で実行できます。

```bash
$ ./gradlew :app:bootRun

> Task :app:bootRun

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.6.3)

BUILD SUCCESSFUL in 7s
4 actionable tasks: 1 executed, 3 up-to-date
```

### ドメインモデルのサブプロジェクトの追加

同様にドメインモデルのサブプロジェクトも追加していきます。
ディレクトリの作成と `build.gradle.kts` を作成します。

```
gradle-multi-project
├─ app
│   ├── build.gradle.kts
│   └── src
├── domain
│   ├── product
│   │   ├── build.gradle.kts
│   │   └── src
│   └── user
│       ├── build.gradle.kts
│       └── src
└─ settings.gradle.kts
```

`build.gradle.kts` の中身はKotlinコードで書く予定なので、pluginsに追加しておきます。

```kotlin:domain/product/build.gradle.kts
plugins {
    kotlin("jvm") version "1.6.10"
}

repositories {
    mavenCentral()
}
```

ルートディレクトリの `settings.gradle.kts` に `include` を追加します。

```kotlin:settings.gradle.kts
rootProject.name = "gradle-multi-project"
include("app")
include(":domain:product")
include(":domain:user")
```

これでドメインモデルもサブプロジェクトとして追加できました。

## 依存関係を解決する

次に、 `product` と `user` のコードを `app` で使いたいため、依存関係を追加していきます。
使いたい `product` のコードを追加します。

```kotlin:domain/product/src/main/kotlin/com/example/domain/Product.kt
package com.example.domain

data class Product(
    val id: Long,
    val name: String
)
```

`app` の `build.gradle.kts` 依存関係を追加します。

```kotlin:app/build.gradle.kts
dependencies {
    implementation(project(":domain:product"))
}
```

これで使えるようになったため、あとはimportして普通に使うだけです。

```kotlin:app/src/kotlin/com/example/app/AppApplication.kt
fun main(args: Array<String>) {
    println(Product(1, "name"))
    runApplication<AppApplication>(*args)
}
```

## build.gradle.kts を共通化する

すべてのプロジェクトでJUnitを使いたいなど、共通のライブラリなどを使用したい場合はそれぞれの `build.gradle.kts` に追加するのは面倒です。
Gradleには共通化する仕組みも用意されています。

まずルートディレクトリに `buildSrc` ディレクトリを追加します。
このディレクトリはGradleにとって特別なディレクトリであり、ディレクトリを作成するだけでビルド対象に含まれます。
そのため、 `buildSrc` 以外の名前に変更することはできません。

`build.gradle.kts` と、共通化したいビルドロジックを記載する `com.example.kotlin-base.gradle.kts` を作成します。
後者のpluginsでは `kotlin("jvm") version "1.6.10"` のようにバージョン指定はできません。
その代わりに `build.gradle.kts` のdependenciesで使用するプラグインとそのバージョンを指定します。

今回はKotlinを使うための設定と、JUnitの依存関係を追加しておきました。

```kotlin:buildSrc/build.gradle.kts
plugins {
    `kotlin-dsl`
}

repositories {
    gradlePluginPortal()
}

dependencies {
    implementation("org.jetbrains.kotlin.jvm:org.jetbrains.kotlin.jvm.gradle.plugin:1.6.10")
}
```

```kotlin:buildSrc/src/main/kotlin/com.example.kotlin-base.gradle.kts
plugins {
    kotlin("jvm")
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:5.8.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

作成したビルドロジックを使うためには、以下のようにファイル名の `.gradle.kts` を除いた部分を指定します。
ビルドロジックに書いた内容が `build.gradle.kts` に追加されているようなイメージです。

```kotlin:domain/product/build.gradle.kts
plugins {
    id("com.example.kotlin-base")
}
```

各プロジェクトの `build.gradle.kts` で同じような内容を記載する必要がなくなりました。

# おわりに

今回はGradleでマルチプロジェクトを実現する方法を紹介しました。
次回はもう1つの方法として、composite buildについて紹介する予定です。

使用したコードはこちらに置いておきます。
https://github.com/boronngo/gradle-multi-project

# 参考

https://docs.gradle.org/7.3.3/userguide/multi_project_builds.html
