---
name: golang-samber-slog
description: "Structured logging extensions for Golang using samber/slog-**** packages — multi-handler pipelines (slog-multi), log sampling (slog-sampling), attribute formatting (slog-formatter), HTTP middleware (slog-fiber, slog-gin, slog-chi, slog-echo), and backend routing (slog-datadog, slog-sentry, slog-loki, slog-syslog, slog-logstash, slog-graylog...). Apply when using or adopting slog, or when the codebase already imports any github.com/samber/slog-* package."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.0.3"
  openclaw:
    emoji: "🪵"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** You are a Go logging architect. You design log pipelines where every record flows through the right handlers — sampling drops noise early, formatters strip PII before records leave the process, and routers send errors to Sentry while info goes to Loki.

# samber/slog-\*\*\*\* — Go向け構造化ログパイプライン

Go 1.21+向けの20以上のコンポーザブルな `slog.Handler` パッケージ。3つのコアパイプラインライブラリに加え、標準の `slog.Handler` インターフェースを実装するHTTPミドルウェアとバックエンドシンクを提供します。

**公式リソース:**

- [github.com/samber/slog-multi](https://github.com/samber/slog-multi) — ハンドラー合成
- [github.com/samber/slog-sampling](https://github.com/samber/slog-sampling) — スループット制御
- [github.com/samber/slog-formatter](https://github.com/samber/slog-formatter) — 属性変換

このスキルは網羅的ではありません。詳細はライブラリのドキュメントとコード例を参照してください。Context7はディスカバリプラットフォームとして活用できます。

## パイプラインモデル

すべてのsamber/slogパイプラインは標準的な順序に従います。レコードは左から右に流れます。サンプリングを最初に配置して、シンクに到達しないレコードを早期にドロップし、CPUの無駄遣いを避けてください。

```
record → [Sampling] → [Pipe: trace/PII] → [Router] → [Sinks]
```

順序が重要です。サンプリングをフォーマットの前に行うことでCPUを節約できます。フォーマットをルーティングの前に行うことで、すべてのシンクがクリーンな属性を受け取ることが保証されます。この順序を逆にすると、ドロップされるレコードに対して無駄な処理が発生します。

## コアライブラリ

| ライブラリ | 目的 | 主なコンストラクタ |
| --- | --- | --- |
| `slog-multi` | ハンドラー合成 | `Fanout`, `Router`, `FirstMatch`, `Failover`, `Pool`, `Pipe` |
| `slog-sampling` | スループット制御 | `UniformSamplingOption`, `ThresholdSamplingOption`, `AbsoluteSamplingOption`, `CustomSamplingOption` |
| `slog-formatter` | 属性変換 | `PIIFormatter`, `ErrorFormatter`, `FormatByType[T]`, `FormatByKey`, `FlattenFormatterMiddleware` |

## slog-multi — ハンドラー合成

6つの合成パターン。それぞれ異なるルーティングニーズに対応します:

| パターン | 動作 | レイテンシへの影響 |
| --- | --- | --- |
| `Fanout(handlers...)` | すべてのハンドラーに順次ブロードキャスト | 全ハンドラーのレイテンシの合計 |
| `Router().Add(h, predicate).Handler()` | マッチするすべてのハンドラーにルーティング | マッチしたハンドラーの合計 |
| `Router().Add(...).FirstMatch().Handler()` | 最初のマッチのみにルーティング | 単一ハンドラーのレイテンシ |
| `Failover()(handlers...)` | 成功するまで順次試行 | プライマリハンドラーのレイテンシ（正常時） |
| `Pool()(handlers...)` | すべてのハンドラーに並行ブロードキャスト | 全ハンドラーのレイテンシの最大値 |
| `Pipe(middlewares...).Handler(sink)` | シンク前のミドルウェアチェーン | ミドルウェアのオーバーヘッド + シンク |

```go
// Route errors to Sentry, all logs to stdout
logger := slog.New(
    slogmulti.Router().
        Add(sentryHandler, slogmulti.LevelIs(slog.LevelError)).
        Add(slog.NewJSONHandler(os.Stdout, nil)).
        Handler(),
)
```

組み込みの述語: `LevelIs`, `LevelIsNot`, `MessageIs`, `MessageIsNot`, `MessageContains`, `MessageNotContains`, `AttrValueIs`, `AttrKindIs`。

すべてのパターンの完全なコード例は [Pipeline Patterns](references/pipeline-patterns.md) を参照してください。

## slog-sampling — スループット制御

| 戦略 | 動作 | 最適な用途 |
| --- | --- | --- |
| Uniform | 全レコードの固定%をドロップ | 開発/ステージングのノイズ削減 |
| Threshold | インターバルごとに最初のN件をログ出力し、その後レートRでサンプリング | 本番環境 — 初期の可視性を維持 |
| Absolute | グローバルでインターバルごとにN件に制限 | 厳密なコスト管理 |
| Custom | ユーザー関数がレコードごとのサンプルレートを返す | レベル対応またはタイムベースのルール |

サンプリングはパイプラインの最も外側のハンドラーでなければなりません。フォーマットの後に配置すると、ドロップされるレコードに対してCPUが無駄に消費されます。

```go
// Threshold: log first 10 per 5s, then 10% — errors always pass through via Router
logger := slog.New(
    slogmulti.
        Pipe(slogsampling.ThresholdSamplingOption{
            Tick: 5 * time.Second, Threshold: 10, Rate: 0.1,
        }.NewMiddleware()).
        Handler(innerHandler),
)
```

マッチャーは重複排除のために類似レコードをグループ化します: `MatchByLevel()`, `MatchByMessage()`, `MatchByLevelAndMessage()`（デフォルト）, `MatchBySource()`, `MatchByAttribute(groups, key)`。

戦略の比較と設定の詳細は [Sampling Strategies](references/sampling-strategies.md) を参照してください。

## slog-formatter — 属性変換

`Pipe` ミドルウェアとして適用し、すべての下流ハンドラーがクリーンな属性を受け取るようにします。

```go
logger := slog.New(
    slogmulti.Pipe(slogformatter.NewFormatterMiddleware(
        slogformatter.PIIFormatter("user"),          // mask PII fields
        slogformatter.ErrorFormatter("error"),       // structured error info
        slogformatter.IPAddressFormatter("client"),  // mask IP addresses
    )).Handler(slog.NewJSONHandler(os.Stdout, nil)),
)
```

主要なフォーマッター: `PIIFormatter`, `ErrorFormatter`, `TimeFormatter`, `UnixTimestampFormatter`, `IPAddressFormatter`, `HTTPRequestFormatter`, `HTTPResponseFormatter`。汎用フォーマッター: `FormatByType[T]`, `FormatByKey`, `FormatByKind`, `FormatByGroup`, `FormatByGroupKey`。ネストされた属性のフラット化には `FlattenFormatterMiddleware` を使用します。

## HTTPミドルウェア

フレームワーク間で一貫したパターン: `router.Use(slogXXX.New(logger))`。

利用可能: `slog-gin`, `slog-echo`, `slog-fiber`, `slog-chi`, `slog-http` (net/http)。

すべてが共通の `Config` 構造体を持ちます: `DefaultLevel`, `ClientErrorLevel`, `ServerErrorLevel`, `WithRequestBody`, `WithResponseBody`, `WithUserAgent`, `WithRequestID`, `WithTraceID`, `WithSpanID`, `Filters`。

```go
// Gin with filters — skip health checks
router.Use(sloggin.NewWithConfig(logger, sloggin.Config{
    DefaultLevel:     slog.LevelInfo,
    ClientErrorLevel: slog.LevelWarn,
    ServerErrorLevel: slog.LevelError,
    WithRequestBody:  true,
    Filters: []sloggin.Filter{
        sloggin.IgnorePath("/health", "/metrics"),
    },
}))
```

フレームワーク固有のセットアップについては [HTTP Middlewares](references/http-middlewares.md) を参照してください。

## バックエンドシンク

すべて `Option{}.NewXxxHandler()` コンストラクタパターンに従います。

| カテゴリ     | パッケージ                                                   |
| ------------ | ---------------------------------------------------------- |
| クラウド        | `slog-datadog`, `slog-sentry`, `slog-loki`, `slog-graylog` |
| メッセージング    | `slog-kafka`, `slog-fluentd`, `slog-logstash`, `slog-nats` |
| 通知 | `slog-slack`, `slog-telegram`, `slog-webhook`              |
| ストレージ      | `slog-parquet`                                             |
| ブリッジ      | `slog-zap`, `slog-zerolog`, `slog-logrus`                  |

**バッチハンドラーにはグレースフルシャットダウンが必要です** — `slog-datadog`, `slog-loki`, `slog-kafka`, `slog-parquet` はレコードを内部的にバッファリングします。シャットダウン時にフラッシュしてください（例: Datadogの `handler.Stop(ctx)`, Lokiの `lokiClient.Stop()`, Kafkaの `writer.Close()`）。そうしないとバッファされたログが失われます。

設定例とシャットダウンパターンについては [Backend Handlers](references/backend-handlers.md) を参照してください。

## よくある間違い

| 間違い | 失敗する理由 | 修正方法 |
| --- | --- | --- |
| フォーマット後にサンプリング | ドロップされるレコードのフォーマットにCPUが無駄に使われる | サンプリングを最も外側のハンドラーに配置する |
| 多数の同期ハンドラーへのFanout | 呼び出し元をブロック — レイテンシはすべてのハンドラーの合計 | 並行ディスパッチに `Pool()` を使用する |
| バッチハンドラーのシャットダウンフラッシュ漏れ | シャットダウン時にバッファされたログが失われる | `defer handler.Stop(ctx)` (Datadog), `defer lokiClient.Stop()` (Loki), `defer writer.Close()` (Kafka) |
| デフォルト/キャッチオールハンドラーなしのRouter | マッチしないレコードが無視される | 述語なしのハンドラーをキャッチオールとして追加する |
| HTTPミドルウェアなしで `AttrFromContext` を使用 | コンテキストにリクエスト属性が含まれていない | まず `slog-gin`/`echo`/`fiber`/`chi` ミドルウェアをインストールする |
| ミドルウェアなしで `Pipe` を使用 | レコードごとのオーバーヘッドが追加されるだけのno-opラッパー | ミドルウェアが不要な場合は `Pipe()` を削除する |

## パフォーマンスに関する警告

- **Fanoutのレイテンシ** = すべてのハンドラーのレイテンシの合計（順次実行）。10msのハンドラーが5つある場合、ログ呼び出しごとに50msかかります。`Pool()` を使ってmax(latencies)に削減してください
- **Pipeミドルウェア** はレコードごとの関数呼び出しオーバーヘッドを追加します — チェーンは短く保ってください（2〜4ミドルウェア）
- **slog-formatter** は属性を順次処理します — フォーマッターが多いと複合的に遅くなります。ホットパスの属性フォーマットには、代わりに型に `slog.LogValuer` を実装することを推奨します
- 本番デプロイ前にパイプラインを `go test -bench` で**ベンチマーク**してください

**診断:** パイプラインのレコードごとのアロケーションとレイテンシを測定し、チェーン内でどのハンドラーが最もアロケーションしているかを特定してください。

## ベストプラクティス

1. **サンプリングを最初に、フォーマットを次に、ルーティングを最後に** — この標準的な順序により無駄な処理が最小化され、すべてのシンクがクリーンなデータを受け取ることが保証されます
2. **横断的関心事にはPipeを使用する** — トレースIDの注入やPIIスクラビングはミドルウェアに属し、ハンドラーごとのロジックには属しません
3. **`slogmulti.NewHandleInlineHandler` でパイプラインをテストする** — 実際のシンクなしで各ステージに到達するレコードを検証できます
4. **`AttrFromContext` を使用して** HTTPミドルウェアからすべてのハンドラーにリクエストスコープの属性を伝播させてください
5. **ハンドラーが異なるレコードサブセットを必要とする場合はFanoutよりRouterを優先する** — Routerは述語を評価し、マッチしないハンドラーをスキップします

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-observability` skill for slog fundamentals (levels, context, handler setup, migration)
- → See `samber/cc-skills-golang@golang-error-handling` skill for the log-or-return rule
- → See `samber/cc-skills-golang@golang-security` skill for PII handling in logs
- → See `samber/cc-skills-golang@golang-samber-oops` skill for structured error context with `samber/oops`

samber/slog-\* パッケージでバグや予期しない動作に遭遇した場合は、該当するリポジトリでissueを作成してください（例: [slog-multi/issues](https://github.com/samber/slog-multi/issues), [slog-sampling/issues](https://github.com/samber/slog-sampling/issues)）。
