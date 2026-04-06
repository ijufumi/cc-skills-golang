---
name: golang-structs-interfaces
description: 'Golang struct and interface design patterns — composition, embedding, type assertions, type switches, interface segregation, dependency injection via interfaces, struct field tags, and pointer vs value receivers. Use this skill when designing Go types, defining or implementing interfaces, embedding structs or interfaces, writing type assertions or type switches, adding struct field tags for JSON/YAML/DB serialization, or choosing between pointer and value receivers. Also use when the user asks about "accept interfaces, return structs", compile-time interface checks, or composing small interfaces into larger ones.'
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "🧩"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a Go type system designer. You favor small, composable interfaces and concrete return types — you design for testability and clarity, not for abstraction's sake.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-structs-interfaces` skill takes precedence.

# Go 構造体 & インターフェース

## インターフェース設計の原則

### インターフェースを小さく保つ

> "The bigger the interface, the weaker the abstraction." — Go Proverbs

インターフェースは1〜3個のメソッドを持つべきです。小さなインターフェースは実装、モック、合成が容易です。より大きな契約が必要な場合は、小さなインターフェースから合成してください:

→ See `samber/cc-skills-golang@golang-naming` skill for interface naming conventions (method + "-er" suffix, canonical names)

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composed from small interfaces
type ReadWriter interface {
    Reader
    Writer
}
```

小さなインターフェースから大きなインターフェースを合成します:

```go
type ReadWriteCloser interface {
    io.Reader
    io.Writer
    io.Closer
}
```

### インターフェースは使用される場所で定義する

インターフェースはコンシューマーに属します。

インターフェースは実装側ではなく、使用される側で定義しなければなりません。これによりコンシューマーが契約を制御でき、インターフェースのためだけにパッケージをインポートすることを避けられます。

```go
// package notification — defines only what it needs
type Sender interface {
    Send(to, body string) error
}

type Service struct {
    sender Sender
}
```

`email` パッケージは具象型の `Client` 構造体をエクスポートします。`Sender` について知る必要はありません。

### インターフェースを受け取り、構造体を返す

関数は柔軟性のためにインターフェースパラメータを受け取り、明確さのために具象型を返すべきです。呼び出し元は返された型のフィールドとメソッドに完全にアクセスでき、上流のコンシューマーは必要に応じて結果をインターフェース変数に代入できます。

```go
// Good — accepts interface, returns concrete
func NewService(store UserStore) *Service { ... }

// BAD — NEVER return interfaces from constructors
func NewService(store UserStore) ServiceInterface { ... }
```

### インターフェースを早期に作成しない

> "Don't design with interfaces, discover them."

インターフェースを早期に作成してはなりません。2つ以上の実装またはテスタビリティの要件が出るまで待ってください。早期のインターフェースは価値のないインダイレクションを追加します。具象型から始め、2番目のコンシューマーやテストモックが必要とした時にインターフェースを抽出してください。

```go
// Bad — premature interface with a single implementation
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
}
type userRepository struct { db *sql.DB }

// Good — start concrete, extract an interface later when needed
type UserRepository struct { db *sql.DB }
```

## ゼロ値を有用にする

明示的な初期化なしで動作するように構造体を設計してください。適切に設計されたゼロ値はコンストラクタのボイラープレートを削減し、nil関連のバグを防ぎます:

```go
// Good — zero value is ready to use
var buf bytes.Buffer
buf.WriteString("hello")

var mu sync.Mutex
mu.Lock()

// Bad — zero value is broken, requires constructor
type Registry struct {
    items map[string]Item // nil map, panics on write
}

// Good — lazy initialization guards the zero value
func (r *Registry) Register(name string, item Item) {
    if r.items == nil {
        r.items = make(map[string]Item)
    }
    r.items[name] = item
}
```

## 特定の型で十分な場合は `any` / `interface{}` を避ける

Go 1.18以降では、型安全な操作にはジェネリクスを `any` より優先しなければなりません。`any` は型が本当に不明な真の境界（例: JSONデコード、リフレクション）でのみ使用してください:

```go
// Bad — loses type safety
func Contains(slice []any, target any) bool { ... }

