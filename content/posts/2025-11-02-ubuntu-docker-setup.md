---
title: "Ubuntu 24.04にDocker環境を構築する"
date: 2025-11-02T07:00:00+09:00
draft: false
tags: ["Docker", "Ubuntu", "環境構築"]
categories: ["技術"]
series: ["環境構築"]
---

## はじめに

このブログでは、Ubuntu 24.04にDocker環境を構築する手順を、実際の構築経験に基づいて説明します。

## 前提条件

- Ubuntu 24.04がインストール済み
- sudo権限を持つユーザーアカウント
- インターネット接続

## インストール手順

### 1. システムパッケージの更新

まず、パッケージリストを最新の状態に更新します。

```bash
sudo apt-get update -y
```

このコマンドで、APTパッケージマネージャーがすべての利用可能なパッケージリストを更新します。

### 2. Dockerのインストール

Docker.ioとDocker Composeをインストールします。

```bash
sudo apt-get install -y docker.io docker-compose
```

**インストール内容：**
- **docker.io** (バージョン 27.5.1) - Dockerコンテナランタイム
- **docker-compose** - Docker Composeツール（複数コンテナの管理用）
- **containerd** (バージョン 1.7.28) - コンテナランタイムの依存関係
- **runc** (バージョン 1.3.0) - コンテナ実行環境

### 3. インストール確認

Dockerが正しくインストールされたか確認します。

```bash
docker --version
```

出力例：
```
Docker version 27.5.1, build 27.5.1-0ubuntu3~24.04.2
```

### 4. Docker Compose v2のインストール（重要）

APTリポジトリからインストールされるdocker-composeは古いバージョン（1.29.2）であり、Python 3.12との互換性の問題があります。公式のDocker Compose v2パッケージをインストールすることをお勧めします。

```bash
sudo apt install docker-compose-v2 -y
```

バージョン確認：
```bash
docker compose version
```

出力例：
```
Docker Compose version 2.37.1+ds1-0ubuntu2~24.04.1
```

**重要：** Docker Compose v2では、コマンドが`docker-compose`（ハイフン）から`docker compose`（スペース）に変更されています。

### 5. ユーザーをdockerグループに追加

Dockerコマンドを実行する際に、毎回sudoを入力する必要を避けるため、現在のユーザーをdockerグループに追加します。

```bash
sudo usermod -aG docker $USER
```

その後、ターミナルを再起動するか、以下のコマンドで権限を反映させます。

```bash
newgrp docker
```

## 動作確認

以下のコマンドでDockerが正常に動作しているか確認します。

```bash
docker ps
```

このコマンドで現在実行中のコンテナ一覧が表示されます。初回実行時は何も表示されない場合があります。

Helloコンテナを実行して確認することもできます。

```bash
docker run hello-world
```

成功すると以下のようなメッセージが表示されます。

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

## トラブルシューティング

### docker-composeが起動しない場合

**症状：**
```
ModuleNotFoundError: No module named 'distutils'
```

**原因：** Python 3.12ではdistutilsモジュールが削除されました。Ubuntu 24.04のデフォルトのdocker-compose（v1.29.2）はこれに依存しているため、互換性の問題が発生します。

**解決方法：** Docker Compose v2をインストールしてください。

```bash
sudo apt install docker-compose-v2 -y
```

インストール後は、`docker compose`（スペース付き）コマンドを使用します。古い`docker-compose`（ハイフン付き）コマンドは削除することをお勧めします。

```bash
sudo apt remove docker-compose
```

### permissionエラーが出る場合

```
permission denied while trying to connect to the Docker daemon socket
```

**解決方法：** 以下の手順を実行してください。

1. `sudo usermod -aG docker $USER` でユーザーをグループに追加
2. ターミナルをログアウト/ログイン、または `newgrp docker` を実行
3. 改めて `docker ps` で確認

## インストール後の推奨設定

### 1. Dockerデーモンの有効化

システム起動時にDockerが自動起動するように設定します。

```bash
sudo systemctl enable docker
```

### 2. ユーザーのログイン再実行

dockerグループへの追加を完全に反映させるため、ログアウト/ログインしてください。

## まとめ

このガイドに従うことで、Ubuntu 24.04に最新のDocker環境を構築できます。主なポイントは：

- 古いdocker-composeの互換性問題に対処する
- ユーザーをdockerグループに追加してsudo不要にする
- インストール後の動作確認を忘れずに

これでコンテナベースの開発環境が整いました。次のステップとしては、実際にDockerfileを書いてカスタムイメージを作成したり、docker-composeで複数コンテナを管理したりすることができます。

## 参考リンク

- [Docker公式ドキュメント](https://docs.docker.com/)
- [Docker Composeリリースページ](https://github.com/docker/compose/releases)
- [Ubuntuパッケージ管理](https://help.ubuntu.com/community/AptGetHowto)
