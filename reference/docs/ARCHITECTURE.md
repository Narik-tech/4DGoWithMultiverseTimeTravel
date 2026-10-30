# go4d: Technical Architecture (Godot 4.7.x)

See [RULES.md](RULES.md) for the game rules. This document describes how to build them.

---

## 1. Principles

1. **Pure, deterministic core.** All game logic lives in `RefCounted` classes with no `Node`, scene,
   autoload or randomness dependencies. The same core then serves the UI, tests, replays, AI and netplay.
2. **Immutable boards, append-only multiverse.** A board never changes after it is created. Undo just removes the most recently created boards and timelines (LIFO).
3. **Results, not side effects.** Core operations return result objects (`ActionResult`, `EchoEvent`).
   The view layer turns those into animations. The core doesn't emit signals.
4. **Static typing everywhere** (`var x: int`, typed arrays, typed dictionaries) for speed and for catching errors early.
5. **Data-oriented positions.** A position is a `PackedByteArray` of N² cells plus a 64-bit Zobrist hash.

---

## 2. Layers

```
┌──────────────────────────────────────────────────────────────────┐
│ app/      Main menu, settings, mode select (hotseat / AI / net)  │
├──────────────────────────────────────────────────────────────────┤
│ ui/       HUD: turn banner, Submit/Undo, timeline list, score    │
│ view/     MultiverseView, BoardView, connectors, present line,   │
│           echo animator, camera, input → intents                 │
├──────────────────────────────────────────────────────────────────┤
│ session/  GameSession (Node): owns Game, translates intents,     │
│           emits signals, drives AI / network adapters            │
├──────────────────────────────────────────────────────────────────┤
│ core/     Game · Multiverse · Timeline · MvBoard · Position      │
│           GoRules · EchoResolver · Scoring · RuleSet · Notation  │
└──────────────────────────────────────────────────────────────────┘
        dependencies point downward only; core imports nothing above
```

---

## 3. Project layout

```
res://
├── project.godot
├── core/
│   ├── rules/rule_set.gd            # class_name RuleSet (Resource, serialisable)
│   ├── go/stone.gd                  # enum + helpers (EMPTY/BLACK/WHITE, invert)
│   ├── go/position.gd               # class_name Position  – cells + hash
│   ├── go/zobrist.gd                # fixed-seed key table per board size
│   ├── go/go_rules.gd               # static: groups, liberties, play, apply_echo
│   ├── multiverse/mv_board.gd       # class_name MvBoard
│   ├── multiverse/timeline.gd       # class_name Timeline
│   ├── multiverse/multiverse.gd     # class_name Multiverse – graph + queries
│   ├── actions/action.gd            # Place / Pass
│   ├── actions/action_result.gd     # created boards/timelines, captures, echoes
│   ├── actions/echo_resolver.gd     # capture → echo → cascade pipeline
│   ├── game/game.gd                 # class_name Game – turn state machine, undo
│   ├── game/scoring.gd              # Tromp-Taylor per timeline + aggregate
│   └── io/notation.gd, game_record.gd
├── session/game_session.gd          # Node; the only bridge core ↔ scene tree
├── view/
│   ├── multiverse_view.tscn/.gd
│   ├── board_view.tscn/.gd
│   ├── branch_connector.gd
│   ├── present_line.gd
│   ├── echo_animator.gd
│   ├── camera_controller.gd
│   └── layout.gd                    # pure math: (L, ply) → world position
├── ui/hud.tscn, timeline_navigator.tscn, score_panel.tscn, menus/
├── ai/ (later)  net/ (later)
├── assets/  (stone/board textures, sfx, shaders)
└── tests/   (GUT or gdUnit4 suites + scenario fixtures)
```

---

## 4. Core data model

### 4.1 `Stone` constants
```gdscript
enum { EMPTY = 0, BLACK = 1, WHITE = 2 }
static func invert(c: int) -> int: return 3 - c      # BLACK<->WHITE
static func side_to_move(ply: int) -> int: return BLACK if ply % 2 == 0 else WHITE
```

### 4.2 `Position` (immutable value)
```gdscript
class_name Position extends RefCounted
var size: int
var cells: PackedByteArray   # index = y * size + x
var hash: int                # Zobrist, 64-bit

func at(i: int) -> int
func with_cells(changes: Dictionary[int, int]) -> Position   # copy + incremental hash
func equals(o: Position) -> bool   # hash check first, then byte compare
```

### 4.3 `MvBoard` (a node in the multiverse graph)
```gdscript
class_name MvBoard extends RefCounted
enum Origin { ROOT, PLAY, PASS, ECHO }
var id: int
var timeline: int            # L index
var ply: int
var position: Position
var parent_id: int           # -1 for root
var origin: Origin
var move_point: int = -1     # placement index for PLAY (for last-move marker)
var consecutive_passes: int  # 2 ⇒ settled
var created_turn: int        # global turn counter that created it (undo / tentative)

func side_to_move() -> int: return Stone.side_to_move(ply)
func is_settled() -> bool:  return consecutive_passes >= 2
```

