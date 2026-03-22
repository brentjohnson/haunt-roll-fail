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
- `game-*.scala` — Game logic split by concern (battle, movement, leaders, etc.) when `game.scala` gets large.
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

Akka HTTP server that serves static files from the `haunt-roll-fail/` build output and provides a REST API for game journals (append-only log of serialized game actions per session). Uses HSQLDB via Slick for persistence. The backend is game-agnostic — it stores journal entries as opaque strings and has no knowledge of individual game rules.

### Key Conventions

- **`colmat` idioms** are used everywhere: `$[T]` (List), `|[T]` (Option), `./(f)` (map), `.%(f)` (filter), `.num` (length), `.any` (nonEmpty), `.|(default)` (getOrElse), `./~(f)` (flatMap). Read `colmat.scala` before editing any game logic.
- **Scala 2.13** with postfix ops and implicit conversions enabled. Compiler warnings for common patterns are suppressed in `common.sbt`.
- **`GoodMatch`** trait marks types safe to use in pattern match position; `@@` is used as a typed match operator.
- Each game is self-contained in its subdirectory and registered through its `MetaGame` implementation.
- `BuildInfo` (auto-generated at compile time) provides `version`, `time`, and `seed` values used for cache-busting.

---

## Implementing a New Game

All existing games follow the same pattern. Use `coup` (simpler) or `yarg` as reference implementations. Use a short lowercase package name (3–4 characters: `coup`, `yarg`, `inis`, etc.).

### Step 1: Add the source directory to `common.sbt`

```scala
Compile / unmanagedSourceDirectories += baseDirectory.value / "mygame"
```

### Step 2: Create `mygame/package.scala`

The package object declares the `Gaming` mix-in and wires the type aliases used throughout all other files in the package.

```scala
package object mygame extends hrf.base.Gaming with hrf.bot.BotGaming with hrf.ui.GreyUI {
    type F = Faction
    type G = Game

    val gaming = this

    val styles = elem.styles
}
```

Include `hrf.base.SelectSubset` if the game needs subset-selection actions (see `coup`). The `GreyUI` mix-in provides the standard grey-board rendering infrastructure used by most games.

### Step 3: Create `mygame/game.scala`

Define factions, game state, and all `Action` subtypes.

- **Factions** extend `BasePlayer` (and typically `Named`, `Styling`, `Record`). Each faction object is the canonical player identity throughout the system.
- **Game state** is a mutable class (not a case class) that holds all game state. It is reconstructed by replaying actions from the journal.
- **Actions** are `case class`/`case object` that extend either `ForcedAction` (engine-driven, not player choices) or `UserAction` (player choices presented in the UI). All action types must extend `Record` and `ActionClass[Self]`. The serializer uses reflection on `Record` subtypes — every action case class/object must be directly reachable from the package at compile time.
- **`StartAction`** (from `hrf.base`) is the mandatory first action in every journal; `gaming.version` is baked in for compatibility checks.

```scala
package mygame

import hrf.colmat._
import hrf.logger._
import hrf.tracker._
import hrf.elem._

trait Faction extends BasePlayer with Named with Styling { ... }
case object PlayerA extends Faction { ... }
case object PlayerB extends Faction { ... }

// All actions must be Records and ActionClass[Self]
case class SomeAction(f : Faction, n : Int) extends ForcedAction with ActionClass[SomeAction]
case class SomeChoice(f : Faction) extends UserAction with ActionClass[SomeChoice]
case class GameOverAction(winner : Faction) extends ForcedAction with ActionClass[GameOverAction]

class Game(val setup : $[Faction]) extends BaseGame {
    // mutable game state fields
    var currentPlayer : Faction = setup.head
    // ...

    def perform(action : Action)(using gaming.type) : Receive = action @@ {
        case StartAction(v) =>
            // initialize game state
            Nil
        case SomeAction(f, n) =>
            // mutate state, return list of next forced actions or user actions
            $(SomeChoice(f))
        case SomeChoice(f) =>
            // ...
            Nil
    }
}
```

The `perform` method returns a `Receive` (a list of next actions). Returning `Nil` means it is the next player's turn to provide a `UserAction`.

### Step 4: Create `mygame/serialize.scala`

This is almost always a three-liner. The `Serializer` uses Scala reflection to serialize/deserialize `Record` case classes by name automatically, using the given `prefix` to namespace the action names in the journal.

```scala
package mygame

import hrf.colmat._
import hrf.logger._
import hrf.serialize._

object Serialize extends Serializer {
    val gaming = mygame.gaming

    def writeFaction(f : F) = Meta.writeFaction(f)
    def parseFaction(s : String) = Meta.parseFaction(s)

    val prefix = "mygame."
}
```

### Step 5: Create `mygame/meta.scala`

`MetaGame` is the registry entry for the game. It must implement all abstract members of `MetaBase` and `MetaBots`.

