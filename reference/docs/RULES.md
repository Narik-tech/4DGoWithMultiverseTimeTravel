# 4D Go with Multiverse Time Travel: Rules Specification

Codename: `go4d` · Target engine: Godot 4.7.x · Status: design draft v0.1

The four dimensions are **x, y** (the Go board), **time** (turns), and **timeline** (parallel universes).
Turn structure and board layout follow 5D Chess with Multiverse Time Travel. The new mechanic is the
**Echo**: captured stones travel back in time and flip color.

---

## 1. Glossary

| Term | Meaning |
|---|---|
| **Board** | One immutable N×N Go position at one moment of one timeline. |
| **Ply** | Half-turn index, starting at 0. Even ply = Black to move, odd ply = White to move. |
| **Turn** | `turn = ply / 2 + 1` (integer division). Written `T4b` / `T4w`. One turn is one Black move plus one White move. |
| **Timeline (L)** | A row of consecutive boards. The root is `L0`. Black-created timelines are `L+1, L+2…`, White-created ones are `L-1, L-2…`. |
| **Tip** | The latest board of a timeline. Only tips can be played on. |
| **History** | A board's chain of parent boards back to ply 0, which crosses into parent timelines before a branch point. |
| **Present** | The lowest tip ply among active, unsettled timelines. The side to move at the present owns the global turn. |
| **Active** | A timeline that counts for the present and for scoring (see §7). |
| **Echo** | The set of stones removed by one capture. They travel back in time with inverted color. |
| **Settled** | A timeline whose last two actions were passes. It is frozen. |

---

## 2. Setup

- One empty board, `L0` at ply 0 (`T1b`). Black moves first.
- Default board size is **9×9**. 5, 7, 13 and 19 are also allowed; small boards are recommended because the multiverse grows quickly.
- Komi is 7 points **per timeline** by default, and it is configurable.

---

## 3. Multiverse layout (same as 5D Chess)

```
            ply →   0     1     2     3     4     5     6     7
 L+1 (Black)                              ┌──[T3w]─[T4b]─[T4w] ← tip
                                          │
 L0  (root)       [T1b]─[T1w]─[T2b]─[T2w]─[T3b]─[T3w]─[T4b]─[T4w]─[T5b] ← tip
                                    │
 L-1 (White)                        └──[T2w]─[T3b] ← tip
                               ▲
                     present = lowest active tip ply
```

- The horizontal axis is ply (time). The vertical axis is the timeline index.
- A branch is drawn as a connector from the parent board to the first board of the new timeline. That first board sits one ply to the right of its parent.
- Black-created timelines stack on one side of `L0` and White-created ones on the other.

---

## 4. Turn structure

1. The **turn owner** is the side to move at the present ply.
2. A **playable board** for player P is any tip that is not settled and has P to move. This includes tips on inactive timelines.
3. On a playable board, P may **place** a stone or **pass**. Either action creates that timeline's next board, with the opponent to move.
4. **Mandatory boards**: every playable board that is active and sits at the present ply. P must act on all of them.
5. **Optional boards**: P may also act on any other playable board, at most once per board per turn. A board created this turn always has the opponent to move, so it can't be acted on again.
6. Actions are **tentative** until submitted, and P can undo them one at a time, most recent first, back to the start of the turn.
7. **Submit** is allowed when the side to move at the present ply is no longer P. If every active timeline is settled, the game ends (§9).

> Invariant: all tips at the same ply have the same side to move, because side is determined by ply parity.
> So "the present belongs to P" is always well defined. Every action only moves tips forward, so a
> player can always reach a submittable state, if necessary by passing.

---

## 5. Placing and capturing (per board, standard Go)

Each board follows normal Go rules in 2D. Adjacency is orthogonal and never crosses boards.

1. The stone must go on an empty intersection.
2. Remove every opponent group that has zero liberties. These stones are the **captured set**.
3. Suicide is illegal: if the new stone's group has zero liberties after step 2, the move is not allowed.
4. Superko: the resulting position must not appear earlier in this board's history (§8).
5. If the captured set is non-empty, an **Echo** is spawned (§6).

