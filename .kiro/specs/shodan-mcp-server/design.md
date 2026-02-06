# 設計書

## 概要

Shodan MCP Serverは、Shodan APIの全機能をMCP（Model Context Protocol）ツールとして公開する非同期Pythonサーバーです。本システムは以下の3つの主要APIカテゴリをサポートします：

- **REST API**: ホスト情報、検索、DNS、データセット、アカウント、ユーティリティツール
- **Streaming API**: リアルタイムイベントストリーム（firehose、ポート別、アラート）
- **Trends API**: 統計情報（トップポート、組織、国）

技術スタック：
- Python 3.10以上
- httpx（非同期HTTPクライアント）
- MCP Python SDK（FastMCP）
- 環境変数ベースの認証

## アーキテクチャ

### システム構成

```
┌─────────────────────────────────────────┐
│     MCPクライアント                      │
│  (Claude Desktop, CLI, etc.)            │
└──────────────┬──────────────────────────┘
               │ MCP Protocol
               ↓
┌─────────────────────────────────────────┐
│      Shodan MCP Server                  │
│  ┌───────────────────────────────────┐  │
│  │   FastMCP Framework               │  │
│  │   - Tool Registration             │  │
│  │   - Request Handling              │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │   API Key Manager                 │  │
│  │   - Priority Resolution           │  │
│  │   - Environment Variable Lookup   │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │   HTTP Client Layer (httpx)       │  │
│  │   - Async Request Execution       │  │
│  │   - SSE Stream Parsing            │  │
│  └───────────────────────────────────┘  │
└──────────────┬──────────────────────────┘
               │ HTTPS
               ↓
┌─────────────────────────────────────────┐
│         Shodan API Services             │
│  ┌─────────────┬─────────────┬────────┐ │
│  │  REST API   │ Streaming   │ Trends │ │
│  │  (api.)     │ (stream.)   │(trends)│ │
│  └─────────────┴─────────────┴────────┘ │
└─────────────────────────────────────────┘
```

### レイヤー構造

1. **MCPインターフェース層**: FastMCPフレームワークを使用してツールを登録・公開
2. **認証層**: APIキーの優先順位解決と検証
3. **HTTPクライアント層**: httpxを使用した非同期リクエスト実行
4. **APIエンドポイント層**: 各Shodan APIエンドポイントへのマッピング

## コンポーネントとインターフェース

### 1. API Key Manager

**責務**: APIキーの取得と優先順位解決

**関数**: `get_api_key(api_key: Optional[str], api_type: Optional[str]) -> str`

**ロジック**:
```
1. api_key引数が提供されている場合 → その値を返す
2. api_type == "stream" の場合 → SHODAN_STREAM_API_KEY環境変数を確認
3. api_type == "trends" の場合 → SHODAN_TRENDS_API_KEY環境変数を確認
4. 上記すべてが該当しない場合 → SHODAN_API_KEY環境変数を確認
5. すべて未設定の場合 → ValueErrorを発生
```

**環境変数**:
- `SHODAN_API_KEY`: デフォルトのAPIキー（すべてのAPI種別で使用）
- `SHODAN_STREAM_API_KEY`: Streaming API専用キー（オプション）
- `SHODAN_TRENDS_API_KEY`: Trends API専用キー（オプション）

### 2. REST APIツール群

各ツールは以下の共通パターンに従います：

