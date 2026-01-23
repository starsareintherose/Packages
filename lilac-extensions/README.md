# Lilac Extensions for BioArchLinux

This directory contains extension utilities for the lilac build system used in BioArchLinux.

## Files

- **lilac_r_utils.py**: Utilities for building and checking R packages
  - Version and release number management
  - DESCRIPTION file parsing
  - PKGBUILD validation
  - **NEW**: Automatic dependency fixing

## New Feature: Automatic Dependency Management

The `r_pre_build` function now supports automatic dependency fixing for R packages. This eliminates the need for manual PKGBUILD updates when dependencies change.

### Quick Start

For packages already using `r_pre_build`, no changes are needed - automatic fixing is enabled by default.

For packages using manual version updates, see [README_AUTO_FIX.md](./README_AUTO_FIX.md) for migration instructions.

### What Gets Fixed

- ✅ Unnecessary R dependencies (removes them)
- ✅ Missing R dependencies (adds them)
- ✅ Unnecessary R optional dependencies (removes them)
- ✅ Missing R optional dependencies (adds them)
- ✅ R make dependencies (LinkingTo packages)
- ✅ R dependency management (implicit vs explicit)
- ✅ lilac.yaml repo_depends and repo_makedepends (R packages only)

**Note**: Only R package dependencies (r-*) are modified. Non-R system dependencies like gcc-fortran, cmake, libxml2, etc. are preserved as-is.

### Documentation

See [README_AUTO_FIX.md](./README_AUTO_FIX.md) for complete documentation on the automatic dependency fixing feature.

## Usage in lilac.py

```python
#!/usr/bin/env python3
from lilaclib import *

import os
import sys
sys.path.append(os.path.normpath(f'{__file__}/../../../lilac-extensions'))
from lilac_r_utils import r_pre_build

def pre_build():
    r_pre_build(_G)  # Auto-fix enabled by default

def post_build():
    git_pkgbuild_commit()
    update_aur_repo()
```

## Disabling Auto-fix

If you prefer the old behavior (fail without fixing), set `auto_fix=False`:

```python
def pre_build():
    r_pre_build(_G, auto_fix=False)
```
