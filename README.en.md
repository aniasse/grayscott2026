# Gray Scott School 2026 — Local Environment

*🇫🇷 Version française : [README.md](README.md)*

How to reproduce the HPC course environment *"Performance With Stencil"*
(Gray Scott School 2026) **on your own machine, without a container**.

The course relies on the **PHOENIX** libraries (LAPP), **HDF5** and **TBB**.
Good news: everything is available through [**pixi**](https://pixi.sh) (a conda-based
environment manager), because the PHOENIX packages are published on a public conda
channel (`prefix.dev/phoenix`). **No apptainer and no in2p3 access required.**

> ✅ Platforms: **Linux x86-64** natively. **Windows** via **WSL2** (see below).
> macOS: not supported (packages are `linux-64`) → use a Linux VM/server.

---

## TL;DR (Linux)

```bash
# 1. Install pixi
curl -fsSL https://pixi.sh/install.sh | bash && source ~/.bashrc

# 2. Get the course code
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil          # the directory containing pixi.toml

# 3. Install the environment (gcc, hdf5, tbb, phoenix…)
pixi install

# 4. Build
pixi run bash -c 'mkdir -p build && cd build && cmake .. $(phoenixcmake-config --cmake) -G Ninja && ninja -j$(nproc)'

# 5. Run an example
pixi shell
cd build
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5
mkdir -p pics && ./8-ImagePlotting/gray_scott_image -i output.h5 -o pics/
```

> 📌 The working directory is **the one containing `pixi.toml`** (depending on the
> repo version: root, `Examples/`, or `day-2/`). Adjust the `cd` accordingly.

---

## Prerequisites

- **git**
- Internet access (packages are downloaded from `prefix.dev`)
- ~2–3 GB of disk space (gcc toolchain + dependencies)
- No `root` needed: pixi installs into `~/.pixi`

---

## Linux (detailed)

### 1. Install pixi
```bash
curl -fsSL https://pixi.sh/install.sh | bash
source ~/.bashrc          # or restart the terminal → the `pixi` command is available
pixi --version            # check (>= 0.70)
```

### 2. Get the course
```bash
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil
ls pixi.toml              # you must be in the directory that contains this file
```

### 3. Install the environment
```bash
pixi install
```
This downloads and installs (versions pinned by `pixi.lock`):
`gcc/g++ 15`, `cmake 4`, `ninja`, `make`, **`hdf5 1.14`**, **`tbb-devel 2022`**,
and the **`phoenixcmake / phoenixcore / phoenixpng / phoenixoptionparser /
phoenixprogress / phoenixdatastream / phoenixtypestream`** libraries.

Check:
```bash
pixi list | grep -E 'hdf5|tbb|phoenix|gcc|cmake'
```

### 4. Build
```bash
pixi run bash -c '
  mkdir -p build && cd build
  cmake .. $(phoenixcmake-config --cmake) -G Ninja
  ninja -j$(nproc)
'
```

### 5. Run & verify
```bash
pixi shell                # enter the environment
cd build

# Ex 7 — HDF5 data-format test
ctest -R TestGrayScottDataFormat --output-on-failure

# Ex 9 — simulation -> HDF5 file
./9-Simulation/autovec_gray_scott_O3 -n 10 -e 200 -r 512 -c 512 -o output.h5

# Ex 8 — HDF5 -> PNG images
mkdir -p pics
./8-ImagePlotting/gray_scott_image -i output.h5 -o pics/
ls pics/                  # a series of .png (Turing patterns)
```

Rebuild after a change:
```bash
pixi run bash -c 'cd build && ninja -j$(nproc)'
```

---

## Windows (via WSL2)

The packages are `linux-64`, so on Windows you use **WSL2** (Linux integrated into
Windows). It is fast and is the recommended method.

### 1. Install WSL2 (once)
In **PowerShell as administrator**:
```powershell
wsl --install -d Ubuntu
```
Restart if asked, then open **Ubuntu** from the Start menu and create your Linux user.

> If `wsl` already exists: `wsl --update` then `wsl --install -d Ubuntu`.

### 2. In the Ubuntu (WSL) terminal, follow the Linux steps
```bash
sudo apt update && sudo apt install -y git curl       # base tools
curl -fsSL https://pixi.sh/install.sh | bash && source ~/.bashrc
git clone https://gitlab.in2p3.fr/CTA-LAPP/COURS/PerformanceWithStencil.git
cd PerformanceWithStencil
pixi install
pixi run bash -c 'mkdir -p build && cd build && cmake .. $(phoenixcmake-config --cmake) -G Ninja && ninja -j$(nproc)'
```
Then run the examples as in the **Linux** section above.

### WSL tips
- Work inside the **Linux filesystem** (`~/` in WSL), **not** `/mnt/c/...` → much
  faster compilation.
- To view PNG images: from WSL, `explorer.exe .` opens the current folder in Windows
  Explorer.
- VS Code: install the **WSL** extension and open the folder with `code .`.

---

## View / convert images

`.png` files open directly. To make an animated GIF (if ImageMagick is installed):
```bash
convert -delay 20 pics/*.png animation.gif
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `pixi: command not found` | `source ~/.bashrc` or reopen the terminal |
| `cmake`/`phoenixcmake-config` not found | run commands via `pixi run …` or `pixi shell` (not the system cmake) |
| resolution / download error | check access to `prefix.dev`; re-run `pixi install` |
| `pixi install` fails on native Windows | use **WSL2** (packages are `linux-64`) |
| `*_gray_scott_O3` binary missing | make sure `ninja` finished; `ls build/9-Simulation/` |
| `.so` not found at runtime | run from `pixi shell` (the env sets `LD_LIBRARY_PATH`) |

---

## Why pixi (instead of the container)?

The course also ships an **apptainer** image. Here we use **pixi** because:
- it is **cross-platform** and **root-free**,
- the environment is **fully reproducible** (`pixi.lock` pins versions),
- no need to download a large container image or use a private registry.

Both approaches give the **same environment** (same versions of gcc, HDF5, TBB,
PHOENIX).

---

*Official course: <https://cta-lapp.pages.in2p3.fr/COURS/PerformanceWithStencil/>*
