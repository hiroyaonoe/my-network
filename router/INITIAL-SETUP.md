# Catalyst 初期セットアップ手順

新品のCatalystまたは設定がリセットされた状態からSSH接続できるようにする手順。

## 必要なもの

- コンソールケーブル (RJ-45 to USB)
- PC (ターミナルソフトウェア: screen, minicom, Tera Term等)
- LANケーブル (PC ↔ Catalyst Gi0/2用)

## 手順

### 1. コンソール接続

```bash
# macOS/Linux
screen /dev/tty.usbserial 115200

# または minicom
minicom -D /dev/tty.usbserial -b 115200

# Windows: Tera Term等で COM ポート, 115200 baud
```

### 2. 初期設定の投入

```bash
# Catalystのプロンプトが表示されたら
enable
configure terminal

# initial-setup.conf の内容をコピペ
# ただし <strong-password> の部分は実際のパスワードに置き換える

# 例:
username admin privilege 15 secret MySecurePass123!
enable secret MySecurePass123!

# SSH鍵生成時に "Do you really want to replace them?" と聞かれたら yes
```

### 3. PCの準備

Catalystの Gi0/2 に接続するPCのネットワーク設定:

```
IPアドレス: 172.16.0.100
サブネットマスク: 255.255.255.0 (または /24)
ゲートウェイ: 172.16.0.1
```

### 4. LAN接続

```
PC <--LANケーブル--> Catalyst Gi0/2
```

### 5. SSH接続確認

```bash
# PC から Catalyst へ SSH接続
ssh admin@172.16.0.1

# パスワード入力後、接続できることを確認
```

### 6. 残りの設定を投入

SSH接続できたら、`catalyst-3560cx.conf` の残りの設定を投入:

```bash
# SSH接続中に実行
enable
configure terminal

# catalyst-3560cx.conf から以下のセクションをコピペ:
# - VLAN定義 (vlan 20, 100-103)
# - SVI (Vlan20, Vlan100-103)
# - 物理ポート設定 (Gi0/1, Gi0/3-8)
# - Port-Channel設定
# - ルーティング (ip route)

end
copy running-config startup-config
```

### 7. Gi0/2 を無効化

全設定完了後、一時的な管理ポートを無効化:

```bash
configure terminal
interface GigabitEthernet0/2
 shutdown
exit
end
copy running-config startup-config
```

## トラブルシューティング

### SSH接続できない

```bash
# Catalyst側で確認
show ip interface brief
# → Vlan10が up/up になっているか

show ip ssh
# → SSH Enabled, version 2.0 になっているか

show crypto key mypubkey rsa
# → RSA鍵が生成されているか

# PC側から ping確認
ping 172.16.0.1
```

### パスワードを忘れた場合

コンソールケーブルで接続し、パスワードリカバリ手順を実施。
詳細は Cisco公式ドキュメント参照。

## 次のステップ

SSH接続確認後:
1. `catalyst-3560cx.conf` の残り設定を投入
2. 物理マシン(mandolin1-3)を接続
3. Port-Channelの状態確認: `show etherchannel summary`
4. ルーティング確認: `show ip route`
