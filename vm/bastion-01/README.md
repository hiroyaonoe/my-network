# bastion-01

SSH踏み台サーバー（Ubuntu 24.04 LTS）

## 基本情報

| 項目 | 値 |
|------|-----|
| VM ID | 100001 |
| VM名 | bastion-01 |
| IPアドレス | 10.2.0.2/27 |
| ゲートウェイ | 10.2.0.1 |
| DNS | 10.0.0.1 |
| VLAN | 100（踏み台） |
| Proxmoxノード | mandolin1 |

## ハードウェア構成

| リソース | スペック |
|----------|----------|
| CPU | 4 vCPU (host) |
| メモリ | 8 GiB |
| ディスク | 64 GiB (vm-pool, thin provisioning) |
| ネットワーク | VirtIO (vmbr0, VLAN 100) |

## 用途

### 主要機能

1. **SSH踏み台（ジャンプホスト）**
   - 外部からの初回アクセスポイント
   - SSHポートフォワーディング
   - 多段SSH接続の起点

2. **管理ツール実行環境**
   - Ansible、Terraform等の実行
   - kubectl、helm等のk8s管理ツール
   - 各種CLIツールのインストール

3. **セキュリティゲートウェイ**
   - 踏み台VLANから他VLANへのアクセス制御
   - ログ記録・監査

## インストール済みソフトウェア

```bash
# 基本ツール
sudo apt install -y vim tmux git curl wget

# SSH設定
sudo apt install -y openssh-server

# Proxmox連携
sudo apt install -y qemu-guest-agent

# 管理ツール（必要に応じて）
# - Ansible
# - kubectl
# - helm
# - terraform
```

## SSH接続方法

### 直接接続

```bash
ssh admin@10.2.0.2
```

### ジャンプホストとして使用

```bash
# 他のVMへの多段SSH
ssh -J admin@10.2.0.2 user@10.2.0.34

# ~/.ssh/config に設定
Host bastion
    HostName 10.2.0.2
    User admin

Host lab-01
    HostName 10.2.0.34
    User admin
    ProxyJump bastion
```

## セキュリティ設定

### SSH設定（/etc/ssh/sshd_config）

```
# パスワード認証を無効化（鍵認証のみ）
PasswordAuthentication no

# rootログインを無効化
PermitRootLogin no

# 公開鍵認証を有効化
PubkeyAuthentication yes
```

### ファイアウォール設定（ufw）

```bash
# UFWを有効化
sudo ufw enable

# SSH許可
sudo ufw allow 22/tcp

# 状態確認
sudo ufw status
```

## メンテナンス

### システム更新

```bash
# パッケージ更新
sudo apt update && sudo apt upgrade -y

# 再起動（カーネル更新時）
sudo reboot
```

### バックアップ

Proxmox Web UIから定期バックアップを設定:
- Datacenter → Backup
- Schedule: 毎日 2:00 AM
- Storage: vm-pool
- Mode: Snapshot
- Retention: 7日分

## トラブルシューティング

### SSH接続できない

```bash
# Proxmoxノードから直接確認
ssh root@10.1.1.11
qm status 100001  # VM状態確認
qm start 100001   # 停止していたら起動

# ネットワーク確認
ping 10.2.0.2

# VMコンソールで直接ログイン
# Web UI: bastion-01 → Console
```

### ディスク容量不足

```bash
# 使用状況確認
df -h

# 不要ファイル削除
sudo apt autoremove -y
sudo apt clean

# ディスク拡張（Proxmox Web UI）
# bastion-01 → Hardware → Hard Disk → Resize
```

## 関連ドキュメント

- [VM管理ガイド](../README.md)
- [ネットワーク設計](../../docs/design.md)
- [Proxmoxクラスタ構成](../../pm/mandolin/CLUSTER.md)
