# terrain-service

TCP microservice (port 4002). Builds the height grid and the land-cover class grid for an area. Read `../../CLAUDE.md` and the root `CLAUDE.md` first.

## Message pattern

- `terrain.buildGrid { bbox, cellMeters, withLandCover }` returns `{ gridId, cols, rows, heightPath, classPath, stats }`.

Grids are written to `storage/grids`. Never return grid data over TCP.

## Elevation (AWS Terrain Tiles, Terrarium PNG)

- Fetch tiles with built-in `fetch` and decode them with `pngjs`.
- Height in meters: `(R * 256 + G + B / 256) - 32768`.
- Cache tiles on disk in `storage/tile-cache`.
- Pick the zoom so one tile pixel is no larger than one grid cell in meters.
- Resample from Web Mercator into a local metric grid (equirectangular around the area center).
- Oceans contain bathymetry: clamp water cells to the water surface level.

## Land cover (ESA WorldCover 10 m)

- Read the COG from the public S3 bucket `esa-worldcover` with `geotiff` (HTTP range requests). Cache the results.
- Map WorldCover classes to `TerrainType`:
  - 80 → `water`
  - 70 → `snow`
  - 10, 20, 30, 40, 90, 95, 100 → `vegetation`
  - everything else → `terrain`
  - This mapping is a default and can be adjusted.
- Cleanup: a majority filter removes specks smaller than about 1 mm² at print scale. Flatten water cells to one level.

## Attribution

- Terrain Tiles sources (see the AWS Terrain Tiles attribution doc).
- ESA WorldCover, CC-BY 4.0.
