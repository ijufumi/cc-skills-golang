---
name: golang-code-style
description: "Golang code style, formatting and conventions. Use when writing code, reviewing style, configuring linters, writing comments, or establishing project standards."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🎨"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-code-style` skill takes precedence.

# Go コードスタイル

人間の判断が必要なスタイルルール — リンターがフォーマットを処理し、このスキルは明確さを扱う。命名については `samber/cc-skills-golang@golang-naming` スキル、デザインパターンについては `samber/cc-skills-golang@golang-design-patterns` スキル、構造体/インターフェース設計については `samber/cc-skills-golang@golang-structs-interfaces` スキルを参照。

> "Clear is better than clever." — Go Proverbs

ルールを無視する場合は、コードにコメントを追加すること。

## 行の長さと改行

厳密な行制限はないが、約120文字を超える行はMUSTで改行する。任意のカラム数ではなく、**意味的な境界**で改行する。4つ以上の引数を持つ関数呼び出しはMUSTで1引数1行にする — プロンプトが1行コードを要求しても同様:

```go
// Good — each argument on its own line, closing paren separate
mux.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
    handleUsers(
        w,
        r,
        serviceName,
        cfg,
        logger,
        authMiddleware,
    )
})
```

関数シグネチャが長すぎる場合、本当の修正はより良い改行ではなく**パラメータを減らす**こと（オプション構造体を使用）である。複数行のシグネチャでは、各パラメータを独立した行に置く。

## 変数宣言

非ゼロ値には `:=` を、ゼロ値初期化には `var` をSHOULDで使用する。この形式は意図を示す: `var` は「これはゼロから始まる」を意味する。

```go
var count int              // zero value, set later
name := "default"          // non-zero, := is appropriate
var buf bytes.Buffer       // zero value is ready to use
```

### スライスとマップの初期化

スライスとマップはMUSTで明示的に初期化し、nilにしない。nilマップは書き込み時にpanicする。nilスライスはJSONで `null` にシリアライズされ（空スライスの `[]` とは異なる）、APIの利用者を驚かせる。

```go
users := []User{}                       // always initialized
m := map[string]int{}                   // always initialized
users := make([]User, 0, len(ids))      // preallocate when capacity is known
m := make(map[string]int, len(items))   // preallocate when size is known
```

投機的に事前確保しないこと — 一般的なケースが10要素の場合、`make([]T, 0, 1000)` はメモリの無駄になる。

### 複合リテラル

複合リテラルはMUSTでフィールド名を使用する — 位置指定フィールドは型がフィールドを追加・並べ替えた際に壊れる:

```go
srv := &http.Server{
    Addr:         ":8080",
    ReadTimeout:  5 * time.Second,
    WriteTimeout: 10 * time.Second,
}
```

## 制御フロー

### ネストの削減

エラーとエッジケースはMUSTで最初に処理する（早期リターン）。正常パスは最小限のインデントに保つ:

```go
func process(data []byte) (*Result, error) {
    if len(data) == 0 {
        return nil, errors.New("empty data")
    }

    parsed, err := parse(data)
    if err != nil {
        return nil, fmt.Errorf("parsing: %w", err)
    }

    return transform(parsed), nil
}
```

### 不要な `else` の排除

`if` 本体が `return`/`break`/`continue` で終わる場合、`else` はMUSTで削除する。単純な代入にはデフォルト値設定後に上書きパターンを使用する — デフォルト値を代入し、独立した条件または `switch` で上書きする:

```go
// Good — default-then-override with switch (cleanest for mutually exclusive overrides)
level := slog.LevelInfo
switch {
case debug:
    level = slog.LevelDebug
case verbose:
    level = slog.LevelWarn
}

// Bad — else-if chain hides that there's a default
if debug {
    level = slog.LevelDebug
} else if verbose {
    level = slog.LevelWarn
} else {
    level = slog.LevelInfo
}
```

### 複雑な条件式と初期化スコープ

`if` 条件が3つ以上のオペランドを持つ場合、MUSTで名前付きブール値に抽出する — `||` の壁は読みにくく、ビジネスロジックを隠す。短絡評価の利点のため、コストの高いチェックはインラインに保つ。[Details](./references/details.md)

```go
// Good — named booleans make intent clear
isAdmin := user.Role == RoleAdmin
isOwner := resource.OwnerID == user.ID
isPublicVerified := resource.IsPublic && user.IsVerified
if isAdmin || isOwner || isPublicVerified || permissions.Contains(PermOverride) {
    allow()
}
```

