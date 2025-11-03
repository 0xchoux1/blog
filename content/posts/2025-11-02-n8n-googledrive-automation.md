---
title: "n8nでGoogle Driveへのファイル自動アップロードを実現する完全ガイド"
date: 2025-11-02T20:30:00+09:00
draft: false
tags: ["n8n", "Google Drive", "ワークフロー自動化", "Docker", "OAuth", "API", "バックアップ"]
categories: ["技術"]
series: ["AI開発環境"]
---

## はじめに

本記事では、ワークフロー自動化ツール「n8n」を使用して、Google Driveにフォルダを自動作成し、ローカルファイルを自動でアップロードする方法を実践的に解説します。

### 対象読者

- ワークフロー自動化に興味のあるエンジニア
- Google Driveを業務で活用している企業・チーム
- n8nを使ってファイル管理を効率化したい方

### 記事の目的

1. n8nとGoogle Drive APIの連携方法を理解する
2. フォルダ作成とファイルアップロードのワークフローを構築する
3. 実際に動作検証を行い、実用レベルで使えるようにする

---

## なぜn8nとGoogle Driveなのか

### 課題：手動でのファイルアップロードの煩わしさ

日々の業務で、こんな経験はありませんか？

- 📁 毎日決まったフォルダに、決まった形式のファイルをアップロード
- 📸 定期的にスクリーンショットや画像をバックアップ
- 📊 レポートやログファイルをGoogle Driveで共有

これらの作業は単純ですが、**繰り返すたびに時間を取られ、ミスも発生しがち**です。

### 解決策：n8nによる自動化

**n8n**は、オープンソースのワークフロー自動化ツールで、Zapierのようにさまざまなサービスを連携できます。

**n8nを使うメリット:**
- ✅ **無料でセルフホスト可能** - 実行回数無制限
- ✅ **ビジュアルエディタ** - プログラミング不要
- ✅ **豊富な連携** - 1000以上のサービスと統合
- ✅ **柔軟なカスタマイズ** - JavaScriptで拡張可能

### 本記事で実現すること

今回構築するワークフローで実現できること：

```
1. n8nがトリガー（手動 or スケジュール実行）を受け取る
2. Google Drive上に指定した名前のフォルダを自動作成
3. ローカルに保存されているファイルを読み込む
4. 作成したフォルダにファイルをアップロード
5. 完了通知（オプション）
```

---

## Google Drive API の設定

n8nからGoogle Driveにアクセスするには、Google Cloud Platform (GCP) での設定が必要です。

### Google Cloud プロジェクトの作成

#### ステップ 1: GCPコンソールへアクセス

