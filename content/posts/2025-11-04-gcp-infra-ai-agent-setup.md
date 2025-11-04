---
title: "Google Cloud Platform上でAIエージェントによるインフラ自動運用環境を構築する【完全ガイド】"
date: 2025-11-04T21:00:00+09:00
draft: false
tags: ["GCP", "AI", "インフラ自動化", "Python", "Terraform", "Ansible", "uv", "Claude"]
categories: ["技術"]
series: ["AIエージェント開発"]
---

## はじめに

クラウドインフラの運用は複雑化の一途を辿っています。VMインスタンスの起動・停止、リソースの監視、コスト最適化、トラブルシューティング…これらの日々の運用タスクに追われていませんか？

本記事では、**Google Cloud Platform（GCP）上でAIエージェントがインフラを自律的に運用する環境**を、ゼロから構築する方法を解説します。人間が指示を出すだけで、AIがGCPのリソースを操作し、監視し、最適化する—そんな未来志向のインフラ運用を実現しましょう。

## なぜインフラAIエージェントなのか

### 従来のインフラ運用の課題

- **手作業の多さ**: CLIコマンドを手動で実行、GUIを何度もクリック
- **知識の属人化**: 複雑な手順書、ベテランエンジニアへの依存
- **対応の遅さ**: アラート発生から対応まで時間がかかる
- **単調な繰り返し**: 同じ操作を何度も実行

### AIエージェントが実現する世界

```
[エンジニア] 「asia-northeast1-aのインスタンス状態を確認して」
    ↓
[AIエージェント] GCP APIを呼び出し → 情報を取得 → 整形して報告
    ↓
[エンジニア] 「CPU使用率が高いインスタンスがあれば教えて」
    ↓
[AIエージェント] メトリクスを分析 → 異常検知 → アラート
```

**AIエージェントの利点：**
- ✅ **自然言語で操作**: コマンドを覚える必要なし
- ✅ **自律的な判断**: 状況に応じた最適なアクションを選択
- ✅ **24時間監視**: 人間が寝ている間も動作
- ✅ **学習と改善**: 過去のオペレーションから学習

## プロジェクト概要

### 構築する環境

```
┌─────────────────────────────────────────────────┐
│         AI Agent (Python CLI)                   │
│  - 自然言語での指示受付                          │
│  - GCP操作ツール（起動/停止/削除）               │
│  - 監視ツール（メトリクス収集・異常検知）        │
└────────────┬────────────────────────────────────┘
             │ Google Cloud SDK
┌────────────▼────────────────────────────────────┐
│         Google Cloud Platform                   │
│  - Compute Engine (VM)                          │
│  - Cloud Storage (バケット)                     │
│  - Cloud Monitoring (メトリクス)                │
└─────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────┐
│    Infrastructure as Code                       │
│  Terraform: インフラプロビジョニング             │
│  Ansible: サーバー構成管理                       │
└─────────────────────────────────────────────────┘
```

### 技術スタック

- **Python 3.12**: メイン言語
- **uv**: 高速Pythonパッケージマネージャー
- **gcloud CLI**: GCP認証とプロジェクト管理
- **Terraform**: GCPリソース作成・管理
- **Ansible**: サーバー内構成管理
- **structlog**: 構造化ロギング
- **click**: CLIフレームワーク

## アーキテクチャ設計の重要な選択

### ツールの使い分け：Terraform vs Ansible

この2つのツールは**役割が異なる**ため、併用することで最大の効果を発揮します。

#### Terraform: インフラプロビジョニングに特化

**役割:**
- GCPリソース（VM、VPC、ストレージ）の作成・削除
- ネットワーク構成の定義
- IAMロールとサービスアカウントの管理

**特徴:**
- 宣言的な記述（「あるべき姿」を定義）
- 状態管理（terraform.state）
- `terraform plan`で変更内容を事前確認

**使用例:**
```hcl
resource "google_compute_instance" "vm" {
  name         = "web-server"
  machine_type = "e2-medium"
  zone         = "asia-northeast1-a"
}
```

