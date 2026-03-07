# K8s クラスタ (Talos Linux + Cilium)

VLAN 102 (vm-k8s, 10.2.0.64/27) 上の HA Kubernetes クラスタ。

## クラスタ情報

| 項目 | 値 |
|------|-----|
| OS | Talos Linux |
| K8s バージョン | 1.35.2 |
| CNI | Cilium (Native Routing + L2 Announcement) |
| Storage | Rook Ceph External (RBD → Proxmox Ceph) |
| Default StorageClass | ceph-block |
| API Server VIP | 10.2.0.94 |
| API Endpoint | `https://10.2.0.94:6443` |
| Pod CIDR | 10.3.0.0/16 |
| Service CIDR | 10.4.0.0/16 |
| LB VIP Pool | 10.5.0.0/24 |

## ノード一覧

| VM名 | VM ID | IP | Proxmoxノード | 役割 | vCPU | RAM | Disk | 状態 |
|-------|-------|-----|---------------|------|------|-----|------|------|
| k8s-cp-01 | 102001 | 10.2.0.66 | mandolin1 | control-plane | 4 | 8GB | 32GB | 稼働中 |
| k8s-cp-02 | 102002 | 10.2.0.67 | mandolin2 | control-plane | 4 | 8GB | 32GB | 停止中 |
| k8s-cp-03 | 102003 | 10.2.0.68 | mandolin3 | control-plane | 4 | 8GB | 32GB | 停止中 |
| k8s-worker-01 | 102004 | 10.2.0.69 | mandolin1 | worker | 4 | 8GB | 64GB | 停止中 |
| k8s-worker-02 | 102005 | 10.2.0.70 | mandolin2 | worker | 4 | 8GB | 64GB | 稼働中 |
| k8s-worker-03 | 102006 | 10.2.0.71 | mandolin3 | worker | 4 | 8GB | 64GB | 稼働中 |

> **Note**: メモリリソース不足のため 1 CP + 2 Worker の縮退構成で運用中。HA 復帰手順は [node-scaling.md](docs/node-scaling.md) を参照。

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

# ブラウザで https://argocd.internal.onoe.dev にアクセス
```

### 12. Root Application 投入 (GitOps 開始)

```bash
kubectl apply -f k8s/argocd/bootstrap/root.yaml
```

> この時点で ArgoCD が Cilium・cilium-config・argocd (自身) を検出し、同期を開始する。
> 手動 Helm install した状態と Git の定義が一致していれば、差分なく Synced 状態になる。

## TLS (cert-manager + Self-Signed CA)

cert-manager で内部 CA を構築し、ArgoCD 等のサービスに TLS 証明書を発行する。
`.dev` TLD は HSTS preload リストにより HTTPS が必須のため、この設定が必要。

### アーキテクチャ

```
cert-manager (cert-manager namespace)
  ├── SelfSigned ClusterIssuer    ← 自己署名で CA 証明書を発行
  ├── CA Certificate              ← internal-ca-root (ルート CA 証明書)
  └── CA ClusterIssuer            ← internal-ca (全 namespace で利用可能)

ArgoCD (argocd namespace)
  └── Certificate CR → argocd-server-tls Secret
        ↑ cert-manager が internal-ca で署名
```

### デプロイ (ArgoCD 自動)

`k8s/argocd/apps/cert-manager.yaml` と `k8s/argocd/apps/cert-manager-config.yaml` が main にマージされると、ArgoCD が自動的に:
1. cert-manager (CRDs + controller) をデプロイ
2. ClusterIssuer + CA Certificate を作成
3. ArgoCD の Certificate CR から `argocd-server-tls` Secret を生成・TLS 有効化

### CA 証明書の信頼 (手動)

自己署名 CA のため、ブラウザの証明書警告を回避するには CA 証明書をシステムに信頼させる:

```bash
# CA 証明書のエクスポート
kubectl -n cert-manager get secret internal-ca-root-secret \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > internal-ca.crt

# macOS: キーチェーンに追加
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain internal-ca.crt
```

### 検証

```bash
# ClusterIssuer 確認
kubectl get clusterissuer
# → selfsigned-issuer, internal-ca ともに Ready

# CA Certificate 確認
kubectl -n cert-manager get certificate internal-ca-root
# → Ready: True

