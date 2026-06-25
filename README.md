# LibOrbitGlow-1.0

A drop-in World of Warcraft library for drawing animated **glows** on any frame — action buttons, aura icons, cooldowns, nameplates, anything you can point it at.

It comes with **six distinct glows** built from Blizzard's own UI art, so they look native and most players already recognise them. It can also render **Glow Packs** like [Orbit Pack: Glows](https://www.curseforge.com/wow/addons/orbit-pack-glows) — addons that register their own animated glow atlases for the library to play. Developers add this library to their addon to display and customise glows with a single call, and to consume glow atlases (their own or a third party's) for distribution to their users.

> Targets retail **12.0+**. Requires only **LibStub** — no other addon dependencies.

---

## How it works

LibOrbitGlow is split into an **engine** and a **registry**:

- The **engine** draws the six built-in glows and provides pooling, recolouring, throttling, and secret-value safety.
- The **registry** is an open list of glows that **Glow Packs** populate at load. The library owns no pack art itself — a pack calls `RegisterGlow` for each of its glows, and from then on every addon embedding LibOrbitGlow can play them.

The upshot: a host addon writes its glow code **once**, and any Glow Pack the user installs automatically appears in that addon's glow options with **zero extra code** on the host's side.

---

## The built-in glows

Six glows, each rendered from native Blizzard assets so they match the game's look:

| Glow | What it looks like |
|---|---|
| `Pixel` | Crisp pixel lines tracing the frame's border |
| `Autocast` | The pet-bar autocast shimmer — rotating shine squares |
| `Classic` | The classic spell-activation flash and ant swirl |
| `Thin` | A thin swirling ring of ants |
| `Thick` | A thick proc-loop ring |
| `Medium` | The standard action-bar proc glow |

Every glow recolours to any RGBA you pass, is reused from a shared frame/texture pool, throttles its animation to 60fps, and is safe to drive from WoW 12.0 secret values.

## Glow Packs

