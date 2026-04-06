# 本番環境向けGolangプロジェクトのためのAgent Skills

AIエージェントスキルは、コーディングアシスタントをドメイン固有の専門知識で拡張する再利用可能な命令セットです。必要に応じて読み込まれるため、コンテキストを圧迫しません。このリポジトリは**Go固有**のスキル（言語、テスト、セキュリティ、オブザーバビリティなど）のみを対象としています。開発ワークフロースキル（gitの規約、CI/CD、PRレビュー）については、別のスキルプラグインを追加してください。

汎用スキルについては [cc-skills](https://github.com/samber/cc-skills) をご覧ください。

> [!IMPORTANT]
> Claude Codeを使い、私のGoプロジェクトのコミットを蒸留してブートストラップしました。**人間が編集・テスト・レビュー・リワークしています**。
>
> **AIのスロップ（低品質な出力）はありません。** AI製のスキルは役に立ちません。

<img width="1414" height="491" alt="image" src="https://github.com/user-attachments/assets/620b5835-c1ba-4ea9-bf47-2293b58b879e" />

## 🚀 使い方

**[skills](https://skills.sh/) CLIでインストール**（ユニバーサル、[Agent Skills](https://agentskills.io)互換ツールで動作）:

```bash
npx skills add https://github.com/samber/cc-skills-golang --all
# または単一のスキル:
npx skills add https://github.com/samber/cc-skills-golang --skill golang-performance
```

<!-- prettier-ignore-start -->

<details>
<summary>Claude Code</summary>

```bash
/plugin marketplace add samber/cc
/plugin install cc-skills-golang@samber
```

</details>

<details>
<summary>Openclaw</summary>

スキルをクロスクライアント検出ディレクトリにコピー:

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.openclaw/skills/cc-skills-golang
# またはワークスペースに:
git clone https://github.com/samber/cc-skills-golang.git ~/.openclaw/workspace/skills/cc-skills-golang
```

</details>

<details>
<summary>Gemini CLI</summary>

```bash
gemini extensions install https://github.com/samber/cc-skills-golang
```

`gemini extensions update cc-skills-golang` で更新できます。

</details>

<details>
<summary>Cursor</summary>

スキルをクロスクライアント検出ディレクトリにコピー:

```bash
git clone https://github.com/samber/cc-skills-golang.git  ~/.cursor/skills/cc-skills-golang
```

Cursorは `.agents/skills/` と `.cursor/skills/` からスキルを自動検出します。

</details>

<details>
<summary>Copilot</summary>

スキルをクロスクライアント検出ディレクトリにコピー:

```bash
/plugin install https://github.com/samber/cc-skills-golang
# または
git clone https://github.com/samber/cc-skills-golang.git ~/.copilot/skills/cc-skills-golang
```

Copilotは `.copilot/skills/` からスキルを自動検出します。

</details>

<details>
<summary>OpenCode</summary>

スキルをクロスクライアント検出ディレクトリにコピー:

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.agents/skills/cc-skills-golang
```

OpenCodeは `.agents/skills/`、`.opencode/skills/`、`.claude/skills/` からスキルを自動検出します。

</details>

<details>
<summary>Codex (OpenAI)</summary>

クロスクライアント検出パスにクローン:

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.agents/skills/cc-skills-golang
```

Codexは `~/.agents/skills/` と `.agents/skills/` からスキルを自動検出します。`cd ~/.agents/skills/cc-skills-golang && git pull` で更新できます。

</details>

<details>
<summary>Antigravity</summary>

クロスクライアント検出パスにクローンしてシンボリックリンクを作成:

```bash
git clone https://github.com/samber/cc-skills-golang.git ~/.antigravity/skills/cc-skills-golang
```

`cd ~/.antigravity/skills/cc-skills-golang && git pull` で更新できます。

</details>

<!-- prettier-ignore-end -->

## 🧩 スキル一覧

これらのスキルは**アトミックで相互参照する単位**として設計されています。あるスキルが別のスキルで定義された規約を参照することがあります（例：ロギングに影響するエラーハンドリングルールは `golang-observability` ではなく `golang-error-handling` に存在します）。一部のスキルのみをインストールすると、ガイドラインの部分的で一貫性のないビューになる可能性があります。最良の結果を得るには、汎用スキルをすべてまとめてインストールしてください。

- ⭐️ 推奨
- ✅ 公開済み
- 👷 作業中
- ❌ 未着手
- ⚡ コマンド利用可能
- 🧠 自動でUltrathink
- ⚙️ オーバーライド可能（下記ドキュメント参照）
- **Description (tok)**: YAMLフロントマターの `description` フィールドの重み。スキルトリガーのためにClaudeのコンテキストに常に読み込まれます
- **SKILL.md (tok)**: スキルがトリガーされた時に読み込まれる完全な `SKILL.md` ファイルの重み
- **Directory (tok)**: スキルディレクトリ内の全ファイルの重み（SKILL.md + 参照されるマークダウンファイル）

**汎用:**

<!-- markdownlint-disable table-column-style -->

|  | Skill | Flags | Error rate gap | Description (tok) | SKILL.md (tok) | Directory (tok) |
| --- | --- | --- | --- | --- | --- | --- |
| ⭐️ | ✅ `golang-code-style` | ⚙️ | -40% | 31 | 2,069 | 2,685 |
| ⭐️ | ✅ `golang-data-structures` |  | -39% | 92 | 2,464 | 6,176 |
| ⭐️ | ✅ `golang-database` | ⚙️ | -38% | 112 | 2,725 | 7,248 |
| ⭐️ | ✅ `golang-design-patterns` | ⚙️ | -37% | 66 | 2,610 | 9,316 |
| ⭐️ | ✅ `golang-documentation` | ⚡ ⚙️ | -53% | 73 | 2,678 | 10,549 |
| ⭐️ | ✅ `golang-error-handling` | ⚙️ | -26% | 90 | 1,520 | 4,394 |
| ⭐️ | 👷 `golang-how-to` |  | — | 0 | 0 | 0 |
| ⭐️ | ✅ `golang-modernize` | ⚡ | -61% | 113 | 2,476 | 7,599 |
| ⭐️ | ✅ `golang-naming` | ⚙️ | -23% | 158 | 2,865 | 7,233 |
| ⭐️ | ✅ `golang-safety` |  | -58% | 85 | 2,457 | 5,227 |
| ⭐️ | ✅ `golang-testing` | ⚡ 🧠 ⚙️ | -32% | 98 | 3,105 | 6,212 |
| ⭐️ | ✅ `golang-troubleshooting` | ⚡ 🧠 | -32% | 106 | 2,735 | 15,901 |
| ⭐️ | ✅ `golang-security` | ⚡ 🧠 | -32% | 84 | 2,873 | 20,894 |
|  | ✅ `golang-benchmark` | ⚡ 🧠 | -50% | 92 | 2,135 | 29,248 |
|  | ✅ `golang-cli` |  | -43% | 73 | 2,274 | 6,089 |
|  | ✅ `golang-concurrency` | ⚙️ | -39% | 71 | 1,873 | 6,338 |
|  | ✅ `golang-context` | ⚙️ | -34% | 41 | 1,144 | 3,940 |
|  | ✅ `golang-continuous-integration` | ⚡ | -59% | 105 | 2,835 | 6,477 |
|  | ✅ `golang-dependency-injection` | ⚙️ | -47% | 104 | 2,842 | 5,113 |
|  | ✅ `golang-dependency-management` |  | -54% | 94 | 1,877 | 4,957 |
|  | ✅ `golang-structs-interfaces` | ⚙️ | -35% | 110 | 2,999 | 2,999 |
|  | ✅ `golang-linter` |  | -41% | 119 | 1,714 | 5,493 |
|  | ✅ `golang-observability` | ⚡ ⚙️ | -37% | 144 | 2,921 | 18,453 |
|  | ✅ `golang-performance` | ⚡ 🧠 | -39% | 108 | 1,953 | 17,855 |
|  | ✅ `golang-popular-libraries` |  | -30% | 61 | 788 | 4,131 |
|  | ✅ `golang-project-layout` | ⚡ | -38% | 66 | 1,510 | 5,718 |
|  | ✅ `golang-stay-updated` |  | -56% | 43 | 1,916 | 1,916 |

**ツール:**

| Skill | Flags | Error rate gap | Description (tok) | SKILL.md (tok) | Directory (tok) |
| --- | --- | --- | --- | --- | --- |
| ❌ `golang-google-wire` |  | — | 0 | 0 | 0 |
| ❌ `golang-graphql` |  | — | 0 | 0 | 0 |
| ✅ `golang-grpc` |  | -41% | 69 | 2,149 | 4,965 |
| ❌ `golang-spf13-cobra` |  | — | 0 | 0 | 0 |
| ❌ `golang-spf13-viper` |  | — | 0 | 0 | 0 |
| ❌ `golang-swagger` |  | — | 0 | 0 | 0 |
| ❌ `golang-uber-dig` |  | — | 0 | 0 | 0 |
| ❌ `golang-uber-fx` |  | — | 0 | 0 | 0 |
| ✅ `golang-samber-do` |  | -81% | 70 | 1,746 | 3,269 |
| ✅ `golang-samber-hot` |  | -54% | 118 | 1,843 | 7,273 |
| ✅ `golang-samber-lo` |  | -40% | 155 | 2,410 | 10,031 |
| ✅ `golang-samber-mo` | 🧠 | -48% | 81 | 2,800 | 11,215 |
| ✅ `golang-samber-oops` |  | -59% | 69 | 2,380 | 2,692 |
| ✅ `golang-samber-ro` | 🧠 | -50% | 140 | 2,845 | 11,136 |
| ✅ `golang-samber-slog` |  | -19% | 118 | 2,588 | 9,234 |
| ❌ `golang-temporal` |  | — | 0 | 0 | 0 |
| ✅ `golang-stretchr-testify` |  | -47% | 89 | 1,714 | 2,533 |

## 🧪 スキル評価

|             | スキルあり          | スキルなし          | 差分      |
| ----------- | ------------------- | ------------------- | --------- |
| **全体**    | **3065/3141 (98%)** | **1691/3141 (54%)** | **+44pp** |

スキルごとの詳細な内訳は [EVALUATIONS.md](./EVALUATIONS.md) をご覧ください。

## 🎯 スキルトリガーの調整

スキルのトリガーが多すぎる、または少なすぎる場合は、descriptionの変更を提案する [issue を作成](https://github.com/samber/cc-skills-golang/issues) してください。SKILL.mdフロントマターの `description` フィールドがトリガーの主要メカニズムです。わずかな文言の調整でトリガーの精度を大幅に改善できます。一部の `SKILL.md` には `When to use` セクションがあり、これも除外の基準になります。最後に、`SKILL.md` は `references/` にある詳細な知識を遅延読み込みするためのエントリーポイントです。

## 🔄 重複について

相互参照のおかげで、このリポジトリのスキル間の重複はほとんどありません。ほとんどのスキルを有効にし、遅延読み込みを活用することをお勧めします。推奨⭐️スキルは起動時に約1,100トークンのdescriptionを読み込みます。スキルの完全な内容は、関連する場合にのみ取得されます。注意:

- `golang-naming` と `golang-code-style` の約50%はリンター（golangci-lint）と重複していると推定しています。
- `golang-security` のセキュリティルールの大部分はBearer（SAST）のチェックリストから抽出されたものです。方法論としてはスキルは依然として有用です。
- チーム独自の規約がある場合は、カンパニースキルを作成し、そのボディの先頭付近でオーバーライドを明示的に宣言してください: `This skill supersedes samber/cc-skills-golang@golang-naming skill for [company] projects.` 上のテーブルで⚙️マークが付いたスキルがこのメカニズムをサポートしています。

## ✍️ コントリビュート

- **スキルdescriptionごとに100トークン** - 何を？いつこのスキルを使う？
- **SKILL.mdごとに1,000〜2,500トークン** — メインファイルは要点に絞る
- **詳細にはセカンダリマークダウンファイルを使用** — SKILL.mdから相対リンクで参照（例：`[Logging](./logging.md)`）。Claudeはトピックが関連する場合にのみこれらを読み込むため、必要になるまでコンテキスト予算にカウントされません
- **スキル全体とセカンダリファイルで最大10,000トークン**
- **2〜4スキルが同時に読み込まれる**のが一般的なセッション — スキルは共存できるように設計
- **読み込まれるSKILL.mdの合計は常に約10kトークン以下に** — レスポンス品質の低下を防ぐため

詳細なガイドラインは `CLAUDE.md` をご確認ください。

## 💫 応援してください

- ⭐️ **このリポジトリにスターを** - あなたのスターがカフェインエンジンの燃料になります！
- ☕️ **コーヒーをおごってください** - 実際にコーヒーを飲みながらさらにスキルを作ります

[![GitHub Sponsors](https://img.shields.io/github/sponsors/samber?style=for-the-badge)](https://github.com/sponsors/samber)

## 📝 ライセンス

Copyright © 2026 [Samuel Berthe](https://github.com/samber).

このプロジェクトは [MIT](./LICENSE) ライセンスの下で公開されています。
