# Shodan MCP Server

完全非同期、プロダクショングレードのMCPサーバーで、Shodan APIの全機能（REST、Streaming、Trends）をMCPツールとして公開します。Python、`httpx`、MCP Python SDKで構築されており、Shodanの強力なインターネットインテリジェンスツールを、自動化、研究、セキュリティワークフローにシームレスに統合できます。

## 機能

- **Shodan APIの完全カバレッジ:**
  - REST API: ホスト情報、検索、DNS、データセット、アカウント、ツールなど
  - Streaming API: リアルタイムfirehose、ポート別、アラートイベントストリーム
  - Trends API: 任意のクエリに対するトップポート、組織、国の統計
- **非同期＆高パフォーマンス:**
  - `httpx`とasync/awaitで構築され、最大の並行性と速度を実現
- **安全な認証:**
  - APIキーは各API種別ごとに環境変数から読み込まれます
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
uv add "mcp[cli]"
uv add "sseclient-py"
```

### 2. 環境変数の設定

```sh
export SHODAN_API_KEY=your_main_shodan_api_key
# オプション:
export SHODAN_STREAM_API_KEY=your_streaming_key
export SHODAN_TRENDS_API_KEY=your_trends_key
```

### 3. サーバーの起動 (uvicornまたはuvを使用)

```sh
uv --directory /Users/haji/mcp-servers/shodan-mcp run server.py
```

### 4. Claude Desktop用のMCPサーバー設定

以下を`.json`として保存し、Claude DesktopまたはCursorで読み込んでください：

```json
{
  "mcpServers": {
    "ShodanMCP": {
      "command": "uv",
      "args": [
        "--directory", "/Users/haji/mcp-servers/shodan-mcp",
        "run", "server.py"
      ]
    }
  }
}
```

## 影響とユースケース

- **セキュリティ研究:** Shodanのグローバルなインターネットインテリジェンスを即座にクエリし、脅威ハンティング、資産発見、脆弱性調査を実施
- **自動化:** MCPを介してShodanツールをセキュリティパイプライン、SIEM、カスタムダッシュボードに統合
- **リアルタイム監視:** ライブバナーとアラートをストリーミングし、インフラストラクチャやオープンインターネットをプロアクティブに監視
- **データサイエンス:** Shodanの Trends APIを活用して、グローバルなインターネットトレンドの分析、レポート作成、可視化を実現

## クレジット

- **Shodan** — インターネット接続デバイスの世界をリードする検索エンジン

---

© 2024 Haji & Contributors. このプロジェクトはShodanと提携していません。Shodanデータの商用利用については、[Shodanの利用規約](https://www.shodan.io/terms)に準拠してください。
