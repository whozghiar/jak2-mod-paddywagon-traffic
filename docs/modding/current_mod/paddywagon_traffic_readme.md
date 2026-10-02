# Krimzon Guard Paddy Wagon in City Traffic — Mod Readme

> - **Branch:** `jak2/features/paddywagon/traffic`
> - **Game:** Jak II (OpenGOAL)
> - **Status:** Compiles clean (`CWI.DGO`, `LWIDE{A,B,C}.DGO`, `GAME.CGO`, full `build-game`); awaiting in-game verification after `task extract`

---

## 1. Overview & Objective

`paddywagon-v` is a **real ground-traffic vehicle**: a `vehicle-guard` subtype
(sibling of `hellcat` / `guard-bike` / `cara`) that spawns in Haven City's
ambient car lanes, follows the city nav-branches, shows a **red dot on the
minimap**, is **driven by a Crimson Guard**, carries a **random civilian
prisoner standing in its rear cage**, can be **boarded and driven by the
player**, and can be **destroyed** like any guard vehicle.

**Stealing it is a crime** — and this comes entirely from stock code, not from
mod code: the driver is a `crimson-guard-rider`, whose `vehicle-rider` `flags`
bit 3 makes the `'knocked-off` branch of `vehicle-rider-event-handler` do
`(send-event *traffic-manager* 'increase-alert-level 2)` and respawn a
`crimson-guard-1` on the street. Identical to hijacking a hellcat.

The wagon is **unarmed**. The `paddy-wagon` skeleton has only the joints
"steering", "hatch", "main", "prejoint", "align" — there is no gun joint — so
`turret info` is left at 0 and `turret-control-method-11`'s
`(when (nonzero? (-> this info)))` guard makes the inherited guard AI pursue
without ever firing.

This branch is *inspired by* `jak2/features/transport-ag/traffic` (the drivable
air gunship) and reuses its two structural techniques — the spare traffic slot
20 and the `extra_art_groups_by_dgo` merc `.fr3` injection — but shares no code
with it.

## 1b. Runtime toggle (mandatory procedure)

Per CLAUDE.md's golden rules the mod ships **OFF** and is switched from
`Debug ▸ Mods ▸ paddywagon-traffic ▸ Enable`.

With the flag `#f` the traffic want-count for slot 20 is 0, so the traffic
engine never constructs a `paddywagon-v`; no code in `paddywagon-v.gc` runs and
no `paddywagon-prisoner` is ever spawned. The rest of the wiring is **inert
rather than gated**, because none of it is reachable without a slot-20 object:
the renamed enum entries, the extra `case` arms in `traffic-object-spawn` /
`type-from-vehicle-type`, and `guard.gc`'s object-type-20 knock-off animation
entry. Toggling the mod via the in-game Mods menu immediately triggers a traffic
recycle pass (`'kill-all` + `'spawn-all`), dynamically updating `want-count`
and regenerating city traffic pools so **the toggle applies immediately**,
without requiring a city reload.

The retail `paddywagon` type in `meet-brutter.gc` is **not touched at all**, so
the *Escort Brutter* mission is bit-for-bit stock whether the mod is on or off.

## 2. Why a new type instead of reusing retail `paddywagon`

The retail type cannot be used in free roam, for three independent reasons:

| Retail `paddywagon` does… | …and why that blocks traffic use |
|---|---|
| ships in `LMEETBRT.DGO` | a borrowed mission level — the type isn't even loaded in free roam |
| `init-skel-and-rigid-body` sets `(-> pp level)` to `'lmeetbrt` | art would resolve from a level that isn't resident; the van would art-error or vanish |
| sets `rigid-body-object-flag no-hijack` | `vehicle-util.gc`'s boarding probe tests this flag before it will even show the "press triangle" prompt — the mission van is deliberately un-stealable |
| forces `choose-next-branch-no-exit-level` | the mission van is pinned to one level's nav graph |
| `object-type` `#x13` | it impersonates the hellcat's traffic slot |
| overrides `vehicle-method-134` / `-108` / `active` / `hostile` / `die` | the meet-brutter chase script, not city traffic behaviour |

So `paddywagon-v` is a plain traffic `vehicle-guard` that reuses only the retail
**hull, physics constants and collision shape** — verbatim.

## 3. What `vehicle-guard` gives us for free

