# Architecture overview

OpenFrontIO is a full-stack TypeScript project that reuses a single simulation core on both the browser and the Node.js server. The runtime is split into three cooperating subsystems:

1. **The clustered Node server** (`src/server`) launches a primary process that supervises worker processes responsible for hosting games and serving HTTP/WebSocket traffic.
2. **The browser client** (`src/client`) boots a Pixi.js renderer, custom UI components, and a transport layer that speaks WebSocket to the game server.
3. **The shared game core** (`src/core`) implements the simulation, data schemas, configuration, and worker bridge that both tiers load to stay deterministic.

Static assets in `resources` (maps, languages, sprites, sounds, cosmetics) are copied into the production bundle by webpack. The Go map generator and deployment scripts live at the repository root and support the runtime but do not execute at run time.

## Process layout

- `src/server/Server.ts` is the entry point. It reads configuration, optionally prepares Cloudflare tunnels for non-dev deployments, and forks `config.numWorkers()` Node workers via `cluster` before starting the master HTTP server. Workers call `startWorker()` to host games, while the primary process calls `startMaster()` to coordinate them.
- `src/server/Master.ts` owns the lightweight Express app on port 3000 that serves `/api/env`, `/api/public_lobbies`, and admin endpoints, proxies kick-player requests to the correct worker, and schedules public games by polling worker lobby state. It also distributes static assets from the webpack `static/` directory.
- `src/server/Worker.ts` spins up an Express server plus a `ws` WebSocket server. Each worker keeps a `GameManager` instance that creates and tracks `GameServer` objects, exposes REST endpoints for lobby management (`/api/create_game`, `/api/start_game`, `/api/game/:id`, etc.), and attaches middlewares for compression, rate limiting, privilege refreshing, and JWT validation.
- `src/server/GameManager.ts` and `src/server/GameServer.ts` host the authoritative game loop on the server: `GameManager` owns the active `GameServer` set and advances them via `tick()`, while `GameServer` manages lobby membership, turn buffers, hash validation, archiving, and WebSocket fan-out.
- The `src/server/Worker.ts` WebSocket handler wraps sockets in `Client` objects (`src/server/Client.ts`) that authenticate, join games, and forward intents to their owning `GameServer` instance.

## Shared simulation flow

- Every game is described by the Zod schemas in `src/core/Schemas.ts`. These types are shared verbatim by the server, client, and worker to avoid drift.
- `GameRunner` (`src/core/GameRunner.ts`) is the engine that consumes buffered turns, executes deterministic ticks via the `Executor` pipeline (`src/core/execution`), and emits packed update streams. Both the browser and the server host an instance when they need to run the simulation locally.
- The browser never executes the simulation on the main thread. Instead, `WorkerClient` (`src/core/worker/WorkerClient.ts`) instantiates a dedicated web worker (`src/core/worker/Worker.worker.ts`) that loads the shared core, replays turns delivered by the server, and streams computed `GameUpdateViewData` back to the client UI.
- Terrain data, map manifests, and nation definitions are fetched by `TerrainMapLoader`/`FetchGameMapLoader` (`src/core/game`) using the static map assets produced by the Go map generator.

## Client/server messaging

- On the client, `ClientGameRunner` (`src/client/ClientGameRunner.ts`) wires together the UI event bus, input system, renderer, and `Transport` (`src/client/Transport.ts`). `Transport` translates DOM/UI events into strongly typed intents, serializes them through the shared schemas, and sends them through a WebSocket or a local single-player shim (`LocalServer.ts`).
- On the server, `GameServer` receives these intents, validates them with the shared Zod schemas, applies them to its authoritative simulation, and multicasts turn updates to connected clients. Periodic hash comparisons ensure clients stay in sync; divergences trigger a desync response path.

## Build and asset pipeline

- `webpack.config.js` bundles the client starting at `src/client/Main.ts`, copies everything from `resources` and `proprietary` into `static/`, injects environment variables, and fingerprints all binary assets.
- `tailwind.config.js`/`postcss.config.js` configure styling for the client CSS under `src/client/styles` and `src/client/styles.css`.
- Runtime configuration (turn timing, map sizing, JWT and tunnel credentials, etc.) is abstracted behind `ServerConfig`/`Config` interfaces (`src/core/configuration`). `ConfigLoader.ts` selects the correct implementation based on the `GAME_ENV` provided by the master HTTP endpoint or environment variables.

Keep this mental model in mind as you read the subsystem-specific guides—the rest of the documentation references the files above when explaining individual responsibilities.
