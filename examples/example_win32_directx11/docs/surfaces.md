# Surfaces — the 118 demo "mini-apps"

The **surfaces** subsystem is the gallery of demo app screens (Weather, Music, Calculator, Settings, Maps, …) that the overlay shows off. Each one is a self-contained mini-app rendered into the scrolling content child. They exist to exercise the Liquid-Glass widget kit against realistic glassmorphic layouts.

This subsystem lives entirely in:

| File | Role |
|------|------|
| `src/glass/surfaces.h` | Public API (`SurfaceCount`, `SurfaceName`, `SurfaceDraw`, `SurfaceLaunchpad`, `SurfaceDock`) |
| `src/glass/surfaces.cpp` | Shared helpers, the `g_surfaces[]` registry, the app grid, the bottom dock, and the public accessors |
| `src/glass/surfaces/part1..6.inc` | The actual `Draw_X` bodies, `#include`d into `surfaces.cpp` in order |

`surfaces.cpp` is one of the four real translation units in the unity build; `part1..6.inc` are fragments pulled in via `#include` near the end of the file (after the shared helpers are defined, before the registry that references the draw functions).

---

## 1. Each surface is a `static void Draw_X(float w)`

Every surface is a free function with one signature:

```cpp
static void Draw_Weather(float w);   // w = available content width (px)
```

It draws into `ImGui::GetWindowDrawList()` and advances ImGui layout with `ImGui::Dummy(...)`. There is no shared base class and no per-surface struct — **all demo state is held in function-`static` variables** inside the body (e.g. Calculator's `static double acc`, Weather's `static City cities[4]`). That keeps `main.cpp` and the registry tiny: a surface is just a function pointer plus four pieces of metadata.

To add a surface you write one `Draw_X` in a `part*.inc` and add one row to `g_surfaces[]`. No `main.cpp` change is required.

### How a surface is reached

Surfaces are not real nav pages. The dashboard nav uses a page integer; surfaces occupy the range `>= 100`:

```cpp
// src/app/pages.inc
else if (dispPage == 7) {                 // Launchpad / apps gallery (off-nav page 7)
    int sel = Glass::SurfaceLaunchpad(contentW, searchbuf);
    if (sel >= 0) page = 100 + sel;       // open surface `sel`
}
else if (dispPage >= 100) {               // an open surface
    if (Glass::Button("‹  Apps", false)) page = 7;
    Glass::SurfaceDraw(dispPage - 100, contentW);   // page index → surface index
}
```

So **`page = 100 + surfaceIndex`**. The Launchpad (page 7) and the Dock both return a surface index, which the caller turns into a page.

---

## 2. The `g_surfaces[]` registry

The registry is the single source of truth that maps an index to a surface:

```cpp
struct SurfaceDef { const char* name; const char* sub; Icon icon; ImU32 tint; void(*draw)(float); };
static SurfaceDef g_surfaces[] = {
    { "Weather", "Today & 10-day", Icon::CloudSun, IM_COL32(64,156,255,255), Draw_Weather },
    { "Music",   "Now Playing",    Icon::Music,    IM_COL32(255,55,95,255),  Draw_Music   },
    ...
};
```

| Field | Meaning |
|-------|---------|
| `name` | Title (Launchpad label, Dock hover label) |
| `sub` | Subtitle (shown in Launchpad search; group rows use ` · `-joined member names) |
| `icon` | `Glass::Icon` enum value drawn by `DrawIcon` |
| `tint` | Accent color for the app's tile/branding |
| `draw` | The `Draw_X` function pointer |

There are **118 rows**: the first **107** are individual app surfaces; the **last 11** are *merged group shells* (see §4). The public accessors are thin index-bounded lookups:

```cpp
int   SurfaceCount(){ return (int)(sizeof(g_surfaces)/sizeof(g_surfaces[0])); }   // 118
void  SurfaceDraw(int i, float w){ if (i>=0 && i<SurfaceCount() && g_surfaces[i].draw) g_surfaces[i].draw(w); }
```

`SurfaceName/Subtitle/Icon/Tint` are the same shape, returning a safe default on out-of-range.

---

## 3. POSITIONAL dispatch — the index *is* the identity

