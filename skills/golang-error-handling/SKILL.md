---
name: golang-error-handling
description: "Idiomatic Golang error handling — creation, wrapping with %w, errors.Is/As, errors.Join, custom error types, sentinel errors, panic/recover, the single handling rule, structured logging with slog, HTTP request logging middleware, and samber/oops for production errors. Built to make logs usable at scale with log aggregation 3rd-party tools. Apply when creating, wrapping, inspecting, or logging errors in Go code."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.1"
  openclaw:
    emoji: "⚠️"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

**Persona:** You are a Go reliability engineer. You treat every error as an event that must either be handled or propagated with context — silent failures and duplicate logs are equally unacceptable.

**Modes:**

- **Coding mode** — writing new error handling code. Follow the best practices sequentially; optionally launch a background sub-agent to grep for violations in adjacent code (swallowed errors, log-and-return pairs) without blocking the main implementation.
- **Review mode** — reviewing a PR's error handling changes. Focus on the diff: check for swallowed errors, missing wrapping context, log-and-return pairs, and panic misuse. Sequential.
- **Audit mode** — auditing existing error handling across a codebase. Use up to 5 parallel sub-agents, each targeting an independent category (creation, wrapping, single-handling rule, panic/recover, structured logging).

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-error-handling` skill takes precedence.

# Go エラーハンドリングのベストプラクティス

このスキルはGoアプリケーションにおける堅牢で慣用的なエラーハンドリングの作成をガイドする。保守性が高く、デバッグ可能で、本番品質のエラーコードを書くために以下の原則に従う。

## ベストプラクティスまとめ

1. **返されたエラーはMUSTで常にチェックする** — `_` で破棄しない
2. **エラーはMUSTでコンテキスト付きでラップする** `fmt.Errorf("{context}: %w", err)` を使用
3. **エラー文字列はMUSTで小文字**、末尾の句読点なし
4. **内部では `%w`、システム境界では `%v` を使用**してエラーチェーンの露出を制御
5. **直接比較や型アサーションの代わりにMUSTで `errors.Is` と `errors.As` を使用**
6. **独立したエラーの結合にはSHOULDで `errors.Join`** を使用（Go 1.20+）
7. **エラーはMUSTでログ出力または返却のいずれか一方のみ**、両方は行わない（単一処理ルール）
8. **期待される条件にはセンチネルエラー**、データを持つ場合はカスタム型を使用
9. **期待されるエラー条件にはpanicを使用しない** — 真に回復不可能な状態のために予約
10. **構造化エラーログにはSHOULDで `slog`** を使用（Go 1.21+）— `fmt.Println` や `log.Printf` ではなく
11. **スタックトレース、ユーザー/テナントコンテキスト、または構造化属性が必要な本番エラーには `samber/oops` を使用**
12. **HTTPリクエストをログ**する — メソッド、パス、ステータス、所要時間をキャプチャする構造化ミドルウェアで
13. **ログレベルを使用**してエラーの深刻度を示す
14. **技術的なエラーをユーザーに公開しない** — 内部エラーをユーザーフレンドリーなメッセージに変換し、技術的な詳細は別途ログする
15. **エラーメッセージは低カーディナリティに保つ** — 可変データ（ID、パス、行番号）をエラー文字列に埋め込まない。代わりに構造化属性として付加する（ログサイトでは `slog` 経由、エラー自体には `samber/oops` の `.With()` 経由）。これによりAPM/ログアグリゲーター（Datadog、Loki、Sentry）がエラーを適切にグループ化できる

## 詳細リファレンス

- **[エラーの作成](./references/error-creation.md)** — ストーリーを伝えるエラーの作り方: エラーメッセージは小文字で句読点なし、何が起きたかを記述するがアクションを規定しない。センチネルエラー（パフォーマンスのための一度限りの事前割り当て）、カスタムエラー型（リッチなコンテキストを運ぶため）、どちらをいつ使うかの判断表をカバー。

- **[エラーラッピングと検査](./references/error-wrapping.md)** — なぜ `fmt.Errorf("{context}: %w", err)` が `fmt.Errorf("{context}: %v", err)` より優れているか（チェーンと結合の違い）。型安全なエラーハンドリングのための `errors.Is`/`errors.As` によるチェーン検査と、独立したエラーの結合のための `errors.Join`。

- **[エラーハンドリングパターンとロギング](./references/error-handling.md)** — 単一処理ルール: エラーはログ出力または返却のいずれか一方のみ、両方は行わない（アグリゲーターを散らかす重複ログを防止）。panic/recover設計、本番エラー用の `samber/oops`、APMツール用の `slog` 構造化ログ統合。

## エラーハンドリング監査の並列化

大規模コードベースでエラーハンドリングを監査する場合、最大5つの並列サブエージェント（Agentツール経由）を使用する — 各サブエージェントは独立したエラーカテゴリを担当:

- サブエージェント1: エラーの作成 — `errors.New`/`fmt.Errorf` の使用、低カーディナリティメッセージ、カスタム型を検証
- サブエージェント2: エラーラッピング — `%w` と `%v` を監査、`errors.Is`/`errors.As` パターンを検証
- サブエージェント3: 単一処理ルール — ログ出力と返却の同時実行、握りつぶされたエラー、破棄されたエラー（`_`）を検出
- サブエージェント4: panic/recover — `panic` の使用を監査、ゴルーチン境界でのリカバリを検証
- サブエージェント5: 構造化ログ — エラーサイトでの `slog` 使用を検証、エラーメッセージ内のPIIをチェック

## クロスリファレンス

- → See `samber/cc-skills-golang@golang-samber-oops` for full samber/oops API, builder patterns, and logger integration
- → See `samber/cc-skills-golang@golang-observability` for structured logging setup, log levels, and request logging middleware
- → See `samber/cc-skills-golang@golang-safety` for nil interface trap and nil error comparison pitfalls
- → See `samber/cc-skills-golang@golang-naming` for error naming conventions (ErrNotFound, PathError)

## リファレンス

- [lmittmann/tint](https://github.com/lmittmann/tint)
- [samber/oops](https://github.com/samber/oops)
- [samber/slog-multi](https://github.com/samber/slog-multi)
- [samber/slog-sampling](https://github.com/samber/slog-sampling)
- [samber/slog-formatter](https://github.com/samber/slog-formatter)
- [samber/slog-http](https://github.com/samber/slog-http)
- [samber/slog-sentry](https://github.com/samber/slog-sentry)
- [log/slog package](https://pkg.go.dev/log/slog)
