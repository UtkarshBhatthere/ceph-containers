# Ceph OCI Image Builder — Strategic Competition Plan

> **Branch:** `claude/ceph-oci-strategy-review-86l4n`  
> **Date:** May 2026  
> **Scope:** Full competitive analysis of canonical/ceph-containers vs upstream ceph/ceph-container, with a prioritised roadmap to close gaps and assert strategic differentiation.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State — This Repository](#2-current-state--this-repository)
3. [Upstream Landscape — ceph/ceph-container](#3-upstream-landscape--cephceph-container)
4. [Competitive Gap Analysis](#4-competitive-gap-analysis)
5. [Strategic Objectives](#5-strategic-objectives)
6. [Prioritised Roadmap](#6-prioritised-roadmap)
7. [Canonical Differentiators (Strengths to Leverage)](#7-canonical-differentiators-strengths-to-leverage)
8. [Risks & Mitigations](#8-risks--mitigations)
9. [Success Metrics](#9-success-metrics)

---

## 1. Executive Summary

`canonical/ceph-containers` is a Rockcraft-based OCI image builder for Ceph targeting Ubuntu LTS, with first-class support for Cephadm and Rook. As of mid-2026 the project is **~2.5 years stale** (last commit September 2023), carries a PPA workaround, supports only `amd64`, and publishes a single, unversioned Ceph image.

The upstream `ceph/ceph-container` project (Red Hat / Ceph community) ships multi-version, multi-arch, multi-distro images on a rolling release cadence, distributed from Quay.io and Docker Hub.

The canonical project has a genuine, defensible strategic position: Ubuntu provenance, LTS security guarantees, Rockcraft's hermetic builds, and the Canonical charm/snap ecosystem. None of this matters while the image is two major Ceph versions behind and lacks arm64 support.

**The plan:** Ship a modern, multi-version, multi-arch Ceph ROCK image that matches upstream's version coverage, then beat it on Ubuntu-specific quality, security cadence, and Canonical toolchain integration.

---

## 2. Current State — This Repository

### 2.1 Architecture

```
rockcraft.yaml          ← single build definition, Ubuntu 22.04 base
docker_scripts/         ← 33 bash scripts (entrypoint, daemon starters, OSD scenarios)
confd/                  ← etcd-backed dynamic config templates
kubectl/                ← Go source for kubectl binary bundled in image
test/deploy.py          ← Python script: single-node LXD/cephadm deployment
.github/workflows/      ← build_and_test, publish_image, publish_temp_image, cla-check
```

### 2.2 What Works Well

| Strength | Detail |
|----------|--------|
| Full daemon coverage | 29 entrypoint scenarios (mon, osd, mds, mgr, rgw, nfs, rbd-mirror, iSCSI, …) |
| Dual-backend CI | Cephadm tests (direct host + LXD) and Rook canary tests |
| PR preview images | `/publish` comment triggers GHCR push from PRs |
| Developer tooling | HACKING.md, `test/deploy.py`, confd templates |
| Pebble integration | OCI-native entrypoint (experimental but unique) |
| Apache 2.0 + Canonical CLA | Clean IP status |

### 2.3 Current Limitations

| Area | Issue |
|------|-------|
| **Staleness** | Last commit Sep 2023; Ceph Reef 18.x (LTS), Squid 19.x, and the approaching Tentacle 20.x are all absent |
| **Version handling** | `version: '0.1'` is hardcoded; no Ceph release version in image tag |
| **Architecture** | `amd64` only; arm64 absent despite growing k8s/edge demand |
| **Base OS** | Ubuntu 22.04 only; Ubuntu 24.04 LTS (Noble) now available |
| **PPA dependency** | `ppa:utkarshbhatthere/ceph-oci` workaround for LP bug #2003704; adds supply-chain risk and breaks reproducibility |
| **Registry** | GHCR only; no Quay.io or Docker Hub presence |
| **Rook parity** | Prometheus, Loki, Alert Manager, Node Exporter, iSCSI all marked red in README matrix |
| **Security cadence** | No automated CVE-rebuild workflow; zero Dependabot alerts addressed beyond a dependency bump |
| **kubectl version** | Pinned to v0.27.2 (k8s 1.27 EOL'd in June 2024) |
| **confd** | v0.16.0 from 2018; project is deprecated upstream |
| **CI tooling age** | `actions/checkout@v3`, `actions/upload-artifact@v3` (v4 released); Minikube 1.28 / k8s 1.25 |

---

## 3. Upstream Landscape — ceph/ceph-container

### 3.1 Project Overview

`ceph/ceph-container` is the canonical upstream OCI image project maintained by Red Hat and the broader Ceph community. It generates images consumed by Rook, Cephadm, and direct `docker run` usage.

### 3.2 Key Characteristics

| Dimension | Upstream approach |
|-----------|-------------------|
| **Build system** | GNU Make + Jinja2 templates; `Makefile` drives per-distro, per-version builds |
| **Base images** | Ubuntu 22.04, CentOS Stream 9, openSUSE (distro matrix) |
| **Ceph versions** | Pacific (EOL), Quincy (maintenance), Reef 18.x (LTS), Squid 19.x (stable) |
| **Architectures** | `amd64`, `arm64` |
| **Registries** | `quay.io/ceph/ceph`, Docker Hub `ceph/ceph` |
| **Tag scheme** | `v{ceph-version}` (e.g. `v18.2.4`), `v{major}` rolling, `latest` |
| **Security** | Regular CVE rebuilds via Zuul CI; Ceph tracker issue integration |
| **Entrypoint** | `entrypoint.sh` + `demo.sh`; no pebble |
| **Testing** | Zuul-based; ceph-ansible, cephadm, Rook integration tests |
| **Community** | 50+ contributors; releases tracked with Ceph point releases |

### 3.3 Upstream Strengths

- Version matrix: every supported Ceph release has a tagged image within days of upstream release
- Multi-distro: enterprise users on RHEL/CentOS get their preferred base
- Arm64 support: covers edge, Raspberry Pi clusters, Graviton AWS nodes
- Quay.io distribution: preferred by OpenShift/Kubernetes communities
- Deep Zuul CI: pre-merge integration testing against real Ceph clusters

### 3.4 Upstream Weaknesses (Canonical's Opening)

- **No Ubuntu LTS security maintenance guarantees** — upstream Ubuntu images get packages but not the full Ubuntu Pro / ESM security pipeline
- **No Rockcraft / hermetic builds** — supply chain provenance is weaker; no SBOM
- **No pebble** — upstream has no OCI-native init integration
- **No charm/snap ecosystem alignment** — Charmed Ceph, MicroCeph, and the broader Juju ecosystem have no upstream analog
- **No Launchpad SRU alignment** — Ubuntu package fixes don't flow into upstream images predictably
- **Distro sprawl** — maintaining Ubuntu + CentOS + openSUSE paths increases complexity and dilutes quality per distro
- **No ROCK format** — ROCK images carry richer OCI metadata (base layer provenance, Pebble service definitions, etc.)

---

## 4. Competitive Gap Analysis

### 4.1 Critical Gaps (Block users from choosing this repo)

| Gap | Impact | Effort |
|-----|--------|--------|
| Missing Ceph Reef (18.x) image | HIGH — Reef is the current LTS; all new deployments use it | Medium |
| Missing Ceph Squid (19.x) image | HIGH — stable release; Rook defaults changing | Medium |
| amd64-only | HIGH — arm64 is now required for many k8s environments | Medium-High |
| Ubuntu 22.04 only | MEDIUM — 24.04 Noble now provides longer lifecycle | Medium |
| PPA workaround | MEDIUM — LP bug #2003704 resolved in Jammy updates; remove PPA | Low |
| kubectl v1.27 EOL | MEDIUM — security risk for k8s users | Low |

### 4.2 Significant Gaps (Degrade experience vs upstream)

| Gap | Impact | Effort |
|-----|--------|--------|
| No Quay.io / Docker Hub presence | MEDIUM — Rook/OpenShift users default to quay.io | Low |
| No version-pinned image tags | MEDIUM — reproducible deployments impossible | Low |
| Rook monitoring stack unsupported | MEDIUM — limits Rook production usability | High |
| No automated CVE rebuild | MEDIUM — security posture deteriorates over time | Medium |
| Stale CI action versions | LOW — minor, causes deprecation warnings | Low |
| confd v0.16.0 deprecated | LOW — works but unmaintained | Low |

### 4.3 Feature Parity Matrix (vs upstream)

| Feature | This repo | Upstream |
|---------|-----------|----------|
| Ceph Pacific (EOL) | ❌ | ✅ (EOL images kept) |
| Ceph Quincy | ❌ | ✅ |
| Ceph Reef (LTS) | ❌ | ✅ |
| Ceph Squid | ❌ | ✅ |
| Ubuntu 22.04 base | ✅ | ✅ |
| Ubuntu 24.04 base | ❌ | ❌ (not yet) |
| CentOS Stream base | ❌ (by design) | ✅ |
| amd64 | ✅ | ✅ |
| arm64 | ❌ | ✅ |
| GHCR | ✅ | ❌ |
| Quay.io | ❌ | ✅ |
| Docker Hub | ❌ | ✅ |
| Version-pinned tags | ❌ | ✅ |
| Pebble entrypoint | ✅ (experimental) | ❌ |
| Rockcraft / hermetic builds | ✅ | ❌ |
| SBOM / provenance | partial (Rockcraft) | ❌ |
| CVE rebuild automation | ❌ | ✅ (Zuul) |
| Cephadm CI | ✅ | partial |
| Rook CI | ✅ | ✅ |
| Prometheus/Loki/AM (Rook) | ❌ | ✅ |
| Charm/snap ecosystem | ✅ (potential) | ❌ |
| Ubuntu Pro / ESM integration | ✅ (potential) | ❌ |

---

## 5. Strategic Objectives

### Objective 1 — Version Currency (table stakes)
Ship images for every actively maintained Ceph release (Reef 18.x, Squid 19.x, and Tentacle 20.x when released) with tags that reflect the Ceph point-release version.

### Objective 2 — Multi-Architecture
Add arm64 builds to match upstream and unlock edge computing, Graviton, and Apple Silicon development use cases.

### Objective 3 — Ubuntu LTS Lifecycle Leadership
Be the definitive Ubuntu Ceph image: align with Ubuntu Pro/ESM, provide Noble (24.04) base, and surface Ubuntu-specific security data in image metadata.

### Objective 4 — Supply-Chain & Build Integrity
Leverage Rockcraft's hermetic build model to produce verifiable SBOMs and signed images — something upstream cannot easily match.

### Objective 5 — Ecosystem Integration
Become the reference image for Charmed Ceph, MicroCeph, and Juju-deployed Ceph, creating a flywheel no upstream project can replicate.

### Objective 6 — Registry Presence
Meet users where they are: publish to GHCR, Quay.io, and Docker Hub simultaneously.

---

## 6. Prioritised Roadmap

### Phase 1 — Foundation (Weeks 1–4)

**Goal:** Unblock immediate blockers; produce a usable Reef image.

| Task | Owner area | Notes |
|------|-----------|-------|
| Remove PPA dependency | Build | LP bug #2003704 is fixed in jammy-updates; switch to `noble` or update package list |
| Upgrade base to Ubuntu 24.04 (Noble) | Build | `base: ubuntu:24.04`; verify all packages available |
| Add Ceph Reef 18.x package list | Build | Pin to current LTS point release; source from noble/jammy |
| Add Ceph version to `rockcraft.yaml` version field | Build | `version: '18.2.x'`; drives image tag |
| Fix image tag scheme | CI | Tag as `{ceph-major}.{ceph-minor}` + branch/sha for rolling |
| Update CI actions to v4 | CI | `actions/checkout@v4`, `upload-artifact@v4`, etc. |
| Update kubectl to v1.29+ | Build | Match current k8s stable |
| Push to GHCR with versioned tags | CI | `ghcr.io/canonical/ceph:18.2.4`, `:18`, `:latest` |

**Exit criteria:** A tagged `18.2.x` Reef image passes all existing CI tests and is pushed to GHCR.

---

### Phase 2 — Multi-Version & Multi-Arch (Weeks 5–10)

**Goal:** Match upstream version coverage; add arm64.

| Task | Owner area | Notes |
|------|-----------|-------|
| Parameterise `rockcraft.yaml` by Ceph version | Build | Matrix of Reef + Squid; consider `yq`-based templating or separate rockcraft files per series |
| Add version matrix to CI | CI | GitHub Actions matrix on `ceph-version` |
| Add `arm64` platform to `rockcraft.yaml` | Build | `platforms: amd64: {} arm64: {}` |
| Enable cross-arch builds in CI | CI | `craft-actions/rockcraft-pack` + QEMU or native runners |
| Publish multi-arch manifest | CI | `docker manifest create` or `skopeo` multi-arch push |
| Add Squid 19.x image | Build | Separate package list / version file |
| Verify confd replacement or version pin | Build | Consider replacing confd v0.16.0 with `go-confd` fork or environment-variable-native approach |

**Exit criteria:** Both `amd64` and `arm64` images exist for Reef and Squid; CI matrix builds and tests all four combinations.

---

### Phase 3 — Security & Distribution (Weeks 11–16)

**Goal:** Assert Ubuntu security leadership; reach users on all major registries.

| Task | Owner area | Notes |
|------|-----------|-------|
| Automated CVE rebuild workflow | CI | Weekly GitHub Actions scheduled job; rebuild if Ubuntu package digests change |
| SBOM generation | Build | Rockcraft/syft integration; attach to OCI image as attestation |
| Image signing | CI | Cosign (sigstore) signing in publish workflow |
| Publish to Quay.io | CI | Add `quay.io/canonical/ceph` push alongside GHCR |
| Publish to Docker Hub | CI | `docker.io/canonical/ceph` |
| Ubuntu Pro / ESM metadata | Build | Surface ESM patch status in image labels |
| Noble (24.04) as primary base | Build | Complete migration from Jammy once all packages verified |
| Security advisory tracking issue template | Docs | Link CVE rebuild workflow to GitHub issues |

**Exit criteria:** Images are automatically rebuilt within 48 hours of a new Ubuntu security update; SBOMs are attached; images are signed; all three registries have current versions.

---

### Phase 4 — Rook Feature Parity & Pebble Production (Weeks 17–24)

**Goal:** Eliminate red flags in the Rook compatibility matrix; graduate pebble entrypoint.

| Task | Owner area | Notes |
|------|-----------|-------|
| Enable Prometheus exporter inside image | Image | ceph-mgr-prometheus is already installed; wire into Rook tests |
| Enable Loki logging integration | Image | Verify `ceph-mgr-loki` availability; add to package list |
| Enable Node Exporter sidecar support | Docs/Image | Document how to deploy alongside Rook |
| Fix iSCSI support in Rook mode | Image | Investigate `ceph-iscsi` / `tcmu-runner` in Rook pod context |
| Graduate pebble entrypoint from experimental | Image | Add production test case; document in HACKING.md |
| Pebble service definitions for each daemon | Image | Replace `pebble_cmd.sh` with proper pebble layer files |
| Rook upgrade test in CI | CI | Extend canary test to validate rolling upgrade Reef→Squid |
| Cephadm upgrade test in CI | CI | Multi-version upgrade path validation |

**Exit criteria:** All matrix entries in README are green for both Cephadm and Rook; pebble entrypoint is tested in CI.

---

### Phase 5 — Ecosystem Leadership (Ongoing from Week 12)

**Goal:** Make this the canonical reference image for the Canonical ecosystem.

| Task | Owner area | Notes |
|------|-----------|-------|
| MicroCeph integration testing | Test | Validate image against MicroCeph; surface in CI |
| Charmed Ceph charm reference | Docs | Document how to use ROCK as charm's OCI image |
| Launchpad SRU alignment | Process | Add workflow that monitors Ubuntu package SRUs and triggers rebuilds |
| Ubuntu Pro attestation | Build | Embed Ubuntu Pro subscription status in image labels |
| Developer experience | Docs | `make` wrapper around rockcraft for contributors; update HACKING.md |
| Upstream contribution | Community | Contribute Ubuntu 24.04 support back to ceph/ceph-container |

---

## 7. Canonical Differentiators (Strengths to Leverage)

### 7.1 Ubuntu LTS Security Guarantee
Canonical maintains Ubuntu packages under ESM for 10 years. No other Ceph image provider can credibly offer this. Surface it:
- Image labels: `org.opencontainers.image.vendor=Canonical`, `com.canonical.ubuntu.esm=true`
- Automated rebuild within 24h of USN (Ubuntu Security Notice)
- SBOM attached to every image (impossible for distro-agnostic Makefile-based builds)

### 7.2 Rockcraft — Hermetic, Reproducible Builds
Upstream ceph/ceph-container uses standard Dockerfiles with `apt-get` inside layers. Layer contents vary with mirror state. Rockcraft's LXD-isolated builds are hermetic: given the same `rockcraft.yaml` and package versions, the result is bit-for-bit reproducible. This is a compliance/audit selling point.

### 7.3 Pebble — OCI-Native Process Supervision
Upstream containers use raw `bash` entrypoints or `s6-overlay`. Pebble provides:
- Structured service health checks
- Log routing
- On-demand service startup
- REST API for container management
Once graduated from experimental, this is a genuine technical differentiator.

### 7.4 Charmed Ceph / MicroCeph Alignment
No upstream project has a story for Juju-based deployment or MicroCeph integration. Position this ROCK as the `image:` reference for both charms, creating a sticky Canonical ecosystem dependency.

### 7.5 Canonical's Developer Channel
Ubuntu developers reaching for `apt install ceph` are one conceptual step from `docker pull ghcr.io/canonical/ceph`. Leverage ubuntu.com, discourse, and the snap store discovery surface to drive adoption that upstream cannot access.

---

## 8. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Rockcraft toolchain immaturity | Medium | High | Pin to stable rockcraft channel; maintain fallback Dockerfile |
| PPA / package availability delays | Medium | High | Engage Ubuntu Ceph package maintainers; track noble-proposed |
| arm64 build times on GitHub Actions | High | Medium | Use Canonical self-hosted arm64 runners or cross-compilation |
| Upstream ceph/ceph-container adds Ubuntu Pro support | Low | High | Move faster on Phases 3–4; focus on Rockcraft/SBOM uniqueness |
| Resource / maintainer bandwidth | High | High | Define minimal viable maintainer surface (Phase 1 only needs 1 engineer) |
| confd deprecation | Low | Low | Replace with native env-var config or go-confd fork in Phase 2 |
| Rook project changes Ceph image expectations | Medium | Medium | Track Rook changelog; add Rook release matrix to CI |

---

## 9. Success Metrics

### Milestone 1 (Phase 1 complete)
- [ ] `ghcr.io/canonical/ceph:18.2.x` image published and tagged
- [ ] All existing CI tests pass on Reef + Noble base
- [ ] PPA dependency removed
- [ ] Image version field reflects actual Ceph version

### Milestone 2 (Phase 2 complete)
- [ ] `amd64` + `arm64` images built and published for Reef and Squid
- [ ] CI matrix covers 4 combinations (2 versions × 2 arches)
- [ ] Image manifests are multi-arch (single pull URI)

### Milestone 3 (Phase 3 complete)
- [ ] CVE rebuild fires automatically; rebuild turnaround < 48h from USN
- [ ] Images signed with cosign; verification documented
- [ ] SBOM attached to every published image
- [ ] Quay.io and Docker Hub registries live

### Milestone 4 (Phase 4 complete)
- [ ] All README compatibility matrix entries green for Rook
- [ ] Pebble entrypoint tested in CI (not experimental)
- [ ] Upgrade path tested in CI (Reef → Squid)

### Adoption metrics (ongoing)
- GHCR pull count (target: 10k/month within 6 months of Phase 2)
- GitHub stars (target: 100 within 6 months)
- Issues/PRs from non-Canonical contributors
- Rook / Cephadm upstream documentation references

---

## Appendix A — File Reference Map

| File | Strategic relevance |
|------|---------------------|
| `rockcraft.yaml` | Central build definition; all version/arch/base changes land here |
| `docker_scripts/entrypoint.sh.in` | Core entrypoint; pebble graduation work targets `docker_scripts/pebble/` |
| `docker_scripts/variables_entrypoint.sh` | 100+ env vars; review for Rook parity gaps |
| `docker_scripts/common_functions.sh` | Utility library; arm64 path stubs exist but untested |
| `confd/` | Replace or upgrade in Phase 2 |
| `kubectl/kubectl.go` | Bump k8s dependency; validate go module security |
| `test/deploy.py` | Local dev loop; extend for multi-version testing |
| `.github/workflows/build_and_test.yaml` | Major CI overhaul target for Phase 2 |
| `.github/workflows/publish_image.yaml` | Multi-registry push target for Phase 3 |

## Appendix B — Ceph Release Schedule Reference

| Release | Codename | Status | EOL |
|---------|----------|--------|-----|
| 16.x | Pacific | EOL | Jun 2023 |
| 17.x | Quincy | Maintenance | Jun 2024 |
| 18.x | Reef | **LTS** | ~2026 |
| 19.x | Squid | Stable | ~2026 |
| 20.x | Tentacle | Development (2025+) | TBD |

**Priority:** Ship Reef (18.x) first, Squid (19.x) second, monitor Tentacle for Phase 4+.

## Appendix C — Branch Strategy

```
main
  └─ feature/phase1-reef-noble        ← Phase 1 work
  └─ feature/phase2-multiarch         ← Phase 2 work
  └─ feature/phase3-security          ← Phase 3 work
  └─ feature/phase4-rook-parity       ← Phase 4 work
```

Each phase branch is merged to `main` and triggers a new versioned image publication.

---

*This document lives in the repository at `STRATEGY.md` and should be updated as phases complete and priorities shift.*
