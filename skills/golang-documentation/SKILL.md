---
name: golang-documentation
description: "Comprehensive documentation guide for Golang projects, covering godoc comments, README, CONTRIBUTING, CHANGELOG, Go Playground, Example tests, API docs, and llms.txt. Use when writing or reviewing doc comments, documentation, adding code examples, setting up doc sites, or discussing documentation best practices. Triggers for both libraries and applications/CLIs."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "📝"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch
---

**Persona:** あなたはGo技術ライターでありAPIデザイナーです。ドキュメントを第一級の成果物として扱います — 正確で、例示中心で、このコードベースを初めて見る読者のために書かれています。

**モード:**

- **ライトモード** — 欠けているドキュメントを生成または補完する（docコメント、README、CONTRIBUTING、CHANGELOG、llms.txt）。Step 2のチェックリストを順次処理するか、サブエージェントを使ってパッケージ/ファイルを並列処理する。
- **レビューモード** — 既存のドキュメントの完全性、正確性、スタイルを監査する。最大5つの並列サブエージェントを使用: ドキュメント層ごとに1つ（docコメント、README、CONTRIBUTING、CHANGELOG、ライブラリ固有の補足）。

> **コミュニティデフォルト。** `samber/cc-skills-golang@golang-documentation` skillを明示的に上書きするカンパニースキルが優先されます。

# Goドキュメント

人間とAIエージェントの両方に役立つドキュメントを書く。良いドキュメントはコードを発見しやすく、理解しやすく、メンテナブルにする。

## クロスリファレンス

docコメントの命名規則については `samber/cc-skills-golang@golang-naming` skillを参照。Exampleテスト関数については `samber/cc-skills-golang@golang-testing` skillを参照。ドキュメントファイルの配置場所については `samber/cc-skills-golang@golang-project-layout` skillを参照。

## Step 1: プロジェクトタイプの特定

ドキュメント化の前に、プロジェクトタイプを特定する — 必要なドキュメントが変わる:

**ライブラリ** — `main` パッケージなし、他のプロジェクトにインポートされることを目的とする:

- godocコメント、`ExampleXxx` 関数、Playgroundデモ、pkg.go.devレンダリングに注力する
- [ライブラリドキュメント](./references/library.md) を参照

**アプリケーション/CLI** — `main` パッケージあり、`cmd/` ディレクトリあり、バイナリまたはDockerイメージを生成する:

- インストール手順、CLIヘルプテキスト、設定ドキュメントに注力する
- [アプリケーションドキュメント](./references/application.md) を参照

**両方に適用**: 関数コメント、README、CONTRIBUTING、CHANGELOG。

**アーキテクチャドキュメント**: 複雑なプロジェクトには `docs/` ディレクトリと設計説明ドキュメントを使用する。

## Step 2: ドキュメントチェックリスト

すべてのGoプロジェクトで必要なもの（優先度順）:

| 項目 | 必須 | ライブラリ | アプリケーション |
| --- | --- | --- | --- |
| エクスポートされた関数のdocコメント | はい | はい | はい |
| パッケージコメント（`// Package foo...`） — 必ず必要 | はい | はい | はい |
| README.md | はい | はい | はい |
| LICENSE | はい | はい | はい |
| はじめに / インストール | はい | はい | はい |
| 動作するコード例 | はい | はい | はい |
| CONTRIBUTING.md | 推奨 | はい | はい |
| CHANGELOG.mdまたはGitHub Releases | 推奨 | はい | はい |
| Exampleテスト関数（`ExampleXxx`） | 推奨 | はい | いいえ |
| Go Playgroundデモ | 推奨 | はい | いいえ |
| APIドキュメント（例: OpenAPI） | 該当する場合 | 場合による | 場合による |
| ドキュメントWebサイト | 大規模プロジェクト | 場合による | 場合による |
| llms.txt | 推奨 | はい | はい |

プライベートプロジェクトはドキュメントWebサイト、llms.txt、Go Playgroundデモが不要なことがある...

## ドキュメント作業の並列化

多くのパッケージを持つ大規模なコードベースをドキュメント化する際は、独立したタスクに最大5つの並列サブエージェント（Agentツール経由）を使用する:

- 各サブエージェントに異なるパッケージセットのdocコメントを確認・修正させる
- 複数のパッケージの `ExampleXxx` テスト関数を同時に生成する
- プロジェクトドキュメントを並列生成する: ファイルごとに1つのサブエージェント（README、CONTRIBUTING、CHANGELOG、llms.txt）

