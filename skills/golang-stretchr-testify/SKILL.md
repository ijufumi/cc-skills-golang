---
name: golang-stretchr-testify
description: "Comprehensive guide to stretchr/testify for Golang testing. Covers assert, require, mock, and suite packages in depth. Use whenever writing tests with testify, creating mocks, setting up test suites, or choosing between assert and require. Essential for testify assertions, mock expectations, argument matchers, call verification, suite lifecycle, and advanced patterns like Eventually, JSONEq, and custom matchers. Trigger on any Go test file importing testify."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "✅"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - gotests
    install:
      - kind: go
        package: github.com/cweill/gotests/...@latest
        bins: [gotests]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs Bash(gotests:*) AskUserQuestion
---

**Persona:** You are a Go engineer who treats tests as executable specifications. You write tests to constrain behavior and make failures self-explanatory — not to hit coverage targets.

**Modes:**

- **Write mode** — コードベースに新しいテストやモックを追加する。
- **Review mode** — 既存のテストコードのtestify誤用を監査する。

# stretchr/testify

testifyはGoの `testing` パッケージを補完し、読みやすいアサーション、モック、スイートを提供します。`testing` を置き換えるものではありません。常に `*testing.T` をエントリポイントとして使用してください。

このスキルは網羅的ではありません。詳細はライブラリのドキュメントとコード例を参照してください。Context7はディスカバリプラットフォームとして活用できます。

## assert と require

どちらも同一のアサーションを提供します。違いは失敗時の動作です:

- **assert**: 失敗を記録し、処理を続行 — すべての失敗を一度に確認できる
- **require**: `t.FailNow()` を呼び出す — 続行するとパニックや誤解を招く前提条件に使用

可読性のために `assert.New(t)` / `require.New(t)` を使用してください。それぞれ `is` と `must` と命名します:

```go
func TestParseConfig(t *testing.T) {
    is := assert.New(t)
    must := require.New(t)

    cfg, err := ParseConfig("testdata/valid.yaml")
    must.NoError(err)    // stop if parsing fails — cfg would be nil
    must.NotNil(cfg)

    is.Equal("production", cfg.Environment)
    is.Equal(8080, cfg.Port)
    is.True(cfg.TLS.Enabled)
}
```

**ルール**: 前提条件（セットアップ、エラーチェック）には `require`、検証には `assert` を使用する。無秩序に混ぜないこと。

## コアアサーション

```go
is := assert.New(t)

// Equality
is.Equal(expected, actual)              // DeepEqual + exact type
is.NotEqual(unexpected, actual)
is.EqualValues(expected, actual)        // converts to common type first
is.EqualExportedValues(expected, actual)

// Nil / Bool / Emptiness
is.Nil(obj)                  is.NotNil(obj)
is.True(cond)                is.False(cond)
is.Empty(collection)         is.NotEmpty(collection)
is.Len(collection, n)

// Contains (strings, slices, map keys)
is.Contains("hello world", "world")
is.Contains([]int{1, 2, 3}, 2)
is.Contains(map[string]int{"a": 1}, "a")

// Comparison
is.Greater(actual, threshold)     is.Less(actual, ceiling)
is.Positive(val)                  is.Negative(val)
is.Zero(val)

// Errors
is.Error(err)                     is.NoError(err)
is.ErrorIs(err, ErrNotFound)      // walks error chain
is.ErrorAs(err, &target)
is.ErrorContains(err, "not found")

// Type
is.IsType(&User{}, obj)
is.Implements((*io.Reader)(nil), obj)
```

**引数の順序**: 常に `(expected, actual)` — 順序を入れ替えると紛らわしいdiff出力になります。

## 高度なアサーション

```go
is.ElementsMatch([]string{"b", "a", "c"}, result)             // unordered comparison
is.InDelta(3.14, computedPi, 0.01)                            // float tolerance
is.JSONEq(`{"name":"alice"}`, `{"name": "alice"}`)             // ignores whitespace/key order
is.WithinDuration(expected, actual, 5*time.Second)
is.Regexp(`^user-[a-f0-9]+$`, userID)

// Async polling
is.Eventually(func() bool {
    status, _ := client.GetJobStatus(jobID)
    return status == "completed"
}, 5*time.Second, 100*time.Millisecond)

// Async polling with rich assertions
is.EventuallyWithT(func(c *assert.CollectT) {
    resp, err := client.GetOrder(orderID)
    assert.NoError(c, err)
    assert.Equal(c, "shipped", resp.Status)
}, 10*time.Second, 500*time.Millisecond)
```

## testify/mock

テスト対象のユニットを分離するためにインターフェースをモック化します。`mock.Mock` を埋め込み、`m.Called()` でメソッドを実装し、常に `AssertExpectations(t)` で検証してください。

主要なマッチャー: `mock.Anything`, `mock.AnythingOfType("T")`, `mock.MatchedBy(func)`。呼び出し修飾子: `.Once()`, `.Times(n)`, `.Maybe()`, `.Run(func)`。

モックの定義、引数マッチャー、呼び出し修飾子、戻り値シーケンス、検証については [Mock reference](./references/mock.md) を参照してください。

## testify/suite

スイートは共有のセットアップ/ティアダウンで関連するテストをグループ化します。

### ライフサイクル

```
SetupSuite()    → once before all tests
  SetupTest()   → before each test
    TestXxx()
  TearDownTest() → after each test
TearDownSuite() → once after all tests
```

### 例

```go
type TokenServiceSuite struct {
    suite.Suite
    store   *MockTokenStore
    service *TokenService
}

func (s *TokenServiceSuite) SetupTest() {
    s.store = new(MockTokenStore)
    s.service = NewTokenService(s.store)
}

func (s *TokenServiceSuite) TestGenerate_ReturnsValidToken() {
    s.store.On("Save", mock.Anything, mock.Anything).Return(nil)
    token, err := s.service.Generate("user-42")
    s.NoError(err)
    s.NotEmpty(token)
    s.store.AssertExpectations(s.T())
}

// 必須のランチャー
func TestTokenServiceSuite(t *testing.T) {
    suite.Run(t, new(TokenServiceSuite))
}
```

スイートメソッドの `s.Equal()` は `assert` と同様に動作します。requireの場合: `s.Require().NotNil(obj)`。

## よくある間違い

- **`AssertExpectations(t)` の呼び忘れ** — モックのexpectationが検証なしで暗黙的にパスしてしまう
- **`is.Equal(ErrNotFound, err)`** — ラップされたエラーで失敗する。エラーチェーンをたどるには `is.ErrorIs` を使用する
- **引数の順序の入れ替え** — testifyは `(expected, actual)` を前提としている。入れ替えると逆向きのdiffが出力される
- **ガードに `assert` を使用** — 失敗後もテストが続行し、nilデリファレンスでパニックする。`require` を使用する
- **`suite.Run()` の欠落** — ランチャー関数がないと、テストが一つも実行されず無視される
- **ポインタの比較** — `is.Equal(ptr1, ptr2)` はアドレスを比較する。デリファレンスするか `EqualExportedValues` を使用する

## リンター

`testifylint` を使用して、引数の順序の間違い、assert/requireの誤用などを検出できます。`samber/cc-skills-golang@golang-linter` スキルを参照してください。

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-testing` skill for general test patterns, table-driven tests, and CI
- → See `samber/cc-skills-golang@golang-linter` skill for testifylint configuration
