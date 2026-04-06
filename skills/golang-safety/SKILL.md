---
name: golang-safety
description: "Defensive Golang coding to prevent panics, silent data corruption, and subtle runtime bugs. Use whenever writing or reviewing Go code that involves nil-prone types (pointers, interfaces, maps, slices, channels), numeric conversions, resource lifecycle (defer in loops), or defensive copying. Also triggers on questions about nil panics, append aliasing, map concurrent access, float comparison, or zero-value design."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🛡️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

**Persona:** You are a defensive Go engineer. You treat every untested assumption about nil, capacity, and numeric range as a latent crash waiting to happen.

# Go 安全性: 正確性と防御的コーディング

プログラマのミスを防止する — 通常の（非悪意的な）コードにおけるバグ、panic、サイレントなデータ破損。セキュリティは攻撃者に対処し、安全性は自分たち自身に対処する。

## ベストプラクティスまとめ

1. **型のセットが既知の場合はSHOULDで `any` よりジェネリクスを優先** — コンパイラがランタイムpanicの代わりにミスマッチを検出
2. **型アサーションには常にカンマokを使用** — 素のアサーションはミスマッチ時にpanicする
3. **インターフェース内の型付きnilポインタは `== nil` ではない** — 型記述子がnon-nilにする
4. **nilマップへの書き込みはpanicする** — 使用前に必ず初期化
5. **`append` はバッキング配列を再利用する可能性がある** — 容量が許せば両方のスライスがメモリを共有し、互いをサイレントに破損
6. **エクスポートされた関数からは防御的コピーを返す** — そうしないと呼び出し側が内部を変更する
7. **`defer` はループの反復ではなく関数の終了時に実行される** — ループ本体を関数に抽出
8. **整数変換はサイレントに切り捨てる** — `int64` から `int32` はエラーなしでラップする
9. **浮動小数点演算は正確ではない** — イプシロン比較または `math/big` を使用
10. **有用なゼロ値を設計する** — nilマップフィールドは最初の書き込みでpanic。遅延初期化を使用
11. **遅延初期化には `sync.Once` を使用** — 並行状況下でも正確に一度だけを保証

## Nil安全性

Nil関連のpanicはGoで最も一般的なクラッシュである。

### nilインターフェーストラップ

インターフェースは(type, value)を格納する。インターフェースは両方がnilの場合のみ `nil` になる。型付きnilポインタを返すと型記述子が設定され、non-nilになる:

```go
// ✗ Dangerous — interface{type: *MyHandler, value: nil} is not == nil
func getHandler() http.Handler {
    var h *MyHandler // nil pointer
    if !enabled {
        return h // interface{type: *MyHandler, value: nil} != nil
    }
    return h
}

// ✓ Good — return nil explicitly
func getHandler() http.Handler {
    if !enabled {
        return nil // interface{type: nil, value: nil} == nil
    }
    return &MyHandler{}
}
```

### nilマップ、スライス、チャネルの動作

| 型 | nilからの読み取り | nilへの書き込み | nilのLen/Cap | nilに対するRange |
| --- | --- | --- | --- | --- |
| Map | ゼロ値 | **panic** | 0 | 0回の反復 |
| Slice | **panic**（インデックス） | **panic**（インデックス） | 0 | 0回の反復 |
| Channel | 永久にブロック | 永久にブロック | 0 | 永久にブロック |

```go
// ✗ Bad — nil map panics on write
var m map[string]int
m["key"] = 1

// ✓ Good — initialize or lazy-init in methods
m := make(map[string]int)

func (r *Registry) Add(name string, val int) {
    if r.items == nil { r.items = make(map[string]int) }
    r.items[name] = val
}
```

nilレシーバー、ジェネリクスにおけるnil、nilインターフェースのパフォーマンスについては **[Nil Safety Deep Dive](./references/nil-safety.md)** を参照。

## スライスとマップの安全性

### スライスエイリアシング — appendトラップ

`append` は容量が許せばバッキング配列を再利用する。両方のスライスがメモリを共有する:

```go
// ✗ Dangerous — a and b share backing array
a := make([]int, 3, 5)
b := append(a, 4)
b[0] = 99 // also modifies a[0]

// ✓ Good — full slice expression forces new allocation
b := append(a[:len(a):len(a)], 4)
```

### マップの並行アクセス

マップはMUSTで並行アクセスしてはならない — → see `samber/cc-skills-golang@golang-concurrency` for sync primitives.

rangeの落とし穴、サブスライスのメモリ保持、`slices.Clone`/`maps.Clone` については **[Slice and Map Deep Dive](./references/slice-map-safety.md)** を参照。

## 数値の安全性

### 暗黙の型変換はサイレントに切り捨てる

```go
// ✗ Bad — silently wraps around if val > math.MaxInt32 (3B becomes -1.29B)
var val int64 = 3_000_000_000
i32 := int32(val) // -1294967296 (silent wraparound)

// ✓ Good — check before converting
if val > math.MaxInt32 || val < math.MinInt32 {
    return fmt.Errorf("value %d overflows int32", val)
}
i32 := int32(val)
```

### 浮動小数点比較

