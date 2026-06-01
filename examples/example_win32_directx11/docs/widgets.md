# The Glass Widget Library (`src/glass/widgets/`)

This directory holds every reusable UI control in Glass Overlay. The files are **`.inc` fragments**, not independent translation units: they are `#include`d at the *bottom* of `src/glass/glass.cpp` (inside `namespace Glass`), so they all share one translation unit, one anonymous namespace of file-static helpers, and a single set of shared building blocks. There are no headers per widget — the public prototypes live in `src/glass/glass.h`.

```cpp
// src/glass/glass.cpp (tail)
#include "widgets/audio.inc"          // UI sound engine (PlaySfx)
#include "widgets/widgets_core.inc"   // Button, Toggle, Slider, DrawIcon, SmoothCol, rows, notifications
#include "widgets/widgets_input.inc"  // Segmented, Checkbox, Radio, Stepper, Chip, panels, Dropdown
#include "widgets/fields_color.inc"   // ColorButton, color-picker modal, text/search/password fields
#include "widgets/widgets_misc.inc"   // SidebarNav, ProgressRing, Badge, Avatar, Rating, Spinner, Disclosure
#include "widgets/modals.inc"         // Alert, ActionSheet, ContextMenu, CommandPalette
#include "widgets/widgets_extra.inc"  // TabBar, DynamicIsland, Scrubber, ActivityRings, Gauge, MorphButton, Calendar, OutlineTree
#include "widgets/chrome.inc"         // Banner, ShareSheet, GlassWidget, ControlCenter, Onboarding, TrafficLights, MenuBar, SwipeRow
```

---

## Include order — and why it is load-bearing

These are textual `#include`s, so a symbol must be **defined or forward-declared above the point it is used**. The order is not cosmetic:

| # | File | Why it is here |
|---|------|----------------|
| 1 | `audio.inc` | Defines `PlaySfx()` (and the `kSfx` spec table). Almost every interactive widget calls `PlaySfx()` on activation, so it must exist first. |
| 2 | `widgets_core.inc` | Defines the **shared widget vocabulary** the rest of the library leans on: `DrawIcon`, `SmoothCol`, `Button`, `Toggle`, `Slider`, the settings rows, and `Notify`/`DrawNotifications`. Everything below uses at least one of these. |
| 3 | `widgets_input.inc` | Form controls (Segmented, Checkbox, Radio, Stepper, Chip, IconButton, panels, Dropdown). Uses core + icons. |
| 4 | `fields_color.inc` | `ColorButton` + the full color-picker modal, and the text/search/password field family. Heavier controls built on core. |
| 5 | `widgets_misc.inc` | Grab-bag of display/indicator widgets **and the canonical `SubmitBlob` and `AccentCol` definitions** (both forward-declared in `glass.h`/core and finally defined here). |
| 6 | `modals.inc` | Overlay surfaces (Alert, ActionSheet, ContextMenu, CommandPalette) that draw on top of pages. |
| 7 | `widgets_extra.inc` | Fancy/novelty widgets (TabBar, DynamicIsland, Scrubber, ActivityRings, Gauge, MorphButton, Calendar, OutlineTree, Odometer…). |
| 8 | `chrome.inc` | App-shell chrome (Banner, ShareSheet, GlassWidget container, ControlCenter, Onboarding, TrafficLights, MenuBar, SwipeRow). |

> **Rule of thumb when adding a widget:** if it only needs `Button`/`Toggle`/`Slider`/`DrawIcon`/`SmoothCol`, drop it in `widgets_input` or later. If something else needs *it*, put it earlier (or add a forward declaration the way `widgets_core.inc` forward-declares `SmoothCol` at its top).

---

## File → widgets defined

### `audio.inc` — UI sound engine
Not visual widgets; the sound layer every control uses.

| Symbol | Purpose |
|--------|---------|
| `PlaySfx(Sfx)` | Synthesizes (once, lazily) and async-plays one of ~58 PCM16 blips via `winmm`/`PlaySoundA`; also sets the per-frame "consumed" flag. |
| `SetSfxEnabled` / `SfxEnabled` | Master mute. |
| `SfxConsumed` / `SfxResetConsumed` | Lets `main.cpp` suppress its global click fallback when a widget already played a sound this frame. |
| `kSfx[]`, `SfxBuild`, `SfxOsc` (file-static) | Spec table + oscillator/WAV builder; `static_assert`’d to match `enum class Sfx` order in `glass.h`. |

