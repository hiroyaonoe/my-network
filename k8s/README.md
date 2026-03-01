# K8s クラスタ (Talos Linux + Cilium)

VLAN 102 (vm-k8s, 10.2.0.64/27) 上の HA Kubernetes クラスタ。

## クラスタ情報

| 項目 | 値 |
|------|-----|
| OS | Talos Linux |
| K8s バージョン | 1.35.2 |
| CNI | Cilium (Native Routing + L2 Announcement) |
| API Server VIP | 10.2.0.94 |
| API Endpoint | `https://10.2.0.94:6443` |
| Pod CIDR | 10.3.0.0/16 |
| Service CIDR | 10.4.0.0/16 |
| LB VIP Pool | 10.5.0.0/24 |

## ノード一覧

| VM名 | VM ID | IP | Proxmoxノード | 役割 | vCPU | RAM | Disk |
|-------|-------|-----|---------------|------|------|-----|------|
| k8s-cp-01 | 102001 | 10.2.0.66 | mandolin1 | control-plane | 4 | 8GB | 32GB |
| k8s-cp-02 | 102002 | 10.2.0.67 | mandolin2 | control-plane | 4 | 8GB | 32GB |
| k8s-cp-03 | 102003 | 10.2.0.68 | mandolin3 | control-plane | 4 | 8GB | 32GB |
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

Talos `nocloud` AMD64 ISO をダウンロードし、各 Proxmox ノードにアップロードする。

> **重要**: nocloud qcow2/raw イメージではなく **ISO** を使用する。ISO からブートした場合のみ `machine.install` に従ったディスクインストールが行われる。qcow2/raw は既にインストール済みのイメージであり、ネットワーク設定が正しく反映されない場合がある。

```bash
# ISO ダウンロード
wget https://factory.talos.dev/image/<schematic-id>/v1.12.4/nocloud-amd64.iso

# Proxmox にアップロード (各ノードの ISO ストレージ)
scp nocloud-amd64.iso root@10.1.1.11:/var/lib/vz/template/iso/
scp nocloud-amd64.iso root@10.1.1.12:/var/lib/vz/template/iso/
scp nocloud-amd64.iso root@10.1.1.13:/var/lib/vz/template/iso/
```

### 3. VM 作成 (各 Proxmox ノードで実行)

`vm/k8s-*/vm-config.json` の内容に従い、CLI で VM を作成する。

> **重要**: SCSI コントローラは `virtio-scsi-pci` を使用すること (Talos 公式推奨)。デフォルトの LSI や VirtIO SCSI Single ではディスクが認識されない場合がある。

```bash
# k8s-cp-01 (mandolin1 上で実行)
ssh root@10.1.1.11

qm create 102001 --name k8s-cp-01 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 0 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:32,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1

# k8s-worker-01 (同じノード上)
qm create 102004 --name k8s-worker-01 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:64,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1
```

```bash
# k8s-cp-02 + k8s-worker-02 (mandolin2 上で実行)
ssh root@10.1.1.12

qm create 102002 --name k8s-cp-02 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 0 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:32,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1

qm create 102005 --name k8s-worker-02 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:64,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1
```

```bash
# k8s-cp-03 + k8s-worker-03 (mandolin3 上で実行)
ssh root@10.1.1.13

qm create 102003 --name k8s-cp-03 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 0 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:32,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1

qm create 102006 --name k8s-worker-03 \
  --cpu cputype=host --cores 4 --memory 8192 --balloon 2048 \
  --scsihw virtio-scsi-pci \
  --net0 virtio,bridge=vmbr0,tag=102 \
  --ide2 local:iso/nocloud-amd64.iso,media=cdrom \
  --scsi0 vm-pool:64,discard=on,ssd=1 \
  --boot order='ide2;scsi0' --onboot 1
```

### 4. Machine Config 生成

```bash
cd k8s/talos  # 作業ディレクトリに移動

talosctl gen config k8s-cluster https://10.2.0.94:6443 \
  --kubernetes-version 1.35.2 \
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

### 6. VM 起動と Machine Config 適用

ISO からブートし、machine config を適用してディスクにインストールする。

> **フロー**: ISO ブート → DHCP で一時 IP 取得 → ブート順序をディスク優先に変更 → `apply-config` → ディスクにインストール → 自動リブート → 静的 IP で起動

**手順 (各 VM で繰り返す):**

```bash
# 1. VM 起動 (ISO からブート)
qm start <VMID>

