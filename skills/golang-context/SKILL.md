---
name: golang-context
description: "Idiomatic context.Context usage in Golang — creation, propagation, cancellation, timeouts, deadlines, context values, and cross-service tracing. Apply when working with context.Context in any Go code."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🔗"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-context` skill takes precedence.

# Go context.Context ベストプラクティス

`context.Context`はGoにおけるキャンセルシグナル、デッドライン、リクエストスコープの値をAPI境界やgoroutine間で伝播するためのメカニズムである。リクエストの「セッション」と考えるとよい — 同じ作業単位に属するすべての操作を結びつける。

## ベストプラクティスの要約

1. 同じcontextをリクエストのライフサイクル全体で伝播しなければならない: HTTPハンドラ → サービス → DB → 外部API
2. `ctx`は最初のパラメータとし、`ctx context.Context`と命名しなければならない
3. contextを構造体に格納してはならない — 関数パラメータを通じて明示的に渡す
4. `nil`のcontextを渡してはならない — 不明な場合は`context.TODO()`を使用する
5. `cancel()`は`WithCancel`/`WithTimeout`/`WithDeadline`の直後に必ずdeferしなければならない
6. `context.Background()`はトップレベル（main、init、テスト）でのみ使用しなければならない
7. contextが必要だがまだ利用可能でない場合は、プレースホルダーとして**`context.TODO()`を使用する**
8. リクエストパスの途中で新しい`context.Background()`を作成してはならない
9. contextの値のキーは衝突を防ぐためにエクスポートされない型でなければならない
10. contextの値はリクエストスコープのメタデータのみを運ぶべきであり、関数パラメータを含めてはならない
11. 親リクエストより長く生存する必要があるバックグラウンド作業を生成する場合は**`context.WithoutCancel`**（Go 1.21+）を使用する

## contextの作成

| 状況 | 使用するもの |
| --- | --- |
| エントリーポイント（main、init、テスト） | `context.Background()` |
| 関数がcontextを必要とするが、呼び出し元がまだ提供していない | `context.TODO()` |
| HTTPハンドラ内 | `r.Context()` |
| キャンセル制御が必要 | `context.WithCancel(parentCtx)` |
| デッドライン/タイムアウトが必要 | `context.WithTimeout(parentCtx, duration)` |

## contextの伝播: 核心原則

最も重要なルール: **コールチェーン全体で同じcontextを伝播する**。正しく伝播すれば、親のcontextをキャンセルするとすべての下流の処理が自動的にキャンセルされる。

```go
// ✗ Bad — creates a new context, breaking the chain
func (s *OrderService) Create(ctx context.Context, order Order) error {
    return s.db.ExecContext(context.Background(), "INSERT INTO orders ...", order.ID)
}

// ✓ Good — propagates the caller's context
func (s *OrderService) Create(ctx context.Context, order Order) error {
    return s.db.ExecContext(ctx, "INSERT INTO orders ...", order.ID)
}
```

## 詳細

- **[キャンセル、タイムアウト、デッドライン](./references/cancellation.md)** — キャンセルの伝播方法: 手動キャンセル用の`WithCancel`、一定時間後の自動キャンセル用の`WithTimeout`、絶対時刻デッドライン用の`WithDeadline`。並行コードでの待ち受けパターン（`<-ctx.Done()`）、`AfterFunc`コールバック、親リクエストより長く生存する必要がある操作（例: 監査ログ）のための`WithoutCancel`。

- **[contextの値とサービス間トレーシング](./references/values-tracing.md)** — 安全なcontext値パターン: 名前空間の衝突を防ぐエクスポートされないキー型、context値（リクエストID、ユーザーID）と関数パラメータの使い分け。トレースcontextの伝播: OpenTelemetryトレースヘッダー、ログ集約用の相関ID、サービス境界を越えたcontextのマーシャリング/アンマーシャリング。

- **[HTTPサーバーとサービス呼び出しにおけるcontext](./references/http-services.md)** — HTTPハンドラのcontext: リクエストスコープのキャンセル用`r.Context()`、ミドルウェア統合、サービスへの伝播。HTTPクライアントパターン: `NewRequestWithContext`、クライアントタイムアウト、contextを考慮したリトライ。データベース操作: デッドラインを尊重するため常に`*Context`バリアント（`QueryContext`、`ExecContext`）を使用する。

## 相互参照

- → See the `samber/cc-skills-golang@golang-concurrency` skill for goroutine cancellation patterns using context
- → See the `samber/cc-skills-golang@golang-database` skill for context-aware database operations (QueryContext, ExecContext)
- → See the `samber/cc-skills-golang@golang-observability` skill for trace context propagation with OpenTelemetry
- → See the `samber/cc-skills-golang@golang-design-patterns` skill for timeout and resilience patterns

## リンターによる検出

contextの多くの落とし穴はリンターによって自動的に検出される: `govet`、`staticcheck`。→ See the `samber/cc-skills-golang@golang-linter` skill for configuration and usage.