| Requirement | Inherited from | Mechanism |
|---|---|---|
| Crimson Guard driver | `vehicle-guard::vehicle-method-137` | spawns a `crimson-guard-rider` into seat 0. Traffic sets `trsflags-01` + `behavior 1` on every pooled vehicle → a hidden rider, revealed on `'traffic-on`. |
| Red dot on the minimap | `vehicle-guard::vehicle-method-128` | `add-icon! *minimap* … 14` — the red guard marker. |
| Follows the city lanes | `vehicle-guard-choose-branch`, installed by `vehicle-guard::alloc-and-init-rigid-body-control` | ordinary nav-branch following |
| Hostile pursuit during alerts | `vehicle-guard` `hostile` / `stop-and-shoot` states | **retail already** gives slot 20 `trtflags-0` in `traffic-engine::restore-default-settings` (line ~2918), so it receives `'alert-begin` / `'alert-end` and never parks |
| Player boards & drives | `vehicle::check-player-get-on` | we simply do **not** set `no-hijack` |
| Theft raises the alert | `vehicle-rider-event-handler` `'knocked-off` + `crimson-guard-rider`'s `flags` bit 3 | alert level 2 + a `crimson-guard-1` spawned on the spot |
| Pooling (hide/show/re-seat) | `vehicle-method-127` / `-128` broadcast `'traffic-off` / `'traffic-on` to **every child** | both riders are children, so both follow the pool automatically |
| Rider positioning | `vehicle-method-119` | places each `rider-array` entry at its seat and applies the seat's `angle` |

## 4. The prisoner

`paddywagon-prisoner` is a `vehicle-rider` — the same lightweight,
seat-parented, nav-mesh-free actor retail uses for car passengers
(`citizen-norm-rider`) and guard pilots (`crimson-guard-rider`).

**The seat was already there.** Retail `*paddywagon-constants*` declares
`seat-count 2`:

| seat | `position` (x, y, z) | `angle` | `flags` | role |
|---|---|---|---|---|
| 0 | 0, 9420.8, 2457.6 | 0 | 1 | driver / player seat |
| 1 | 0, 819.2, **-13107.2** | `#x8000` = 180° | 4 | **the rear cage**, facing backwards |

Seat 1 is exactly where the meet-brutter mission drops the captured lurker
(`(get-best-seat-for-vehicle … 4 0)` → `put-rider-in-seat`). This mod puts a
civilian there instead. Because the seat's `flags` is 4 and the player seat's is
1, `target-pilot`'s `(get-best-seat-for-vehicle … 1 1)` can never put Jak in the
cage.

**Body-type roll and the arms-crossed pose.** Unlike `citizen-norm-rider` —
which uses the stripped-down `citizen-norm-rider-ag` art group whose only poses
are idle / bike-stance / car-stance — the prisoner uses the **full** civilian
art groups, because those are the only ones carrying the standing arms-crossed
pose:

| variant | skeleton group | pose | art elt |
|---|---|---|---|
| `norm` | `skel-citizen-norm` | `citizen-norm-arms-crossed-ja` | 8 |
| `fat` | `skel-citizen-fat` | `citizen-fat-arms-crossed-ja` | 8 |
| `chick` | `skel-citizen-chick` | `citizen-chick-idle-ja` | 10 |

> [!IMPORTANT]
> **`citizen-chick` genuinely has no arms-crossed animation in retail.** Its only
> standing poses in `engine/data/art-elts.gc` are `idle` (10), `shuffle` (26)
> and `riding-stance` (27). The female prisoner therefore stands at ease. To
> change that, edit `*paddywagon-prisoner-chick-anim*` in `paddywagon-v.gc`
> (27 = riding-stance is the other plausible choice).

**Zone-dependent variety.** `skeleton-group->draw-control` resolves a model
through the **process's own level** (`(-> (-> proc level) art-group)`), and
traffic riders are re-homed onto the lwide level by `(lwide-entity-hack)`.
Retail only ships:

| level | `citizen-norm-ag` | `citizen-fat-ag` | `citizen-chick-ag` |
|---|---|---|---|
| LWIDEA | ✅ | ✅ | ✅ |
| LWIDEB | ✅ | ❌ | ❌ |
| LWIDEC | ✅ | ❌ | ✅ |

So `paddywagon-prisoner-pick-variant` **checks residency first** (via
`paddywagon-prisoner-art-group-loaded?`, which reads
`(-> lvl art-group string-array)` directly rather than going through
`art-group-load-check`, whose debug-build disk load would make debug and release
behave differently) and rolls only among what is actually available, falling
back to `norm`, which ships everywhere. Result: all three body types in the
LWIDEA zone, norm+chick in LWIDEC, norm only in LWIDEB — which mirrors what
retail pedestrians look like in those zones anyway.

**The prisoner stays put when Jak steals the wagon.** `target-pilot` walks
*every* seat on boarding and sends `'knocked-off` to whoever is in it. That is
correct for the guard driver; for a prisoner it is not, so
`paddywagon-prisoner-event-handler` intercepts `'knocked-off` and returns `#f`.
That matters twice over: `target-pilot` only frees the seat when `send-event`
comes back truthy, so refusing the event also keeps the prisoner registered in
`rider-array` and therefore still positioned every frame by
`vehicle-method-119`.

