# Hermes Desktop — Ventura Intel Compatibility

## 1. Status

This fork maintains a compatibility adaptation for macOS Ventura 13.x on Intel
`x86_64`. The official upstream project currently requires macOS 14 or newer.

This is not official support from the upstream project. The compatibility
baseline has been compiled in GitHub Actions and validated on real hardware,
but future upstream versions may introduce APIs or build requirements that need
additional, localized adaptations.

## 2. Validated Environment

- Hardware: MacBook Pro 13-inch, Late 2011
- Architecture: Intel `x86_64`
- Operating system: macOS Ventura 13.6
- Validated commit: `3802e10a669ce655dde94dbce5b8e964add8cf12`
- Validated tag: `ventura-intel-working-2026-10-01`
- Fork: <https://github.com/wcsg108-ops/hermes-desktop>
- Upstream: <https://github.com/dodo-reach/hermes-desktop>

## 3. Branch Strategy

### `main`

- Tracks the original/upstream base.
- Must not receive Ventura-specific changes directly.

### `ventura-intel-experiment`

- Incorporates new upstream versions.
- Is used to resolve compatibility problems and run GitHub Actions.
- Is the laboratory for new Ventura/Intel adaptations.
- Must never be considered stable until its artifact has been tested on the
  real Ventura Mac.

### `ventura-intel`

- Holds the stable, validated Ventura/Intel baseline.
- Advances only after the corresponding experimental build compiles, passes CI,
  and runs successfully on the real Ventura Mac.

## 4. Current Compatibility Changes

- `Package.swift`: deployment platform changed from macOS 14 to macOS 13.
- `packaging/Info.plist`: `LSMinimumSystemVersion` changed from `14.0` to `13.0`.
- `onChange`: a compatibility bridge preserves the modern overload on macOS
  14+ and equivalent new-value behavior on macOS 13.
- `ContentUnavailableView`: uses the native component on macOS 14+ and a simple
  fallback on macOS 13.
- `Animation.snappy`: uses `snappy` on macOS 14+ and a `spring` fallback on
  macOS 13.
- `SectorMark`: retains the original chart on macOS 14+ and uses a simple
  fallback on macOS 13.
- `Vendor/SwiftTerm`: not modified.
- Experimental target architecture: `x86_64`.

## 5. CI / Build

The experimental workflow is:

```text
.github/workflows/ventura-intel-experiment.yml
```

It runs on `macos-15-intel` and must:

- set `HERMES_MAC_ARCHS=x86_64`;
- set `MACOSX_DEPLOYMENT_TARGET=13.0`;
- run tests and build the application bundle;
- confirm that the main executable is exactly `x86_64`;
- confirm `LSMinimumSystemVersion = 13.0`;
- inspect `LC_BUILD_VERSION` and confirm `minos = 13.0`;
- create a ZIP containing `HermesDesktop.app`;
- publish the ZIP as a GitHub Actions artifact.

## 6. Update Procedure

Use the experimental branch for every upstream update. Keep the working tree
clean and inspect each operation before continuing.

1. Update the remote references:

   ```bash
   git fetch upstream --prune
   git fetch origin --prune --tags
   ```

2. Start from the experimental branch and confirm its state:

   ```bash
   git switch ventura-intel-experiment
   git status --short --branch
   git log -1 --oneline
   ```

3. Incorporate `upstream/main` without modifying `ventura-intel`:

   ```bash
   git merge upstream/main
   ```

4. Resolve conflicts only on `ventura-intel-experiment`, preserving upstream
   behavior wherever possible.

5. Search for new macOS 14+ APIs and review all availability boundaries:

   ```bash
   rg -n 'ContentUnavailableView|SectorMark|\.snappy|\.onChange\(of:|#available|@available' Package.swift Sources packaging scripts
   rg -n 'macOS\(\.v14\)|LSMinimumSystemVersion|MACOSX_DEPLOYMENT_TARGET' . --glob '!Vendor/**' --glob '!.git/**'
   ```

6. Keep any new fallback small and localized. Do not rewrite working upstream
   components merely to support Ventura.

7. Run available static validation before publishing:

   ```bash
   git diff --check
   swiftc -parse $(rg --files Sources/HermesDesktop -g '*.swift')
   ```

   A local parse does not replace the GitHub Actions build with a modern Xcode
   and SDK.

8. Commit reviewed compatibility changes and push only the experimental branch:

   ```bash
   git push origin ventura-intel-experiment
   ```

9. Find and wait for the experimental workflow:

   ```bash
   gh run list --repo wcsg108-ops/hermes-desktop --workflow ventura-intel-experiment.yml --branch ventura-intel-experiment
   gh run watch RUN_ID --repo wcsg108-ops/hermes-desktop --exit-status
   ```

10. Download the successful artifact:

    ```bash
    gh run download RUN_ID --repo wcsg108-ops/hermes-desktop --name ARTIFACT_NAME
    ```

11. Copy the artifact to the real Ventura Mac and test the application there.

12. Only after successful real-hardware validation, promote the approved commit
    to `ventura-intel`.

13. Create a new annotated tag for the validated version and record the date,
    commit, hardware, and operating system used for the test.

## 7. Promotion Procedure

Promotion must preserve history and use a normal fast-forward push. Replace
`VALIDATED_SHA` only with the exact commit whose CI artifact was tested on the
real Ventura Mac.

```bash
git fetch origin --prune --tags
git show --stat VALIDATED_SHA
git merge-base --is-ancestor origin/ventura-intel VALIDATED_SHA
git switch ventura-intel
git merge --ff-only VALIDATED_SHA
git push origin ventura-intel
```

If the ancestor check or `--ff-only` merge fails, stop and inspect the branch
history. Do not force push the stable branch.

After the normal push, create and publish an annotated validation tag:

```bash
git tag -a ventura-intel-working-YYYY-MM-DD VALIDATED_SHA \
  -m "Validated Ventura Intel build on HARDWARE / MACOS_VERSION"
git push origin refs/tags/ventura-intel-working-YYYY-MM-DD
```

Record the validation date and environment in this document when the stable
baseline changes.

## 8. Rollback

The known working baseline can be inspected or tested without moving any
permanent branch.

Create a temporary recovery branch from the validated tag:

```bash
git fetch origin --tags
git switch -c ventura-intel-recovery-2026-10-01 ventura-intel-working-2026-10-01
```

Alternatively, restore that version locally in detached-HEAD mode for testing:

```bash
git fetch origin --tags
git switch --detach ventura-intel-working-2026-10-01
# Run inspection or local tests.
git switch ventura-intel
```

Do not use force push as the normal rollback mechanism. Prefer a temporary
branch for diagnosis and a reviewed forward fix on `ventura-intel-experiment`.

## 9. Validation Checklist

- [ ] GitHub Actions success
- [ ] `x86_64` confirmed
- [ ] `LSMinimumSystemVersion = 13.0`
- [ ] `LC_BUILD_VERSION minos = 13.0`
- [ ] App opens on Ventura
- [ ] Hermes connection works
- [ ] Sessions works
- [ ] Terminal works
- [ ] Files works
- [ ] Skills works
- [ ] Workflows works
- [ ] Usage works
- [ ] Cron/Kanban works
- [ ] No evident regression on macOS 14+

## 10. Maintenance Principles

- Make the smallest necessary changes.
- Prefer localized wrappers and fallbacks.
- Do not modify `Vendor` without a demonstrated need.
- Preserve the original behavior on macOS 14+.
- Use the compiler and CI to discover compatibility problems.
- Always test on real hardware before promoting a build.
- Do not put Ventura-specific patches on `main`.
- Do not use force push in the normal workflow.