---

## 6. Echoes (the core mechanic)

When placing a stone on board **B** (ply `p`, owner P) captures a set **S** of `n = |S|` stones:

### 6.1 Color inversion
The echo stones take the color opposite to the stones they were. The captured stones were the
opponent's, so **an echo is always the capturer's color**. A capture plants reinforcements for you in the past.

### 6.2 Distance
- The echo travels `n` **turns** back, which is `2n` plies.
- Target ply: `t = p − 2n`. P is to move there, because a whole number of turns keeps the ply parity.
- Clamp: if `t` is below P's earliest ply (0 for Black, 1 for White), use the earliest ply instead.
- Grouping (default `PER_MOVE`): every stone captured by one placement travels together as one echo, and `n` is the total count.
  Alternative `PER_GROUP`: each captured group becomes its own echo, with its own distance.

> Distance is measured in turns, not plies. With an odd ply distance the echo would arrive when the
> opponent is to move, which breaks the turn structure.

### 6.3 Target board
The target is the board at ply `t` in **B's own history**: follow parent links back from B. If B's
timeline branched after ply `t`, this walk continues into the parent timeline. An echo never jumps sideways into an unrelated timeline.

### 6.4 Arrival
The echo is applied to a copy of the target board's position, at the **same coordinates** where the
stones were captured, as if P played all of them at once:

| Intersection at target | Result (default `ANNIHILATE`) |
|---|---|
| Empty | Place the echo stone. |
| Already P's color | No effect. |
| Opponent stone | Remove both the opponent stone and the echo stone, leaving the point empty. This is "the stone meets its past self." |

Alternative collision modes: `OVERWRITE` (the echo replaces the opponent stone) and `FIZZLE` (the echo stone is dropped).

The echo case is common: the captured group usually already existed, partly or fully, in the past.

After the stones are placed:
1. Remove opponent groups adjacent to the echo stones that have zero liberties. This is the **cascade set**.
2. Remove any of P's groups that have zero liberties. These collapse with no echo.
3. If the resulting position equals the target position exactly, the echo **fizzles** and no timeline is created.

### 6.5 Branching
Otherwise, a **new timeline** created by P starts at ply `t + 1`, with the opponent to move. Its parent is
the target board. The new board is marked *echo-born*.

### 6.6 Cascades
If the cascade set is non-empty, a new echo is spawned from the target board, with ply `t` and the cascade set as its stones, and it is resolved the same way.
- The cascade travels back through the new timeline's history.
- An echo that arrives at the earliest ply doesn't cascade: the stones it captures are simply removed.
- `max_cascade_depth` is 8 by default as a safety limit. Cascades end anyway because time strictly decreases.
- Echoes are processed breadth-first, in deterministic order: sort by source ply descending, then by point index.

### 6.7 Timeline cap
If `max_timelines` (default 32) is reached, echoes **dissipate**. The capture still removes the stones from the present board, but no new timeline is created.

---

## 7. Timeline creation and activity (same as 5D Chess)

- P's new timelines get the next free index on P's side: `+k` for Black, `−k` for White.
- Let `nB` be the number of Black-created timelines and `nW` the number of White-created ones.
  - `L0` is always active.
  - `L+k` is active if `k ≤ nW + 1`.
  - `L−k` is active if `k ≤ nB + 1`.
- Inactive timelines can still be played on, but they don't define the present and aren't scored.
  They become active when the opponent creates enough timelines.

A new active timeline in the past pulls the present back to it, so the opponent must answer there first.
This is the main source of tempo in the game, exactly as in 5D Chess.

---

## 8. Ko and superko

- **Positional superko within the history**: a placement can't produce a position that appears earlier in the
  board's own history chain, including ancestors in parent timelines.
- Matching positions in other, sibling timelines are allowed. Copies of the past are the whole point of the game.
- Echo-born boards and passes are exempt from the check.
- Hashing uses Zobrist (§ARCHITECTURE 5.6).

Each ko capture spawns a 1-stone echo that goes 1 turn back. In this game, ko fights spread across the multiverse.

---

## 9. Passing, settling and end of game

