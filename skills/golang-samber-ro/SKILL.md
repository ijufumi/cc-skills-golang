---
name: golang-samber-ro
description: "Reactive streams and event-driven programming in Golang using samber/ro — ReactiveX implementation with 150+ type-safe operators, cold/hot observables, 5 subject types (Publish, Behavior, Replay, Async, Unicast), declarative pipelines via Pipe, 40+ plugins (HTTP, cron, fsnotify, JSON, logging), automatic backpressure, error propagation, and Go context integration. Apply when using or adopting samber/ro, when the codebase imports github.com/samber/ro, or when building asynchronous event-driven pipelines, real-time data processing, streams, or reactive architectures in Go. Not for finite slice transforms (-> See golang-samber-lo skill)."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.0.3"
  openclaw:
    emoji: "👁️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** あなたはデータが非同期または無限に流れる場合にリアクティブストリームを使うGoエンジニアです。samber/roを使って手動のgoroutine/channelの配線の代わりに宣言的なパイプラインを構築しますが、シンプルなスライス + samber/loで十分な場合も知っています。

**Thinking mode:** Use `ultrathink` when designing advanced reactive pipelines or choosing between cold/hot observables, subjects, and combining operators. Wrong architecture leads to resource leaks or missed events.

# samber/ro — GoのReactive Streams

