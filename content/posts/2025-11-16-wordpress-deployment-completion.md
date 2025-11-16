---
title: "WordPress環境デプロイ完了！DNS設定からSSL証明書プロビジョニングまで【トラブル解決編】"
date: 2025-11-16T22:00:00+09:00
draft: false
tags: ["GCP", "WordPress", "SSL", "DNS", "トラブルシューティング", "デプロイ"]
categories: ["技術", "運用"]
series: ["AIエージェント開発"]
---

## はじめに

[前回の記事](../2025-11-16-terraform-apply-実践編/)で、Terraform applyを実行してGCPインフラをデプロイしました。

今回は、**DNS設定後に発生した問題を解決し、SSL証明書のプロビジョニングを完了させ、WordPress環境を本番稼働可能な状態にする**までの全工程を記録します。

---

## 📋 今回の作業フロー

```
1. DNS設定と確認
   ↓
2. 起動スクリプトエラー発見（mysql-client不在）
   ↓
3. 修正 & 再デプロイ（Autoscaler問題）
   ↓
4. 起動スクリプトエラー再発（Logging Agent問題）
   ↓
5. 修正 & 再デプロイ
   ↓
6. SSL証明書プロビジョニング完了
   ↓
7. HTTPS動作確認 ✅
```

---

## 1. DNS設定と確認

### 設定内容

10ドメイン全てにAレコードを追加：

| ドメイン | タイプ | 値 |
|---------|--------|-----|
| ai-jisso.tech | A | 34.50.146.93 |
| infra-career.tech | A | 34.50.146.93 |
| cloud-migration.tech | A | 34.50.146.93 |
| remote-dev.tech | A | 34.50.146.93 |
| monitoring-tools.tech | A | 34.50.146.93 |
| learn-code.tech | A | 34.50.146.93 |
| tech-books.tech | A | 34.50.146.93 |
| engineer-money.tech | A | 34.50.146.93 |
| travel-hack.tech | A | 34.50.146.93 |
| self-hosting.tech | A | 34.50.146.93 |

### DNS確認コマンド

```bash
for domain in ai-jisso.tech infra-career.tech cloud-migration.tech; do
  dig +short $domain A
done
```

### 結果

```
34.50.146.93  ✅
34.50.146.93  ✅
34.50.146.93  ✅
... (全10ドメイン正常)
```

🎉 **DNS設定完璧！**

---

## ❌ 問題1: 起動スクリプトがmysql-clientで失敗

### 問題発見

VMインスタンスのシリアルポート出力を確認：

```bash
gcloud compute instances get-serial-port-output prod-web-7511 \
  --zone asia-northeast1-a | tail -50
```

### エラー内容

```
E: Package 'mysql-client' has no installation candidate
Script "startup-script" failed with error: exit status 100
```

### 原因

**Debian 12では`mysql-client`パッケージが存在しない**

Debian 12から、MySQLクライアントパッケージ名が変更されました：

| Debian 11以前 | Debian 12 |
|--------------|-----------|
| `mysql-client` | `default-mysql-client` |

### 解決方法

**terraform/scripts/startup_script.sh を修正:**

```bash
# ❌ 修正前
apt-get install -y \
  ...
  mysql-client \
  nfs-common \

# ✅ 修正後
apt-get install -y \
  ...
  default-mysql-client \
  nfs-common \
```

### terraform apply実行

```bash
cd terraform/environments/prod
terraform apply -auto-approve
```

### 新たな問題: Autoscalerエラー

```
Error: Error resizing RegionInstanceGroupManager: googleapi: Error 412: 
Resizing of autoscaled regional managed instance groups is not allowed.
```

**原因**: Autoscalerが有効な状態でMIGのサイズ変更不可

**解決**: Autoscalerを一時停止

```bash
gcloud compute instance-groups managed stop-autoscaling prod-web-mig \
  --region=asia-northeast1
```

### 再度terraform apply

```bash
terraform apply -auto-approve

# 成功！
Apply complete! Resources: 1 added, 2 changed, 1 destroyed.
```

✅ **新しいインスタンステンプレート作成完了**

---

## ❌ 問題2: 起動スクリプトがLogging Agentで失敗

### ローリングアップデート確認

新しいVMインスタンスが起動：

```bash
gcloud compute instances list --filter="labels.service=wordpress"

# 結果
prod-web-65vr  asia-northeast1-a  RUNNING  (NEW)
prod-web-jfxp  asia-northeast1-b  RUNNING  (NEW)
prod-web-7511  asia-northeast1-a  RUNNING  (OLD)
prod-web-3t70  asia-northeast1-b  RUNNING  (OLD)
```

### 新VMのログ確認

```bash
gcloud compute instances get-serial-port-output prod-web-65vr \
  --zone asia-northeast1-a | tail -100
```

