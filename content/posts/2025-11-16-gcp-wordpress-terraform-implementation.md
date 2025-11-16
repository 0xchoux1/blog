---
title: "Terraform実装完了！GCP上でWordPressマルチテナント環境を構築【terraform plan検証済み】"
date: 2025-11-16T17:30:00+09:00
draft: false
tags: ["GCP", "Terraform", "WordPress", "実装", "インフラコード", "IaC", "terraform plan"]
categories: ["技術", "実装"]
series: ["AIエージェント開発"]
---

## はじめに

[前回の記事](../2025-11-10-gcp-wordpress-multitenancy-design/)では、WordPressマルチテナント環境の要件定義とTerraform設計を完成させました。

今回は、**設計書をもとに実際のTerraformコードを実装し、terraform planで検証**するまでの全プロセスを解説します。

**成果物:**
- ✅ 7モジュール、46ファイル、2,525行のTerraformコード
- ✅ terraform plan検証済み（prod: 94リソース, dev: 59リソース）
- ✅ エラー0件で実装完了

---

## 🚀 実装フロー

### 全体の流れ

```
1. Terraform環境準備
   ├─ Terraform 1.13.5インストール
   ├─ GCP認証設定
   └─ 必要なAPI有効化

2. モジュール実装（7モジュール）
   ├─ Network（VPC/NAT/Firewall）
   ├─ IAM（最小権限）
   ├─ Filestore（NFS共有ストレージ）
   ├─ Database（Cloud SQL + 10サイト自動生成）
   ├─ Compute（MIG + Auto Scaling）
   ├─ Load Balancer（LB/CDN/WAF/SSL）
   └─ Monitoring（アラート/ログ）

3. 環境別設定
   ├─ prod環境（10サイト、HA構成）
   └─ dev環境（3サイト、単一構成）

4. 検証
   ├─ terraform fmt（フォーマット）
   ├─ terraform validate（構文チェック）
   └─ terraform plan（実行計画）

5. コミット＆プッシュ
```

---

## 1. Terraform環境準備

### Terraform 1.13.5インストール

**課題**: HomebrewのTerraformは1.5.7（MPL 2.0）で止まっている

**選択肢:**
- Terraform 1.5.7（完全オープンソース）
- Terraform 1.13.5（BUSL - 商用制限あり）
- OpenTofu（Terraformフォーク、Apache 2.0）

**判断**: 自社インフラ管理なのでBUSL制限に該当しない → **最新版1.13.5を採用**

```bash
# Homebrewのterraformをアンインストール
brew uninstall terraform

# 最新版バイナリをダウンロード
cd /tmp
wget https://releases.hashicorp.com/terraform/1.13.5/terraform_1.13.5_linux_amd64.zip
unzip terraform_1.13.5_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# バージョン確認
terraform version
# Terraform v1.13.5
```

**ライセンス確認:**

```bash
cat LICENSE.txt | head -20
# Business Source License (BUSL)
# 競合サービス提供時のみ制限
# 自社インフラ管理には影響なし
```

### GCP認証とAPI有効化

```bash
# Application Default Credentials設定
gcloud auth application-default login

# 必要なAPI一括有効化
gcloud services enable \
  compute.googleapis.com \
  servicenetworking.googleapis.com \
  sqladmin.googleapis.com \
  file.googleapis.com \
  secretmanager.googleapis.com \
  cloudresourcemanager.googleapis.com \
  --project=infra-ai-agent

# 成功: Operation "operations/..." finished successfully.
```

---

## 2. モジュール実装

### 実装したモジュール一覧

| モジュール | ファイル数 | 主な機能 |
|-----------|----------|---------|
| **Network** | 5 | VPC, サブネット, NAT, Firewall, Service Networking |
| **IAM** | 3 | サービスアカウント, 最小権限設定（3ロール分離） |
| **Filestore** | 3 | NFS共有ストレージ（1TB）, IP予約 |
| **Database** | 4 | Cloud SQL HA, 10サイトDB自動生成, Secret Manager |
| **Compute** | 6 | Instance Template, MIG, Auto Scaling, 起動スクリプト |
| **Load Balancer** | 6 | Backend, Frontend, SSL, Cloud Armor, CDN |
| **Monitoring** | 3 | HTTP Alert, Health Check Alert, Log Sink |

### Network モジュールのポイント

**Service Networking（Cloud SQL Private IP）:**

```hcl
# modules/network/service_networking.tf

# API有効化
resource "google_project_service" "servicenetworking" {
  service            = "servicenetworking.googleapis.com"
  disable_on_destroy = false
}

# IP範囲予約
resource "google_compute_global_address" "private_ip_address" {
  name          = "${var.env}-sql-private-ip"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

# VPC Peering接続
resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_address.name]
  
  depends_on = [google_project_service.servicenetworking]
}
```

