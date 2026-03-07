# K8s ノードスケーリングガイド

## 概要

Proxmox ノード (mandolin1-3) のメモリリソース不足のため、K8s ノード数を 6 (3 CP + 3 Worker) から 3 (1 CP + 2 Worker) に削減した。

各物理マシンに 1 VM を残し、停止した VM と Talos パッチファイルは将来の HA 復帰のため保持している。

### 現在の構成 (縮退)

| ノード | IP | 物理マシン | 役割 | 状態 |
|--------|-----|-----------|------|------|
| k8s-cp-01 | 10.2.0.66 | mandolin1 | control-plane | 稼働中 |
| k8s-cp-02 | 10.2.0.67 | mandolin2 | control-plane | 停止中 |
| k8s-cp-03 | 10.2.0.68 | mandolin3 | control-plane | 停止中 |
| k8s-worker-01 | 10.2.0.69 | mandolin1 | worker | 停止中 |
| k8s-worker-02 | 10.2.0.70 | mandolin2 | worker | 稼働中 |
| k8s-worker-03 | 10.2.0.71 | mandolin3 | worker | 稼働中 |

### HA 構成との差分

| コンポーネント | HA (6 ノード) | 縮退 (3 ノード) |
|---------------|-------------|-----------------|
| Vault | 3 replicas (HA Raft) | 1 replica (standalone) |
| k8s-gateway | 3 replicas | 2 replicas |
| node-reboot | 6 CronJobs | 削除済み |
| etcd | 3 メンバー | 1 メンバー |

## ノード削減手順 (6 → 3)

### Phase 1: Vault HA → standalone

```bash
# 1. StatefulSet を 1 replica にスケールダウン (vault-1, vault-2 の Pod を停止)
#    retry_join により Pod が生存していると Raft から削除しても自動で再参加するため、
#    先に Pod を停止する必要がある
kubectl -n vault scale statefulset vault --replicas=1

# 2. vault-1, vault-2 の Pod が完全に消えるまで待機
kubectl -n vault get pods -w
#    Terminating が長引く場合は強制削除:
#    kubectl -n vault delete pod vault-1 --force --grace-period=0
#    kubectl -n vault delete pod vault-2 --force --grace-period=0
```

vault-0 のみになったら、Raft のリカバリを行う。
vault-0 が standby のまま active にならない場合 (quorum 喪失)、`peers.json` による強制リカバリが必要。

```bash
# 3. vault-0 の node ID を確認
#    事前に vault operator raft list-peers で確認しておく
#    (例: 60a2ef8d-5608-3e92-110e-a4f436571947)

# 4. Raft リカバリファイルを作成 (vault-0 のみの単一ノードクラスタに強制変更)
kubectl -n vault exec vault-0 -- sh -c 'cat > /vault/data/raft/peers.json << EOF
[
  {
    "id": "<vault-0-node-id>",
    "address": "vault-0.vault-internal:8201",
    "non_voter": false
  }
]
EOF'

# 5. Pod 再起動 (peers.json を読み込んで単一ノードクラスタとして起動)
kubectl -n vault delete pod vault-0

# 6. vault-0 が active になったことを確認
kubectl -n vault exec vault-0 -- vault status
# → HA Mode: active であること

# 7. Raft peer が vault-0 のみであることを確認
kubectl -n vault exec vault-0 -- sh -c \
  'vault login token=$(cat /vault/root-token/token) && vault operator raft list-peers'
# → vault-0 のみ表示されること
```

Git 変更を反映し、不要なリソースを削除する。

```bash
# 8. Git push (values.yaml: replicas: 1) → ArgoCD sync

# 9. 不要な PVC を削除
kubectl -n vault delete pvc data-vault-1 data-vault-2
```

### Phase 2: ワークロード退避

```bash
# 1. 削除ノードを cordon
kubectl cordon k8s-cp-02
kubectl cordon k8s-cp-03
kubectl cordon k8s-worker-01

# 2. drain (DaemonSet は無視)
kubectl drain k8s-cp-02 --ignore-daemonsets --delete-emptydir-data
kubectl drain k8s-cp-03 --ignore-daemonsets --delete-emptydir-data
kubectl drain k8s-worker-01 --ignore-daemonsets --delete-emptydir-data
```

### Phase 3: etcd メンバー削除

```bash
# 1. etcd メンバー ID を確認
talosctl --nodes 10.2.0.66 etcd members
# → ID (16桁の16進数), HOSTNAME, PEER URLS が表示される
#    例: ac116a9530506c52  k8s-cp-02  https://10.2.0.67:2380
#        1ddcd87332950850  k8s-cp-03  https://10.2.0.68:2380

# 2. cp-02, cp-03 をメンバー ID で etcd から削除
#    注意: ホスト名や IP ではなく ID を指定する
talosctl --nodes 10.2.0.66 etcd remove-member <cp-02-member-id>
talosctl --nodes 10.2.0.66 etcd remove-member <cp-03-member-id>

# 3. 確認
talosctl --nodes 10.2.0.66 etcd members
# → k8s-cp-01 のみ
```

