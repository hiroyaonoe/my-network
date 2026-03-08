# VM基盤 設計書

## 1. 概要

Proxmox VEを用いた3ノードの仮想化基盤を構築する。
ストレージはCephクラスタによる分散ストレージを採用し、ネットワークはCatalyst 3560-CXをローカルルータ兼L3スイッチとして利用する。

## 2. 物理構成

### 2.1 機器一覧

| 機器 | 役割 | 備考 |
|------|------|------|
| WAN側ルータ | インターネット接続、DHCP、DNS/NTP提供 | 既存 |
| Catalyst WS-C3560CX-8PC-S | ローカルルータ / L3スイッチ | 8x 1GbE PoE+ |
| 物理マシン1 (mandolin1) | Proxmoxノード | |
| 物理マシン2 (mandolin2) | Proxmoxノード | |
| 物理マシン3 (mandolin3) | Proxmoxノード | |

### 2.2 物理マシンスペック (各ノード共通)

| 項目 | スペック |
|------|---------|
| CPU | AMD Ryzen 7 5000 series |
| RAM | 16GB |
| SSD 1 | 512GB (Proxmox OS用) |
| SSD 2 | 1TB (Ceph OSD用) |
| NIC | 2.5GbE x 2ポート |

> **注意**: WS-C3560CX-8PC-Sのポートは1GbEのため、2.5GbE NICは1Gbpsでネゴシエーションされる。
> LACP bonding時の最大帯域は2Gbps（ただしハッシュベース分散のため単一フローは1Gbps上限）。

## 3. ネットワーク設計

### 3.1 全体構成図

```
[Internet]
    |
[WAN側ルータ]
    |  LAN: 10.0.0.0/16
    |  DHCP: 10.0.0.0/24 (Wi-Fi等)
    |  DNS/NTP提供
    |
    | (10.0.0.1 --- 10.0.1.1)
    |
[Catalyst 3560-CX] ローカルルータ / L3スイッチ
    |  default route → 10.0.0.1
    |
    |-- Gi0/1: WAN側ルータ (routed port)
    |-- Gi0/2: 空き
    |-- Gi0/3-4: mandolin1 (LACP Po1, trunk)
    |-- Gi0/5-6: mandolin2 (LACP Po2, trunk)
    |-- Gi0/7-8: mandolin3 (LACP Po3, trunk)
    |
    +-- VLAN 10  [mgmt]       10.1.1.0/24    Proxmox管理
    +-- VLAN 20  [ceph]       10.1.2.0/24    Cephレプリケーション
    +-- VLAN 100-227 [vm-*]   10.2.0.0/16内の/27を連番で割当 (最大128個)
```

### 3.2 VLAN設計

#### インフラVLAN (固定)

| VLAN ID | 名称 | サブネット | 用途 |
|---------|------|-----------|------|
| 10 | mgmt | 10.1.1.0/24 | Proxmox管理、Corosync、Ceph Public |
| 20 | ceph | 10.1.2.0/24 | Ceph Cluster (レプリケーション専用) |

#### VM用VLAN (100-227)

10.2.0.0/16 の中で /27 (30ホスト) を隙間なく連番で割り当てる。最大128個。

採番ルール: **VLAN (100 + N)** → **10.2.(N/8).(N%8\*32)/27** (N = 0〜127)

| N | VLAN ID | サブネット | ゲートウェイ | ホスト範囲 |
|---|---------|-----------|-------------|-----------|
| 0 | 100 | 10.2.0.0/27 | 10.2.0.1 | .2 〜 .30 |
| 1 | 101 | 10.2.0.32/27 | 10.2.0.33 | .34 〜 .62 |
| 2 | 102 | 10.2.0.64/27 | 10.2.0.65 | .66 〜 .94 |
| 3 | 103 | 10.2.0.96/27 | 10.2.0.97 | .98 〜 .126 |
| 4 | 104 | 10.2.0.128/27 | 10.2.0.129 | .130 〜 .158 |
| ... | ... | ... | ... | ... |
| 8 | 108 | 10.2.1.0/27 | 10.2.1.1 | .2 〜 .30 |
| ... | ... | ... | ... | ... |
| 127 | 227 | 10.2.15.224/27 | 10.2.15.225 | .226 〜 .254 |

初期割当:

| VLAN ID | N | 名称 | サブネット | 用途 |
|---------|---|------|-----------|------|
| 100 | 0 | vm-bastion | 10.2.0.0/27 | 踏み台 |
| 101 | 1 | vm-lab | 10.2.0.32/27 | 検証環境 |
| 102 | 2 | vm-k8s | 10.2.0.64/27 | k8sクラスタ |
| 103 | 3 | vm-ai | 10.2.0.96/27 | AI動作用 |

