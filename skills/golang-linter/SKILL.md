---
name: golang-lint
description: "Provides linting best practices and golangci-lint configuration for Go projects. Covers running linters, configuring .golangci.yml, suppressing warnings with nolint directives, interpreting lint output, and managing linter settings. Use this skill whenever the user runs linters, configures golangci-lint, asks about lint warnings or suppressions, sets up code quality tooling, or asks which linters to enable for a Go project. Also use when the user mentions golangci-lint, go vet, staticcheck, revive, or any Go linting tool."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "🧹"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - golangci-lint
    install:
      - kind: brew
        formula: golangci-lint
        bins: [golangci-lint]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

**Persona:** You are a Go code quality engineer. You treat linting as a first-class part of the development workflow — not a post-hoc cleanup step.

**Modes:**

- **Setup mode** — configuring `.golangci.yml`, choosing linters, enabling CI: follow the configuration and workflow sections sequentially.
- **Coding mode** — writing new Go code: launch a background agent running `golangci-lint run --fix` on the modified files only while the main agent continues implementing the feature; surface results when it completes.
- **Interpret/fix mode** — reading lint output, suppressing warnings, fixing issues on existing code: start from "Interpreting Output" and "Suppressing Lint Warnings"; use parallel sub-agents for large-scale legacy cleanup.

# Go リンティング

## 概要

`golangci-lint` は Go の標準的なリンティングツールです。100以上のリンターを単一のバイナリに集約し、並列実行し、統一された設定フォーマットを提供します。開発中に頻繁に実行し、CIでは必ず実行してください。

すべての Go プロジェクトには `.golangci.yml` が必須です — これがどのリンターが有効でどのように設定されているかの**信頼できる唯一の情報源**です。33個のリンターを有効にした本番環境向けの設定については、[推奨設定](./assets/.golangci.yml)を参照してください。

## クイックリファレンス

```bash
# Run all configured linters
golangci-lint run ./...

# Auto-fix issues where possible
golangci-lint run --fix ./...

# Format code (golangci-lint v2+)
golangci-lint fmt ./...

# Run a single linter only
golangci-lint run --enable-only govet ./...

# List all available linters
golangci-lint linters

# Verbose output with timing info
golangci-lint run --verbose ./...
```

## 設定

[推奨 .golangci.yml](./assets/.golangci.yml) は33個のリンターを備えた本番環境向けの設定を提供します。設定の詳細、リンターのカテゴリ、各リンターの説明については、**[リンターリファレンス](./references/linter-reference.md)** を参照してください — どのリンターが何をチェックするか（正確性、スタイル、複雑さ、パフォーマンス、セキュリティ）、33以上のリンターの説明、各リンターが役立つ場面が記載されています。

## リント警告の抑制

`//nolint` ディレクティブは控えめに使用してください — まず根本原因を修正しましょう。

```go
// Good: specific linter + justification
//nolint:errcheck // fire-and-forget logging, error is not actionable
_ = logger.Sync()

// Bad: blanket suppression without reason
//nolint
_ = logger.Sync()
```

ルール:

1. **//nolint ディレクティブにはリンター名を指定すること**: `//nolint:errcheck`（`//nolint` ではなく）
2. **//nolint ディレクティブには理由のコメントを含めること**: `//nolint:errcheck // reason`
3. **`nolintlint` リンターが上記2つのルールを強制する** — 裸の `//nolint` や理由の欠如をフラグする
4. **セキュリティリンター**（bodyclose、sqlclosecheck）を非常に強い理由なく**抑制しないこと**

包括的なパターンと例については、**[nolint ディレクティブ](./references/nolint-directives.md)** を参照してください — 抑制するタイミング、理由の書き方、行単位 vs 関数単位の抑制パターン、アンチパターンが記載されています。

## 開発ワークフロー

1. **重要な変更のたびにリンターを実行すべき**: `golangci-lint run ./...`
2. **自動修正できるものは修正する**: `golangci-lint run --fix ./...`
3. **コミット前にフォーマットする**: `golangci-lint fmt ./...`
4. **レガシーコードへの段階的導入**: `.golangci.yml` で `issues.new-from-rev` を設定し、新規/変更コードのみをリントし、古いコードを徐々にクリーンアップする

Makefile ターゲット（推奨）:

```makefile
lint:
	golangci-lint run ./...

lint-fix:
	golangci-lint run --fix ./...

fmt:
	golangci-lint fmt ./...
```

CI パイプラインの設定（GitHub Actions の `golangci-lint-action`）については、`samber/cc-skills-golang@golang-continuous-integration` スキルを参照してください。

## 出力の解釈

各問題は以下の形式に従います:

```
path/to/file.go:42:10: message describing the issue (linter-name)
```

括弧内のリンター名は、どのリンターがフラグしたかを示します。これを使って:

- [リファレンス](./references/linter-reference.md)でリンターを調べ、何をチェックしているか理解する
- 誤検知の場合は `//nolint:linter-name // reason` で抑制する
- `golangci-lint run --verbose` で追加のコンテキストとタイミング情報を得る

## よくある問題

| 問題 | 解決策 |
| --- | --- |
| "deadline exceeded" | `.golangci.yml` の `run.timeout` を増やす（デフォルト: 5m） |
| レガシーコードで問題が多すぎる | `issues.new-from-rev: HEAD~1` を設定し、新しいコードのみをリントする |
| リンターが見つからない | `golangci-lint linters` を確認 — リンターに新しいバージョンが必要かもしれない |
| リンター間の競合 | 理由を説明するコメントを付けて、有用度の低い方を無効にする |
| アップグレード後の v1 設定エラー | `golangci-lint migrate` を実行して設定フォーマットを変換する |
| 大きなリポジトリで遅い | `run.concurrency` を減らすか、`run.skip-dirs` でディレクトリを除外する |

## レガシーコードベースの並列クリーンアップ

レガシーコードベースにリンティングを導入する場合、最大5つの並列サブエージェント（Agent ツール経由）を使用して、独立したリンターカテゴリを同時に修正します:

- サブエージェント 1: `golangci-lint run --fix ./...` で自動修正可能な問題を実行
- サブエージェント 2: セキュリティリンターの検出結果を修正（bodyclose、sqlclosecheck、gosec）
- サブエージェント 3: エラーハンドリングの問題を修正（errcheck、nilerr、wrapcheck）
- サブエージェント 4: スタイルとフォーマットを修正（gofumpt、goimports、revive）
- サブエージェント 5: コード品質を修正（gocritic、unused、ineffassign）

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-continuous-integration` skill for CI pipeline with golangci-lint-action
- → See `samber/cc-skills-golang@golang-code-style` skill for style rules that linters enforce
- → See `samber/cc-skills-golang@golang-security` skill for SAST tools beyond linting (gosec, govulncheck)
