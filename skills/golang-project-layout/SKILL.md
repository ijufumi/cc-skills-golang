---
name: golang-project-layout
description: "Provides a guide for setting up Golang project layouts and workspaces. Use this whenever starting a new Go project, organizing an existing codebase, setting up a monorepo with multiple packages, creating CLI tools with multiple main packages, or deciding on directory structure. Apply this for any Go project initialization or restructuring work."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "📁"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** あなたはGoプロジェクトアーキテクトです。問題に合わせた適切なサイズの構造を選択します — スクリプトはフラットのまま、サービスは実際の複雑さで正当化される場合のみ層を追加します。

# Goプロジェクトレイアウト

## アーキテクチャの決定: まず尋ねる

新しいプロジェクトを始める際は、**開発者に**好みのソフトウェアアーキテクチャ（クリーンアーキテクチャ、ヘキサゴナル、DDD、フラット構造など）を尋ねる。小規模プロジェクトを過剰に構造化してはならない — 100行のCLIツールに抽象化レイヤーや依存性注入は不要。

ファイルツリーとコード例を含む詳細なアーキテクチャガイドについては → See `samber/cc-skills-golang@golang-design-patterns` skill。

## 依存性注入: 次に尋ねる

アーキテクチャが決まったら、**開発者に**どの依存性注入アプローチを使いたいか尋ねる: 手動コンストラクタインジェクション、DIライブラリ（samber/do、google/wire、uber-go/dig+fx）、またはなし。この選択はサービスの配線方法、ライフサイクル（ヘルスチェック、グレースフルシャットダウン）の管理方法、プロジェクトの構造に影響する。完全な比較とデシジョンテーブルについては `samber/cc-skills-golang@golang-dependency-injection` skillを参照。

## 12-Factor App

アプリケーション（サービス、API、ワーカー）には [12-Factor App](https://12factor.net/) の規約に従う: 環境変数による設定、標準出力へのログ、ステートレスプロセス、グレースフルシャットダウン、アタッチされたリソースとしてのバッキングサービス、ワンオフコマンドとしての管理タスク（例: `cmd/migrate/`）。

## クイックスタート: プロジェクトタイプの選択

| プロジェクトタイプ | 使用するタイミング | 主要なディレクトリ |
| --- | --- | --- |
| **CLIツール** | コマンドラインアプリケーションのビルド | `cmd/{name}/`、`internal/`、オプション `pkg/` |
| **ライブラリ** | 他者向けの再利用可能なコードの作成 | `pkg/{name}/`、プライベートコード用 `internal/` |
| **サービス** | HTTP API、マイクロサービス、またはWebアプリ | `cmd/{service}/`、`internal/`、`api/`、`web/` |
| **モノリポ** | 複数の関連パッケージ/モジュール | `go.work`、パッケージごとの別モジュール |
| **ワークスペース** | 複数のローカルモジュールの開発 | `go.work`、replaceディレクティブ |

## モジュール命名規則

### モジュール名（go.mod）

`go.mod` のモジュールパスは:

- **リポジトリURLと一致しなければならない**: `github.com/username/project-name`
- **小文字のみ使用**: `github.com/you/my-app`（`MyApp` ではない）
- **複数単語にはハイフンを使用**: `user-auth`（`user_auth` や `userAuth` ではない）
- **セマンティックである**: 名前が目的を明確に表現すること

**例:**

```go
// ✅ Good
module github.com/jdoe/payment-processor
module github.com/company/cli-tool

// ❌ Bad
module myproject
module github.com/jdoe/MyProject
module utils
```

### パッケージ命名

パッケージは小文字、単数形で、ディレクトリ名と一致しなければならない。完全なパッケージ命名規則と例については → See `samber/cc-skills-golang@golang-naming` skill。

## ディレクトリレイアウト

すべての `main` パッケージは最小限のロジックで `cmd/` に置く — フラグのパース、依存関係の配線、`Run()` の呼び出し。ビジネスロジックは `internal/` または `pkg/` に属する。非エクスポートパッケージには `internal/` を使用し、`pkg/` は外部消費者に有用なコードの場合のみ使用する。

ユニバーサル、小規模プロジェクト、ライブラリのレイアウトとよくある間違いについては [ディレクトリレイアウトの例](references/directory-layouts.md) を参照。

## 必須設定ファイル

すべてのGoプロジェクトはルートに以下を含めるべき:

- **Makefile** — ビルド自動化。[Makefileテンプレート](assets/Makefile) を参照
- **.gitignore** — gitの無視パターン。[.gitignoreテンプレート](assets/.gitignore) を参照
- **.golangci.yml** — リンター設定。推奨設定については `samber/cc-skills-golang@golang-linter` skillを参照

Cobra + Viperを使ったアプリケーション設定については [config reference](references/config.md) を参照。

## テスト、ベンチマーク、サンプル

`_test.go` ファイルはテスト対象コードと同じ場所に置く。フィクスチャには `testdata/` を使用する。ファイル命名、配置、整理の詳細については [テストレイアウト](references/testing-layout.md) を参照。

## Goワークスペース

モノリポで複数の関連モジュールを開発する際は `go.work` を使用する。セットアップ、構造、コマンドについては [workspaces](references/workspaces.md) を参照。

## 初期化チェックリスト

新しいGoプロジェクトを始める際:

- [ ] **開発者に**好みのソフトウェアアーキテクチャ（クリーン、ヘキサゴナル、DDD、フラットなど）を尋ねる
- [ ] **開発者に**好みのDIアプローチを尋ねる — `samber/cc-skills-golang@golang-dependency-injection` skillを参照
- [ ] プロジェクトタイプを決定する（CLI、ライブラリ、サービス、モノリポ）
- [ ] プロジェクトスコープに合った適切なサイズの構造を選択する
- [ ] モジュール名を選択する（リポジトリURLと一致、小文字、ハイフン）
- [ ] `go version` を実行して現在のgoバージョンを確認する
- [ ] `go mod init github.com/user/project-name` を実行する
- [ ] エントリーポイントに `cmd/{name}/main.go` を作成する
- [ ] プライベートコード用に `internal/` を作成する
- [ ] パブリックライブラリがある場合のみ `pkg/` を作成する
- [ ] モノリポの場合: `go work` を初期化してモジュールを追加する
- [ ] フォーマットを確保するために `gofmt -s -w .` を実行する
- [ ] `/vendor/` とバイナリパターンを含む `.gitignore` を追加する

## 関連スキル

CLIツール構造とCobra/Viperパターンについては → See `samber/cc-skills-golang@golang-cli` skill。DIアプローチの比較と配線については → See `samber/cc-skills-golang@golang-dependency-injection` skill。golangci-lint設定については → See `samber/cc-skills-golang@golang-linter` skill。CI/CDパイプラインのセットアップについては → See `samber/cc-skills-golang@golang-continuous-integration` skill。アーキテクチャパターンについては → See `samber/cc-skills-golang@golang-design-patterns` skill。
