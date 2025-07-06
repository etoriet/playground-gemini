# Gemini CLI ネットワーク解析レポート

## 1. はじめに

このレポートは、`gemini-cli` のネットワーク関連の動作について解析した結果をまとめたものです。
`gemini-cli` は、コア機能であるGeminiモデルとの対話、認証、Webコンテンツ取得、外部ツール連携、テレメトリー送信など、様々な目的で外部とネットワーク通信を行います。

## 2. 通信の種類と目的

`gemini-cli` が行う主なネットワーク通信は以下の通りです。

### 2.1. Gemini API (Google AI) へのリクエスト

- **目的**: Geminiモデルとの対話（コンテンツ生成、トークン数計算など）の実現。
- **エンドポイント**: `https://cloudcode-pa.googleapis.com`
- **プロトコル**: HTTPS
- **実装箇所**: `packages/core/src/code_assist/server.ts` の `CodeAssistServer` クラスがリクエストを管理しています。`gaxios` ライブラリ（`google-auth-library`経由）を用いてAPIリクエストを送信します。

### 2.2. Google OAuth2.0 認証

- **目的**: Googleアカウントを使用したユーザー認証と、APIアクセスのためのトークン取得。
- **エンドポイント**:
    - 認証URL生成: `https://accounts.google.com/o/oauth2/v2/auth`
    - トークン交換: `https://oauth2.googleapis.com/token`
    - ユーザー情報: `https://www.googleapis.com/auth/userinfo.email` など
- **プロトコル**: HTTPS
- **実装箇所**: `packages/core/src/code_assist/oauth2.ts` に実装されています。ローカルで一時的なHTTPサーバー(`http://localhost:{port}/oauth2callback`)を起動し、認証コードを受け取ることでOAuthフローを完了させます。

### 2.3. Webコンテンツ取得 (`web_fetch` ツール)

- **目的**: ユーザーがプロンプトで指定したURLのコンテンツを取得し、モデルのコンテキストとして利用する。
- **エンドポイント**: ユーザーが指定する任意のURL (`http://` または `https://`)。
- **プロトコル**: HTTP/HTTPS
- **実装箇所**: `packages/core/src/tools/web-fetch.ts` の `WebFetchTool` クラスに実装されています。
    - Gemini APIの`urlContext`機能を優先的に使用します。
    - プライベートIPアドレス範囲へのアクセスや、APIが直接アクセスできないURLの場合は、`node-fetch`を使用したフォールバック機能がローカルマシンから直接コンテンツを取得します。
    - GitHubのBlob URLは、RawコンテンツのURL (`raw.githubusercontent.com`) に自動的に変換されます。

### 2.4. MCP (Model Context Protocol) サーバーとの通信

- **目的**: 外部で動作しているツールサーバーと連携し、CLIの機能を拡張する。
- **エンドポイント**: 設定ファイルで指定された任意のサーバー。
- **プロトコル**: Streamable HTTP, Server-Sent Events (SSE), Stdio。
- **実装箇所**: `packages/core/src/tools/mcp-client.ts` に実装されています。`@modelcontextprotocol/sdk`ライブラリを利用して、設定に応じたトランスポートでサーバーに接続し、ツール定義を取得・実行します。

### 2.5. テレメトリーデータ送信

- **目的**: CLIの利用状況やパフォーマンスに関する匿名データを収集し、サービス改善に役立てる。
- **エンドポイント**: `https://play.googleapis.com/log`
- **プロトコル**: HTTPS
- **実装箇所**: `packages/core/src/telemetry/clearcut-logger/clearcut-logger.ts` に実装されています。Node.jsの標準`https`モジュールを使用して、収集したログイベントを定期的にバッチ送信します。

## 3. 使用ライブラリ

ネットワーク通信には、主に以下のライブラリが使用されています。

- **`google-auth-library`**: Googleの各種APIへの認証とリクエスト送信の基盤。内部で`gaxios`を使用。
- **`gaxios`**: `google-auth-library`が利用するHTTPクライアント。
- **`@modelcontextprotocol/sdk`**: MCPサーバーとの通信に使用。
- **`node-fetch`**: `web_fetch`ツールのフォールバック機能など、直接的なHTTPリクエストに使用。
- **`http` / `https` (Node.js標準)**: OAuthのコールバック処理や、Clearcutへのロギングなど、基本的なHTTP通信に使用。

## 4. エラーハンドリングとリトライ

- **実装箇所**: `packages/core/src/utils/retry.ts`
- **動作**: `retryWithBackoff` 関数により、APIリクエストが失敗した際の自動リトライ処理が行われます。
- **対象エラー**: HTTPステータスコード `429 (Too Many Requests)` および `5xx` 系のサーバーエラーが対象です。
- **戦略**:
    - **Exponential Backoff with Jitter**: リトライ間隔を指数関数的に増加させつつ、ランダムな揺らぎ（ジッター）を加えることで、サーバーへの負荷集中を避けます。
    - **`Retry-After`ヘッダー**: サーバーからのレスポンスに`Retry-After`ヘッダーが含まれている場合は、その指示に従って待機します。

## 5. セキュリティ

- **認証**: OAuth 2.0プロトコルに基づいたセキュアな認証フローを採用しています。認証情報はローカルの `~/.gemini/` ディレクトリに保存されます。
- **プロキシ**: サンドボックス環境での実行時、`GEMINI_SANDBOX_PROXY_COMMAND` 環境変数を設定することで、全ての通信を指定したプロキシサーバー経由に強制できます。
    - `scripts/example-proxy.js` には、特定のドメインへのHTTPS通信のみを許可するプロキシの実装例が含まれており、きめ細やかなアクセス制御が可能です。
- **プライベートIPの検知**: `web_fetch`ツールは、リクエスト先のURLがプライベートIPアドレスでないかを確認し、挙動を切り替えることで意図しない内部ネットワークへのアクセスを防ぎます。
- **CSRF対策**: OAuth認証フローにおいて、`state`パラメータを使用することでクロスサイトリクエストフォージェリ（CSRF）攻撃を防止しています。
