---
name: golang-samber-oops
description: "Structured error handling in Golang with samber/oops — error builders, stack traces, error codes, error context, error wrapping, error attributes, user-facing vs developer messages, panic recovery, and logger integration. Apply when using or adopting samber/oops, or when the codebase already imports github.com/samber/oops."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "💥"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs
---

**Persona:** あなたはエラーを構造化データとして扱うGoエンジニアです。すべてのエラーは、オンコールエンジニアが開発者に確認することなく問題を診断できるだけの十分なコンテキスト（ドメイン、属性、トレース）を持ちます。

# samber/oops 構造化エラーハンドリング

**samber/oops** は、Goの標準エラーハンドリングのドロップイン代替品であり、構造化コンテキスト、スタックトレース、エラーコード、パブリックメッセージ、パニックリカバリーを追加します。変数データはメッセージ文字列ではなく `.With()` 属性に格納されるため、APMツール（Datadog、Loki、Sentry）がエラーを適切にグループ化できます。ログサイトで `slog` 属性を追加するstdlibのアプローチとは異なり、oopsの属性はコールスタックを通じてエラーとともに伝播します。

## samber/oopsを使う理由

標準のGoエラーはコンテキストが不足しています — `connection failed` は見えても、どのユーザーがトリガーしたか、どのクエリが実行中だったか、完全なコールスタックは分かりません。`samber/oops` が提供するもの:

- **構造化コンテキスト** — 任意のエラーへのキーバリュー属性
- **スタックトレース** — 自動コールスタックキャプチャ
- **エラーコード** — マシンが読める識別子
- **パブリックメッセージ** — 技術的詳細とは別のユーザー向けメッセージ
- **低カーディナリティメッセージ** — 変数データはメッセージ文字列ではなく `.With()` 属性に格納し、APMツールがエラーを適切にグループ化できるようにする

このスキルは網羅的ではありません。最新のAPIシグネチャと使用パターンについては、ライブラリの公式ドキュメントとコード例を参照してください。Context7をディスカバリープラットフォームとして活用できます。

## コアパターン: エラービルダーチェーン

すべての `oops` エラーはフルエントビルダーパターンを使用する:

```go
err := oops.
    In("user-service").           // domain/feature
    Tags("database", "postgres").  // categorization
    Code("network_failure").       // machine-readable identifier
    User("user-123", "email", "foo@bar.com").  // user context
    With("query", query).          // custom attributes
    Errorf("failed to fetch user: %s", "timeout")
```

ターミナルメソッド:

- `.Errorf(format, args...)` — 新しいエラーを作成
- `.Wrap(err)` — 既存のエラーをラップ
- `.Wrapf(err, format, args...)` — メッセージ付きでラップ
- `.Join(err1, err2, ...)` — 複数のエラーを結合
- `.Recover(fn)` / `.Recoverf(fn, format, args...)` — パニックをエラーに変換

### エラービルダーメソッド

| メソッド | ユースケース |
| --- | --- |
| `.With("key", value)` | カスタムキーバリュー属性を追加（遅延評価 `func() any` 値もサポート） |
| `.WithContext(ctx, "key1", "key2")` | Go contextから属性として値を抽出（遅延評価値もサポート） |
| `.In("domain")` | 機能/サービス/ドメインを設定 |
| `.Tags("auth", "sql")` | カテゴリ分類タグを追加（`err.HasTag("tag")` でクエリ可能） |
| `.Code("iam_authz_missing_permission")` | マシンが読めるエラー識別子/スラグを設定 |
| `.Public("Could not fetch user.")` | ユーザー向けメッセージを設定（技術的詳細とは別） |
| `.Hint("Runbook: https://doc.acme.org/doc/abcd.md")` | 開発者向けデバッグヒントを追加 |
| `.Owner("team/slack")` | 責任チーム/オーナーを識別 |
| `.User(id, "k", "v")` | ユーザー識別子と属性を追加 |
| `.Tenant(id, "k", "v")` | テナント/組織のコンテキストと属性を追加 |
| `.Trace(id)` | トレース/相関IDを追加（デフォルト: ULID） |
| `.Span(id)` | 作業単位/操作を表すスパンIDを追加（デフォルト: ULID） |
| `.Time(t)` | エラータイムスタンプを上書き（デフォルト: `time.Now()`） |
| `.Since(t)` | `t` からの経過時間に基づいて期間を設定（`err.Duration()` で公開） |
| `.Duration(d)` | 明示的なエラー期間を設定 |
| `.Request(req, includeBody)` | `*http.Request` を添付（オプションでボディを含む） |
| `.Response(res, includeBody)` | `*http.Response` を添付（オプションでボディを含む） |
| `oops.FromContext(ctx)` | Go contextに格納された `OopsErrorBuilder` から開始 |

## よくあるシナリオ

### データベース/リポジトリ層

