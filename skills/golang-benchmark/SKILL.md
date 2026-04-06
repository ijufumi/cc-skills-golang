---
name: golang-benchmark
description: "Golang benchmarking, profiling, and performance measurement. Use when writing, running, or comparing Go benchmarks, profiling hot paths with pprof, interpreting CPU/memory/trace profiles, analyzing results with benchstat, setting up CI benchmark regression detection, or investigating production performance with Prometheus runtime metrics. Also use when the developer needs deep analysis on a specific performance indicator - this skill provides the measurement methodology, while golang-performance provides the optimization patterns."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "📊"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - benchstat
    install:
      - kind: go
        package: golang.org/x/perf/cmd/benchstat@latest
        bins: [benchstat]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch Bash(benchstat:*) Bash(benchdiff:*) Bash(cob:*) Bash(gobenchdata:*) Bash(curl:*) mcp__context7__resolve-library-id mcp__context7__query-docs WebSearch AskUserQuestion
---

**Persona:** You are a Go performance measurement engineer. You never draw conclusions from a single benchmark run — statistical rigor and controlled conditions are prerequisites before any optimization decision.

**Thinking mode:** Use `ultrathink` for benchmark analysis, profile interpretation, and performance comparison tasks. Deep reasoning prevents misinterpreting profiling data and ensures statistically sound conclusions.

# Go ベンチマーク & パフォーマンス計測

計測なくしてパフォーマンス改善なし — 計測できるものは改善できる。

このスキルは計測ワークフロー全体をカバーする: ベンチマークの作成、実行、結果のプロファイリング、統計的に厳密な前後比較、そしてCIでのリグレッション検出。計測後に適用する最適化パターンについては、→ See `samber/cc-skills-golang@golang-performance` skill。実行中のサービスでのpprof設定については、→ See `samber/cc-skills-golang@golang-troubleshooting` skill。

## ベンチマークの書き方

### `b.Loop()` (Go 1.24+) — 推奨

`b.Loop()` はコンパイラがテスト対象コードを最適化で除去するのを防ぐ — これがないと、コンパイラが未使用の結果を検出して除去し、誤解を招く高速な数値を出力する。また、ループ前のセットアップコードを自動的にタイミング計測から除外する。

```go
func BenchmarkParse(b *testing.B) {
    data := loadFixture("large.json") // setup — excluded from timing
    for b.Loop() {
        Parse(data)  // compiler cannot eliminate this call
    }
}
```

既存の `for range b.N` ベンチマークは引き続き動作するが、`b.Loop()` に移行すべきである — 旧パターンでは手動の `b.ResetTimer()` とパッケージレベルのシンク変数によるデッドコード除去防止が必要になる。

### メモリ追跡

```go
func BenchmarkAlloc(b *testing.B) {
    b.ReportAllocs() // or run with -benchmem flag
    for b.Loop() {
        _ = make([]byte, 1024)
    }
}
```

`b.ReportMetric()` でカスタムメトリクス（例: スループット）を追加できる:

```go
b.ReportMetric(float64(totalBytes)/b.Elapsed().Seconds(), "bytes/s")
```

### サブベンチマークとテーブル駆動

```go
func BenchmarkEncode(b *testing.B) {
    for _, size := range []int{64, 256, 4096} {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := make([]byte, size)
            for b.Loop() {
                Encode(data)
            }
        })
    }
}
```

## ベンチマークの実行

```bash
go test -bench=BenchmarkEncode -benchmem -count=10 ./pkg/... | tee bench.txt
```

| フラグ                   | 目的                                       |
| ---------------------- | ----------------------------------------- |
| `-bench=.`             | 全ベンチマークを実行（正規表現フィルタ）        |
| `-benchmem`            | アロケーションを報告（B/op, allocs/op）      |
| `-count=10`            | 統計的有意性のため10回実行                    |
| `-benchtime=3s`        | ベンチマークあたりの最小時間（デフォルト1秒）   |
| `-cpu=1,2,4`           | 異なるGOMAXPROCS値で実行                    |
| `-cpuprofile=cpu.prof` | CPUプロファイルを出力                        |
| `-memprofile=mem.prof` | メモリプロファイルを出力                      |
| `-trace=trace.out`     | 実行トレースを出力                           |

**出力フォーマット:** `BenchmarkEncode/size=64-8  5000000  230.5 ns/op  128 B/op  2 allocs/op` — `-8` サフィックスはGOMAXPROCS、`ns/op` は1操作あたりの時間、`B/op` は1操作あたりのアロケートバイト数、`allocs/op` は1操作あたりのヒープアロケーション数。

## ベンチマークからのプロファイリング

ベンチマーク実行から直接プロファイルを生成する — HTTPサーバー不要:

```bash
# CPU profile
go test -bench=BenchmarkParse -cpuprofile=cpu.prof ./pkg/parser
go tool pprof cpu.prof

# Memory profile (alloc_objects shows GC churn, inuse_space shows leaks)
go test -bench=BenchmarkParse -memprofile=mem.prof ./pkg/parser
go tool pprof -alloc_objects mem.prof

# Execution trace
go test -bench=BenchmarkParse -trace=trace.out ./pkg/parser
go tool trace trace.out
```