**学んだこと:**
- Cloud SQLをPrivate IPで使うには、Service Networking必須
- API有効化 → IP予約 → Peering接続の順序が重要
- `depends_on`でリソース作成順序を制御

### IAM モジュール - 最小権限の原則

**Secret Manager権限を3ロールに分離:**

```hcl
# modules/iam/main.tf

# ✅ 読み取り専用
resource "google_project_iam_member" "web_secret_accessor" {
  role = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_service_account.web_server.email}"
}

# ✅ Secret新規作成のみ
resource "google_project_iam_member" "web_secret_creator" {
  role = "roles/secretmanager.secretCreator"
  member  = "serviceAccount:${google_service_account.web_server.email}"
}

# ✅ 既存Secretへのバージョン追加のみ
resource "google_project_iam_member" "web_secret_version_adder" {
  role = "roles/secretmanager.secretVersionAdder"
  member  = "serviceAccount:${google_service_account.web_server.email}"
}
```

**セキュリティ効果:**

| 操作 | admin（NG） | 3ロール分離（OK） |
|------|------------|-----------------|
| Secret読み取り | ✅ | ✅ |
| Secret作成 | ✅ | ✅ |
| バージョン追加 | ✅ | ✅ |
| **Secret削除** | ⚠️ 可能 | ❌ 不可 |
| **IAM設定変更** | ⚠️ 可能 | ❌ 不可 |

### Database モジュール - ドメインリスト駆動

**10サイト分のDB/ユーザーを自動生成:**

```hcl
# modules/database/databases.tf

# サイト数 = ドメイン数
locals {
  site_count = length(var.domains)
}

# DB自動生成
resource "google_sql_database" "wordpress_sites" {
  count     = local.site_count
  name      = "wordpress_site_${count.index + 1}"
  instance  = google_sql_database_instance.wordpress.name
  charset   = "utf8mb4"
  collation = "utf8mb4_unicode_ci"
}

# DBユーザー自動生成
resource "google_sql_user" "wordpress_users" {
  count    = local.site_count
  name     = "wp_user_${count.index + 1}"
  instance = google_sql_database_instance.wordpress.name
  password = random_password.db_passwords[count.index].result
}

# パスワードをSecret Managerに保存
resource "google_secret_manager_secret" "db_passwords" {
  count     = local.site_count
  secret_id = "${var.env}-wordpress-db-password-${count.index + 1}"
  
  replication {
    auto {}
  }
}
```

**メリット:**
- ドメインリスト追加 → 自動的にDB/Secret生成
- 手動設定ミスがゼロ
- スケーラブル（10サイトでも100サイトでも同じコード）

### Compute モジュール - 起動スクリプト

**438行の起動スクリプトで実現:**

```bash
# terraform/scripts/startup_script.sh

#!/bin/bash
set -e

# Filestore NFSマウント
mount -t nfs ${NFS_IP}:${NFS_PATH} /var/www/wordpress

# ドメインリストから動的にNginx設定生成
DOMAINS_JSON='${domains_json}'
DOMAIN_COUNT=$(echo "$DOMAINS_JSON" | jq '. | length')

for i in $(seq 1 $DOMAIN_COUNT); do
  DOMAIN=$(echo "$DOMAINS_JSON" | jq -r ".[$((i-1))]")
  
  # Nginx仮想ホスト生成
  cat > /etc/nginx/sites-available/site${i} << EOF
server {
    listen 80;
    server_name ${DOMAIN};
    root /var/www/wordpress/site${i};
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        # ...
    }
}
EOF
  ln -sf /etc/nginx/sites-available/site${i} /etc/nginx/sites-enabled/
done

# WordPress自動セットアップスクリプト生成
cat > /usr/local/bin/setup-wordpress-site.sh << 'SCRIPT'
#!/bin/bash
SITE_NUM=$1
DB_PASS=$(gcloud secrets versions access latest \
  --secret="${ENV}-wordpress-db-password-${SITE_NUM}")

# WordPress インストール
wp core download
wp config create --dbname="wordpress_site_${SITE_NUM}" --dbpass="$DB_PASS"
wp core install --url="https://${DOMAIN}" --admin_password="$(openssl rand -base64 32)"

# 管理者パスワードをSecret Managerに保存
gcloud secrets create "${ENV}-wordpress-admin-password-${SITE_NUM}" || \
gcloud secrets versions add "${ENV}-wordpress-admin-password-${SITE_NUM}"
SCRIPT
```

---

## 3. 検証フェーズ

### terraform fmt（フォーマット）

```bash
cd terraform
terraform fmt -recursive

# 22ファイル自動整形
environments/dev/main.tf
environments/prod/main.tf
modules/compute/autoscaling.tf
modules/database/databases.tf
# ...
```

### terraform validate（構文チェック）

