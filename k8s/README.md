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

> **作業ディレクトリ**: 以下のコマンド (手順 4〜7) は **`k8s/talos/`** で実行してください。

### 1. Catalyst DHCP サーバー設定

VM 初回起動時に DHCP で一時 IP を取得するため、Catalyst に DHCP サーバーを設定する。

```
configure terminal

! DHCP プール作成
ip dhcp pool k8s-dhcp
 network 10.2.0.64 255.255.255.224
 default-router 10.2.0.65
 dns-server 10.0.0.1
 lease 0 1 0
 exit

! 静的 IP 範囲を除外
ip dhcp excluded-address 10.2.0.64 10.2.0.79
ip dhcp excluded-address 10.2.0.91 10.2.0.95

end
copy running-config startup-config
```

DHCP 範囲: 10.2.0.80〜90 (11 個、初回起動用)
静的 IP: 10.2.0.66〜71 (K8s ノード), 10.2.0.94 (VIP)

### 2. Talos イメージの準備

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

### 3. VM 作成 (各 Proxmox ノードで実行)

`vm/k8s-*/vm-config.json` の内容に従い、CLI で VM を作成する。

```bash
# k8s-cp-01 (mandolin1 上で実行)
ssh root@10.1.1.11

qm create 102001 --name k8s-cp-01 \
  --cpu cputype=host --cores 2 --memory 4096 --balloon 1024 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102001 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102001 --scsi0 vm-pool:vm-102001-disk-0,discard=on,ssd=1

# k8s-worker-01 (同じノード上)
qm create 102004 --name k8s-worker-01 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102004 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102004 --scsi0 vm-pool:vm-102004-disk-0,discard=on,ssd=1
```

```bash
# k8s-cp-02 + k8s-worker-02 (mandolin2 上で実行)
ssh root@10.1.1.12

qm create 102002 --name k8s-cp-02 \
  --cpu cputype=host --cores 2 --memory 4096 --balloon 1024 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102002 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102002 --scsi0 vm-pool:vm-102002-disk-0,discard=on,ssd=1

qm create 102005 --name k8s-worker-02 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102005 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102005 --scsi0 vm-pool:vm-102005-disk-0,discard=on,ssd=1
```

```bash
# k8s-cp-03 + k8s-worker-03 (mandolin3 上で実行)
ssh root@10.1.1.13

qm create 102003 --name k8s-cp-03 \
  --cpu cputype=host --cores 2 --memory 4096 --balloon 1024 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102003 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102003 --scsi0 vm-pool:vm-102003-disk-0,discard=on,ssd=1

qm create 102006 --name k8s-worker-03 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --boot order=scsi0 --onboot 1

qm importdisk 102006 /var/lib/vz/template/nocloud-amd64.raw vm-pool
qm set 102006 --scsi0 vm-pool:vm-102006-disk-0,discard=on,ssd=1
```

### 4. Machine Config 生成

```bash
cd k8s/talos  # 作業ディレクトリに移動

talosctl gen config k8s-cluster https://10.2.0.94:6443 \
  --kubernetes-version 1.32 \
  --with-docs=false \
  --with-examples=false
```

生成されるファイル (`controlplane.yaml`, `worker.yaml`, `talosconfig`) は `k8s/talos/` に配置され、シークレットを含むため Git 管理外。

**Talos 1.12のマルチドキュメント形式について**:
- パッチファイルは Talos 1.12 の新しいマルチドキュメント形式を使用
- ネットワーク設定は `LinkConfig`, `Layer2VIPConfig`, `ResolverConfig` で構成
- Hostname は `HostnameConfig` ドキュメントで設定
- この形式は Talos 1.12 で推奨される設定方法

### 5. パッチ適用と Machine Config の結合

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

### 6. VM 起動

全 6 台の VM を起動する。

**Proxmox Web UI の場合:**
1. 各 VM (102001〜102006) を選択
2. **Start** をクリック

**CLI の場合:**
```bash
# 各 Proxmox ノードで実行
ssh root@10.1.1.11
qm start 102001  # k8s-cp-01
qm start 102004  # k8s-worker-01

ssh root@10.1.1.12
qm start 102002  # k8s-cp-02
qm start 102005  # k8s-worker-02

ssh root@10.1.1.13
qm start 102003  # k8s-cp-03
qm start 102006  # k8s-worker-03
```

起動後、Talos が Catalyst DHCP から一時 IP (10.2.0.80〜90) を取得するまで待つ (約 30 秒)。

**DHCP IP の確認:**
```bash
# Catalyst でリース確認
show ip dhcp binding
```

または Proxmox Web UI の VM Console で確認。

### 7. Machine Config 適用

DHCP で取得した一時 IP に対して machine config を適用する。

```bash
# k8s/talos ディレクトリで実行

# DHCP で取得した IP を確認後、各 VM に machine config を適用
# 例: k8s-cp-01 が 10.2.0.80 を取得した場合
talosctl apply-config --insecure -n 10.2.0.80 -f k8s-cp-01.yaml
talosctl apply-config --insecure -n 10.2.0.81 -f k8s-cp-02.yaml
talosctl apply-config --insecure -n 10.2.0.82 -f k8s-cp-03.yaml
talosctl apply-config --insecure -n 10.2.0.83 -f k8s-worker-01.yaml
talosctl apply-config --insecure -n 10.2.0.84 -f k8s-worker-02.yaml
talosctl apply-config --insecure -n 10.2.0.85 -f k8s-worker-03.yaml
```

> **Note**: `--insecure` フラグは初回適用時のみ使用。Machine config 適用後、各 VM は静的 IP (10.2.0.66〜71) で再起動される。

### 8. クラスタブートストラップ

VM が静的 IP で再起動完了後、クラスタをブートストラップする。

```bash
# k8s/talos ディレクトリで実行

# talosconfig を設定
export TALOSCONFIG=talosconfig
talosctl config endpoint 10.2.0.94

# 最初の control-plane で etcd ブートストラップ
talosctl bootstrap -n 10.2.0.66

# kubeconfig 取得
talosctl kubeconfig -n 10.2.0.94
```

### 9. Cilium インストール

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

### 10. Catalyst LB VIP ルート設定

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