# ArgoCD Certificate 確認
kubectl -n argocd get certificate
# → argocd-server-tls Ready: True

# HTTPS アクセス確認
curl -k https://argocd.internal.onoe.dev
```

## Rook Ceph (External Cluster)

Proxmox が管理する既存の Ceph クラスタに Rook の external cluster モードで接続し、K8s ワークロードに RBD ブロックストレージを提供する。

### アーキテクチャ

```
K8s Pod (PVC: ceph-block)
  ↓ CSI Driver (rook-ceph)
K8s Node (10.2.0.66-71, VLAN 102)
  ↓ Catalyst L3 routing
Ceph Monitor (10.1.1.11-13, VLAN 10)
  ↓
Ceph OSD (k8s-rbd pool, 3x replication)
```

Rook は external モードで動作し、Ceph デーモン (MON/OSD) は管理しない。CSI ドライバのみを提供する。

### ブートストラップ手順 (手順 12 の後に実施)

#### 13a. Ceph プール作成 (Proxmox)

```bash
ssh root@10.1.1.11
ceph osd pool create k8s-rbd 32 32 replicated
ceph osd pool set k8s-rbd size 3
ceph osd pool set k8s-rbd min_size 2
ceph osd pool application enable k8s-rbd rbd
```

#### 13b. クレデンシャル生成 (Proxmox)

```bash
wget https://raw.githubusercontent.com/rook/rook/v1.19.2/deploy/examples/create-external-cluster-resources.py

python3 create-external-cluster-resources.py \
  --rbd-data-pool-name k8s-rbd \
  --namespace rook-ceph \
  --format bash \
  --skip-monitoring-endpoint \
  --v2-port-enable
# → export 文が出力される。保存しておく
```

#### 13c. K8s に Secrets 投入

```bash
source /path/to/ceph-export.sh

wget https://raw.githubusercontent.com/rook/rook/v1.19.2/deploy/examples/import-external-cluster.sh
chmod +x import-external-cluster.sh
NAMESPACE=rook-ceph ./import-external-cluster.sh
```

#### 13d. Git push (ArgoCD が自動デプロイ)

`k8s/argocd/apps/rook-ceph.yaml` と `k8s/argocd/apps/rook-ceph-cluster.yaml` が main ブランチにマージされると、ArgoCD が自動的に:
1. Rook Operator + CSI ドライバをデプロイ
2. CephCluster CR + StorageClass `ceph-block` を作成

> Operator の CRD インストールが完了するまで cluster app は失敗するが、ArgoCD の auto-retry で自動回復する。

#### 13e. 検証

```bash
# Operator Pod 確認
kubectl -n rook-ceph get pods

# CephCluster 状態
kubectl -n rook-ceph get cephcluster
# → Connected / HEALTH_OK

# StorageClass 確認
kubectl get sc
# → ceph-block (default)

# テスト PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-rbd-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-block
EOF
kubectl get pvc test-rbd-pvc  # → Bound

# 後片付け
kubectl delete pvc test-rbd-pvc
```

## HA DNS サーバー (k8s_gateway)

CoreDNS + k8s_gateway プラグインによる HA DNS サーバー。静的レコード・K8s Service 自動登録・上流 DNS 転送を提供する。

### アーキテクチャ

```
Client (VM / Tailscale)
  ↓ DNS query (any domain)
k8s_gateway (10.5.0.53:53, LoadBalancer, 2 replicas)
  ↓ CoreDNS (.:1053)
  ├── hosts plugin → 静的レコード → fallthrough
  ├── k8s_gateway plugin → K8s Service/HTTPRoute 自動解決 → fallthrough
  └── forward plugin → 10.0.0.1 (上流 DNS)
```

| 項目 | 値 |
|------|-----|
| ドメイン | internal.onoe.dev |
| DNS VIP | 10.5.0.53 |
| レプリカ数 | 2 (縮退構成、通常 3) |

> **アノテーションと自動解決の排他性**: `external-dns.alpha.kubernetes.io/hostname` アノテーションを設定した LB Service は、カスタム名のみで解決される。自動の `<service>.<namespace>.<domain>` 形式は生成されない。アノテーションなしの LB Service は自動形式で解決される。

### デプロイ (ArgoCD 自動)

`k8s/argocd/apps/k8s-gateway.yaml` が main にマージされると、ArgoCD が自動的に k8s-gateway namespace にデプロイする。

### 検証

```bash
# 1. Pod 確認 (3 replicas on different nodes)
kubectl -n k8s-gateway get pods -o wide

