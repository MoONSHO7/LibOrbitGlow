# LibOrbitGlow-1.0

A high-performance utility library for creating and managing visual frame glows, animations, and particle effects. Designed for World of Warcraft 12.0.0 and above. Features unified resource pooling.

> **Standalone library.** Requires only **LibStub** (shipped by virtually every addon, and by any host that embeds this lib) — no other addon dependencies. Embed the `LibOrbitGlow-1.0/` folder (or install it standalone) and consume it via `LibStub("LibOrbitGlow-1.0")`.

## Usage

The library exposes a simplified facade that abstracts specific animation groups, textures, and geometry engines. Consumers only need to ask for a specific glow type and pass an options table.

### Showing a glow

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
if not lib then return end

lib.Show(frame, "Pixel", {
    key = "myComponentGlow",
    color = { 0.2, 0.8, 1, 1 },
    lines = 8,
    frequency = 0.5,
    thickness = 2
})
```

### Hiding a glow

It's crucial to hide glows using the exact same type and key you used to show them, so the library can gracefully stop animations and recycle frames back into the global pool.

```lua
lib.Hide(frame, "Pixel", "myComponentGlow")
```

### One call for any glow (recommended)

`lib.Show`/`lib.Hide` reach the built-in **engine** types directly. If your addon stores a glow choice as a setting, that value might be an engine type (`"Pixel"`) *or* the name of a glow a media pack registered (`"pinring"`) — and you shouldn't have to branch on which. `lib.Apply` / `lib.Remove` resolve either:

```lua
local id = mySettings.glow   -- "Pixel", "Medium", or any registered pack glow like "pinring"
lib.Apply(frame, id, { key = "proc", color = { 0.3, 0.8, 1, 1 } })
-- ... later:
lib.Remove(frame, id, "proc")

-- A continuous glow that stays until you Remove it (no intro; the outro still plays on Remove) -- add loop = true:
lib.Apply(frame, id, { key = "buff", color = { 1, 0.8, 0.2, 1 }, loop = true })
```

A registered name plays its full proc lifecycle (one-shot start → loop, one-shot end on `Remove`); `loop = true` plays the loop continuously with no intro (the end phase still plays on `Remove` — use `Proc:Clear` for an instant cut); an engine type shows/hides directly. `Remove` accepts either a bare key or the same options table you passed to `Apply`. `Apply` writes `glow = id` into the options table you pass. Use these everywhere you drive a glow from user choice.

> **Call style:** the top-level verbs are **dot**-called — `lib.Show` / `lib.Hide` / `lib.Apply` / `lib.Remove` / `lib.PreLoad`. The registry and the `Proc`/engine sub-namespaces are **colon** (method) calls — `lib:RegisterGlow(...)`, `lib.Proc:Start(...)`, `lib.Pixel:Show(...)`.

### Combat and secret safety (alpha hiding)

When tracking auras, cooldowns, or power states that return WoW 12.0 'secret values', you **cannot** execute logic branches based on those values. This means you cannot conditionally call `lib.Hide()` in response to an aura dropping during combat.

To safely hide glows in these scenarios, rely on native alpha propagation or update the `options.color` alpha channel instead of calling `lib.Hide()`. Pass only a plain (non-secret) number as the alpha — `x and 1 or 0` on a secret boolean is itself a Lua-side branch and will throw.

```lua
-- Safe: alpha is a plain number sourced from a non-secret read (e.g. a numeric curve)
lib.Show(frame, "Pixel", { color = { 1, 0, 0, alpha } })
```

If show/hide must follow a secret boolean, derive visibility through a C++ sink on the glow's parent (`parent:SetAlphaFromBoolean(secretBool, 1.0, 0.0)`) rather than branching the secret in Lua to compute an alpha.

## Glow types

Consumers pass these exact string identifiers to `lib.Show` as the second argument. The engine automatically resolves the underlying math, animation groups, and atlases. Passing an unknown type will `error()`.

| Type | Description | Engine |
|---|---|---|
| `"Thin"` | Thin swirling ants | Flipbook |
| `"Thick"` | Thick proc loop | Flipbook |
| `"Medium"` | Standard action bar proc | Flipbook |
| `"Classic"` | Classic WoW action button flash and ant swirl | Manual stepper |
| `"Pixel"` | Modern pixel lines tracing the outer border | Geometry |
| `"Autocast"` | Native pet bar autocast shine squares | Geometry |

## Proc lifecycle (registry + media packs)

The library is the **engine**; the animated glows come from a **registry** that media packs populate — it owns no glow textures itself beyond a couple of WoW baselines. A pack (e.g. **Orbit-Glow-Pack**) calls `lib:RegisterGlow(name, def)` for each glow; `lib.Proc` (or `lib:Apply`) then resolves and plays whatever is registered. A baseline `"blizzard"` glow is always present, so `Proc` works with no pack installed, and any unknown glow name falls back to it.

**Registry API** — `lib:RegisterGlow(name, def)` (returns `false` and warns on a malformed def) · `lib:UnregisterGlow(name)` · `lib:IsGlowRegistered(name)` · `lib:GetGlowInfo(name)` · `lib:GetGlowList()` (sorted names — populate UI dropdowns from this; `GetGlowInfo(name).source` groups them by pack).

**Proc methods** — the consumer names the glow and the icon's corner shape; the library never decides the shape (the host maps its own border style to a shape the glow's `def.shapes` provides, defaulting to `"square"`):

| Method | Plays |
|---|---|
| `lib.Proc:Start(frame, o)` | one-shot **start** → looping **loop** (skips straight to loop if the glow has no start phase) |
| `lib.Proc:Stop(frame, o)` | one-shot **end** → release (skips straight to release if the glow has no end phase) |
| `lib.Proc:Loop(frame, o)` | **loop** only, forever — for previews / persistent glows that never "expire" |
| `lib.Proc:Clear(frame, o)` | stop immediately, no end animation |

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
lib.Proc:Start(frame, { glow = "pinring", shape = "square", color = {0.3,0.8,1,1}, key = "proc" })
-- ... later, when the proc expires:
lib.Proc:Stop(frame, { glow = "pinring", shape = "square", key = "proc" })
```

