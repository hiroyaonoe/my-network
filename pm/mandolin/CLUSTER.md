# Proxmoxクラスタ・Cephクラスタ セットアップガイド

mandolinクラスタ（3ノード）のProxmoxクラスタとCephストレージクラスタの構築手順。

## 前提条件

- mandolin1, mandolin2, mandolin3のProxmox VEインストールが完了していること
- ネットワーク設定が完了し、各ノードがインターネットにアクセスできること
- 各ノードが相互に通信できること（管理VLAN: 10.1.1.11-13）

## 1. Proxmoxクラスタの作成

### 1.1 クラスタ作成（mandolin1で実行）

```bash
# mandolin1にSSH接続
ssh root@10.1.1.11

# クラスタ作成
pvecm create mandolin

# クラスタ状態の確認
pvecm status

# 期待される出力:
# Cluster information
# ───────────────────
# Name:             mandolin
# Config Version:   1
# Transport:        knet
# Secure auth:      yes
#
# Quorum information
# ──────────────────
# Date:             ...
# Quorum provider:  corosync_votequorum
# Nodes:            1
# Node ID:          0x00000001
# Ring ID:          1.5
# Quorate:          Yes
```

### 1.2 ノードの追加（mandolin2, mandolin3）

#### mandolin2の追加

```bash
# mandolin2にSSH接続
ssh root@10.1.1.12

# クラスタに参加
pvecm add 10.1.1.11

# プロンプトが表示されたら:
# - fingerprint確認: yes を入力
# - rootパスワード入力: mandolin1のrootパスワードを入力

# 期待される出力:
# Please enter superuser (root) password for '10.1.1.11': ****
# Establishing API connection with host '10.1.1.11'
# ...
# successfully added node 'mandolin2' to cluster
```

#### mandolin3の追加

```bash
# mandolin3にSSH接続
ssh root@10.1.1.13

# クラスタに参加
pvecm add 10.1.1.11

# プロンプトが表示されたら:
# - fingerprint確認: yes を入力
# - rootパスワード入力: mandolin1のrootパスワードを入力
```

### 1.3 クラスタ状態の確認

```bash
# いずれかのノードで実行
pvecm status

# 期待される出力:
# Quorum information
# ──────────────────
# ...
# Nodes:            3
# ...
# Quorate:          Yes
#
# Membership information
# ──────────────────────
# Nodeid  Votes  Name
# 0x00000001     1  mandolin1 (local)
# 0x00000002     1  mandolin2
# 0x00000003     1  mandolin3

# ノード一覧の確認
pvecm nodes

# Web UIでも確認
# Datacenter → mandolin → Nodes
# 3台すべてが表示されればOK
```

## 2. Cephクラスタのセットアップ

### 2.1 Cephのインストール（全ノードで実行）

```bash
# mandolin1で実行
ssh root@10.1.1.11
pveceph install --repository no-subscription

# mandolin2で実行
ssh root@10.1.1.12
pveceph install --repository no-subscription

# mandolin3で実行
ssh root@10.1.1.13
pveceph install --repository no-subscription

# インストール時間: 各ノード約5〜10分
```

### 2.2 Ceph設定の初期化（mandolin1のみ）

```bash
# mandolin1で実行
ssh root@10.1.1.11

# Ceph設定の初期化
pveceph init --network 10.1.1.0/24 --cluster-network 10.1.2.0/24

# パラメータ説明:
# --network: Public Network（管理VLAN、クライアント通信用）
# --cluster-network: Cluster Network（Ceph VLAN、OSD間レプリケーション用）
```

### 2.3 Monitor（MON）の作成（全ノードで実行）

```bash
# mandolin1でMON作成
ssh root@10.1.1.11
pveceph mon create

# mandolin2でMON作成
ssh root@10.1.1.12
pveceph mon create

# mandolin3でMON作成
ssh root@10.1.1.13
pveceph mon create

# MON状態の確認
ceph mon stat

# 期待される出力:
# e3: 3 mons at {mandolin1=10.1.1.11:6789/0,mandolin2=10.1.1.12:6789/0,mandolin3=10.1.1.13:6789/0}
```

### 2.4 Manager（MGR）の作成（全ノードで実行）