### `widgets_core.inc` — the shared vocabulary
| Symbol | Widget / role |
|--------|---------------|
| `Button(label, primary, size)` | Gel-squish glass button: bouncy scale spring, hover lift, touch ripple, eased label color. |
| `Toggle(label, bool*)` | Spring switch: spring-slid knob with velocity stretch + gooey metaball trail (`SubmitBlob`). |
| `Slider(label, float*, min, max, fmt)` | Pixel-accurate-while-dragging slider with lagging gooey thumb. |
| `SmoothCol(id, slot, target, style)` | **Per-channel eased `ImU32`** color glide (4 spring slots). Core helper used everywhere for hover/selection color. |
| `DrawIcon(dl, Icon, center, size, col, th)` | **The icon primitive.** Renders a FontAwesome glyph (with emboss shadow+highlight); falls back to procedural `ImDrawList` vector art per `Icon`. |
| `RowToggle` / `RowValue` | Settings list rows (icon + label + switch / disclosure value + chevron) with hover-lift plate. |
| `Notify` (no-op) / `DrawNotifications` | glassmorphic banner stack (currently `Notify` is intentionally a no-op). |

### `widgets_input.inc` — form controls
| Symbol | Widget |
|--------|--------|
| `SegmentedControl(id, labels, count, current)` | Sliding-pill segmented selector → returns new index. |
| `Checkbox(label, bool*)` | Animated check. |
| `RadioGroup(id, opts, count, current)` | Radio list → returns index. |
| `Stepper(label, int*, step, min, max)` | −/+ integer stepper. |
| `ProgressBar(frac, size)` | Determinate fill bar. |
| `Chip(label, selected)` | Filter chip / pill toggle. |
| `IconButton(id, Icon, size)` | Round glyph button. |
| `Keybind(label, int*)` | Captures a key and stores it. |
| `StatRow`, `SectionHeader`, `Divider` | Layout/text helpers. |
| `BeginPanel`/`EndPanel` (+ `PanelInfo`) | Glass sub-panel container (submits a backing primitive on `End`). |
| `Dropdown(label, items, count, int*)` | Popup select. |

### `fields_color.inc` — color picker + text fields
| Symbol | Widget |
|--------|--------|
| `ColorButton(label, rgb[3])` | Swatch button that opens the picker modal. |
| `DrawColorPickerModal()` | Full HSV/wheel color-picker overlay (large). |
| `TextCore` (file-static) | Shared text-input core; tracks `g_lastFieldFocused`. |
| `FieldWasFocused()` | Query last field's focus. |
| `TextField` / `SearchField` / `PasswordField` | Public field variants. |
| `FieldRaw(...)` | The configurable underlying field (leading icon, password mask, custom height). |

### `widgets_misc.inc` — indicators + canonical primitives
| Symbol | Widget / role |
|--------|---------------|
| `SubmitBlob(...)` | **Canonical definition** of the metaball helper (two rounded-rects → smooth-min blob). Forward-declared in `glass.h`; used by `Toggle`/`Slider`/`RowToggle` above. |
| `AccentCol()` (file-static) | Reads `Params(Material::Accent).tint_rgb` → `ImU32`. The system-tint color used throughout. |
| `SidebarNav(id, icons, labels, …)` | Vertical nav rail → returns selection. |
| `ProgressRing(id, frac, radius, center)` | Circular progress with accent arc. |
| `Badge`, `Avatar`, `StatusPill` | Small content chips. |
| `Rating(id, stars, max)` | Star rating input. |
| `VSlider(id, float*, min, max, size)` | Vertical slider. |
| `SegmentedIcons(id, icons, count, current)` | Icon-only segmented control. |
| `Tooltip(text)`, `SeparatorText(text)` | Helpers. |
| `NumberDrag(label, float*, speed, …)` | Drag-to-scrub numeric. |
| `Spinner(id, radius, col)` | Indeterminate rotating spinner. |
| `BeginDisclosure`/`EndDisclosure` (+ `DiscInfo`) | Collapsible section with backing glass plate. |

