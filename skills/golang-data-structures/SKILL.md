---
name: golang-data-structures
description: "Golang data structures — slices (internals, capacity growth, preallocation, slices package), maps (internals, hash buckets, maps package), arrays, container/list/heap/ring, strings.Builder vs bytes.Buffer, generic collections, pointers (unsafe.Pointer, weak.Pointer), and copy semantics. Use when choosing or optimizing Go data structures, implementing generic containers, using container/ packages, unsafe or weak pointers, or questioning slice/map internals."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🗃️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

**Persona:** You are a Go engineer who understands data structure internals. You choose the right structure for the job — not the most familiar one — by reasoning about memory layout, allocation cost, and access patterns.

# Go データ構造

組み込みおよび標準ライブラリのデータ構造: 内部実装、正しい使用法、選択のガイダンス。安全性の落とし穴（nilマップ、appendのエイリアシング、防御的コピー）については`samber/cc-skills-golang@golang-safety`スキルを参照。チャネルと同期プリミティブについては`samber/cc-skills-golang@golang-concurrency`スキルを参照。string/byte/runeの選択については`samber/cc-skills-golang@golang-design-patterns`スキルを参照。

## ベストプラクティスの要約

1. サイズが既知または推定可能な場合、`make(T, 0, n)` / `make(map[K]V, n)`で**スライスとマップを事前確保する** — 繰り返しの成長コピーとリハッシュを回避する
2. **配列**は固定でコンパイル時に既知のサイズ（ハッシュダイジェスト、IPv4アドレス、行列次元）の場合にのみスライスより優先すべきである
3. **スライスのキャパシティ成長タイミングに依存してはならない** — 成長アルゴリズムはGoのバージョン間で変更されており、再度変更される可能性がある。新しいバッキング配列がいつ割り当てられるかにコードが依存すべきではない
4. 優先度キューには**`container/heap`**を使用し、頻繁な中間挿入が必要な場合にのみ**`container/list`**を使用し、固定サイズの循環バッファには**`container/ring`**を使用する
5. 文字列構築には**`strings.Builder`**を優先しなければならない。双方向I/O（`io.Reader`と`io.Writer`の両方を実装）には**`bytes.Buffer`**を優先しなければならない
6. ジェネリックデータ構造は可能な限り**最も厳密な制約**を使用すべきである — キーには`comparable`、順序付けにはカスタムインターフェース
7. **`unsafe.Pointer`**はGo仕様の6つの有効な変換パターンにのみ従わなければならない — 文をまたいで`uintptr`変数に格納してはならない
8. GCがエントリを回収できるようにするため、キャッシュと正規化マップには**`weak.Pointer[T]`**（Go 1.24+）を使用すべきである

## スライスの内部実装

スライスは3ワードのヘッダ: ポインタ、長さ、キャパシティ。複数のスライスがバッキング配列を共有できる（→ see `samber/cc-skills-golang@golang-safety` for aliasing traps and the header diagram）。

### キャパシティの成長

- 256要素未満: キャパシティは2倍になる
- 256要素以上: 約25%成長する（`newcap += (newcap + 3*256) / 4`）
- 成長のたびにバッキング配列全体がコピーされる — O(n)

### 事前確保

```go
// Exact size known
users := make([]User, 0, len(ids))

// Approximate size known
results := make([]Result, 0, estimatedCount)

// Pre-grow before bulk append (Go 1.21+)
s = slices.Grow(s, additionalNeeded)
```

### `slices`パッケージ（Go 1.21+）

主要な関数: `Sort`/`SortFunc`、`BinarySearch`、`Contains`、`Compact`、`Grow`。`Clone`、`Equal`、`DeleteFunc`については → see `samber/cc-skills-golang@golang-safety` skill。

**[スライス内部実装の詳細](./references/slice-internals.md)** — 完全な`slices`パッケージリファレンス、成長メカニズム、`len` vs `cap`、ヘッダコピー、バッキング配列のエイリアシング。

## マップの内部実装

マップは8エントリのバケットとオーバーフローチェーンを持つハッシュテーブルである。参照型であり、マップを代入するとデータではなくポインタがコピーされる。

### 事前確保

```go
m := make(map[string]*User, len(users)) // avoids rehashing during population
```

### `maps`パッケージクイックリファレンス（Go 1.21+）

| 関数              | 目的                         |
| ----------------- | ---------------------------- |
| `Collect` (1.23+) | イテレータからマップを構築   |
| `Insert` (1.23+)  | イテレータからエントリを挿入 |
| `All` (1.23+)     | 全エントリのイテレータ       |
| `Keys`, `Values`  | キー/値のイテレータ          |

`Clone`、`Equal`、ソート済みイテレーションについては → see `samber/cc-skills-golang@golang-safety` skill。

**[マップ内部実装の詳細](./references/map-internals.md)** — Goのマップがデータをどのように格納しハッシュするか、バケットオーバーフローチェーン、マップが縮小しない理由（と対処法）、代替手段とのパフォーマンス比較。

## 配列

固定サイズ、値型。代入時に全体がコピーされる。コンパイル時に既知のサイズに使用する:

```go
type Digest [32]byte           // fixed-size, value type
var grid [3][3]int             // multi-dimensional
cache := map[[2]int]Result{}   // arrays are comparable — usable as map keys
```

それ以外のすべてにはスライスを使用する — 配列は成長できず、値渡し（大きなサイズではコストが高い）。

## container/ 標準ライブラリ

