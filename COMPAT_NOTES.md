# VRAM Write Filtering + FF7/LoD Compat Mode — Branch Notes

Custom DuckStation branch (`vram-write-filtering`, private repo
`LuismaSP89/Duckstation_Luisma_Privado`) that makes texture filters (xBR etc.) apply to
pre-rendered backgrounds, 2D images and FMVs like the old PeteOpenGL2Tweak xBRZ scaler did,
plus a compatibility mode that makes filtering usable in Final Fantasy VII and The Legend of
Dragoon without artifacts.

Maintained with the help of Claude (Anthropic). This file documents everything needed to
understand and rebase the branch without any other context. The prebuilt Windows x64 ZIP is
attached to this repo's Releases; new pushes to the branch also build automatically via the
GitHub Actions workflow in `.github/workflows/windows-x64-custom.yml` (artifact
`duckstation-windows-x64-vram-write-filtering`, downloadable from each Actions run for 90
days). Note: in a private repo, Actions minutes count against the account's free quota.

## What the branch adds

### 1. VRAM write filtering (all games, opt-in)

- New setting: `GPU / FilterVRAMWrites` (checkbox *"Filter Pre-Rendered Backgrounds (VRAM
  Writes)"* in Graphics → Rendering, Qt UI only). Uses the sprite texture filter, falling back
  to the main texture filter (`GetVRAMWriteTextureFilter()` in `gpu_hw.cpp`).
- `GenerateVRAMWriteFragmentShader` applies the selected batch texture filter when upscaling
  CPU→VRAM uploads into the scaled VRAM texture, reusing `WriteBatchTextureFilter`.
  **Critical invariant**: the first subpixel of every scaled block keeps the exact source
  value, because palette/CLUT fetches and the 24-bit display decode sample scaled block
  corners — this keeps 3D textures, save states and FMV decoding bit-exact.
- VRAM readback (`GenerateVRAMReadFragmentShader`) switches from box-filtering to corner
  point sampling while the option is on, keeping CPU-visible VRAM round-trips exact.
- 24-bit displays (FMVs/stills) are extracted at 1x by DuckStation; a separate upscale pass
  (`GenerateDisplay24FilterFragmentShader`, `m_display_24bit_filter_pipeline`) applies the
  filter to them, skipping interlaced modes.

### 2. FF7/LoD compat mode (gamedb-gated)

Only active when a game has the `DisableSpriteTextureFiltering` gamedb trait (FF7 and Legend
of Dragoon, all discs/regions) **and** `FilterVRAMWrites` is enabled. `game_database.cpp` then
skips the trait's sprite-filter disable and sets the runtime-only flag
`Settings::gpu_sprite_nearest_coverage` (not persisted). `gpu_hw.cpp` passes it into
`GenerateBatchFragmentShader` as `filter_nearest_coverage` / `filter_chroma_key`
(macros `FILTER_NEAREST_COVERAGE` / `FILTER_CHROMA_KEY`), **only for opaque render passes**
(`TransparencyDisabled` / `OnlyOpaque`) — semitransparent passes (FF7's dithered shadow
layers, glows) keep stock filtering, which renders them softly.

Why: FF7/LoD compose fields from layered tiles whose textures contain **chroma matte garbage**
(pure green 0x03E0, blue 0x7C00, cyan 0x7FE0, red, magenta — varies per scene) next to real
content. Invisible with nearest sampling (draw order covers it), but any texture filter blends
it into content edges as colored dashes/lines, and silhouette erosion/growth can reveal or
extend it. This is exactly why upstream's gamedb disables sprite filtering for these games.

Mechanisms (all in the `TEXTURE_FILTERING` branch of the batch fragment shader,
`gpu_hw_shadergen.cpp`), each one exists because a specific artifact was observed:

| Mechanism | Prevents |
|-----------|----------|
| Chroma key: saturated cold/pure hues (5 families: green, blue, red, cyan, magenta; dominant ≥0.55, others ≤0.28, ≥2× dominance) treated as transparent in filter taps (`SampleFromVRAM` wrapper over `SampleFromVRAMRaw`) | Colored dashes/tints along content edges |
| Exact-coverage base: discard iff exact center texel (`ncol`) is transparent — silhouettes can never erode | Colored dots from occluded matte revealed at concave silhouette spots |
| Supported outward growth: center-transparent pixels kept only if `ialpha ≥ 0.45` and filtered luminance sum ≥ 0.3 | Black dots from dark art outlines being extended onto layers behind, while keeping smooth cut-out silhouettes |
| Luminance floor: center-opaque pixels whose filtered color sum < 0.15 (near-pure black) and < 50% of the exact texel's sum snap back to the exact color | Dark hairline seams along layer cuts (residual matte/transparency darkening) without touching legitimate dark-outline blends |
| Opaque writes (`ialpha = 1`, `texcol.a = ncol.a`) | Filtered edges alpha-blending over occluded matte |

Tunable thresholds if a game shows residuals: 0.15/0.5 (luminance floor), 0.45/0.3 (growth
support), 0.55/0.28/2× (chroma families).

## Files touched (conflict hotspots for rebases)

- `src/core/gpu_hw_shadergen.cpp` / `.h` — main shader changes (biggest conflict risk).
- `src/core/gpu_hw.cpp` / `.h` — pipeline plumbing, `GetVRAMWriteTextureFilter`, 24-bit pass,
  compat gating at the `GenerateBatchFragmentShader` call site.
- `src/core/settings.h` / `settings.cpp` — `gpu_filter_vram_writes`, `gpu_sprite_nearest_coverage`.
- `src/core/system.cpp` — settings-change detection lines.
- `src/core/game_database.cpp` — `DisableSpriteTextureFiltering` trait handling.
- `src/duckstation-qt/graphicssettingswidget.{cpp,ui}` — the checkbox.
- `.github/workflows/windows-x64-custom.yml` — CI (x64 only, artifact
  `duckstation-windows-x64-vram-write-filtering`).

## How to rebase onto upstream

```bash
# once: clone this private repo (needs your GitHub auth) and add the official repo as upstream
git clone https://github.com/LuismaSP89/Duckstation_Luisma_Privado.git
cd Duckstation_Luisma_Privado
git remote add upstream https://github.com/stenzek/duckstation.git

# each update:
git fetch upstream
git checkout vram-write-filtering
git rebase upstream/master
# resolve conflicts (most likely in gpu_hw_shadergen.cpp; re-apply the blocks guarded by
# FILTER_NEAREST_COVERAGE / FILTER_CHROMA_KEY and the SampleFromVRAM wrapper)
git push --force-with-lease origin vram-write-filtering
```

The push triggers the Windows x64 build automatically (`windows-x64-custom.yml`); download the
artifact from the Actions run (or create a new Release with the ZIP for permanent storage:
`gh release create <tag> --target vram-write-filtering <zip>`). If upstream renames
`SampleFromVRAM` or restructures the batch fragment shader, re-apply the concepts from the
table above rather than the literal diff.

## Verifying after a rebase

Test with FF7 (SCES-20900 etc.), settings: `SpriteTextureFilter = xBR`,
`FilterVRAMWrites = true`, resolution scale ≥ 4x. Check: (1) pre-rendered fields show **no**
colored dashes/dots or dark hairlines along layer contours; (2) cut-out objects (foliage,
cursors, exit arrows) keep smooth silhouettes; (3) other games (e.g. Resident Evil 2
backgrounds, FMVs) still get filtered; (4) save states and 3D textures stay intact.

## Known limitations

- Compat mode trades a small amount of edge smoothing at layer cuts for artifact-free output;
  it is intentionally FF7/LoD-only via the gamedb trait.
- Semitransparent passes keep stock filtering by design.
- A per-game off switch is the *Filter Pre-Rendered Backgrounds* checkbox in game properties.
