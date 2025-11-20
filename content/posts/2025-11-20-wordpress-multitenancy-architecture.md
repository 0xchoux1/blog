---
title: "GCP上でWordPress マルチテナント環境を構築 - 10サイトを支えるアーキテクチャ設計"
date: 2025-11-20T14:00:00+09:00
draft: false
tags: ["GCP", "WordPress", "アーキテクチャ", "設計", "コスト最適化"]
categories: ["技術", "アーキテクチャ"]
series: ["WordPress マルチテナント環境構築"]
---


## はじめに

「WordPressを10サイト運用したい」

この要件に対して、どのようなインフラ構成を選びますか？

- 各サイトごとに独立したサーバーを立てる？
- 1台のサーバーにすべてを詰め込む？
- マネージドサービスを使う？

この記事では、**GCP上でWordPress マルチテナント環境（10サイト）を構築**した実例を紹介します。単なる技術選定だけでなく、**なぜその選択をしたのか**という意思決定プロセスも含めて解説します。

**この記事で扱う内容**:
1. 要件定義とアーキテクチャ選定理由
2. システム全体構成
3. コスト試算と最適化
4. セキュリティ設計
5. スケーラビリティと可用性

---

## 1. 要件定義

### ビジネス要件

```yaml
目的:
  - 10個のWordPressサイトをホスティング
  - 各サイトは独立したドメイン
  - 技術ブログ・情報サイトとして運用

トラフィック想定:
  - 初期: 各サイト 1,000 PV/day
  - 成長後: 各サイト 10,000 PV/day

運用方針:
  - Infrastructure as Code で管理
  - 自動化重視
  - コスト効率重視
```

### 技術要件

```yaml
パフォーマンス:
  □ ページロード時間: 3秒以内
  □ TTFB (Time to First Byte): 500ms以内
  □ 同時アクセス: 各サイト100ユーザー

可用性:
  □ 稼働率: 99.5%以上
  □ 自動復旧機能
  □ バックアップ: 日次

セキュリティ:
  □ SSL/TLS必須
  □ DDoS対策
  □ ファイアウォール
  □ アクセス制御（IAP）

運用性:
  □ 監視・アラート
  □ ログ集約
  □ デプロイ自動化
  □ スケーリング自動化
```

---

## 2. アーキテクチャ選定

### 検討した3つのパターン

#### パターンA: シングルテナント（各サイトごとに独立サーバー）

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Site 1       │  │ Site 2       │  │ Site 10      │
│ VM + MySQL   │  │ VM + MySQL   │  │ VM + MySQL   │
└──────────────┘  └──────────────┘  └──────────────┘
```

**メリット**:
- 完全な独立性
- 障害の影響範囲が限定的

**デメリット**:
- コストが高い（VM 10台 + MySQL 10インスタンス）
- 運用管理コストが高い

**コスト試算**: 約 $500/月

**判定**: ❌ コストが高すぎる

#### パターンB: モノリシック（1台のサーバーにすべて）

```
┌────────────────────────────┐
│  Single VM                 │
│  ┌──────────────────────┐  │
│  │ Site 1, 2, ... 10    │  │
│  │ MySQL                │  │
│  └──────────────────────┘  │
└────────────────────────────┘
```

**メリット**:
- コストが最小
- シンプルな構成

**デメリット**:
- SPOFになる
- スケーリングが困難
- 障害時の影響範囲が大きい

**コスト試算**: 約 $50/月

**判定**: ❌ 可用性とスケーラビリティが不十分

#### パターンC: マルチテナント + オートスケーリング（採用）

```
                    ┌─────────────────┐
                    │ Cloud CDN       │
                    │ (Cloudflare)    │
                    └────────┬────────┘
                             ▼
                    ┌─────────────────┐
                    │ Cloud Load      │
                    │ Balancer        │
                    └────────┬────────┘
                             ▼
        ┌────────────────────┴────────────────────┐
        │                                         │
   ┌────▼────┐  ┌──────────┐  ┌──────────┐      │
   │ VM 1    │  │ VM 2     │  │ VM 3     │      │
   │ (site*) │  │ (site*)  │  │ (site*)  │      │
   └────┬────┘  └────┬─────┘  └────┬─────┘      │
        │            │              │             │
        └────────────┴──────────────┴─────────────┘
                     │              │
          ┌──────────▼───┐  ┌──────▼──────────┐
          │ Cloud SQL    │  │ Cloud Filestore │
          │ (MySQL 8.0)  │  │ (NFS)           │
          └──────────────┘  └─────────────────┘
