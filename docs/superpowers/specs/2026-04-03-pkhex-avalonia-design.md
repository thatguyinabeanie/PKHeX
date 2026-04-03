# PKHeX Avalonia — Cross-Platform GUI Design Spec

## Goal

Build a full cross-platform replacement for PKHeX.WinForms using Avalonia UI, targeting macOS, Linux, and Windows. The app references `PKHeX.Core` for all domain logic (save loading, entity parsing, legality, editing) and provides an identical user experience to the existing WinForms app with modern visual polish and light/dark mode support.

## Architecture

MVVM architecture using CommunityToolkit.Mvvm source generators. Views are pure AXAML with data bindings. ViewModels wrap `PKHeX.Core` types (`SaveFile`, `PKM`, `LegalityAnalysis`). No changes to `PKHeX.Core` are required.

Two new projects are added to the existing `PKHeX.sln`:
- **PKHeX.Avalonia** — The main application (desktop executable)
- **PKHeX.Drawing.Avalonia** — Sprite loading and composition using Avalonia-native rendering

## Tech Stack

- **Avalonia UI** (+ Avalonia.Desktop) — Cross-platform .NET UI framework
- **Avalonia.Themes.Fluent** — Built-in Fluent theme with light/dark mode variants
- **CommunityToolkit.Mvvm** — MVVM source generators (`[ObservableProperty]`, `[RelayCommand]`)
- **PKHeX.Core** — All domain logic (existing, unchanged)
- **Target:** `net10.0` (cross-platform)

## Visual Design

**Style:** Clean Hybrid — compact information density matching the current WinForms PKHeX, with modern typography, subtle borders, gentle rounding, and consistent spacing. Not a redesign — a visual polish of the same layout.

**Theme:** Avalonia FluentTheme. Defaults to system preference (light or dark). Manual toggle in menu bar. Both themes must maintain readability and contrast for all controls.

**Layout:** Identical to current WinForms PKHeX:
- Menu bar (File, Tools, Options) with legality indicator and theme toggle
- Split pane: PKM editor (left, ~400px fixed), Save editor (right, flexible)
- Status bar showing game, trainer, TID, box info

## Project Structure

```
PKHeX.sln (modified — add two new projects)

PKHeX.Drawing.Avalonia/          (NEW — net10.0)
  PKHeX.Drawing.Avalonia.csproj
  References: PKHeX.Core, Avalonia
  ├── SpriteBuilder.cs           — Compose sprites (shiny, gender, item overlays)
  ├── SpriteFetcher.cs           — Load sprite images from embedded/file resources
  └── SpriteUtil.cs              — Helpers (type colors, ball images, item images)

PKHeX.Avalonia/                  (NEW — net10.0)
  PKHeX.Avalonia.csproj
  References: PKHeX.Core, PKHeX.Drawing.Avalonia,
              Avalonia, Avalonia.Desktop, Avalonia.Themes.Fluent,
              CommunityToolkit.Mvvm
  ├── App.axaml / App.axaml.cs   — Application entry, theme setup, service registration
  ├── Program.cs                 — Desktop entry point
  ├── ViewModels/                — All ViewModels (MainWindowVM, PKMEditorVM, SaveEditorVM, etc.)
  ├── Views/                     — All AXAML views bound to ViewModels
  ├── Controls/                  — Reusable custom Avalonia controls
  ├── Converters/                — IValueConverters (species→sprite, type→color, bool→visibility, etc.)
  └── Services/                  — File I/O, save loading, settings persistence
```

## ViewModel Architecture

```
MainWindowViewModel              — Top-level: holds SaveEditorVM + PKMEditorVM, menu commands
├── PKMEditorViewModel           — Wraps PKM entity, exposes all editable properties
│   ├── MainTabViewModel         — Species, nickname, level, nature, ability, item, PID/EC
│   ├── MetTabViewModel          — Origin game, met location, ball, dates, fateful encounter
│   ├── StatsTabViewModel        — IVs, EVs, computed stats, hyper training, characteristic
│   ├── MovesTabViewModel        — 4 current moves + PP/PPUps, 4 relearn moves, move shop/tech records
│   └── CosmeticTabViewModel     — OT/HT info, markings, ribbons, contest stats, memories
└── SaveEditorViewModel          — Wraps SaveFile, exposes boxes/party/tools
    ├── BoxViewModel             — 30 slots per box, box navigation, box names/wallpapers
    ├── PartyViewModel           — 6 party member slots
    ├── OtherSlotsViewModel      — Extra slots (fused, rental, daycare, ride, surprise trade, etc.)
    └── SaveToolsViewModel       — Item editor, event flags, mystery gifts, etc.

SlotViewModel                    — Single Pokémon slot: sprite bitmap, species name, level, legality
```

**Data flow:**
1. User opens save file → `SaveUtil.GetVariantSAV()` returns a `SaveFile` → `SaveEditorViewModel` wraps it
2. User clicks a box slot → `SlotViewModel` raises selection → `PKMEditorViewModel` loads that `PKM`
3. User edits a field → ViewModel property setter writes to the `PKM`'s backing `Memory<byte>`
4. User saves → `SaveFile` encrypts/checksums and writes final data

ViewModels use `[ObservableProperty]` for automatic `INotifyPropertyChanged`. Commands use `[RelayCommand]`. Views are pure AXAML — no code-behind beyond `InitializeComponent()`.

## Main Window Layout

Matches the current WinForms PKHeX exactly:

