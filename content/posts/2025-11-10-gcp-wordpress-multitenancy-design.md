---
title: "GCP上でWordPressマルチテナント環境を設計する【要件定義からTerraform設計まで完全解説】"
date: 2025-11-10T15:00:00+09:00
draft: false
tags: ["GCP", "WordPress", "Terraform", "マルチテナント", "インフラ設計", "Cloud SQL", "Filestore", "Cloud CDN", "セキュリティ"]
categories: ["技術", "インフラ設計"]
series: ["AIエージェント開発"]
---

## はじめに

前回の記事「[Google Cloud Platform上でAIエージェントによるインフラ自動運用環境を構築する](../2025-11-04-gcp-infra-ai-agent-setup/)」では、GCP上でAIエージェントの基盤を構築しました。

今回は、**実際にホスティングサービスとして運用可能なWordPressマルチテナント環境**の設計に挑戦します。単なるWordPressホスティングではなく、**HA構成、動的キャッシュ、WAF、セキュリティ監視まで含めた本格的なプロダクション環境**を、要件定義から詳細なTerraform設計まで一気に作り上げます。

## 設計の背景と目標

### やりたいこと

- 10サイト程度のWordPressをマルチテナントでホスティング
- 小規模トラフィック（同時接続100程度）から始める
- GCPのマネージドサービスを活用してHA構成を実現
- セキュリティとコストのバランスを取る
- 将来的な拡張性を確保

### 制約条件

- **予算**: 無料枠4万円を活用、長期的には月3,000円目標
- **トラフィック**: 初期は小規模、スケーラビリティ確保
- **学習目的**: GCPネイティブサービスの理解を深めたい

## Phase 1: 要件定義

詳細な要件定義は`docs/requirements.md`に記載していますが、ここでは主要なポイントを紹介します。

### 機能要件

#### 1. マルチテナント構成

```
10サイトの独立したWordPress環境
├── サイト1: example1.com
├── サイト2: example2.com
...
└── サイト10: example10.com

各サイトごとに:
- 独立したデータベース
- 独立したWordPressファイルディレクトリ
- Nginx仮想ホスト
```

#### 2. 高可用性（HA）アーキテクチャ

```
[Cloud Load Balancer (Global)]
         ↓
[Managed Instance Group (Auto Scaling)]
    ↓           ↓
  VM1 ←───────→ VM2 (Filestore NFS共有)
    ↓
[Cloud SQL (Primary + Standby)]
```

**ポイント**:
- Managed Instance Group による自動スケーリング・ヘルスチェック
- Cloud SQL のリージョナルHA（Primary + Standby）
- Filestore NFS による WordPress ファイルの共有ストレージ

#### 3. パフォーマンス最適化

**2層キャッシュ戦略**:

```
[ユーザー] → [Cloud CDN (L1 Cache)] → [Nginx + PHP-FPM + OPcache (L2 Cache)] → [Cloud SQL]
```

| 層 | 役割 | キャッシュ対象 | TTL |
|----|------|---------------|-----|
| **L1: Cloud CDN** | グローバルエッジキャッシュ | 静的コンテンツ + 動的ページ | Cache-Control準拠 |
| **L2: OPcache** | PHPオペコードキャッシュ | コンパイル済みPHP | メモリ上で永続化 |

**動的コンテンツのキャッシュ**:

Cloud CDNは`USE_ORIGIN_HEADERS`モードで動作し、WordPressが出力する`Cache-Control`ヘッダーに従ってキャッシュします：

```php
// WordPressプラグインでの実装例
header('Cache-Control: public, max-age=300, s-maxage=600');
```

- 認証済みユーザー: `Cache-Control: private, no-cache`（キャッシュなし）
- ゲストユーザー: `Cache-Control: public, max-age=300`（5分間キャッシュ）

**キャッシュ整合性の確保**:

```
記事更新時:
1. WordPress側でCDNパージAPIを呼び出し
2. Cloud CDN → 該当URLのキャッシュクリア
3. OPcache → 次回アクセス時に自動再コンパイル
```

### 非機能要件

#### 1. セキュリティ

**多層防御アーキテクチャ**:

```
[外部] → [Cloud Armor (WAF)] → [Cloud Load Balancer]
            ↓
      [Private VPC]
            ↓
   [VM (内部IP のみ)]
            ↓
   [Wazuh Agent (SIEM)]
```

**セキュリティコンポーネント**:

| コンポーネント | 役割 | 実装内容 |
|-------------|------|---------|
| **Cloud Armor** | WAF | OWASP Top 10対策、地理的アクセス制限、DDoS防御 |
| **Private VPC** | ネットワーク分離 | 外部IP無効化、Cloud NAT経由での外部通信 |
| **Wazuh** | SIEM | ファイル整合性監視、脆弱性スキャン、ログ分析 |
| **Secret Manager** | 機密情報管理 | DBパスワード、WordPress管理者パスワード |

**最小権限の原則**:

```hcl
# Web ServerサービスアカウントのIAM権限
roles/secretmanager.secretAccessor       // DB読み取り専用
roles/secretmanager.secretCreator        // Secret新規作成のみ
roles/secretmanager.secretVersionAdder   // バージョン追加のみ
roles/cloudsql.client                    // Cloud SQL接続
roles/file.editor                        // Filestoreアクセス
```

❌ 削除: `roles/secretmanager.admin`（過剰権限）

**攻撃シナリオへの耐性**:
- VM侵害時でもSecret削除・IAM設定変更は不可能
- 横展開: 他のSecretへのアクセス不可
- 証跡消去: Secret削除不可で痕跡が残る

#### 2. バックアップ

```
06:00 JST: DBスナップショット作成
   ↓ (5分以内)
06:05 JST: VM永続ディスクスナップショット作成

保持期間: 2日間（4世代）
リストア手順: Terraform変数で指定
```

**整合性の確保**:
- DBとVMのスナップショット間隔を5分以内に設定
- リストア時はペアで復元（DB 06:00 + VM 06:05）

#### 3. 監視・アラート

```
[Cloud Monitoring] → [Slack Webhook] → [LLMエージェント]
                           ↓
                    自律的なアクション
```

**Phase 1のアラート対象**:
- ✅ HTTP接続失敗（サービス監視）
- ✅ ヘルスチェック失敗
- ⏱️ レスポンス速度（Phase 2以降）

### SSL証明書管理

```
[Cloud DNS] ← Let's Encrypt DNS-01 Challenge
     ↓
[Load Balancer SSL証明書]
     ↓
自動更新（certbotでcron実行）
```

## Phase 2: Terraform設計

詳細設計は`docs/terraform-design.md`（2,200行超）に記載していますが、ここでは設計の核心部分を解説します。

### モジュール構成

```
terraform/
├── environments/
│   ├── prod/
│   │   ├── main.tf          # モジュール呼び出し
│   │   ├── variables.tf     # 共通変数定義
│   │   └── terraform.tfvars # 本番環境値
│   └── dev/
│       └── terraform.tfvars # 開発環境値
├── modules/
│   ├── network/             # VPC、サブネット、NAT
│   ├── compute/             # MIG、Instance Template
│   ├── database/            # Cloud SQL
│   ├── loadbalancer/        # LB、Cloud CDN、Cloud Armor
│   ├── filestore/           # NFS共有ストレージ
│   ├── monitoring/          # アラート、ログ
│   └── iam/                 # サービスアカウント、権限
└── scripts/
    └── startup_script.sh    # VM初期化スクリプト
```

**設計原則**:
- **再利用性**: 環境（prod/dev）間で同じモジュールを使用
- **変数駆動**: ドメインリスト1つで全リソース生成
- **依存関係の明確化**: `depends_on`で順序を保証

### 重要な技術的決定

#### 1. Service Networking（Cloud SQL Private IP）

**課題**: Cloud SQLをPrivate IPで接続するには、VPCとのピアリングが必要。

**解決策**:

```hcl
# modules/network/service_networking.tf

# Service Networking API有効化
resource "google_project_service" "servicenetworking" {
  service = "servicenetworking.googleapis.com"
}

# Private IP範囲を予約（/16）
resource "google_compute_global_address" "private_ip_address" {
  name          = "private-ip-address"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = google_compute_network.vpc.id
}

# Service Networking接続を作成
resource "google_service_networking_connection" "private_vpc_connection" {
  network                 = google_compute_network.vpc.id
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = [google_compute_global_address.private_ip_address.name]
  
  depends_on = [google_project_service.servicenetworking]
}
```

**依存関係の解決**:

```hcl
# environments/prod/main.tf

module "database" {
  source = "../../modules/database"
  
  # 重要: Service Networking接続が完了してからCloud SQL作成
  depends_on = [module.network]
}
```

❌ **NGパターン**: `depends_on = [module.network.service_networking_connection]`  
（Terraformはモジュール出力への`depends_on`を許可しない）

✅ **OKパターン**: `depends_on = [module.network]`  
（モジュール全体への依存）

#### 2. Filestore（共有ストレージ）

**課題**: MIGでVMが複数台起動する際、WordPress ファイルを共有する必要がある。

