---
name: golang-security
description: "Security best practices and vulnerability prevention for Golang. Covers injection (SQL, command, XSS), cryptography, filesystem safety, network security, cookies, secrets management, memory safety, and logging. Apply when writing, reviewing, or auditing Go code for security, or when working on any risky code involving crypto, I/O, secrets management, user input handling, or authentication. Includes configuration of security tools."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "🔒"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - govulncheck
    install:
      - kind: go
        package: golang.org/x/vuln/cmd/govulncheck@latest
        bins: [govulncheck]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch Bash(govulncheck:*) WebSearch AskUserQuestion
---

**Persona:** You are a senior Go security engineer. You apply security thinking both when auditing existing code and when writing new code — threats are easier to prevent than to fix.

**Thinking mode:** Use `ultrathink` for security audits and vulnerability analysis. Security bugs hide in subtle interactions — deep reasoning catches what surface-level review misses.

**Modes:**

- **Review mode** — reviewing a PR for security issues. Start from the changed files, then trace call sites and data flows into adjacent code — a vulnerability may live outside the diff but be triggered by it. Sequential.
- **Audit mode** — full codebase security scan. Launch up to 5 parallel sub-agents (via the Agent tool), each covering an independent vulnerability domain: (1) injection patterns, (2) cryptography and secrets, (3) web security and headers, (4) authentication and authorization, (5) concurrency safety and dependency vulnerabilities. Aggregate findings, score with DREAD, and report by severity.
- **Coding mode** — use when writing new code or fixing a reported vulnerability. Follow the skill's sequential guidance. Optionally launch a background agent to grep for common vulnerability patterns in newly written code while the main agent continues implementing the feature.

# Go セキュリティ

## 概要

Goにおけるセキュリティは**多層防御**の原則に従う: 複数のレイヤーで保護し、すべての入力を検証し、安全なデフォルトを使用し、標準ライブラリのセキュリティを意識した設計を活用する。Goの型システムと並行性モデルはある程度の固有の保護を提供するが、引き続き注意が必要である。

## セキュリティ思考モデル

コードを書くまたはレビューする前に、3つの質問をする:

1. **信頼境界はどこか?** — 信頼できないデータはどこからシステムに入るか?（HTTPリクエスト、ファイルアップロード、環境変数、他のサービスが書き込んだデータベース行）
2. **攻撃者は何を制御できるか?** — どの入力が機密操作に流れるか?（SQLクエリ、シェルコマンド、HTML出力、ファイルパス、暗号操作）
3. **影響範囲はどれくらいか?** — この防御が失敗した場合、最悪の結果は何か?（データ漏洩、RCE、権限昇格、サービス拒否）

## 深刻度レベル

| レベル | DREAD | 意味 |
| --- | --- | --- |
| Critical | 8-10 | RCE、完全なデータ侵害、認証情報の窃取 — 即座に修正 |
| High | 6-7.9 | 認証バイパス、重大なデータ露出、暗号の破綻 — 現在のスプリントで修正 |
| Medium | 4-5.9 | 限定的な露出、セッション問題、防御の弱体化 — 次のスプリントで修正 |
| Low | 1-3.9 | 軽微な情報開示、ベストプラクティスからの逸脱 — 適宜修正 |

レベルは [DREAD scoring](./references/threat-modeling.md) に準拠。

## 報告前の調査

セキュリティ問題を報告する前に、コードベース全体のデータフローを追跡する — コードの断片を孤立して評価しない。

1. **データの起源を追跡する** — 変数がシステムに入る場所まで遡る。ユーザー入力か、ハードコードされた定数か、内部専用の値か?
2. **上流のバリデーションを確認する** — コールチェーンの早い段階での入力バリデーション、サニタイゼーション、型パース、許可リストを探す。
3. **信頼境界を検査する** — データが信頼境界を越えない場合（例: mTLSを使用した内部サービス間通信）、リスクプロファイルは異なる。
4. **差分だけでなく周囲のコードを読む** — ミドルウェア、インターセプター、ラッパー関数が既に防御レイヤーを提供している場合がある。

**深刻度の調整であり、却下ではない:** 上流の保護は発見を排除しない — 多層防御はすべてのレイヤーが自身を保護すべきことを意味する。しかし深刻度は変わる: 厳密な入力パーサーを通じてのみ到達可能なSQL結合はcriticalではなくmediumである。常に調整された深刻度で発見を報告し、どの上流防御が存在し、それらが除去またはバイパスされた場合に何が起こるかを記載する。

