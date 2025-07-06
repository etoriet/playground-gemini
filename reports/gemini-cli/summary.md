# gemini-cli 基本解析レポート

## 1. システムの概要

`@google/gemini-cli`は、Geminiモデルとの対話や、関連する開発タスクを支援するためのコマンドラインインターフェース（CLI）ツールです。
リポジトリは、UI層とコアロジック層が分離されたモノレポ構成となっています。

### 1.1. システム構成

*   **`@google/gemini-cli` (cli パッケージ):**
    *   ユーザーとのインタラクションを担当するUI層のパッケージです。
    *   `ink`および`react`を利用し、ターミナル上でリッチなUIを構築しています。
    *   `yargs`を用いて、コマンドライン引数の解釈および処理を行います。
*   **`@google/gemini-cli-core` (core パッケージ):**
    *   CLIのコア機能を提供するパッケージです。
    *   `@google/genai`ライブラリを介して、Geminiモデルとの連携機能を提供します。
    *   `@modelcontextprotocol/sdk`を利用し、Model-centric Protocol (MCP) に準拠した機能を提供します。
    *   `simple-git`を利用し、Gitリポジトリの操作機能を提供します。

### 1.2. 使用言語、バージョン

*   **言語:** TypeScript
*   **Node.js:** `>=20.0.0`

### 1.3. 各システムのエントリポイント

*   **CLI実行:**
    *   `gemini`コマンドとして実行されます。
    *   エントリポイントのファイルは `packages/cli/dist/index.js` です。

## 2. ネットワーク概要

`gemini-cli`は、そのコア機能を実現するために複数の外部サービスと通信します。

*   **主要な通信**:
    *   **Gemini API**: モデルとの対話のため `cloudcode-pa.googleapis.com` と通信します。
    *   **Google OAuth2.0**: ユーザー認証とトークン取得のためにGoogleの認証エンドポイントと通信します。
    *   **Webコンテンツ取得**: `web_fetch`ツールがユーザー指定のURLからコンテンツを取得します。
    *   **MCPサーバー**: 外部ツールと連携するために、設定されたMCPサーバーと通信します。
    *   **テレメトリー**: サービス改善のため、利用状況データを `play.googleapis.com/log` に送信します。
*   **実装**:
    *   通信には主に `google-auth-library`, `@modelcontextprotocol/sdk`, `node-fetch` などのライブラリが使用されています。
    *   APIリクエスト失敗時には、Exponential Backoff戦略による自動リトライ処理が実装されています。
    *   プロキシ経由での通信もサポートされています。

## 3. データベース概要

`gemini-cli`は、設定や認証トークンといった情報をローカルのファイルシステム（`~/.gemini/` ディレクトリ）に永続化します。

リレーショナルデータベース（RDBMS）のような専用のデータベースシステムは使用していません。