---
name: golang-naming
description: "Go (Golang) naming conventions — covers packages, constructors, structs, interfaces, constants, enums, errors, booleans, receivers, getters/setters, functional options, acronyms, test functions, and subtest names. Use this skill when writing new Go code, reviewing or refactoring, choosing between naming alternatives (New vs NewTypeName, isConnected vs connected, ErrNotFound vs NotFoundError, StatusReady vs StatusUnknown at iota 0), debating Go package names (utils/helpers anti-patterns), or asking about Go naming best practices. Also trigger when the user mentions MixedCaps vs snake_case, ALL_CAPS constants, Get-prefix on getters, or error string casing. Do NOT use for general Go implementation questions that don't involve naming decisions."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🏷️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-naming` skill takes precedence.

# Go 命名規則

Goは短く読みやすい名前を好みます。大文字・小文字で可視性が制御されます — 大文字はエクスポート、小文字は非エクスポートです。すべての識別子はMixedCapsを使用しなければならず（MUST）、アンダースコアは使用してはいけません（NEVER）。

> "Clear is better than clever." — Go Proverbs
>
> "Design the architecture, name the components, document the details." — Go Proverbs

ルールを無視する場合は、コードにコメントを追加してください。

## クイックリファレンス

| 要素 | 規則 | 例 |
| --- | --- | --- |
| パッケージ | 小文字、単一単語 | `json`, `http`, `tabwriter` |
| ファイル | 小文字、アンダースコア可 | `user_handler.go` |
| エクスポート名 | UpperCamelCase | `ReadAll`, `HTTPClient` |
| 非エクスポート | lowerCamelCase | `parseToken`, `userCount` |
| インターフェース | メソッド名 + `-er` | `Reader`, `Closer`, `Stringer` |
| 構造体 | MixedCaps 名詞 | `Request`, `FileHeader` |
| 定数 | MixedCaps（ALL_CAPSではない） | `MaxRetries`, `defaultTimeout` |
| レシーバー | 1-2文字の略称 | `func (s *Server)`, `func (b *Buffer)` |
| エラー変数 | `Err` プレフィックス | `ErrNotFound`, `ErrTimeout` |
| エラー型 | `Error` サフィックス | `PathError`, `SyntaxError` |
| コンストラクタ | `New`（単一型）または `NewTypeName`（複数型） | `ring.New`, `http.NewRequest` |
| ブーリアンフィールド | **フィールド**とメソッドに `is`, `has`, `can` プレフィックス | `isReady`, `IsConnected()` |
| テスト関数 | `Test` + 関数名 | `TestParseToken` |
| 頭字語 | すべて大文字またはすべて小文字 | `URL`, `HTTPServer`, `xmlParser` |
| バリアント: context | `WithContext` サフィックス | `FetchWithContext`, `QueryContext` |
| バリアント: in-place | `In` サフィックス | `SortIn()`, `ReverseIn()` |
| バリアント: error | `Must` プレフィックス | `MustParse()`, `MustLoadConfig()` |
| オプション関数 | `With` + フィールド名 | `WithPort()`, `WithLogger()` |
| 列挙型 (iota) | 型名プレフィックス、ゼロ値 = unknown | `StatusUnknown` at 0, `StatusReady` |
| 名前付き戻り値 | 説明的、ドキュメント用途のみ | `(n int, err error)` |
| エラー文字列 | 小文字（頭字語含む）、句読点なし | `"image: unknown format"`, `"invalid id"` |
| インポートエイリアス | 短く、衝突時のみ | `mrand "math/rand"`, `pb "app/proto"` |
| フォーマット関数 | `f` サフィックス | `Errorf`, `Wrapf`, `Logf` |
| テストテーブルフィールド | `got`/`expected` プレフィックス | `input string`, `expected int` |

## MixedCaps

すべてのGo識別子は `MixedCaps`（または `mixedCaps`）を使用しなければなりません（MUST）。識別子にアンダースコアを使用してはいけません（NEVER） — 唯一の例外はテスト関数のサブケース（`TestFoo_InvalidInput`）、生成コード、OS/cgo連携です。これは装飾的なものではなく機能的に重要です — Goのエクスポート機構は大文字・小文字に依存しており、ツールはMixedCapsを前提としています。

```go
// ✓ Good
MaxPacketSize
userCount
parseHTTPResponse