**No lean.** The base `vehicle-rider-method-34` pins the channel to
`num-func-identity` and drives `frame-num` off a sine blended with the vehicle's
steering — the lean a driver does into a turn. The override reduces it to a bare
`(ja-post)`, and the `active` state's `:enter` installs a plain `:num! (loop!)`,
so the prisoner just holds his pose.

**Wardrobe.** `vehicle-rider-method-33` re-rolls the `setup-masks` submesh
groups transcribed from each civilian's own `citizen-init!`
(`citizen-norm.gc` / `citizen-fat.gc` / `citizen-chick.gc`), using
`rand-vu-int-count` in place of the `citizen`-only `rnd-int-count` method. It
runs on first spawn **and** on every `'traffic-on`, so each traffic life gets a
freshly dressed prisoner.

## 4b. Boarding, hanging, and fleeing under fire

**Grab rails.** Retail `*paddywagon-constants*` carries `:grab-rail-array #f`
and no `:grab-rail-count` — zero rails — because the mission van is `no-hijack`
and Jak is never meant to touch it. Every stealable city vehicle has them
(cara 6, carb 5, carc 9, hellcat 6, the bikes 2). `check-player-get-on` offers a
vehicle two different ways:

| path | condition | triangle sends |
|---|---|---|
| direct board | Jak roughly level with the hull (`floats2 > -14336`) | `change-mode pilot` — he gets in |
| grab rail | Jak more than 2m **below** the hull origin (`floats2 < -8192`) **and** not already edge-grabbing | `pilot-edge-grab` — he **hangs off the side** |

The rail loop is `(dotimes (s2-1 (-> this info grab-rail-count)) …)`, so with no
rails the second path could never fire and the wagon could never be hung off.
This mod adds four rails (front bumper, both flanks, rear above the cage doors)
at `y` 9216 — the body's real light/window line, taken from the retail
headlight/taillight positions in the same constants block. **These four figures
are estimates**; if Jak hangs clipped into the bodywork or floating off it, they
are the only values to nudge.

**Fleeing under fire.** The wagon is carrying a prisoner, so it breaks off and
leaves the area rather than behaving like an ordinary traffic vehicle:

- `apply-damage` override — `vehicle::apply-damage` is the single funnel for
  every kind of damage (gunfire and explosions via
  `rigid-body-object-method-46`'s attack branch, ramming via the impulse branch
  in `vehicle.gc`). Anything at or above
  `*paddywagon-v-flee-damage-threshold*` (0.1) starts a flee and records Jak's
  position in `flee-from`. The threshold lets every projectile hit through
  (they arrive as 1.0 / 0.25 / 0.125) while ignoring ordinary bumper contact
  from other traffic. Skipped while the player is driving it, and once it is
  dead.
- `vehicle-method-120` override — the per-frame tick (run from
  `vehicle-method-122`, which `vehicle-guard`'s `active` state calls in its
  :post). While fleeing it (a) pushes `controller target-speed-offset` to
  `*paddywagon-v-flee-speed-boost*` (the retail value is `(meters -2)` — a paddy
  wagon normally trundles *below* the lane limit), (b) sets `ignore-others` so
  it barges past slower traffic instead of queueing, and (c) clears
  `pursuit-target` plus the `alert` / `in-pursuit` / `target-in-sight` /
  `rammed-target` flags every frame. That last part is what makes it **flee
  rather than fight**: with `pursuit-target` cleared, `vehicle-guard`'s
  `hostile` :post finds no target through `vehicle-guard-method-151` and calls
  `vehicle-method-109`, dropping it back to ordinary traffic driving. Both
  controller settings are re-asserted every frame because `vehicle-method-109`
  itself clears `ignore-others`.
- `paddywagon-v-choose-branch` — the controller's nav-branch chooser, installed
  in `init-skel-and-rigid-body`. Normally defers to the stock
  `vehicle-guard-choose-branch`; while fleeing it picks the branch whose heading
  points most directly **away** from `flee-from`, so the wagon actually leaves
  the area instead of looping back past its attacker. Falls back to the stock
  chooser if the attacker is right on top of it (no meaningful direction to
  normalise). Safe across pooling: only `vehicle-controller-method-9` resets
  `choose-branch-callback`, and it runs once inside
  `alloc-and-init-rigid-body-control`.
- `vehicle-method-134` override — refuses to accept a pursuit target while
  fleeing, otherwise the ram branch of
  `vehicle-guard::rigid-body-object-method-46` would re-acquire Jak the moment
  he clipped the fleeing wagon.
- `vehicle-method-128` override — flee state is per-traffic-life, so a recycled
  hull starts calm.