**検討した選択肢**:
- ❌ **永続ディスク**: Regional MIGではstateful設定不可
- ✅ **Filestore**: NFSマウントで複数VMから同時アクセス可能

**実装**:

```hcl
# modules/filestore/main.tf

# Filestore用IP範囲を予約（/29）
resource "google_compute_global_address" "filestore_ip" {
  name          = "filestore-ip-range"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 29
  network       = var.network_id
}

resource "google_filestore_instance" "wordpress" {
  name     = "wordpress-filestore"
  tier     = "BASIC_HDD"  # コスト最適化
  
  file_shares {
    name        = "wordpress"
    capacity_gb = 1024
  }
  
  networks {
    network           = var.network_id  # フルリソース名が必須
    modes             = ["MODE_IPV4"]
    connect_mode      = "DIRECT_PEERING"
    reserved_ip_range = google_compute_global_address.filestore_ip.name
  }
  
  depends_on = [google_project_service.filestore]
}
```

**起動スクリプトでのマウント**:

```bash
# scripts/startup_script.sh

# Filestoreマウント
FILESTORE_IP="${filestore_ip}"
FILESTORE_SHARE="wordpress"

apt-get install -y nfs-common
mkdir -p /mnt/wordpress
mount -t nfs $FILESTORE_IP:/$FILESTORE_SHARE /mnt/wordpress

# fstabに追加（再起動後も自動マウント）
echo "$FILESTORE_IP:/$FILESTORE_SHARE /mnt/wordpress nfs defaults 0 0" >> /etc/fstab
```

#### 3. 動的10サイト生成

**課題**: 10サイト分のNginx設定、DBテーブル、ディレクトリを手動で作るのは非効率。

**解決策**: Terraformの`count`と起動スクリプトのループで自動生成。

**Terraform側（変数駆動）**:

```hcl
# environments/prod/terraform.tfvars

domains = [
  "example1.com",
  "example2.com",
  // ... 10サイト分
]
```

```hcl
# modules/database/databases.tf

locals {
  site_count = length(var.domains)  # ドメイン数から自動算出
}

# 各サイト用のDB作成
resource "google_sql_database" "wordpress_db" {
  count    = local.site_count
  name     = "wordpress_site${count.index + 1}"
  instance = google_sql_database_instance.main.name
}

# 各サイト用のDBユーザー作成
resource "google_sql_user" "wordpress_user" {
  count    = local.site_count
  name     = "wp_user_site${count.index + 1}"
  instance = google_sql_database_instance.main.name
  password = random_password.db_password[count.index].result
}
```

**起動スクリプト側（動的生成）**:

```bash
# scripts/startup_script.sh

# メタデータからドメインリストを取得
DOMAINS_JSON=$(curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/domains)

# 各サイト用のディレクトリ作成
for i in {1..10}; do
  mkdir -p /mnt/wordpress/site${i}
  chown -R www-data:www-data /mnt/wordpress/site${i}
  
  # Nginx仮想ホスト生成
  DOMAIN=$(echo $DOMAINS_JSON | jq -r ".[$((i-1))]")
  cat > /etc/nginx/sites-available/site${i}.conf <<EOF
server {
    listen 80;
    server_name $DOMAIN;
    root /mnt/wordpress/site${i};
    
    location / {
        try_files \$uri \$uri/ /index.php?\$args;
    }
    
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF
  ln -s /etc/nginx/sites-available/site${i}.conf /etc/nginx/sites-enabled/
done

nginx -t && systemctl reload nginx
```

**WordPress自動セットアップスクリプト**:

```bash
# /usr/local/bin/setup-wordpress-site.sh

#!/bin/bash
SITE_NUM=$1
DB_PASSWORD=$(gcloud secrets versions access latest --secret="prod-wordpress-db-password-${SITE_NUM}")
ADMIN_PASSWORD=$(openssl rand -base64 24)

cd /mnt/wordpress/site${SITE_NUM}
wp core download --allow-root
wp config create \
  --dbname="wordpress_site${SITE_NUM}" \
  --dbuser="wp_user_site${SITE_NUM}" \
  --dbpass="$DB_PASSWORD" \
  --dbhost="10.x.x.x" \
  --allow-root

wp core install \
  --url="https://$(echo $DOMAINS_JSON | jq -r ".[${SITE_NUM}-1]")" \
  --title="Site ${SITE_NUM}" \
  --admin_user="admin" \
  --admin_password="$ADMIN_PASSWORD" \
  --admin_email="admin@example.com" \
  --allow-root

# 管理者パスワードをSecret Managerに保存
gcloud secrets create prod-wordpress-admin-password-${SITE_NUM} \
  --data-file=<(echo -n "$ADMIN_PASSWORD") || \
gcloud secrets versions add prod-wordpress-admin-password-${SITE_NUM} \
  --data-file=<(echo -n "$ADMIN_PASSWORD")

echo "✅ Site ${SITE_NUM} セットアップ完了"
echo "管理者パスワードは以下で取得:"
echo "gcloud secrets versions access latest --secret=prod-wordpress-admin-password-${SITE_NUM}"
```

