---
name: golang-troubleshooting
description: "Troubleshoot Golang programs systematically - find and fix the root cause. Use when encountering bugs, crashes, deadlocks, or unexpected behavior in Go code. Covers debugging methodology, common Go pitfalls, test-driven debugging, pprof setup and capture, Delve debugger, race detection, GODEBUG tracing, and production debugging. Start here for any 'something is wrong' situation. Not for interpreting profiles or benchmarking (see golang-benchmark skill) or applying optimization patterns (see golang-performance skill)."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🔍"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - dlv
    install:
      - kind: go
        package: github.com/go-delve/delve/cmd/dlv@latest
        bins: [dlv]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Bash(dlv:*) Agent WebFetch WebSearch AskUserQuestion
---

**Persona:** あなたはGoシステムデバッガーです。直感ではなく証拠に従います — 体系的に計装し、再現し、根本原因をトレースします。

**Thinking mode:** Use `ultrathink` for debugging and root cause analysis. Rushed reasoning leads to symptom fixes — deep thinking finds the actual root cause.

**モード:**

- **単一問題デバッグ**（デフォルト）: ゴールデンルールに順次従う — エラーを読む、再現する、一度に一つの仮説。サブエージェントを起動しない。単一の既知の症状には集中した順次調査が速い。
- **コードベースバグハント**（大規模コードベースの明示的な監査）: バグカテゴリごとに1つ、最大5つの並列サブエージェントを起動する（nil/interface、リソース、エラーハンドリング、レース、context/slice/map）。ユーザーが広範囲のスキャンを求める場合に使用し、特定の報告された問題のデバッグには使用しない。

# Goトラブルシューティングガイド

**根本原因調査なしに修正は行わない。** 症状の修正は新しいバグを生み出し時間を無駄にする。このプロセスは時間的プレッシャー下において特に適用される — 急ぐことで解決に時間がかかるカスケード失敗につながる。

ユーザーがGoコードのバグ、クラッシュ、パフォーマンス問題、または予期しない動作を報告した場合:

1. **以下のデシジョンツリーから始める** — 症状カテゴリを特定し関連セクションにジャンプする。
2. **ゴールデンルールに従う** — 特に: 修正前に再現する、一度に一つの仮説、根本原因を見つける。
3. **一般的なデバッグ手法を一歩ずつ進める。** ステップをスキップしない。
4. **自分の推論でレッドフラグに注意する。** 原因を理解せずに修正を推測している場合は止まってより多くの証拠を集める。
5. **ツールを段階的にエスカレートする。** 最もシンプルな診断（`fmt.Println`、テスト分離）から始め、シンプルなツールが不十分な場合のみpprof、Delve、またはGODEBUGに頼る。
6. **説明できない修正は提案しない。** バグが起きる理由を理解していない場合は、そう言ってさらに調査する。

## クイックデシジョンツリー

```
WHAT ARE YOU SEEING?

"Build won't compile"
  → go build ./... 2>&1, go vet ./...
  → See [compilation.md](./references/compilation.md)

"Wrong output / logic bug"
  → Write a failing test → Check error handling, nil, off-by-one
  → See [common-go-bugs.md](./references/common-go-bugs.md), [testing-debug.md](./references/testing-debug.md)

"Random crashes / panics"
  → GOTRACEBACK=all ./app → go test -race ./...
  → See [common-go-bugs.md](./references/common-go-bugs.md), [diagnostic-tools.md](./references/diagnostic-tools.md)

"Sometimes works, sometimes fails"
  → go test -race ./...
  → See [concurrency-debug.md](./references/concurrency-debug.md), [testing-debug.md](./references/testing-debug.md)

"Program hangs / frozen"
  → curl localhost:6060/debug/pprof/goroutine?debug=2
  → See [concurrency-debug.md](./references/concurrency-debug.md), [pprof.md](./references/pprof.md)

"High CPU usage"
  → pprof CPU profiling
  → See [performance-debug.md](./references/performance-debug.md), [pprof.md](./references/pprof.md)

"Memory growing over time"
  → pprof heap profiling
  → See [performance-debug.md](./references/performance-debug.md), [concurrency-debug.md](./references/concurrency-debug.md)

"Slow / high latency / p99 spikes"
  → CPU + mutex + block profiles
  → See [performance-debug.md](./references/performance-debug.md), [diagnostic-tools.md](./references/diagnostic-tools.md)

"Simple bug, easy to reproduce"
  → Write a test, add fmt.Println / log.Debug
  → See [testing-debug.md](./references/testing-debug.md)
```

