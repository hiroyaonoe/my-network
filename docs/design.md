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
    |  LAN: 192.168.0.0/20
    |  DHCP: 192.168.0.0/24 (Wi-Fi等)
    |  DNS/NTP提供
    |  static route: 10.0.0.0/8     → 192.168.1.1
    |                172.16.0.0/12  → 192.168.1.1
    |
    | (192.168.0.1 --- 192.168.1.1)
    |
[Catalyst 3560-CX] ローカルルータ / L3スイッチ
    |  default route → 192.168.0.1
    |
    |-- Gi0/1: WAN側ルータ (routed port)
    |-- Gi0/2: 空き
    |-- Gi0/3-4: mandolin1 (LACP Po1, trunk)
    |-- Gi0/5-6: mandolin2 (LACP Po2, trunk)
    |-- Gi0/7-8: mandolin3 (LACP Po3, trunk)
    |
    +-- VLAN 10  [mgmt]       172.16.0.0/24  Proxmox管理
    +-- VLAN 20  [ceph]       172.16.1.0/24  Cephレプリケーション
    +-- VLAN 100-227 [vm-*]   10.0.0.0/20内の/27を連番で割当 (最大128個)
```

### 3.2 VLAN設計

#### インフラVLAN (固定)

| VLAN ID | 名称 | サブネット | 用途 |
|---------|------|-----------|------|
| 10 | mgmt | 172.16.0.0/24 | Proxmox管理、Corosync、Ceph Public |
| 20 | ceph | 172.16.1.0/24 | Ceph Cluster (レプリケーション専用) |

#### VM用VLAN (100-227)

10.0.0.0/20 の中で /27 (30ホスト) を隙間なく連番で割り当てる。最大128個。

採番ルール: **VLAN (100 + N)** → **10.0.(N/8).(N%8\*32)/27** (N = 0〜127)

| N | VLAN ID | サブネット | ゲートウェイ | ホスト範囲 |
|---|---------|-----------|-------------|-----------|
| 0 | 100 | 10.0.0.0/27 | 10.0.0.1 | .2 〜 .30 |
| 1 | 101 | 10.0.0.32/27 | 10.0.0.33 | .34 〜 .62 |
| 2 | 102 | 10.0.0.64/27 | 10.0.0.65 | .66 〜 .94 |
| 3 | 103 | 10.0.0.96/27 | 10.0.0.97 | .98 〜 .126 |
| 4 | 104 | 10.0.0.128/27 | 10.0.0.129 | .130 〜 .158 |
| ... | ... | ... | ... | ... |
| 8 | 108 | 10.0.1.0/27 | 10.0.1.1 | .2 〜 .30 |
| ... | ... | ... | ... | ... |
| 127 | 227 | 10.0.15.224/27 | 10.0.15.225 | .226 〜 .254 |

初期割当:

| VLAN ID | N | 名称 | サブネット | 用途 |
|---------|---|------|-----------|------|
| 100 | 0 | vm-bastion | 10.0.0.0/27 | 踏み台 |
| 101 | 1 | vm-lab | 10.0.0.32/27 | 検証環境 |
| 102 | 2 | vm-k8s | 10.0.0.64/27 | k8sクラスタ |
| 103 | 3 | vm-ai | 10.0.0.96/27 | AI動作用 |

> trunk の allowed vlan および Proxmox の bridge-vids は 100-227 の範囲指定としている。
> VLAN追加時はCatalystのVLAN定義とSVIの追加のみで利用開始できる。

### 3.3 IPアドレス設計

#### WAN側ネットワーク (192.168.0.0/20)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| WAN側ルータ (LAN側) | 192.168.0.1 | |
| DHCP範囲 | 192.168.0.100〜192.168.0.254 (想定) | Wi-Fi等 |
| ローカルルータ (Gi0/1) | 192.168.1.1 | 静的割り当て |

#### VLAN 10: 管理ネットワーク (172.16.0.0/24)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| ローカルルータ (SVI) | 172.16.0.1 | ゲートウェイ |
| mandolin1 | 172.16.0.11 | Proxmox管理IP |
| mandolin2 | 172.16.0.12 | Proxmox管理IP |
| mandolin3 | 172.16.0.13 | Proxmox管理IP |

Proxmox Web UI: `https://172.16.0.1x:8006`

#### VLAN 20: Cephネットワーク (172.16.1.0/24)

| 機器 | IPアドレス | 備考 |
|------|-----------|------|
| mandolin1 | 172.16.1.11 | Ceph Cluster Network |
| mandolin2 | 172.16.1.12 | Ceph Cluster Network |
| mandolin3 | 172.16.1.13 | Ceph Cluster Network |

> ゲートウェイ不要。ノード間のOSDレプリケーション通信のみ。

#### VLAN 100〜227: VMネットワーク (10.0.0.0/20)

採番ルールに従い、/27を連番で詰めて割り当てる（3.2節参照）。

> 各VMにはサブネット内で静的にIPを割り当てる。各/27サブネットで使用可能なホストアドレスは30個。

#### k8sクラスタ内部ネットワーク (10.0.0.0/8 の /20除外範囲)