```

**メリット**:
- 適度なコスト
- オートスケーリング対応
- マネージドサービス活用

**デメリット**:
- 構成が複雑
- 初期構築コストが高い

**コスト試算**: 約 $150/月

**判定**: ✅ バランスが良い

---

## 3. システム全体構成

### コンポーネント一覧

| コンポーネント | GCPサービス | 役割 | 冗長化 |
|--------------|------------|------|--------|
| Webサーバー | Compute Engine | WordPress実行 | MIG (3-10台) |
| データベース | Cloud SQL | MySQL 8.0 | マスター/スタンバイ可能 |
| 共有ストレージ | Cloud Filestore | WordPressファイル共有 | NFSレプリケーション可能 |
| ロードバランサ | Cloud Load Balancing | トラフィック分散 | グローバル冗長 |
| CDN | Cloudflare | 静的コンテンツキャッシュ | グローバル分散 |
| DNS | Cloud DNS | ドメイン管理 | Anycast |
| 証明書 | Let's Encrypt | SSL/TLS | 自動更新 |
| シークレット | Secret Manager | パスワード管理 | - |
| 監視 | Cloud Monitoring | メトリクス収集 | - |
| ログ | Cloud Logging | ログ集約 | - |

### ネットワーク構成

```
VPC: prod-vpc (10.0.0.0/16)

Subnets:
  - web-subnet (10.0.1.0/24)     # Webサーバー用
  - mgmt-subnet (10.0.2.0/24)    # 管理用（将来）
  - db-subnet (10.168.0.0/20)    # Cloud SQL (自動作成)
  - nfs-subnet (10.0.3.0/24)     # Cloud Filestore

Private Service Connection:
  - Cloud SQL用プライベート接続
  - Cloud Filestore用プライベート接続

Firewall Rules:
  - allow-health-check (35.191.0.0/16 → 80, 443)
  - allow-iap-ssh (35.235.240.0/20 → 22)
  - deny-all-ingress (デフォルト拒否)
```

### ストレージ構成

```
Cloud Filestore: /wordpress (1TB, BASIC_HDD)
  ├── site1/              # ai-jisso.tech
  │   ├── wp-config.php
  │   ├── wp-content/
  │   └── ...
  ├── site2/              # dev-ops.tech
  ├── site3/              # cloud-native.tech
  ├── ...
  └── site10/             # edge-computing.tech

Cloud SQL: prod-wordpress-db
  ├── wordpress_db_1      # site1用
  ├── wordpress_db_2      # site2用
  ├── ...
  └── wordpress_db_10     # site10用

ユーザー:
  ├── wp_user_1 → wordpress_db_1
  ├── wp_user_2 → wordpress_db_2
  ├── ...
  └── wp_user_10 → wordpress_db_10
```

### Nginx仮想ホスト構成

各VMは10サイトすべてをホストします：

```nginx
# /etc/nginx/sites-available/wordpress.conf

# Site 1
server {
    listen 80;
    server_name ai-jisso.tech;
    root /var/www/wordpress/site1;
    # ... (共通設定)
}

# Site 2
server {
    listen 80;
    server_name dev-ops.tech;
    root /var/www/wordpress/site2;
    # ... (共通設定)
}

# ... (site3 ~ site10)
```

---

## 4. 技術選定理由

### なぜCompute Engineか？

**候補**:
- Google Kubernetes Engine (GKE)
- Cloud Run
- Compute Engine

**選定理由**:
```yaml
GKE:
  メリット: コンテナオーケストレーション、宣言的管理
  デメリット:
    - オーバーエンジニアリング（10サイトには大きすぎる）
    - 学習コストが高い
    - コストが高い（ノードプール + コントロールプレーン）

Cloud Run:
  メリット: サーバーレス、完全マネージド
  デメリット:
    - ステートフル (WordPress) との相性が悪い
    - NFSマウントの制約
    - MySQL接続のコネクションプール管理が複雑

Compute Engine: ✅
  メリット:
    - シンプル
    - NFSマウントが容易
    - オートスケーリング対応
    - コストが適切
  デメリット:
    - OS管理が必要（Ansibleで自動化）
```

### なぜCloud SQLか？

**候補**:
- 自前のMySQL on Compute Engine
- Cloud SQL

**選定理由**:
```yaml
自前MySQL:
  メリット: コストが安い、完全な制御
  デメリット:
    - バックアップ管理が必要
    - 高可用性構成が複雑
    - セキュリティパッチ適用が手動

