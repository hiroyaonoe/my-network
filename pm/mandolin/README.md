# Proxmox VE クラスタ: mandolin

## クラスタ構成

- **クラスタ名**: mandolin
- **ノード数**: 3
- **ノード**: mandolin1 (10.1.1.11), mandolin2 (10.1.1.12), mandolin3 (10.1.1.13)

## ドキュメント

- **[INSTALL.md](INSTALL.md)** - Proxmox VEインストール手順（3ノード）
- **README.md** (このファイル) - 設定ファイルの適用方法

## ディレクトリ構造

```
pm/mandolin/
├── mandolin1/etc/
│   ├── network/interfaces  # mandolin1のネットワーク設定
│   └── resolv.conf         # mandolin1のDNS設定
├── mandolin2/etc/
│   ├── network/interfaces  # mandolin2のネットワーク設定
│   └── resolv.conf         # mandolin2のDNS設定
├── mandolin3/etc/
│   ├── network/interfaces  # mandolin3のネットワーク設定
│   └── resolv.conf         # mandolin3のDNS設定
├── etc/ceph/ceph.conf      # Ceph設定 (全ノード共通)
└── README.md
```

## 適用方法

### ネットワーク設定

```bash
# 各ノードへコピー
scp -r mandolin1/etc root@10.1.1.11:/
scp -r mandolin2/etc root@10.1.1.12:/
scp -r mandolin3/etc root@10.1.1.13:/

# 各ノードで実行 (慎重に！SSH接続が切れる可能性あり)
ssh root@10.1.1.11 'ifreload -a'
ssh root@10.1.1.12 'ifreload -a'
ssh root@10.1.1.13 'ifreload -a'
```

### Ceph設定

```bash
# 全ノードにCeph設定をコピー
for node in 10.1.1.{11,12,13}; do
  scp -r etc/ceph root@$node:/etc/
done
```

Proxmox Web UIまたはCLIで初期セットアップ:

```bash
# Cephのインストール (各ノードで実行)
pveceph install

# モニター作成 (各ノードで実行)
pveceph mon create

# OSD作成 (各ノードで1TB SSDを指定)
pveceph osd create /dev/sdX
```

詳細は `docs/design.md` のセクション4.3参照。