`o` accepts `glow`, `shape`, `color`, `key`, `frameLevel`, `scale`, `padding`, `offsetScale` / `offsetX` / `offsetY` (nudge the glow off-center), and per-phase durations `startDuration` (0.28), `loopDuration` (1.0), `endDuration` (0.20) seconds. (Most callers should prefer the higher-level `lib.Apply` / `lib.Remove`, which route engine types *and* registered names — see above.)

## Build a glow pack

A pack is just an addon that calls `lib:RegisterGlow` for each of its glows during load (no dependency on Orbit — only on this library being present). The `def` table describes where the art lives and how to play it. **One** of `path` / `resolve` / `atlas` / `engine` is required; everything else is optional.

| Field | Type | Default | Purpose |
|---|---|---|---|
| `path` | `string` | — | File-prefix for a flipbook sheet; resolves to `"<path>-<phase>[-<shape>]<layer><ext>"` |
| `resolve` | `function(phase, shape, layer) → path\|nil` | — | Full override of path resolution — use any naming you like; return `nil` for a phase/layer you don't ship |
| `atlas` | `string` | — | A Blizzard **atlas name** (single layer, `SetAtlas`) instead of files |
| `engine` | `string` | — | Delegate to a built-in engine (`"Pixel"`, `"Autocast"`, `"Classic"`, `"Thin"`, `"Thick"`, `"Medium"`) |
| `layered` | `boolean` | `false` | Draw a tinted **BLEND body** + near-white **ADD core** (depth). `false` = one tinted layer |
| `core` | `boolean` | `true` | When `layered`, include the `-core` layer (set `false` for body-only with custom blends) |
| `blendMode` | `string` | `"ADD"` | Blend for a single-layer `path`/`atlas` def |
| `bodyBlend` / `coreBlend` | `string` | `"BLEND"` / `"ADD"` | Per-layer blend overrides for a `layered` def |
| `phases` | `table` | all | Set of phases the art provides, e.g. `{ loop = true }` |
| `loopOnly` | `boolean` | `false` | Shorthand for `phases = { loop = true }` |
| `shaped` | `boolean` | `true` | Include the `-<shape>` segment in the default name (set `false` if your files aren't shape-specific) |
| `ext` | `string` | `".tga"` | File extension for `path` resolution |
| `rows` / `cols` / `frames` | `number` | `6` / `5` / `30` | Flipbook grid (Blizzard's 30-frame 5×6 layout) |
| `shapes` | `table` | `{ square = true }` | Corner shapes your art ships, e.g. `{ square = true, round = true }` |
| `scale` | `number` | engine default | Default size multiplier (path/atlas defs). For an `engine` def, sizing goes through `options` instead |
| `desaturated` | `boolean` | `true` | (`atlas` defs) desaturate before tinting |
| `options` | `table` | — | (`engine` defs) extra engine options (`lines`, `length`, `particles`, …) |
| `source` | `string` | `"Unknown"` | Group label for the showcase / picker (use your pack name) |

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
if not lib then return end
local ROOT = "Interface\\AddOns\\MyGlowPack\\Textures\\"

-- 1. Full Orbit-style layered glow: start/loop/end × body+core, shape-suffixed.
lib:RegisterGlow("myproc", {
    layered = true, path = ROOT .. "myproc",
    shapes = { square = true }, source = "MyGlowPack",
})   -- expects myproc-loop-square.tga, myproc-loop-square-core.tga, myproc-start-square.tga, ...

-- 2. Single-file loop-only flipbook (one sheet, no start/end, no core, no shape suffix):
lib:RegisterGlow("spark", {
    path = ROOT .. "spark", loopOnly = true, shaped = false,
    rows = 4, cols = 4, frames = 16, blendMode = "ADD", source = "MyGlowPack",
})   -- expects exactly one file: spark-loop.tga

-- 3. Arbitrary naming via a resolver (return nil for phases you don't have):
lib:RegisterGlow("ribbon", {
    source = "MyGlowPack", rows = 5, cols = 6, frames = 30,
    resolve = function(phase, shape, layer)
        if phase ~= "loop" or layer ~= "" then return nil end
        return ROOT .. "fx\\ribbon_sheet.tga"
    end,
})
```

The flipbook grid (`rows`/`cols`/`frames`) describes how the sheet is sliced — a 5-wide, 6-tall sheet of 30 frames plays one frame per cell over `loopDuration` seconds (`.tga`, any cell size; the shipped pack uses 128×128 cells on a 640×768 sheet). **Layered/`path` art must be grayscale** — the layers are *tinted* by your `color` at draw time (not desaturated), so coloured source art multiplies muddily; the body takes your colour and the core is auto-brightened toward white for depth. The host (Orbit) discovers your glows purely through `GetGlowList()` — no Orbit-side code change is needed to surface a new pack.

If a glow renders blank, call `lib:GetResolvedPaths(name [, shape])` to get the exact path strings the lib is trying (eyeball them against disk; it returns `{ engine = … }` / `{ atlas = … }` for non-file glows, or `nil` if unregistered — omit `shape` for the default square paths), or set `lib.DEBUG = true` to print each path as it plays — WoW's `SetTexture` fails **silently** on a missing/mistyped file. A loop-only pack must declare `loopOnly = true` (or `phases`); otherwise the lib tries to play `-start`/`-end` art it assumes exists.

### Lower-level flipbook

`lib.Flipbook:Show(frame, opts)` is the raw sink the registry feeds. It accepts `atlas` (a Blizzard atlas name, or a file path with `isTexture = true`), `rows`/`cols`/`frames`, `speed`, `blendMode`, `color`, `key`, plus `once = true` + `onFinished` for a single one-shot. Use it directly only if the registry/`Proc` model doesn't fit.

## Showcase

The library bundles a self-contained visual showcase (`GlowShowcase.lua`) — a movable, scrollable grid of every registered glow, grouped into collapsible sections by source pack. Left-click a glow to apply it to `ActionButton1`, right-click to re-roll colours, collapse sections with the `±` headers. It depends only on LibOrbitGlow + Blizzard UI, so it travels with the library and works in any host:

```lua
LibStub("LibOrbitGlow-1.0").Showcase:Toggle()   -- also :Show() / :Hide()
```

A `/orbitglow` slash command is registered as a built-in convenience for the same toggle (guarded — if the command is already taken, the lib leaves it alone and you drive the showcase via `Showcase:Toggle()`).

## Embedding & versioning

Targets **retail 12.0+** (the showcase uses retail-only `ScrollUtil`/`MinimalScrollBar`; the core engine uses `CreateMaskTexture`/`CreateFramePool`). Include the in-folder manifest from your TOC or XML — it loads the core then the optional showcase:

```xml
<Include file="Libs\LibOrbitGlow-1.0\LibOrbitGlow-1.0.xml"/>
```

Delete the `GlowShowcase.lua` line from that XML to embed the engine without the demo panel/slash command.

LibStub hands every embedder whichever copy loaded **first** (highest `MINOR` wins), so feature-probe rather than assume when you rely on a newer method — a co-installed older copy may not have it:

```lua
if lib.Apply then lib.Apply(frame, id, opts) else lib.Show(frame, id, opts) end
```

## Global options

These options act globally across almost all glow engines. They are passed as the 3rd argument to `lib.Show(frame, glowType, options)`.

| Field | Type | Description |
|---|---|---|
| `key` | `string` | Unique identifier used for tracking and hiding. Default: `"Default"` |
| `color` | `table` | `{r,g,b,a}` array or a 12.0 `Color` object. Default: white |
| `frameLevel` | `number` | Relative frame level above the parent frame. Default: `8` |
| `desaturated`| `boolean` | Atlas-based engines (`Thin`/`Thick`/`Medium`) desaturate their atlas by default so `color` tints it cleanly; pass `false` to keep the atlas's native colors. Ignored by `Pixel`/`Autocast` (solid-color textures). Default: `true` |
| `force` | `boolean` | On a re-show of a live glow, rebuild even when the options are unchanged (defeats the re-tint fast path). Honored by all engines. Default: `false` |

## Engine-specific options

Specific glow engines support additional fine-tuning properties.

### Flipbook engines (`Thin`, `Thick`, `Medium`)
- `scale` (number): Multiplier applied to width and height. Default: `1.4`.

### Pixel engine (`Pixel`)
- `lines` (number): How many tracing particles are generated. Default: `8`
- `frequency` (number): Speed scaler. Positive = faster than baseline (`period = 0.25 / frequency`); `0` or unset = engine baseline (4s); negative = slower than baseline (`period = baseline * (1 + |frequency| * 8)`). Default: unset.
- `thickness` (number): Width of the tracing particles. Default: `2`
- `length` (number): Arc length of each tracing dash. Default: auto-derived from the frame perimeter and `lines`.
- `border` (boolean): Renders a dark, translucent backdrop strictly inside the pixel bounds to obscure the parent frame slightly for contrast. Default: `true`
- `xOffset` / `yOffset` (number): Margin expanding the trace tracking box away from the edges.
- `pixelScale` (number): Physical-pixel size used to snap the trace to the device grid (`pixelScale / frame:GetEffectiveScale()`). Default `1` (whole-UI-unit rounding). Pass the host's screen scale for crisp lines at any UI scale.

### Autocast engine (`Autocast`)
- `particles` (number): Number of points mapping the border. Default: `4`
- `frequency` (number): Rotation speed scaler. Same sign convention as Pixel — positive = faster (`period = 1 / frequency`); `0` or unset = engine baseline (8s); negative = slower (`period = baseline * (1 + |frequency| * 8)`). Default: unset.
- `scale` (number): Scaling scalar on each particle. Default: `1`
- `xOffset` / `yOffset` (number): Margin expanding the tracking box away from the edges (same as Pixel).

## Best practices

- Always provide a specific `key`. If your addon renders multiple glows to the same frame (e.g. tracking multiple auras), failure to provide a key will overwrite the `"Default"` bucket.
- Always pair a single `lib.Show` explicitly with a single `lib.Hide` when out of combat. This returns frames to the pool explicitly.
- `lib.PreLoad(glowType, count)` warms the shared pool by allocating then releasing `count` glows of a type against a hidden dummy frame. Call it during load/idle to absorb first-use frame/texture **allocation** (per-size geometry still computes on the first real `lib.Show`), so the first show in combat doesn't hitch. Engine types only (`Pixel`/`Thin`/`Thick`/`Medium`/`Autocast`/`Classic`) — registered glow-pack names are not poolable this way.
- Re-showing a live glow with the **same** options only re-tints (no teardown or animation restart); changing any geometry/timing/atlas option, or passing `force = true`, rebuilds it. So driving a glow from a per-event handler is cheap even without your own dedup.
