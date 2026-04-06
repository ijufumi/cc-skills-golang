---
name: golang-dependency-injection
description: "Comprehensive guide for dependency injection (DI) in Golang. Covers why DI matters (testability, loose coupling, separation of concerns, lifecycle management), manual constructor injection, and DI library comparison (google/wire, uber-go/dig, uber-go/fx, samber/do). Use this skill when designing service architecture, setting up dependency injection, refactoring tightly coupled code, managing singletons or service factories, or when the user asks about inversion of control, service containers, or wiring dependencies in Go."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🔌"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** You are a Go software architect. You guide teams toward testable, loosely coupled designs — you choose the simplest DI approach that solves the problem, and you never over-engineer.

**Modes:**

- **Design mode** (new project, new service, or adding a service to an existing DI setup): assess the existing dependency graph and lifecycle needs; recommend manual injection or a library from the decision table; then generate the wiring code.
- **Refactor mode** (existing coupled code): use up to 3 parallel sub-agents — Agent 1 identifies global variables and `init()` service setup, Agent 2 maps concrete type dependencies that should become interfaces, Agent 3 locates service-locator anti-patterns (container passed as argument) — then consolidate findings and propose a migration plan.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-dependency-injection` skill takes precedence.

# Goにおける依存性注入

依存性注入（DI）とは、コンポーネントが自ら依存関係を作成・検索するのではなく、外部から渡すことを意味する。Goでは、テスト可能で疎結合なアプリケーションを構築する方法であり、サービスが必要なものを宣言し、呼び出し元（またはコンテナ）がそれを提供する。

このスキルは網羅的ではない。DIライブラリ（google/wire、uber-go/dig、uber-go/fx、samber/do）を使用する際は、最新のAPIシグネチャについてライブラリの公式ドキュメントとコード例を参照すること。

インターフェースベースの設計基盤（インターフェースを受け取り、構造体を返す）については、`samber/cc-skills-golang@golang-structs-interfaces`スキルを参照。

## ベストプラクティスの要約

1. 依存関係はコンストラクタ経由で注入しなければならない — サービスセットアップにグローバル変数や`init()`を使用してはならない
2. 小規模プロジェクト（10サービス未満）は手動コンストラクタ注入を使用すべき — ライブラリは不要
3. インターフェースは実装側ではなく使用側で定義しなければならない — インターフェースを受け取り、構造体を返す
4. グローバルレジストリやパッケージレベルのサービスロケータを使用してはならない
5. DIコンテナはコンポジションルート（`main()`またはアプリ起動時）にのみ存在しなければならない — コンテナを依存関係として渡してはならない
6. **遅延初期化を優先する** — サービスは最初にリクエストされた時にのみ作成する
7. ステートフルなサービス（DB接続、キャッシュ）には**シングルトンを使用し**、ステートレスなものにはトランジェントを使用する
8. **インターフェース境界でモックする** — DIによりこれが容易になる
9. **依存関係グラフを浅く保つ** — 深いチェーンは設計上の問題を示す
10. プロジェクトの規模とチームに適した**DIライブラリを選択する** — 以下の判断テーブルを参照

## なぜ依存性注入が必要か？

| DIなしの問題 | DIによる解決方法 |
| --- | --- |
| 関数が自身の依存関係を作成する | 依存関係が注入される — 実装を自由に入れ替え可能 |
| テストに実際のデータベースやAPIが必要 | テストでモック実装を渡す |
| 1つのコンポーネントの変更が他を壊す | インターフェースによる疎結合 — コンポーネントは互いの内部を知らない |
| サービスがあちこちで初期化される | 集中管理されたコンテナがライフサイクルを管理（シングルトン、ファクトリ、遅延） |
| すべてのサービスが起動時にロードされる | 遅延ロード — サービスは最初にリクエストされた時にのみ作成 |
| グローバル状態と`init()`関数 | 起動時の明示的なワイヤリング — 予測可能でデバッグ可能 |

DIは多くのサービスが相互接続されたアプリケーション（HTTPサーバー、マイクロサービス、プラグイン付きCLIツール）で真価を発揮する。2-3個の関数からなる小さなスクリプトでは、手動ワイヤリングで十分。過剰設計しないこと。

## 手動コンストラクタ注入（ライブラリなし）

小規模プロジェクトでは、コンストラクタを通じて依存関係を渡す。完全なアプリケーション例については[手動DIの例](./references/manual-di.md)を参照。

```go
// ✓ Good — explicit dependencies, testable
type UserService struct {
    db     UserStore
    mailer Mailer
    logger *slog.Logger
}

