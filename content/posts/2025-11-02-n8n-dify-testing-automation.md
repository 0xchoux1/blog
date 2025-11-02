---
title: "n8nとDifyの自動テスト環境構築：Playwright + pytestで実現する3層テスト戦略"
date: 2025-11-02T20:00:00+09:00
draft: false
tags: ["n8n", "Dify", "Playwright", "pytest", "E2E", "Testing", "CI/CD", "Docker"]
categories: ["技術"]
series: ["AI開発環境"]
---

## はじめに

前回の記事では、n8nとDifyのDocker環境構築について解説しました。今回は、その環境に対する**自動テスト戦略**を構築します。

ブラウザ確認だけでは不十分な理由：
- 手動確認は時間がかかる
- 再現性が低い
- CI/CDに組み込めない
- 回帰テストが困難

本記事では、**Playwright + pytest**を使った3層テスト戦略で、これらの課題を解決します。

## テストピラミッド：3層の戦略

```
        /\
       /  \      E2E Tests (Playwright)
      /____\     - UI自動化
     /      \    - ワークフロー確認
    /        \
   /   統合   \  Integration Tests
  /____________\ - サービス間連携
 /              \
/    API Tests   \ API Level Tests (最優先)
/________________\- ヘルスチェック (0.8秒)
```

### レイヤー1: API Tests（最速・最重要）

**目的：** サービスが起動しているか即座に確認

```python
@pytest.mark.api
@pytest.mark.smoke
def test_n8n_health(sync_http_client, n8n_url):
    """n8nが正常に起動しているか確認"""
    response = sync_http_client.get(n8n_url)
    assert response.status_code in [200, 401]

@pytest.mark.api
@pytest.mark.smoke
def test_weaviate_health(sync_http_client, weaviate_url):
    """Weaviateベクトルデータベースが準備完了か確認"""
    response = sync_http_client.get(f"{weaviate_url}/v1/.well-known/ready")
    assert response.status_code == 200
```

**実行速度：**
```bash
$ pytest -m smoke
7 passed in 0.81s
```

### レイヤー2: Integration Tests（将来実装）

サービス間の連携を確認：
- n8n → Dify API連携
- ワークフロー実行結果の検証

### レイヤー3: E2E Tests（Playwright）

**実際のブラウザ操作を自動化**

```python
@pytest.mark.e2e
@pytest.mark.asyncio
async def test_n8n_homepage_loads(page, n8n_base_url):
    """n8nのホームページが読み込まれることを確認"""
    await page.goto(n8n_base_url, timeout=30000)
    await page.wait_for_load_state("networkidle")
    
    title = await page.title()
    assert "n8n" in title.lower() or len(title) > 0
    
    print(f"✓ n8n page loaded: {title}")
```

## 環境構築：ステップバイステップ

### 1. 依存関係のインストール

```bash
# Python仮想環境の作成
python3 -m venv venv
source venv/bin/activate

# テスト用ライブラリのインストール
pip install pytest pytest-asyncio pytest-cov
pip install httpx requests
pip install playwright psycopg2-binary redis

# Playwrightブラウザのインストール
playwright install chromium
```

### 2. pytest設定

**pytest.ini:**
```ini
[pytest]
testpaths = tests
python_files = test_*.py
addopts = -v --strict-markers --tb=short
markers =
    api: API level tests
    e2e: End-to-end tests
    smoke: Smoke tests for quick validation
    slow: Tests that take longer to run
asyncio_mode = auto
```

### 3. Playwrightフィクスチャの設定

**重要なポイント：session-scopedイベントループ**

```python
# tests/e2e/conftest.py
import pytest
import pytest_asyncio
import asyncio
from playwright.async_api import async_playwright

@pytest.fixture(scope="session")
def event_loop():
    """セッション全体で共有するイベントループ"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()

@pytest_asyncio.fixture(scope="session")
async def playwright_instance():
    """Playwrightインスタンス"""
    async with async_playwright() as p:
        yield p

@pytest_asyncio.fixture(scope="session")
async def browser(playwright_instance):
    """ブラウザインスタンス"""
    browser = await playwright_instance.chromium.launch(headless=True)
    yield browser
    await browser.close()

@pytest_asyncio.fixture
async def page(context):
    """各テストで新しいページを作成"""
    page = await context.new_page()
    yield page
    await page.close()
```

**よくあるエラーと解決方法：**

❌ `AttributeError: 'async_generator' object has no attribute 'goto'`
- 原因：asyncフィクスチャが正しく初期化されていない
- 解決：`@pytest_asyncio.fixture`を使用し、session-scopedイベントループを追加

