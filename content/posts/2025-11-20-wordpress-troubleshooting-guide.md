---
title: "WordPress環境構築で遭遇した5つのハマりポイントと解決策"
date: 2025-11-20T11:00:00+09:00
draft: false
tags: ["WordPress", "GCP", "Ansible", "Terraform", "トラブルシューティング"]
categories: ["技術", "運用"]
series: ["WordPress マルチテナント環境構築"]
---


## はじめに

GCP上でWordPress マルチテナント環境（10サイト）を構築する過程で、予想外のエラーに何度も遭遇しました。

前回の記事「[Cloud SQL SSL設定で2時間ハマった話](04_cloud_sql_ssl_failure_analysis.md)」では判断ミスの分析に焦点を当てましたが、今回は**具体的なトラブル事例と即効性のある解決策**を共有します。

**この記事で扱うハマりポイント**:
1. Ansibleのメタデータ取得で404エラー
2. Terraform applyでインスタンス再作成
3. データベースパスワード不一致
4. WP-CLIの権限エラー
5. カスタムdb.phpの残骸

同じエラーに遭遇した時、この記事がトラブルシューティングの時間短縮につながれば幸いです。

---

## ハマりポイント1: Ansibleのメタデータ取得で404エラー

### 症状

Ansibleでセットアップスクリプトを生成したところ、スクリプト内に **HTML404エラーページ** が埋め込まれていました。

```bash
# /usr/local/bin/setup-wordpress-site.sh の中身
DB_HOST="<!DOCTYPE html>
<html lang=en>
  <meta charset=utf-8>
  <title>Error 404 (Not Found)!!1</title>
  ...
```

スクリプトを実行すると：
```bash
./setup-wordpress-site.sh: line 27: [: too many arguments
```

### 原因

GCPインスタンスのメタデータから`db_host`などの値を取得しようとしましたが、**メタデータが存在しない**ため404エラーが返されました。

Ansibleの該当コード：
```yaml
- name: DB_HOST取得
  uri:
    url: "http://metadata.google.internal/computeMetadata/v1/instance/attributes/db_host"
    headers:
      Metadata-Flavor: "Google"
    return_content: yes
  register: db_host_metadata
  failed_when: false  # ← エラーでも続行

- name: メタデータから変数を設定
  set_fact:
    db_host: "{{ db_host_metadata.content }}"  # ← 404 HTMLがそのまま代入される
```

### 解決策

HTTPステータスコードをチェックして、200の場合のみ値を使用するように修正：

```yaml
- name: メタデータから変数を設定
  set_fact:
    db_host: "{{ db_host_metadata.content if db_host_metadata is defined
                 and db_host_metadata.content is defined
                 and db_host_metadata.status == 200
                 else db_host | default('') }}"
```

### 学んだこと

- `failed_when: false` はエラーを無視するが、エラーレスポンスは残る
- GCPメタデータが存在しない場合、404 HTMLが返される
- **ステータスコードの確認は必須**

---

## ハマりポイント2: Terraform applyでインスタンス再作成

### 症状

Cloud SQLの設定変更（`require_ssl = false`）のために`terraform apply`を実行したところ、**既存のComputeインスタンスが削除・再作成**されました。

```
# module.compute.google_compute_region_instance_template.web must be replaced
-/+ resource "google_compute_region_instance_template" "web" {
      # ...
    }

# module.compute.google_compute_region_instance_group_manager.web will be updated
  ~ resource "google_compute_region_instance_group_manager" "web" {
      ~ version {
          ~ instance_template = "..." -> (known after apply)
        }
    }
```

### 原因

Instance Templateに関連する設定（今回はスタートアップスクリプト内のデータベース設定）が変更されたため、Terraformは：

1. 新しいInstance Templateを作成
2. Managed Instance Groupの設定を更新
3. 古いインスタンスを削除し、新しいインスタンスを起動

これはTerraformの **正常な動作** です。

### 解決策（事前対策）

#### 対策1: `create_before_destroy` を使用

```hcl
resource "google_compute_region_instance_template" "web" {
  lifecycle {
    create_before_destroy = true
  }
}
```

これにより、新しいインスタンスを先に作成してから古いものを削除します。

#### 対策2: ステートフルな構成要素をNFS/Filestoreに配置

今回の構成では、WordPressファイルをCloud Filestoreに配置していたため、インスタンス再作成後もデータは保持されました。

```
/var/www/wordpress  # ← NFSマウント（Cloud Filestore）
├── site1/
├── site2/
└── ...
```

#### 対策3: Ansibleで再デプロイ

新しいインスタンスには環境設定が含まれていないため、Ansibleで再デプロイが必要：

