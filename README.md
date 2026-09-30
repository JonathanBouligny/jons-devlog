# jons-devlog

Engineering devlog source for [bouligny.dev](https://bouligny.dev), powered by [Hugo](https://gohugo.io/) and hosted on Cloudflare Pages.

---

## 📚 Content Organization & Guides

- **Style & Authoring Guide**: See [STYLE_GUIDE.md](STYLE_GUIDE.md) for post structure, frontmatter schemas, date backfilling, sanitization rules, and cross-posting conventions.
- **Series Landing Pages**:
  - `content/fleet-platform/`: Posts covering zero-drift automated Kubernetes infrastructure on Proxmox VE.
  - `layouts/fleet-platform/list.html`: Custom layout containing the **Architecture & Build Roadmap** and paginated devlog stream.

---

## 📝 Roadmap & Multi-Post Phase Architecture Note

> **Future Navigation Note**:
> - **Multiple Posts per Phase / Per Node**: Posts may be written per node or broken into multiple parts per milestone.
> - **Button Transition**: If a phase contains multiple articles, the roadmap table button should transition from `Read Post →` to **`Go to Phase Posts →`**.
> - **Sub-directory Organization**: Subsequent multi-post phases can use sub-folders under `content/fleet-platform/` (e.g. `content/fleet-platform/phase-02/` with an `_index.md`) to provide a dedicated phase-level landing and index page.
