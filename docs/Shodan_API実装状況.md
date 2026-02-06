# Shodan API 実装状況ドキュメント

## 目次

1. [概要](#概要)
2. [REST API実装状況](#rest-api実装状況)
3. [Streaming API実装状況](#streaming-api実装状況)
4. [Trends API実装状況](#trends-api実装状況)
5. [Exploits API実装状況](#exploits-api実装状況)
6. [未実装機能の詳細](#未実装機能の詳細)
7. [実装推奨度](#実装推奨度)

---

## 概要

本ドキュメントは、Shodan MCP Serverの現在の実装状況を、[Shodan Developer API Reference](https://developer.shodan.io/api)と照らし合わせて分析したものです。

### 実装状況サマリー

| APIカテゴリ | 実装済み | 未実装 | 実装率 |
|-----------|---------|--------|--------|
| REST API | 24 | 9 | 73% |
| Streaming API | 3 | 4 | 43% |
| Trends API | 3 | 0 | 100% |
| Exploits API | 0 | 4 | 0% |
| **合計** | **30** | **17** | **64%** |

---

## REST API実装状況

### ✅ 実装済み機能（24個）

#### 1. Search Methods（検索メソッド）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/shodan/host/{ip}` | `shodan_host_info()` | ホスト情報取得 | ✅ 完全実装 |
| `/shodan/host/count` | `shodan_host_count()` | 検索結果カウント | ✅ 完全実装 |
| `/shodan/host/search` | `shodan_host_search()` | ホスト検索 | ✅ 完全実装 |
| `/shodan/host/search/facets` | `shodan_host_search_facets()` | 検索ファセット一覧 | ✅ 完全実装 |

**実装詳細:**
- すべてのオプションパラメータをサポート（history、minify、facets、page）
- 非同期実装で高パフォーマンス
- エラーハンドリング実装済み

#### 2. On-Demand Scanning（オンデマンドスキャン）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/shodan/ports` | `shodan_ports()` | クロール中ポート一覧 | ✅ 完全実装 |
| `/shodan/protocols` | `shodan_protocols()` | サポートプロトコル一覧 | ✅ 完全実装 |

#### 3. Network Alerts（ネットワークアラート）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| なし | なし | アラート管理機能 | ❌ 未実装 |

**注意:** Streaming APIの`shodan_stream_alert()`はアラートストリームを取得しますが、アラートの作成・管理機能は未実装です。

#### 4. Notifiers（通知機能）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| なし | なし | 通知プロバイダー管理 | ❌ 未実装 |

#### 5. Directory Methods（ディレクトリメソッド）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/shodan/query` | `shodan_query()` | 保存済みクエリ一覧 | ✅ 完全実装 |
| `/shodan/query/search` | `shodan_query_search()` | クエリディレクトリ検索 | ✅ 完全実装 |
| `/shodan/query/tags` | `shodan_query_tags()` | 人気タグ一覧 | ✅ 完全実装 |

#### 6. Bulk Data（バルクデータ）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/shodan/data` | `shodan_data()` | データセット一覧 | ✅ 完全実装 |
| `/shodan/data/{dataset}` | `shodan_data_dataset()` | データセット詳細 | ✅ 完全実装 |
| `/shodan/data/{dataset}/search` | `shodan_data_dataset_search()` | データセット内検索 | ✅ 完全実装 |

#### 7. Account Methods（アカウントメソッド）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/account/profile` | `shodan_account_profile()` | アカウントプロファイル | ✅ 完全実装 |
| `/api-info` | `shodan_api_info()` | APIプラン情報 | ✅ 完全実装 |

#### 8. DNS Methods（DNSメソッド）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/dns/resolve` | `shodan_dns_resolve()` | DNS正引き | ✅ 完全実装 |
| `/dns/reverse` | `shodan_dns_reverse()` | DNS逆引き | ✅ 完全実装 |

#### 9. Utility Methods（ユーティリティメソッド）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/tools/httpheaders` | `shodan_tools_httpheaders()` | HTTPヘッダー確認 | ✅ 完全実装 |
| `/tools/myip` | `shodan_tools_myip()` | 現在のIP取得 | ✅ 完全実装 |
| `/tools/ping` | `shodan_tools_ping()` | ホスト到達性確認 | ✅ 完全実装 |
| `/tools/ports` | `shodan_tools_ports()` | ポート一覧（ツール版） | ✅ 完全実装 |
| `/tools/protocols` | `shodan_tools_protocols()` | プロトコル一覧（ツール版） | ✅ 完全実装 |
| `/tools/uptime` | `shodan_tools_uptime()` | サーバー稼働時間 | ✅ 完全実装 |
| `/tools/whois` | `shodan_tools_whois()` | WHOIS情報取得 | ✅ 完全実装 |


### ❌ 未実装機能（9個）

#### 1. On-Demand Scanning（オンデマンドスキャン）

| エンドポイント | 説明 | 優先度 | 実装難易度 |
|--------------|------|--------|-----------|
| `POST /shodan/scan` | IPアドレスのスキャンをリクエスト | 高 | 中 |
| `GET /shodan/scan/{id}` | スキャンの進捗状況を確認 | 高 | 低 |
| `POST /shodan/scan/internet` | インターネット全体のスキャンをリクエスト | 中 | 中 |

**説明:**
- オンデマンドスキャン機能は、特定のIPアドレスやネットワークをShodanにスキャンさせる機能
- スキャンクレジットが必要（有料機能）
- 実装すると、リアルタイムでの脆弱性スキャンが可能

**実装例:**
```python
@mcp.tool()
async def shodan_scan(ips: str, api_key: Optional[str] = None) -> dict:
    """Request Shodan to crawl a network."""
    key = get_api_key(api_key, api_type="rest")
    data = {"ips": ips}
    url = f"{SHODAN_API_BASE}/shodan/scan"
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, params={"key": key}, json=data)
        resp.raise_for_status()
        return resp.json()
```

#### 2. Network Alerts（ネットワークアラート）

| エンドポイント | 説明 | 優先度 | 実装難易度 |
|--------------|------|--------|-----------|
| `GET /shodan/alert/info` | アラート一覧取得 | 高 | 低 |
| `POST /shodan/alert` | 新しいアラートを作成 | 高 | 中 |
| `DELETE /shodan/alert/{id}` | アラートを削除 | 中 | 低 |
| `GET /shodan/alert/{id}/info` | アラート詳細取得 | 中 | 低 |
| `PUT /shodan/alert/{id}` | アラートを更新 | 中 | 中 |

**説明:**
- ネットワークアラート機能は、特定のネットワークを監視し、変更があった場合に通知する機能
- 企業のセキュリティ監視に重要
- Streaming APIの`shodan_stream_alert()`と組み合わせて使用

**実装例:**
```python
@mcp.tool()
async def shodan_alert_list(api_key: Optional[str] = None) -> dict:
    """List all network alerts."""
    key = get_api_key(api_key, api_type="rest")
    url = f"{SHODAN_API_BASE}/shodan/alert/info"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params={"key": key})
        resp.raise_for_status()
        return resp.json()

@mcp.tool()
async def shodan_alert_create(name: str, ip: str, api_key: Optional[str] = None) -> dict:
    """Create a new network alert."""
    key = get_api_key(api_key, api_type="rest")
    data = {"name": name, "filters": {"ip": [ip]}}
    url = f"{SHODAN_API_BASE}/shodan/alert"
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, params={"key": key}, json=data)
        resp.raise_for_status()
        return resp.json()
```

#### 3. Notifiers（通知機能）

| エンドポイント | 説明 | 優先度 | 実装難易度 |
|--------------|------|--------|-----------|
| `GET /notifier` | 通知プロバイダー一覧 | 低 | 低 |
| `GET /notifier/provider` | 利用可能なプロバイダー一覧 | 低 | 低 |
| `POST /notifier` | 新しい通知プロバイダーを作成 | 低 | 中 |
| `DELETE /notifier/{id}` | 通知プロバイダーを削除 | 低 | 低 |
| `PUT /notifier/{id}` | 通知プロバイダーを更新 | 低 | 中 |

**説明:**
- 通知機能は、アラートが発生した際にSlack、Email、Webhookなどに通知する機能
- ネットワークアラートと組み合わせて使用
- 優先度は低いが、自動化には有用

---

## Streaming API実装状況

### ✅ 実装済み機能（3個）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/shodan/banners` | `shodan_stream_firehose()` | グローバルfirehose | ✅ 完全実装 |
| `/shodan/port/{port}` | `shodan_stream_ports()` | ポート別ストリーム | ✅ 完全実装 |
| `/shodan/alert` | `shodan_stream_alert()` | アラートストリーム | ✅ 完全実装 |

**実装詳細:**
- SSE（Server-Sent Events）形式のストリーミングを手動で解析
- `_parse_sse_events()`ヘルパー関数で効率的に処理
- `limit`パラメータで返すイベント数を制御
- タイムアウトなし（`timeout=None`）で長時間接続を維持

### ❌ 未実装機能（4個）

| エンドポイント | 説明 | 優先度 | 実装難易度 |
|--------------|------|--------|-----------|
| `/shodan/asn/{asn}` | ASN別ストリーム | 中 | 低 |
| `/shodan/countries/{countries}` | 国別ストリーム | 中 | 低 |
| `/shodan/ports/{ports}` | 複数ポートストリーム | 中 | 低 |
| `/shodan/vulns/{vulns}` | 脆弱性別ストリーム | 高 | 低 |

**説明:**

1. **ASN別ストリーム (`/shodan/asn/{asn}`)**
   - 特定のAS（Autonomous System）番号のバナーをストリーミング
   - 例: AS15169（Google）のすべてのバナー
   - 用途: 特定のISPやクラウドプロバイダーの監視

2. **国別ストリーム (`/shodan/countries/{countries}`)**
   - 特定の国のバナーをストリーミング
   - 例: "US,JP,CN"で米国、日本、中国のバナー
   - 用途: 地理的な脅威分析

3. **複数ポートストリーム (`/shodan/ports/{ports}`)**
   - 複数のポートのバナーをストリーミング
   - 例: "80,443,8080"でHTTP/HTTPS関連ポート
   - 用途: 特定のサービスの監視

4. **脆弱性別ストリーム (`/shodan/vulns/{vulns}`)**
   - 特定のCVEを持つバナーのみをストリーミング
   - 例: "CVE-2017-7679,CVE-2018-15919"
   - 用途: 特定の脆弱性の監視（最も重要）

**実装例（脆弱性別ストリーム）:**
```python
@mcp.tool()
async def shodan_stream_vulns(vulns: str, api_key: Optional[str] = None, limit: Optional[int] = 10) -> list:
    """Stream banners for specific vulnerabilities. Returns up to 'limit' events."""
    key = get_api_key(api_key, api_type="stream")
    url = f"{SHODAN_STREAM_BASE}/shodan/vulns/{vulns}?key={key}"
    limit_val = limit if limit is not None else 10
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as resp:
            resp.raise_for_status()
            return await _parse_sse_events(resp.aiter_bytes(), limit_val)
```

---

## Trends API実装状況

### ✅ 実装済み機能（3個）

| エンドポイント | 実装関数 | 説明 | 実装状況 |
|--------------|---------|------|---------|
| `/api/v1/top-ports` | `shodan_trends_top_ports()` | トップポート統計 | ✅ 完全実装 |
| `/api/v1/top-orgs` | `shodan_trends_top_orgs()` | トップ組織統計 | ✅ 完全実装 |
| `/api/v1/top-countries` | `shodan_trends_top_countries()` | トップ国統計 | ✅ 完全実装 |

**実装詳細:**
- すべてのTrends APIエンドポイントを実装
- `days`パラメータでデフォルト30日間の統計を取得
- 検索クエリベースで柔軟な統計分析が可能

### ❌ 未実装機能（0個）

Trends APIはすべて実装済みです。

---

## Exploits API実装状況

### ❌ 未実装機能（4個）

| エンドポイント | 説明 | 優先度 | 実装難易度 |
|--------------|------|--------|-----------|
| `GET /api/search` | エクスプロイト検索 | 中 | 低 |
| `GET /api/count` | エクスプロイトカウント | 低 | 低 |

**説明:**
- Exploits APIは、Shodanが収集した脆弱性とエクスプロイト情報を検索する機能
- CVE、Metasploit、ExploitDBなどのデータベースを統合
- セキュリティ研究に有用だが、Shodan本体のAPIとは別のサービス

**実装例:**
```python
SHODAN_EXPLOITS_BASE = "https://exploits.shodan.io"

@mcp.tool()
async def shodan_exploits_search(query: str, api_key: Optional[str] = None, facets: Optional[str] = None, page: Optional[int] = 1) -> dict:
    """Search for exploits."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query, "page": page}
    if facets:
        params["facets"] = facets
    url = f"{SHODAN_EXPLOITS_BASE}/api/search"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()

@mcp.tool()
async def shodan_exploits_count(query: str, api_key: Optional[str] = None, facets: Optional[str] = None) -> dict:
    """Count exploits matching the query."""
    key = get_api_key(api_key, api_type="rest")
    params = {"key": key, "query": query}
    if facets:
        params["facets"] = facets
    url = f"{SHODAN_EXPLOITS_BASE}/api/count"
    async with httpx.AsyncClient() as client:
        resp = await client.get(url, params=params)
        resp.raise_for_status()
        return resp.json()
```

---

## 未実装機能の詳細

### 1. オンデマンドスキャン機能

**重要度: 高**

**機能概要:**
- 特定のIPアドレスやネットワークをShodanにスキャンさせる
- スキャン結果をリアルタイムで取得
- スキャンクレジットが必要（有料）

**ユースケース:**
1. 自社ネットワークの脆弱性スキャン
2. 新しいサーバーのセキュリティチェック
3. 定期的なセキュリティ監査

**実装に必要なエンドポイント:**
- `POST /shodan/scan` - スキャンリクエスト
- `GET /shodan/scan/{id}` - スキャン進捗確認
- `POST /shodan/scan/internet` - インターネット全体スキャン

**実装の課題:**
- POSTリクエストの実装（現在はGETのみ）
- スキャンIDの管理
- 非同期スキャンの進捗追跡

### 2. ネットワークアラート機能

**重要度: 高**

**機能概要:**
- 特定のネットワークを監視
- 変更があった場合に通知
- Streaming APIと連携

**ユースケース:**
1. 自社ネットワークの監視
2. 新しいサービスの検出
3. 脆弱性の早期発見

**実装に必要なエンドポイント:**
- `GET /shodan/alert/info` - アラート一覧
- `POST /shodan/alert` - アラート作成
- `DELETE /shodan/alert/{id}` - アラート削除
- `GET /shodan/alert/{id}/info` - アラート詳細
- `PUT /shodan/alert/{id}` - アラート更新

**実装の課題:**
- POST/PUT/DELETEリクエストの実装
- アラート設定の複雑性
- 通知機能との連携

### 3. 脆弱性別ストリーム

**重要度: 高**

**機能概要:**
- 特定のCVEを持つバナーのみをストリーミング
- リアルタイムで脆弱性を監視
- 帯域幅を節約

**ユースケース:**
1. 特定の脆弱性の監視
2. ゼロデイ脆弱性の追跡
3. パッチ適用の優先順位付け

**実装に必要なエンドポイント:**
- `GET /shodan/vulns/{vulns}` - 脆弱性別ストリーム

**実装の課題:**
- CVE IDのバリデーション
- 複数CVEの処理
- ストリーミングデータの解析

### 4. Exploits API

**重要度: 中**

**機能概要:**
- エクスプロイトとCVE情報の検索
- Metasploit、ExploitDBなどのデータベース統合
- 脆弱性情報の詳細取得

**ユースケース:**
1. 脆弱性情報の調査
2. エクスプロイトの可用性確認
3. セキュリティレポート作成

**実装に必要なエンドポイント:**
- `GET /api/search` - エクスプロイト検索
- `GET /api/count` - エクスプロイトカウント

**実装の課題:**
- 別のベースURL（exploits.shodan.io）
- 異なるレスポンス形式
- APIキーの互換性


---

## 実装推奨度

### 優先度: 高（すぐに実装すべき）

#### 1. 脆弱性別ストリーム (`/shodan/vulns/{vulns}`)

**理由:**
- セキュリティ監視に最も重要
- 実装が簡単（既存のストリーミングコードを流用可能）
- 帯域幅を大幅に節約
- ゼロデイ脆弱性の追跡に不可欠

**実装工数:** 1-2時間

**実装例:**
```python
@mcp.tool()
async def shodan_stream_vulns(vulns: str, api_key: Optional[str] = None, limit: Optional[int] = 10) -> list:
    """Stream banners for specific vulnerabilities (comma-separated CVE IDs).
    
    Example: vulns="CVE-2017-7679,CVE-2018-15919"
    """
    key = get_api_key(api_key, api_type="stream")
    url = f"{SHODAN_STREAM_BASE}/shodan/vulns/{vulns}?key={key}"
    limit_val = limit if limit is not None else 10
    async with httpx.AsyncClient(timeout=None) as client:
        async with client.stream("GET", url) as resp:
            resp.raise_for_status()
            return await _parse_sse_events(resp.aiter_bytes(), limit_val)
```

#### 2. ネットワークアラート管理 (`/shodan/alert/*`)

**理由:**
- 企業のセキュリティ監視に必須
- Streaming APIの`shodan_stream_alert()`と連携
- 自動化されたセキュリティ監視を実現

**実装工数:** 4-6時間

**必要な機能:**
- アラート一覧取得
- アラート作成
- アラート削除
- アラート詳細取得
- アラート更新

#### 3. オンデマンドスキャン (`/shodan/scan`)

**理由:**
- リアルタイムでの脆弱性スキャンが可能
- 自社ネットワークのセキュリティチェックに有用
- スキャン結果の追跡が可能

**実装工数:** 3-4時間

**必要な機能:**
- スキャンリクエスト（POST）
- スキャン進捗確認（GET）

### 優先度: 中（時間があれば実装）

#### 4. 国別・ASN別ストリーム

**理由:**
- 地理的な脅威分析に有用
- 特定のISPやクラウドプロバイダーの監視
- 実装が簡単

**実装工数:** 2-3時間

**必要な機能:**
- `/shodan/asn/{asn}` - ASN別ストリーム
- `/shodan/countries/{countries}` - 国別ストリーム
- `/shodan/ports/{ports}` - 複数ポートストリーム

#### 5. Exploits API

**理由:**
- 脆弱性情報の詳細取得
- エクスプロイトの可用性確認
- セキュリティレポート作成に有用

**実装工数:** 2-3時間

**必要な機能:**
- `/api/search` - エクスプロイト検索
- `/api/count` - エクスプロイトカウント

### 優先度: 低（必要に応じて実装）

#### 6. 通知機能 (`/notifier/*`)

**理由:**
- アラート機能の補完
- Slack、Email、Webhookへの通知
- 自動化には有用だが、外部サービスで代替可能

**実装工数:** 3-4時間

**必要な機能:**
- 通知プロバイダー一覧
- 通知プロバイダー作成
- 通知プロバイダー削除
- 通知プロバイダー更新

---

## 実装ロードマップ

### フェーズ1: 高優先度機能（1-2週間）

1. **脆弱性別ストリーム** - 1-2時間
   - 実装が簡単で効果が大きい
   - 既存のストリーミングコードを流用

2. **ネットワークアラート管理** - 4-6時間
   - POST/PUT/DELETEリクエストの実装
   - アラート設定のバリデーション

3. **オンデマンドスキャン** - 3-4時間
   - POSTリクエストの実装
   - スキャン進捗の追跡

### フェーズ2: 中優先度機能（1週間）

4. **国別・ASN別ストリーム** - 2-3時間
   - 地理的な脅威分析
   - 特定のISP監視

5. **Exploits API** - 2-3時間
   - 脆弱性情報の詳細取得
   - エクスプロイト検索

### フェーズ3: 低優先度機能（必要に応じて）

6. **通知機能** - 3-4時間
   - Slack、Email、Webhook統合
   - アラート通知の自動化

---

## 実装時の注意事項

### 1. HTTPメソッドの拡張

現在の実装はGETリクエストのみです。未実装機能の多くはPOST/PUT/DELETEを使用します。

**必要な変更:**
```python
# POST リクエストの例
async def shodan_scan(ips: str, api_key: Optional[str] = None) -> dict:
    key = get_api_key(api_key, api_type="rest")
    data = {"ips": ips}
    url = f"{SHODAN_API_BASE}/shodan/scan"
    async with httpx.AsyncClient() as client:
        resp = await client.post(url, params={"key": key}, json=data)
        resp.raise_for_status()
        return resp.json()

# DELETE リクエストの例
async def shodan_alert_delete(alert_id: str, api_key: Optional[str] = None) -> dict:
    key = get_api_key(api_key, api_type="rest")
    url = f"{SHODAN_API_BASE}/shodan/alert/{alert_id}"
    async with httpx.AsyncClient() as client:
        resp = await client.delete(url, params={"key": key})
        resp.raise_for_status()
        return resp.json()
```

### 2. エラーハンドリングの強化

未実装機能の多くは、より複雑なエラー処理が必要です。

**推奨事項:**
- スキャンクレジット不足のエラー処理
- アラート作成時のバリデーションエラー
- レート制限エラーの適切な処理

### 3. 入力検証の追加

新しい機能には、より厳密な入力検証が必要です。

**推奨事項:**
- CVE IDの形式検証（CVE-YYYY-NNNNN）
- IPアドレス範囲の検証
- アラート設定の検証

### 4. ドキュメントの更新

新機能を実装する際は、以下のドキュメントを更新してください。

**更新が必要なドキュメント:**
- `README.md` - 機能一覧
- `requirements.md` - 要件定義
- `design.md` - 設計書
- `tasks.md` - 実装タスク
- `server_py_詳細説明.md` - コード説明

---

## まとめ

### 現在の実装状況

**強み:**
- REST APIの主要機能（検索、DNS、アカウント、ツール）は完全実装
- Streaming APIの基本機能（firehose、ポート別、アラート）は実装済み
- Trends APIは100%実装済み
- 非同期実装で高パフォーマンス
- エラーハンドリングが適切

**弱み:**
- オンデマンドスキャン機能が未実装
- ネットワークアラート管理が未実装
- 脆弱性別ストリームが未実装（最も重要）
- Exploits APIが未実装
- POST/PUT/DELETEリクエストが未実装

### 推奨される次のステップ

1. **すぐに実装すべき:**
   - 脆弱性別ストリーム（1-2時間）
   - ネットワークアラート管理（4-6時間）
   - オンデマンドスキャン（3-4時間）

2. **時間があれば実装:**
   - 国別・ASN別ストリーム（2-3時間）
   - Exploits API（2-3時間）

3. **必要に応じて実装:**
   - 通知機能（3-4時間）

### 総合評価

現在の実装は、Shodan APIの**64%**をカバーしており、基本的な機能は十分に実装されています。
ただし、セキュリティ監視に重要な機能（脆弱性別ストリーム、ネットワークアラート）が未実装のため、
これらを優先的に実装することを強く推奨します。

**実装完了後の予想カバレッジ:**
- フェーズ1完了後: 約**85%**
- フェーズ2完了後: 約**95%**
- フェーズ3完了後: 約**100%**

このドキュメントが、Shodan MCP Serverの機能拡張に役立つことを願っています。