Cloud SQL: ✅
  メリット:
    - 自動バックアップ
    - 高可用性オプション
    - 自動パッチ適用
    - Private IP接続
    - ポイントインタイムリカバリ
  デメリット:
    - コストがやや高い（→ 小さいインスタンスで対応）
```

### なぜCloud Filestoreか？

**候補**:
- Cloud Storage (GCS)
- Persistent Disk
- Cloud Filestore

**選定理由**:
```yaml
Cloud Storage:
  メリット: 安価、無制限容量
  デメリット:
    - WordPressとの互換性問題
    - FUSE経由のマウントでパフォーマンス低下
    - ファイルロックの制約

Persistent Disk:
  メリット: 高パフォーマンス
  デメリット:
    - 1つのVMにしかアタッチできない
    - マルチVMでの共有が不可能

Cloud Filestore: ✅
  メリット:
    - NFSプロトコル（標準的）
    - 複数VMから同時マウント可能
    - WordPressとの完全互換性
    - 高パフォーマンス
  デメリット:
    - コストがやや高い（→ BASIC_HDDで対応）
```

### なぜCloudflareか？

**候補**:
- Cloud CDN
- Cloudflare

**選定理由**:
```yaml
Cloud CDN:
  メリット: GCPネイティブ、設定が簡単
  デメリット:
    - DDoS対策が限定的
    - WAF機能がない（Cloud Armorが別途必要）
    - DNS管理が別サービス

Cloudflare: ✅
  メリット:
    - 無料プランで十分な機能
    - DDoS対策が標準装備
    - WAF機能
    - DNS管理も一元化
    - SSL証明書自動管理
  デメリット:
    - GCPの外部サービス
```

---

## 5. コスト試算

### 月額コスト内訳（本番環境）

| 項目 | スペック | 月額料金 | 備考 |
|------|---------|---------|------|
| **Compute Engine** | | | |
| - VM (e2-micro × 3台) | 2vCPU, 1GB RAM | $21 | 最小構成 |
| - Persistent Disk (10GB × 3台) | Standard | $1 | OS用 |
| **Cloud SQL** | | | |
| - インスタンス (db-f1-micro) | 1vCPU, 0.6GB RAM | $15 | 最小構成 |
| - ストレージ (10GB) | SSD | $2 | |
| - バックアップ (10GB) | | $1 | 7日保持 |
| **Cloud Filestore** | | | |
| - 1TB BASIC_HDD | | $51 | 最小容量 |
| **Cloud Load Balancing** | | | |
| - フォワーディングルール | | $18 | |
| - データ転送 (100GB) | | $8 | |
| **Cloud Logging** | | | |
| - ログ保存 (10GB) | | $1 | |
| **Cloud Monitoring** | | | |
| - メトリクス | | $2 | |
| **Secret Manager** | | | |
| - 10シークレット | | $0.06 | ほぼ無料 |
| **Cloudflare** | | | |
| - Free プラン | | $0 | 無料 |
| **合計** | | **$120** | |

### コスト最適化のポイント

#### 1. Computeインスタンスの選択

```yaml
# 最小構成（採用）
machine_type: e2-micro
cost: $7/月/台

# 中間構成
machine_type: e2-small
cost: $14/月/台

# 高性能構成
machine_type: e2-medium
cost: $28/月/台
```

**判断**: e2-microで開始し、必要に応じてスケールアップ

#### 2. Cloud SQLのティア選択

```yaml
# 最小構成（採用）
tier: db-f1-micro
cost: $15/月

# 推奨構成
tier: db-g1-small
cost: $35/月

# 本番推奨
tier: db-custom-2-7680
cost: $85/月
```

**判断**: 初期はdb-f1-micro、トラフィック増加後にスケールアップ

#### 3. Cloud Filestoreのティア選択

```yaml
# 最小容量（採用）
tier: BASIC_HDD
capacity: 1TB
cost: $51/月

# 高性能
tier: BASIC_SSD
capacity: 1TB
cost: $204/月
```

**判断**: BASIC_HDDで十分なパフォーマンス

#### 4. コミットメント割引

```yaml
# 1年コミットメント
割引率: 25%
節約額: $30/月

# 3年コミットメント
割引率: 52%
節約額: $62/月
```

**トラフィック成長時のコスト予測**:

| トラフィック | VM台数 | 月額コスト | 備考 |
|------------|--------|----------|------|
| 10,000 PV/day | 3台 | $120 | 初期構成 |
| 50,000 PV/day | 5台 | $140 | スケールアップ |
| 100,000 PV/day | 7台 | $160 | |
| 500,000 PV/day | 10台 | $190 | 最大構成 |

---

## 6. セキュリティ設計

### 多層防御アーキテクチャ

```
Layer 7: Application (WordPress)
  □ プラグインの脆弱性管理
  □ 定期アップデート
  □ セキュリティプラグイン

