# PLATEAU SDK for Godot

![Godot](https://img.shields.io/badge/Godot-4.2%2B-478cbf?logo=godotengine&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-green)
![Status](https://img.shields.io/badge/status-beta-yellow)

A pure-GDScript toolkit that brings [Project PLATEAU](https://www.mlit.go.jp/plateau/en/)'s open 3D city data into Godot: browse the official dataset catalog, import CityGML buildings, import aerial imagery, and explore the result with a playable character, all from a single editor dock.

No GDExtension, no native build, no third-party binary. Just GDScript files you can read, modify, and drop into any Godot 4.2+ project.

<img width="640" height="340" alt="plateau_sdk_for_godot" src="https://github.com/user-attachments/assets/375b2434-54e0-4b2b-b29b-af248a2f71d5" />


## Table of contents

- [What is Project PLATEAU](#what-is-project-plateau)
- [What this SDK does](#what-this-sdk-does)
- [Why pure GDScript](#why-pure-gdscript)
- [Installation](#installation)
- [Usage guide](#usage-guide)
- [Project architecture](#project-architecture)
- [Technical notes](#technical-notes)
- [Known limitations](#known-limitations)
- [License and attribution](#license-and-attribution)
- [Related projects](#related-projects)
- [Contributing](#contributing)

---

## What is Project PLATEAU

[Project PLATEAU](https://www.mlit.go.jp/plateau/en/) is an open-data initiative led by Japan's Ministry of Land, Infrastructure, Transport and Tourism (MLIT). Since 2020 it has produced and released free, standardized 3D city models (CityGML) covering municipalities across Japan, aimed at urban planning, simulation, and digital-twin applications.

MLIT already publishes official SDKs for [Unity](https://github.com/Project-PLATEAU/PLATEAU-SDK-for-Unity) and [Unreal Engine](https://github.com/Project-PLATEAU/PLATEAU-SDK-for-Unreal), both MIT-licensed. At the time of writing, no official or community SDK exists for Godot. This project is an independent, unofficial attempt to fill that gap.

## What this SDK does

Enable the plugin and a single **PLATEAU SDK** dock appears in the Godot editor, with a shared **Project Setup** section and three tabs:

| Tab | Purpose |
|---|---|
| **3D (PLATEAU)** | Browse the official dataset catalog, analyze a downloaded dataset zip, select the categories and mesh tiles you need, and import CityGML buildings (LOD0-4, textured) into the open scene. |
| **2D (PLATEAU Ortho)** | Pan a minimap, select an area, download PLATEAU's aerial orthophoto imagery, and optionally generate a textured ground plane. |
| **Exploration** | Drop a playable character (first-person or third-person) into whatever you just imported, and press Play to walk or fly around it. |

The **Project Setup** section holds a single geographic origin (JGD2011 zone + latitude/longitude) shared by both the 3D and 2D tabs, so an imported building and a downloaded aerial photo always line up in the same coordinate space instead of each import re-centering independently.

## Why pure GDScript

This SDK was originally meant to build on [shiena/godot-plateau](https://github.com/shiena/godot-plateau), a community GDExtension wrapping the native `libplateau` library. That approach could not be gotten to run reliably, so everything here was rewritten from scratch as plain `.gd` scripts instead: no C++ toolchain, no compiled binary to keep working across Godot versions or platforms, and no build step. The trade-off is a more modest feature set than a mature native pipeline would offer; see [Known limitations](#known-limitations).

## Installation

1. Copy the `addons/plateau_sdk/` folder into your project's `addons/` folder.
2. In Godot, go to **Project Settings > Plugins** and enable **PLATEAU SDK**.
3. The **PLATEAU SDK** dock appears (default: right side of the editor).

Enabling the plugin also registers two things the Exploration tab's characters need at Play time:

- An `Audio` autoload singleton, used for footstep/jump/landing sounds.
- A set of Input Map actions (movement, jump, camera look, zoom, mouse capture), bound to WASD / arrow keys / Space / mouse, matching the convention of the source character kits (see [License and attribution](#license-and-attribution)).

These are only added if your project doesn't already define an action with the same name, and they are **not** removed automatically if you later disable the plugin, since scenes you've built by then may depend on them. Remove them by hand from **Project Settings > Autoload** / **Input Map** if you no longer need them.

## Usage guide

1. **Project Setup** — Pick a JGD2011 zone (1-19) and an origin latitude/longitude, or leave it: step 3 below can auto-detect it for you from the data itself. Any tab that auto-detects an origin keeps this section in sync.
2. **3D (PLATEAU) tab — find a dataset** — Load the catalog, pick a prefecture and municipality, and open its dataset zip URL in your browser. Downloads happen outside the editor, since multi-gigabyte files have no pause/resume support inside it.
3. **3D (PLATEAU) tab — extract** — Point the panel at the downloaded zip, analyze it, choose the feature categories and mesh tiles you want, and extract. This fills in the import path below automatically, and auto-detects a project origin and zone if you skipped step 1.
4. **3D (PLATEAU) tab — import** — Choose a level of detail (or leave it on "maximum available, automatic") and press Import. Buildings appear in the currently open scene.
5. **2D (PLATEAU Ortho) tab** — Pan the minimap to your area of interest, draw a selection, and download. You can optionally generate a textured ground plane aligned to the same shared origin, and you can set the download area directly from the mesh tiles selected in the 3D tab instead of drawing a rectangle by hand.
6. **Exploration tab** — Pick a character, click a spot on the schematic top-down map of what you just imported, and press **Place**. Press **Play** (F6) to actually walk or look around; placing only sets a starting position, it doesn't move anything in the editor viewport itself.

## Project architecture

```
addons/plateau_sdk/
├── plugin.cfg, plugin.gd        Entry point: registers the dock, the Audio
│                                 autoload, and the Exploration input actions.
├── main_panel.gd                Dock root: Project Setup section + the
│                                 three-tab TabContainer.
├── core/                        Shared, addon-agnostic utilities.
│   ├── jgd2011_projection.gd    JGD2011 (Gauss-Krüger) lat/lon → meters.
│   ├── tile_math.gd             Web Mercator XYZ tile math.
│   ├── mesh_code_math.gd        JIS X0410 mesh code → lat/lon bounds.
│   ├── xml_tree.gd              Generic XML/GML tree, built on XMLParser.
│   ├── zip_central_directory.gd ZIP metadata reader (no decompression).
│   ├── gml_file_finder.gd       Recursive .gml file lookup.
│   ├── origin_finder.gd         Origin/zone auto-detection from a .gml file.
│   ├── project_origin.gd        Shared origin/zone state and change signal.
│   └── scene_geometry_utils.gd  Reads buildings/ground already present in
│                                 the edited scene (used by the schematic
│                                 map and by height-matching).
├── dataset/                     "3D (PLATEAU)" tab: catalog browsing, zip
│                                 selection/extraction, and CityGML import.
├── ortho/                       "2D (PLATEAU Ortho)" tab: aerial imagery.
├── converter/                   CityGML → Godot scene import engine, used
│                                 internally by the 3D tab's import step.
└── exploration/                 "Exploration" tab: places a playable
    ├── exploration_panel.gd      character into the imported scene.
    ├── scene_top_view.gd         Top-down schematic placement widget.
    └── characters/                Runtime character scenes/scripts, adapted
                                    from Kenney's MIT-licensed starter kits;
                                    see LICENSE-THIRD-PARTY.md in that folder.
```

## Technical notes

A few implementation details worth knowing if you plan to read, extend, or simply trust this code:

- **JGD2011 projection** — Implements the official GSI (Geospatial Information Authority of Japan) simplified Gauss-Krüger formula for the Japanese Plane Rectangular Coordinate System (19 zones), following the [GSI reference calculation](https://vldb.gsi.go.jp/sokuchi/surveycalc/surveycalc/algorithm/bl2xy/bl2xy.htm). The zone origin table comes from [MLIT Notice No. 9 of 2002](https://www.gsi.go.jp/LAW/heimencho.html). Before relying on it for precision work, spot-check at least one known point against the [official GSI calculator](https://vldb.gsi.go.jp/sokuchi/surveycalc/surveycalc/bl2xyf.html).
- **CityGML parsing** — Handles CityGML 2.0's LOD1-4 geometry (via `boundedBy`-classified surfaces, falling back to a raw `Solid` where a dataset doesn't classify surfaces) and LOD0 footprints/roof edges, which use a different element structure entirely.
- **PLATEAU catalog API** — Queries `GET https://api.plateauview.mlit.go.jp/datacatalog/plateau-datasets`. MLIT labels this endpoint **experimental**, with no uptime or schema-stability guarantee; the client parses defensively and reports a schema mismatch instead of failing silently.
- **ZIP handling** — Reads the raw ZIP central directory to get each entry's size without decompressing it first (the same principle as `unzip -l`), since Godot's built-in `ZIPReader` doesn't expose this directly. Handles ZIP64 for the global fields (needed for archives over 4 GB, such as all of Tokyo's 23 wards).
- **Basemap attribution** — The 3D tab's mesh-selection map uses GSI "pale" basemap tiles and displays the required "国土地理院" (GSI) attribution in the tab itself.
- **Scene layout** — Each import creates one node per mesh code, holding one node per feature category (`bldg`, `brid`, ...), holding the actual mesh instances. Collision is opt-in, off by default, and generates an exact trimesh collider per building when enabled.

## Known limitations

- Exploration-tab character placement is geometric (ground-plane height or lowest imported building), not physics-based: there's no editor-time "drop to floor" raycast.
- The Exploration tab's schematic map is a flat, axis-aligned diagram from mesh bounding boxes, not a rendered preview.
- No support for polygon holes (e.g. courtyards): a polygon with an interior ring is filled as if it had none.
- No TIFF texture support (PNG/JPG/BMP/TGA/WebP only).
- Texture V-axis flip is a documented default guess; flip it back if textures appear upside down.
- Attributes exposed per building are minimal (`gml:id`, `measuredHeight` only); no extended i-UR attributes (land use, construction year, etc.).
- Zip extraction and CityGML import are synchronous; large selections can freeze the editor temporarily with no progress reporting.
- One mesh instance per building, no mesh merging; re-importing into the same scene duplicates the hierarchy rather than updating it in place.
- Import zoom level 19 has been unreliable on the Ortho tab in practice; zoom 18 is more reliable and still high resolution.

## License and attribution

This SDK's own code is released under the MIT License (see `LICENSE` at the repository root).

The `exploration/characters/` folder contains code and assets adapted from Kenney's official Godot starter kits ([3D Platformer](https://github.com/Kenney-NL/Starter-Kit-3D-Platformer) and [FPS](https://github.com/Kenney-NL/Starter-Kit-FPS)), both MIT-licensed. Full attribution and an exact list of changes are in `exploration/characters/LICENSE-THIRD-PARTY.md`.

3D city model and catalog data are provided by [Project PLATEAU](https://www.mlit.go.jp/plateau/en/) / MLIT. Basemap tiles are provided by the Geospatial Information Authority of Japan (国土地理院 / GSI).

This project is not affiliated with or endorsed by MLIT, the Project PLATEAU team, or the Godot Foundation.

## Related projects

- [PLATEAU SDK for Unity](https://github.com/Project-PLATEAU/PLATEAU-SDK-for-Unity) and [PLATEAU SDK for Unreal](https://github.com/Project-PLATEAU/PLATEAU-SDK-for-Unreal) — the official MLIT-backed SDKs (documentation in Japanese only).
- [shiena/godot-plateau](https://github.com/shiena/godot-plateau) — a community GDExtension wrapping the native `libplateau` library; an earlier inspiration for this project, not a dependency of it.

## Contributing

This is a beta, first public release. Bug reports, pull requests, and results from real-world testing (particularly against live PLATEAU datasets and in a running Godot editor) are all welcome via GitHub Issues.