Tunables (all REPL-editable): `*paddywagon-v-flee-duration*` (12 s),
`*paddywagon-v-flee-speed-boost*` (`(meters 15)`),
`*paddywagon-v-flee-damage-threshold*` (0.1).

## 4c. Two crash-adjacent hazards of an unarmed, pilotable `vehicle-guard`

**The turret crash (fixed).** `paddywagon-v` is the first `vehicle-guard` the
player can pilot that has **no turret**. The `paddy-wagon` skeleton has no gun
joint (only "steering", "hatch", "main", "prejoint", "align"), so
`init-skel-and-rigid-body` never calls `set-info` and `turret info` stays 0.

The AI path tolerates that — `vehicle-guard-method-153` goes through
`turret-control-method-11`, whose whole body is wrapped in
`(when (nonzero? (-> this info)) …)`. That is why AI paddy wagons drive around
the city perfectly happily.

The **player** path does not. `vehicle-guard::vehicle-method-94` aims and fires
the hull turret before delegating to `vehicle::vehicle-method-94` (the actual
stick read), completely unguarded:

```lisp
(set! (-> this turret inaccuracy) 0.0)
(turret-control-method-9 (-> this turret) this ...)   ;; every frame
(when (cpad-hold? 0 r1) ... (turret-control-method-17 (-> this turret) this))
```

and `turret-control-method-9`'s first act is

```lisp
(-> arg0 node-list data (-> this info joint-index) bone transform)
```

— a dereference of a null `info`, producing a garbage joint index that is then
used to index `node-list`. The runtime dies instantly with **no GOAL error**.
The call chain is `player-control` :post → `vehicle-method-124` →
`vehicle-method-94`, so it fires on the very first frame after `pilot-on`.

Retail never reaches it: the only two pilotable `vehicle-guard`s, `hellcat`
(car.gc) and `guard-bike` (bike.gc), both `set-info` their turret, and the
mission `paddywagon` is `no-hijack`. `paddywagon-v` therefore overrides
`vehicle-method-94` to skip `vehicle-guard`'s turret block and call
`vehicle::vehicle-method-94` directly. **If this wagon is ever given a real gun,
call `set-info` in `init-skel-and-rigid-body` and delete that override.**

**Theft alert (hardened).** Stealing a guard vehicle already raises the city
alert through stock code: the `crimson-guard-rider` driver carries
`vehicle-rider` `flags` bit 3, so `vehicle-rider-event-handler`'s `knocked-off`
branch sends `increase-alert-level 2` to `*traffic-manager*` and respawns a
Crimson Guard on the street. That path needs a driver still aboard to knock off,
though — shoot the pilot first, or take a wagon whose rider was already cleared,
and the theft goes unnoticed. `paddywagon-v` overrides `vehicle-method-87` (the
one-shot boarding hook, self-guarded by `rigid-body-object-flag camera`) to send
the same event itself. `traffic-engine::increase-alert-level` takes the `max` of
current and requested level, so the two paths cannot fight; it is also gated
internally on the `target-jak` alert flag, so it stays inert during scripted
missions.

## 4d. Making it a proper player vehicle — `info flags` #xc → #x6c

Retail `*paddywagon-constants*` carries `:flags #xc` (bits 2 and 3). That was
enough for a `no-hijack` mission van but not for one the player drives, so the
mod raises it to **`#x6c` = 4 | 8 | 32 | 64**:

| bit | value | source | what it does |
|---|---|---|---|
| 2 | 4 | retail | while `ai-driving`, being shot makes the vehicle acquire Jak as a pursuit target (`rigid-body-object-method-46`). Kept — the flee tick clears `pursuit-target` every frame, so a hit still ends in flight. |
| 3 | 8 | retail | the hull has a real collide-MESH prim, so `alloc-and-init-rigid-body-control` gives the mesh `jak`/`player-list` and Jak can stand on and board it. |
| 5 | 32 | **added** | `gun?` |
| 6 | 64 | **added** | flight-lane switching |

**Bit 5 — Jak keeps his gun.** `target-pilot`'s `enter-vehicle` reads it once:
`(set! (-> s5-0 gun?) (logtest? (-> vehicle info flags) 32))`, and then

```lisp
(when (not (-> s5-0 gun?))
  (if arg1 (logior! (-> self control current-surface flags) (surface-flag gun-fast-exit)))
  (target-gun-end-mode arg1))
```

— with the bit clear, boarding holsters the gun, which is why Jak could not
shoot from the wagon. `target-pilot-init` optimistically sets `gun?` to `#t`
beforehand, but `enter-vehicle` overwrites it from the flag, so this bit is the
single switch. cara/carb/carc (`#x68`) are the retail vehicles you can shoot
from; the hellcat (`#x4c`) is the retail example of the bit being deliberately
off.

