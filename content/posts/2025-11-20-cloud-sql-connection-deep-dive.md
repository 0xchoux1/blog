---
title: "Cloud SQL接続の落とし穴 - プライベートIP、SSL設定、パスワード管理の完全ガイド"
date: 2025-11-20T12:00:00+09:00
draft: false
tags: ["GCP", "Cloud SQL", "MySQL", "セキュリティ", "ネットワーク"]
categories: ["技術", "インフラ"]
series: ["WordPress マルチテナント環境構築"]
---


## はじめに

「データベースに接続できない」

この一言の裏には、ネットワーク、認証、SSL証明書、パスワード管理など、**複数の階層にわたる設定が絡み合っています**。

GCP上でWordPress マルチテナント環境を構築する過程で、Cloud SQLへの接続で何度も壁にぶつかりました。この記事では、その経験から得た**Cloud SQL接続のベストプラクティスとトラブルシューティング手法**を共有します。

**この記事で扱う内容**:
1. Cloud SQLの接続方式（プライベートIP vs パブリックIP）
2. SSL/TLS設定の理解と選択
3. Secret Managerを使ったパスワード管理
4. 接続エラーのデバッグ手法
5. Terraformでの適切な設定方法

---

## 1. Cloud SQL接続方式の選択

### プライベートIP vs パブリックIP

Cloud SQLには2つの接続方式があります：

| 接続方式 | 説明 | ユースケース | SSL要件 |
|---------|------|------------|---------|
| **プライベートIP** | VPC内部からのみアクセス可能 | 本番環境、セキュアな接続 | 不要（推奨） |
| **パブリックIP** | インターネットから接続可能 | 開発環境、外部ツール接続 | 必須 |

**今回の構成**: プライベートIP接続

```
┌────────────────────────────────────────┐
│            VPC Network                 │
│                                        │
│  ┌──────────────┐    ┌─────────────┐ │
│  │ Compute VM   │───▶│ Cloud SQL   │ │
│  │ 10.0.1.x     │    │ 10.168.0.2  │ │
│  └──────────────┘    └─────────────┘ │
│                                        │
│  Private IP Connection                 │
│  No SSL Required                       │
└────────────────────────────────────────┘
```

### プライベートIP接続のメリット

1. **セキュリティ**: インターネットに公開されない
2. **レイテンシ**: VPC内部通信で低遅延
3. **コスト**: エグレス料金が発生しない
4. **シンプルさ**: SSL証明書の管理が不要

### Terraformでの設定

```hcl
# terraform/modules/database/main.tf
resource "google_sql_database_instance" "wordpress" {
  name             = "${var.env}-wordpress-db"
  database_version = "MYSQL_8_0"
  region           = var.region

  settings {
    tier = "db-f1-micro"

    # プライベートIP接続設定
    ip_configuration {
      ipv4_enabled    = false               # パブリックIPを無効化
      private_network = var.network_id      # VPCネットワークを指定
      require_ssl     = false               # SSL要件を無効化
    }

    # バックアップ設定
    backup_configuration {
      enabled            = true
      start_time         = "03:00"
      binary_log_enabled = true
    }
  }
}
```

### 重要なポイント

**`ipv4_enabled = false`**
- パブリックIPアドレスを割り当てない
- インターネットからのアクセスを完全にブロック

**`private_network = var.network_id`**
- VPCネットワークを指定
- Private Service Connectionを使用

**`require_ssl = false`**
- プライベートIP接続ではSSL不要
- これを `true` にすると証明書管理が必要になる

---

## 2. SSL/TLS設定の理解

### SSL証明書が必要なケース

```yaml
必要:
  □ パブリックIP接続
  □ インターネット経由でのアクセス
  □ コンプライアンス要件

不要:
  □ プライベートIP接続（VPC内部）
  □ トラフィックが暗号化されたネットワーク
```

### SSL証明書のライフサイクル

Cloud SQLでSSL証明書を使う場合の手順：

#### 1. サーバーCA証明書の取得

```bash
# サーバーCA証明書をダウンロード
gcloud sql ssl-certs describe server-ca \
  --instance=prod-wordpress-db \
  --format="value(cert)" > server-ca.pem
```

#### 2. クライアント証明書の作成