// ✗ Bad — these conventions conflict with Go's export mechanism and tooling expectations
MAX_PACKET_SIZE   // C/Python style
max_packet_size   // snake_case
kMaxBufferSize    // Hungarian notation
```

## 吃音（スタッタリング）を避ける

Goの呼び出し元には常にパッケージ名が含まれるため、識別子でそれを繰り返すと読み手の時間を無駄にします — `http.HTTPClient` は「HTTP」を2回読ませることになります。名前はパッケージ名、型名、または周囲のコンテキストに既に存在する情報を繰り返してはなりません（MUST NOT）。

```go
// Good — clean at the call site
http.Client       // not http.HTTPClient
json.Decoder      // not json.JSONDecoder
user.New()        // not user.NewUser()
config.Parse()    // not config.ParseConfig()

// In package sqldb:
type Connection struct{}  // not DBConnection — "db" is already in the package name

// Anti-stutter applies to ALL exported types, not just the primary struct:
// In package dbpool:
type Pool struct{}        // not DBPool
type Status struct{}      // not PoolStatus — callers write dbpool.Status
type Option func(*Pool)   // not PoolOption
```

## 見落とされがちな規則

これらの規則は正しいものの自明ではなく、命名ミスの最も一般的な原因です：

**コンストラクタの命名:** パッケージが単一のプライマリ型をエクスポートする場合、コンストラクタは `New()` であり、`NewTypeName()` ではありません。これによりスタッタリングを避けます — 呼び出し元は `apiclient.NewClient()` ではなく `apiclient.New()` と書きます。`NewTypeName()` を使うのは、パッケージに複数のコンストラクト可能な型がある場合のみです（例: `http.NewRequest`, `http.NewServeMux`）。

**ブーリアン構造体フィールド:** 非エクスポートのブーリアンフィールドは `is`/`has`/`can` プレフィックスを使用しなければなりません（MUST） — `connected` や `permission` ではなく `isConnected`, `hasPermission` です。エクスポートされたgetterもプレフィックスを保持します: `IsConnected() bool`。これは質問として自然に読め、ブーリアンを他の型と区別します。

**エラー文字列は頭字語を含めすべて小文字です。** `"invalid message ID"` ではなく `"invalid message id"` と書いてください。エラー文字列は他のコンテキストと結合されることが多く（`fmt.Errorf("parsing token: %w", err)`）、文中に大文字小文字が混在すると不自然に見えるためです。センチネルエラーにはパッケージ名をプレフィックスとして含めるべきです: `errors.New("apiclient: not found")`。

**列挙型のゼロ値:** iotaの位置0には常に明示的な `Unknown`/`Invalid` センチネルを配置してください。`var s Status` は暗黙的に0になります — それが `StatusReady` のような実際の状態にマッピングされている場合、意図的にステータスが選択されたかのようにコードが動作する可能性があります。

**サブテスト名:** `t.Run()` のテーブル駆動テストケース名はすべて小文字の説明的なフレーズにすべきです: `"valid id"`, `"empty input"` — `"valid ID"` や `"Valid Input"` ではありません。

## 詳細カテゴリ

完全なルール、例、根拠については以下を参照してください：

- **[パッケージ、ファイル、インポートエイリアス](./references/packages-files.md)** — パッケージ命名（単一単語、小文字、複数形なし）、ファイル命名規則、インポートエイリアスパターン（認知負荷を避けるため衝突時のみ使用）、ディレクトリ構造。

- **[変数、ブーリアン、レシーバー、頭字語](./references/identifiers.md)** — スコープベースの命名（長さはスコープに一致: 3行ループには `i`、パッケージレベルにはより長い名前）、1文字レシーバー規則（Serverには `s`）、頭字語の大文字化（UrlではなくURL、HttpServerではなくHTTPServer）、ブーリアン命名パターン（isReady, hasPrefix）。

- **[関数、メソッド、オプション](./references/functions-methods.md)** — Getter/Setterパターン（Goは `Get` を省略するため `user.Name()` が自然に読める）、コンストラクタ規則（`New` または `NewTypeName`）、名前付き戻り値（ドキュメント用途のみ）、フォーマット関数サフィックス（`Errorf`, `Wrapf`）、関数オプション（`WithPort`, `WithLogger`）。

- **[型、定数、エラー](./references/types-errors.md)** — インターフェース命名（`Reader`, `Closer` の `-er` サフィックス）、構造体命名（名詞、MixedCaps）、定数（MixedCaps、ALL_CAPSではない）、列挙型（`StatusReady` のような型名プレフィックス）、センチネルエラー（`ErrNotFound` 変数）、エラー型（`PathError` サフィックス）、エラーメッセージ規則（小文字、句読点なし）。

- **[テスト命名](./references/testing.md)** — テスト関数命名（`TestFunctionName`）、テーブル駆動テストフィールド規則（`input`, `expected`）、テストヘルパー命名、サブケース命名パターン。

## よくある間違い

| 間違い | 修正 |
| --- | --- |
| `ALL_CAPS` 定数 | Goは大文字小文字を可視性のために使用し、強調のためではない — `MixedCaps` を使用（`MaxRetries`） |
| `GetName()` getter | Goは `Get` を省略する。`user.Name()` は呼び出し元で自然に読めるため。ただしブーリアン述語には `Is`/`Has`/`Can` プレフィックスを保持: `Healthy() bool` ではなく `IsHealthy() bool` |
| `Url`, `Http`, `Json` 頭字語 | 大文字小文字混在の頭字語は曖昧さを生む（`HttpsUrl` — `Https+Url`？）。すべて大文字またはすべて小文字を使用 |
| `this` または `self` レシーバー | Goのメソッドは頻繁に呼ばれる — 視覚的ノイズを減らすため1-2文字の略称を使用（`Server` には `s`） |
| `util`, `helper` パッケージ | これらの名前は内容について何も語らない — 抽象化を説明する具体的な名前を使用 |
| `http.HTTPClient` スタッタリング | パッケージ名は呼び出し元に常に存在する — `http.Client` は「HTTP」を2回読むことを避ける |
| `user.NewUser()` コンストラクタ | 単一のプライマリ型には `New()` を使用 — `user.New()` は型名の繰り返しを避ける |
| `connected bool` フィールド | 裸の形容詞は曖昧 — `isConnected` にすることで真偽の質問として読める |
| `"invalid message ID"` エラー | エラー文字列は頭字語を含めすべて小文字でなければならない — `"invalid message id"` |
| `StatusReady` を iota 0 に配置 | ゼロ値はセンチネルであるべき — `StatusUnknown` を0にすることで未初期化値を検出 |
| `"not found"` エラー文字列 | センチネルエラーにはパッケージ名を含めるべき — `"mypackage: not found"` で発生元を特定 |
| `userSlice` 型名を含む名前 | 型は実装の詳細をエンコードする — `users` は何を保持するかを説明し、方法ではない |
| 一貫しないレシーバー名 | 同じ型のメソッド間で名前を切り替えると読み手が混乱する — 一貫して1つの名前を使用 |
| `snake_case` 識別子 | アンダースコアはGoのMixedCaps規則とツールの期待に反する — `mixedCaps` を使用 |
| 短いスコープに長い名前 | 名前の長さはスコープに一致すべき — 3行ループに `i` で十分、`userIndex` はノイズ |
| 値で定数を命名 | 値は変わるが役割は変わらない — `DefaultPort` はポート変更後も生き残るが `Port8080` は生き残らない |
| `FetchCtx()` context バリアント | `WithContext` がGoの標準サフィックス — `FetchWithContext()` は即座に認識可能 |
| `sort()` がin-placeだが `In` なし | 読み手は関数が新しい値を返すと仮定する。`SortIn()` は変異を示す |
| `parse()` がエラー時にpanic | `MustParse()` は失敗時にpanicすることを呼び出し元に警告する — サプライズは名前に含めるべき |
| `With*`, `Set*`, `Use*` の混在 | コードベース全体での一貫性 — `With*` が関数オプションのGo規則 |
| 複数形パッケージ名 | Go規則は単数形（`net/urls` ではなく `net/url`） — インポートパスの一貫性を保つ |
| `Wrapf` に `f` サフィックスなし | `f` サフィックスはフォーマット文字列セマンティクスを示す — `Wrapf`, `Errorf` はフォーマット引数を渡すことを伝える |
| 不要なインポートエイリアス | エイリアスは認知負荷を増やす。衝突時のみエイリアスを使用 — `mrand "math/rand"` |
| 一貫しない概念名 | 同じ概念に `user`/`account`/`person` を使用すると読み手が同義語を追跡する必要がある — 1つの名前を選ぶ |

## リンターによる強制

多くの命名規則の問題はリンターによって自動的に検出されます: `revive`, `predeclared`, `misspell`, `errname`。設定と使用方法については `samber/cc-skills-golang@golang-linter` skill を参照してください。

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-code-style` skill for broader formatting and style decisions
- → See `samber/cc-skills-golang@golang-structs-interfaces` skill for interface naming depth and receiver design
- → See `samber/cc-skills-golang@golang-linter` skill for automated enforcement (revive, predeclared, misspell, errname)
