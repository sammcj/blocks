# CLAUDE.md

Non-obvious notes for future coding agents. User-facing docs live in `README.md`; don't duplicate them here.

## One script does everything

`blocks.py` is the only entry point. It writes the full SVG tree, rasterises PNGs in parallel if `cairosvg` is importable, and writes one PPTX per enabled palette if `python-pptx` is importable. There used to be a separate `svg2png.py` and a `scripts/README.md`; don't recreate either.

Each run `shutil.rmtree`s the palette-set subdirectory before writing (`svg/<set>/` and `png/<set>/`). Anything a user parks under those dirs gets wiped on regen. Palette-set subdirs for sets _not_ being regenerated are left alone — so `--palettes mantel` doesn't touch `svg/classic/`.

## Dependencies via PEP 723

`blocks.py` declares `cairosvg` and `python-pptx` in an inline `# /// script` block at the top, so `uv run blocks.py` just works without a separate requirements file. There's no `requirements.txt` — if you add a dep, update the inline block. Both deps are also lazy-imported with graceful fallback, so the script still runs (with reduced output) if a user installs it under plain `python3` without the deps.

## Geometry — read before changing constants

- `PLATE_H = 12` matches real-world studded-brick proportions (brick body / stud pitch = 1.2). It was 5 previously and rendered bricks at ~half-height — that bug went unnoticed for a while. Don't lower it without re-tuning TALL specs and eyeballing output in a browser.
- `U = 30`, `S = U // 2` is 2:1 pixel dimetric, not true 30° iso. Intentional for clean strokes and pixel-art iso convention. Real 30° would be `S = U * tan(30°) ≈ 17.3`.
- Iso blocks anchor rear-top corner at origin `(0, 0)`. viewBox calculations assume this.

## Mirror + dedup logic

`iso_block(..., mirror=True)` flips the x-axis _and_ swaps `p.left`/`p.right` fills so the sun stays upper-left. Two dedup rules live in `main()`:

- Square blocks (`W == D`) skip the mirror variant — rotationally symmetric, so the mirror SVG renders identically.
- `dedupe_for_side()` keeps the smallest-D representative per `(W, H_units, show_studs)` tuple. Side filenames like `brick-2x2-<colour>-side.svg` cover all 2-wide bricks of that height, not literally the 2x2.

When adding a new shape category, decide whether either rule applies.

## Slope renderer is W=1 only

The `slope=True` branch in `iso_block()` drops `left` and `front` corners by `slope_drop` and draws trapezoidal side walls. Angles are hard-coded:

- D ≤ 2 → 45° (slope_drop = H, touches ground, triangular side)
- D ≥ 3 → 33° (slope_drop = 2H/3, H/3 front wall remains)

Widening to W > 1 needs separate side-wall geometry on both sides — current code would render wrong silently.

## Colour ramps

`MANTEL_PALETTES` uses hand-tuned hex values. Everything else uses `auto_ramp(name, base_hex)` which derives 6 shades via HSL darken/lighten.

Edge case: when `l > 0.75` (white and other very-light bases), `auto_ramp` uses a dark desaturated grey for the outline instead of a darker tint of the base. Without this, white bricks get invisible outlines.

Accent palettes — `Palette.accent=True` — render only on `ACCENT_SLUGS` (`brick-2x2`, `brick-2x4`). Top and side views skip accent palettes entirely. Currently marked accent: mantel's `cloud`, anthropic's `ivory` (both are light-on-very-light and unreadable across the full catalogue).

## PPTX packer

Relative scale across pieces is preserved by setting `width` and `height` in absolute pt on each shape (`viewBox_dim × --pptx-density`). Slide-tool auto-fit never enters the picture. This replaces the earlier "render PNGs and pray the slide tool doesn't rescale" approach.

Layout invariants worth keeping:

- **Row = family.** A family is `(is_baseplate, view, is_iso_mirror, category, W, D)`. Sort groups all colour variants of, say, `brick-2x4-iso` contiguously, and the packer places them on a single row so users can compare colours at a glance. Smaller families can share a row with their neighbours when combined widths fit.
- **Family atomicity.** A family never splits across slides; if it doesn't fit below the current cursor, the packer flushes to a new slide. Only exception is families physically too big for one slide (rare; the 4-palette 16x32 baseplate is right on the threshold).
- **break_key quarantine.** `break_key = (is_baseplate, is_iso_mirror)` — crossing it forces a slide break. This keeps iso right-facing, iso left-facing, and giant baseplates on separate slides. The user explicitly asked for this after seeing mixed-facing iso on the same slide.
- **Baseplate threshold.** `_BASEPLATE_MIN_AREA = 32` matches the `# Baseplates` comment in `PLATES`. Don't invent a new threshold; update both together if the catalogue grows.

Sort order is `(is_baseplate, view, is_iso_mirror, category, w*d, w, d, name)`. Baseplates last so giants don't orphan small pieces on shared slides.

### brick-top == plate-top visually

Top-view only sees W×D, not `H_units`, so `brick-2x4-top.png` and `plate-2x4-top.png` are pixel-identical for the same palette. `python-pptx` deduplicates the image bytes on save. Any diagnostic that inspects the pptx via blob hashes will return an ambiguous name. The packer still treats them as distinct families (category differs), so layout is correct even though the pixels aren't.

## Palette default is mantel

`--palettes` defaults to the first key in `PALETTE_SETS` (currently `mantel`). The user explicitly wants public-facing runs to ship a single curated palette by default, not render everything. `--pptx-palettes` tracks `--palettes` unless overridden. Don't flip the defaults back to `all` without asking.

## PNG parallelism gotchas

- `ProcessPoolExecutor` on macOS reads `sysconf(SC_SEM_NSEMS_MAX)` which the Claude Code sandbox blocks with `PermissionError`. Run `blocks.py` with the sandbox disabled when regenerating PNGs, or use `--no-png` inside the sandbox.
- `_rasterise_one` takes a single tuple argument (not `*args`) because `ty` can't type-check `pool.submit(fn, *tuple)` through ParamSpec.
- `cairosvg` is lazy-imported in `_try_import_cairosvg()` and inside the worker. Same pattern for `python-pptx` in `_try_import_pptx()`. Don't add either to module-level imports — the PEP 723 block is for `uv run`, but plain-Python runs should still degrade gracefully.

## Linting and verification

User prefers `uvx ty check blocks.py` before accepting changes. Pyright flags `import cairosvg` and `import pptx` separately; that's expected — ty's `# type: ignore[import-not-found]` suppression is what counts and is enough for the user.

No automated tests. Verification is visual: read the generated SVG, cross-check a few polygon coordinates against the geometry, and/or open `Building Blocks.pptx` in a viewer. For the PPTX specifically, slide-coverage percentages and per-row image counts (both visible via `python-pptx` introspection) are useful diagnostics.

## Safety net friction

Do not run `rm -rf` on directories even within cwd. Cleanup scripts use Python's `shutil.rmtree` instead. This is why `blocks.py` does its own cleanup rather than shelling out.
