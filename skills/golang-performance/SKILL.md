---
name: golang-performance
description: "Golang performance optimization patterns and methodology - if X bottleneck, then apply Y. Covers allocation reduction, CPU efficiency, memory layout, GC tuning, pooling, caching, and hot-path optimization. Use when profiling or benchmarks have identified a bottleneck and you need the right optimization pattern to fix it. Also use when performing performance code review to suggest improvements or benchmarks that could help identify quick performance gains. Not for measurement methodology (see golang-benchmark skill) or debugging workflow (see golang-troubleshooting skill)."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🏎️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - benchstat
    install:
      - kind: go
        package: golang.org/x/perf/cmd/benchstat@latest
        bins: [benchstat]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch Bash(benchstat:*) Bash(fieldalignment:*) Bash(staticcheck:*) Bash(curl:*) Bash(fgprof:*) Bash(perf:*) WebSearch AskUserQuestion
---

**Persona:** あなたはGoパフォーマンスエンジニアです。まずプロファイリングなしには最適化しない — 計測し、仮説を立て、一つ変更し、再計測する。

**Thinking mode:** Use `ultrathink` for performance optimization. Shallow analysis misidentifies bottlenecks — deep reasoning ensures the right optimization is applied to the right problem.

**モード:**

- **レビューモード（アーキテクチャ）** — パッケージやサービス全体で構造的なアンチパターン（コネクションプールの欠如、無制限のゴルーチン、誤ったデータ構造）を広範囲にスキャンする。3つの並列サブエージェントに分割: (1) アロケーションとメモリレイアウト、(2) I/OとConcurrency、(3) アルゴリズムの複雑さとキャッシュ。
- **レビューモード（ホットパス）** — 呼び出し元が特定した単一の関数またはタイトなループの集中分析。順次処理し、サブエージェント1つで十分。
- **最適化モード** — プロファイリングでボトルネックが特定された場合。反復サイクル（メトリクス定義 → ベースライン → 診断 → 改善 → 比較）を順次実行 — 一度に一つの変更が原則。

# Go パフォーマンス最適化

## コア哲学

1. **最適化前にプロファイリング** — ボトルネックに関する直感は約80%の確率で間違い。pprofを使って実際のホットスポットを見つける（→ See `samber/cc-skills-golang@golang-troubleshooting` skill）
2. **アロケーション削減が最大のROI** — GoのGCは速いが無料ではない。リクエストあたりのアロケーション削減は、CPUのマイクロ最適化より重要なことが多い
3. **最適化をドキュメント化** — パターンが速い理由を説明するコードコメントを追加し、可能であればベンチマーク数値も記載する。将来の読者は「不要な」最適化を元に戻さないためにコンテキストが必要

## まず外部ボトルネックを排除する

Goコードを最適化する前に、ボトルネックがプロセス内にあることを確認する — レイテンシの90%が遅いDBクエリやAPIコールなら、アロケーション削減は役に立たない。

**診断:** 1- `fgprof` — オンCPUとオフCPU（I/O待ち）時間を取得。オフCPUが支配的なら、ボトルネックは外部 2- `go tool pprof`（goroutineプロファイル） — `net.(*conn).Read` や `database/sql` でブロックされるゴルーチンが多い = 外部待ち 3- 分散トレーシング（OpenTelemetry） — スパンの内訳で遅い上流を特定

**外部の場合:** そのコンポーネントを最適化する — クエリチューニング、キャッシュ、コネクションプール、サーキットブレーカー（→ See `samber/cc-skills-golang@golang-database` skill、[キャッシュパターン](references/caching.md)）。

## 反復的な最適化手法

### サイクル: 目標定義 → ベンチマーク → 診断 → 改善 → ベンチマーク

1. **メトリクスを定義する** — レイテンシ、スループット、メモリ、CPU? 目標がなければ最適化はランダム
2. **アトミックなベンチマークを書く** — 結果の汚染を防ぐためにベンチマークごとに一つの関数を分離する（→ See `samber/cc-skills-golang@golang-benchmark` skill）
3. **ベースラインを計測する** — `go test -bench=BenchmarkMyFunc -benchmem -count=6 ./pkg/... | tee /tmp/report-1.txt`
4. **診断する** — 各詳細セクションの **診断:** 行を使って適切なツールを選択
5. **改善する** — 説明コメント付きで一度に一つの最適化を適用
6. **比較する** — `benchstat /tmp/report-1.txt /tmp/report-2.txt` で統計的有意性を確認
7. **繰り返す** — レポート番号をインクリメントし、次のボトルネックに取り組む

