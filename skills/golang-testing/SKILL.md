---
name: golang-testing
description: "Provides a comprehensive guide for writing production-ready Golang tests. Covers table-driven tests, test suites with testify, mocks, unit tests, integration tests, benchmarks, code coverage, parallel tests, fuzzing, fixtures, goroutine leak detection with goleak, snapshot testing, memory leaks, CI with GitHub Actions, and idiomatic naming conventions. Use this whenever writing tests, asking about testing patterns or setting up CI for Go projects. Essential for ANY test-related conversation in Go."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🧪"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - gotests
    install:
      - kind: go
        package: github.com/cweill/gotests/gotests@latest
        bins: [gotests]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent Bash(gotests:*) AskUserQuestion
---

**Persona:** You are a Go engineer who treats tests as executable specifications. You write tests to constrain behavior, not to hit coverage targets.

**Thinking mode:** Use `ultrathink` for test strategy design and failure analysis. Shallow reasoning misses edge cases and produces brittle tests that pass today but break tomorrow.

**Modes:**

- **Write mode** — generating new tests for existing or new code. Work sequentially through the code under test; use `gotests` to scaffold table-driven tests, then enrich with edge cases and error paths.
- **Review mode** — reviewing a PR's test changes. Focus on the diff: check coverage of new behaviour, assertion quality, table-driven structure, and absence of flakiness patterns. Sequential.
- **Audit mode** — auditing an existing test suite for gaps, flakiness, or bad patterns (order-dependent tests, missing `t.Parallel()`, implementation-detail coupling). Launch up to 3 parallel sub-agents split by concern: (1) unit test quality and coverage gaps, (2) integration test isolation and build tags, (3) goroutine leaks and race conditions.
- **Debug mode** — a test is failing or flaky. Work sequentially: reproduce reliably, isolate the failing assertion, trace the root cause in production code or test setup.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-testing` skill takes precedence.

# Go テストのベストプラクティス

このスキルはGoアプリケーションの本番品質のテスト作成をガイドする。保守性が高く、高速で信頼性のあるテストを書くために以下の原則に従う。

## ベストプラクティスまとめ

1. テーブル駆動テストはMUSTで名前付きサブテストを使用する -- すべてのテストケースに `t.Run` に渡す `name` フィールドが必要
2. 統合テストはMUSTでビルドタグ（`//go:build integration`）を使用してユニットテストと分離する
3. テストはMUSTで実行順序に依存しない -- 各テストはMUSTで独立して実行可能でなければならない
4. 独立したテストはSHOULDで可能な限り `t.Parallel()` を使用する
5. 実装の詳細をテストしない -- 観測可能な振る舞いと公開APIの契約をテストする
6. ゴルーチンを持つパッケージはSHOULDで `TestMain` 内の `goleak.VerifyTestMain` を使用してゴルーチンリークを検出する
7. testifyは標準ライブラリの代替ではなく、ヘルパーとして使用する
8. 具象型ではなくインターフェースをモックする
9. ユニットテストは高速に保ち（< 1ms）、統合テストにはビルドタグを使用する
10. CIではレースディテクション付きでテストを実行する
11. 実行可能なドキュメントとしてExampleを含める

## テストの構造と構成

### ファイル規約

```go
// package_test.go - tests in same package (white-box, access unexported)
package mypackage

// mypackage_test.go - tests in test package (black-box, public API only)
package mypackage_test
```

### 命名規約

```go
func TestAdd(t *testing.T) { ... }              // function test
func TestMyStruct_MyMethod(t *testing.T) { ... } // method test
func BenchmarkAdd(b *testing.B) { ... }          // benchmark
func ExampleAdd() { ... }                        // example
```

## テーブル駆動テスト

テーブル駆動テストは複数のシナリオをテストするGoの慣用的な方法である。各テストケースには必ず名前を付ける。

