# Seeded runs, challenges, daily seeds, badges

Element 4 of the [queue](queue.md). Plan items: Tier 2 #16 (challenges, custom and daily seeds, badges) and the Tier 5 `run_seed` finding in [improvement-plan.md](../improvement-plan.md).
Checked against `claude/project-thread-0h6j6a` on 2026-10-14.

## Where things stand

- **`run_seed` exists but drives almost nothing.** `GameManager.new_game` (`src/autoloads/game_manager.gd:141-150`) calls `Generator.full_reset()` first, which itself rolls `randi()` to choose decks, and only then picks `run_seed = randi()`. The seed is saved and restored (`game_manager.gd:683/704`, `save_manager.gd:436/457`) and is passed through `loading_scene.gd:73-74` and the online start (`hero_select_scene.gd:988`, `lobby_scene.gd:835`, `network_manager.gd:562`).
- **The only consumer is `ItemAppearance.reset_for_new_run`**, which uses its own seeded `RandomNumberGenerator`. So potion colors and scroll runes are already reproducible per seed; nothing else is.
- **There are 403 global RNG calls** (`randf` 149, `randi_range` 138, `randi` 69, `randf_range` 28, `shuffle` 19). By folder: levels 132, actors 126, items 100, scenes 17, effects 8, autoloads 8, ui 7.
- **Level generation is one synchronous call.** Fresh floors come only from `LevelFactory.create_for_depth` (`src/levels/level_factory.gd:8`, called from `loading_scene.gd:287`). No `await` or threads exist under `src/levels`. Revisited floors come from `_level_cache`, and loaded games from the save. Online clients receive host snapshots, so only the host generates.
- **Generator deck state is already saved** (`serialize_decks` and `restore_decks`, `generator.gd:353/367`, written at `game_manager.gd:695`).
- **There are no challenges.** Code comments mark where upstream challenge branches were dropped (`hero.gd:2137`, `herbal_armor.gd:32`, `wand.gd:1301`, `sorrowmoss.gd:10`).
- **There are 28 badges** (`badges.gd:413`, against about 100 upstream), saved to `user://badges.dat`. `tests/cases/test_badges_registry.gd` already guards that the badge count matches what `unlock()` can reach.
- **Rankings exist** on the title, death and victory scenes, which matters because seeded runs must be kept out of them.

## Best path

**Don't touch the 403 call sites.** Do what Shattered does instead. A seed fixes the dungeon: layouts, floor loot, traps, mobs at spawn, and item identities. Combat rolls stay random. Godot's global `seed()` drives every bare `randi()`/`randf()`/`shuffle()`. So the whole generation pass can be made deterministic by wrapping `LevelFactory.create_for_depth` in `seed(hash([run_seed, depth]))` and calling `randomize()` afterwards. This is one choke point and needs no API churn.

Because the seed is set per depth, it doesn't matter whether the global RNG state was saved. The next fresh floor after a save and load is still the same.

Three things to get right:

1. **Ordering in `new_game`.** Pick the seed first. Then run `Generator.full_reset()` under `seed(run_seed)` so deck choice is part of the seed.
2. **Generator deck drift.** Mob drops and other mid-play draws advance the shared decks, so later floor loot can differ between two plays of one seed. Layouts and mobs won't differ. Shattered accepts the same drift. Step 1 accepts it too and documents it. If exact loot parity matters later, floor-generation draws can take a deck snapshot keyed by depth.
3. **Hidden nondeterminism.** `standard_painter.gd:96` keys room pairs by `get_instance_id()`. That should be stable relative order, but the determinism test will prove it either way. Any `RandomNumberGenerator.new()` without a seed inside generation would also break it. None exist under `src/levels` today; `surface_scene.gd` is cosmetic only.

The order after that is set by what each step unlocks. Seeds are the foundation. A seed-entry UI and daily seeds are cheap once seeds work. Challenges are the biggest gameplay payoff, but they need a flags field threaded through the same new-run path, so they follow the seed UI. Badges come last because many of the new ones are about seeds and challenges ("win with N challenges"). Badges also need a rule: no badges or rankings on custom-seed runs, as upstream does.

## Steps (one PR each, in order)

1. **Deterministic generation.** _Shipped as PR #30 (https://github.com/jvmisxn/jmo-pixel-dungeon/pull/30), awaiting merge._ Seed before `Generator.full_reset()`. Wrap `LevelFactory.create_for_depth` in a per-depth `seed()` and `randomize()` pair, plus `random_item` draws done during level creation. Add `tests/cases/test_seeded_generation.gd`: generate depths 1 to 26 twice from one seed and assert `serialize()` matches; a different seed must differ. Fix any nondeterminism the test finds. No UI change.
2. **Seed codes and custom seed entry.** Add a `DungeonSeed` helper that converts between an int and a 9-letter `ABC-DEF-GHI` code, as upstream does. Show the code on the death, victory and in-game info screens. Add a "Custom seed" field to `hero_select_scene.gd` that passes `run_seed` through `LoadingScene`. Flag the run `custom_seed`, which skips badges and rankings. The online host reuses the same field.
3. **Daily run.** The seed comes from the UTC date. Add a "Daily" button on hero select, with one attempt per day stored in `PlayerProfile`. Show the best daily score in rankings with its own tag. Test the date-to-seed function.
4. **Challenge framework plus the first three challenges.** Add a `challenges: int` bitmask on `GameManager` that is saved and loaded and shown on hero select. Unlock it after the first win, as upstream does. Start with the three that hook into a single place each: On Diet (food yield), Pharmacophobia (healing potions), and Into Darkness (light radius and fog). Restore the dropped branches noted above as each one lands. Each challenge gets one test.
5. **The remaining challenges.** Faith is my Armor, Barren Land, Swarm Intelligence, Forbidden Runes, Hostile Champions, Badder Bosses. Badder Bosses may wait on the boss overhaul element (queue #1). If so, it ships as a stub and is listed as such.
6. **Badges, first batch.** Add seed and challenge badges: wins with 1, 3 and 6+ challenges, and the first daily win. Add per-class wins, the missing depth tiers (5/15/25), item-level and strength tiers, and upstream "catalog" badges where systems already exist. Extend `test_badges_registry.gd` coverage. Guard so `custom_seed` runs never unlock badges.
7. **Badges, second batch.** Add the subclass, boss-specific and alchemy ones once their systems exist (boss overhaul, alchemy element). The target is upstream's roughly 100. Then add a badge screen grouped by category if the current list doesn't scale.

Tests for every step run with `godot --headless --path . -s res://tests/run_tests.gd`.