**共通構造**:
```python
@mcp.tool()
async def tool_name(
    required_params: str,
    api_key: Optional[str] = None,
    optional_params: Optional[type] = default
) -> dict:
    """ツールの説明"""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, ...}
    url = f"{SHODAN_API_BASE}/endpoint/path"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**実装されるツール**:

| ツール名 | エンドポイント | 主要パラメータ |
|---------|--------------|--------------|
| shodan_host_info | /shodan/host/{ip} | ip, history, minify |
| shodan_host_search | /shodan/host/search | query, facets, page, minify |
| shodan_host_count | /shodan/host/count | query, facets |
| shodan_host_search_facets | /shodan/host/search/facets | - |
| shodan_dns_resolve | /dns/resolve | hostnames |
| shodan_dns_reverse | /dns/reverse | ips |
| shodan_ports | /shodan/ports | - |
| shodan_protocols | /shodan/protocols | - |
| shodan_query | /shodan/query | page |
| shodan_query_search | /shodan/query/search | query, page |
| shodan_query_tags | /shodan/query/tags | size |
| shodan_tools_httpheaders | /tools/httpheaders | - |
| shodan_tools_myip | /tools/myip | - |
| shodan_tools_ping | /tools/ping | host |
| shodan_tools_ports | /tools/ports | - |
| shodan_tools_protocols | /tools/protocols | - |
| shodan_tools_uptime | /tools/uptime | - |
| shodan_tools_whois | /tools/whois | domain |
| shodan_data | /shodan/data | - |
| shodan_data_dataset | /shodan/data/{dataset} | dataset |
| shodan_data_dataset_search | /shodan/data/{dataset}/search | dataset, query, page |
| shodan_account_profile | /account/profile | - |
| shodan_api_info | /api-info | - |

### 3. Streaming APIツール群

**SSEパーサー**: `_parse_sse_events(aiter, limit: int) -> list`

**責務**: Server-Sent Eventsストリームを解析してイベントリストを返す

**アルゴリズム**:
```
1. バッファを初期化
2. 非同期イテレータからチャンクを読み取る
3. チャンクをバッファに追加
4. "\n\n"でイベントを分割
5. "data: "プレフィックスを持つ行を抽出
6. limit数のイベントが収集されたら返す
```

**実装されるツール**:

| ツール名 | エンドポイント | 説明 |
|---------|--------------|------|
| shodan_stream_firehose | /shodan/banners | グローバルfirehoseストリーム |
| shodan_stream_ports | /shodan/port/{port} | 特定ポートのバナーストリーム |
| shodan_stream_alert | /shodan/alert | アカウント監視ネットワークのストリーム |

**共通パラメータ**:
- `limit`: 返すイベント数（デフォルト: 10）
- `api_key`: 認証キー（オプション）

**ストリーミング処理フロー**:
```
1. APIキーを取得（api_type="stream"）
2. httpx.AsyncClientでストリーム接続を確立（timeout=None）
3. resp.aiter_bytes()で非同期イテレーション
4. _parse_sse_events()でSSEイベントを解析
5. limit数のイベントを収集して返す
```

### 4. Trends APIツール群

**実装されるツール**:

| ツール名 | エンドポイント | パラメータ |
|---------|--------------|-----------|
| shodan_trends_top_ports | /api/v1/top-ports | query, days |
| shodan_trends_top_orgs | /api/v1/top-orgs | query, days |
| shodan_trends_top_countries | /api/v1/top-countries | query, days |

**共通パラメータ**:
- `query`: 検索クエリ（必須）
- `days`: 期間（デフォルト: 30）
- `api_key`: 認証キー（オプション）

### 5. MCPサーバー初期化

**FastMCPインスタンス**:
```python
mcp = FastMCP("Shodan MCP Server")
```

**ツール登録**: `@mcp.tool()`デコレータを使用して各関数をMCPツールとして登録

**サーバー起動**:
```python
if __name__ == "__main__":
    mcp.run()
```

## データモデル

### APIキー解決の優先順位

```
Priority 1: 関数引数 (api_key parameter)
    ↓ (未提供の場合)
Priority 2: API種別専用環境変数
    - stream → SHODAN_STREAM_API_KEY
    - trends → SHODAN_TRENDS_API_KEY
    ↓ (未設定の場合)
Priority 3: デフォルト環境変数 (SHODAN_API_KEY)
    ↓ (未設定の場合)