#### Ansible: 構成管理とデプロイに特化

**役割:**
- サーバー内のパッケージインストール
- 設定ファイルの配置
- アプリケーションのデプロイ
- サービスの起動・再起動

**特徴:**
- 手続き型の記述（「どうやって実現するか」を定義）
- エージェントレス（SSH経由で実行）
- 冪等性の保証

**使用例:**
```yaml
- name: Webサーバーのセットアップ
  hosts: all
  tasks:
    - name: Nginxをインストール
      apt:
        name: nginx
        state: present
```

#### 使い分けの原則

| 作業内容 | ツール | 理由 |
|---------|--------|------|
| VMインスタンス作成 | Terraform | インフラリソース |
| Nginxインストール | Ansible | サーバー構成 |
| VPCネットワーク構築 | Terraform | インフラリソース |
| アプリデプロイ | Ansible | サーバー構成 |
| IAMロール設定 | Terraform | インフラリソース |
| 設定ファイル配置 | Ansible | サーバー構成 |

### Python環境管理：なぜuvなのか

#### uvとは

**uv**は、Astral社（Ruffの開発元）が開発したRust製の超高速Pythonパッケージマネージャーです。

**従来のpip:**
```bash
pip install -r requirements.txt
# → 30秒〜数分
```

**uvを使用:**
```bash
uv pip install -r requirements.txt
# → 2〜3秒（10-100倍高速！）
```

#### uvを選んだ理由

1. **圧倒的な速度**: pipの10〜100倍高速
2. **pipとの互換性**: requirements.txtをそのまま使える
3. **仮想環境管理**: `uv venv`で即座に仮想環境作成
4. **信頼性**: Rust製で安全性が高い
5. **モダンな設計**: 最新のPythonエコシステムに対応

#### インストール方法（推奨）

```bash
# pipxでuvをインストール
sudo apt install pipx
pipx install uv

# これで完了！
uv --version
```

この方法なら：
- ✅ システムのPython環境を汚染しない
- ✅ aptでpipxをインストール（システムパッケージ管理）
- ✅ pipxでuvを分離インストール（Pythonツール管理）

## セットアップ手順

### Phase 0: 前提条件のインストール

#### 必要なツール

```bash
# 前提条件チェックスクリプトを実行
bash scripts/check_prerequisites.sh
```

**インストールが必要なもの:**
- Python 3.10以上
- pipx + uv
- gcloud CLI
- Terraform（オプション）

#### gcloud CLIのインストール

```bash
# gcloud CLIをダウンロード&インストール
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xzf google-cloud-cli-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh

# PATHを更新
source ~/.bashrc

# バージョン確認
gcloud version
```

#### uvのインストール

```bash
# pipxをインストール
sudo apt install -y pipx

# uvをインストール
pipx install uv

# 確認
uv --version  # → uv 0.9.7
```

### Phase 1: GCPプロジェクトとリポジトリ

#### 1. GCPプロジェクト作成

Google Cloud Consoleで `infra-ai-agent` プロジェクトを作成します。

#### 2. リポジトリ構造

```
infra-ai-agent/
├── README.md                 # プロジェクト概要
├── CLAUDE.md                 # AIエージェント向けガイド
├── .gitignore                # 機密情報の除外
├── env.example               # 環境変数テンプレート
├── requirements.txt          # Python依存関係
│
├── scripts/                  # セットアップ・テストスクリプト
│   ├── check_prerequisites.sh
│   ├── setup.sh
│   └── test_connection.py
│
├── terraform/                # インフラコード（Terraform）
│   ├── provider.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars.example
│
├── ansible/                  # 構成管理（Ansible）
│   ├── ansible.cfg
│   ├── inventory/
│   │   └── gcp.yml          # GCP動的インベントリ
│   ├── playbooks/
│   │   └── setup.yml
│   └── requirements.yml
│
└── agent/                    # AIエージェントコア
    ├── __init__.py
    ├── main.py               # CLIエントリーポイント
    └── tools/
        ├── gcp_tools.py      # GCP操作ツール
        └── monitoring.py     # 監視ツール
```

