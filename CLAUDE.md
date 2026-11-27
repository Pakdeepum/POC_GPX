# MyTrophy

Web app that turns a `.gpx` route into a 3D-printable terrain piece, used like a trophy. The system generates only the terrain piece. The base is a separate, pre-made part, and the terrain piece plugs into it.

## Repositories

| Folder | Role | Stack | Git |
|---|---|---|---|
| `MyTrophy/` | Frontend | Next.js (App Router, TypeScript), Tailwind | own repo |
| `MyTrophyService/` | Backend | NestJS monorepo, microservices over TCP | own repo |

Each folder has its own `CLAUDE.md`. Each backend service has one too, under `MyTrophyService/apps/<service>/`.

- Package manager: npm.
- Root `POC_GPX/` is its own git repo (remote `Pakdeepum/POC_GPX`). Its `.gitignore` excludes `MyTrophy/` and `MyTrophyService/`, so app code changes are committed only inside those two repos.
- `Exam/` holds sample GPX files from the user for manual tests.

## Requirements

1. Upload a `.gpx` file and show the route on a map.
2. Use real map location and real terrain elevation.
3. Frame shape is selectable: pentagon, square, hexagon, circle, ellipse.
4. Each terrain type has a configurable color. Route color and base color are configured separately.
5. A print has at most 4 or 7 colors (a per-project setting). Terrain types, route, buildings and roads all count. The base color does not.
6. Generate only the terrain piece, with sockets to assemble onto the pre-made base.
7. Frontend: Next.js. Backend: NestJS microservices.
8. Database: JSON files in a `db` folder for now. PostgreSQL later.
9. Build screens so the core functions work end to end first. UI polish comes later.
10. Show real buildings on the map and add them to the 3D file.

## Product decisions

- **Base:** one pre-made base per frame shape. Terrain sizes are fixed per base variant.
- **Sizes:** user picks 100, 150 or 200 mm (widest dimension). 50 mm was dropped: too small for the terrain detail. 200 mm uses 0.67 mm grid cells (the 300 cells per side cap).
- **Frame:** cut outline only, no raised rim. The base acts as the frame.
- **Shape defaults:** pentagon point-up, hexagon flat-top, ellipse 4:3.
- **Connectors:** 3 sockets, Ø5 mm × 4 mm deep, in the underside of the terrain piece, for separate pins or magnets. 0.2 mm clearance. The base is adapted to match.
- **Colors are filaments:** no free RGB. Every color slot picks a filament from a catalog of market filaments (Bambu Lab PLA Basic and PETG HF for now).
  - Slots: the 6 terrain types, route, buildings, roads, base.
  - One material per project (`pla` or `petg`), because PLA and PETG bond poorly. A material change moves every slot to the same color name in the new material, else the nearest color.
  - Catalog: committed seed `MyTrophyService/seed/filaments.json`, merged into `db/filaments.json` on start (a seed entry replaces a stored entry with the same id). A filament that is no longer sold stays usable by the projects that already have it.
  - Defaults (PLA): Beige land, Blue water, Red route, Light Gray buildings and roads, Black base. PETG: Cream land, Blue water, Red route, Gray buildings and roads, Black base. A new project prints 3 filaments; buildings and roads share a 4th, so turning both on stays within 4.
- **Color limit:** 4 or 7 distinct filaments over the whole print, a per-project setting (default 4). Most printers take 4 filaments; 7 is for testing.
  - Counted: all 6 terrain slots (even when a type is not in the area), the route, and buildings and roads while they are on. The base is printed separately and never counts.
  - Slots that share a filament print as one part. In the 3MF the part is named after all its members, for example `Green + Urban - Bambu Lab PLA Basic Beige (10201)`.
  - A change that puts the project further over its limit is rejected with `TOO_MANY_COLORS` (400). Generation over the limit fails the same way.
- **Base color:** the filament of the separately printed base. Used in the 3D preview only.
- **Terrain types (6), from ESA WorldCover:** `terrain` (default: bare ground and every class not listed), `water` (80), `forest` (10 trees, 95 mangroves), `green` (20, 30, 40, 90, 100: shrubs, grass, crops, wetland, moss), `urban` (50), `snow` (70). They work for GPX files from abroad too.
  - Cells without WorldCover data (open sea, missing tile) are water at or below sea level, else terrain.
  - Tie-break priority: water > snow > urban > forest > green > terrain.
- **Terrain color parts:** each terrain filament other than the `terrain` one is a cap: the top 1 mm of the terrain, cut along the region outline. Hard boundaries, no gradients.
  - The class grid is fetched only when some terrain type has its own filament.
  - Patches under 2 mm² and strips under 0.8 mm wide take the color around them.
  - A water patch whose height (10th to 90th percentile) spans at most 10 m prints flat at its 10th percentile, so lakes print flat. Only when water has its own filament.
  - Color regions show in the 3D preview only, not on the map.