```bash
# prod環境
cd environments/prod
terraform init
terraform validate
# ✅ Success! The configuration is valid.

# dev環境
cd ../dev
terraform init
terraform validate
# ✅ Success! The configuration is valid.
```

### terraform plan（実行計画）

#### prod環境（10サイト構成）

```bash
cd environments/prod
terraform plan

# Plan: 94 to add, 0 to change, 0 to destroy.
```

**作成されるリソース内訳:**

| カテゴリ | リソース数 | 主な内容 |
|---------|----------|---------|
| Network | 11 | VPC, サブネット×2, NAT, Firewall×4, Service Networking |
| IAM | 11 | サービスアカウント×2, IAM権限×9 |
| Filestore | 3 | NFS×1, IP予約×2 |
| Database | 33 | Cloud SQL×1, DB×10, User×10, Secret×20, Password×10 |
| Compute | 14 | Template×1, MIG×1, Autoscaler×1, Health Check×1, etc |
| Load Balancer | 19 | Backend, URL Map×2, SSL×1, Cloud Armor, Forwarding×2, etc |
| Monitoring | 3 | Alert×2, Log Sink×1 |
| **合計** | **94** | |

#### dev環境（3サイト構成）

```bash
cd ../dev
terraform plan

# Plan: 59 to add, 0 to change, 0 to destroy.
```

**差分:**
- prod: 10サイト → DB×10 + Secret×20
- dev: 3サイト → DB×3 + Secret×6
- リソース差: 35個（主にサイト数の違い）

---

## 4. 遭遇した問題と解決

### 問題1: Secret Manager構文エラー

**エラー内容:**

```
Error: Unsupported argument
  on modules/database/databases.tf line 36:
  36:     automatic = true

An argument named "automatic" is not expected here.
```

**原因**: Secret Managerの`replication`ブロック構文が変更された

**修正:**

```hcl
# ❌ 古い構文
replication {
  automatic = true
}

# ✅ 新しい構文
replication {
  auto {}
}
```

### 問題2: Cloud SQL PITR設定エラー

**エラー内容:**

```
Error: point_in_time_recovery_enabled is only available for 
[POSTGRES SQLSERVER]. You may want to consider using 
binary_log_enabled instead.
```

**原因**: MySQLでは`point_in_time_recovery_enabled`は使えない

**修正:**

```hcl
# ❌ PostgreSQL/SQL Server用
backup_configuration {
  point_in_time_recovery_enabled = true
}

# ✅ MySQL用
backup_configuration {
  binary_log_enabled = var.availability_type == "REGIONAL"
}
```

**学んだこと**: バイナリログはHA構成時のみ有効化すべき

---

## 5. 実装の工夫

### 1. 変数駆動の設計

**ドメインリスト1つで全て決まる:**

```hcl
# terraform.tfvars
domains = [
  "example1.com",
  "example2.com",
  # ... 10個
]
```

**自動生成されるもの:**
- データベース×10
- DBユーザー×10
- Secret Manager Secret×10（DBパスワード）
- Secret Manager Secret×10（WordPress管理者パスワード）
- Nginx仮想ホスト×10
- URL Map Host Rule×10

### 2. 環境別パラメータ化

**prod環境:**

```hcl
db_availability_type = "REGIONAL"  # HA有効
db_tier              = "db-custom-2-7680"  # 2 vCPU
machine_type         = "e2-small"
min_replicas         = 2
```

**dev環境:**

```hcl
db_availability_type = "ZONAL"  # 単一インスタンス
db_tier              = "db-custom-1-3840"  # 1 vCPU
machine_type         = "e2-micro"
min_replicas         = 1
```

### 3. .gitignore設定

**機密情報を確実に除外:**

```gitignore
# Terraform
**/.terraform/
**/.terraform.lock.hcl
**/terraform.tfstate*
*.tfvars
!*.tfvars.example
```

---

## 📊 最終成果物

### ファイル構成

```
terraform/
├── README.md                     # セットアップガイド
├── VALIDATION.md                 # 検証手順
├── environments/
│   ├── prod/                     # 本番環境（10サイト）
│   │   ├── provider.tf
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars.example
│   └── dev/                      # 開発環境（3サイト）
│       └── (同上)
├── modules/
│   ├── network/                  # 5ファイル
│   ├── iam/                      # 3ファイル
│   ├── filestore/                # 3ファイル
│   ├── database/                 # 4ファイル
│   ├── compute/                  # 6ファイル
│   ├── loadbalancer/             # 6ファイル
│   └── monitoring/               # 3ファイル
└── scripts/
    └── startup_script.sh         # 438行
```

**統計:**
- **総ファイル数**: 46ファイル
- **総行数**: 2,525行
- **モジュール数**: 7モジュール
- **環境数**: 2環境（prod/dev）

### terraform plan結果