```bash
# クライアント証明書を作成
gcloud sql ssl-certs create wordpress-client \
  --instance=prod-wordpress-db

# クライアント証明書をダウンロード
gcloud sql ssl-certs describe wordpress-client \
  --instance=prod-wordpress-db \
  --format="value(cert)" > client-cert.pem

# 秘密鍵をダウンロード
gcloud sql ssl-certs describe wordpress-client \
  --instance=prod-wordpress-db \
  --format="value(privateKey)" > client-key.pem
```

#### 3. MySQLクライアントでの使用

```bash
mysql \
  --host=10.168.0.2 \
  --user=wp_user_1 \
  --password \
  --ssl-ca=server-ca.pem \
  --ssl-cert=client-cert.pem \
  --ssl-key=client-key.pem
```

### SSL証明書の有効期限管理

```bash
# 証明書一覧と有効期限確認
gcloud sql ssl-certs list \
  --instance=prod-wordpress-db \
  --format="table(commonName,expirationTime)"

# 出力例
# COMMON_NAME        EXPIRATION_TIME
# wordpress-client   2025-01-20T12:00:00Z
# server-ca          2035-01-18T12:00:00Z
```

**注意点**:
- クライアント証明書: 有効期限10年
- サーバーCA証明書: 有効期限10年
- 期限切れ前に再発行が必要

### SSL無効化の判断基準

**SSL無効化を推奨するケース**:
- ✅ プライベートIP接続のみ
- ✅ VPC内部の通信
- ✅ 証明書管理コストを削減したい
- ✅ パフォーマンスを優先したい

**SSL有効化が必須のケース**:
- ❌ パブリックIP接続
- ❌ インターネット経由のアクセス
- ❌ コンプライアンス要件（PCI-DSS等）
- ❌ 監査要件

---

## 3. Secret Managerを使ったパスワード管理

### なぜSecret Managerを使うのか

**NGパターン**:
```hcl
# ❌ 平文でパスワードを記述
resource "google_sql_user" "wordpress_user" {
  password = "my_super_secret_password"  # 絶対にダメ！
}
```

**問題点**:
- Terraformステートファイルに平文で保存される
- Gitにコミットされるリスク
- チームメンバー全員がパスワードを知ることになる

**正しいアプローチ**:
```hcl
# ✅ Secret Managerでパスワード管理
resource "random_password" "db_passwords" {
  count   = 10
  length  = 20
  special = true
}

resource "google_secret_manager_secret" "db_passwords" {
  count     = 10
  secret_id = "${var.env}-wordpress-db-password-${count.index + 1}"

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "db_passwords" {
  count       = 10
  secret      = google_secret_manager_secret.db_passwords[count.index].id
  secret_data = random_password.db_passwords[count.index].result
}
```

### パスワードのライフサイクル管理

#### 1. パスワード生成（Terraform）

```hcl
resource "random_password" "db_passwords" {
  length  = 20
  special = true

  lifecycle {
    ignore_changes = [length, special]  # 再生成を防ぐ
  }
}
```

#### 2. Secret Managerに保存

```bash
# 手動でパスワードを設定する場合
echo -n "your_secure_password" | \
  gcloud secrets versions add prod-wordpress-db-password-1 \
  --data-file=-
```

#### 3. Cloud SQLユーザーに設定

```bash
# Secret Managerからパスワードを取得してCloud SQLに設定
PASSWORD=$(gcloud secrets versions access latest \
  --secret=prod-wordpress-db-password-1)

gcloud sql users set-password wp_user_1 \
  --instance=prod-wordpress-db \
  --password="$PASSWORD"
```

#### 4. アプリケーションからの取得

```bash
# Ansibleでの取得例
- name: Secret Managerからパスワード取得
  command: >
    gcloud secrets versions access latest
    --secret={{ env }}-wordpress-db-password-{{ item }}
    --project={{ gcp_project_id }}
  register: db_password
  no_log: true  # ログに出力しない
  loop: "{{ range(1, 11) | list }}"
```

### パスワード不一致のトラブルシューティング

**症状**:
```
ERROR 1045 (28000): Access denied for user 'wp_user_1'@'10.0.1.21' (using password: YES)
```

**原因**:
- Secret ManagerとCloud SQLのパスワードが不一致
- Terraform applyでパスワードが再生成された
- 手動でパスワードを変更した

**解決策**: 全ユーザーのパスワードを同期

