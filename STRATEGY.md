# Ceph OCI Image Builder — Strategic Competition Plan

> **Branch:** `claude/ceph-oci-strategy-review-86l4n`  
> **Date:** May 2026  
> **Scope:** Full competitive analysis of canonical/ceph-containers vs the Ceph community container build (formerly ceph/ceph-container, now ceph/ceph `container/`), with a prioritised roadmap to close gaps and assert strategic differentiation.  
> **⚠ Key finding:** `ceph/ceph-container` was **archived December 19, 2024**. All upstream container development moved into the main `ceph/ceph` repository. Canonical has a narrow window to fill the Ubuntu container vacuum.

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

**The upstream picture has fundamentally changed.** The dedicated `ceph/ceph-container` repository was **archived on December 19, 2024**. All container build assets migrated into the primary `ceph/ceph` monorepo under `container/` — a plain Containerfile per branch, shell scripts, and a Jenkins/Shaman pipeline. The base OS is CentOS Stream 9 (Reef, Squid) and Rocky Linux 10 (Tentacle). Docker Hub is frozen at v16 (no new pushes since 2021). Active releases are distributed from `quay.io/ceph/ceph` only.

This creates a **strategic vacuum**: the Ceph community has no actively maintained, Ubuntu-native OCI image builder. Canonical is already positioned to fill it — if the project is modernised promptly.

The canonical project has a genuine, defensible strategic position: Ubuntu provenance, LTS security guarantees, Rockcraft's hermetic builds, and the Canonical charm/snap ecosystem. None of this matters while the image is two major Ceph versions behind and lacks arm64 support.

**The plan:** Modernise fast — ship Squid (19.x) and Tentacle (20.x) images immediately, add arm64, assert Ubuntu-native leadership, and become the definitive community alternative to the RHEL/CentOS-based upstream. The window is narrow: if another Ubuntu-based community project fills the vacuum first, the opportunity is lost.

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

## 3. Upstream Landscape — ceph/ceph (formerly ceph/ceph-container)

### 3.1 Project Overview — Key Status Change

> **`ceph/ceph-container` was archived (read-only) on December 19, 2024.** All container build assets are now inside the primary `ceph/ceph` monorepo under the `container/` directory. There is no active standalone container repository.

The previous project produced `ceph/daemon-base` (runtime image) and `ceph/daemon` (with bash entrypoint scripts for ceph-ansible/ceph-nano). The new model is a single `Containerfile` per release branch inside `ceph/ceph`, built and published by a Jenkins/Shaman CI pipeline.

### 3.2 Current Build Architecture (ceph/ceph `container/`)

| File | Purpose |
|------|---------|
| `container/Containerfile` | Per-branch image definition |
| `container/build.sh` | Build/push orchestration; driven by environment variables; uses `podman` |
| `container/make-manifest-list.py` | Multi-arch manifest creation and `--promote` to production registry |

Active branches (each has its own Containerfile):

| Branch | Release | Status |
|--------|---------|--------|
| `main` | Tentacle v20.2.x | Current stable (released April 2026) |
| `squid` | v19.2.x | Stable |
| `reef` | v18.2.x | **Approaching EOL** |
| `quincy` | v17.2.x | EOL |

### 3.3 Key Characteristics

