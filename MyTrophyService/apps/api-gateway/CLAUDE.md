# api-gateway

The only HTTP entry point of MyTrophyService. It holds no data. Read `../../CLAUDE.md` and the root `CLAUDE.md` first.

## Responsibilities

- Serve REST under `/api` on port 4000. Enable CORS for the frontend (`http://localhost:3000`).
- Accept GPX upload (multer through `@nestjs/platform-express`). Hand the file to `StorageService`, then call project-service.
- Validate input with `class-validator` DTOs and `ValidationPipe`.
- Orchestrate model generation: project-service, then terrain-service, then model-service, then project-service again.
- Stream generated files (3MF, STL, ZIP) from `storage/`.

## REST endpoints

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/projects` | Multipart `gpx` file plus `name`. Returns the project and a route summary. |
| GET | `/api/projects` | List projects. |
| GET | `/api/projects/:id` | Project, including the `frameOutline` GeoJSON. |
| GET | `/api/projects/:id/route` | Route as a GeoJSON LineString. |
| PATCH | `/api/projects/:id` | Update area, frame, terrain, route and colors. |
| POST | `/api/projects/:id/models` | Generate synchronously. Returns the model record. |
| GET | `/api/projects/:id/models` | List models of a project. |
| GET | `/api/models/:id/files/:role.:ext` | Stream one file. |
| GET | `/api/wordings` | All wording keys (`{ key: { th, en } }`). |
| GET | `/api/health` | Ping every TCP service. |

## Rules

- Use `ClientProxy` with message-pattern constants from `libs/contracts`.
- Apply an RxJS `timeout` to every call (120 s for generation).
- Pass only ids and paths between services. Never pass grids or meshes.
