---
name: golang-grpc
description: "Provides gRPC usage guidelines, protobuf organization, and production-ready patterns for Golang microservices. Use when implementing, reviewing, or debugging gRPC servers/clients, writing proto files, setting up interceptors, handling gRPC errors with status codes, configuring TLS/mTLS, testing with bufconn, or working with streaming RPCs."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.1.3"
  openclaw:
    emoji: "🌐"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - protoc
    install:
      - kind: brew
        formula: protobuf
        bins: [protoc]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent WebFetch mcp__context7__resolve-library-id mcp__context7__query-docs Bash(protoc:*) AskUserQuestion
---

**Persona:** あなたはGo分散システムエンジニアです。正確性と運用性を考慮してgRPCサービスを設計します — 適切なステータスコード、デッドライン、インターセプター、グレースフルシャットダウンはハッピーパスと同じくらい重要です。

**モード:**

- **ビルドモード** — 新しいgRPCサーバーまたはクライアントをゼロから実装する。
- **レビューモード** — 既存のgRPCコードの正確性、セキュリティ、運用上の問題を監査する。

# Go gRPC ベストプラクティス

gRPCを純粋なトランスポート層として扱う — ビジネスロジックと分離する。公式のGo実装は `google.golang.org/grpc`。

このスキルは網羅的ではありません。最新のAPIシグネチャと使用パターンについては、ライブラリの公式ドキュメントとコード例を参照してください。Context7をディスカバリープラットフォームとして活用できます。

## クイックリファレンス

| 関心事 | パッケージ / ツール |
| --- | --- |
| サービス定義 | `.proto` ファイルによる `protoc` または `buf` |
| コード生成 | `protoc-gen-go`、`protoc-gen-go-grpc` |
| エラーハンドリング | `codes` と共に `google.golang.org/grpc/status` |
| リッチエラー詳細 | `google.golang.org/genproto/googleapis/rpc/errdetails` |
| インターセプター | `grpc.ChainUnaryInterceptor`、`grpc.ChainStreamInterceptor` |
| ミドルウェアエコシステム | `github.com/grpc-ecosystem/go-grpc-middleware` |
| テスト | `google.golang.org/grpc/test/bufconn` |
| TLS / mTLS | `google.golang.org/grpc/credentials` |
| ヘルスチェック | `google.golang.org/grpc/health` |

## Protoファイルの整理

バージョン付きディレクトリ（`proto/user/v1/`）でドメイン別に整理する。`Request`/`Response` ラッパーメッセージを常に使用する — `string` などのベア型は後でフィールドを追加できない。`buf generate` または `protoc` で生成する。

[Proto & コード生成リファレンス](references/protoc-reference.md)

## サーバー実装

- ヘルスチェックサービス（`grpc_health_v1`）を実装する — Kubernetesプローブが準備完了を判定するために必要
- ロギング、認証、リカバリーなどの横断的関心事にインターセプターを使用する — ビジネスロジックをクリーンに保つ
- タイムアウトフォールバック付きの `GracefulStop()` を使用する — 進行中のRPCをドレインしながらハングを防ぐ
- プロダクションではリフレクションを無効にする — API全体が公開されてしまう

```go
srv := grpc.NewServer(
    grpc.ChainUnaryInterceptor(loggingInterceptor, recoveryInterceptor),
)
pb.RegisterUserServiceServer(srv, svc)
healthpb.RegisterHealthServer(srv, health.NewServer())

go srv.Serve(lis)

// On shutdown signal:
stopped := make(chan struct{})
go func() { srv.GracefulStop(); close(stopped) }()
select {
case <-stopped:
case <-time.After(15 * time.Second):
    srv.Stop()
}
```

### インターセプターパターン

```go
func loggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s duration=%s code=%s", info.FullMethod, time.Since(start), status.Code(err))
    return resp, err
}
```

## クライアント実装

- コネクションを再利用する — gRPCは単一のHTTP/2コネクション上でRPCを多重化する。リクエストごとに作成するとTCP/TLSハンドシェイクが無駄になる
- すべての呼び出しにデッドラインを設定する（`context.WithTimeout`） — これがないと遅い上流が無期限にゴルーチンをハングさせる
- `dns:///` スキーム経由でヘッドレスKubernetesサービスと共に `round_robin` を使用する
- `metadata.NewOutgoingContext` 経由でメタデータ（認証トークン、トレースID）を渡す

```go
conn, err := grpc.NewClient("dns:///user-service:50051",
    grpc.WithTransportCredentials(creds),
    grpc.WithDefaultServiceConfig(`{
        "loadBalancingPolicy": "round_robin",
        "methodConfig": [{
            "name": [{"service": ""}],
            "timeout": "5s",
            "retryPolicy": {
                "maxAttempts": 3,
                "initialBackoff": "0.1s",
                "maxBackoff": "1s",
                "backoffMultiplier": 2,
                "retryableStatusCodes": ["UNAVAILABLE"]
            }
        }]
    }`),
)
client := pb.NewUserServiceClient(conn)
```

## エラーハンドリング