# 2. Service VIP 確認
kubectl -n k8s-gateway get svc
# → EXTERNAL-IP: 10.5.0.53

# 3. 静的レコード
dig @10.5.0.53 mandolin1.internal.onoe.dev +short
# → 10.1.1.11

# 4. アノテーション付きカスタム名
dig @10.5.0.53 argocd.internal.onoe.dev +short
# → 10.5.0.1

# 6. 外部 DNS 転送
dig @10.5.0.53 google.com +short
# → 正常解決

# 7. 存在しない内部名
dig @10.5.0.53 nonexistent.internal.onoe.dev
# → NXDOMAIN
```

### Tailscale Split DNS (手動)

1. bastion-01 で LB VIP ネットワークを Tailscale に広告:

   ```bash
   # 既存の advertise-routes に 10.5.0.0/24 を追加
   sudo tailscale up --advertise-routes=10.1.1.0/24,10.2.0.0/16,10.5.0.0/24 --accept-routes
   ```

2. Tailscale admin console (https://login.tailscale.com/admin/machines) で bastion-01 の `10.5.0.0/24` ルートを承認

3. Tailscale admin console → DNS → Split DNS:
   - `internal.onoe.dev` → 10.5.0.53

## 検証手順

```bash
# 1. Talos ノード起動確認
talosctl health -n 10.2.0.94

# 2. K8s クラスタ確認
kubectl get nodes
# → 3 ノードが Ready であること (cp-01, worker-02, worker-03)

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
├── cert-manager/
│   ├── values.yaml                         # cert-manager Helm values
│   └── manifests/
│       ├── self-signed-issuer.yaml        # SelfSigned ClusterIssuer
│       ├── ca-certificate.yaml            # Internal CA root Certificate
│       └── ca-issuer.yaml                 # Internal CA ClusterIssuer
├── rook-ceph/
│   ├── operator-values.yaml               # Rook Operator Helm values
│   ├── cluster-values.yaml                # Rook Ceph Cluster Helm values
│   └── manifests/
│       ├── storageclass.yaml              # StorageClass (ceph-block)
│       ├── externalsecret-rook-ceph-mon.yaml          # ExternalSecret
│       ├── externalsecret-rook-ceph-config.yaml       # ExternalSecret
│       ├── externalsecret-rook-csi-rbd-provisioner.yaml # ExternalSecret
│       └── externalsecret-rook-csi-rbd-node.yaml      # ExternalSecret
├── openclaw/
│   └── manifests/
│       ├── statefulset.yaml              # StatefulSet + headless Service
│       ├── service.yaml                  # LoadBalancer Service (Control UI)
│       ├── certificate.yaml             # TLS Certificate (cert-manager)
│       ├── configmap.yaml               # openclaw.json
│       ├── externalsecret.yaml          # ExternalSecret (Vault → tokens/API keys)
│       ├── networkpolicy-default-deny.yaml           # Default deny all
│       ├── networkpolicy-allow-dns.yaml              # Allow DNS to CoreDNS
│       ├── networkpolicy-allow-external-http.yaml    # Allow HTTP/HTTPS to external
│       └── networkpolicy-allow-ui-ingress.yaml       # Allow ingress to Control UI
├── docs/
│   └── node-scaling.md                    # ノード削減/HA 復帰手順
├── k8s-gateway/
│   └── values.yaml                         # k8s-gateway Helm values
├── vault/
│   ├── values.yaml                         # Vault Helm values (Raft standalone)
│   └── manifests/
│       └── certificate.yaml               # Vault TLS Certificate
├── external-secrets/
│   ├── values.yaml                         # ESO Helm values
│   └── manifests/
│       └── cluster-secret-store.yaml      # ClusterSecretStore (Vault)
├── monitoring/
│   ├── values.yaml                         # kube-prometheus-stack Helm values
│   └── manifests/
│       ├── grafana-certificate.yaml       # Grafana TLS Certificate
│       ├── pve-exporter.yaml              # PVE Exporter Deployment + Service
│       ├── externalsecret-alertmanager-slack-webhook.yaml  # ExternalSecret
│       └── externalsecret-pve-exporter-credentials.yaml   # ExternalSecret
├── loki/
│   └── values.yaml                         # Loki Helm values
├── alloy/
│   └── values.yaml                         # Grafana Alloy Helm values
├── tempo/
│   └── values.yaml                         # Grafana Tempo Helm values
├── opentelemetry/
│   └── values.yaml                         # OTel Collector Helm values
└── argocd/
    ├── values.yaml                         # ArgoCD Helm values
    ├── bootstrap/
    │   └── root.yaml                      # Root Application (手動適用)
    └── apps/
        ├── cilium.yaml                    # Cilium Application
        ├── cilium-config.yaml             # Cilium CRD Application
        ├── argocd.yaml                    # ArgoCD self-management
        ├── cert-manager.yaml              # cert-manager Application
        ├── cert-manager-config.yaml       # cert-manager config Application
        ├── rook-ceph.yaml                 # Rook Operator Application
        ├── rook-ceph-cluster.yaml         # Rook Ceph Cluster Application
        ├── k8s-gateway.yaml               # k8s-gateway DNS Application
        ├── monitoring.yaml                # kube-prometheus-stack Application
        ├── monitoring-config.yaml         # Grafana TLS + PVE Exporter Application
        ├── loki.yaml                      # Loki Application
        ├── alloy.yaml                     # Grafana Alloy Application
        ├── tempo.yaml                     # Grafana Tempo Application
        ├── opentelemetry.yaml             # OTel Collector Application
        ├── vault.yaml                     # Vault Application
        ├── vault-config.yaml              # Vault TLS Certificate Application
        ├── external-secrets.yaml          # ESO Application
        ├── external-secrets-config.yaml   # ClusterSecretStore Application
        └── openclaw.yaml                # OpenClaw AI Gateway Application
