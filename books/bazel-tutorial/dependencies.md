# 依存関係の解決

アプリケーションを開発する際に外部ライブラリを利用するケースは頻繁に発生します。Goの標準では、外部ライブラリへの依存はGo Modulesによって管理され、その情報は `go.mod` ファイルに記録されていきます。

また、リポジトリの内部でいくつかの処理をpackageとして切り出して管理することもあるでしょう。例えば `github.com/user/repo` というGoのmodule内に `github.com/user/repo/pkg/handler` というpackageを作るようなイメージです。同じmodule内のpackageは `go.mod` などに手を加えずとも、パスを指定してimportするだけで利用可能ですね。

こういった依存関係の解決はbazelの中でも必要です。前述したGo Modulesのような仕組みで管理されているのだから、bazelもそれを参照すればいいんじゃないかと思うかもしれませんが、bazelの中でも同様に依存関係のツリーを構築する必要があります。ただし、手動で行うのではなくGoの場合は [gazelle](https://github.com/bazelbuild/bazel-gazelle) というツールが用意されています。

gazelleの説明に入る前に、これまでの説明がどういうことなのか動かして確認してみましょう。

## module内部でpackageを分割したアプリケーションのビルド

次のような構成のGoプロジェクトを用意しました。このコードは[こちら](https://github.com/tokibi/bazel-tutorial/tree/master/dependencies/internal)で公開しています。

```
.
├── WORKSPACE
├── cmd
│   └── hello
│       ├── BUILD.bazel
│       └── main.go
├── go.mod
└── pkg
    └── common
        └── common.go
```

main関数は `cmd/hello/main.go` にあります。ここでは同じmodule内にある `common` packageをimportして、その中で定義された `HelloString` 関数を呼び出しています。実際こんなコードを書くことはないでしょうが、あくまでもサンプルなので気にしないでください。

```go
package main

import (
	"fmt"

	"github.com/tokibi/bazel-tutorial/dependencies/internal/pkg/common"
)

func main() {
	fmt.Println(common.HelloString())
}
```

このファイルをgoコマンドに渡して実行すると正常に動きます。

```console
$ go run ./cmd/hello/main.go
hello
```

では、bazelではどうでしょうか。これまでにも利用した `go_binary` ルールを利用して、ソースコードから実行可能バイナリを生成してみましょう。`cmd/hello/BUILD.bazel` は以下のようになっています。

```python
load("@io_bazel_rules_go//go:def.bzl", "go_binary")

go_binary(
    name = "hello",
    srcs = ["main.go"],
)
```

この `:hello` targetを指定して実行するとエラーになります。

```console
$ bazel run //cmd/hello:hello
...
compilepkg: missing strict dependencies:
	/private/var/tmp/_bazel_ry-tamura/ff6ffe544ce884d759cd518fc3a437ce/sandbox/darwin-sandbox/3/execroot/__main__/cmd/hello/main.go: import of "github.com/tokibi/bazel-tutorial/4-deps/pkg/common"
No dependencies were provided.
Check that imports in Go sources match importpath attributes in deps.
Target //cmd/hello:hello failed to build
...
```

一部抜粋したエラーメッセージを見ると、依存関係の解決に失敗しているようです。これはGoのソースコード上で行われている `pkg/common` のimportにあたる依存関係の解決が、bazelのworkspace内のどこにも記述されていないために発生してます。

この問題を解決するためには、rules_goの `go_library` ルールを利用して依存関係をbazelのworkspace内で明示してあげる必要があります。ですが、既にGoのソースコード内に記述されている依存関係を、またbazelに違う記法で手書きするなんてやってられませんよね。そこでgazelleを利用します。

## gazelle

[gazelle](https://github.com/bazelbuild/bazel-gazelle) はworkspace内のソースコードなどを元に、必要なBUILDファイルを新規作成したり、既存のBUILDファイルを更新してくれるジェネレーターです。ネイティブでGoとprotobufに対応しています。

gazelleの実行には、bazelを通して実行する方法と、コマンドラインツールとしてインストールして実行する方法の2種類が存在します。以降の説明では、前者のbazelを通して実行する方法を採用しています。

### 実行サンプル

初めにWORKSPACEファイル内でgazelleのダウンロードを行うように追記します。既存のrules_goの初期化処理の下に追記しています。

```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "io_bazel_rules_go",
    sha256 = "69de5c704a05ff37862f7e0f5534d4f479418afc21806c887db544a316f3cb6b",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/rules_go/releases/download/v0.27.0/rules_go-v0.27.0.tar.gz",
        "https://github.com/bazelbuild/rules_go/releases/download/v0.27.0/rules_go-v0.27.0.tar.gz",
    ],
)

load("@io_bazel_rules_go//go:deps.bzl", "go_register_toolchains", "go_rules_dependencies")

go_rules_dependencies()

go_register_toolchains(version = "1.16")

# gazelleのダウンロード
http_archive(
    name = "bazel_gazelle",
    sha256 = "62ca106be173579c0a167deb23358fdfe71ffa1e4cfdddf5582af26520f1c66f",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.23.0/bazel-gazelle-v0.23.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.23.0/bazel-gazelle-v0.23.0.tar.gz",
    ],
)

load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")

# gazelleの初期化
gazelle_dependencies()
```

また、workspaceのルートに次のようなBUILDファイルを作成しています。コメントの部分ですが、Goプロジェクトの場合は `# gazelle:prefix` 以降の文字列をリポジトリに対応するインポートパスに置き換える必要があります。ほとんどの場合は `go.mod` ファイルの1行目に書かれているmoduleのパスと同じになると思います。

```python
load("@bazel_gazelle//:def.bzl", "gazelle")

# gazelle:prefix github.com/tokibi/bazel-tutorial/dependencies/internal
gazelle(name = "gazelle")
```

これでgazelleが実行できるようになりました。gazelleを使ってBUILDファイルを実際に生成する前に、これまでに手動で作成したBUILDファイルが、gazelleによってどう変更されるかを確認してみましょう。以下のコマンドで確認できます。

```console
$ bazel run //:gazelle -- update -mode diff
```

> 補足: `--` 以降の文字列はサブコマンドとしてgazelleにそのまま渡されます。そのため上記コマンドは、bazelを通して `gazelle update -mode diff` を実行していることになります。`--` 以降が存在しない場合はgazelle実行時のデフォルトサブコマンドである `gazelle update` が実行されます。

出力は次のようになります。

```diff
--- cmd/hello/BUILD.bazel	1970-01-01 00:00:00.000000000 +0000
+++ cmd/hello/BUILD.bazel	1970-01-01 00:00:00.000000000 +0000
@@ -1,7 +1,16 @@
-load("@io_bazel_rules_go//go:def.bzl", "go_binary")
+load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")

 go_binary(
     name = "hello",
-    srcs = ["main.go"],
+    embed = [":hello_lib"],
+    visibility = ["//visibility:public"],
 )

+go_library(
+    name = "hello_lib",
+    srcs = ["main.go"],
+    importpath = "github.com/tokibi/bazel-tutorial/4-deps/cmd/hello",
+    visibility = ["//visibility:private"],
+    deps = ["//pkg/common"],
+)
+
--- /dev/null	1970-01-01 00:00:00.000000000 +0000
+++ pkg/common/BUILD.bazel	1970-01-01 00:00:00.000000000 +0000
@@ -0,0 +1,9 @@
+load("@io_bazel_rules_go//go:def.bzl", "go_library")
+
+go_library(
+    name = "common",
+    srcs = ["common.go"],
+    importpath = "github.com/tokibi/bazel-tutorial/4-deps/pkg/common",
+    visibility = ["//visibility:public"],
+)
+
```

これまで `cmd/hello/BUILD.bazel` には `go_binary` ルールを使ったtargetしかありませんでしたが、新たに `go_library` ルールのtargetが追加されました。また、`pkg/common` にもBUILDファイルが新規生成されており、このpackageが `:common` targetでライブラリとしてworkspace内に公開されていることがわかります。

`:hello_lib` targetの `deps` 属性を見ると、新規作成された `//pkg/common:common` targetが指定されていますね。これで、Goのソースコード内のimport文に相当する処理が、bazelのworkspace内でも行われるようになりました。変更内容の確認ができたところで、実際にこれらの変更をファイルに反映して、先ほど失敗した `:hello` targetを再度実行してみましょう。

```console
$ bazel run //:gazelle
...

$ bazel run //cmd/hello:hello
INFO: Analyzed target //cmd/hello:hello (2 packages loaded, 170 targets configured).
INFO: Found 1 target...
Target //cmd/hello:hello up-to-date:
  bazel-bin/cmd/hello/hello_/hello
INFO: Elapsed time: 1.011s, Critical Path: 0.76s
INFO: 6 processes: 3 internal, 3 darwin-sandbox.
INFO: Build completed successfully, 6 total actions
INFO: Build completed successfully, 6 total actions
hello
```

今度は正常に実行できました。Goアプリケーションのリポジトリでpackageを分割した場合には、gazelleを実行しておけば依存関係を正しく解決してくれることがわかります。

## 外部ライブラリを利用するアプリケーションのビルド

次に外部ライブラリを利用するGoアプリケーションをビルドするケースを見ていきましょう。今回は [labstack/echo](https://github.com/labstack/echo) を使ったHTTPサーバーをサンプルとして利用します。サンプルは[こちら](https://github.com/tokibi/bazel-tutorial/tree/master/dependencies/external)で公開しています。

ディレクトリの構成は次のようになっています。
```
.
├── BUILD.bazel
├── WORKSPACE
├── cmd
│   └── echo
│       ├── BUILD.bazel
│       └── main.go
├── go.mod
└── go.sum
```

WORKSPACEファイルでは、`rules_go` と `gazelle` のダウンロードおよび初期化処理行っており、内容は[先ほど](#実行サンプル)と同じなので省略します。`cmd/echo/main.go` は公式のサンプルをそのまま使用しています。

```go
package main

import (
  "net/http"
  "github.com/labstack/echo/v4"
  "github.com/labstack/echo/v4/middleware"
)

func main() {
  // Echo instance
  e := echo.New()

  // Middleware
  e.Use(middleware.Logger())
  e.Use(middleware.Recover())

  // Routes
  e.GET("/", hello)

  // Start server
  e.Logger.Fatal(e.Start(":1323"))
}

// Handler
func hello(c echo.Context) error {
  return c.String(http.StatusOK, "Hello, World!")
}
```

まず普通に実行してみましょう。

```console
$ go mod init
$ go get github.com/labstack/echo/v4
$ go run ./cmd/echo/main.go
go run ./cmd/echo/main.go

   ____    __
  / __/___/ /  ___
 / _// __/ _ \/ _ \
/___/\__/_//_/\___/ v4.2.2
High performance, minimalist Go web framework
https://echo.labstack.com
____________________________________O/_______
                                    O\
⇨ http server started on [::]:1323
```

正常に起動することが確認できたら、次はbazelで実行していきます。現在の `cmd/echo/BUILD.bazel` の内容は以下のようになっていますが、このまま `bazel run //cmd/echo:echo` を実行しても先ほどと同様に失敗します。

```
load("@io_bazel_rules_go//go:def.bzl", "go_binary")

go_binary(
    name = "echo",
    srcs = ["main.go"],
)
```

gazelleを利用して正しく実行できる状態にしましょう。まず、gazelleを使ってBUILDファイルの内容を更新します。

```console
$ bazel run //:gazelle
```

gazelle実行前後の差分は以下のようになっています。

```diff
--- cmd/echo/BUILD.bazel	1970-01-01 00:00:00.000000000 +0000
+++ cmd/echo/BUILD.bazel	1970-01-01 00:00:00.000000000 +0000
@@ -1,7 +1,19 @@
-load("@io_bazel_rules_go//go:def.bzl", "go_binary")
+load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")

 go_binary(
     name = "echo",
-    srcs = ["main.go"],
+    embed = [":echo_lib"],
+    visibility = ["//:__subpackages__"],
 )

+go_library(
+    name = "echo_lib",
+    srcs = ["main.go"],
+    importpath = "github.com/tokibi/bazel-tutorial/dependencies/internal/cmd/echo",
+    visibility = ["//visibility:private"],
+    deps = [
+        "@com_github_labstack_echo_v4//:go_default_library",
+        "@com_github_labstack_echo_v4//middleware:go_default_library",
+    ],
+)
+
```

これで実行可能になっているでしょうか？試してみると、これだけでは動かないことがわかります。

```console
$ bazel run //cmd/echo:echo
ERROR: /Users/ry-tamura/src/github.com/tokibi/bazel-tutorial/dependencies/external/cmd/echo/BUILD.bazel:9:11: no such package '@com_github_labstack_echo_v4//': The repository '@com_github_labstack_echo_v4' could not be resolved and referenced by '//cmd/echo:echo_lib'
...
```

理由は `:echo_lib` targetの `deps` 属性に指定されている以下2つのtargetがworkspace内に存在しないからです。

- `@com_github_labstack_echo_v4//:go_default_library`
- `@com_github_labstack_echo_v4//middleware:go_default_library`

外部ライブラリを利用する場合は、追加で以下のコマンドを実行する必要があります。

```console
$ bazel run //:gazelle -- update-repos -from_file=go.mod
```

実行すると、WORKSPACEファイルの内容が更新されます(長いので後半は省略)。gazelleの [update-repos](https://github.com/bazelbuild/bazel-gazelle#update-repos) コマンドは、外部ライブラリをworkspaceに取り込むための `go_repository` ルールを自動的に生成してくれます。今回は `go.mod` ファイルをインポートすることで、必要なライブラリが何かをgazelleに伝えています。

```diff
--- a/dependencies/external/WORKSPACE
+++ b/dependencies/external/WORKSPACE
@@ -24,6 +24,139 @@
-load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
+load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies", "go_repository")
+
+go_repository(
+    name = "com_github_davecgh_go_spew",
+    importpath = "github.com/davecgh/go-spew",
+    sum = "h1:ZDRjVQ15GmhC3fiQ8ni8+OwkZQO4DARzQgrnXU1Liz8=",
+    version = "v1.1.0",
+)
+
+go_repository(
+    name = "com_github_dgrijalva_jwt_go",
+    importpath = "github.com/dgrijalva/jwt-go",
+    sum = "h1:7qlOGliEKZXTDg6OTjfoBKDXWrumCAMpl/TFQ4/5kLM=",
+    version = "v3.2.0+incompatible",
+)
+
+go_repository(
+    name = "com_github_labstack_echo_v4",
+    importpath = "github.com/labstack/echo/v4",
+    sum = "h1:bq2fdZCionY1jck8rzUpQEu2YSmI8QbX6LHrCa60IVs=",
+    version = "v4.2.2",
+)
+
...
```

もし、WORKSPACEファイル内にずらずらとこのようなルールが追加されるのが嫌だという方は、`update-repos` コマンドの `-to_macro` オプションを指定してください。

```
$ bazel run //:gazelle -- update-repos -from_file=go.mod -to_macro=deps.bzl%go_dependencies
```

こうすることで、`go_repository` ルールは新規作成される `deps.bzl` ファイル中に全て記載されます。この場合WORKSPACEファイルの変更は初期化処理の追記のみになります。

```diff
--- a/dependencies/external/WORKSPACE
+++ b/dependencies/external/WORKSPACE
@@ -25,5 +25,9 @@ http_archive(
 )

 load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
+load("//:deps.bzl", "go_dependencies")
+
+# gazelle:repository_macro deps.bzl%go_dependencies
+go_dependencies()
```

Go Modulesの管理する依存関係を `update-repos` コマンドを使ってworkspace内に取り込むことで、bazelでも外部ライブラリを利用するGoアプリケーションが実行できるようになりました。

```console
$ bazel run //cmd/echo:echo
INFO: Analyzed target //cmd/echo:echo (30 packages loaded, 614 targets configured).
INFO: Found 1 target...
Target //cmd/echo:echo up-to-date:
  bazel-bin/cmd/echo/echo_/echo
INFO: Elapsed time: 29.376s, Critical Path: 3.15s
INFO: 28 processes: 1 internal, 27 darwin-sandbox.
INFO: Build completed successfully, 28 total actions
INFO: Build completed successfully, 28 total actions

   ____    __
  / __/___/ /  ___
 / _// __/ _ \/ _ \
/___/\__/_//_/\___/ v4.2.2
High performance, minimalist Go web framework
https://echo.labstack.com
____________________________________O/_______
                                    O\
⇨ http server started on [::]:1323
```
