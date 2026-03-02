# opencraw-01

OpenCraw実行用隔離VM（Ubuntu 24.04 LTS）

## 基本情報

| 項目 | 値 |
|------|-----|
| VM ID | 103002 |
| VM名 | opencraw-01 |
| IPアドレス | 10.2.0.99/27 |
| ゲートウェイ | 10.2.0.97 |
| DNS | 10.0.0.1 |
| VLAN | 103（AI動作用） |
| Proxmoxノード | mandolin3 |

## ハードウェア構成

| リソース | スペック |
|----------|----------|
| CPU | 4 vCPU (host) |
| メモリ | 16 GiB |
| ディスク | 128 GiB (vm-pool, thin provisioning) |
| ネットワーク | VirtIO (vmbr0, VLAN 103) |

## 用途

Slackで対話可能なOpenCrawを実行する隔離環境。

### 要件

1. **Slackとの対話** — Slack APIを通じてOpenCrawを操作
2. **許可されたSaaSのみアクセス** — HTTPS (443) のアウトバウンドのみ許可
3. **内部ネットワーク隔離** — 他のVM・PMへのネットワークアクセスを遮断

## ネットワーク隔離（Proxmox Firewall）

Proxmox Firewallを使用してVM単位でアクセス制御を行う。

### ファイアウォールルール

Proxmox Web UI: `opencraw-01 → Firewall → Add` または `/etc/pve/firewall/103002.fw`

```
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: DROP

[RULES]
# SSH管理アクセス
IN ACCEPT -p tcp -dport 22
# DNS（名前解決に必要）
OUT ACCEPT -dest 10.0.0.1 -p udp -dport 53
# NTP（時刻同期）
OUT ACCEPT -dest 10.0.0.1 -p udp -dport 123
# 内部ネットワークへのアクセスを拒否（上記ルールより後に評価）
OUT DROP -dest 10.0.0.0/8
# 許可されたSaaS（HTTP/HTTPS）へのアウトバウンド
OUT ACCEPT -p tcp -dport 80
OUT ACCEPT -p tcp -dport 443
```

### 適用手順

```bash
# Proxmox CLIで設定する場合
ssh root@10.1.1.13

# ファイアウォール設定ファイルを作成
cat > /etc/pve/firewall/103002.fw << 'EOF'
[OPTIONS]
enable: 1
policy_in: DROP
policy_out: DROP

[RULES]
IN ACCEPT -p tcp -dport 22
OUT ACCEPT -dest 10.0.0.1 -p udp -dport 53
OUT ACCEPT -dest 10.0.0.1 -p udp -dport 123
OUT DROP -dest 10.0.0.0/8
OUT ACCEPT -p tcp -dport 80
OUT ACCEPT -p tcp -dport 443
EOF
```

> **注意**: Proxmox Firewallはデフォルトでconntrack（ステートフル）が有効のため、
> OUTで許可したTCPセッションの応答パケットは自動的にINで許可されます。

## VM作成手順

```bash
# Proxmoxノード (mandolin3) でVM作成
qm create 103002 --name opencraw-01 --memory 16384 --cores 4 \
  --cpu host --scsihw virtio-scsi-single \
  --scsi0 vm-pool:128,discard=on,ssd=1 \
  --net0 virtio,bridge=vmbr0,tag=103 \
  --ide2 local:iso/ubuntu-24.04.1-live-server-amd64.iso,media=cdrom \
  --ostype l26 --agent enabled=1 --onboot 1

# VM起動
qm start 103002
```

## 関連ドキュメント

- [VM管理ガイド](../README.md)
- [ネットワーク設計](../../docs/design.md)