Layer 6: CDN/WAF (Cloudflare)
  □ DDoS対策
  □ WAFルール
  □ レート制限

Layer 5: Load Balancer (Cloud LB)
  □ SSL/TLS終端
  □ ヘッダー検証

Layer 4: Network (VPC)
  □ ファイアウォールルール
  □ プライベートIP
  □ サブネット分離

Layer 3: Compute (VM)
  □ OS自動パッチ
  □ 最小権限の原則
  □ IAP SSH

Layer 2: Data (Cloud SQL)
  □ 暗号化（保存時・転送時）
  □ プライベートIP
  □ 自動バックアップ

Layer 1: Identity (IAM)
  □ Service Account
  □ 最小権限ロール
  □ Workload Identity
```

### ファイアウォールルール

```hcl
# Health Check許可
resource "google_compute_firewall" "allow_health_check" {
  name    = "allow-health-check"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = [
    "35.191.0.0/16",    # Cloud Load Balancing
    "130.211.0.0/22"    # Legacy health checks
  ]

  target_tags = ["web-server"]
}

# IAP SSH許可
resource "google_compute_firewall" "allow_iap_ssh" {
  name    = "allow-iap-ssh"
  network = google_compute_network.vpc.name

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["35.235.240.0/20"]  # IAP
  target_tags   = ["web-server"]
}

# デフォルト拒否
resource "google_compute_firewall" "deny_all" {
  name     = "deny-all-ingress"
  network  = google_compute_network.vpc.name
  priority = 65534

  deny {
    protocol = "all"
  }

  source_ranges = ["0.0.0.0/0"]
}
```

### IAM設計

```hcl
# Webサーバー用Service Account
resource "google_service_account" "web_server" {
  account_id   = "${var.env}-web-server"
  display_name = "Web Server Service Account"
}

# 必要最小限のロール
resource "google_project_iam_member" "web_server_roles" {
  for_each = toset([
    "roles/logging.logWriter",           # ログ書き込み
    "roles/monitoring.metricWriter",     # メトリクス書き込み
    "roles/cloudsql.client",             # Cloud SQL接続
    "roles/secretmanager.secretAccessor" # Secret Manager読み取り
  ])

  project = var.project_id
  role    = each.key
  member  = "serviceAccount:${google_service_account.web_server.email}"
}
```

---

## 7. スケーラビリティと可用性

### オートスケーリング設定

```hcl
resource "google_compute_region_autoscaler" "web" {
  name   = "${var.env}-web-autoscaler"
  region = var.region
  target = google_compute_region_instance_group_manager.web.id

  autoscaling_policy {
    min_replicas    = 3
    max_replicas    = 10
    cooldown_period = 60

    cpu_utilization {
      target = 0.6  # CPU使用率60%で スケールアウト
    }

    load_balancing_utilization {
      target = 0.8  # 負荷分散使用率80%でスケールアウト
    }
  }
}
```

### 高可用性構成

```yaml
Compute Engine:
  □ Managed Instance Group
  □ 複数ゾーンに分散配置可能
  □ 自動復旧（ヘルスチェック失敗時）
  □ ローリングアップデート対応

Cloud SQL:
  □ 高可用性構成オプション（マスター/スタンバイ）
  □ 自動フェイルオーバー
  □ ポイントインタイムリカバリ
  □ 自動バックアップ

Cloud Filestore:
  □ ゾーンレベルの冗長性
  □ 自動スナップショット

Cloud Load Balancer:
  □ グローバル冗長
  □ 自動フェイルオーバー
```

### ヘルスチェック設定

```hcl
resource "google_compute_health_check" "web" {
  name = "${var.env}-web-health-check"

  http_health_check {
    port         = 80
    request_path = "/health"  # ヘルスチェック用エンドポイント
  }

  check_interval_sec  = 10
  timeout_sec         = 5
  healthy_threshold   = 2
  unhealthy_threshold = 3
}
```

---

## 8. 監視とアラート

### 監視メトリクス

```yaml
Compute Engine:
  - CPU使用率
  - メモリ使用率
  - ディスクI/O
  - ネットワークトラフィック

Cloud SQL:
  - クエリ実行時間
  - 接続数
  - レプリケーション遅延（高可用性時）
  - ストレージ使用率