**Bit 6 — the high/low lane switch.** It gates `switch-zone-high!` /
`switch-zone-low!` (vehicle-util.gc), both of which begin
`(when (and (logtest? (-> this info flags) 64) …))` and are otherwise no-ops.
Those two methods move a vehicle between the two `flight-level-index` modes that
vehicle-physics.gc implements:

- **index 1** — the lift thrusters target `flight-level + 6144`, i.e. the
  `*traffic-height-map*` **air lane**, and `on-flight-level` gets set;
- **index 0** — the thrusters use ordinary ground probes, i.e. the **low lane**.

`vehicle::active` :enter sets `flight-level-index` to 1, so every traffic vehicle
starts high — which is why the wagon was always in the upper lane and R2 did
nothing. The player toggle lives in `vehicle::vehicle-method-94`:

```lisp
(when (and (cpad-pressed? 0 r2) (not *pause-lock*))
  (if (zero? (-> this flight-level-index)) (switch-zone-high! this) (switch-zone-low! this)))
```

which this mod's `vehicle-method-94` override still routes to (see §4c — that
override skips `vehicle-guard`'s turret block but keeps the `vehicle` base).

> Setting bit 6 also fixes a latent bug that predates the player being able to
> drive at all: `player-control` :exit calls `vehicle-method-83`, which forces
> `flight-level-index` to 0, and only `switch-zone-high!` — reached from
> `vehicle-method-93` and the `vehicle-guard` AI tick, both gated on this bit —
> puts it back. Without bit 6, a wagon Jak abandoned would have stayed stuck in
> the low lane for the rest of its traffic life.

**Button budget:** R1 is Jak's gun now, and `vehicle-guard::vehicle-method-94`'s
turret block used R1 too. The §4c override removes that block entirely, so the
two never compete.

## 5. Traffic-type wiring

- `engine/ai/traffic-h.gc` — renames the spare `(traffic-type-20 20)` →
  `(paddywagon-v 20)`. **Slot 20 is the last usable slot**: every engine loop is
  `(dotimes … 21)` and `traffic-engine`'s `object-type-info-array` is 21 inline
  entries with `inactive-object-array` sized 420 = 21 × 20, so slot 21 cannot be
  used without resizing those structures (and the hard `:offset 7024` on
  `vehicle-tracker-array`). Retail already wired slot 20 as a `ctywide`-level,
  vehicle-tracker, `trtflags-0` guard slot — it just had want-count 0 and no
  spawn handler.
- `levels/city/traffic/vehicle/vehicle-h.gc`, `engine/entity/entity-h.gc`,
  `decompiler/config/jak2/all-types.gc` — add `(vehicle-type paddywagon-v 11)`.
- `levels/city/traffic/traffic-manager.gc` — `define-perm
  *mod-paddywagon-traffic-enable*`, the `traffic-object-spawn` and
  `type-from-vehicle-type` `case` arms, and
  `want-count[20] = (if *mod-paddywagon-traffic-enable* 2 0)`.
- `levels/city/traffic/citizen/guard.gc` — object-type 20 added to the **car**
  knock-off animation group `(14 15 16 19)`, so a guard thrown out of the van
  uses the car animation rather than the bike one.
- `levels/city/{ctywide-tasks, protect/protect, slums/kor/hal3-course,
  kiddogescort/hal4-course}.gc` — the four mission scripts that
  `deactivate-by-type` slot 20 now use the new enum name.

## 6. Rendering — the merc geometry `.fr3` injection

The paddy wagon's merc geometry only ever shipped in `LMEETBRT.DGO`.
`extra_art_groups_by_dgo` in `decompiler/config/jak2/jak2_config.jsonc` bakes
`paddy-wagon-ag:LMEETBRT.DGO` into `lwidea.fr3` / `lwideb.fr3` / `lwidec.fr3`
(the `:LMEETBRT.DGO` suffix resolves the texture ids through LMEETBRT's remap
table — the `lmeetbrt-pris` tpage — without which the van renders white), and
`paddy-wagon-ag.go` + `tpage-2438.go` are added to the three `lwide*.gd`.
See [`docs/modding/tools/model_and_entity_level_injection_guide.md`](../tools/model_and_entity_level_injection_guide.md).

> **Requires a re-extraction** (`task extract`) so the three `.fr3` are rebuilt.
> Without it the process runs (sounds, collision, riders) but the hull is
> **invisible**.

## 7. How to Test

1. **Extract (once):** `task extract` — rebuilds `lwide*.fr3` with
   `paddy-wagon-ag`. Look for
   `extra_art_groups_by_dgo: baking 'paddy-wagon-ag' into LWIDEA.DGO (.fr3)` in
   the log.
2. **Rebuild:** `task repl` then `(mi)` (new deftypes + a new `.o` in `cwi.gd` —
   restart the REPL if `(mi)` complains).