```bash
# mandolin1でMGR作成
ssh root@10.1.1.11
pveceph mgr create

# mandolin2でMGR作成
ssh root@10.1.1.12
pveceph mgr create

# mandolin3でMGR作成
ssh root@10.1.1.13
pveceph mgr create

# MGR状態の確認
ceph mgr stat

# 期待される出力:
# {
#     "epoch": ...,
#     "available": true,
#     "active_name": "mandolin1",
#     "standby": [
#         {
#             "name": "mandolin2"
#         },
#         {
#             "name": "mandolin3"
#         }
#     ]
# }
```

### 2.5 OSD（ストレージ）の作成（全ノードで実行）

**重要**: 各ノードの1TB SSDをOSDとして使用します。OSがインストールされている512GB SSDは**使用しません**。

#### ディスクの確認

```bash
# 各ノードで実行
lsblk

# 期待される出力例:
# NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
# sda      8:0    0 476.9G  0 disk           ← 512GB SSD（Proxmox OS用、使用しない）
# ├─sda1   8:1    0  1007K  0 part
# ├─sda2   8:2    0     1G  0 part /boot/efi
# └─sda3   8:3    0 475.9G  0 part
# sdb      8:16   0 931.5G  0 disk           ← 1TB SSD（Ceph OSD用、これを使用）

# ディスクが空かどうか確認
pvesm status

# sdb（1TB SSD）にパーティションがないことを確認
fdisk -l /dev/sdb
```

#### OSDの作成

```bash
# mandolin1で実行
ssh root@10.1.1.11
pveceph osd create /dev/sdb

# mandolin2で実行
ssh root@10.1.1.12
pveceph osd create /dev/sdb

# mandolin3で実行
ssh root@10.1.1.13
pveceph osd create /dev/sdb

# 作成時間: 各ノード約2〜5分

# OSD一覧の確認
ceph osd tree

# 期待される出力:
# ID  CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
# -1         2.73196  root default
# -3         0.91065      host mandolin1
#  0    hdd  0.91065          osd.0           up   1.00000  1.00000
# -5         0.91065      host mandolin2
#  1    hdd  0.91065          osd.1           up   1.00000  1.00000
# -7         0.91065      host mandolin3
#  2    hdd  0.91065          osd.2           up   1.00000  1.00000
```

### 2.6 ストレージプールの作成

```bash
# いずれかのノードで実行
ssh root@10.1.1.11

# プール作成
pveceph pool create vm-pool --add_storages

# パラメータ:
# - プール名: vm-pool
# - --add_storages: Proxmoxストレージとして自動登録

# デフォルト設定:
# - size (replica): 3
# - min_size: 2
# - pg_num: 128（自動計算）

# プール状態の確認
ceph osd pool ls detail

# 期待される出力:
# pool 1 'vm-pool' replicated size 3 min_size 2 ...
```

### 2.7 Proxmoxストレージの確認

```bash
# ストレージ一覧の確認
pvesm status

# 期待される出力:
# Name             Type     Status           Total            Used       Available        %
# local            dir      active      475934208        15728640       435846568    3.31%
# local-lvm        lvmthin  active      475934208        15728640       435846568    3.31%
# vm-pool          rbd      active       976773120               0       976773120    0.00%
#                                       ↑ 約931GB（1TB / 3 replica）

# Web UIでも確認
# Datacenter → Storage
# vm-pool が表示され、Nodes列に mandolin1, mandolin2, mandolin3 すべてが表示されればOK
```

## 3. Ceph状態の確認

### 3.1 クラスタヘルス確認

```bash
# Ceph全体の状態確認
ceph status

# 期待される出力:
#   cluster:
#     id:     xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
#     health: HEALTH_OK
#
#   services:
#     mon: 3 daemons, quorum mandolin1,mandolin2,mandolin3 (age 10m)
#     mgr: mandolin1(active, since 5m), standbys: mandolin2, mandolin3
#     osd: 3 osds: 3 up (since 3m), 3 in (since 3m)
#
#   data:
#     pools:   1 pools, 128 pgs
#     objects: 0 objects, 0 B
#     usage:   60 MiB used, 2.7 TiB / 2.7 TiB avail
#     pgs:     128 active+clean

# ヘルスチェック
ceph health detail

# HEALTH_OK と表示されればOK
```

### 3.2 OSD使用状況の確認