> trunk の allowed vlan および Proxmox の bridge-vids は 100-227 の範囲指定としている。
> VLAN追加時はCatalystのVLAN定義とSVIの追加のみで利用開始できる。

### 3.3 IPアドレス設計

#### WAN側ネットワーク (10.0.0.0/16)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| WAN側ルータ (LAN側) | 10.0.0.1 | |
| DHCP範囲 | 10.0.0.100〜10.0.0.254 (想定) | Wi-Fi等 |
| ローカルルータ (Gi0/1) | 10.0.1.1 | 静的割り当て |

#### VLAN 10: 管理ネットワーク (10.1.1.0/24)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| ローカルルータ (SVI) | 10.1.1.1 | ゲートウェイ |
| mandolin1 | 10.1.1.11 | Proxmox管理IP |
| mandolin2 | 10.1.1.12 | Proxmox管理IP |
| mandolin3 | 10.1.1.13 | Proxmox管理IP |

Proxmox Web UI: `https://10.1.1.1x:8006`

#### VLAN 20: Cephネットワーク (10.1.2.0/24)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| mandolin1 | 10.1.2.11 | Ceph Cluster Network |
| mandolin2 | 10.1.2.12 | Ceph Cluster Network |
| mandolin3 | 10.1.2.13 | Ceph Cluster Network |

> ゲートウェイ不要。ノード間のOSDレプリケーション通信のみ。

#### VLAN 100〜227: VMネットワーク (10.2.0.0/16)

採番ルールに従い、/27を連番で詰めて割り当てる（3.2節参照）。

> 各VMにはサブネット内で静的にIPを割り当てる。各/27サブネットで使用可能なホストアドレスは30個。

#### k8sクラスタ内部ネットワーク (10.0.0.0/8 の未使用範囲)

| 用途 | CIDR | 備考 |
|------|------|------|
| Pod Network | 10.3.0.0/16 | Cilium IPAM (kubernetes mode), per-node /24 |
| Service Network | 10.4.0.0/16 | Kubernetes ClusterIP Service CIDR |
| LoadBalancer VIP | 10.5.0.0/24 | CiliumLoadBalancerIPPool + L2 Announcement |

> Pod Network と Service Network は K8s クラスタ内部でのみ使用され、外部からは直接到達不可。
> LoadBalancer VIP は Cilium L2 Announcement により VLAN 102 上で ARP 応答し、Catalyst のインターフェースルートで到達可能。

### 3.4 ルーティング設計

#### WAN側ルータ

```
default → WAN (インターネット)
10.0.0.0/16 (LAN) → directly connected
```

> すべてのネットワークが10.0.0.0/8内に統一されているため、追加のスタティックルートは不要。

#### ローカルルータ (Catalyst 3560-CX)

```
default         → 10.0.0.1 (WAN側ルータ)
10.0.0.0/16     → Gi0/1 (directly connected, WAN側ルータ接続)
10.1.1.0/24     → VLAN 10 (directly connected, 管理)
10.1.2.0/24     → VLAN 20 (directly connected, Ceph)
10.2.x.x/27     → VLAN 100-227 (SVIが存在するVLANのみ、directly connected)
10.5.0.0/24     → Vlan102 (インターフェースルート, K8s LB VIP)
```

> SVI間ルーティングにより、SVIが定義された全VLANは相互に通信可能。
> VM用VLANの追加時はSVIを作成するだけでconnected routeが自動生成される。
> 必要に応じてACLでVLAN間のアクセス制御を実施する。
> 10.5.0.0/24 宛のパケットは VLAN 102 上で ARP を発行し、Cilium L2 Announcement のリーダーノードが応答する。

### 3.5 Bonding (LACP) 設計

#### Catalyst側 Port-Channel

| Port-Channel | メンバーポート | 接続先 |
|-------------|--------------|--------|
| Po1 | Gi0/3, Gi0/4 | mandolin1 |
| Po2 | Gi0/5, Gi0/6 | mandolin2 |
| Po3 | Gi0/7, Gi0/8 | mandolin3 |

各Port-Channelの設定:
- モード: LACP (802.3ad) active
- Switchport mode: trunk
- Native VLAN: 10 (管理)
- Allowed VLANs: 10, 20, 100-227

#### Proxmox側 Linux Bond

各ノードで bond0 を構成:
- mode: 802.3ad (LACP)
- xmit_hash_policy: layer3+4
- miimon: 100

## 4. VM基盤設計

### 4.1 Proxmox VEクラスタ

| 項目 | 値 |
|------|-----|
| バージョン | Proxmox VE 8.x (no-subscription) |
| クラスタ名 | mandolin |
| ノード数 | 3 |
| Quorumデバイス | 不要 (3ノードで過半数を満たす) |

ノード一覧:

| ノード名 | 管理IP | Ceph IP |
|---------|--------|---------|
| mandolin1 | 10.1.1.11 | 10.1.2.11 |
| mandolin2 | 10.1.1.12 | 10.1.2.12 |
| mandolin3 | 10.1.1.13 | 10.1.2.13 |

### 4.2 Proxmoxネットワーク構成

各ノードの `/etc/network/interfaces` 構成:

```
# 物理NIC
auto eno1
iface eno1 inet manual

auto eno2
iface eno2 inet manual

# LACP Bond
auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-miimon 100
    bond-mode 802.3ad
    bond-xmit-hash-policy layer3+4

# VLAN-aware Bridge (メインブリッジ)
auto vmbr0
iface vmbr0 inet static
    address 10.1.1.1x/24       # 各ノードに応じて .11/.12/.13
    gateway 10.1.1.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 20 100-227

# Cephネットワーク (VLAN 20)
auto vmbr0.20
iface vmbr0.20 inet static
    address 10.1.2.1x/24       # 各ノードに応じて .11/.12/.13
```

> NIC名 (eno1, eno2) は実機に合わせて変更する。
> vmbr0のnative VLAN (untagged) がVLAN 10 (管理)、tagged VLANでCeph・VM通信を行う。
> VMは作成時にvmbr0を指定し、VLANタグを設定することで各VLANに配置する。

### 4.3 Cephクラスタ

| 項目 | 値 |
|------|-----|
| Cephバージョン | Proxmox VE同梱 (Reef / Squid) |
| Monitor (MON) | 3台 (mandolin1, mandolin2, mandolin3) |
| Manager (MGR) | 3台 (mandolin1, mandolin2, mandolin3) |
| OSD数 | 3 (各ノード 1TB SSD x 1) |
| Public Network | 10.1.1.0/24 (VLAN 10) |
| Cluster Network | 10.1.2.0/24 (VLAN 20) |

> Public Network: クライアント (VM) → OSD のI/O通信に使用。管理ネットワークと共用。
> Cluster Network: OSD間レプリケーション/リカバリ通信専用。管理トラフィックと分離。

#### ストレージプール

| プール名 | size (replica) | min_size | 用途 |
|----------|---------------|----------|------|
| vm-pool | 3 | 2 | VMディスク |
| k8s-rbd | 3 | 2 | K8s PVC (RBD) |

| 項目 | 値 |
|------|-----|
| 総物理容量 | 3TB (1TB x 3 OSD) |
| 実効容量 (replica=3) | 約1TB |

> **容量に関する注意**: replica=3 で実効約1TB。容量が不足する場合は replica=2 (実効約1.5TB) も検討可能だが、1ノード障害時のデータ冗長性が低下する。

#### リソース見積もり (RAMの内訳)

| 用途 | 消費量 (目安) | 備考 |
|------|-------------|------|
| Proxmox OS | 約1GB | |
| Ceph MON + MGR | 約1〜2GB | |
| Ceph OSD (1基) | 約2GB | osd_memory_target=2GB |
| VMに割当可能 | 約11〜12GB | 残りをVMに使用 |

## 5. Catalyst 3560-CX 設定例

### 5.1 基本設定

```
hostname local-router

ip routing

ip name-server 10.0.0.1
ntp server 10.0.0.1
```

### 5.2 VLAN定義

```
vlan 10
 name mgmt
vlan 20
 name ceph
! 初期VM用VLAN (採番ルールに従い追加可能)
vlan 100
 name vm-bastion
vlan 101
 name vm-lab
vlan 102
 name vm-k8s
vlan 103
 name vm-ai
```

### 5.3 SVI (VLAN Interface)

```
interface Vlan10
 ip address 10.1.1.1 255.255.255.0
 no shutdown

interface Vlan20
 ip address 10.1.2.1 255.255.255.0
 no shutdown

! 初期VM用SVI (採番ルール: VLAN 100+N → 10.2.(N/8).(N%8*32)/27)
interface Vlan100
 ip address 10.2.0.1 255.255.255.224
 no shutdown

interface Vlan101
 ip address 10.2.0.33 255.255.255.224
 no shutdown

interface Vlan102
 ip address 10.2.0.65 255.255.255.224
 no shutdown

interface Vlan103
 ip address 10.2.0.97 255.255.255.224
 no shutdown
```

### 5.4 物理ポート設定

```
! WAN側ルータ接続 (Routed Port)
interface GigabitEthernet0/1
 no switchport
 ip address 10.0.1.1 255.255.0.0
 no shutdown

! 空きポート
interface GigabitEthernet0/2
 shutdown

! mandolin1 LACP
interface GigabitEthernet0/3
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 1 mode active
 no shutdown

interface GigabitEthernet0/4
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 1 mode active
 no shutdown

! mandolin2 LACP
interface GigabitEthernet0/5
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 2 mode active
 no shutdown

interface GigabitEthernet0/6
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 2 mode active
 no shutdown

! mandolin3 LACP
interface GigabitEthernet0/7
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 3 mode active
 no shutdown

interface GigabitEthernet0/8
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
 channel-group 3 mode active
 no shutdown
```

