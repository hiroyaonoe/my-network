# K8s クラスタ (Talos Linux + Cilium)

VLAN 102 (vm-k8s, 10.2.0.64/27) 上の HA Kubernetes クラスタ。

## クラスタ情報

| 項目 | 値 |
|------|-----|
| OS | Talos Linux |
| K8s バージョン | 1.32 |
| CNI | Cilium (Native Routing + L2 Announcement) |
| API Server VIP | 10.2.0.94 |
| API Endpoint | `https://10.2.0.94:6443` |
| Pod CIDR | 10.3.0.0/16 |
| Service CIDR | 10.4.0.0/16 |
| LB VIP Pool | 10.5.0.0/24 |

## ノード一覧

| VM名 | VM ID | IP | Proxmoxノード | 役割 | vCPU | RAM | Disk |
|-------|-------|-----|---------------|------|------|-----|------|
| k8s-cp-01 | 102001 | 10.2.0.66 | mandolin1 | control-plane | 2 | 4GB | 32GB |
| k8s-cp-02 | 102002 | 10.2.0.67 | mandolin2 | control-plane | 2 | 4GB | 32GB |
| k8s-cp-03 | 102003 | 10.2.0.68 | mandolin3 | control-plane | 2 | 4GB | 32GB |
| k8s-worker-01 | 102004 | 10.2.0.69 | mandolin1 | worker | 4 | 8GB | 64GB |
| k8s-worker-02 | 102005 | 10.2.0.70 | mandolin2 | worker | 4 | 8GB | 64GB |
| k8s-worker-03 | 102006 | 10.2.0.71 | mandolin3 | worker | 4 | 8GB | 64GB |

## セットアップ手順

> **作業ディレクトリ**: 以下のコマンド (手順 3〜6) は **`k8s/talos/`** で実行してください。

### 1. Talos イメージの準備

Talos `nocloud` AMD64 イメージ (raw.xz) をダウンロードし、解凍して Proxmox にアップロードする。

```bash
# イメージダウンロード
wget https://factory.talos.dev/image/<schematic-id>/v1.9/nocloud-amd64.raw.xz

# 解凍
xz -d nocloud-amd64.raw.xz

# Proxmox にアップロード (各ノードの local ストレージ)
scp nocloud-amd64.raw root@10.1.1.11:/var/lib/vz/template/
scp nocloud-amd64.raw root@10.1.1.12:/var/lib/vz/template/
scp nocloud-amd64.raw root@10.1.1.13:/var/lib/vz/template/
```

### 2. VM 作成 (各 Proxmox ノードで実行)

`vm/k8s-*/vm-config.json` の内容に従い、CLI で VM を作成する。

```bash
# 例: k8s-cp-01 (mandolin1 上で実行)
ssh root@10.1.1.11

# VM 設定作成
qm create 102001 --name k8s-cp-01 \
  --cpu cputype=host --cores 2 --memory 4096 --balloon 1024 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

# Talos イメージをインポート
qm importdisk 102001 /var/lib/vz/template/nocloud-amd64.raw vm-pool

# インポートしたディスクをアタッチ
qm set 102001 --scsi0 vm-pool:vm-102001-disk-0,discard=on,ssd=1
```

他の 5 台も同様に作成する（VM ID、ノード、スペックは上記テーブル参照）。

### 3. Machine Config 生成

```bash
cd k8s/talos  # 作業ディレクトリに移動

talosctl gen config k8s-cluster https://10.2.0.94:6443 \
  --kubernetes-version 1.32 \
  --with-docs=false \
  --with-examples=false
```

生成されるファイル (`controlplane.yaml`, `worker.yaml`, `talosconfig`) は `k8s/talos/` に配置され、シークレットを含むため Git 管理外。

### 4. パッチ適用と Machine Config の結合

`patches/` 配下のパッチを使用してノード別の設定を生成する。

```bash
# k8s/talos にいることを確認
pwd  # → /path/to/my-network/k8s/talos

# Control Plane ノード
talosctl machineconfig patch controlplane.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-cp-01.yaml \
  --output k8s-cp-01.yaml

talosctl machineconfig patch controlplane.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-cp-02.yaml \
  --output k8s-cp-02.yaml

talosctl machineconfig patch controlplane.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-cp-03.yaml \
  --output k8s-cp-03.yaml

# Worker ノード
talosctl machineconfig patch worker.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-worker-01.yaml \
  --output k8s-worker-01.yaml

talosctl machineconfig patch worker.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-worker-02.yaml \
  --output k8s-worker-02.yaml

talosctl machineconfig patch worker.yaml \
  --patch @patches/common.yaml \
  --patch @patches/k8s-worker-03.yaml \
  --output k8s-worker-03.yaml
```