**覚えておくこと:** エラーを読む → 再現する → 一つのことを計測する → 修正する → 検証する

ほとんどのGoバグは: エラーチェックの欠如、nilポインター、コンテキストキャンセルの忘れ、閉じられていないリソース、レース条件、またはサイレントなエラー飲み込み。

## ゴールデンルール

### 1. まずエラーメッセージを読む

Goのエラーメッセージは正確です。他のことをする前に完全に読む:

- **ファイルと行番号** → そこに直接行く
- **型の不一致** → 関数シグネチャ、インターフェース充足を確認する
- **"undefined"** → インポート、エクスポートされた名前、ビルドタグを確認する
- **"cannot use X as Y"** → 具体的な型とインターフェースを確認する

### 2. 修正前に再現する

推測によるデバッグは絶対にしない — まず再現する。常に:

- バグを捕捉する失敗テストを書く
- 決定論的にする
- 最小限の失敗例を分離する
- `git bisect` を使って壊れたコミットを見つける

### 3. 計測しないなら推測している

パフォーマンスやConcurrencyバグに直感を頼らない:

- **直感よりpprof**
- **推論よりレースデテクター**
- **仮定よりベンチマーク**

### 4. 一度に一つの仮説

一つのことを変更し、計測し、確認する。3つのことを同時に変更すると何も学べない。

### 5. 根本原因を見つける — 回避策なし

症状を隠すバンドエイド修正は許容されない。修正を書く前にバグが起きる**理由**を理解しなければならない。

問題を理解していない場合:

- **症状から起源へデータフローを逆トレースする。**
- **仮定を疑う。** 信頼しているコードが間違っている可能性がある。
- **「なぜ」を5回尋ねる。** 実際の根本原因に到達するまで続ける。
- **より多くのトラブルシューティングチェックを実施する。** さらなるfmt.Println、さらなる出力検査...

### 6. 差分だけでなくコードベースを調査する

バグを指摘したり修正を提案する前に、データフローをトレースしアップストリームの処理を確認する。単独で壊れているように見える関数がコンテキストでは正しいかもしれない — 呼び出し元が入力を検証し、ミドルウェアが不変条件を強制し、または周囲のコードが関数が依存する条件を保証しているかもしれない。

1. **呼び出し元をトレースする** — この関数を誰がどんな値で呼び出すか? すべての呼び出しサイトを見つけるためにGrep/Agentを使用。
2. **アップストリームのバリデーションを確認する** — チェーンの上流での入力パース、型変換、またはガード節が「バグ」を到達不可能にしているかもしれない。
3. **周囲のコードを読む** — ミドルウェア、インターセプター、またはinit関数が関数が依存する状態をセットアップしているかもしれない。

**コンテキストが深刻度を下げるが問題を排除しない場合:** 保護しているアップストリームの保証を説明するノート付きで低優先度として報告する。簡潔なインラインコメントを追加する（例: `// note: safe because caller validates via parseID() which returns uint`）ことで将来のレビュアーに推論がドキュメント化される。

### 7. シンプルに始める

ローカルデバッグには `fmt.Println` が適切なツールな場合もある。シンプルなアプローチが失敗した場合のみツールをエスカレートする。プロダクションデバッグには `fmt.Println` を絶対に使用しない — `slog` を使用する。

## レッドフラグ: デバッグが間違っている

以下のいずれかが起きている場合は止まってStep 1に戻る:

- **「今は素早い修正、後で調査」** — 「後で」は来ない。根本原因を見つける。
- **複数の同時変更** — 一度に一つの仮説。
- **原因を理解せずに修正を提案** — 「ここにnilチェックを追加すればいいかも...」は推測であり、デバッグではない。
- **各修正が新しい問題を明らかにする** — 症状を治療している。実際のバグは別の場所にある。
- **同じ問題に3回以上の修正試行** — 間違ったメンタルモデルを持っている。コードを再読み、データフローをゼロからトレースする。
- **「自分のマシンでは動く」** — 環境の違いを分離していない。
- **フレームワーク/stdlib/コンパイラを責める** — Goのバグであることはほぼない。まず自分のコードを確認する。

