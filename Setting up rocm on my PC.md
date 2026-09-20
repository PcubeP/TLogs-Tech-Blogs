# ROCm 10.0 Setup on WSL2 --- RX 9070 XT

## Overview

This document records the complete process used to set up the ROCm
userspace compute stack for an AMD Radeon RX 9070 XT inside WSL2 on
Windows.

The goal was to:

-   Make the RX 9070 XT available to Linux/WSL applications through the
    Windows GPU virtualization path.
-   Install ROCm at the system level.
-   Target the GPU's RDNA 4 `gfx1201` architecture.
-   Avoid reinstalling or replacing the Windows AMD display driver.
-   Avoid using AMD's `amdgpu-install` stack.
-   Keep ROCm independent from future ComfyUI installations so that
    reinstalling ComfyUI does not require reinstalling ROCm.

No usernames, credentials, account information, or other
machine-specific secrets are included here.

------------------------------------------------------------------------

## 1. Initial Environment

The relevant environment was:

-   Windows 11 25H2
-   WSL2
-   Ubuntu 26.04 LTS (`resolute`)
-   WSL kernel 6.18.x
-   AMD Radeon RX 9070 XT
-   GPU architecture / ISA: `gfx1201`
-   ROCm target: ROCm 10.0
-   WSL GPU device: `/dev/dxg`

The Windows AMD driver was already installed and functioning. The
objective was **not** to replace it.

The WSL GPU path was based on Microsoft's `/dev/dxg` interface rather
than a traditional Linux `/dev/dri` device.

------------------------------------------------------------------------

## 2. The First Major Decision: Do Not Use `amdgpu-install`

An initial attempt involved investigating AMD's `amdgpu-install`
package.

The package was present, but using the AMD installer raised concerns
because it appeared to be preparing a much larger AMD driver/software
installation. At one point, the installer path indicated approximately
17 GB of space requirements.

The important distinction was established:

### What we wanted

ROCm userspace components:

-   HIP
-   HSA runtime
-   ROCm libraries
-   ROCm compiler/toolchain
-   GPU-specific ROCm packages
-   WSL ROCDXG support

### What we did not want

A full Linux AMD kernel/display driver installation through
`amdgpu-install`.

Because WSL already exposes the Windows GPU through `/dev/dxg`,
reinstalling a traditional Linux AMD driver stack was unnecessary for
this setup.

**Decision:** remove `amdgpu-install` and build the ROCm userspace stack
directly from the ROCm repository.

------------------------------------------------------------------------

## 3. Cleaning Up `amdgpu-install`

The `amdgpu-install` package was removed and then purged.

After cleanup, the package check showed that the installer itself and
ROCm/HIP packages were no longer present.

The remaining `libdrm-amdgpu1` package was a normal userspace DRM
library and was not treated as an AMD driver installation that needed to
be removed.

The AMD installer repository files were also removed as part of the
purge.

This left a clean starting point for installing ROCm directly.

------------------------------------------------------------------------

## 4. Adding the ROCm Repository

The official AMD ROCm repository for Ubuntu 26.04 / `resolute` was
verified first.

The repository used was:

``` text
https://repo.radeon.com/rocmradeon/apt/26.14
```

A repository entry was then created using a dedicated keyring:

``` text
deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocmradeon/apt/26.14 resolute main
```

### First hurdle: missing repository signing key

The first `apt update` failed with:

``` text
NO_PUBKEY 9386B48A1A693C5C
```

The official ROCm repository signing key was restored into:

``` text
/etc/apt/keyrings/rocm.gpg
```

After that, `apt update` succeeded and downloaded the ROCm repository
package index.

------------------------------------------------------------------------

## 5. Avoiding Ubuntu's Generic `rocm` Package

During package investigation, Ubuntu's own repositories exposed a
generic package named:

``` text
rocm
```

Its candidate version came from Ubuntu's `resolute/universe` repository
rather than the AMD ROCm 10.0 repository.

