# UV for Termux - Fixed Build

Automated builds of [UV](https://github.com/astral-sh/uv) for Termux with a patch that forces `manylinux` platform tags on Android, fixing cache mismatches that cause endless package rebuilds.

## Problem

UV rebuilds cached packages on every command in Termux due to a platform tag mismatch (issue [#9559](https://github.com/astral-sh/uv/issues/9559)). Build backends like setuptools and maturin produce `manylinux` wheels, but UV on Android expects `android`-tagged wheels, so cached packages are never reused.

On Python 3.13+, the problem is worse: `sys.getandroidapilevel()` returns the NDK API level baked into the Python build (e.g. 24), while build backends use the device's actual API level (e.g. 36), producing wheels that UV rejects as incompatible.

## Solution

This repository automatically builds UV with a patch that always returns `manylinux_2_17` platform tags instead of `android`, matching what build backends actually produce on Termux.

## Installation

```bash
# Download and install (aarch64 only)
pkg install wget
wget https://github.com/S0methingSomething/uv-termux-fix/releases/latest/download/uv_VERSION_aarch64.deb
pkg install ./uv_*.deb
```

Replace `VERSION` with the version number from the [latest release](https://github.com/S0methingSomething/uv-termux-fix/releases/latest).

## How It Works

1. Daily checks for new UV releases
2. Fetches the official Termux build scripts
3. Applies the manylinux platform tag patch
4. Builds for aarch64 (64-bit ARM)
5. Creates a GitHub release with the .deb package

## Patch Details

The patch modifies `crates/uv-python/python/get_interpreter_info.py` to always report `manylinux` instead of `android` as the platform:

```python
# Before: Uses android tag (causes cache mismatches)
elif hasattr(sys, "getandroidapilevel"):
    operating_system = {
        "name": "android",
        "api_level": sys.getandroidapilevel(),
    }

# After: Always uses manylinux tag (matches build backend output)
elif hasattr(sys, "getandroidapilevel"):
    operating_system = {
        "name": "manylinux",
        "major": 2,
        "minor": 17,
    }
```

This works because:

- Almost no PyPI packages publish native `android` wheels
- Build backends on Termux produce `manylinux`-tagged wheels
- Using matching tags prevents UV from rebuilding packages on every invocation

See the full patch at [`patches/0001-always-use-manylinux-tags-on-android-termux.patch`](patches/0001-always-use-manylinux-tags-on-android-termux.patch).

## Verification

After installation, verify the fix works:

```bash
# Create test project
mkdir test-uv && cd test-uv
uv init
uv add pyzmq==25.1.1

# This should use cache, not rebuild
cd ..
mkdir test-uv2 && cd test-uv2
uv init
uv add pyzmq==25.1.1  # Should be instant from cache
```

## Credits

- [UV](https://github.com/astral-sh/uv) by Astral
- [Termux](https://github.com/termux/termux-packages)
- Fix based on discussion in [issue #9559](https://github.com/astral-sh/uv/issues/9559)