Error: ValueError("Shodan API key must be provided...")
```

### HTTPリクエストパラメータ構造

**REST API共通パラメータ**:
```python
{
    "key": str,           # APIキー（必須）
    "query": str,         # 検索クエリ（検索系エンドポイント）
    "page": int,          # ページ番号（デフォルト: 1）
    "facets": str,        # ファセット指定（カンマ区切り）
    "history": bool,      # 履歴を含める
    "minify": bool        # レスポンスを最小化
}
```

**Streaming API共通パラメータ**:
```python
{
    "key": str,           # APIキー（必須、URLクエリパラメータ）
    "limit": int          # 返すイベント数（内部処理用）
}
```

**Trends API共通パラメータ**:
```python
{
    "key": str,           # APIキー（必須）
    "query": str,         # 検索クエリ（必須）
    "days": int           # 期間（デフォルト: 30）
}
```

### SSEイベント構造

**生のSSE形式**:
```
data: {"ip": "1.2.3.4", "port": 80, ...}

data: {"ip": "5.6.7.8", "port": 443, ...}

```

**解析後の構造**:
```python
[
    '{"ip": "1.2.3.4", "port": 80, ...}',
    '{"ip": "5.6.7.8", "port": 443, ...}',
    ...
]
```

### エラーレスポンス

**ValueError（APIキー未設定）**:
```python
ValueError("Shodan API key must be provided as argument or in SHODAN_API_KEY env var.")
```

**HTTPステータスエラー**:
```python
httpx.HTTPStatusError
# resp.raise_for_status()によって発生
# Shodan APIのエラーレスポンスを含む
```


## 正確性プロパティ

プロパティとは、システムのすべての有効な実行において真であるべき特性や動作のことです。本質的には、システムが何をすべきかについての形式的な記述です。プロパティは、人間が読める仕様と機械で検証可能な正確性保証との橋渡しとなります。

### プロパティ1: APIキー引数の優先順位

*任意の* APIキー文字列に対して、それを引数として`get_api_key()`に渡した場合、返されるキーは渡した文字列と同一である

**検証: 要件 1.1**

### プロパティ2: Streaming API環境変数フォールバック

*任意の* APIキー文字列に対して、SHODAN_STREAM_API_KEY環境変数にそのキーを設定し、引数なしで`get_api_key(api_type="stream")`を呼び出した場合、返されるキーは環境変数の値と同一である

**検証: 要件 1.2**

### プロパティ3: Trends API環境変数フォールバック

*任意の* APIキー文字列に対して、SHODAN_TRENDS_API_KEY環境変数にそのキーを設定し、引数なしで`get_api_key(api_type="trends")`を呼び出した場合、返されるキーは環境変数の値と同一である

**検証: 要件 1.3**

### プロパティ4: デフォルト環境変数フォールバック

*任意の* APIキー文字列とapi_type値に対して、SHODAN_API_KEYのみを設定し、API種別専用の環境変数を設定せず、引数なしで`get_api_key(api_type=type)`を呼び出した場合、返されるキーはSHODAN_API_KEYの値と同一である

**検証: 要件 1.4**

### プロパティ5: オプションパラメータのリクエスト含有

*任意の* REST APIツールとオプションパラメータ（history、minify、facets、page）に対して、そのパラメータを指定して呼び出した場合、構築されるHTTPリクエストのクエリパラメータにそのパラメータが含まれる

**検証: 要件 2.2, 2.3, 3.2, 3.3**

### プロパティ6: HTTPエラーの伝播

*任意の* REST APIツールに対して、モックHTTPクライアントが4xxまたは5xxステータスコードを返す場合、ツール関数はHTTPStatusErrorを発生させる

**検証: 要件 2.4, 11.1, 11.3**

### プロパティ7: SSEイベント解析の正確性

*任意の* 有効なSSE形式文字列（"data: {json}\n\n"の繰り返し）に対して、`_parse_sse_events()`で解析した場合、返されるリストの各要素は元のdata行のJSON部分と一致する

**検証: 要件 8.5**

### プロパティ8: ストリーミングlimitパラメータの遵守

*任意の* limit値（1以上の整数）に対して、十分なイベントを含むSSEストリームを`_parse_sse_events(aiter, limit)`で解析した場合、返されるリストの長さはlimit以下である

**検証: 要件 8.4**

### プロパティ9: Trends APIデフォルトdays値

*任意の* Trends APIツール（top_ports、top_orgs、top_countries）に対して、daysパラメータを指定せずに呼び出した場合、構築されるHTTPリクエストのクエリパラメータに"days": 30が含まれる

**検証: 要件 9.4**

## エラーハンドリング

### 1. APIキー未設定エラー

**発生条件**: すべての環境変数が未設定で、かつapi_key引数も提供されていない

**エラー型**: `ValueError`

**メッセージ**: "Shodan API key must be provided as argument or in SHODAN_API_KEY env var."

**処理**: 
- `get_api_key()`関数内で発生
- 呼び出し元に伝播
- MCPクライアントにエラーとして返される

### 2. HTTPステータスエラー

**発生条件**: Shodan APIが4xx（クライアントエラー）または5xx（サーバーエラー）を返す

**エラー型**: `httpx.HTTPStatusError`

**処理**:
- `resp.raise_for_status()`によって自動的に発生
- エラーレスポンスボディを含む
- 呼び出し元に伝播
- MCPクライアントにエラーとして返される

**一般的なHTTPエラー**:
- 401 Unauthorized: 無効なAPIキー
- 403 Forbidden: APIプランの制限超過
- 404 Not Found: 存在しないリソース
- 429 Too Many Requests: レート制限超過
- 500 Internal Server Error: Shodanサーバーエラー

### 3. ネットワークエラー

**発生条件**: ネットワーク接続の問題、タイムアウト、DNS解決失敗

**エラー型**: `httpx.RequestError`（またはそのサブクラス）

**処理**:
- httpxによって自動的に発生
- 呼び出し元に伝播
- MCPクライアントにエラーとして返される

### 4. ストリーミングエラー

**発生条件**: ストリーミング接続の中断、不正なSSE形式

**エラー型**: `httpx.HTTPStatusError`または`httpx.StreamError`

**処理**:
- ストリーム接続確立時またはイテレーション中に発生
- 部分的に収集されたイベントは失われる
- 呼び出し元に伝播

### エラーハンドリング戦略

1. **早期失敗**: APIキーの検証は最初に実行
2. **透過的な伝播**: HTTPエラーはそのまま呼び出し元に伝播
3. **明確なメッセージ**: すべてのエラーに説明的なメッセージを含める
4. **リトライなし**: リトライロジックは実装しない（クライアント側の責任）
5. **リソースクリーンアップ**: httpx.AsyncClientのコンテキストマネージャーで自動クリーンアップ

## テスト戦略

### デュアルテストアプローチ

本システムのテストは、ユニットテストとプロパティベーステストの両方を使用します：

- **ユニットテスト**: 特定の例、エッジケース、エラー条件を検証
- **プロパティベーステスト**: すべての入力にわたる普遍的なプロパティを検証

両者は補完的であり、包括的なカバレッジに必要です。

### 1. プロパティベーステスト

**使用ライブラリ**: `hypothesis`（Python用プロパティベーステストライブラリ）

**設定**:
- 各プロパティテストは最低100回の反復を実行
- 各テストは設計書のプロパティを参照するコメントタグを含める
- タグ形式: `# Feature: shodan-mcp-server, Property {番号}: {プロパティテキスト}`

