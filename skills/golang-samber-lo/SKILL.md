---
name: golang-samber-lo
description: "Functional programming helpers for Golang using samber/lo — 500+ type-safe generic functions for slices, maps, channels, strings, math, tuples, and concurrency (Map, Filter, Reduce, GroupBy, Chunk, Flatten, Find, Uniq, etc.). Core immutable package (lo), concurrent variants (lo/parallel aka lop), in-place mutations (lo/mutable aka lom), lazy iterators (lo/it aka loi for Go 1.23+), and experimental SIMD (lo/exp/simd). Apply when using or adopting samber/lo, when the codebase imports github.com/samber/lo, or when implementing functional-style data transformations in Go. Not for streaming pipelines (→ See golang-samber-ro skill)."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.0.3"
  openclaw:
    emoji: "🧰"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** You are a Go engineer who prefers declarative collection transforms over manual loops. You reach for `lo` to eliminate boilerplate, but you know when the stdlib is enough and when to upgrade to `lop`, `lom`, or `loi`.

# samber/lo — Go向け関数型ユーティリティ

Lodashにインスパイアされた、ジェネリクスファーストのユーティリティライブラリ。スライス、マップ、文字列、数学、チャネル、タプル、並行処理のための500以上の型安全なヘルパー。外部依存なし。デフォルトでイミュータブル。

**公式リソース:**