### エラー内容

```
Err:2 https://packages.cloud.google.com/apt google-cloud-logging-bookworm-all Release
  404  Not Found
E: The repository does not have a Release file.
Script "startup-script" failed with error: exit status 1
```

### 原因

**旧Cloud Logging Agentのリポジトリが削除されている**

Googleは旧Logging Agentを非推奨とし、**Ops Agent**（Logging + Monitoring統合）への移行を推奨しています。

### 解決方法

**Ops Agentに置き換え:**

```bash
# ❌ 旧Cloud Logging Agent
curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
bash add-logging-agent-repo.sh --also-install

# ✅ Ops Agent（Logging + Monitoring統合）
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
bash add-google-cloud-ops-agent-repo.sh --also-install
```

### Ops Agentの利点

| 項目 | 旧Agent | Ops Agent |
|------|---------|-----------|
| **Logging** | Cloud Logging Agent | ✅ 統合 |
| **Monitoring** | Cloud Monitoring Agent | ✅ 統合 |
| **設定** | 2つの設定ファイル | 1つの設定ファイル |
| **パフォーマンス** | 個別プロセス | 統合プロセス |
| **サポート** | 非推奨 | 推奨 |

### terraform apply再実行

```bash
terraform apply -auto-approve
```

### 再びAutoscalerエラー

同じエラーが発生。Autoscalerを停止して再実行：

```bash
gcloud compute instance-groups managed stop-autoscaling prod-web-mig \
  --region=asia-northeast1

terraform apply -auto-approve

# 成功！
Apply complete! Resources: 1 added, 2 changed, 1 destroyed.
```

---

## ✅ 起動スクリプト完了確認

### 新VMインスタンス確認

```bash
gcloud compute instances list --filter="labels.service=wordpress"

NAME           ZONE               STATUS   CREATION_TIMESTAMP
prod-web-tng4  asia-northeast1-c  RUNNING  2025-11-16T05:34:24
prod-web-pd9s  asia-northeast1-a  RUNNING  2025-11-16T05:36:45
```

### 起動スクリプトログ確認

```bash
gcloud compute instances get-serial-port-output prod-web-pd9s \
  --zone asia-northeast1-a | tail -50
```

### 結果

```
google-cloud-ops-agent installation succeeded. ✅
Finished google-startup-scripts.service ✅
```

🎉 **両方のVM起動スクリプト完了！**

---

## 🔧 Autoscalerの再有効化

停止していたAutoscalerを再度有効化：

```bash
gcloud compute instance-groups managed set-autoscaling prod-web-mig \
  --region=asia-northeast1 \
  --mode=on \
  --min-num-replicas=2 \
  --max-num-replicas=4 \
  --target-cpu-utilization=0.7 \
  --cool-down-period=60
```

### 結果

```yaml
status: ACTIVE
mode: ON
recommendedSize: 2
minNumReplicas: 2
maxNumReplicas: 4
cpuUtilization:
  utilizationTarget: 0.7
```

✅ **Autoscaler正常稼働**

---

## 🔐 SSL証明書プロビジョニング完了！

### 初回確認（DNS設定直後）

```bash
gcloud compute ssl-certificates describe prod-wordpress-ssl --global
```

### 結果（初回）

| 状態 | ドメイン数 |
|------|----------|
| ✅ ACTIVE | 3 |
| ⚠️ FAILED_NOT_VISIBLE | 7 |

**FAILED_NOT_VISIBLE**: GoogleがDNS経由でLoad Balancerを確認中

### 最終確認（約30分後）

```bash
gcloud compute ssl-certificates describe prod-wordpress-ssl --global \
  --format="yaml" | grep -A 15 "domainStatus:"
```

### 結果（最終）

```yaml
domainStatus:
  ai-jisso.tech: ACTIVE                ✅
  cloud-migration.tech: ACTIVE         ✅
  engineer-money.tech: ACTIVE          ✅
  infra-career.tech: ACTIVE            ✅
  learn-code.tech: ACTIVE              ✅
  monitoring-tools.tech: ACTIVE        ✅
  remote-dev.tech: ACTIVE              ✅
  self-hosting.tech: ACTIVE            ✅
  tech-books.tech: ACTIVE              ✅
  travel-hack.tech: ACTIVE             ✅
```

🎉 **全10ドメインのSSL証明書がACTIVE！**

### プロビジョニング時間

| フェーズ | 時間 |
|---------|------|
| DNS設定 | 即時 |
| DNS伝播 | 5-10分 |
| SSL検証 | 15-30分 |
| **合計** | **約30-40分** |

---

## 🚀 HTTPS動作確認

### テスト1: HTTPSアクセス