// Good — generic, type-safe
func Contains[T comparable](slice []T, target T) bool { ... }
```

## 主要な標準ライブラリインターフェース

| インターフェース     | パッケージ         | メソッド                                |
| ------------- | --------------- | ------------------------------------- |
| `Reader`      | `io`            | `Read(p []byte) (n int, err error)`   |
| `Writer`      | `io`            | `Write(p []byte) (n int, err error)`  |
| `Closer`      | `io`            | `Close() error`                       |
| `Stringer`    | `fmt`           | `String() string`                     |
| `error`       | builtin         | `Error() string`                      |
| `Handler`     | `net/http`      | `ServeHTTP(ResponseWriter, *Request)` |
| `Marshaler`   | `encoding/json` | `MarshalJSON() ([]byte, error)`       |
| `Unmarshaler` | `encoding/json` | `UnmarshalJSON([]byte) error`         |

標準的なメソッドシグネチャを尊重しなければなりません。型に `String()` メソッドがある場合、`fmt.Stringer` と一致する必要があります。`ToString()` や `ReadData()` を発明しないでください。

## コンパイル時インターフェースチェック

ブランク識別子への代入でコンパイル時に型がインターフェースを実装しているか検証します。型定義の近くに配置してください:

```go
var _ io.ReadWriter = (*MyBuffer)(nil)
```

実行時コストはゼロです。`MyBuffer` が `io.ReadWriter` を満たさなくなった場合、ビルドが即座に失敗します。

## 型アサーション & 型スイッチ

### 安全な型アサーション

型アサーションはパニックを避けるためにcomma-ok形式を使用しなければなりません:

```go
// Good — safe
s, ok := val.(string)
if !ok {
    // handle
}

// Bad — panics if val is not a string
s := val.(string)
```

### 型スイッチ

インターフェース値の動的な型を判別します:

```go
switch v := val.(type) {
case string:
    fmt.Println(v)
case int:
    fmt.Println(v * 2)
case io.Reader:
    io.Copy(os.Stdout, v)
default:
    fmt.Printf("unexpected type %T\n", v)
}
```

### 型アサーションによるオプション動作

値が追加の機能をサポートしているかを、事前に要求せずに確認します:

```go
type Flusher interface {
    Flush() error
}

func writeData(w io.Writer, data []byte) error {
    if _, err := w.Write(data); err != nil {
        return err
    }
    // Flush only if the writer supports it
    if f, ok := w.(Flusher); ok {
        return f.Flush()
    }
    return nil
}
```

このパターンは標準ライブラリで広く使用されています（例: `http.Flusher`, `io.ReaderFrom`）。

## 構造体 & インターフェースの埋め込み

### 構造体の埋め込み

埋め込みは内部型のメソッドとフィールドを外部型に昇格させます。継承ではなくコンポジションです:

```go
type Logger struct {
    *slog.Logger
}

type Server struct {
    Logger
    addr string
}

// s.Info(...) works — promoted from slog.Logger through Logger
s := Server{Logger: Logger{slog.Default()}, addr: ":8080"}
s.Info("starting", "addr", s.addr)
```

昇格されたメソッドのレシーバーは外部型ではなく_内部_型です。外部型は同名のメソッドを定義することでオーバーライドできます。

### 埋め込み vs 名前付きフィールドの使い分け

| 使用方法 | 使用する場面 |
| --- | --- |
| **埋め込み** | 内部型の完全なAPIを昇格させたい場合 — 外部型は拡張バージョン（"is a"関係） |
| **名前付きフィールド** | 内部型を内部的にのみ必要とする場合 — 外部型は依存関係を持つ（"has a"関係） |

```go
// Embed — Server exposes all http.Handler methods
type Server struct {
    http.Handler
}

// Named field — Server uses the store but doesn't expose its methods
type Server struct {
    store *DataStore
}
```

## インターフェースによる依存性注入

コンストラクタでインターフェースとして依存関係を受け取ります。これによりコンポーネントが疎結合になり、テストが容易になります:

```go
type UserStore interface {
    FindByID(ctx context.Context, id string) (*User, error)
}

type UserService struct {
    store UserStore
}

