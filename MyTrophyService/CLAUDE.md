# MyTrophyService (backend)

NestJS monorepo (Nest CLI monorepo mode, one `package.json`, npm) for MyTrophy. Read the root `../CLAUDE.md` first for product decisions and coding rules.

## Layout

| Path | Type | Role | Port |
|---|---|---|---|
| `apps/api-gateway` | HTTP | The only HTTP entry point. REST, upload, download, orchestrates generation. | 4000 |
| `apps/project-service` | TCP | Projects, GPX parsing, wordings. The only owner of `db/`. | 4001 |
| `apps/terrain-service` | TCP | Elevation tiles, land cover, height and class grids. | 4002 |
| `apps/model-service` | TCP | Mesh generation, 3MF and STL export. | 4003 |
| `libs/contracts` | lib | Message-pattern constants, DTOs, enums (`FrameShape`, `TerrainType`, `FrameSizeMm`). | — |
| `libs/common` | lib | Local projection math, `StorageService` for paths under `storage/`. | — |

Ports and hosts are set in `.env`, loaded with `@nestjs/config`.

## Transport rules

- Services talk over TCP (`@nestjs/microservices`), with no broker. Use constants from `libs/contracts`, not string literals.
- Never send large data over TCP. Height grids, class grids and meshes are written to `storage/`, and messages carry only ids and paths.
- All services share the local filesystem for the POC. Later it becomes object storage.
- The gateway applies an RxJS `timeout` to every client call (120 s for generation).

## Folders (gitignored)

- `db/`: JSON collections, one file per collection.
- `storage/gpx`, `storage/grids`, `storage/models`, `storage/tile-cache`: uploaded and generated files.

## Database

- JSON files now, PostgreSQL later.
- Access data only through the `Repository<T>` interface: `FindById`, `FindAll(filter)`, `Create`, `Update`, `Delete`.
- `JsonFileRepository<T>` writes atomically (temp file + rename) and serializes writes per collection.
- Repositories bind to DI tokens (`PROJECT_REPOSITORY` and so on). The PostgreSQL move adds new repository classes and changes only the provider binding.

## Validation

- Use `class-validator` DTOs with `ValidationPipe` at the gateway.
- Rules: at most 4 terrain colors, size in {50, 100, 150}, shape in the 5 allowed shapes.

## Commands

Fill in after scaffold (P0).