[ReactiveX](https://reactivex.io/) のGo実装。ジェネリクスファースト、型安全、自動バックプレッシャー、エラー伝播、コンテキスト統合、リソースクリーンアップを備えた非同期データストリームの合成可能パイプライン。150以上のオペレーター、5つのSubjectタイプ、40以上のプラグイン。

**公式リソース:**

- [github.com/samber/ro](https://github.com/samber/ro)
- [ro.samber.dev](https://ro.samber.dev)
- [pkg.go.dev/github.com/samber/ro](https://pkg.go.dev/github.com/samber/ro)

このスキルは網羅的ではありません。最新のAPIシグネチャと使用パターンについては、ライブラリの公式ドキュメントとコード例を参照してください。Context7をディスカバリープラットフォームとして活用できます。

## samber/roを使う理由（ストリーム vs スライス）

Goのchannels + goroutinesは複雑な非同期パイプラインで扱いにくくなる: 手動のチャネルクローズ、冗長なgoroutineライフサイクル、ネストされたselectを通じたエラー伝播、合成可能なオペレーターがない。`samber/ro` は宣言的でチェーン可能なストリームオペレーターでこれを解決する。

**どのツールをいつ使うか:**

| シナリオ | ツール | 理由 |
| --- | --- | --- |
| スライスの変換（map、filter、reduce） | `samber/lo` | 有限、同期、先行評価 — ストリームオーバーヘッドは不要 |
| エラーハンドリング付きシンプルなgoroutineファンアウト | `errgroup` | 標準lib、軽量、有界Concurrencyに十分 |
| 無限イベントストリーム（WebSocket、ティッカー、ファイルウォッチャー） | `samber/ro` | バックプレッシャー、リトライ、タイムアウト、結合を持つ宣言的パイプライン |
| 複数の非同期ソースからのリアルタイムデータエンリッチメント | `samber/ro` | CombineLatest/Zipが手動selectなしで依存ストリームを合成する |
| 一つのソースを共有する複数のコンシューマーのPub/sub | `samber/ro` | ホットObservable（Share/Subjects）がマルチキャストをネイティブに処理する |

**主要な違い: lo vs ro**

| 側面 | `samber/lo` | `samber/ro` |
| --- | --- | --- |
| データ | 有限スライス | 無限ストリーム |
| 実行 | 同期、ブロッキング | 非同期、ノンブロッキング |
| 評価 | 先行評価（中間スライスをアロケート） | 遅延評価（アイテムが到着する際に処理） |
| タイミング | 即時 | 時間認識（delay、throttle、interval、timeout） |
| エラーモデル | 呼び出しごとに `(T, error)` を返す | エラーチャネルがパイプラインを伝播 |
| ユースケース | コレクション変換 | イベント駆動、リアルタイム、非同期パイプライン |

## インストール

```bash
go get github.com/samber/ro
```

## コアコンセプト

4つの構成要素:

1. **Observable** — 時間の経過とともに値を発行するデータソース。デフォルトでコールド: 各サブスクライバーが独立した実行をゼロからトリガーする
2. **Observer** — 3つのコールバックを持つコンシューマー: `onNext(T)`、`onError(error)`、`onComplete()`
3. **Operator** — ObservableをObservableに変換する関数。`Pipe` でチェーンする
4. **Subscription** — ObservableとObserverの接続。`.Wait()` でブロックするか `.Unsubscribe()` でキャンセルする

```go
observable := ro.Pipe2(
    ro.RangeWithInterval(0, 5, 1*time.Second),
    ro.Filter(func(x int) bool { return x%2 == 0 }),
    ro.Map(func(x int) string { return fmt.Sprintf("even-%d", x) }),
)

observable.Subscribe(ro.NewObserver(
    func(s string) { fmt.Println(s) },      // onNext
    func(err error) { log.Println(err) },    // onError
    func() { fmt.Println("Done!") },         // onComplete
))
// Output: "even-0", "even-2", "even-4", "Done!"

// Or collect synchronously:
values, err := ro.Collect(observable)
```

## コールド vs ホットObservable

**コールド**（デフォルト）: 各 `.Subscribe()` が新しい独立した実行を開始する。安全で予測可能 — デフォルトで使用。

**ホット**: 複数のサブスクライバーが単一の実行を共有する。ソースが高コスト（WebSocket、DBポーリング）の場合、またはサブスクライバーが同じイベントを見なければならない場合に使用。

| 変換方法 | 動作 |
| --- | --- |
| `Share()` | コールド → リファレンスカウント付きホット。最後のアンサブスクライブで終了 |
| `ShareReplay(n)` | Shareと同じ + 遅延サブスクライバーのために最後のN値をバッファリング |
| `Connectable()` | コールド → ホット、ただし明示的な `.Connect()` 呼び出しを待つ |
| Subjects | ネイティブホット — `.Send()`、`.Error()`、`.Complete()` を直接呼び出す |

| Subject | コンストラクター | リプレイ動作 |
| --- | --- | --- |
| `PublishSubject` | `NewPublishSubject[T]()` | なし — 遅延サブスクライバーは過去のイベントを見逃す |
| `BehaviorSubject` | `NewBehaviorSubject[T](initial)` | 新しいサブスクライバーに最後の値をリプレイ |
| `ReplaySubject` | `NewReplaySubject[T](bufferSize)` | 最後のN値をリプレイ |
| `AsyncSubject` | `NewAsyncSubject[T]()` | 完了時にのみ最後の値を発行 |
| `UnicastSubject` | `NewUnicastSubject[T](bufferSize)` | 単一サブスクライバーのみ |

Subjectの詳細とホットObservableパターンについては [Subjects Guide](./references/subjects-guide.md) を参照。

## オペレータークイックリファレンス

| カテゴリ | 主要オペレーター | 目的 |
| --- | --- | --- |
| 生成 | `Just`、`FromSlice`、`FromChannel`、`Range`、`Interval`、`Defer`、`Future` | 様々なソースからObservableを作成 |
| 変換 | `Map`、`MapErr`、`FlatMap`、`Scan`、`Reduce`、`GroupBy` | ストリーム値の変換または蓄積 |
| フィルター | `Filter`、`Take`、`TakeLast`、`Skip`、`Distinct`、`Find`、`First`、`Last` | 値を選択的に発行 |
| 結合 | `Merge`、`Concat`、`Zip2`〜`Zip6`、`CombineLatest2`〜`CombineLatest5`、`Race` | 複数のObservableをマージ |
| エラー | `Catch`、`OnErrorReturn`、`OnErrorResumeNextWith`、`Retry`、`RetryWithConfig` | エラーからの回復 |
| タイミング | `Delay`、`DelayEach`、`Timeout`、`ThrottleTime`、`SampleTime`、`BufferWithTime` | 発行タイミングの制御 |
| 副作用 | `Tap`/`Do`、`TapOnNext`、`TapOnError`、`TapOnComplete` | ストリームを変更せずに観察 |
| ターミナル | `Collect`、`ToSlice`、`ToChannel`、`ToMap` | ストリームをGoの型に変換 |

オペレーターチェーン全体でコンパイル時型安全性のために、型付き `Pipe2`、`Pipe3` ... `Pipe25` を使用する。型なし `Pipe` は `any` を使用し型チェックを失う。

完全なオペレーターカタログ（シグネチャ付き150以上のオペレーター）については [Operators Guide](./references/operators-guide.md) を参照。

## よくある間違い

| 間違い | なぜ失敗するか | 修正 |
| --- | --- | --- |
| エラーハンドラーなしで `ro.OnNext()` を使用 | エラーがサイレントにドロップされる — バグがプロダクションで隠れる | 3つのコールバック全てで `ro.NewObserver(onNext, onError, onComplete)` を使用 |
| `Pipe2`/`Pipe3` の代わりに型なし `Pipe()` を使用 | コンパイル時型安全性を失い、エラーが実行時に表面化する | 型付きオペレーターチェーンに `Pipe2`、`Pipe3`...`Pipe25` を使用 |
| 無限ストリームで `.Unsubscribe()` を忘れる | goroutineリーク — Observableが永遠に実行し続ける | `TakeUntil(signal)`、コンテキストキャンセル、または明示的な `Unsubscribe()` を使用 |
| コールドで十分な場合に `Share()` を使用 | 不必要な複雑さ、ライフサイクルの推論が難しくなる | 複数のコンシューマーが同じストリームを必要とする場合のみホットObservableを使用 |
| 有限スライス変換に `samber/ro` を使用 | 同期操作にストリームオーバーヘッド（goroutines、subscriptions） | `samber/lo` を使用 — よりシンプルで速く、スライス専用 |
| キャンセルのためにコンテキストを伝播しない | ストリームがシャットダウンシグナルを無視し、終了時にリソースリークを引き起こす | パイプラインに `ContextWithTimeout` または `ThrowOnContextCancel` をチェーンする |

## ベストプラクティス

1. **常に3つのイベント全てを処理する** — `OnNext` だけでなく `NewObserver(onNext, onError, onComplete)` を使用する。未処理のエラーはサイレントなデータロスを引き起こす
2. **同期消費に `Collect()` を使用する** — ストリームが有限で `[]T` が必要な場合、`Collect` は完了までブロックしてスライス + エラーを返す
3. **型付きPipe関数を優先する** — `Pipe2`、`Pipe3`...`Pipe25` はコンパイル時に型の不一致を検出する。動的オペレーターチェーンには型なし `Pipe` を予約する
4. **無限ストリームを制限する** — `Take(n)`、`TakeUntil(signal)`、`Timeout(d)`、またはコンテキストキャンセルを使用する。制限のないストリームはgoroutineをリークする
5. **可観測性に `Tap`/`Do` を使用する** — ストリームを変更せずにログ、トレース、または計測する。エラーモニタリングに `TapOnError` をチェーンする
6. **シンプルな変換には `samber/lo` を優先する** — データが有限スライスでMap/Filter/Reduceが必要な場合は `lo` を使用する。データが時間の経過で到着する、複数のソースからの、またはリトライ/タイムアウト/バックプレッシャーが必要な場合に `ro` を使う

## プラグインエコシステム

40以上のプラグインがroをドメイン固有のオペレーターで拡張する:

| カテゴリ | プラグイン | インポートパスプレフィックス |
| --- | --- | --- |
| エンコーディング | JSON、CSV、Base64、Gob | `plugins/encoding/...` |
| ネットワーク | HTTP、I/O、FSNotify | `plugins/http`、`plugins/io`、`plugins/fsnotify` |
| スケジューリング | Cron、ICS | `plugins/cron`、`plugins/ics` |
| 可観測性 | Zap、Slog、Zerolog、Logrus、Sentry、Oops | `plugins/observability/...`、`plugins/samber/oops` |
| レート制限 | Native、Ulule | `plugins/ratelimit/...` |
| データ | Bytes、Strings、Sort、Strconv、Regexp、Template | `plugins/bytes`、`plugins/strings` など |
| システム | Process、Signal | `plugins/proc`、`plugins/signal` |

インポートパスと使用例を含む完全なプラグインカタログについては [Plugin Ecosystem](./references/plugin-ecosystem.md) を参照。

リアルワールドのリアクティブパターン（retry+timeout、WebSocketファンアウト、グレースフルシャットダウン、ストリーム結合）については [Patterns](./references/patterns.md) を参照。

samber/roでバグや予期しない動作が発生した場合は [github.com/samber/ro/issues](https://github.com/samber/ro/issues) でissueを作成してください。

## クロスリファレンス

- 有限スライス変換（Map、Filter、Reduce、GroupBy）については → See `samber/cc-skills-golang@golang-samber-lo` skill — データがすでにスライスにある場合はloを使用
- roパイプラインと合成するモナド型（Option、Result、Either）については → See `samber/cc-skills-golang@golang-samber-mo` skill
- インメモリキャッシュ（roプラグインとしても利用可能）については → See `samber/cc-skills-golang@golang-samber-hot` skill
- リアクティブストリームが過剰な場合のgoroutine/channelパターンについては → See `samber/cc-skills-golang@golang-concurrency` skill
- プロダクションでのリアクティブパイプライン監視については → See `samber/cc-skills-golang@golang-observability` skill