```

> `talosctl gen config` および `talosctl machineconfig patch` で生成されるファイルは `k8s/talos/` に配置され、シークレットを含むため `.gitignore` で除外している。

## オブザーバビリティ (Grafana Stack)

メトリクス (Prometheus)、ログ (Loki)、トレース (Tempo) のフルスタック可観測性環境。

### アーキテクチャ

```
[Grafana] (10.5.0.2, grafana.internal.onoe.dev, TLS)
  ├── datasource: Prometheus (metrics)
  ├── datasource: Loki (logs)
  └── datasource: Tempo (traces)

[Prometheus] ← scrape ← node-exporter, kube-state-metrics, pve-exporter
  ├── additionalScrapeConfigs → Ceph MGR (10.1.1.11-13:9283)
  └── remoteWriteReceiver ← Tempo metrics-generator

[Alertmanager] → Slack webhook (Secret)

[Loki] (monolithic) ← [Alloy DaemonSet] (pod logs)

[Tempo] (monolithic) ← [OTel Collector] ← apps (OTLP)
  └── metrics-generator → Prometheus (RED metrics)
```

### 手動前提条件 (Git push 前)

#### 1. Proxmox: API トークン作成

```bash
ssh root@10.1.1.11
pveum user add monitor@pve
pveum acl modify / -user monitor@pve -role PVEAuditor
pveum user token add monitor@pve prometheus --privsep 0
# → トークン値を記録
```

#### 2. Proxmox: Ceph Prometheus Module 有効化

```bash
ssh root@10.1.1.11
ceph mgr module enable prometheus
# 確認: curl http://10.1.1.11:9283/metrics | head
```

#### 3. Slack: Webhook URL 取得

https://api.slack.com/apps → Create New App → Incoming Webhooks → Add to Channel

#### 4. K8s: Secrets 作成

```bash
kubectl create namespace monitoring

kubectl -n monitoring create secret generic alertmanager-slack-webhook \
  --from-literal=url='https://hooks.slack.com/services/T.../B.../xxx'

kubectl -n monitoring create secret generic pve-exporter-credentials \
  --from-literal=user='monitor@pve' \
  --from-literal=token_name='prometheus' \
  --from-literal=token_value='<token-value>'