### Phase 4: ノード削除

```bash
# 1. K8s からノード削除
kubectl delete node k8s-cp-02
kubectl delete node k8s-cp-03
kubectl delete node k8s-worker-01

# 2. Proxmox で VM 停止 (削除しない: 将来 HA 復帰用)
# mandolin2: qm stop 102002 (k8s-cp-02)
# mandolin3: qm stop 102003 (k8s-cp-03)
# mandolin1: qm stop 102004 (k8s-worker-01)
```

### Phase 5: Git push & 確認

```bash
# 1. node-reboot 削除 + Vault/k8s-gateway 更新を push

# 2. ArgoCD sync 確認
kubectl -n argocd get applications

# 3. 全 Pod 確認
kubectl get pods --all-namespaces
```

## HA 構成への復帰手順 (3 → 6)

### Phase 1: 停止 VM の起動

```bash
# Proxmox で VM 起動
# mandolin2: qm start 102002 (k8s-cp-02)
# mandolin3: qm start 102003 (k8s-cp-03)
# mandolin1: qm start 102004 (k8s-worker-01)

# VM 起動待ち
ping -c 3 10.2.0.67
ping -c 3 10.2.0.68
ping -c 3 10.2.0.69
```

### Phase 2: Talos 再参加

既存 VM には Talos がインストール済み。ディスクが初期化された場合は `k8s/talos/` から machine config を再適用する。

```bash
# 必要に応じて machine config を再適用
cd k8s/talos
talosctl apply-config -n 10.2.0.67 -f k8s-cp-02.yaml
talosctl apply-config -n 10.2.0.68 -f k8s-cp-03.yaml
talosctl apply-config -n 10.2.0.69 -f k8s-worker-01.yaml
```

### Phase 3: etcd & K8s 参加

CP ノードは etcd と K8s に再参加する必要がある。

```bash
# Worker ノードは apply-config 後に自動で K8s に参加する
# CP ノードは etcd bootstrap が必要な場合がある

# ノード確認
kubectl get nodes
# → 6 ノードが Ready

# etcd 確認
talosctl --nodes 10.2.0.66 etcd members
# → 3 メンバー
```

### Phase 4: Vault HA 復帰

```bash
# 1. k8s/vault/values.yaml を更新:
#    - replicas: 1 → replicas: 3
#    - vault-1, vault-2 の retry_join ブロックをアンコメント
#    - affinity ブロックをアンコメント
#    - standalone コメントを削除

# 2. Git push → ArgoCD sync

# 3. Raft クラスタ確認
kubectl -n vault exec vault-0 -- vault operator raft list-peers
# → 3 voters
```

### Phase 5: k8s-gateway replicas 復帰

```bash
# 1. k8s/k8s-gateway/values.yaml を更新:
#    - replicaCount: 2 → replicaCount: 3
#    - 縮退コメントを削除

# 2. Git push → ArgoCD sync

# 3. 確認
kubectl -n k8s-gateway get pods -o wide
# → 3 replicas が異なるノードで稼働
```

### Phase 6: node-reboot 再作成 (任意)

定期リブートが再度必要な場合は以下を再作成する:
- `k8s/argocd/apps/node-reboot.yaml`
- `k8s/node-reboot/manifests/` (CronJobs + ExternalSecret)

元の設定は git 履歴から参照可能。

## 検証チェックリスト

### ノード削減後 (6 → 3)

- [ ] `kubectl get nodes` → 3 ノード Ready (k8s-cp-01, k8s-worker-02, k8s-worker-03)
- [ ] `kubectl -n argocd get applications` → 全て Synced/Healthy、node-reboot なし
- [ ] `kubectl -n vault get pods` → vault-0 のみ Running
- [ ] `kubectl -n vault exec vault-0 -- vault status` → Sealed=false
- [ ] `kubectl -n k8s-gateway get pods` → 2 replicas Running
- [ ] `kubectl get pods --all-namespaces` → 全 Pod Running
- [ ] `nslookup argocd.internal.onoe.dev` → 正常解決
- [ ] `kubectl -n openclaw get pods` → 1/1 Ready

### HA 復帰後 (3 → 6)

- [ ] `kubectl get nodes` → 6 ノード Ready
- [ ] `talosctl --nodes 10.2.0.66 etcd members` → 3 メンバー
- [ ] `kubectl -n vault exec vault-0 -- vault operator raft list-peers` → 3 voters
- [ ] `kubectl -n k8s-gateway get pods` → 3 replicas Running
- [ ] `kubectl get pods --all-namespaces` → 全 Pod Running
