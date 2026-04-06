---
name: golang-design-patterns
description: "Idiomatic Golang design patterns — functional options, constructors, error flow and cascading, resource management and lifecycle, graceful shutdown, resilience, architecture, dependency injection, data handling, and streaming. Apply when designing Go APIs, structuring applications, choosing between patterns, making design decisions, architectural choices, or production hardening."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🏗️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a Go architect who values simplicity and explicitness. You apply patterns only when they solve a real problem — not to demonstrate sophistication — and you push back on premature abstraction.

**Modes:**

- **Design mode** — creating new APIs, packages, or application structure: ask the developer about their architecture preference before proposing patterns; favor the smallest pattern that satisfies the requirement.
- **Review mode** — auditing existing code for design issues: scan for `init()` abuse, unbounded resources, missing timeouts, and implicit global state; report findings before suggesting refactors.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-design-patterns` skill takes precedence.

# Go デザインパターンとイディオム

本番環境対応コードのためのGoのイディオマティックなパターン。エラーハンドリングの詳細は `samber/cc-skills-golang@golang-error-handling` スキル、コンテキストの伝播は `samber/cc-skills-golang@golang-context` スキル、構造体/インターフェース設計は `samber/cc-skills-golang@golang-structs-interfaces` スキルを参照してください。

## ベストプラクティスまとめ

1. コンストラクタは **Functional Options** を使うべき — APIの進化に伴いスケールしやすい（オプションごとに1関数、破壊的変更なし）
2. Functional Optionsはバリデーションが失敗する可能性がある場合、必ず **エラーを返す** こと — 実行時ではなくコンストラクション時に不正な設定をキャッチする
3. **`init()` を避ける** — 暗黙的に実行され、エラーを返せず、テストを予測不能にする。明示的なコンストラクタを使用する
4. 列挙型は **1から開始** すべき（または0にUnknownセンチネル） — Goのゼロ値が最初の列挙型メンバーとして暗黙的に通過する
5. エラーケースは **早期リターンで最初に処理** しなければならない — ハッピーパスをフラットに保つ
6. **パニックはバグ用であり、期待されるエラー用ではない** — 呼び出し元は返されたエラーを処理できる。パニックはプロセスをクラッシュさせる
7. **オープン直後に `defer Close()` する** — 後のコード変更がクリーンアップを誤ってスキップする可能性がある
8. **`runtime.SetFinalizer` より `runtime.AddCleanup`** — ファイナライザは予測不能で、オブジェクトを復活させる可能性がある
9. すべての外部呼び出しには **タイムアウトを設定** すべき — 遅いアップストリームがゴルーチンを無期限にハングさせる
10. **すべてを制限する**（プールサイズ、キューの深さ、バッファ） — 制限のないリソースはクラッシュするまで増大する
11. リトライロジックは試行間で必ず **コンテキストのキャンセルを確認** すること
12. ループ内の連結には **`strings.Builder` を使用** → `samber/cc-skills-golang@golang-code-style` を参照
13. string vs []byte: **変異とI/Oには `[]byte`**、表示とキーには `string` を使用 — 変換はアロケーションを伴う
14. イテレータ (Go 1.23+): **遅延評価に使用** — すべてをメモリにロードすることを避ける
15. **大量転送はストリーミングする** — 数百万行のロードはOOMを引き起こす。ストリーミングはメモリを一定に保つ
16. **静的アセット** には `//go:embed` — コンパイル時に埋め込み、実行時のファイルI/Oエラーを排除する
17. キー/トークンには **`crypto/rand` を使用** — `math/rand` は予測可能 → `samber/cc-skills-golang@golang-security` を参照
18. 正規表現は必ず **パッケージレベルで一度だけコンパイル** すること — コンパイルはO(n)でアロケーションを伴う
19. コンパイル時インターフェースチェック: **`var _ Interface = (*Type)(nil)`**
20. **少しの再実装 > 大きな依存関係** — 各依存は攻撃面とメンテナンス負担を増やす
21. **テスタビリティのために設計する** — インターフェースを受け入れ、依存関係を注入する

## コンストラクタパターン: Functional Options vs Builder

### Functional Options（推奨）

```go
type Server struct {
    addr         string
    readTimeout  time.Duration
    writeTimeout time.Duration
    maxConns     int
}

type Option func(*Server)

func WithReadTimeout(d time.Duration) Option {
    return func(s *Server) { s.readTimeout = d }
}

func WithWriteTimeout(d time.Duration) Option {
    return func(s *Server) { s.writeTimeout = d }
}

func WithMaxConns(n int) Option {
    return func(s *Server) { s.maxConns = n }
}

func NewServer(addr string, opts ...Option) *Server {
    // Default options
    s := &Server{
        addr:         addr,
        readTimeout:  5 * time.Second,
        writeTimeout: 10 * time.Second,
        maxConns:     100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
srv := NewServer(":8080",
    WithReadTimeout(30*time.Second),
    WithMaxConns(500),
)
```

コンストラクタは **Functional Options** を使うべき — APIの進化に伴いスケールしやすく、コード量も少ない。設定ステップ間で複雑なバリデーションが必要な場合のみビルダーパターンを使用する。

## コンストラクタと初期化

### `init()` とミュータブルなグローバル変数を避ける

`init()` は暗黙的に実行され、テストを困難にし、隠れた依存関係を作る:

- 複数の `init()` 関数は宣言順に実行され、ファイル間では **ファイル名のアルファベット順** — 脆弱
- エラーを返せない — 失敗時はパニックまたは `log.Fatal` するしかない
- `main()` やテストの前に実行される — 副作用がテストを予測不能にする