3. **Launch:** `task boot-game`, enter Haven City free-roam.
4. **Enable:** `Mods ▸ paddywagon-traffic ▸ Enable` (OFF by default).
   The traffic pool is immediately recycled in real-time, spawning paddy wagons
   without needing to reload the city.
5. **Ambient:** drive around — a boxy armoured van in the car lanes, a Crimson
   Guard at the wheel, a civilian standing arms-crossed in the open rear cage,
   red dot on the minimap.
6. **Alert:** aggro a guard. The wagon joins the pursuit like a hellcat but
   never fires.
7. **Steal it:** walk up — "press triangle to use" should appear from up to
   ~12.75m away (`floats4` = 8192 + the 6.5m root sphere, ×1.5 while
   AI-driving). Press triangle. The guard is thrown clear, the **city alarm
   sounds (alert level 2)**, a Crimson Guard respawns on the street, and the
   **prisoner is still standing in the back** as you drive off.
8. **Hang off it:** get *below* the wagon (a ramp, a walkway, or jump at one
   passing overhead) so Jak is more than 2m under the hull origin — that is the
   `floats2 < -8192` gate — and press triangle. He should grab a flank rail and
   hang there. Note the direct-board path takes priority whenever Jak is roughly
   level with the hull, so on flat ground you get straight in rather than
   hanging; that is stock `check-player-get-on` behaviour, identical for cars.
9. **Shoot one:** it should immediately speed up, push past traffic and take the
   turns that lead away from you, without ever turning to fight — for
   `*paddywagon-v-flee-duration*` (12 s) after the last hit.
10. **While driving:** tap **R2** — the wagon should drop out of the high air
    lane down to the low one (with the "bike-down" sound) and back up on the
    next press. Hold **R1** — Jak should still have his gun out and fire it.
11. **Get out and watch it leave:** the abandoned wagon should climb back to the
    air lane and rejoin traffic rather than staying low (see §4d).
12. **Destroy one:** it explodes like any guard vehicle.
9. From the REPL:
   `(send-event *traffic-manager* 'set-object-target-count (traffic-type paddywagon-v) 4)`
   for more of them.

**Key source files:**

- `goal_src/jak2/levels/city/traffic/vehicle/paddywagon-v.gc` *(new)* —
  `skel-paddywagon-v`, `*paddywagon-v-constants*`, `paddywagon-prisoner`
  (+ its variant roll, wardrobe, pose and `'knocked-off` refusal), `paddywagon-v`
  and its `allocate-and-init-cshape` / `init-skel-and-rigid-body` /
  `vehicle-method-137` overrides.
- `goal_src/jak2/engine/ai/traffic-h.gc` — `(traffic-type paddywagon-v 20)`.
- `goal_src/jak2/levels/city/traffic/vehicle/vehicle-h.gc`,
  `goal_src/jak2/engine/entity/entity-h.gc`,
  `decompiler/config/jak2/all-types.gc` — `(vehicle-type paddywagon-v 11)`.
- `goal_src/jak2/levels/city/traffic/traffic-manager.gc` — spawn case + gated
  `want-count` + `define-perm *mod-paddywagon-traffic-enable*`.
- `goal_src/jak2/levels/city/traffic/citizen/guard.gc` — car knock-off anim group.
- `goal_src/jak2/pc/debug/paddywagon-traffic-menu.gc` *(new)* — Debug ▸ Mods toggle.
- `goal_src/jak2/dgos/cwi.gd` — `"paddywagon-v.o"` after `car.o`.
- `goal_src/jak2/dgos/game.gd` — `"paddywagon-traffic-menu.o"` after `mods-menu.o`.
- `goal_src/jak2/dgos/lwide{a,b,c}.gd`, `decompiler/config/jak2/jak2_config.jsonc`
  — the `.fr3` merc injection.
- `goal_src/jak2/levels/city/{ctywide-tasks, protect/protect,
  slums/kor/hal3-course, kiddogescort/hal4-course}.gc` — enum rename.

## 8. Current State & Known Tradeoffs

- **Seen in game (2026-09-10):** the wagon spawns, renders and drives the city
  lanes; the grab rails work (`pilot-edge-grab` accepted, Jak hangs off a flank);
  boarding works. The turret-crash fix and the flee behaviour compile clean but
  have **not** been observed in game yet.
- **The flee behaviour is still unverified in game** — the branch chooser in
  particular has only been reasoned about, not watched.
- **Slot 20 is shared with `jak2/features/transport-ag/traffic`.** Both mods
  claim the same last free traffic slot, so as written they are **mutually
  exclusive**. Merging them would require extending `traffic-engine`'s
  `object-type-info-array` (21 → 22), `inactive-object-array` (420 → 440), the
  hard `:offset 7024` on `vehicle-tracker-array`, and every `(dotimes … 21)` in
  `traffic-engine.gc` / `traffic-manager.gc`.