```

### デプロイ (ArgoCD 自動)

`k8s/argocd/apps/` 配下の monitoring, monitoring-config, loki, alloy, tempo, opentelemetry が main にマージされると ArgoCD が自動デプロイする。

### 検証

```bash
# 1. 全 Pod 起動確認
kubectl -n monitoring get pods
kubectl -n loki get pods
kubectl -n alloy get pods
kubectl -n tempo get pods
kubectl -n opentelemetry get pods

# 2. Grafana TLS アクセス
curl -k https://grafana.internal.onoe.dev

# 3. Grafana 初期パスワード
kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d

# 4. Loki ログ確認 (Grafana Explore → Loki → {namespace="monitoring"})

# 5. Ceph メトリクス確認 (Grafana Explore → Prometheus → ceph_health_status)

# 6. ArgoCD Application 状態
kubectl -n argocd get applications
# → monitoring, monitoring-config, loki, alloy, tempo, opentelemetry 全て Synced/Healthy
```

## Secret 管理 (Vault + External Secrets Operator)

HashiCorp Vault (Raft standalone) + External Secrets Operator で K8s Secret を GitOps 管理する。

> **Note**: メモリリソース不足のため standalone (1 replica) で運用中。HA 復帰手順は [node-scaling.md](docs/node-scaling.md) を参照。

### アーキテクチャ

```
[Vault] (10.5.0.3, vault.internal.onoe.dev, TLS)
  ├── Raft storage (1 replica, standalone)
  ├── KV v2 Secret Engine (path: secret/)
  └── Kubernetes Auth (ESO 用 ServiceAccount)

[External Secrets Operator] (external-secrets namespace)
  └── ClusterSecretStore → Vault KV v2

[ExternalSecret CR] (各 namespace)
  ↓ ESO が定期的に同期
[K8s Secret] (自動生成)
```

| 項目 | 値 |
|------|-----|
| Vault VIP | 10.5.0.3 |
| Vault UI | https://vault.internal.onoe.dev |
| Namespace | vault, external-secrets |

### デプロイ手順

#### Phase 1: Vault

1. `k8s/vault/` と ArgoCD Apps を Git push
2. ArgoCD sync: cert-manager → TLS 証明書 → Vault pods 起動 (sealed 状態)
3. **手動**: Vault 初期化・Unseal・設定 (vault-0 コンテナ内で実施)

```bash
# vault-0 に exec (VAULT_ADDR, VAULT_CACERT は環境変数で設定済み)
kubectl -n vault exec -it vault-0 -- sh

# 初期化 (unseal keys を安全に保管すること)
vault operator init -key-shares=3 -key-threshold=2

# vault-0 を Unseal (2回)
vault operator unseal  # key 1
vault operator unseal  # key 2

# Root token でログイン
vault login

# KV v2 有効化
vault secrets enable -path=secret kv-v2

# Kubernetes auth 有効化
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"

# ESO 用 policy 作成
vault policy write external-secrets - <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

# ESO 用 role 作成
vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h

exit
```

4. **手動**: 自動 unseal 用 Secret を作成 (init で取得した unseal key のうち threshold 分)

```bash
# ブートストラップ用 (初回のみ手動作成)
kubectl -n vault create secret generic vault-unseal-keys \
  --from-literal=key-0='<unseal-key-1>' \
  --from-literal=key-1='<unseal-key-2>'
kubectl -n vault create secret generic vault-root-token \
  --from-literal=token='<root-token>'

# Vault に unseal keys と root token を格納 (ESO 移行用)
kubectl -n vault exec -it vault-0 -- sh
vault login
vault kv put secret/vault/unseal-keys \
  key-0='<unseal-key-1>' \
  key-1='<unseal-key-2>'
vault kv put secret/vault/root-token \
  token='<root-token>'
exit

# 手動作成した Secret を削除 (ESO が ExternalSecret から再生成する)
kubectl -n vault delete secret vault-unseal-keys vault-root-token
```

> Pod 起動時に postStart hook が unseal key を読み取り自動的に unseal する。
> vault-1, vault-2 は Raft retry_join でリーダーに join 後、自動 unseal される。
> ブートストラップ後は ESO が Vault から unseal keys を同期するため手動管理不要。
> 全 Vault pod が同時停止しても、K8s Secret は残っているため自動復旧する。

#### Phase 2: External Secrets Operator

5. `k8s/external-secrets/` と ArgoCD Apps を Git push
6. ArgoCD sync: ESO Operator → ClusterSecretStore

#### Phase 3: 既存 Secret 移行

7. **手動**: 既存 Secret 値を Vault に格納 (vault-0 コンテナ内で実施)

```bash
kubectl -n vault exec -it vault-0 -- sh
vault login

