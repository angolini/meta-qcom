# CI helper scripts

The `ci/` directory contains scripts used by both GitHub Actions and local developer/agent workflows.
Running them locally gives you the same checks CI runs, using the same container environment.

## Prerequisites

- `kas-container` on `$PATH`, or `KAS_CONTAINER` set to its absolute path
  (download from [siemens/kas](https://github.com/siemens/kas/blob/master/kas-container))
- Docker or Podman available on the host

## The two wrappers

### `ci/kas-container-shell-helper.sh` — run a script inside the build container

```sh
ci/kas-container-shell-helper.sh <script>
```

Enters the kas container (via `kas-container shell ci/base.yml`) and runs `<script>` inside it,
passing the container-internal paths `/repo` (the meta-qcom checkout) and `/work` (the kas
workspace with oe-core, bitbake, and the build directory) as arguments.

Use this whenever you need Yocto tools (`bitbake`, `oe-selftest`, `yocto-check-layer`, etc.)
that are not installed on your host — which is the normal case.

### `ci/kas-shell-helper.sh` — same, but without a container

```sh
ci/kas-shell-helper.sh <script>
```

Like the above but calls plain `kas shell` instead of `kas-container shell`. For use when kas
and all Yocto build dependencies are already installed on the host. Manages `KAS_WORK_DIR`
automatically if not already set.

## The check scripts

All three accept `REPO_DIR` and `WORK_DIR` as positional arguments, which the wrappers above
supply automatically. Invoke them through a wrapper rather than directly.

### `ci/yocto-patchreview.sh` — validate patch metadata

Runs the oe-core `patchreview.py` tool against the meta-qcom tree and exits non-zero if any
patch has a malformed `Signed-off-by` or missing `Upstream-Status` header.

Run before every commit:

```sh
ci/kas-container-shell-helper.sh ci/yocto-patchreview.sh
```

### `ci/yocto-check-layer.sh` — validate layer compliance

Runs the official Yocto Project `yocto-check-layer` tool against meta-qcom. Checks that all
recipes parse correctly, signatures are consistent, and the layer is compliant across every
machine defined in `conf/machine/`. Uses a fresh throwaway build directory to avoid inheriting
state from a previous build.

Run before opening or updating a pull request:

```sh
ci/kas-container-shell-helper.sh ci/yocto-check-layer.sh
```

### `ci/oe-selftest.sh` — run layer self-tests

Runs `oe-selftest` for all test modules found under `lib/oeqa/selftest/cases/`. Honors
`DL_DIR` and `SSTATE_DIR` from the environment so the shared cache is reused.

Run routinely alongside patchreview:

```sh
ci/kas-container-shell-helper.sh ci/oe-selftest.sh
```

To run a specific test module instead of the full suite, call `kas-container shell` directly:

```sh
kas-container shell ci/base.yml \
  --command "/repo/ci/oe-selftest.sh /repo /work qcom_fitimage.QcomFitImageMatrixTests"
```

## Recommended order before a pull request

```sh
ci/kas-container-shell-helper.sh ci/yocto-patchreview.sh
ci/kas-container-shell-helper.sh ci/yocto-check-layer.sh
ci/kas-container-shell-helper.sh ci/oe-selftest.sh
```