## リファレンスファイル

- **[一般的なデバッグ手法](./references/methodology.md)** — 体系的な10ステッププロセス: 症状の定義、再現の分離、一つの仮説の形成、テスト、根本原因の検証、リグレッション防御。エスカレーションガイド: `fmt.Println` からログ、pprof、Delveへのエスカレートタイミングと複数の同時変更のトラップを避ける方法。

- **[GoのよくあるバグS](./references/common-go-bugs.md)** — Goコードをクラッシュさせるバグ: nilポインターデリファレンス、インターフェースnilの落とし穴（typed nil ≠ nil）、変数シャドーイング、slice/map/defer/error/contextの落とし穴、レース条件、JSONアンマーシャリングの驚き、閉じられていないリソース。各々に再現パターンと修正付き。

- **[テスト駆動デバッグ](./references/testing-debug.md)** — なぜ失敗テストを書くことがデバッグの最初のステップなのか。テスト分離技術、失敗の絞り込みのためのテーブル駆動テスト整理、便利な `go test` フラグ（`-v`、`-run`、不安定テスト用 `-count=10`）、不安定テストのデバッグをカバー。

- **[Concurrencyデバッグ](./references/concurrency-debug.md)** — レース条件、デッドロック、goroutineリーク。レースデテクター（`-race`）を使うタイミング、レースデテクター出力の読み方、レースを隠すパターン、`goleak` でのリーク検出、デッドロックの手がかりのためのスタックダンプ分析。

- **[パフォーマンストラブルシューティング](./references/performance-debug.md)** — コードが遅い場合: CPUプロファイリングワークフロー、メモリ分析（ヒープ vs alloc_objectsプロファイル、リーク発見）、ロック競合（mutexプロファイル）、I/Oブロッキング（goroutineプロファイル）。フレームグラフの読み方、ホット関数の特定、ベンチマークでの改善計測。

- **[pprofリファレンス](./references/pprof.md)** — 完全なpprofマニュアル。プロダクションでpprofエンドポイントを有効にする方法（認証付き）、プロファイルタイプ（CPU、ヒープ、goroutine、mutex、block、trace）、ローカルとリモートでのプロファイル取得、インタラクティブな分析コマンド（`top`、`list`、`web`）、フレームグラフの解釈。

- **[診断ツール](./references/diagnostic-tools.md)** — 特定の症状のための補助ツール。GODEBUG環境変数（GCトレーシング、スケジューラートレーシング）、ブレークポイントデバッグのためのDelveデバッガー、エスケープ解析（意図しないヒープアロケーションを見つける `go build -gcflags="-m"`）、goroutineスケジューリングを理解するためのGoの実行トレーサー。

- **[プロダクションデバッグ](./references/production-debug.md)** — 停止せずにライブプロダクションシステムをデバッグする。プロダクションチェックリスト、検索可能なログの構造化、pprofの安全な有効化（認証、ネットワーク分離）、実行中サービスからのプロファイル取得、ネットワークデバッグ（tcpdump、netstat）、HTTPリクエスト/レスポンス検査。

- **[コンパイルの問題](./references/compilation.md)** — ビルド失敗: モジュールバージョン競合、CGOリンクの問題、`go.mod` とインストールされたGoバージョン間のバージョン不一致、クロスコンパイルを妨げるプラットフォーム固有のビルドタグ。

- **[コードレビューのレッドフラグ](./references/code-review-flags.md)** — コードレビュー中に潜在的なバグを示すパターン: 確認されていないエラー、nilチェックの欠如、並行マップアクセス、明確な終了のないgoroutine、ループ内deferからのリソースリーク。

## クロスリファレンス

- ボトルネックの特定後の最適化パターンについては → See `samber/cc-skills-golang@golang-performance` skill
- Goランタイム監視のメトリクス、アラート、Grafanaダッシュボードについては → See `samber/cc-skills-golang@golang-observability` skill
- プロダクションインシデント調査中のPrometheusメトリクスクエリについては → See `samber/cc-skills@promql-cli` skill
- `samber/cc-skills-golang@golang-concurrency`、`samber/cc-skills-golang@golang-safety`、`samber/cc-skills-golang@golang-error-handling` skillを参照
