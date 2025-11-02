---
title: "n8nとDifyで構築する最強のAIワークフロー自動化環境【Dockerセルフホスト完全ガイド】"
date: 2025-11-02T16:50:00+09:00
draft: false
tags: ["n8n", "Dify", "Docker", "AI", "ワークフロー自動化", "LLM", "RAG"]
categories: ["技術"]
series: ["AI開発環境"]
---

## はじめに

業務効率化のためのSaaSツールは増え続けていますが、それぞれが孤立してしまい、「ツール疲れ」を感じていませんか？データが分散し、手動作業が増え、本来の業務に集中できない…そんな課題を解決するのが、ワークフロー自動化ツールです。

本記事では、**n8n**（汎用ワークフロー自動化）と**Dify**（LLM特化型プラットフォーム）という2つの強力なオープンソースツールをDockerでセルフホストし、AI駆動のワークフロー環境を構築する方法を解説します。

## なぜn8nとDifyの両方が必要なのか

### それぞれの役割

この2つのツールは互いに補完し合う関係にあります：

```
┌─────────────────────────────────────────────────┐
│                   n8n                           │
│  - トリガー管理（Webhook、スケジュール）        │
│  - 外部SaaSとのデータ連携                       │
│  - データの前処理・後処理                       │
│  - 通知・レポート配信                           │
└──────────────┬──────────────────────────────────┘
               │ API連携
┌──────────────▼──────────────────────────────────┐
│                   Dify                          │
│  - LLMによる自然言語処理                        │
│  - RAGによる知識ベース検索                      │
│  - エージェント実行                             │
│  - プロンプト最適化                             │
└─────────────────────────────────────────────────┘
```

### n8n: 汎用ワークフロー自動化の王道

**主な特徴：**
- Zapier/Makeのオープンソース代替
- 1000以上の統合ノード（Slack, Notion, GitHub, Google Workspace等）
- ビジュアルワークフローエディタ
- JavaScriptによるカスタムロジック記述

**得意なこと：**
- 外部サービス間のデータ連携
- 定期的なデータ収集・集計
- 通知・レポート配信
- バックオフィス業務の自動化

**使用例：**
```
1. Slackで特定のキーワードが投稿されたら
2. Notionに自動でタスクを作成
3. 担当者にメール通知
```

### Dify: LLM特化型のワークフローエンジン

**主な特徴：**
- LLMOps（LLM Operations）プラットフォーム
- 複数LLMプロバイダーの一元管理（OpenAI, Anthropic, Azure OpenAI等）
- RAG（検索拡張生成）パイプライン構築
- プロンプトのバージョン管理とA/Bテスト

**得意なこと：**
- LLMを使った自然言語処理
- 社内ナレッジベースの活用（RAG）
- プロンプトの最適化
- AIチャットボット・アシスタントの構築

**使用例：**
```
1. 社内FAQをベクトルDBに格納
2. ユーザーの質問を受け取る
3. RAGで関連情報を検索
4. LLMで文脈を考慮した回答を生成
```

### 比較表

| 観点 | n8n | Dify |
|------|-----|------|
| **主な用途** | SaaS連携・データ自動化 | LLM処理・AIアプリ開発 |
| **統合数** | 1000+ | LLM中心 |
| **プロンプト管理** | ❌ | ✅ |
| **RAG機能** | ❌ | ✅ |
| **Webhook/スケジューラ** | ✅ | △ |
| **条件分岐の柔軟性** | ✅ | △ |
| **ライセンス** | Sustainable Use | Apache 2.0 |

## 実践的なユースケース

### ケース1: 自動カスタマーサポート

```
[顧客] Slackで質問
    ↓
[n8n] 質問を受信 → テキスト前処理
    ↓
[Dify] RAGで社内ナレッジ検索 → LLMで回答生成
    ↓
[n8n] Slackに返信 + Notionに記録 + 未解決なら担当者に通知
```

### ケース2: コンテンツ生成パイプライン

```
[n8n] Google Sheetsからトピックリスト取得
    ↓
[Dify] 各トピックについてLLMで記事生成
    ↓
[n8n] WordPressに投稿 + SEO最適化 + SNSでシェア
```

### ケース3: データ分析レポート自動化

```
[n8n] 定期実行（毎週月曜9時）
    ↓
[n8n] データベースから週次データ抽出
    ↓
[Dify] LLMでデータ分析と洞察レポート生成
    ↓
[n8n] レポートをPDF化 → メール送信 → Slackに通知
```

## Docker環境の構築

### 前提条件

- Docker 20.10以上
- Docker Compose 2.0以上
- 最低8GBのメモリ（推奨16GB）

### セルフホストのメリット

- ✅ **コスト削減**: 従量課金なし、無制限の実行回数
- ✅ **データプライバシー**: 機密情報を自社環境で管理
- ✅ **カスタマイズ性**: 環境変数、ネットワーク設定の自由度
- ✅ **スケーラビリティ**: 必要に応じてリソースを調整可能

### インストール手順

#### 1. リポジトリのクローン

```bash
git clone https://github.com/0xchoux1/ai-workflow-engine.git
cd ai-workflow-engine
```

#### 2. 環境変数の設定

```bash
cp .env.example .env
vim .env
```

**重要な設定項目：**