### 5.5 Port-Channel

```
interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227

interface Port-channel2
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227

interface Port-channel3
 switchport mode trunk
 switchport trunk native vlan 10
 switchport trunk allowed vlan 10,20,100-227
```

### 5.6 ルーティング

```
ip route 0.0.0.0 0.0.0.0 10.0.0.1

! K8s LoadBalancer VIP → VLAN 102 上で ARP (Cilium L2 Announcement が応答)
ip route 10.5.0.0 255.255.255.0 Vlan102
```

> Connected routeにより各SVI配下のサブネットは自動的にルーティングされる。
> すべてのネットワークが10.0.0.0/8内に統一されているため、WAN側ルータでの追加ルート設定は不要。
> 10.5.0.0/24 のインターフェースルートにより、K8s LoadBalancer VIP への通信が VLAN 102 上の Cilium L2 Announcement で処理される。

### 5.7 DHCP サーバー

```
ip dhcp pool k8s-dhcp
 network 10.2.0.64 255.255.255.224
 default-router 10.2.0.65
 dns-server 10.0.0.1
 lease 0 1 0

ip dhcp excluded-address 10.2.0.64 10.2.0.79
ip dhcp excluded-address 10.2.0.91 10.2.0.95
```

> VLAN 102 (vm-k8s) 用の DHCP サーバー。Talos Linux VM の初回起動時に一時 IP (10.2.0.80〜90) を割り当てる。
> Machine config 適用後、各 VM は静的 IP (10.2.0.66〜71) に切り替わる。

## 6. 制約事項・注意点

1. **帯域制約**: WS-C3560CX-8PC-Sは1GbEポートのため、2.5GbE NICは1Gbpsでネゴシエーションされる。LACPにより最大2Gbps（ハッシュベース分散のため単一フローは1Gbps上限）。
2. **Ceph最小構成**: 3ノード x 1 OSDはCephの最小構成。1ノード障害時、replica=3/min_size=2ではデータ読み取りは可能だが新規書き込みが制限される可能性がある。
3. **RAM制約**: 16GBのうちOS+Cephで約4〜5GB消費。VMに割当可能なRAMは11〜12GB/ノード。クラスタ全体で約33〜36GB。
4. **Proxmox無償版**: Enterprise repositoryは使用不可。`pve-no-subscription` repositoryを使用する。サブスクリプション警告が表示されるが機能制限はない。
5. **VLAN間アクセス制御**: デフォルトでは全VLAN間が通信可能。必要に応じてCatalystのACLでアクセス制御を行う。

## 7. 運用手順

### 7.1 VM用VLANの追加

trunk/bridge設定は範囲指定済みのため、以下のみで完了する。

1. **Catalyst: VLAN定義を追加**
   ```
   vlan <100+N>
    name vm-<用途名>
   ```
2. **Catalyst: SVIを追加** (採番ルールからアドレスを算出)
   ```
   interface Vlan<100+N>
    ip address 10.2.<N/8>.<N%8*32 + 1> 255.255.255.224
    no shutdown
   ```
3. VMを作成（→ 7.2参照）

> Proxmoxノード側やtrunkの設定変更は不要。

### 7.2 VMの作成とVLAN配置

vmbr0は「VLAN-aware bridge」として構成されているため、VM作成時にVLANタグを指定するだけで任意のVLANに配置できる。**VM内部ではVLAN設定は不要。**

#### Web UIでの設定

1. Proxmox Web UI (`https://10.1.1.1x:8006`) にログイン
2. VM作成または既存VMの編集
3. **Hardware** タブ → **Add** → **Network Device** (または既存のネットワークデバイスを編集)
4. 以下を設定：
   - **Bridge**: `vmbr0`
   - **VLAN Tag**: 配置先のVLAN ID (例: `100` = 踏み台, `102` = k8s)
   - **Model**: `VirtIO (paravirtualized)` 推奨
5. **Add** / **OK**

#### CLIでの設定

```bash
# 新規VM作成時
qm create <vmid> --name <vm-name> --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0,tag=102

# 既存VMのネットワーク変更
qm set <vmid> --net0 virtio,bridge=vmbr0,tag=102

# 設定確認
qm config <vmid> | grep net0
```

#### VLAN別の用途例