# 2. DHCP IP を確認 (約 30 秒待機)
#    Catalyst: show ip dhcp binding
#    または Proxmox コンソールで確認

# 3. ブート順序をディスク優先に変更 (リブート後に ISO から起動しないように)
qm set <VMID> --boot order='scsi0;ide2'

# 4. machine config を適用 (インストール→自動リブート)
talosctl apply-config --insecure -n <DHCP_IP> -f <node>.yaml

# 5. 静的 IP で起動確認 (約 60 秒待機)
ping -c 3 <STATIC_IP>
```

**具体例:**
```bash
# k8s/talos ディレクトリで実行

# k8s-cp-01 (DHCP IP が 10.2.0.80 の場合)
qm set 102001 --boot order='scsi0;ide2'
talosctl apply-config --insecure -n 10.2.0.80 -f k8s-cp-01.yaml
# → 自動リブート後、10.2.0.66 で起動

# 残りのノードも同様に apply-config
talosctl apply-config --insecure -n <DHCP_IP> -f k8s-cp-02.yaml
talosctl apply-config --insecure -n <DHCP_IP> -f k8s-cp-03.yaml
talosctl apply-config --insecure -n <DHCP_IP> -f k8s-worker-01.yaml
talosctl apply-config --insecure -n <DHCP_IP> -f k8s-worker-02.yaml
talosctl apply-config --insecure -n <DHCP_IP> -f k8s-worker-03.yaml
```

> **Note**: `--insecure` フラグは初回適用時のみ使用。`apply-config` 実行後、Talos がディスクにインストールし自動リブートする。リブート後は静的 IP (10.2.0.66〜71) で起動する。

### 7. クラスタブートストラップ

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

### 8. Cilium インストール

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

### 9. Catalyst LB VIP ルート設定

LB VIP 用のインターフェースルートを追加する。

```
configure terminal
ip route 10.5.0.0 255.255.255.0 Vlan102
end
copy running-config startup-config
```

### 10. ArgoCD インストール

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --values k8s/argocd/values.yaml
```

### 11. ArgoCD 初期パスワード取得・ログイン

```bash
# 初期 admin パスワード
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# ArgoCD LB IP 確認
kubectl -n argocd get svc argocd-server
# → EXTERNAL-IP: 10.5.0.1

# ブラウザで http://10.5.0.1 にアクセス
```

### 12. Root Application 投入 (GitOps 開始)

```bash
kubectl apply -f k8s/argocd/bootstrap/root.yaml
```

> この時点で ArgoCD が Cilium・cilium-config・argocd (自身) を検出し、同期を開始する。
> 手動 Helm install した状態と Git の定義が一致していれば、差分なく Synced 状態になる。

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

# 7. ArgoCD Pod 確認
kubectl -n argocd get pods

# 8. ArgoCD LB VIP 確認
kubectl -n argocd get svc argocd-server
# → EXTERNAL-IP: 10.5.0.1

# 9. Application 同期状態確認
kubectl -n argocd get applications
# → root, cilium, cilium-config, argocd が全て Synced/Healthy
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
├── cilium/
│   ├── values.yaml                         # Helm values
│   └── manifests/
│       ├── lb-ip-pool.yaml                # CiliumLoadBalancerIPPool
│       └── l2-announcement-policy.yaml    # CiliumL2AnnouncementPolicy
└── argocd/
    ├── values.yaml                         # ArgoCD Helm values
    ├── bootstrap/
    │   └── root.yaml                      # Root Application (手動適用)
    └── apps/
        ├── cilium.yaml                    # Cilium Application
        ├── cilium-config.yaml             # Cilium CRD Application
        └── argocd.yaml                    # ArgoCD self-management
```

> `talosctl gen config` および `talosctl machineconfig patch` で生成されるファイルは `k8s/talos/` に配置され、シークレットを含むため `.gitignore` で除外している。

## 関連ドキュメント

- [ネットワーク設計](../docs/design.md)
- [Catalyst ルータ設定](../router/catalyst-3560cx.conf)
- [VM 設定ファイル](../vm/)