### `modals.inc` — overlay surfaces
Stateful, two-call pattern: an `…Open()` sets module-static state, a `Draw…()` renders it each frame and returns the chosen index (−1 = none yet).

| Symbol | Widget |
|--------|--------|
| `AlertOpen` / `DrawAlert` | Alert dialog (optional destructive style), scrim via `ModalScrim`. |
| `ActionSheetOpen` / `DrawActionSheet` | Bottom action sheet. |
| `ContextMenuOpen` / `DrawContextMenu` | Right-click context menu at (x,y). |
| `CommandPaletteToggle` / `CommandPaletteOpen` / `DrawCommandPalette` | ⌘K command palette with fuzzy `ContainsCI` filter. |

### `widgets_extra.inc` — novelty / rich widgets
| Symbol | Widget |
|--------|--------|
| `TabBar(id, icons, labels, count, current)` | Bottom tab bar. |
| `ControlTile(Icon, label, bool*, size)` | Control-Center-style toggle tile. |
| `DrumPicker(id, items, count, current, size)` | Scrolling drum/wheel picker. |
| `DynamicIsland(...)` | Pill that expands at the top. |
| `Scrubber(id, float*, min, max, width)` | Media scrubber. |
| `Skeleton`, `AttentionRing`, `LevelIndicator`, `PageDots` | Misc indicators. |
| `ActivityRings(id, fracs, cols, count, radius)` | Concentric fitness activity rings. |
| `Gauge(id, v, min, max, radius, fmt)` | Arc gauge → returns value. |
| `MorphButton(id, MorphIcon, bool*, size)` | Glyph that morphs (PlayPause / PlusClose / MenuClose). |
| `Odometer`, `SuccessHUD` | Animated counter / success checkmark HUD. |
| `TokenField`, `ComboBox` | Token entry / editable combo. |
| `Calendar(id, day, month, year)` (+ `CalLeap`/`CalDaysInMonth`/`CalDow`) | Month calendar picker. |
| `OutlineTree(id, labels, depth, icons, count, current)` | Indented tree list. |

### `chrome.inc` — app-shell chrome
| Symbol | Widget |
|--------|--------|
| `BannerOpen` / `DrawBanner` | Top notification banner. |
| `ShareSheetOpen` / `DrawShareSheet` | Share sheet grid. |
| `GlassWidget(id, title, size)` / `EndGlassWidget` (+ `WInfo`) | Free-floating glass widget container. |
| `ControlCenterToggle` / `ControlCenterOpen` / `DrawControlCenter` | Pull-down Control Center (wifi/bt/airplane/focus/brightness/volume). |
| `Onboarding(...)` | Paged onboarding flow. |
| `TrafficLights(id, x, y)` | Window close/min/zoom buttons. |
| `MenuBar(id, menus, count)` | Top menu bar. |
| `CollapsingTitle(text, scrollY)` | Large-title-that-shrinks-on-scroll. |
| `SwipeRow(id, label, actions, cols, count)` | Swipe-to-reveal-actions row. |

---

## Shared building blocks

Every widget is composed from the same small toolkit. Understanding these five is enough to read any `.inc` file.

### 1. `Primitive` + `g->Submit` / `g->At` / `SubmitOverlay`
A widget never draws glass directly — it **submits a `Primitive`** to the global `Glass::Renderer* g`, which batches them and runs the glass pixel shader. The struct (in `glass.h`):

```cpp
struct Primitive {
    float cx, cy;            // window-local center px
    float hw, hh;            // half size px
    float corner_radius;     // px
    float fade;              // 0..1 alpha
    float elevate = 1.0f;    // shadow lift (hover)
    float clip[4];           // scissor rect
    Material material = Material::Regular;
    float b1x,b1y,b1w,b1h, b2x,b2y,b2w,b2h, blob_k = 0; // optional metaball
};
```

- `size_t Submit(const Primitive&)` queues a blob (returns its index).
- `Primitive& At(size_t)` retrieves a queued slot to patch later — used by `BeginCard/EndCard` and panels that reserve a backing plate *before* they know their final size.
- `SubmitOverlay(const Primitive&)` queues a blob drawn after everything (popups/notifications).

