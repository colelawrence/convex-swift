# Upstream Sync Notes

This fork tracks upstream `convex-swift` while carrying a very small local patch set for Wave.

## Fork purpose

This fork exists to preserve a simple invariant for Wave App:

> If Wave supports macOS 14, the bundled `ConvexMobile` macOS artifact must honestly declare macOS 14 compatibility.

Upstream `convex-swift` `0.8.1` shipped a macOS XCFramework slice whose static archive contained many object files with `LC_BUILD_VERSION minos 26.2`, which produced linker warnings when Wave linked for macOS 14. This fork fixes the artifact instead of adding compatibility code in Wave.

Rules:
- Keep the downstream delta as small as possible.
- Fix packaging truth here, not in downstream app code.
- Do not add Wave-specific runtime behavior to this repo.
- Retire this fork as soon as upstream ships an artifact that satisfies the same invariant.

## Remotes and branches

Expected remotes:

- `origin` → `https://github.com/colelawrence/convex-swift.git`
- `upstream` → `https://github.com/get-convex/convex-swift.git`
- `upstream/main` = canonical upstream baseline
- `main` = downstream branch carrying the minimal Wave-required patch
- `sync/upstream-YYYY-MM-DD` = temporary sync branch for each upstream integration

If `upstream` is missing, add it once per clone:

```bash
git remote add upstream https://github.com/get-convex/convex-swift.git
```

## Default sync workflow

```bash
cd vendor/convex-swift

git fetch upstream origin --tags
git switch main
git switch -c sync/upstream-YYYY-MM-DD

git merge upstream/main
# resolve conflicts
# reapply only the minimal carry still required
# run targeted verification
# update this file with any new resolution notes

git switch main
git merge sync/upstream-YYYY-MM-DD
git push origin main
```

Why merge instead of rebase:
- preserves explicit upstream sync points
- avoids rewriting shared downstream history
- makes the carry patch easier to audit over time

Enable rerere once per clone:

```bash
git config rerere.enabled true
```

## Primary checkout hygiene

Treat the checked-out `vendor/convex-swift` worktree as the canonical local reference repo.
If you need a temporary worktree for a sync, that is a means to complete the sync — not the final place to leave the repo.

Rules:
- Check `git status --short` in the primary checkout before starting.
- If the primary checkout is dirty and the dirt is not obviously disposable generated output, do the merge in a temporary worktree instead of forcing cleanup.
- Do **not** use blanket destructive cleanup (`git reset --hard`, `git checkout --`, `git clean -fd`, `git stash`) just to make a sync easier.
- Before calling the sync done, return the primary `vendor/convex-swift` checkout to the completed sync commit.
- If this repo is checked out as a submodule inside Wave, update and commit the superproject submodule pointer only after the primary checkout is in the intended final state.
- If you cannot explain each remaining dirty path, the sync is not done.

## Rules for sync commits

- Prefer upstream package structure, product definitions, and Swift source.
- Reapply only the minimal local behavior still required.
- Keep local carry patches isolated by concern.
- Do not mix Wave feature work into upstream sync commits.
- If a binary refresh is required, keep it separate from unrelated handwritten changes when possible.
- Sync frequently; small regular merges are cheaper than large jumps.
- Retire carry patches aggressively when upstream makes them unnecessary.

## Current carry patches to watch

### `libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a`

Intent:
- The macOS archive members in the XCFramework must not advertise a deployment target newer than Wave supports.
- Wave currently requires all macOS archive members to report `minos <= 14.0`.
- The fork should carry only an artifact fix, not a source-level behavior fork.

Current downstream delta:
- Fork commit `b295cb0` rebuilds the macOS static archive so the previously bad objects report `minos 14.0` instead of `26.2`.
- iOS slices are intentionally unchanged.

Resolution pattern:
- Prefer upstream Swift source and package manifest shape.
- If upstream still ships a bad macOS slice, rebuild only the macOS static archive from the matching upstream source and replace:
  - `libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a`
- Do **not** refresh iOS artifacts unless there is an intentional multi-platform update.
- Keep the rebuild pinned to the same upstream source revision as the rest of the package.
- If upstream ships a corrected artifact, delete this carry patch instead of preserving compatibility logic.

## Rebuild workflow for the macOS archive

This package’s bundled XCFramework is produced from `convex-mobile`.
Use the matching upstream source and rebuild only the macOS static archive unless a broader artifact refresh is intentional.

Reference flow:

```bash
git clone https://github.com/get-convex/convex-mobile.git
cd convex-mobile
git submodule update --init --recursive

# Ensure ios/ points at the convex-swift revision you intend to patch.
cd ios
git checkout <matching-convex-swift-revision>
cd ../rust

export MACOSX_DEPLOYMENT_TARGET=14.0
cargo build --lib --release --target aarch64-apple-darwin

cp target/aarch64-apple-darwin/release/libconvexmobile.a \
  ../ios/libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a
```

