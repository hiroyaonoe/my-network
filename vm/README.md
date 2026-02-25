# VM管理ガイド

このディレクトリには各仮想マシンの設定を保存します。

## 目次

- [命名規則](#命名規則)
- [VLAN別設定](#vlan別設定)
- [VM作成手順](#vm作成手順)
- [ディレクトリ構造](#ディレクトリ構造)

## 命名規則

### VM ID

```
VM ID = VLAN Tag (3桁) + VMインデックス (3桁)
```

- **範囲**: 100001〜227999（VLAN 100〜227に対応）
- **例**: 100001, 100002, 101001, 102001

**ルール**:
- VLAN Tag: 3桁（先頭ゼロ埋め）
- VMインデックス: 3桁（001から開始、先頭ゼロ埋め）
- クラスタ全体で一意であること

### VM名

```
<prefix>-<インデックス(2桁)>
```

- **例**: bastion-01, bastion-02, k8s-master-01
- **ルール**:
  - prefix: 用途を表す英数字（小文字、ハイフン可）
  - インデックス: 2桁（01から開始、先頭ゼロ埋め）
  - VM IDのインデックス部分の下2桁と一致させる

### IPアドレス

各サブネット内で順番に割り当て:

- **ゲートウェイ**: 各サブネットの `.1`（例: 10.2.0.1）
- **VM**: `.2` から順番に割り当て

**例**:
- 10.2.0.0/27（踏み台VLAN）:
  - 10.2.0.1: ゲートウェイ（Catalyst）
  - 10.2.0.2: bastion-01（VM ID 100001）
  - 10.2.0.3: bastion-02（VM ID 100002）

## VLAN別設定

### VLAN 100: 踏み台

| 項目 | 値 |
|------|-----|
| 用途 | 踏み台サーバー |
| サブネット | 10.2.0.0/27 |
| ゲートウェイ | 10.2.0.1 |
| 利用可能IP | 10.2.0.2〜10.2.0.30（29台） |
| VM ID範囲 | 100001〜100999 |
| Prefix例 | bastion |

**VM例**:
- VM ID: 100001, VM名: bastion-01, IP: 10.2.0.2
- VM ID: 100002, VM名: bastion-02, IP: 10.2.0.3

### VLAN 101: 検証環境

| 項目 | 値 |
|------|-----|
| 用途 | 検証・テスト環境 |
| サブネット | 10.2.0.32/27 |
| ゲートウェイ | 10.2.0.33 |
| 利用可能IP | 10.2.0.34〜10.2.0.62（29台） |
| VM ID範囲 | 101001〜101999 |
| Prefix例 | lab, test |

**VM例**:
- VM ID: 101001, VM名: lab-01, IP: 10.2.0.34
- VM ID: 101002, VM名: test-01, IP: 10.2.0.35

### VLAN 102: k8sクラスタ

| 項目 | 値 |
|------|-----|
| 用途 | Kubernetesクラスタ |
| サブネット | 10.2.0.64/27 |
| ゲートウェイ | 10.2.0.65 |
| 利用可能IP | 10.2.0.66〜10.2.0.94（29台） |
| VM ID範囲 | 102001〜102999 |
| Prefix例 | k8s-master, k8s-worker |

**VM例**:
- VM ID: 102001, VM名: k8s-master-01, IP: 10.2.0.66
- VM ID: 102002, VM名: k8s-worker-01, IP: 10.2.0.67
- VM ID: 102003, VM名: k8s-worker-02, IP: 10.2.0.68

### VLAN 103: AI動作用

| 項目 | 値 |
|------|-----|
| 用途 | AI/機械学習ワークロード |
| サブネット | 10.2.0.96/27 |
| ゲートウェイ | 10.2.0.97 |
| 利用可能IP | 10.2.0.98〜10.2.0.126（29台） |
| VM ID範囲 | 103001〜103999 |
| Prefix例 | ai-gpu, ml |

**VM例**:
- VM ID: 103001, VM名: ai-gpu-01, IP: 10.2.0.98
- VM ID: 103002, VM名: ml-01, IP: 10.2.0.99

## VM作成手順

### 前提条件

- Proxmoxクラスタが構築済み（mandolin1, mandolin2, mandolin3）
- Cephストレージ（vm-pool）が利用可能
- OSのISOイメージがダウンロード済み

### 方法1: Proxmox Web UIで作成（推奨）

#### 1. ISOイメージのダウンロード

```bash
# いずれかのノードで実行
ssh root@10.1.1.11

# Ubuntu 24.04の場合
cd /var/lib/vz/template/iso
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.1-live-server-amd64.iso
```

#### 2. VMの作成

Proxmox Web UI（https://10.1.1.11:8006）にアクセス:

**General タブ**:
- Node: mandolin1（またはmandolin2, mandolin3）
- VM ID: **100001**（命名規則に従う）
- Name: **bastion-01**（命名規則に従う）
- Start at boot: チェック（必要に応じて）

**OS タブ**:
- Use CD/DVD disc image file (iso): 選択
- Storage: local
- ISO image: ubuntu-24.04.1-live-server-amd64.iso
- Guest OS Type: Linux
- Version: 6.x - 2.6 Kernel

**System タブ**:
- Graphic card: Default
- Machine: Default (i440fx)
- SCSI Controller: VirtIO SCSI single
- Qemu Agent: **チェック✓**（推奨）

**Disks タブ**:
- Bus/Device: SCSI 0
- Storage: **vm-pool**（Cephストレージ）
- Disk size (GiB): 64（要件に応じて）
- Cache: Default (No cache)
- Discard: **チェック✓**（thin provisioningのため）
- SSD emulation: **チェック✓**

**CPU タブ**:
- Sockets: 1
- Cores: 4（要件に応じて）
- Type: host（推奨）

**Memory タブ**:
- Memory (MiB): 8192（8GiB、要件に応じて）
- Minimum memory (MiB): 2048（バルーニング用、オプション）

**Network タブ**:
- Bridge: **vmbr0**
- VLAN Tag: **100**（VLANに応じて変更）
- Model: VirtIO (paravirtualized)

**Confirm タブ**:
- 設定を確認して「Finish」

#### 3. OSのインストール

1. VM（bastion-01）を選択 → 「Start」
2. 「Console」をクリック
3. OSインストーラの指示に従う

**Ubuntu 24.04の場合のネットワーク設定**:
- IPv4 Method: Manual
- Subnet: **10.2.0.0/27**（VLANに応じて変更）
- Address: **10.2.0.2**（命名規則に従う）
- Gateway: **10.2.0.1**（VLANに応じて変更）
- Name servers: **10.0.0.1**
- Search domains: internal.onoe.dev

**その他の設定**:
- SSH Setup: **Install OpenSSH server** にチェック✓
- インストール完了後、Reboot

#### 4. インストール後の設定

```bash
# VMにSSH接続
ssh admin@10.2.0.2

# QEMU Guest Agentをインストール（Proxmox連携用）
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent

# 再起動（必要に応じて）
sudo reboot
```

### 方法2: CLIで作成（上級者向け）

```bash
# いずれかのノードで実行
ssh root@10.1.1.11

# 踏み台VM（bastion-01）の作成例
qm create 100001 \
  --name bastion-01 \
  --memory 8192 \
  --cores 4 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=100 \
  --scsihw virtio-scsi-single \
  --scsi0 vm-pool:64,discard=on,ssd=1 \
  --cdrom local:iso/ubuntu-24.04.1-live-server-amd64.iso \
  --boot order=scsi0 \
  --ostype l26 \
  --agent enabled=1

# VMを起動
qm start 100001

# コンソールで接続してOSインストール
# または Web UIのConsoleを使用
```

### 標準的なVM構成例

#### 踏み台サーバー（bastion-01）

```bash
qm create 100001 \
  --name bastion-01 \
  --memory 8192 \
  --cores 4 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=100 \
  --scsi0 vm-pool:64,discard=on,ssd=1 \
  --agent enabled=1
```

- **用途**: SSH踏み台、管理ツール実行
- **スペック**: 4 vCPU, 8GiB RAM, 64GiB Disk
- **VLAN**: 100（踏み台）
- **IP**: 10.2.0.2

#### k8s Masterノード（k8s-master-01）

```bash
qm create 102001 \
  --name k8s-master-01 \
  --memory 16384 \
  --cores 4 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --scsi0 vm-pool:100,discard=on,ssd=1 \
  --agent enabled=1
```

- **用途**: Kubernetes Masterノード
- **スペック**: 4 vCPU, 16GiB RAM, 100GiB Disk
- **VLAN**: 102（k8s）
- **IP**: 10.2.0.66

#### k8s Workerノード（k8s-worker-01）

```bash
qm create 102002 \
  --name k8s-worker-01 \
  --memory 32768 \
  --cores 8 \
  --cpu host \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --scsi0 vm-pool:200,discard=on,ssd=1 \
  --agent enabled=1
```

- **用途**: Kubernetes Workerノード
- **スペック**: 8 vCPU, 32GiB RAM, 200GiB Disk
- **VLAN**: 102（k8s）
- **IP**: 10.2.0.67

### VMの確認

```bash
# VM一覧の表示
qm list

# 特定VMの状態確認
qm status 100001

# VM設定の確認
qm config 100001

# VM起動
qm start 100001

# VM停止
qm stop 100001

# VM削除（注意: データも削除されます）
qm destroy 100001
```

## ディレクトリ構造

```
vm/
├── README.md                      # このファイル
├── bastion-01/
│   ├── vm-config.json             # VM設定（CPU, RAM, VLAN等）
│   ├── cloud-init.yaml            # cloud-init設定（オプション）
│   └── README.md                  # VMの用途説明
├── k8s-master-01/
│   ├── vm-config.json
│   └── README.md
└── k8s-worker-01/
    ├── vm-config.json
    └── README.md
```

### vm-config.json 例

```json
{
  "vm_id": 100001,
  "vm_name": "bastion-01",
  "vlan_tag": 100,
  "ip_address": "10.2.0.2/27",
  "gateway": "10.2.0.1",
  "dns": "10.0.0.1",
  "cpu_cores": 4,
  "memory_mb": 8192,
  "disk_gb": 64,
  "os": "Ubuntu 24.04 LTS",
  "purpose": "SSH踏み台、管理ツール実行環境",
  "created_at": "2026-02-26",
  "node": "mandolin1"
}
```

## 参考情報

- Proxmoxクラスタセットアップ: [pm/mandolin/CLUSTER.md](../pm/mandolin/CLUSTER.md)
- ネットワーク設計: [docs/design.md](../docs/design.md)
- Catalyst設定: [router/catalyst-3560cx.conf](../router/catalyst-3560cx.conf)
