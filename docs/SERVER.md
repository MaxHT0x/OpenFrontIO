# Server guide

The game server is a clustered Node.js application that supervises lobby scheduling, hosts game simulations, and exposes HTTP/WebSocket APIs. All source lives under `src/server` and relies on shared configuration from `src/core/configuration`.

## Process roles

- **Primary process (`Server.ts`)** – Reads configuration via `getServerConfigFromServer()`, optionally provisions Cloudflare tunnels for production, forks `config.numWorkers()` worker processes with `cluster`, and starts the master Express server.
- **Master HTTP server (`Master.ts`)** – Serves `/api/env` (environment discovery for clients), `/api/public_lobbies`, and admin-only kick endpoints. It proxies static assets from the webpack `static/` directory, rate-limits requests, and manages the playlist of public games through `MapPlaylist`.
- **Workers (`Worker.ts`)** – Each worker launches Express + `ws`, mounts rate limiting, compression, privilege refreshing, and JWT verification, and exposes endpoints to create/start/end games. Workers forward WebSocket messages to `GameManager`/`GameServer` instances and publish metrics if OpenTelemetry is enabled.

## Game lifecycle

1. **Lobby creation** – `GameManager.createGame()` instantiates a `GameServer` with merged defaults and requested `GameConfig`. Public lobbies are auto-seeded by the master using `MapPlaylist`; private lobbies are created by REST clients.
2. **Client connection** – `Worker.ts` wraps accepted sockets in `Client` objects, validates tokens via `verifyClientToken`/`getUserMe` (`jwt.ts`), and registers them with the appropriate `GameServer`.
3. **Prestart/start** – `GameManager.tick()` transitions lobbies from `Lobby` to `Active`, triggers `GameServer.prestart()` (prompting clients to preload terrain maps), then delays `GameServer.start()` to give players time to join.
4. **Turn loop** – Each `GameServer` buffers intents received through WebSockets, assembles turns, advances the shared simulation, and broadcasts `ServerTurnMessage`s. Hash reports and winner votes are aggregated to detect desyncs and declare victors.
5. **Archiving & cleanup** – Completed games are finalized via `finalizeGameRecord()` and persisted with `archive()` (`Archive.ts`) using the configured API endpoint. `GameManager.tick()` prunes finished games from its map.

## Supporting services

- **Privilege refresher (`PrivilegeRefresher.ts`)** – Periodically fetches cosmetics metadata and builds a `PrivilegeChecker` used to gate premium cosmetics. Fails open when the endpoint cannot be reached, so monitor logs.
- **Metrics (`WorkerMetrics.ts` + `OtelResource.ts`)** – When `otelEnabled()` is true, workers expose gauges for active games, connected clients, and heap usage. Exported to the configured OTLP endpoint every 15 seconds.
- **Logging (`Logger.ts`)** – Provides a Winston logger hierarchy. Master/worker components create child loggers (`comp` field) for consistent structured output.
- **Cloudflare integration (`Cloudflare.ts`)** – Automates tunnel creation/startup for prod/preprod environments. Reads account details from `ServerConfig` and binds worker subdomains to local ports.

## HTTP & WebSocket surface

- **Environment discovery:** `GET /api/env` (master) returns `{ game_env }` so the client can load the correct `ServerConfig`.
- **Lobby discovery:** `GET /api/public_lobbies` (master) returns cached lobby info. Individual workers also expose `/api/game/:id` for admin diagnostics and `/api/create_game/:id`/`/api/start_game/:id` for lobby management.
- **Admin control:** `POST /api/kick_player/:gameID/:clientID` (master) forwards to the worker hosting the game, validating via the configured admin header/token.
- **Gameplay WebSockets:** Each worker hosts `ws://<host>/` connections. Messages must satisfy `ClientMessageSchema`; `GameServer` responds with `ServerMessage` payloads. Hash checking and winner voting flows use dedicated message types defined in the shared schemas.

## Configuration and environment

- `ServerConfig` (see [Core logic guide](./CORE.md)) abstracts access to admin tokens, worker counts, Cloudflare info, JWT metadata, OTLP targets, and R2 storage. Implementations live in `src/core/configuration`.
- Environment variables such as `GAME_ENV`, `ADMIN_TOKEN`, Cloudflare credentials, and Docker registry settings can be set via `.env`. `example.env` documents all available keys.
- Deployment scripts (`build.sh`, `deploy.sh`, `update.sh`) rely on these values to build Docker images, update version files, and roll out updates to remote hosts.

When you change server behavior, check whether the corresponding schema, client transport, and worker simulation need updates—the cluster expects all three tiers to agree on the protocol and configuration.