```go
func TestCalculatePrice(t *testing.T) {
    tests := []struct {
        name     string
        quantity int
        unitPrice float64
        expected  float64
    }{
        {
            name:      "single item",
            quantity:  1,
            unitPrice: 10.0,
            expected:  10.0,
        },
        {
            name:      "bulk discount - 100 items",
            quantity:  100,
            unitPrice: 10.0,
            expected:  900.0, // 10% discount
        },
        {
            name:      "zero quantity",
            quantity:  0,
            unitPrice: 10.0,
            expected:  0.0,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := CalculatePrice(tt.quantity, tt.unitPrice)
            if got != tt.expected {
                t.Errorf("CalculatePrice(%d, %.2f) = %.2f, want %.2f",
                    tt.quantity, tt.unitPrice, got, tt.expected)
            }
        })
    }
}
```

## ユニットテスト

ユニットテストは高速（< 1ms）、独立（外部依存なし）、決定論的であるべき。

## HTTPハンドラのテスト

ハンドラテストにはテーブル駆動パターンで `httptest` を使用する。リクエスト/レスポンスボディ、クエリパラメータ、ヘッダー、ステータスコードアサーションの例は [HTTP Testing](./references/http-testing.md) を参照。

## goleakによるゴルーチンリーク検出

特に並行コードでリークしているゴルーチンを検出するために `go.uber.org/goleak` を使用する:

```go
import (
    "testing"
    "go.uber.org/goleak"
)

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

特定のゴルーチンスタックを除外する場合（既知のリークやライブラリのゴルーチン）:

```go
func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m,
        goleak.IgnoreCurrent(),
    )
}
```

またはテスト単位で:

```go
func TestWorkerPool(t *testing.T) {
    defer goleak.VerifyNone(t)
    // ... test code ...
}
```

## 決定論的ゴルーチンテストのための testing/synctest

> **実験的:** `testing/synctest` はまだGoの互換性保証の対象外。APIは将来のリリースで変更される可能性がある。安定した代替手段として `clockwork` を使用（[Mocking](./references/mocking.md) を参照）。

`testing/synctest`（Go 1.24+）は並行コードテストに決定論的な時間を提供する。すべてのゴルーチンがブロックされた場合にのみ時間が進み、順序が予測可能になる。

実際の時間の代わりに `synctest` を使用する場合:

- 時間ベースの操作（time.Sleep、time.After、time.Ticker）を含む並行コードのテスト
- レースコンディションを再現可能にする必要がある場合
- タイミングの問題でテストがフレーキーな場合

```go
import (
    "testing"
    "time"
    "testing/synctest"
    "github.com/stretchr/testify/assert"
)

func TestChannelTimeout(t *testing.T) {
    synctest.Run(func(t *testing.T) {
        is := assert.New(t)

        ch := make(chan int, 1)
        go func() {
            time.Sleep(50 * time.Millisecond)
            ch <- 42
        }()

        select {
        case v := <-ch:
            is.Equal(42, v)
        case <-time.After(100 * time.Millisecond):
            t.Fatal("timeout occurred")
        }
    })
}
```

`synctest` の主な違い:

- `time.Sleep` はゴルーチンがブロックした時点で合成時間を即座に進める
- `time.After` は合成時間が期間に達した時点で発火する
- すべてのゴルーチンが時間が進む前にブロッキングポイントまで実行される
- テスト実行は決定論的で再現可能

## テストタイムアウト

ハングする可能性のあるテストには、呼び出し元の位置情報付きでpanicするタイムアウトヘルパーを使用する。[Helpers](./references/helpers.md) を参照。

## ベンチマーク

→ See `samber/cc-skills-golang@golang-benchmark` skill for advanced benchmarking: `b.Loop()` (Go 1.24+), `benchstat`, profiling from benchmarks, and CI regression detection.

パフォーマンスを測定しリグレッションを検出するためにベンチマークを書く:

```go
func BenchmarkStringConcatenation(b *testing.B) {
    b.Run("plus-operator", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            result := "a" + "b" + "c"
            _ = result
        }
    })

    b.Run("strings.Builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var builder strings.Builder
            builder.WriteString("a")
            builder.WriteString("b")
            builder.WriteString("c")
            _ = builder.String()
        }
    })
}
```

異なる入力サイズでのベンチマーク:

```go
func BenchmarkFibonacci(b *testing.B) {
    sizes := []int{10, 20, 30}
    for _, size := range sizes {
        b.Run(fmt.Sprintf("n=%d", size), func(b *testing.B) {
            b.ReportAllocs()
            for i := 0; i < b.N; i++ {
                Fibonacci(size)
            }
        })
    }
}
```

## 並列テスト

`t.Parallel()` を使用してテストを同時実行する:

```go
func TestParallelOperations(t *testing.T) {
    tests := []struct {
        name string
        data []byte
    }{
        {"small data", make([]byte, 1024)},
        {"medium data", make([]byte, 1024*1024)},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            is := assert.New(t)

            result := Process(tt.data)
            is.NotNil(result)
        })
    }
}
```

## ファジング

ファジングを使用してエッジケースやバグを発見する:

```go
func FuzzReverse(f *testing.F) {
    f.Add("hello")
    f.Add("")
    f.Add("a")

    f.Fuzz(func(t *testing.T, input string) {
        reversed := Reverse(input)
        doubleReversed := Reverse(reversed)
        if input != doubleReversed {
            t.Errorf("Reverse(Reverse(%q)) = %q, want %q", input, doubleReversed, input)
        }
    })
}
```

## ドキュメントとしてのExample

Exampleは `go test` で検証される実行可能なドキュメントである:

```go
func ExampleCalculatePrice() {
    price := CalculatePrice(100, 10.0)
    fmt.Printf("Price: %.2f\n", price)
    // Output: Price: 900.00
}

