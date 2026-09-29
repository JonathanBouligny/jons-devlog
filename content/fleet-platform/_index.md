---
title: "Fleet Platform — Build Log & Architecture Roadmap"
description: "A fully reproducible, GitOps-managed Kubernetes platform on a 3-node Proxmox cluster, scaling to an Argo Workflows + Gazebo robotics simulation farm."
cascade:
  featured_image: ''
---

**Fleet Platform** is an end-to-end, fully reproducible private cloud platform built on a 3-node Proxmox cluster (40 cores, 157 GB RAM). Every layer—from virtual machine provisioning to cluster networking, GitOps controllers, ingress, and wildcard TLS—is declared in code and version-controlled.

* **Source Code**: Full infrastructure code, Terraform manifests, Ansible playbooks, and GitOps declarations are public at [github.com/JonathanBouligny/fleet-platform](https://github.com/JonathanBouligny/fleet-platform).
* **Cattle, Not Pets**: The real test of reproducibility is a full teardown. The entire platform is continuously verified by rebuilding from empty hypervisors via a single automated script to guarantee zero configuration drift.
* **The Endgame**: A parallel headless **Gazebo robotics simulation farm** driven by Argo Workflows (50 deterministic, seeded runs per commit yielding pass/fail regression reports) backed by dedicated GPU acceleration.