Notes:
- A full `./build-ios.sh` refresh may be appropriate later, but this fork currently exists because Wave needed the macOS slice fixed without widening the change set.
- If you intentionally do a full XCFramework rebuild, re-verify every shipped slice before merging.

## Verification

### Verify the macOS archive metadata directly

Run from this repo:

```bash
tmpdir="$(mktemp -d)"
cp libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a "$tmpdir/libconvexmobile.a"
cd "$tmpdir"
ar -x libconvexmobile.a
python3 - <<'PY'
import os, re, subprocess, sys
max_minos = (14, 0)
violations = []
count = 0
for name in sorted(os.listdir('.')):
    if not name.endswith('.o'):
        continue
    count += 1
    out = subprocess.run(['xcrun', 'vtool', '-show-build', name], capture_output=True, text=True).stdout
    match = re.search(r'minos\s+([^\n]+)', out)
    if not match:
        continue
    minos_text = match.group(1).strip()
    minos = tuple(int(part) for part in minos_text.split('.'))
    if minos > max_minos:
        violations.append((name, minos_text))
if violations:
    print(f"ERROR: found {len(violations)} object files with minos > 14.0", file=sys.stderr)
    for name, minos_text in violations[:50]:
        print(f"  {name}: {minos_text}", file=sys.stderr)
    raise SystemExit(1)
print(f"OK: inspected {count} object files; all macOS min versions are <= 14.0")
PY
```

### Verify the package still builds/tests

```bash
swift test
```

### Verify Wave integration before moving the submodule pointer

From the Wave repo root:

```bash
macOS-app/scripts/verify-convexmobile-macos-metadata.sh \
  vendor/convex-swift/libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a 14.0

cd macOS-app
xcodebuild build -scheme Phosphor -destination 'platform=macOS'
```

Expectation:
- no Convex linker warning about macOS deployment target mismatch
- Wave still builds cleanly against the submodule

## Suggested sync verification checklist

Run the smallest relevant checks for the actual carry touched:

```bash
cd vendor/convex-swift
swift test

# If the macOS archive changed, run the metadata verification above.
# Then validate downstream Wave integration from the superproject.
```

## Long-term maintenance guidance

This fork is maintainable only if the downstream delta stays tiny and explicit.

Operational rules:
- Prefer regular sync cadence while the fork is active.
- The main cost driver is binary carry, not Swift source divergence.
- Do not let this fork accumulate Wave-specific behavior.
- Prefer upstreaming the packaging fix over long-lived downstream ownership.
- A healthy downstream branch should have shrinking carry, not accumulating carry.

Review questions before finishing a sync:
- Is the macOS archive carry patch still required?
- Did upstream ship a corrected XCFramework so we can delete the patch?
- Did we accidentally widen the change set beyond the macOS packaging fix?
- Did we verify the actual archive members instead of trusting `Package.swift` metadata?

## Patch retirement rule

Whenever upstream absorbs this carry patch:
- delete the local artifact delta instead of preserving compatibility logic
- update this file to remove the carry patch from the inventory
- note the upstream commit / release / PR that made the fork unnecessary
- update Wave to point back at upstream and remove the submodule if no local carry remains

## Conflict log template

Append a short entry for each sync:

```md
## Sync YYYY-MM-DD
- merged: upstream/main @ <sha>
- branch: sync/upstream-YYYY-MM-DD
- conflicts:
  - <path> — <how it was resolved>
- verification:
  - [x] swift test
  - [x] macOS archive metadata <= 14.0
  - [x] Wave macOS build
- notes:
  - was the macOS archive rebuilt? yes/no
  - if yes, from which convex-mobile revision?
  - were any non-macOS slices changed intentionally? yes/no
- reflection:
  - what was surprisingly easy?
  - what repeatedly caused friction?
  - what carry patch should be reduced, upstreamed, or retired before the next sync?
  - what should be added or corrected in UPSTREAM_SYNC.md based on this round?
```

## Post-sync reflection rule

At the end of every sync:
- update the sync entry in this file with the actual conflict set and verification used
- record any patch retirement, new carry patch, or carry shrinkage
- capture one or two lessons that would make the next sync easier
- update the living guidance in this file during the same sync instead of relying on memory

## Sync 2026-04-14
- merged: upstream `0.8.1` / `125b71b2f8725931a5b0e8252f0799ef2adad120`
- branch: `main`
- carry applied:
  - `libconvexmobile-rs.xcframework/macos-arm64/libconvexmobile.a` — replaced with a rebuilt macOS archive whose object files report `minos <= 14.0`
- verification:
  - [x] direct archive metadata inspection (`minos <= 14.0`)
  - [x] downstream Wave verifier script
  - [x] downstream `xcodebuild build -scheme Phosphor -destination 'platform=macOS'`
- notes:
  - the fork exists only because upstream packaged the macOS XCFramework slice with bad deployment metadata for Wave’s macOS 14 support claim
  - the intended retirement path is to delete this fork as soon as upstream ships a corrected XCFramework