```go
func (r *UserRepository) FetchUser(id string) (*User, error) {
    query := "SELECT * FROM users WHERE id = $1"
    row, err := r.db.Query(query, id)
    if err != nil {
        return nil, oops.
            In("user-repository").
            Tags("database", "postgres").
            With("query", query).
            With("user_id", id).
            Wrapf(err, "failed to fetch user from database")
    }
    // ...
}
```

### HTTPハンドラ層

```go
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)

    err := h.service.CreateUser(r.Context(), userID)
    if err != nil {
        return oops.
            In("http-handler").
            Tags("endpoint", "/users").
            Request(r, false).
            User(userID).
            Wrapf(err, "create user failed")
    }

    w.WriteHeader(http.StatusCreated)
}
```

### サービス層: 再利用可能なビルダー

```go
func (s *UserService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    builder := oops.
        In("order-service").
        Tags("orders", "checkout").
        Tenant(req.TenantID, "plan", req.Plan).
        User(req.UserID, "email", req.UserEmail)

    product, err := s.catalog.GetProduct(ctx, req.ProductID)
    if err != nil {
        return builder.
            With("product_id", req.ProductID).
            Wrapf(err, "product lookup failed")
    }

    if product.Stock < req.Quantity {
        return builder.
            Code("insufficient_stock").
            Public("Not enough items in stock.").
            With("requested", req.Quantity).
            With("available", product.Stock).
            Errorf("insufficient stock for product %s", req.ProductID)
    }

    return nil
}
```

## エラーラップのベストプラクティス

### DO: 直接ラップする、nilチェック不要

```go
// ✓ Good — Wrap returns nil if err is nil
return oops.Wrapf(err, "operation failed")

// ✗ Bad — unnecessary nil check
if err != nil {
    return oops.Wrapf(err, "operation failed")
}
return nil
```

### DO: 各層でコンテキストを追加する

各アーキテクチャ層は Wrap/Wrapf でコンテキストを追加すべきである — パッケージ境界ごとに少なくとも1回（すべての関数呼び出しで必須ではない）。

```go
// ✓ Good — each layer adds relevant context
func Controller() error {
    return oops.In("controller").Trace(traceID).Wrapf(Service(), "user request failed")
}

func Service() error {
    return oops.In("service").With("op", "create_user").Wrapf(Repository(), "db operation failed")
}

func Repository() error {
    return oops.In("repository").Tags("database", "postgres").Errorf("connection timeout")
}
```

### DO: エラーメッセージを低カーディナリティに保つ

エラーメッセージはAPM集約のために低カーディナリティでなければならない。変数データをメッセージに補間すると、Datadog、Loki、Sentryでのグループ化が壊れる。

```go
// ✗ Bad — high-cardinality, breaks APM grouping
oops.Errorf("failed to process user %s in tenant %s", userID, tenantID)

// ✓ Good — static message + structured attributes
oops.With("user_id", userID).With("tenant_id", tenantID).Errorf("failed to process user")
```

## パニックリカバリー

`oops.Recover()` はゴルーチン境界で使用しなければならない。パニックを構造化エラーに変換する:

```go
func ProcessData(data string) (err error) {
    return oops.
        In("data-processor").
        Code("panic_recovered").
        Hint("Check input data format and dependencies").
        With("panic_value", r).
        Recover(func() {
            riskyOperation(data)
        })
}
```

## エラー情報へのアクセス

`samber/oops` エラーは標準の `error` インターフェースを実装する。追加情報へのアクセス:

```go
if oopsErr, ok := err.(oops.OopsError); ok {
    fmt.Println("Code:", oopsErr.Code())
    fmt.Println("Domain:", oopsErr.Domain())
    fmt.Println("Tags:", oopsErr.Tags())
    fmt.Println("Context:", oopsErr.Context())
    fmt.Println("Stacktrace:", oopsErr.Stacktrace())
}

// Get public-facing message with fallback
publicMsg := oops.GetPublic(err, "Something went wrong")
```

### 出力フォーマット

```go
fmt.Printf("%+v\n", err)       // verbose with stack trace
bytes, _ := json.Marshal(err)  // JSON for logging
slog.Error(err.Error(), slog.Any("error", err))  // slog integration
```

## コンテキスト伝播

Goのコンテキストを通じてエラーコンテキストを伝播させる:

```go
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        builder := oops.
            In("http").
            Request(r, false).
            Trace(r.Header.Get("X-Trace-ID"))

        ctx := oops.WithBuilder(r.Context(), builder)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func handler(ctx context.Context) error {
    return oops.FromContext(ctx).Tags("handler", "users").Errorf("something failed")
}
```

アサーション、設定、追加のロガー例については [Advanced patterns](./references/advanced.md) を参照。

## 参考資料

- [github.com/samber/oops](https://github.com/samber/oops)
- [pkg.go.dev/github.com/samber/oops](https://pkg.go.dev/github.com/samber/oops)

## クロスリファレンス

- 一般的なエラーハンドリングパターンについては → See `samber/cc-skills-golang@golang-error-handling` skill
- ロガー統合と構造化ログについては → See `samber/cc-skills-golang@golang-observability` skill
