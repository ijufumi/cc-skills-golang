---
name: golang-continuous-integration
description: "Provides CI/CD pipeline configuration using GitHub Actions for Golang projects. Covers testing, linting, SAST, security scanning, code coverage, Dependabot, Renovate, GoReleaser, code review automation, and release pipelines. Use this whenever setting up CI for a Go project, configuring workflows, adding linters or security scanners, setting up Dependabot or Renovate, automating releases, or improving an existing CI pipeline. Also use when the user wants to add quality gates to their Go project."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.2"
  openclaw:
    emoji: "🚀"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - goreleaser
        - gh
    install:
      - kind: brew
        formula: goreleaser
        bins: [goreleaser]
      - kind: brew
        formula: gh
        bins: [gh]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch Bash(goreleaser:*) Bash(gh:*) AskUserQuestion
---

**Persona:** You are a Go DevOps engineer. You treat CI as a quality gate — every pipeline decision is weighed against build speed, signal reliability, and security posture.

**Modes:**

- **Setup** — adding CI to a project for the first time: start with the Quick Reference table, then generate workflows in this order: test → lint → security → release. Always check latest action versions before writing YAML.
- **Improve** — auditing or extending an existing pipeline: read current workflow files first, identify gaps against the Quick Reference table, then propose targeted additions without duplicating existing steps.

# Go 継続的インテグレーション

GitHub Actionsを使用してGoプロジェクト向けのプロダクショングレードCI/CDパイプラインを構築する。

## アクションバージョン

以下の例に示されているバージョンは参考バージョンであり、古くなっている可能性がある。ワークフローファイルを生成する前に、使用する各GitHub Action（`actions/checkout`、`actions/setup-go`、`golangci/golangci-lint-action`、`codecov/codecov-action`、`goreleaser/goreleaser-action`など）の最新の安定メジャーバージョンをインターネットで検索すること。例にハードコードされているバージョンではなく、見つけた最新バージョンを使用する。

## クイックリファレンス

| ステージ      | ツール                      | 目的                          |
| ------------- | --------------------------- | ----------------------------- |
| **テスト**    | `go test -race`             | ユニットテスト + 競合検出     |
| **カバレッジ**| `codecov/codecov-action`    | カバレッジレポート            |
| **リント**    | `golangci-lint`             | 包括的なリント                |
| **Vet**       | `go vet`                    | 組み込み静的解析              |
| **SAST**      | `gosec`, `CodeQL`, `Bearer` | セキュリティ静的解析          |
| **脆弱性スキャン** | `govulncheck`          | 既知の脆弱性検出              |
| **Docker**    | `docker/build-push-action`  | マルチプラットフォームイメージビルド |
| **依存関係**  | Dependabot / Renovate       | 自動依存関係更新              |
| **リリース**  | GoReleaser                  | 自動バイナリリリース          |

---

## テスト

`.github/workflows/test.yml` — [test.yml](./assets/test.yml)を参照

Goバージョンのマトリクスを`go.mod`に合わせて調整する:

```
go 1.23   → matrix: ["1.23", "1.24", "1.25", "1.26", "stable"]
go 1.24   → matrix: ["1.24", "1.25", "1.26", "stable"]
go 1.25   → matrix: ["1.25", "1.26", "stable"]
go 1.26   → matrix: ["1.26", "stable"]
```

1つのGoバージョンの失敗が他をキャンセルしないように`fail-fast: false`を使用する。

テストフラグ:

- `-race`: CIではデータ競合（Goにおける未定義動作）を検出するために`-race`フラグを付けてテストを実行しなければならない
- `-shuffle=on`: テスト間の依存関係を検出するためにテスト順序をランダム化する
- `-coverprofile`: カバレッジデータを生成する
- `git diff --exit-code`: `go mod tidy`で変更が発生した場合に失敗する

### カバレッジの設定

CIはコードカバレッジの閾値を強制すべきである。閾値はリポジトリルートの`codecov.yml`で設定する — [codecov.yml](./assets/codecov.yml)を参照

---

## インテグレーションテスト

`.github/workflows/integration.yml` — [integration.yml](./assets/integration.yml)を参照

テストキャッシュを無効にするために`-count=1`を使用する — キャッシュされた結果は不安定なサービスインタラクションを隠す可能性がある。