```bash
#!/bin/bash
# sync-db-passwords.sh

PROJECT_ID="infra-ai-agent"
INSTANCE_NAME="prod-wordpress-db"
ENV="prod"

for i in {1..10}; do
  echo "Syncing password for wp_user_$i..."

  # Secret Managerから最新のパスワードを取得
  PASSWORD=$(gcloud secrets versions access latest \
    --secret="${ENV}-wordpress-db-password-${i}" \
    --project="${PROJECT_ID}")

  # Cloud SQLユーザーのパスワードを更新
  gcloud sql users set-password "wp_user_${i}" \
    --instance="${INSTANCE_NAME}" \
    --password="${PASSWORD}" \
    --project="${PROJECT_ID}"

  echo "✅ wp_user_$i synchronized"
done

echo "🎉 All passwords synchronized!"
```

実行結果:
```bash
chmod +x sync-db-passwords.sh
./sync-db-passwords.sh

# Syncing password for wp_user_1...
# ✅ wp_user_1 synchronized
# Syncing password for wp_user_2...
# ✅ wp_user_2 synchronized
# ...
# 🎉 All passwords synchronized!
```

---

## 4. 接続エラーのデバッグ手法

### デバッグの基本フロー

```
┌─────────────────────────────────────────┐
│ Step 1: ネットワーク疎通確認            │
│  ping, nc, telnet                       │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Step 2: MySQLクライアントで接続テスト   │
│  mysql -h HOST -u USER -p               │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Step 3: Cloud Loggingでログ確認         │
│  gcloud logging read                    │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Step 4: 設定の検証                      │
│  Terraform, wp-config.php               │
└─────────────────────────────────────────┘
```

### Step 1: ネットワーク疎通確認

```bash
# 1. Cloud SQLインスタンスのIPアドレス確認
gcloud sql instances describe prod-wordpress-db \
  --format="value(ipAddresses[0].ipAddress)"
# 出力: 10.168.0.2

# 2. Compute VMからping
gcloud compute ssh prod-web-l0br \
  --zone=asia-northeast1-a \
  --tunnel-through-iap \
  --command="ping -c 3 10.168.0.2"

# 3. ポート疎通確認（3306）
gcloud compute ssh prod-web-l0br \
  --zone=asia-northeast1-a \
  --tunnel-through-iap \
  --command="nc -zv 10.168.0.2 3306"
# 出力: Connection to 10.168.0.2 3306 port [tcp/mysql] succeeded!
```

### Step 2: MySQLクライアントで接続テスト

```bash
# Cloud SQL Proxyを使った接続テスト
cloud_sql_proxy -instances=infra-ai-agent:asia-northeast1:prod-wordpress-db=tcp:3306 &

# ローカルから接続
mysql -h 127.0.0.1 -u wp_user_1 -p

# 接続成功の確認
mysql> SELECT USER(), DATABASE();
+-------------------------+----------+
| USER()                  | DATABASE()|
+-------------------------+----------+
| wp_user_1@10.0.1.21     | NULL     |
+-------------------------+----------+
```

### Step 3: Cloud Loggingでログ確認

```bash
# Cloud SQLの接続ログを確認
gcloud logging read \
  'resource.type="cloudsql_database"
   AND logName="projects/infra-ai-agent/logs/cloudsql.googleapis.com%2Fmysql.err"' \
  --limit 50 \
  --format json \
  --project=infra-ai-agent

# 認証エラーの検索
gcloud logging read \
  'resource.type="cloudsql_database"
   AND textPayload=~"Access denied"' \
  --limit 10 \
  --format json
```

### Step 4: 設定ファイルの検証

#### wp-config.php の確認

```bash
# リモートサーバーでwp-config.phpを確認
gcloud compute ssh prod-web-l0br \
  --zone=asia-northeast1-a \
  --tunnel-through-iap \
  --command="sudo grep -E '^define.*DB_' /var/www/wordpress/site1/wp-config.php"

# 出力例
# define('DB_NAME', 'wordpress_db_1');
# define('DB_USER', 'wp_user_1');
# define('DB_PASSWORD', '***');
# define('DB_HOST', '10.168.0.2');
```

#### データベース存在確認

```bash
# データベース一覧を取得
gcloud sql databases list \
  --instance=prod-wordpress-db \
  --format="table(name,charset,collation)"

# ユーザー一覧を取得
gcloud sql users list \
  --instance=prod-wordpress-db \
  --format="table(name,host)"
```

### よくあるエラーと解決策

