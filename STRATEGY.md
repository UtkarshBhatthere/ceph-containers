# Ceph OCI Image Builder — Strategic Competition Plan

> **Branch:** `claude/ceph-oci-strategy-review-86l4n`  
> **Date:** May 2026  
> **Based on:** `upstream/main` at `7fd4f77` ("Prepare main for Ceph Tentacle")  
> **Scope:** Accurate current-state assessment of canonical/ceph-containers, comparison with upstream Ceph community containers, and a gap-closing roadmap.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Actual Current State — This Repository](#2-actual-current-state--this-repository)
3. [Upstream Landscape — ceph/ceph container/](#3-upstream-landscape--cephceph-container)
4. [Competitive Gap Analysis](#4-competitive-gap-analysis)
5. [Strategic Objectives](#5-strategic-objectives)
6. [Prioritised Roadmap](#6-prioritised-roadmap)
7. [Canonical Differentiators](#7-canonical-differentiators)
8. [Risks & Mitigations](#8-risks--mitigations)
9. [Success Metrics](#9-success-metrics)

---

## 1. Executive Summary

`canonical/ceph-containers` is significantly more advanced than a fork snapshot suggested. The repository has been actively developed: it now targets **Ubuntu 24.04 (Noble)**, packages **Ceph Tentacle (20.x)** from a MicroCeph-aligned PPA, ships a clean Rockcraft build with upstream bug patches, and has a three-tier CI/CD publishing system (edge, hotfix, release).

**The upstream situation has also changed materially.** `ceph/ceph-container` was archived December 2024. All upstream container builds moved into `ceph/ceph container/` using CentOS Stream 9 / Rocky Linux 10. There is no active Ubuntu-native Ceph OCI image project except this one.

**The gap is real but focused.** The main remaining gaps are: `amd64`-only builds (no arm64), single-version-per-branch architecture (no `stable/squid` or `stable/reef` branches yet), stale CI action versions, deprecated GitHub Actions syntax, and no multi-registry distribution beyond GHCR.

**The window is now.** Being the sole Ubuntu Ceph OCI image maintainer with Tentacle support is a strong position. The work needed to consolidate it is incremental, not a rewrite.

---

## 2. Actual Current State — This Repository

### 2.1 Architecture (Post-Upstream-Merge, `7fd4f77`)

```
rockcraft.yaml          <- Ubuntu 24.04 base; Tentacle (20.x) via MicroCeph PPA
patches/                <- 3 upstream Ceph bug fixes applied at build time
kubectl/                <- Go source for kubectl bundled in image
logrotate.d/            <- Log rotation config (for Rook)
ceph.defaults           <- Default Ceph config values
scripts/
  add-oci-label         <- Stamps cephadm-compatible OCI label on .rock
  deploy-helper.sh      <- Rook cluster deployment helper
  test-osd-removal.sh   <- OSD removal validation
test/
  scripts/cephadm_helper.sh  <- Cephadm bootstrap + LXD test automation
.github/workflows/
  build_and_test.yaml   <- Full CI: build rock + Cephadm + Rook canary tests
  publish_edge.yaml     <- Push to GHCR on commit to main/stable/* (edge tags)
  publish_hotfix.yaml   <- Push to GHCR on 'hotfix' PR label
  publish_release.yaml  <- Push to GHCR on version tag (v**)
  registry_housekeeping.yaml <- Cleans stale GHCR images
  cla-check.yaml        <- Canonical CLA enforcement
```

**What was removed** (relative to the September 2023 fork point):
- Entire `docker_scripts/` legacy entrypoint suite (33 bash scripts) — cleaned by PR #50
- `confd/` directory (etcd KV config management) — removed
- `s3cfg` — removed
- `test/deploy.py` — removed (replaced by `test/scripts/cephadm_helper.sh`)
- `publish_image.yaml` / `publish_temp_image.yaml` — replaced by the three new publish workflows

### 2.2 Build System — rockcraft.yaml

| Dimension | Current value |
|-----------|--------------|
| Base | `ubuntu@24.04` (Noble) |
| Ceph version | Tentacle 20.x (from `lmlogiudice/ceph-tentacle-noble` PPA) |
| Architecture | `amd64` only |
| Version field | `'0.1'` — patched at CI time via `sed` on `pkg_info` part output |
| Parts | `pkg_info`, `ceph`, `nfs-ganesha`, `kubectl`, `local-files` |
| Patches | 3 patches applied at overlay time: `verify_cacrt_content`, prometheus fix, SMB stub |
| Pebble service | Defined (`ceph-container`); still experimental |

### 2.3 CI/CD Workflows — Publishing Pipeline

| Workflow | Trigger | Tags pushed |
|---------|---------|------------|
| `publish_edge.yaml` | Push to `main` or `stable/*` | `tentacle-edge`, `squid-edge`, `reef-edge`, `quincy-edge`, `dev-edge` (main only) |
| `publish_hotfix.yaml` | PR labeled `hotfix` | `hotfix-{PR#}-{ceph_version}` |
| `publish_release.yaml` | Push a `v**` tag | `v{semver}`, `{pkg_version}`, codename (`tentacle`, `squid`, `reef`, `quincy`) |

All publish to `ghcr.io/canonical/ceph`.

### 2.4 Patching Strategy

Three patches applied during overlay build to work around upstream Ceph Tentacle bugs not yet fixed in Ubuntu packages:
- `0001`: Adds missing `verify_cacrt_content` to `mgr_util.py`
- `0002`: Fixes `TypeError` in Prometheus module string formatting
- `0003`: Stubs missing `smb` mgr module (not yet packaged for Ubuntu/Debian)

These are adapted from MicroCeph patches, demonstrating intentional Canonical-ecosystem alignment.

### 2.5 Feature Support Matrix (Current)

| Feature | Cephadm | Rook |
|---------|---------|------|
| Status / ps / ls / Hosts | ✅ | ✅ |
| Device ops | ✅ | ✅ |
| Dashboard | ✅ | ✅ |
| OSD ops | ✅ | ✅ |
| RBD Mirror | ✅ | ✅ |
| RadosGW | ✅ | ⚠️ |
| Upgrade | ✅ | ⚠️ |
| Hosts (maintenance) | ✅ | ❌ |
| iSCSI | ✅ | ❌ |
| Loki | ✅ | ❌ |
| Prometheus | ✅ | ❌ |
| Alert Manager | ✅ | ❌ |
| Node Exporter | ✅ | ❌ |

### 2.6 Remaining Technical Debt

| Item | Detail |
|------|--------|
| `set-output` deprecated syntax | All three publish workflows use `echo "::set-output name=..."` — deprecated since Oct 2022; should use `$GITHUB_OUTPUT` |
| GitHub Actions v3 | `actions/checkout@v3`, `docker/login-action@v1`, `docker/metadata-action@...sha` — all have newer versions |
| LXD pinned to `5.21/edge` | Unstable channel; should track a stable LXD release |
| Rockcraft `latest/edge` | All publish workflows install rockcraft from edge; should use a pinned stable channel |
| `kubectl` version | May reference old k8s version; verify against current stable |
| PPA dependency | `lmlogiudice/ceph-tentacle-noble` — needed until Tentacle enters Ubuntu main archive (tracked at LP #2150762) |

---

## 3. Upstream Landscape — ceph/ceph container/

### 3.1 Status

`ceph/ceph-container` was **archived December 19, 2024**. All builds are now in `ceph/ceph` under `container/`:

| File | Purpose |
|------|---------|
| `container/Containerfile` | Per-branch image definition |
| `container/build.sh` | Build/push via `podman`; driven by env vars |
| `container/make-manifest-list.py` | Multi-arch manifest creation and `--promote` |

### 3.2 Key Characteristics

| Dimension | Upstream |
|-----------|----------|
| Base OS | CentOS Stream 9 (reef, squid); Rocky Linux 10 (tentacle) — **no Ubuntu** |
| Architectures | `amd64` + `arm64` |
| Active releases | Tentacle (v20), Squid (v19), Reef (v18, approaching EOL) |
| Registry | `quay.io/ceph/ceph` (active); Docker Hub frozen at v16 |
| CI | Jenkins + Shaman; not GitHub Actions; not community-forkable |
| Entrypoint | **None defined** — orchestrators inject their own |
| Rebuild cadence | < 24h from new package publication |
| SBOM / signing | None |

### 3.3 Canonical's Opening vs Upstream

| Upstream gap | Canonical advantage |
|---|---|
| CentOS/Rocky Linux only | **Ubuntu LTS + ESM security pipeline** — exclusive to Canonical |
| No SBOM or image signing | **Rockcraft** generates hermetic, attestable builds |
| No Pebble | **Pebble** — structured OCI-native process supervision |
| No charm/snap ecosystem | **Charmed Ceph, MicroCeph, Juju** — Canonical-exclusive surfaces |
| Jenkins CI (opaque) | **GitHub Actions** — accessible to any contributor |
| Docker Hub dead at v16 | Canonical can claim Docker Hub `canonical/ceph` |
| No Launchpad SRU alignment | Canonical controls Ubuntu package lifecycle end-to-end |

---

## 4. Competitive Gap Analysis

### 4.1 Critical Gaps (Blocking parity or adoption)

| Gap | Current state | Upstream | Effort |
|-----|---------------|----------|--------|
| **arm64 support** | amd64 only | amd64+arm64 | Medium-High |
| **Stable release branches** | `main` only (tracks Tentacle); no `stable/squid`, `stable/reef` | Per-branch builds | Medium |
| **Multi-registry distribution** | GHCR only | `quay.io/ceph/ceph` (primary) | Low |

### 4.2 Significant Gaps (Quality and maintainability)

| Gap | Detail | Effort |
|-----|--------|--------|
| Deprecated `set-output` syntax | All publish workflows use EOL GitHub Actions output syntax | Low |
| Stale action versions | `checkout@v3`, `login-action@v1`, `metadata-action@...sha` | Low |
| LXD/rockcraft on unstable channels | `5.21/edge` + rockcraft `latest/edge` in all publish jobs | Low |
| No image signing | No cosign/sigstore attestation | Medium |
| No SBOM | No SBOM attached to published images | Medium |
| No CVE rebuild automation | No scheduled rebuild triggered by Ubuntu security updates | Medium |
| Rook monitoring stack | Prometheus, Loki, Alert Manager, Node Exporter unsupported via Rook | High |
| PPA dependency (LP #2150762) | Needed until Tentacle enters Ubuntu main | Blocked on upstream |

### 4.3 Canonical Advantages Already Realised

| Feature | Status |
|---------|--------|
| Ubuntu 24.04 (Noble) base | Done |
| Ceph Tentacle (20.x) | Done (via MicroCeph PPA) |
| Rockcraft hermetic builds | Done |
| Upstream bug patches (MicroCeph-aligned) | Done (3 patches) |
| Three-tier publish pipeline (edge/hotfix/release) | Done |
| cephadm OCI label | Done (`scripts/add-oci-label`) |
| Dependabot | Done |
| SECURITY.md | Done |
| Registry housekeeping | Done |
| Pebble service definition | Present (experimental) |

### 4.4 Feature Parity Matrix

| Feature | This repo | Upstream ceph/ceph |
|---------|-----------|-------------------|
| Ceph Tentacle (20.x) | ✅ | ✅ |
| Ceph Squid (19.x) | ❌ no stable/squid branch | ✅ |
| Ceph Reef (18.x) | ❌ no stable/reef branch | ✅ |
| Ubuntu base | ✅ | ❌ |
| Ubuntu 24.04 (Noble) | ✅ | ❌ |
| CentOS Stream / Rocky Linux | ❌ by design | ✅ |
| amd64 | ✅ | ✅ |
| arm64 | ❌ | ✅ |
| GHCR | ✅ | ❌ |
| Quay.io | ❌ | ✅ |
| Docker Hub (active) | ❌ | ❌ (frozen at v16) |
| Version + codename tags | ✅ | ✅ |
| Edge channel images | ✅ | ✅ via Shaman |
| Hotfix/PR preview images | ✅ `hotfix` label | ❌ |
| Pebble process supervision | ✅ experimental | ❌ |
| Defined OCI entrypoint | ✅ | ❌ |
| Rockcraft hermetic builds | ✅ | ❌ |
| SBOM | ❌ | ❌ |
| Image signing (cosign) | ❌ | ❌ |
| CVE rebuild automation | ❌ | ✅ <24h |
| Upstream bug patches | ✅ 3 patches | ✅ |
| GitHub Actions CI | ✅ | ❌ Jenkins |
| Cephadm CI | ✅ | ❌ |
| Rook CI | ✅ | ❌ |
| Rook monitoring stack | ❌ | n/a no entrypoint |
| MicroCeph ecosystem alignment | ✅ PPA + patches | ❌ |

---

## 5. Strategic Objectives

### Objective 1 — Multi-Version Coverage
Create `stable/squid` and `stable/reef` branches. The existing `publish_edge.yaml` already triggers on `stable/*` — creating the branches immediately enables edge publishing.

### Objective 2 — arm64 Support
Add `arm64` to `rockcraft.yaml` platforms and produce multi-arch manifest lists.

### Objective 3 — CI/CD Modernisation
Eliminate deprecated GitHub Actions syntax and pin toolchain to stable channels. Small effort, large maintainability dividend.

### Objective 4 — Supply-Chain Integrity
Add cosign signing and SBOM — things upstream cannot offer — to assert Canonical's security differentiation.

### Objective 5 — Multi-Registry Presence
Publish to Quay.io (primary Kubernetes/OpenShift registry) and Docker Hub `canonical/ceph`.

### Objective 6 — Rook Monitoring Stack
Close the red-light items in the Rook compatibility matrix.

### Objective 7 — PPA Graduation
Track LP #2150762 and remove the PPA dependency when Tentacle enters Ubuntu main.

---

## 6. Prioritised Roadmap

### Phase 1 — Quick Wins (Weeks 1–2)

**Goal:** Fix technical debt; zero risk.

| Task | Detail |
|------|--------|
| Fix deprecated `set-output` | Replace with `echo "X=Y" >> $GITHUB_OUTPUT` in all 3 publish workflows |
| Update action versions | `actions/checkout@v4`, `docker/login-action@v3`, `docker/metadata-action@v5`, `actions/upload-artifact@v4` |
| Pin rockcraft to stable channel | Change `--channel latest/edge` to `--channel latest/stable` |
| Pin LXD to stable channel | Change `channel: 5.21/edge` to `channel: latest/stable` |
| Verify kubectl version | Audit `kubectl/go.mod`; bump to k8s 1.31+ if behind |

**Exit criteria:** No deprecated syntax; stable toolchain channels in all workflows.

---

### Phase 2 — Stable Release Branches (Weeks 3–5)

**Goal:** Cover all active Ceph releases with Ubuntu-based images.

| Task | Detail |
|------|--------|
| Create `stable/squid` branch | Update `rockcraft.yaml` for Squid (19.x) — likely from noble-backports/universe; no PPA needed |
| Create `stable/reef` branch | Update `rockcraft.yaml` for Reef (18.x) from Jammy or Noble archives |
| Validate `publish_edge.yaml` fires | Confirm `squid-edge` and `reef-edge` tags appear in GHCR |
| Update README | Add version/branch table (main=tentacle, stable/squid, stable/reef) |
| Extend CI matrix | Ensure `build_and_test.yaml` runs on `stable/*` PRs |

**Exit criteria:** `squid-edge` and `reef-edge` images in GHCR; all CI passes.

---

### Phase 3 — arm64 (Weeks 4–7, parallel with Phase 2)

**Goal:** Multi-arch images matching upstream.

| Task | Detail |
|------|--------|
| Add `arm64` to `rockcraft.yaml` platforms | `platforms: amd64: {} arm64: {}` |
| Enable cross-arch or native arm64 CI | QEMU via `canonical/craft-actions/rockcraft-pack@main` or self-hosted runners |
| Produce multi-arch OCI manifest | `skopeo` or `docker manifest` manifest list per tag |
| Update all three publish workflows | Add arm64 build job + manifest step |
| Validate on arm64 host | CI test on arm64 runner |

**Exit criteria:** `docker pull ghcr.io/canonical/ceph:tentacle-edge` on arm64 resolves correctly.

---

### Phase 4 — Security & Distribution (Weeks 7–12)

**Goal:** Assert Ubuntu security leadership; reach users on all major registries.

| Task | Detail |
|------|--------|
| Add cosign signing | Sign every published image; attach public key to repo |
| Generate SBOM | Use `syft` or `trivy`; attach as OCI referrer |
| Publish to Quay.io | Add `quay.io/canonical/ceph` push |
| Publish to Docker Hub | Add `docker.io/canonical/ceph` push |
| Automated CVE rebuild | Weekly scheduled workflow; rebuild if Ubuntu package checksums changed |
| Ubuntu Pro image labels | `com.canonical.ubuntu.esm=true`, `org.opencontainers.image.vendor=Canonical` |

**Exit criteria:** Images signed + SBOM; available on GHCR + Quay.io + Docker Hub; auto-rebuild fires within 48h of USN.

---

### Phase 5 — Rook Monitoring Stack (Weeks 12–20)

| Feature | Investigation | Notes |
|---------|--------------|-------|
| Prometheus | `ceph-prometheus-alerts` now in image; test Rook mode | Likely config/exposure issue |
| Loki | Test mgr Loki module in Rook pod | May need Rook-side config |
| Alert Manager | Related to Prometheus | Test jointly |
| Node Exporter | Typically Rook sidecar; document the pattern | May be a docs issue |
| Hosts (maintenance) | Investigate Rook RBAC / networking issue | |
| iSCSI | `ceph-iscsi` installed; test tcmu-runner in Rook pod context | |

**Exit criteria:** All previously-red Rook matrix entries are green or documented.

---

### Phase 6 — Ecosystem Leadership (Ongoing from Phase 2)

| Task | Detail |
|------|--------|
| MicroCeph patch tracking | Monitor; remove patches when Tentacle fixes land in Ubuntu packages |
| Track LP #2150762 | Remove PPA when Tentacle enters Ubuntu main |
| Charmed Ceph / MicroCeph reference | Designate this as the reference image |
| Cephadm docs contribution | Add Ubuntu image alternative to cephadm docs |
| Rook docs contribution | Add to Rook Ubuntu deployment guide |
| Community blog | Publish canonical.com post on Ubuntu Ceph ROCK |

---

## 7. Canonical Differentiators

**Ubuntu LTS + ESM security guarantee** — 10-year package maintenance; USN rebuilds can be automated to be faster than upstream's 24h cadence. No other Ceph image provider can credibly offer this.

**Rockcraft hermetic builds** — Given the same `rockcraft.yaml` and package versions, the result is reproducible. Upstream `Containerfile` builds with `dnf install` vary with mirror state. SBOM generation is natural in this model.

**MicroCeph / Charmed Ceph alignment** — Patches in `patches/` are adapted from MicroCeph. This creates a shared maintenance surface that no upstream project can replicate.

**Pebble OCI-native supervision** — Upstream images have no `CMD/ENTRYPOINT`. Pebble provides structured health checks, log routing, and a REST API. Once graduated, this is unique in the Ceph container ecosystem.

**GitHub Actions CI (community-accessible)** — Upstream uses Jenkins + Shaman, opaque and not forkable. This repo invites and enables community contributions.

**Three-tier publish pipeline** — Edge/hotfix/release model enables faster developer feedback than the upstream Jenkins pipeline.

---

## 8. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Tentacle packages not in Ubuntu main (LP #2150762) | Certain (current) | Medium | PPA in place; track bug; remove when resolved |
| Squid/Reef not in Noble archives | Medium | Medium | Use Jammy base for stable/reef; check noble-backports for Squid |
| arm64 build times on GitHub Actions | High | Medium | Canonical self-hosted arm64 runners or QEMU |
| Rockcraft `latest/edge` instability | Medium | High | Pin to `latest/stable` (Phase 1) |
| Upstream patches 0001–0003 conflict after Ceph update | Medium | Low | `--forward --dry-run` guard in overlay-script handles this gracefully |
| Rook API changes break canary tests | Medium | Medium | Pin Rook version in CI; track Rook changelog |
| Another project fills Ubuntu container vacuum | Low (we're already here) | Medium | Publish blog + registry claims quickly |
| cephadm defaults to upstream images for Ubuntu | Medium | Medium | Contribute Ubuntu alternative to cephadm docs |

---

## 9. Success Metrics

### Milestone 1 (Phase 1)
- [ ] Zero deprecated `set-output` / v3 actions in workflows
- [ ] All publish workflows use stable rockcraft and LXD channels
- [ ] kubectl version verified and current

### Milestone 2 (Phase 2)
- [ ] `stable/squid` and `stable/reef` branches created; CI passes
- [ ] `squid-edge`, `reef-edge`, `tentacle-edge` images in GHCR
- [ ] README branch/version table updated

### Milestone 3 (Phase 3)
- [ ] Multi-arch manifest published for all active branches
- [ ] arm64 Cephadm + Rook CI tests passing

### Milestone 4 (Phase 4)
- [ ] Every image is cosign-signed; SBOM attached as OCI referrer
- [ ] Images on GHCR + Quay.io + Docker Hub
- [ ] CVE rebuild within 48h of USN

### Milestone 5 (Phase 5)
- [ ] All Rook matrix entries green or documented
- [ ] Pebble entrypoint CI-tested

### Adoption metrics
- GHCR pull count (target: 10k/month within 6 months of arm64 release)
- GitHub stars (target: 100 within 6 months)
- PRs from non-Canonical contributors
- Rook / cephadm docs referencing this image

---

## Appendix A — Ceph Release Reference

| Release | Codename | Status | EOL | Branch |
|---------|----------|--------|-----|--------|
| 17.x | Quincy | EOL | 2024 | Skip |
| 18.x | Reef | Maintenance / approaching EOL | ~2026 | `stable/reef` (low priority) |
| 19.x | Squid | **Stable** | ~2027 | `stable/squid` (high priority) |
| 20.x | Tentacle | **Current stable** (v20.2.1, Apr 2026) | TBD | `main` (done) |

## Appendix B — The Archived Upstream Opportunity

`ceph/ceph-container` archival (Dec 2024) created a vacuum:

1. No Ubuntu Ceph OCI images exist upstream — CentOS/Rocky Linux only from `quay.io/ceph/ceph`
2. cephadm links to upstream images; Ubuntu Server users get RHEL-based images
3. Rook has an `image:` override field — Ubuntu clusters can and should use this repo
4. Docker Hub `ceph/ceph` frozen at v16 — claiming `docker.io/canonical/ceph` fills a visible gap

**Immediate actions:** Open PRs in cephadm and Rook docs listing `ghcr.io/canonical/ceph` as the Ubuntu alternative; claim Docker Hub and Quay.io namespaces; publish canonical.com/blog post.

---

*Update this document when phases complete.*