| Dimension | Upstream approach |
|-----------|-------------------|
| **Build system** | Shell scripts + Python (`build.sh`, `make-manifest-list.py`); **no Makefile** |
| **Base images** | CentOS Stream 9 (reef, squid); Rocky Linux 10 (tentacle/main) — **no Ubuntu** |
| **Architectures** | `amd64`, `arm64` |
| **Registries** | `quay.io/ceph/ceph` (active); Docker Hub `ceph/ceph` frozen at v16 since 2021 |
| **Tag scheme** | `v{major}.{minor}.{patch}-{date}` (e.g. `v19.2.3-20250717`); `v{major}` rolling |
| **Security** | Rebuilt within 24h of new Ceph package; Jenkins/Shaman pipeline |
| **Entrypoint** | **No CMD/ENTRYPOINT defined** — orchestrators (cephadm, Rook) inject entrypoints |
| **CI** | Jenkins + Jenkins Job Builder (JJB) in `ceph/ceph-build`; **not GitHub Actions** |
| **Package source** | Shaman (https://shaman.ceph.com) for CI builds; download.ceph.com for releases |
| **OSD flavors** | `default` and `crimson` (tech preview → more stable in Tentacle) |

### 3.4 Upstream Strengths

- Near-daily rebuilds: images track Ceph point releases within 24 hours
- Multi-arch: `amd64` + `arm64` manifest lists from day one of each release
- Quay.io: preferred registry for OpenShift, Kubernetes, and RHEL-based environments
- Jenkins/Shaman integration: deep pipeline to Ceph's own package build infrastructure
- Crimson OSD image: separate experimental flavor for next-gen OSD users

### 3.5 Upstream Weaknesses — Canonical's Strategic Opening

| Weakness | Canonical's advantage |
|----------|-----------------------|
| **CentOS/Rocky Linux only** — no Ubuntu base | Ubuntu LTS, ESM, and Pro security pipeline are exclusive to Canonical |
| **Archived standalone repo** — community confusion about where to find containers | A clearly maintained, standalone Ubuntu container repo fills the void |
| **No Rockcraft / hermetic builds** — Containerfile + apt/dnf inside layers is non-reproducible | Rockcraft produces hermetic, bit-for-bit reproducible builds with SBOM |
| **No Pebble** — no OCI-native process supervision | Pebble provides structured health checks, log routing, REST API |
| **No charm/snap ecosystem** — no Juju, Charmed Ceph, or MicroCeph story | These are Canonical-exclusive adoption surfaces |
| **Jenkins-based CI** — opaque, not community-forkable | GitHub Actions CI is accessible and forkable by any contributor |
| **Docker Hub frozen at v16** — community users who try Docker Hub get stale images | Canonical can claim Docker Hub `canonical/ceph` and Quay.io presence |
| **No Ubuntu LTS SRU alignment** — package security fixes don't flow predictably | Canonical controls the Ubuntu package lifecycle end-to-end |
| **No SBOM or image signing** — no provenance story | SBOM + cosign signing are first-class in Rockcraft/GitHub ecosystem |

---

## 4. Competitive Gap Analysis

### 4.1 Critical Gaps (Block users from choosing this repo)

| Gap | Impact | Effort |
|-----|--------|--------|
| Missing Ceph Tentacle (20.x) image | CRITICAL — current stable release (April 2026); upstream at v20.2.1 | Medium |
| Missing Ceph Squid (19.x) image | HIGH — stable release; many production deployments | Medium |
| Missing Ceph Reef (18.x) image | HIGH — still in use; approaching EOL | Low-Medium |
| amd64-only | HIGH — arm64 required for many k8s, edge, Graviton environments | Medium-High |
| Ubuntu 22.04 only | MEDIUM — Noble (24.04) offers longer lifecycle and newer packages | Medium |
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

### 4.3 Feature Parity Matrix (vs upstream ceph/ceph `container/`)

| Feature | This repo | Upstream |
|---------|-----------|----------|
| Ceph Quincy (EOL) | ❌ | ✅ (archived) |
| Ceph Reef (18.x, approaching EOL) | ❌ | ✅ |
| Ceph Squid (19.x, stable) | ❌ | ✅ |
| Ceph Tentacle (20.x, current stable) | ❌ | ✅ |
| Ubuntu base | ✅ | ❌ (CentOS/Rocky only) |
| Ubuntu 22.04 | ✅ | ❌ |
| Ubuntu 24.04 | ❌ | ❌ |
| CentOS Stream 9 base | ❌ (by design) | ✅ |
| Rocky Linux 10 base | ❌ (by design) | ✅ (Tentacle) |
| amd64 | ✅ | ✅ |
| arm64 | ❌ | ✅ |
| GHCR | ✅ | ❌ |
| Quay.io | ❌ | ✅ (`quay.io/ceph/ceph`) |
| Docker Hub (active) | ❌ | ❌ (frozen at v16) |
| Version-pinned tags | ❌ | ✅ (`v19.2.3-20250717`) |
| Defined OCI entrypoint | ✅ (pebble, experimental) | ❌ (no CMD defined) |
| Pebble process supervision | ✅ (experimental) | ❌ |
| Rockcraft / hermetic builds | ✅ | ❌ (Containerfile) |
| SBOM / provenance | partial (Rockcraft) | ❌ |
| Image signing (cosign) | ❌ | ❌ |
| CVE rebuild automation | ❌ | ✅ (<24h rebuild on new pkg) |
| GitHub Actions CI | ✅ | ❌ (Jenkins) |
| Cephadm CI | ✅ | ❌ |
| Rook CI | ✅ | ❌ |
| Prometheus/Loki/AM (Rook) | ❌ | n/a (no defined entrypoint) |
| Charm/snap ecosystem | ✅ (potential) | ❌ |
| Ubuntu Pro / ESM integration | ✅ (potential) | ❌ |
| Crimson OSD flavor | ❌ | ✅ (Tentacle) |

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

**Goal:** Unblock immediate blockers; produce usable Squid and Tentacle images on Ubuntu 24.04.

> Reef (18.x) is approaching EOL — it is not the priority target. Ship Squid (19.x) first as the stable release and Tentacle (20.x) as the current stable. Add Reef as a best-effort backport if packages are available on Noble.

| Task | Owner area | Notes |
|------|-----------|-------|
| Remove PPA dependency | Build | LP bug #2003704 fixed in jammy-updates; migrate to Noble packages |
| Upgrade base to Ubuntu 24.04 (Noble) | Build | `base: ubuntu:24.04`; verify Ceph packages available from Ubuntu archives |
| Add Ceph Squid (19.x) package list | Build | Pin to current stable point release; source from noble |
| Add Ceph Tentacle (20.x) package list | Build | Current upstream stable (v20.2.1); verify Noble availability |
| Add Ceph version to `rockcraft.yaml` version field | Build | `version: '19.2.x'`; drives image tag |
| Fix image tag scheme | CI | Tag as `v{ceph-version}` (e.g. `v19.2.3`) + `v{major}` rolling; match upstream convention |
| Update CI actions to v4 | CI | `actions/checkout@v4`, `upload-artifact@v4`, etc. |
| Update kubectl to v1.31+ | Build | Match current k8s stable |
| Push to GHCR with versioned tags | CI | `ghcr.io/canonical/ceph:v19.2.3`, `:v19`, `:latest` |
| Rename table of contents / README to reflect new scope | Docs | Prominently note Ubuntu-only, Rockcraft-based, and distinct from archived upstream |

**Exit criteria:** Tagged `v19.2.x` (Squid) and `v20.2.x` (Tentacle) images pass all existing CI tests and are pushed to GHCR.

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
| Ceph Squid/Tentacle packages not yet in Ubuntu Noble | Medium | High | Check packages.ubuntu.com; engage Ubuntu Ceph maintainers; track noble-proposed / backports |
| Rockcraft toolchain immaturity | Medium | High | Pin to stable rockcraft channel; maintain fallback Dockerfile |
| arm64 build times on GitHub Actions | High | Medium | Use Canonical self-hosted arm64 runners or cross-compilation via QEMU |
| Another project fills the Ubuntu Ceph container vacuum first | Medium | High | Move fast on Phase 1; claim Docker Hub and Quay.io `canonical/ceph` namespaces now |
| Resource / maintainer bandwidth | High | High | Phase 1 needs only 1 engineer; scope tightly before expanding |
| confd v0.16.0 deprecated | Low | Low | Replace with native env-var config or go-confd fork in Phase 2 |
| Rook project changes Ceph image expectations | Medium | Medium | Track Rook changelog; add Rook release matrix to CI |
| cephadm defaults to upstream Ceph images | Medium | Medium | Contribute to cephadm docs to list Canonical image as Ubuntu alternative |

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
| 15.x | Octopus | EOL | — |
| 16.x | Pacific | EOL | Jun 2023 |
| 17.x | Quincy | EOL | 2024 |
| 18.x | Reef | Maintenance / approaching EOL | ~2026 |
| 19.x | Squid | **Stable** | ~2027 |
| 20.x | Tentacle | **Current Stable** (v20.2.1, Apr 2026) | TBD |

**Priority:** Ship Squid (19.x) first (stable, most production deployments), then Tentacle (20.x) (current upstream). Add Reef (18.x) as backfill if Ubuntu Noble packages are available. Do NOT invest in Quincy or Pacific.

**Ubuntu package availability note:** Check `ubuntu.com/server/docs/service-ceph` and `packages.ubuntu.com` for Ceph package availability per Ubuntu LTS before committing to a version target.

## Appendix C — The Archived Upstream Opportunity

The archival of `ceph/ceph-container` (Dec 2024) creates a community vacuum that Canonical can fill:

1. **Community users searching for Ubuntu Ceph images have nowhere to go.** The only active upstream images are CentOS/Rocky Linux from `quay.io/ceph/ceph`. Ubuntu users who want OCI images must either build their own or adopt a non-Ubuntu base.

2. **cephadm defaults to upstream images.** If Canonical can get `ghcr.io/canonical/ceph` listed as an alternative in cephadm documentation, Ubuntu Server installs would naturally pull the Ubuntu image.

3. **Rook has an `image:` field.** Rook's documentation recommends overriding the default CentOS image for Ubuntu-based clusters. This is exactly the use case this project serves — and Rook docs could link directly to this repo.

4. **The Jenkins/Shaman pipeline is not community-forkable.** GitHub Actions CI in this repo is accessible to any contributor. A community that wants to contribute Ubuntu-native Ceph container improvements has no upstream home — this repo can be that home.

**Action:** File issues / open discussions in the Rook and cephadm projects pointing to this repo as the Ubuntu alternative image. Draft a community announcement for ubuntu.com/blog and ceph.io.

## Appendix D — Branch Strategy

```
main
  └─ feature/phase1-squid-tentacle-noble   ← Phase 1 work
  └─ feature/phase2-multiarch              ← Phase 2 work
  └─ feature/phase3-security              ← Phase 3 work
  └─ feature/phase4-rook-parity           ← Phase 4 work
```

Each phase branch is merged to `main` and triggers a new versioned image publication.

---

*This document lives in the repository at `STRATEGY.md` and should be updated as phases complete and priorities shift.*