#### エラー1: `Can't connect to MySQL server`

**原因**:
- ネットワーク疎通がない
- ファイアウォールルールが不足
- Cloud SQLが起動していない

**確認**:
```bash
# Cloud SQLの状態確認
gcloud sql instances describe prod-wordpress-db \
  --format="value(state)"
# 期待値: RUNNABLE

# ファイアウォールルール確認
gcloud compute firewall-rules list \
  --filter="network:prod-vpc" \
  --format="table(name,direction,allowed[].ports)"
```

#### エラー2: `Access denied for user`

**原因**:
- パスワードが間違っている
- ユーザーが存在しない
- ホスト制限

**確認**:
```bash
# ユーザーの存在確認
gcloud sql users list --instance=prod-wordpress-db

# パスワード同期
PASSWORD=$(gcloud secrets versions access latest --secret=prod-wordpress-db-password-1)
gcloud sql users set-password wp_user_1 \
  --instance=prod-wordpress-db \
  --password="$PASSWORD"
```

#### エラー3: `SSL connection error`

**原因**:
- `require_ssl = true` だが証明書がない
- 証明書のパスが間違っている

**解決**:
```hcl
# Terraformで設定変更
resource "google_sql_database_instance" "wordpress" {
  settings {
    ip_configuration {
      require_ssl = false  # プライベートIP接続ならfalse
    }
  }
}
```

---

## 5. ベストプラクティス

### セキュリティ

```yaml
□ プライベートIP接続を使用
□ パブリックIPは無効化（ipv4_enabled = false）
□ Secret Managerでパスワード管理
□ 最小権限の原則（データベース権限）
□ IAMによるCloud SQLへのアクセス制御
```

### 可用性

```yaml
□ 自動バックアップ有効化
□ バイナリログ有効化（ポイントインタイムリカバリ用）
□ 高可用性構成（本番環境）
□ メンテナンスウィンドウの設定
```

### パフォーマンス

```yaml
□ 適切なマシンタイプ選択
□ ストレージタイプの選択（SSD推奨）
□ 接続プーリングの使用
□ スロークエリログの監視
```

### 運用

```yaml
□ Cloud Loggingでログ監視
□ Cloud Monitoringでメトリクス監視
□ アラートポリシーの設定
□ 定期的なバックアップテスト
```

### Terraformでの完全な設定例

```hcl
# terraform/modules/database/main.tf
resource "google_sql_database_instance" "wordpress" {
  name             = "${var.env}-wordpress-db"
  database_version = "MYSQL_8_0"
  region           = var.region

  settings {
    tier              = "db-custom-2-7680"  # 本番用
    availability_type = "REGIONAL"          # 高可用性

    # IPアドレス設定
    ip_configuration {
      ipv4_enabled    = false
      private_network = var.network_id
      require_ssl     = false
    }

    # バックアップ設定
    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      binary_log_enabled             = true
      transaction_log_retention_days = 7
      backup_retention_settings {
        retained_backups = 30
        retention_unit   = "COUNT"
      }
    }

    # メンテナンス設定
    maintenance_window {
      day          = 7  # 日曜日
      hour         = 3  # 午前3時
      update_track = "stable"
    }

    # データベースフラグ
    database_flags {
      name  = "max_connections"
      value = "200"
    }

    database_flags {
      name  = "slow_query_log"
      value = "on"
    }

    # ログ設定
    insights_config {
      query_insights_enabled  = true
      query_plans_per_minute  = 5
      query_string_length     = 1024
      record_application_tags = true
    }
  }

  deletion_protection = true  # 本番環境では必須

  lifecycle {
    prevent_destroy = true
  }
}

# データベース作成
resource "google_sql_database" "wordpress_dbs" {
  count    = 10
  name     = "wordpress_db_${count.index + 1}"
  instance = google_sql_database_instance.wordpress.name
  charset  = "utf8mb4"
  collation = "utf8mb4_unicode_ci"
}

# ユーザー作成
resource "google_sql_user" "wordpress_users" {
  count    = 10
  name     = "wp_user_${count.index + 1}"
  instance = google_sql_database_instance.wordpress.name
  password = random_password.db_passwords[count.index].result

  lifecycle {
    ignore_changes = [password]  # パスワード再生成を防ぐ
  }
}
```

---

## 6. トラブルシューティングチェックリスト

### 接続できない時の確認順序