### 4.4 `Timeline`
```gdscript
class_name Timeline extends RefCounted
var index: int               # 0, +k (Black), -k (White)
var creator: int             # Stone color; EMPTY for L0
var start_ply: int
var branch_parent_id: int    # board id it branched from (-1 for L0)
var board_ids: PackedInt32Array   # board_ids[ply - start_ply]

func tip_id() -> int: return board_ids[board_ids.size() - 1]
func tip_ply() -> int: return start_ply + board_ids.size() - 1
```

### 4.5 `Multiverse`
```gdscript
class_name Multiverse extends RefCounted
var rules: RuleSet
var boards: Dictionary[int, MvBoard]
var timelines: Dictionary[int, Timeline]
var n_black_tl: int   # count of +k timelines
var n_white_tl: int   # count of -k timelines

# queries
func tip(L: int) -> MvBoard
func ancestor_at(board: MvBoard, ply: int) -> MvBoard   # walk parent_id
func history(board: MvBoard) -> Array[MvBoard]          # for superko (cache hashes)
func is_active(L: int) -> bool
func present_ply() -> int        # min tip ply over active & unsettled; -1 if none
func playable(color: int) -> Array[MvBoard]
func mandatory(color: int) -> Array[MvBoard]

# mutation (only called by Game / EchoResolver)
func append_board(L: int, parent: MvBoard, pos: Position, origin, ...) -> MvBoard
func create_timeline(creator: int, branch_from: MvBoard) -> Timeline
func remove_last_board(id: int)       # undo; must be the newest id
func remove_last_timeline(L: int)     # undo; must be the newest on its side
```

Activity check (RULES §7):
```gdscript
func is_active(L: int) -> bool:
    if L == 0: return true
    if L > 0:  return L <= n_white_tl + 1
    return -L <= n_black_tl + 1
```

### 4.6 `Action` and `ActionResult`
```gdscript
class_name Action extends RefCounted
enum Kind { PLACE, PASS }
var kind: Kind
var board_id: int
var point: int = -1

class_name ActionResult extends RefCounted
var ok: bool
var error: String                     # "occupied", "suicide", "superko", "not_playable"…
var action: Action
var created_boards: PackedInt32Array  # in creation order (for undo + view)
var created_timelines: PackedInt32Array
var captured: PackedInt32Array        # points removed on the played board
var echoes: Array[EchoEvent]          # for animation, in resolution order

class_name EchoEvent extends RefCounted
var color: int
var points: PackedInt32Array
var source_board_id: int
var target_board_id: int
var result_board_id: int              # -1 if fizzled / dissipated
var annihilated: PackedInt32Array
var cascade_depth: int
```

### 4.7 `Game` (turn state machine)
```gdscript
class_name Game extends RefCounted
enum Phase { PLAYING, GAME_OVER }
var mv: Multiverse
var phase: Phase
var turn_owner: int
var turn_counter: int
var draft: Array[ActionResult]        # this turn's tentative actions (undo stack)
var acted_boards: Dictionary[int, bool]
var result: ScoreResult

func try_action(a: Action) -> ActionResult
func undo() -> ActionResult           # pops draft
func can_submit() -> bool
func submit() -> bool                 # commits draft, flips owner or ends game
func resign(color: int)
```

---

## 5. Key algorithms

### 5.1 Groups and liberties (`GoRules`)
Flood fill over `PackedByteArray` with a precomputed neighbor table per board size
(`PackedInt32Array` of 4 neighbors per point, -1 off-board). Use an iterative stack, not recursion.
On 9×9 this costs microseconds.

```gdscript
static func play(pos: Position, p: int, color: int) -> PlayResult   # ok, new_pos, captured, error
static func apply_echo(pos: Position, pts: PackedInt32Array, color: int,
                       mode: RuleSet.Collision) -> EchoApply      # new_pos, annihilated, cascade_captured
static func groups_without_liberties(pos, color, seeds) -> PackedInt32Array
```

### 5.2 Applying a placement
```
try_action(PLACE b, p):
  validate: phase, b is a tip, !settled, side_to_move == turn_owner, b ∉ acted_boards
  r = GoRules.play(b.position, p, owner)            → error out if illegal
  superko: r.new_pos.hash ∉ mv.history_hashes(b)    → error "superko"
  child = mv.append_board(b.timeline, b, r.new_pos, PLAY)
  result.created_boards += child
  if r.captured and rules.echo_enabled:
      EchoResolver.resolve(mv, source=b, captured=r.captured, color=owner, result)
  acted_boards[b.id] = true;  draft.push(result)
```

