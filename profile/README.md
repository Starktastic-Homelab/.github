<div align="center">

# Starktastic Homelab

### Production-Grade Infrastructure, Automated End-to-End

*A fully automated Kubernetes homelab built by **Ben Faingold** — from bare metal to running services in a single pipeline*

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Packer](https://img.shields.io/badge/Packer-02A8EF?style=for-the-badge&logo=packer&logoColor=white)](https://www.packer.io/)
[![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)](https://www.proxmox.com/)

</div>

---

## What Is This?

This organization contains a **complete, production-grade infrastructure platform** running on a single server in my home. Every layer — from VM image baking to application deployment — is defined as code, version-controlled, and automated through CI/CD pipelines.

It's not a toy setup. It runs **60+ self-hosted services** across media streaming, home automation, document management, and operations tooling — all behind SSO, intrusion detection, automated TLS, and full observability. And it can be **rebuilt from scratch, unattended**, by merging a single PR.

---

## The Pipeline

The crown jewel of this project is the **end-to-end automation pipeline** — four repositories that chain together through GitHub Actions, each triggering the next:

```mermaid
flowchart LR
    subgraph "1 · Image Factory"
        P["📦 Packer\nBuild immutable\nDebian template"]
    end

    subgraph "2 · VM Provisioning"
        T["🏗️ Terraform\nClone template into\ncluster VMs"]
    end

    subgraph "3 · Cluster Setup"
        A["⚙️ Ansible\nInstall K3s + HA\nBootstrap ArgoCD"]
    end

    subgraph "4 · Application Layer"
        K["☸️ Apps\n60+ services via\nGitOps reconciliation"]
    end

    P -->|"manifest PR\n(auto-created)"| T
    T -->|"repository_dispatch\n(auto-triggered)"| A
    A -->|"ArgoCD bootstrap\n(App-of-Apps)"| K

    style P fill:#02A8EF,color:#fff
    style T fill:#7B42BC,color:#fff
    style A fill:#EE0000,color:#fff
    style K fill:#326CE5,color:#fff
```

**How it works:**

1. **Packer** builds a hardened Debian VM template on Proxmox with Intel GPU drivers, cloud-init, and modern networking — then pushes a manifest as a PR to the Terraform repo
2. **Terraform** clones that template into master and worker VMs with dual-NIC networking and GPU passthrough — then triggers Ansible via `repository_dispatch`
3. **Ansible** installs K3s with HA (Kube-VIP), pre-seeds secrets, restores TLS certificates from backup, and deploys ArgoCD with SSO — which immediately points at the Apps repo
4. **ArgoCD** takes over and reconciles the entire application layer through a 4-phase ordered deployment, pulling from the Apps repo as the single source of truth

The result: merge a Packer PR → a fully operational cluster with 60+ running services deploys itself, hands-free.

---

## Architecture Overview

```mermaid
flowchart TB
    subgraph hw["Hardware · Intel i7-12700K · 128 GB RAM"]
        PVE["Proxmox VE"]

        subgraph cluster["K3s Cluster (3 nodes)"]
            CP["Control Plane\n1× Master · Kube-VIP HA"]
            WK["Worker Pool\n2× Workers · Intel iGPU SR-IOV"]
        end

        subgraph storage["Storage"]
            ZFS["8×14TB HDDs\n2× RAIDz1\nMedia + Backups"]
            NVME_PV["NVMe SSD\nPersistent Volumes\n(NFS to cluster)"]
            NVME_VM["NVMe SSD\nVM Disks"]
        end
    end

    subgraph platform["Platform Layer"]
        TRAEFIK["Traefik\nIngress + TLS"]
        AUTH["Authentik\nSSO (OIDC/LDAP)"]
        CS["CrowdSec\nIntrusion Detection"]
        CERT["cert-manager\nLet's Encrypt"]
        SS["Sealed Secrets\nGitOps-safe encryption"]
        MLB["MetalLB\nL2 Load Balancer"]
    end

    subgraph apps["Application Layer (60+ services)"]
        HA["🏠 Home Automation\nHA · MQTT · Zigbee"]
        MEDIA["🎬 Media\nStreaming · Acquisition\nGPU Transcoding"]
        OPS["🔧 Operations\nMonitoring · Docs\nProductivity"]
    end

    subgraph observe["Observability"]
        PROM["Prometheus"] --- GRAF["Grafana"]
        LOKI["Loki"] --- GRAF
        TEMPO["Tempo"] --- GRAF
        GRAF --- NTFY["ntfy Alerts"]
    end

    PVE --> cluster
    cluster --> platform
    platform --> apps
    platform --> observe
    storage -.-> cluster

    style hw fill:#3C3C3C,color:#fff
    style platform fill:#7B42BC,color:#fff
    style apps fill:#326CE5,color:#fff
    style observe fill:#F46800,color:#fff
```

---

## Hardware

| Component | Specification |
|-----------|--------------|
| **CPU** | Intel Core i7-12700K (12C/20T) |
| **RAM** | 128 GB DDR4 |
| **Form Factor** | Desktop tower |
| **Hypervisor** | Proxmox VE |
| **Boot** | 2× 2.5" SSDs (RAID 1) |
| **VM Storage** | 1× NVMe SSD |
| **Persistent Volumes** | 1× NVMe SSD (NFS-exported to cluster) |
| **Bulk Storage** | 8× 14TB HDDs (2× RAIDz1 vdevs) |
| **GPU** | Intel UHD 770 iGPU with SR-IOV (7 virtual functions) |
| **Network** | Dual VLAN (management + services) |

One machine. Three K3s nodes. 60+ services. Full HA. GPU-accelerated transcoding.

---

## Repositories

| Repository | Purpose | Key Tech |
|------------|---------|----------|
| [**📦 packer**](https://github.com/Starktastic-Homelab/packer) | Immutable Debian VM template factory | Packer · HCL · Preseed · Cloud-Init · SR-IOV DKMS |
| [**🏗️ terraform**](https://github.com/Starktastic-Homelab/terraform) | Declarative VM provisioning with GPU passthrough | Terraform · Proxmox · S3 State · Drift Detection |
| [**⚙️ ansible**](https://github.com/Starktastic-Homelab/ansible) | K3s cluster setup with HA and GitOps bootstrap | Ansible · K3s · Kube-VIP · ArgoCD · Sealed Secrets |
| [**☸️ apps**](https://github.com/Starktastic-Homelab/apps) | GitOps application platform (60+ services) | ArgoCD · Helm · Traefik · Authentik · Prometheus |

---

## Engineering Highlights

What makes this more than "just a homelab":

- **🔗 End-to-End Pipeline** — Four repos chain together via GitHub Actions. A Packer build automatically triggers Terraform, which triggers Ansible, which bootstraps ArgoCD. Zero manual steps from image to running cluster.

- **🔄 Infrastructure Drift Detection** — Daily Terraform plans detect and report configuration drift via GitHub Issues, auto-closing when resolved.

- **🔒 Cross-Repo Driver Synchronization** — The Packer CI blocks PR merge if the VM's GPU driver version exceeds the Proxmox host's, preventing boot failures. Renovate coordinates version bumps across repos.

- **📐 4-Phase Ordered Deployment** — ArgoCD's RollingSync deploys CRDs → Foundation → Controllers → Services in strict order, ensuring clean bootstrap from an empty cluster.

- **🧬 Value Cascade Architecture** — A single `globals.yaml` propagates cluster-wide config (domains, IPs, storage) to 60+ services. Changing one value updates everything on the next sync.

- **🎭 Base/Variant Inheritance** — Multi-language service variants inherit 95% of their config from a base service, overriding only what differs in a 3-5 line YAML delta.

- **🔐 Day-Zero Secrets** — Sealed Secrets keypair is pre-seeded by Ansible before ArgoCD deploys, enabling encrypted secrets in Git from the first sync — no chicken-and-egg problem.

- **📡 Full Observability Stack** — Prometheus metrics, Loki logs, Tempo traces, Grafana dashboards, and AlertManager → ntfy push notifications — all deployed via GitOps.

- **🎮 GPU Virtualization** — Intel iGPU SR-IOV exposes 7 virtual functions, shared across workers for hardware-accelerated video transcoding and ML inference.

- **🤖 17 CI/CD Workflows** — Across all repos: automated builds, plan-on-PR, drift detection, manifest validation, ArgoCD diff previews, ISO version tracking, driver sync checks, and scoped refresh.

---

## CI/CD Overview

17 GitHub Actions workflows across all repositories:

| Repo | Workflows | Highlights |
|------|-----------|------------|
| **Packer** (5) | build · validate · format · check-debian-iso · check-host-driver | Weekly ISO scraping, cross-repo driver sync |
| **Terraform** (4) | validate-and-plan · apply · drift · format | Plan-on-PR, daily drift detection, drain/destroy modes |
| **Ansible** (5) | deploy · validate · format · i915-sriov-upgrade · ser2net | Terraform-triggered deploy, GPU driver lifecycle |
| **Apps** (3) | validate-and-diff · refresh · format | Kubeconform + ArgoCD diff preview, smart scope refresh |

Every PR gets validated. Every merge triggers the right downstream action. No manual deployment steps exist.

---

<div align="center">

*Built with care, automated with purpose, and maintained with the same rigor I bring to production environments.*

**Ben Faingold** · Senior DevOps Engineer

</div>
