---
title: "Terraform Apply実践編！GCPにWordPress環境をデプロイ【エラー解決の全記録】"
date: 2025-11-16T19:00:00+09:00
draft: false
tags: ["GCP", "Terraform", "WordPress", "実践", "トラブルシューティング", "terraform apply"]
categories: ["技術", "実装"]
series: ["AIエージェント開発"]
---

## はじめに

[前回の記事](../2025-11-16-gcp-wordpress-terraform-implementation/)で、Terraform実装を完了し、`terraform plan`で検証しました。

今回は、**実際にterraform applyを実行してGCPリソースをデプロイ**します。

ところが、予想以上に多くのエラーに遭遇しました。しかし、**一つ一つ丁寧に解決していくことで、最終的にデプロイに成功**しました。

この記事では、**実際に遭遇したエラーとその解決方法を全て記録**します。

---

## 📋 デプロイするリソース

terraform planの結果：

```
Plan: 94 to add, 0 to change, 0 to destroy.
```

**内訳:**
- Network: 11リソース（VPC、NAT、Firewall等）
- IAM: 11リソース（Service Account、権限）
- Filestore: 3リソース（NFS 1TB）
- Database: 33リソース（Cloud SQL HA、10DB、Secret 20個）
- Compute: 14リソース（MIG、Auto Scaling）
- Load Balancer: 19リソース（LB、CDN、WAF、SSL）
- Monitoring: 3リソース（Alert、Log Sink）

---

## 🚀 terraform apply開始

```bash
cd terraform/environments/prod
terraform apply -auto-approve
```

### 初回実行の結果

✅ **順調なスタート**: Network、IAM、Secret Managerが成功

⏳ **長時間待機**: Cloud SQL REGIONAL HAの作成に10分以上

❌ **エラー発生**: その後、連続してエラーが...

---

## ❌ エラー1: Cloud SQL `innodb_buffer_pool_size` が小さすぎる

### エラー内容

```
Error: Value requested is not valid. 
Failed to set innodb_buffer_pool_size: 268435456 is smaller than 20% of the instance memory.
For instances in tier GCE_CUSTOM with 7680MB RAM, 'innodb_buffer_pool_size' cannot be less than 1610612736 (1536MB).
```

### 原因

- 設計時に256MBを指定していた
- Cloud SQLには**最小20%のメモリ制約**がある
- 7680MB RAMの場合、最小1536MB必要

### 解決方法

```hcl
# ❌ 設計時の値
database_flags {
  name  = "innodb_buffer_pool_size"
  value = "268435456" # 256MB
}

# ✅ 修正後
database_flags {
  name  = "innodb_buffer_pool_size"
  value = "5637144576" # 5376MB (70% of 7680MB RAM)
}
```

### 再実行 → また別のエラー

```
Error: 5637144576 is greater than 55% of the instance memory.
Cannot exceed 4429185024 (4224MB).
```

### 追加修正

**Cloud SQLの制約: 20% ≤ innodb_buffer_pool_size ≤ 55%**

```hcl
# ✅ 最終版
database_flags {
  name  = "innodb_buffer_pool_size"
  value = "4429185024" # 4224MB (55% of 7680MB RAM - maximum allowed)
}
```

---

## ❌ エラー2: Cloud SQL `point_in_time_recovery_enabled` for MySQL

### エラー内容

```
Error: point_in_time_recovery_enabled is only available for [POSTGRES SQLSERVER].
You may want to consider using binary_log_enabled instead.
```

### 原因

- PostgreSQL/SQL Server用の設定をMySQLに適用していた

### 解決方法

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

**学び**: バイナリログはHA構成（REGIONAL）時のみ有効化すべき

---

## ❌ エラー3: Filestore IP範囲が衝突

### エラー内容

```
Error: reserved IP range 10.0.3.0/29 overlaps with the existing allocated IP range 10.0.3.0/29 
in network "prod-wordpress-vpc".
```

### 原因

- 前回のterraform applyで既にFilestoreインスタンスが作成されていた
- Terraform stateと実際のリソースの不整合

### 試行1: terraform import

```bash
terraform import 'module.filestore.google_filestore_instance.wordpress' \
  'projects/infra-ai-agent/locations/asia-northeast1-a/instances/prod-wordpress-nfs'

# Error: Resource already managed by Terraform
```

すでにstate管理下にあった。

### 試行2: network指定の修正

```hcl
# ❌ 問題のあったコード（フルリソース名）
networks {
  network = var.network_id  # "projects/.../networks/prod-wordpress-vpc"
}

# ✅ 修正後（名前のみ）
networks {
  network = var.network_name  # "prod-wordpress-vpc"
}
```

**prod/main.tf も修正:**