```bash
# OSD容量の確認
ceph osd df

# 期待される出力:
# ID  CLASS  WEIGHT   REWEIGHT  SIZE     RAW USE  DATA     OMAP  META   AVAIL    %USE  VAR   PGS  STATUS
#  0    hdd  0.91065   1.00000  931 GiB   20 MiB  0 B      0 B   20 MiB  931 GiB  0.00  1.00  128      up
#  1    hdd  0.91065   1.00000  931 GiB   20 MiB  0 B      0 B   20 MiB  931 GiB  0.00  1.00  128      up
#  2    hdd  0.91065   1.00000  931 GiB   20 MiB  0 B      0 B   20 MiB  931 GiB  0.00  1.00  128      up
#                      TOTAL     2.7 TiB   60 MiB  0 B      0 B   60 MiB  2.7 TiB  0.00
# MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

### 3.3 ネットワーク確認

```bash
# Public Network確認
ceph config get mon public_network
# 出力: 10.1.1.0/24

# Cluster Network確認
ceph config get osd cluster_network
# 出力: 10.1.2.0/24

# Mon接続確認
ceph mon dump

# 期待される出力:
# dumped monmap epoch 3
# epoch 3
# ...
# 0: [v2:10.1.1.11:3300/0,v1:10.1.1.11:6789/0] mon.mandolin1
# 1: [v2:10.1.1.12:3300/0,v1:10.1.1.12:6789/0] mon.mandolin2
# 2: [v2:10.1.1.13:3300/0,v1:10.1.1.13:6789/0] mon.mandolin3
```

## 4. トラブルシューティング

### 4.1 クラスタがQuorateにならない

```bash
# クォーラム状態の確認
pvecm status

# expected votes と total votes を確認
# 3ノード構成の場合、expected votes: 3

# 各ノードのCorosync状態確認
systemctl status corosync

# Corosync再起動（全ノードで実行）
systemctl restart corosync
systemctl restart pve-cluster
```

### 4.2 Cephが HEALTH_WARN になる

```bash
# 警告の詳細確認
ceph health detail

# よくある警告と対処法:

# 1. "too few PGs per OSD"
# → pg_num を増やす
ceph osd pool set vm-pool pg_num 256

# 2. "clock skew detected"
# → 各ノードの時刻を同期
timedatectl set-ntp true
systemctl restart chronyd

# 3. "1 pool(s) do not have an application enabled"
# → プールにアプリケーションタグを設定
ceph osd pool application enable vm-pool rbd
```

### 4.3 OSD作成に失敗

```bash
# ディスクが使用中か確認
lsblk /dev/sdb

# パーティションテーブルを削除（注意: データが消えます）
wipefs -a /dev/sdb
sgdisk --zap-all /dev/sdb

# 再度OSD作成
pveceph osd create /dev/sdb
```

### 4.4 Web UIでクラスタノードが見えない

```bash
# 各ノードでpve-cluster再起動
systemctl restart pve-cluster

# pmxcfs状態の確認
pvecm status
pmxcfs status

# 設定ファイルの確認
cat /etc/pve/corosync.conf
```

## 5. Web UIでの操作

### 5.1 クラスタ状態の確認

1. https://10.1.1.11:8006 (またはmandolin2, mandolin3のいずれか) にアクセス
2. Datacenter → mandolin をクリック
3. Summary タブで全体の状態を確認
4. Nodes タブで各ノードの状態を確認

### 5.2 Ceph状態の確認

1. Datacenter → mandolin → Ceph をクリック
2. 各タブで状態確認:
   - **Summary**: クラスタヘルス、容量使用状況
   - **Monitors**: MON一覧
   - **OSD**: OSD一覧と状態
   - **Pools**: プール一覧
   - **Configuration**: Ceph設定

### 5.3 VMストレージとしての使用

VM作成時:
1. Create VM → Hard Disk タブ
2. Storage: **vm-pool** を選択
3. Disk size: 必要なサイズを入力

これで、Cephストレージ上にVMディスクが作成されます。

## 6. 次のステップ

クラスタセットアップ完了後:

1. **VMの作成** → `docs/design.md` セクション7.2参照
2. **バックアップの設定** → Datacenter → Backup
3. **HAの設定（オプション）** → Datacenter → HA
4. **モニタリング** → 各種メトリクスの確認

## 参考情報

- Proxmox VE公式ドキュメント: https://pve.proxmox.com/pve-docs/
- Proxmox Ceph Server: https://pve.proxmox.com/pve-docs/chapter-pveceph.html
- Ceph公式ドキュメント: https://docs.ceph.com/
