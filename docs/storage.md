# Homelab Storage Plan (PoC → Scale‑out)

> **Goal:** Run Vault, NetBox, PowerDNS, OpenStack, and lab apps *today* on existing hardware **without data‑loss risk** and **with a zero‑reinstall path** to bigger disks, more nodes, and 10 GbE.

---

## 1. Hardware‐Reality Snapshot (Day 1)

| Node / VM          | Role                     | Storage Tier                                 | Capacity         | Main Workloads                                                 |
| ------------------ | ------------------------ | -------------------------------------------- | ---------------- | -------------------------------------------------------------- |
| **GMKtec M5 host** | Proxmox VE hypervisor    | `local‑fast` (ZFS mirror, nvme0n1 + nvme1n1) |  ≈ 1 TB          | ISO cache, Ansible repo, qcow2 templates                       |
| **`vm‑ceph‑rook`** | 1‑node Ceph SAN          | 2 × 100 GB raw OSD zvols + 20 GB NVMe DB     |  ≈ 200 GB usable | OpenStack Cinder/Glance, K8s PVCs (`ceph‑block`, `ceph‑files`) |
| **3 × Pi 5**       | k3s control‑plane + apps | USB‑SSD Longhorn (`longhorn`, replica = 3)   |  ≈ 500 GB total  | **Vault (raft), NetBox + PG, PowerDNS + PG**, Grafana/Loki     |

Backups → MinIO S3 bucket on Pis via **Velero + restic** & `zfs send` snapshots.

---

## 2. Storage Classes (Kubernetes)

```yaml
local-fast   # hostPath → ZFS mirror on GMKtec
ceph-block   # RBD (rook-ceph), size=1 → size=3 later
ceph-files   # CephFS (rook-ceph)
longhorn     # USB‑SSD replica=3 on Pi nodes
ephemeral    # tmpfs / emptyDir
```

*Namespaces annotate their default SC* so Vault/NetBox stay on `longhorn` until you flip a single annotation.

---

## 3. Upgrade Path (Zero‑Reinstall)

| Stage                               | Action                                  | What Changes                      | What *Doesn’t* Change        |
| ----------------------------------- | --------------------------------------- | --------------------------------- | ---------------------------- |
| **Add 10 GbE**                      | Dual‑port NIC → `vmbr1`, MTU 9000       | Ceph latency ⇣, throughput ⇡      | No PVC edits, no IP moves    |
| **Ceph Node #2 (VM or bare‑metal)** | `ceph orch daemon add osd …`            | Pool `size`→2                     | Apps & SCs untouched         |
| **Ceph Node #3 (bare‑metal)**       | Add NVMe + HDD OSDs                     | Pool `size`→3, MDS‑HA             | Same SCs, same data          |
| **Migrate Vault/NetBox to Ceph**    | `velero backup`, patch SC, restore      | Data now on `ceph-block` SSD tier | Helm chart unchanged         |
| **Retire Longhorn (optional)**      | Remove default flag, delete HelmRelease | Pi PVCs re‑provision on Ceph      | App manifests stay identical |

---

## 4. Backup & DR

```text
- Hourly ZFS snapshots  → restic → MinIO → S3/Glacier
- Nightly RBD snapshots → Velero  → MinIO → S3/Glacier
- Quarterly restore‑day playbook:  spin‑up tmp ns → Velero restore → health check
```

---

## 5. Known Risks & Mitigations

| Risk                           | Current Mitigation                  | Future Fix                          |
| ------------------------------ | ----------------------------------- | ----------------------------------- |
| Single‑node Ceph = SPOF        | Nightly RBD snapshots off‑cluster   | Add OSD node #2/#3; pool `size` ≥ 2 |
| USB‑SSD endurance & power loss | UPS on Pi stack; 3‑replica Longhorn | Migrate to Ceph after 10 GbE        |
| 1 GbE bottleneck for image IO  | OK for PoC; monitor fio & Ceph dash | 10 GbE NIC + storage VLAN           |
| GMKtec RAM pressure            | Ceph VM balloon limits, swap alert  | 64 GB RAM upgrade or extra host     |

---

## 6. Day‑1 Task Checklist

* [ ] **Install Proxmox VE 8** on ZFS mirror
* [ ] Provision `vm‑ceph‑rook`, `vm‑openstack‑ctrl`, `vm‑bastion`, `vm‑nova‑test`
* [ ] `rook-ceph` single‑node install (pool `size=1`)
* [ ] Deploy Longhorn on Pi nodes; label + taint appropriately
* [ ] Create StorageClass YAMLs & namespace annotations via Flux
* [ ] Configure Velero → MinIO (Pi cluster)
* [ ] Run fio baseline on `longhorn`, `ceph-block`, `local-fast`

---

## 7. Future Shopping List

1. **APC or CyberPower 600–850 VA UPS** (Pis + GMKtec)
2. **Dual‑port 10 GbE SFP+ NIC** (Intel X520 or Mellanox ConnectX‑3)
3. **Two RM22‑312 bare‑metal OSD nodes** (NVMe cache + 4 × HDD)
4. **64 GB SO‑DIMM kit** for GMKtec if Ceph & OpenStack RAM creep

---

*Document version 1.0 • Last updated 2025‑05‑03*