```bash
for domain in ai-jisso.tech infra-career.tech cloud-migration.tech; do
  echo "=== Testing HTTPS: $domain ==="
  curl -I https://$domain/health
done
```

### 結果

```
=== Testing HTTPS: ai-jisso.tech ===
HTTP/2 404                              ✅ HTTP/2で応答！

=== Testing HTTPS: infra-career.tech ===
HTTP/2 404                              ✅

=== Testing HTTPS: cloud-migration.tech ===
HTTP/2 404                              ✅
```

**404は正常**: WordPressがまだインストールされていないため

### テスト2: Health Check

```bash
curl -I -k https://34.50.146.93/health
```

### 結果

```
HTTP/2 200                              ✅
Content-Type: text/plain
Content-Length: 8
```

🎉 **HTTPS完全動作！**

---

## 📊 最終構成

### インフラサマリー

| カテゴリ | リソース | 状態 |
|---------|---------|------|
| **Network** | VPC, Subnet×2, NAT, Firewall | ✅ 稼働中 |
| **Compute** | MIG (2-4台), Autoscaler | ✅ 稼働中 |
| **Database** | Cloud SQL HA, 10DB | ✅ 稼働中 |
| **Storage** | Filestore NFS (1TB) | ✅ 稼働中 |
| **Load Balancer** | Global HTTP(S) LB, CDN, WAF | ✅ 稼働中 |
| **SSL** | Google-managed SSL (10ドメイン) | ✅ ACTIVE |
| **DNS** | 10ドメイン → 34.50.146.93 | ✅ 設定済み |
| **Monitoring** | Ops Agent, Alert Policies | ✅ 稼働中 |

### VMインスタンス詳細

| インスタンス | ゾーン | ステータス |
|-------------|--------|-----------|
| prod-web-tng4 | asia-northeast1-c | RUNNING |
| prod-web-pd9s | asia-northeast1-a | RUNNING |

### 起動スクリプト実行内容

✅ システム更新（Debian 12）
✅ Nginx + PHP 8.2-FPM インストール
✅ WP-CLI インストール
✅ NFS マウント（Filestore）
✅ Nginx設定生成（10サイト分）
✅ PHP OPcache 最適化
✅ WordPress セットアップスクリプト配置
✅ **Ops Agent インストール**（Logging + Monitoring）

---

## 🎓 学んだこと

### 1. Debian 12のパッケージ名変更に注意

**Debian 12での変更点:**
- `mysql-client` → `default-mysql-client`
- `python` → `python3`
- その他多数のパッケージ名変更

**教訓**: OSアップグレード時は公式ドキュメントを必ず確認

### 2. GCP Ops Agentへの移行

**旧Agent vs Ops Agent:**

| 項目 | 旧構成 | 新構成（Ops Agent） |
|------|--------|-------------------|
| プロセス数 | 2個 | 1個 |
| 設定ファイル | 2個 | 1個 |
| メモリ使用量 | 高め | 最適化 |
| サポート状況 | 非推奨 | 推奨 |

**インストールコマンド:**
```bash
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
bash add-google-cloud-ops-agent-repo.sh --also-install
```

### 3. Autoscaler有効時のMIG更新

**問題**: Autoscaler ONの状態でMIG更新不可

**解決手順:**
1. Autoscalerを停止
2. Terraform apply実行
3. Autoscalerを再有効化

**自動化の提案:**
```hcl
# Terraformで自動管理
lifecycle {
  create_before_destroy = true
}
```

### 4. SSL証明書プロビジョニングの待機時間

**フェーズ別所要時間:**

| フェーズ | 時間 | 確認方法 |
|---------|------|---------|
| DNS設定 | 即時 | `dig +short` |
| DNS伝播 | 5-10分 | 同上 |
| SSL検証 | 15-30分 | `gcloud compute ssl-certificates describe` |
| SSL ACTIVE | 30-60分 | 同上 |

**Tips**: `FAILED_NOT_VISIBLE`は一時的な状態。DNS伝播待ち。

### 5. HTTP/2対応の確認

**確認コマンド:**
```bash
curl -I https://example.com
# HTTP/2 200 ← HTTP/2で応答していることを確認
```

**HTTP/2の利点:**
- 多重化（Multiplexing）
- ヘッダー圧縮
- サーバープッシュ
- パフォーマンス向上

---

## 🔍 トラブルシューティングのベストプラクティス

### 1. シリアルポート出力の活用

VMの起動スクリプトログを確認：

```bash
gcloud compute instances get-serial-port-output <INSTANCE_NAME> \
  --zone <ZONE> | tail -100
```

**検索テクニック:**
```bash
# エラー検索
| grep -i error

# 特定のキーワード前後
| grep -A 5 -B 5 "startup-script"

# 最新ログのみ
| tail -100
```

