[![Build Status](https://travis-ci.org/libretro/beetle-psx-libretro.svg?branch=master)](https://travis-ci.org/libretro/beetle-psx-libretro)
[![Build status](https://ci.appveyor.com/api/projects/status/qd1ew088woadbqhc/branch/master?svg=true)](https://ci.appveyor.com/project/bparker06/beetle-psx-libretro/branch/master)

# Beetle PSX libretro

Beetle PSX is a port/fork of Mednafen's PSX module to the libretro API. It can be compiled in C++98 mode, excluding the Vulkan renderer, which is written in C++11 for the time being. Beetle PSX currently runs on Linux, OSX and Windows.

Notable additions in this fork are:
* PBP and CHD file format support, developed by Zapeth;
* Software renderer internal resolution upscaling, implemented by simias;
* An OpenGL 3.3 renderer, developed by simias;
* A Vulkan renderer, developed by TinyTiger;
* PGXP perspective correct texturing and subpixel precision, developed by iCatButler;
* OpenBIOS, allowing the emulator to be used without a BIOS file;
* HD texture replacement caching overhaul (Vulkan renderer), see [HD_TEXTURE_CACHE.md](HD_TEXTURE_CACHE.md);
* Page-aligned texture dump/replacement + HD texture QoL (Vulkan renderer, experimental), see [PAGE_ALIGN.md](PAGE_ALIGN.md);

## HD texture replacement caching

This fork overhauls the Vulkan renderer's HD texture replacement pipeline so packs stay smooth on demanding content — particularly multi-palette animated sprites like Alucard in *Castlevania: Symphony of the Night*. It adds a three-tier, decode-once cache (VRAM images → RAM pixels → disk, LRU-evicted), binds cached textures in the same frame they're drawn to eliminate per-frame pop-in, and decodes PNGs on a 4-thread pool. New core options let you choose the **caching method** — *Eager* (the stock-Beetle default: prefetch all of a texture's palettes) or *Lazy* (load each texture+palette on demand) — and set the **VRAM/RAM cache budgets** (defaults 3 GB / 2 GB). The on-disk pack format is unchanged. Full details: [HD_TEXTURE_CACHE.md](HD_TEXTURE_CACHE.md).

## Page-aligned texture replacement + HD QoL (experimental)

This fork layers an opt-in **page-aligned** dump/replacement mode on top of the HD
cache above: it works at whole VRAM texture-page granularity (clean 256×256 tiles)
instead of per-upload rectangles, which is friendlier for authoring static art, UI
and 3D content. It reuses the same three-tier cache, IO pool and budgets. Also added:

* **HD Dump Mode** — `upload_rect` (default) / `page_aligned` / `both` (collect both pack types in one playthrough).
* **HD Replacement Mode** — `upload_rect` (default) / `page_aligned`, with an optional **Cross-Mode Fallback** so an upload-rect pack can fill gaps from a page pack and vice-versa.
* **HD Texture Caching Method** gains **Lazy (synchronous)** — load on first use but block until ready (no pop-in, may briefly stutter).
* **HD Texture Folder** — keep the dump/replacement folders under the Content, System or Save directory; folders are auto-created.
* Live hotkeys (RetroArch **Game Focus** required): **`]`** toggles HD replacements on/off with an on-screen message; **`'`** reloads replacements from disk.

Page packs and upload-rect packs are **not** interchangeable. Full details and the
authoring workflow: [PAGE_ALIGN.md](PAGE_ALIGN.md). All of this is implemented in **C**
on upstream's C/C89 Vulkan renderer (`rhi/rhi_lib_vulkan.c`).

> **Recommended settings (current upstream caveat):** as of this build, upstream
> master (the C base this fork tracks) has a **CPU Dynarec** regression — *Maximum*
> can hard-crash, and live **PGXP** toggles on the *Beetle Interpreter* can crash —
> independent of this fork (reproduces on stock upstream). Until upstream settles,
> run with **CPU Dynarec = Disabled (Beetle Interpreter)** and **PGXP disabled**.
> The texture features are unaffected by these settings.

Tested with **RetroArch 1.22.2** on Windows (built via MSYS2 / MinGW-w64, GCC 16.1.0).

## Building

Beetle PSX can be built with `make`. To build with hardware renderer support, run `make HAVE_HW=1`. `make clean` is required when switching between HW and non-HW builds.

The prebuilt core in this fork is built and tested on **Windows** (`mednafen_psx_hw_libretro.dll`, via MSYS2 / MinGW-w64; `strip` the result to shrink it). The source is cross-platform, so the same `make HAVE_HW=1` produces `mednafen_psx_hw_libretro.so` on **Linux** and `mednafen_psx_hw_libretro.dylib` on **macOS** with no fork-specific changes — only the Windows binary is provided/tested here.

## Coding Style

The preferred coding style for Beetle PSX is the libretro coding style. See: https://docs.libretro.com/development/coding-standards/. Preexisting Mednafen code and various subdirectories may adhere to different styles; in those instances the preexisting style is preferred.

## Documentation

https://docs.libretro.com/library/beetle_psx/

https://docs.libretro.com/library/beetle_psx_hw/