**テスト対象プロパティ**:

1. **APIキー優先順位解決**（プロパティ1-4）
   - ランダムなAPIキー文字列を生成
   - 環境変数と引数の組み合わせをテスト
   - 正しい優先順位で解決されることを検証

2. **リクエストパラメータ構築**（プロパティ5）
   - ランダムなパラメータ値を生成
   - モックHTTPクライアントでリクエストをキャプチャ
   - パラメータが正しく含まれることを検証

3. **エラー伝播**（プロパティ6）
   - ランダムなHTTPステータスコード（4xx、5xx）を生成
   - モックレスポンスを返すHTTPクライアントを使用
   - 適切な例外が発生することを検証

4. **SSE解析**（プロパティ7-8）
   - ランダムなSSE形式データを生成
   - 解析結果が正しいことを検証
   - limit値が遵守されることを検証

5. **デフォルト値処理**（プロパティ9）
   - パラメータなしで呼び出し
   - デフォルト値がリクエストに含まれることを検証

**テスト実装例**:
```python
from hypothesis import given, strategies as st
import pytest

# Feature: shodan-mcp-server, Property 1: APIキー引数の優先順位
@given(api_key=st.text(min_size=1))
def test_api_key_argument_priority(api_key):
    """任意のAPIキー文字列に対して、引数として渡した場合、同じキーが返される"""
    result = get_api_key(api_key=api_key)
    assert result == api_key
```