---

## リント

`golangci-lint`はすべてのPRでCIにおいて実行しなければならない。`.github/workflows/lint.yml` — [lint.yml](./assets/lint.yml)を参照

### golangci-lintの設定

プロジェクトのルートに`.golangci.yml`を作成する。推奨設定については`samber/cc-skills-golang@golang-linter`スキルを参照。

---

## セキュリティ & SAST

`.github/workflows/security.yml` — [security.yml](./assets/security.yml)を参照

CIは`govulncheck`を実行しなければならない。一般的なCVEスキャナーと異なり、プロジェクトが実際に呼び出すコードパスの脆弱性のみを報告する。CodeQLの結果はリポジトリのSecurityタブに表示される。Bearerは機密データフローの問題検出に優れている。

### CodeQLの設定

拡張セキュリティクエリスイートを使用するために`.github/codeql/codeql-config.yml`を作成する — [codeql-config.yml](./assets/codeql-config.yml)を参照

利用可能なクエリスイート:

- **default**: 標準セキュリティクエリ
- **security-extended**: やや低い精度の追加セキュリティクエリ
- **security-and-quality**: セキュリティクエリに加え、保守性と信頼性のチェック

### コンテナイメージスキャン

プロジェクトがDockerイメージを生成する場合、TrivyコンテナスキャンがDockerワークフローに含まれている — [docker.yml](./assets/docker.yml)を参照

---

## 依存関係管理

### Dependabot

`.github/dependabot.yml` — [dependabot.yml](./assets/dependabot.yml)を参照

マイナー/パッチ更新は単一のPRにグループ化される。メジャー更新は破壊的変更の可能性があるため個別のPRとなる。

#### Dependabotの自動マージ

`.github/workflows/dependabot-auto-merge.yml` — [dependabot-auto-merge.yml](./assets/dependabot-auto-merge.yml)を参照

