# MyTrophy (frontend)

Next.js web app for MyTrophy. Read the root `../CLAUDE.md` first for product decisions and coding rules.

## Stack

- Next.js App Router, TypeScript, npm
- Tailwind for styling
- `maplibre-gl` + `react-map-gl` for the map (OpenFreeMap style, no API key)
- `three` + `@react-three/fiber` + `@react-three/drei` for the 3D preview (`STLLoader`, `OrbitControls`)

## Backend

- All calls go to the api-gateway: `http://localhost:4000/api`. Set it in `.env.local` as `NEXT_PUBLIC_API_URL`.
- API types live in `src/types/api.ts`, copied from `MyTrophyService/libs/contracts` for now.
- All HTTP calls go through `src/lib/apiClient.ts`.

## Screens (core functions first)

| Route | Purpose |
|---|---|
| `/` | Project list and GPX upload. A new upload creates a project and opens its editor. |
| `/projects/[id]` | Editor with sections: Map and area, Frame, Colors, Preview, Generate and download. |

**Editor flow:**
1. **Map:** show the route. The area auto-fits the route plus a padding % slider. Draw the frame outline GeoJSON returned by the backend. The backend is the single source of truth for the outline.
2. **Frame:** shape (pentagon, square, hexagon, circle, ellipse) and size (50, 100, 150 mm). Exaggeration: auto, or a manual value.
3. **Colors:** one color for each terrain type (`terrain`, `water`, `snow`, `vegetation`), plus route color and base color. At most 4 terrain colors.
4. **Generate:** call the API, then load each STL part in the preview and color it. Show the base color in the preview only.
5. **Download:** 3MF, or the STL ZIP.

## Components (planned)

- `src/components/map/RouteMap.tsx`
- `src/components/preview/ModelPreview.tsx`
- `src/components/forms/AreaForm.tsx`
- `src/components/forms/FrameForm.tsx`
- `src/components/forms/ColorForm.tsx`

## Wording (th/en)

- Never hard-code UI text. Load wordings from `GET /api/wordings` and look up text by key.
- The current language (`th` or `en`) is user-selectable.
- Add new keys to `MyTrophyService/db/wordings.json`.

## Rules

- Show map and data attribution: OpenStreetMap/OpenFreeMap, AWS Terrain Tiles sources, ESA WorldCover (CC-BY 4.0).
- Map and three.js components are client-only (`"use client"`; use dynamic import where SSR breaks them).
- Core function first. Keep styling minimal until the flow works end to end.

## Commands

Fill in after scaffold (P0).
