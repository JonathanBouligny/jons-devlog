---
title: "About Jonathan Bouligny"
date: 2026-09-29
draft: false
---

I am a **Platform and Infrastructure Engineer** based in Austin, Texas, with 6+ years of experience designing, automating, and operating enterprise Linux platforms at **Red Hat** and **Lockheed Martin**. 

I hold the **Red Hat Certified Architect (RHCA) in Ansible** credential and specialize in building deterministic, reproducible systems from the bare hypervisor up using **Terraform, Ansible, Kubernetes (k3s/OpenShift), and GitOps (Argo CD)**.

---

## What I'm Building Now: Fleet Platform

My current flagship engineering project is **[fleet-platform](/fleet-platform/)**—a fully reproducible, GitOps-managed Kubernetes platform deployed on a self-hosted 3-node Proxmox cluster (40 cores, 157 GB RAM).

* **Infrastructure as Code**: VMs provisioned via Terraform (`bpg/proxmox`) from cloud-init templates; nodes configured and bootstrapped via Ansible.
* **Declarative GitOps**: Full cluster state managed via Argo CD (app-of-apps pattern) syncing from a code-provisioned Forgejo instance with remote Terraform state locked in AWS S3.
* **Cattle, Not Pets**: The real test of reproducibility is a full teardown. The entire platform—from empty hypervisors to running workloads, ingress, wildcard TLS, and synced apps—is continuously verified by rebuilding from scratch via a single automated script to eliminate configuration drift.
* **The Endgame**: Scaling the cluster with a dedicated GPU node to run automated, parallel, deterministic **Gazebo robotics simulations** via Argo Workflows batch pipelines.

---

## Professional Background

### **Red Hat** — Automation Consultant *(2022 – Present)*
* Delivered enterprise infrastructure automation across 7 federal and healthcare engagements, including complex air-gapped, security-hardened environments.
* **Dynamic CMDB Integration**: Engineered a custom Python dynamic inventory plugin enabling Ansible Automation Platform (AAP) to consume ~1,300 hosts across two datacenters from an Ivanti CMDB, filtering systems by lifecycle status and team ownership to eliminate mis-targeted automation.
* **Multi-Node Platform Installs**: Architected and deployed multi-node AAP clusters (controller, private automation hub, database) on VMware vSphere, converting manual runbooks into fully versioned Ansible code for zero-drift rebuilds.
* **CI/CD Image Optimization**: Converted Ansible Execution Environment (EE) image builds into a pipeline-driven IaC workflow, slashing build times from 10 minutes to 2 minutes on disconnected networks.
* **Air-Gapped Source Control**: Deployed containerized GitLab on Podman and systemd in an isolated enclave to establish the customer’s first version-control pipeline for enterprise automation.

### **Lockheed Martin** — Software Engineer *(2020 – 2022)*
* Resolved mission-critical Red Hat OpenShift deployment blockers by analyzing pod logs, isolating container lifecycle failures, and correcting Kubernetes deployment manifests to restore service.

---

## Technical Competencies

* **Languages**: Python, Bash, YAML, Go (learning), C++, JavaScript
* **Cloud & Infrastructure**: Linux (RHEL, Ubuntu), Proxmox VE, VMware vSphere, AWS (S3, IAM, VPC), Bare-metal Networking
* **Kubernetes & GitOps**: Kubernetes, k3s, Red Hat OpenShift, Argo CD (app-of-apps), Helm, Podman, Docker
* **Automation & IaC**: Ansible (RHCA), Terraform, cloud-init, Red Hat Satellite, CI/CD pipelines

---

## Certifications & Education

* **Red Hat Certified Architect (RHCA)** in Ansible
* **Red Hat Certified Engineer (RHCE)** — Enterprise Linux & Ansible
* **Red Hat Certified System Administrator (RHCSA)**
* **Red Hat Certified Specialist**:
  * Managing Automation with Ansible Automation Platform (EX467)
  * Developing Automation with Ansible (EX374)
  * Ansible Network Automation (EX457)
* **B.S. in Computer Science, Magna Cum Laude** — Texas Tech University *(2015 – 2019)*

---

## Contact & Links

* **Email**: [jonathanbouligny@gmail.com](mailto:jonathanbouligny@gmail.com)
* **LinkedIn**: [linkedin.com/in/jonathan-j-bouligny](https://www.linkedin.com/in/jonathan-j-bouligny)
* **GitHub**: [github.com/JonathanBouligny](https://github.com/JonathanBouligny)
* **Location**: Austin, TX
