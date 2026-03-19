# Homelab Kubernetes Cluster

A hybrid Kubernetes cluster combining a physical control plane with Proxmox-virtualized workers, managed through GitOps principles. Features GPU passthrough for AI/ML workloads.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                     │
│                                                             │
│  ┌──────────────┐     Control Plane (Physical)              │
│  │  x270-ctrl   │     Lenovo ThinkPad X270                  │
│  │  (ARM: CP)   │     kubeadm / etcd / API server           │
│  └──────┬───────┘                                           │
│         │                                                   │
│         ├──────────────────────────────────────┐            │
│         │          Proxmox VE (prox-nix)       │            │
│         │       AMD Ryzen 3900x · 64GB RAM     │            │
│         │                                      │            │
│  ┌──────┴───────┐  ┌──────────────┐  ┌─────────┴────┐       │
│  │ k8s-gpu-01   │  │k8s-worker-01 │  │k8s-worker-02 │       │
│  │ RTX 3090     │  │ General      │  │ General      │       │
│  │ (PCIe Pass)  │  │ Purpose      │  │ Purpose      │       │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  ┌──────────────┐                                           │
│  │  TrueNAS     │     NAS (NFS/SMB)                         │
│  │  Scale       │     Bulk Storage                          │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
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
| **Storage** | TrueNAS Scale (NFS), OpenEBS ZFS-LocalPV (planned) |
| **Networking** | MetalLB (L2 mode) |

## Repository Structure

```
├── apps/                  # Application workloads
│   ├── glance/            # Dashboard
│   └── ollama/            # LLM inference (GPU)
├── clusters/              # Cluster-wide kustomization
└── infra/                 # Infrastructure components
    └── nvidia-device-plugin/  # GPU scheduling for Kubernetes
```

## GPU Passthrough

The RTX 3090 is passed from the Proxmox host to the `k8s-gpu-01` VM using VFIO-PCI, enabling GPU-accelerated workloads in Kubernetes:

- **Host**: IOMMU (AMD-Vi) enabled, GPU bound to `vfio-pci` driver
- **VM**: OVMF/q35 machine type with PCIe passthrough
- **K8s**: NVIDIA device plugin exposes `nvidia.com/gpu` as a schedulable resource

## Planned Services

- **Immich** - Self-hosted photo backup with ML-powered search
- **Ollama** - Local LLM inference on RTX 3090
- **Grafana + Prometheus** - Cluster observability
- **Cert Manager** - Automated TLS certificates