We deliberately did **not** install that generic package.

Instead, the AMD repository provided versioned ROCm 10.0 packages,
including:

``` text
amdrocm-base10.0
amdrocm-runtime10.0
amdrocm-llvm10.0
amdrocm10.0-gfx1201
```

The GPU-specific package was selected:

``` text
amdrocm10.0-gfx1201
```

This ensured that the installation matched the RX 9070 XT's `gfx1201`
target.

------------------------------------------------------------------------

## 6. Inspecting the ROCm Package Before Installation

Several packages were inspected to understand what would actually be
installed.

The ROCm runtime depended on components such as:

``` text
amdrocm-sysdeps10.0
amdrocm-base10.0
amdrocm-llvm10.0
```

The LLVM component was substantial, while the GPU-specific core package
pulled in the ROCm libraries needed for:

-   BLAS
-   FFT
-   solver
-   sparse
-   DNN
-   RCCL
-   HIP
-   ROCm runtime
-   AMD SMI
-   other ROCm components

The `amdrocm-core10.0-gfx1201` dependency chain effectively represented
the complete ROCm compute stack for the GPU.

------------------------------------------------------------------------

## 7. Simulating the Full Installation

Before committing to the installation, the following simulation was
used:

``` bash
sudo apt install --simulate amdrocm10.0-gfx1201
```

The simulation showed a large dependency set:

-   93 packages to be installed
-   4 packages to be upgraded
-   No packages to be removed

The dependency set included compiler and multilib packages such as
GCC/G++ components.

The exact final disk-summary line was difficult to extract from the
simulation output, so the installation size was ultimately confirmed by
APT when the real installation was started.

### Actual size reported by APT

APT reported:

-   Download size: approximately **980 MB**
-   Additional disk space required: approximately **6.2 GB**

This was considered acceptable.

The user explicitly decided to proceed with the full ROCm userspace
stack, provided that it did not reinstall the AMD Windows/Linux driver
stack through `amdgpu-install`.

------------------------------------------------------------------------

## 8. Installing ROCm 10.0 for `gfx1201`

The actual installation command was:

``` bash
sudo apt install amdrocm10.0-gfx1201
```

This installed the ROCm 10.0 userspace stack and the GPU-specific
`gfx1201` components.

The installation completed successfully.

APT configured the ROCm alternatives, including:

``` text
/opt/rocm/core-10.0
/opt/rocm/core-10.0/bin
/opt/rocm/core-10.0/lib
/opt/rocm/core-10.0/include
/opt/rocm/core-10.0/llvm
/opt/rocm/core-10.0/amdgcn
```

It also registered commands such as:

``` text
hipcc
hipconfig
rocminfo
amd-smi
rocm_agent_enumerator
rocm-smi
rocprofv3
amdclang
amdclang++
amdflang
```

The resulting ROCm installation was therefore system-level and available
independently of any future Python environment.

------------------------------------------------------------------------

## 9. First ROCm Verification --- Initial Failure

The first GPU verification attempted:

``` bash
rocminfo | grep -E 'Name:|gfx1201' | head -20
```

It failed with:

``` text
Cannot load librocdxg.so
```

followed by:

``` text
undefined symbol: hsaKmtOpenKFD
```

This was an important WSL-specific hurdle.

### Diagnosis

ROCm itself was installed, but the WSL ROCDXG bridge library was
missing.

The relevant missing component was:

``` text
librocdxg.so
```

ROCDXG is the component that allows the ROCm/HSA userspace stack to
communicate through WSL's GPU virtualization interface.

At this point, the decision was **not** to reinstall ROCm and **not** to
use `amdgpu-install`.

Instead, the missing WSL bridge was installed separately.

------------------------------------------------------------------------

## 10. Installing the WSL ROCDXG Component

Searching the configured APT repository for `rocdxg` returned no
package.

The official ROCm `librocdxg` release package was therefore downloaded
separately.