```bash
cd ansible
ansible-playbook -i inventory/gcp.yml playbooks/deploy-wordpress.yml \
  -e "db_host=192.168.0.2" \
  -e "nfs_ip=10.0.3.2" \
  -e "nfs_path=/wordpress"
```

### 学んだこと

- Instance Template変更 = インスタンス再作成
- ステートフルな構成はNFS/データベースに配置
- `create_before_destroy` でダウンタイム最小化

---

## ハマりポイント3: データベースパスワード不一致

### 症状

WordPressから以下のエラー：

```
ERROR 1045 (28000): Access denied for user 'wp_user_1'@'10.0.1.21' (using password: YES)
```

Secret Managerには正しいパスワードが保存されているのに、データベース接続ができません。

### 原因

Terraformで最初にCloud SQLユーザーを作成した後、SSL設定変更のために`terraform apply`を実行しました。

この時、**ユーザーのパスワードはリセットされませんでした**が、Secret Managerの値とCloud SQLのパスワードが不一致になっていました。

原因は、Terraformリソースの`random_password`が再生成されたためです：

```hcl
resource "random_password" "db_passwords" {
  count   = 10
  length  = 20
  special = true
}

resource "google_sql_user" "wordpress_users" {
  count    = 10
  name     = "wp_user_${count.index + 1}"
  password = random_password.db_passwords[count.index].result  # ← 再生成される
}
```

### 解決策

全ユーザーのパスワードをSecret Managerの値に同期：

```bash
for i in {1..10}; do
  PASS=$(gcloud secrets versions access latest \
    --secret=prod-wordpress-db-password-$i \
    --project=infra-ai-agent)

  gcloud sql users set-password wp_user_$i \
    --instance=prod-wordpress-db \
    --password="$PASS" \
    --project=infra-ai-agent

  echo "Updated wp_user_$i"
done
```

### 学んだこと

- `random_password`は状態ファイルに保存されるが、再計画時に変更される場合がある
- Secret Managerとデータベースのパスワード同期を確認
- パスワード変更時は**両方**を更新する必要がある

### 改善案

パスワードを`lifecycle { ignore_changes }`で保護：

```hcl
resource "google_sql_user" "wordpress_users" {
  lifecycle {
    ignore_changes = [password]
  }
}
```

---

## ハマりポイント4: WP-CLIの権限エラー

### 症状

Ansibleで WP-CLI のバージョン確認を実行したところ、以下のエラー：

```
Error: YIKES! It looks like you're running this as root.
You probably meant to run this as the user that your WordPress installation exists under.

If you REALLY mean to run this as root, we won't stop you, but just bear in mind
that any code on this site will then have full control of your server, making it quite DANGEROUS.

If you'd like to continue as root, please run this again, adding this flag:  --allow-root
```

### 原因

AnsibleはデフォルトでSSH経由でrootユーザー（またはsudo権限を持つユーザー）として実行されます。

WP-CLIは**セキュリティ上の理由**でrootユーザーでの実行を推奨していません。

### 解決策

`become_user`でwww-dataユーザーに切り替え：

**修正前**:
```yaml
- name: WP-CLI バージョン確認
  command: "{{ wpcli_bin_path }} --version"
  register: wpcli_version
  changed_when: false
```

**修正後**:
```yaml
- name: WP-CLI バージョン確認
  command: "{{ wpcli_bin_path }} --version"
  register: wpcli_version
  changed_when: false
  become_user: www-data  # ← 追加
```

### WordPress セットアップスクリプトでも同様

```bash
# 修正後
sudo -u www-data /usr/local/bin/wp core download
sudo -u www-data /usr/local/bin/wp config create ...
sudo -u www-data /usr/local/bin/wp core install ...
```

### 学んだこと

- WP-CLIはwww-dataユーザーで実行するのがベストプラクティス
- ファイルパーミッション問題を回避できる
- セキュリティリスクを軽減

---

## ハマりポイント5: カスタムdb.phpの残骸

### 症状

WordPress サイトにアクセスすると HTTP 500 エラー：

```
HTTP/2 500
server: nginx/1.22.1
```

エラーログを確認すると：

```php
Fatal error: Uncaught Error: Call to undefined function wp_kses()
in /var/www/wordpress/site1/wp-includes/functions.php:6105
```

### 原因

前回の記事で作成した **カスタム`db.php`** がNFSマウントされたディレクトリに残っていました。

```
/var/www/wordpress/site1/
├── wp-content/
│   └── db.php  # ← これが原因！
├── wp-admin/
└── wp-includes/
```