1. [Google Cloud Console](https://console.cloud.google.com/) にアクセス
2. Googleアカウントでログイン

#### ステップ 2: 新規プロジェクトの作成

1. 画面上部の「プロジェクトを選択」をクリック
2. 「新しいプロジェクト」をクリック
3. プロジェクト名を入力（例: `n8n-googledrive-automation`）
4. 「作成」をクリック

### Google Drive API の有効化

#### ステップ 1: APIライブラリへ移動

1. 左側のナビゲーションメニューから「APIとサービス」→「ライブラリ」を選択

#### ステップ 2: Google Drive APIを検索

1. 検索バーに「Google Drive API」と入力
2. 検索結果から「Google Drive API」をクリック
3. 「有効にする」をクリック

### OAuth 2.0 認証情報の作成

n8nからGoogle Driveにアクセスするには、OAuth 2.0の認証情報が必要です。

#### ステップ 1: 認証情報画面へ移動

1. 「APIとサービス」→「認証情報」を選択
2. 「認証情報を作成」→「OAuth クライアント ID」を選択

#### ステップ 2: 同意画面の設定

初めて作成する場合、OAuth同意画面の設定が必要です：

1. **ユーザータイプ**: 「外部」を選択（テスト用）
2. **アプリ情報**:
   - アプリ名: `n8n Google Drive Integration`
   - ユーザーサポートメール: 自分のメールアドレス
   - デベロッパーの連絡先: 自分のメールアドレス
3. 「保存して次へ」をクリック
4. **スコープ**: 一旦スキップ（後で設定可能）
5. **テストユーザー**: 自分のメールアドレスを追加
6. 「保存して次へ」をクリック

#### ステップ 3: OAuth クライアント ID の作成

1. 「アプリケーションの種類」: **ウェブアプリケーション**を選択
2. 名前: `n8n OAuth Client`
3. **承認済みのリダイレクト URI**: 
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
   ※ n8nがリモートサーバーにある場合は、`localhost`を該当のドメインに変更

4. 「作成」をクリック

#### ステップ 4: 認証情報の保存

作成されたクライアントIDとクライアントシークレットが表示されます。

**⚠️ 重要:** この情報は後でn8nの設定で使用するので、安全な場所にメモしてください。

---

## n8nの準備

### n8nのインストール

Docker環境を使用する場合：

```bash
# docker-compose.ymlを使用してn8nを起動
docker-compose up -d n8n

# 起動確認
docker-compose ps
```

### n8nの起動とアクセス

1. ブラウザで `http://localhost:5678` にアクセス
2. 初回の場合、管理者アカウントを作成

### Google Drive認証情報の設定

#### ステップ 1: Credentialsページへ移動

1. n8nのメニューから「Credentials」をクリック
2. 「New」→「Google Drive OAuth2 API」を選択

#### ステップ 2: 認証情報の入力

以下の情報を入力します：

| 項目 | 値 |
|------|-----|
| **Credential Name** | `Google Drive - My Account` |
| **Client ID** | GCPで取得したクライアントID |
| **Client Secret** | GCPで取得したクライアントシークレット |

#### ステップ 3: 認証の実行

1. 「Connect my account」ボタンをクリック
2. Googleのログイン画面が開く
3. Google Driveへのアクセスを許可
4. 認証が成功すると、n8nに戻る
5. 「Save」をクリックして保存

---

## ワークフローの作成

実際にワークフローを作成していきます。

### ワークフローの全体像

今回作成するワークフローは以下の4つのノードで構成されます：

```
[1. Manual Trigger]
     ↓
[2. Google Drive - Create Folder]
     ↓
[3. Read Binary File]
     ↓
[4. Google Drive - Upload File]
```

### ノード1: Manual Trigger（手動トリガー）

#### 作成手順

1. 新しいワークフローを作成（「Add workflow」をクリック）
2. ワークフロー名を `Google Drive Auto Upload` に変更
3. 最初のノードとして「Manual Trigger」が配置されている（そのまま使用）

**説明:**  
手動でワークフローを実行するためのトリガーです。後で「Schedule Trigger」に変更すれば、定期実行も可能です。

### ノード2: Google Drive - Create Folder（フォルダ作成）

#### 作成手順

1. 「+」ボタンをクリックして新しいノードを追加
2. 「Google Drive」を検索して選択
3. 以下の設定を行う：

| 項目 | 値 |
|------|-----|
| **Credential** | 先ほど作成した `Google Drive - My Account` を選択 |
| **Operation** | `Create a Folder` を選択 |
| **Folder Name** | `={{ $now.format('yyyy-MM-dd') }}_upload` （日付付きフォルダ名） |
| **Parent Folder** | （空欄 = ルートディレクトリ） |

**ポイント:**
- `={{ $now.format('yyyy-MM-dd') }}_upload` は、実行日の日付を含むフォルダ名を自動生成します
  - 例: `2025-11-02_upload`
- 特定のフォルダ配下に作成したい場合は、「Parent Folder」にフォルダIDを指定

### ノード3: Read Binary File（ファイル読み込み）

#### 作成手順

1. 「+」ボタンをクリックして新しいノードを追加
2. 「Read Binary File」を検索して選択
3. 以下の設定を行う：

| 項目 | 値 |
|------|-----|
| **File Path** | `/tmp/upload/sample.txt` （アップロードしたいファイルのパス） |
| **Property Name** | `data` （デフォルト） |

**注意:**
- n8nがDockerコンテナで動作している場合、ホストのファイルシステムをマウントする必要があります
- docker-compose.ymlに以下のようなボリュームマウントを追加：
  ```yaml
  volumes:
    - ./upload:/tmp/upload
  ```

### ノード4: Google Drive - Upload File（ファイルアップロード）

#### 作成手順

1. 「+」ボタンをクリックして新しいノードを追加
2. 「Google Drive」を検索して選択
3. 以下の設定を行う：

| 項目 | 値 |
|------|-----|
| **Credential** | `Google Drive - My Account` を選択 |
| **Operation** | `Upload` を選択 |
| **Binary Data** | `true` （チェックを入れる） |
| **File Name** | `=uploaded_{{ $now.format('yyyyMMdd_HHmmss') }}.txt` |

**Parent Folder（親フォルダ）の設定:**

| 項目 | 値 |
|------|-----|
| **Parent Drive** | `From list` → `My Drive` |
| **Parent Folder** | **重要：ドロップダウンを「By ID」または「Expression」に変更** |
| **Folder ID** | `{{ $node["Google Drive - Create Folder"].json.id }}` |

**⚠️ 重要なポイント:**

1. **File Name の先頭に `=` が必須**
   - ❌ 間違い: `uploaded_{{ $now.format(...) }}.txt`
   - ✅ 正しい: `=uploaded_{{ $now.format(...) }}.txt`
   - `=` がないと変数が展開されず、そのまま表示されます

2. **Parent Folder は「By ID」モードに変更**
   - デフォルトの「From list」モードでは前のノードの結果を参照できません
   - 必ず左側のドロップダウンを **「By ID」** または **「Expression」** に変更してください
   - これを忘れると、ファイルがフォルダ内ではなくルートにアップロードされます

3. **ノード名の完全一致**
   - `$node["Google Drive - Create Folder"]` のノード名は完全に一致している必要があります
   - 大文字小文字、スペース、ハイフンなど、すべて正確に入力してください

**設定後の確認:**
- Folder IDに長い文字列（例: `1oXaNfVRGfkRg678QmRkRgotq0lB-l4cC`）が表示されていれば成功です

### ワークフローの保存と実行

#### 保存

1. 右上の「Save」ボタンをクリック
2. ワークフロー名を確認して保存

#### テスト実行

1. 左上の「Execute Workflow」ボタンをクリック
2. 各ノードが順番に実行される
3. 緑のチェックマークが表示されれば成功

#### 実行結果の確認

1. Google Driveにアクセス
2. 日付入りフォルダが作成されているか確認
3. フォルダ内にファイルがアップロードされているか確認

---

## 動作検証

### 検証環境の準備

#### テスト用ファイルの作成

```bash
# アップロード用ディレクトリの作成
mkdir -p upload

# テスト用ファイルの作成
echo "This is a test file for n8n Google Drive upload" > upload/sample.txt
```

#### docker-compose.ymlの更新

n8nコンテナがローカルファイルにアクセスできるように、ボリュームマウントを追加します：

```yaml
services:
  n8n:
    volumes:
      - n8n_data:/home/node/.n8n
      - ./upload:/tmp/upload  # この行を追加
```

変更後、n8nを再起動：

```bash
docker-compose restart n8n
```

### 実際の動作検証手順

#### ステップ1: ワークフローの実行

1. n8nのワークフローエディタを開く
2. 「Execute Workflow」をクリック
3. 実行が完了するまで待機（通常5〜10秒）

#### ステップ2: 各ノードの結果確認

各ノードをクリックして、出力を確認します：

**ノード2（Create Folder）の出力例:**
```json
{
  "kind": "drive#file",
  "id": "1a2b3c4d5e6f7g8h9i0j",
  "name": "2025-11-02_upload",
  "mimeType": "application/vnd.google-apps.folder"
}
```

**ノード4（Upload File）の出力例:**
```json
{
  "kind": "drive#file",
  "id": "9j8i7h6g5f4e3d2c1b0a",
  "name": "uploaded_20251102_143025.txt",
  "mimeType": "text/plain"
}
```

#### ステップ3: Google Driveでの確認

1. [Google Drive](https://drive.google.com) にアクセス
2. マイドライブに `2025-11-02_upload` フォルダがあることを確認
3. フォルダ内にアップロードされたファイルがあることを確認

### 検証チェックリスト

- [ ] フォルダが正しい名前（日付入り）で作成されている
- [ ] フォルダがGoogle Driveの正しい場所に作成されている
- [ ] ファイルが作成されたフォルダ内にアップロードされている
- [ ] アップロードされたファイルの内容が元のファイルと一致している
- [ ] ファイル名がタイムスタンプ付きで正しく命名されている

### 実際の成功例

実際にこのワークフローを動作検証した結果を紹介します。

#### 実行環境
- n8n: 最新版（Dockerコンテナ）
- Google Drive API: v3
- 実行日時: 2025年11月2日 20:01

#### 実行結果

**1. フォルダの作成**
```
名前: 2025-11-02_upload
場所: マイドライブ（ルート）
作成時刻: 19:50
フォルダID: 1oXaNfVRGfkRg678QmRkRgotq0lB-l4cC
```

**2. ファイルのアップロード**
```
ファイル名: uploaded_20251102_200116.txt
場所: 2025-11-02_upload フォルダ内
ファイルサイズ: 584 バイト
アップロード時刻: 20:01
```

#### 成功のポイント

✅ **File Nameに `=` を付けた**
- 変数 `{{ $now.format('yyyyMMdd_HHmmss') }}` が正しく展開
- タイムスタンプ `200116` が自動生成された

✅ **Parent Folderを「By ID」モードに設定**
- フォルダIDが正しく参照された
- ファイルがフォルダ内に配置された

✅ **ボリュームマウントが正しく設定されている**
- `./upload:/tmp/upload` の設定により、ローカルファイルにアクセス可能

#### つまづきやすいポイント

この検証中に遭遇した問題：

1. **最初の試行**: 変数が展開されず、ファイル名がそのまま表示
   → **解決**: File Nameの先頭に `=` を追加

2. **2回目の試行**: ファイルがマイドライブのルートに配置
   → **解決**: Parent Folderを「From list」から「By ID」に変更

3. **3回目の試行**: ✅ **成功！**

---

## 実用的な応用例

基本的なワークフローができたので、実用的な応用例を紹介します。

### 定期的なバックアップ自動化

**ユースケース:** 毎日深夜に、特定のディレクトリ内のファイルを自動的にGoogle Driveにバックアップ

#### 変更点

1. **Triggerをスケジュール実行に変更**
   - Manual Triggerを削除
   - 「Schedule Trigger」ノードを追加
   - 実行時刻を設定（例: 毎日午前2時）

2. **複数ファイルの処理**
   - 「Read Binary Files」ノードで、ディレクトリ内の全ファイルを読み込む
   - ファイルパスに `*` ワイルドカードを使用: `/tmp/upload/*`

3. **ファイルごとにアップロード**
   - 「Split In Batches」ノードを追加して、ファイルを1つずつ処理

### Slackへの通知

**ユースケース:** アップロード完了後、Slackに通知を送信

#### 追加ノード

1. **Slackノード**
   - Operation: `Post Message`
   - Channel: `#general`（または指定のチャンネル）
   - Message: 
     ```
     ✅ ファイルアップロード完了
     📁 フォルダ: {{ $node["Google Drive - Create Folder"].json["name"] }}
     📄 ファイル: {{ $node["Google Drive - Upload File"].json["name"] }}
     ```

---

## トラブルシューティング

### よくあるエラーと解決方法

#### エラー1: 変数が展開されない

**症状:**
```
ファイル名が「uploaded_{{ $now.format('yyyyMMdd_HHmmss') }}.txt」のまま
フォルダ名が「{{ $now.format('yyyy-MM-dd') }}_upload」のまま
```

**原因:**
- n8nの式には先頭に `=` が必要なのに、付けていない

**解決方法:**

1. 該当するノードをクリック
2. 「Name」または「File Name」フィールドを確認
3. 先頭に `=` を追加

**間違い:**
```
uploaded_{{ $now.format('yyyyMMdd_HHmmss') }}.txt
```

**正しい:**
```
=uploaded_{{ $now.format('yyyyMMdd_HHmmss') }}.txt
```

4. 「Save」をクリックして保存
5. ワークフローを再実行

**成功時の結果:**
```
ファイル名: uploaded_20251102_200116.txt （タイムスタンプが正しく展開される）
```

#### エラー2: ファイルがフォルダ内ではなくルートにアップロードされる

**症状:**
```
- フォルダは正しく作成される
- ファイルもアップロードされる
- しかし、ファイルがマイドライブのルートに配置される
- フォルダ内が空のまま
```

**原因:**
- Parent Folderの設定が「From list」モードのまま
- または、Parent Folderの設定が正しくない

**解決方法:**

1. **「Google Drive - Upload File」ノードをクリック**
2. **Parent Folderセクションを確認**
3. 左側のドロップダウンが **「From list」** になっている場合：
   - **「By ID」** または **「Expression」** に変更
4. 入力欄に以下を入力：
   ```
   {{ $node["Google Drive - Create Folder"].json.id }}
   ```
5. フォルダIDが表示されることを確認（例: `1oXaNfVRGfkRg678QmRkRgotq0lB-l4cC`）
6. 「Save」→ ワークフローを再実行

**成功時の結果:**
```
📁 2025-11-02_upload/
   └── 📄 uploaded_20251102_200116.txt  ← フォルダ内に配置される
```

**重要なポイント:**
- 「From list」モードでは、ドロップダウンからフォルダを選択する必要がありますが、動的に作成されたフォルダは選択できません
- 「By ID」または「Expression」モードを使用して、前のノードから取得したIDを指定する必要があります

#### エラー3: 認証エラー（401 Unauthorized）

**症状:**
```
ERROR: Unauthorized - Please check your OAuth credentials
```

**原因:**
- OAuth認証が正しく設定されていない
- アクセストークンが期限切れ

**解決方法:**
1. n8nのCredentialsページへ移動
2. Google Drive認証情報を開く
3. 「Reconnect」をクリックして再認証

#### エラー4: ファイルが見つからない（File Not Found）

**症状:**
```
ERROR: ENOENT: no such file or directory, open '/tmp/upload/sample.txt'
```

**原因:**
- 指定したファイルパスが存在しない
- Dockerコンテナ内からファイルにアクセスできない

**解決方法:**
1. docker-compose.ymlでボリュームマウントを確認
2. マウントパスが正しいか確認
3. ファイルのパーミッションを確認

```bash
# コンテナ内でファイルが見えるか確認
docker-compose exec n8n ls -la /tmp/upload
```

---

## まとめ

### 達成したこと

本記事では、以下を実現しました：

✅ **Google Drive APIの設定** - OAuth 2.0認証の構成  
✅ **n8nとの連携** - 認証情報の登録  
✅ **基本ワークフローの作成** - フォルダ作成とファイルアップロード  
✅ **動作検証** - 実際にファイルがアップロードされることを確認  
✅ **応用例の紹介** - 定期実行、Webhook、条件分岐、通知

### このワークフローの利点

- 🚀 **作業時間の大幅削減** - 手動アップロードから解放
- 🔄 **自動化による安定性** - ヒューマンエラーを削減
- 📅 **スケジュール実行** - 決まった時間に自動実行
- 🔧 **柔軟なカスタマイズ** - 業務に合わせて拡張可能

### 次のステップ

さらに発展させるためのアイデア：

1. **他サービスとの連携**
   - Dropbox、OneDrive、AWS S3などへの同時アップロード
   - メール添付ファイルの自動保存

2. **画像処理の追加**
   - アップロード前に画像のリサイズ
   - PDFへの変換

3. **通知の強化**
   - Slack、Microsoft Teams、Discordへの通知
   - エラー時のアラート

4. **監視とログ**
   - アップロード履歴をデータベースに記録
   - ダッシュボードでの可視化

### 参考リンク

- [n8n公式ドキュメント](https://docs.n8n.io/)
- [Google Drive API リファレンス](https://developers.google.com/drive/api/v3/reference)
- [n8n Community Forum](https://community.n8n.io/)

---

🎉 これでn8nを使ったGoogle Driveへの自動ファイルアップロードが実現できました！

ぜひ、あなたの業務に合わせてカスタマイズして活用してください。

関連記事:
- [n8nとDifyで構築する最強のAIワークフロー自動化環境](/posts/2025-11-02-n8n-dify-workflow-automation/)
- [Ubuntu 22.04でのDocker環境構築](/posts/2025-11-02-ubuntu-docker-setup/)

