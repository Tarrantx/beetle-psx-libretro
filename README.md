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
* Page-aligned texture dumping & replacement (Vulkan renderer), see [PAGE_ALIGN.md](PAGE_ALIGN.md);

## HD texture replacement caching

This fork overhauls the Vulkan renderer's HD texture replacement pipeline so packs stay smooth on demanding content — particularly multi-palette animated sprites like Alucard in *Castlevania: Symphony of the Night*. At its core is a three-tier, decode-once cache (VRAM images → RAM pixels → disk, LRU-evicted). New core options let you choose the **caching method** — *Eager* (the stock-Beetle default: prefetch all of a texture's palettes), *Lazy* (load each texture+palette on demand, in the background), or *Lazy (synchronous)* (also on demand, but blocks until ready: no pop-in, may briefly stutter when many new textures appear at once) — and set the **VRAM/RAM cache budgets** (defaults 3 GB / 2 GB). The on-disk pack format is unchanged. The internal mechanics that keep it smooth are described under **Under the hood**. Full details: [HD_TEXTURE_CACHE.md](HD_TEXTURE_CACHE.md).

Tested with **RetroArch 1.22.2** (git 69a4f0e, build date Nov 20 2025, Compiler: MinGW 10.2.0 64-bit) on Windows.
<img width="1726" height="436" alt="BeetleVRAM" src="https://github.com/user-attachments/assets/6d39bed5-8ead-4b3e-af27-c056e3513818" />

## Page-aligned texture dumping & replacement

An optional dumping/replacement mode that works at whole VRAM **texture-page** granularity — one clean 256×256 tile per page — instead of the default per-upload rectangles, addressing the fragmented "sections" produced by the stock dumper ([libretro/beetle-psx-libretro#918](https://github.com/libretro/beetle-psx-libretro/issues/918)). It is controlled by two independent core options:

* **HD Dump Mode** — how textures are dumped: *Upload-rect (default)*, *Page-aligned*, or *Both*.
* **HD Replacement Mode** — how replacements are looked up: *Upload-rect (default)* or *Page-aligned*.

**HD Dump Mode = Both** writes both pack types to their two separate folders in a single playthrough, so you don't have to replay a game once per mode. A separate **HD Replacement Cross-Mode Fallback** option lets the two replacement modes back each other up: when the active mode finds no match for a draw, the *other* mode's pack is checked before falling back to native (works both directions) — so a mostly-page pack can fill its gaps with a few upload-rect textures, and vice versa, without converting anything. It is off by default and only does the extra lookup on a genuine miss.

Page-aligned packs use their own folders (`<game>-texture-dump-pages/`, `<game>-texture-replacements-pages/`) so they never collide with upload-rect packs — the two are not interchangeable. The mode reuses the same three-tier cache and loader as the HD caching system above, with no shader changes. It suits static, reused art (backgrounds, UI, 3D games); upload-rect remains better for animated multi-palette sprites, and since the two sides are independent you can pick per game. Default behaviour is unchanged. Full details: [PAGE_ALIGN.md](PAGE_ALIGN.md).

## Texture folder location

By default, dump and replacement folders are created next to the loaded content (`<game>-texture-dump/` and friends). A new **HD Texture Folder** core option can instead place them under RetroArch's **System** or **Save** directory — a single central location for every game's packs, each game keeping its own `<name>-…` subfolder. Point that directory wherever you like in RetroArch's *Settings → Directory*.

The required folders are also **created automatically** when texture dumping or replacement is enabled and a game is running, so there's no need to make them by hand first.

## Under the hood

These changes have no core options of their own but shape how the HD texture system behaves:

* **Same-frame binding** — a cached GPU texture is bound on the *same* frame it's needed rather than one frame later, removing the persistent per-frame pop-in that affected animated sprites even when their textures were already cached.
* **Multithreaded decode** — PNG decode + mipmap generation runs on a 4-thread worker pool instead of one thread, so first-appearance loads land faster.
* **IO priority queue** — on-demand loads (textures needed this frame) jump ahead of background work (prefetch, dumps, save-state warm-up), so a burst of background loading can't stall textures about to be drawn.
* **Off-thread dump decode** — when dumping, the render thread only snapshots raw VRAM words + the palette; the index→RGBA decode *and* PNG encode run on the worker pool. Collecting dumps (especially *HD Dump Mode = Both*) no longer introduces input lag when many new textures first appear.
* **Diagnostics** — an `[hdcache]` line is written to the RetroArch INFO log periodically (decodes / GPU uploads / binds, cache occupancy, active caching + replacement mode, page-mode counters) for tuning.

## Building

Beetle PSX can be built with `make`. To build with hardware renderer support, run `make HAVE_HW=1`. `make clean` is required when switching between HW and non-HW builds.

The prebuilt core in this fork is built and tested on **Windows** (`mednafen_psx_hw_libretro.dll`, via MSYS2 / MinGW-w64; `strip` the result to shrink it). The source is cross-platform, so the same `make HAVE_HW=1` produces `mednafen_psx_hw_libretro.so` on **Linux** and `mednafen_psx_hw_libretro.dylib` on **macOS** with no fork-specific changes — only the Windows binary is provided/tested here.

## Coding Style

The preferred coding style for Beetle PSX is the libretro coding style. See: https://docs.libretro.com/development/coding-standards/. Preexisting Mednafen code and various subdirectories may adhere to different styles; in those instances the preexisting style is preferred.

## Documentation

https://docs.libretro.com/library/beetle_psx/

https://docs.libretro.com/library/beetle_psx_hw/