Cloud Filestore:
  - IOPS
  - スループット
  - ストレージ使用率

Application:
  - レスポンスタイム
  - エラー率
  - リクエスト数
```

### アラートポリシー

```hcl
resource "google_monitoring_alert_policy" "cpu_high" {
  display_name = "${var.env} - High CPU Usage"

  conditions {
    display_name = "CPU usage above 80%"

    condition_threshold {
      filter          = "resource.type = \"gce_instance\""
      comparison      = "COMPARISON_GT"
      threshold_value = 0.8
      duration        = "300s"

      aggregations {
        alignment_period   = "60s"
        per_series_aligner = "ALIGN_MEAN"
      }
    }
  }

  notification_channels = [google_monitoring_notification_channel.email.name]
}
```

---

## 9. デプロイフロー

### 初回構築

```bash
# 1. Terraformでインフラ構築
cd terraform/environments/prod
terraform init
terraform plan
terraform apply

# 2. Ansibleでアプリケーションデプロイ
cd ../../ansible
ansible-playbook -i inventory/gcp.yml playbooks/deploy-wordpress.yml \
  -e "db_host=10.168.0.2" \
  -e "nfs_ip=10.0.3.2" \
  -e "nfs_path=/wordpress"

# 3. 各サイトのWordPressインストール
for i in {1..10}; do
  gcloud compute ssh prod-web-$(hostname) \
    --zone=asia-northeast1-a \
    --tunnel-through-iap \
    --command="sudo /usr/local/bin/setup-wordpress-site.sh $i domain$i.tech 'Site $i'"
done
```

### 更新デプロイ

```bash
# アプリケーション設定の更新
cd ansible
ansible-playbook -i inventory/gcp.yml playbooks/deploy-wordpress.yml \
  --tags nginx  # Nginxだけ更新

# インフラ設定の更新
cd terraform/environments/prod
terraform apply
```

---

## まとめ

### アーキテクチャ選定の判断基準

| 要素 | 重視度 | 選択結果 |
|------|--------|---------|
| コスト | ⭐⭐⭐⭐ | マネージドサービスとVMのバランス |
| パフォーマンス | ⭐⭐⭐ | NFSとCDNで対応 |
| スケーラビリティ | ⭐⭐⭐⭐ | オートスケーリング |
| 運用性 | ⭐⭐⭐⭐⭐ | IaCで完全自動化 |
| セキュリティ | ⭐⭐⭐⭐⭐ | 多層防御 |

### 成功のポイント

1. **マネージドサービスの活用**
   - Cloud SQL: バックアップ・パッチ自動化
   - Cloud Filestore: NFSの簡単な共有
   - Cloud Load Balancer: グローバル冗長

2. **Infrastructure as Code**
   - Terraform: インフラ層
   - Ansible: 構成管理層
   - 完全な再現性

3. **コスト最適化**
   - 最小構成から開始
   - オートスケーリングで必要な時だけ拡張
   - コミットメント割引の活用

4. **セキュリティファースト**
   - プライベートIP接続
   - 最小権限の原則
   - 多層防御

### 今後の改善予定

```yaml
Phase 2:
  □ Cloud CDN統合（現在はCloudflare）
  □ Cloud Armorの導入（WAF）
  □ 高可用性構成（Cloud SQL）

Phase 3:
  □ CI/CDパイプライン構築
  □ Blue-Greenデプロイ
  □ カナリアリリース

Phase 4:
  □ マルチリージョン展開
  □ Wazuhによるセキュリティ監視
  □ コンテナ化の検討
```

---

## 参考リンク

- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [WordPress on Google Cloud](https://cloud.google.com/solutions/wordpress)
- [Cloud SQL Best Practices](https://cloud.google.com/sql/docs/mysql/best-practices)
- [Compute Engine Autoscaling](https://cloud.google.com/compute/docs/autoscaler)

---

## この記事のコード

GitHub: [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)

Terraform:
- [terraform/environments/prod/](https://github.com/0xchoux1/infra-ai-agent/tree/main/terraform/environments/prod)
- [terraform/modules/](https://github.com/0xchoux1/infra-ai-agent/tree/main/terraform/modules)

Ansible:
- [ansible/playbooks/deploy-wordpress.yml](https://github.com/0xchoux1/infra-ai-agent/blob/main/ansible/playbooks/deploy-wordpress.yml)

---

**この記事が役に立ったら**: GitHub Starをいただけると嬉しいです！ [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)
