---
title: "Phase 2, Part 1: cert-manager, Rate Limits, and the Restore Race"
date: 2026-09-29
series: ["fleet-platform"]
tags: ["cert-manager", "cloudflare", "dns-01", "kubernetes", "letsencrypt", "ansible"]
phase: "2"
summary: "Handling cert-manager Helm installation, Let's Encrypt DNS-01 wildcard certificates, rate limiting, and bootstrap secret timing races during full teardowns."
draft: false
---

> **A note on order:** I started this devlog partway through, around the cert-manager and ingress work, so the posts aren't showing up in the order I actually built things. Phase 0 (the Proxmox/Terraform/Ansible foundation and the k3s bootstrap) and Phase 1 (Argo CD and ingress) came first, and I'm backfilling those writeups now. The phase numbers in the titles are the real order to read them in.

The full platform is on GitHub: [github.com/JonathanBouligny/fleet-platform](https://github.com/JonathanBouligny/fleet-platform). This post covers the cert-manager piece.

## cert-manager

Setting up cert-manager was pretty easy with a Helm chart. I did spend a lot of time on this step but it was mostly googling and figuring out what to do about bootstrap secrets which I detail below. The most interesting thing I learned was part of the values file, specifically `installCRDs` (or `crds.enabled`, depending on chart version). CRDs, Custom Resource Definitions, are objects that teach the Kubernetes API server about new kinds of resources, like `Certificate` and `ClusterIssuer`. They're not backed by Go directly, they're schema definitions, and a controller (cert-manager's own pod, running Go code) is what actually watches for objects of that kind and acts on them. So a CRD without its controller running is just an inert schema. The controller is the operator, the CRD is the vocabulary it teaches the cluster.

I needed these CRDs present before cert-manager's controller could start successfully. Without them, the install would fail, or the controller would come up but be unable to do anything, since the kinds it needs wouldn't exist yet. The chart handles this by installing the CRDs and the controller together, which is convenient, but it comes with a real downside: if I ever uninstall the Helm release, the CRDs get removed too, and when a CRD is removed, every object of that kind is deleted with it, cascade, immediately. That means an uninstall could silently delete every `Certificate` and `ClusterIssuer` in the cluster, not just cert-manager's own pods.

To guard against that, I set `prune: false` on the Argo Application that installs cert-manager. That way, if the Application is ever removed from Git, Argo won't prune the CRDs (or anything else it manages) along with it. Tradeoff: if I ever do want a real teardown, I have to remove those resources by hand instead of Argo doing it automatically. But that's the point. It means an accidental deletion doesn't take my certs with it, and a real, intentional teardown still works, it's just not automatic. Left a comment in the file explaining why. (The chart actually has its own protection for this too. `crds.keep` stops a helm uninstall from taking the CRDs with it but that's a different path than Argo's prune.)

The actual cert-issuing part: I created a `ClusterIssuer` since I'm not doing multi-tenancy. One issuer, cluster-wide, no need to scope it per-namespace. I issued a cert for `argocd.wehaveaproblem.net` against Let's Encrypt's staging environment first to confirm the whole DNS-01/Cloudflare flow worked, then switched to production once it did. But I hit Let's Encrypt's rate limit fast. It only allows a handful of identical certs per week, and every full rebuild was requesting a fresh one for the same name. So instead of a per-hostname cert, I switched to a single wildcard (`*.wehaveaproblem.net`), which covers every service I'll ever put behind the ingress with one certificate instead of one per host. I exported the issued cert and saved it in my local secrets folder (which moves to OpenBao eventually) so a rebuild can restore the existing cert instead of requesting a new one from Let's Encrypt every time.

## The setup

I have a `Certificate` manifest that always points at the same saved object, `secretName: wehaveaproblem-net-tls`, in the `haproxy-controller` namespace. It requests the wildcard, pinned to that `ClusterIssuer`, 90-day duration with a 15-day renewal window. (Not Before/Not After are just the cert's validity window, issuance time and expiry, and `renewBefore` tells cert-manager how far ahead of expiry to start renewing so it never waits until the last second.) The manifest itself never changes between rebuilds. Same file every time, declaring "this is the cert I want."

What makes the reuse actually work is a bootstrap step that runs before that manifest gets applied. My Ansible playbook that stands up the cluster has a task that loads a handful of secrets, including the saved cert, into the right namespaces first:

```yaml
- name: Configure bootstrap secrets
  hosts: localhost
  vars:
    kubeconfig_path: "~/.kube/config"
    secrets_dir: "~/.secrets/fleet-platform" # Stored locally outside the repo
    bootstrap_secrets:
      - { namespace: argocd, file: "{{ secrets_dir }}/secret-argo-forgejo.yml" }
      - { namespace: cert-manager, file: "{{ secrets_dir }}/cloudflare-api-token.yml" }
      - { namespace: haproxy-controller, file: "{{ secrets_dir }}/wehaveaproblem-net.yml" }
  tasks:
    - name: Make sure namespaces for the secrets exist
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        state: present
        name: "{{ item.namespace }}"
        kind: Namespace
      loop: "{{ bootstrap_secrets }}"

    - name: Apply secrets in loop
      kubernetes.core.k8s:
        kubeconfig: "{{ kubeconfig_path }}"
        namespace: "{{ item.namespace }}"
        state: present
        apply: true
        src: "{{ item.file }}"
      loop: "{{ bootstrap_secrets }}"
      no_log: true
```

By the time Argo brings up the Certificate manifest, the Secret it's asking for already exists, restored from my saved copy, not freshly issued. When cert-manager reconciles, it sees a valid cert sitting there and does nothing. That ordering is the whole trick. The saved cert has to land before the Certificate object does, or cert-manager finds nothing and requests a new one from Let's Encrypt anyway.

## The proof

After the rebuild, I checked the live Certificate object:

```text
Metadata:
  Creation Timestamp:  2026-09-23T21:48:08Z
...
Status:
  Conditions:
    Message:  Certificate is up to date and has not expired
    Reason:   Ready
  Not Before:  2026-09-23T14:29:47Z
  Not After:   2026-12-22T14:29:46Z
Events:  <none>
```

The Certificate object itself was created fresh at 21:48. This rebuild spun it up from nothing, like everything else in the cluster. But the cert it's pointing at was issued at 14:29, over seven hours earlier. cert-manager looked at the Secret my bootstrap step had already restored, confirmed it was still valid ("Certificate is up to date and has not expired"), and did nothing. No `CertificateRequest`, no `Order`, no `Challenge`. `Events: <none>` says it plainly: nothing happened, because nothing needed to.

That's the whole point of the exercise. The cert survived a full teardown-and-rebuild without touching Let's Encrypt at all, which means I can rebuild this cluster as many times as I want during development without ever burning down my weekly rate limit.

I'm not entirely sure how this workflow will look when I switch to a secrets manager. It's still an order issue, so I think I still need to load bootstrap secrets the way I do. I'll see if there's a better way to load bootstrap secrets.