```go
// ✗ Bad — floating point arithmetic is not exact
0.1+0.2 == 0.3 // false

// ✓ Good — use epsilon comparison
const epsilon = 1e-9
math.Abs((0.1+0.2)-0.3) < epsilon // true
```

### ゼロ除算

整数のゼロ除算はpanicする。浮動小数点のゼロ除算は `+Inf`、`-Inf`、または `NaN` を生成する。

```go
func avg(total, count int) (int, error) {
    if count == 0 {
        return 0, errors.New("division by zero")
    }
    return total / count, nil
}
```

セキュリティ脆弱性としての整数オーバーフローについては `samber/cc-skills-golang@golang-security` スキルのセクションを参照。

## リソースの安全性

### ループ内のdefer — リソースの蓄積

`defer` はループの反復ではなく_関数_の終了時に実行される。関数が返るまでリソースが蓄積する:

```go
// ✗ Bad — all files stay open until function returns
for _, path := range paths {
    f, _ := os.Open(path)
    defer f.Close() // deferred until function exits
    process(f)
}

// ✓ Good — extract to function so defer runs per iteration
for _, path := range paths {
    if err := processOne(path); err != nil { return err }
}
func processOne(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close()
    return process(f)
}
```

### ゴルーチンリーク

→ See `samber/cc-skills-golang@golang-concurrency` for goroutine lifecycle and leak prevention.

## イミュータビリティと防御的コピー

スライス/マップを返すエクスポートされた関数はSHOULDで防御的コピーを返す。

### 構造体の内部を保護する

```go
// ✗ Bad — exported slice field, anyone can mutate
type Config struct {
    Hosts []string
}

// ✓ Good — unexported field with accessor returning a copy
type Config struct {
    hosts []string
}

func (c *Config) Hosts() []string {
    return slices.Clone(c.hosts)
}
```

## 初期化の安全性

### ゼロ値設計

`var x MyType` が安全になるように型を設計する — 「初期化忘れ」バグを防止:

```go
var mu sync.Mutex   // ✓ usable at zero value
var buf bytes.Buffer // ✓ usable at zero value

// ✗ Bad — nil map panics on write
type Cache struct { data map[string]any }
```

### 遅延初期化のための sync.Once

```go
type DB struct {
    once sync.Once
    conn *sql.DB
}

func (db *DB) connection() *sql.DB {
    db.once.Do(func() {
        db.conn, _ = sql.Open("postgres", connStr)
    })
    return db.conn
}
```

### init()関数の落とし穴

→ See `samber/cc-skills-golang@golang-design-patterns` for why init() should be avoided in favor of explicit constructors.

## リンターによる強制

多くの安全性の落とし穴はリンターによって自動的に検出される: `errcheck`、`forcetypeassert`、`nilerr`、`govet`、`staticcheck`。設定と使用方法は `samber/cc-skills-golang@golang-linter` スキルを参照。

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-concurrency` skill for concurrent access patterns and sync primitives
- → See `samber/cc-skills-golang@golang-data-structures` skill for slice/map internals, capacity growth, and container/ packages
- → See `samber/cc-skills-golang@golang-error-handling` skill for nil error interface trap
- → See `samber/cc-skills-golang@golang-security` skill for security-relevant safety issues (memory safety, integer overflow)
- → See `samber/cc-skills-golang@golang-troubleshooting` skill for debugging panics and race conditions

## よくある間違い

| 間違い | 修正 |
| --- | --- |
| 素の型アサーション `v := x.(T)` | 型ミスマッチ時にpanicし、プログラムがクラッシュ。`v, ok := x.(T)` で優雅に処理 |
| インターフェース関数で型付きnilを返す | インターフェースは(type, nil)を保持し、!= nilになる。nilの場合は型なし `nil` を返す |
| nilマップへの書き込み | nilマップにはバッキングストレージがない — 書き込みでpanic。`make(map[K]V)` または遅延初期化で初期化 |
| `append` が常にコピーすると仮定 | 容量が許せば、両方のスライスがバッキング配列を共有。`s[:len(s):len(s)]` でコピーを強制 |
| ループ内の `defer` | `defer` はループの反復ではなく関数終了時に実行 — リソースが蓄積。本体を別の関数に抽出 |
| 境界チェックなしの `int64` から `int32` | 値がサイレントにラップ（3B → -1.29B）。先に `math.MaxInt32`/`math.MinInt32` と比較 |
| `==` での浮動小数点比較 | IEEE 754表現は正確ではない（`0.1+0.2 != 0.3`）。`math.Abs(a-b) < epsilon` を使用 |
| ゼロチェックなしの整数除算 | 整数のゼロ除算はpanic。除算前に `if divisor == 0` でガード |
| 内部スライス/マップ参照の返却 | 共有バッキング配列を通じて呼び出し側が構造体の内部を変更可能。防御的コピーを返す |
| 順序を仮定した複数の `init()` | ファイル間の `init()` 実行順序は未規定。→ See `samber/cc-skills-golang@golang-design-patterns` — 明示的コンストラクタを使用 |
| nilチャネルでの永久ブロック | nilチャネルは送信と受信の両方でブロックする。使用前に必ず初期化 |
