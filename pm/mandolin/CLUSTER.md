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

#### mandolin1でMON作成

```bash
# mandolin1にSSH接続
ssh root@10.1.1.11

# MON作成を試す
pveceph mon create
```

**エラーが出た場合（"unable to get monitor info"、"rados_connect failed"）**: 手動でMONを作成します。

```bash
# mandolin1で実行

# 1. fsidを確認
FSID=$(grep fsid /etc/pve/ceph.conf | awk '{print $3}')
echo "FSID: $FSID"

# 2. MONディレクトリを作成
mkdir -p /var/lib/ceph/mon/ceph-mandolin1
chown ceph:ceph /var/lib/ceph/mon/ceph-mandolin1

# 3. monmapを作成
monmaptool --create --add mandolin1 10.1.1.11 --fsid $FSID /tmp/monmap

# 4. MONを手動で初期化
ceph-mon --mkfs -i mandolin1 --monmap /tmp/monmap --keyring /etc/pve/priv/ceph.mon.keyring

# 5. 所有者を修正
chown -R ceph:ceph /var/lib/ceph/mon/ceph-mandolin1

# 6. MONサービスを有効化して起動
systemctl enable ceph-mon@mandolin1
systemctl start ceph-mon@mandolin1

# 7. 状態確認
systemctl status ceph-mon@mandolin1
sleep 5
ceph -s
```

#### mandolin2, mandolin3でMON作成

mandolin1のMONが起動したら、mandolin2とmandolin3でもMONを作成します。

```bash
# mandolin2で実行
ssh root@10.1.1.12

rm -rf /var/lib/ceph/mon/ceph-mandolin2
FSID=$(grep fsid /etc/pve/ceph.conf | awk '{print $3}')
mkdir -p /var/lib/ceph/mon/ceph-mandolin2
ceph mon getmap -o /tmp/monmap
ceph-mon --mkfs -i mandolin2 --monmap /tmp/monmap --keyring /etc/pve/priv/ceph.mon.keyring
chown -R ceph:ceph /var/lib/ceph/mon/ceph-mandolin2
systemctl enable ceph-mon@mandolin2
systemctl start ceph-mon@mandolin2
systemctl status ceph-mon@mandolin2

# mandolin3で実行
ssh root@10.1.1.13

rm -rf /var/lib/ceph/mon/ceph-mandolin3
FSID=$(grep fsid /etc/pve/ceph.conf | awk '{print $3}')
mkdir -p /var/lib/ceph/mon/ceph-mandolin3
ceph mon getmap -o /tmp/monmap
ceph-mon --mkfs -i mandolin3 --monmap /tmp/monmap --keyring /etc/pve/priv/ceph.mon.keyring
chown -R ceph:ceph /var/lib/ceph/mon/ceph-mandolin3
systemctl enable ceph-mon@mandolin3
systemctl start ceph-mon@mandolin3
systemctl status ceph-mon@mandolin3
```

#### MON状態の確認

```bash
# いずれかのノードで実行
ceph -s
ceph mon stat

# 期待される出力:
# mon: 3 daemons, quorum mandolin1,mandolin2,mandolin3
```

### 2.4 Manager（MGR）の作成（全ノードで実行）

```bash
# mandolin1でMGR作成を試す
ssh root@10.1.1.11
pveceph mgr create
```

**エラーが出た場合（"unable to open file"、keyringエラー）**: 手動でMGRを作成します。

```bash
# mandolin1で実行
mkdir -p /var/lib/ceph/mgr/ceph-mandolin1
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin1
ceph auth get-or-create mgr.mandolin1 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-mandolin1/keyring
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin1/keyring
systemctl enable ceph-mgr@mandolin1
systemctl start ceph-mgr@mandolin1
systemctl status ceph-mgr@mandolin1
sleep 3
ceph -s

# mandolin2で実行
ssh root@10.1.1.12
mkdir -p /var/lib/ceph/mgr/ceph-mandolin2
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin2
ceph auth get-or-create mgr.mandolin2 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-mandolin2/keyring
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin2/keyring
systemctl enable ceph-mgr@mandolin2
systemctl start ceph-mgr@mandolin2
systemctl status ceph-mgr@mandolin2

# mandolin3で実行
ssh root@10.1.1.13
mkdir -p /var/lib/ceph/mgr/ceph-mandolin3
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin3
ceph auth get-or-create mgr.mandolin3 mon 'allow profile mgr' osd 'allow *' mds 'allow *' -o /var/lib/ceph/mgr/ceph-mandolin3/keyring
chown ceph:ceph /var/lib/ceph/mgr/ceph-mandolin3/keyring
systemctl enable ceph-mgr@mandolin3
systemctl start ceph-mgr@mandolin3
systemctl status ceph-mgr@mandolin3
```

#### MGR状態の確認

```bash
# いずれかのノードで実行
ceph -s

# 期待される出力:
# mgr: mandolin1(active, since 10s), standbys: mandolin2, mandolin3
```

### 2.5 OSD（ストレージ）の作成（全ノードで実行）

**重要**: 各ノードの1TB SSDをOSDとして使用します。OSがインストールされている512GB SSDは**使用しません**。

#### ディスクの確認

