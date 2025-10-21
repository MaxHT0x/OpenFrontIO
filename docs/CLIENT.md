# Client guide

The browser client is a TypeScript/webpack bundle that renders the game with Pixi.js, manages UI with custom web components, and streams gameplay updates from the Node server. Everything lives inside `src/client`.

## Entry points and boot sequence

- `Main.ts` is the webpack entry file. It imports global styles, registers custom elements (modals, buttons, inputs), fetches version data, initializes localization, and constructs the `Client` class.
- The `Client` class composes UI widgets like `UsernameInput`, `LangSelector`, `DarkModeButton`, `NewsModal`, `HelpModal`, and lobby modals. It wires the DOM to an `EventBus`, attaches button handlers, and coordinates login (`jwt.ts`) plus cosmetics loading (`Cosmetics.ts`).
- When a user joins a game (multiplayer lobby, single-player skirmish, or replay) the client dispatches a `join-lobby` event. `ClientGameRunner.joinLobby` creates a `Transport`, subscribes to lobby events, and kicks off the gameplay pipeline.

## Gameplay pipeline on the client

1. **Transport** – `src/client/Transport.ts` builds the WebSocket connection (or a `LocalServer` shim for replays/single-player). It translates DOM-driven intents (spawn, attack, donate, chat, emoji, alliance, etc.) into the shared Zod schemas and forwards them to the server. Incoming `ServerMessage` payloads are validated before they hit the UI.
2. **Worker bridge** – `ClientGameRunner` spins up a `WorkerClient` (`src/core/worker/WorkerClient.ts`), which loads the shared simulation inside `Worker.worker.ts`. Turns received from the server are pushed into the worker; the worker executes deterministic ticks and emits `GameUpdateViewData` back to the main thread.
3. **Rendering** – `createRenderer` in `src/client/graphics/GameRenderer.ts` consumes update events and draws the game board with Pixi.js. `graphics/layers` contains per-layer renderers (territories, units, overlays) while `graphics/fx` houses particle and shader effects. UI interaction state lives under `graphics/uiState`.
4. **Input system** – `InputHandler.ts` watches pointer/keyboard events and raises domain-specific `GameEvent`s (`SendAttackIntentEvent`, `SendUpgradeStructureIntentEvent`, etc.) that the transport consumes. It also supports replay speed, camera movement, and accessibility toggles.
5. **User settings & persistence** – `UserSettings.ts` in the shared core stores toggles such as dark mode, zoom preferences, and minimap visibility. The client persists session stats via `LocalPersistentStats.ts` and generates deterministic persistent IDs for anonymous players in `Main.ts`.

## UI components & styling

- `components/` contains lightweight web components for home-screen cards, lobby rows, modal overlays, etc. `components/baseComponents` defines reusable primitives (buttons, modals) while `components/ui` covers game HUD widgets.
- `styles.css` imports Tailwind utilities generated from `tailwind.config.js`. Additional CSS modules live under `styles/` (core layout, modal framing, component-specific themes). These classes coexist with Pixi rendering for HUD overlays.
- Internationalization data is loaded from `resources/lang/*.json`. `LangSelector.ts`, `LanguageModal.ts`, and helper utilities in `Utils.ts` handle locale switching and message formatting.
- Audio assets are orchestrated by `sound/SoundManager.ts`, which uses Howler.js to play effects defined in `resources/sounds`.

## Single-player, replays, and offline flows

- The `LocalServer` class mimics server turn progression locally using archived `GameRecord`s or newly spawned single-player sessions. It runs the shared simulation in-process and feeds the same message types back into the transport.
- Replays load compressed game records via the shared `Util` helpers (`createPartialGameRecord`, `decompressGameRecord`). The client verifies turn hashes against archived data and warns about desyncs.
- When the browser cannot reach a remote server, you can still exercise most flows (UI, renderer, worker bridge) by starting a single-player game, which never leaves the browser process.

## When to modify what

- Adjust HUD visuals, overlays, or gameplay feedback → edit `graphics/` and the relevant component under `components/ui`.
- Add or change lobby flows → update the modal/component files (`HostLobbyModal.ts`, `JoinPrivateLobbyModal.ts`, etc.) and their CSS.
- Introduce new intents or change gameplay messaging → extend the shared schemas in `src/core/Schemas.ts`, update `Transport.ts` to emit them, and handle them in the server `GameServer`.
- Touching localization or assets → update the files under `resources/` (see [Resources & tooling](./RESOURCES.md)).

Understanding this pipeline will help you trace a user action from the DOM, through the event bus, across the WebSocket, into the worker simulation, and back out to the renderer.