### 2. ユニットテスト

**使用ライブラリ**: `pytest`、`pytest-asyncio`、`pytest-mock`

**テスト対象**:

1. **APIキー未設定エラー**
   - すべての環境変数をクリア
   - `get_api_key()`を引数なしで呼び出し
   - ValueErrorが発生することを検証

2. **SSE解析の具体例**
   - 既知のSSE形式文字列を使用
   - 期待される出力と一致することを検証
   - 空のストリーム、単一イベント、複数イベントをテスト

3. **HTTPクライアントのモック**
   - `pytest-mock`を使用してhttpx.AsyncClientをモック
   - 各REST APIツールが正しいURLとパラメータで呼び出されることを検証
   - レスポンスが正しく処理されることを検証

4. **エッジケース**
   - 空のクエリ文字列
   - 非常に長いクエリ文字列
   - 特殊文字を含むパラメータ
   - limit=0のストリーミング

**テスト実装例**:
```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_host_info_with_history():
    """historyオプションがリクエストに含まれることを検証"""
    with patch('httpx.AsyncClient') as mock_client:
        mock_response = AsyncMock()
        mock_response.json.return_value = {"ip": "1.2.3.4"}
        mock_response.raise_for_status = AsyncMock()
        mock_client.return_value.__aenter__.return_value.get.return_value = mock_response
        
        result = await shodan_host_info("1.2.3.4", api_key="test", history=True)
        
        # リクエストにhistory=trueが含まれることを検証
        call_args = mock_client.return_value.__aenter__.return_value.get.call_args
        assert call_args[1]['params']['history'] == 'true'
```

### 3. 統合テスト

**目的**: 実際のShodan APIとの統合を検証（オプション）

**実装**:
- 実際のAPIキーを使用（CI/CD環境変数から取得）
- レート制限を考慮して少数のテストのみ実行
- 本番環境では実行しない（開発環境のみ）

**テスト対象**:
- 基本的なホスト情報取得
- 簡単な検索クエリ
- アカウント情報取得

### 4. テストカバレッジ目標

- **行カバレッジ**: 90%以上
- **分岐カバレッジ**: 85%以上
- **すべてのプロパティ**: プロパティベーステストで検証
- **すべてのエラーパス**: ユニットテストで検証

### 5. CI/CDパイプライン

**テスト実行順序**:
1. ユニットテスト（高速、常に実行）
2. プロパティベーステスト（中速、常に実行）
3. 統合テスト（低速、オプション）

**品質ゲート**:
- すべてのテストが合格
- カバレッジ目標を達成
- リンターエラーなし（ruff、mypy）

### 6. モックとフィクスチャ

**共通フィクスチャ**:
```python
@pytest.fixture
def mock_env_vars(monkeypatch):
    """環境変数をクリーンな状態にする"""
    monkeypatch.delenv("SHODAN_API_KEY", raising=False)
    monkeypatch.delenv("SHODAN_STREAM_API_KEY", raising=False)
    monkeypatch.delenv("SHODAN_TRENDS_API_KEY", raising=False)

@pytest.fixture
def mock_httpx_client():
    """httpx.AsyncClientをモックする"""
    with patch('httpx.AsyncClient') as mock:
        yield mock
```

### テストバランス

- ユニットテストは特定の例とエッジケースに焦点を当てる
- プロパティベーステストは多数の入力をカバーするため、過度なユニットテストは避ける
- 統合ポイント、エッジケース、エラー条件にユニットテストを集中させる
- 普遍的なプロパティと包括的な入力カバレッジにプロパティテストを集中させる
