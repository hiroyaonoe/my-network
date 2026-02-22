# Catalyst WS-C3560CX-8PC-S 設定

## ファイル

- `catalyst-3560cx.conf` - Catalyst IOS設定（完全版）

## 設定適用方法

### 方法1: SSH経由でコピペ（推奨）

```bash
# 1. Catalystに既存IPでSSH接続
ssh admin@<current-ip>

# 2. 設定モードに入る
enable
configure terminal

# 3. ローカルで設定ファイルを開き、セクションごとにコピペ
# （VLAN定義、SVI、ポート設定、Port-Channel、ルーティング）

# 4. 設定を保存
end
copy running-config startup-config
```

### 方法2: コンソールケーブル経由

```bash
# 1. コンソールケーブルで接続（115200 baud）
screen /dev/tty.usbserial 115200
# または
minicom -D /dev/tty.usbserial

# 2. 設定モードに入る
enable
configure terminal

# 3. 設定ファイルの内容をコピペ
# （! で始まるコメント行も含めてOK）

# 4. 設定を保存
end
copy running-config startup-config
```

### 方法3: TFTPサーバー経由

```bash
# 事前準備: TFTPサーバーに catalyst-3560cx.conf を配置

# Catalyst側で実行
enable
copy tftp://192.168.0.100/catalyst-3560cx.conf running-config

# 設定を確認
show running-config

# 問題なければ保存
copy running-config startup-config
```

## 初期セットアップ時の注意

### IPアドレスが未設定の場合

1. コンソールケーブルで接続
2. 管理用IPを先に設定:
   ```
   enable
   configure terminal
   interface Vlan10
    ip address 172.16.0.1 255.255.255.0
    no shutdown
   exit

   # ポート1つをVLAN 10に割り当て（一時的）
   interface GigabitEthernet0/2
    switchport mode access
    switchport access vlan 10
    no shutdown
   exit
   end
   ```
3. PCをGi0/2に接続し、172.16.0.0/24のIPを設定
4. SSH接続可能になったら残りの設定を投入

### ネットワーク変更時の注意

**重要**: ポート設定やVLAN設定を変更すると通信が切れる可能性があります。

- **初回セットアップ**: コンソールケーブル必須
- **変更時**: バックアップ用の接続経路を確保するか、コンソールアクセスを用意

## 設定の確認

```bash
# 現在の設定を表示
show running-config

# VLAN情報
show vlan brief

# Port-Channel状態
show etherchannel summary

# インターフェース状態
show ip interface brief

# ルーティングテーブル
show ip route
```

## 部分的な設定変更

### 新しいVM用VLANを追加

```bash
enable
configure terminal

# VLAN定義
vlan 104
 name vm-newservice
exit

# SVI (ゲートウェイ) 作成
# 採番ルール: VLAN (100+N) → 10.0.(N/8).(N%8*32)/27
# VLAN 104 = N=4 → 10.0.0.128/27, GW=10.0.0.129
interface Vlan104
 ip address 10.0.0.129 255.255.255.224
 no shutdown
exit

end
copy running-config startup-config
```

> trunk の allowed vlan は既に `100-227` で範囲指定済みのため、
> Port-Channel や物理ポートの設定変更は不要。

## トラブルシューティング

### 設定投入後に接続が切れた場合

1. コンソールケーブルで接続
2. 設定を確認: `show running-config`
3. 問題箇所を修正または前の設定に戻す:
   ```
   enable
   configure replace flash:startup-config
   ```

### 設定を完全にリセット

```bash
enable
write erase
reload
```

**警告**: これは全設定を消去します。実行前に必ずバックアップを取得してください。
