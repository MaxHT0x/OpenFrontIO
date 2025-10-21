# Resources & tooling

OpenFrontIO ships a large collection of static assets, generators, and automation scripts alongside the TypeScript sources. This guide covers where assets live, how to regenerate them, and what tooling keeps quality high.

## Static assets (`resources/`)

- **Maps (`resources/maps/<map>`):** Each folder contains `manifest.json`, `map.bin`, `map4x.bin`, `map16x.bin`, and `thumbnail.webp`. The binary files encode multiple zoom levels used by the renderer and worker pathfinding; manifests list metadata like display name, size, and supported game modes.
- **Translations (`resources/lang/*.json`):** Locale dictionaries consumed by the client localization utilities. Add new languages or update strings here, then refresh Crowdin if you use the hosted translation workflow.
- **Sprites & images (`resources/sprites`, `resources/images`, `resources/icons`):** Artwork for units, UI chrome, help modals, and marketing. Webpack copies these into `static/` and fingerprints filenames.
- **Audio (`resources/sounds`, `resources/sounds/effects`):** Background music and SFX played through `SoundManager.ts`.
- **Cosmetics (`resources/cosmetics`), flags (`resources/flags`), fonts (`resources/fonts`):** Additional visual assets referenced by cosmetics entitlements and UI components.
- **Version & changelog (`resources/version.txt`, `resources/changelog.md`):** Updated during releases (see `build.sh`). `Main.ts` displays the current version in the UI, and `ClientGameRunner` persists it in session stats.

## Map generation tooling

The Go project under `map-generator/` converts raw assets into the binary map bundles stored in `resources/maps`.

1. Place source assets in `map-generator/assets/maps/<map_name>` with `image.png` and `info.json` metadata.
2. Register the map in `map-generator/main.go`.
3. Run `go run .` inside `map-generator/` to produce `generated/maps/<map_name>` outputs.
4. Copy the generated files into `resources/maps/<map_name>` and run `npm run format` to keep JSON formatted.

See `map-generator/README.md` for full instructions, size recommendations, and prerequisites.

## Tests and quality gates

- **Unit/integration tests:** Jest is configured via `jest.config.ts` to run any `*.test.ts`/`*.spec.ts` under `tests/`. Shared fixtures live in `tests/util`, while feature-focused suites cover core gameplay (`tests/core/game/*.ts`) and client graphics utilities (`tests/client/graphics`).
- **Performance harness:** `npm run perf` executes deterministic scenarios in `tests/perf/*.ts` using `tsx` for quick profiling of hot paths.
- **Mocks:** Static asset imports resolve through `__mocks__/fileMock.js` during tests so Jest can load client modules without bundling.
- **Linting & formatting:** `eslint.config.js`, `tsconfig.json`, and `postcss.config.js` define repository-wide lint and build rules. Run `npm run lint`/`npm run format` before committing.

## Build, deploy, and runtime helpers

- **Webpack (`webpack.config.js`):** Bundles the client, copies assets from `resources/` and `proprietary/`, injects environment variables, and emits the production `static/` directory consumed by the server.
- **Tailwind/PostCSS (`tailwind.config.js`, `postcss.config.js`):** Style pipeline for `src/client/styles.css` and component-specific sheets.
- **Environment scaffolding:** `example.env` documents all required secrets (admin token, Cloudflare, Docker Hub, API keys). Copy it to `.env` locally.
- **Automation scripts:**
  - `build.sh` – Builds and pushes Docker images, updating `resources/version.txt`/`resources/changelog.md` when provided.
  - `deploy.sh`/`update.sh` – SSH helpers for deploying to staging/prod hosts defined in `.env`.
  - `setup.sh`/`startup.sh` – Provisioning scripts for fresh machines and supervisor-managed processes.

## TypeScript project layout

- `tsconfig.json` configures the ESM TypeScript compiler, while `tsconfig.jest.json` tailors settings for Jest.
- `src/global.d.ts` declares asset module types (images, audio, text) so both the client and tests can import them without TypeScript errors.
- `eslint.config.js` and `tailwind.config.js` live at the repo root to keep IDE integrations working without extra setup.

Keep assets and generated files under source control—new developers rely on this repository as the single source of truth for art, localization, and map data.