vault kv put secret/monitoring/alertmanager-slack-webhook \
  url='https://hooks.slack.com/services/T.../B.../xxx'

vault kv put secret/monitoring/pve-exporter-credentials \
  user='monitor@pve' \
  token_name='prometheus' \
  token_value='<token-value>'

# rook-ceph (import-external-cluster.sh の出力値を使用)
vault kv put secret/rook-ceph/rook-ceph-mon \
  cluster-name='rook-ceph' \
  fsid='<fsid>' \
  admin-secret='<admin-key>' \
  mon-secret='' \
  ceph-username='client.admin' \
  ceph-secret='<admin-key>'

vault kv put secret/rook-ceph/rook-ceph-config \
  mon_host='[v2:10.1.1.11:3300],[v2:10.1.1.12:3300],[v2:10.1.1.13:3300]' \
  mon_initial_members='mandolin1,mandolin2,mandolin3'

vault kv put secret/rook-ceph/rook-csi-rbd-provisioner \
  userID='csi-rbd-provisioner' \
  userKey='<cephx-key>'

vault kv put secret/rook-ceph/rook-csi-rbd-node \
  userID='csi-rbd-node' \
  userKey='<cephx-key>'

exit
```

8. ExternalSecret CRs を Git push
9. ESO が K8s Secret を自動生成

### 検証

```bash
# Vault status (standalone)
kubectl -n vault get pods  # vault-0 Running
kubectl -n vault exec vault-0 -- vault status               # Sealed=false

# DNS/TLS
dig @10.5.0.53 vault.internal.onoe.dev  # → 10.5.0.3
curl -k https://vault.internal.onoe.dev:8200/v1/sys/health

# ESO
kubectl get clustersecretstore vault  # Ready=True

