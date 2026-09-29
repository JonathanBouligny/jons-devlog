---
title: "Fleet Platform — Build Log & Architecture Roadmap"
description: "A fully reproducible, GitOps-managed Kubernetes platform on a 3-node Proxmox cluster, scaling to an Argo Workflows + Gazebo robotics simulation farm."
cascade:
  featured_image: ''
---

**Fleet Platform** is an end-to-end, fully reproducible private cloud platform built on a 3-node Proxmox cluster (40 cores, 157 GB RAM). 

Every layer—from virtual machine provisioning to cluster networking, GitOps controllers, ingress, and wildcard TLS—is declared in code and version-controlled.

* **Acceptance Test**: The entire platform tears down to empty hypervisors and rebuilds to a fully operational state in **3m17s to 7m19s** via a single automated orchestrator script (`timed-rebuild.sh`).
* **The Endgame**: A parallel headless **Gazebo robotics simulation farm** driven by Argo Workflows (50 deterministic, seeded runs per commit yielding pass/fail regression reports) backed by dedicated GPU acceleration.

---

## Architecture & Build Roadmap

Below is the live tracker for the platform build. Entries are marked with their publication status and link directly to technical write-ups:

| Phase | Milestone / Topic | Focus Areas | Status |
| :--- | :--- | :--- | :--- |
| **Phase 0** | **Cluster Foundation** | 2-node k3s bootstrap, Proxmox hypervisors, cloud-init templates | 📝 *Backfilling write-up* |
| **Phase 1** | **GitOps & Ingress** | Ansible AAP bootstrap, HAProxy ingress, Argo CD app-of-apps | 📝 *Backfilling write-up* |
| **Phase 2.1** | **TLS & Ingress Security** | cert-manager, Cloudflare DNS-01, rate limits, restore race | [Read Post &rarr;](./phase-02-part-01-cert-manager-rate-limits-restore-race/) |
| **Phase 2.2** | **Isolated GitOps Engine** | Self-hosted Forgejo server provisioned as code, GitHub mirror | 🔨 *In Progress* |
| **Phase 2.3** | **AWS Cloud Footprint** | Terraform S3 remote state w/ locking, IAM least-privilege, VPC | 🔨 *In Progress* |
| **Phase 3** | **Platform Observability** | kube-prometheus-stack via Argo, Grafana, HAProxy ServiceMonitors | ⏳ *Planned* |
| **Phase 4** | **Synthetic Workloads & k6** | CPU-burner app, HPA, external AWS EC2 k6 load generation | ⏳ *Planned* |
| **Phase 5** | **Zero-Trust Secrets** | SOPS + age encryption, gitleaks CI audit, OpenBao / ESO | ⏳ *Planned* |
| **Phase 6** | **Cloud-Native Storage** | Longhorn distributed storage, MinIO S3-compatible backup tier | ⏳ *Planned* |
| **Phase 7** | **GPU Node Integration** | Passthrough, `nvidia-smi` batch jobs, headless Vulkan/EGL render | ⏳ *Planned* |
| **Phase 8** | **Robotics Networking** | ROS 2 DDS discovery characterization, Zenoh bridge benchmarks | ⏳ *Planned* |
| **Phase 9** | **Sim Farm: Gazebo + Argo** | 50 seeded parallel runs, deterministic physics, pass/fail report | 🎯 *Flagship Capstone* |

---

## Published Engineering Logs
