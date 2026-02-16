# UV for Termux - Fixed Build

Automated builds of [UV](https://github.com/astral-sh/uv) for Termux with the Android platform detection fix for Python <3.13.

## Problem

UV rebuilds cached packages on every command in Termux due to platform tag mismatch (issue [#9559](https://github.com/astral-sh/uv/issues/9559)).

## Solution

This repository automatically builds UV with a patch that returns `linux` platform tags on Python <3.13 instead of `android`, matching what build backends like setuptools produce.

## Installation

### From Releases

```bash
# Determine your architecture
uname -m  # aarch64, armv7l, x86_64, or i686

# Download and install (replace VERSION and ARCH)
pkg install wget
wget https://github.com/YOUR_USERNAME/uv-termux-fix/releases/latest/download/uv_VERSION_ARCH.deb
pkg install ./uv_*.deb
```

### As Package Source

Add this repository as a package source:

```bash
# Create sources list
echo "deb [trusted=yes] https://github.com/YOUR_USERNAME/uv-termux-fix/releases/latest/download/ termux extras" > $PREFIX/etc/apt/sources.list.d/uv-termux-fix.list

# Update and install
pkg update
pkg install uv
```

## How It Works

1. Daily checks for new UV releases
2. Fetches official Termux build script
3. Applies platform detection patch
4. Builds for all Termux architectures (aarch64, arm, x86_64, i686)
5. Creates GitHub release with .deb packages

## Patch Details

The patch modifies `crates/uv-python/python/get_interpreter_info.py` to only use Android platform tags on Python 3.13+, where they're officially supported per PEP 738/783.

```python
# Before: Always uses android tag if available
elif hasattr(sys, "getandroidapilevel"):

# After: Only uses android tag on Python 3.13+
elif hasattr(sys, "getandroidapilevel") and sys.version_info >= (3, 13):
```

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
