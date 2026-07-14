# Contributing

This repository ports the Kotlin compiler to the Android platform. It does not
hold a full copy of Kotlin. Instead it holds a set of patches (`patches/*.patch`)
that are applied on top of a pinned upstream Kotlin release, plus the scripts
that download, patch, and build that source.

This guide explains how the patches are maintained.

## Branch roles

| Branch        | Contents                                                          | Role |
|---------------|-------------------------------------------------------------------|------|
| `kotlin-base` | Pristine upstream Kotlin at a pinned tag (e.g. `v2.3.20`).        | The base the patches apply on top of. Local and disposable; obtained from the `kotlin` (JetBrains) remote, not stored on `origin`. |
| `patch-stack` | `kotlin-base` + the fork commits = the full patched Kotlin tree.  | Where you edit fixes. Local, disposable, and reconstructed on demand (see below). Rewritten by every rebase, so it is not a durable history and is never shared. |
| `main`        | The scripts, `patches/`, and repo metadata. No Kotlin source.     | The single source of truth. The only branch pushed to `origin` and consumed by Code On the Go. One commit == one regeneration of the patch files. |

Each fork commit on `patch-stack` becomes one numbered file in `patches/`.

### Source of truth and reconstruction

`patches/` on `main` is canonical. `patch-stack` is a *derived* view of it, not a
branch you keep around or share. Because the patches are plain `git format-patch`
output, `git am` rebuilds `patch-stack` deterministically from them:

```
git fetch kotlin --tags
git branch -f kotlin-base v2.3.20        # match $KT_VERSION in common.sh
git switch -C patch-stack kotlin-base
git am patches/*.patch
```

You edit on this rebuilt branch, regenerate `patches/`, then throw the branch
away. All collaboration happens through `main`, never through `patch-stack`.

### Critical invariant

`kotlin-base` MUST point at the same upstream tag whose source tarball the
consumer downloads. That tag is derived from `KT_VERSION` in `common.sh`
(the tarball is `v$KT_VERSION`). If `kotlin-base` and `KT_VERSION` drift apart,
the patches will fail to apply downstream (`patch -p1` in `patch.sh`).

Do not push `kotlin-base` to `origin`. It is ~130k commits of full upstream
history; `origin` is a thin repo that holds only `main`. Fetch the base from the
`kotlin` remote instead.

## The regeneration step

Every maintenance flow below ends with the same step: rebuild `patches/` on
`main` from the current `patch-stack`.

```
git switch main
rm -rf patches
git format-patch kotlin-base..patch-stack -o patches/
git add patches
git commit
```

Then publish it - see [Submitting changes](#submitting-changes).

`format-patch` numbers files by commit order (`0001-...`, `0002-...`), so
inserting or reordering commits on `patch-stack` renumbers every later file.
Amending one patch also rewrites the embedded commit SHA of every later patch
file. Both are expected noise. Write a commit message on `main` describing the
change; that commit is the durable record of this regeneration.

## Update an existing patch

Reconstruct `patch-stack` (see above), then:

```
git rebase -i kotlin-base
```

Mark the target commit as `edit`, make your changes, then:

```
git add -A
git commit --amend
git rebase --continue
```

Then run the regeneration step.

## Add a new patch

Reconstruct `patch-stack`, then append the new commit at the tip:

```
# make changes
git add -A
git commit -m "fix: ..."
```

Prefer appending. Inserting a patch mid-stack with `git rebase -i kotlin-base`
renumbers every later file and creates a large, conflict-prone diff, so only do
it when ordering genuinely requires it. Then run the regeneration step; the new
commit shows up as an additional numbered file in `patches/`.

## Bump the upstream Kotlin version

Done rarely and by hand. Move the base, replay the stack, update the scripts.

1. Fetch the new release and repoint `kotlin-base` to its tag:

   ```
   git fetch kotlin --tags
   git branch -f kotlin-base <new-tag>
   ```

2. Rebase the fork commits from the old base onto the new one:

   ```
   git rebase --onto kotlin-base <old-base-commit> patch-stack
   ```

   Resolve conflicts as upstream code has moved. Each patch that no longer
   applies cleanly must be fixed here, commit by commit, before continuing.

3. Update `KT_VERSION` and `KT_SRC_CHECKSUM` in `common.sh` to match the new
   tarball. Keep them consistent with `kotlin-base` (see the invariant above).

4. Run the regeneration step.

## Submitting changes

Your change ultimately lands as edits to `patches/` on `main`, delivered as a
pull request.

1. Fork the repository and clone your fork.
2. Reconstruct `patch-stack` (see [Source of truth and
   reconstruction](#source-of-truth-and-reconstruction)) and make your change by
   [updating an existing patch](#update-an-existing-patch) or [adding a new
   one](#add-a-new-patch).
3. Run the [regeneration step](#the-regeneration-step) to rebuild `patches/` and
   commit the result on a branch off `main`.
4. Push the branch to your fork and open a pull request against `main`.

A few things that make review smoother:

- **Keep the change as a patch edit, not a raw `patches/` hand-edit.** Editing a
  `.patch` file directly is error-prone; go through `patch-stack` so the diff is
  reproducible.
- **Append new patches at the tip.** Inserting mid-stack renumbers every later
  file and bloats the diff. Only reorder when correctness needs it.
- **Rebase on the latest `main` before pushing.** If your PR and another both
  touch the same patch, they conflict inside the `.patch` file, which is awkward
  to merge - rebasing early keeps it small.

## How the build is published

Code On the Go does not build the compiler itself - it consumes a prebuilt JAR
published from this repository. The build runs here, in CI:

- On every push, the `Build Kotlin Patched Artifact` GitHub Actions workflow
  runs `get_source.sh` -> `patch.sh` -> `build.sh` and uploads the resulting JAR
  as a build artifact.
- `ci_release.sh` cuts a GitHub Release (tagged by Kotlin version) with that JAR
  attached, so consumers can download a versioned, checksummed build.

The scripts, for reference:

- `get_source.sh` - downloads and extracts the upstream `v$KT_VERSION` tarball,
  verifying its checksum.
- `patch.sh` - applies every `patches/*.patch` in order with `patch -p1`.
- `build.sh` - runs the Gradle build and copies out the resulting JAR.