The package used was:

``` text
rocdxg-roct_1.2.2_amd64.deb
```

It was downloaded from the official ROCm/librocdxg GitHub release.

The package was inspected before installation:

``` bash
dpkg-deb -I rocdxg-roct_1.2.2_amd64.deb | grep -E 'Package:|Version:|Depends:'
```

It reported:

``` text
Package: rocdxg-roct
Version: 1.2.2
```

No dependency requirements were reported.

The package was then installed with:

``` bash
sudo apt install ./rocdxg-roct_1.2.2_amd64.deb
```

------------------------------------------------------------------------

## 11. Verifying ROCDXG

After installation, the expected library chain was checked.

First:

``` bash
ls -l /opt/rocm/core-10.0/lib/librocdxg.so
```

which showed:

``` text
librocdxg.so -> librocdxg.so.1
```

Then:

``` bash
ls -l /opt/rocm/core-10.0/lib/librocdxg.so.1
```

which showed:

``` text
librocdxg.so.1 -> librocdxg.so.1.2.2
```

Finally, the actual library was checked:

``` bash
ls -lh /opt/rocm/core-10.0/lib/librocdxg.so.1.2.2
```

The actual file existed and was approximately 475 KB.

This confirmed that the missing WSL bridge library was now physically
present and correctly linked.

------------------------------------------------------------------------

## 12. Successful `rocminfo` GPU Detection

The original `rocminfo` command was run again:

``` bash
rocminfo | grep -E 'Name:|gfx1201' | head -20
```

This time it successfully reported:

``` text
Name:                    AMD Ryzen 7 9800X3D 8-Core Processor
Marketing Name:          AMD Ryzen 7 9800X3D 8-Core Processor
Vendor Name:             CPU
Name:                    gfx1201
Marketing Name:          AMD Radeon RX 9070 XT
Vendor Name:             AMD
Name:                    amdgcn-amd-amdhsa--gfx1201
Name:                    amdgcn-amd-amdhsa--gfx12-generic
```

This was the critical confirmation that the GPU was accessible from ROCm
inside WSL.

------------------------------------------------------------------------

## 13. HIP Verification

Next, HIP was inspected using:

``` bash
hipconfig --full
```

Important results included:

``` text
HIP version: 7.15.26333-0000000
HIP_PATH: /opt/rocm/core-10.0
ROCM_PATH: /opt/rocm/core-10.0
HIP_COMPILER: clang
HIP_PLATFORM: amd
HIP_RUNTIME: rocclr
```

The ROCm LLVM/Clang installation was also detected:

``` text
AMD clang version 23.0.0git
```

The output contained two non-blocking warnings:

``` text
/opt/rocm/core-10.0/lib/llvm/bin/llc: not found
```

and:

``` text
Syntax error: "(" unexpected
```

The important point was that these warnings did not prevent HIP from
identifying the AMD platform or the installed ROCm toolchain.

Rather than modifying unrelated components immediately, the setup was
tested further.

------------------------------------------------------------------------

## 14. Final GPU Architecture Enumeration

The final verification was:

``` bash
/opt/rocm/core-10.0/bin/rocm_agent_enumerator
```

It returned:

``` text
gfx1201
```

This confirmed that ROCm's agent enumerator recognizes the RX 9070 XT's
GPU target.

------------------------------------------------------------------------

# Final Working Architecture

The resulting stack is effectively:

``` text
Windows 11
    │
    ├── AMD Radeon RX 9070 XT Windows driver
    │
    ▼
WSL2
    │
    ├── /dev/dxg
    │
    ▼
ROCDXG 1.2.2
    │
    ▼
ROCm 10.0 userspace
    │
    ├── HSA runtime
    ├── HIP
    ├── ROCm libraries
    ├── LLVM / AMD Clang
    ├── ROCm tools
    └── gfx1201-specific components
    │
    ▼
AMD Radeon RX 9070 XT
    │
    └── gfx1201
```

