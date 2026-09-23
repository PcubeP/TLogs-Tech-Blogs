# ComfyUI + Qwen Image 2.1 + ROCm 10 Setup on RX 9070 XT

**Machine:** Windows 11 25H2 + WSL2 Ubuntu 26.04 LTS  
**GPU:** AMD Radeon RX 9070 XT 16 GB (RDNA4, `gfx1201`)  
**CPU:** Ryzen 9 9800X3D  
**RAM:** 64 GB  
**Primary goal:** Run ComfyUI natively inside WSL2 using the existing system-level ROCm installation, while keeping large ComfyUI models on the Windows `E:` drive. Initial model test: **Qwen Image 2.1 text-to-image**.

> This document records the setup that was actually performed, including the decisions, exact versions, paths, commands, failures, fixes, and the final known-good combination. It is intentionally detailed so the environment can be rebuilt without guessing.

---

## 1. Final architecture

The final working stack is:

```text
Windows 11 25H2
        │
        │ AMD Windows GPU driver
        ▼
WSL2
Ubuntu 26.04 LTS (resolute)
        │
        │ /dev/dxg
        ▼
ROCDXG
        │
        ▼
ROCm 10.0 userspace
        │
        ├── HIP
        ├── ROCr / HSA components
        └── gfx1201 support
        │
        ▼
PyTorch 2.13.0 + ROCm 10.0.0
        │
        ▼
ComfyUI Python virtual environment
        │
        ├── Qwen Image 2.1 INT8 ConvRot diffusion model
        ├── Qwen3-VL 8B INT8 ConvRot text encoder
        └── Qwen Image 2.1 BF16 VAE
        │
        ▼
ComfyUI
        │
        ▼
Qwen Image 2.1 text-to-image
        │
        ▼
~/ComfyUI/output/
```

### Important design decision

ROCm was installed **system-wide in WSL**, while ComfyUI itself lives in its own Python virtual environment.

This was deliberate.

The goal was:

- ROCm should not have to be reinstalled if ComfyUI is deleted/recreated.
- ComfyUI's Python packages should remain isolated.
- Large model files should not consume the Linux filesystem.
- Model storage should remain on `E:\Coding\AI\ComfyUI`.
- The Windows AMD driver remains the Windows GPU driver; `amdgpu-install` was intentionally avoided.

---

# 2. Hardware and OS baseline

## Windows

- Windows 11 25H2
- Windows build observed during setup: `10.0.26200.9445`
- AMD Windows GPU driver for RX 9070 XT:
  - `32.0.31041.1004`
- AMD Adrenalin version reported on the machine:
  - `26.8.1`

The machine also exposes an integrated/secondary AMD GPU:

```text
AMD Radeon(TM) Graphics
```

but the ComfyUI workload was targeted at the discrete:

```text
AMD Radeon RX 9070 XT
```

## WSL

Observed WSL components:

```text
WSL:    2.7.12.0
Kernel: 6.18.33.2-2
WSLg:   1.0.73.2
```

Inside Ubuntu:

```text
Linux 6.18.33.2-microsoft-standard-WSL2
```

WSL GPU integration exposes:

```text
/dev/dxg
```

`/dev/dri` was not present. This was not treated as a blocker because the working ROCm path uses the WSL DirectX GPU bridge through `dxg`/ROCDXG.

---

# 3. Ubuntu

Distribution:

```text
Ubuntu 26.04 LTS
Codename: resolute
```

The Ubuntu WSL distribution is installed on the `E:` drive:

```text
E:\WSL\Ubuntu\
```

with the WSL virtual disk:

```text
E:\WSL\Ubuntu\ext4.vhdx
```

Linux username:

```text
pcube
```

Home directory:

```text
/home/pcube
```

---

# 4. ROCm strategy

## What we deliberately did NOT do

We initially investigated AMD's `amdgpu-install` route.

It was rejected.

`amdgpu-install` attempted to pull a very large driver/package stack and at one point indicated roughly **17 GB** of additional space.

That was not what we wanted.

The objective was:

> Use the existing Windows AMD driver + WSL GPU bridge, and install the ROCm userspace required for AI workloads without reinstalling the Windows/AMD driver stack.

Therefore:

- `amdgpu-install` was removed/purged.
- ROCm was installed directly from AMD's ROCm Radeon repository.
- No second AMD Linux driver stack was intentionally installed.

---

# 5. ROCm repository

The AMD ROCm Radeon repository was configured for Ubuntu `resolute`:

```text
https://repo.radeon.com/rocmradeon/apt/26.14
```

Repository entry used:

```text
deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocmradeon/apt/26.14 resolute main
```

The repository signing key was:

```text
9386B48A1A693C5C
```

There was initially a repository key problem:

```text
NO_PUBKEY 9386B48A1A693C5C
```

The ROCm GPG key was restored and `apt update` then completed successfully.

---

# 6. ROCm package selection

Ubuntu's generic repository exposed a `rocm` package candidate around:

```text
7.1.0-0ubuntu6
```

We intentionally did **not** use that generic package.

The AMD ROCm Radeon repository exposed the newer ROCm 10 path appropriate for this setup.

The package selected for the RX 9070 XT was:

```text
amdrocm10.0-gfx1201
```

The candidate observed was:

```text
10.0.0~pre4-32424980240
```

Installation size observed:

```text
~980 MB download
~6.2 GB additional disk usage
```

The resulting ROCm installation lives under:

```text
/opt/rocm/core-10.0
```

Alternatives were configured so tools such as:

```text
hipcc
rocminfo
amd-smi
```

resolve through the ROCm 10 installation.

---

# 7. ROCm / WSL issue: missing librocdxg

After installing the ROCm userspace, the first `rocminfo` attempt failed.

The important problem was that the WSL ROCm stack could not find:

```text
librocdxg.so
```

and there was also an HSA/KFD-related error involving:

```text
hsaKmtOpenKFD
```

This was not fixed by reinstalling the AMD driver.

Instead, the missing WSL DirectX ROCm bridge library was installed from AMD's official `librocdxg` GitHub release.

Downloaded package:

```text
rocdxg-roct_1.2.2_amd64.deb
```

Source:

```text
https://github.com/ROCm/librocdxg/releases/download/v1.2.2/rocdxg-roct_1.2.2_amd64.deb
```

Installed with:

```bash
sudo apt install ./rocdxg-roct_1.2.2_amd64.deb
```

Verification showed:

```text
/opt/rocm/core-10.0/lib/librocdxg.so
    -> librocdxg.so.1
```

and the actual library:

```text
librocdxg.so.1.2.2
```

was present.

---

# 8. ROCm verification

After installing `librocdxg`, `rocminfo` successfully detected the GPU.

Important output:

```text
Name: gfx1201
Marketing Name: AMD Radeon RX 9070 XT
Vendor Name: AMD
```

The GPU target is therefore:

```text
gfx1201
```

The corresponding compiler/architecture target was also reported as:

```text
amdgcn-amd-amdhsa--gfx1201
```

ROCm's agent enumerator returned:

```text
gfx1201
```

using:

```bash
/opt/rocm/core-10.0/bin/rocm_agent_enumerator
```

This established the critical hardware chain:

```text
RX 9070 XT
   ↓
gfx1201
   ↓
WSL /dev/dxg
   ↓
ROCDXG
   ↓
ROCm 10
   ↓
HIP
```

---

# 9. HIP verification

`hipconfig --full` reported:

```text
HIP version: 7.15.26333-0000000
HIP_PATH: /opt/rocm/core-10.0
ROCM_PATH: /opt/rocm/core-10.0
HIP compiler: clang
Platform: amd
Runtime: rocclr
AMD clang: 23.0.0git
```

There were some non-blocking messages involving:

```text
llc: not found
```

and a shell syntax warning.

These did not prevent GPU detection or the eventual ComfyUI workload from running, so they were not treated as blockers.

---

# 10. ComfyUI installation location

ComfyUI itself was installed on the **native WSL Linux filesystem**:

```text
/home/pcube/ComfyUI
```

or:

```bash
~/ComfyUI
```

This was intentional.

The Python environment, application source, and runtime files remain on the Linux filesystem, while the large model files live on `E:`.

This gives the following separation:

```text
Linux / WSL
└── /home/pcube/ComfyUI
    ├── source code
    ├── venv
    ├── workflow
    ├── configuration
    └── output

Windows E:
└── E:\Coding\AI\ComfyUI
    ├── diffusion_models
    ├── text_encoders
    └── vae
```

---

# 11. ComfyUI repository

The actual ComfyUI repository used was:

```text
https://github.com/comfyanonymous/ComfyUI.git
```

The repository was cloned once.

It was **not** repeatedly recloned when troubleshooting models or workflows.

---

# 12. Python environment

A dedicated Python virtual environment was created:

```text
/home/pcube/ComfyUI/venv
```

Activation:

```bash
cd ~/ComfyUI
source venv/bin/activate
```

Python version:

```text
Python 3.14.4
```

ComfyUI itself was:

```text
ComfyUI 0.37.0
```

### Python 3.14 note

The current ComfyUI README says Python 3.14 works, although it notes that users may encounter issues with torch compile nodes; Python 3.13 is described as very well supported.

In our actual setup, Python 3.14 was retained because the environment was already working after resolving the missing development headers. citeturn0search3

---

# 13. Python 3.14 development headers — important fix

The first Qwen Image 2.1 text-encoding attempt failed inside Triton's AMD/HIP initialization.

The hidden underlying error was:

```text
fatal error: Python.h: No such file or directory
```

The failing compiler command was using:

```text
/usr/bin/gcc
```

and:

```text
-I/usr/include/python3.14
```

The system package:

```text
python3.14-dev
```

was missing.

We verified it explicitly:

```bash
dpkg -s python3.14-dev >/dev/null 2>&1 && echo "python3.14-dev: INSTALLED" || echo "python3.14-dev: MISSING"
```

Output before the fix:

```text
python3.14-dev: MISSING
```

The package was then installed.

Verification:

```bash
ls -l /usr/include/python3.14/Python.h
```

Result:

```text
-rw-r--r-- 1 root root 4079 Aug 20 10:41 /usr/include/python3.14/Python.h
```

We then reran the Triton HIP initialization test.

Before installing `python3.14-dev`:

```text
Python.h: No such file or directory
```

After installing it:

```text
no output / no error
```

That confirmed `HIPUtils()` could successfully initialize.

---

# 14. PyTorch selection

This was one of the most important compatibility choices.

ComfyUI's normal dependency installation initially attempted to put an AMD-incompatible/CUDA-oriented torch build into the environment.

That was corrected manually.

Final PyTorch:

```text
torch 2.13.0+rocm10.0.0
```

Final torchvision:

```text
torchvision 0.28.0+rocm10.0.0
```

PyTorch was installed from AMD's ROCm package index:

```text
https://stable.repo.amd.com/rocm/whl-next/
```

The selected torch package was:

```text
torch[device-gfx1201]==2.13.0+rocm10.0.0
```

The `device-gfx1201` extra is important because the GPU is the RX 9070 XT / `gfx1201`.

Final verified GPU information from PyTorch:

```text
GPU: AMD Radeon RX 9070 XT
VRAM: ~15.81 GB usable
HIP: 7.15.26333
```

ComfyUI later reported approximately:

```text
16.97 GB total
```

The difference is due to how the respective layers report/measure available memory.

We also ran:

```bash
python -m pip check
```

and there were no broken Python package requirements.

---

# 15. Final Python/ROCm combination

The known-good combination is therefore:

| Component | Selected version/config |
|---|---|
| OS | Windows 11 25H2 |
| WSL | 2.7.12.0 |
| WSL kernel | 6.18.33.2-2 |
| Ubuntu | 26.04 LTS / resolute |
| GPU | RX 9070 XT 16 GB |
| GPU architecture | `gfx1201` |
| Windows AMD driver | 32.0.31041.1004 |
| ROCm | 10.0 |
| ROCm package | `amdrocm10.0-gfx1201` |
| ROCDXG | `1.2.2` |
| HIP | 7.15.26333 |
| Python | 3.14.4 |
| Python dev headers | `python3.14-dev` |
| PyTorch | `2.13.0+rocm10.0.0` |
| torchvision | `0.28.0+rocm10.0.0` |
| ComfyUI | 0.37.0 |
| ComfyUI compiler workaround | `--disable-comfy-compiler` |

---

# 16. Model storage strategy

Large model files were deliberately moved to the Windows `E:` drive.

Windows location:

```text
E:\Coding\AI\ComfyUI
```

WSL-visible location:

```text
/mnt/e/Coding/AI/ComfyUI
```

The three directories used are:

```text
/mnt/e/Coding/AI/ComfyUI/diffusion_models
/mnt/e/Coding/AI/ComfyUI/text_encoders
/mnt/e/Coding/AI/ComfyUI/vae
```

This avoids putting many gigabytes of model files inside the WSL virtual disk.

---

# 17. ComfyUI extra model configuration

File:

```text
~/ComfyUI/extra_model_paths.yaml
```

Contents:

```yaml
comfyui:
    base_path: /mnt/e/Coding/AI/ComfyUI
    diffusion_models: diffusion_models/
    text_encoders: text_encoders/
    vae: vae/
```

This tells ComfyUI that the external model base path is:

```text
/mnt/e/Coding/AI/ComfyUI
```

and that the model subdirectories are:

```text
diffusion_models/
text_encoders/
vae/
```

ComfyUI officially supports `extra_model_paths.yaml` for defining external/central model locations. citeturn0search2turn0search3

---

# 18. Why this model layout was chosen

Instead of:

```text
~/ComfyUI/models/...
```

the large models are stored externally:

```text
E:\Coding\AI\ComfyUI
```

This gives a clean separation:

### Application

```text
/home/pcube/ComfyUI
```

### Models

```text
E:\Coding\AI\ComfyUI
```

### ROCm

```text
/opt/rocm/core-10.0
```

Therefore the three major layers are independently replaceable:

```text
ROCm
  independent from
ComfyUI
  independent from
Model files
```

This was an explicit design goal.

---

# 19. Qwen Image 2.1 workflow

The official Comfy-Org workflow template used was:

```text
image_qwen_image_2_1_t2i.json
```

Repository:

```text
https://github.com/Comfy-Org/workflow_templates
```

The workflow was downloaded directly into:

```text
~/ComfyUI/qwen_image_2.1_t2i.json
```

using:

```bash
wget -O ~/ComfyUI/qwen_image_2.1_t2i.json "https://raw.githubusercontent.com/Comfy-Org/workflow_templates/main/templates/image_qwen_image_2_1_t2i.json"
```

The downloaded file was approximately:

```text
37.7 KB
```

The official current workflow specifies the same three model files used here:

```text
qwen_image_2.1_int8_convrot.safetensors
qwen3vl_8b_int8_convrot.safetensors
qwen_image_2.1_vae_bf16.safetensors
```

and the official template's default T2I setup uses:

```text
1:1
1 megapixel
25 steps
euler
simple
cfg 1
```

as reflected in the workflow definition. citeturn0search1

---

# 20. Qwen Image 2.1 models

## Diffusion model

File:

```text
qwen_image_2.1_int8_convrot.safetensors
```

Location:

```text
/mnt/e/Coding/AI/ComfyUI/diffusion_models/
```

Observed size:

```text
~6.8 GB
```

---

## Text encoder

Correct file:

```text
qwen3vl_8b_int8_convrot.safetensors
```

Location:

```text
/mnt/e/Coding/AI/ComfyUI/text_encoders/
```

Observed size:

```text
~8.8 GB
```

### Important mistake during setup

An earlier text encoder was downloaded:

```text
qwen3vl_8b_w4a8.safetensors
```

Observed size:

```text
~5.9 GB
```

This was **not** the text encoder expected by the current official Qwen Image 2.1 workflow.

