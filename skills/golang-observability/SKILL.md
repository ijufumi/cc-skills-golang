---
name: golang-observability
description: "Golang everyday observability — the always-on signals in production. Covers structured logging with slog, Prometheus metrics, OpenTelemetry distributed tracing, continuous profiling with pprof/Pyroscope, server-side RUM event tracking, alerting, and Grafana dashboards. Apply when instrumenting Go services for production monitoring, setting up metrics or alerting, adding OpenTelemetry tracing, correlating logs with traces, migrating legacy loggers (zap/logrus/zerolog) to slog, adding observability to new features, or implementing GDPR/CCPA-compliant tracking with Customer Data Platforms (CDP). Not for temporary deep-dive performance investigation (→ See golang-benchmark and golang-performance skills)."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "📡"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch WebSearch AskUserQuestion
---

**Persona:** You are a Go observability engineer. You treat every unobserved production system as a liability — instrument proactively, correlate signals to diagnose, and never consider a feature done until it is observable.

**Modes:**

- **Coding / instrumentation** (default): Add observability to new or existing code — declare metrics, add spans, set up structured logging, wire pprof toggles. Follow the sequential instrumentation guide.
- **Review mode** — reviewing a PR's instrumentation changes. Check that new code exports the expected signals (metrics declared, spans opened and closed, structured log fields consistent). Sequential.
- **Audit mode** — auditing existing observability coverage across a codebase. Launch up to 5 parallel sub-agents — one per signal (metrics, logging, tracing, profiling, RUM) — to check coverage simultaneously.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-observability` skill takes precedence.

# Go オブザーバビリティベストプラクティス

オブザーバビリティとは、外部出力からシステムの内部状態を理解する能力です。Goサービスにおいては、5つの補完的なシグナルを意味します: **ログ**、**メトリクス**、**トレース**、**プロファイル**、**RUM**。それぞれが異なる質問に答え、合わせてシステムの動作とユーザー体験の両方に対する完全な可視性を提供します。

オブザーバビリティライブラリ（Prometheusクライアント、OpenTelemetry SDK、ベンダー統合）を使用する場合は、最新のAPIシグネチャについてライブラリの公式ドキュメントとコード例を参照してください。

## ベストプラクティスの要約

1. **構造化ログを使用する** `log/slog` — 本番サービスは構造化ログ（JSON）を出力しなければならず（MUST）、自由形式の文字列は不可
2. **適切なログレベルを選択する** — Debugは開発用、Infoは通常操作、Warnは劣化状態、Errorは対応が必要な障害
3. **コンテキスト付きでログを記録する** — `slog.InfoContext(ctx, ...)` を使用してログとトレースを相関させる
4. **レイテンシメトリクスにはSummaryよりHistogramを優先する** — Histogramはサーバーサイド集約とパーセンタイルクエリをサポート。すべてのHTTPエンドポイントにレイテンシとエラーレートのメトリクスが必要（MUST）
5. **Prometheusのラベルカーディナリティを低く保つ** — 無制限の値（ユーザーID、完全なURL）をラベル値として使用してはならない（NEVER）
6. **パーセンタイルを追跡する**（P50, P90, P99, P99.9） Histograms + PromQLの `histogram_quantile()` を使用
7. **新規プロジェクトにOpenTelemetryトレーシングを設定する** — TracerProviderを早期に設定し、あらゆる場所にspanを追加
8. **すべての意味のある操作にspanを追加する** — サービスメソッド、DBクエリ、外部API呼び出し、メッセージキュー操作
9. **コンテキストをあらゆる場所に伝播する** — コンテキストはtrace_id、span_id、デッドラインをサービス境界を越えて運ぶ手段
10. **環境変数でプロファイリングを有効にする** — 再デプロイなしでpprofと継続的プロファイリングのオン/オフを切り替え
11. **シグナルを相関させる** — ログにtrace_idを注入し、exemplarsを使用してメトリクスとトレースをリンク
12. **フィーチャーはオブザーバブルになるまで完了ではない** — メトリクスを宣言し、適切なログを追加し、spanを作成
13. **インフラストラクチャと依存関係のアラートの出発点として [awesome-prometheus-alerts](https://samber.github.io/awesome-prometheus-alerts/) を使用する** — テクノロジー別に閲覧し、ルールをコピーし、しきい値をカスタマイズ

## クロスリファレンス

See `samber/cc-skills-golang@golang-error-handling` skill for the single handling rule. See `samber/cc-skills-golang@golang-troubleshooting` skill for using observability signals to diagnose production issues. See `samber/cc-skills-golang@golang-security` skill for protecting pprof endpoints and avoiding PII in logs. See `samber/cc-skills-golang@golang-context` skill for propagating trace context across service boundaries. See `samber/cc-skills@promql-cli` skill for querying and exploring PromQL expressions against Prometheus from the CLI.

## 5つのシグナル

| シグナル | 答える質問 | ツール | 使用場面 |
| --- | --- | --- | --- |
| **ログ** | 何が起こったか？ | `log/slog` | 離散イベント、エラー、監査証跡 |
| **メトリクス** | どれくらい／どれくらい速く？ | Prometheusクライアント | 集約測定、アラート、SLO |
| **トレース** | 時間はどこで使われたか？ | OpenTelemetry | サービス間のリクエストフロー、レイテンシ内訳 |
| **プロファイル** | なぜ遅い／メモリを使っているのか？ | pprof, Pyroscope | CPUホットスポット、メモリリーク、ロック競合 |
| **RUM** | ユーザーはどう体験しているか？ | PostHog, Segment | プロダクト分析、ファネル、セッションリプレイ |

## 詳細ガイド

各シグナルには、完全なコード例、設定パターン、コスト分析を含む専用ガイドがあります:

- **[構造化ログ](references/logging.md)** — 大規模なログ集約において構造化ログが重要な理由。`log/slog` のセットアップ、ログレベル（Debug/Info/Warn/Error）と各レベルの使用場面、トレースIDによるリクエスト相関、`slog.InfoContext` によるコンテキスト伝播、リクエストスコープの属性、slogエコシステム（ハンドラー、フォーマッター、ミドルウェア）、zap/logrus/zerologからの移行戦略。

- **[メトリクス収集](references/metrics.md)** — Prometheusクライアントのセットアップと4つのメトリクスタイプ（変化率のCounter、スナップショットのGauge、レイテンシ集約のHistogram）。詳細: HistogramがSummaryに勝る理由（サーバーサイド集約、`histogram_quantile` PromQLサポート）、命名規則、PromQL-as-comments規則（発見性のためメトリクス宣言の上にクエリを記述）、本番グレードのPromQL例、マルチウィンドウSLOバーンレートアラート、高カーディナリティラベル問題（ユーザーIDのような無制限の値がパフォーマンスを破壊する理由）。

- **[分散トレーシング](references/tracing.md)** — OpenTelemetry SDKを使用してサービス間のリクエストフローをトレースする方法とタイミング。span（作成、属性、ステータス記録）、HTTP計装のための `otelhttp` ミドルウェア、`span.RecordError()` によるエラー記録、トレースサンプリング（大規模ですべてを収集できない理由）、サービス境界を越えたトレースコンテキストの伝播、コスト最適化。

- **[プロファイリング](references/profiling.md)** — pprofによるオンデマンドプロファイリング（CPU、ヒープ、ゴルーチン、ミューテックス、ブロックプロファイル） — 本番環境での有効化方法、認証によるセキュリティ確保、再デプロイなしでの環境変数による切り替え。Pyroscopeによる継続的プロファイリングで常時パフォーマンス可視性を実現。各プロファイリングタイプのコスト影響と緩和戦略。

- **[リアルユーザーモニタリング](references/rum.md)** — ユーザーがサービスを実際にどう体験しているかの理解。プロダクト分析（イベントトラッキング、ファネル）、カスタマーデータプラットフォーム統合、重要なコンプライアンス: GDPR/CCPA同意チェック、データ主体の権利（ユーザー削除エンドポイント）、トラッキングのプライバシーチェックリスト。サーバーサイドイベントトラッキング（PostHog, Segment）とIDキーのベストプラクティス。

- **[アラート](references/alerting.md)** — プロアクティブな問題検出。4つのゴールデンシグナル（レイテンシ、トラフィック、エラー、飽和）、テクノロジー別に約500のすぐに使えるルールを持つルールライブラリとしての [awesome-prometheus-alerts](https://samber.github.io/awesome-prometheus-alerts/)、Goランタイムアラート（ゴルーチンリーク、GC圧力、OOMリスク）、重大度レベル、アラートを壊すよくある間違い（`rate` の代わりに `irate` を使用、フラッピング防止の `for:` 期間の欠落）。

- **[Grafanaダッシュボード](references/dashboards.md)** — Goランタイムモニタリング用のプリビルトダッシュボード（ヒープアロケーション、GCポーズ頻度、ゴルーチン数、CPU）。インストールすべき標準ダッシュボード、サービスに合わせたカスタマイズ方法、各ダッシュボードがどの運用上の質問に答えるかを説明。

## Correlating Signals

Signals are most powerful when connected. A trace_id in your logs lets you jump from a log line to the full request trace. An exemplar on a metric links a latency spike to the exact trace that caused it.

### Logs + Traces: `otelslog` bridge

```go
import "go.opentelemetry.io/contrib/bridges/otelslog"

