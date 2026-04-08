# README: Building a Custom Yocto Recipe — Troubleshooting Guide

## Table of Contents
1. [Overview](#overview)
2. [Recipe Structure](#recipe-structure)
3. [Yocto Variables Explained](#yocto-variables-explained)
4. [Common Errors & Solutions](#common-errors--solutions)
5. [Useful BitBake Commands](#useful-bitbake-commands)
6. [Final Working Recipe](#final-working-recipe)

---

## Overview

This guide documents the process of creating a custom Yocto recipe (`hello-ayman`) that fetches source code from a GitHub repository, compiles it, and packages it for a **Raspberry Pi 3 (64-bit)** target using the **Scarthgap** release of Poky.

---

## Recipe Structure

A proper Yocto recipe must follow a specific directory structure:

```
meta-main/
├── conf/
│   └── layer.conf
└── recipes-test/
    └── hello-ayman/          # Subdirectory (required!)
        └── hello-ayman.bb    # Recipe file
```

> **Important:** The `BBFILES` pattern in `layer.conf` typically expects:
> ```
> BBFILES += "${LAYERDIR}/recipes-*/*/*.bb"
> ```
> This means recipes must be inside a **subdirectory** under `recipes-*`, not directly in it.

### ❌ Wrong Structure
```
recipes-test/
└── hello-ayman.bb        # BitBake won't find this!
```

### ✅ Correct Structure
```
recipes-test/
└── hello-ayman/
    └── hello-ayman.bb    # BitBake finds this
```

---

## Yocto Variables Explained

### Recipe Metadata Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SUMMARY` | Short one-line description of the recipe | `"Hello World Application"` |
| `DESCRIPTION` | Longer description of what the recipe does | `"A simple test Yocto recipe"` |
| `LICENSE` | License of the source code. Must use valid SPDX identifiers separated by `&`, `\|`, `(`, `)`. Use `"CLOSED"` for proprietary/closed-source software | `"MIT"`, `"GPLv2"`, `"CLOSED"` |
| `LIC_FILES_CHKSUM` | Checksum of the license file in source code (not needed for `CLOSED`) | `"file://LICENSE;md5=abc123..."` |

### Source Code Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SRC_URI` | Where to fetch the source code from. Supports `git://`, `http://`, `file://`, etc. | `"git://github.com/user/repo.git;protocol=https;branch=main"` |
| `SRCREV` | The exact Git commit hash to use. Ensures reproducible builds | `"3c801282b99e0f40ebd044bb3fd7c2e03b8b09a5"` |
| `S` | The source directory — where BitBake expects the unpacked source code to be. For `git://` fetcher, this must be `"${WORKDIR}/git"` | `"${WORKDIR}/git"` |

### Build Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `${CC}` | The cross-compiler (e.g., `aarch64-poky-linux-gcc`) | Used in `do_compile()` |
| `${CFLAGS}` | Compiler flags | Used in `do_compile()` |
| `${LDFLAGS}` | Linker flags | Used in `do_compile()` |
| `${WORKDIR}` | The working directory for the recipe | Auto-set by BitBake |
| `${D}` | The destination directory (fakeroot) for `do_install()` | Used in `do_install()` |
| `${bindir}` | The binary installation directory (usually `/usr/bin`) | Used in `do_install()` |

### Image Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `IMAGE_INSTALL:append` | Add packages to the final root filesystem image | `" hello-ayman"` (note the leading space!) |
| `MACHINE` | The target hardware | `"raspberrypi3-64"` |
| `DISTRO` | The distribution being built | `"poky"` |

### SRC_URI Parameters (for git)

| Parameter | Description | Example |
|-----------|-------------|---------|
| `protocol` | Protocol to use for fetching | `https`, `ssh`, `git` |
| `branch` | Git branch to fetch from | `main`, `master`, `dev` |
| `tag` | Git tag to fetch | `v1.0` |
| `subpath` | Subdirectory within the repo | `src` |

---

## Common Errors & Solutions

### Error 1: `LICENSE value "CLOSE SOURCE" has an invalid format`

**Cause:** Invalid license format. License names cannot have spaces.

```bash
# ❌ Wrong
LICENSE = "CLOSE SOURCE"

# ✅ Correct
LICENSE = "CLOSED"
```

> Valid examples: `"MIT"`, `"GPLv2"`, `"Apache-2.0"`, `"MIT & GPLv2"`, `"CLOSED"`

---

### Error 2: `Port could not be cast to integer value as 'ayman4105'`

**Cause:** Using a colon (`:`) instead of a slash (`/`) in the GitHub URL. This is a common mistake when copying the SSH-style URL.

```bash
# ❌ Wrong (SSH format — colon after github.com)
SRC_URI = "git://github.com:ayman4105/test_yocto.git;protocol=https;branch=main"

# ✅ Correct (slash after github.com)
SRC_URI = "git://github.com/ayman4105/test_yocto.git;protocol=https;branch=main"
```

> The parser interprets `github.com:ayman4105` as `host:port`, and since `ayman4105` is not a number, it fails.

---

### Error 3: `Nothing PROVIDES 'hello-ayman'`

**Cause 1:** The layer containing the recipe is not added to `bblayers.conf`.

```bash
# Check active layers
bitbake-layers show-layers

# Add your layer
bitbake-layers add-layer /path/to/meta-main
```

**Cause 2:** The recipe is not in the correct directory structure.

```bash
# ❌ Wrong — recipe directly in recipes-test/
recipes-test/
└── hello-ayman.bb

# ✅ Correct — recipe in a subdirectory
recipes-test/
└── hello-ayman/
    └── hello-ayman.bb
```

---

### Error 4: `undefined reference to 'main'`

**Cause:** A bug in the source code — `intmain()` instead of `int main()`.

```c
// ❌ Wrong
intmain() {
    printf("Hello\n");
}

// ✅ Correct
int main() {
    printf("Hello\n");
    return 0;
}
```

**Important:** After fixing the code on GitHub:
1. **Commit and push** the fix
2. **Update `SRCREV`** in the recipe to the new commit hash
3. **Delete the git download cache** and rebuild

```bash
# Get the new commit hash
git ls-remote https://github.com/user/repo.git HEAD

# Update SRCREV in the recipe, then:
rm -rf /path/to/downloads/git2/github.com.user.repo.git
bitbake -c cleansstate hello-ayman
bitbake hello-ayman
```

---

### Error 5: Old Code Still Being Compiled After Fix

**Cause:** BitBake caches the downloaded source. Even after `cleansstate`, the git download cache may still have old code.

```bash
# Step 1: Delete git download cache
rm -rf /path/to/downloads/git2/github.com.username.repo.git

# Step 2: Update SRCREV in recipe to new commit hash

# Step 3: Clean and rebuild
bitbake -c cleansstate hello-ayman
bitbake hello-ayman
```

---

### Error 6: Editing the Wrong Recipe File

**Cause:** Having **duplicate recipe files** — BitBake may parse a different one than you're editing.

```bash
# ❌ Two files with the same recipe name
recipes-test/
├── Aman/
│   └── hello-ayman.bb    ← BitBake parses THIS one
└── hello-ayman.bb        ← You're editing THIS one

# ✅ Keep only ONE recipe file
recipes-test/
└── hello-ayman/
    └── hello-ayman.bb
```

---

## Useful BitBake Commands

| Command | Description |
|---------|-------------|
| `bitbake hello-ayman` | Build the recipe |
| `bitbake -c compile -f hello-ayman` | Force recompile |
| `bitbake -c cleansstate hello-ayman` | Clean everything (build + cache) |
| `bitbake -c fetch -f hello-ayman` | Force re-fetch source code |
| `bitbake -c devshell hello-ayman` | Open a development shell in the build directory |
| `bitbake-layers show-layers` | Show all active layers |
| `bitbake-layers add-layer /path/to/layer` | Add a layer |
| `bitbake-layers remove-layer meta-main` | Remove a layer |
| `bitbake -e hello-ayman \| grep ^SRC_URI=` | Show the expanded value of `SRC_URI` |
| `bitbake -e hello-ayman \| grep ^SRCREV=` | Show the expanded value of `SRCREV` |

---

## Final Working Recipe

**File:** `meta-main/recipes-test/hello-ayman/hello-ayman.bb`

```bitbake
SUMMARY = "ayman"
DESCRIPTION = "TEST YOCTO RECIPE"
LICENSE = "CLOSED"

SRC_URI = "git://github.com/ayman4105/test_yocto.git;protocol=https;branch=main"
SRCREV = "<your_latest_commit_hash>"

S = "${WORKDIR}/git"

do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} ${S}/main.c -o ayman
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${WORKDIR}/build/ayman ${D}${bindir}/ayman
}
```

### Source Code (`main.c`)

```c
#include <stdio.h>

int main() {
    printf("===========================================\n");
    printf("        Hello Ayman\n");
    printf("===========================================\n");
    return 0;
}
```

### Adding to Image

In `build/conf/local.conf`:
```
IMAGE_INSTALL:append = " hello-ayman"
```

Then rebuild the image:
```bash
bitbake core-image-minimal
```

---

## Summary of Lessons Learned

| # | Lesson |
|---|--------|
| 1 | `LICENSE = "CLOSED"` — no spaces, exact keyword |
| 2 | Use `/` not `:` in `git://` URLs |
| 3 | Recipes must be in a **subdirectory** under `recipes-*` |
| 4 | Your custom layer must be **added** to `bblayers.conf` |
| 5 | Always update `SRCREV` when source code changes |
| 6 | Delete git download cache when old code persists |
| 7 | `S = "${WORKDIR}/git"` is **required** for git-based recipes |
| 8 | Don't have **duplicate** recipe files |
