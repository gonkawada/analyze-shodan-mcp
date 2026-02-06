# 要件定義書

## はじめに

本システムは、Shodan APIの全機能（REST API、Streaming API、Trends API）をMCPツールとして公開する非同期MCPサーバーです。Python、httpx、MCP Python SDKを使用して実装され、セキュリティ研究、自動化、リアルタイム監視、データ分析などのユースケースに対応します。

## 用語集

- **MCP_Server**: Model Context Protocol サーバー。クライアントアプリケーションにツールを提供するサーバー
- **Shodan_API**: インターネット接続デバイスを検索するためのShodanサービスのAPI
- **REST_API**: Shodanの標準的なHTTP APIエンドポイント群
- **Streaming_API**: Shodanのリアルタイムイベントストリーミングエンドポイント群
- **Trends_API**: Shodanの統計・トレンド分析エンドポイント群
- **API_Key**: Shodan APIへのアクセスを認証するための秘密鍵
- **SSE**: Server-Sent Events。サーバーからクライアントへの一方向リアルタイム通信プロトコル
- **Async_Client**: 非同期HTTPクライアント（httpxライブラリを使用）

## 要件

### 要件1: API認証管理

**ユーザーストーリー:** セキュリティ研究者として、複数のShodan APIキーを安全に管理したいので、環境変数を通じて認証情報を提供できるようにする

#### 受入基準

1. WHEN APIキーが関数引数として明示的に提供された場合、THE System SHALL その引数のキーを使用する
2. WHEN APIキーが引数として提供されず、かつapi_typeが"stream"の場合、THE System SHALL SHODAN_STREAM_API_KEY環境変数を確認する
3. WHEN APIキーが引数として提供されず、かつapi_typeが"trends"の場合、THE System SHALL SHODAN_TRENDS_API_KEY環境変数を確認する
4. WHEN 特定のAPI種別用の環境変数が設定されていない場合、THE System SHALL SHODAN_API_KEY環境変数にフォールバックする
5. WHEN すべての環境変数が設定されておらず、かつ引数も提供されていない場合、THE System SHALL ValueErrorを発生させる

### 要件2: REST APIツール - ホスト情報

**ユーザーストーリー:** セキュリティアナリストとして、特定のIPアドレスに関する詳細情報を取得したいので、ホスト情報検索機能を使用する

#### 受入基準

1. WHEN 有効なIPアドレスが提供された場合、THE System SHALL そのホストで発見されたすべてのサービス情報を返す
2. WHERE historyオプションが有効な場合、THE System SHALL 過去のスキャン履歴を含める
3. WHERE minifyオプションが有効な場合、THE System SHALL 最小化されたレスポンスを返す
4. WHEN Shodan APIがエラーを返す場合、THE System SHALL HTTPステータスエラーを発生させる

### 要件3: REST APIツール - 検索機能

**ユーザーストーリー:** 脅威ハンターとして、特定の条件に一致するホストを検索したいので、検索クエリを実行できるようにする

#### 受入基準

1. WHEN 検索クエリが提供された場合、THE System SHALL Shodanの検索構文を使用して最大100件の結果を返す
2. WHERE facetsパラメータが指定された場合、THE System SHALL ファセット情報を含める
3. WHERE pageパラメータが指定された場合、THE System SHALL 指定されたページの結果を返す
4. WHERE minifyオプションが有効な場合、THE System SHALL 最小化されたレスポンスを返す
5. WHEN 検索カウントのみが必要な場合、THE System SHALL 結果なしで総数とファセット情報のみを返す

### 要件4: REST APIツール - DNS解決

**ユーザーストーリー:** ネットワーク管理者として、ホスト名とIPアドレス間の変換を行いたいので、DNS解決機能を使用する

#### 受入基準

1. WHEN カンマ区切りのホスト名リストが提供された場合、THE System SHALL 各ホスト名のIPアドレスを返す
2. WHEN カンマ区切りのIPアドレスリストが提供された場合、THE System SHALL 各IPアドレスのホスト名を返す（逆引き）

### 要件5: REST APIツール - アカウント情報

**ユーザーストーリー:** ユーザーとして、自分のShodanアカウントの状態を確認したいので、アカウント情報とAPIプラン情報を取得できるようにする

#### 受入基準

1. WHEN APIキーが提供された場合、THE System SHALL そのキーに関連するAPIプラン情報を返す
2. WHEN アカウントプロファイルが要求された場合、THE System SHALL APIキーにリンクされたShodanアカウント情報を返す

### 要件6: REST APIツール - データセット管理

**ユーザーストーリー:** データサイエンティストとして、Shodanのデータセットにアクセスしたいので、利用可能なデータセットを一覧表示し検索できるようにする

#### 受入基準

1. WHEN データセット一覧が要求された場合、THE System SHALL ダウンロード可能なすべてのデータセットを返す
2. WHEN 特定のデータセット名が提供された場合、THE System SHALL そのデータセットの詳細情報を返す
3. WHEN データセット内検索クエリが提供された場合、THE System SHALL 指定されたデータセット内で検索を実行する