// Create a logger that automatically injects trace_id and span_id
logger := otelslog.NewHandler("my-service")
slog.SetDefault(slog.New(logger))

// Now every slog call with context includes trace correlation
slog.InfoContext(ctx, "order created", "order_id", orderID)
// Output includes: {"trace_id":"abc123", "span_id":"def456", "msg":"order created", ...}
```

### Metrics + Traces: Exemplars

```go
// When recording a histogram observation, attach the trace_id as an exemplar
// so you can jump from a P99 spike directly to the offending trace
histogram.WithLabelValues("POST", "/orders").
    Exemplar(prometheus.Labels{"trace_id": traceID}, duration)
```

## Migrating Legacy Loggers

If the project currently uses `zap`, `logrus`, or `zerolog`, migrate to `log/slog`. It is the standard library logger since Go 1.21, has a stable API, and the ecosystem has consolidated around it. Continuing with third-party loggers means maintaining an extra dependency for no benefit.

**Migration strategy:**

1. Add `slog` as the new logger with `slog.SetDefault()`
2. Use bridge handlers during migration to route slog output through the existing logger: [samber/slog-zap](https://github.com/samber/slog-zap), [samber/slog-logrus](https://github.com/samber/slog-logrus), [samber/slog-zerolog](https://github.com/samber/slog-zerolog)
3. Gradually replace all `zap.L().Info(...)` / `logrus.Info(...)` / `log.Info().Msg(...)` calls with `slog.Info(...)`
4. Once fully migrated, remove the bridge handler and the old logger dependency

## Definition of Done for Observability

A feature is not production-ready until it is observable. Before marking a feature as done, verify:

- [ ] **Metrics declared** — counters for operations/errors, histograms for latencies, gauges for saturation. Each metric var has PromQL queries and alert rules as comments above its declaration.
- [ ] **Logging is proper** — structured key-value pairs with `slog`, context variants used (`slog.InfoContext`), no PII in logs, errors MUST be either logged OR returned (NEVER both).
- [ ] **Spans created** — every service method, DB query, and external API call has a span with relevant attributes, errors recorded with `span.RecordError()`.
- [ ] **Dashboards and alerts exist** — the PromQL from your metric comments is wired into Grafana dashboards and Prometheus alerting rules. Check [awesome-prometheus-alerts](https://samber.github.io/awesome-prometheus-alerts/) for ready-to-use rules covering your infrastructure dependencies (databases, caches, brokers, proxies).
- [ ] **RUM events tracked** — key business events tracked server-side (PostHog/Segment), identity key is `user_id` (not email), consent checked before tracking.

## Common Mistakes

```go
// ✗ Bad — log AND return (error gets logged multiple times up the chain)
if err != nil {
    slog.Error("query failed", "error", err)
    return fmt.Errorf("query: %w", err)
}

// ✓ Good — return with context, log once at the top level
if err != nil {
    return fmt.Errorf("querying users: %w", err)
}
```

```go
// ✗ Bad — high-cardinality label (unbounded user IDs)
httpRequests.WithLabelValues(r.Method, r.URL.Path, userID).Inc()

// ✓ Good — bounded label values only
httpRequests.WithLabelValues(r.Method, routePattern).Inc()
```

```go
// ✗ Bad — not passing context (breaks trace propagation)
result, err := db.Query("SELECT ...")

// ✓ Good — context flows through, trace continues
result, err := db.QueryContext(ctx, "SELECT ...")
```

```go
// ✗ Bad — using Summary for latency (can't aggregate across instances)
prometheus.NewSummary(prometheus.SummaryOpts{
    Name:       "http_request_duration_seconds",
    Objectives: map[float64]float64{0.99: 0.001},
})

// ✓ Good — use Histogram (aggregatable, supports histogram_quantile)
prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "http_request_duration_seconds",
    Buckets: prometheus.DefBuckets,
})
```
