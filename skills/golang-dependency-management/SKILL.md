---
name: golang-dependency-management
description: "Provides dependency management strategies for Golang projects including go.mod management, installing/upgrading packages, semantic versioning, Minimal Version Selection, vulnerability scanning, outdated dependency tracking, dependency size analysis, automated updates with Dependabot/Renovate, conflict resolution, and dependency graph visualization. Use this skill whenever adding, removing, updating, or auditing Go dependencies, resolving version conflicts, setting up automated dependency updates, analyzing binary size, or working with go.work workspaces."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "📦"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - govulncheck
    install:
      - kind: go
        package: golang.org/x/vuln/cmd/govulncheck@latest
        bins: [govulncheck]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent Bash(govulncheck:*) AskUserQuestion
---

**Persona:** You are a Go dependency steward. You treat every new dependency as a long-term maintenance commitment — you ask whether the standard library already solves the problem before reaching for an external package.

# Go 依存関係管理

## AIエージェントルール: 依存関係追加前に確認する

**`go get` で新しい依存関係を追加する前に、AIエージェントは必ずユーザーに確認を求めなければなりません。** AIエージェントはメンテナンスされていない、品質の低い、または標準ライブラリで同等の機能が提供されている不要なパッケージを提案することがあります。`go get -u` で既存の依存関係をアップグレードすることは安全です。

依存関係を提案する前に、以下を提示してください:

- パッケージ名とインポートパス
- 何をするのか、なぜ必要なのか
- 標準ライブラリでユースケースをカバーできるかどうか
- GitHubスター数、最終コミット日、メンテナンス状況（`gh repo view` で確認）
- ライセンスの互換性
- 既知の代替手段

`samber/cc-skills-golang@golang-popular-libraries` スキルには、厳選された本番環境対応ライブラリのリストが含まれています。そのリストからパッケージを推奨することを優先してください。厳選されたオプションがない場合は、無名の代替手段よりもGoチーム（`golang.org/x/...`）や確立された組織の有名なパッケージを優先してください。

## 主要ルール

- `go.sum` は必ずコミットすること — すべての依存バージョンの暗号チェックサムを記録し、`go mod verify` でサプライチェーンの改ざんを検出できるようにします。これがないと、侵害されたプロキシが悪意のあるコードを静かに差し替える可能性があります
- リリース前に必ず `govulncheck ./...` を実行すること — 依存関係ツリー内の既知のCVEを本番環境に到達する前にキャッチします
- 依存関係を追加する前にメンテナンス状況、ライセンス、標準ライブラリの代替手段を確認すること — 依存関係を追加するたびに攻撃面、メンテナンス負担、バイナリサイズが増加します
- 依存関係を変更するコミットの前に必ず `go mod tidy` を実行すること — 未使用のモジュールを削除し、不足しているモジュールを追加して、go.modを正確に保ちます

## go.mod と go.sum

### 基本コマンド

| コマンド            | 目的                                          |
| ----------------- | -------------------------------------------- |
| `go mod tidy`     | 不足している依存関係を追加し、未使用のものを削除する  |
| `go mod download` | モジュールをローカルキャッシュにダウンロードする      |
| `go mod verify`   | キャッシュされたモジュールがgo.sumのチェックサムと一致するか検証する |
| `go mod vendor`   | 依存関係を `vendor/` ディレクトリにコピーする       |
| `go mod edit`     | go.modをプログラム的に編集する（スクリプト、CI）     |
| `go mod graph`    | モジュール依存関係グラフを表示する                  |
| `go mod why`      | モジュールやパッケージが必要な理由を説明する          |

### ベンダリング

ハーメティックビルド（ネットワークアクセスなし）、チェックサム以上の再現性保証、またはモジュールプロキシアクセスがない環境へのデプロイが必要な場合に `go mod vendor` を使用します。CIパイプラインやDockerビルドはベンダリングの恩恵を受けることがあります。依存関係の変更後に `go mod vendor` を実行し、`vendor/` ディレクトリをコミットしてください。

## 依存関係のインストールとアップグレード

### 依存関係の追加

```bash
go get github.com/pkg/errors           # Latest version
go get github.com/pkg/errors@v0.9.1    # Specific version
go get github.com/pkg/errors@latest    # Explicitly latest
go get github.com/pkg/errors@master    # Specific branch (pseudo-version)
```

