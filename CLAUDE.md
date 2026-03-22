# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repo contains three sub-projects, each with its own `sbt` build:

- **`scala-js-dom-reduced/`** — A local fork/reduction of the `scalajs-dom` library. Must be published locally before building the main app.
- **`haunt-roll-fail/`** — The main Scala.js frontend application (the browser-side game client).
- **`good-game/`** — The JVM backend server (Akka HTTP + Slick + HSQLDB), which hosts game sessions and persists game journals.

## Build Commands

### 1. Publish the local DOM library (only needed once or after changes)
```
cd scala-js-dom-reduced
sbt publishLocal
```

### 2. Build the frontend (Scala.js)
```
cd haunt-roll-fail
sbt fastOptJS
```
`fastOptJS` produces an unoptimized JS bundle (optimizer is explicitly disabled in `build.sbt`). Use `fullOptJS` for production.

### 3. Run the backend server
```
cd good-game
# First-time database creation:
sbt "run create ../good-game-database ../haunt-roll-fail http://localhost:7070 http://localhost:7070/hrf/ 7070"

# Normal startup:
sbt "run run ../good-game-database ../haunt-roll-fail http://localhost:7070 http://localhost:7070/hrf/ 7070"
```

The server arguments are: `<command> <db-dir> <static-files-dir> <base-url> <hrf-prefix-url> <port>`.

## Architecture

### Frontend (`haunt-roll-fail/`)

The frontend is a single Scala.js application (`hrf.HRF` main object in `hrf.scala`). Source directories are added via `common.sbt` — the root `.scala` files plus one directory per game (`arcs/`, `root/`, `coup/`, `inis/`, `cthw/`, `dwam/`, `vast/`, `doms/`, `suok/`, `sehi/`, `yarg/`, `bsg/`).

**Core framework packages (`hrf.*`):**
- `hrf.colmat` — Custom collection/option utilities. `$[T]` is an alias for `List[T]`, `|[T]` for `Option[T]`. Extensively used throughout all code.
- `hrf.base` — Core game abstractions: `Gaming` trait (defines `G`=game state, `F`=faction/player types), `Record` trait for serializable game actions, dice, timelines.
- `hrf.meta` — `MetaBase`/`MetaGame` traits that each game implements to register itself (name, options, image assets, UI entry points).
- `hrf.elem` / `hrf.html` — UI element DSL for building DOM trees.
- `hrf.ui` — Game UI rendering loop and interaction handling.
- `hrf.loader` — Asset loading (images, strings) with caching via browser Cache API.
- `hrf.journal` — Journal abstraction for reading/writing ordered game action logs to the server.
- `hrf.web` — HTTP utilities (get/post wrappers over XHR).
- `hrf.serialize` — Serialization for game records.
- `hrf.compute` — Computation/bot utilities.
- `hrf.bot` — Bot/AI base infrastructure.

**Per-game packages** (e.g., `arcs`, `root`, `coup`, etc.) each contain:
- `game.scala` — Game state types, factions, resources, board elements, all game-specific domain types.
- `game-*.scala` — Game logic split by concern (battle, movement, leaders, etc.).
- `meta.scala` — `MetaGame` implementation: registers options, image assets, serialization.
- `host.scala` — Game session hosting logic (turn management, action dispatch).
- `ui.scala` — UI rendering for that game.
- `serialize.scala` — Serialization/deserialization of game records.
- `bot*.scala` — Bot/AI implementations.
- `styles.scala` — Game-specific CSS styles.

**JVM-only files** (excluded from Scala.js build via `build.sbt`'s `excludeFilter`):
- `host-jvm.scala`, `reflect-jvm.scala`, `log-jvm.scala`, `grey-jvm.scala`, `timeline-jvm.scala`
- `host.scala`, `convert-images.scala`

### Backend (`good-game/`)

Akka HTTP server that serves static files from the `haunt-roll-fail/` build output and provides a REST API for game journals (append-only log of serialized game actions per session). Uses HSQLDB via Slick for persistence.

### Key Conventions

- **`colmat` idioms** are used everywhere: `$[T]` (List), `|[T]` (Option), `./(f)` (map), `.%(f)` (filter), `.num` (length), `.any` (nonEmpty), `.|(default)` (getOrElse), `./~(f)` (flatMap). Read `colmat.scala` before editing any game logic.
- **Scala 2.13** with postfix ops and implicit conversions enabled. Compiler warnings for common patterns are suppressed in `common.sbt`.
- **`GoodMatch`** trait marks types safe to use in pattern match position; `@@` is used as a typed match operator.
- Each game is self-contained in its subdirectory and registered through its `MetaGame` implementation.
- `BuildInfo` (auto-generated at compile time) provides `version`, `time`, and `seed` values used for cache-busting.
