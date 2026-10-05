# Jon's Devlog — Post Style Guide & Authoring Standards

This document defines the structure, tone, formatting, and security rules for technical posts published to [bouligny.dev](https://bouligny.dev).

---

## 1. File & Slug Conventions

* **Location**: Place all series posts in `content/fleet-platform/`.
* **File Naming Pattern**: `phase-XX-part-YY-topic-slug.md`
  * Always zero-pad numbers so posts sort sequentially in terminals and IDEs.
  * Examples:
    * `phase-01-proxmox-cloudinit-terraform.md`
    * `phase-02-part-01-cert-manager-rate-limits-restore-race.md`
    * `phase-02-part-02-argocd-bootstrap-app-of-apps.md`
    * `phase-03-argo-workflows-gazebo-simulations.md`

---

## 2. Standard Frontmatter Template

Every post **must** start with this YAML frontmatter header:

```yaml
---
title: "Phase X, Part Y: Concise, Compelling Engineering Title"
date: 2026-09-29
series: ["fleet-platform"]
tags: ["proxmox", "terraform", "kubernetes", "ansible", "argocd", "gazebo"]
draft: false
---
```

* `title`: Full human-readable title with proper punctuation and capitalization.
* `date`: `YYYY-MM-DD` format.
* `series`: Matches the project name (`fleet-platform`) so Hugo groups it automatically.
* `tags`: 3–6 lowercase tags relevant to tools and concepts.
* `draft`: Set to `false` when ready to publish.

### Chronological Sorting & Backfilling Strategy
* **How Hugo Orders Articles**: Hugo sorts list and archive pages strictly by the frontmatter `date` in descending order.
* **Backfilling Prior Phases**: When writing earlier phases out-of-order (e.g., backfilling Phase 0 or Phase 1 after publishing Phase 2), set their `date` field to match when the engineering milestone occurred (e.g. `2026-09-14` for Phase 0, `2026-09-18` for Phase 1).
* **Automatic Ordering**: Hugo will automatically slot backfilled posts into their correct chronological position on `/fleet-platform/` and the homepage without changing URLs or directory structure.

---

## 3. Narrative Architecture (The Senior Engineering Formula)

Write in a direct, first-person, pragmatic engineering voice. Avoid academic fluff. Each post should follow this arc:

1. **The Context / Problem**: What broke, what limit did you hit, or what architectural requirement triggered this work? (e.g., *"Let's Encrypt weekly rate limits burned out during full cluster rebuild tests."*)
2. **The Discovery / Deep-Dive**: What did you learn under the hood? (e.g., CRD lifecycle, controller reconciliation loops, ordering dependencies).
3. **The Setup**: The exact configuration, playbook, or manifest used to solve the problem.
4. **The Proof**: Terminal output, kubectl condition logs, or benchmark comparisons proving it works.
5. **Trade-offs & Next Steps**: What compromises were made (e.g., manual pruning vs. automated cleanup)? What is the next planned improvement (e.g., moving to OpenBao)?

---

## 4. Code Blocks & Formatting Rules

### A. Always Use Language-Specific Code Fences
Never paste un-fenced code. Always specify the language:
* ```` ```yaml ```` for Ansible playbooks, Kubernetes manifests, Helm values.
* ```` ```bash ```` for shell scripts, CLI commands.
* ```` ```hcl ```` for Terraform files.
* ```` ```text ```` for terminal output, logs, kubectl describe tables.

### B. Inline Formatting
* Use backticks for CLI commands, flags, tools, and Kubernetes kinds:
  * Good: `Certificate`, `ClusterIssuer`, `prune: false`, `kubectl apply`
  * Bad: Certificate, ClusterIssuer, prune: false

### C. Headings
* Use `##` for primary sections (`## The Setup`, `## The Proof`).
* Use `###` for sub-points. Avoid using `#` (H1 is reserved for the post title).

---

## 5. Security & Sanitization Checklist (MANDATORY)

Before committing any post, verify the following:

- [ ] **No Hardcoded Absolute Paths**: Avoid absolute home workstation paths or local scratchpad directories.
  - *Replace with*: `~/.secrets/fleet-platform/` or `{{ secrets_dir }}`.
- [ ] **No Plaintext Secrets**: No tokens, API keys, private keys, passwords, or hashes.
- [ ] **Parameterization**: Code snippets should use variables, environment lookups, or generic placeholders so readers can reuse the logic.
- [ ] **Domain Check**: If showcasing `wehaveaproblem.net`, verify that public-facing endpoints (like Argo CD) are secured behind authentication or Cloudflare Access.
- [ ] **Pre-commit Verification**: Ensure `.git/hooks/pre-commit` is active (`chmod +x .git/hooks/pre-commit`) and passes without errors on `git commit`.

---

## 6. Cross-Posting Rules (dev.to / LinkedIn)

When cross-posting to **dev.to**, **Medium**, or **LinkedIn Articles**:
* **Canonical URL**: Always set the canonical link in dev.to frontmatter back to your personal site:
  ```markdown
  canonical_url: https://bouligny.dev/fleet-platform/phase-xx-part-yy-slug/
  ```
  *Why*: This signals to Google search algorithms that `bouligny.dev` is the original authority, driving search ranking and domain reputation to your personal site rather than the third-party platform.

---

## 7. Future Note: Multi-Post Phases & Roadmap Navigation

* **Multiple Posts per Phase**: We may author more than one post per phase (e.g., individual posts per node, multi-part deep dives).
* **Roadmap Button Transition**: When a phase contains more than one article, the single `Read Post →` button on the `/fleet-platform/` roadmap table should change into a **`Go to Phase Posts →`** (or `View Phase Posts →`) button.
* **Sub-folder / Sub-index Organization**: If a phase expands into multiple posts, consider organizing them inside a sub-folder under `content/fleet-platform/` (e.g. `content/fleet-platform/phase-02/` with an `_index.md` or dedicated listing template) so readers land on a clean phase-specific index page before diving into individual posts.

---

## 8. Punctuation Standard: No Em Dashes

* **Strictly Prohibit Em Dashes (`—` / `--`)**: Do not use em dashes in post prose. Modern AI models heavily overuse em dashes to stitch clauses together, creating a distinct, repetitive pattern. Break thoughts into separate, clear sentences or use standard commas with conjunctions (`and`, `but`, `which`).
* **Colons (`:`) in Technical Writing**: Natural colons are completely acceptable when introducing code blocks, configuration manifests, or CLI outputs. Avoid mechanical AI bullet formats (such as formulaic `Problem: ...`, `Solution: ...` repetitive labeling).