func NewUserService(db UserStore, mailer Mailer, logger *slog.Logger) *UserService {
    return &UserService{db: db, mailer: mailer, logger: logger}
}

// main.go — manual wiring
func main() {
    logger := slog.Default()
    db := postgres.NewUserStore(connStr)
    mailer := smtp.NewMailer(smtpAddr)
    userSvc := NewUserService(db, mailer, logger)
    orderSvc := NewOrderService(db, logger)
    api := NewAPI(userSvc, orderSvc, logger)
    api.ListenAndServe(":8080")
}
```

```go
// ✗ Bad — hardcoded dependencies, untestable
type UserService struct {
    db *sql.DB
}

func NewUserService() *UserService {
    db, _ := sql.Open("postgres", os.Getenv("DATABASE_URL")) // hidden dependency
    return &UserService{db: db}
}
```

手動DIが破綻するケース:

- 相互依存のある15以上のサービスがある
- ライフサイクル管理（ヘルスチェック、グレースフルシャットダウン）が必要
- 遅延初期化やスコープ付きコンテナが必要
- ワイヤリング順序が脆弱になり保守が困難

## DIライブラリの比較

GoにはDIライブラリの3つの主要なアプローチがある:

- [google/wireの例](./references/google-wire.md) — コンパイル時コード生成
- [uber-go/dig + fxの例](./references/uber-dig-fx.md) — リフレクションベースのフレームワーク
- [samber/doの例](./references/samber-do.md) — ジェネリクスベース、コード生成なし

### 判断テーブル

| 基準 | 手動 | google/wire | uber-go/dig + fx | samber/do |
| --- | --- | --- | --- | --- |
| **プロジェクト規模** | 小規模（10サービス未満） | 中〜大規模 | 大規模 | あらゆる規模 |
| **型安全性** | コンパイル時 | コンパイル時（コード生成） | 実行時（リフレクション） | コンパイル時（ジェネリクス） |
| **コード生成** | なし | 必須（`wire_gen.go`） | なし | なし |
| **リフレクション** | なし | なし | あり | なし |
| **APIスタイル** | N/A | プロバイダセット + ビルドタグ | 構造体タグ + デコレータ | シンプルなジェネリック関数 |
| **遅延ロード** | 手動 | N/A（すべて即時） | 組み込み（fx） | 組み込み |
| **シングルトン** | 手動 | 組み込み | 組み込み | 組み込み |
| **トランジェント/ファクトリ** | 手動 | 手動 | 組み込み | 組み込み |
| **スコープ/モジュール** | 手動 | プロバイダセット | モジュールシステム（fx） | 組み込み（階層型） |
| **ヘルスチェック** | 手動 | 手動 | 手動 | 組み込みインターフェース |
| **グレースフルシャットダウン** | 手動 | 手動 | 組み込み（fx） | 組み込みインターフェース |
| **コンテナクローン** | N/A | N/A | N/A | 組み込み |
| **デバッグ** | print文 | コンパイルエラー | `fx.Visualize()` | `ExplainInjector()`、Webインターフェース |
| **Goバージョン** | 任意 | 任意 | 任意 | 1.18+（ジェネリクス） |
| **学習コスト** | なし | 中 | 高 | 低 |

### クイック比較: 同じアプリを4つの方法で

依存関係グラフ: `Config -> Database -> UserStore -> UserService -> API`

**手動**:

```go
cfg := NewConfig()
db := NewDatabase(cfg)
store := NewUserStore(db)
svc := NewUserService(store)
api := NewAPI(svc)
api.Run()
// No automatic shutdown, health checks, or lazy loading
```

**google/wire**:

```go
// wire.go — then run: wire ./...
func InitializeAPI() (*API, error) {
    wire.Build(NewConfig, NewDatabase, NewUserStore, NewUserService, NewAPI)
    return nil, nil
}
// No shutdown or health check support
```

**uber-go/fx**:

```go
app := fx.New(
    fx.Provide(NewConfig, NewDatabase, NewUserStore, NewUserService),
    fx.Invoke(func(api *API) { api.Run() }),
)
app.Run() // manages lifecycle, but reflection-based
```

**samber/do**:

```go
i := do.New()
do.Provide(i, NewConfig)
do.Provide(i, NewDatabase)    // auto shutdown + health check
do.Provide(i, NewUserStore)
do.Provide(i, NewUserService)
api := do.MustInvoke[*API](i)
api.Run()
// defer i.Shutdown() — handles all cleanup automatically
```

## DIを使ったテスト

DIによりテストが簡単になる — 実際の実装の代わりにモックを注入する:

```go
// Define a mock
type MockUserStore struct {
    users map[string]*User
}