### 2. Terraform state管理

```bash
# リソース一覧
terraform state list

# 特定リソース詳細
terraform state show <RESOURCE>

# State更新
terraform refresh
```

### 3. GCP CLIでの直接確認

```bash
# SSL証明書
gcloud compute ssl-certificates list

# MIG状態
gcloud compute instance-groups managed describe <MIG_NAME> --region <REGION>

# Autoscaler
gcloud compute instance-groups managed describe <MIG_NAME> \
  --region <REGION> --format="yaml" | grep -A 10 autoscaler
```

### 4. 段階的なロールバック

問題発生時:
1. Autoscaler停止
2. 問題のあるVMを手動削除
3. 修正後、terraform apply
4. 動作確認
5. Autoscaler再有効化

---

## 📝 コミット履歴

### コミット1: mysql-client修正

```bash
git commit -m "fix: terraform apply成功のための修正

- Cloud SQL innodb_buffer_pool_sizeを4224MBに調整
- Filestoreネットワーク設定修正
- 起動スクリプトTerraform変数エスケープ
- IAM Secret Manager権限調整
- Instance Template Service Account Scope修正
- Monitoring Log Sink Phase 2延期"
```

### コミット2: 起動スクリプト修正

```bash
git commit -m "fix: 起動スクリプトのパッケージエラーを修正

## 修正内容

1. MySQL Clientパッケージ名修正
   - mysql-client → default-mysql-client
   
2. Logging Agent を Ops Agent に更新
   - 旧Cloud Logging Agent → Google Cloud Ops Agent
   - Ops AgentはLogging + Monitoring統合

## 検証結果

✅ 新VM 2台で起動スクリプト正常完了
✅ Health Check正常応答

## デプロイ状況

- VM: 2台稼働中
- Load Balancer IP: 34.50.146.93
- SSL証明書: 全10ドメインACTIVE
- DNS: 全10ドメイン設定完了"
```

---

## 🚀 次のステップ

### 1. WordPressインストール

各サイトにWordPressをセットアップ：

```bash
# VMにSSH接続
gcloud compute ssh prod-web-pd9s --zone=asia-northeast1-a

# WordPressセットアップ（サイト1）
sudo /usr/local/bin/setup-wordpress-site.sh 1 ai-jisso.tech "AI実装ブログ"

# 管理者パスワード取得
gcloud secrets versions access latest \
  --secret=prod-wordpress-admin-password-1
```

### 2. Ansibleプレイブック作成

自動化のため、Ansible Playbookを作成：

- WordPress一括インストール
- プラグイン導入（Cache、SEO等）
- テーマ設定
- 初期設定自動化

### 3. CI/CDパイプライン構築

- GitHub Actions
- Cloud Build
- 自動デプロイ

### 4. AIエージェント統合

- 運用監視
- 異常検知
- 自動復旧

---

## まとめ

今回、**DNS設定からSSL証明書プロビジョニング完了まで**を実施し、いくつかの問題に遭遇しながらも解決しました。

**解決した問題:**
1. ✅ `mysql-client`パッケージ不在 → `default-mysql-client`に変更
2. ✅ 旧Logging Agent非推奨 → Ops Agentに移行
3. ✅ Autoscaler競合 → 一時停止して更新
4. ✅ SSL証明書プロビジョニング → 全10ドメインACTIVE

**最終構成:**
- ✅ VM 2台稼働（Autoscaler管理）
- ✅ HTTPS完全動作（HTTP/2）
- ✅ 全10ドメインSSL対応
- ✅ Health Check正常
- ✅ Ops Agent導入

**重要な学び:**
- Debianバージョンごとのパッケージ名変更に注意
- GCP推奨のOps Agentへの移行が必須
- Autoscaler有効時のMIG更新手順
- SSL証明書プロビジョニングには30-60分必要
- シリアルポート出力でのデバッグが有効

次回は、**WordPress初期セットアップとAnsibleによる自動化**を実施します！

---

## 参考リンク

- [infra-ai-agent リポジトリ](https://github.com/0xchoux1/infra-ai-agent)
  - コミット: e061cac (起動スクリプト修正完了)
- [前回の記事: Terraform Apply実践編](../2025-11-16-terraform-apply-実践編/)
- [Google Cloud Ops Agent](https://cloud.google.com/stackdriver/docs/solutions/agents/ops-agent)
- [Debian 12 Release Notes](https://www.debian.org/releases/bookworm/)
- [Google-managed SSL certificates](https://cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs)

---

**前回の記事**: [Terraform Apply実践編！GCPにWordPress環境をデプロイ【エラー解決の全記録】](../2025-11-16-terraform-apply-実践編/)

**次回予告**: WordPress初期セットアップとAnsible自動化編