| VLAN Tag | 用途 | サブネット | VMに割当可能なIP範囲 |
|----------|------|-----------|-------------------|
| 100 | 踏み台 | 10.2.0.0/27 | 10.2.0.2 〜 10.2.0.30 |
| 101 | 検証環境 | 10.2.0.32/27 | 10.2.0.34 〜 10.2.0.62 |
| 102 | k8sクラスタ | 10.2.0.64/27 | 10.2.0.66 〜 10.2.0.94 |
| 103 | AI動作用 | 10.2.0.96/27 | 10.2.0.98 〜 10.2.0.126 |

> VM内のOS設定では、通常のイーサネットとして静的IPまたはDHCP（別途構築する場合）を設定する。
> ゲートウェイは各VLANのSVI (10.2.0.1, 10.2.0.33, 10.2.0.65, 10.2.0.97...) を指定。

## 8. K8sクラスタ構成

### 8.1 概要

VLAN 102 (vm-k8s, 10.2.0.64/27) に Talos Linux ベースの HA Kubernetes クラスタを構築する。
CNI には Cilium を採用し、Native Routing + L2 Announcement で LoadBalancer VIP を提供する。

| 項目 | 値 |
|------|-----|
| OS | Talos Linux |
| K8s バージョン | 1.35.2 |
| CNI | Cilium (Native Routing) |
| API Server VIP | 10.2.0.94 (Talos 内蔵 VIP) |
| API Endpoint | `https://10.2.0.94:6443` |

### 8.2 ノード構成

| VM名 | VM ID | IP | Proxmoxノード | 役割 | vCPU | RAM | Disk |
|-------|-------|-----|---------------|------|------|-----|------|
| k8s-cp-01 | 102001 | 10.2.0.66 | mandolin1 | control-plane | 4 | 8GB | 32GB |
| k8s-cp-02 | 102002 | 10.2.0.67 | mandolin2 | control-plane | 4 | 8GB | 32GB |
| k8s-cp-03 | 102003 | 10.2.0.68 | mandolin3 | control-plane | 4 | 8GB | 32GB |
| k8s-worker-01 | 102004 | 10.2.0.69 | mandolin1 | worker | 4 | 8GB | 64GB |
| k8s-worker-02 | 102005 | 10.2.0.70 | mandolin2 | worker | 4 | 8GB | 64GB |
| k8s-worker-03 | 102006 | 10.2.0.71 | mandolin3 | worker | 4 | 8GB | 64GB |

> 各ノードで memory ballooning を有効化し、実使用量に応じた動的調整を行う。
> ストレージ合計: 3x32 + 3x64 = 288GB (Ceph vm-pool 使用)

### 8.3 ネットワーク方式

```
[他 VLAN のクライアント]
    |
    | dst: 10.5.0.x (LB VIP)
    v
[Catalyst SVI Vlan102]
    | ip route 10.5.0.0/24 Vlan102 → ARP for 10.5.0.x on VLAN 102
    v
[Cilium L2 Announcement]  ← リーダーノードが ARP 応答
    |
    v
[K8s Service → Backend Pod]
```

- **Pod 間通信**: Cilium `autoDirectNodeRoutes` により各ノードのカーネルルーティングテーブルに直接ルート挿入 (全ノード同一 L2 のため BGP 不要)
- **LB VIP 公開**: Cilium L2 Announcement (ARP 応答) + Catalyst インターフェースルート。ノード障害時は Gratuitous ARP でフェイルオーバー
- **Pod の外部通信**: SNAT (マスカレード、Pod は Node IP で外部に出る)
- **kube-proxy**: 無効化 (Cilium で代替)

### 8.4 ストレージ (Rook Ceph External)

Rook の external cluster モードで Proxmox Ceph クラスタに接続し、RBD ブロックストレージを提供する。

```
K8s Pod (PVC: ceph-block)
  ↓ CSI Driver (rook-ceph)
K8s Node (10.2.0.66-71, VLAN 102)
  ↓ Catalyst L3 routing
Ceph Monitor (10.1.1.11-13, VLAN 10)
  ↓
Ceph OSD (k8s-rbd pool, 3x replication)
```

| 項目 | 値 |
|------|-----|
| モード | External Cluster (CSI ドライバのみ) |
| Ceph プール | k8s-rbd (3x replication) |
| StorageClass | ceph-block (default) |
| ストレージ種別 | RBD (ブロック) |

> Rook は Ceph デーモン (MON/OSD) を管理しない。CSI ドライバのみを K8s 内にデプロイし、PVC → RBD イメージのプロビジョニングを行う。
> vm-pool (~288GB) と k8s-rbd が Ceph の実効容量 (~1TB) を共有するため、計画的に利用する。

### 8.5 リソース見積もり (オーバーコミット)

