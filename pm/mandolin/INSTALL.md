# Proxmox VE インストール手順

mandolin クラスタ（3ノード）のインストールガイド。

## 事前準備

### 必要なもの

| 項目 | 数量 | 備考 |
|------|------|------|
| Proxmox VE ISOイメージ | 1 | [公式サイト](https://www.proxmox.com/en/downloads)からダウンロード |
| USBメモリ（8GB以上） | 1 | インストールメディア作成用 |
| 物理マシン | 3 | mandolin1, mandolin2, mandolin3 |
| LANケーブル | 6 | 各マシン2本（bonding用） |
| モニター・キーボード | 1 | インストール時のみ使用 |

### インストールメディア作成

```bash
# macOS/Linux
dd if=proxmox-ve_8.x.iso of=/dev/sdX bs=1M status=progress

# Windows: Rufus, Etcher等を使用
```

### ネットワーク接続

インストール前に物理配線を確認：

```
[Catalyst Gi0/3-4] <--- 2本 ---> [mandolin1 NIC1, NIC2]
[Catalyst Gi0/5-6] <--- 2本 ---> [mandolin2 NIC1, NIC2]
[Catalyst Gi0/7-8] <--- 2本 ---> [mandolin3 NIC1, NIC2]
```

> インストール時は1本だけ接続でもOK。bonding設定は後で追加。

## インストール手順

### 1. 起動

1. USBメディアを挿入
2. BIOS/UEFIでUSBブートを選択
3. Proxmox VE起動画面で **Install Proxmox VE (Graphical)** を選択

### 2. インストーラー設定

#### 2.1 EULA

- **I agree** をクリック

#### 2.2 Target Harddisk

| 設定項目 | 値 |
|---------|-----|
| Target Harddisk | **512GB SSD** を選択 |
| Filesystem | ZFS (RAID0) または ext4 |

> **重要**: 1TB SSD は選択しない（Ceph OSD用に残す）

**Options** をクリックして詳細設定（任意）:
- **swapsize**: 4GB（RAM 16GBの場合、デフォルトでOK）
- **maxroot**: 50GB（残りはデータ用）

#### 2.3 Location and Time Zone

| 設定項目 | 値 |
|---------|-----|
| Country | Japan |
| Time zone | Asia/Tokyo |
| Keyboard Layout | Japanese |

#### 2.4 Administration Password and Email

| 設定項目 | 値 | 備考 |
|---------|-----|------|
| Password | (強力なパスワード) | rootユーザーのパスワード |
| Confirm | (同じパスワード) | |
| Email | admin@internal.onoe.dev | アラート通知先（任意） |

#### 2.5 Management Network Configuration

**mandolin1の場合:**

| 設定項目 | 値 | 備考 |
|---------|-----|------|
| Management Interface | eno1（最初のNIC） | インストール時は1本でOK |
| Hostname (FQDN) | mandolin1.internal.onoe.dev | |
| IP Address (CIDR) | 172.16.0.11/24 | |
| Gateway | 172.16.0.1 | Catalyst（設定済みの場合） |
| DNS Server | 192.168.0.1 | WAN側ルータ |

**mandolin2の場合:**
- Hostname: `mandolin2.internal.onoe.dev`
- IP Address: `172.16.0.12/24`

**mandolin3の場合:**
- Hostname: `mandolin3.internal.onoe.dev`
- IP Address: `172.16.0.13/24`

> Gateway/DNSは全ノード共通。

#### 2.6 確認とインストール

設定内容を確認して **Install** をクリック。

インストール所要時間: 約5〜10分

### 3. インストール完了

再起動後、以下が表示されます:

```
Welcome to the Proxmox Virtual Environment. Please use your web browser to
configure this server - connect to:

  https://172.16.0.11:8006/

Login with username 'root' and your root password.
```

## 初期設定

### 4. Web UIアクセス

ブラウザで `https://172.16.0.11:8006` にアクセス:

1. **証明書警告**: 「詳細設定」→「アクセスする」（自己署名証明書のため）
2. **ログイン**:
   - Username: `root`
   - Password: インストール時に設定したパスワード
   - Realm: `Linux PAM standard authentication`

### 5. サブスクリプション警告の無効化（任意）

ログイン後に「No valid subscription」警告が出る場合:

```bash
# SSH接続して実行
ssh root@172.16.0.11

# リポジトリ変更（Enterprise → No-Subscription）
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# 更新
apt update && apt upgrade -y
```

### 6. ネットワーク設定の適用

**重要**: この手順でbonding + VLAN設定を適用します。

```bash
# SSH接続
ssh root@172.16.0.11

# NIC名を確認
ip link show
# eno1, eno2 等が表示される。以下の設定ファイルでNIC名を確認・修正。

# 設定ファイルをバックアップ
cp /etc/network/interfaces /etc/network/interfaces.bak

# ローカルPCから設定ファイルをコピー
# （リポジトリのpm/mandolin/mandolin1/etc/network/interfaces）
exit

# ローカルPCで実行
scp -r pm/mandolin/mandolin1/etc root@172.16.0.11:/

# 再度SSH接続
ssh root@172.16.0.11

# 設定確認
cat /etc/network/interfaces

# NIC名が正しいか確認（eno1, eno2 → 実際のNIC名に修正）
vi /etc/network/interfaces

# ネットワーク再起動（SSH接続が切れる可能性あり）
# 念のためコンソールアクセスを確保してから実行
ifreload -a

# または安全に再起動
reboot
```

### 7. 動作確認

再起動後、SSH再接続:

```bash
ssh root@172.16.0.11

# インターフェース確認
ip addr show

# bond0が作成されているか
ip link show bond0

# vmbr0の確認
ip link show vmbr0

# Cephネットワーク確認
ip addr show vmbr0.20
# 172.16.1.11 が割り当てられているか

# ルーティング確認
ip route
```

## 2台目・3台目のインストール

mandolin2, mandolin3 も同様の手順で：

1. 同じISOメディアでインストール
2. IP: `172.16.0.12`, `172.16.0.13`
3. Hostname: `mandolin2.internal.onoe.dev`, `mandolin3.internal.onoe.dev`
4. ネットワーク設定適用: `mandolin2/etc`, `mandolin3/etc`

## 次のステップ

3台のインストール完了後:

1. **Proxmoxクラスタ作成** → 別途手順書参照
2. **Cephセットアップ** → `docs/design.md` セクション4.3参照
3. **VM作成** → `docs/design.md` セクション7.2参照

## トラブルシューティング

### ネットワーク設定適用後にSSH接続できない

- **原因**: bonding設定が正しくない、またはCatalyst側のPort-Channelが未設定
- **対処**:
  1. コンソールで接続
  2. `ip link show bond0` でbond0の状態確認
  3. Catalyst側で `show etherchannel summary` 確認
  4. 必要に応じて設定を `/etc/network/interfaces.bak` から復旧

### Web UIにアクセスできない

```bash
# Proxmox Web UIサービス確認
systemctl status pveproxy

# 再起動
systemctl restart pveproxy
```

### インストール時にディスクが認識されない

- BIOS/UEFIでAHCIモードになっているか確認
- ディスクの接続を確認