> **Security warning:** This workflow requires `contents: write` and `pull-requests: write` — these are elevated permissions that allow merging PRs and modifying repository content. The `if: github.actor == 'dependabot[bot]'` guard restricts execution to Dependabot only. Do not remove this guard. Note that `github.actor` checks are not fully spoof-proof — **branch protection rules are the real safety net**. Ensure branch protection is configured (see [Repository Security Settings](#repository-security-settings)) with required status checks and required approvals so that auto-merge only succeeds after all checks pass, regardless of who triggered the workflow.

### Renovate（代替手段）

RenovateはDependabotのより成熟した、設定可能な代替手段である。automerge、グループ化、スケジューリング、正規表現マネージャー、モノレポ対応の更新をネイティブにサポートする。Dependabotに制限を感じる場合、Renovateが最適な選択肢である。

[Renovate GitHub App](https://github.com/apps/renovate)をインストールし、リポジトリルートに`renovate.json`を作成する — [renovate.json](./assets/renovate.json)を参照

Dependabotに対する主な利点:

- **`gomodTidy`**: 更新後に自動的に`go mod tidy`を実行する
- **ネイティブautomerge**: 別途ワークフローが不要
- **より良いグループ化**: PRグループ化のより柔軟なルール
- **正規表現マネージャー**: Dockerfile、Makefileなどのバージョンを更新可能
- **モノレポサポート**: Goワークスペースとマルチモジュールリポジトリに対応

---

## リリース自動化

GoReleaserはバイナリビルド、チェックサム、GitHub Releasesを自動化する。設定はプロジェクトの種類によって大きく異なる。

### リリースワークフロー

`.github/workflows/release.yml` — [release.yml](./assets/release.yml)を参照

> **Security warning:** This workflow requires `contents: write` to create GitHub Releases. It is restricted to tag pushes (`tags: ["v*"]`) so it cannot be triggered by pull requests or branch pushes. Only users with push access to the repository can create tags.

### CLI/プログラム向けGoReleaser

プログラムにはクロスコンパイルされたバイナリ、アーカイブ、およびオプションでDockerイメージが必要である。

`.goreleaser.yml` — [goreleaser-cli.yml](./assets/goreleaser-cli.yml)を参照

### ライブラリ向けGoReleaser

ライブラリはバイナリを生成しない — チェンジログ付きのGitHub Releaseのみが必要である。ビルドをスキップする最小限の設定を使用する。

`.goreleaser.yml` — [goreleaser-lib.yml](./assets/goreleaser-lib.yml)を参照

ライブラリの場合、GoReleaserは不要かもしれない — UIまたは`gh release create`で作成するシンプルなGitHub Releaseで十分なことが多い。

### モノレポ / マルチバイナリ向けGoReleaser

リポジトリに複数のコマンドが含まれる場合（例: `cmd/api/`、`cmd/worker/`）。

`.goreleaser.yml` — [goreleaser-monorepo.yml](./assets/goreleaser-monorepo.yml)を参照

### Docker ビルド & プッシュ

Dockerイメージを生成するプロジェクト向け。このワークフローはマルチプラットフォームイメージのビルド、SBOMとprovenance証明書の生成、GitHub Container Registry（GHCR）とDocker Hubの両方へのプッシュ、Trivyコンテナスキャンを含む。

`.github/workflows/docker.yml` — [docker.yml](./assets/docker.yml)を参照

> **Security warning:** Permissions are scoped per job: the `container-scan` job only gets `contents: read` + `security-events: write`, while the `docker` job gets `packages: write` (to push to GHCR) and `attestations: write` + `id-token: write` (for provenance/SBOM signing). This ensures the scan job cannot push images even if compromised. The `push` flag is set to `false` on pull requests so untrusted code cannot publish images. The `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets must be configured in the repository secrets settings — never hardcode credentials.

主な詳細:

- **QEMU + Buildx**: マルチプラットフォームビルド（`linux/amd64,linux/arm64`）に必須。不要なプラットフォームは削除する。
- **PRでの`push: false`**: プルリクエストではイメージはビルドされるがプッシュされない — 信頼されていないコードを公開せずにDockerfileを検証する。
- **Metadataアクション**: semverタグ（`v1.2.3` → `1.2.3`、`1.2`、`1`）、ブランチタグ（`main`）、SHAタグを自動生成する。
- **Provenance + SBOM**: `provenance: mode=max`と`sbom: true`がサプライチェーン証明書を生成する。`attestations: write`と`id-token: write`権限が必要。
- **デュアルレジストリ**: GHCR（`GITHUB_TOKEN`を使用、追加シークレット不要）とDocker Hub（`DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN`シークレットが必要）の両方にプッシュする。不要な場合はDocker Hubのログインとイメージ行を削除する。
- **Trivy**: ビルドされたイメージをCRITICALおよびHIGHの脆弱性についてスキャンし、結果をSecurityタブにアップロードする。
- イメージ名とレジストリをプロジェクトに合わせて調整する。GHCRのみの場合、Docker Hubのログインステップと`images:`の`docker.io/`行を削除する。

---

## リポジトリセキュリティ設定

ワークフローファイル作成後、開発者にGitHubリポジトリ設定（ブランチ保護、ワークフロー権限、シークレット、環境）を構成するよう必ず伝えること — [repo-security.md](./references/repo-security.md)を参照

---

## よくある間違い

| 間違い | 修正方法 |
| --- | --- |
| CIテストで`-race`がない | 常に`go test -race`を使用する |
| `-shuffle=on`がない | テスト間の依存関係を検出するためにテスト順序をランダム化する |
| インテグレーションテスト結果のキャッシュ | キャッシュを無効にするために`-count=1`を使用する |
| `go mod tidy`がチェックされていない | `go mod tidy && git diff --exit-code`ステップを追加する |
| `fail-fast: false`がない | 1つのGoバージョンの失敗が他のジョブをキャンセルすべきではない |
| アクションバージョンを固定していない | GitHub Actionsはメジャーバージョンを固定しなければならない（例: `@master`ではなく`@vN`） |
| `permissions`ブロックがない | ジョブごとに最小権限の原則に従う |
| govulncheckの検出結果を無視する | 修正するか、正当な理由を付けて抑制する |

## 関連スキル

See `samber/cc-skills-golang@golang-linter`, `samber/cc-skills-golang@golang-security`, `samber/cc-skills-golang@golang-testing`, `samber/cc-skills-golang@golang-dependency-management` skills.
