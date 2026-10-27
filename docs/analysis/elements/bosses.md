# Boss overhaul: best path

Element 1 in the [queue](queue.md). Plan item: Tier 2 #9 in [improvement-plan.md](../improvement-plan.md).
Analysis of `main` at 421178c (2026-10-14), after the safety PR (#26) removed DM-300's pylon heal.

## Where the bosses stand

Each boss from Tengu on is a ~110-line `Mob` subclass with one arena built from scratch. They all use the same shape: cooldown counters, one special attack, and `level.unseal()` on death. Goo (depth 5) is in better shape and is out of scope.

| Boss | What exists | What's missing vs SPD | Evidence |
|---|---|---|---|
| Tengu (d10) | Teleport every 3 turns, a shuriken that skips the hit roll, a "phase 2" that only adds +4 shuriken damage. Arena is one random maze with secret traps. | The two-arena fight: phase 1 in a small cell room, then at 50% HP Tengu leaps away, the arena reshapes, and phase 2 is ability-driven (bombs, fire, shocker) with cooldowns that shorten as HP falls. Shurikens should roll to hit. | `tengu.gd:31-39,62-71`, `prison_boss_level.gd:21-45` |
| DM-300 (d15) | Charge that teleports adjacent and paralyses, poison on all 8 neighbours, `pylons_active` counter nothing reads. Pylons are `STATUE_SP` tiles. | Real Pylon actors. At HP thresholds DM-300 becomes supercharged (invulnerable, faster, drills through walls) until the hero destroys the pylon that lit up. Toxic gas should be a vented gas blob, and rockfall should be telegraphed. | `dm300.gd:34-37,60-92,94-97`, `caves_boss_level.gd:44-54` |
| Dwarf King (d20) | Summons weak skeletons, 30% cripple on hit, phase flags that only fire a 3-skeleton wave. | **Bug:** `undead_count` only goes up, so after 4 summons the king never summons again. SPD's phases: (1) melee, summons, lifelink, teleport to throne; (2) shielded on the throne while waves of ghouls/monks/warlocks come from the summoning pedestals; (3) final melee. Should drop the King's Crown (that item belongs to the endgame element). | `king.gd:50,72-89`, `city_boss_level.gd:32-42` |
| Yog (d25) | Spawns two fists on first sight, a flat 25-damage laser every 3 turns with no hit roll or beam. | **Bug:** Yog takes damage while fists live (the class comment says it shouldn't). Laser needs `Char.hit` and a Ballistica beam that hits whatever is in the line. SPD spawns fist pairs per phase (6 fist types) and Larva minions; this port has 2 fist types and no Larva mob. | `yog.gd:4,83-89`, `yog_fist.gd:5` |

### Shared problems the steps must fix first

1. **`is_invulnerable()` overrides do nothing.** `Char.take_damage` checks `has_buff("Invulnerable")` directly (`char.gd:312-313`), not `is_invulnerable(source)` (`char.gd:258`). Yog's fist shield, DM-300's supercharge and the King's throne phase all need a per-boss invulnerability rule, so this one-line change comes first.
2. **No boss behaviour tests.** Only `test_boss_floor_seal` and `test_boss_hp_signals` exist. Each step below adds a headless test, so CI (now strict after #26) protects it.
3. **Arenas share scaffolding by copy-paste.** All five boss `_build()`s duplicate carve-arena, entrance, locked exit and boss spawn (backlog S09). Tengu's and DM-300's steps both rewrite their arenas, so a small `BossArenaLevel` helper set should land with the first arena rewrite, not as a separate refactor.
4. **Two boss files are on `TRUNCATED_FILES.txt`** (`tengu.gd`, `king.gd`). The list is stale (plan Tier 5), but each step re-reads those files whole before editing, per CLAUDE.md.
5. **Assets are already there.** `assets/spd/sprites/` has `pylon.png`, `yog_fists.png` (all 6 fists), `larva.png`, `ghoul.png`, `monk.png`, `warlock.png`. The prison/caves/city/halls boss tile sets are unused, but using them belongs to the visuals element.

## Approach

Go in dungeon order as the plan says, but fix the two outright bugs (King's summon cap, Yog damage-through-fists) first, because they are small and make the late game wrong today. Then rebuild one boss per PR, smallest risk first within each boss. Numbers (HP, damage, thresholds, cooldowns) are taken from the upstream SPD source at implementation time and cited in each PR, not guessed.

Keep each PR to one boss file plus its arena and a test. New actors (Pylon, Larva, extra fist types, Tengu's ability objects) get their own files so no existing file grows past what a reviewer can read.

## Out of scope here

- Goo, and the `death_by_goo` badge false positive (seeded-runs element covers badges).
- King's Crown and armor abilities (endgame element). The King step only drops a placeholder hook for it.
- Boss arena tile art and boss-finale music (visuals element; music path is backlog S30).

## Steps (each is one PR)

1. **[Shipped as PR #29, awaiting merge] Boss groundwork and the two bugs.** Route `Char.take_damage` through `is_invulnerable(source)`. Decrement the King's `undead_count` when a summoned minion dies (track minions by reference, so loads also work). Make Yog invulnerable while any fist is alive, with a log line. Give the Yog laser and Tengu shuriken a `Char.hit` roll. Tests: King summons again after a minion dies; Yog takes 0 damage with a fist alive and full damage after; invulnerable override is respected.
2. **DM-300 pylons and supercharge.** Add a `Pylon` actor (destructible, immobile, uses `pylon.png`), place them in `caves_boss_level.gd` instead of statue tiles, and add the supercharge cycle: at each HP threshold DM-300's HP locks, it becomes invulnerable and faster, and one pylon activates; destroying it ends the supercharge. Remove the dead `pylons_active`/`destroy_pylon`. Save/load the pylon state. Extract the shared arena helpers here. Tests: supercharge triggers at the threshold, pylon death ends it, save round trip.
3. **DM-300 abilities.** Replace the 8-neighbour poison with a vented toxic gas blob in a cone, add telegraphed rockfall (marked cells, damage and paralysis next turn), and let supercharged DM-300 dig through walls. Tests: rockfall only hits marked cells; gas spawns as a blob.
4. **Tengu two-arena fight.** Rewrite `prison_boss_level.gd` into the phase 1 cell room and the phase 2 arena. At 50% HP Tengu leaps out, the level reshapes, and the fight resumes in the arena. Persist the level state for saves. Tests: phase change reshapes the level and moves Tengu; save mid-transition restores it.
5. **Tengu phase 2 abilities.** Add Tengu's bomb, fire wave and shocker as separate files, picked on a cooldown that shortens as HP falls. Tests: each ability fires and resolves; cooldown scales with HP.
6. **Dwarf King phases.** Phase 1 summons real subjects (ghoul, monk, warlock) and teleports to the throne; phase 2 shields the King on the throne while pedestal waves spawn until the wave is cleared; phase 3 is the final melee. Leave a clearly named hook for the Crown drop. Tests: phase 2 shield holds until the wave dies; summons respect the cap.
7. **Yog phases and fists.** Add the remaining four fist types (soiled, rusted, bright, dark) and a Larva mob. Yog summons fist pairs by phase and larvae between them, and the laser becomes a Ballistica beam that hits every character in its path. Tests: a new fist pair arrives each phase; the beam hits a character standing between Yog and the hero.

Steps 1 and 2 are the highest value per line: step 1 fixes two real bugs and unblocks every later step, and step 2 turns the one boss whose mechanic is visibly broken into a real fight.

## Verification note

Godot 4.5 can be downloaded into the container (github.com release zip) to run the suite locally; CI runs it too (`godot --headless --path . -s res://tests/run_tests.gd`, strict since #26). Each PR waits for green CI before it's called ready.