### 2. `SubmitBlob` — metaballs (the "liquid" merge)
Defined in `widgets_misc.inc`, declared in `glass.h`. Takes **two** rounded-rects and a smooth-min strength `k`; the shader blends them into one gooey shape. This is what makes a toggle knob *stretch and trail* while sliding, instead of teleporting. The helper sizes the draw quad to cover both shapes and fills `b1*`/`b2*`/`blob_k` on the `Primitive`.

```cpp
SubmitBlob(headX, y, hw, hh,  tailX, y, hw*0.82f, hh*0.82f,  min(hw,hh), 9.0f, 1.0f, Material::Knob);
```

### 3. `DrawIcon` + `Icon` (FontAwesome glyphs)
`DrawIcon(dl, Icon, center, size, col, th)` is the icon primitive (in `widgets_core.inc`). It prefers a real **FontAwesome 6** glyph (via `IconCodepoint` / `IconIsBrand` and the fonts set by `SetIconFont`/`SetIconFontBrands`), rendering it with a soft drop shadow + top highlight ("glass emboss"). If no font is loaded it falls back to hand-drawn `ImDrawList` vector art per `Icon` enum value. **Callers pass RAW sizes** — `DrawIcon` multiplies by `g_ui` internally (see §5).

### 4. `Material` + `Params` + `AccentCol`
`enum class Material { Thin, Regular, Thick, Accent, Knob, Clear }` chooses the frost recipe (`MaterialParams`: blur, vibrancy, refraction, sheen, shadow, etc.) the shader applies to a `Primitive`. Convention:

| Material | Used for |
|----------|----------|
| `Thin` | tracks, hover plates, secondary buttons |
| `Regular` | default cards |
| `Thick` | modal panels, banners |
| `Accent` | system-tint fills (toggle-on, slider fill, primary button) |
| `Knob` | bright control lens (toggle/slider thumb) |

`AccentCol()` (`widgets_misc.inc`) reads `Params(Material::Accent).tint_rgb` so accent *line/text* art matches the accent *fill*.

### 5. Animation: `springs()` + `SmoothCol`, and `Dt()`
All motion is physical, via the spring system in `spring.h`:

- `g->springs().Get(imgui_id, slot, SpringStyle, initial) -> Spring&` returns a persistent critically-damped/bouncy/overdamped spring. The cache key is `(imgui_id << 32) | slot`, so **a widget gets independent animation state without storing anything itself** — the only retained state lives in this cache (and occasionally `ImGui::GetStateStorage()`), keyed by ImGui id.
- `sp.target = …; sp.Tick(Dt());` each frame; read `sp.x`. `Dt()` is `ImGui::GetIO().DeltaTime`.
- `SmoothCol(id, slot, targetColor)` (4 slots) is the color analog: it springs each RGBA channel toward the target so hover/selection colors glide instead of snapping.

### 6. Scaling: `US()` and `g_ui`
`g_ui` is the global widget pixel-scale (`SetWidgetScale`/`WidgetScale`, clamped 0.25–4.0). `US(v)` = `v * g_ui`. **Widgets express every pixel size through `US(...)`** so the whole UI scales uniformly. Caveat: `DrawIcon` applies `g_ui` *internally*, so pass it raw sizes (do **not** wrap icon sizes in `US()` or you double-scale).

---

## Programming model: pure immediate mode

These widgets are **pure immediate-mode**. Each call, every frame:
1. carves a hit-rect (`ImGui::InvisibleButton` / `Dummy`),
2. reads interaction state (`IsItemHovered/Active/Activated/Deactivated`),
3. plays a sound (`PlaySfx`) on activation,
4. drives springs (`springs().Get(...).Tick(Dt())`) and colors (`SmoothCol`),
5. submits glass `Primitive`s (`g->Submit` / `SubmitBlob`) and draws text/vector art on the ImGui draw list.

**No widget owns retained state.** Persistent animation/visual state lives only in the spring cache and `ImGui::GetStateStorage()`, keyed by the ImGui id — which is why `PushID`/unique labels matter. The stateful *modal* surfaces (`modals.inc`, parts of `chrome.inc`) are the exception: they keep module-static open/contents state and use the `…Open()` setter + `Draw…()` per-frame-renderer split.