```
1. ネットワーク層
   □ Cloud SQLのIPアドレスは正しいか？
   □ VPCネットワークは同じか？
   □ Private Service Connectionは設定されているか？
   □ ファイアウォールルールは適切か？
   □ pingは通るか？
   □ ポート3306は開いているか？

2. 認証層
   □ ユーザー名は正しいか？
   □ パスワードは正しいか？
   □ Secret Managerと同期されているか？
   □ ユーザーは存在するか？
   □ ホスト制限は適切か？

3. SSL層
   □ require_ssl設定は正しいか？
   □ 証明書は有効期限内か？
   □ 証明書のパスは正しいか？
   □ プライベートIP接続ならSSL不要か確認

4. データベース層
   □ データベースは存在するか？
   □ ユーザーに権限はあるか？
   □ Cloud SQLは起動しているか？
   □ メンテナンス中ではないか？

5. アプリケーション層
   □ wp-config.phpの設定は正しいか？
   □ PHPのmysqli拡張は有効か？
   □ 接続プーリングは適切か？
   □ タイムアウト設定は十分か？
```

### コマンド集

```bash
# === Cloud SQL情報取得 ===
# インスタンス詳細
gcloud sql instances describe INSTANCE_NAME

# IPアドレス取得
gcloud sql instances describe INSTANCE_NAME \
  --format="value(ipAddresses[0].ipAddress)"

# 状態確認
gcloud sql instances describe INSTANCE_NAME \
  --format="value(state)"

# === データベース管理 ===
# データベース一覧
gcloud sql databases list --instance=INSTANCE_NAME

# ユーザー一覧
gcloud sql users list --instance=INSTANCE_NAME

# パスワード変更
gcloud sql users set-password USER_NAME \
  --instance=INSTANCE_NAME \
  --password="NEW_PASSWORD"

# === 接続テスト ===
# ポート疎通確認
nc -zv DB_IP 3306

# MySQLクライアント接続
mysql -h DB_IP -u USER_NAME -p

# === Secret Manager ===
# シークレット一覧
gcloud secrets list

# シークレット取得
gcloud secrets versions access latest --secret=SECRET_NAME

# === ログ確認 ===
# Cloud SQLエラーログ
gcloud logging read \
  'resource.type="cloudsql_database"
   AND logName=~"mysql.err"' \
  --limit 50

# 認証エラー検索
gcloud logging read \
  'resource.type="cloudsql_database"
   AND textPayload=~"Access denied"' \
  --limit 10
```

---

## まとめ

### Cloud SQL接続の3大ポイント

1. **プライベートIP接続 + SSL無効化**
   - セキュアかつシンプル
   - 証明書管理不要
   - パフォーマンス向上

2. **Secret Managerでパスワード管理**
   - 平文でのパスワード保存を避ける
   - 定期的な同期スクリプト実行
   - `lifecycle.ignore_changes`で再生成防止

3. **段階的なデバッグ**
   - ネットワーク → 認証 → SSL → アプリケーション
   - Cloud Loggingを活用
   - 各層で確実に検証

### よくあるミスと対策

| ミス | 対策 |
|------|------|
| パスワード不一致 | 同期スクリプトの定期実行 |
| SSL証明書エラー | プライベートIP接続ならSSL無効化 |
| ネットワーク疎通なし | Private Service Connection確認 |
| 権限不足 | IAMロールとデータベース権限の両方確認 |

### 次のステップ

- Cloud SQL Proxyの導入検討
- 読み取りレプリカの設定
- パフォーマンスチューニング
- 監視とアラート強化

---

## 参考リンク

- [Cloud SQL - Private IP](https://cloud.google.com/sql/docs/mysql/private-ip)
- [Cloud SQL - SSL/TLS certificates](https://cloud.google.com/sql/docs/mysql/configure-ssl-instance)
- [Secret Manager](https://cloud.google.com/secret-manager/docs)
- [Terraform - google_sql_database_instance](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/sql_database_instance)

---

## この記事のコード

GitHub: [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)

関連ファイル:
- [terraform/modules/database/main.tf](https://github.com/0xchoux1/infra-ai-agent/blob/main/terraform/modules/database/main.tf)
- [sync-db-passwords.sh](https://github.com/0xchoux1/infra-ai-agent/blob/main/scripts/sync-db-passwords.sh)

---

**この記事が役に立ったら**: GitHub Starをいただけると嬉しいです！ [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)
