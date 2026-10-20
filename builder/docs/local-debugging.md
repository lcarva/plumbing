# Debugging Python package builds locally

When an onboarding PR in the [index repository](https://github.com/calungaproject/index)
fails to build a Python package, the cause is usually something missing from the builder
image's build root — a header file, a library, a `pkg-config` file, or a build tool. This
document describes the local debugging loop:

1. **Reproduce** the failure with `hack/build-locally.sh` from the index repository.
2. **Iterate** on the builder image as an *overlay*: a small Dockerfile layered on top of
   the published builder image, built locally with podman. No registry, no full rebuild.
3. **Land** the fix in this repository and roll it out to CI.

The loop is deliberately local-first. Building the full builder image from
`builder/Containerfile` is expensive: it compiles CPython, two OpenSSLs, curl, git, Rust,
OpenBLAS, Arrow, ffmpeg and more, and Konflux gives that build 8 CPUs, 12 GiB of RAM and a
3-hour timeout. Layering a few `RUN` instructions on top of the published image takes
minutes on a laptop.

For tracing a package through Konflux instead (PipelineRuns, snapshots, releases, Pulp),
see `hack/debug-package.sh` and `.claude/agents/debug-package.md` in the index repository.
Those answer "where did it fail in CI?"; this document answers "how do I fix the build?".

## How the pieces fit

```
plumbing/builder/Containerfile                     (this repository)
        │  Konflux push pipeline
        │  (.tekton/plumbing-builder-push.yaml, on merge to main)
        ▼
quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder:<commit>
        │  pinned by digest in
        ▼
plumbing/tasks/build-python-wheels-oci-ta.yaml     (this repository)
        │  Konflux bundle-build pipeline
        │  (.tekton/task-build-python-wheels-push.yaml)
        ▼
quay.io/redhat-user-workloads/calunga-tenant/task-build-python-wheels:<commit>
        │  referenced by digest in
        ▼
index/.tekton/build-pipeline.yaml                  (index repository)
        ▼
index/hack/build-locally.sh                        (default image, or --builder-image)
```

Two consequences:

- `hack/build-locally.sh` with no flags builds with exactly the image CI uses.
- A builder image change only reaches CI after three rollouts: image → task → index
  pipeline. [Landing the fix](#step-6--land-the-fix) walks through all three.

## Prerequisites

```bash
# Adjust to wherever your clones live:
INDEX_DIR=~/src/github.com/calungaproject/index
PLUMBING_DIR=~/src/github.com/calungaproject/plumbing
```

- `podman` — runs and builds every image in this workflow.
- `yq` and `tkn` — needed only to resolve the *default* builder image
  (`hack/build-locally.sh` uses them). Once you pass `--builder-image`, podman is enough.
- `skopeo` — needed only for digest resolution (`hack/latest-image ... -d`).
- An x86_64 host. Wheels target `manylinux_2_28` on x86_64 and the builder image is
  x86_64; emulation works but is painfully slow.

## The loop at a glance

```bash
cd "${PLUMBING_DIR}/builder"
podman build -t localhost/calunga-builder:debug -f Dockerfile.debug .   # cheap: your layers only

cd "${INDEX_DIR}"
hack/build-locally.sh --builder-image localhost/calunga-builder:debug "pkg==1.2.3"

# Repeat until the build succeeds: read error → edit Dockerfile.debug → rebuild → retry.
```

## Step 1 — Reproduce the failure

```bash
cd "${INDEX_DIR}"
hack/build-locally.sh "somepackage==1.2.3"

# If overrides/constraints/ has a file for this package, mirror what CI applies:
hack/build-locally.sh -c overrides/constraints/somepackage-1.2.3.txt "somepackage==1.2.3"
```

Run this from an interactive terminal: `build-locally.sh` invokes `podman run -it`.
(When driving this from a script or agent, replace `-it` with `-i` — or nothing — in
your own invocations.)

The output is fromager's: dependency resolution and the build order first, then
per-package builds. The failing package's error appears at the end. Classify it:

| Symptom in the log | Usual cause | Fix lives in |
|---|---|---|
| `fatal error: xyz.h: No such file or directory` | missing header | builder image (this document) |
| `Package xyz was not found in the pkg-config search path` | missing `.pc` / `-devel` RPM | builder image (this document) |
| `command not found`, `CMake Error: ... not found` | missing build tool | builder image (this document) |
| `cannot find -lxyz`, `undefined reference to ...` | missing library | builder image (this document) |
| fromager resolves wrong/unsatisfiable versions | greedy resolver, non-newest pin | index repo: `hack/generate-constraints.py`, `overrides/constraints/README.md` |
| package needs different build dir / env / patch | package-specific build settings | fromager settings: index repo `overrides/settings/` (per-package) or `builder/overrides/` (builtin) |

If the fix belongs in the builder image, continue. Before moving on, record the image the
failing run used — the overlay must be based on it. This is the same resolution
`build-locally.sh` performs by default:

```bash
cd "${INDEX_DIR}"
BUILD_TASK_BUNDLE="$(
    yq '.spec.tasks[] | select(.name == "build-wheels").taskRef.params[] | select(.name == "bundle") | .value' \
        .tekton/build-pipeline.yaml
)"
BUILDER_IMAGE="$(
    tkn bundle list -o json "${BUILD_TASK_BUNDLE}" | \
        yq -r '.spec.steps[] | select(.name == "build-wheels") | .image'
)"
echo "${BUILDER_IMAGE}"
# quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder@sha256:...
```

## Step 2 — Write the overlay Dockerfile

Do **not** rebuild `builder/Containerfile` locally to iterate. Instead, add layers on top
of a published builder image.

Create `builder/Dockerfile.debug` (throwaway; gitignored):

```dockerfile
# Throwaway overlay for local debugging — do not commit.
# Base = the image that reproduced the failure (from Step 1).
FROM quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder@sha256:8f85ac...

# Whatever the build root is missing, for example:
RUN dnf -y install --setopt install_weak_deps=0 \
        libpq-devel \
    && dnf clean all \
    && rm -rf /var/cache/dnf
```

Rules of thumb:

- Pin `FROM` **by digest** — the digest recorded in Step 1 — so every iteration of the
  loop is reproducible.
- Keep changes in one or a few `RUN` instructions. Podman caches layers: editing the
  last `RUN` rebuilds only that layer, in seconds to minutes.
- If the fix needs a library built from source, structure the overlay in two stages:
  build the library in an earlier stage, then in the final stage copy or symlink its
  headers, libraries and `.pc` files into `/usr/local` and run `ldconfig`.

Choosing the base image:

- **Reproducing a specific CI failure:** use the digest from Step 1.
- **Preparing the fix that will land on `main`:** use the latest published image:

  ```bash
  cd "${PLUMBING_DIR}"
  git fetch origin
  hack/latest-image plumbing-builder -d
  # quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder@sha256:...
  ```

  If that digest differs from Step 1's, the task bundle is lagging; base the overlay on
  the Step 1 digest while iterating, then rebase onto the latest before landing.

## Step 3 — Build the overlay locally with podman

```bash
cd "${PLUMBING_DIR}/builder"
podman build -t localhost/calunga-builder:debug -f Dockerfile.debug .
```

- The base image is pulled from quay.io on the first build (it is large, multiple GiB)
  and cached afterwards.
- Only your added layers build.
- The result stays in local podman storage. Nothing is pushed anywhere; no registry is
  involved at any point.

## Step 4 — Build the package against the overlay

```bash
cd "${INDEX_DIR}"
hack/build-locally.sh --builder-image localhost/calunga-builder:debug "somepackage==1.2.3"
```

`build-locally.sh` checks `podman image exists` before pulling, so the local image is
used as-is. On success, the wheels are collected into `${INDEX_DIR}/output/`.

## Step 5 — Iterate until the build succeeds

Each iteration is: read the error → edit `builder/Dockerfile.debug` → `podman build`
(cached, fast) → `hack/build-locally.sh --builder-image ...`.

### Exploring the image interactively

The image entrypoint (`manylinux-entrypoint`) execs whatever command you pass, so a
shell is one command away:

```bash
podman run -it --rm localhost/calunga-builder:debug bash
```

Inside the container, find out which package provides what the build is missing:

```bash
dnf provides '*/xyz.h'         # which -devel RPM ships a header
dnf provides 'pkgconfig(xyz)'  # which RPM ships a pkg-config file
dnf provides '*/libxyz.so*'    # which RPM ships a library
pkg-config --list-all | grep -i xyz
```

Install candidates with `dnf -y install ...` to try them out before writing them into
the Dockerfile (changes made this way are lost when the container exits — they are for
exploration only).

### Keeping the build logs

`hack/build-locally.sh` deletes its temp workdir on exit, and `build-wheels` wipes its
`output/` directory at the *start* of every run. To keep a failed run's logs, replicate
the script's podman invocation with a persistent workdir:

```bash
mkdir -p ~/calunga-debug/workdir
cd "${INDEX_DIR}"
podman run -it --rm \
    -v ~/calunga-debug/workdir:/var/workdir:Z \
    -v "${INDEX_DIR}":/var/workdir/source:ro,Z \
    -w /var/workdir \
    -e DEBUG=1 \
    localhost/calunga-builder:debug \
    build-wheels "somepackage==1.2.3" \
        --cache-wheel-server-url https://packages.redhat.com/api/pypi/public-trusted-libraries/main/simple/ \
        --package-settings-dir /var/workdir/source/overrides/settings
        # If CI applied constraints, also add:
        #   -c /var/workdir/source/overrides/constraints/somepackage-1.2.3.txt
```

Drop `-it` when running non-interactively. After a failure, inspect (the next run wipes
them):

- `~/calunga-debug/workdir/output/work-dir/logs/somepackage-1.2.3.log` — fromager's
  per-package log (hyphens in the package name become underscores in the filename).
- `~/calunga-debug/workdir/output/work-dir/somepackage-1.2.3/build.log` — raw output of
  the PEP 517 build backend; this is where compiler and linker errors land.
- `~/calunga-debug/workdir/output/work-dir/build-order.json` — the resolved build order.

`-e DEBUG=1` turns on shell tracing (`set -x`) in `build-wheels`; useful but noisy.

Success is only success when `hack/build-locally.sh` (the canonical checker) completes —
switch back to it for the final green run of the loop.

## Step 6 — Land the fix

The overlay is a debugging artifact. The durable fix must be expressed in
`builder/Containerfile` so the image builds from source in CI. Rollout happens in three
steps, one per repository boundary in the chain diagram above.

### 6.1 Port the overlay into the real Containerfile

First, if you based the overlay on the Step 1 digest, rebase it onto the latest published
image (`hack/latest-image plumbing-builder -d`) and re-verify the package still builds —
your change will land on top of that image, not the one CI failed on.

Then port the delta into `builder/Containerfile`:

| What the package needs | Where it goes |
|---|---|
| An RPM (header, tool, library) at wheel-build time | The `dnf -y install` in the **final stage**, next to `perl-IPC-Cmd`, `cmake`, `zip`, `unzip` |
| A library built from source | A new `build_scripts/build-<lib>.sh` plus a build stage, with `COPY --from` pulling its `/manylinux-buildfs` (headers, pkg-config) and `/manylinux-rootfs` (libraries) — model it on an existing stage such as `build_zstd`; see also `OPENSSL_SOURCE_BUILD.md` |
| More Python tooling in the build venv | `builder/requirements.in`, then regenerate the hash-pinned `builder/requirements.txt` |
| Different fromager behavior | `builder/overrides/` (settings baked into the image); per-package settings belong in the index repo's `overrides/settings/` instead |

RPMs that end up linked into wheels are bundled into the wheel by `auditwheel repair`,
so an image change can change wheel contents. The wheel-check integration tests (UBI9,
Ubuntu, Fedora) and Enterprise Contract exist to catch that — watch them after rollout.

If the change is easily testable, consider extending `builder/tests/` (smoke tests run
inside the image).

### 6.2 Merge the plumbing PR and wait for the image

Open a PR with the Containerfile change. The PR pipeline builds the full image in
Konflux (that is the expensive build we avoided locally). After merge to `main`, the
push pipeline publishes:

```
quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder:<merge-commit>
```

Wait for that pipeline to finish (it can take up to ~3 hours), then resolve the new
digest:

```bash
cd "${PLUMBING_DIR}"
git fetch origin
hack/latest-image plumbing-builder -d
# quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder@sha256:<new>
```

Tip: while a plumbing PR is open, its (unmerged) image is published as
`quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder:on-pr-<head-sha>`
(expires after 5 days). You can review someone's PR without building anything:

```bash
cd "${INDEX_DIR}"
hack/build-locally.sh \
    --builder-image quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder:on-pr-<head-sha> \
    "somepackage==1.2.3"
```

### 6.3 Bump the task

`tasks/build-python-wheels-oci-ta.yaml` pins the builder image by digest in **two**
places — the `build-wheels` and `collect-build-files` steps:

```bash
cd "${PLUMBING_DIR}"
grep -n 'image: quay.io/redhat-user-workloads/calunga-tenant/plumbing-builder@' \
    tasks/build-python-wheels-oci-ta.yaml
```

Update both to the new digest from 6.2 and open a PR. On merge, the
`task-build-python-wheels` bundle is rebuilt and published. Resolve its digest:

```bash
cd "${PLUMBING_DIR}"
git fetch origin
hack/latest-image task-build-python-wheels -d
# quay.io/redhat-user-workloads/calunga-tenant/task-build-python-wheels@sha256:<new>
```

Optionally verify the new bundle really contains the new builder image:

```bash
tkn bundle list -o json \
    quay.io/redhat-user-workloads/calunga-tenant/task-build-python-wheels@sha256:<new> | \
    yq -r '.spec.steps[] | select(.name == "build-wheels") | .image'
```

### 6.4 Bump the index pipeline and retest the onboarding PR

In the index repository, update the `build-wheels` taskRef bundle digest in
`.tekton/build-pipeline.yaml`:

```bash
cd "${INDEX_DIR}"
# Current reference:
yq '.spec.tasks[] | select(.name == "build-wheels").taskRef.params[] | select(.name == "bundle") | .value' \
    .tekton/build-pipeline.yaml
```

Change the `@sha256:...` suffix to the new task bundle digest from 6.3 and open a PR.

You can verify the whole chain **before** merging: `hack/build-locally.sh` reads the
pipeline file from your local checkout, so with the bump applied locally (or on the PR
branch):

```bash
cd "${INDEX_DIR}"
hack/build-locally.sh "somepackage==1.2.3"   # no --builder-image
```

It should print `Using local builder image:` or pull the new digest, and the package
should build.

Finally, rebase or re-push the onboarding PR for the package so its CI rebuild runs with
the new task bundle, and watch the build plus the wheel-check integration tests go green.

## Checklist

- [ ] Reproduced the failure with `hack/build-locally.sh` (with constraints, if any).
- [ ] Recorded the builder image digest the failure reproduced on.
- [ ] Iterated on `builder/Dockerfile.debug` (never rebuilt the full image locally).
- [ ] Explored the image with `podman run -it --rm ... bash` and `dnf provides`.
- [ ] Final green run done via `hack/build-locally.sh` itself.
- [ ] Overlay rebased onto the latest published image and re-verified.
- [ ] Fix ported into `builder/Containerfile` at the right layer; PR opened.
- [ ] Task bundle digest bumped in `tasks/build-python-wheels-oci-ta.yaml`; PR merged.
- [ ] Index pipeline bundle digest bumped; chain verified locally with
      `hack/build-locally.sh` (no override).
- [ ] Onboarding PR rebased/re-pushed; CI and wheel-check tests green.
