# Releasing `hanzozt/zt-builder`

This repo publishes the Ziti builder container image to Docker Hub.

The release process is designed to support:

- A **current major line** on `main` (currently v3)
- **Maintenance major lines** on branches named `v<MAJOR>` (e.g. `v1`)

## Concepts

### Branches

- `main`
  - Development branch for the current major line.
- `v<MAJOR>` (e.g. `v1`)
  - Maintenance branch for a previous major line.

### GitHub Releases

Finalized releases are identified by semver tags:

- `vMAJOR.MINOR.PATCH` (e.g. `v3.0.4`, `v1.0.15`)

A GitHub Release may be a draft or published.

### Docker tags

The `Docker` GitHub Action publishes these tags:

- `:main`
  - Produced on merges/pushes to `main`.
- `:v<MAJOR>-dev` and `:v<MAJOR>-<sha>`
  - Produced on merges/pushes to maintenance branches (e.g. `v1`).
  - These are **preview** tags and **do not** move the stable `:v<MAJOR>` tag.
- `:MAJOR.MINOR.PATCH`
  - Produced on semver tag pushes.
- `:vMAJOR`
  - Produced on semver tag pushes.
  - This is the **stable major tag** for consumers who want “latest release within a major line”.
- `:latest`
  - Only produced on semver tag pushes **when that tag is the highest published semver**.

## Workflows

### Drafting release notes and selecting the next version

Workflow: `.github/workflows/release-drafter.yml`

- Trigger: `push` to `main` and `v*` branches.

Behavior on maintenance branches (e.g. `v1`):

- Computes the next patch version by looking at existing GitHub Releases that match `v<MAJOR>.*.*`.
- Passes that computed version into Release Drafter, so the draft version is deterministic (e.g. next `v1` patch becomes `v1.0.15`).
- Uses `commitish: refs/heads/<branch>`.

Behavior on `main`:

- Runs Release Drafter normally using label-based semver resolution.

Notes:

- Release Drafter uses the GitHub Release `target_commitish` (aka commitish) to associate a draft release with a branch.

### Publishing Docker images

Workflow: `.github/workflows/docker-publish.yml`

Triggers:

- `push` to branches `main` and `v*`
- `push` to tags `v*.*.*`
- PRs targeting `main` or `v*` (build only; does not push)

Tagging rules:

- On `main` pushes: publishes `:main`.
- On `v*` branch pushes (e.g. `v1`): publishes only preview tags:
  - `:<branch>-dev` (e.g. `:v1-dev`)
  - `:<branch>-<sha>` (e.g. `:v1-a1b2c3d`)
- On semver tag pushes (e.g. `v1.0.15`): publishes release tags:
  - `:1.0.15`
  - `:v1`
  - `:latest` only if that release is the highest published semver

### GitHub “Latest” release policy

Workflow: `.github/workflows/major-version.yml`

- Trigger: when a GitHub Release is **published**.
- If the published release is **not** the highest published semver, the workflow patches it so it is **not** marked as GitHub “Latest”.

This prevents out-of-band hotfix releases (e.g. publishing `v1.0.15` after `v3.0.4`) from taking over the GitHub “Latest” designation.

## Procedures

### Current major release (via `main`, currently v3)

1. Merge changes into `main`.
2. Confirm the draft release notes (Release Drafter updates the draft automatically).
3. Finalize the release by creating/publishing a GitHub Release with a new tag `v3.x.y`.
4. The `Docker` workflow builds and pushes:
   - `hanzozt/zt-builder:3.x.y`
   - `hanzozt/zt-builder:v3`
   - `hanzozt/zt-builder:latest` (only if it’s the highest published semver)

### Maintenance/hotfix release (via `v1`)

1. Merge changes into the `v1` branch.
   - This publishes preview images:
     - `hanzozt/zt-builder:v1-dev`
     - `hanzozt/zt-builder:v1-<sha>`
   - The stable `hanzozt/zt-builder:v1` is **not** updated yet.
2. Confirm the Release Drafter draft version is correct for the next patch (e.g. `v1.0.15`).
3. Finalize by creating/publishing a GitHub Release with tag `v1.0.15`.
4. The tag push triggers Docker publishing of:
   - `hanzozt/zt-builder:1.0.15`
   - `hanzozt/zt-builder:v1`

The release will **not** be marked as GitHub “Latest” unless it is the highest published semver.

## Troubleshooting

- If a maintenance branch draft shows the wrong major (e.g. drafts `v3.*` on `v1`):
  - Ensure you are pushing to a `v<MAJOR>` branch.
  - Ensure there is at least one prior `v<MAJOR>.*.*` GitHub Release tag to base the next patch on.
  - Confirm Release Drafter has permission to read releases and write the draft.