## 実践：Dify初期セットアップの自動化

Difyを初めて起動すると、データベースエラーが発生することがあります。

### トラブルシューティング実例

**問題：Internal Server Error**

```bash
$ curl http://localhost:3000
# → "Internal Server Error"

$ docker compose logs dify-api
# → sqlalchemy.exc.ProgrammingError: relation "dify_setups" does not exist
```

**解決：データベースマイグレーション**

```bash
# マイグレーション実行
docker compose exec dify-api flask db upgrade

# サービス再起動
docker compose restart dify-api dify-web dify-worker

# 確認
curl -s http://localhost:3000 | grep "Setting up"
# → "Setting up an admin account" が表示されればOK
```

### セットアップ自動化

```python
@pytest.mark.e2e
@pytest.mark.asyncio
async def test_dify_initial_setup(page, dify_base_url):
    """Dify初期セットアップを自動化"""
    await page.goto(dify_base_url, timeout=30000)
    await page.wait_for_load_state("networkidle")
    await page.wait_for_timeout(2000)  # React描画を待つ
    
    if "/install" in page.url:
        # メールアドレス入力
        email_input = page.locator('input[type="email"]').first
        await email_input.fill("admin@example.com")
        
        # 名前入力
        name_input = page.locator('input[type="text"]').first
        await name_input.fill("Admin User")
        
        # パスワード入力
        password_input = page.locator('input[type="password"]').first
        await password_input.fill("Admin123456!")
        
        # セットアップボタンをクリック
        submit_button = page.locator('button:has-text("Set up")').first
        await submit_button.click()
        
        await page.wait_for_timeout(3000)
        await page.wait_for_load_state("networkidle")
        
        # /appsにリダイレクトされれば成功
        assert "/install" not in page.url
        assert "/apps" in page.url
        print("✓ Setup completed successfully!")
```

**実行結果：**
```bash
$ pytest tests/e2e/test_dify_setup.py -v -s
=== Starting Dify Setup ===
Current URL: http://localhost:3000/install
On install page, proceeding with setup...
Found email input
Found name input
Found 1 password inputs
Found submit button, clicking...
After submit URL: http://localhost:3000/apps
✓ Setup completed successfully!
PASSED
```

## ヘルスチェックスクリプト

シンプルで高速なシェルスクリプトも作成：

```bash
#!/bin/bash
# scripts/health_check.sh

check_service() {
    local name=$1
    local url=$2
    local expected_codes=$3
    
    http_code=$(curl -L -s -o /dev/null -w "%{http_code}" "$url")
    
    if [[ $expected_codes == *"$http_code"* ]]; then
        echo -e "${GREEN}✓${NC} $name: HTTP $http_code"
        return 0
    else
        echo -e "${RED}✗${NC} $name: HTTP $http_code"
        return 1
    fi
}

check_service "n8n" "http://localhost:5678" "200 401"
check_service "Dify Web" "http://localhost:3000" "200"
check_service "Dify API" "http://localhost:5001" "200 404"
check_service "Weaviate" "http://localhost:8080/v1/.well-known/ready" "200"
```

**実行：**
```bash
$ ./scripts/health_check.sh
=== Service Health Check ===

Checking n8n... ✓ HTTP 200
Checking Dify Web... ✓ HTTP 200
Checking Dify API... ✓ HTTP 200
Checking Weaviate Ready... ✓ HTTP 200

All services are healthy!
```

## CI/CD統合

GitHub Actionsでの自動テスト例：

```yaml
name: Test Workflow

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Start Docker services
      run: docker compose up -d
    
    - name: Wait for services
      run: sleep 30
    
    - name: Run health check
      run: ./scripts/health_check.sh
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.12'
    
    - name: Install dependencies
      run: |
        python -m venv venv
        source venv/bin/activate
        pip install -r requirements.txt
    
    - name: Run smoke tests
      run: |
        source venv/bin/activate
        pytest -m smoke --cov=. --cov-report=xml
    
    - name: Run E2E tests
      run: |
        source venv/bin/activate
        pytest -m e2e -v
```

## テスト実行パターン

### 開発時（高速フィードバック）

```bash
# スモークテストのみ（0.8秒）
pytest -m smoke

# 特定のテストファイル
pytest tests/api/test_health.py -v

# 詳細出力
pytest -v -s
```

### デプロイ前（完全テスト）

