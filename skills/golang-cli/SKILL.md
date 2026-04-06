---
name: golang-cli
description: "Golang CLI application development. Use when building, modifying, or reviewing a Go CLI tool — especially for command structure, flag handling, configuration layering, version embedding, exit codes, I/O patterns, signal handling, shell completion, argument validation, and CLI unit testing. Also triggers when code uses cobra, viper, or urfave/cli."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "💻"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
    install: []
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a Go CLI engineer. You build tools that feel native to the Unix shell — composable, scriptable, and predictable under automation.

**Modes:**

- **Build** — creating a new CLI from scratch: follow the project structure, root command setup, flag binding, and version embedding sections sequentially.
- **Extend** — adding subcommands, flags, or completions to an existing CLI: read the current command tree first, then apply changes consistent with the existing structure.
- **Review** — auditing an existing CLI for correctness: check the Common Mistakes table, verify `SilenceUsage`/`SilenceErrors`, flag-to-Viper binding, exit codes, and stdout/stderr discipline.

# Go CLI ベストプラクティス

Go CLIアプリケーションのデフォルトスタックとしてCobra + Viperを使用する。Cobraはコマンド/サブコマンド/フラグの構造を提供し、Viperはファイル、環境変数、フラグからの設定を自動レイヤリングで処理する。この組み合わせはkubectl、docker、gh、hugoなど、ほとんどのプロダクションGo CLIで使われている。

CobraまたはViperを使用する際は、最新のAPIシグネチャについてライブラリの公式ドキュメントとコード例を参照すること。

サブコマンドがなくフラグも少ない単純なツールの場合、標準ライブラリの`flag`で十分である。

## クイックリファレンス

| 項目                | パッケージ / ツール                  |
| ------------------- | ------------------------------------ |
| コマンドとフラグ    | `github.com/spf13/cobra`             |
| 設定管理            | `github.com/spf13/viper`             |
| フラグ解析          | `github.com/spf13/pflag` (via Cobra) |
| カラー出力          | `github.com/fatih/color`             |
| テーブル出力        | `github.com/olekukonko/tablewriter`  |
| 対話型プロンプト    | `github.com/charmbracelet/bubbletea` |
| バージョン注入      | `go build -ldflags`                  |
| 配布                | `goreleaser`                         |

## プロジェクト構成

CLIコマンドは`cmd/myapp/`に配置し、コマンドごとに1ファイルとする。`main.go`は最小限にし、`Execute()`を呼び出すだけにする。

```
myapp/
├── cmd/
│   └── myapp/
│       ├── main.go              # package main, only calls Execute()
│       ├── root.go              # Root command + Viper init
│       ├── serve.go             # "serve" subcommand
│       ├── migrate.go           # "migrate" subcommand
│       └── version.go           # "version" subcommand
├── go.mod
└── go.sum
```

`main.go`は最小限にすべき — [assets/examples/main.go](assets/examples/main.go)を参照。

## ルートコマンドの設定

ルートコマンドはViperの設定を初期化し、`PersistentPreRunE`を通じてグローバルな動作を設定する。[assets/examples/root.go](assets/examples/root.go)を参照。

重要なポイント:

- `SilenceUsage: true`は必ず設定すること — エラーのたびに完全な使用方法テキストが出力されるのを防ぐ
- `SilenceErrors: true`は必ず設定すること — エラー出力のフォーマットを自分で制御できるようにする
- `PersistentPreRunE`はすべてのサブコマンドの前に実行されるため、設定は常に初期化される
- ログはstderrへ、出力はstdoutへ

## サブコマンド

`cmd/myapp/`にファイルを分けてサブコマンドを追加し、`init()`で登録する。コマンドグループを含む完全なサブコマンドの例は[assets/examples/serve.go](assets/examples/serve.go)を参照。

## フラグ

すべてのフラグパターンについては[assets/examples/flags.go](assets/examples/flags.go)を参照:

### 永続フラグ vs ローカルフラグ

- **永続(Persistent)**フラグはすべてのサブコマンドに継承される（例: `--config`）
- **ローカル(Local)**フラグは定義されたコマンドにのみ適用される（例: `--port`）

### 必須フラグ

フラグの制約には`MarkFlagRequired`、`MarkFlagsMutuallyExclusive`、`MarkFlagsOneRequired`を使用する。

### RegisterFlagCompletionFuncによるフラグ検証

フラグの値に対する補完候補を提供する。

### フラグは必ずViperにバインドする

これにより`viper.GetInt("port")`がフラグの値、環境変数`MYAPP_PORT`、または設定ファイルの値のうち、最も優先度の高いものを返すようになる。

## 引数の検証

Cobraは位置引数のバリデータを組み込みで提供している。組み込みおよびカスタム検証の例は[assets/examples/args.go](assets/examples/args.go)を参照。

| バリデータ                  | 説明                                 |
| --------------------------- | ------------------------------------ |
| `cobra.NoArgs`              | 引数が提供されると失敗               |
| `cobra.ExactArgs(n)`        | 正確にn個の引数が必要                |
| `cobra.MinimumNArgs(n)`     | 最低n個の引数が必要                  |
| `cobra.MaximumNArgs(n)`     | 最大n個の引数まで許可               |
| `cobra.RangeArgs(min, max)` | minからmaxの間の引数が必要           |
| `cobra.ExactValidArgs(n)`   | 正確にn個の引数、ValidArgsに含まれる必要あり |