**発見をダウングレードまたはスキップする場合:** 簡潔なインラインコメントを追加する（例: `// security: SQL concat safe here — input is validated by parseUserID() which returns int`）。これにより決定が文書化され、レビュー可能になり、将来の監査で再度フラグされなくなる。

## 脅威モデリング (STRIDE)

システム内のすべての信頼境界の横断とデータフローにSTRIDEを適用する: **S**poofing（認証）、**T**ampering（整合性）、**R**epudiation（監査ログ）、**I**nformation Disclosure（暗号化）、**D**enial of Service（レート制限）、**E**levation of Privilege（認可）。DREADスコアリング（Damage、Reproducibility、Exploitability、Affected users、Discoverability）を使用して各脅威に優先順位を付け — Critical（8-10）は即座の対応が必要。

Goの例、DFD信頼境界、DREADスコアリング、OWASP Top 10マッピングを含む完全な方法論については **[Threat Modeling Guide](./references/threat-modeling.md)** を参照。

## クイックリファレンス

| 深刻度 | 脆弱性 | 防御 | 標準ライブラリの解決策 |
| --- | --- | --- | --- |
| Critical | SQLインジェクション | パラメータ化クエリでデータとコードを分離 | `database/sql` の `?` プレースホルダー |
| Critical | コマンドインジェクション | 引数を個別に渡し、シェル結合を使わない | `exec.Command` で引数を個別に指定 |
| High | XSS | 自動エスケープでユーザーデータをHTML/JSではなくテキストとして表示 | `html/template`、`text/template` |
| High | パストラバーサル | ファイルアクセスをルートに限定し、`../` エスケープを防止 | `os.Root`（Go 1.24+）、`filepath.Clean` |
| Medium | タイミング攻撃 | 定数時間比較でバイト単位のリークを防止 | `crypto/subtle.ConstantTimeCompare` |
| High | 暗号問題 | 検証済みアルゴリズムを使用し、独自暗号を作らない | `crypto/aes`、`crypto/rand` |
| Medium | HTTPセキュリティ | TLS + セキュリティヘッダーでダウングレード攻撃を防止 | `net/http`、TLSConfig設定 |
| Low | ヘッダー欠落 | HSTS、CSP、X-Frame-Optionsでブラウザ攻撃を防止 | セキュリティヘッダーミドルウェア |
| Medium | レート制限 | レート制限でブルートフォースとリソース枯渇を防止 | `golang.org/x/time/rate`、サーバータイムアウト |
| High | レースコンディション | 共有状態を保護してデータ破損を防止 | `sync.Mutex`、チャネル、共有状態の回避 |

## 詳細カテゴリ

完全な例、コードスニペット、CWEマッピングについては以下を参照:

- **[暗号](./references/cryptography.md)** — アルゴリズム、鍵導出、TLS設定。
- **[インジェクション脆弱性](./references/injection.md)** — SQL、コマンド、テンプレートインジェクション、XSS、SSRF。
- **[ファイルシステムセキュリティ](./references/filesystem.md)** — パストラバーサル、Zip爆弾、ファイルパーミッション、シンボリックリンク。
- **[ネットワーク/Webセキュリティ](./references/network.md)** — SSRF、オープンリダイレクト、HTTPヘッダー、タイミング攻撃、セッション固定。
- **[Cookieセキュリティ](./references/cookies.md)** — Secure、HttpOnly、SameSiteフラグ。
- **[サードパーティデータ漏洩](./references/third-party.md)** — アナリティクスのプライバシーリスク、GDPR/CCPAコンプライアンス。
- **[メモリ安全性](./references/memory-safety.md)** — 整数オーバーフロー、メモリエイリアシング、`unsafe` の使用。
- **[シークレット管理](./references/secrets.md)** — ハードコードされた認証情報、環境変数、シークレットマネージャー。
- **[ロギングセキュリティ](./references/logging.md)** — ログ内のPII、ログインジェクション、サニタイゼーション。
- **[脅威モデリングガイド](./references/threat-modeling.md)** — STRIDE、DREADスコアリング、信頼境界、OWASP Top 10。
- **[セキュリティアーキテクチャ](./references/architecture.md)** — 多層防御、ゼロトラスト、認証パターン、レート制限、アンチパターン。

## コードレビューチェックリスト

ドメイン別（入力処理、データベース、暗号、Web、認証、エラー、依存関係、並行性）に整理された完全なセキュリティレビューチェックリストについては **[Security Review Checklist](./references/checklist.md)** を参照 — すべての主要な脆弱性カテゴリをカバーするコードレビュー用の包括的なチェックリスト。

