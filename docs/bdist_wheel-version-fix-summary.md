# bdist_wheel Version Consistency Fix - Summary

## Problem

CI builds were producing wheels with mixed `bdist_wheel` metadata versions:
- Some wheels: `bdist_wheel (0.41.2)`
- Others: `bdist_wheel (0.42.0)`

This inconsistency was flagged by Seal Security's wheel metadata comparer and Python verifier.

## Root Causes

1. **CI upgrades to latest** – The workflow uses `pip install -U pip wheel setuptools`, which installs the latest versions from PyPI.

2. **Different environments per platform** – cibuildwheel runs builds in:
   - **Linux**: Docker containers (manylinux images) with their own bundled wheel version
   - **macOS/Windows**: Host Python with potentially different versions
   - **Matrix jobs**: ubuntu, windows, macos, plus emulated arches (aarch64, ppc64le, s390x)

3. **Runner OS upgrades** – GitHub Actions runner images (ubuntu-latest, macos-latest, etc.) change over time, bringing newer Python/pip/wheel versions.

4. **Repair step overwrites metadata** – `auditwheel` (Linux) and `delocate` (macOS) run in the **base/container environment**, not the build venv. When they repack wheels, they use the wheel package from that environment, overwriting the `bdist_wheel` version in the WHEEL file.

## Attempted Solutions (What Didn't Work)

### 1. Pinning wheel in host pip install
```yaml
python -m pip install -U pip wheel==0.41.2 setuptools build twine
```
**Why it failed:** cibuildwheel creates its own isolated environments; the host's wheel version is not used.

### 2. CIBW_BEFORE_BUILD only
```yaml
CIBW_BEFORE_BUILD: "pip install wheel==0.41.2"
```
**Why it failed:**
- Build runs in a **PEP 517 isolated environment**; the wheel installed by before-build is ignored.
- The repair step runs in a different environment (base/container) and overwrites the metadata.

### 3. Wrong key in pyproject.toml
```toml
CIBW_BEFORE_BUILD = "pip install wheel==0.41.2"  # Wrong!
```
**Why it failed:** In `pyproject.toml`, the correct key is `before-build` (lowercase with hyphen). `CIBW_BEFORE_BUILD` is for environment variables only.

### 4. Time-machine index
Using Seal Security's time-machine PyPI index (2023-10-07):
**Why it failed:** That index only has wheel up to **0.37.1**. Versions 0.41.2 and 0.42.0 were released later and are not available.

### 5. build-system.requires with wheel==0.41.2
```toml
[build-system]
requires = ["setuptools >= 46.4.0", "wheel==0.41.2"]
```
**Why it wasn't enough:** The build may produce wheels with 0.41.2, but the **repair step** (auditwheel/delocate) runs in the base environment and repacks using its own wheel package (0.42.0), overwriting the metadata.

### 6. CIBW_BUILD_FRONTEND with --no-isolation
```yaml
CIBW_BUILD_FRONTEND: "build; args: --no-isolation"
```
**Why it wasn't enough:** Even with the build using the correct wheel version, the repair step still runs in a different environment and overwrites the result.

## Solution: CIBW_BEFORE_ALL

The repair step runs in the **base/container environment**. Install the desired wheel version there using `CIBW_BEFORE_ALL`, which runs before any builds in that environment.

### In pyproject.toml
```toml
[tool.cibuildwheel]
test-command = ""
skip = "pp*"
before-all = "pip install --index-url https://pypi.org/simple wheel==0.41.2"
before-build = "pip install --index-url https://pypi.org/simple wheel==0.41.2"
```

### In workflow env
```yaml
env:
  CIBW_ARCHS_MACOS: x86_64 arm64 universal2
  CIBW_SKIP: cp312-* pp*
  CIBW_BEFORE_ALL: "pip install --index-url https://pypi.org/simple wheel==0.41.2"
  CIBW_BEFORE_BUILD: "pip install --index-url https://pypi.org/simple wheel==0.41.2"
```

### Why use both?
- **before-all**: Ensures the base/container environment (where auditwheel/delocate runs) has wheel 0.41.2.
- **before-build**: Ensures each build venv has wheel 0.41.2 (needed if using `--no-isolation`).

### Alternative: build-system.requires
```toml
[build-system]
requires = [
    "setuptools >= 46.4.0",
    "wheel==0.41.2",
]
```
Use this if the build can access PyPI. Combine with `before-all` so the repair step also uses 0.41.2.

## Key Takeaways

| Concept | Detail |
|--------|--------|
| **Build isolation** | `python -m build` creates a fresh env by default; `--no-isolation` uses the current env |
| **pyproject.toml keys** | Use `before-build`, `before-all` (lowercase with hyphens), not `CIBW_BEFORE_BUILD` |
| **Environment split** | Build venv ≠ repair environment; repair uses base/container Python |
| **Time-machine index** | Snapshot from 2023-10-07 only has wheel up to 0.37.1 |
| **Index for wheel** | Use `--index-url https://pypi.org/simple` when time-machine doesn't have the version |

## References

- [cibuildwheel options](https://cibuildwheel.pypa.io/en/stable/options/)
- [before-build](https://cibuildwheel.pypa.io/en/stable/options/#before-build)
- [before-all](https://cibuildwheel.pypa.io/en/stable/options/#before-all)