```hcl
module "filestore" {
  network_id = module.network.vpc_name  # vpc_id → vpc_name
}
```

---

## ❌ エラー4: Terraform変数とシェル変数の衝突

### エラー内容

```
Error: failed to render : <template_file>:64,7-13: Unknown variable; 
There is no variable named "NFS_IP"., and 38 other diagnostic(s)
```

### 原因

起動スクリプト内で、TerraformテンプレートとBashシェルの両方で`${}`記法を使用していたため、Terraformがシェル変数をTerraform変数として解釈しようとした。

### 問題のあったコード

```bash
# 環境変数（Terraform変数）
NFS_IP="${nfs_ip}"         # ✅ Terraform変数
NFS_PATH="${nfs_path}"     # ✅ Terraform変数

# NFSマウント（シェル変数）
mount -t nfs ${NFS_IP}:${NFS_PATH} /var/www/wordpress  # ❌ Terraform変数と解釈される
```

### 解決方法: `$$`エスケープ

```bash
# シェル変数として使用する箇所は$$でエスケープ
mount -t nfs $${NFS_IP}:$${NFS_PATH} /var/www/wordpress  # ✅
```

### 大量の修正が必要

起動スクリプト全体で、**シェル変数を全て`$$`にエスケープ**：

- Nginx設定生成関数内の`${site_num}`、`${domain}` → `$${site_num}`、`$${domain}`
- WordPressセットアップスクリプト内の全シェル変数
- ディレクトリ作成ループの`${i}` → `$${i}`

**修正箇所: 約50箇所**

### HereDocumentの引用符制御

```bash
# ❌ 引用符付き（Terraform変数が展開されない）
cat > /usr/local/bin/setup-wordpress-site.sh << 'SETUP_SCRIPT'
  echo "${ENV}"  # ${ENV}がそのまま出力される
SETUP_SCRIPT

# ✅ 引用符なし（Terraform変数を展開、シェル変数は$$）
cat > /usr/local/bin/setup-wordpress-site.sh << SETUP_SCRIPT
  echo "$${ENV}"  # prod が出力される
SETUP_SCRIPT
```

---

## ❌ エラー5: Service Account Scope 無効

### エラー内容

```
Error: One or more service account scopes are invalid: 
'https://www.googleapis.com/auth/secretmanager'
```

### 原因

`https://www.googleapis.com/auth/secretmanager` という個別scopeは存在しない。

### 解決方法

```hcl
# ❌ 個別scope（無効）
service_account {
  email = var.service_account_email
  scopes = [
    "https://www.googleapis.com/auth/cloud-platform",
    "https://www.googleapis.com/auth/logging.write",
    "https://www.googleapis.com/auth/monitoring.write",
    "https://www.googleapis.com/auth/secretmanager",  # ❌ 存在しない
  ]
}

# ✅ cloud-platform統合scope
service_account {
  email = var.service_account_email
  scopes = [
    "https://www.googleapis.com/auth/cloud-platform",  # すべてのGCP APIにアクセス可能
  ]
}
```

**学び**: `cloud-platform` scopeがあれば、IAM権限に基づいて全サービスにアクセス可能

---

## ❌ エラー6: IAM `secretCreator` ロール不可

### エラー内容

```
Error: Role roles/secretmanager.secretCreator is not supported for this resource.
```

### 原因

`roles/secretmanager.secretCreator` はプロジェクトレベルのIAMバインディングには使用できない。

### 解決方法

```hcl
# ❌ プロジェクトレベル不可
resource "google_project_iam_member" "web_secret_creator" {
  role = "roles/secretmanager.secretCreator"
  member  = "serviceAccount:${google_service_account.web_server.email}"
}

# ✅ 最小権限の2ロールのみ
# 1. 読み取り
resource "google_project_iam_member" "web_secret_accessor" {
  role = "roles/secretmanager.secretAccessor"
}

# 2. バージョン追加
resource "google_project_iam_member" "web_secret_version_adder" {
  role = "roles/secretmanager.secretVersionAdder"
}
```

**結果**: Secretの新規作成は別の方法（事前作成、gcloud CLI等）で対応

---

## ❌ エラー7: Logging Sink GCSバケット不足

### エラー内容

```
Error: GCS bucket must be of form: storage.googleapis.com/[BUCKET_ID]
```

### 原因

```hcl
resource "google_logging_project_sink" "wordpress_logs" {
  destination = "storage.googleapis.com/"  # バケット名が空
}
```

### 解決方法

**Phase 2に延期** - GCS bucketを先に作成する必要がある

```hcl
# Phase 2で実装: ログ用のGCS bucketを作成してから有効化
# resource "google_logging_project_sink" "wordpress_logs" {
#   name        = "${var.env}-wordpress-logs-sink"
#   destination = "storage.googleapis.com/${var.project_id}-wordpress-logs"
#   filter      = "resource.type = \"gce_instance\" AND labels.service = \"wordpress\""
#   unique_writer_identity = true
# }
```