- **Route:** raised ribbon, 0.8 mm high above the surface. Route height comes from elevation data, not GPX elevation.
  - Width is a per-project setting: 0.4–2.0 mm in 0.1 mm steps. 0.4 mm is one standard nozzle line, the thinnest printable width. New projects start at 0.8 mm; older projects keep 1.2 mm until edited. The height is not a setting.
  - The ribbon is a separate part that sits 0.6 mm into the terrain. The terrain part has a matching groove, so the parts do not overlap.
  - At the frame edge the ribbon is cut flush with the outline. Pieces narrower than min(0.8 mm, 0.8 × width), measured before the flush trim, are removed, so with 0 % padding a route that runs along the edge can lose those pieces.
  - Gaps under 0.8 mm (two nozzle lines) between route passes are filled with route, so no unprintable terrain strip is left between them.
  - If nothing printable is left, the model has no route part and the UI shows a notice.
- **Buildings:** OSM buildings from the Overpass API, drawn on the map and printed as their own color part.
  - The map and the model use the same printable layer file, so the map shows exactly the buildings that print. While buildings are on, the map style's own buildings are hidden.
  - On/off is a per-project setting, off for new projects (older projects read as on). While on, the building filament counts toward the color limit.
  - Height: OSM `height`, else `building:levels` × 3 m, else 6 m. The same vertical exaggeration as the terrain, clamped to 20 mm above the ground, and at least 0.4 mm above the highest ground under the footprint. Flat roofs; the bottom sits 0.6 mm below the lowest ground under the footprint.
  - Footprints narrower than 0.8 mm or smaller than 1 mm² at print scale are dropped (smaller holes are filled). At most 5000 buildings / 60k vertices, largest first. `building=roof` (open canopies) is left out. Touching row houses are not merged.
  - Areas wider than 40 km get no buildings (the UI shows a notice). At that scale only very large buildings would remain.
  - The route wins: buildings are cleared from the route footprint. Priority is route > buildings > roads > terrain color parts > terrain.
  - If Overpass fails during generation, generation fails with `BUILDINGS_UNAVAILABLE` (502).
- **Roads:** OSM major roads from Overpass, printed as a flush color inlay (the top 1 mm of the terrain, like a terrain color part). Not drawn on the map.
  - On/off is a per-project setting, off by default (older projects read as off). While on, the road filament counts toward the color limit.
  - Classes: `motorway`, `trunk`, `primary`, `secondary`, `tertiary`; `_link` roads are left out. Areas wider than 40 km keep only motorway, trunk and primary.
  - Width at print scale: motorway and trunk 1.2 mm, primary 1.0 mm, secondary and tertiary 0.8 mm. Gaps under 0.8 mm are filled, so a dual carriageway prints as one road.
  - Pieces shorter than 2 mm inside the outline are dropped. At most 100k vertices, most important class first, then the longest.
  - With buildings on, one Overpass request fetches buildings and roads together. Otherwise a roads-only request is made.
  - If Overpass fails during generation, generation fails with `ROADS_UNAVAILABLE` (502).
- **Part gap:** parts never touch exactly. Each part gives up its neighbor's volume grown by 0.001 mm, because export rounding (3MF 4 decimals, STL float32) would make touching faces overlap. The gap is far below print resolution.
- **Area:** auto-fit to the route bounding box, plus a user padding %.
  - Padding is 0–100 %, default 10.
  - `area.sizeMeters` is the east-west ground width of the outline, which maps to `sizeMm`.
  - The area is fitted again when the shape or padding changes, so the padded route box always fits inside the cut outline.
- **Shape geometry:**
  - The map outline and the mesh outline come from one shared function (`libs/common/src/frame-outline.ts`).
  - Circle and ellipse use 128 segments.
  - The ellipse is wide east-west, and north is always up.