func (m *MockUserStore) FindByID(ctx context.Context, id string) (*User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, ErrNotFound
    }
    return u, nil
}

// Test with manual injection
func TestUserService_GetUser(t *testing.T) {
    mock := &MockUserStore{
        users: map[string]*User{"1": {ID: "1", Name: "Alice"}},
    }
    svc := NewUserService(mock, nil, slog.Default())

    user, err := svc.GetUser(context.Background(), "1")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("got %q, want %q", user.Name, "Alice")
    }
}
```

### samber/doを使ったテスト — クローンとオーバーライド

コンテナのクローンは、モックが必要なサービスのみをオーバーライドした分離されたコピーを作成する:

```go
func TestUserService_WithDo(t *testing.T) {
    // Create a test injector with mock implementation
    testInjector := do.New()

    // Provide the mock UserStore interface
    do.Override[UserStore](testInjector, &MockUserStore{
        users: map[string]*User{"1": {ID: "1", Name: "Alice"}},
    })

    // Provide other real services as needed
    do.Provide[*slog.Logger](testInjector, func(i *do.Injector) (*slog.Logger, error) {
        return slog.Default(), nil
    })

    svc := do.MustInvoke[*UserService](testInjector)
    user, err := svc.GetUser(context.Background(), "1")
    // ... assertions
}
```

これは特に、ほとんどのサービスを実際のものにしつつ特定の境界（データベース、外部API、メーラー）をモックしたいインテグレーションテストで有用である。

## DIライブラリを採用するタイミング

| シグナル | アクション |
| --- | --- |
| 10サービス未満、単純な依存関係 | 手動コンストラクタ注入を続ける |
| 10-20サービス、横断的関心事がある | DIライブラリを検討する |
| 20以上のサービス、ライフサイクル管理が必要 | 強く推奨 |
| ヘルスチェック、グレースフルシャットダウンが必要 | 組み込みライフサイクルサポート付きのライブラリを使用する |
| チームがDIの概念に不慣れ | 手動から始め、段階的に移行する |

## よくある間違い

| 間違い | 修正方法 |
| --- | --- |
| 依存関係としてのグローバル変数 | コンストラクタまたはDIコンテナを通じて渡す |
| サービスセットアップに`init()`を使用 | `main()`またはコンテナでの明示的な初期化 |
| 具象型への依存 | 使用境界でインターフェースを受け取る |
| コンテナをどこにでも渡す（サービスロケータ） | コンテナではなく、特定の依存関係を注入する |
| 深い依存関係チェーン (A->B->C->D->E) | フラット化する — ほとんどのサービスはリポジトリと設定に直接依存すべき |
| リクエストごとに新しいコンテナを作成 | アプリケーションにつき1つのコンテナ; リクエストレベルの分離にはスコープを使用する |

## 相互参照

- → See `samber/cc-skills-golang@golang-samber-do` skill for detailed samber/do usage patterns
- → See `samber/cc-skills-golang@golang-structs-interfaces` skill for interface design and composition
- → See `samber/cc-skills-golang@golang-testing` skill for testing with dependency injection
- → See `samber/cc-skills-golang@golang-project-layout` skill for DI initialization placement

## 参考資料

- [samber/do/v2 documentation](https://do.samber.dev) | [github.com/samber/do/v2](https://github.com/samber/do)
- [google/wire user guide](https://github.com/google/wire/blob/main/docs/guide.md)
- [uber-go/fx documentation](https://uber-go.github.io/fx/)
- [uber-go/dig](https://github.com/uber-go/dig)