- [github.com/samber/lo](https://github.com/samber/lo)
- [lo.samber.dev](https://lo.samber.dev)
- [pkg.go.dev/github.com/samber/lo](https://pkg.go.dev/github.com/samber/lo)

このスキルは網羅的ではありません。最新のAPIシグネチャや使用方法については、ライブラリの公式ドキュメントとコード例を参照してください。Context7がディスカバリプラットフォームとして役立ちます。

## なぜ samber/lo か

Go標準ライブラリの `slices` と `maps` パッケージは基本的なヘルパー約10個（sort、contains、keys）しかカバーしていない。それ以外のMap、Filter、Reduce、GroupBy、Chunk、Flatten、Zipにはすべて手動のforループが必要。`lo` はこのギャップを埋める:

- **型安全なジェネリクス** — `interface{}` キャスト不要、リフレクション不要、コンパイル時チェック、インターフェースボクシングのオーバーヘッドなし
- **デフォルトでイミュータブル** — 新しいコレクションを返すため、並行読み取りに安全で推論が容易
- **合成可能** — 関数はスライス/マップを受け取り返すため、ラッパー型なしでチェーン可能
- **依存関係ゼロ** — Go標準ライブラリのみ、推移的依存リスクなし
- **段階的な複雑さ** — `lo` から始めて、プロファイリングで必要と判明した場合のみ `lop`/`lom`/`loi` にアップグレード
- **エラーバリアント** — ほとんどの関数に `Err` サフィックス（`MapErr`、`FilterErr`、`ReduceErr`）があり、最初のエラーで停止

## インストール

```bash
go get github.com/samber/lo
```

| パッケージ | インポート | エイリアス | Goバージョン |
| --- | --- | --- | --- |
| コア（イミュータブル） | `github.com/samber/lo` | `lo` | 1.18+ |
| 並列 | `github.com/samber/lo/parallel` | `lop` | 1.18+ |
| ミュータブル | `github.com/samber/lo/mutable` | `lom` | 1.18+ |
| イテレータ | `github.com/samber/lo/it` | `loi` | 1.23+ |
| SIMD（実験的） | `github.com/samber/lo/exp/simd` | — | 1.25+（amd64のみ） |

## 適切なパッケージの選択

`lo` から始める。プロファイリングでボトルネックが判明した場合、または遅延評価が明示的に必要な場合のみ他のパッケージに移行する。

| パッケージ | 使うべき場面 | トレードオフ |
| --- | --- | --- |
| `lo` | すべての変換のデフォルト | 新しいコレクションを割り当てる（安全、予測可能） |
| `lop` | 大規模データセット（1000以上のアイテム）でのCPUバウンド処理 | goroutineオーバーヘッド、I/Oや小さいスライスには不向き |
| `lom` | `pprof -alloc_objects` で確認されたホットパス | 入力をミューテートする — 呼び出し元は副作用を理解する必要あり |
| `loi` | チェーン変換を伴う大規模データセット（Go 1.23+） | 遅延評価でメモリを節約するが、イテレータの複雑さが増す |
| `simd` | ベンチマーク後の数値バルク演算（実験的） | 不安定なAPI、バージョン間で破壊的変更の可能性あり |

**重要なルール:**

- `lop` はCPU並列性のためであり、I/O並行性のためではない — I/Oファンアウトには代わりに `errgroup` を使う
- `lom` はイミュータビリティを壊す — アロケーションプレッシャーが計測された場合のみ使用し、推測で使わない
- `loi` は `Map → Filter → Take` のようなチェーンで遅延評価により中間アロケーションを排除する
- 無限イベントストリーム上のリアクティブ/ストリーミングパイプラインには、→ see `samber/cc-skills-golang@golang-samber-ro` skill + `samber/ro` パッケージ

詳細なパッケージ比較とデシジョンフローチャートについては [Package Guide](./references/package-guide.md) を参照。

## コアパターン

### スライスの変換

```go
// ✓ lo — declarative, type-safe
names := lo.Map(users, func(u User, _ int) string {
    return u.Name
})

// ✗ Manual — boilerplate, error-prone
names := make([]string, 0, len(users))
for _, u := range users {
    names = append(names, u.Name)
}
```

### Filter + Reduce

```go
total := lo.Reduce(
    lo.Filter(orders, func(o Order, _ int) bool {
        return o.Status == "paid"
    }),
    func(sum float64, o Order, _ int) float64 {
        return sum + o.Amount
    },
    0,
)
```

### GroupBy

```go
byStatus := lo.GroupBy(tasks, func(t Task, _ int) string {
    return t.Status
})
// map[string][]Task{"open": [...], "closed": [...]}
```

### エラーバリアント — 最初のエラーで停止

```go
results, err := lo.MapErr(urls, func(url string, _ int) (Response, error) {
    return http.Get(url)
})
```

## よくある間違い

| 間違い | なぜ問題か | 修正方法 |
| --- | --- | --- |
| `slices.Contains` が存在するのに `lo.Contains` を使う | 標準ライブラリでカバーされている操作に不要な依存を追加 | Go 1.21+ では `slices.Contains`、`slices.Sort`、`maps.Keys` を優先 |
| 10アイテムに `lop.Map` を使う | goroutine作成のオーバーヘッドが変換コストを上回る | `lo.Map` を使う — `lop` の利点はCPUバウンド処理で約1000以上のアイテムから |
| `lo.Filter` が入力を変更すると想定する | `lo` はデフォルトでイミュータブル — 新しいスライスを返す | インプレースミューテーションが明示的に必要な場合は `lom.Filter` を使う |
| 本番コードパスで `lo.Must` を使う | `Must` はエラー時にpanicする — テストとinitでは問題ないが、リクエストハンドラでは危険 | Must以外のバリアントを使いエラーをハンドリングする |
| 大規模データでイーガーな変換を多数チェーンする | 各ステップが中間スライスを割り当てる | `loi`（遅延イテレータ）を使い中間アロケーションを回避する |

## ベストプラクティス

1. **標準ライブラリが利用可能な場合はそちらを優先** — `slices.Contains`、`slices.Sort`、`maps.Keys` は依存なし。標準ライブラリにない変換（Map、Filter、Reduce、GroupBy、Chunk、Flatten）には `lo` を使う
2. **lo関数を合成する** — ネストしたループを書く代わりに `lo.Filter` → `lo.Map` → `lo.GroupBy` をチェーンする。各関数はビルディングブロック
3. **最適化前にプロファイルする** — `go tool pprof` でアロケーションまたはCPUがボトルネックと確認された後のみ `lo` から `lom`/`lop` に切り替える
4. **エラーバリアントを使う** — `lo.Map` + 手動エラー収集よりも `lo.MapErr` を優先する。エラーバリアントは早期停止しクリーンに伝播する
5. **`lo.Must` はテストとinitのみで使う** — 本番ではエラーを明示的にハンドリングする

## クイックリファレンス

| 関数 | 機能 |
| --- | --- |
| `lo.Map` | 各要素を変換 |
| `lo.Filter` / `lo.Reject` | 述語に一致する要素を保持/除外 |
| `lo.Reduce` | 要素を単一の値に畳み込む |
| `lo.ForEach` | 副作用付きイテレーション |
| `lo.GroupBy` | キーで要素をグループ化 |
| `lo.Chunk` | 固定サイズのバッチに分割 |
| `lo.Flatten` | ネストしたスライスを1レベル平坦化 |
| `lo.Uniq` / `lo.UniqBy` | 重複を除去 |
| `lo.Find` / `lo.FindOrElse` | 最初の一致またはデフォルト |
| `lo.Contains` / `lo.Every` / `lo.Some` | メンバーシップテスト |
| `lo.Keys` / `lo.Values` | マップのキーまたは値を抽出 |
| `lo.PickBy` / `lo.OmitBy` | マップエントリをフィルタ |
| `lo.Zip2` / `lo.Unzip2` | 2つのスライスをペア/アンペア |
| `lo.Range` / `lo.RangeFrom` | 数列を生成 |
| `lo.Ternary` / `lo.If` | インライン条件式 |
| `lo.ToPtr` / `lo.FromPtr` | ポインタヘルパー |
| `lo.Must` / `lo.Try` | エラー時panic / boolとしてリカバー |
| `lo.Async` / `lo.Attempt` | 非同期実行 / バックオフ付きリトライ |
| `lo.Debounce` / `lo.Throttle` | レート制限 |
| `lo.ChannelDispatcher` | 複数チャネルへのファンアウト |

完全な関数カタログ（300以上の関数）については [API Reference](./references/api-reference.md) を参照。

合成パターン、標準ライブラリとの相互運用、イテレータパイプラインについては [Advanced Patterns](./references/advanced-patterns.md) を参照。

samber/lo でバグや予期しない動作に遭遇した場合は、[github.com/samber/lo/issues](https://github.com/samber/lo/issues) で Issue を作成してください。

## 相互参照

- → See `samber/cc-skills-golang@golang-samber-ro` skill for reactive/streaming pipelines over infinite event streams (`samber/ro` package)
- → See `samber/cc-skills-golang@golang-samber-mo` skill for monadic types (Option, Result, Either) that compose with lo transforms
- → See `samber/cc-skills-golang@golang-data-structures` skill for choosing the right underlying data structure
- → See `samber/cc-skills-golang@golang-performance` skill for profiling methodology before switching to `lom`/`lop`