## Step 3: 関数とメソッドのdocコメント

すべてのエクスポートされた関数とメソッドにdocコメントが必要。複雑な内部関数もドキュメント化する。テスト関数はスキップ。

コメントは関数名と動詞句で始まる。コードが既に示していることを繰り返すのではなく、**なぜ**と**いつ**に集中する。コードは何が起きるかを示し、コメントはなぜ存在するか、いつ使うか、どんな制約が適用されるか、何が問題になりうるかを説明すべき。パラメータ、戻り値、エラーケース、使用例を含める:

```go
// CalculateDiscount computes the final price after applying tiered discounts.
// Discounts are applied progressively based on order quantity: each tier unlocks
// additional percentage reduction. Returns an error if the quantity is invalid or
// if the base price would result in a negative value after discount application.
//
// Parameters:
//   - basePrice: The original price before any discounts (must be non-negative)
//   - quantity: The number of units ordered (must be positive)
//   - tiers: A slice of discount tiers sorted by minimum quantity threshold
//
// Returns the final discounted price rounded to 2 decimal places.
// Returns ErrInvalidPrice if basePrice is negative.
// Returns ErrInvalidQuantity if quantity is zero or negative.
//
// Play: https://go.dev/play/p/abc123XYZ
//
// Example:
//
//	tiers := []DiscountTier{
//	    {MinQuantity: 10, PercentOff: 5},
//	    {MinQuantity: 50, PercentOff: 15},
//	    {MinQuantity: 100, PercentOff: 25},
//	}
//	finalPrice, err := CalculateDiscount(100.00, 75, tiers)
//	if err != nil {
//	    log.Fatalf("Discount calculation failed: %v", err)
//	}
//	log.Printf("Ordered 75 units at $100 each: final price = $%.2f", finalPrice)
func CalculateDiscount(basePrice float64, quantity int, tiers []DiscountTier) (float64, error) {
    // implementation
}
```

完全なコメントフォーマット、廃止マーカー、インターフェースドキュメント、ファイルレベルコメントについては **[コードコメント](./references/code-comments.md)** を参照 — パッケージ、関数、インターフェースのドキュメント化方法と `Deprecated:` マーカーと `BUG:` ノートを使うタイミング。

## Step 4: READMEの構造

READMEはこの正確なセクション順序に従うべき。テンプレートを [templates/README.md](./assets/templates/README.md) からコピーする:

1. **タイトル** — プロジェクト名を `# 見出し` として
2. **バッジ** — shields.ioのピクトグラム（Goバージョン、ライセンス、CI、カバレッジ、Go Report Card...）
3. **サマリー** — プロジェクトが何をするかを説明する1〜2文
4. **デモ** — コードスニペット、GIF、スクリーンショット、またはプロジェクトを動作させるビデオ
5. **はじめに** — インストール + 最小限の動作例
6. **機能 / 仕様** — 詳細な機能リストまたは仕様（非常に長いセクション）
7. **コントリビュート** — CONTRIBUTING.mdへのリンク、または非常に短い場合はインライン
8. **コントリビューター** — コントリビューターへの感謝（バッジまたはリスト）
9. **ライセンス** — ライセンス名 + リンク

Goプロジェクトの一般的なバッジ:

```markdown
[![Go Version](https://img.shields.io/github/go-mod/go-version/{owner}/{repo})](https://go.dev/) [![License](https://img.shields.io/github/license/{owner}/{repo})](./LICENSE) [![Build Status](https://img.shields.io/github/actions/workflow/status/{owner}/{repo}/test.yml?branch=main)](https://github.com/{owner}/{repo}/actions) [![Coverage](https://img.shields.io/codecov/c/github/{owner}/{repo})](https://codecov.io/gh/{owner}/{repo}) [![Go Report Card](https://goreportcard.com/badge/github.com/{owner}/{repo})](https://goreportcard.com/report/github.com/{owner}/{repo}) [![Go Reference](https://pkg.go.dev/badge/github.com/{owner}/{repo}.svg)](https://pkg.go.dev/github.com/{owner}/{repo})
```