#### 4. Cloud SQL HA設定の環境別パラメータ化

**課題**: 本番環境はHA（REGIONAL）、開発環境は単一VM（ZONAL）でコスト削減したい。

**解決策**:

```hcl
# modules/database/variables.tf

variable "availability_type" {
  description = "Cloud SQL HA設定"
  type        = string
  
  validation {
    condition     = contains(["REGIONAL", "ZONAL"], var.availability_type)
    error_message = "availability_type must be REGIONAL or ZONAL."
  }
}
```

```hcl
# modules/database/main.tf

resource "google_sql_database_instance" "main" {
  name             = "${var.environment}-mysql"
  database_version = "MYSQL_8_0"
  region           = var.region
  
  settings {
    tier              = "db-custom-2-7680"  # HA対応スペック
    availability_type = var.availability_type
    
    backup_configuration {
      enabled            = true
      start_time         = "03:00"
      binary_log_enabled = var.availability_type == "REGIONAL"  # HAの場合のみ有効
    }
    
    ip_configuration {
      ipv4_enabled    = false
      private_network = var.network_id
    }
  }
  
  depends_on = [module.network]
}
```

```hcl
# environments/prod/terraform.tfvars
db_availability_type = "REGIONAL"  # 本番: HA有効

# environments/dev/terraform.tfvars
db_availability_type = "ZONAL"     # 開発: 単一インスタンス
```

### 最小権限の原則（Secret Manager）

**当初の設計（問題あり）**:

```hcl
# ❌ 過剰権限
resource "google_project_iam_member" "web_secret_admin" {
  role = "roles/secretmanager.admin"  # 何でもできてしまう
}
```

**問題点**:
- Secret削除可能
- IAM設定変更可能
- 全Secret一覧取得可能

**改善後（最小権限）**:

```hcl
# modules/iam/main.tf

# ✅ 読み取り専用
resource "google_project_iam_member" "web_secret_accessor" {
  role = "roles/secretmanager.secretAccessor"
}

# ✅ Secret新規作成のみ
resource "google_project_iam_member" "web_secret_creator" {
  role = "roles/secretmanager.secretCreator"
}

# ✅ 既存Secretへのバージョン追加のみ
resource "google_project_iam_member" "web_secret_version_adder" {
  role = "roles/secretmanager.secretVersionAdder"
}
```

**セキュリティ効果**:

| 操作 | admin | 分離後 |
|------|-------|--------|
| Secret読み取り | ✅ | ✅ |
| Secret作成 | ✅ | ✅ |
| バージョン追加 | ✅ | ✅ |
| **Secret削除** | ⚠️ 可能 | ❌ 不可 |
| **IAM設定変更** | ⚠️ 可能 | ❌ 不可 |
| **全Secret一覧** | ⚠️ 可能 | ❌ 不可 |

**攻撃シナリオへの耐性**:
- VM侵害時でもSecret削除・IAM変更不可
- 横展開: 他のSecretへのアクセス不可
- 証跡消去: Secret削除不可で痕跡が残る

### 設計の進化過程（修正履歴）

この設計書は**6回のレビュー**を経て完成しました：

#### v1.1: Cloud SQL Private IP対応
- `google_service_networking_connection`追加
- Filestore NFSによる共有ストレージ実装
- 10サイト動的生成

#### v1.2-v1.4: 依存関係の修正
- `depends_on = [module.network]`の正しい使い方
- Filestore の`network_id`パラメータ修正
- API有効化の依存関係追加

#### v1.5: Cloud SQL HA設定とパスワード管理
- `availability_type`の環境別パラメータ化
- WordPress管理者パスワードのSecret Manager保存

#### v1.6: 最小権限の原則（Secret Manager）
- 権限を3ロールに分解
- セキュリティリスク評価: High → Low

## コスト試算

### Phase 1（初期構成）

