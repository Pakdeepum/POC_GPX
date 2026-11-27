# project-service

TCP microservice (port 4001). The only service that reads and writes `db/`. Read `../../CLAUDE.md` and the root `CLAUDE.md` first.

## Responsibilities

- Project CRUD.
- GPX parsing with `fast-xml-parser`: read track points, compute distance and bounding box, simplify the path for display.
- Compute the frame outline GeoJSON (area plus shape) for the frontend map.
- Store model records.
- Serve wordings (th/en).

## Message patterns

- `project.create`, `project.findAll`, `project.findById`, `project.update`, `project.getRoute`
- `model.save`, `model.findByProject`
- `wording.findAll`

## Collections (`db/*.json`)

- **`projects.json`**:
  `{ id, name, gpxFileId, area{ centerLat, centerLon, sizeMeters, paddingPct }, frame{ shape, sizeMm }, terrain{ exaggerationMode, exaggeration, targetReliefMm, minThicknessMm, cellMm }, route{ widthMm, heightMm }, colors{ baseColor, routeColor, terrainColors{ terrain, water?, snow?, vegetation? } }, createdAt, updatedAt }`
  - `shape` is one of: `pentagon`, `square`, `hexagon`, `circle`, `ellipse`.
  - `sizeMm` is one of: `50`, `100`, `150`.
  - `exaggerationMode` is `auto` or `manual`.
  - A terrain type without its own color falls back to `terrain`.
- **`gpx-files.json`**: `{ id, projectId, originalName, storedPath, pointCount, distanceMeters, bbox, routePath, uploadedAt }`
- **`models.json`**: `{ id, projectId, configHash, files[{ role, format, path, bytes }], triangleCount, status, error?, createdAt }`
  - `role` is `body`, `route` or a terrain type.
- **`wordings.json`**: `{ "<key>": { "th": "...", "en": "..." } }`

Terrain grids are cache only (in `storage/grids`). They are not a collection.

## Rules

- Access every collection through a `Repository<T>` bound to a DI token. Never touch the JSON files directly from services.
- Write atomically (temp file + rename). Serialize writes per collection.
- Generate ids with `crypto.randomUUID()`.
