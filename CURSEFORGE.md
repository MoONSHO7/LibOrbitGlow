# LibOrbitGlow-1.0

A drop-in library for drawing animated **glows** on any WoW frame — action buttons, aura icons, cooldowns, nameplates, anything you can point it at.

It comes with **six distinct glows** built from Blizzard's own UI art, so they look native and most players already recognise them. It can also render **Glow Packs** like [Orbit Pack: Glows](https://www.curseforge.com/wow/addons/orbit-pack-glows) — addons that register their own animated glow atlases for the library to play. Developers add this library to their addon to display and customise glows with a single call, and to consume glow atlases — their own or a third party's — for distribution to their users.

The six built-in glows are **Pixel**, **Autocast**, **Classic**, **Thin**, **Thick**, and **Medium**. Every glow recolours to any RGBA you pass, is reused from a shared pool, throttles to 60fps, and is safe to drive from WoW 12.0 secret values.

Requires only **LibStub**. Retail 12.0+.

---

## Preview every glow: /orbitglow

Type **`/orbitglow`** in-game to open the built-in showcase — a movable grid of every registered glow, including any Glow Packs you have installed, grouped by source. Left-click a glow to preview it on your first action button, right-click to re-roll its colour.

---

## Basic usage

Show and hide a built-in glow:

```lua
local lib = LibStub("LibOrbitGlow-1.0", true)
if not lib then return end

lib.Show(frame, "Pixel", { key = "myGlow", color = { 0.2, 0.8, 1, 1 } })
-- ... later:
lib.Hide(frame, "Pixel", "myGlow")
```

Drive a glow from a saved setting — `Apply` / `Remove` accept an engine type **or** any glow a pack registered, so you never branch on which:

```lua
local id = mySettings.glow   -- "Pixel", "Medium", or a pack glow like "pinring"
lib.Apply(frame, id, { key = "proc", color = { 0.3, 0.8, 1, 1 } })
-- ... later:
lib.Remove(frame, id, "proc")
```

Populate a settings dropdown with everything available (built-ins plus installed packs):

```lua
for _, name in ipairs(lib:GetGlowList()) do
    -- name, and lib:GetGlowInfo(name).source for grouping by pack
end
```

---

## Documentation

Full API reference, the glow-pack format, engine options, combat / secret-value safety, and embedding & versioning notes are in the repository README:

**https://github.com/MoONSHO7/LibOrbitGlow**