------------------------------------------------------------------------

# Key Decisions Made

## 1. No `amdgpu-install`

The AMD `amdgpu-install` route was intentionally abandoned.

Reason:

-   It was oriented toward a broader AMD driver/software stack.
-   It produced a much larger installation footprint during
    investigation.
-   The existing Windows AMD driver already provided the GPU to WSL.
-   The goal was ROCm userspace compute support, not a Linux
    display/kernel driver installation.

## 2. Use AMD's ROCm 10.0 repository

The AMD ROCm repository was used instead of Ubuntu's generic `rocm`
package.

This provided the intended ROCm 10.0 package family and the GPU-specific
`gfx1201` package.

## 3. Install the complete ROCm stack

The user explicitly accepted the full ROCm userspace stack after APT
reported approximately:

``` text
980 MB download
6.2 GB additional disk space
```

This was preferable to trying to manually assemble a minimal subset and
potentially missing required runtime components.

## 4. Install ROCDXG separately

The ROCm package installation alone did not provide the required WSL
`librocdxg.so` bridge.

The separate `rocdxg-roct` package was therefore installed.

## 5. Keep ROCm separate from ComfyUI

ROCm is now installed system-wide under:

``` text
/opt/rocm/core-10.0
```

Future ComfyUI work should use a separate Python environment.

The intention is:

``` text
System WSL
└── ROCm 10.0
    └── RX 9070 XT support

ComfyUI environment
└── Python environment
    ├── ComfyUI
    ├── PyTorch / ROCm-compatible packages
    └── ComfyUI dependencies
```

This means reinstalling ComfyUI should not require reinstalling the
underlying ROCm system stack.

------------------------------------------------------------------------

# Problems Encountered and Resolutions

  -----------------------------------------------------------------------
  Problem                             Resolution
  ----------------------------------- -----------------------------------
  `amdgpu-install` appeared to        Abandoned the installer route
  require a very large installation   

  Need to preserve existing Windows   Used ROCm userspace rather than
  AMD driver                          reinstalling AMD drivers

  Ubuntu provided a generic `rocm`    Used AMD's ROCm 10.0 repository
  package                             instead

  ROCm repository initially failed    Restored the official ROCm signing
  with `NO_PUBKEY`                    key

  `amdrocm10.0-gfx1201` initially     Added and refreshed the correct AMD
  could not be located                ROCm repository

  Full ROCm dependency tree was large Simulated installation before
                                      proceeding

  APT reported \~980 MB download /    Accepted full ROCm userspace
  \~6.2 GB disk usage                 installation

  `rocminfo` initially failed to load Installed the separate ROCDXG WSL
  `librocdxg.so`                      package

  `apt-cache search rocdxg` returned  Downloaded the official
  nothing                             `rocdxg-roct` release package
                                      directly

  `rocminfo` reported `hsaKmtOpenKFD` Resolved after installing ROCDXG
  symbol failure                      

  `hipconfig` reported missing `llc`  Did not block GPU detection; left
                                      for later investigation if required
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# Current Verification Status

At the end of the setup:

-   [x] ROCm 10.0 installed
-   [x] ROCm configured under `/opt/rocm/core-10.0`
-   [x] HIP available
-   [x] AMD HIP platform detected
-   [x] ROCDXG installed
-   [x] `librocdxg.so` present
-   [x] `rocminfo` detects the RX 9070 XT
-   [x] RX 9070 XT identified as `gfx1201`
-   [x] `rocm_agent_enumerator` reports `gfx1201`
-   [x] No `amdgpu-install` driver stack used
-   [x] Windows AMD driver left in place
-   [ ] ComfyUI installed
-   [ ] ComfyUI's Python environment configured
-   [ ] PyTorch GPU execution tested
-   [ ] ComfyUI GPU inference tested

The ROCm foundation is therefore complete. The remaining work is
application-level setup and validation rather than GPU/ROCm
installation.
