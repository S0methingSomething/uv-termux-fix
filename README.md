# UV for Termux - Fixed Build

Automated builds of [UV](https://github.com/astral-sh/uv) for Termux with a patch that makes UV accept both `android` and `manylinux` tagged wheels, fixing cache mismatches that cause endless package rebuilds.

## Problem

UV rebuilds cached packages on every command in Termux due to a platform tag mismatch (issue [#9559](https://github.com/astral-sh/uv/issues/9559)). Build backends like setuptools and maturin produce `manylinux` wheels, but UV on Android only accepts `android`-tagged wheels, so cached packages are never reused.

On Python 3.13+, PEP 738/783 formalized `android` platform tags, but they remain impractical on Termux — almost no PyPI packages publish android wheels, and the API level reported by `sys.getandroidapilevel()` can easily mismatch what build backends use, causing further compatibility issues.

## Solution

This repository builds UV with a Rust-level patch that adds `manylinux` to the list of compatible platform tags when running on Android. UV still correctly identifies the platform as Android, but now accepts both `android_*` and `manylinux_*` tagged wheels.

## Installation

```bash
# Download and install (aarch64 only)
pkg install wget
wget https://github.com/S0methingSomething/uv-termux-fix/releases/latest/download/uv_VERSION_aarch64.deb
pkg install ./uv_*.deb
```

Replace `VERSION` with the version number from the [latest release](https://github.com/S0methingSomething/uv-termux-fix/releases/latest).

### Optional: Fix build backend tags

Some build backends (like maturin on Python 3.13+) produce `android`-tagged wheels. The patched UV accepts these, but if you want build backends to also produce `manylinux` tags for consistency, add this to your `~/.bashrc`:

```bash
export _PYTHON_HOST_PLATFORM=linux-aarch64
```

This overrides `sysconfig.get_platform()` so all build backends produce `manylinux`-tagged wheels.

## How It Works

1. Daily checks for new UV releases
2. Fetches the official Termux build scripts
3. Applies the manylinux compatibility patch
4. Builds for aarch64 (64-bit ARM)
5. Creates a GitHub release with the .deb package

## Patch Details

The patch modifies `crates/uv-platform-tags/src/tags.rs` to add `manylinux` compatible tags to the Android platform tag list:

```rust
// Before: Android only generates android_* tags
(Os::Android { api_level }, arch) => {
    for ver in (16..=*api_level).rev() {
        platform_tags.push(PlatformTag::Android { api_level: ver, abi });
    }
    platform_tags
}

// After: Also generates manylinux_* tags for compatibility
(Os::Android { api_level }, arch) => {
    for ver in (16..=*api_level).rev() {
        platform_tags.push(PlatformTag::Android { api_level: ver, abi });
    }
    // Accept manylinux wheels too (produced by setuptools/maturin on Termux)
    if let Some(min_minor) = arch.get_minimum_manylinux_minor() {
        for minor in (min_minor..=17).rev() {
            platform_tags.push(PlatformTag::Manylinux { major: 2, minor, arch });
        }
        platform_tags.push(PlatformTag::Linux { arch });
    }
    platform_tags
}
```

This works because:

- UV still truthfully reports Android as the platform
- Both `android` and `manylinux` tagged wheels are accepted
- Build backends can produce either tag and UV will use the cached result
- No need to lie about the platform or patch Python's sysconfig

See the full patch at [`patches/0001-android-accept-manylinux-wheels.patch`](patches/0001-android-accept-manylinux-wheels.patch).

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
