---
title: "MCPサーバーのPython実装：基本的な動作の確認"
date: 2025-05-06
draft: false
---

# MCPサーバーのPython実装：基本的な動作の確認

## はじめに

前回の記事で実装したMCPサーバーの基本的な動作を確認しました。この記事では、サーバーの起動から初期化リクエストの処理までの流れを詳しく解説します。

## サーバーの起動

まず、以下のコマンドでサーバーを起動します：

```bash
uvicorn src.main:app --reload
```

このコマンドの各部分の意味：
- `uvicorn`: ASGIサーバー実装
- `src.main:app`: `src/main.py`の`app`オブジェクトを指定
- `--reload`: コード変更時に自動再読み込み（開発用）

起動すると以下のようなログが表示されます：

```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [43633] using StatReload
INFO:     Started server process [43635]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

## 初期化リクエストの送信

サーバーが起動したら、初期化リクエストを送信してみます：

```bash
curl -X POST "http://127.0.0.1:8000/" -H "Content-Type: application/json" -d '{
  "method": "initialize",
  "params": {
    "capabilities": {},
    "clientInfo": {
      "name": "Test Client",
      "version": "1.0.0"
    }
  }
}'
```

このリクエストの構造：
- `method`: "initialize"（初期化リクエストを示す）
- `params`: 
  - `capabilities`: クライアントの機能（今回は空）
  - `clientInfo`: クライアントの情報

## サーバーの応答

サーバーは以下のような応答を返します：

```json
{
  "protocolVersion": "2025-03-26",
  "capabilities": {
    "logging": {},
    "completions": {},
    "prompts": {
      "listChanged": true
    },
    "resources": {
      "subscribe": true,
      "listChanged": true
    },
    "tools": {
      "listChanged": true
    }
  },
  "serverInfo": {
    "name": "MCP Basic Server",
    "version": "1.0.0"
  },
  "instructions": "This is a basic MCP server implementation for learning purposes."
}
```

応答の各フィールドの意味：
- `protocolVersion`: MCPのプロトコルバージョン
- `capabilities`: サーバーが提供する機能
  - `logging`: ログ機能
  - `completions`: 補完機能
  - `prompts`: プロンプト管理機能
  - `resources`: リソース管理機能
  - `tools`: ツール管理機能
- `serverInfo`: サーバーの情報
- `instructions`: クライアントへの説明

## 内部の処理の流れ

1. リクエストの受信（`main.py`）:
   ```python
   @app.post("/")
   async def handle_request(request: Request) -> dict:
       data = await request.json()
       method = data.get("method")
   ```

2. リクエストの検証と変換（`models.py`）:
   ```python
   class InitializeRequest(BaseModel):
       method: str = "initialize"
       params: Dict[str, Any]
   ```

3. 初期化処理（`server.py`）:
   ```python
   async def handle_initialize(self, request: InitializeRequest) -> InitializeResult:
       result = InitializeResult(
           protocolVersion=self.protocol_version,
           capabilities=self.capabilities,
           serverInfo=self.server_info,
           instructions="..."
       )
   ```

## 設計のポイント

1. **非同期処理**:
   - FastAPIとUvicornによる効率的な非同期処理
   - `async/await`を使用した非同期関数の実装

2. **型安全性**:
   - Pydanticモデルによるリクエスト/レスポンスの検証
   - 型ヒントによるコードの安全性確保

3. **モジュール分割**:
   - `models.py`: データモデルの定義
   - `server.py`: ビジネスロジック
   - `main.py`: HTTPインターフェース

## 次のステップ

現在の実装は基本的な機能のみですが、以下の機能を追加する予定です：

1. エラーハンドリング
   - 不正なリクエストの処理
   - エラーレスポンスの標準化

2. ログ機能の強化
   - 構造化ログの導入
   - デバッグ情報の充実

3. テストの追加
   - ユニットテスト
   - 統合テスト

---

> **注記**
> 本記事はMCPサーバーの基本的な動作確認にフォーカスしていますが、現状の実装では logger.py（ログ機能）や errors.py（エラーハンドリング）などの拡張ファイルも既に存在します。
> これらの詳細については、今後の記事で順次解説していきます。

## まとめ

MCPサーバーの基本的な動作を確認し、以下の点が確認できました：

- FastAPIとUvicornによる効率的なサーバー実装
- Pydanticを使用した型安全なデータ処理
- 非同期処理による効率的なリクエスト処理

次の記事では、エラーハンドリングの実装について説明します。 