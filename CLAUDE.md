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
5. A model has at most 4 terrain colors. Route color and base color do not count toward this limit.
6. Generate only the terrain piece, with sockets to assemble onto the pre-made base.
7. Frontend: Next.js. Backend: NestJS microservices.
8. Database: JSON files in a `db` folder for now. PostgreSQL later.
9. Build screens so the core functions work end to end first. UI polish comes later.

## Product decisions

- **Base:** one pre-made base per frame shape. Terrain sizes are fixed per base variant.
- **Sizes:** user picks 50, 100 or 150 mm (widest dimension).
- **Frame:** cut outline only, no raised rim. The base acts as the frame.
- **Shape defaults:** pentagon point-up, hexagon flat-top, ellipse 4:3.
- **Connectors:** 3 sockets, Ø5 mm × 4 mm deep, in the underside of the terrain piece, for separate pins or magnets. 0.2 mm clearance. The base is adapted to match.
- **Base color:** the color of the separately printed base. Used in the 3D preview only. The terrain print needs at most 5 filaments (4 terrain + route).
- **Terrain types (fixed set of 4):** `water`, `snow`, `vegetation`, `terrain` (default). A type without its own color falls back to `terrain`.
- **Route:** raised ribbon, 1.2 mm wide × 0.8 mm high above the surface. Route height comes from elevation data, not GPX elevation.
- **Area:** auto-fit to the route bounding box, plus a user padding %.
- **GPX parsing:** all tracks and segments are joined into one route line. Upload limit is 20 MB.
- **Vertical exaggeration:** auto mode fits relief to about 20 mm, capped at 10×, so flat areas do not amplify data noise. The user can override it manually.
- **3D preview:** the terrain piece uses the terrain color and sits on a simple plinth in the base color. The plinth is for the preview only.
- **Output:** 3MF with one part per color, plus a ZIP of binary STL files (one per color).
- **Generation:** synchronous request (up to about 60 s). No auth for the POC.

## Data sources (free, no API keys)

| Purpose | Source | Notes |
|---|---|---|
| Map display | OpenFreeMap style via MapLibre GL | OSM attribution required |
| Elevation | AWS Terrain Tiles (Terrarium PNG) | Decode `(R*256 + G + B/256) - 32768`. Oceans contain bathymetry, so clamp water. Attribution required. |
| Land cover | ESA WorldCover 10 m (COG on public S3) | CC-BY 4.0, attribution required |

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
| P3 | All 5 shapes, sizes 50/100/150, padding, exaggeration | Every shape gives a manifold mesh whose outline matches the map overlay. |
| P4 | Route part, 3MF and STL ZIP | Opens in a slicer as one object with aligned parts. |
| P5 | Terrain colors from WorldCover | A lake becomes its own part. A 5th terrain color is rejected. |
| P6 | Connector sockets | Sockets match the standard spec. Test print fits the base. |
| — | Wordings (th/en) | Wired in during P1 and used by every screen after that. |