チェックにのみ必要な変数は `if` ブロックにスコープを限定する:

```go
if err := validate(input); err != nil {
    return err
}
```

### if-elseチェーンよりswitchを優先

同じ変数を複数回比較する場合は `switch` を優先する:

```go
switch status {
case StatusActive:
    activate()
case StatusInactive:
    deactivate()
default:
    panic(fmt.Sprintf("unexpected status: %d", status))
}
```

## 関数設計

- 関数はSHOULDで**短く焦点を絞る** — 1つの関数に1つの仕事。
- 関数はSHOULDで**4つ以下のパラメータ**にする。それ以上の場合はオプション構造体を使用する（`samber/cc-skills-golang@golang-design-patterns` スキルを参照）。
- **パラメータの順序**: `context.Context` を最初に、次に入力、次に出力先。
- 素の戻り値は非常に短い関数（1-3行）で戻り値が明らかな場合に有用だが、戻り値を見つけるためにスクロールが必要になると混乱を招く — 長い関数では戻り値に明示的に名前を付ける。

```go
func FetchUser(ctx context.Context, id string) (*User, error)
func SendEmail(ctx context.Context, msg EmailMessage) error  // grouped into struct
```

### イテレーションには `range` を優先

インデックスベースのループよりSHOULDで `range` を使用する。単純なカウントには `range n`（Go 1.22+）を使用する。

```go
for _, user := range users {
    process(user)
}
```

## 値渡しとポインタ引数

小さな型（`string`、`int`、`bool`、`time.Time`）は値渡しする。ミューテーションが必要な場合、大きな構造体（約128バイト以上）、またはnilに意味がある場合はポインタを使用する。[Details](./references/details.md)

## ファイル内のコード構成

- **関連する宣言をグループ化**: 型、コンストラクタ、メソッドをまとめる
- **順序**: パッケージドキュメント、インポート、定数、型、コンストラクタ、メソッド、ヘルパー
- **1ファイルに1つの主要な型** — 重要なメソッドがある場合
- **ブランクインポート**（`_ "pkg"`）は副作用（init関数）を登録する。`main` パッケージとテストパッケージに限定することで、副作用がアプリケーションルートで可視化され、ライブラリコード内に隠れない
- **ドットインポート**は名前空間を汚染し、名前の出所が分からなくなる — ライブラリコードでは決して使用しない
- **積極的に非公開にする** — 後からいつでも公開できるが、非公開にするのは破壊的変更

## 文字列の扱い

単純な変換には `strconv`（より高速）、複雑なフォーマットには `fmt.Sprintf` を使用する。エラーメッセージでは `%q` を使って文字列の境界を可視化する。ループには `strings.Builder`、単純な結合には `+` を使用する。

## 型変換

明示的で狭い変換を優先する。具象型で十分な場合は `any` よりジェネリクスを使用する:

```go
func Contains[T comparable](slice []T, target T) bool  // not []any
```

## 哲学

- **「少しのコピーは少しの依存関係より良い」**
- **`slices` と `maps` 標準パッケージを使用する**。filter/group-by/chunkには `github.com/samber/lo` を使用
- **「リフレクションは決して明確ではない」** — 必要でない限り `reflect` を避ける
- **早すぎる抽象化をしない** — パターンが安定してから抽出する
- **公開サーフェスを最小化する** — すべてのエクスポートされた名前はコミットメント

## コードスタイルレビューの並列化

大規模コードベースでコードスタイルをレビューする場合、最大5つの並列サブエージェント（Agentツール経由）を使用し、それぞれ独立したスタイルの関心事（例: 制御フロー、関数設計、変数宣言、文字列処理、コード構成）を担当させる。

## リンターによる強制

多くのルールは自動的に強制される: `gofmt`、`gofumpt`、`goimports`、`gocritic`、`revive`、`wsl_v5`。→ See the `samber/cc-skills-golang@golang-linter` skill.

## クロスリファレンス

- → See the `samber/cc-skills-golang@golang-naming` skill for identifier naming conventions
- → See the `samber/cc-skills-golang@golang-structs-interfaces` skill for pointer vs value receivers, interface design
- → See the `samber/cc-skills-golang@golang-design-patterns` skill for functional options, builders, constructors
- → See the `samber/cc-skills-golang@golang-linter` skill for automated formatting enforcement