- A **pass** on a board creates the next board with the same position.
- A timeline is **settled** when its tip was produced by two consecutive passes, one from each player.
  A settled timeline takes no more actions and is excluded from the present.
- An echo can still branch *from* a settled timeline's history, because the new timeline is a separate row.
- **The game ends** when:
  - every active timeline is settled, or
  - a player resigns, or
  - an optional `max_ply` limit is reached, in which case the game is scored as it stands.

---

## 10. Scoring

Area scoring (Tromp-Taylor) is used, which needs no dead-stone negotiation:
- Each **active** timeline's tip is scored on its own: stones on the board, plus empty regions that touch only one color, plus komi for White.
- **Winner of the game (default `TIMELINE_MAJORITY`)**: the player who wins more active timelines.
  - Tie-break 1: the sum of point margins across active timelines.
  - Tie-break 2: White wins. A half-point komi option avoids this.
- Alternative `TOTAL_AREA`: add up the points from all active timelines.

Players must capture dead stones before passing, as in Tromp-Taylor rules. The UI should warn
about stones that will be counted as alive.

---

## 11. Worked example

1. On `L0` at `T6b` (ply 10), Black plays and captures **2** White stones at d4 and d5.
   `L0` continues to `T6w` (ply 11), with the stones removed.
2. Echo: 2 **Black** stones (inverted), 2 turns back, so the target is ply 6 (`T4b`) in `L0`'s history.
3. At `L0 T4b`, d4 is empty and d5 holds a White stone (played at `T3w`).
   d4 gets a Black stone. At d5 the White stone and the echo stone annihilate. Captures are resolved; none happen.
4. Black-created timeline **`L+1`** begins with board `T4w` (ply 7), branching from `L0 T4b`.
5. Activity: nB = 1, nW = 0, and `L+1` is active because 1 ≤ 0 + 1.
6. Present = min(11, 7) = **7**, which is White to move, so Black can submit.
7. White **must** play on `L+1 T4w` and **may** also play on `L0 T6w`.
   `L0` stays frozen in the future until `L+1` catches up or White chooses to play there.

---

## 12. Rule options (`RuleSet` defaults)

| Option | Default | Values |
|---|---|---|
| `board_size` | 9 | 5, 7, 9, 13, 19 |
| `komi` | 7.0 | float, per timeline |
| `echo_enabled` | true | bool (false gives plain multiverse Go) |
| `echo_grouping` | `PER_MOVE` | `PER_MOVE`, `PER_GROUP` |
| `echo_collision` | `ANNIHILATE` | `ANNIHILATE`, `OVERWRITE`, `FIZZLE` |
| `cascades_enabled` | true | bool |
| `max_cascade_depth` | 8 | int |
| `max_timelines` | 32 | int |
| `superko` | `HISTORY_POSITIONAL` | `NONE`, `SIMPLE_KO`, `HISTORY_POSITIONAL` |
| `scoring` | `TIMELINE_MAJORITY` | `TIMELINE_MAJORITY`, `TOTAL_AREA` |
| `max_ply` | 0 (off) | int |

Every option is part of the saved game record, so a replay always uses the rules it was played with.

---

## 13. Strategic notes and playtest questions

- **Bigger captures reach further back.** That means more leverage, but older boards have fewer stones, so more echo stones land on empty points and fewer annihilate.
- **One-stone captures** (including ko) make cheap 1-turn branches, which work as tempo moves that drag the present back.
- **Timeline majority** combined with the activity rule means you can't simply out-branch your opponent. You can only be one active timeline ahead.

Questions to answer in playtesting:
1. Is `ANNIHILATE` or `OVERWRITE` more fun? `OVERWRITE` makes captures much stronger.
2. `PER_MOVE` or `PER_GROUP` grouping?
3. Is `TIMELINE_MAJORITY` too swingy compared with `TOTAL_AREA`?
4. Is 9×9 too big for a readable multiverse? 7×7 might be the sweet spot.
5. Variant idea: allow cross-board adjacency, so groups connect to the same point on neighboring boards (true 4D liberties). Keep it behind a flag; it is not part of v1.