```
┌──────────────────────────────────────────────────────────┐
│ PKHeX   File   Tools   Options              [●] [🌙]    │  ← Menu bar
├────────────────────────┬─────────────────────────────────┤
│ [Sprite] Pikachu       │ Boxes | Party | Other | Tools   │  ← Tab bars
│ Lv.50 ♂ Timid  [Legal] │ ◀ Box 1 ▶                      │
├────────────────────────┤                                 │
│ Main|Met|Stats|Mvs|Cos │ ┌──┬──┬──┬──┬──┬──┐            │
│                        │ │  │  │  │  │  │  │            │
│ Species:  [Pikachu  ]  │ ├──┼──┼──┼──┼──┼──┤            │
│ Nickname: [Sparky   ]  │ │  │  │  │  │  │  │            │
│ Level:    [50       ]  │ ├──┼──┼──┼──┼──┼──┤            │  ← 6×5 box grid
│ Nature:   [Timid    ]  │ │  │  │  │  │  │  │            │
│ Ability:  [Static   ]  │ ├──┼──┼──┼──┼──┼──┤            │
│ Item:     [Light Ba ]  │ │  │  │  │  │  │  │            │
│ Friendship:[255     ]  │ ├──┼──┼──┼──┼──┼──┤            │
│ Language: [English  ]  │ │  │  │  │  │  │  │            │
│                        │ └──┴──┴──┴──┴──┴──┘            │
│ PID: A3B7C912          │                                 │
│ EC:  F8D2E401          │                                 │
├────────────────────────┴─────────────────────────────────┤
│ Scarlet — Gabe — TID: 123456          Box 1/32 · 7 PKM  │  ← Status bar
└──────────────────────────────────────────────────────────┘
```

## Phasing

Each phase produces a working, shippable application. Later phases build on earlier ones.

### Phase 1: Foundation + Save Loading + Box Viewer
- Project scaffolding (csproj, solution integration, App.axaml)
- Main window shell with menu bar, split pane, status bar
- Save file loading via `SaveUtil.GetVariantSAV()` with file picker
- Box viewer: 6×5 grid of sprite slots, box navigation (prev/next/name)
- Party viewer: 6 vertical slots
- Click a slot → display basic Pokémon info in the left pane (species, level, nature, item — read-only)
- Sprite loading from pokesprite resources (static PNGs)
- Light/dark theme toggle
- Save file export (write back to file)

### Phase 2: Full PKM Editor
- All 5 editor tabs with full read/write capability:
  - Main: Species, nickname, level, EXP, nature, ability, item, friendship, language, PID/EC, gender, shiny
  - Met: Origin game, met location, ball, met date/level, egg info, fateful encounter
  - Stats: IVs, EVs, computed stats display, hyper training, AVs/GVs where applicable
  - Moves: 4 current moves with PP/PPUps, 4 relearn moves, move shop/tech records
  - Cosmetic: OT/HT info, trainer IDs, markings, contest stats, ribbons (launcher), memories
- Format-aware controls (show/hide fields based on entity generation)
- Combobox population from `PKHeX.Core` game data (species list, item list, move list, etc.)

### Phase 3: Legality + Import/Export
- `LegalityAnalysis` integration — run on every loaded entity
- Legality indicator (green/red) with detailed results panel
- Showdown format import/export
- Individual `.pk*` file import/export (drag-and-drop where platform supports it)
- QR code import (using existing QRCoder dependency)

### Phase 4: Save Tools
- Item/inventory editor
- Event flags/work editor
- Mystery gift manager
- Rental team viewer
- Pokédex editor
- Trainer info editor
- All remaining save-specific sub-editors (mapped from WinForms Subforms)

### Phase 5: Polish
- Drag-and-drop between box slots, party, and editor
- Undo/redo
- Box search/filter
- PKM database browser
- Batch editor
- Settings persistence (JSON, matching WinForms `cfg.json` format where possible)
- Keyboard shortcuts matching WinForms app

## Sprite Handling (PKHeX.Drawing.Avalonia)

Uses Avalonia's native rendering APIs (backed by Skia) for sprite composition:

- **SpriteFetcher** — Loads base sprite PNGs from the pokesprite collection (embedded resources or file-based). Maps `(species, form, gender, shiny)` → `Avalonia.Media.Imaging.Bitmap`.
- **SpriteBuilder** — Composes final display sprites by layering: base sprite + shiny overlay + held item icon + egg icon + status indicators. Uses Avalonia `DrawingContext` for composition.
- **SpriteUtil** — Helpers for type-to-color mapping, ball sprite lookup, item sprite lookup.

Resources are the same raw sprite files used by `PKHeX.Drawing.PokeSprite` but loaded through Avalonia's asset system instead of `System.Drawing`.

## Save File Handling

- Open: File picker → `File.ReadAllBytes()` → `SaveUtil.GetVariantSAV()` → wrap in `SaveEditorViewModel`
- Save: `SaveFile.Write()` → file picker for destination
- Auto-detect save format from file contents (PKHeX.Core handles this)
- Support all formats PKHeX.Core supports (Gen 1–9, GameCube, Stadium, etc.)
- Memory card files (`.raw`, `.bin`) supported via existing Core logic

## Theme System

- `FluentTheme` with `ThemeVariant.Light` and `ThemeVariant.Dark`
- Toggle via menu bar icon or Options menu
- Persisted in settings (default: follow system preference)
- All custom controls must respect theme variants via Avalonia style system
- Box slot backgrounds use subtle type-tinted colors in both themes

## Platform Notes

- **macOS:** Native menu bar integration via Avalonia's `NativeMenu`. `.app` bundle packaging.
- **Linux:** X11 and Wayland support via Avalonia defaults. AppImage or Flatpak packaging.
- **Windows:** Standard windowed app. Can coexist with WinForms PKHeX.
- All platforms: single-file publish, self-contained optional.
