# server.py 詳細説明ドキュメント

## 目次

1. [概要](#概要)
2. [インポート文](#インポート文)
3. [定数定義](#定数定義)
4. [ヘルパー関数](#ヘルパー関数)
5. [REST APIツール群](#rest-apiツール群)
6. [Streaming APIツール群](#streaming-apiツール群)
7. [Trends APIツール群](#trends-apiツール群)
8. [サーバー起動](#サーバー起動)

---

## 概要

`server.py`は、Shodan APIの全機能をMCP（Model Context Protocol）ツールとして公開する非同期Pythonサーバーの実装です。
約400行のコードで、REST API、Streaming API、Trends APIの3つのカテゴリをカバーしています。

---

## インポート文

### 1-5行目: 必要なライブラリのインポート

```python
import os
import httpx
import asyncio
from typing import Optional
from mcp.server.fastmcp import FastMCP
```

**詳細説明:**

- **1行目 `import os`**: 環境変数へのアクセスに使用（APIキーの取得）
- **2行目 `import httpx`**: 非同期HTTPクライアントライブラリ。すべてのAPI呼び出しに使用
- **3行目 `import asyncio`**: 非同期処理のための標準ライブラリ（直接使用はしていないが、async/awaitの基盤）
- **4行目 `from typing import Optional`**: 型ヒントで「値またはNone」を表現するために使用
- **5行目 `from mcp.server.fastmcp import FastMCP`**: MCPサーバーフレームワーク。ツールの登録と公開を簡素化

---

## 定数定義

### 7行目: MCPサーバーインスタンスの作成

```python
mcp = FastMCP("Shodan MCP Server")
```

**詳細説明:**
- FastMCPクラスのインスタンスを作成し、サーバー名を"Shodan MCP Server"として設定
- このインスタンスは、後続のすべてのツール登録に使用される
- `@mcp.tool()`デコレータでツールを登録する際の基盤となる

### 9-11行目: APIベースURL定義

```python
SHODAN_API_BASE = "https://api.shodan.io"
SHODAN_STREAM_BASE = "https://stream.shodan.io"
SHODAN_TRENDS_BASE = "https://trends.shodan.io"
```

**詳細説明:**
- **9行目**: REST APIのベースURL。ホスト情報、検索、DNS、データセットなどのエンドポイントに使用
- **10行目**: Streaming APIのベースURL。リアルタイムイベントストリームに使用
- **11行目**: Trends APIのベースURL。統計情報（トップポート、組織、国）に使用

これらの定数により、エンドポイントURLの構築が簡潔になり、変更時の保守性が向上します。

---


## ヘルパー関数

### 13-30行目: get_api_key() - APIキー取得関数

```python
def get_api_key(api_key: Optional[str] = None, api_type: Optional[str] = None) -> str:
    """
    Get the API key for the given API type (rest, stream, trends).
    Priority: explicit argument > env var for type > SHODAN_API_KEY
    """
    if api_key:
        return api_key
    if api_type == "stream":
        key = os.getenv("SHODAN_STREAM_API_KEY")
        if key:
            return key
    if api_type == "trends":
        key = os.getenv("SHODAN_TRENDS_API_KEY")
        if key:
            return key
    key = os.getenv("SHODAN_API_KEY")
    if not key:
        raise ValueError("Shodan API key must be provided as argument or in SHODAN_API_KEY env var.")
    return key
```

**詳細説明:**

**13行目**: 関数定義
- `api_key: Optional[str] = None`: 明示的に提供されるAPIキー（オプション）
- `api_type: Optional[str] = None`: API種別（"rest"、"stream"、"trends"）
- `-> str`: 返り値は文字列型のAPIキー

**14-17行目**: ドキュメント文字列
- 関数の目的と優先順位ロジックを説明

**18-19行目**: 優先順位1 - 明示的な引数
- `api_key`引数が提供されている場合、それを即座に返す
- これが最高優先順位

**20-23行目**: 優先順位2 - Streaming API専用環境変数
- `api_type`が"stream"の場合、`SHODAN_STREAM_API_KEY`環境変数を確認
- 設定されていれば、その値を返す

**24-27行目**: 優先順位3 - Trends API専用環境変数
- `api_type`が"trends"の場合、`SHODAN_TRENDS_API_KEY`環境変数を確認
- 設定されていれば、その値を返す

**28-30行目**: 優先順位4 - デフォルト環境変数とエラー処理
- `SHODAN_API_KEY`環境変数を確認（すべてのAPI種別で使用可能）
- 設定されていない場合、説明的な`ValueError`を発生させる
- これにより、APIキーが完全に未設定の場合に早期に失敗する

**設計の利点:**
- 柔軟性: 異なるAPI種別に異なるキーを使用可能
- セキュリティ: 環境変数を使用してキーをコードから分離
- 明確性: 優先順位が明確で予測可能

---

## REST APIツール群

REST APIツールは、Shodan APIの標準的なHTTPエンドポイントにアクセスします。
すべてのツールは共通のパターンに従います：

1. `@mcp.tool()`デコレータでMCPツールとして登録
2. 非同期関数として定義（`async def`）
3. `get_api_key()`でAPIキーを取得
4. `httpx.AsyncClient`でHTTPリクエストを実行
5. `resp.raise_for_status()`でエラーチェック
6. JSONレスポンスを返す

### 34-47行目: shodan_host_info() - ホスト情報取得

```python
@mcp.tool()
async def shodan_host_info(ip: str, api_key: Optional[str] = None, history: Optional[bool] = False, minify: Optional[bool] = False) -> dict:
    """Returns all services that have been found on the given host IP."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    if history:
        params["history"] = "true"
    if minify:
        params["minify"] = "true"
    url = f"{SHODAN_API_BASE}/shodan/host/{ip}"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**34行目**: `@mcp.tool()`デコレータ
- この関数をMCPツールとして登録
- MCPクライアント（Claude Desktopなど）から呼び出し可能になる

**35行目**: 関数定義
- `ip: str`: 検索対象のIPアドレス（必須）
- `api_key: Optional[str] = None`: APIキー（オプション）
- `history: Optional[bool] = False`: 過去のスキャン履歴を含めるか
- `minify: Optional[bool] = False`: レスポンスを最小化するか
- `-> dict`: 返り値は辞書型（JSONレスポンス）

**37行目**: APIキー取得
- `get_api_key()`を呼び出し、優先順位ロジックに従ってキーを取得
- `api_type="rest"`を指定（REST API用）

**38-42行目**: リクエストパラメータ構築
- 基本パラメータとして`key`を設定
- `history`が`True`の場合、パラメータに追加（文字列"true"として）
- `minify`が`True`の場合、パラメータに追加

**43行目**: URL構築
- f-stringを使用してIPアドレスをURLに埋め込む
- 例: `https://api.shodan.io/shodan/host/8.8.8.8`

**44-47行目**: 非同期HTTPリクエスト
- `async with`でAsyncClientのコンテキストマネージャーを使用（自動クリーンアップ）
- `await client.get()`で非同期GETリクエストを実行
- `resp.raise_for_status()`で4xx/5xxエラーをチェック（エラー時は例外発生）
- `resp.json()`でJSONレスポンスをPython辞書に変換して返す


### 49-60行目: shodan_host_count() - 検索結果カウント

```python
@mcp.tool()
async def shodan_host_count(query: str, api_key: Optional[str] = None, facets: Optional[str] = None) -> dict:
    """Search Shodan without results, only returns the total number of results and facet info."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query}
    if facets:
        params["facets"] = facets
    url = f"{SHODAN_API_BASE}/shodan/host/count"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**50行目**: 関数定義
- `query: str`: 検索クエリ（必須）。Shodanの検索構文を使用
- `facets: Optional[str]`: ファセット指定（オプション）。カンマ区切りで複数指定可能

**54-56行目**: パラメータ構築
- `query`は必須パラメータとして常に含まれる
- `facets`が提供された場合のみ追加（例: "country,port"）

**57行目**: URL構築
- `/shodan/host/count`エンドポイントを使用
- 実際の検索結果は返さず、総数とファセット情報のみを返す（高速）

**用途:**
- 検索結果の総数を確認したい場合
- ファセット情報（国別、ポート別などの統計）のみが必要な場合
- 完全な検索結果を取得する前の事前確認

### 62-72行目: shodan_dns_resolve() - DNS正引き

```python
@mcp.tool()
async def shodan_dns_resolve(hostnames: str, api_key: Optional[str] = None) -> dict:
    """Look up the IP address for the provided list of hostnames (comma-separated)."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "hostnames": hostnames}
    url = f"{SHODAN_API_BASE}/dns/resolve"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**63行目**: 関数定義
- `hostnames: str`: カンマ区切りのホスト名リスト（例: "google.com,github.com"）

**65行目**: パラメータ構築
- `hostnames`パラメータをそのまま渡す（カンマ区切り文字列）

**66行目**: URL構築
- `/dns/resolve`エンドポイントを使用（DNS正引き）

**用途:**
- ホスト名からIPアドレスを取得
- 複数のホスト名を一度に解決可能
- Shodanのインフラを使用した信頼性の高いDNS解決

### 74-83行目: shodan_api_info() - APIプラン情報取得

```python
@mcp.tool()
async def shodan_api_info(api_key: Optional[str] = None) -> dict:
    """Returns information about the API plan belonging to the given API key."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/api-info"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**75行目**: 関数定義
- パラメータは`api_key`のみ（オプション）

**79行目**: URL構築
- `/api-info`エンドポイントを使用

**用途:**
- APIキーに関連するプラン情報を確認
- クエリクレジット残高の確認
- スキャンクレジット残高の確認
- プランの種類（無料、メンバー、企業など）の確認

### 85-98行目: shodan_host_search() - ホスト検索

```python
@mcp.tool()
async def shodan_host_search(query: str, api_key: Optional[str] = None, facets: Optional[str] = None, page: Optional[int] = 1, minify: Optional[bool] = False) -> dict:
    """Search Shodan using the same query syntax as the website and return up to 100 results per page."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query, "page": page}
    if facets:
        params["facets"] = facets
    if minify:
        params["minify"] = "true"
    url = f"{SHODAN_API_BASE}/shodan/host/search"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**86行目**: 関数定義
- `query: str`: 検索クエリ（必須）
- `facets: Optional[str]`: ファセット指定（オプション）
- `page: Optional[int] = 1`: ページ番号（デフォルト: 1）
- `minify: Optional[bool] = False`: レスポンス最小化フラグ

**88行目**: パラメータ構築
- `query`と`page`は常に含まれる
- `page`はデフォルト値1を持つが、明示的に指定可能

**89-92行目**: オプションパラメータ処理
- `facets`と`minify`は提供された場合のみ追加

**93行目**: URL構築
- `/shodan/host/search`エンドポイントを使用
- これがShodanの主要な検索機能

**用途:**
- Shodanウェブサイトと同じ検索構文を使用
- 最大100件の結果を1ページあたり返す
- ページネーションで大量の結果を取得可能


### 100-109行目: shodan_host_search_facets() - 検索ファセット一覧

```python
@mcp.tool()
async def shodan_host_search_facets(api_key: Optional[str] = None) -> dict:
    """List all search facets that can be used when searching Shodan."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/shodan/host/search/facets"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- 検索時に使用可能なすべてのファセットを一覧表示
- ファセット例: country（国）、port（ポート）、org（組織）、asn（AS番号）など
- 検索結果の統計情報を取得する際に使用するファセット名を確認

### 111-120行目: shodan_ports() - クロール中ポート一覧

```python
@mcp.tool()
async def shodan_ports(api_key: Optional[str] = None) -> dict:
    """List all ports that Shodan is crawling on the Internet."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/shodan/ports"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- Shodanが現在クロールしているすべてのポート番号を取得
- 検索可能なポートの範囲を確認
- 新しいポートがクロール対象に追加されたかを確認

### 122-131行目: shodan_protocols() - サポートプロトコル一覧

```python
@mcp.tool()
async def shodan_protocols(api_key: Optional[str] = None) -> dict:
    """List all protocols that can be used when performing on-demand Internet scans via Shodan."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/shodan/protocols"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- オンデマンドスキャンで使用可能なすべてのプロトコルを一覧表示
- プロトコル例: http、https、ssh、ftp、telnet、smtp、dns など
- カスタムスキャンを実行する際に使用可能なプロトコルを確認

### 133-143行目: shodan_query() - 保存済みクエリ一覧

```python
@mcp.tool()
async def shodan_query(api_key: Optional[str] = None, page: Optional[int] = 1) -> dict:
    """List the saved search queries in Shodan."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "page": page}
    url = f"{SHODAN_API_BASE}/shodan/query"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**134行目**: 関数定義
- `page: Optional[int] = 1`: ページ番号（デフォルト: 1）

**用途:**
- ユーザーがShodanに保存した検索クエリを一覧表示
- ページネーションで大量の保存済みクエリを取得可能
- 頻繁に使用する検索クエリの管理

### 145-155行目: shodan_query_search() - クエリディレクトリ検索

```python
@mcp.tool()
async def shodan_query_search(query: str, api_key: Optional[str] = None, page: Optional[int] = 1) -> dict:
    """Search the directory of search queries that users have saved in Shodan."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query, "page": page}
    url = f"{SHODAN_API_BASE}/shodan/query/search"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**146行目**: 関数定義
- `query: str`: 検索キーワード（必須）
- `page: Optional[int] = 1`: ページ番号

**用途:**
- 他のユーザーが保存した検索クエリを検索
- 特定のトピックに関連するクエリを発見
- コミュニティの検索パターンを学習

### 157-167行目: shodan_query_tags() - 人気タグ一覧

```python
@mcp.tool()
async def shodan_query_tags(api_key: Optional[str] = None, size: Optional[int] = 10) -> dict:
    """List the most popular tags for the saved search queries in Shodan."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "size": size}
    url = f"{SHODAN_API_BASE}/shodan/query/tags"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**158行目**: 関数定義
- `size: Optional[int] = 10`: 返すタグ数（デフォルト: 10）

**用途:**
- 保存済みクエリの最も人気のあるタグを取得
- トレンドのトピックを発見
- タグ別にクエリを整理


### 169-179行目: shodan_dns_reverse() - DNS逆引き

```python
@mcp.tool()
async def shodan_dns_reverse(ips: str, api_key: Optional[str] = None) -> dict:
    """Look up the hostnames that have been defined for the given list of IP addresses (comma-separated)."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "ips": ips}
    url = f"{SHODAN_API_BASE}/dns/reverse"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**170行目**: 関数定義
- `ips: str`: カンマ区切りのIPアドレスリスト（例: "8.8.8.8,1.1.1.1"）

**用途:**
- IPアドレスからホスト名を取得（DNS逆引き）
- 複数のIPアドレスを一度に解決可能
- `shodan_dns_resolve()`の逆操作

### 181-190行目: shodan_tools_httpheaders() - HTTPヘッダー確認

```python
@mcp.tool()
async def shodan_tools_httpheaders(api_key: Optional[str] = None) -> dict:
    """Shows the HTTP headers that your client sends when connecting to a webserver."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/tools/httpheaders"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- クライアントが送信するHTTPヘッダーを確認
- User-Agent、Accept、Accept-Encodingなどのヘッダー情報を取得
- デバッグやトラブルシューティングに有用

### 192-202行目: shodan_tools_myip() - 現在のIP取得

```python
@mcp.tool()
async def shodan_tools_myip(api_key: Optional[str] = None) -> str:
    """Get your current IP address as seen from the Internet."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/tools/myip"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.text
```

**詳細説明:**

**193行目**: 関数定義
- `-> str`: 返り値は文字列型（IPアドレス）

**201行目**: レスポンス処理
- `resp.text`を使用（JSONではなくプレーンテキスト）
- IPアドレス文字列を直接返す

**用途:**
- インターネットから見える自分のIPアドレスを確認
- プロキシやVPNの動作確認
- ネットワーク設定の検証

### 204-213行目: shodan_data() - データセット一覧

```python
@mcp.tool()
async def shodan_data(api_key: Optional[str] = None) -> dict:
    """List all datasets that are available for download."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/shodan/data"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- ダウンロード可能なすべてのデータセットを一覧表示
- データセット名、サイズ、説明などの情報を取得
- 大規模データ分析の準備

### 215-225行目: shodan_data_dataset() - データセット詳細

```python
@mcp.tool()
async def shodan_data_dataset(dataset: str, api_key: Optional[str] = None) -> dict:
    """Get information about a specific dataset available for download."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/shodan/data/{dataset}"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**216行目**: 関数定義
- `dataset: str`: データセット名（必須）

**219行目**: URL構築
- データセット名をURLパスに埋め込む
- 例: `/shodan/data/asn-data`

**用途:**
- 特定のデータセットの詳細情報を取得
- ファイルサイズ、更新日時、含まれるレコード数などを確認

### 227-238行目: shodan_data_dataset_search() - データセット内検索

```python
@mcp.tool()
async def shodan_data_dataset_search(dataset: str, query: str, api_key: Optional[str] = None, page: Optional[int] = 1) -> dict:
    """Search within a dataset available for download."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query, "page": page}
    url = f"{SHODAN_API_BASE}/shodan/data/{dataset}/search"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**228行目**: 関数定義
- `dataset: str`: データセット名（必須）
- `query: str`: 検索クエリ（必須）
- `page: Optional[int] = 1`: ページ番号

**用途:**
- 特定のデータセット内で検索を実行
- データセット全体をダウンロードせずに必要なデータを検索
- ページネーションで大量の結果を取得


### 240-249行目: shodan_account_profile() - アカウントプロファイル

```python
@mcp.tool()
async def shodan_account_profile(api_key: Optional[str] = None) -> dict:
    """Get information about the Shodan account linked to the API key."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/account/profile"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- APIキーにリンクされたShodanアカウント情報を取得
- ユーザー名、メールアドレス、作成日などの情報
- アカウントの状態確認

### 251-261行目: shodan_tools_ping() - ホスト到達性確認

```python
@mcp.tool()
async def shodan_tools_ping(host: str, api_key: Optional[str] = None) -> dict:
    """Ping a host to see if it is reachable from Shodan's servers."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "host": host}
    url = f"{SHODAN_API_BASE}/tools/ping"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**252行目**: 関数定義
- `host: str`: ホスト名またはIPアドレス（必須）

**用途:**
- Shodanサーバーから特定のホストが到達可能かを確認
- ネットワーク接続性のテスト
- ファイアウォールやフィルタリングの確認

### 263-272行目: shodan_tools_ports() - ポート一覧（ツール版）

```python
@mcp.tool()
async def shodan_tools_ports(api_key: Optional[str] = None) -> dict:
    """List all ports that Shodan is crawling on the Internet (utility endpoint)."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/tools/ports"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**注意:**
- `shodan_ports()`と同じ機能（111-120行目）
- エンドポイントが異なる（`/tools/ports` vs `/shodan/ports`）
- Shodan APIの設計上、両方のエンドポイントが存在

### 274-283行目: shodan_tools_protocols() - プロトコル一覧（ツール版）

```python
@mcp.tool()
async def shodan_tools_protocols(api_key: Optional[str] = None) -> dict:
    """List all protocols that can be used when performing on-demand Internet scans via Shodan (utility endpoint)."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/tools/protocols"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**注意:**
- `shodan_protocols()`と同じ機能（122-131行目）
- エンドポイントが異なる（`/tools/protocols` vs `/shodan/protocols`）

### 285-294行目: shodan_tools_uptime() - サーバー稼働時間

```python
@mcp.tool()
async def shodan_tools_uptime(api_key: Optional[str] = None) -> dict:
    """Get the uptime of Shodan's servers."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key}
    url = f"{SHODAN_API_BASE}/tools/uptime"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**用途:**
- Shodanサーバーの稼働時間を取得
- サービスの可用性確認
- メンテナンス情報の確認

### 296-306行目: shodan_tools_whois() - WHOIS情報取得

```python
@mcp.tool()
async def shodan_tools_whois(domain: str, api_key: Optional[str] = None) -> dict:
    """Get WHOIS information for a domain."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "domain": domain}
    url = f"{SHODAN_API_BASE}/tools/whois"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**297行目**: 関数定義
- `domain: str`: ドメイン名（必須）

**用途:**
- ドメインのWHOIS情報を取得
- 登録者情報、登録日、有効期限などを確認
- ドメインの所有者調査

---

## Streaming APIツール群

Streaming APIは、Shodanがリアルタイムで収集するデータをストリーミングで提供します。
Server-Sent Events（SSE）プロトコルを使用し、継続的なデータフローを実現します。

### 310-323行目: _parse_sse_events() - SSEパーサー

```python
async def _parse_sse_events(aiter, limit: int = 10):
    events = []
    buffer = ""
    async for chunk in aiter:
        buffer += chunk.decode()
        while "\n\n" in buffer:
            event, buffer = buffer.split("\n\n", 1)
            if event.strip():
                data_lines = [line[6:] for line in event.splitlines() if line.startswith("data: ")]
                if data_lines:
                    events.append("\n".join(data_lines))
            if limit and len(events) >= limit:
                return events
    return events
```

**詳細説明:**

**310行目**: 関数定義
- `aiter`: 非同期イテレータ（バイトチャンクのストリーム）
- `limit: int = 10`: 返すイベント数の上限（デフォルト: 10）
- 注意: この関数は内部ヘルパーで、MCPツールとして登録されていない（`@mcp.tool()`なし）

**311-312行目**: 初期化
- `events`: 解析されたイベントを格納するリスト
- `buffer`: 受信したデータを一時的に保持するバッファ

**313-314行目**: 非同期イテレーション
- `async for chunk in aiter`: ストリームからチャンクを非同期に読み取る
- `chunk.decode()`: バイト列を文字列にデコードしてバッファに追加

**315-320行目**: イベント解析ループ
- **315行目**: `"\n\n"`でイベントを分割（SSE形式の区切り）
- **316行目**: バッファを分割し、最初のイベントと残りを分離
- **317行目**: 空白のみのイベントをスキップ
- **318行目**: "data: "プレフィックスを持つ行を抽出
  - リスト内包表記で各行をチェック
  - `line[6:]`で"data: "（6文字）を除去
- **319-320行目**: データ行が存在する場合、結合してイベントリストに追加

**321-322行目**: 制限チェック
- `limit`が指定され、かつイベント数が制限に達した場合、即座に返す
- これにより、無限ストリームから必要な数だけイベントを取得可能

**323行目**: 最終返却
- ストリームが終了した場合、収集したすべてのイベントを返す

**SSE形式の例:**
```
data: {"ip": "1.2.3.4", "port": 80}

data: {"ip": "5.6.7.8", "port": 443}

```


### 325-335行目: shodan_stream_firehose() - グローバルfirehose

```python
@mcp.tool()
async def shodan_stream_firehose(api_key: Optional[str] = None, limit: Optional[int] = 10) -> list:
    """Stream the global firehose of all data Shodan collects in real time. Returns up to 'limit' events."""
    key = get_api_key(api_key, api_type="stream")
    url = f"{SHODAN_STREAM_BASE}/shodan/banners?key={key}"
    limit_val = limit if limit is not None else 10
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as resp:
            resp.raise_for_status()
            return await _parse_sse_events(resp.aiter_bytes(), limit_val)
```

**詳細説明:**

**326行目**: 関数定義
- `limit: Optional[int] = 10`: 返すイベント数（デフォルト: 10）
- `-> list`: 返り値はリスト型（イベント文字列のリスト）

**328行目**: APIキー取得
- `api_type="stream"`を指定（Streaming API用）
- `SHODAN_STREAM_API_KEY`環境変数が優先される

**329行目**: URL構築
- APIキーをクエリパラメータとして直接URLに埋め込む
- `/shodan/banners`エンドポイント（グローバルfirehose）

**330行目**: limit値の正規化
- `None`の場合はデフォルト値10を使用

**331行目**: AsyncClientの作成
- `timeout=None`: タイムアウトなし（ストリーミング接続のため）
- 通常のHTTPリクエストと異なり、接続を長時間保持

**332行目**: ストリーミング接続
- `client.stream("GET", url)`: ストリーミングGETリクエスト
- コンテキストマネージャーで自動クリーンアップ

**333-334行目**: レスポンス処理
- `resp.raise_for_status()`: エラーチェック
- `resp.aiter_bytes()`: バイトチャンクの非同期イテレータを取得
- `_parse_sse_events()`: SSEイベントを解析
- `await`: 非同期関数の完了を待機

**用途:**
- Shodanが収集するすべてのデータをリアルタイムで取得
- グローバルなインターネット活動の監視
- 大規模なセキュリティ監視システムの構築
- 注意: 大量のデータが流れるため、limitを適切に設定

### 337-347行目: shodan_stream_ports() - ポート別ストリーム

```python
@mcp.tool()
async def shodan_stream_ports(port: int, api_key: Optional[str] = None, limit: Optional[int] = 10) -> list:
    """Stream banners for a specific port in real time. Returns up to 'limit' events."""
    key = get_api_key(api_key, api_type="stream")
    url = f"{SHODAN_STREAM_BASE}/shodan/port/{port}?key={key}"
    limit_val = limit if limit is not None else 10
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as resp:
            resp.raise_for_status()
            return await _parse_sse_events(resp.aiter_bytes(), limit_val)
```

**詳細説明:**

**338行目**: 関数定義
- `port: int`: 監視するポート番号（必須）
- 例: 80（HTTP）、443（HTTPS）、22（SSH）、3389（RDP）など

**340行目**: URL構築
- ポート番号をURLパスに埋め込む
- 例: `/shodan/port/80?key=...`

**用途:**
- 特定のポートに関するバナー情報をリアルタイムで取得
- ポート別のセキュリティ監視
- 特定のサービス（例: HTTPサーバー）の動向追跡
- firehoseよりもデータ量が少なく、特定の用途に最適

### 349-359行目: shodan_stream_alert() - アラートストリーム

```python
@mcp.tool()
async def shodan_stream_alert(api_key: Optional[str] = None, limit: Optional[int] = 10) -> list:
    """Stream banners for the networks monitored by your Shodan account. Returns up to 'limit' events."""
    key = get_api_key(api_key, api_type="stream")
    url = f"{SHODAN_STREAM_BASE}/shodan/alert?key={key}"
    limit_val = limit if limit is not None else 10
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as resp:
            resp.raise_for_status()
            return await _parse_sse_events(resp.aiter_bytes(), limit_val)
```

**詳細説明:**

**352行目**: URL構築
- `/shodan/alert`エンドポイント

**用途:**
- Shodanアカウントで監視しているネットワークのバナー情報を取得
- 自社ネットワークのリアルタイム監視
- セキュリティアラートの即時検知
- 注意: Shodanアカウントでアラート設定が必要

**ストリーミングAPIの共通特徴:**
1. **タイムアウトなし**: `timeout=None`で長時間接続を維持
2. **SSE形式**: Server-Sent Eventsプロトコルを使用
3. **limit制御**: 無限ストリームから必要な数だけイベントを取得
4. **非同期処理**: `async/await`で効率的なリソース利用
5. **自動クリーンアップ**: コンテキストマネージャーで接続を自動的に閉じる

---

## Trends APIツール群

Trends APIは、Shodanのデータを集計して統計情報を提供します。
特定の検索クエリに対するトップポート、組織、国などの情報を取得できます。

### 363-373行目: shodan_trends_top_ports() - トップポート統計

```python
@mcp.tool()
async def shodan_trends_top_ports(query: str, days: int = 30, api_key: Optional[str] = None) -> dict:
    """Get the top ports for a search query over a period of days."""
    key = get_api_key(api_key, api_type="trends")
    url = f"{SHODAN_TRENDS_BASE}/api/v1/top-ports"
    params = {"key": key, "query": query, "days": days}
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**364行目**: 関数定義
- `query: str`: 検索クエリ（必須）。Shodanの検索構文を使用
- `days: int = 30`: 集計期間（デフォルト: 30日）
- `api_key: Optional[str] = None`: APIキー（オプション）

**366行目**: APIキー取得
- `api_type="trends"`を指定（Trends API用）
- `SHODAN_TRENDS_API_KEY`環境変数が優先される

**367行目**: URL構築
- `/api/v1/top-ports`エンドポイント
- Trends APIはバージョン付きエンドポイント（v1）を使用

**368行目**: パラメータ構築
- `query`、`days`、`key`をすべて含める
- `days`はデフォルト値30を持つが、明示的に指定可能

**用途:**
- 特定の検索クエリに対する最も一般的なポートを取得
- 例: "country:US"で米国内の最も使用されているポートを確認
- トレンド分析、レポート作成、可視化に使用


### 375-385行目: shodan_trends_top_orgs() - トップ組織統計

```python
@mcp.tool()
async def shodan_trends_top_orgs(query: str, days: int = 30, api_key: Optional[str] = None) -> dict:
    """Get the top organizations for a search query over a period of days."""
    key = get_api_key(api_key, api_type="trends")
    url = f"{SHODAN_TRENDS_BASE}/api/v1/top-orgs"
    params = {"key": key, "query": query, "days": days}
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**379行目**: URL構築
- `/api/v1/top-orgs`エンドポイント

**用途:**
- 特定の検索クエリに対する最も一般的な組織（ISP、ホスティングプロバイダーなど）を取得
- 例: "product:nginx"でNginxを使用している主要な組織を確認
- インフラストラクチャの分布分析
- 競合分析やマーケットリサーチ

### 387-397行目: shodan_trends_top_countries() - トップ国統計

```python
@mcp.tool()
async def shodan_trends_top_countries(query: str, days: int = 30, api_key: Optional[str] = None) -> dict:
    """Get the top countries for a search query over a period of days."""
    key = get_api_key(api_key, api_type="trends")
    url = f"{SHODAN_TRENDS_BASE}/api/v1/top-countries"
    params = {"key": key, "query": query, "days": days}
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

**詳細説明:**

**391行目**: URL構築
- `/api/v1/top-countries`エンドポイント

**用途:**
- 特定の検索クエリに対する最も一般的な国を取得
- 例: "port:3389"でRDPポートが開いている国の分布を確認
- 地理的分布の分析
- グローバルなセキュリティトレンドの把握

**Trends APIの共通特徴:**
1. **集計期間**: `days`パラメータで集計期間を指定（デフォルト: 30日）
2. **検索クエリベース**: Shodanの検索構文を使用して対象を絞り込み
3. **統計情報**: カウント、パーセンテージなどの集計データを返す
4. **バージョン管理**: `/api/v1/`でAPIバージョンを明示
5. **専用APIキー**: `SHODAN_TRENDS_API_KEY`環境変数で専用キーを設定可能

---

## サーバー起動

### 399-400行目: メインエントリーポイント

```python
if __name__ == "__main__":
    mcp.run()
```

**詳細説明:**

**399行目**: メインモジュールチェック
- `if __name__ == "__main__"`: このスクリプトが直接実行された場合のみ実行
- モジュールとしてインポートされた場合は実行されない

**400行目**: MCPサーバー起動
- `mcp.run()`: FastMCPサーバーを起動
- すべての登録済みツール（`@mcp.tool()`デコレータを持つ関数）を公開
- MCPプロトコルでクライアントからの接続を待機

**起動方法:**
```bash
# 直接実行
python server.py

# uvを使用（推奨）
uv run server.py

# 特定のディレクトリから実行
uv --directory /path/to/shodan-mcp run server.py
```

**起動後の動作:**
1. FastMCPサーバーが起動
2. すべてのツール（約40個）がMCPプロトコルで公開される
3. MCPクライアント（Claude Desktop、CLIなど）からの接続を待機
4. クライアントからツール呼び出しを受信
5. 対応する非同期関数を実行
6. 結果をクライアントに返す

---

## 全体的な設計パターン

### 1. 非同期処理の一貫性

すべてのツール関数は`async def`で定義され、非同期処理を使用しています：

**利点:**
- **高い並行性**: 複数のAPI呼び出しを同時に処理可能
- **効率的なリソース利用**: I/O待機中に他のタスクを実行
- **スケーラビリティ**: 多数のクライアントリクエストを効率的に処理

### 2. エラーハンドリング戦略

エラーハンドリングは透過的で一貫しています：

**APIキーエラー:**
```python
# get_api_key()内で発生
raise ValueError("Shodan API key must be provided...")
```

**HTTPエラー:**
```python
# すべてのツールで使用
resp.raise_for_status()  # 4xx/5xxで例外発生
```

**利点:**
- **早期失敗**: 問題を早期に検出
- **明確なエラーメッセージ**: デバッグが容易
- **透過的な伝播**: エラーをクライアントに直接伝達

### 3. リソース管理

すべてのHTTPクライアントはコンテキストマネージャーで管理されています：

```python
async with httpx.AsyncClient() as client:
    # リクエスト処理
    # 自動的にクリーンアップ
```

**利点:**
- **自動クリーンアップ**: 接続が確実に閉じられる
- **リソースリーク防止**: メモリやソケットの適切な解放
- **例外安全性**: エラー発生時も確実にクリーンアップ

### 4. パラメータの柔軟性

すべてのツールは柔軟なパラメータ設計を採用しています：

```python
async def tool_name(
    required_param: str,              # 必須パラメータ
    api_key: Optional[str] = None,    # オプション（環境変数フォールバック）
    optional_param: Optional[type] = default  # オプション（デフォルト値あり）
) -> return_type:
```

**利点:**
- **使いやすさ**: 必須パラメータのみで呼び出し可能
- **柔軟性**: 詳細な制御が必要な場合はオプションパラメータを使用
- **環境変数統合**: APIキーをコードに埋め込む必要なし

### 5. 型ヒントの活用

すべての関数は型ヒントを使用しています：

```python
def get_api_key(api_key: Optional[str] = None, api_type: Optional[str] = None) -> str:
async def shodan_host_info(ip: str, ...) -> dict:
async def shodan_stream_firehose(...) -> list:
```

**利点:**
- **コードの可読性**: 期待される型が明確
- **IDE支援**: 自動補完とエラー検出
- **ドキュメント**: 型情報が自己文書化

---

## まとめ

`server.py`は、以下の特徴を持つ高品質なMCPサーバー実装です：

### 主要な特徴

1. **包括的なAPI カバレッジ**: Shodan APIの3つのカテゴリ（REST、Streaming、Trends）をすべてカバー
2. **非同期アーキテクチャ**: 高い並行性とスケーラビリティを実現
3. **柔軟な認証**: 環境変数と引数の優先順位ロジック
4. **堅牢なエラーハンドリング**: 早期失敗と明確なエラーメッセージ
5. **リソース管理**: コンテキストマネージャーによる自動クリーンアップ
6. **型安全性**: 型ヒントによる明確なインターフェース

### コード統計

- **総行数**: 約400行
- **ツール数**: 約40個
- **API カテゴリ**: 3つ（REST、Streaming、Trends）
- **エンドポイント数**: 30以上

### 使用技術

- **Python**: 3.10以上
- **httpx**: 非同期HTTPクライアント
- **FastMCP**: MCPサーバーフレームワーク
- **SSE**: Server-Sent Eventsプロトコル

### ユースケース

1. **セキュリティ研究**: 脅威ハンティング、資産発見、脆弱性調査
2. **自動化**: セキュリティパイプライン、SIEM統合、カスタムダッシュボード
3. **リアルタイム監視**: ライブバナーストリーム、アラート監視
4. **データ分析**: トレンド分析、レポート作成、可視化

このドキュメントが`server.py`の理解と活用に役立つことを願っています。