```go
// Bad — hidden global state
var db *sql.DB

func init() {
    var err error
    db, err = sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
}

// Good — explicit initialization, injectable
func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}
```

### 列挙型: 1から開始する

ゼロ値は無効/未設定の状態を表すべき:

```go
type Status int

const (
    StatusUnknown Status = iota // 0 = invalid/unset
    StatusActive                // 1
    StatusInactive              // 2
    StatusSuspended             // 3
)
```

### 正規表現は一度だけコンパイルする

```go
// Good — compiled once at package level
var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

func ValidateEmail(email string) bool {
    return emailRegex.MatchString(email)
}
```

### 静的アセットには `//go:embed` を使用する

```go
import "embed"

//go:embed templates/*
var templateFS embed.FS

//go:embed version.txt
var version string
```

### コンパイル時インターフェースチェック

→ See `samber/cc-skills-golang@golang-structs-interfaces` for the `var _ Interface = (*Type)(nil)` pattern.

## エラーフローパターン

エラーケースは早期リターンで最初に処理しなければならない — ハッピーパスを最小限のインデントに保つ。→ See `samber/cc-skills-golang@golang-code-style` for the full pattern and examples.

### パニックを使うべきとき vs エラーを返すべきとき

- **エラーを返す**: ネットワーク障害、ファイルが見つからない、無効な入力 — 呼び出し元が処理できるすべてのケース
- **パニック**: あり得ないはずの場所でのnilポインタ、不変条件の違反、初期化時に使用される `Must*` コンストラクタ
- **`.Close()` エラー**: チェックしなくても許容される — エラーハンドリングなしの `defer f.Close()` で問題ない

## データハンドリング

### string vs []byte vs []rune

| 型       | デフォルトの用途 | 使用する場面                                          |
| -------- | -------------- | --------------------------------------------------- |
| `string` | すべて          | イミュータブル、安全、UTF-8                             |
| `[]byte` | I/O            | `io.Writer` への書き込み、文字列構築、変異              |
| `[]rune` | Unicode操作    | `len()` がバイトではなく文字数を意味する必要がある場合    |

繰り返しの変換を避ける — 各変換はアロケーションを伴う。他方が必要になるまで一つの型にとどまる。

### イテレータとストリーミングによる大量データ処理

イテレータ (Go 1.23+) とストリーミングパターンを使用して、すべてをメモリにロードせずに大量のデータセットを処理する。サービス間の大量転送（例: DBからHTTPへの100万行）には、OOMを防ぐためにストリーミングする。

コード例は [Data Handling Patterns](references/data-handling.md) を参照。

## リソース管理

オープン直後に `defer Close()` する — 待たない、忘れない:

```go
f, err := os.Open(path)
if err != nil {
    return err
}
defer f.Close() // right here, not 50 lines later

rows, err := db.QueryContext(ctx, query)
if err != nil {
    return err
}
defer rows.Close()
```

グレースフルシャットダウン、リソースプール、`runtime.AddCleanup` については [Resource Management](references/resource-management.md) を参照。

## レジリエンスと制限

### すべての外部呼び出しにタイムアウトを設定する

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

resp, err := httpClient.Do(req.WithContext(ctx))
```

### リトライとコンテキストチェック

リトライロジックは試行間で `ctx.Err()` を確認し、`ctx.Done()` の `select` による指数/線形バックオフを使用しなければならない。長いループは定期的に `ctx.Err()` を確認しなければならない。→ See `samber/cc-skills-golang@golang-context` skill.

## データベースパターン

→ See `samber/cc-skills-golang@golang-database` skill for sqlx/pgx, transactions, nullable columns, connection pools, repository interfaces, testing.

## アーキテクチャ

開発者にどのアーキテクチャを好むか確認する: クリーンアーキテクチャ、ヘキサゴナル、DDD、またはフラットレイアウト。小さなプロジェクトに複雑なアーキテクチャを押し付けない。

アーキテクチャに関わらず共通の原則:

- **ドメインを純粋に保つ** — ドメイン層にフレームワーク依存を持たない
- **早期に失敗する** — 境界でバリデーションし、内部コードを信頼する
- **不正な状態を表現不可能にする** — 型を使って不変条件を強制する
- **12-factor app** の原則を尊重する — → see `samber/cc-skills-golang@golang-project-layout`

## 詳細ガイド

| ガイド | 範囲 |
| --- | --- |
| [Architecture Patterns](references/architecture.md) | 高レベルの原則、各アーキテクチャが適する場面 |
| [Clean Architecture](references/clean-architecture.md) | ユースケース、依存関係ルール、レイヤードアダプタ |
| [Hexagonal Architecture](references/hexagonal-architecture.md) | ポートとアダプタ、ドメインコアの分離 |
| [Domain-Driven Design](references/ddd.md) | アグリゲート、値オブジェクト、境界づけられたコンテキスト |

## コード哲学

- **繰り返しのコードを避ける** — ただし早すぎる抽象化はしない
- **依存関係を最小化する** — 少しの再実装 > 大きな依存関係
- **テスタビリティのために設計する** — インターフェースを受け入れ、依存関係を注入し、関数を純粋に保つ

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-data-structures` skill for data structure selection, internals, and container/ packages
- → See `samber/cc-skills-golang@golang-error-handling` skill for error wrapping, sentinel errors, and the single handling rule
- → See `samber/cc-skills-golang@golang-structs-interfaces` skill for interface design and composition
- → See `samber/cc-skills-golang@golang-concurrency` skill for goroutine lifecycle and graceful shutdown
- → See `samber/cc-skills-golang@golang-context` skill for timeout and cancellation patterns
- → See `samber/cc-skills-golang@golang-project-layout` skill for architecture and directory structure