```bash
# 各ノードで実行
lsblk

# 期待される出力例（SATA SSDの場合）:
# NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
# sda      8:0    0 476.9G  0 disk           ← 512GB SSD（Proxmox OS用、使用しない）
# ├─sda1   8:1    0  1007K  0 part
# ├─sda2   8:2    0     1G  0 part /boot/efi
# └─sda3   8:3    0 475.9G  0 part
# sdb      8:16   0 931.5G  0 disk           ← 1TB SSD（Ceph OSD用、これを使用）

# NVMe SSDの場合は /dev/nvme0n1, /dev/nvme1n1 などになります
# nvme0n1: OS用（使用しない）
# nvme1n1: Ceph OSD用（これを使用）

# ディスクが空かどうか確認
pvesm status

# OSD用ディスクにパーティションがないことを確認
fdisk -l /dev/sdb  # または /dev/nvme1n1
```

#### bootstrap-osd keyringの準備（全ノードで実行）

OSD作成前に、bootstrap-osd keyringを準備します。

```bash
# 全ノード（mandolin1, mandolin2, mandolin3）で実行
mkdir -p /var/lib/ceph/bootstrap-osd
chown ceph:ceph /var/lib/ceph/bootstrap-osd
ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring
```

#### OSDの作成

```bash
# mandolin1で実行
ssh root@10.1.1.11
pveceph osd create /dev/sdb  # または /dev/nvme1n1（環境に合わせて変更）

# mandolin2で実行
ssh root@10.1.1.12
pveceph osd create /dev/sdb  # または /dev/nvme1n1

# mandolin3で実行
ssh root@10.1.1.13
pveceph osd create /dev/sdb  # または /dev/nvme1n1

# 作成時間: 各ノード約2〜5分

# OSD一覧の確認
ceph osd tree

# 期待される出力:
# ID  CLASS  WEIGHT   TYPE NAME           STATUS  REWEIGHT  PRI-AFF
# -1         2.79446  root default
# -3         0.93149      host mandolin1
#  0    ssd  0.93149          osd.0           up   1.00000  1.00000
# -5         0.93149      host mandolin2
#  1    ssd  0.93149          osd.1           up   1.00000  1.00000
# -7         0.93149      host mandolin3
#  2    ssd  0.93149          osd.2           up   1.00000  1.00000
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
# pool 2 'vm-pool' replicated size 3 min_size 2 ... pg_num 128
```

#### PG数の最適化（推奨）

デフォルトのPG数（128）は小規模クラスタには多すぎる場合があります。最適化しましょう。

**PG（Placement Group）とは**: Cephがデータを効率的に管理するための論理的なグループ。多すぎるとメモリ/CPU使用量が増加し、少なすぎるとデータ分散が不均一になります。

**推奨PG数の計算**:
```
PG数 = (OSD数 × 100) / レプリカ数
     = (3 × 100) / 3
     = 100

小規模クラスタでは64が推奨
```

```bash
# PG数を64に最適化
ceph osd pool set vm-pool pg_num 64

# 完了まで1〜2分待つ（PGのマージ処理）
sleep 60
ceph -s

# 期待される出力:
# pgs: 65 active+clean  ← 全てactive+clean
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

### 4.5 pveceph mon create が失敗する

**エラー**: `unable to get monitor info from DNS SRV`, `rados_connect failed`

**原因**: 最初のMON作成時に、まだ接続すべきMONが存在しない

**解決策**: セクション2.3の手動手順を使用してMONを作成

### 4.6 pveceph mgr create が失敗する

**エラー**: `unable to open file`, keyringエラー

**原因**: MGRディレクトリまたはkeyringが正しく作成されない

**解決策**: セクション2.4の手動手順を使用してMGRを作成

### 4.7 pveceph osd create が失敗する

**エラー**: `unable to open file '/var/lib/ceph/bootstrap-osd/ceph.keyring'`

**原因**: bootstrap-osd keyringが存在しない

**解決策**:
```bash
# bootstrap-osd keyringを作成
mkdir -p /var/lib/ceph/bootstrap-osd
chown ceph:ceph /var/lib/ceph/bootstrap-osd
ceph auth get client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring
chown ceph:ceph /var/lib/ceph/bootstrap-osd/ceph.keyring

# 再度OSD作成
pveceph osd create /dev/sdb  # または環境に合わせてデバイス名を変更
```

### 4.8 Cephを完全に再インストールする

設定が複雑になりすぎた場合、Cephを完全にアンインストールして再インストールできます。

```bash
# 全ノード（mandolin1, mandolin2, mandolin3）で実行

# 1. すべてのCephサービスを停止
systemctl stop ceph.target
systemctl stop ceph-mon.target
systemctl stop ceph-mgr.target
systemctl stop ceph-osd.target
systemctl stop ceph-crash

# 2. Cephサービスを無効化
systemctl disable ceph-mon@$(hostname) 2>/dev/null || true
systemctl disable ceph-mgr@$(hostname) 2>/dev/null || true

# 3. Cephパッケージをアンインストール
apt purge -y ceph ceph-common ceph-mds ceph-mgr ceph-mon ceph-osd

# 4. 設定ファイルとデータをすべて削除
rm -rf /var/lib/ceph/*
rm -rf /etc/ceph/*
rm -f /etc/pve/ceph.conf
rm -f /etc/pve/priv/ceph.*

# 5. 再インストール
# セクション2.1から再開
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
