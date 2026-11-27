# model-service

TCP microservice (port 4003). Turns grids and project settings into printable mesh files. CPU-heavy, so it is kept separate. Read `../../CLAUDE.md` and the root `CLAUDE.md` first.

## Message pattern

- `model.generate { projectConfig, routePath, grid }` returns `{ files[], triangleCount }`.

Input grids are read from `storage/grids`. Output files are written to `storage/models`.

## Pipeline (manifold-3d)

1. **Scale:** `scale = sizeMm / areaMeters`. Mesh units are mm. Sizes are 50, 100 or 150 mm.
2. **Exaggeration:** auto mode fits the relief to about 20 mm. Manual mode uses the project value.
3. **Thickness:** the solid under the lowest point must be at least socket depth (4 mm) + 1.5 mm.
4. **Frame solid:**
   - Build the outline with `CrossSection`: pentagon (point-up), square, hexagon (flat-top), circle, ellipse (4:3, a scaled circle).
   - Extrude it, run `refineToLength(cell)`, then `warp()` the top vertices to the terrain height.
   - The result is a watertight solid. Cut outline only, no rim.
5. **Route:**
   - Clip the GPX path to the frame. Simplify it with Douglas–Peucker (about 0.2 mm tolerance).
   - Build a stroke from the `hull` of circles at consecutive points, then union.
   - Make a raised ribbon 1.2 mm wide × 0.8 mm high that follows the terrain surface. It is a separate part.
6. **Terrain colors:**
   - Turn class masks into polygons (marching squares, custom code).
   - Each non-default type becomes a cap part: region prism ∩ top skin (about 1.0 mm). Subtract the cap from the body.
   - Priority: route > terrain types > body.
   - At most 4 terrain colors.
7. **Sockets:** subtract 3 sockets, Ø5 mm × 4 mm deep, from the underside, with 0.2 mm clearance.
8. **Export:**
   - **3MF:** one object per color part, grouped as components of one parent object, with `basematerials` display colors. Written by hand (XML plus `fflate` zip).
   - **ZIP:** one binary STL per color part (`fflate`).

## Rules

- Call `delete()` on every manifold-3d WASM object when done, or memory leaks.
- Keep the minimum feature size at or above 0.8 mm.
- Default cell size is 0.4 mm. Cap the cell count to bound memory and triangle count.