Richer, hand-animated glows come from **packs** — small addons that register their atlases with this library. The flagship is [**Orbit Pack: Glows**](https://www.curseforge.com/wow/addons/orbit-pack-glows): a library of animated proc / pandemic glows. Install a pack and it shows up automatically anywhere LibOrbitGlow is used. Developers can ship their own packs too — see [Build a glow pack](#build-a-glow-pack).

---

## Installation

**Players:** install it from CurseForge. It loads on its own, gives you a `/orbitglow` showcase to preview every glow (including any installed packs), and any addon that embeds it uses it automatically.

**Developers (embedding):** drop the `LibOrbitGlow-1.0/` folder into your addon and include its manifest from your `.toc` or XML — it loads the engine then the optional showcase:

```xml
<Include file="Libs\LibOrbitGlow-1.0\LibOrbitGlow-1.0.xml"/>
```

Then consume it via LibStub:

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
if not lib then return end
```

(Delete the `GlowShowcase.lua` line from that XML to embed the engine without the demo panel and slash command.)

---

## Usage

The library exposes a simplified facade that abstracts away the underlying animation groups, textures, and geometry engines. You only ask for a glow and pass an options table.

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

Hide a glow using the exact same type and key you showed it with, so the library can gracefully stop the animation and recycle the frame back into the pool.

```lua
lib.Hide(frame, "Pixel", "myComponentGlow")
```

### One call for any glow (recommended)

`lib.Show` / `lib.Hide` reach the built-in **engine** types directly. But if your addon stores a glow choice as a setting, that value might be an engine type (`"Pixel"`) *or* the name of a glow a pack registered (`"pinring"`) — and you shouldn't have to branch on which. `lib.Apply` / `lib.Remove` resolve either:

```lua
local id = mySettings.glow   -- "Pixel", "Medium", or any registered pack glow like "pinring"
lib.Apply(frame, id, { key = "proc", color = { 0.3, 0.8, 1, 1 } })
-- ... later:
lib.Remove(frame, id, "proc")

-- A continuous glow that stays until you Remove it (no intro; the outro still plays on Remove) -- add loop = true:
lib.Apply(frame, id, { key = "buff", color = { 1, 0.8, 0.2, 1 }, loop = true })
```

A registered name plays its full proc lifecycle (one-shot start, looping body, one-shot end on `Remove`); `loop = true` plays the loop continuously with no intro (call `Proc:Clear` instead of `Remove` for an instant cut). An engine type shows and hides directly. `Remove` accepts either a bare key or the same options table you passed to `Apply`. Use `Apply` / `Remove` everywhere you drive a glow from user choice.

> **Call style:** the top-level verbs are **dot**-called — `lib.Show`, `lib.Hide`, `lib.Apply`, `lib.Remove`, `lib.PreLoad`. The registry and the `Proc` / engine sub-namespaces are **colon** (method) calls — `lib:RegisterGlow(...)`, `lib.Proc:Start(...)`, `lib.Pixel:Show(...)`.

### Combat and secret-value safety

When tracking auras, cooldowns, or power states that return WoW 12.0 *secret values*, you cannot branch Lua logic on those values — so you cannot conditionally call `lib.Hide()` when an aura drops during combat.

Instead, drive visibility through the glow's alpha with a plain (non-secret) number, or through a C++ sink on the parent. Note that `x and 1 or 0` on a secret boolean is itself a Lua-side branch and will throw.

```lua
-- Safe: alpha is a plain number from a non-secret read (e.g. a numeric curve)
lib.Show(frame, "Pixel", { color = { 1, 0, 0, alpha } })

-- Or derive visibility from a secret boolean via a C++ sink on the parent:
parent:SetAlphaFromBoolean(secretBool, 1.0, 0.0)
```

---

## Proc lifecycle (registry glows)

Registered glows (the baselines and anything a pack adds) play through `lib.Proc`. You name the glow and the icon's corner shape; the library never decides the shape — your host maps its own border style to a shape the glow's `def.shapes` provides, defaulting to `"square"`.

| Method | Plays |
|---|---|
| `lib.Proc:Start(frame, o)` | one-shot **start**, then the looping **body** (skips straight to the loop if the glow has no start phase) |
| `lib.Proc:Stop(frame, o)` | one-shot **end**, then release (skips straight to release if the glow has no end phase) |
| `lib.Proc:Loop(frame, o)` | the **loop** only, forever — for persistent glows that never "expire" |
| `lib.Proc:Clear(frame, o)` | stop immediately, no end animation |

```lua
lib.Proc:Start(frame, { glow = "pinring", shape = "square", color = {0.3,0.8,1,1}, key = "proc" })
-- ... later, when the proc expires:
lib.Proc:Stop(frame, { glow = "pinring", shape = "square", key = "proc" })
```

`o` accepts `glow`, `shape`, `color`, `key`, `frameLevel`, `scale`, `padding`, `offsetScale` / `offsetX` / `offsetY` (nudge the glow off-centre), and per-phase durations `startDuration` (0.28), `loopDuration` (1.0), `endDuration` (0.20) seconds. Most callers should prefer the higher-level `lib.Apply` / `lib.Remove`, which route engine types **and** registered names.

**Registry API** — `lib:RegisterGlow(name, def)` (returns `false` and warns on a malformed def) · `lib:UnregisterGlow(name)` · `lib:IsGlowRegistered(name)` · `lib:GetGlowInfo(name)` · `lib:GetGlowList()` (sorted names — populate UI dropdowns from this; `GetGlowInfo(name).source` groups them by pack) · `lib:GetResolvedPaths(name [, shape])` (the texture paths a glow will try, for debugging).

A baseline `"blizzard"` glow is always registered, so `Proc` works with no pack installed, and any unknown glow name falls back to it.

---

## Build a glow pack

A pack is just an addon that calls `lib:RegisterGlow` for each of its glows during load — no dependency on any host, only on this library being present. The `def` table describes where the art lives and how to play it. **One** of `path` / `resolve` / `atlas` / `engine` is required; everything else is optional.

| Field | Type | Default | Purpose |
|---|---|---|---|
| `path` | `string` | — | File-prefix for a flipbook sheet; resolves to `"<path>-<phase>[-<shape>]<layer><ext>"` |
| `resolve` | `function(phase, shape, layer)` | — | Full override of path resolution — use any naming you like; return `nil` for a phase/layer you don't ship |
| `atlas` | `string` | — | A Blizzard **atlas name** (single layer, `SetAtlas`) instead of files |
| `engine` | `string` | — | Delegate to a built-in engine (`"Pixel"`, `"Autocast"`, `"Classic"`, `"Thin"`, `"Thick"`, `"Medium"`) |
| `layered` | `boolean` | `false` | Draw a tinted **BLEND body** + a near-white **ADD core** (depth). `false` = one tinted layer |
| `core` | `boolean` | `true` | When `layered`, include the `-core` layer (set `false` for a body-only layered glow) |
| `blendMode` | `string` | `"ADD"` | Blend for a single-layer `path` / `atlas` def |
| `bodyBlend` / `coreBlend` | `string` | `"BLEND"` / `"ADD"` | Per-layer blend overrides for a `layered` def |
| `phases` | `table` | all | Set of phases the art provides, e.g. `{ loop = true }` |
| `loopOnly` | `boolean` | `false` | Shorthand for `phases = { loop = true }` |
| `shaped` | `boolean` | `true` | Include the `-<shape>` segment in the default name (set `false` if your files aren't shape-specific) |
| `ext` | `string` | `".tga"` | File extension for `path` resolution |
| `rows` / `cols` / `frames` | `number` | `6` / `5` / `30` | Flipbook grid (Blizzard's 30-frame, 5-wide × 6-tall layout) |
| `shapes` | `table` | `{ square = true }` | Corner shapes your art ships, e.g. `{ square = true, round = true }` |
| `scale` | `number` | engine default | Default size multiplier (path / atlas defs). For an `engine` def, sizing goes through `options` |
| `desaturated` | `boolean` | `true` | (`atlas` defs) desaturate before tinting |
| `options` | `table` | — | (`engine` defs) extra engine options (`lines`, `length`, `particles`, …) |
| `source` | `string` | `"Unknown"` | Group label for the showcase / picker — use your pack name |

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
if not lib then return end
local ROOT = "Interface\\AddOns\\MyGlowPack\\Textures\\"

-- 1. Full layered glow: start/loop/end x body+core, shape-suffixed.
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

The flipbook grid (`rows` / `cols` / `frames`) describes how the sheet is sliced — one frame per cell over `loopDuration` seconds (`.tga`, any cell size; Orbit Pack: Glows uses 128×128 cells on a 640×768 sheet). **Layered / `path` art must be grayscale** — the layers are *tinted* by your `color` at draw time (not desaturated), so coloured source art multiplies muddily; the body takes your colour and the core is auto-brightened toward white for depth.

The host discovers your glows purely through `GetGlowList()` — no host-side code change is needed to surface a new pack. If a glow renders blank, call `lib:GetResolvedPaths(name [, shape])` to get the exact paths the lib is trying (WoW's `SetTexture` fails **silently** on a missing file), or set `lib.DEBUG = true` to print each path as it plays. A loop-only pack must declare `loopOnly = true` (or `phases`), or the lib will try to play `-start` / `-end` art it assumes exists.

### Lower-level flipbook

`lib.Flipbook:Show(frame, opts)` is the raw sink the registry feeds. It accepts `atlas` (a Blizzard atlas name, or a file path with `isTexture = true`), `rows` / `cols` / `frames`, `speed`, `blendMode`, `color`, `key`, plus `once = true` + `onFinished` for a single one-shot. Use it directly only if the registry / `Proc` model doesn't fit.

---

## Showcase

The library bundles a self-contained visual showcase — a movable, scrollable grid of every registered glow, grouped into collapsible sections by source pack. Left-click a glow to apply it to `ActionButton1`, right-click to re-roll colours, collapse a section with its header arrow. It depends only on LibOrbitGlow and the Blizzard UI, so it travels with the library and works in any host.

```lua
LibStub("LibOrbitGlow-1.0").Showcase:Toggle()   -- also :Show() / :Hide()
```

A `/orbitglow` slash command toggles it (guarded — if the command is already taken, the library leaves it alone and you drive the showcase via `Showcase:Toggle()`).

---

## Reference

### Global options

Passed as the 3rd argument to `lib.Show(frame, glowType, options)`, honoured across almost all engines.

| Field | Type | Description |
|---|---|---|
| `key` | `string` | Unique id used for tracking and hiding. Default: `"Default"` |
| `color` | `table` | `{ r, g, b, a }` array or a 12.0 `Color` object. Default: white |
| `frameLevel` | `number` | Relative frame level above the parent. Default: `8` |
| `desaturated` | `boolean` | Atlas engines (`Thin` / `Thick` / `Medium`) desaturate before tinting; pass `false` to keep native colours. Ignored by `Pixel` / `Autocast`. Default: `true` |
| `force` | `boolean` | On a re-show of a live glow, rebuild even when options are unchanged (defeats the re-tint fast path). Honoured by all engines. Default: `false` |

### Flipbook engines (`Thin`, `Thick`, `Medium`)

- `scale` (number): multiplier applied to width and height. Default: `1.4`.

### Pixel engine (`Pixel`)

- `lines` (number): number of tracing particles. Default: `8`
- `frequency` (number): speed scaler. Positive = faster than baseline (`period = 0.25 / frequency`); `0` or unset = baseline (4s); negative = slower (`period = baseline * (1 + |frequency| * 8)`).
- `thickness` (number): width of the tracing particles. Default: `2`
- `length` (number): arc length of each dash. Default: auto-derived from the frame perimeter and `lines`.
- `border` (boolean): a dark translucent backdrop strictly inside the pixel bounds, for contrast. Default: `true`
- `xOffset` / `yOffset` (number): margin expanding the tracking box away from the edges.
- `pixelScale` (number): physical-pixel size used to snap the trace to the device grid (`pixelScale / frame:GetEffectiveScale()`). Default `1`. Pass the host's screen scale for crisp lines at any UI scale.

### Autocast engine (`Autocast`)

- `particles` (number): number of points mapping the border. Default: `4`
- `frequency` (number): rotation speed scaler. Same sign convention as Pixel — positive = faster (`period = 1 / frequency`); `0` or unset = baseline (8s); negative = slower.
- `scale` (number): scaling scalar on each particle. Default: `1`
- `xOffset` / `yOffset` (number): margin expanding the tracking box away from the edges.

### Best practices

- Always provide a specific `key`. If you render multiple glows to the same frame (e.g. tracking several auras), omitting the key overwrites the shared `"Default"` bucket.
- Pair a single `lib.Show` with a single `lib.Hide` (same type + key) so frames return to the pool.
- `lib.PreLoad(glowType, count)` warms the shared pool against a hidden dummy frame so the first show in combat doesn't hitch. It amortises object **allocation** (per-size geometry still computes on the first real show). Engine types only — registered pack names are not poolable this way.
- Re-showing a live glow with the **same** options only re-tints (no teardown, no animation restart); changing any geometry / timing / atlas option, or passing `force = true`, rebuilds it. Driving a glow from a per-event handler is therefore cheap even without your own dedup.

---

## Embedding & versioning

Targets **retail 12.0+** (the showcase uses retail-only `ScrollUtil` / `MinimalScrollBar`; the core engine uses `CreateMaskTexture` / `CreateFramePool`). LibStub hands every embedder whichever copy of the library loaded **first** (highest minor wins), so feature-probe rather than assume when you rely on a newer method — a co-installed older copy may not have it:

```lua
if lib.Apply then lib.Apply(frame, id, opts) else lib.Show(frame, id, opts) end
```

---

## License

MIT. See [LICENSE](LICENSE).