WordPressは`wp-content/db.php`が存在すると、それをデータベースドライバとして読み込みます（Drop-in File機能）。

しかし、カスタムdb.phpは不完全な実装だったため、WordPressのコア関数が読み込まれず、`wp_kses()`などが未定義になっていました。

### 解決策

カスタムdb.phpを削除：

```bash
# すべてのサイトから削除
gcloud compute ssh prod-web-z11p --zone=asia-northeast1-a \
  --tunnel-through-iap --project=infra-ai-agent \
  --command="sudo find /var/www/wordpress -name 'db.php' -path '*/wp-content/db.php' -delete"
```

削除後、サイトは即座に正常に戻りました：

```bash
curl -I https://ai-jisso.tech
HTTP/2 200  # ← 成功！
```

### 学んだこと

- WordPressのDrop-in Fileは強力だが危険
- NFSマウントされたファイルは複数サーバーで共有される
- 手動で配置したファイルは**必ず記録・削除**

### 予防策

Ansibleでクリーンアップタスクを追加：

```yaml
- name: カスタムdb.phpの削除（もし存在すれば）
  file:
    path: "/var/www/wordpress/site{{ item }}/wp-content/db.php"
    state: absent
  loop: "{{ range(1, 11) | list }}"
```

---

## トラブルシューティングの心得

### 1. エラーメッセージを正確に読む

```
ERROR 1045 (28000): Access denied
```

- ❌ 「パスワードが間違っている」
- ✅ 「認証に失敗している（パスワード、ユーザー名、ホスト制限の可能性）」

### 2. ログを確認する順序

```
1. アプリケーションログ (/var/log/nginx/error.log)
2. ミドルウェアログ (/var/log/php8.2-fpm.log)
3. システムログ (journalctl -u service-name)
4. GCPコンソール (Cloud Logging)
```

### 3. 問題の切り分け

```
□ ローカル環境で再現するか？
□ 他のサーバーでも発生するか？
□ 最近何を変更したか？
□ ロールバックすると解決するか？
```

### 4. 「動かない」時のチェックリスト

```yaml
network:
  □ ファイアウォールルール
  □ VPC接続
  □ プライベートIP vs パブリックIP

authentication:
  □ パスワード
  □ ユーザー名
  □ ホスト制限

permissions:
  □ ファイルパーミッション (644/755)
  □ オーナー (www-data:www-data)
  □ SELinux/AppArmor

configuration:
  □ 設定ファイルの構文エラー
  □ 環境変数
  □ デフォルト設定の確認
```

---

## まとめ

### 各ハマりポイントの教訓

| ハマりポイント | 根本原因 | 教訓 |
|--------------|---------|------|
| Ansibleメタデータ404 | HTTPステータスコード未確認 | レスポンス検証は必須 |
| インスタンス再作成 | Terraformの正常動作 | ステートレス設計 |
| パスワード不一致 | Secret ManagerとDB不整合 | 同期スクリプト必要 |
| WP-CLI権限 | rootユーザー実行 | www-dataで実行 |
| カスタムdb.php | 手動ファイル配置 | IaC原則の遵守 |

### トラブルシューティングの時短テクニック

1. **エラーログの優先度**: アプリ → ミドルウェア → システム → クラウド
2. **問題の切り分け**: 最小構成で再現させる
3. **変更履歴の確認**: `git log`、Terraformステート、Ansibleログ
4. **公式ドキュメント**: エラーメッセージをそのまま検索
5. **コミュニティ**: Stack Overflow、GitHub Issues

### 次回予告

次回は「Cloud SQL接続の落とし穴 - SSL設定とトラブルシューティング」で、Cloud SQL特有の設定とデバッグ手法を深掘りします。

---

## 参考リンク

- [Ansible - URI Module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html)
- [Terraform - Lifecycle Meta-Argument](https://www.terraform.io/language/meta-arguments/lifecycle)
- [WP-CLI - Best Practices](https://make.wordpress.org/cli/handbook/guides/installing/)
- [WordPress - Drop-in Files](https://developer.wordpress.org/reference/functions/_get_dropins/)

---

## この記事のコード

GitHub: [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)

修正したAnsible Playbook:
- [playbooks/deploy-wordpress.yml](https://github.com/0xchoux1/infra-ai-agent/blob/main/ansible/playbooks/deploy-wordpress.yml)
- [roles/wordpress/tasks/wpcli.yml](https://github.com/0xchoux1/infra-ai-agent/blob/main/ansible/roles/wordpress/tasks/wpcli.yml)

---

**この記事が役に立ったら**: GitHub Starをいただけると嬉しいです！ [infra-ai-agent](https://github.com/0xchoux1/infra-ai-agent)