func NewUserService(store UserStore) *UserService {
    return &UserService{store: store}
}
```

テストでは、`UserStore` を満たすモックやスタブを渡します。実際のデータベースは不要です。

## 構造体フィールドタグ

シリアライゼーション制御にフィールドタグを使用します。シリアライズされる構造体のエクスポートされたフィールドにはフィールドタグが必須です:

```go
type Order struct {
    ID        string    `json:"id"         db:"id"`
    UserID    string    `json:"user_id"    db:"user_id"`
    Total     float64   `json:"total"      db:"total"`
    Items     []Item    `json:"items"      db:"-"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
    DeletedAt time.Time `json:"-"          db:"deleted_at"`
    Internal  string    `json:"-"          db:"-"`
}
```

| ディレクティブ               | 意味                                     |
| ----------------------- | ------------------------------------------- |
| `json:"name"`           | JSON出力でのフィールド名                   |
| `json:"name,omitempty"` | ゼロ値の場合フィールドを省略                    |
| `json:"-"`              | 常にJSONから除外                    |
| `json:",string"`        | 数値/boolをJSON文字列としてエンコード           |
| `db:"column"`           | データベースカラムマッピング（sqlx等）        |
| `yaml:"name"`           | YAMLフィールド名                             |
| `xml:"name,attr"`       | XML属性                               |
| `validate:"required"`   | 構造体バリデーション（go-playground/validator） |

## ポインタレシーバー vs 値レシーバー

| ポインタ `(s *Server)` を使う場合 | 値 `(s Server)` を使う場合 |
| --- | --- |
| メソッドがレシーバーを変更する | レシーバーが小さく不変である |
| レシーバーに `sync.Mutex` などが含まれる | レシーバーが基本型（int, string）である |
| レシーバーが大きな構造体である | メソッドが読み取り専用のアクセサである |
| 一貫性: いずれかのメソッドがポインタを使用する場合、すべてそうすべき | mapと関数の値（既に参照型） |

レシーバー型は型のすべてのメソッドで一貫していなければなりません。いずれかのメソッドがポインタレシーバーを使用する場合、すべてのメソッドがそうすべきです。

## `noCopy` による構造体コピーの防止

一部の構造体は最初の使用後にコピーしてはなりません（例: mutex、チャネル、内部ポインタを含むもの）。`noCopy` センチネルを埋め込んで `go vet` が偶発的なコピーを検出するようにします:

```go
// noCopy may be added to structs which must not be copied after first use.
// See https://pkg.go.dev/sync#noCopy
type noCopy struct{}

func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}

type ConnPool struct {
    noCopy noCopy
    mu     sync.Mutex
    conns  []*Conn
}
```

`go vet` は `ConnPool` の値がコピーされた場合（値渡し、代入など）にエラーを報告します。これは標準ライブラリが `sync.WaitGroup`、`sync.Mutex`、`strings.Builder` などで使用しているのと同じ手法です。

これらの構造体は常にポインタで渡してください:

```go
// Good
func process(pool *ConnPool) { ... }

// Bad — go vet will flag this
func process(pool ConnPool) { ... }
```

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-naming` skill for interface naming conventions (Reader, Closer, Stringer)
- → See `samber/cc-skills-golang@golang-design-patterns` skill for functional options, constructors, and builder patterns
- → See `samber/cc-skills-golang@golang-dependency-injection` skill for DI patterns using interfaces
- → See `samber/cc-skills-golang@golang-code-style` skill for value vs pointer function parameters (distinct from receivers)

## よくある間違い

| 間違い | 修正方法 |
| --- | --- |
| 大きなインターフェース（5+メソッド） | 1〜3メソッドの焦点を絞ったインターフェースに分割し、必要に応じて合成する |
| 実装側パッケージでインターフェースを定義 | 使用される場所で定義する |
| コンストラクタからインターフェースを返す | 具象型を返す |
| comma-okなしの型アサーション | 常に `v, ok := x.(T)` を使用する |
| 数個のメソッドしか必要ないのに埋め込み | 名前付きフィールドを使用し明示的に委譲する |
| シリアライズされる構造体にフィールドタグがない | マーシャルされる型のすべてのエクスポートフィールドにタグを付ける |
| 型のポインタレシーバーと値レシーバーの混在 | 一つを選んで一貫させる |
| コンパイル時インターフェースチェックの忘れ | `var _ Interface = (*Type)(nil)` を追加する |
| `String()` の代わりに `ToString()` を使用 | 標準的なメソッド名を尊重する |
| 単一実装での早期インターフェース | 具象型から始め、必要になった時にインターフェースを抽出する |
| ゼロ値構造体でのnil map/slice | メソッド内で遅延初期化を使用する |
| 型安全な操作に `any` を使用 | 代わりにジェネリクス（`[T comparable]`）を使用する |
