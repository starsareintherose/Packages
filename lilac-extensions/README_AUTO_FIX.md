# Automatic Dependency Management for R Packages

This document describes the automatic dependency fixing feature in `lilac_r_utils.py`.

## Overview

The `r_pre_build` function now includes automatic dependency fixing capability. When enabled (which is the default), it will automatically correct dependency issues in PKGBUILD files for R packages.

## What Gets Fixed Automatically

The automatic fix feature handles the following dependency issues:

### 1. **Unnecessary Dependencies**
- Removes R packages listed in `depends` that are not in `Depends` or `Imports` fields of the DESCRIPTION file
- Removes R packages that are included in the base R distribution
- Example: If `r-dplyr` is in PKGBUILD `depends` but not in DESCRIPTION `Imports`, it will be removed

### 2. **Missing Dependencies**
- Adds R packages from DESCRIPTION `Depends` and `Imports` that are missing from PKGBUILD `depends`
- Example: If `r-ggplot2` is in DESCRIPTION `Imports` but not in PKGBUILD `depends`, it will be added

### 3. **Unnecessary Optional Dependencies**
- Removes R packages from `optdepends` that are already in `depends`
- Removes R packages from `optdepends` that are not in DESCRIPTION `Suggests`
- Example: If `r-knitr` is in both `depends` and `optdepends`, it will be removed from `optdepends`

### 4. **Missing Optional Dependencies**
- Adds R packages from DESCRIPTION `Suggests` that are missing from PKGBUILD `optdepends`
- Example: If `r-testthat` is in DESCRIPTION `Suggests` but not in PKGBUILD `optdepends`, it will be added

### 5. **Make Dependencies**
- Removes `makedepends` that are already in `depends`
- Removes R package `makedepends` that are not in DESCRIPTION `LinkingTo`
- Adds missing packages from DESCRIPTION `LinkingTo` to `makedepends`
- Automatically handles `gcc-fortran` based on presence of Fortran source files

### 6. **R Dependency**
- Correctly manages the `r` dependency:
  - If there are other R package dependencies (r-*), `r` is implicit and removed if explicitly listed
  - If there are no R package dependencies, `r` is added explicitly

## Usage

### Migrating Existing Packages

If your package currently uses a simple `lilac.py` that manually updates versions, you can migrate to use `r_pre_build` for automatic dependency management.

**Before (manual version update only):**
```python
#!/usr/bin/env python3
from lilaclib import *

def pre_build():
    for line in edit_file('PKGBUILD'):
        if line.startswith('_pkgver='):
            line = f'_pkgver={_G.newver}'
        print(line)
    update_pkgver_and_pkgrel(_G.newver.replace(':', '.').replace('-', '.'))

def post_build():
    git_pkgbuild_commit()
    update_aur_repo()
```

**After (with automatic dependency fixing):**
```python
#!/usr/bin/env python3
from lilaclib import *

import os
import sys
sys.path.append(os.path.normpath(f'{__file__}/../../../lilac-extensions'))
from lilac_r_utils import r_pre_build

def pre_build():
    r_pre_build(_G)

def post_build():
    git_pkgbuild_commit()
    update_aur_repo()
```

By default, `auto_fix=True`, so dependency issues will be automatically corrected.

### Default Behavior (Auto-fix Enabled)

The simplest way to use this feature is to just call `r_pre_build` with `_G`:

```python
#!/usr/bin/env python3
from lilaclib import *

import os
import sys
sys.path.append(os.path.normpath(f'{__file__}/../../../lilac-extensions'))
from lilac_r_utils import r_pre_build

def pre_build():
    r_pre_build(_G)

def post_build():
    git_pkgbuild_commit()
    update_aur_repo()
```

By default, `auto_fix=True`, so dependency issues will be automatically corrected.

### Disable Auto-fix

If you want to keep the old behavior (fail on dependency issues without fixing), set `auto_fix=False`:

```python
def pre_build():
    r_pre_build(_G, auto_fix=False)
```

### With Additional Configuration

You can combine auto-fix with other CheckConfig options:

```python
def pre_build():
    r_pre_build(
        _G,
        auto_fix=True,  # Enable automatic fixing (default)
        expect_license="GPL-3",  # Still check license
        extra_r_makedepends=['r-rcpp']  # Additional make dependencies
    )
```

## How It Works

1. **Update Version**: Updates `_pkgver` and `pkgrel` in PKGBUILD
2. **Update Checksums**: Runs `updpkgsums` to update package checksums
3. **Check Dependencies**: Runs `r_check_pkgbuild` to detect dependency issues
4. **Auto-fix (if enabled)**:
   - If dependency errors are detected, automatically fixes them by:
     - Parsing the current PKGBUILD to get current dependencies
     - Parsing the DESCRIPTION file from the R package to get expected dependencies
     - Computing the correct dependencies
     - Rewriting the PKGBUILD with corrected dependency arrays
     - Re-running `updpkgsums`
     - Re-checking to ensure fixes were successful
5. **Final Check**: Validates that all checks pass after fixing

## Example

### Before Auto-fix

```bash
PKGBUILD depends=(
  r
  r-dplyr
  r-glue        # Unnecessary - not in DESCRIPTION
  r-pillar      # Unnecessary - not in DESCRIPTION
)
optdepends=(
  r-knitr
  # Missing: r-testthat
)
```

### After Auto-fix

```bash
PKGBUILD depends=(
  r-dplyr
  r-ggplot2     # Added - was missing
)
optdepends=(
  r-knitr
  r-testthat    # Added - was missing
)
```

## Benefits

1. **Reduces Manual Work**: No need to manually update PKGBUILD files when R package dependencies change
2. **Consistency**: Ensures PKGBUILD dependencies match DESCRIPTION metadata
3. **Error Prevention**: Catches and fixes dependency issues before build failures
4. **Time Saving**: Automated bot can handle dependency updates without maintainer intervention

## Limitations

- Only fixes dependency-related issues
- Other issues (license changes, system requirements, etc.) still require manual intervention or configuration
- Non-R dependencies (system libraries) are preserved but not validated
- The feature requires the package source tarball to be available for parsing the DESCRIPTION file

## Backward Compatibility

This feature is backward compatible:
- Existing `lilac.py` files will automatically benefit from auto-fix
- Packages that don't want auto-fix can explicitly set `auto_fix=False`
- All existing CheckConfig parameters still work as before