カスタムソリューションを発明する前に、既知のパターンについてライブラリドキュメントを参照すること。すべての `/tmp/report-*.txt` ファイルを監査証跡として保持する。

## デシジョンツリー: 時間はどこで消費されているか?

| ボトルネック | シグナル（pprofより） | アクション |
| --- | --- | --- |
| アロケーション過多 | ヒーププロファイルで `alloc_objects` が高い | [メモリ最適化](references/memory.md) |
| CPUバウンドなホットループ | 関数がCPUプロファイルを支配 | [CPU最適化](references/cpu.md) |
| GCポーズ / OOM | 高GC%、コンテナ制限 | [ランタイムチューニング](references/runtime.md) |
| ネットワーク / I/Oレイテンシ | I/Oでブロックされるゴルーチン | [I/Oとネットワーキング](references/io-networking.md) |
| 繰り返しの高コスト処理 | 同じ計算/フェッチを複数回 | [キャッシュパターン](references/caching.md) |
| 誤ったアルゴリズム | O(n)があるのにO(n²) | [アルゴリズムの複雑さ](references/caching.md#algorithmic-complexity) |
| ロック競合 | mutex/blockプロファイルが高い | → See `samber/cc-skills-golang@golang-concurrency` skill |
| 遅いクエリ | DBの時間がトレースを支配 | → See `samber/cc-skills-golang@golang-database` skill |

## よくある間違い

| 間違い | 修正 |
| --- | --- |
| プロファイリングなしで最適化 | まずpprofでプロファイリング — 直感は約80%の確率で間違い |
| Transportなしのデフォルト `http.Client` | `MaxIdleConnsPerHost` のデフォルトは2。Concurrencyレベルに合わせて設定 |
| ホットループ内でのログ | ログ呼び出しはインライン化を妨げ、レベルが無効でもアロケートする。`slog.LogAttrs` を使用 |
| 制御フローとしての `panic`/`recover` | panicはスタックトレースをアロケートしスタックを巻き戻す。エラーリターンを使用 |
| ベンチマーク証明なしの `unsafe` | 検証されたホットパスでプロファイリングが10%以上の改善を示す場合のみ正当化される |
| コンテナでGCチューニングなし | OOMキルを防ぐため `GOMEMLIMIT` をコンテナメモリの80-90%に設定 |
| プロダクションでの `reflect.DeepEqual` | 型付き比較より50〜200倍遅い。`slices.Equal`、`maps.Equal`、`bytes.Equal` を使用 |

## 詳細ガイド

- [メモリ最適化](references/memory.md) — アロケーションパターン、バッキング配列リーク、sync.Pool、struct alignment
- [CPU最適化](references/cpu.md) — インライン化、キャッシュ局所性、false sharing、ILP、リフレクション回避
- [I/Oとネットワーキング](references/io-networking.md) — HTTPトランスポート設定、ストリーミング、JSONパフォーマンス、cgo、バッチ処理
- [ランタイムチューニング](references/runtime.md) — GOGC、GOMEMLIMIT、GC診断、GOMAXPROCS、PGO
- [キャッシュパターン](references/caching.md) — アルゴリズムの複雑さ、コンパイル済みパターン、singleflight、処理の回避
- [プロダクション可観測性](references/observability.md) — Prometheusメトリクス、PromQLクエリ、継続的プロファイリング、アラートルール

## CIリグレッション検出

CIでベンチマーク比較を自動化し、リグレッションがプロダクションに届く前に検出する。`benchdiff` と `cob` のセットアップについては → See `samber/cc-skills-golang@golang-benchmark` skill。

## クロスリファレンス

- ベンチマーク手法、`benchstat`、`b.Loop()` (Go 1.24+) については → See `samber/cc-skills-golang@golang-benchmark` skill
- pprofワークフロー、エスケープ解析診断、パフォーマンスデバッグについては → See `samber/cc-skills-golang@golang-troubleshooting` skill
- スライス/マップの事前アロケーションと `strings.Builder` については → See `samber/cc-skills-golang@golang-data-structures` skill
- ワーカープール、`sync.Pool` API、ゴルーチンライフサイクル、ロック競合については → See `samber/cc-skills-golang@golang-concurrency` skill
- ループ内のdeferとスライスバッキング配列エイリアシングについては → See `samber/cc-skills-golang@golang-safety` skill
- コネクションプールチューニングとバッチ処理については → See `samber/cc-skills-golang@golang-database` skill
- プロダクションでの継続的プロファイリングについては → See `samber/cc-skills-golang@golang-observability` skill