| 用途 | CIDR | 備考 |
|------|------|------|
| Pod Network | 10.1.0.0/16 | CNI (Calico, Cilium等) が使用 |
| Service Network | 10.2.0.0/16 | Kubernetes Service CIDR |
| LoadBalancer | 10.3.0.0/24 | MetalLB等で使用 |

> k8sクラスタ内部ネットワークの詳細はクラスタ構築時に決定する。

### 3.4 ルーティング設計

#### WAN側ルータ

```
default → WAN (インターネット)
10.0.0.0/8     → 192.168.1.1 (ローカルルータ)
172.16.0.0/12  → 192.168.1.1 (ローカルルータ)
```

#### ローカルルータ (Catalyst 3560-CX)

```
default         → 192.168.0.1 (WAN側ルータ)
172.16.0.0/24   → VLAN 10 (directly connected)
172.16.1.0/24   → VLAN 20 (directly connected)
10.0.x.x/27     → VLAN 100-227 (SVIが存在するVLANのみ、directly connected)
```

> SVI間ルーティングにより、SVIが定義された全VLANは相互に通信可能。
> VM用VLANの追加時はSVIを作成するだけでconnected routeが自動生成される。
> 必要に応じてACLでVLAN間のアクセス制御を実施する。

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
| mandolin1 | 172.16.0.11 | 172.16.1.11 |
| mandolin2 | 172.16.0.12 | 172.16.1.12 |
| mandolin3 | 172.16.0.13 | 172.16.1.13 |

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
    address 172.16.0.1x/24       # 各ノードに応じて .11/.12/.13
    gateway 172.16.0.1
    bridge-ports bond0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 20 100-227

# Cephネットワーク (VLAN 20)
auto vmbr0.20
iface vmbr0.20 inet static
    address 172.16.1.1x/24       # 各ノードに応じて .11/.12/.13
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
| Public Network | 172.16.0.0/24 (VLAN 10) |
| Cluster Network | 172.16.1.0/24 (VLAN 20) |

> Public Network: クライアント (VM) → OSD のI/O通信に使用。管理ネットワークと共用。
> Cluster Network: OSD間レプリケーション/リカバリ通信専用。管理トラフィックと分離。

#### ストレージプール

| プール名 | size (replica) | min_size | 用途 |
|----------|---------------|----------|------|
| vm-pool | 3 | 2 | VMディスク |

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

ip name-server 192.168.0.1
ntp server 192.168.0.1
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
 ip address 172.16.0.1 255.255.255.0
 no shutdown

interface Vlan20
 ip address 172.16.1.1 255.255.255.0
 no shutdown

! 初期VM用SVI (採番ルール: VLAN 100+N → 10.0.(N/8).(N%8*32)/27)
interface Vlan100
 ip address 10.0.0.1 255.255.255.224
 no shutdown

interface Vlan101
 ip address 10.0.0.33 255.255.255.224
 no shutdown

interface Vlan102
 ip address 10.0.0.65 255.255.255.224
 no shutdown

interface Vlan103
 ip address 10.0.0.97 255.255.255.224
 no shutdown
```

### 5.4 物理ポート設定

```
! WAN側ルータ接続 (Routed Port)
interface GigabitEthernet0/1
 no switchport
 ip address 192.168.1.1 255.255.240.0
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
ip route 0.0.0.0 0.0.0.0 192.168.0.1
```

> Connected routeにより各SVI配下のサブネットは自動的にルーティングされる。
> WAN側ルータ側で10.0.0.0/8, 172.16.0.0/12 → 192.168.1.1の静的ルートを設定する。

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
    ip address 10.0.<N/8>.<N%8*32 + 1> 255.255.255.224
    no shutdown
   ```
3. VMを作成（→ 7.2参照）

> Proxmoxノード側やtrunkの設定変更は不要。

### 7.2 VMの作成とVLAN配置

vmbr0は「VLAN-aware bridge」として構成されているため、VM作成時にVLANタグを指定するだけで任意のVLANに配置できる。**VM内部ではVLAN設定は不要。**

#### Web UIでの設定

1. Proxmox Web UI (`https://172.16.0.1x:8006`) にログイン
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
| 100 | 踏み台 | 10.0.0.0/27 | 10.0.0.2 〜 10.0.0.30 |
| 101 | 検証環境 | 10.0.0.32/27 | 10.0.0.34 〜 10.0.0.62 |
| 102 | k8sクラスタ | 10.0.0.64/27 | 10.0.0.66 〜 10.0.0.94 |
| 103 | AI動作用 | 10.0.0.96/27 | 10.0.0.98 〜 10.0.0.126 |

> VM内のOS設定では、通常のイーサネットとして静的IPまたはDHCP（別途構築する場合）を設定する。
> ゲートウェイは各VLANのSVI (10.0.0.1, 10.0.0.33, 10.0.0.65, 10.0.0.97...) を指定。

## 8. 将来の拡張案

- **NIC増設・スイッチ更新**: マルチギガビット対応スイッチへの更新で帯域向上が可能。
- **Ceph OSD追加**: 各ノードにSSDを追加することでCeph容量・性能を拡張可能。
- **GPU Passthrough**: AI用VMにGPUを搭載する場合、PCIパススルーの設定が必要。