gRPCエラーは常に特定のコードを持つ `status.Error` を使用して返す — 生の `error` は `codes.Unknown` になり、クライアントに何もアクション可能な情報を与えない。クライアントはコードを使ってリトライ、フェイルファスト、デグレードを判断する。

| コード               | 使用するタイミング                              |
| -------------------- | ------------------------------------------- |
| `InvalidArgument`    | 不正な入力（フィールド欠落、不正なフォーマット） |
| `NotFound`           | エンティティが存在しない                       |
| `AlreadyExists`      | 作成失敗、エンティティが既に存在する             |
| `PermissionDenied`   | 呼び出し元が権限を持たない                     |
| `Unauthenticated`    | トークンが欠落または無効                       |
| `FailedPrecondition` | システムが必要な状態にない                     |
| `ResourceExhausted`  | レート制限またはクォータ超過                   |
| `Unavailable`        | 一時的な問題、安全にリトライ可能               |
| `Internal`           | 予期しないバグ                               |
| `DeadlineExceeded`   | タイムアウト                                |

```go
// ✗ Bad — caller gets codes.Unknown, can't decide whether to retry
return nil, fmt.Errorf("user not found")

// ✓ Good — specific code lets clients act appropriately
if errors.Is(err, ErrNotFound) {
    return nil, status.Errorf(codes.NotFound, "user %q not found", req.UserId)
}
return nil, status.Errorf(codes.Internal, "lookup failed: %v", err)
```

フィールドレベルのバリデーションエラーには、`status.WithDetails` 経由で `errdetails.BadRequest` を添付する。

## ストリーミング

| パターン | ユースケース |
| --- | --- |
| サーバーストリーミング | サーバーがシーケンスを送信（ログテーリング、結果セット） |
| クライアントストリーミング | クライアントがシーケンスを送信し、サーバーが一度レスポンス（ファイルアップロード、バッチ） |
| 双方向 | 両方が独立して送信（チャット、リアルタイム同期） |

大きな単一メッセージよりもストリーミングを優先する — メッセージサイズ制限を回避しメモリ圧力を低減する。

```go
func (s *server) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    for _, u := range users {
        if err := stream.Send(u); err != nil {
            return err
        }
    }
    return nil
}
```

## テスト

ネットワークオーバーヘッドなしにgRPCスタック全体（シリアライゼーション、インターセプター、メタデータ）を検証するインメモリコネクションに `bufconn` を使用する。エラーシナリオが期待されるgRPCステータスコードを返すことを常にテストする。

[テストパターンと例](references/testing.md)

## セキュリティ

- TLSはプロダクションで有効にしなければならない — 認証情報はメタデータで送信される
- サービス間認証には mTLS を使用するか、サービスメッシュ（Istio、Linkerd）に委任する
- ユーザー認証には `credentials.PerRPCCredentials` を実装し、認証インターセプターでトークンを検証する
- API検出を防ぐため、プロダクションではリフレクションを無効にすべき

## パフォーマンス

| 設定 | 目的 | 典型的な値 |
| --- | --- | --- |
| `keepalive.ServerParameters.Time` | アイドル接続のpingインターバル | 30秒 |
| `keepalive.ServerParameters.Timeout` | ping ack タイムアウト | 10秒 |
| `grpc.MaxRecvMsgSize` | 大きなペイロード用に4MBデフォルトを上書き | 16MB |
| コネクションプーリング | 高負荷ストリーミング用の複数接続 | 4接続 |

ほとんどのサービスはコネクションプーリングを必要としない — 複雑さを追加する前にプロファイリングする。

## よくある間違い

| 間違い | 修正 |
| --- | --- |
| 生の `error` を返す | `codes.Unknown` になる — クライアントがリトライするか判断できない。特定のコードで `status.Errorf` を使用 |
| クライアント呼び出しにデッドラインなし | 遅い上流が無期限にハング。常に `context.WithTimeout` を使用 |
| リクエストごとに新規コネクション | TCP/TLSハンドシェイクが無駄。一度作成して再利用 — HTTP/2はRPCを多重化する |
| プロダクションでリフレクションを有効化 | 攻撃者がすべてのメソッドを列挙できる。開発/ステージングのみで有効化 |
| すべてのエラーに `codes.Internal` | 誤ったコードはクライアントのリトライロジックを壊す。`Unavailable` はリトライをトリガーし、`InvalidArgument` はしない |
| RPC引数にベア型 | `string` にフィールドを追加できない。ラッパーメッセージで後方互換の進化が可能 |
| ヘルスチェックサービスの欠如 | Kubernetesが準備完了を判定できず、デプロイ中にPodを終了させる |
| コンテキストキャンセルを無視 | 呼び出し元が諦めた後も長い操作が継続する。`ctx.Err()` を確認する |

## クロスリファレンス

- デッドラインとキャンセルパターンについては → See `samber/cc-skills-golang@golang-context` skill
- gRPCエラーからGoエラーへのマッピングについては → See `samber/cc-skills-golang@golang-error-handling` skill
- gRPCインターセプター（ログ、トレーシング、メトリクス）については → See `samber/cc-skills-golang@golang-observability` skill
- bufconnを使ったgRPCテストについては → See `samber/cc-skills-golang@golang-testing` skill