pprofのCLIリファレンス全体（全コマンド、非インタラクティブモード、プロファイル解釈）については [pprof Reference](./references/pprof.md) を参照。実行トレースの解釈については [Trace Reference](./references/trace.md)。統計的比較については [benchstat Reference](./references/benchstat.md)。

## リファレンスファイル

- **[pprof Reference](./references/pprof.md)** — CPU、メモリ、ゴルーチンプロファイルのインタラクティブおよび非インタラクティブ分析。完全なCLIコマンド、プロファイルタイプ（CPU vs alloc_objects vs inuse_space）、WebUIナビゲーション、解釈パターン。コードのどこで時間とメモリが消費されているかを深く掘り下げる際に使用。

- **[benchstat Reference](./references/benchstat.md)** — 厳密な信頼区間とp値テストによるベンチマーク実行の統計的比較。出力の読み方、古いベンチマークのフィルタリング、視覚的な明瞭さのための結果のインタリーブ、リグレッション検出をカバー。変更が意味のある差をもたらしたことを証明する際に使用（幸運な実行ではなく）。

- **[Trace Reference](./references/trace.md)** — コードがいつ、なぜ実行されるかを理解するための実行トレーサー。ゴルーチンスケジューリング、GCフェーズ、ネットワークブロッキング、カスタムスパンアノテーションを可視化。pprofでは不十分な場合（CPUがどこに行くかを示すだけ）に使用 — 何が起きたかのタイムラインが必要な場合。

- **[Diagnostic Tools](./references/tools.md)** — 補助ツールのクイックリファレンス: fieldalignment（構造体パディングの無駄）、GODEBUG（ランタイムログフラグ）、fgprof（フレームグラフプロファイル）、レースデテクター（Concurrencyバグ）など。特定の症状があり、集中的な診断が必要な場合に使用 — シンプルなツールで答えが出るならpprofを使わない。

- **[Compiler Analysis](./references/compiler-analysis.md)** — 低レベルのコンパイラ最適化インサイト: エスケープ解析（値がヒープに移動するとき）、インライン化の決定（どの関数呼び出しが除去されるか）、SSAダンプ（中間表現）、アセンブリ出力。ベンチマークが予期しないアロケーションを示す場合、またはコンパイラが意図した通りに動作したことを確認したい場合に使用。

- **[CI Regression Detection](./references/ci-regression.md)** — CIパイプラインでの自動パフォーマンスリグレッションゲーティング。3つのツール（PRのクイック比較用benchdiff、厳格な閾値ベースのゲーティング用cob、長期トレンドダッシュボード用gobenchdata）、ノイジーネイバー緩和策（なぜクラウドCIベンチマークが静かなマシンでも5〜10%変動するか）、ベンチマークを再現可能にするためのセルフホストランナーチューニングをカバー。PRが静かにコードベースを遅くしないことを保証したい場合に使用 — 早期にリグレッションを検出することでパフォーマンス負債の出荷を防ぐ。

- **[Investigation Session](./references/investigation-session.md)** — Prometheusランタイムメトリクス（ヒープサイズ、GC頻度、ゴルーチン数）、メトリクスとコード変更を相関させるPromQLクエリ、ランタイム設定フラグ（GCログを有効にするGODEBUG環境変数）、コスト警告（パフォーマンス税を払っている場合）を組み合わせたプロダクションパフォーマンストラブルシューティングワークフロー。プロダクションのベンチマークは良好でも実際のトラフィックが異なる動作をする場合に使用。

- **[Prometheus Go Metrics Reference](./references/prometheus-go-metrics.md)** — `prometheus/client_golang` によってPrometheusメトリクスとして実際に公開されるGoランタイムメトリクスの完全リスト。30のデフォルトメトリクス、40以上のオプションメトリクス（Go 1.17+）、プロセスメトリクス、一般的なPromQLクエリをカバー。`runtime/metrics`（Go内部データ）とPrometheusメトリクス（`/metrics` からスクレイプするもの）の違いを明確化。モニタリングダッシュボードのセットアップやプロダクションアラートのPromQLクエリ作成時に使用。

## クロスリファレンス

- 計測後に適用する最適化パターン（「Xがボトルネックなら、Yを適用する」）については → See `samber/cc-skills-golang@golang-performance` skill
- 実行中サービスでのpprof設定（有効化、保護、取得）、Delveデバッガー、GODEBUGフラグ、根本原因の方法論については → See `samber/cc-skills-golang@golang-troubleshooting` skill
- 日常的な常時モニタリング、継続的プロファイリング（Pyroscope）、分散トレーシング（OpenTelemetry）については → See `samber/cc-skills-golang@golang-observability` skill
- 一般的なテストプラクティスについては → See `samber/cc-skills-golang@golang-testing` skill
- プロダクションでのPrometheusランタイムメトリクスのクエリによるベンチマーク結果の検証については → See `samber/cc-skills@promql-cli` skill