# 既存 Secret 移行確認
kubectl -n monitoring get externalsecret  # SecretSynced=True
```

## OpenClaw (AI Coding Agent Gateway)

チャットアプリ (Slack 等) と AI コーディングエージェントを接続するセルフホスト型ゲートウェイ。VM (opencraw-01) から K8s コンテナに移行。

ref: https://github.com/hiroyaonoe/my-network/issues/11

### アーキテクチャ

```
[StatefulSet] (openclaw namespace, 1 replica)
  ├── image: ghcr.io/openclaw/openclaw:latest
  ├── port: 18789 (gateway + Control UI, wss://)
  ├── PVC: 32Gi (ceph-block) → /home/node/.openclaw
  ├── ConfigMap → openclaw.json (OPENCLAW_CONFIG_PATH), exec-approvals.json (subPath)
  ├── ExternalSecret → Vault (tokens/API keys)
  └── TLS: gateway.tls.enabled=true (自己生成証明書)

[Service] openclaw (headless)
  └── ClusterIP: None (StatefulSet 必須)

[Service] openclaw-ui (LoadBalancer)
  └── 443 → 18789, openclaw.internal.onoe.dev

[NetworkPolicy] (4段階)
  ├── default-deny-all (ingress + egress 全拒否)
  ├── allow-dns → kube-system/kube-dns (UDP/TCP 53)
  ├── allow-external-http → 0.0.0.0/0 (except 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) on TCP 80/443
  └── allow-ui-ingress → TCP 18789
```

### リソース要件

| | requests | limits |
|---|---|---|
| CPU | 200m | 2 |
| Memory | 512Mi | 2Gi |

- Node.js プロセスの実測値: アイドル 300-500MB、通常 500MB-1GB、長時間セッション最大 1.9GB
- `NODE_OPTIONS=--max-old-space-size=1536` で V8 ヒープを 1.5GB にキャップ
- ワーカーノードは 6.8GB RAM のため、limits を 2Gi に制限

### 設定ファイル

#### openclaw.json (ConfigMap)

JSON5 形式。主要な設定:

| セクション | 設定内容 |
|---|---|
| `gateway` | port 18789, bind "lan", auth token, TLS enabled, Control UI |
| `channels.slack` | Socket Mode (WebSocket), dmPolicy "open" |
| `discovery` | mDNS off |
| `logging` | JSON 形式, info レベル |
| `tools` | fs workspaceOnly, exec allowlist, loopDetection |
| `session` | daily reset (04:00), idle 120min, 30d prune |
| `models.providers.anthropic` | apiKey, baseUrl, models (opus/sonnet/haiku) |
| `agents.defaults` | primary: sonnet-4-5, thinking: low, context: 200k, pruning, compaction, heartbeat |

秘匿値は `${VAR}` プレースホルダーで環境変数から解決 (OpenClaw ネイティブ機能)。

#### exec-approvals.json (ConfigMap, subPath マウント)

exec ツールの allowlist 設定。許可するコマンドを定義。

### ボリューム構成

| マウント先 | ソース | 用途 |
|---|---|---|
| `/home/node/.openclaw` | PVC 32Gi (ceph-block) | セッション、ワークスペース等の永続データ |
| `/config/openclaw.json` | ConfigMap | 設定ファイル (OPENCLAW_CONFIG_PATH で参照) |
| `/home/node/.openclaw/exec-approvals.json` | ConfigMap (subPath) | exec allowlist |
| `/tls` | Secret (cert-manager) | TLS 証明書 (将来の cert-manager 連携用に保持) |
| `/tmp` | emptyDir | 一時ファイル |

### TLS

- `gateway.tls.enabled: true` により自己生成証明書で wss:// リスニング
- cert-manager 証明書 (`/tls/tls.crt`, `/tls/tls.key`) はマウント済みだが、gateway は `cert`/`key` フィールドを無視して自己生成する
- cert-manager 証明書への切り替えは issue #12 で対応予定
- `.dev` gTLD は HSTS preload により HTTPS 必須

### セキュリティ

#### SecurityContext
- `runAsNonRoot: true`, `runAsUser: 1000` (node ユーザー)
- `privileged: false`, `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`, `seccompProfile: RuntimeDefault`

#### Pod レベル
- `automountServiceAccountToken: false` (K8s API アクセス不可)
- hostNetwork/hostPID/hostIPC なし

#### ネットワーク隔離
- Default deny all → DNS のみ許可 → 外部 HTTP/HTTPS のみ許可 (プライベートアドレス全拒否)
- 内部ネットワーク (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) への通信は全て遮断

#### exec ツール
- `tools.exec.security: "allowlist"` + `exec-approvals.json` で許可コマンドを限定
- 環境変数は個別に `secretKeyRef` で注入 (`envFrom` ではなく `env` を使用し、シェルからの全変数列挙を防止)

### 手動前提条件 (Git push 前)

#### 1. Slack Bot 作成

1. https://api.slack.com/apps で新規 App 作成
2. **Socket Mode** を有効化 → App-Level Token (`xapp-...`) を取得
3. **OAuth & Permissions** → Bot Token Scopes: `chat:write`, `im:history`, `im:read`, `im:write`
4. ワークスペースにインストール → Bot Token (`xoxb-...`) を取得

#### 2. Gateway Auth Token 生成

```bash
openssl rand -hex 32
```

#### 3. Anthropic API Key 取得

https://console.anthropic.com/ で API Key (`sk-ant-...`) を取得。

#### 4. Vault にシークレット格納

```bash
kubectl -n vault exec -it vault-0 -- sh
vault login
vault kv put secret/openclaw/config \
  slack_app_token="xapp-..." \
  slack_bot_token="xoxb-..." \
  llm_api_key="sk-ant-..." \
  gateway_auth_token="<step 2 で生成した値>"
exit
```

> Vault ポリシー `external-secrets` は `secret/data/*` の read を許可済みのため、追加設定は不要。

### デプロイ (ArgoCD 自動)

`k8s/argocd/apps/openclaw.yaml` が main にマージされると、ArgoCD が自動的に openclaw namespace にデプロイする。

### セットアップ (デプロイ後)

#### 1. Pod 起動確認

```bash
kubectl get pods -n openclaw
# → openclaw-0  1/1  Running

kubectl logs -n openclaw openclaw-0
# → "listening on wss://0.0.0.0:18789" が表示されること
# → "slack socket mode connected" が表示されること
```

#### 2. Control UI アクセス

```bash
open https://openclaw.internal.onoe.dev
```

> 自己署名証明書のため、ブラウザで証明書警告が出る。cert-manager の CA をシステムに信頼させるか、警告を許可して続行。

#### 3. Control UI ペアリング

1. Control UI にアクセス
2. Gateway Auth Token (手順 2 で生成した値) を入力してペアリング
3. `dangerouslyDisableDeviceAuth: true` のためデバイス認証はスキップされる

#### 4. Onboarding (オプション)

```bash
kubectl exec -it -n openclaw openclaw-0 -- openclaw onboard
```

対話形式でセットアップウィザードが起動。ConfigMap で設定済みの場合はスキップ可。

#### 5. ステータス確認

```bash
kubectl exec -n openclaw openclaw-0 -- openclaw status
```

確認ポイント:
- Gateway: `wss://` で listening
- Channels: Slack ON / OK
- Agents: model `claude-sonnet-4-5`
- Sessions: 表示されること

#### 6. Slack 動作確認

Slack で Bot に DM を送信し、応答があることを確認。

### 検証

```bash
# 1. Pod Running + Ready
kubectl get pods -n openclaw
# → 1/1 Running

# 2. ExternalSecret → Secret 生成確認
kubectl -n openclaw get externalsecret
# → SecretSynced

# 3. TLS 証明書発行確認
kubectl get certificate -n openclaw
# → Ready: True

# 4. DNS 解決
nslookup openclaw.internal.onoe.dev
# → 10.5.0.2

# 5. Control UI アクセス
curl -sk https://openclaw.internal.onoe.dev | head -5
# → HTML が返ること

# 6. DNS 疎通 (Pod 内)
kubectl exec -n openclaw openclaw-0 -- nslookup google.com
# → 正常解決

# 7. 外部疎通 (Slack API)
kubectl exec -n openclaw openclaw-0 -- wget -qO- https://api.slack.com/
# → レスポンスあり

# 8. 内部ネットワーク遮断
kubectl exec -n openclaw openclaw-0 -- wget -T 5 -qO- http://10.2.0.1
# → タイムアウト (NetworkPolicy で遮断)

# 9. PVC
kubectl get pvc -n openclaw
# → data-openclaw-0  32Gi  Bound
```

### トラブルシューティング

#### Pod が起動しない (startup probe 失敗)

```bash
# イベント確認
kubectl get events -n openclaw --field-selector involvedObject.name=openclaw-0 --sort-by='.lastTimestamp'

# ログ確認
kubectl logs -n openclaw openclaw-0
```

- `connection refused`: Gateway プロセスがまだ起動中。起動には 30-40 秒程度かかる
- `context deadline exceeded`: TLS ハンドシェイクがタイムアウト。probe の `timeoutSeconds` が短い可能性 (現在 5s)
- `http: server gave HTTP response to HTTPS client`: TLS が有効になっていない。`gateway.tls.enabled: true` を確認

#### ノードが NotReady になる

- OpenClaw の memory limit がノードの物理メモリを超えていないか確認
- 現在のワーカーノードは 6.8GB RAM。limit は 2Gi に設定済み
- `kubectl describe node <node>` の Allocated resources で overcommit を確認

#### ConfigMap 変更が反映されない

- ConfigMap は直接マウントのため、更新は自動反映される (kubelet の sync 間隔: 約 60 秒)
- ただし OpenClaw の設定変更は Pod 再起動が必要な場合がある:
  ```bash
  kubectl delete pod -n openclaw openclaw-0
  ```

### 既知の制限事項

- **TLS 証明書**: Gateway は `tls.cert`/`tls.key` フィールドを無視し自己生成証明書を使用する (issue #12)
- **`/healthz`, `/readyz`**: 専用のヘルスエンドポイントではなく、SPA の catch-all で HTTP 200 を返す
- **exec allowlist**: `exec-approvals.json` は subPath でマウント。ConfigMap 更新時は Pod 再起動が必要

## 関連ドキュメント

- [ネットワーク設計](../docs/design.md)
- [Catalyst ルータ設定](../router/catalyst-3560cx.conf)
- [VM 設定ファイル](../vm/)
- [ノード削減/HA 復帰手順](docs/node-scaling.md)