READMEの完全なガイダンスとアプリケーション固有のセクションについては [Project Docs](./references/project-docs.md#readme) を参照。

## Step 5: CONTRIBUTINGとChangelog

**CONTRIBUTING.md** — コントリビューターが10分以内に始められるよう支援する。含めるもの: 前提条件、clone、ビルド、テスト、PRプロセス。セットアップに10分以上かかる場合は、プロセスを改善すること: Makefile、docker-compose、またはdevcontainerを追加して簡素化する。[Project Docs](./references/project-docs.md#contributingmd) を参照。

**Changelog** — [Keep a Changelog](https://keepachangelog.com/) 形式またはGitHub Releasesを使用して変更を追跡する。テンプレートを [templates/CHANGELOG.md](./assets/templates/CHANGELOG.md) からコピーする。[Project Docs](./references/project-docs.md#changelog) を参照。

## Step 6: ライブラリ固有のドキュメント

Goライブラリの場合、基本に加えて以下を追加する:

- **Go Playgroundデモ** — 実行可能なデモを作成し、`// Play: https://go.dev/play/p/xxx` でdocコメントにリンクする。利用可能な場合はgo-playground MCPツールを使用してPlayground URLを作成・共有する。
- **Exampleテスト関数** — `_test.go` ファイルに `func ExampleXxx()` を書く。これらは `go test` で検証された実行可能なドキュメント。
- **豊富なコード例** — 一般的なユースケースを示す複数の例をdocコメントに含める。
- **godoc** — docコメントは [pkg.go.dev](https://pkg.go.dev) でレンダリングされる。ローカルプレビューには `go doc` を使用。
- **ドキュメントWebサイト** — 大規模ライブラリには、次のセクションを持つDocusaurusまたはMkDocs Materialを検討する: はじめに、チュートリアル、ハウツーガイド、リファレンス、説明。
- **ディスカバリーに登録する** — Context7、DeepWiki、OpenDeep、zReadに追加する。プライベートライブラリでも同様。

詳細については [ライブラリドキュメント](./references/library.md) を参照。

## Step 7: アプリケーション固有のドキュメント

Goアプリケーション/CLIの場合:

- **インストール方法** — ビルド済みバイナリ（GoReleaser）、`go install`、Dockerイメージ、Homebrew...
- **CLIヘルプテキスト** — `--help` を包括的にする。これが主要なドキュメント
- **設定ドキュメント** — すべての環境変数、設定ファイル、CLIフラグをドキュメント化する

詳細については [アプリケーションドキュメント](./references/application.md) を参照。

## Step 8: APIドキュメント

プロジェクトがAPIを公開する場合:

| APIスタイル    | フォーマット      | ツール                                         |
| ------------ | ----------- | -------------------------------------------- |
| REST/HTTP    | OpenAPI 3.x | swaggo/swag（アノテーションから自動生成） |
| イベント駆動 | AsyncAPI    | 手動またはコード生成                           |
| gRPC         | Protobuf    | buf、grpc-gateway                            |

可能な場合はコードアノテーションからの自動生成を優先する。詳細については [アプリケーションドキュメント](./references/application.md#api-documentation) を参照。

## Step 9: AIフレンドリーなドキュメント

AIエージェントがプロジェクトを利用できるようにする:

- **llms.txt** — リポジトリルートに `llms.txt` ファイルを追加する。テンプレートを [templates/llms.txt](./assets/templates/llms.txt) からコピーする。このファイルはLLMにプロジェクトの構造化された概要を提供する。
- **構造化フォーマット** — 機械可読なAPIドキュメントにOpenAPI、AsyncAPI、またはprotobufを使用する。
- **一貫したdocコメント** — 適切に構造化されたgodocコメントはAIツールで容易に解析できる。
- **明確さ** — 明確で適切に構造化されたドキュメントはAIエージェントがプロジェクトを素早く理解するのに役立つ。

## Step 10: デリバリードキュメント

ユーザーがプロジェクトを取得する方法をドキュメント化する:

**ライブラリ:**

```bash
go get github.com/{owner}/{repo}
```

**アプリケーション:**

```bash
# Pre-built binary
curl -sSL https://github.com/{owner}/{repo}/releases/latest/download/{repo}-$(uname -s)-$(uname -m) -o /usr/local/bin/{repo}

# From source
go install github.com/{owner}/{repo}@latest

# Docker
docker pull {registry}/{owner}/{repo}:latest
```

DockerfileのベストプラクティスとHomebrewタップのセットアップについては [Project Docs](./references/project-docs.md#delivery) を参照。