| パッケージ | データ構造 | 最適な用途 |
| --- | --- | --- |
| `container/list` | 双方向リンクリスト | LRUキャッシュ、頻繁な中間挿入/削除 |
| `container/heap` | 最小ヒープ（優先度キュー） | Top-K、スケジューリング、ダイクストラ |
| `container/ring` | 循環バッファ | ローリングウィンドウ、ラウンドロビン |
| `bufio` | バッファ付きリーダー/ライター/スキャナー | 小さな読み書きでの効率的なI/O |

コンテナ型は`any`を使用する（型安全性なし） — ジェネリックラッパーを検討する。**[コンテナパターン、bufio、例](./references/containers.md)** — 各コンテナ型の使い分け、型安全性を追加するジェネリックラッパー、効率的なI/Oのための`bufio`パターン。

## strings.Builder vs bytes.Buffer

純粋な文字列結合には`strings.Builder`を使用する（`String()`でのコピーを回避）。`io.Reader`やバイト操作が必要な場合は`bytes.Buffer`を使用する。両方とも`Grow(n)`をサポートする。**[詳細と比較](./references/containers.md)**

## ジェネリックコレクション（Go 1.18+）

可能な限り最も厳密な制約を使用する。マップキーには`comparable`、ソートには`cmp.Ordered`、ドメイン固有の順序付けにはカスタムインターフェース。

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(v T)          { s[v] = struct{}{} }
func (s Set[T]) Contains(v T) bool { _, ok := s[v]; return ok }
```

**[ジェネリックデータ構造の作成](./references/generics.md)** — Go 1.18+のジェネリクスを使用した型安全なコンテナ、制約充足の理解、ドメイン固有のジェネリック型の構築。

## ポインタ型

| 型 | ユースケース | ゼロ値 |
| --- | --- | --- |
| `*T` | 通常の間接参照、ミューテーション、オプション値 | `nil` |
| `unsafe.Pointer` | FFI、低レベルメモリレイアウト（仕様の6パターンのみ） | `nil` |
| `weak.Pointer[T]` (1.24+) | キャッシュ、正規化、弱参照 | N/A |

**[ポインタ型の詳細](./references/pointers.md)** — 通常のポインタ、`unsafe.Pointer`（仕様の6つの有効なパターン）、クリーンアップを妨げないGCセーフなキャッシュのための`weak.Pointer[T]`。

## コピーセマンティクスクイックリファレンス

| 型 | コピー動作 | 独立性 |
| --- | --- | --- |
| `int`, `float`, `bool`, `string` | 値（ディープコピー） | 完全に独立 |
| `array`, `struct` | 値（ディープコピー） | 完全に独立 |
| `slice` | ヘッダがコピー、バッキング配列は共有 | `slices.Clone`を使用 |
| `map` | 参照がコピー | `maps.Clone`を使用 |
| `channel` | 参照がコピー | 同じチャネル |
| `*T` (pointer) | アドレスがコピー | 同じ基底値 |
| `interface` | 値がコピー（型+値のペア） | 保持する型に依存 |

## サードパーティライブラリ

標準ライブラリを超える高度なデータ構造（ツリー、セット、キュー、スタック）:

- **`emirpasic/gods`** — 包括的なコレクションライブラリ（ツリー、セット、リスト、スタック、マップ、キュー）
- **`deckarep/golang-set`** — スレッドセーフおよび非スレッドセーフなセット実装
- **`gammazero/deque`** — 高速な両端キュー

サードパーティライブラリを使用する際は、最新のAPIシグネチャについて公式ドキュメントとコード例を参照すること。Context7 can help as a discoverability platform.

## 相互参照

- → See `samber/cc-skills-golang@golang-performance` skill for struct field alignment, memory layout optimization, and cache locality
- → See `samber/cc-skills-golang@golang-safety` skill for nil map/slice pitfalls, append aliasing, defensive copying, `slices.Clone`/`Equal`
- → See `samber/cc-skills-golang@golang-concurrency` skill for channels, `sync.Map`, `sync.Pool`, and all sync primitives
- → See `samber/cc-skills-golang@golang-design-patterns` skill for `string` vs `[]byte` vs `[]rune`, iterators, streaming
- → See `samber/cc-skills-golang@golang-structs-interfaces` skill for struct composition, embedding, and generics vs `any`
- → See `samber/cc-skills-golang@golang-code-style` skill for slice/map initialization style

## よくある間違い

| 間違い | 修正方法 |
| --- | --- |
| 事前確保せずにループ内でスライスを成長させる | 成長のたびにバッキング配列全体がコピーされる — 成長ごとにO(n)。`make([]T, 0, n)`または`slices.Grow`を使用する |
| スライスで十分な場合に`container/list`を使用する | リンクリストはキャッシュ局所性が低い（各ノードが個別のヒープ割り当て）。まずベンチマークを行う |
| 純粋な文字列構築に`bytes.Buffer`を使用する | Bufferの`String()`は基底バイトをコピーする。`strings.Builder`はこのコピーを回避する |
| `unsafe.Pointer`を文をまたいで`uintptr`として格納する | GCが文間でオブジェクトを移動できる — `uintptr`はダングリング参照となる |
| マップ内の大きな構造体値（コピーオーバーヘッド） | マップアクセスは値全体をコピーする。コピーを避けるため大きな値型には`map[K]*V`を使用する |

## 参考資料

- [Go Data Structures (Russ Cox)](https://research.swtch.com/godata)
- [The Go Memory Model](https://go.dev/ref/mem)
- [Effective Go](https://go.dev/doc/effective_go)