### 要件7: REST APIツール - ユーティリティ機能

**ユーザーストーリー:** 開発者として、Shodanのメタデータとユーティリティ情報にアクセスしたいので、各種ツールエンドポイントを使用する

#### 受入基準

1. WHEN HTTPヘッダー情報が要求された場合、THE System SHALL クライアントが送信するHTTPヘッダーを返す
2. WHEN 現在のIPアドレスが要求された場合、THE System SHALL インターネットから見えるクライアントのIPアドレスを返す
3. WHEN ポート一覧が要求された場合、THE System SHALL Shodanがクロールしているすべてのポートを返す
4. WHEN プロトコル一覧が要求された場合、THE System SHALL オンデマンドスキャンで使用可能なすべてのプロトコルを返す
5. WHEN 検索ファセット一覧が要求された場合、THE System SHALL 検索時に使用可能なすべてのファセットを返す
6. WHEN 保存済みクエリが要求された場合、THE System SHALL ユーザーの保存済み検索クエリを返す
7. WHEN クエリ検索が要求された場合、THE System SHALL 保存済みクエリディレクトリ内を検索する
8. WHEN クエリタグが要求された場合、THE System SHALL 最も人気のあるタグを返す
9. WHEN pingが要求された場合、THE System SHALL 指定されたホストがShodanサーバーから到達可能かを確認する
10. WHEN uptimeが要求された場合、THE System SHALL Shodanサーバーの稼働時間を返す
11. WHEN WHOIS情報が要求された場合、THE System SHALL 指定されたドメインのWHOIS情報を返す

### 要件8: Streaming APIツール

**ユーザーストーリー:** セキュリティ監視担当者として、リアルタイムでインターネットイベントを監視したいので、ストリーミングAPIを使用する

#### 受入基準

1. WHEN firehoseストリームが要求された場合、THE System SHALL Shodanが収集するすべてのデータのグローバルストリームを返す
2. WHEN 特定のポートストリームが要求された場合、THE System SHALL 指定されたポートのバナー情報をリアルタイムで返す
3. WHEN アラートストリームが要求された場合、THE System SHALL ユーザーアカウントで監視されているネットワークのバナー情報を返す
4. WHEN limitパラメータが指定された場合、THE System SHALL 指定された数のイベントまでを返す
5. WHEN ストリーミング接続が確立された場合、THE System SHALL SSE形式のデータを解析する

### 要件9: Trends APIツール

**ユーザーストーリー:** データアナリストとして、インターネットのトレンドを分析したいので、統計情報を取得できるようにする

#### 受入基準

1. WHEN トップポートが要求された場合、THE System SHALL 指定された検索クエリと期間に対するトップポートを返す
2. WHEN トップ組織が要求された場合、THE System SHALL 指定された検索クエリと期間に対するトップ組織を返す
3. WHEN トップ国が要求された場合、THE System SHALL 指定された検索クエリと期間に対するトップ国を返す
4. WHERE daysパラメータが指定されていない場合、THE System SHALL デフォルトで30日間を使用する

### 要件10: 非同期処理

**ユーザーストーリー:** システムアーキテクトとして、高いパフォーマンスと並行性を実現したいので、すべてのAPI呼び出しを非同期で実行する

#### 受入基準

1. WHEN 任意のAPIツールが呼び出された場合、THE System SHALL 非同期関数として実行する
2. WHEN HTTPリクエストが発行される場合、THE System SHALL httpx.AsyncClientを使用する
3. WHEN ストリーミング接続が確立される場合、THE System SHALL 非同期イテレーションを使用してデータを処理する

### 要件11: エラーハンドリング

**ユーザーストーリー:** 開発者として、API呼び出しの失敗を適切に処理したいので、明確なエラーメッセージを受け取る

#### 受入基準

1. WHEN HTTPリクエストが失敗した場合、THE System SHALL HTTPステータスエラーを発生させる
2. WHEN APIキーが提供されず環境変数も設定されていない場合、THE System SHALL 説明的なValueErrorを発生させる
3. WHEN Shodan APIがエラーレスポンスを返す場合、THE System SHALL そのエラーを呼び出し元に伝播させる

### 要件12: MCPサーバー統合

**ユーザーストーリー:** ユーザーとして、Claude DesktopやCLIからShodan機能にアクセスしたいので、MCPプロトコルを通じてツールを公開する

#### 受入基準

1. WHEN サーバーが起動した場合、THE System SHALL FastMCPを使用してMCPサーバーを初期化する
2. WHEN 各APIツールが定義された場合、THE System SHALL @mcp.tool()デコレータを使用してMCPツールとして登録する
3. WHEN サーバーがメインモジュールとして実行された場合、THE System SHALL mcp.run()を呼び出してサーバーを起動する