## ツールと検証

### 静的解析とリンティング

セキュリティ関連リンター: `bodyclose`、`sqlclosecheck`、`nilerr`、`errcheck`、`govet`、`staticcheck`。設定と使用方法は `samber/cc-skills-golang@golang-linter` スキルを参照。

より深いセキュリティ特化の解析:

```bash
# Go security checker (SAST)
go install github.com/securego/gosec/v2/cmd/gosec@latest
gosec ./...

# Vulnerability scanner — see golang-dependency-management for full govulncheck usage
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

### セキュリティテスト

```bash
# Race detector
go test -race ./...

# Fuzz testing
go test -fuzz=Fuzz
```

## よくある間違い

| 深刻度 | 間違い | 修正 |
| --- | --- | --- | --- |
| High | トークンに `math/rand` | 出力は予測可能 — 攻撃者がシーケンスを再現できる。`crypto/rand` を使用 |
| Critical | SQL文字列結合 | 攻撃者がクエリロジックを変更可能。パラメータ化クエリでデータとコードを分離 |
| Critical | `exec.Command("bash -c")` | シェルがメタ文字（`;`、` | `、`` ` ``）を解釈する。シェル解析を避けるため引数を個別に渡す |
| High | サニタイズされていない入力の信頼 | 信頼境界でバリデーションする — 内部コードは境界を信頼するため、そこで不正な入力を捕捉すればすべてを保護できる |
| Critical | ハードコードされたシークレット | ソースコード内のシークレットはバージョン履歴、CIログ、バックアップに残る。環境変数またはシークレットマネージャーを使用 |
| Medium | `==` でのシークレット比較 | `==` は最初の異なるバイトで短絡し、タイミング情報を漏洩する。`crypto/subtle.ConstantTimeCompare` を使用 |
| Medium | 詳細なエラーの返却 | スタックトレースとDBエラーは攻撃者のシステム把握を助ける。汎用メッセージを返し、詳細はサーバーサイドでログ |
| High | `-race` の発見を無視 | レースはデータ破損を引き起こし、並行状況下で認可チェックをバイパスできる。すべてのレースを修正 |
| High | パスワードに MD5/SHA1 | 両方とも既知の衝突攻撃があり、ブルートフォースが高速。Argon2id または bcrypt を使用（意図的に低速でメモリハード） |
| High | GCMなしの AES | ECB/CBCモードは認証がない — 攻撃者が暗号文を検出されずに変更可能。GCMで暗号化+認証を提供 |
| Medium | 0.0.0.0 へのバインド | すべてのネットワークインターフェースにサービスを公開する。攻撃対象領域を制限するために特定のインターフェースにバインド |

## セキュリティアンチパターン

| 深刻度 | アンチパターン | なぜ失敗するか | 修正 |
| --- | --- | --- | --- |
| High | 隠蔽によるセキュリティ | 隠しURLはファジング、ログ、ソースで発見可能 | すべてのエンドポイントに認証+認可 |
| High | クライアントヘッダーの信頼 | `X-Forwarded-For`、`X-Is-Admin` は容易に偽造可能 | サーバーサイドのID検証 |
| High | クライアントサイド認可 | JavaScriptチェックはどのHTTPクライアントでもバイパス可能 | すべてのハンドラにサーバーサイドの権限チェック |
| High | 環境間でのシークレット共有 | ステージングの侵害が本番を危険にさらす | シークレットマネージャーによる環境別シークレット |
| Critical | 暗号エラーの無視 | `_, _ = encrypt(data)` が暗号化なしで黙って続行 | 常にエラーをチェック — 閉じた状態で失敗し、開いた状態にしない |
| Critical | 独自暗号の作成 | カスタム暗号は暗号学者に分析されていない | `crypto/aes` GCM、`golang.org/x/crypto/argon2` を使用 |

Goのコード例を含む詳細なアンチパターンについては **[Security Architecture](./references/architecture.md)** を参照。

## クロスリファレンス

See `samber/cc-skills-golang@golang-database`, `samber/cc-skills-golang@golang-safety`, `samber/cc-skills-golang@golang-observability`, `samber/cc-skills-golang@golang-continuous-integration` skills.

## 追加リソース

- [Go Security Best Practices](https://go.dev/doc/security/best-practices)
- [gosec Security Linter](https://github.com/securego/gosec)
- [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck)
- [OWASP Go Secure Coding Practices](https://owasp.org/www-project-go-secure-coding-practices-guide/)
