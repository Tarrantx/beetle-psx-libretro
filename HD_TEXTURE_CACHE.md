# Beetle PSX HW — HD Texture Replacement Caching Overhaul

## TL;DR (one paragraph)

A Beetle PSX HW fork that overhauls the Vulkan renderer's HD texture replacement
pipeline for smooth, pop-in-free packs on demanding content — particularly
multi-palette animated sprites like Alucard in *Castlevania: Symphony of the
Night*. It replaces the stock eager loader with lazy per-(texture, palette)
loading and a three-tier, decode-once cache (VRAM images → RAM pixels → disk,
LRU-evicted with default budgets of 3 GB VRAM / 2 GB RAM), binds cached textures
in the same frame they're drawn to eliminate per-frame pop-in, and decodes on a
4-thread pool. The on-disk pack format and core options are unchanged.

## Summary

This fork reworks the **HD texture replacement** pipeline in the Vulkan
("Beetle PSX HW") renderer so replacement packs stay smooth on demanding content
— particularly animated sprites that reuse one texture across many palettes
(e.g. Alucard in *Castlevania: Symphony of the Night*), where the stock
implementation suffers persistent texture pop-in and load stalls.

What's different from upstream (Vulkan backend only — the on-disk pack format and
core options are unchanged):

- **Lazy, per-(texture, palette) loading.** A replacement is loaded only when
  that specific texture+palette combination is actually drawn, instead of eagerly
  loading every palette variant of a texture the moment it enters VRAM. Removes
  the load burst when textures first appear.
- **Three-tier, decode-once cache.** A VRAM cache of ready-to-bind GPU images,
  backed by a RAM cache of decoded pixels, backed by disk. Each combination is
  read and decoded at most once; re-draws are free. Both tiers are LRU-evicted
  with configurable budgets (default **3 GB VRAM / 2 GB RAM**).
- **Immediate in-frame binding.** A cached GPU image is bound on the *same* frame
  it's needed rather than one frame later — this eliminates the persistent
  per-frame pop-in that affected animated sprites even when their textures were
  already cached.
- **Multithreaded decode.** PNG decode + mipmap generation runs on a 4-thread
  pool instead of a single thread, so first-appearance loads land faster.
- **Built-in diagnostics.** An `[hdcache]` line is written to the RetroArch INFO
  log every 300 frames (decodes / GPU uploads / binds + cache occupancy) for
  tuning.

Packs are authored exactly as before: textures live in
`<content>-texture-replacements/`, named `<texturehash>-<palettehash>.png`.

## Technical detail

All changes are in `rhi/rhi_lib_vulkan.cpp` (the texture system is compiled only
with `HAVE_VULKAN` / `TEXTURE_DUMPING_ENABLED`).

### New types

- **`CachedHdImage` / `HdImageCache`** — a byte-budgeted LRU cache (`std::list` +
  `std::map`, keyed by `HdTextureId = {texture hash, palette hash}`) holding
  **decoded CPU pixel levels** (RGBA + mips) and alpha flags. Default budget
  `HD_CACHE_RAM_BUDGET = 2 GB`.
- **`CachedGpuImage` / `HdGpuCache`** — an analogous LRU cache holding
  **uploaded `Vulkan::ImageHandle`s** (ready to bind, in VRAM). Default budget
  `HD_CACHE_VRAM_BUDGET = 3 GB`. Eviction releases the handle, freeing VRAM once
  no live draw references it.
- Added `#include <list>`.

### `TextureTracker` changes

New members: `hd_gpu_cache`, `hd_cache`, `requested` (combos with an in-flight
load or no file on disk — a negative cache), `pending_attach` (combos to bind at
the next safe point), and `dbg_*` diagnostic counters.

- **`upload()`** — removed the eager `load_hd_texture(hash)` call that queued
  every palette variant of a texture on VRAM upload.
- **`want_combo(HdTextureId)`** *(new)* — queues a single disk load for one
  combination, skipping it if already cached, already requested, or absent from
  disk (inserted into `requested` as a permanent negative cache so missing files
  aren't retried).
- **`request_hd_texture(upload, palette)`** *(new; replaces the eager path)* —
  called from the draw path on a cache miss:
  1. **GPU-cache hit →** bind the existing image into `upload->textures`
     **immediately** (a ref-counted handle copy, no Vulkan commands, safe
     mid-draw);
  2. **CPU-cache hit →** add to `pending_attach` for a GPU upload at the next
     safe point;
  3. **miss →** `want_combo()` (disk load).
- **`get_hd_texture_index()`** — the per-draw overlap loop now calls
  `request_hd_texture()` on a miss and **re-checks `upload->textures`
  immediately afterward**, so an in-frame GPU-cache bind is used by the current
  draw. *(This is the pop-in fix: previously every bind was deferred to
  `on_queues_reset` one frame late, so a sprite frame drawn once per VRAM
  residency always displayed the native texture.)*
- **`on_queues_reset()`** — rewritten:
  - drains IO responses into `hd_cache` (decode-once) and clears them from
    `requested`;
  - an **attach pass** over `pending_attach` + new responses binds combos whose
    base texture is currently resident — GPU-cache hit = handle copy; CPU-cache
    hit = `upload_texture()` then store the result in `hd_gpu_cache`;
  - combos whose base texture isn't resident yet are **kept in the cache, not
    discarded**, and bind on a later frame when that animation frame's data
    returns to VRAM (this removes the stock path's decode-then-discard waste);
  - dimension-mismatched replacements are evicted and negatively cached.
- **`reload_textures_from_disk()`** — clears all caches (`hd_gpu_cache` /
  `hd_cache` / `requested` / `pending_attach`) so edited files take effect.
- **`endFrame()`** — emits the `[hdcache]` INFO diagnostics every 300 frames.
- `load_hd_texture()` is retained (now used only by `load_state()` to re-warm HD
  textures after a savestate load).

### IO subsystem

- **`io_thread`** — converted from a single worker that drained the entire queue
  into a **pool worker**: it takes one request at a time, runs PNG decode +
  mipmap generation **outside** the channel lock (workers run in parallel),
  pushes the response under the lock, and cascade-signals the next worker.
- **`IOThread`** — the constructor spawns `NUM_IO_THREADS` (4) detached workers,
  each given its own heap-allocated `shared_ptr` to the channel; the destructor
  uses `scond_broadcast` to wake all workers for shutdown.

### Build

Build the HW core as usual — `make HAVE_HW=1` — then optionally `strip` the
resulting `mednafen_psx_hw_libretro.dll`. No new dependencies; the caches use
only the C++ standard library plus the `stb_image` and libretro threading already
vendored in the tree.
