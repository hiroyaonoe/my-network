# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository manages infrastructure design documentation and configuration files for a home VM platform:
- **Proxmox VE** 3-node cluster (`mandolin`) with **Ceph** distributed storage
- **Catalyst WS-C3560CX-8PC-S** as L3 switch and local router
- VLAN-segmented networking for VM isolation

## Directory Structure

Configuration files mirror actual filesystem paths on target systems:

```
.
├── docs/
│   └── design.md                    # Complete infrastructure design document
├── router/
│   └── catalyst-3560cx.conf         # Catalyst IOS configuration
├── pm/mandolin/                     # Proxmox cluster configs
│   ├── mandolin1/etc/network/
│   │   └── interfaces               # → /etc/network/interfaces on mandolin1
│   ├── mandolin2/etc/network/
│   │   └── interfaces               # → /etc/network/interfaces on mandolin2
│   ├── mandolin3/etc/network/
│   │   └── interfaces               # → /etc/network/interfaces on mandolin3
│   ├── etc/ceph/
│   │   └── ceph.conf                # → /etc/ceph/ceph.conf (all nodes)
│   └── README.md
└── vm/                              # VM-specific configurations
    └── <vm-name>/                   # Per-VM directory (created as needed)
```

**Path mapping principle**: Files under `pm/mandolin/<node>/` mirror the filesystem structure on that node. To deploy, copy the entire `etc/` subtree to the target node's root.

## Naming Conventions

- **Cluster name**: `mandolin`
- **Node names**: `mandolin1`, `mandolin2`, `mandolin3`
- **IP addressing**:
  - Management (VLAN 10): `172.16.0.11-13`
  - Ceph cluster (VLAN 20): `172.16.1.11-13`
  - VM networks: `10.0.0.0/20` subdivided by VLAN

## Critical VLAN Design Rules

VM VLANs (100-227) use a **mathematical allocation formula** to pack /27 subnets sequentially within 10.0.0.0/20:

```
VLAN (100 + N) → 10.0.(N/8).(N%8*32)/27
Gateway = subnet first IP + 1
Subnet mask = 255.255.255.224 (/27)
```

Examples:
- VLAN 100 (N=0): 10.0.0.0/27, GW 10.0.0.1
- VLAN 101 (N=1): 10.0.0.32/27, GW 10.0.0.33
- VLAN 108 (N=8): 10.0.1.0/27, GW 10.0.1.1

**When adding new VLANs**:
1. Calculate subnet using the formula above
2. Add VLAN definition to `router/catalyst-3560cx.conf`
3. Add SVI (interface Vlan<N>) with calculated gateway IP
4. Update `docs/design.md` section 3.2 table
5. No changes needed to trunk configs (already range-specified: `100-227`)

## Key Architecture Decisions

1. **Range-specified trunks**: Catalyst uses `allowed vlan 10,20,100-227` and Proxmox uses `bridge-vids 20 100-227`. This allows VLAN additions without modifying trunk/bridge configs.

2. **Network constraints**: The Catalyst switch has 1GbE ports, so 2.5GbE NICs negotiate down to 1Gbps. LACP bonding provides 2Gbps aggregate (hash-based distribution).

3. **VLAN-aware bridging**: Proxmox uses a single VLAN-aware bridge (`vmbr0`). VMs specify VLAN tags at the hypervisor level (via Proxmox UI or `qm set`), not inside the guest OS.

4. **Ceph networks**: Public network (172.16.0.0/24) for client I/O, Cluster network (172.16.1.0/24) for OSD replication, physically separated via VLAN 20.

## Configuration Management

### Router Configuration

- **File**: `router/catalyst-3560cx.conf`
- Apply via Catalyst CLI (copy-paste or `copy tftp: running-config`)
- After changes: `copy running-config startup-config`

### Proxmox Node Configuration

- **Files**: `pm/mandolin/mandolin{1,2,3}/etc/network/interfaces`
- Apply: `scp -r mandolin1/etc root@172.16.0.11:/`
- Then: `ssh root@172.16.0.11 'ifreload -a'`
- **Warning**: Network changes can break SSH. Use console access or test carefully.

### Ceph Configuration

- **File**: `pm/mandolin/etc/ceph/ceph.conf`
- Deploy to all nodes: `scp -r etc/ceph root@172.16.0.11:/etc/`
- Initial setup via Proxmox CLI (`pveceph install`, `pveceph mon create`, etc.)

### VM Configuration

- **Directory**: `vm/<vm-name>/`
- Create per-VM subdirectories as needed
- Include: VM config JSON, cloud-init YAML, README with purpose
- VLAN tags are specified in Proxmox VM config, not in guest OS

## Common Operations

### Add a new VM VLAN

```bash
# 1. Calculate N from desired VLAN ID: N = VLAN_ID - 100
# 2. Calculate subnet: 10.0.(N/8).(N%8*32)/27
# 3. Add to router/catalyst-3560cx.conf:
vlan <VLAN_ID>
 name vm-<purpose>
interface Vlan<VLAN_ID>
 ip address 10.0.X.Y 255.255.255.224
 no shutdown

# 4. Update docs/design.md section 3.2
# 5. Commit changes
```

### Create a VM

```bash
# Via CLI (specify VLAN tag):
qm create <vmid> --name <vm-name> --memory 2048 --cores 2 \
  --net0 virtio,bridge=vmbr0,tag=<VLAN_ID>

# Via Web UI: Hardware → Network Device → set VLAN Tag
```

See `docs/design.md` sections 7.1 (VLAN addition) and 7.2 (VM creation) for detailed procedures.

## Editing Guidelines

- **VLAN additions**: Always use the N-based formula to maintain sequential packing
- **IP address changes**: Update all references across `docs/`, `router/`, and `pm/` configs
- **Catalyst config**: All VM VLAN SVIs use `/27` netmask (`255.255.255.224`)
- **Node configs**: When editing `pm/mandolin/mandolinN/etc/*`, maintain the actual filesystem path structure
- Maintain consistency between `docs/design.md` (documentation) and config files
- When adding nodes: Follow the `mandolinN` naming pattern and IP sequence