- **Manual exaggeration:** 0.5–20×.
- **Settings changes:** older models are kept. The UI shows an "outdated" hint until the user generates again.
- **GPX parsing:** all tracks and segments are joined into one route line. Upload limit is 20 MB.
- **Vertical exaggeration:** auto mode fits relief to about 20 mm, capped at 10×, so flat areas do not amplify data noise. The user can override it manually.
- **3D preview:** each part uses the color stored with the model (its filament's hex), and the piece sits on a simple plinth in the base color. The plinth is for the preview only.
- **Output:** 3MF with one part per filament, plus a ZIP of binary STL files (one per filament).
  - The 3MF holds one parent object made of the parts, so a slicer loads one object with aligned parts. Each part has a display color and a name with the full filament label, so the user knows which spool to load.
  - The UI offers only the 3MF and the ZIP. The per-part STLs are stored for the preview.
  - The "outdated" check (config hash) counts the material, the filament of every printed slot (id, hex and name), and the buildings and roads toggles. The building and road filaments count only while those are on. The base filament and the color limit do not count.
  - The 3MF description and an `ATTRIBUTION.txt` in the ZIP name the data sources: ESA WorldCover only when there is a terrain color part, OSM ODbL only when there is a buildings or roads part.
- **Generation:** synchronous request (up to about 60 s). No auth for the POC.

## Data sources (free, no API keys)

| Purpose | Source | Notes |
|---|---|---|
| Map display | OpenFreeMap style via MapLibre GL | OSM attribution required |
| Elevation | AWS Terrain Tiles (Terrarium PNG) | Decode `(R*256 + G + B/256) - 32768`. Oceans contain bathymetry, so clamp water. Attribution required. |
| Land cover | ESA WorldCover 10 m v200 (COG on public S3) | CC-BY 4.0, attribution required. Read with HTTP range requests, cached per block. |
| Buildings and roads | Overpass API (OpenStreetMap), `overpass-api.de` | ODbL, attribution required on the map and in the files. Public server: one request at a time, cache 30 days. It can queue a request for 30–45 s. |

## UI language

- Two languages: Thai (`th`) and English (`en`).
- All UI wording lives in the database as JSON keys: `db/wordings.json`, shape `{ "<key>": { "th": "...", "en": "..." } }`.
- New keys go in the committed seed `MyTrophyService/seed/wordings.json`. project-service merges missing keys into `db/wordings.json` on start and never overwrites existing values.
- `project-service` owns wordings. The frontend loads them through `GET /api/wordings`.
- Default language is Thai. The user's choice is saved in the browser (`localStorage`).
- If wordings fail to load, the frontend shows one hard-coded bilingual message with a retry button. This is the only hard-coded UI text.
- No i18n package.

## Coding rules (all projects)

- PascalCase for functions, const functions, class methods, classes and interfaces, for example `ParseGpx()`.
- lowerCamelCase for variables, including those inside functions, consts, classes and interfaces.
- Names a framework requires are exempt and follow the framework:
  - React hooks `useXxx`
  - Next.js `generateMetadata`, route handlers `GET`/`POST`
  - Nest lifecycle and interface methods (`onModuleInit`, `canActivate`, `transform`)
  - Nest kebab-case file names
- Put a newline before `if`, `return` and comments.
- Write a description comment above every function and every block of complex logic.
- Do not install a package that is not in the approved list without asking first.

## Approved packages

**Frontend:**
- `next`, `react`, `react-dom`, `typescript`, `@types/*`, `eslint`, `eslint-config-next`
- `tailwindcss`
- `maplibre-gl`, `react-map-gl`
- `three`, `@types/three`, `@react-three/fiber`, `@react-three/drei`

**Backend:**
- Nest CLI defaults: `@nestjs/core`, `@nestjs/common`, `@nestjs/platform-express`, `reflect-metadata`, `rxjs`, plus dev tooling
- `@nestjs/microservices`, `@nestjs/config`
- `class-validator`, `class-transformer`
- `@types/multer`
- `fast-xml-parser`, `pngjs` (typed by a local `pngjs.d.ts`, not `@types/pngjs`), `geotiff`
- `manifold-3d`, `fflate`

Use built-in `fetch` (no axios) and `crypto.randomUUID()` (no uuid).

## Milestones

| Phase | Scope | Done when |
|---|---|---|
| P0 (done) | Scaffold both repos and the 4 services | All services start. `GET /api/health` reports every service OK over TCP. |
| P1 (done) | GPX upload, parse, persist, map view | Sample `.gpx` shows on the map. `db/projects.json` and `db/gpx-files.json` are written. |
| P2 (done) | First full flow: square frame, one color | Terrain grid, solid mesh, STL download and 3D preview work. STL opens as manifold at the chosen size. |
| P3 (done) | All 5 shapes, sizes 50/100/150, padding, exaggeration | Every shape gives a manifold mesh whose outline matches the map overlay. |
| P4 (done) | Route part, 3MF and STL ZIP | Opens in a slicer as one object with aligned parts. |
| P4.5 (done) | OSM buildings on the map and as their own part | Map buildings match the model. `check:3mf` shows disjoint parts (0 overlap). |
| P5 (done) | Filament catalog colors, terrain colors from WorldCover, optional OSM major roads | A lake becomes its own part. A color over the limit (4 or 7) is rejected. `check:3mf` shows disjoint parts (0 overlap). |
| P6 | Connector sockets | Sockets match the standard spec. Test print fits the base. |
| — | Wordings (th/en) | Wired in during P1 and used by every screen after that. |
