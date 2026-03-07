# K8s Node Scaling Guide

## Overview

Memory resource constraints on Proxmox nodes (mandolin1-3) require reducing K8s nodes from 6 (3 CP + 3 Worker) to 3 (1 CP + 2 Worker).

Each physical machine retains one VM. Stopped VMs and Talos patch files are preserved for future HA restoration.

### Current Configuration (Reduced)

| Node | IP | Physical Host | Role | Status |
|------|-----|---------------|------|--------|
| k8s-cp-01 | 10.2.0.66 | mandolin1 | control-plane | Running |
| k8s-cp-02 | 10.2.0.67 | mandolin2 | control-plane | Stopped |
| k8s-cp-03 | 10.2.0.68 | mandolin3 | control-plane | Stopped |
| k8s-worker-01 | 10.2.0.69 | mandolin1 | worker | Stopped |
| k8s-worker-02 | 10.2.0.70 | mandolin2 | worker | Running |
| k8s-worker-03 | 10.2.0.71 | mandolin3 | worker | Running |

### Changes from HA Configuration

| Component | HA (6 nodes) | Reduced (3 nodes) |
|-----------|-------------|-------------------|
| Vault | 3 replicas (HA Raft) | 1 replica (standalone) |
| k8s-gateway | 3 replicas | 2 replicas |
| node-reboot | 6 CronJobs | Removed |
| etcd | 3 members | 1 member |

## Node Reduction Procedure (6 → 3)

### Phase 1: Vault HA → Standalone

```bash
# 1. Confirm vault-0 is the leader
kubectl -n vault exec vault-0 -- vault operator raft list-peers

# 2. Remove vault-1, vault-2 from Raft peers
kubectl -n vault exec vault-0 -- vault operator raft remove-peer vault-1
kubectl -n vault exec vault-0 -- vault operator raft remove-peer vault-2

# 3. Git push (values.yaml: replicas: 1) → ArgoCD sync

# 4. Delete unused PVCs
kubectl -n vault delete pvc data-vault-1 data-vault-2
```

### Phase 2: Workload Evacuation

```bash
# 1. Cordon nodes to be removed
kubectl cordon k8s-cp-02
kubectl cordon k8s-cp-03
kubectl cordon k8s-worker-01

# 2. Drain (ignore DaemonSets)
kubectl drain k8s-cp-02 --ignore-daemonsets --delete-emptydir-data
kubectl drain k8s-cp-03 --ignore-daemonsets --delete-emptydir-data
kubectl drain k8s-worker-01 --ignore-daemonsets --delete-emptydir-data
```

### Phase 3: etcd Member Removal

```bash
# 1. Remove cp-02, cp-03 from etcd
talosctl --nodes 10.2.0.66 etcd remove-member k8s-cp-02
talosctl --nodes 10.2.0.66 etcd remove-member k8s-cp-03

# 2. Verify
talosctl --nodes 10.2.0.66 etcd members
# → k8s-cp-01 only
```

### Phase 4: Node Deletion

```bash
# 1. Delete nodes from K8s
kubectl delete node k8s-cp-02
kubectl delete node k8s-cp-03
kubectl delete node k8s-worker-01

# 2. Stop VMs in Proxmox (do NOT delete — preserved for HA restoration)
# mandolin2: qm stop 102002 (k8s-cp-02)
# mandolin3: qm stop 102003 (k8s-cp-03)
# mandolin1: qm stop 102004 (k8s-worker-01)
```

### Phase 5: Git Push & Verification

```bash
# 1. Push node-reboot removal + Vault/k8s-gateway updates

# 2. Verify ArgoCD sync
kubectl -n argocd get applications

# 3. Verify all pods
kubectl get pods --all-namespaces
```

## HA Restoration Procedure (3 → 6)

### Phase 1: Start Stopped VMs

```bash
# Start VMs in Proxmox
# mandolin2: qm start 102002 (k8s-cp-02)
# mandolin3: qm start 102003 (k8s-cp-03)
# mandolin1: qm start 102004 (k8s-worker-01)

# Wait for VMs to boot and get IPs
ping -c 3 10.2.0.67
ping -c 3 10.2.0.68
ping -c 3 10.2.0.69
```

### Phase 2: Talos Bootstrap (Re-join)

Existing VMs should have Talos installed. If disks were wiped, re-apply machine configs from `k8s/talos/`.

```bash
# Re-apply machine config if needed
cd k8s/talos
talosctl apply-config -n 10.2.0.67 -f k8s-cp-02.yaml
talosctl apply-config -n 10.2.0.68 -f k8s-cp-03.yaml
talosctl apply-config -n 10.2.0.69 -f k8s-worker-01.yaml
```

### Phase 3: etcd & K8s Join

Control plane nodes need to rejoin etcd and K8s.

```bash
# Worker nodes should auto-join the cluster after apply-config
# Control plane nodes may need etcd bootstrap

# Verify nodes
kubectl get nodes
# → 6 nodes should be Ready

# Verify etcd
talosctl --nodes 10.2.0.66 etcd members
# → 3 members
```

### Phase 4: Vault HA Restoration

```bash
# 1. Update k8s/vault/values.yaml:
#    - replicas: 1 → replicas: 3
#    - Uncomment retry_join blocks for vault-1 and vault-2
#    - Uncomment affinity block
#    - Remove standalone comments

# 2. Git push → ArgoCD sync

# 3. Verify Raft cluster
kubectl -n vault exec vault-0 -- vault operator raft list-peers
# → 3 voters
```

### Phase 5: k8s-gateway Replicas Restoration

```bash
# 1. Update k8s/k8s-gateway/values.yaml:
#    - replicaCount: 2 → replicaCount: 3
#    - Remove reduced comments

# 2. Git push → ArgoCD sync

# 3. Verify
kubectl -n k8s-gateway get pods -o wide
# → 3 replicas on different nodes
```

### Phase 6: node-reboot Re-creation (Optional)

If periodic reboot is needed again, re-create:
- `k8s/argocd/apps/node-reboot.yaml`
- `k8s/node-reboot/manifests/` (CronJobs + ExternalSecret)

Refer to git history for the original configuration.

## Verification Checklist

### After Node Reduction (6 → 3)

- [ ] `kubectl get nodes` → 3 nodes Ready (k8s-cp-01, k8s-worker-02, k8s-worker-03)
- [ ] `kubectl -n argocd get applications` → All Synced/Healthy, no node-reboot
- [ ] `kubectl -n vault get pods` → vault-0 only, Running
- [ ] `kubectl -n vault exec vault-0 -- vault status` → Sealed=false
- [ ] `kubectl -n k8s-gateway get pods` → 2 replicas Running
- [ ] `kubectl get pods --all-namespaces` → All pods Running
- [ ] `nslookup argocd.internal.onoe.dev` → Resolves correctly
- [ ] `kubectl -n openclaw get pods` → 1/1 Ready

### After HA Restoration (3 → 6)

- [ ] `kubectl get nodes` → 6 nodes Ready
- [ ] `talosctl --nodes 10.2.0.66 etcd members` → 3 members
- [ ] `kubectl -n vault exec vault-0 -- vault operator raft list-peers` → 3 voters
- [ ] `kubectl -n k8s-gateway get pods` → 3 replicas Running
- [ ] `kubectl get pods --all-namespaces` → All pods Running