func ExampleCalculatePrice_singleItem() {
    price := CalculatePrice(1, 25.50)
    fmt.Printf("Price: %.2f\n", price)
    // Output: Price: 25.50
}
```

## コードカバレッジ

```bash
# Generate coverage file
go test -coverprofile=coverage.out ./...

# View coverage in HTML
go tool cover -html=coverage.out

# Coverage by function
go tool cover -func=coverage.out

# Total coverage percentage
go tool cover -func=coverage.out | grep total
```

## 統合テスト

ビルドタグを使用して統合テストをユニットテストから分離する:

```go
//go:build integration

package mypackage

func TestDatabaseIntegration(t *testing.T) {
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()

    // Test real database operations
}
```

統合テストを個別に実行する:

```bash
go test -tags=integration ./...
```

Docker Composeフィクスチャ、SQLスキーマ、統合テストスイートについては [Integration Testing](./references/integration-testing.md) を参照。

## モッキング

具象型ではなくインターフェースをモックする。インターフェースは使用する側で定義し、モック実装を作成する。

モックパターン、テストフィクスチャ、時間のモッキングについては [Mocking](./references/mocking.md) を参照。

## リンターによる強制

多くのテストベストプラクティスはリンターで自動的に強制される: `thelper`、`paralleltest`、`testifylint`。設定と使用方法は `samber/cc-skills-golang@golang-linter` スキルを参照。

## クロスリファレンス

- -> See `samber/cc-skills-golang@golang-stretchr-testify` skill for detailed testify API (assert, require, mock, suite)
- -> See `samber/cc-skills-golang@golang-database` skill (testing.md) for database integration test patterns
- -> See `samber/cc-skills-golang@golang-concurrency` skill for goroutine leak detection with goleak
- -> See `samber/cc-skills-golang@golang-continuous-integration` skill for CI test configuration and GitHub Actions workflows
- -> See `samber/cc-skills-golang@golang-linter` skill for testifylint and paralleltest configuration

## クイックリファレンス

```bash
go test ./...                          # all tests
go test -run TestName ./...            # specific test by exact name
go test -run TestName/subtest ./...    # subtests within a test
go test -run 'Test(Add|Sub)' ./...     # multiple tests (regexp OR)
go test -run 'Test[A-Z]' ./...         # tests starting with capital letter
go test -run 'TestUser.*' ./...        # tests matching prefix
go test -run '.*Validation.*' ./...    # tests containing substring
go test -run TestName/. ./...          # all subtests of TestName
go test -run '/(unit|integration)' ./... # filter by subtest name
go test -race ./...                    # race detection
go test -cover ./...                   # coverage summary
go test -bench=. -benchmem ./...       # benchmarks
go test -fuzz=FuzzName ./...           # fuzzing
go test -tags=integration ./...        # integration tests
```
