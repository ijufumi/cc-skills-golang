---
name: golang-samber-do
description: "Implements dependency injection in Golang using samber/do. Apply this skill when working with dependency injection, setting up service containers, managing service lifecycles, or when you see code using github.com/samber/do/v2. Also use when refactoring manual dependency injection, implementing health checks, graceful shutdown, or organizing services into scopes/modules."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "💉"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs
---

**Persona:** You are a Go architect setting up dependency injection. You keep the container at the composition root, depend on interfaces not concrete types, and treat provider errors as first-class failures.

# Go における samber/do を使った依存性注入

Go 1.18+ のジェネリクスに基づく型安全な依存性注入ツールキット。

**公式リソース:**

- [pkg.go.dev/github.com/samber/do/v2](https://pkg.go.dev/github.com/samber/do/v2)
- [do.samber.dev](https://do.samber.dev)
- [github.com/samber/do/v2](https://github.com/samber/do)

このスキルは網羅的ではありません。詳細はライブラリのドキュメントやコード例を参照してください。Context7 がディスカバリプラットフォームとして役立ちます。

このライブラリの v1 は使用しないでください。代わりに v2 をインストールしてください:

```bash
go get -u github.com/samber/do/v2
```

## コアコンセプト

### インジェクター（コンテナ）

```go
import "github.com/samber/do/v2"

injector := do.New()
```

### サービスタイプ

- **Lazy**（デフォルト）: 最初にリクエストされたときに作成される
- **Eager**: コンテナの起動時に即座に作成される
- **Transient**: リクエストごとに新しいインスタンスが作成される
- **Value**: 事前に作成された値、インスタンス化なし

### プロバイダー関数

サービスはプロバイダー関数を通じて登録しなければなりません:

```go
type Provider[T any] func(i Injector) (T, error)
```

## 基本的な使い方

### 1. サービスの定義と登録

「インターフェースを受け取り、構造体を返す」に従ってください:

```go
// サービスを登録（デフォルトは lazy）
do.Provide(injector, func(i do.Injector) (Database, error) {
    return &PostgreSQLDatabase{connString: "postgres://..."}, nil
})

// 事前作成済みの値を登録
do.ProvideValue(injector, &Config{Port: 8080})

// transient サービスを登録（毎回新しいインスタンス）
do.ProvideTransient(injector, func(i do.Injector) (*Logger, error) {
    return &Logger{}, nil
})

// eager サービスを登録（即座に作成）
do.Provide(injector, do.Eager(&Config{Port: 8080}))
```

### 2. サービスの呼び出し

コンテナはコンポジションルートでのみアクセスしなければなりません:

```go
// Invoke with error handling
db, err := do.Invoke[Database](injector)

// MustInvoke panics on error (use when confident service exists)
db := do.MustInvoke[Database](injector)
```

### 3. Service Dependencies

```go
func NewUserService(i do.Injector) (UserService, error) {
    db := do.MustInvoke[Database](i)
    cache := do.MustInvoke[Cache](i)
    return &userService{db: db, cache: cache}, nil
}

do.Provide(injector, NewUserService)
```

### 4. Implicit Aliasing (Preferred)

Register a concrete type and invoke as an interface without explicit aliasing:

```go
// Register concrete type
do.Provide(injector, func(i do.Injector) (*PostgreSQLDatabase, error) {
    return &PostgreSQLDatabase{}, nil
})

// Invoke directly as interface (implicit aliasing)
db := do.MustInvokeAs[Database](injector)
```

### 5. Named Services

Register multiple services of the same type:

```go
do.ProvideNamed(injector, "primary-db", func(i do.Injector) (*Database, error) {
    return &Database{URL: "postgres://primary..."}, nil
})

mainDB := do.MustInvokeNamed[*Database](injector, "primary-db")
```

## Package Organization

Use `do.Package()` to organize service registration by module:

```go
// infrastructure/package.go
var Package = do.Package(
    do.Lazy(func(i do.Injector) (*postgres.DB, error) {
        cfg := do.MustInvoke[*Config](i)
        return postgres.Connect(cfg.DatabaseURL)
    }),
    do.Lazy(func(i do.Injector) (*redis.Client, error) {
        cfg := do.MustInvoke[*Config](i)
        return redis.NewClient(cfg.RedisURL), nil
    }),
)

// main.go
injector := do.New(infrastructure.Package, service.Package)
```

## Full Application Setup

```go
func main() {
    injector := do.New(
        infrastructure.Package,
        repository.Package,
        service.Package,
        transport.Package,
    )

    server := do.MustInvoke[*http.Server](injector)
    go server.ListenAndServe()

    _ = injector.ShutdownOnSignalsWithContext(context.Background(), os.Interrupt)
}
```

## Best Practices

1. Depend on interfaces, not concrete types — lets you swap implementations in tests without touching production code
2. Each service should have one job — services with multiple responsibilities are harder to test and harder to replace
3. Keep dependency trees shallow — chains beyond 3-4 levels make initialization order fragile and errors harder to trace
4. Handle errors in provider functions — a silently failing provider creates a broken service that crashes later in unexpected places
5. Use scopes to organize services by lifecycle — request-scoped services prevent leaks, global services prevent redundant initialization

For scopes, lifecycle management, struct injection, and debugging, see [Advanced Usage](./references/advanced.md).

For testing patterns (cloning, overrides, mocks), see [Testing](./references/testing.md).

## Quick Reference

### Registration

| Function                        | Purpose                          |
| ------------------------------- | -------------------------------- |
| `do.Provide[T]()`               | Register lazy service (default)  |
| `do.ProvideNamed[T]()`          | Register named lazy service      |
| `do.ProvideValue[T]()`          | Register pre-created value       |
| `do.ProvideNamedValue[T]()`     | Register named value             |
| `do.ProvideTransient[T]()`      | Register new instance each time  |
| `do.ProvideNamedTransient[T]()` | Register named transient service |
| `do.Package()`                  | Group service registrations      |

### Invocation

| Function                   | Purpose                                   |
| -------------------------- | ----------------------------------------- |
| `do.Invoke[T]()`           | Get service (with error)                  |
| `do.InvokeNamed[T]()`      | Get named service                         |
| `do.InvokeAs[T]()`         | Get first service matching interface      |
| `do.InvokeStruct[T]()`     | Inject into struct fields using tags      |
| `do.MustInvoke[T]()`       | Get service (panic on error)              |
| `do.MustInvokeNamed[T]()`  | Get named service (panic on error)        |
| `do.MustInvokeAs[T]()`     | Get service by interface (panic on error) |
| `do.MustInvokeStruct[T]()` | Inject into struct (panic on error)       |

## Cross-References

- → See `samber/cc-skills-golang@golang-dependency-injection` skill for DI concepts, comparison, and when to adopt a DI library
- → See `samber/cc-skills-golang@golang-structs-interfaces` skill for interface design patterns
- → See `samber/cc-skills-golang@golang-testing` skill for general testing patterns
