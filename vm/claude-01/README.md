# claude-01

Claude Code開発環境（Ubuntu 24.04 LTS）

## 基本情報

| 項目 | 値 |
|------|-----|
| VM ID | 103001 |
| VM名 | claude-01 |
| IPアドレス | 10.2.0.98/27 |
| ゲートウェイ | 10.2.0.97 |
| DNS | 10.0.0.1 |
| VLAN | 103（AI動作用） |
| Proxmoxノード | mandolin1 |

## ハードウェア構成

| リソース | スペック |
|----------|----------|
| CPU | 4 vCPU (host) |
| メモリ | 8 GiB |
| ディスク | 64 GiB (vm-pool, thin provisioning) |
| ネットワーク | VirtIO (vmbr0, VLAN 103) |

## 用途

### 主要機能

1. **Claude Code開発環境**
   - AI支援コーディングツールの実行環境
   - 各種プログラミング言語の開発
   - Git/GitHub連携

2. **開発ツール実行**
   - Node.js、Python、Go等の開発
   - Docker/Podmanコンテナ実行
   - CI/CDパイプラインのテスト

3. **AI/ML実験環境**
   - 軽量なAI/MLモデルの実験
   - データ処理・分析

## セットアップ手順

### 1. 基本パッケージのインストール

```bash
# システム更新
sudo apt update && sudo apt upgrade -y

# 基本ツール
sudo apt install -y \
  vim \
  tmux \
  git \
  curl \
  wget \
  build-essential

# Proxmox連携
sudo apt install -y qemu-guest-agent
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

### 2. 開発ツールのインストール

#### Node.js（推奨: nvm経由）

```bash
# nvmのインストール
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# シェルを再起動
source ~/.bashrc

# Node.js LTSをインストール
nvm install --lts
nvm use --lts

# 確認
node --version
npm --version
```

#### Python

```bash
# Python 3とpip
sudo apt install -y \
  python3 \
  python3-pip \
  python3-venv

# 確認
python3 --version
pip3 --version
```

#### Docker

```bash
# Dockerのインストール
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 現在のユーザーをdockerグループに追加
sudo usermod -aG docker $USER

# 再ログイン後、確認
docker --version
docker run hello-world
```

#### Go（オプション）

```bash
# Go 1.22のインストール
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz

# PATHの設定（~/.bashrcに追加）
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# 確認
go version
```

### 3. Git設定

```bash
# Gitの設定
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# SSHキーの生成（GitHub用）
ssh-keygen -t ed25519 -C "your.email@example.com"

# 公開鍵を表示（GitHubに登録）
cat ~/.ssh/id_ed25519.pub
```

### 4. Claude Code（CLI版）のセットアップ

```bash
# Claude CLIのインストール（npmの場合）
npm install -g @anthropic-ai/claude-cli

# または、バイナリ版をダウンロード
# https://claude.ai/code

# 認証設定
claude auth login

# 動作確認
claude --version
```

## SSH接続方法

### 踏み台経由での接続

```bash
# bastion-01経由でアクセス
ssh -J admin@10.2.0.2 admin@10.2.0.98

# ~/.ssh/config に設定
Host bastion
    HostName 10.2.0.2
    User admin

Host claude-01
    HostName 10.2.0.98
    User admin
    ProxyJump bastion
```

### 直接接続（AI VLAN内から）

```bash
ssh admin@10.2.0.98
```

## 開発ワークフロー例

### 1. プロジェクトのクローン

```bash
# GitHubからクローン
git clone git@github.com:username/project.git
cd project
```

### 2. 依存関係のインストール

```bash
# Node.jsプロジェクトの場合
npm install

# Pythonプロジェクトの場合
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 3. Claude Codeで開発

```bash
# Claude Codeを起動
claude code .

# または、IDEと連携して使用
```

### 4. Dockerコンテナでテスト

```bash
# Dockerfileからビルド
docker build -t myapp:latest .

# コンテナ実行
docker run -p 8080:8080 myapp:latest
```

## リソース監視

### CPU/メモリ使用状況

```bash
# topコマンド
top

# htop（より見やすい）
sudo apt install -y htop
htop

# リソース使用状況のサマリ
free -h
df -h
```

### Proxmox側での監視

Web UIで確認:
- claude-01 → Summary
- CPU、メモリ、ディスクI/Oのグラフ

## ディスク容量管理

### 容量確認

```bash
# 使用状況確認
df -h

# ディレクトリ別使用量
du -sh ~/*
```

### クリーンアップ

```bash
# Dockerイメージ・コンテナのクリーンアップ
docker system prune -a

# Aptキャッシュのクリーンアップ
sudo apt autoremove -y
sudo apt clean

# npm/node_modulesのクリーンアップ
find . -name "node_modules" -type d -prune -exec rm -rf '{}' +
```

### ディスク拡張

容量不足の場合、Proxmox Web UIで拡張:
1. claude-01 → Hardware → Hard Disk
2. Resize → 追加容量を入力
3. VM内で確認: `df -h`

## トラブルシューティング

### SSH接続できない

```bash
# Proxmoxノードから確認
ssh root@10.1.1.11
qm status 103001  # VM状態確認
qm start 103001   # 停止していたら起動

# ネットワーク確認
ping 10.2.0.98

# VMコンソールで直接ログイン
# Web UI: claude-01 → Console
```

### Dockerが動作しない

```bash
# Dockerサービスの確認
sudo systemctl status docker

# 再起動
sudo systemctl restart docker

# ログ確認
sudo journalctl -u docker -n 50
```

### メモリ不足

```bash
# メモリ使用状況確認
free -h

# 大きなプロセスを特定
ps aux --sort=-%mem | head -10

# 必要に応じて、Proxmox Web UIでメモリ増設
# claude-01 → Hardware → Memory → Edit
```

## バックアップ

Proxmox Web UIから定期バックアップを設定:
- Datacenter → Backup
- Schedule: 毎日 3:00 AM
- Storage: vm-pool
- Mode: Snapshot
- Retention: 7日分

## セキュリティ設定

### ファイアウォール（ufw）

```bash
# UFWを有効化
sudo ufw enable

# SSH許可
sudo ufw allow 22/tcp

# アプリケーションポート（必要に応じて）
sudo ufw allow 3000/tcp  # Node.js開発サーバー
sudo ufw allow 8080/tcp  # その他のアプリ

# 状態確認
sudo ufw status
```

### SSH鍵認証

```bash
# パスワード認証を無効化（推奨）
sudo vim /etc/ssh/sshd_config

# 以下を設定:
# PasswordAuthentication no
# PubkeyAuthentication yes

# SSHサービスを再起動
sudo systemctl restart sshd
```

## 関連ドキュメント

- [VM管理ガイド](../README.md)
- [ネットワーク設計](../../docs/design.md)
- [Proxmoxクラスタ構成](../../pm/mandolin/CLUSTER.md)
- [Claude Code公式ドキュメント](https://claude.ai/code)
