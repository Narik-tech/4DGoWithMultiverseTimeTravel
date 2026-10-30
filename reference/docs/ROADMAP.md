# go4d: Implementation Roadmap

Rules: [RULES.md](RULES.md) · Architecture: [ARCHITECTURE.md](ARCHITECTURE.md)

The core comes first and is fully tested headless. The first playable build is a hotseat prototype at **M5**.

| # | Milestone | Depends on | Size |
|---|---|---|---|
| M0 | Project setup | – | S |
| M1 | Single-board Go core | M0 | M |
| M2 | Multiverse model | M1 | M |
| M3 | Echo mechanic | M2 | M |
| M4 | Turn controller, end of game, scoring | M3 | M |
| M5 | Playable prototype view (hotseat) | M4 | L |
| M6 | UX polish and readability | M5 | L |
| M7 | Save, load and replays | M4 (core), M5 (UI) | S |
| M8 | AI opponent | M4 | L |
| M9 | Online multiplayer | M7 | L |

---

## M0: Project setup
- Create a Godot 4.7.x project and the folder layout from ARCHITECTURE §3. Put it in git, with a `.gitignore` for `.godot/`.
- Install the test framework (GUT or gdUnit4). Add a headless test script, and CI (GitHub Actions with a Godot headless image).
- Turn on GDScript warnings as errors for `core/`: untyped declarations and unsafe calls.
- **Done when** an empty test suite passes in CI.

## M1: Single-board Go core
- `Stone`, `Position`, `Zobrist`, `GoRules` (neighbor tables, groups, liberties, `play`, suicide check).
- Simple ko and positional superko over a plain list of hashes.
- Tromp-Taylor `Scoring` for a single position.
- **Done when** tests cover captures (single, multi-group, edge, corner), suicide, ko, superko, scoring, and Zobrist incremental updates matching a full recompute.

## M2: Multiverse model
- `MvBoard`, `Timeline`, `Multiverse`: append, branch, `ancestor_at`, history hashes, activity, `present_ply`, playable and mandatory queries, LIFO removal.
- **Done when** tests show the ancestor walk crossing branch points, that the activity formula matches RULES §7 for different nB/nW, and that the present skips inactive and settled timelines.

## M3: Echo mechanic
- `GoRules.apply_echo` with all three collision modes, and `EchoResolver` with clamping, fizzles, cascades, the depth cap and the timeline cap.
- Add both grouping modes, and wire `ActionResult` / `EchoEvent`.
- **Done when** the RULES §11 worked example passes as a fixture and every rule option in RULES §12 that affects echoes has at least one test. A fuzz test also passes: 1,000 random games with echoes on, invariants hold, and nothing crashes.

## M4: Turn controller, end of game, scoring
- `Game`: drafts, the `acted_boards` guard, undo, `can_submit` and `submit`, passing and settling, resigning, `max_ply`.
- Aggregate scoring (`TIMELINE_MAJORITY`, `TOTAL_AREA`, tie-breaks).
- A minimal **text/debug harness**: a headless script that plays a record and prints the multiverse as ASCII. It is very useful for debugging before any graphics exist.
- **Done when** the property test passes (the owner flips on every submit, and undo-all followed by replay gives an identical hash) and full games run start to finish headless.

## M5: Playable prototype view (hotseat)
- `GameSession` with signals; `Layout`; `MultiverseView`; `BoardView` (`_draw`, state frames, hover ghost stone); `BranchConnector`; `PresentLine`; `CameraController`.
- A HUD with Submit, Undo, Pass, a turn banner and a mandatory-board count. Illegal-move toasts.
- Echoes appear instantly at this stage, with no animation yet.
- **Done when** two people can play a full 7×7 or 9×9 game at one machine. Echoes and branches show up in the right place, and the end-of-game score screen is correct.

## M6: UX polish and readability
- An `EchoAnimator` arc that flips color in flight, plus annihilation flashes, and a camera that follows newly created past timelines.
- LOD textures and culling. Run a performance pass with 300+ boards.
- `TimelineNavigator` minimap, highlighting of a board's history chain when hovered, "jump to present", and highlights on mandatory boards.
- A live score preview for each timeline, warnings about dead stones before passing, sound effects, and a short interactive tutorial (place → capture → echo → present moves back).
- **Done when** a new player can tell, without help, where they must play and why the present moved.

## M7: Save, load and replays
- `Notation` and `GameRecord` (JSON), plus autosave every turn.
- A replay viewer that steps through turns and keeps camera bookmarks.
- **Done when** saving, then loading, then continuing produces the same hashes as uninterrupted play.

## M8: AI opponent
- Level 0: random legal moves on mandatory boards. Level 1: greedy heuristic covering captures and echo value, atari, and the timeline-majority evaluation.
- Level 2: MCTS over whole turns, with per-board candidate sampling, running on `WorkerThreadPool`.
- If needed, port `GoRules` and `EchoResolver` to a GDExtension and check it against the same test fixtures.

## M9: Online multiplayer
- Lockstep turn exchange (ENet or WebSocket), a desync check with per-turn state hashes, and reconnecting by replaying the record.
- Lobby or invite codes. Time controls (per-turn and total clocks, as in 5D Chess).

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| The multiverse gets unreadable | Default to 7×7 or 9×9, `max_timelines`, LOD, minimap, dimming inactive timelines, history highlighting. |
| Echo rules aren't fun as defined | Every echo behavior is a `RuleSet` option. Playtest with the M4 debug harness and the M5 hotseat build before polishing. |
| Rule edge cases make the game end ambiguously | Tromp-Taylor scoring, a deterministic resolution order, and `max_ply` as a backstop. |
| GDScript is too slow for AI | Keep the core interface narrow so it can be swapped for a GDExtension. |
| Godot 4.7.x API changes | Keep `core/` free of engine APIs beyond `RefCounted` and Packed arrays. Engine-dependent code goes in `view/` and `session/`. |

## Decisions to confirm before M3
1. Echo collision: `ANNIHILATE` (default) or `OVERWRITE`?
2. Echo grouping: `PER_MOVE` (default) or `PER_GROUP`?
3. Scoring: `TIMELINE_MAJORITY` (default) or `TOTAL_AREA`?
4. Default board size: 9×9 or 7×7?
