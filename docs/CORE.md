# Core logic guide

The shared core (`src/core`) contains all domain logic that must stay identical between the client, server, and worker. It is written in TypeScript, type-safe via Zod schemas, and designed to run in both Node and the browser.

## Schemas and serialization

- `Schemas.ts` – Canonical Zod definitions for IDs, lobby/game configuration, intent and message payloads, winner votes, hashes, and game archives. Both transports import these to validate runtime data.
- `WorkerSchemas.ts` – Narrower schemas describing worker-facing REST payloads (`CreateGameInput`, etc.) so server endpoints can perform structured validation.
- `ApiSchemas.ts`, `CosmeticSchemas.ts`, `StatsSchemas.ts`, `CustomFlag.ts` – DTO definitions for REST integrations (user info, cosmetics entitlements, stats snapshots, custom flag uploads).
- Helpers like `Base64.ts` and `PatternDecoder.ts` provide utility codecs shared by the worker simulation and the client renderer.

## Configuration system

- `configuration/Config.ts` exposes the `Config` and `ServerConfig` interfaces. `Config` controls gameplay tuning (spawn timing, troop rates, donation rules), while `ServerConfig` supplies infrastructure details (workers, JWT issuer/audience, admin tokens, Cloudflare, OTLP, R2 storage).
- `configuration/ConfigLoader.ts` loads the correct implementation based on `GAME_ENV`. On the server it synchronously selects between `DevServerConfig`, `preprodConfig`, and `prodConfig`. On the client it fetches `/api/env`, caches the response, and instantiates `DevConfig`/`DefaultConfig` wrappers.
- `configuration/ColorAllocator.ts` plus `PastelTheme`/`PastelThemeDark` define the theme palettes applied to player territories; these feed into the renderer via the shared config.

## Game simulation

- `game/GameImpl.ts` builds a `Game` instance from `GameStartInfo`, map data, and configuration. It wires all subsystems (economy, combat, alliances, nukes, transport ships, trains) into a cohesive simulation.
- `game/Game.ts` defines enums (`GameType`, `GameMode`, `UnitType`, etc.), value objects, and utility types used throughout the project.
- `game/GameMap.ts`, `TerrainMapLoader.ts`, `BinaryLoaderGameMapLoader.ts`, and `FetchGameMapLoader.ts` load binary map data (`map.bin`, `map4x.bin`, `map16x.bin`) from `resources/maps` or the network.
- `game/GameUpdates.ts` describes the granular update stream emitted every tick. `GameView.ts` turns raw updates into view models consumed by the renderer.
- Specialized modules (`RailNetwork*.ts`, `TrainStation.ts`, `AllianceImpl.ts`, `AttackImpl.ts`, `TerraNulliusImpl.ts`, etc.) encapsulate specific gameplay mechanics. When you tweak a mechanic, look here before touching the client/server.

## Execution pipeline

- `GameRunner.ts` orchestrates turn playback. It buffers incoming turns, uses an `Executor` (`execution/ExecutionManager.ts`) to translate intents into executable actions, and pushes `GameUpdateViewData` back to its caller.
- `execution/` is split by concern: `alliance` covers alliance management, `nation` implements NPC behavior, and `utils` holds shared helpers. Executions are composed each tick by `Executor.createExecs()` and run in deterministic order.
- `validations/` contains data guards for usernames, colors, etc. These are enforced both server-side and inside the worker.

## Worker bridge

- `worker/WorkerClient.ts` runs in the browser main thread and exposes an async API (`initialize`, `sendTurn`, `playerProfile`, etc.). It marshals requests/responses across the worker boundary using `WorkerMessages.ts`.
- `worker/Worker.worker.ts` is the actual Web Worker bundle. It calls `createGameRunner()`, streams updates back with `postMessage`, services query requests (`player_profile`, `player_border_tiles`, `attack_average_position`, `transport_ship_spawn`), and executes turns when it receives `heartbeat` signals from the client.

## Utilities

- `EventBus.ts` implements a simple publish/subscribe system for UI and transport events.
- `PseudoRandom.ts` provides deterministic randomness seeded from game IDs.
- `Util.ts` contains helpers for hashing, ID generation, sanitization, archiving, and replay compression. These are shared by the server archiver, client replays, and worker simulation.

When making gameplay changes, ensure that any new types or logic are added here first—both the server and the client worker import this directory wholesale, so divergence can cause desyncs.