```bash
# n8n認証情報（必ず変更！）
N8N_BASIC_AUTH_USER=your_username
N8N_BASIC_AUTH_PASSWORD=strong_password

# Difyシークレットキー（必ず変更！）
DIFY_SECRET_KEY=your-secret-key-here

# データベースパスワード
POSTGRES_PASSWORD=your_db_password
REDIS_PASSWORD=your_redis_password
```

#### 3. Docker環境の起動

```bash
docker compose up -d
```

#### 4. 起動確認

```bash
# コンテナの状態確認
docker compose ps

# ログの確認
docker compose logs -f
```

### 構成の詳細

**起動するサービス：**

```
n8n (port 5678)           - ワークフローエディタ
dify-web (port 3000)      - Dify フロントエンド
dify-api (port 5001)      - Dify API
dify-worker               - バックグラウンドタスク
postgres (port 5432)      - データベース
redis (port 6379)         - キャッシュ & キュー
weaviate (port 8080)      - ベクトルDB
```

**データの永続化：**
- `n8n_data`: n8nのワークフローとクレデンシャル
- `dify_app_storage`: Difyのアップロードファイル
- `postgres_data`: データベース
- `redis_data`: Redisのスナップショット
- `weaviate_data`: ベクトルデータベース

### 初回セットアップ

#### n8nのセットアップ

1. ブラウザで `http://localhost:5678` にアクセス
2. 管理者アカウントを作成
3. 初期ワークフローを試す

#### Difyのセットアップ

1. ブラウザで `http://localhost:3000` にアクセス
2. 管理者アカウントを作成
3. LLMプロバイダーを設定:
   - OpenAI APIキー
   - または Anthropic Claude
   - または Azure OpenAI

## n8nとDifyを連携させる

### 基本的な連携方法

**方法1: HTTP Request（n8n → Dify）**

1. n8nでHTTP Requestノードを使用
2. DifyのAPIエンドポイントを呼び出し
3. レスポンスを次のノードに渡す

**方法2: Webhook（Dify → n8n）**

1. n8nでWebhookノードを作成
2. DifyのワークフローからWebhook URLを呼び出し
3. n8nで後続処理を実行

### 実装例：Slackボットの構築

**ゴール:** Slackでメンションされたら、AIが返答する

**n8n側の設定：**
```
1. Slack Trigger: @botにメンション
2. HTTP Request: Dify APIに質問を送信
3. Slack: 回答を投稿
4. Notion: ログを記録
```

**Dify側の設定：**
```
1. Chatbot Appを作成
2. RAGでナレッジベースを設定
3. システムプロンプトを最適化
4. APIキーを発行
```

## トラブルシューティング

### ポート競合

既に使用されているポートを変更する場合：

```bash
# docker-compose.ymlを編集
ports:
  - "15678:5678"  # n8nのポート変更例
```

### メモリ不足

```bash
# Dockerのリソース制限を確認
docker stats

# docker-compose.ymlにメモリ制限を追加
services:
  dify-api:
    mem_limit: 2g
```

### Dify起動エラー

データベースの初期化を待つ：

```bash
docker compose up -d postgres
sleep 10
docker compose up -d
```

## ベストプラクティス

### セキュリティ

- [ ] すべてのデフォルトパスワードを変更
- [ ] `.env`ファイルをGit管理外に
- [ ] 本番環境ではHTTPS（SSL/TLS）を設定
- [ ] APIキーの権限を最小限に
- [ ] 定期的なセキュリティアップデート

### バックアップ戦略

**n8nデータのバックアップ：**

```bash
docker compose exec n8n sh -c \
  "cd /home/node/.n8n && tar czf - ." \
  > n8n-backup-$(date +%Y%m%d).tar.gz
```

**Difyデータベースのバックアップ：**

```bash
docker compose exec postgres pg_dump \
  -U postgres dify \
  > dify-backup-$(date +%Y%m%d).sql
```

### 運用Tips

- **ワークフローのバージョン管理**: n8nの設定をJSON出力してGit管理
- **ログモニタリング**: Grafana/Prometheusとの統合検討
- **リソース監視**: `docker stats`で定期チェック

## まとめ

n8nとDifyを併用することで、以下のような強力なAIワークフロー環境を構築できます：

- **n8n**: 外部SaaSとの連携、トリガー管理、データ処理
- **Dify**: LLMによる知的処理、RAG、プロンプト管理

セルフホストすることで、コスト削減とデータプライバシーを両立しながら、柔軟なカスタマイズが可能になります。

### 次のステップ

1. **基本を試す**: まずはシンプルなワークフローから
2. **連携を深める**: n8nとDifyのAPI連携を実装
3. **独自のユースケース**: 自社の業務に適用
4. **コミュニティ参加**: GitHub、Discord等で情報交換

### 今後の可能性

- **マルチエージェント**: 複数のAIエージェントが協調
- **より高度なRAG**: GraphRAG、HyDEなどの手法
- **オープンソースLLMの活用**: Llama、Mixtralなど

## 参考リソース

- [プロジェクトリポジトリ](https://github.com/0xchoux1/ai-workflow-engine)
- [n8n公式ドキュメント](https://docs.n8n.io/)
- [Dify公式ドキュメント](https://docs.dify.ai/)
- [n8n GitHub](https://github.com/n8n-io/n8n)
- [Dify GitHub](https://github.com/langgenius/dify)

---

この環境を使って、あなたの業務を効率化する最初の一歩を踏み出してみませんか？