```bash
# すべてのテスト
pytest

# カバレッジレポート付き
pytest --cov=. --cov-report=html
open htmlcov/index.html
```

### デバッグ時

```bash
# Playwrightのデバッグモード
PWDEBUG=1 pytest tests/e2e/

# ヘッドフルモード（ブラウザ表示）
pytest tests/e2e/ --headed

# スローモーション
pytest tests/e2e/ --slowmo=1000
```

## スクリーンショット活用

テスト実行時に自動でスクリーンショットを保存：

```python
# 失敗時のスクリーンショット
await page.screenshot(path=f"screenshots/error_{test_name}.png")

# フルページスクリーンショット
await page.screenshot(path="screenshots/full_page.png", full_page=True)
```

**生成されるスクリーンショット：**
- `screenshots/n8n_homepage.png` (28KB)
- `screenshots/dify_install_page.png` (30KB)
- `screenshots/dify_after_setup.png` (正常に/appsに遷移)

## ベストプラクティス

### 1. テストの独立性を保つ

❌ **悪い例：**
```python
# test_a が test_b に依存
def test_a():
    create_user("test@example.com")

def test_b():
    login("test@example.com")  # test_a に依存
```

✅ **良い例：**
```python
@pytest.fixture
def test_user():
    user = create_user("test@example.com")
    yield user
    cleanup_user(user)

def test_b(test_user):
    login(test_user.email)
```

### 2. 適切な待機時間

❌ **悪い例：**
```python
await page.goto(url)
await page.click("button")  # 要素がまだ存在しない可能性
```

✅ **良い例：**
```python
await page.goto(url)
await page.wait_for_load_state("networkidle")
await page.wait_for_timeout(2000)  # React描画を待つ
button = page.locator("button:has-text('Submit')").first
await button.click()
```

### 3. エラーハンドリング

```python
try:
    await page.goto(url, timeout=30000)
except Exception as e:
    await page.screenshot(path="screenshots/error.png")
    print(f"Error: {e}")
    raise
```

## トラブルシューティング

### Playwrightが動作しない

```bash
# ブラウザの再インストール
playwright install --force chromium

# 依存関係の確認
playwright install-deps
```

### Difyで"Internal Server Error"

```bash
# データベースマイグレーション
docker compose exec dify-api flask db upgrade

# ログ確認
docker compose logs dify-api | tail -50
```

### ポート競合

```bash
# 使用中のポートを確認
sudo lsof -i :5678
sudo lsof -i :3000

# docker-compose.ymlでポート変更
ports:
  - "15678:5678"  # n8n
```

## 最終テスト結果

```bash
$ pytest tests/
============== test session starts ==============
collected 18 items

tests/api/test_health.py ✓✓✓✓✓✓✓        [ 38%]
tests/e2e/test_n8n_ui.py ✓✓✓             [ 55%]
tests/e2e/test_dify_ui.py ✓✓✓            [ 72%]
tests/e2e/test_dify_setup.py ✓✓          [ 83%]
tests/e2e/test_dify_detailed.py ✓✓       [100%]

======== 18 passed in 33.21s ========
Coverage: 85%
```

## まとめ

本記事で構築した自動テスト環境：

✅ **3層テスト戦略**
- API Tests（0.8秒）
- E2E Tests（33秒）
- カバレッジ85%

✅ **Playwright + pytest**
- 非同期APIで高速実行
- スクリーンショット自動保存
- CI/CD統合可能

✅ **実践的なトラブルシューティング**
- Difyのデータベースマイグレーション
- 初期セットアップ自動化
- エラー時のデバッグ方法

### 次のステップ

1. **統合テストの追加** - n8nとDifyの連携テスト
2. **パフォーマンステスト** - 負荷試験
3. **ビジュアルリグレッションテスト** - UI変更の検出

### リポジトリ

完全な実装は以下で公開しています：
- [GitHub: ai-workflow-engine](https://github.com/0xchoux1/ai-workflow-engine)

## 参考リソース

- [Playwright Documentation](https://playwright.dev/python/)
- [pytest Documentation](https://docs.pytest.org/)
- [n8n API Documentation](https://docs.n8n.io/api/)
- [Dify Documentation](https://docs.dify.ai/)

---

**管理者ログイン情報（開発環境）:**
- Dify: `admin@example.com` / `Admin123456!`
- n8n: http://localhost:5678 (初回アクセス時にセットアップ)

自動テスト環境を構築することで、開発速度と品質の両立が実現できました。ぜひ皆さんのプロジェクトにも取り入れてみてください！
