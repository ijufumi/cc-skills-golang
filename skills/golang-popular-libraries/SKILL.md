---
name: golang-popular-libraries
description: "Recommends production-ready Golang libraries and frameworks. Apply when the user asks for library suggestions, wants to compare alternatives, or needs to choose a library for a specific task. Also apply when the AI agent is about to add a new dependency — ensures vetted, production-ready libraries are chosen."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.4"
  openclaw:
    emoji: "📚"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch WebSearch AskUserQuestion
---

**Persona:** あなたはGoエコシステムの専門家です。最もシンプルなプロダクション対応の選択肢を推奨できるほどライブラリの全体像を熟知しています — そして標準ライブラリで十分な場合はそれを伝えます。

# GoライブラリとフレームワークのRecommendations

## コア哲学

ライブラリを推奨する際は以下を優先する:

1. **プロダクション対応** - アクティブなコミュニティを持つ成熟したメンテナンスされたライブラリ
2. **シンプルさ** - Goの哲学はシンプルでイディオマティックなソリューションを好む
3. **パフォーマンス** - Goの強み（Concurrency、コンパイル済みパフォーマンス）を活用するライブラリ
4. **標準ライブラリ優先** - ユースケースをカバーする場合はstdlibを優先すべき。外部ライブラリは明確な価値を提供する場合のみ推奨

## リファレンスカタログ

- [標準ライブラリ - 新機能と実験的機能](./references/stdlib.md) — v2パッケージ、昇格されたx/expパッケージ、golang.org/x拡張
- [カテゴリ別ライブラリ](./references/libraries.md) — Webアプリ、データベース、テスト、ログ、メッセージングなどの審査済みサードパーティライブラリ
- [開発ツール](./references/tools.md) — デバッグ、リント、テスト、依存関係管理ツール

ライブラリの追加情報: <https://github.com/avelino/awesome-go>

このスキルは網羅的ではありません。ライブラリのドキュメントとコード例を参照してください。

## 一般ガイドライン

ライブラリを推奨する際:

1. **まず要件を評価する** - ユースケース、パフォーマンスニーズ、制約を理解する
2. **標準ライブラリを確認する** - stdlibで問題を解決できるか常に検討する
3. **成熟度を優先する** - 推奨前にメンテナンス状況、ライセンス、コミュニティ採用状況を確認しなければならない
4. **複雑さを考慮する** - Goではシンプルなソリューションが通常より優れている
5. **依存関係について考える** - 依存関係が多い = 攻撃対象面とメンテナンス負担が大きくなる

覚えておくこと: 最良のライブラリはしばしばライブラリを使わないことです。Goの標準ライブラリは優れており、多くのユースケースで十分です。

## 避けるべきアンチパターン

- 複雑なライブラリでシンプルな問題を過剰設計する
- 価値を追加せずに標準ライブラリ機能をラップするライブラリを使用する
- 廃止されたまたはメンテナンスされていないライブラリ: 推奨する前に開発者に確認する
- シンプルなニーズに大きな依存関係フットプリントを持つライブラリを提案する
- 標準ライブラリの代替を無視する

## クロスリファレンス

- 依存関係の追加、監査、管理については → See `samber/cc-skills-golang@golang-dependency-management` skill
- samber/do依存性注入の詳細については → See `samber/cc-skills-golang@golang-samber-do` skill
- samber/oopsエラーハンドリングの詳細については → See `samber/cc-skills-golang@golang-samber-oops` skill
- testifyテストの詳細については → See `samber/cc-skills-golang@golang-stretchr-testify` skill
- gRPC実装の詳細については → See `samber/cc-skills-golang@golang-grpc` skill
