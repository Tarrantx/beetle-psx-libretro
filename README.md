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

## HD texture replacement caching

This fork overhauls the Vulkan renderer's HD texture replacement pipeline so packs stay smooth on demanding content — particularly multi-palette animated sprites like Alucard in *Castlevania: Symphony of the Night*. It replaces the stock eager loader with lazy per-(texture, palette) loading and a three-tier, decode-once cache (VRAM images → RAM pixels → disk, LRU-evicted with default budgets of 3 GB VRAM / 2 GB RAM), binds cached textures in the same frame they're drawn to eliminate per-frame pop-in, and decodes on a 4-thread pool. The on-disk pack format and core options are unchanged. Full details: [HD_TEXTURE_CACHE.md](HD_TEXTURE_CACHE.md).

## Building

Beetle PSX can be built with `make`. To build with hardware renderer support, run `make HAVE_HW=1`. `make clean` is required when switching between HW and non-HW builds.

## Coding Style

The preferred coding style for Beetle PSX is the libretro coding style. See: https://docs.libretro.com/development/coding-standards/. Preexisting Mednafen code and various subdirectories may adhere to different styles; in those instances the preexisting style is preferred.

## Documentation

https://docs.libretro.com/library/beetle_psx/

https://docs.libretro.com/library/beetle_psx_hw/