It was therefore left alone rather than used for this workflow.

The correct encoder was then downloaded:

```text
qwen3vl_8b_int8_convrot.safetensors
```

After downloading it, the workflow was refreshed/reloaded and the **Missing Models** warning disappeared.

The current official workflow confirms that the expected text encoder is the INT8 ConvRot file. citeturn0search1

---

## VAE

File:

```text
qwen_image_2.1_vae_bf16.safetensors
```

Location:

```text
/mnt/e/Coding/AI/ComfyUI/vae/
```

Observed size:

```text
~645 MB
```

---

# 21. Final Qwen model layout

```text
E:\Coding\AI\ComfyUI\
│
├── diffusion_models\
│   └── qwen_image_2.1_int8_convrot.safetensors
│
├── text_encoders\
│   ├── qwen3vl_8b_w4a8.safetensors              ← older/wrong-for-this-workflow file
│   └── qwen3vl_8b_int8_convrot.safetensors      ← correct
│
└── vae\
    └── qwen_image_2.1_vae_bf16.safetensors
```

The older W4A8 encoder is not part of the known-good Qwen Image 2.1 T2I pipeline.

---

# 22. Initial ComfyUI launch

Before the launcher script was created, ComfyUI was started manually with:

```bash
cd ~/ComfyUI
source venv/bin/activate
python main.py --extra-model-paths-config extra_model_paths.yaml
```

The normal URL is:

```text
http://127.0.0.1:8188
```

---

# 23. First Qwen text-to-image failure

The Qwen Image 2.1 workflow loaded, but the first execution failed at:

```text
TextEncodeQwenImage21
```

The exception was:

```text
subprocess.CalledProcessError
```

The important part of the stack trace showed Triton attempting to compile a small HIP/Python native extension.

The compiler command used:

```text
/usr/bin/gcc
```

and included:

```text
-I/usr/include/python3.14
```

The actual compiler error was:

```text
fatal error: Python.h: No such file or directory
```

This initially appeared inside a much larger ComfyUI/Triton stack trace.

---

# 24. How the hidden GCC error was isolated

A targeted test was run to force the underlying compiler output to be visible:

```bash
cd ~/ComfyUI && source venv/bin/activate && python -c "import subprocess; o=subprocess.check_call; subprocess.check_call=lambda cmd,**kw:o(cmd,**{k:v for k,v in kw.items() if k!='stdout'}); from triton.backends.amd.driver import HIPUtils; HIPUtils()"
```

Before the fix, it produced:

```text
/tmp/.../hip_utils.c:5:10: fatal error: Python.h: No such file or directory
```

followed by:

```text
subprocess.CalledProcessError
```

This established that the GPU/ROCm path itself was not the primary failure; Triton simply could not compile its helper extension because the Python development headers were missing.

---

# 25. Fix for first Qwen failure

Install:

```text
python3.14-dev
```

Then verify:

```bash
ls -l /usr/include/python3.14/Python.h
```

Expected result:

```text
/usr/include/python3.14/Python.h
```

Then rerun the HIPUtils test:

```bash
cd ~/ComfyUI && source venv/bin/activate && python -c "import subprocess; o=subprocess.check_call; subprocess.check_call=lambda cmd,**kw:o(cmd,**{k:v for k,v in kw.items() if k!='stdout'}); from triton.backends.amd.driver import HIPUtils; HIPUtils()"
```

Successful result:

```text
no output
```

That meant the native HIP helper could now compile.

---

# 26. Second Qwen failure: aimdo memory compile

After fixing Python.h, Qwen Image 2.1 progressed further.

The models loaded correctly.

Then sampling reached:

```text
0/25
```

and failed in the KSampler.

The error was:

```text
RuntimeError: aimdo memory compile error: could not start recording
```

The relevant execution path was approximately:

```text
KSampler
  ↓
Qwen Image 2.1 model
  ↓
comfy.model_prefetch.malloc_graph_begin()
  ↓
comfy_aimdo.malloc_graph.record()
  ↓
aimdo memory compile error
```