| Proxmoxノード | 既存VM | K8s VM | 合計 RAM | オーバーコミット比 |
|--------------|--------|--------|---------|-----------------|
| mandolin1 | bastion-01 (8GB) | cp-01 (8GB) + worker-01 (8GB) | 24GB | ~2.2x |
| mandolin2 | なし | cp-02 (8GB) + worker-02 (8GB) | 16GB | ~1.5x |
| mandolin3 | なし | cp-03 (8GB) + worker-03 (8GB) | 16GB | ~1.5x |

> Proxmox の memory ballooning で実使用量に応じた動的調整を行う。

## 9. ArgoCD GitOps

### 9.1 概要

ArgoCD を用いた GitOps でクラスタコンポーネントを管理する。
ArgoCD 自身も self-managed とし、全ての設定変更を Git push → ArgoCD sync で反映する。

| 項目 | 値 |
|------|-----|
| ArgoCD UI | https://argocd.internal.onoe.dev (10.5.0.1, LoadBalancer) |
| パターン | App of Apps |
| Self-managed | Yes |

### 9.2 ブートストラップ順序

Cilium (CNI) → ArgoCD → Root Application の順で手動インストールし、以降は ArgoCD が引き継ぐ。

```
1. helm install cilium          ← CNI がないと Pod が動かない (手動)
2. kubectl apply cilium CRDs    ← LB IP Pool + L2 Policy (手動)
3. helm install argo-cd         ← ArgoCD 本体 (手動)
4. kubectl apply root app       ← Root Application 投入 (手動)
5. ArgoCD が全てを引き継ぐ      ← 以降は Git → ArgoCD の自動同期
```

### 9.3 App of Apps 構成

```
Root Application (k8s/argocd/apps/)
├── cilium              … Cilium Helm chart (multi-source)
├── cilium-config       … Cilium CRD マニフェスト (LB IP Pool, L2 Policy)
├── argocd              … ArgoCD self-management (multi-source)
├── cert-manager        … cert-manager Helm chart (multi-source)
├── cert-manager-config … ClusterIssuer + CA Certificate マニフェスト
├── rook-ceph           … Rook Operator + CSI ドライバ (multi-source)
├── rook-ceph-cluster   … CephCluster CR + StorageClass (multi-source)
├── k8s-gateway         … HA DNS サーバー (multi-source)
├── monitoring          … kube-prometheus-stack (multi-source, ServerSideApply)
├── monitoring-config   … Grafana TLS Certificate + PVE Exporter マニフェスト
├── loki                … Loki ログ収集 (multi-source)
├── alloy               … Grafana Alloy ログ転送 DaemonSet (multi-source)
├── tempo               … Grafana Tempo 分散トレース (multi-source)
├── opentelemetry       … OTel Collector トレース収集 (multi-source)
├── vault              … HashiCorp Vault (HA Raft, Secret 管理)
├── vault-config       … Vault TLS Certificate マニフェスト
├── external-secrets   … External Secrets Operator (Helm)
├── external-secrets-config … ClusterSecretStore マニフェスト
└── (将来のアプリ)       … Application YAML を追加するだけ
```

### 9.4 アプリ追加フロー

1. `k8s/<app-name>/` に values.yaml やマニフェストを作成
2. `k8s/argocd/apps/<app-name>.yaml` に Application YAML を作成
3. Git push → ArgoCD が自動検出・デプロイ

## 10. HA DNS サーバー (k8s_gateway)

### 10.1 概要

K8s クラスタ上に HA な DNS サーバーを構築し、全 VM・Tailscale デバイスから利用可能にする。
CoreDNS + k8s_gateway プラグインにより、静的レコード・K8s Service 自動登録・上流 DNS 転送を一元的に処理する。

| 項目 | 値 |
|------|-----|
| ドメイン | internal.onoe.dev |
| DNS VIP | 10.5.0.53 (LoadBalancer, Cilium L2 Announcement) |
| レプリカ数 | 3 (topologySpreadConstraints で各 worker に分散) |
| 上流 DNS | 10.0.0.1 (WAN 側ルータ) |

### 10.2 アーキテクチャ

```
Client (VM / Tailscale)
  ↓ DNS query (any domain)
k8s_gateway (10.5.0.53:53, LoadBalancer, 3 replicas)
  ↓ CoreDNS (.:1053)
  ├── hosts plugin → 静的レコード (mandolin1.internal.onoe.dev 等) → fallthrough
  ├── k8s_gateway plugin → K8s Service/HTTPRoute 自動解決 → fallthrough
  └── forward plugin → 10.0.0.1 (上流 DNS)
```

- `internal.onoe.dev` のクエリ → hosts (静的) → k8s_gateway (K8s リソース) → forward (上流)
- それ以外のクエリ (google.com 等) → hosts ミス → k8s_gateway スキップ → forward で上流に転送
- `external-dns.alpha.kubernetes.io/hostname` アノテーションでカスタム名を設定可能
- アノテーション**なし**の LB Service は `<service>.<namespace>.internal.onoe.dev` で自動解決
- アノテーション**あり**の LB Service はカスタム名**のみ**で解決 (自動の `service.namespace.domain` 形式は生成されない)