### Phase 2: 環境変数の設定

```bash
# テンプレートをコピー
cp env.example .env

# 編集
vim .env
```

**`.env`の内容:**
```bash
# GCPプロジェクトID
GCP_PROJECT_ID=infra-ai-agent

# GCPリージョンとゾーン
GCP_REGION=asia-northeast1
GCP_ZONE=asia-northeast1-a

# Terraform設定
TF_VAR_project_id=infra-ai-agent
TF_VAR_region=asia-northeast1

# ログレベル
LOG_LEVEL=INFO

# 環境識別子
ENVIRONMENT=development
```

### Phase 3: GCP認証

```bash
# gcloud認証（ブラウザが開く）
gcloud auth login

# プロジェクト設定
gcloud config set project infra-ai-agent

# Application Default Credentials設定
gcloud auth application-default login
```

**Application Default Credentials（ADC）とは？**
- PythonスクリプトがGCP APIを使うための認証方式
- `~/.config/gcloud/application_default_credentials.json`に保存
- Google Cloud SDKが自動的に読み込む

### Phase 4: セットアップスクリプト実行

```bash
# 自動セットアップ
bash scripts/setup.sh
```

**このスクリプトが実行すること:**
1. ✅ 環境変数の確認
2. ✅ Pythonバージョンチェック
3. ✅ uvを使って仮想環境作成（`.venv`）
4. ✅ 61個の依存パッケージをインストール
5. ✅ gcloud CLI認証確認
6. ✅ GCP APIを有効化（Compute Engine、Cloud Monitoring等）

### Phase 5: 接続テスト

```bash
# GCP接続テストを実行
source .venv/bin/activate
python scripts/test_connection.py
```

**テスト内容:**
- 認証テスト
- プロジェクトアクセステスト
- Compute Engine API呼び出し
- ゾーン一覧取得

**成功例:**
```
==================================================
🚀 Infra AI Agent - GCP接続テスト
==================================================

✓ 認証成功
✓ プロジェクトアクセス成功
✓ Compute Engine API 呼び出し成功
✓ すべてのテストが成功しました (4/4)

✨ GCP接続が正常に確認できました！
```

## 実装したAIエージェントツール

### CLIインターフェース

```bash
# AIエージェントのヘルプ
python -m agent.main --help

# 使用可能なコマンド:
#   status   - インフラの現在の状態を確認
#   zones    - 利用可能なゾーン一覧
#   start    - VMインスタンスを起動
#   stop     - VMインスタンスを停止
#   monitor  - インスタンスのメトリクスを監視
```

### 実行例1: インフラ状態確認

```bash
python -m agent.main status
```

**出力:**
```
📊 インフラステータスチェック

💻 VMインスタンス:
  インスタンスが見つかりません

🪣 Cloud Storage バケット:
  バケットが見つかりません
```

### 実行例2: ゾーン一覧表示

```bash
python -m agent.main zones
```

**出力:**
```
🌏 利用可能なゾーン

📍 asia-northeast1
  🟢 asia-northeast1-a
  🟢 asia-northeast1-b
  🟢 asia-northeast1-c

📍 asia-northeast2
  🟢 asia-northeast2-a
  🟢 asia-northeast2-b
  🟢 asia-northeast2-c

... （127ゾーン取得）
```

### 実行例3: VMインスタンス操作

```bash
# インスタンス起動
python -m agent.main start my-instance --zone asia-northeast1-a

# インスタンス停止
python -m agent.main stop my-instance --zone asia-northeast1-a
```

### 実行例4: 監視とメトリクス

```bash
# 過去1時間のメトリクスを取得
python -m agent.main monitor my-instance --hours 1
```

**出力:**
```
📈 my-instance のメトリクス監視

インスタンス: my-instance
ゾーン: asia-northeast1-a
期間: 過去1時間

💻 CPU:
  平均: 45.23%
  最大: 78.50%
  最小: 12.30%
  データポイント: 60

⚠️  CPU使用率が高くなっています
```

## 実装の詳細

### GCP操作ツール

