---
name: golang-samber-hot
description: "In-memory caching in Golang using samber/hot — eviction algorithms (LRU, LFU, TinyLFU, W-TinyLFU, S3FIFO, ARC, TwoQueue, SIEVE, FIFO), TTL, cache loaders, sharding, stale-while-revalidate, missing key caching, and Prometheus metrics. Apply when using or adopting samber/hot, when the codebase imports github.com/samber/hot, or when the project repeatedly loads the same medium-to-low cardinality resources at high frequency and needs to reduce latency or backend pressure."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.0.3"
  openclaw:
    emoji: "🔥"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs AskUserQuestion
---

**Persona:** You are a Go engineer who treats caching as a system design decision. You choose eviction algorithms based on measured access patterns, size caches from working-set data, and always plan for expiration, loader failures, and monitoring.

# samber/hot を使った Go のインメモリキャッシュ

Go 1.22+ 向けの汎用・型安全なインメモリキャッシュライブラリ。9種類のエビクションアルゴリズム、TTL、singleflight重複排除付きローダーチェーン、シャーディング、stale-while-revalidate、Prometheusメトリクスに対応。

**公式リソース:**

- [pkg.go.dev/github.com/samber/hot](https://pkg.go.dev/github.com/samber/hot)
- [github.com/samber/hot](https://github.com/samber/hot)

このスキルは網羅的ではありません。最新のAPIシグネチャや使用方法については、ライブラリの公式ドキュメントとコード例を参照してください。Context7がディスカバリプラットフォームとして役立ちます。

```bash
go get -u github.com/samber/hot
```

## アルゴリズム選択

アクセスパターンに基づいて選択すること。間違ったアルゴリズムはメモリの浪費やヒット率の低下を招く。

| アルゴリズム | 定数 | 適したケース | 避けるべきケース |
| --- | --- | --- | --- |
| **W-TinyLFU** | `hot.WTinyLFU` | 汎用、混合ワークロード（デフォルト） | デバッグのためにシンプルさが必要な場合 |
| **LRU** | `hot.LRU` | 最新性重視（セッション、最近のクエリ） | 頻度が重要な場合（スキャン汚染によりホットアイテムが追い出される） |
| **LFU** | `hot.LFU` | 頻度重視（人気商品、DNS） | アクセスパターンが変化する場合（古い人気アイテムが追い出されない） |
| **TinyLFU** | `hot.TinyLFU` | 読み取り中心で頻度バイアスあり | 書き込み中心（アドミッションフィルタのオーバーヘッド） |
| **S3FIFO** | `hot.S3FIFO` | 高スループット、スキャン耐性あり | 小さいキャッシュ（1000アイテム未満） |
| **ARC** | `hot.ARC` | 自己チューニング、パターン不明時 | メモリ制約あり（2倍のトラッキングオーバーヘッド） |
| **TwoQueue** | `hot.TwoQueue` | ホット/コールド分割の混合 | チューニングの複雑さが許容できない場合 |
| **SIEVE** | `hot.SIEVE` | シンプルなスキャン耐性LRU代替 | 極端に偏ったアクセスパターン |
| **FIFO** | `hot.FIFO` | シンプルで予測可能なエビクション順序 | ヒット率が重要な場合（頻度/最新性の認識なし） |

**判断の近道:** `hot.WTinyLFU` から始めること。プロファイリングでミス率がSLOに対して高すぎると判明した場合のみ切り替える。

アルゴリズムの詳細比較、ベンチマーク、デシジョンツリーについては [Algorithm Guide](./references/algorithm-guide.md) を参照。

## 基本的な使い方

### TTL付き基本キャッシュ

```go
import "github.com/samber/hot"

cache := hot.NewHotCache[string, *User](hot.WTinyLFU, 10_000).
    WithTTL(5 * time.Minute).
    WithJanitor().
    Build()
defer cache.StopJanitor()

cache.Set("user:123", user)
cache.SetWithTTL("session:abc", session, 30*time.Minute)

value, found, err := cache.Get("user:123")
```

### ローダーパターン（リードスルー）

ローダーは欠落したキーを自動的にフェッチし、singleflightで重複排除する。同じ欠落キーに対する並行 `Get()` 呼び出しは1つのローダー実行を共有する:

```go
cache := hot.NewHotCache[int, *User](hot.WTinyLFU, 10_000).
    WithTTL(5 * time.Minute).
    WithLoaders(func(ids []int) (map[int]*User, error) {
        return db.GetUsersByIDs(ctx, ids) // batch query
    }).
    WithJanitor().
    Build()
defer cache.StopJanitor()

user, found, err := cache.Get(123) // triggers loader on miss
```

## キャパシティサイジング

キャッシュ容量を設定する前に、メモリ予算内に何アイテム収まるか見積もること:

1. **単一アイテムサイズを見積もる** — 構造体のサイズを推定し、ヒープ割り当てフィールド（スライス、マップ、文字列）のサイズを加算する。キーサイズも含める。エントリあたり約100バイトの概算オーバーヘッドで内部管理（ポインタ、有効期限タイムスタンプ、アルゴリズムメタデータ）をカバーできる。
2. **開発者に確認する** — 本番環境でこのキャッシュにどれだけのメモリを割り当てるか（例: 256 MB、1 GB）。これはサービスの総メモリとプロセスを共有する他の要素に依存する。
3. **容量を計算する** — `capacity = memoryBudget / estimatedItemSize`。余裕を持たせるために切り捨てる。

```
Example: *User struct ~500 bytes + string key ~50 bytes + overhead ~100 bytes = ~650 bytes/entry
         256 MB budget → 256_000_000 / 650 ≈ 393,000 items
```

アイテムサイズが不明な場合は、N個のアイテムを割り当てて `runtime.ReadMemStats` で確認するユニットテストで計測するよう開発者に依頼すること。計測なしに容量を推測するとOOMやメモリの浪費につながる。

## よくある間違い

1. **`WithJanitor()` の付け忘れ** — これがないと、期限切れエントリはアルゴリズムが追い出すまでメモリに残る。ビルダーで常に `.WithJanitor()` をチェーンし、`defer cache.StopJanitor()` を呼ぶこと。
2. **ミッシングキャッシュ設定なしで `SetMissing()` を呼ぶ** — 実行時にpanicする。先にビルダーで `WithMissingCache(algorithm, capacity)` または `WithMissingSharedCache()` を有効にすること。
3. **`WithoutLocking()` + `WithJanitor()`** — 相互排他でpanicする。`WithoutLocking()` はバックグラウンドクリーンアップなしの単一goroutineアクセスでのみ安全。
4. **過大なキャッシュ** — すべてを保持するキャッシュはオーバーヘッド付きのマップにすぎない。ワーキングセット（通常、全データの10-20%）に合わせてサイズを設定する。ヒット率を監視して検証すること。
5. **ローダーエラーの無視** — `Get()` はローダー失敗時に `(zero, false, err)` を返す。`found` だけでなく必ず `err` をチェックすること。

## ベストプラクティス

1. 常にTTLを設定する — 無制限のキャッシュは更新シグナルがないため古いデータを無期限に返し続ける
2. `WithJitter(lambda, upperBound)` で有効期限を分散させる — ジッターなしでは同時に作成されたアイテムが同時に期限切れとなり、ローダーにサンダリングハードが発生する
3. `WithPrometheusMetrics(cacheName)` で監視する — ヒット率が80%未満の場合、通常キャッシュが小さすぎるかワークロードに対してアルゴリズムが不適切
4. ミュータブルな値には `WithCopyOnRead(fn)` / `WithCopyOnWrite(fn)` を使う — コピーなしでは呼び出し元がキャッシュオブジェクトを変更し、共有状態を破損する

高度なパターン（再検証、シャーディング、ミッシングキャッシュ、モニタリング設定）については [Production Patterns](./references/production-patterns.md) を参照。

完全なAPIサーフェスについては [API Reference](./references/api-reference.md) を参照。

samber/hot でバグや予期しない動作に遭遇した場合は、<https://github.com/samber/hot/issues> で Issue を作成してください。

## 相互参照

- → See `samber/cc-skills-golang@golang-performance` skill for general caching strategy and when to use in-memory cache vs Redis vs CDN
- → See `samber/cc-skills-golang@golang-observability` skill for Prometheus metrics integration and monitoring
- → See `samber/cc-skills-golang@golang-database` skill for database query patterns that pair with cache loaders
- → See `samber/cc-skills@promql-cli` skill for querying Prometheus cache metrics via CLI