### 10.3 静的レコード

| ホスト名 | IP | 用途 |
|----------|-----|------|
| mandolin1.internal.onoe.dev | 10.1.1.11 | Proxmox ノード |
| mandolin2.internal.onoe.dev | 10.1.1.12 | Proxmox ノード |
| mandolin3.internal.onoe.dev | 10.1.1.13 | Proxmox ノード |
| bastion-01.internal.onoe.dev | 10.2.0.2 | 踏み台 VM |
| k8s-api.internal.onoe.dev | 10.2.0.94 | K8s API Server VIP |

### 10.4 フォールバック設計

| クライアント | Primary DNS | Fallback | 備考 |
|-------------|-------------|----------|------|
| VM | 10.5.0.53 | 10.0.0.1 | 各 VM の OS 側で DNS を設定 |
| Tailscale | 10.5.0.53 (split DNS) | - | `internal.onoe.dev` のみ転送 |
| K8s ノード自身 | 10.0.0.1 | - | 循環依存回避、変更なし |

> K8s ノードが自身の DNS を使うと、DNS Pod が起動する前にクエリが失敗する循環依存が発生するため、K8s ノードは上流 DNS (10.0.0.1) を使い続ける。
> VM の DNS 変更は各 VM の OS 側 (resolv.conf 等) で個別に設定する。Catalyst DHCP は変更しない。

## 11. TLS (cert-manager + Let's Encrypt)

### 11.1 概要

`.dev` TLD は HSTS preload リストに登録されており、ブラウザは `http://*.dev` を自動的に `https://` にリダイレクトする。
cert-manager + Let's Encrypt で各サービスに TLS 証明書を発行する。
DNS-01 チャレンジに Cloudflare API を使用する。

| 項目 | 値 |
|------|-----|
| cert-manager | cert-manager namespace |
| ClusterIssuer | letsencrypt (ACME DNS-01, Cloudflare) |
| 証明書 | 各サービスごとに発行 (90日、30日前に自動更新) |

### 11.2 アーキテクチャ

```
cert-manager (cert-manager namespace)
  ├── ClusterIssuer (letsencrypt)  ← Let's Encrypt ACME + Cloudflare DNS-01
  └── ExternalSecret               ← Vault → Cloudflare API Token

各 namespace
  └── Certificate CR → *-server-tls Secret
        ↑ cert-manager が Let's Encrypt で署名
```

### 11.3 Cloudflare API Token

cert-manager が DNS-01 チャレンジで TXT レコードを自動作成・削除するために使用する。
Vault に格納し、ExternalSecret で cert-manager namespace に同期する。

| 項目 | 値 |
|------|-----|
| Permissions | Zone:DNS:Edit, Zone:Zone:Read |
| Zone Resources | onoe.dev のみ |
| Vault パス | secret/cert-manager/cloudflare-api-token |

## 12. オブザーバビリティ (Grafana Stack)

### 12.1 概要

K8s クラスタにフルスタックのオブザーバビリティ環境を構築する。
メトリクス監視 (CPU/メモリ/Pod 状態)、ログ収集・可視化、分散トレース収集・可視化、アラート → Slack 通知、Proxmox/Ceph/VM 監視を提供する。

### 12.2 アーキテクチャ

```
[Grafana] (10.5.0.2, grafana.internal.onoe.dev, TLS)
  ├── datasource: Prometheus (metrics)
  ├── datasource: Loki (logs)
  └── datasource: Tempo (traces)

[Prometheus] ← scrape ← node-exporter, kube-state-metrics, pve-exporter
  ├── additionalScrapeConfigs → Ceph MGR (10.1.1.11-13:9283)
  └── remoteWriteReceiver ← Tempo metrics-generator

[Alertmanager] → Slack webhook (Secret)

[Loki] (monolithic) ← [Alloy DaemonSet] (pod logs)

[Tempo] (monolithic) ← [OTel Collector] ← apps (OTLP)
  └── metrics-generator → Prometheus (RED metrics)
```

### 12.3 Namespace 構成

| Namespace | コンポーネント |
|-----------|-------------|
| monitoring | kube-prometheus-stack (Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics, operator) + pve-exporter + Grafana TLS cert |
| loki | Loki (monolithic) |
| alloy | Grafana Alloy (DaemonSet) |
| tempo | Grafana Tempo (monolithic) |
| opentelemetry | OTel Collector (Deployment) |

### 12.4 ストレージ・リソース見積もり

