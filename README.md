# Homelab Kubernetes Cluster

A hybrid Kubernetes cluster combining a physical control plane with Proxmox-virtualized workers, managed through GitOps principles. Features GPU passthrough for AI/ML workloads.

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
│                                                              │
│  ┌──────────────┐     Control Plane (Physical)               │
│  │  x270-ctrl   │     Lenovo ThinkPad X270                   │
│  │              │     kubeadm / etcd / API server            │
│  └──────┬───────┘                                            │
│         │                                                    │
│         ├───────────────────────────────────────┐            │
│         │          Proxmox VE (prox-nix)        │            │
│         │       AMD Ryzen 3900x · 64GB RAM      │            │
│         │                                       │            │
│  ┌──────┴───────┐  ┌──────────────┐  ┌──────────┴───┐        │
│  │ k8s-gpu-01   │  │k8s-worker-01 │  │k8s-worker-02 │        │
│  │ RTX 3090     │  │ General      │  │ General      │        │ 
│  │ (PCIe Pass)  │  │ Purpose      │  │ Purpose      │        │
│  └──────┬───────┘  └──────────────┘  └──────────────┘        │
│         │                                                    │
│         │ NFS (model storage)                                │
│  ┌──────┴───────┐                                            │
│  │ ZFS on NVMe  │     1TB NVMe — per-app ZFS datasets        │
│  │ (prox-nix)   │     exported via NFS to cluster            │
│  └──────────────┘                                            │
│                                                              │
│  ┌──────────────┐                                            │
│  │  TrueNAS     │     NAS (NFS/SMB)                          │
│  │  Scale       │     Bulk Storage                           │
│  └──────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Hypervisor** | Proxmox VE 8.x |
| **Provisioning** | Terraform (bpg/proxmox provider) |
| **OS** | Ubuntu 22.04 (cloud-init golden image) |
| **Orchestration** | Kubernetes v1.31 (kubeadm) |
| **GPU** | NVIDIA RTX 3090 via VFIO-PCI passthrough |
| **GPU Runtime** | nvidia-container-toolkit + k8s-device-plugin |
| **Storage** | ZFS on NVMe (NFS export), TrueNAS Scale (NFS) |
| **Networking** | MetalLB (L2 mode) |

## Repository Structure

```
├── apps/                      # Application workloads
│   ├── glance/                # Dashboard
│   └── ollama/                # LLM inference (GPU)
├── clusters/                  # Cluster-wide kustomization
└── infra/                     # Infrastructure components
    ├── nfs-storage/           # NFS-backed persistent storage (ZFS)
    └── nvidia-device-plugin/  # GPU scheduling for Kubernetes
```

## GPU Passthrough

The RTX 3090 is passed from the Proxmox host to the `k8s-gpu-01` VM using VFIO-PCI, enabling GPU-accelerated workloads in Kubernetes:

- **Host**: IOMMU (AMD-Vi) enabled, GPU bound to `vfio-pci` driver
- **VM**: OVMF/q35 machine type with PCIe passthrough
- **K8s**: NVIDIA device plugin exposes `nvidia.com/gpu` as a schedulable resource

## Storage

Per-application ZFS datasets on a 1TB NVMe drive, exported via NFS from the Proxmox host to the cluster. Each app gets an isolated dataset for independent snapshots and quota management.

```
zfs-nvme-data/k8s-storage/
└── ollama/     # LLM model storage (200Gi PV)
```

## Running Services

### Ollama (LLM Inference)
Local large language model inference running on the RTX 3090 (24GB VRAM). Deployed as a Kubernetes workload with GPU scheduling via `nvidia.com/gpu` resource requests. Models are stored on NFS-backed ZFS storage. Accessible across the LAN via MetalLB LoadBalancer on port `11434`.

## Planned Services

- **Immich** - Self-hosted photo backup with ML-powered search
- **Grafana + Prometheus** - Cluster observability
- **Cert Manager** - Automated TLS certificates