Dispatch is purely positional: **`SurfaceDraw(i)` calls `g_surfaces[i].draw`**, and the nav page is `100 + i`. There is no name lookup, no ID enum — a surface is identified *only* by its array index. This is what makes adding a surface a one-line registry edit, but it is also the root of the fragility described next.

Two consumers turn a click into an index:

- **`SurfaceLaunchpad(w, query)`** — draws the app grid (page 7), filters by `query` against name+subtitle (`ContainsCI`, case-insensitive), animates per-tile hover/press springs, and returns the clicked surface index or `-1`. It iterates over *all* 118 rows including the group shells.
- **`SurfaceDock(cx, bottomY, maxW, activeSurface)`** — the bottom dock with Gaussian hover magnification. It does **not** show all surfaces; it shows only the 11 group shells (see below) and returns the clicked surface index.

---

## 4. The last 11 rows are merged GROUP shells

The trailing 11 rows are not leaf apps. Each is a **tabbed shell** that composes several existing surfaces behind an animated mini-tab bar (a sliding glass pill). They are what the Dock launches — e.g. tapping "Media" opens a shell that tabs between Music / Podcasts / TV / Radio / Shazam.

```cpp
// part6.inc
static void Draw_GroupMedia(float w){
    static int t=0; if(t<0||t>=5)t=0;
    t = DrawMiniTabs("gmedia", GMEDIA_N, GMEDIA_I, 5, t, w);
    SurfaceDraw(GMEDIA_M[t], w);          // delegate to the selected member surface
}
```

A group shell holds no content of its own; `DrawMiniTabs` renders the tab bar and `SurfaceDraw(member, w)` re-renders an existing surface. Nothing is duplicated, just re-composed.

### The Dock derives the group block by computation, not magic numbers

The Dock must not hardcode `107..117`. Instead the boundary is computed from the table size and protected by `static_assert`:

```cpp
static constexpr int kSurfaceTotal = (int)(sizeof(g_surfaces)/sizeof(g_surfaces[0]));   // 118
static constexpr int kGroupCount   = 11;                       // Media…Create
static constexpr int kGroupBase    = kSurfaceTotal - kGroupCount;   // 107
static_assert(kSurfaceTotal > kGroupCount, "...");
static_assert(kGroupBase >= 0, "...");
```

```cpp
int dk[kGroupCount];
for (int i = 0; i < kGroupCount; ++i) dk[i] = kGroupBase + i;   // Media…Create
```

**Invariant:** the 11 group shells MUST remain the *last* `kGroupCount` rows of `g_surfaces`. Inserting new app rows before the group block shifts everything automatically, and the Dock follows correctly. Adding a new group means appending it inside that trailing block *and* bumping `kGroupCount`.

---

## 5. ⚠️ FRAGILITY — group member tables hardcode `g_surfaces` indices

This is the most important thing to understand before touching this file.

While `kGroupBase` shields the *Dock* from reordering, the **group member tables in `part6.inc` hardcode raw `g_surfaces` indices**, and nothing protects them:

```cpp
// part6.inc — these integers ARE g_surfaces positions
static const int GMEDIA_M[] = { 1, 21, 38, 97, 54 };   // Music, Podcasts, TV, Radio, Shazam
static const int GWEB_M[]   = { 42, 8, 65 };           // Browser, Maps, Transit
static const int GTOOL_M[]  = { 106, 104, 102, 100, 101 };
static const int GTDY_M[]   = { 0, 2, 6, 27, 40 };     // Weather, Clock, Stocks, News, Widgets
// ...one such array per group
```

These indices are matched only by a hand-written comment. If you **insert, delete, or reorder any row in `g_surfaces`**, every one of these arrays silently points at the wrong surface — the build still compiles, the asserts still pass, and the Media tab quietly opens (say) Calculator instead of Podcasts. **There is no compile-time check tying these literals to the named surfaces.**

The same positional coupling exists implicitly anywhere a literal page/index is used, but `part6.inc` is the dense, easy-to-break concentration of it.

### Recommended future fix: `enum class SurfaceId`

Replace the bare integers with a named enum and index `g_surfaces` by it (or build the registry order *from* the enum). A sketch:

```cpp
enum class SurfaceId : int {
    Weather = 0, Music, Clock, Settings, /* ... */ Calculator,
    GroupMedia, GroupWeb, /* ... */ GroupCreate, Count
};
static const SurfaceId GMEDIA_M[] = { SurfaceId::Music, SurfaceId::Podcasts, SurfaceId::TV, SurfaceId::Radio, SurfaceId::Shazam };
```

With an enum, reordering the registry is a rename the compiler enforces, and the group tables read by intent rather than by magic number. A `static_assert(g_surfaces[(int)SurfaceId::Music].draw == &Draw_Music)`-style guard could even pin the mapping at build time. This is the single highest-value refactor in this subsystem.

---

## 6. Shared helpers (top of `surfaces.cpp`)

These file-local helpers are defined before the `.inc` includes, so every `Draw_X` can use them. (They are private copies — `glass.cpp`'s file-static equivalents are not visible across translation units.)

| Helper | Purpose |
|--------|---------|
| `TextC(dl, font, sz, center, col, s)` | Center-anchored text. Multiplies `sz` by `Glass::WidgetScale()` — **callers pass RAW sizes**, never pre-scaled. |
| `Accent()` | Current accent color, from `Params(Material::Accent).tint_rgb`. |
| `SA(c, a)` | Scale a color's alpha by `a` (0..1). |
| `kInk` / `kInkSoft` | Macros → `InkColor()` / `InkSoftColor()` (theme-aware primary/secondary text). |
| `FmtClock` | Clock string respecting the global `Use24Hour()` setting. |
| `RoundedGrad` / `Bilerp4` | Rounded-rect with a true 4-corner gradient (ImGui's built-in multicolor rect can't round). |
| `ImgOrGrad` | Draw a real photo via `PhotoTex(idx)`, else fall back to `RoundedGrad`. |
| `CalcKey` | Calculator-style spring-animated round key button. |
| `TileOn` | Smoothly fade a toggle tile's on-state via a `Critical` spring. |
| `SurfaceHit` | Drop an `InvisibleButton` over an already-drawn rect, then restore the cursor so the caller's `Dummy` still drives layout. Returns release-click; optional hover out. |
| `RowWash` | Subtle theme-aware hover/selection wash over a row. |
| `ContainsCI` | Case-insensitive substring match (Launchpad search). |

### The column-centering boilerplate

Most surfaces clamp their content to a max width and center it. The recurring idiom is a `colX()` lambda plus a clamped width:

```cpp
float cw = w < 720.0f ? w : 720.0f;
auto colX = [&](){ return ImGui::GetCursorScreenPos().x + (w - cw)*0.5f; };
```

Every block then draws from `colX()` and advances layout with `ImGui::Dummy(ImVec2(w, blockH))`. This pattern repeats across nearly all surfaces; it is a candidate for a shared `BeginCenteredColumn(w, maxW)` helper.

---

## 7. `part1..6.inc` are arbitrary line-count splits

The six fragments are **not thematic** — they were split purely to keep individual files at a manageable size. Counts:

| File | `Draw_X` defs | Notes |
|------|---------------|-------|
| `part1.inc` | 7 | Calculator, Weather, … |
| `part2.inc` | 18 | |
| `part3.inc` | 19 | |
| `part4.inc` | 20 | |
| `part5.inc` | 43 | (largest) |
| `part6.inc` | 11 | the GROUP shells + `DrawMiniTabs` + group member tables |
| **total** | **118** | 107 app surfaces + 11 group shells |

Because the split is by line count, related surfaces (e.g. the Music family) are scattered across files, and the registry order in `surfaces.cpp` does not match the file order. A **future thematic regroup** (one file per domain: Media, System, Health, Productivity, …) would make the code easier to navigate — and would pair naturally with the `SurfaceId` enum refactor, since the two together remove the last reasons the current positional layout is load-bearing.

---

## Quick reference

- Add a surface: write `static void Draw_X(float)` in any `part*.inc`; add a row to `g_surfaces[]`. If it's a leaf app, add it **before** the trailing 11 group rows.
- Add a group: append a row in the trailing block, write its `Draw_GroupX` + member tables in `part6.inc`, and bump `kGroupCount`.
- Never reorder existing rows without auditing every `*_M[]` table in `part6.inc` (and ideally introduce `SurfaceId` first).
