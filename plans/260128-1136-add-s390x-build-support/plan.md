---
title: "Add s390x Build Support"
description: "Add IBM System z (s390x) architecture to build matrix for rancher-webhook fork"
status: completed
priority: P2
effort: 2-4 hours
branch: main
tags: [ci, s390x, multi-arch, fork]
created: 2026-01-28
---

# Add s390x Build Support

## Context

Fork of rancher/webhook. Need to:
1. Add s390x architecture support
2. Sync releases from upstream (rancher/webhook)
3. Disable upstream-specific workflows
4. Publish multi-arch images to GHCR

### Current Workflows

| Workflow | Purpose | Action |
|----------|---------|--------|
| `ci.yaml` | CI tests | **KEEP** - add s390x |
| `fossa.yml` | License scan (Rancher) | **DISABLE** |
| `publish-head.yaml` | Push to Docker Hub | **DISABLE** |
| `release-charts.yaml` | Bump in charts | **DISABLE** |
| `release-rancher.yaml` | Bump in rancher | **DISABLE** |
| `release.yaml` | Release to Docker Hub | **DISABLE** |
| `renovate-vault.yml` | Rancher Vault | **DISABLE** |
| `sync-deps.yaml` | Sync dependencies | **DISABLE** |

### New Workflows

| Workflow | Purpose |
|----------|---------|
| `sync-rancher.yaml` | Sync releases from rancher/webhook |
| `release-s390x.yaml` | Build s390x + create multi-arch manifest |

---

## Architecture

```
rancher/webhook (upstream)
        │
        │ releases v0.6.0
        ▼
┌─────────────────────────────────────────┐
│      sync-rancher.yaml (daily)          │
│  Check upstream → Create matching tag   │
└─────────────────────────────────────────┘
        │ triggers
        ▼
┌─────────────────────────────────────────┐
│      release-s390x.yaml                 │
│  1. Build s390x only (QEMU)             │
│  2. Pull amd64/arm64 from Docker Hub    │
│  3. Retag + Create manifest → GHCR      │
└─────────────────────────────────────────┘
```

---

## Implementation Plan

### Phase 1: Disable Upstream Workflows

Rename to `.yml.disabled`:
- `fossa.yml`
- `publish-head.yaml`
- `release-charts.yaml`
- `release-rancher.yaml`
- `release.yaml`
- `renovate-vault.yml`
- `sync-deps.yaml`

### Phase 2: Update CI Workflow

Modify `ci.yaml`:
- Add s390x to build matrix
- Use QEMU for s390x on x64 runner
- Skip integration tests for s390x (QEMU too slow)

```yaml
matrix:
  archBox:
  - { arch: amd64, vmArch: x64, run_integration: true }
  - { arch: arm64, vmArch: arm64, run_integration: true }
  - { arch: s390x, vmArch: x64, run_integration: false }
```

### Phase 3: Create Sync Workflow

New `sync-rancher.yaml`:
- Daily check for new upstream releases
- Create matching local release tag
- Triggers release-s390x workflow

### Phase 4: Create Release Workflow

New `release-s390x.yaml`:
- Triggered by release events
- Build s390x binary + image via QEMU
- Pull amd64/arm64 from `rancher/rancher-webhook` on Docker Hub
- Create multi-arch manifest on GHCR

### Phase 5: Update Makefile

Add s390x arch detection:
```makefile
else ifeq ($(UNAME_ARCH),s390x)
    ARCH ?= s390x
```

---

## Files to Change

| File | Action |
|------|--------|
| `Makefile` | Add s390x arch detection |
| `.github/workflows/ci.yaml` | Add s390x to matrix |
| `.github/workflows/sync-rancher.yaml` | CREATE - sync upstream |
| `.github/workflows/release-s390x.yaml` | CREATE - s390x release |
| `.github/workflows/fossa.yml` | RENAME to .disabled |
| `.github/workflows/publish-head.yaml` | RENAME to .disabled |
| `.github/workflows/release-charts.yaml` | RENAME to .disabled |
| `.github/workflows/release-rancher.yaml` | RENAME to .disabled |
| `.github/workflows/release.yaml` | RENAME to .disabled |
| `.github/workflows/renovate-vault.yml` | RENAME to .disabled |
| `.github/workflows/sync-deps.yaml` | RENAME to .disabled |

---

## Constraints

| Constraint | Impact |
|------------|--------|
| No native s390x runners | Use QEMU on x64 |
| QEMU emulation | 3-10x slower builds |
| No Rancher Vault access | Use GHCR + GITHUB_TOKEN |
| Upstream images on Docker Hub | Pull for manifest |

---

## Environment Variables

| Variable | Value |
|----------|-------|
| REGISTRY | ghcr.io |
| IMAGE_NAME | ${{ github.repository_owner }}/rancher-webhook |
| UPSTREAM_IMAGE | rancher/rancher-webhook |
| UPSTREAM_REPO | rancher/webhook |

---

## Todo List

- [x] Disable upstream workflows (7 files)
- [x] Update Makefile for s390x
- [x] Update ci.yaml with s390x matrix
- [x] Create sync-rancher.yaml
- [x] Create release-s390x.yaml
- [ ] Test CI on feature branch
- [ ] Verify GHCR publish works

---

## Success Criteria

1. CI builds pass for all 3 architectures
2. Sync workflow detects new upstream releases
3. Release workflow creates multi-arch manifest on GHCR
4. Image `ghcr.io/dnse-tech/rancher-webhook:TAG` includes amd64/arm64/s390x

---

## Risks

| Risk | Mitigation |
|------|------------|
| QEMU build timeout | Increase job timeout to 60min |
| Base image no s390x | Verify SUSE images support s390x |
| Upstream tag format change | Monitor and adjust sync script |

---

## Validation Summary

**Validated:** 2026-01-28
**Questions asked:** 6

### Confirmed Decisions

| Decision | Choice |
|----------|--------|
| s390x integration tests | Skip (unit tests + build sufficient) |
| Sync scope | Tags only (v* releases) |
| Sync frequency | Daily at 17:00 UTC |
| Missing upstream images | Fail and notify (no retry) |
| Disabled workflows | Rename to .disabled |
| Notifications | None needed |

### Action Items

- [x] Ensure release-s390x.yaml fails clearly when upstream images missing
- [x] Add descriptive error message for missing image scenario

### Ready for Implementation

All key decisions confirmed. Plan is ready to proceed.
