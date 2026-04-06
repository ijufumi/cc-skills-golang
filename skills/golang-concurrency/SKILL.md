---
name: golang-concurrency
description: "Golang concurrency patterns. Use when writing or reviewing concurrent Go code involving goroutines, channels, select, locks, sync primitives, errgroup, singleflight, worker pools, or fan-out/fan-in pipelines. Also triggers when you detect goroutine leaks, race conditions, channel ownership issues, or need to choose between channels and mutexes."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "⚡"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a Go concurrency engineer. You assume every goroutine is a liability until proven necessary — correctness and leak-freedom come before performance.

**Modes:**

- **Write mode** — implement concurrent code (goroutines, channels, sync primitives, worker pools, pipelines). Follow the sequential instructions below.
- **Review mode** — reviewing a PR's concurrent code changes. Focus on the diff: check for goroutine leaks, missing context propagation, ownership violations, and unprotected shared state. Sequential.
- **Audit mode** — auditing existing concurrent code across a codebase. Use up to 5 parallel sub-agents as described in the "Parallelizing Concurrency Audits" section.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-concurrency` skill takes precedence.

# Go 並行処理のベストプラクティス

Goの並行性モデルはゴルーチンとチャネル上に構築されている。ゴルーチンは軽量だが無料ではない — 生成するすべてのゴルーチンは管理すべきリソースである。目標は構造化された並行性: すべてのゴルーチンが明確な所有者、予測可能な終了、適切なエラー伝播を持つこと。

## 基本原則

1. **すべてのゴルーチンには明確な終了が必要** — シャットダウンメカニズム（context、doneチャネル、WaitGroup）がなければ、リークして蓄積し、最終的にプロセスがクラッシュする
2. **通信によってメモリを共有する** — チャネルは所有権を明示的に転送する。ミューテックスは共有状態を保護するが所有権が暗黙的になる
3. **チャネルにはポインタではなくコピーを送信** — ポインタの送信は不可視の共有メモリを作り、チャネルの目的を無効にする
4. **チャネルを閉じるのは送信側のみ** — 受信側から閉じると、送信側が書き込んだ後にpanicする
5. **チャネルの方向を指定** (`chan<-`, `<-chan`) — コンパイラがビルド時に誤用を防止する
6. **デフォルトでバッファなしチャネルを使用** — 大きなバッファはバックプレッシャーを隠す。計測された根拠がある場合のみ使用
7. **selectには常に `ctx.Done()` を含める** — ないと呼び出し元のキャンセル後にゴルーチンがリークする
8. **ループ内で `time.After` を使用しない** — 各呼び出しが発火するまで存続するタイマーを作成し、メモリが蓄積する。`time.NewTimer` + `Reset` を使用
9. **テストでゴルーチンリークを追跡** `go.uber.org/goleak` を使用

チャネル/selectの詳細なコード例は [Channels and Select Patterns](references/channels-and-select.md) を参照。

## チャネル vs ミューテックス vs アトミック

| シナリオ | 使用 | 理由 |
| --- | --- | --- |
| ゴルーチン間のデータ受け渡し | Channel | 所有権の転送を明示的に伝達 |
| ゴルーチンライフサイクルの調整 | Channel + context | selectによるクリーンなシャットダウン |
| 共有構造体フィールドの保護 | `sync.Mutex` / `sync.RWMutex` | シンプルなクリティカルセクション |
| 単純なカウンタ、フラグ | `sync/atomic` | ロックフリーでオーバーヘッドが少ない |
| マップの読み取り多数、書き込み少数 | `sync.Map` | 読み取り重視のワークロードに最適化。**マップの並行読み書きはハードクラッシュを引き起こす** |
| コストの高い計算のキャッシュ | `sync.Once` / `singleflight` | 一度だけ実行または重複排除 |

## WaitGroup vs errgroup

| 必要性 | 使用 | 理由 |
| --- | --- | --- |
| ゴルーチンを待機、エラー不要 | `sync.WaitGroup` | ファイア・アンド・フォーゲット |
| 待機 + 最初のエラーを収集 | `errgroup.Group` | エラー伝播 |
| 待機 + 最初のエラーで兄弟をキャンセル | `errgroup.WithContext` | エラー時のcontextキャンセル |
| 待機 + 並行数を制限 | `errgroup.SetLimit(n)` | 組み込みワーカープール |

## 同期プリミティブクイックリファレンス

| プリミティブ | ユースケース | 重要な注意点 |
| --- | --- | --- |
| `sync.Mutex` | 共有状態の保護 | クリティカルセクションを短く保つ。I/Oをまたいで保持しない |
| `sync.RWMutex` | 読み取り多数、書き込み少数 | RLockからLockへのアップグレードは不可（デッドロック） |
| `sync/atomic` | 単純なカウンタ、フラグ | 型付きアトミック（Go 1.19+）を優先: `atomic.Int64`、`atomic.Bool` |
| `sync.Map` | 並行マップ、読み取り重視 | 明示的なロック不要。書き込みが支配的な場合は `RWMutex`+mapを使用 |
| `sync.Pool` | 一時オブジェクトの再利用 | `Put()` の前に常に `Reset()`。GCの負荷を軽減 |
| `sync.Once` | 一度限りの初期化 | Go 1.21+: `OnceFunc`、`OnceValue`、`OnceValues` |
| `sync.WaitGroup` | ゴルーチン完了の待機 | `go` の前に `Add`。Go 1.24+: `wg.Go()` で使用を簡素化 |
| `x/sync/singleflight` | 並行呼び出しの重複排除 | キャッシュスタンピード防止 |
| `x/sync/errgroup` | ゴルーチングループ + エラー | `SetLimit(n)` で手製のワーカープールを置き換え |

詳細な例とアンチパターンは [Sync Primitives Deep Dive](references/sync-primitives.md) を参照。

## 並行処理チェックリスト

ゴルーチンを生成する前に確認する:

- [ ] **どのように終了するか?** — contextキャンセル、チャネルクローズ、または明示的なシグナル
- [ ] **停止を指示できるか?** — `context.Context` またはdoneチャネルを渡す
- [ ] **待機できるか?** — `sync.WaitGroup` または `errgroup`
- [ ] **チャネルの所有者は誰か?** — 作成者/送信者が所有し、閉じる
- [ ] **同期処理にすべきではないか?** — 計測された必要性なしに並行性を追加しない

## パイプラインとワーカープール

パイプラインパターン（ファンアウト/ファンイン、制限付きワーカー、ジェネレータチェーン、Go 1.23+イテレータ、`samber/ro`）については [Pipelines and Worker Pools](references/pipelines.md) を参照。

## 並行処理監査の並列化

大規模コードベースで並行処理を監査する場合、最大5つの並列サブエージェント（Agentツール）を使用:

1. すべてのゴルーチン生成（`go func`、`go method`）を見つけ、シャットダウンメカニズムを検証
2. 同期なしの可変グローバルと共有状態を検索
3. チャネル使用を監査 — 所有権、方向、クローズ、バッファサイズ
4. ループ内の `time.After`、selectでの `ctx.Done()` 欠落、無制限のゴルーチン生成を検出
5. ミューテックスの使用、`sync.Map`、アトミック、スレッドセーフティの文書化をチェック

## よくある間違い

| 間違い | 修正 |
| --- | --- |
| ファイア・アンド・フォーゲットのゴルーチン | 停止メカニズムを提供（context、doneチャネル） |
| 受信側からのチャネルクローズ | 送信側のみがクローズする |
| ホットループ内の `time.After` | `time.NewTimer` + `Reset` を再利用 |
| selectでの `ctx.Done()` 欠落 | キャンセルを許可するために常にcontextをselectする |
| 無制限のゴルーチン生成 | `errgroup.SetLimit(n)` またはセマフォを使用 |
| チャネル経由でのポインタ共有 | コピーまたはイミュータブルな値を送信 |
| ゴルーチン内での `wg.Add` | `go` の前に `Add` を呼ぶ — そうでないと `Wait` が早期に返る可能性がある |
| CIでの `-race` 忘れ | 常に `go test -race ./...` を実行 |
| I/Oをまたぐミューテックス保持 | クリティカルセクションを短く保つ |

## クロスリファレンス

- -> See `samber/cc-skills-golang@golang-performance` skill for false sharing, cache-line padding, `sync.Pool` hot-path patterns
- -> See `samber/cc-skills-golang@golang-context` skill for cancellation propagation and timeout patterns
- -> See `samber/cc-skills-golang@golang-safety` skill for concurrent map access and race condition prevention
- -> See `samber/cc-skills-golang@golang-troubleshooting` skill for debugging goroutine leaks and deadlocks
- -> See `samber/cc-skills-golang@golang-design-patterns` skill for graceful shutdown patterns

## リファレンス

- [Go Concurrency Patterns: Pipelines](https://go.dev/blog/pipelines)
- [Effective Go: Concurrency](https://go.dev/doc/effective_go#concurrency)