### 5. Machine Config 適用

```bash
# 各 VM に machine config を適用
talosctl apply-config --insecure -n 10.2.0.66 -f k8s-cp-01.yaml
talosctl apply-config --insecure -n 10.2.0.67 -f k8s-cp-02.yaml
talosctl apply-config --insecure -n 10.2.0.68 -f k8s-cp-03.yaml
talosctl apply-config --insecure -n 10.2.0.69 -f k8s-worker-01.yaml
talosctl apply-config --insecure -n 10.2.0.70 -f k8s-worker-02.yaml
talosctl apply-config --insecure -n 10.2.0.71 -f k8s-worker-03.yaml
```

### 6. クラスタブートストラップ

```bash
# talosconfig を設定
export TALOSCONFIG=talosconfig
talosctl config endpoint 10.2.0.94

# 最初の control-plane で etcd ブートストラップ
talosctl bootstrap -n 10.2.0.66

# kubeconfig 取得
talosctl kubeconfig -n 10.2.0.94
```

### 7. Cilium インストール

```bash
# Helm repo 追加
helm repo add cilium https://helm.cilium.io/
helm repo update

# Cilium インストール
helm install cilium cilium/cilium \
  --namespace kube-system \
  --values k8s/cilium/values.yaml

# Cilium CRD 適用
kubectl apply -f k8s/cilium/manifests/lb-ip-pool.yaml
kubectl apply -f k8s/cilium/manifests/l2-announcement-policy.yaml
```

### 8. Catalyst ルータ設定

LB VIP 用のインターフェースルートを追加する。

```
configure terminal
ip route 10.5.0.0 255.255.255.0 Vlan102
end
copy running-config startup-config
```

## 検証手順

```bash
# 1. Talos ノード起動確認
talosctl health -n 10.2.0.94

# 2. K8s クラスタ確認
kubectl get nodes
# → 6 ノードが Ready であること

# 3. Cilium 状態確認
cilium status

# 4. L2 Announcement 確認
kubectl get leases -n kube-system | grep cilium-l2

# 5. テスト LB Service 作成
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=LoadBalancer --port=80
kubectl get svc nginx
# → EXTERNAL-IP に 10.5.0.x が割当されること

# 6. 別 VLAN (bastion) から LB VIP に curl
ssh admin@10.2.0.2 'curl http://10.5.0.x'
# → nginx のレスポンスが返ること

# テスト後のクリーンアップ
kubectl delete svc nginx
kubectl delete deployment nginx
```

## ディレクトリ構成

```
k8s/
├── README.md                               # 本ファイル
├── talos/
│   ├── patches/
│   │   ├── common.yaml                     # 共通パッチ (CNI, proxy, CIDR)
│   │   ├── k8s-cp-01.yaml                 # CP ノード別 (IP, VIP)
│   │   ├── k8s-cp-02.yaml
│   │   ├── k8s-cp-03.yaml
│   │   ├── k8s-worker-01.yaml             # Worker ノード別 (IP)
│   │   ├── k8s-worker-02.yaml
│   │   └── k8s-worker-03.yaml
│   ├── controlplane.yaml                   # 生成 (Git 管理外)
│   ├── worker.yaml                         # 生成 (Git 管理外)
│   ├── talosconfig                         # 生成 (Git 管理外)
│   ├── k8s-cp-01.yaml                     # 生成 (Git 管理外)
│   ├── k8s-cp-02.yaml                     # 生成 (Git 管理外)
│   ├── k8s-cp-03.yaml                     # 生成 (Git 管理外)
│   ├── k8s-worker-01.yaml                 # 生成 (Git 管理外)
│   ├── k8s-worker-02.yaml                 # 生成 (Git 管理外)
│   └── k8s-worker-03.yaml                 # 生成 (Git 管理外)
└── cilium/
    ├── values.yaml                         # Helm values
    └── manifests/
        ├── lb-ip-pool.yaml                # CiliumLoadBalancerIPPool
        └── l2-announcement-policy.yaml    # CiliumL2AnnouncementPolicy
```

> `talosctl gen config` および `talosctl machineconfig patch` で生成されるファイルは `k8s/talos/` に配置され、シークレットを含むため `.gitignore` で除外している。

## 関連ドキュメント

- [ネットワーク設計](../docs/design.md)
- [Catalyst ルータ設定](../router/catalyst-3560cx.conf)
- [VM 設定ファイル](../vm/)