### アップグレード

```bash
go get -u ./...            # Upgrade ALL direct+indirect deps to latest minor/patch
go get -u=patch ./...      # Upgrade to latest patch only (safer)
go get github.com/pkg@v1.5 # Upgrade specific package
```

定期的なアップデートには **`go get -u=patch` を推奨** — パッチバージョンはパブリックAPIを変更しません（semverの約束）ので、ビルドを壊す可能性は低いです。マイナーバージョンのアップグレードは新しいAPIを追加する場合がありますが、予期せず非推奨化や動作変更が発生することもあります。

### 依存関係の削除

```bash
go get github.com/pkg/errors@none   # Mark for removal
go mod tidy                          # Clean up go.mod and go.sum
```

### CLIツールのインストール

```bash
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
```

`go install` はバイナリをビルドして `$GOPATH/bin` にインストールします。`@latest` または特定のバージョンタグを使用してください — 依存するツールには絶対に `@master` を使わないでください。

### tools.go パターン

本番コードでインポートせずに、モジュール内でツールのバージョンを固定します:

```go
//go:build tools

package tools

import (
    _ "github.com/golangci/golangci-lint/cmd/golangci-lint"
    _ "golang.org/x/vuln/cmd/govulncheck"
)
```

ビルド制約により、このファイルはコンパイルされません。ブランクインポートによりツールが `go.mod` に保持され、`go install` が固定バージョンを使用します。このファイルを作成した後に `go mod tidy` を実行してください。

## 詳細ガイド

- **[バージョニングとMVS](./references/versioning.md)** — セマンティックバージョニングのルール（major.minor.patch）、各番号をいつインクリメントするか、プレリリースバージョン、最小バージョン選択（MVS）アルゴリズム（なぜ単に「最新」を選べないのか）、メジャーバージョンサフィックスの規約（破壊的変更に対するv0、v1、v2サフィックス）。

- **[依存関係の監査](./references/auditing.md)** — `govulncheck` による脆弱性スキャン、古い依存関係の追跡、どの依存関係がバイナリを大きくしているかの分析（`goweight`）、`go.mod` をクリーンに保つためのテスト専用依存とバイナリ依存の区別。

- **[依存関係の競合と解決](./references/conflicts.md)** — バージョン競合の診断（互換性のないバージョンを要求した場合に `go get` が何をするか）、解決戦略（ローカル開発用の `replace` ディレクティブ、壊れたバージョン用の `exclude`、スキップすべき公開バージョン用の `retract`）、依存関係ツリー全体での競合のワークフロー。

- **[Goワークスペース](./references/workspaces.md)** — マルチモジュール開発用の `go.work` ファイル（例: ライブラリ + サンプルアプリケーション）、ワークスペースとモノレポの使い分け、ワークスペースのベストプラクティス。

- **[依存関係の自動更新](./references/automated-updates.md)** — DependabotまたはRenovateによる自動依存関係更新PRの設定、自動マージ戦略（いつ自動マージするか vs レビューを要求するか）、セキュリティアップデートの処理。

- **[依存関係グラフの可視化](./references/visualization.md)** — `go mod graph` で完全な依存関係ツリーを検査し、`modgraphviz` で可視化し、どの依存チェーンが肥大化の原因かを見つけるインタラクティブツール。

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-continuous-integration` skill for Dependabot/Renovate CI setup
- → See `samber/cc-skills-golang@golang-security` skill for vulnerability scanning with govulncheck
- → See `samber/cc-skills-golang@golang-popular-libraries` skill for vetted library recommendations

## クイックリファレンス

```bash
# Start a new module
go mod init github.com/user/project

# Add a dependency
go get github.com/pkg/errors@v0.9.1

# Upgrade all deps (patch only, safer)
go get -u=patch ./...

# Remove unused deps
go mod tidy

# Check for vulnerabilities
govulncheck ./...

# Check for outdated deps
go list -u -m -json all | go-mod-outdated -update -direct

# Analyze binary size by dependency
goweight

# Understand why a dep exists
go mod why -m github.com/some/module

# Visualize dependency graph
go mod graph | modgraphviz | dot -Tpng -o deps.png

# Verify checksums
go mod verify
```
