---
name: golang-samber-mo
description: "Monadic types for Golang using samber/mo — Option, Result, Either, Future, IO, Task, and State types for type-safe nullable values, error handling, and functional composition with pipeline sub-packages. Apply when using or adopting samber/mo, when the codebase imports `github.com/samber/mo`, or when considering functional programming patterns as a safety design for Golang."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.0.3"
  openclaw:
    emoji: "🎭"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** You are a Go engineer bringing functional programming safety to Go. You use monads to make impossible states unrepresentable — nil checks become type constraints, error handling becomes composable pipelines.

**Thinking mode:** Use `ultrathink` when designing multi-step Option/Result/Either pipelines. Wrong type choice creates unnecessary wrapping/unwrapping that defeats the purpose of monads.

# samber/mo — Go向けモナドと関数型抽象

Go 1.18+ 向けの型安全なモナド型ライブラリ。依存関係ゼロ。Scala、Rust、fp-ts にインスパイア。

**公式リソース:**

- [pkg.go.dev/github.com/samber/mo](https://pkg.go.dev/github.com/samber/mo)
- [github.com/samber/mo](https://github.com/samber/mo)

このスキルは網羅的ではありません。最新のAPIシグネチャや使用方法については、ライブラリの公式ドキュメントとコード例を参照してください。Context7がディスカバリプラットフォームとして役立ちます。

```bash
go get github.com/samber/mo
```

関数型プログラミングの概念とGoにおけるモナドの価値についての入門は [Monads Guide](./references/monads-guide.md) を参照。

## コア型の概要

| 型 | 目的 | イメージ |
| --- | --- | --- |
| `Option[T]` | 存在しない可能性のある値 | Rustの `Option`、Javaの `Optional` |
| `Result[T]` | 失敗する可能性のある操作 | Rustの `Result<T, E>`、`(T, error)` の代替 |
| `Either[L, R]` | 2つの型のいずれかの値 | Scalaの `Either`、TypeScriptの判別共用体 |
| `EitherX[L, R]` | X個の型のいずれかの値 | Scalaの `Either`、TypeScriptの判別共用体 |
| `Future[T]` | まだ利用できない非同期値 | JavaScriptの `Promise` |
| `IO[T]` | 遅延同期副作用 | Haskellの `IO` |
| `Task[T]` | 遅延非同期計算 | fp-tsの `Task` |
| `State[S, A]` | ステートフル計算 | Haskellの `State` モナド |

## Option[T] — nilなしのNull許容値

値が存在する（`Some`）か不在（`None`）かを表す。型レベルでnilポインタのリスクを排除する。

```go
import "github.com/samber/mo"

name := mo.Some("Alice")          // Option[string] with value
empty := mo.None[string]()        // Option[string] without value
fromPtr := mo.PointerToOption(ptr) // nil pointer -> None

// 安全な値の取り出し
name.OrElse("Anonymous")  // "Alice"
empty.OrElse("Anonymous")  // "Anonymous"

// 存在すれば変換、不在ならスキップ
upper := name.Map(func(s string) (string, bool) {
    return strings.ToUpper(s), true
})
```

**主要メソッド:** `Some`, `None`, `Get`, `MustGet`, `OrElse`, `OrEmpty`, `Map`, `FlatMap`, `Match`, `ForEach`, `ToPointer`, `IsPresent`, `IsAbsent`.

Option は `json.Marshaler/Unmarshaler`、`sql.Scanner`、`driver.Valuer` を実装しており、JSON構造体やデータベースモデルで直接使用可能。

完全なAPIリファレンスについては [Option Reference](./references/option.md) を参照。

## Result[T] — 値としてのエラーハンドリング

成功（`Ok`）または失敗（`Err`）を表す。`Either[error, T]` と同等だが、Goのエラーパターンに特化。

```go
// Goの (value, error) パターンをラップ
result := mo.TupleToResult(os.ReadFile("config.yaml"))

// 同じ型の変換 — エラーは自動的にショートサーキットされる
upper := mo.Ok("hello").Map(func(s string) (string, error) {
    return strings.ToUpper(s), nil
})
// Ok("HELLO")

// フォールバック付きで値を取り出す
val := upper.OrElse("default")
```

**Goの制限:** 直接メソッド（`.Map`、`.FlatMap`）は型パラメータを変更できない — `Result[T].Map` は `Result[T]` を返し、`Result[U]` ではない。Goのメソッドは新しい型パラメータを導入できない。型が変わる変換（例: `Result[[]byte]` から `Result[Config]`）にはサブパッケージ関数または `mo.Do` を使う:

```go
import "github.com/samber/mo/result"

// 型が変わるパイプライン: []byte -> Config -> ValidConfig
parsed := result.Pipe2(
    mo.TupleToResult(os.ReadFile("config.yaml")),
    result.Map(func(data []byte) Config { return parseConfig(data) }),
    result.FlatMap(func(cfg Config) mo.Result[ValidConfig] { return validate(cfg) }),
)
```

**主要メソッド:** `Ok`, `Err`, `Errf`, `TupleToResult`, `Try`, `Get`, `MustGet`, `OrElse`, `Map`, `FlatMap`, `MapErr`, `Match`, `ForEach`, `ToEither`, `IsOk`, `IsError`.

完全なAPIリファレンスについては [Result Reference](./references/result.md) を参照。

## Either[L, R] — 2つの型の判別共用体

2つの可能な型のいずれかの値を表す。Resultとは異なり、どちらの側も成功や失敗を意味しない — 両方とも有効な選択肢。

```go
// キャッシュデータまたは新鮮なデータのいずれかを返すAPI
func fetchUser(id string) mo.Either[CachedUser, FreshUser] {
    if cached, ok := cache.Get(id); ok {
        return mo.Left[CachedUser, FreshUser](cached)
    }
    return mo.Right[CachedUser, FreshUser](db.Fetch(id))
}

// パターンマッチ
result.Match(
    func(cached CachedUser) mo.Either[CachedUser, FreshUser] { /* use cached */ },
    func(fresh FreshUser) mo.Either[CachedUser, FreshUser] { /* use fresh */ },
)
```

**Either vs Resultの使い分け:** 一方のパスがエラーの場合は `Result[T]` を使う。両方のパスが有効な選択肢（キャッシュ vs 新鮮、左 vs 右、戦略A vs B）の場合は `Either[L, R]` を使う。

`Either3[T1, T2, T3]`、`Either4`、`Either5` はこれを3-5個の型バリアントに拡張する。

完全なAPIリファレンスについては [Either Reference](./references/either.md) を参照。

## Do記法 — モナドの安全性を持つ命令型スタイル

`mo.Do` は命令型コードを `Result` でラップし、`MustGet()` 呼び出しからのpanicをキャッチする:

```go
result := mo.Do(func() int {
    // MustGet は None/Err でpanicする — Do はそれを Result のエラーとしてキャッチ
    a := mo.Some(21).MustGet()
    b := mo.Ok(2).MustGet()
    return a * b  // 42
})
// result is Ok(42)

result := mo.Do(func() int {
    val := mo.None[int]().MustGet()  // panicする
    return val
})
// result is Err("no such element")
```

Do記法は命令型のGoスタイルとモナドの安全性を橋渡しする — 直線的なコードを書き、自動的なエラー伝播を得る。

## パイプラインサブパッケージ vs 直接チェーン

samber/mo は操作を合成する2つの方法を提供する:

**直接メソッド**（`.Map`、`.FlatMap`）— 出力型が入力型と同じ場合に機能:

```go
opt := mo.Some(42)
doubled := opt.Map(func(v int) (int, bool) {
    return v * 2, true
})  // Option[int]
```

**サブパッケージ関数**（`option.Map`、`result.Map`）— 出力型が入力型と異なる場合に必要:

```go
import "github.com/samber/mo/option"

// int -> string の型変更: サブパッケージの Map を使用
strOpt := option.Map(func(v int) string {
    return fmt.Sprintf("value: %d", v)
})(mo.Some(42))  // Option[string]
```

**Pipe関数**（`option.Pipe3`、`result.Pipe3`）— 複数の型変更変換を読みやすくチェーン:

```go
import "github.com/samber/mo/option"

result := option.Pipe3(
    mo.Some(42),
    option.Map(func(v int) string { return strconv.Itoa(v) }),
    option.Map(func(s string) []byte { return []byte(s) }),
    option.FlatMap(func(b []byte) mo.Option[string] {
        if len(b) > 0 { return mo.Some(string(b)) }
        return mo.None[string]()
    }),
)
```

**経験則:** 同じ型の変換には直接メソッドを使う。ステップ間で型が変わる場合はサブパッケージ関数 + パイプを使う。

詳細なパイプラインAPIリファレンスについては [Pipelines Reference](./references/pipelines.md) を参照。

## よくあるパターン

### Optionを使ったJSON APIレスポンス

```go
type UserResponse struct {
    Name     string            `json:"name"`
    Nickname mo.Option[string] `json:"nickname"`  // omits null gracefully
    Bio      mo.Option[string] `json:"bio"`
}
```

### データベースのNULL許容カラム

```go
type User struct {
    ID       int
    Email    string
    Phone    mo.Option[string]  // implements sql.Scanner + driver.Valuer
}

err := row.Scan(&u.ID, &u.Email, &u.Phone)
```

### 既存のGo APIのラッピング

```go
// マップルックアップをOptionに変換
func MapGet[K comparable, V any](m map[K]V, key K) mo.Option[V] {
    return mo.TupleToOption(m[key])  // m[key] returns (V, bool)
}
```

### Foldによる統一的な値の取り出し

`mo.Fold` は `Foldable` インターフェースを通じてOption、Result、Eitherで統一的に機能する:

```go
str := mo.Fold[error, int, string](
    mo.Ok(42),  // Option、Result、Eitherで動作
    func(v int) string { return fmt.Sprintf("got %d", v) },
    func(err error) string { return "failed" },
)
// "got 42"
```

## ベストプラクティス

1. **`MustGet` より `OrElse` を優先** — `MustGet` は不在/エラー値でpanicする。panicがキャッチされる `mo.Do` ブロック内、または値の存在が確実な場合にのみ使用する
2. **APIバウンダリで `TupleToResult` を使う** — バウンダリでGoの `(T, error)` を `Result[T]` に変換し、ドメインロジック内で `Map`/`FlatMap` でチェーンする
3. **エラーには `Result[T]`、選択肢には `Either[L, R]`** — Resultは成功/失敗に特化。Eitherは2つの有効な型のため
4. **NULL許容フィールドにはOption、ゼロ値には使わない** — `Option[string]` は「不在」と「空文字列」を区別する。空文字列が有効な値の場合は通常の `string` を使う
5. **ネストせずチェーンする** — `result.Map(...).FlatMap(...).OrElse(default)` は左から右に読める。モナドのチェーンの方がクリーンな場合はネストしたif/elseパターンを避ける
6. **複数ステップの型変換にはサブパッケージパイプを使う** — 3つ以上のステップでそれぞれ型が変わる場合、`option.Pipe3(...)` はネストした関数呼び出しより読みやすい

高度な型（Future、IO、Task、State）については [Advanced Types Reference](./references/advanced-types.md) を参照。

samber/mo でバグや予期しない動作に遭遇した場合は、<https://github.com/samber/mo/issues> で Issue を作成してください。

## 相互参照

- -> See `samber/cc-skills-golang@golang-samber-lo` skill for functional collection transforms (Map, Filter, Reduce on slices) that compose with mo types
- -> See `samber/cc-skills-golang@golang-error-handling` skill for idiomatic Go error handling patterns
- -> See `samber/cc-skills-golang@golang-safety` skill for nil-safety and defensive Go coding
- -> See `samber/cc-skills-golang@golang-database` skill for database access patterns
- -> See `samber/cc-skills-golang@golang-design-patterns` skill for functional options and other Go patterns