```scala
package mygame

import hrf.colmat._
import hrf.logger._
import hrf.meta._
import hrf.elem._

object Meta extends MetaGame { mmm =>
    val gaming = mygame.gaming

    type F = Faction

    def tagF = implicitly

    val name = "mygame"        // must be unique; used in URLs and journal IDs
    val label = "My Game"      // display name

    val factions = $(PlayerA, PlayerB)

    val minPlayers = 2
    // override val maxPlayers = 2   // defaults to factions.num

    val options = $            // list of GameOption instances, or $ for none

    val quickMin = 2
    val quickMax = 2

    def randomGameName() = "My Game " + random(1000)

    def validateFactionCombination(factions : $[Faction]) =
        (factions.num < 2).?(ErrorResult("Need at least 2")).|InfoResult("")

    def validateFactionSeatingOptions(factions : $[Faction], options : $[O]) = InfoResult("")

    def factionName(f : Faction) = f.name
    def factionElem(f : Faction) = f.name.styled(f)

    def createGame(factions : $[Faction], options : $[O]) = new Game(factions)

    def getBots(f : Faction) = $("Normal")
    def getBot(f : Faction, b : String) = new BotXX(f)
    def defaultBot(f : Faction) = "Normal"

    def writeFaction(f : Faction) = f.short      // short string key for each faction
    def parseFaction(s : String) : |[Faction] = factions.%(_.short == s).single

    def writeOption(f : O) = "n/a"
    def parseOption(s : String) = $

    def parseAction(s : String) : Action = Serialize.parseAction(s)
    def writeAction(a : Action) : String = Serialize.write(a)

    val start = StartAction(gaming.version)

    val assets =
        ConditionalAssetsList((factions : $[Faction], options : $[O]) => true)(
            ImageAsset("some-image") ::
        $) :: $
}
```

Key `MetaGame` fields:
- `name` — short ID used in URLs, journal paths, and serialization namespacing. Must be unique across all games.
- `assets` — list of `ConditionalAssetsList`, each gated by a condition on factions/options. `ImageAsset(name)` resolves to `<name>.png` by default; override `ext`, `path`, `lzy` (laziness), `scale` as needed.
- `start` — always `StartAction(gaming.version)`.
- `quickMin`/`quickMax` — player count range for the "quick game" lobby feature.

### Step 6: Create `mygame/ui.scala`

```scala
package mygame

import hrf.colmat._
import hrf.logger._
import hrf.elem._
import hrf.html._
import hrf.ui._
import hrf.ui.again._   // or hrf.ui.panes._

object UI extends BaseUI {
    val mmeta = Meta

    def create(uir : ElementAttachmentPoint, arity : Int, options : $[mmeta.O], resources : Resources, title : String, callbacks : hrf.Callbacks) =
        new UI(uir, arity, resources, callbacks)
}

class UI(val uir : ElementAttachmentPoint, arity : Int, val resources : Resources, callbacks : hrf.Callbacks) extends GUI {
    def factionElem(f : Faction) = f.name.styled(f)

    // Define panes for each section of the UI
    val board = newPane("board", Content, styles.board)

    def updateStatus() {
        // render per-faction status panels
    }

    def updateGame() {
        // render the main board/game state
    }

    def renderActions(actions : $[UserAction]) {
        // present clickable action buttons to the current player
    }
}
```

The `GUI` base class (from `hrf.ui`) provides `currentGame : G`, `game : G` (alias), `newPane`, `newOuterPane`, and the rendering lifecycle hooks `updateStatus`, `updateGame`, `renderActions`.

### Step 7: Create `mygame/bot.scala`

```scala
package mygame

import hrf.colmat._
import hrf.logger._
import hrf.bot._

class BotXX(val faction : Faction) extends Bot {
    val gaming = mygame.gaming

    def ask(actions : $[UserAction], depth : Int)(implicit game : Game) : UserAction = {
        // return one of the provided actions
        actions.shuffle.head
    }
}
```

### Step 8: Register the game in `hrf.scala`

Add the new `Meta -> UI` pair to the `metaUIs` list in `hrf.scala`:

```scala
val metaUIs : $[(MetaGame, BaseUI)] = $(
    // ... existing entries ...
    mygame.Meta -> mygame.UI,
)
```

This is the single wiring point that makes the game visible to the lobby and routing system. No backend changes are needed — the server is game-agnostic.

### Step 9: Create `mygame/styles.scala` (if needed)

Games that need custom CSS define an `elem` sub-package with a `styles` object extending `hrf.elem.BaseStyleMapping`:

```scala
package mygame

package object elem {
    object styles extends hrf.elem.BaseStyleMapping("mygame") {
        // define style objects here
    }
}
```

The string argument is the CSS class prefix. Style objects are referenced via `"text".styled(styleObj)` or `Image("name", styles.someStyle)` in game and UI code.