| リソース | スペック | 月額（USD） | 月額（円） |
|---------|---------|-----------|-----------|
| Compute Engine (MIG) | e2-small × 2台 | $25 | ¥3,750 |
| Cloud SQL | db-custom-2-7680 (HA) | $180 | ¥27,000 |
| Filestore | BASIC_HDD 1TB | $200 | ¥30,000 |
| Cloud Load Balancer | 100GB/月 | $20 | ¥3,000 |
| Cloud CDN | 100GB/月 | $10 | ¥1,500 |
| Cloud Armor | ポリシー1つ | $5 | ¥750 |
| **合計** | | **$440** | **¥66,000** |

### Phase 2（コスト最適化後）

```
Cloud SQL: db-custom-2-7680 (HA)
   ↓
Cloud SQL: db-f1-micro (Single Zone)
削減: $160/月（¥24,000）
```

**Phase 2合計**: $280/月（¥42,000）

### Phase 3（長期目標）

```
Filestore: BASIC_HDD 1TB
   ↓
VM内蔵ディスク + rsync同期
削減: $200/月（¥30,000）
```

**Phase 3合計**: $80/月（¥12,000）

**最終目標（月3,000円）への道筋**:
- 無料枠4万円を使い切るまでPhase 1で学習
- GCPサービスの理解が深まったらPhase 2へ移行
- 運用が安定したらPhase 3で自前VM構成へ

## 技術的に学んだこと

### 1. Terraformの依存関係は奥が深い

**NG例**:
```hcl
depends_on = [var.some_variable]           # 変数は不可
depends_on = [module.network.output_name]   # 出力値は不可
```

**OK例**:
```hcl
depends_on = [module.network]              # モジュール全体
depends_on = [google_project_service.api]  # リソース
```

### 2. GCPのService Networkingは必須知識

Cloud SQLやFilestoreをPrivate IPで使うには、**VPC Peering**の理解が不可欠：

```
VPC ←(Service Networking)→ Google Service Producer Network
```

- IP範囲を予約（`google_compute_global_address`）
- Service Networking接続を確立（`google_service_networking_connection`）
- API有効化を忘れずに（`google_project_service`）

### 3. 最小権限の原則は細かく設定する

「とりあえず admin ロール」は危険。GCPの細かいロール（`secretAccessor`, `secretCreator`, `secretVersionAdder`）を組み合わせることで、**真の最小権限**を実現できる。

### 4. マルチテナント設計は変数駆動にする

ドメインリスト1つで全リソースを生成する設計にすれば：
- サイト追加は`domains`に1行追加するだけ
- DB、Nginx設定、ディレクトリが自動生成
- 手動設定ミスがゼロに

## 今後の展開

### 次のステップ

1. **Terraform実装**: この設計書をコード化
2. **Ansible Playbook設計**: WordPress細かい設定、Wazuh Agent導入
3. **実環境での検証**: `terraform plan` → `apply`
4. **AIエージェント統合**: 障害検知 → Slack通知 → LLMが自動対応

### 拡張構想

- **CI/CDパイプライン**: GitHubからの自動デプロイ
- **Blue-Green Deployment**: ゼロダウンタイム更新
- **マルチリージョン**: グローバル展開
- **データ分析基盤**: BigQueryとの連携

## まとめ

今回は、**WordPress マルチテナント環境の要件定義からTerraform設計まで**を一気に駆け抜けました。

**得られた成果**:
- ✅ 本格的なプロダクション環境の設計書（2,200行超）
- ✅ HA、セキュリティ、コスト最適化を両立
- ✅ 6回のレビューを経た高品質な設計
- ✅ 最小権限の原則を徹底

**技術的ハイライト**:
- Service Networkingによる Private IP 設計
- Filestore NFSでのマルチVM共有ストレージ
- 変数駆動の動的10サイト生成
- Secret Manager権限の3ロール分離

次回は、この設計書をもとに**実際にTerraformコードを実装**し、`terraform apply`で環境を構築します。お楽しみに！

## 参考資料

- [infra-ai-agent リポジトリ](https://github.com/0xchoux1/infra-ai-agent)
  - `docs/requirements.md`: 詳細な要件定義書
  - `docs/terraform-design.md`: 完全なTerraform設計書
- [Google Cloud Documentation](https://cloud.google.com/docs)
  - [Cloud SQL Private IP](https://cloud.google.com/sql/docs/mysql/configure-private-ip)
  - [Filestore](https://cloud.google.com/filestore/docs)
  - [Secret Manager](https://cloud.google.com/secret-manager/docs)

---

**前回の記事**: [Google Cloud Platform上でAIエージェントによるインフラ自動運用環境を構築する](../2025-11-04-gcp-infra-ai-agent-setup/)

**次回予告**: Terraform実装編 - 設計書をコード化してGCP環境を構築する

