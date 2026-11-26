# Running on Linux (Fedora) with Proton

VIctory is a Windows program, and on Linux it runs under Proton, whose
vkd3d-proton turns Direct3D 12 into Vulkan. **This has not been tried with the
title yet.** What is known (2026-11-26, Wine 11.21): `victory.exe` builds on Fedora
and starts under Wine. The loader's unit tests pass under Wine, and 148 of 150 of the
runtime services' do. One failure is Wine lacking `ws2_32!WSAConnectByList`, the
other an NTFS-only test. Nothing has drawn a frame under vkd3d-proton.

## 1. What the PC needs

- **The NVIDIA driver from RPM Fusion** (`akmod-nvidia`), not nouveau.
  vkd3d-proton needs shader model 6.6 and mesh shaders, which it gets from the
  proprietary driver's Vulkan.
- **umu-launcher**, to run Windows programs under Proton outside Steam. It is not
  in Fedora's own repositories: take it from its GitHub releases. With
  `PROTONPATH=GE-Proton` (the script's default) it downloads GE-Proton itself.
  For Steam's Proton instead, see step 4.
- **Your own complete dump of the title**, outside this checkout.

## 2. Build victory.exe on Fedora

```sh
sudo dnf install clang llvm                       # clang-cl and llvm-lib for C parts of dependencies
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain none
. ~/.cargo/env
cargo install --locked cargo-xwin                 # Windows SDK and MSVC CRT, fetched on first use
cd cool_fox
cargo xwin build --release --locked -p victory --target x86_64-pc-windows-msvc
```

`rust-toolchain.toml` makes rustup install the pinned toolchain. The build ends in
`target/x86_64-pc-windows-msvc/release/victory.exe`. On first use, cargo-xwin downloads
the Windows SDK and CRT files, which come under Microsoft's license.

For the diagnostics build (log lines, F3 panel, frame recording):
`cargo xwin build --locked --profile diag -p victory --features diagnostics --target x86_64-pc-windows-msvc`,
then point `VICTORY_EXE` at `target/x86_64-pc-windows-msvc/diag/victory.exe`.

## 3. Start it

```sh
VICTORY_GAME_DIR=~/Games/MyDump scripts/play-proton.sh
```

[`scripts/play-proton.sh`](../scripts/play-proton.sh) sets what `Play-VIctory.cmd
pareto-residency` sets: the Series S profile and the residency settings. It turns
Linux paths into Wine's `Z:` paths, and keeps the shader cache in
`~/.cache/victory/cache-pareto` and saves in `~/.local/share/victory/data`, so a new
Wine prefix loses neither. The first start translates shaders and stutters; the
cache makes later ones smoother.

Settings you set yourself win, for example
`VICTORY_LOG=info VICTORY_PANEL=1 VICTORY_GAME_DIR=... scripts/play-proton.sh`.
`PROTON_LOG=1` makes Proton write its own log file into your home folder.

## 4. Or through Steam

Add `victory.exe` as a non-Steam game, force a Proton version in its properties
(Proton Experimental or GE-Proton), and set its launch options, with your own paths
written as Wine sees them:

```
VICTORY_GAME_DIR='Z:\home\you\Games\MyDump' VICTORY_CACHE_DIR='Z:\home\you\.cache\victory\cache-pareto' %command% run 'Z:\home\you\Games\MyDump\game_scarlett_master.exe' --console series-s --execute
```

The settings in `scripts/play-proton.sh` go in front of `%command%` the same way.

## What may break first

- **D3D12 features through vkd3d-proton**: mesh shaders, pipeline libraries,
  ray tracing (off by default here). Look in the log for pipelines that could not be made.
- **Our own small HLSL passes**, compiled at run time with `D3DCompile`. Proton
  ships its own `d3dcompiler_47`, which may refuse one of them.
- **Speed of noticing the title's writes**: Windows' write watch is done by Wine
  with page protection and signals, which can cost more than on Windows.
- **A missing function**: `--unimplemented trap` (the default) stops at an
  unimplemented console import and names it. A missing Wine export shows in
  Proton's log.

Logs to keep when something fails: the terminal output, `PROTON_LOG=1`'s file, and
with the diagnostics build a run with `VICTORY_LOG=info`.