```python
# agent/tools/gcp_tools.py
from google.cloud import compute_v1

class GCPTools:
    def __init__(self, project_id):
        self.project_id = project_id
        self.zone = "asia-northeast1-a"
    
    def list_instances(self):
        """VMインスタンス一覧を取得"""
        client = compute_v1.InstancesClient()
        instances = client.list(
            project=self.project_id,
            zone=self.zone
        )
        return [
            {
                'name': inst.name,
                'status': inst.status,
                'machine_type': inst.machine_type,
                'internal_ip': inst.network_interfaces[0].network_i_p
            }
            for inst in instances
        ]
    
    def start_instance(self, instance_name):
        """VMインスタンスを起動"""
        client = compute_v1.InstancesClient()
        operation = client.start(
            project=self.project_id,
            zone=self.zone,
            instance=instance_name
        )
        logger.info("Started instance", name=instance_name)
        return True
```

### 監視ツール

```python
# agent/tools/monitoring.py
from google.cloud import monitoring_v3
from datetime import datetime, timedelta

class MonitoringTools:
    def get_cpu_utilization(self, instance_name, hours=1):
        """CPU使用率を取得"""
        client = monitoring_v3.MetricServiceClient()
        
        # 時間範囲
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)
        
        # メトリクスフィルタ
        filter_str = (
            f'resource.type = "gce_instance" '
            f'AND resource.labels.instance_id = "{instance_name}" '
            f'AND metric.type = "compute.googleapis.com/instance/cpu/utilization"'
        )
        
        # データ取得
        results = client.list_time_series(
            name=f"projects/{self.project_id}",
            filter=filter_str,
            interval=interval
        )
        
        return [
            {
                'timestamp': point.interval.end_time,
                'value': point.value.double_value * 100,
                'unit': '%'
            }
            for point in results
        ]
```

## Terraform設定

### プロバイダー設定

```hcl
# terraform/provider.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}
```

### 変数定義

```hcl
# terraform/variables.tf
variable "project_id" {
  description = "GCPプロジェクトID"
  type        = string
  default     = "infra-ai-agent"
}

variable "region" {
  description = "デフォルトのGCPリージョン"
  type        = string
  default     = "asia-northeast1"
}

variable "environment" {
  description = "環境識別子"
  type        = string
  default     = "development"
}
```

## Ansible設定

### GCP動的インベントリ

```yaml
# ansible/inventory/gcp.yml
---
plugin: gcp_compute

projects:
  - infra-ai-agent

regions:
  - asia-northeast1

# グループ化設定
keyed_groups:
  - key: zone
    prefix: zone
  
  - key: labels
    prefix: label
  
  - key: status
    prefix: status

# ホスト変数設定
compose:
  ansible_host: networkInterfaces[0].accessConfigs[0].natIP
```

### セットアップPlaybook

```yaml
# ansible/playbooks/setup.yml
---
- name: GCPインスタンス基本セットアップ
  hosts: all
  become: yes
  
  vars:
    basic_packages:
      - vim
      - git
      - curl
      - htop
      - python3
    
    timezone: Asia/Tokyo
  
  tasks:
    - name: タイムゾーンの設定
      timezone:
        name: "{{ timezone }}"
    
    - name: パッケージリストの更新
      apt:
        update_cache: yes
        cache_valid_time: 3600
    
    - name: 基本パッケージのインストール
      apt:
        name: "{{ basic_packages }}"
        state: present
```

## トラブルシューティング

### よくある問題と解決方法

#### 1. gcloud CLIが見つからない

```bash
# PATHが通っていない場合
export PATH="/path/to/google-cloud-sdk/bin:$PATH"

# bashrcに追加
echo 'export PATH="/path/to/google-cloud-sdk/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

#### 2. 認証エラー

```bash
# Application Default Credentialsを再設定
gcloud auth application-default login

# 認証情報の確認
gcloud auth list
```

#### 3. API有効化エラー

```bash
# 必要なAPIを手動で有効化
gcloud services enable compute.googleapis.com
gcloud services enable monitoring.googleapis.com
gcloud services enable logging.googleapis.com
```

#### 4. Python依存関係のエラー

```bash
# 仮想環境を再作成
rm -rf .venv
uv venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