At this point the problem was no longer:

- missing model
- missing Python headers
- GPU detection
- ROCm detection
- PyTorch GPU support

The model and GPU stack had successfully loaded.

The failure was specifically in the ComfyUI/aimdo memory-graph/compile path.

---

# 27. Evidence that the core stack was working

The failing run reported:

```text
AMD Radeon RX 9070 XT
```

with:

```text
gfx1201
```

and:

```text
ROCm (7, 15)
```

The device was:

```text
cuda:0 AMD Radeon RX 9070 XT : native
```

The logs also showed:

```text
comfy_kitchen hip backend available
```

and:

```text
aimdo initialized for GPU
```

The installed ComfyUI components included approximately:

```text
comfy-aimdo 0.5.5
comfy-kitchen 0.2.35
```

The Qwen components successfully loaded:

```text
Qwen Image 2.1 VAE
Qwen3-VL 8B text encoder
Qwen Image 2.1 diffusion model
```

The diffusion model staged approximately:

```text
6920 MB
```

So the failure happened after the important pieces had already loaded.

---

# 28. Final workaround

The workaround was to start ComfyUI with:

```text
--disable-comfy-compiler
```

Final working launch command:

```bash
cd ~/ComfyUI && source venv/bin/activate && python main.py --extra-model-paths-config extra_model_paths.yaml --disable-comfy-compiler
```

With that option, the Qwen Image 2.1 text-to-image workflow successfully rendered an image.

This is currently a **workaround**, not a claim that the underlying aimdo/compile issue has been permanently fixed.

The important distinction is:

```text
Normal compiler/aimdo path
        ↓
aimdo memory compile error

--disable-comfy-compiler
        ↓
bypass problematic path
        ↓
Qwen Image 2.1 renders successfully
```

---

# 29. Known-good ComfyUI startup

The currently proven startup command is:

```bash
cd ~/ComfyUI && source venv/bin/activate && python main.py --extra-model-paths-config extra_model_paths.yaml --disable-comfy-compiler
```

ComfyUI listens on:

```text
http://127.0.0.1:8188
```

To stop it:

```text
Ctrl+C
```

To leave the Python virtual environment after stopping:

```bash
deactivate
```

To activate it manually again:

```bash
cd ~/ComfyUI
source venv/bin/activate
```

---

# 30. Startup launcher script

Because the known-good startup command is long, a launcher script was created:

```text
~/ComfyUI/start-comfy.sh
```

Contents:

```bash
#!/bin/bash
cd ~/ComfyUI
source venv/bin/activate
python main.py --extra-model-paths-config extra_model_paths.yaml --disable-comfy-compiler
```

It was made executable with:

```bash
chmod +x ~/ComfyUI/start-comfy.sh
```

The launcher was verified with:

```bash
cat ~/ComfyUI/start-comfy.sh
```

and then actually tested.

It successfully started ComfyUI.

The normal way to start the working setup is now simply:

```bash
~/ComfyUI/start-comfy.sh
```

This script does not reinstall anything. It is only a convenience wrapper around the known-good startup commands.

---

# 31. Output location

ComfyUI saves generated images under:

```text
~/ComfyUI/output/
```

The successful Qwen Image 2.1 test produced:

```text
Qwen_image_2.1_00001.png
Qwen_image_2.1_00002.png
```

Observed size of each:

```text
~2.0 MB
```

There was also:

```text
_output_images_will_be_put_here
```

with zero bytes.

The Linux output directory was verified with:

```bash
ls -lah ~/ComfyUI/output
```

---

# 32. Initial Qwen T2I settings

The official T2I workflow loaded with settings approximately:

```text
Aspect ratio: 1:1
Resolution budget: 1.0 megapixel
Multiple: 8
Steps: 25
CFG: 1.0
Sampler: euler
Scheduler: simple
```

The workflow used:

```text
qwen_image_2.1_int8_convrot.safetensors
qwen3vl_8b_int8_convrot.safetensors
qwen_image_2.1_vae_bf16.safetensors
```