## Viperによる設定管理

Viperは以下の順序で設定値を解決する（優先度の高い順）:

1. **CLIフラグ**（明示的なユーザー入力）
2. **環境変数**（デプロイ設定）
3. **設定ファイル**（永続的な設定）
4. **デフォルト値**（コードで設定）

構造体へのアンマーシャリングや設定ファイルの監視を含む完全なViper統合については[assets/examples/config.go](assets/examples/config.go)を参照。

### 設定ファイルの例 (.myapp.yaml)

```yaml
port: 8080
host: localhost
log-level: info
database:
  dsn: postgres://localhost:5432/myapp
  max-conn: 25
```

上記の設定では、以下はすべて同等:

- フラグ: `--port 9090`
- 環境変数: `MYAPP_PORT=9090`
- 設定ファイル: `port: 9090`

## バージョンとビルド情報

バージョンは`ldflags`を使用してコンパイル時に埋め込むべきである。バージョンコマンドとビルド手順については[assets/examples/version.go](assets/examples/version.go)を参照。

## 終了コード

終了コードはUnixの慣例に従わなければならない:

| コード | 意味              | 使用タイミング                            |
| ------ | ----------------- | ----------------------------------------- |
| 0      | 成功              | 操作が正常に完了                          |
| 1      | 一般エラー        | 実行時の失敗                              |
| 2      | 使用法エラー      | 無効なフラグまたは引数                    |
| 64-78  | BSD sysexits      | 特定のエラーカテゴリ                      |
| 126    | 実行不可          | 権限拒否                                  |
| 127    | コマンド未検出    | 依存関係の欠如                            |
| 128+N  | シグナル N        | シグナルによる終了（例: 130 = SIGINT）    |

エラーを終了コードにマッピングするパターンについては[assets/examples/exit_codes.go](assets/examples/exit_codes.go)を参照。

## I/Oパターン

すべてのI/Oパターンについては[assets/examples/output.go](assets/examples/output.go)を参照:

- **stdout vs stderr**: 診断出力をstdoutに書き込んではならない — stdoutはプログラム出力用（パイプ可能）、stderrはログ/エラー/診断用
- **パイプ vs ターミナルの検出**: stdoutの`os.ModeCharDevice`をチェック
- **機械可読出力**: table/json/plain形式の`--output`フラグをサポート
- **カラー**: 出力がターミナルでない場合に自動的に無効化される`fatih/color`を使用

## シグナル処理

シグナル処理はcontextを通じてキャンセルを伝播するために`signal.NotifyContext`を使用しなければならない。グレースフルなHTTPサーバーシャットダウンについては[assets/examples/signal.go](assets/examples/signal.go)を参照。

## シェル補完

Cobraはbash、zsh、fish、PowerShell用の補完を自動的に生成する。補完コマンドとカスタムフラグ/引数補完については[assets/examples/completion.go](assets/examples/completion.go)を参照。

## CLIコマンドのテスト

コマンドをプログラム的に実行し、出力をキャプチャしてテストする。[assets/examples/cli_test.go](assets/examples/cli_test.go)を参照。

テストで出力をキャプチャできるように、コマンドでは（`os.Stdout` / `os.Stderr`の代わりに）`cmd.OutOrStdout()`と`cmd.ErrOrStderr()`を使用する。

## よくある間違い

| 間違い | 修正方法 |
| --- | --- |
| `os.Stdout`に直接書き込む | テストが出力をキャプチャできない。テストでバッファにリダイレクトできる`cmd.OutOrStdout()`を使用する |
| `RunE`内で`os.Exit()`を呼び出す | Cobraのエラー処理、defer関数、クリーンアップコードが実行されない。エラーを返し、`main()`に判断させる |
| フラグをViperにバインドしていない | フラグが環境変数/設定ファイルで設定できなくなる。設定可能なすべてのフラグに`viper.BindPFlag`を呼び出す |
| `viper.SetEnvPrefix`がない | `PORT`が他のツールと衝突する。環境変数の名前空間化のためにプレフィックスを使用する（`MYAPP_PORT`） |
| stdoutにログを出力する | Unixパイプはstdoutをチェーンする — ログが次のプログラムへのデータストリームを破壊する。ログはstderrへ |
| エラーのたびに使用法を表示する | エラーのたびにフルヘルプテキストはノイズ。`SilenceUsage: true`を設定し、`--help`用にフル使用法を温存する |
| 設定ファイルを必須にする | 設定ファイルがないユーザーがクラッシュする。`viper.ConfigFileNotFoundError`を無視する — 設定は任意であるべき |
| `PersistentPreRunE`を使っていない | 設定の初期化はすべてのサブコマンドの前に行う必要がある。ルートの`PersistentPreRunE`を使用する |
| バージョン文字列のハードコード | バージョンがタグと同期しなくなる。ビルド時にgitタグから`ldflags`で注入する |
| `--output`フォーマットをサポートしていない | スクリプトが人間可読出力を解析できない。機械消費用にJSON/table/plainを追加する |

## 関連スキル

See `samber/cc-skills-golang@golang-project-layout`, `samber/cc-skills-golang@golang-dependency-injection`, `samber/cc-skills-golang@golang-testing`, `samber/cc-skills-golang@golang-design-patterns` skills.
