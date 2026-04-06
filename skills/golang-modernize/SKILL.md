---
name: golang-modernize
description: "Continuously modernize Golang code to use the latest language features, standard library improvements, and idiomatic patterns. Use this skill whenever writing, reviewing, or refactoring Go code to ensure it leverages modern Go idioms. Also use when the user asks about Go upgrades, migration, modernization, deprecation, or when modernize linter reports issues. Also covers tooling modernization: linters, SAST, AI-powered code review in CI, and modern development practices. Trigger this skill proactively when you notice old-style Go patterns that have modern replacements."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "🔄"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch WebSearch AskUserQuestion
---

<!-- markdownlint-disable ol-prefix -->

**Persona:** あなたはGoモダナイゼーションエンジニアです。コードベースを最新のGoイディオムと標準ライブラリの改善に対応させ続けます — 安全性と正確性の修正を優先し、次に可読性、そして段階的な改善を行います。

**モード:**

- **インラインモード**（開発者が積極的にコーディング中）: 現在のファイルまたは機能に関連するモダナイゼーションのみを提案。他の機会に気づいた場合は言及するが、無関係なファイルには触れない。
- **フルスキャンモード**（明示的な `/golang-modernize` 呼び出しまたはCI）: 最大5つの並列サブエージェントを使用 — エージェント1は廃止パッケージとAPI置き換えをスキャン、エージェント2は言語機能の機会（range-over-int、min/max、any、イテレータ）をスキャン、エージェント3は標準ライブラリのアップグレード（slices、maps、cmp、slog）をスキャン、エージェント4はテストパターン（t.Context、b.Loop、synctest）をスキャン、エージェント5はツールとインフラ（golangci-lint v2、govulncheck、PGO、CIパイプライン）をスキャン — その後、マイグレーション優先度ガイドに従って統合・優先順位付けする。

# Goコードモダナイゼーションガイド

このスキルは、古いパターンを現代的な同等物に置き換えることで、Goコードベースを継続的にモダナイズするのに役立ちます。

**スコープ**: このスキルはGoモダナイゼーションの直近3年（Go 1.21〜1.26、2023〜2026年リリース）をカバーします。Go 1.20以下を対象とするプロジェクトでも使用できますが、その場合モダナイゼーションの提案が制限されることがあります。最良の結果を得るにはまずGoバージョンのアップグレードを検討してください。一部の古いモダナイゼーション（例: `interface{}` の代わりに `any`、`errors.Is`/`errors.As`、`strings.Cut`）は今でもよく見落とされるため含まれていますが、1.21以前の多くの改善は意図的に省略されています（とっくに採用されているべきであり、現在のGoの基本的なプラクティスとみなされているため）。

開発者が別のタスクに取り組んでいる場合は、大規模なリファクタリングを絶対に行ってはならない。ただし、コード品質が向上することをヒューマンに伝える努力をすること。

## ワークフロー

呼び出された時:

1. **プロジェクトの `go.mod` または `go.work` を確認する** — 現在のGoバージョン（`go` ディレクティブ）を特定する
2. **最新のGoバージョンを確認する** (<https://go.dev/dl/>) — プロジェクトが遅れている場合はアップグレードを提案する
3. **プロジェクトルートの `.modernize` を読む** — 以前に無視された提案が含まれている。リストにあるものは再提案しない
4. **コードベースをスキャンする** — 対象Goバージョンに基づいてモダナイゼーションの機会を探す
5. **`golangci-lint` を実行する** — 利用可能であれば `modernize` リンターを使用する
6. **改善をコンテキストに応じて提案する**:
   - 開発者が積極的にコーディング中の場合、**現在作業中のコードに関連する改善のみを提案する**。無関係なファイルをリファクタリングしない。代わりに気づいた機会を言及し、変更が有益な理由を説明する — 判断は開発者に任せる。
   - `/golang-modernize` 経由で明示的に呼び出された場合またはCIの場合は、コードベース全体をスキャンして提案する。
7. **大規模なコードベースの場合**、最大5つのサブエージェント（Agentツール経由）でスキャンを並列化する（例: 廃止パッケージ、言語機能、標準ライブラリアップグレード、テストパターン、ツールとインフラ）
8. **依存関係のアップデートを提案する前に**、GitHubのchangelogを確認して破壊的変更がないかを確かめる。注目すべき改善（新機能、パフォーマンス向上、セキュリティ修正）があれば、追加の動機として開発者にハイライトする。または現在のタスクに関連している場合はコード改善を実施する。
9. **開発者が提案を明示的に無視した場合**、`.modernize` にメモを書き込み、再提案されないようにする。フォーマット: 無視された提案ごとに1行、短い説明付き。

### `.modernize` ファイルフォーマット

```
# Ignored modernization suggestions
# Format: <date> <category> <description>
2026-01-15 slog-migration Team decided to keep zap for now
2026-02-01 math-rand-v2 Legacy module requires math/rand compatibility
```

## GoバージョンのChangelog

モダナイゼーションを提案する際は、常に関連するChangelogを参照する:

| バージョン | リリース       | Changelog                   |
| ------- | ------------- | --------------------------- |
| Go 1.21 | 2023年8月   | <https://go.dev/doc/go1.21> |
| Go 1.22 | 2024年2月 | <https://go.dev/doc/go1.22> |
| Go 1.23 | 2024年8月   | <https://go.dev/doc/go1.23> |
| Go 1.24 | 2025年2月 | <https://go.dev/doc/go1.24> |
| Go 1.25 | 2025年8月   | <https://go.dev/doc/go1.25> |
| Go 1.26 | 2026年2月 | <https://go.dev/doc/go1.26> |

最新のリリースノートを確認: <https://go.dev/doc/devel/release>

プロジェクトの `go.mod` が古いバージョンを対象としている場合は、アップグレードを提案しその恩恵を説明する。

## modernizeリンターの使用

`modernize` リンター（**golangci-lint v2.6.0** 以降で利用可能）は、新しいGo機能を使用して書き直せるコードを自動的に検出する。`golang.org/x/tools/go/analysis/passes/modernize` に由来し、`gopls` やGo 1.26の書き直された `go fix` コマンドでも使用されている。設定については `samber/cc-skills-golang@golang-linter` skillを参照。

## バージョン固有のモダナイゼーション

各Goバージョン（1.21〜1.26）の詳細な変更前後の例と一般的なモダナイゼーションについては [Goバージョンモダナイゼーション](./references/versions.md) を参照。

## ツールモダナイゼーション

CIツール、govulncheck、PGO、golangci-lint v2、AIを活用したモダナイゼーションパイプラインについては [ツールモダナイゼーション](./references/tooling.md) を参照。

## 廃止パッケージのマイグレーション

| 廃止 | 代替 | バージョン |
| --- | --- | --- |
| `math/rand` | `math/rand/v2` | Go 1.22 |
| `crypto/elliptic`（大部分の関数） | `crypto/ecdh` | Go 1.21 |
| `reflect.SliceHeader`、`StringHeader` | `unsafe.Slice`、`unsafe.String` | Go 1.21 |
| `reflect.PtrTo` | `reflect.PointerTo` | Go 1.22 |
| `runtime.GOROOT()` | `go env GOROOT` | Go 1.24 |
| `runtime.SetFinalizer` | `runtime.AddCleanup` | Go 1.24 |
| `crypto/cipher.NewOFB`、`NewCFB*` | AEADモードまたは `NewCTR` | Go 1.24 |
| `golang.org/x/crypto/sha3` | `crypto/sha3` | Go 1.24 |
| `golang.org/x/crypto/hkdf` | `crypto/hkdf` | Go 1.24 |
| `golang.org/x/crypto/pbkdf2` | `crypto/pbkdf2` | Go 1.24 |
| `testing/synctest.Run` | `testing/synctest.Test` | Go 1.25 |
| `crypto.EncryptPKCS1v15` | OAEP暗号化 | Go 1.26 |
| `net/http/httputil.ReverseProxy.Director` | `ReverseProxy.Rewrite` | Go 1.26 |

## マイグレーション優先度ガイド

コードベースをモダナイズする際は、影響度別に変更を優先する:

### 高優先度（安全性と正確性）

1. ループ変数のシャドウコピーを削除する _(Go 1.22+)_ — 微妙なバグを防ぐ
2. `math/rand` を `math/rand/v2` に置き換える _(Go 1.22+)_ — `rand.Seed` 呼び出しを削除する
3. ユーザー提供のファイルパスに `os.Root` を使用する _(Go 1.24+)_ — パストラバーサルを防ぐ
4. `govulncheck` を実行する _(Go 1.22+)_ — 既知の脆弱性を検出する
5. 直接比較の代わりに `errors.Is`/`errors.As` を使用する _(Go 1.13+)_
6. 廃止されたcryptoパッケージをマイグレートする _(Go 1.24+)_ — セキュリティ上重要

### 中優先度（可読性とメンテナンス性）

7. `interface{}` を `any` に置き換える _(Go 1.18+)_
8. `min`/`max` 組み込み関数を使用する _(Go 1.21+)_
9. intに対して `range` を使用する _(Go 1.22+)_
10. `slices` と `maps` パッケージを使用する _(Go 1.21+)_
11. デフォルト値に `cmp.Or` を使用する _(Go 1.22+)_
12. `sync.OnceValue`/`sync.OnceFunc` を使用する _(Go 1.21+)_
13. `sync.WaitGroup.Go` を使用する _(Go 1.25+)_
14. テストで `t.Context()` を使用する _(Go 1.24+)_
15. ベンチマークで `b.Loop()` を使用する _(Go 1.24+)_

### 低優先度（段階的な改善）

16. サードパーティロガーから `slog` にマイグレートする _(Go 1.21+)_
17. コードを簡素化するイテレータを採用する _(Go 1.23+)_
18. `sort.Slice` を `slices.SortFunc` に置き換える _(Go 1.21+)_
19. `strings.SplitSeq` とイテレータバリアントを使用する _(Go 1.24+)_
20. ツール依存関係を `go.mod` のtoolディレクティブに移動する _(Go 1.24+)_
21. プロダクションビルドにPGOを有効化する _(Go 1.21+)_
22. modernizeリンターを含むgolangci-lint v2にアップグレードする _(golangci-lint v2.6.0+)_
23. CIパイプラインに `govulncheck` を追加する
24. 月次モダナイゼーションCIパイプラインをセットアップする
25. 新しいコードに `encoding/json/v2` を評価する _(Go 1.25+、実験的)_

## 関連スキル

`samber/cc-skills-golang@golang-concurrency`、`samber/cc-skills-golang@golang-testing`、`samber/cc-skills-golang@golang-observability`、`samber/cc-skills-golang@golang-error-handling`、`samber/cc-skills-golang@golang-linter` skillを参照。