The workflow was used initially without changing resolution or quality parameters because the first objective was simply:

> Prove that Qwen Image 2.1 text-to-image works end-to-end on this RX 9070 XT / ROCm 10 environment.

That objective was achieved.

The official current workflow similarly defines the Qwen T2I path with the same model files and default 1 MP / 25-step / Euler / simple / CFG 1 configuration. citeturn0search1

---

# 33. Things that are intentionally NOT part of this setup

The following were deliberately avoided or are not required for the known-good T2I configuration:

### No `amdgpu-install`

We do not use:

```text
amdgpu-install
```

for this setup.

### No separate CUDA stack

This is an AMD ROCm environment.

### No Windows-native ComfyUI Python environment

ComfyUI is running inside WSL.

### No model duplication inside WSL

The large Qwen model files live on:

```text
E:\Coding\AI\ComfyUI
```

### No recloning for every model/workflow

The ComfyUI repository is already installed.

### No generic Ubuntu `rocm` package

We selected the AMD Radeon ROCm 10/gfx1201 package path instead.

---

# 34. Troubleshooting history at a glance

| Problem | Symptom | Root cause | Resolution |
|---|---|---|---|
| `amdgpu-install` looked huge | ~17 GB requested | Full AMD package/driver stack | Removed/purged; avoided |
| ROCm initially could not see GPU | `librocdxg.so` / HSA errors | Missing WSL DirectX ROCm bridge | Installed `rocdxg-roct_1.2.2_amd64.deb` |
| First text encoder run failed | `Python.h` missing | `python3.14-dev` absent | Installed `python3.14-dev` |
| Qwen model warning | Missing `qwen3vl_8b_int8_convrot.safetensors` | Wrong W4A8 encoder had been downloaded first | Downloaded correct INT8 ConvRot encoder |
| KSampler failed | `aimdo memory compile error: could not start recording` | ComfyUI aimdo/compiler path | Started with `--disable-comfy-compiler` |
| Startup command was long | Easy to forget flags/paths | Convenience issue | Created `start-comfy.sh` |

---

# 35. Current directory map

## WSL

```text
/home/pcube/
└── ComfyUI/
    ├── venv/
    ├── output/
    │   ├── Qwen_image_2.1_00001.png
    │   └── Qwen_image_2.1_00002.png
    ├── extra_model_paths.yaml
    ├── qwen_image_2.1_t2i.json
    ├── start-comfy.sh
    └── ...ComfyUI source...
```

## Windows E drive

```text
E:\Coding\AI\ComfyUI\
├── diffusion_models\
│   └── qwen_image_2.1_int8_convrot.safetensors
│
├── text_encoders\
│   ├── qwen3vl_8b_w4a8.safetensors
│   └── qwen3vl_8b_int8_convrot.safetensors
│
└── vae\
    └── qwen_image_2.1_vae_bf16.safetensors
```

## ROCm

```text
/opt/rocm/core-10.0/
```

---

# 36. Rebuild checklist

If ComfyUI needs to be rebuilt later, the important conceptual order is:

```text
1. Windows AMD driver
        ↓
2. WSL2 + Ubuntu
        ↓
3. AMD ROCm Radeon repository
        ↓
4. amdrocm10.0-gfx1201
        ↓
5. librocdxg / ROCDXG
        ↓
6. Verify rocminfo → gfx1201 / RX 9070 XT
        ↓
7. Create ComfyUI Python venv
        ↓
8. Ensure Python development headers exist
        ↓
9. Install AMD ROCm PyTorch build
        ↓
10. Install/run ComfyUI
        ↓
11. Configure extra_model_paths.yaml
        ↓
12. Put Qwen models on E:
        ↓
13. Load official Qwen Image 2.1 T2I workflow
        ↓
14. Start with --disable-comfy-compiler
        ↓
15. Test text-to-image
```

The critical rule is:

> Verify the GPU/ROCm layer before debugging ComfyUI.

---

# 37. Verification commands

## Check ROCm GPU

```bash
rocminfo
```

Look for:

```text
AMD Radeon RX 9070 XT
gfx1201
```