- **Seat 1's height (`y` 819.2) is retail data tuned for the standing `babak`
  lurker.** A citizen skeleton has a different root height, so the prisoner may
  render slightly sunk into or floating above the cage floor. If so, nudge
  `seat-array 1 position` `y` in `*paddywagon-v-constants*` — it is the only
  cosmetic figure worth touching; the flight/handling figures must not move (see
  the `cm-offset-joint` warning in
  [`docs/modding/jak2_lisp_instructions.md`](../jak2_lisp_instructions.md)).
- **`chick` has no arms-crossed pose** (see §4) — she stands at ease.
- **Zone-dependent prisoner variety** (see §4) — LWIDEB only ever produces a
  `norm` prisoner, because retail ships no other civilian art group there.
  Making all three appear everywhere would mean injecting `citizen-fat-ag` /
  `citizen-chick-ag` into LWIDEB/LWIDEC too, which changes ordinary pedestrian
  residency — deliberately out of scope.
- **Unarmed by design** — no gun joint on the skeleton (see §1).
- **Lane correction (2026-09-11).** An earlier revision of this document claimed
  the wagon "rides the low ground lanes, not the hellcats' air lane, because
  retail `flags` `#xc` does not carry bit 6". **That was wrong**, and observation
  in game disproved it: `vehicle::active` :enter sets `flight-level-index` to 1
  unconditionally, so every traffic vehicle hovers at the
  `*traffic-height-map*` air lane regardless of bit 6. All bit 6 controls is
  whether `switch-zone-high!` / `switch-zone-low!` can *change* that. Without
  it, the wagon was **stuck** in the upper lane, not held in the lower one. See
  §4d.

---

## 9. Modding Changes Log

