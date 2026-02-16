# Shodan MCP Server

完全非同期、プロダクショングレードのMCPサーバーで、Shodan APIの全機能（REST、Streaming、Trends）をMCPツールとして公開します。Python、`httpx`、MCP Python SDKで構築されており、Shodanの強力なインターネットインテリジェンスツールを、自動化、研究、セキュリティワークフローにシームレスに統合できます。

**STDIOとStreamableHTTPの両方のトランスポートに対応しました！**

## 機能

- **Shodan APIの完全カバレッジ:**
  - REST API: ホスト情報、検索、DNS、データセット、アカウント、ツールなど
  - Streaming API: リアルタイムfirehose、ポート別、アラートイベントストリーム
  - Trends API: 任意のクエリに対するトップポート、組織、国の統計
- **非同期＆高パフォーマンス:**
  - `httpx`とasync/awaitで構築され、最大の並行性と速度を実現
- **安全な認証:**
  - APIキーは各API種別ごとに環境変数から読み込まれます
- **複数のトランスポートオプション:**
  - **STDIO**: Claude DesktopとCLI統合用
  - **StreamableHTTP**: Webベースのクライアントとリモートアクセス用（SSE対応）
- **簡単な統合:**
  - Claude Desktop、CLI、または任意のMCP互換クライアントですぐに使用可能

## 環境変数

サーバーを実行する前に、Shodan APIの認証情報を環境変数として設定してください：

- `SHODAN_API_KEY` (すべてのAPIのデフォルト)
- `SHODAN_STREAM_API_KEY` (オプション、Streaming API用)
- `SHODAN_TRENDS_API_KEY` (オプション、Trends API用)

特定のキーが設定されていない場合、サーバーは`SHODAN_API_KEY`にフォールバックします。

## 使用方法

### 1. 依存関係のインストール

```sh
uv add "mcp[cli]>=1.12.0"
uv add "httpx>=0.25.0"
uv add "uvicorn>=0.24.0"
```

### 2. 環境変数の設定

```sh
export SHODAN_API_KEY=your_main_shodan_api_key
# オプション:
export SHODAN_STREAM_API_KEY=your_streaming_key
export SHODAN_TRENDS_API_KEY=your_trends_key
```

### 3. サーバーの起動

#### オプションA: STDIOトランスポート（Claude Desktop用）

```sh
uv --directory /path/to/shodan-mcp run server.py
```

#### オプションB: StreamableHTTPトランスポート（Webクライアント用）

```sh
# デフォルトポート8000
uv --directory /path/to/shodan-mcp run server.py --http

# カスタムポート
uv --directory /path/to/shodan-mcp run server.py --http --port 3000
```

### 4. クライアント設定

#### Claude Desktop用（STDIO）

以下をClaude DesktopのMCP設定に保存してください：

```json
{
  "mcpServers": {
    "ShodanMCP": {
      "command": "uv",
      "args": [
        "--directory", "/path/to/shodan-mcp",
        "run", "server.py"
      ]
    }
  }
}
```

#### Claude Desktop用（SSE - Server-Sent Events）

SSEトランスポートを使用する場合（別途サーバーを起動する必要があります）：

```json
{
  "mcpServers": {
    "ShodanMCP-SSE": {
      "command": "uv",
      "args": [
        "--directory", "/path/to/shodan-mcp",
        "run", "server.py",
        "--http",
        "--port", "8000"
      ]
    }
  }
}
```

#### Claude Desktop用（StreamableHTTP）

StreamableHTTPトランスポートを使用する場合（リモートサーバーに接続）：

```json
{
  "mcpServers": {
    "ShodanMCP-HTTP": {
      "url": "http://localhost:8000/sse",
      "transport": "sse"
    }
  }
}
```

**注意:** StreamableHTTPを使用する場合は、事前に別のターミナルでサーバーを起動しておく必要があります：

```sh
uv --directory /path/to/shodan-mcp run server.py --http --port 8000
```

#### 一般的なHTTPクライアント用

以下のURLでサーバーに接続してください：
```
http://localhost:8000/sse
```

サーバーはリアルタイム通信用のServer-Sent Events（SSE）を提供します。

## トランスポートオプション

### STDIOトランスポート（デフォルト）
- 最適な用途: Claude Desktop、CLIツール、ローカル開発
- 接続: 標準入出力ストリーム
- 設定: 上記の「Claude Desktop用（STDIO）」セクションを参照

### StreamableHTTPトランスポート
- 最適な用途: Webベースのクライアント、リモートアクセス、分散システム
- 接続: Server-Sent Events（SSE）を使用したHTTP
- ポート: 設定可能（デフォルト: 8000）
- エンドポイント: `http://localhost:PORT/sse`
- 機能:
  - SSEによるリアルタイム双方向通信
  - 複数の同時クライアント接続
  - リモートマシンからのネットワークアクセス
  - Webブラウザおよび HTTPクライアントと互換性あり

## 影響とユースケース

- **セキュリティ研究:** Shodanのグローバルなインターネットインテリジェンスを即座にクエリし、脅威ハンティング、資産発見、脆弱性調査を実施
- **自動化:** MCPを介してShodanツールをセキュリティパイプライン、SIEM、カスタムダッシュボードに統合
- **リアルタイム監視:** ライブバナーとアラートをストリーミングし、インフラストラクチャやオープンインターネットをプロアクティブに監視
- **データサイエンス:** Shodanの Trends APIを活用して、グローバルなインターネットトレンドの分析、レポート作成、可視化を実現
- **リモートアクセス:** StreamableHTTPトランスポートを使用して、WebアプリケーションやリモートクライアントからShodanツールにアクセス

## クレジット

- **Shodan** — インターネット接続デバイスの世界をリードする検索エンジン

---

© 2024 Haji & Contributors. このプロジェクトはShodanと提携していません。Shodanデータの商用利用については、[Shodanの利用規約](https://www.shodan.io/terms)に準拠してください。