| PVC | サイズ | リソース Request (CPU/Mem) |
|-----|-------|--------------------------|
| Prometheus TSDB | 50Gi | 200m / 512Mi |
| Alertmanager | 5Gi | 50m / 64Mi |
| Grafana | 10Gi | 100m / 256Mi |
| Loki | 30Gi | 100m / 256Mi |
| Tempo | 20Gi | 100m / 256Mi |
| **合計** | **115Gi** | **~1.5 cores / ~2.3Gi** |

Worker あたり約 770Mi request。各 8GB RAM で十分収容可能。

### 12.5 手動前提条件

デプロイ前に以下を実施する:

1. **Proxmox API トークン作成**: `monitor@pve` ユーザー + `prometheus` トークン (PVEAuditor ロール)
2. **Ceph Prometheus Module 有効化**: `ceph mgr module enable prometheus` (ポート 9283)
3. **Slack Webhook URL 取得**: Incoming Webhooks 設定
4. **K8s Secrets 作成**: `alertmanager-slack-webhook`, `pve-exporter-credentials` (monitoring namespace)

## 13. Secret 管理 (Vault + External Secrets Operator)

### 13.1 概要

K8s Secret を GitOps 管理するため、HashiCorp Vault と External Secrets Operator (ESO) を導入する。
Vault に Secret 値を格納し、ESO が ExternalSecret CR に基づいて K8s Secret を自動生成する。

| 項目 | 値 |
|------|-----|
| Vault | vault namespace, HA Raft 3ノード |
| Vault UI | https://vault.internal.onoe.dev (10.5.0.3, LoadBalancer) |
| ESO | external-secrets namespace |
| Secret Store | ClusterSecretStore (Vault KV v2, Kubernetes auth) |
| TLS | cert-manager (internal-ca ClusterIssuer) |

### 13.2 アーキテクチャ

```
[Vault] (10.5.0.3, vault.internal.onoe.dev, TLS)
  ├── HA Raft storage (3 replicas, anti-affinity)
  ├── KV v2 Secret Engine (path: secret/)
  └── Kubernetes Auth (ESO 用 ServiceAccount)

[External Secrets Operator] (external-secrets namespace)
  └── ClusterSecretStore → Vault KV v2

[ExternalSecret CR] (各 namespace)
  ↓ ESO が定期的に同期
[K8s Secret] (自動生成)
  ↓
[Workload] (Alertmanager, PVE Exporter 等)
```

### 13.3 LoadBalancer VIP 割り当て

| VIP | ホスト名 | 用途 |
|-----|---------|------|
| 10.5.0.1 | argocd.internal.onoe.dev | ArgoCD |
| 10.5.0.2 | grafana.internal.onoe.dev | Grafana |
| 10.5.0.3 | vault.internal.onoe.dev | Vault |
| 10.5.0.53 | (DNS) | k8s-gateway |

### 13.4 手動前提条件

Vault デプロイ後に以下の手動操作が必要:

1. **Vault 初期化**: `vault operator init -key-shares=3 -key-threshold=2`
2. **Vault Unseal**: `vault operator unseal` ×2回 (各 Pod で実施)
3. **KV v2 有効化**: `vault secrets enable -path=secret kv-v2`
4. **Kubernetes Auth 設定**: `vault auth enable kubernetes` + ESO 用 policy/role 作成
5. **既存 Secret 格納**: `vault kv put secret/monitoring/alertmanager-slack-webhook url=...` 等

### 13.5 既存 Secret 移行

| ExternalSecret | Vault Path | K8s Secret | Namespace |
|---------------|-----------|-----------|-----------|
| alertmanager-slack-webhook | monitoring/alertmanager-slack-webhook | alertmanager-slack-webhook | monitoring |
| pve-exporter-credentials | monitoring/pve-exporter-credentials | pve-exporter-credentials | monitoring |
| rook-ceph-mon | rook-ceph/rook-ceph-mon | rook-ceph-mon | rook-ceph |
| rook-ceph-config | rook-ceph/rook-ceph-config | rook-ceph-config | rook-ceph |
| rook-csi-rbd-provisioner | rook-ceph/rook-csi-rbd-provisioner | rook-csi-rbd-provisioner | rook-ceph |
| rook-csi-rbd-node | rook-ceph/rook-csi-rbd-node | rook-csi-rbd-node | rook-ceph |
| vault-unseal-keys | vault/unseal-keys | vault-unseal-keys | vault |
| vault-root-token | vault/root-token | vault-root-token | vault |

## 14. 将来の拡張案

- **NIC増設・スイッチ更新**: マルチギガビット対応スイッチへの更新で帯域向上が可能。
- **Ceph OSD追加**: 各ノードにSSDを追加することでCeph容量・性能を拡張可能。
- **GPU Passthrough**: AI用VMにGPUを搭載する場合、PCIパススルーの設定が必要。
