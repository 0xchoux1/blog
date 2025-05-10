---
title: "MCPサーバーのPython実装：エラーハンドリングの設計と実装"
date: 2025-05-06
draft: false
---

# MCPサーバーのPython実装：エラーハンドリングの設計と実装

## はじめに

前回の記事で基本的な動作を確認したMCPサーバーに、エラーハンドリング機能を追加しました。この記事では、エラーハンドリングの設計と実装について説明します。

## エラーハンドリングの必要性

MCPサーバーでは、以下のような様々なエラーが発生する可能性があります：

1. **リクエスト関連のエラー**
   - 不正なJSONフォーマット
   - 必須フィールドの欠落
   - 不正なメソッド名
   - 不正なパラメータ型

2. **サーバー内部のエラー**
   - 処理中の例外
   - リソースの不足
   - タイムアウト

3. **プロトコル関連のエラー**
   - 未実装のメソッド
   - 不正なプロトコルバージョン
   - 互換性の問題

## エラーレスポンスの実装

`errors.py`でエラー関連のクラスを定義しました：

```python
class ErrorCode(IntEnum):
    # リクエスト関連のエラー (-32000 から -32099)
    INVALID_REQUEST = -32000
    METHOD_NOT_FOUND = -32001
    INVALID_PARAMS = -32002
    
    # サーバー内部のエラー (-32100 から -32199)
    INTERNAL_ERROR = -32100
    RESOURCE_EXHAUSTED = -32101
    TIMEOUT = -32102
    
    # プロトコル関連のエラー (-32200 から -32299)
    PROTOCOL_ERROR = -32200
    VERSION_MISMATCH = -32201

class ErrorInfo(BaseModel):
    code: int
    message: str
    data: Optional[Dict[str, Any]] = None

class ErrorResponse(BaseModel):
    error: ErrorInfo
    id: Optional[str] = None
```

## エラーハンドリングの実装

1. **カスタム例外クラス**
   ```python
   class MCPError(Exception):
       def __init__(self, code: ErrorCode, message: str, data: Optional[Dict] = None):
           self.code = code
           self.message = message
           self.data = data

   class RequestError(MCPError):
       pass

   class ServerError(MCPError):
       pass

   class ProtocolError(MCPError):
       pass
   ```

2. **グローバルエラーハンドラ**
   ```python
   @app.exception_handler(MCPError)
   async def mcp_exception_handler(request: Request, exc: MCPError) -> JSONResponse:
       return JSONResponse(
           status_code=200,
           content=ErrorResponse(
               error=ErrorInfo(
                   code=exc.code,
                   message=exc.message,
                   data=exc.data
               )
           ).model_dump()
       )
   ```

## テストの実装

`test_error_handling.py`で以下のテストケースを実装しました：

1. **不正なJSONフォーマット**
   ```python
   def test_invalid_json():
       response = client.post("/", content=b"invalid json")
       assert response.status_code == 200
       data = response.json()
       assert data["error"]["code"] == -32000
       assert "Invalid JSON format" in data["error"]["message"]
   ```

2. **メソッドの欠落**
   ```python
   def test_missing_method():
       response = client.post("/", json={})
       assert response.status_code == 200
       data = response.json()
       assert data["error"]["code"] == -32000
       assert "Method is required" in data["error"]["message"]
   ```

3. **不正なメソッド**
   ```python
   def test_invalid_method():
       response = client.post("/", json={
           "method": "nonexistent",
           "params": {}
       })
       assert response.status_code == 200
       data = response.json()
       assert data["error"]["code"] == -32001
       assert "Method 'nonexistent' not found" in data["error"]["message"]
   ```

4. **不正な初期化パラメータ**
   ```python
   def test_invalid_initialize_params():
       # capabilitiesが辞書でない場合
       response = client.post("/", json={
           "method": "initialize",
           "params": {
               "capabilities": "not a dict",
               "clientInfo": {
                   "name": "Test Client",
                   "version": "1.0.0"
               }
           }
       })
       assert response.status_code == 200
       data = response.json()
       assert data["error"]["code"] == -32002
       assert "Client capabilities must be a dictionary" in data["error"]["message"]
   ```

5. **GETリクエストの処理**
   ```python
   def test_get_request():
       response = client.get("/")
       assert response.status_code == 200
       data = response.json()
       assert data["error"]["code"] == -32000
       assert "This server only accepts POST requests" in data["error"]["message"]
   ```

## 実装のポイント

1. **エラーの階層化**
   - 基本エラー（`MCPError`）
   - リクエストエラー（`RequestError`）
   - サーバーエラー（`ServerError`）
   - プロトコルエラー（`ProtocolError`）

2. **エラーレスポンスの標準化**
   - MCPプロトコルに準拠した形式
   - 一貫したエラーコード体系
   - 詳細なエラーメッセージ

3. **テストの網羅性**
   - 各エラーケースのテスト
   - エラーレスポンスの形式確認
   - エンドツーエンドのテスト

## 次のステップ

1. ログ機能の実装
   - エラーログの構造化
   - デバッグ情報の付加
   - ログレベルの設定

2. エラーハンドリングの拡張
   - より詳細なエラー情報
   - エラーの追跡機能
   - エラー統計の収集

## まとめ

エラーハンドリングの実装では、以下の点に注意を払いました：

- MCPプロトコルに準拠したエラーレスポンス
- 階層化されたエラー処理
- 網羅的なテストケース
- 拡張可能な設計

次の記事では、ログ機能の実装について説明します。

> **注記**
> 本記事はエラーハンドリングの設計と実装にフォーカスしていますが、現状の実装では logger.py（ログ機能）も既に存在します。
> ログ機能の詳細については、次回以降の記事で順次解説していきます。 