| 環境 | サイト数 | リソース数 | エラー |
|------|---------|-----------|--------|
| prod | 10 | 94 | 0 |
| dev | 3 | 59 | 0 |

---

## 🎓 学んだこと

### 1. Terraformのバージョン選択

- **1.5.7**: MPL 2.0（完全オープンソース）
- **1.13.5**: BUSL（商用制限あり、自社利用はOK）
- **OpenTofu**: Apache 2.0（完全オープンソース、互換性あり）

**判断基準:**
- 自社インフラ管理 → BUSLでも問題なし
- 競合サービス提供 → OpenTofuを検討

### 2. Service Networkingの重要性

Cloud SQLやFilestoreをPrivate IPで使うには：

```
1. API有効化（servicenetworking.googleapis.com）
2. IP範囲予約（google_compute_global_address）
3. VPC Peering（google_service_networking_connection）
```

この順序を守らないとエラーになる。

### 3. 最小権限の原則

`roles/secretmanager.admin`のような強力なロールは避ける：

```
❌ admin（何でもできる）
✅ accessor + creator + versionAdder（必要最小限）
```

**セキュリティ効果:**
- VM侵害時でもSecret削除不可
- IAM設定変更不可
- 監査証跡が残る

### 4. 変数駆動の威力

`length(var.domains)`でサイト数を決定する設計：

```hcl
locals {
  site_count = length(var.domains)
}

resource "google_sql_database" "wordpress_sites" {
  count = local.site_count
  # ...
}
```

**メリット:**
- ドメイン追加 = terraform applyだけ
- 手動設定ミスゼロ
- コードの再利用性が高い

### 5. terraform planの重要性

| ステップ | 検証内容 | GCP接続 |
|---------|---------|---------|
| `terraform fmt` | コードスタイル | 不要 |
| `terraform validate` | **構文チェックのみ** | 不要 |
| `terraform plan` | **実際のリソース作成計画** | **必須** |

**terraform planで初めて:**
- 実際のGCP APIとの整合性を確認
- リソース名の衝突を検出
- 依存関係の問題を発見

---

## 🚀 次のステップ

### Phase 1: デプロイ準備

```bash
# terraform.tfvarsに実際のドメインを設定
vi environments/prod/terraform.tfvars

# 実行計画の保存
terraform plan -out=tfplan

# 実際のデプロイ（慎重に！）
terraform apply tfplan
```

### Phase 2: Ansible統合

- WordPress細かい設定
- Wazuh Manager構築
- SSL証明書の詳細設定
- バックアップスクリプト

### Phase 3: AIエージェント統合

- Slack通知連携
- LLM自動対応
- 異常検知 → 自動修復

---

## 💡 実装のポイント

### 成功の秘訣

1. **設計書を丁寧に作る** - 実装前の設計が8割
2. **terraform planで検証** - validateだけでは不十分
3. **段階的な実装** - モジュール単位で確認
4. **エラーを恐れない** - エラーから学ぶことが多い
5. **最小権限の徹底** - セキュリティは最初から

### ハマりポイント

1. **Service Networkingの依存関係** - API有効化を忘れずに
2. **Secret Manager構文変更** - `automatic = true` → `auto {}`
3. **MySQL vs PostgreSQL** - `binary_log_enabled` vs `point_in_time_recovery_enabled`
4. **`.terraform/`のコミット** - .gitignoreを最初に設定
5. **terraform.tfvarsの管理** - 機密情報を絶対にコミットしない

---

## まとめ

今回は、**設計書からTerraform実装、terraform plan検証まで**を完走しました。

**成果:**
- ✅ 7モジュール、2,525行のコード実装
- ✅ terraform plan成功（prod: 94, dev: 59リソース）
- ✅ エラー0件
- ✅ コミット＆プッシュ完了

**学び:**
- Service Networkingの重要性
- 最小権限の原則（Secret Manager 3ロール分離）
- terraform planの必須性
- ドメインリスト駆動の設計

**次回予告:**
- **terraform apply編** - 実際にGCPリソースをデプロイし、WordPressを動かす
- コスト監視の実装
- 実運用での知見共有

---

## 参考リンク

- [infra-ai-agent リポジトリ](https://github.com/0xchoux1/infra-ai-agent)
  - Terraform実装: `terraform/`
  - 設計書: `docs/terraform-design.md`
  - 要件定義: `docs/requirements.md`
- [前回の記事: 設計編](../2025-11-10-gcp-wordpress-multitenancy-design/)
- [Terraform公式ドキュメント](https://developer.hashicorp.com/terraform/docs)
- [Google Cloud Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)

---

**前回の記事**: [GCP上でWordPressマルチテナント環境を設計する](../2025-11-10-gcp-wordpress-multitenancy-design/)

**次回予告**: Terraform Apply編 - 実際のデプロイと運用開始

