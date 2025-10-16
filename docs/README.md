# OpenFrontIO Developer Handbook

Welcome to OpenFrontIO! This handbook is the primary orientation guide for contributors. It explains how the codebase fits together, how to run the project locally, and where to look when you need to make a change. The remaining markdown files in this folder deep dive into specific areas—use this README as the jumping-off point.

## Getting started quickly

1. **Install dependencies**
   ```bash
   npm install
   ```
2. **Run the full development stack (webpack dev server + Node game server)**
   ```bash
   npm run dev
   ```
3. **Run individual targets when you only need one side**
   ```bash
   npm run start:client     # hot-reloads the Pixi.js client
   npm run start:server-dev # boots the Node master/worker server in dev mode
   ```
4. **Quality gates**
   ```bash
   npm test       # Jest test suite (shared logic + server + client utilities)
   npm run lint   # ESLint using the repository rules
   npm run format # Prettier formatting sweep
   npm run perf   # Deterministic performance harness in tests/perf
   ```

Development expects Node 20+ with npm 10.9.2 or newer. Copy `example.env` to `.env` and fill in credentials (admin token, Cloudflare tunnel access, Docker publishing info, etc.) when you need to exercise infrastructure or deployment scripts. Cloudflare tunnels are only started automatically outside `dev` mode, so local development just uses direct sockets.

## Repository map

- [`src/client`](./CLIENT.md) – browser client (Pixi renderer, UI components, transports, user settings).
- [`src/core`](./CORE.md) – shared simulation, schemas, configuration, worker code. Imported by both client and server.
- [`src/server`](./SERVER.md) – Node master/worker game server, REST API endpoints, websocket orchestration.
- [`resources`](./RESOURCES.md) – static content shipped with builds (maps, translations, sprites, cosmetics, audio).
- [`map-generator`](./RESOURCES.md#map-generation-tooling) – Go utility for producing binary maps and manifests.
- [`tests`](./RESOURCES.md#testing-and-quality) – Jest unit/integration tests plus performance benchmarks.
- Root scripts (`build.sh`, `deploy.sh`, `update.sh`, `setup.sh`) – automation for Docker builds, deployment, and environment bootstrap.

## Navigating the rest of the docs

The remainder of the documentation is divided by responsibility:

- [Architecture overview](./ARCHITECTURE.md) explains the runtime topology, process model, and how shared logic travels between client and server.
- [Client guide](./CLIENT.md) walks through UI entry points, rendering, transports, and how gameplay state is projected into the DOM.
- [Server guide](./SERVER.md) documents the cluster layout, HTTP/WebSocket APIs, lobby scheduling, and supporting services (logging, auth, metrics).
- [Core logic guide](./CORE.md) catalogs the simulation code, schemas, configuration layers, and web worker bridge used by both tiers.
- [Resources & tooling](./RESOURCES.md) details assets, localization, map data, test suites, and build/deploy automation.

Skim the architecture chapter first, then drill into the subsystem you plan to change. Each section links back to concrete files so you can jump straight into the implementation.