### 5.3 `EchoResolver.resolve`
```
queue = [Echo(color, captured, from_board=source, depth=0)]   # PER_GROUP: one entry per group
while queue not empty:
  e = queue.pop_front()
  n = e.points.size()
  earliest = 0 if e.color == BLACK else 1
  t = max(e.from_board.ply - 2*n, earliest)
  target = mv.ancestor_at(e.from_board, t)
  a = GoRules.apply_echo(target.position, e.points, e.color, rules.echo_collision)
  ev = EchoEvent(...)
  if a.new_pos.equals(target.position):       ev.fizzled → continue
  if mv.timelines.size() >= rules.max_timelines: ev.dissipated → continue
  tl = mv.create_timeline(e.color, target)
  nb = mv.append_board(tl.index, target, a.new_pos, ECHO)
  result.created_timelines += tl.index;  result.created_boards += nb.id
  if a.cascade_captured and rules.cascades_enabled
     and t > earliest and e.depth + 1 <= rules.max_cascade_depth:
       queue.push_back(Echo(e.color, a.cascade_captured, from_board=target, depth+1))
  result.echoes.append(ev)
```
`target` is reached by walking parent links, so a cascade follows the history of the new timeline,
which is the same as `target`'s history. The order is deterministic, and that matters for netplay.

### 5.4 Present and submit
```gdscript
func present_ply() -> int:
    var p := -1
    for L in timelines:
        if not is_active(L): continue
        var t := timelines[L]
        if boards[t.tip_id()].is_settled(): continue
        p = t.tip_ply() if p < 0 else mini(p, t.tip_ply())
    return p

func can_submit() -> bool:
    var p := mv.present_ply()
    return p < 0 or Stone.side_to_move(p) != turn_owner
```
On `submit()`: clear `draft` and `acted_boards`, then check for game end (`present_ply() == -1`, which
means every active timeline is settled). Otherwise `turn_owner = side_to_move(present_ply())`. Under
the rules this always equals the opponent. Assert it in debug builds.

### 5.5 Undo
Each `ActionResult` lists what it created. To undo, remove its boards in reverse creation order and
its timelines in reverse order, then restore `acted_boards`. Ids are allocated monotonically and
removed LIFO, so the counters (`next_board_id`, `n_black_tl`, `n_white_tl`) roll back cleanly.

### 5.6 Zobrist hashing and superko
- One key table per board size: `keys[point * 2 + (color - 1)]`. Generate it with a
  `RandomNumberGenerator` that uses a **fixed seed**, so the table is deterministic across machines.
- Store the hash on every `Position`. For superko, gather the history hashes by walking ancestors. Cache this
  per board as a lazily built `Dictionary[int, bool]`, or keep a persistent set per timeline segment if profiling shows a need.

### 5.7 Scoring (`Scoring`)
Score each active tip with Tromp-Taylor: count stones, then flood-fill each empty region and award it
to the one color it touches, if any. White gets komi. The result has per-timeline scores,
per-timeline winners, the totals and the game winner, according to `rules.scoring`.

---

## 6. Presentation layer

### 6.1 Layout math (`view/layout.gd`, pure functions)
```gdscript
const CELL := 24.0                        # px per intersection at zoom 1
static func board_px(size: int) -> float: return (size + 1) * CELL
static func board_origin(L: int, ply: int, size: int) -> Vector2:
    var w := board_px(size)
    return Vector2(ply * (w + GAP_X), -L * (w + GAP_Y))   # Black timelines above L0
```
Optional: draw each ply pair (`b` and `w` boards) closer together, with a wider gap between turns,
as 5D Chess does.

### 6.2 Scenes
| Node | Responsibility |
|---|---|
| `MultiverseView` (Node2D) | Listens to `GameSession` signals. Spawns or frees `BoardView`s and connectors, and owns the layers (connectors < boards < overlays). |
| `BoardView` (Node2D) | Draws its grid, stones, last-move marker and state frame (playable / mandatory / inactive / settled / echo-born / tentative) with `_draw()`. Boards are immutable, so it draws once and caches. Handles hover and click through a `get_point_at(local_pos)` helper. |
| `BranchConnector` | A Bézier `Line2D` from the parent's right edge to the child's left edge. |
| `PresentLine` | A vertical band at the present ply, tinted with the turn owner's color. |
| `EchoAnimator` | Uses `EchoEvent`s to tween ghost stones from the source board to the target board along an arc, flip their color halfway, then flash any annihilations. |
| `CameraController` (Camera2D) | Pan (drag or WASD), zoom to cursor, "jump to present" (Space), focus a board (double-click). |
| `HUD` (CanvasLayer) | Turn banner, a mandatory-boards counter, Submit (Enter), Undo (Ctrl+Z), Pass-on-hovered (P), score preview. |
| `TimelineNavigator` | A minimap of the timeline tree. Clicking an entry moves the camera there. |