outputsも同様にコメントアウト。

---

## ✅ 最終的なterraform apply成功！

```bash
terraform apply -auto-approve
```

### 結果

```
Apply complete! Resources: 0 added, 1 changed, 1 destroyed.

Outputs:

database_names = [
  "wordpress_site_1",
  "wordpress_site_2",
  "wordpress_site_3",
  "wordpress_site_4",
  "wordpress_site_5",
  "wordpress_site_6",
  "wordpress_site_7",
  "wordpress_site_8",
  "wordpress_site_9",
  "wordpress_site_10",
]
database_private_ip = "192.168.0.2"
database_secret_ids = [
  "prod-wordpress-db-password-1",
  ...
  "prod-wordpress-db-password-10",
]
load_balancer_ip = "34.50.146.93"
nfs_mount_command = "mount -t nfs 10.0.3.2:/wordpress /mnt/wordpress"
ssl_certificate_status = "projects/infra-ai-agent/global/sslCertificates/prod-wordpress-ssl"
vpc_id = "projects/infra-ai-agent/global/networks/prod-wordpress-vpc"
web_server_service_account = "prod-web-server@infra-ai-agent.iam.gserviceaccount.com"
```

🎉 **デプロイ完了！**

---

## 📊 デプロイされたインフラ

| カテゴリ | リソース | 状態 |
|---------|---------|------|
| **Network** | VPC, Subnet×2, NAT, Firewall×4 | ✅ |
| **Compute** | MIG (2-4台), Auto Scaling | ✅ |
| **Database** | Cloud SQL HA, 10DB | ✅ |
| **Storage** | Filestore NFS (1TB) | ✅ |
| **Load Balancer** | Global LB, CDN, WAF | ✅ |
| **Security** | Cloud Armor, Secret Manager | ✅ |
| **SSL** | Google-managed SSL | ⏳ プロビジョニング中 |
| **Monitoring** | HTTP/Health Check Alerts | ✅ |

---

## 🎓 学んだこと

### 1. Cloud SQLの制約を理解する

**innodb_buffer_pool_sizeの制約:**
- **最小**: インスタンスメモリの20%
- **最大**: インスタンスメモリの55%

**設計時の注意点:**
- 小さすぎるとパフォーマンス低下
- 制約を超えるとデプロイエラー
- ドキュメントを確認してから設定

### 2. データベースエンジン固有の設定

| 設定 | PostgreSQL/SQL Server | MySQL |
|------|----------------------|-------|
| PITR | `point_in_time_recovery_enabled` | `binary_log_enabled` |
| 用途 | ポイントインタイムリカバリ | バイナリログレプリケーション |

**教訓**: エンジンごとに設定項目が異なる。コピペ厳禁。

### 3. Terraformテンプレートとシェル変数の共存

**問題:**
```bash
${VARIABLE}  # TerraformもBashも同じ記法
```

**解決:**
```bash
${terraform_var}  # Terraform変数
$${shell_var}     # シェル変数（$$でエスケープ）
```

**HereDocument制御:**
```bash
<< 'MARKER'  # 引用符付き: 変数展開なし
<< MARKER    # 引用符なし: 変数展開あり
```

### 4. GCP Service Account Scopeの理解

**個別scope vs 統合scope:**

| アプローチ | Scope | 利点 | 欠点 |
|-----------|-------|------|------|
| 個別 | `auth/logging.write`等 | 細かい制御 | 一部無効なscopeあり |
| 統合 | `auth/cloud-platform` | シンプル、確実 | IAM権限で制御 |

**推奨**: `cloud-platform` + IAMで最小権限を実現

### 5. IAMロールのリソースレベル制約

**secretCreatorの例:**

| ロール | プロジェクトレベル | リソースレベル |
|--------|------------------|---------------|
| `secretmanager.secretAccessor` | ✅ | ✅ |
| `secretmanager.secretVersionAdder` | ✅ | ✅ |
| `secretmanager.secretCreator` | ❌ | ✅ |

**教訓**: ロールごとに使用可能なレベルが異なる。ドキュメント確認必須。

### 6. terraform applyは段階的に実行

**戦略:**
1. **ネットワーク・IAMから**: 依存関係の少ないものから
2. **エラー → 修正 → 再実行**: 諦めずに繰り返す
3. **リソースは残る**: 再実行時は差分のみ適用

**今回の流れ:**
```
初回 → エラー → 修正 → 2回目 → エラー → 修正 → ...
→ 6回目で成功！
```

**重要**: Terraformは冪等性があるので、何度でも安全に再実行可能

---