## Check ROCm architecture

```bash
/opt/rocm/core-10.0/bin/rocm_agent_enumerator
```

Expected:

```text
gfx1201
```

## Check HIP

```bash
hipconfig --full
```

## Check Python version

```bash
python --version
```

Expected:

```text
Python 3.14.4
```

## Check Python development headers

```bash
ls -l /usr/include/python3.14/Python.h
```

## Check PyTorch

Inside the ComfyUI venv:

```bash
python -c "import torch; print(torch.__version__); print(torch.version.hip); print(torch.cuda.get_device_name(0)); print(torch.cuda.get_device_capability(0))"
```

Expected core values:

```text
2.13.0+rocm10.0.0
7.15...
AMD Radeon RX 9070 XT
```

## Check package consistency

```bash
python -m pip check
```

Expected:

```text
No broken requirements found.
```

## Check ComfyUI launcher

```bash
cat ~/ComfyUI/start-comfy.sh
```

Expected:

```bash
#!/bin/bash
cd ~/ComfyUI
source venv/bin/activate
python main.py --extra-model-paths-config extra_model_paths.yaml --disable-comfy-compiler
```

---

# 38. Current known-good state

At the end of this setup:

```text
ROCm 10                         WORKING
ROCDXG                          WORKING
RX 9070 XT detection           WORKING
gfx1201                         WORKING
HIP                             WORKING
PyTorch ROCm                    WORKING
ComfyUI                         WORKING
External model paths            WORKING
Qwen Image 2.1 models          WORKING
Qwen text encoder               WORKING
Qwen VAE                        WORKING
Qwen T2I workflow               WORKING
Triton HIPUtils                 WORKING
Qwen text encoding              WORKING
KSampler                        WORKING with compiler disabled
Image generation                WORKING
PNG output                      WORKING
Launcher script                 WORKING
```

The only known workaround still in place is:

```text
--disable-comfy-compiler
```

because the default ComfyUI compiler/aimdo path produced:

```text
aimdo memory compile error: could not start recording
```

The underlying ROCm/GPU/model stack itself is functional.

---

# 39. Official references used during setup

### ComfyUI

Official repository:

https://github.com/comfyanonymous/ComfyUI

### ComfyUI external model paths

Official example:

https://github.com/Comfy-Org/ComfyUI/blob/master/extra_model_paths.yaml.example

### Official Qwen Image 2.1 T2I workflow

https://github.com/Comfy-Org/workflow_templates/blob/main/templates/image_qwen_image_2_1_t2i.json

### Qwen Image 2.1 model repository referenced by the official workflow

https://huggingface.co/Comfy-Org/Qwen-Image-2.1

---

# 40. Bottom line

The setup is intentionally split into three independent layers:

```text
┌─────────────────────────────────────────────┐
│ Windows AMD Driver                          │
│ RX 9070 XT                                  │
└─────────────────────┬───────────────────────┘
                      │
┌─────────────────────▼───────────────────────┐
│ WSL + ROCm 10 + ROCDXG                      │
│ gfx1201 / HIP 7.15                         │
└─────────────────────┬───────────────────────┘
                      │
┌─────────────────────▼───────────────────────┐
│ ComfyUI venv                                │
│ Python 3.14.4                               │
│ PyTorch 2.13.0+rocm10.0.0                  │
│ ComfyUI 0.37.0                              │
└─────────────────────┬───────────────────────┘
                      │
┌─────────────────────▼───────────────────────┐
│ Qwen Image 2.1                              │
│ INT8 ConvRot diffusion model                │
│ INT8 ConvRot Qwen3-VL 8B text encoder       │
│ BF16 VAE                                    │
└─────────────────────┬───────────────────────┘
                      │
                      ▼
             Successful T2I render
```

This is the configuration that has actually been tested on the RX 9070 XT and produced a successful Qwen Image 2.1 text-to-image render.

**Do not casually change the ROCm/PyTorch versions just because a generic ComfyUI guide recommends another combination.** This particular combination has been verified on this hardware.