### 6.3 Rendering at scale
- **LOD**: below a zoom threshold, each `BoardView` switches to an N×N `ImageTexture`, one pixel per point,
  drawn with nearest filtering. Dozens or hundreds of boards stay cheap. Build it once per board, because boards are immutable.
- **Culling**: use a `VisibleOnScreenNotifier2D` per board, or a manual viewport-rect test in `MultiverseView`.
- If profiling later shows the need, batch all stones into one `MultiMeshInstance2D` per color.

### 6.4 Input flow
`BoardView` turns a click into an intent like `{place, board_id, point}`. `GameSession.request_action()` passes it to
`Game.try_action()`. The `ActionResult` comes back and `GameSession` emits signals.
The view never changes core state directly. Illegal moves show a small toast with `result.error`.

---

## 7. Session layer

`GameSession` (Node; an autoload, or the root of the game scene) is the only bridge between core and nodes.

```gdscript
signal board_created(board: MvBoard, tentative: bool)
signal board_removed(board_id: int)                 # undo
signal timeline_created(tl: Timeline)
signal timeline_removed(L: int)
signal echo_resolved(ev: EchoEvent)
signal present_changed(ply: int, owner: int)
signal turn_submitted(owner: int)
signal game_over(result: ScoreResult)

func new_game(rules: RuleSet)
func request_action(a: Action) -> ActionResult
func request_undo()
func request_submit()
func load_record(rec: GameRecord)                    # replays through Game
```
`PlayerController` adapters plug in here: `LocalPlayer` (input), `AIPlayer` (runs on `WorkerThreadPool` and gets a
`Game` clone or a replayed copy) and `NetPlayer`. Each adapter supplies a list of actions for a turn.

---

## 8. Persistence and notation

- **Board coordinate**: `L{±k}/T{turn}{b|w}`, for example `L+1/T4w`. **Point**: Go letters without `i`, plus the
  row number, for example `c3`.
- **Action**: `L0/T6b:d4`, or `L+1/T4w:pass`.
- **Turn**: the space-separated actions submitted together.
- **Game record** (JSON, `user://saves/*.go4d.json`):
```json
{ "version": 1,
  "rules": { "board_size": 9, "komi": 7.0, "echo_collision": "ANNIHILATE", "...": "..." },
  "turns": [ ["L0/T1b:e5"], ["L0/T1w:c3"], ["L0/T6b:d4"], ["L+1/T4w:e4", "L0/T6w:pass"] ],
  "result": null }
```
Echoes and timelines aren't stored. A replay recreates them deterministically. You can also store a
per-turn hash of the multiverse, to detect desync and to check replays.

---

## 9. Testing

- Use **GUT** or **gdUnit4**, whichever supports 4.7 at the time. Run headless in CI:
  `godot --headless --path . -s addons/gut/gut_cmdln.gd` (or the gdUnit4 runner).
- **Unit tests**: liberties and captures, suicide, ko and superko, Tromp-Taylor scoring, Zobrist
  incremental updates matching a full recompute.
- **Multiverse tests**: ancestor walks across branch points, activity counts, present calculation, submit
  rules, undo restoring everything exactly (compare with a serialized snapshot).
- **Echo tests**: fixtures for each collision mode, clamping at ply 0 and 1, fizzles, cascades,
  the depth limit, dissipation at the timeline cap, and `PER_MOVE` vs `PER_GROUP`.
- **Scenario fixtures**: small `.go4d.json` records with expected final hashes, such as the worked example in RULES §11.
- **Property or fuzz test**: random legal play for N turns. Check invariants on every submit (owner flips,
  tips' parity matches side, undo-all followed by replay gives an identical hash).

---

## 10. Performance budget (9×9, targets)

| Item | Target |
|---|---|
| `try_action` incl. echo cascade | < 1 ms |
| Boards in a long game | 2–5k (≈ 100 bytes each + object overhead) |
| Visible boards at 60 fps | 300+ using LOD |
| Full replay of a 200-turn record | < 200 ms |

If AI search needs more speed, move `GoRules` and `EchoResolver` into a **GDExtension** (C++ or Rust)
behind the same interface. The deterministic tests keep the two implementations matching.

---

## 11. Extension points

- **AI**: start with random legal moves, then greedy (captures and echoes, plus timeline majority eval), then MCTS
  over whole turns. The branching factor is large, so sample per-board candidates.
- **Online**: lockstep. Send each submitted turn's action list. Both peers replay it and compare state hashes.
  Use Godot's high-level multiplayer (ENet or WebSocket) or a small relay server.
- **Variants**: 4D adjacency, voluntary time-travel placement, handicap stones. Each goes behind a `RuleSet` flag and
  inside `GoRules` / `EchoResolver` only.