## 💡 トラブルシューティングのコツ

### 1. エラーメッセージを正確に読む

```
Error: Value requested is not valid. 
Failed to set innodb_buffer_pool_size: 268435456 is smaller than 20% of the instance memory.
For instances in tier GCE_CUSTOM with 7680MB RAM, 'innodb_buffer_pool_size' cannot be less than 1610612736 (1536MB).
```

**読み取るべき情報:**
- ❌ 現在の値: `268435456` (256MB)
- ✅ 必要な値: 最低`1610612736` (1536MB)
- 📐 計算式: 7680MB × 20% = 1536MB

### 2. ドキュメントを確認

エラーが出たら:
1. Google検索: "gcp cloud sql innodb_buffer_pool_size constraint"
2. 公式ドキュメント: Cloud SQL設定ページ
3. Stack Overflow: 同じエラーの人がいないか

### 3. terraform planで事前検証

```bash
# 修正後は必ずplan
terraform plan

# エラーがなければapply
terraform apply
```

### 4. 段階的なコメントアウト

エラーが多い場合:
```hcl
# 一旦コメントアウトして後回し
# resource "google_logging_project_sink" "wordpress_logs" {
#   ...
# }
```

**メリット**: 他のリソースを先にデプロイできる

### 5. terraform state確認

```bash
# 既存リソースの確認
terraform state list

# 特定リソースの詳細
terraform state show module.filestore.google_filestore_instance.wordpress
```

---

## 🚀 次のステップ

### 1. DNSレコード設定

10ドメイン全てにAレコードを追加：

```
ai-jisso.tech               A    34.50.146.93
infra-career.tech           A    34.50.146.93
cloud-migration.tech        A    34.50.146.93
remote-dev.tech             A    34.50.146.93
monitoring-tools.tech       A    34.50.146.93
learn-code.tech             A    34.50.146.93
tech-books.tech             A    34.50.146.93
engineer-money.tech         A    34.50.146.93
travel-hack.tech            A    34.50.146.93
self-hosting.tech           A    34.50.146.93
```

### 2. SSL証明書プロビジョニング待機

Google-managed SSL証明書は **15-60分** かかる。

**確認コマンド:**
```bash
gcloud compute ssl-certificates describe prod-wordpress-ssl --global

# Status: PROVISIONING → ACTIVE
```

### 3. VM起動確認

```bash
# インスタンス一覧
gcloud compute instances list --filter="labels.service=wordpress"

# 起動スクリプトログ確認
gcloud compute instances get-serial-port-output <INSTANCE_NAME> --zone asia-northeast1-a
```

### 4. Health Check確認

```bash
curl -I http://34.50.146.93/health
# HTTP/1.1 200 OK
# healthy
```

### 5. WordPressセットアップ

各サイトにSSHして：
```bash
sudo /usr/local/bin/setup-wordpress-site.sh 1 ai-jisso.tech "AI実装ブログ"
```

---

## まとめ

今回、**terraform applyで6つのエラーに遭遇**しましたが、一つ一つ丁寧に解決することで、**最終的にデプロイに成功**しました。

**重要な学び:**
1. ✅ **Cloud SQLの制約を理解** - 20%~55%の範囲内
2. ✅ **データベースエンジン固有の設定** - MySQLとPostgreSQLは違う
3. ✅ **Terraform変数とシェル変数の区別** - `$$`エスケープ
4. ✅ **GCP Service Account Scopeの選択** - `cloud-platform`が便利
5. ✅ **IAMロールのレベル制約** - ロールごとに異なる
6. ✅ **段階的なデプロイ戦略** - エラーを恐れず、繰り返す

**Terraformの強み:**
- 冪等性があるので何度でも再実行可能
- エラーが出ても、既存リソースは保持される
- 差分だけを適用するので効率的

**次回予告:**
- SSL証明書プロビジョニング完了後の動作確認
- WordPress初期セットアップ
- 実運用開始とモニタリング

---

## 参考リンク

- [infra-ai-agent リポジトリ](https://github.com/0xchoux1/infra-ai-agent)
  - コミット: 22ca359 (terraform apply成功)
- [前回の記事: Terraform実装編](../2025-11-16-gcp-wordpress-terraform-implementation/)
- [Cloud SQL Configuration](https://cloud.google.com/sql/docs/mysql/flags)
- [Compute Engine Scopes](https://cloud.google.com/compute/docs/access/service-accounts#accesscopesiam)
- [Secret Manager IAM](https://cloud.google.com/secret-manager/docs/access-control)

---

**前回の記事**: [Terraform実装完了！GCP上でWordPressマルチテナント環境を構築](../2025-11-16-gcp-wordpress-terraform-implementation/)

**次回予告**: WordPress初期セットアップとSSL証明書確認編