## ベストプラクティス

### セキュリティ

- [ ] `.env`ファイルは**絶対に**Gitにコミットしない
- [ ] サービスアカウントには最小権限の原則を適用
- [ ] APIキーは定期的にローテーション
- [ ] すべてのオペレーションをCloud Loggingに記録
- [ ] 本番環境では別のGCPプロジェクトを使用

### コスト最適化

```python
# 使用していないインスタンスの自動停止
def auto_stop_idle_instances():
    """CPU使用率が低いインスタンスを停止"""
    monitoring = MonitoringTools()
    gcp_tools = GCPTools()
    
    instances = gcp_tools.list_instances()
    for instance in instances:
        metrics = monitoring.get_cpu_utilization(
            instance['name'], 
            hours=24
        )
        
        avg_cpu = sum(m['value'] for m in metrics) / len(metrics)
        
        if avg_cpu < 5.0:  # 5%未満
            logger.warning(
                "Idle instance detected",
                instance=instance['name'],
                avg_cpu=avg_cpu
            )
            # 停止するかどうかは人間が判断
            # gcp_tools.stop_instance(instance['name'])
```

### 運用Tips

**ログの構造化:**
```python
import structlog

logger = structlog.get_logger()
logger.info(
    "Instance started",
    instance_name="web-server-01",
    zone="asia-northeast1-a",
    machine_type="e2-medium"
)
```

**エラーハンドリング:**
```python
try:
    gcp_tools.start_instance("my-instance")
except Exception as e:
    logger.error(
        "Failed to start instance",
        instance="my-instance",
        error=str(e)
    )
    # アラート通知
    send_alert_to_slack(f"エラー: {e}")
```

## まとめ

### 構築した環境

- ✅ GCP接続が確認済みのPython環境
- ✅ VMインスタンス操作ツール（起動/停止/削除）
- ✅ 監視ツール（CPU/メモリ/ディスクI/O）
- ✅ Terraformによるインフラコード管理
- ✅ Ansibleによる構成管理
- ✅ uvによる高速パッケージ管理

### 達成したこと

1. **環境構築の自動化**: セットアップスクリプト一発で完了
2. **ツールの適材適所**: Terraform（インフラ）+ Ansible（構成管理）
3. **モダンなPython環境**: uvで高速インストール
4. **実用的なCLI**: 自然言語に近いコマンド体系
5. **拡張可能な設計**: 新しいツールを簡単に追加可能

### 次のステップ

#### Phase 1: 基本機能の充実
- [ ] より詳細なメトリクス監視
- [ ] 異常検知アルゴリズムの実装
- [ ] 自動スケーリングロジック

#### Phase 2: AI統合
- [ ] Claude APIとの統合
- [ ] 自然言語での指示受付
- [ ] コンテキストを理解した自律的判断

#### Phase 3: 高度な自動化
- [ ] マルチエージェント協調
- [ ] 予測的メンテナンス
- [ ] コスト最適化の自動実行

### 今後の可能性

**AIエージェントの進化:**
```
現在: 「VMを起動して」 → 起動
  ↓
未来: 「顧客が増えそう」 → リソース分析 → 自動スケールアップ → コスト試算 → 承認待ち
```

**学習と改善:**
- 過去のオペレーションログから学習
- より効率的な手順の自動発見
- エラーパターンの事前検知

## 参考リソース

- [プロジェクトリポジトリ](https://github.com/0xchoux1/infra-ai-agent)
- [Google Cloud SDK](https://cloud.google.com/sdk/docs)
- [Terraform Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Ansible Google Cloud Collection](https://docs.ansible.com/ansible/latest/collections/google/cloud/index.html)
- [uv公式ドキュメント](https://docs.astral.sh/uv/)

---

インフラ運用の未来は、AIとの協働にあります。この記事で構築した環境をベースに、あなただけのインフラAIエージェントを育ててみませんか？

