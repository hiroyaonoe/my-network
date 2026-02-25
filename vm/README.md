# VM設定ディレクトリ

このディレクトリには各仮想マシンの設定を保存します。

## ディレクトリ構造

```
vm/
├── <vm-name>/
│   ├── vm-config.json      # VM設定（CPU, RAM, VLAN等）
│   ├── cloud-init.yaml     # cloud-init設定
│   └── README.md           # VMの用途説明
```

## VM作成時のVLANタグ指定

VMはProxmox側でVLANタグを指定することで、各VLANに配置されます。

### 初期VLAN割当

| VLAN Tag | 用途 | サブネット |
|----------|------|-----------|
| 100 | 踏み台 | 10.2.0.0/27 |
| 101 | 検証環境 | 10.2.0.32/27 |
| 102 | k8sクラスタ | 10.2.0.64/27 |
| 103 | AI動作用 | 10.2.0.96/27 |

### VM作成例

```bash
# Web UIの場合: Hardware → Network Device → VLAN Tag に指定
# CLIの場合:
qm create 100 --name bastion-01 --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0,tag=100
```

詳細は `docs/design.md` のセクション7.2参照。