| Date | Touched/Created Files | Technical Description | Objective |
| :--- | :--- | :--- | :--- |
| 2026-09-11 | `levels/city/traffic/vehicle/paddywagon-v.gc` | **Player vehicle flags: `info flags` #xc → #x6c.** Added bit 5 (32) and bit 6 (64) to the retail constants. Bit 5 is `gun?`, read once by `target-pilot`'s `enter-vehicle`; with it clear, boarding ran `(target-gun-end-mode arg1)` and holstered Jak's weapon — set, he keeps it out and fires with R1 while driving, like cara/carb/carc (#x68). Bit 6 gates `switch-zone-high!` / `switch-zone-low!`, which are no-ops without it; those move the vehicle between `flight-level-index` 1 (lift thrusters target `flight-level` + 6144, the `*traffic-height-map*` air lane) and 0 (ground probes, the low lane), and the R2 toggle for them lives in `vehicle::vehicle-method-94` — still reached through this mod's `vehicle-method-94` override. `vehicle::active` :enter sets index 1, so the wagon was permanently stuck in the upper lane. Bit 6 also fixes a latent bug: `player-control` :exit forces index 0 via `vehicle-method-83`, and only `switch-zone-high!` (gated on this bit) restores it, so an abandoned wagon used to stay stuck low. No R1 conflict because the §4c override already removed `vehicle-guard`'s turret block, which also used R1. | Let the player fly the wagon in both city traffic lanes and shoot while driving it. |
| 2026-09-10 | `levels/city/traffic/vehicle/paddywagon-v.gc` | **Fixed the hijack crash + hardened the theft alert (second in-game feedback round).** (1) **Crash:** stealing the wagon killed the runtime one frame after `pilot-on`, with no GOAL error. Cause: `vehicle-guard::vehicle-method-94` (reached every frame from `player-control` :post → `vehicle-method-124`) aims and fires the hull turret *unguarded*, and `turret-control-method-9` opens with `(-> arg0 node-list data (-> this info joint-index) bone transform)`. This wagon's `turret info` is 0 — the `paddy-wagon` skeleton has no gun joint — so that dereferences null and indexes `node-list` with garbage. The AI path was always safe because `vehicle-guard-method-153` goes through `turret-control-method-11`, which *is* wrapped in `(when (nonzero? (-> this info)) …)`. Retail never hits it: `hellcat` and `guard-bike`, the only pilotable `vehicle-guard`s, both `set-info` their turret, and the mission `paddywagon` is `no-hijack`. Fixed by overriding `vehicle-method-94` to skip `vehicle-guard`'s turret block and call `vehicle::vehicle-method-94` directly. (2) **Alert:** added a `vehicle-method-87` override (one-shot, self-guarded by `rigid-body-object-flag camera`) that sends `increase-alert-level 2` on boarding, so the theft is noticed even when no `crimson-guard-rider` is left aboard to trigger the stock `knocked-off` path. `increase-alert-level` takes a `max` and is gated on the `target-jak` flag, so the two paths cannot fight and it stays inert in scripted missions. | Make the wagon actually drivable, and make stealing it always raise the alarm rather than only when a driver happened to still be aboard. |
| 2026-09-10 | `levels/city/traffic/vehicle/paddywagon-v.gc` | **Grab rails + flee-under-fire (first in-game feedback round).** (1) Retail `*paddywagon-constants*` ships `:grab-rail-array #f` / no `:grab-rail-count`, so `check-player-get-on`'s `(dotimes (s2-1 (-> this info grab-rail-count)) …)` ran zero times and the `pilot-edge-grab` path — Jak hanging off the side before committing to the theft — could never fire. Added 4 rails (front, both flanks, rear) at `y` 9216, the body's light/window line per the retail headlight/taillight positions. (2) New flee behaviour: `apply-damage` override starts a flee above `*paddywagon-v-flee-damage-threshold*` and records the attacker position; `vehicle-method-120` override boosts `target-speed-offset`, sets `ignore-others` and clears `pursuit-target` + `alert`/`in-pursuit`/`target-in-sight`/`rammed-target` each frame (so `vehicle-guard`'s `hostile` :post falls through `vehicle-guard-method-151` to `vehicle-method-109` and it drives rather than fights); new `paddywagon-v-choose-branch` controller callback picks the nav-branch heading most directly away from `flee-from`; `vehicle-method-134` refuses pursuit targets while fleeing; `vehicle-method-128` resets flee state per traffic life. | A prisoner transport must be approachable the way every other city vehicle is (hang on it, then decide to steal it), and must run from a firefight instead of joining one — it is carrying a civilian. |
| 2026-09-10 | `levels/city/traffic/vehicle/paddywagon-v.gc` *(new)*<br>`engine/ai/traffic-h.gc`<br>`levels/city/traffic/vehicle/vehicle-h.gc`<br>`engine/entity/entity-h.gc`<br>`decompiler/config/jak2/all-types.gc`<br>`levels/city/traffic/traffic-manager.gc`<br>`levels/city/traffic/citizen/guard.gc`<br>`pc/debug/paddywagon-traffic-menu.gc` *(new)*<br>`dgos/{cwi,game,lwidea,lwideb,lwidec}.gd`<br>`decompiler/config/jak2/jak2_config.jsonc`<br>`levels/city/{ctywide-tasks,protect/protect,slums/kor/hal3-course,kiddogescort/hal4-course}.gc` | **Initial implementation.** New `paddywagon-v` (`vehicle-guard`) on traffic slot 20, reusing retail `paddy-wagon` hull + `*paddywagon-constants*` verbatim except `object-type` → `#x14`; no `no-hijack`, no `'lmeetbrt` re-home, no `choose-branch-callback` override, so it is an ordinary stealable traffic guard vehicle. New `paddywagon-prisoner` (`vehicle-rider`) in retail seat 1 (the rear cage, flags 4 / 180°): rolls `norm`/`fat`/`chick` among the art groups actually resident in the process's lwide level, holds `*-arms-crossed-ja` (idle for chick), re-rolls the retail `setup-masks` wardrobe on every `'traffic-on`, drops the base sine "lean", and refuses `'knocked-off` so it stays caged when Jak steals the van. `vehicle-method-137` override spawns both riders. Merc `.fr3` injection of `paddy-wagon-ag:LMEETBRT.DGO` into the three lwide levels + `paddy-wagon-ag.go`/`tpage-2438.go` in their `.gd`. Mandatory `Debug ▸ Mods ▸ paddywagon-traffic` toggle via `define-perm *mod-paddywagon-traffic-enable*` gating `want-count[20]`. | Put the Krimzon Guard prisoner van into ambient city traffic with a civilian prisoner and a Crimson Guard driver, drivable and stealable under the same conditions as every other guard vehicle — while leaving stock Haven City and the *Escort Brutter* mission untouched when the toggle is OFF. |
| 2026-09-16 | `traffic-manager.gc` | **Empty-city regression fix.** `reset-actors` calls each active level's `activate-func` in `*level*` SLOT ORDER, so `ctywide-activate` (-> `traffic-start` -> `init-params` -> `reset-and-init`) can run AFTER `lwide-activate` and wipe everything it just installed: `object-type-info-array[0..19].level` back to `#f` (no traffic spawns at all) and `(reset alert-state)` dropping the `target-jak` flag (guards ignore Jak's crimes) plus lwideb's forced war-zone alert. `init-params` now re-runs `lwide-activate` on the active lwide level, after `restore-default-settings`. Latent stock bug, shared by every branch that touches `init-params`. | Haven City keeps its population and its guard alerts after a death / checkpoint restart (AI-assisted) |
