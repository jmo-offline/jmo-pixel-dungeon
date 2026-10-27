# JMO Pixel Dungeon: Biggest Improvements

Analysis of `main` at 5729698 (2026-09-30). Every item includes the file:line evidence. The top items in each section were re-checked by hand.

The port is broad and playable, and it already has the real SPD sprites, music and SFX. What's left falls into three groups:
1. A few things are silently broken or unsafe.
2. The late game (bosses, class armor, alchemy) is placeholder.
3. The UI ignores most of the SPD art it ships with, so it looks like a debug build.

---

## Tier 1: Fix first (broken, unsafe, or hiding bugs)

| # | Problem | Evidence | Fix |
|---|---|---|---|
| 1 | **CI can't fail on a test failure.** `godot ... \| tee test.log` has no `pipefail`, so the step only fails on SCRIPT/Parse errors. A failing assertion still shows green. | `.github/workflows/ci.yml:57-63` | Add `set -o pipefail` (or check `${PIPESTATUS[0]}`). |
| 2 | **6 test files never run**, including the newest PR's `test_cause_of_death_parsing.gd`, plus `test_pickup_delay`, `test_shuriken_instant_throw`, `test_mob_descriptions`, `test_ring_of_wealth_deck` and `test_scene_transition_current_scene_stub`. | The hand-kept `CASES` list in `tests/run_tests.gd` | Auto-discover `tests/cases/test_*.gd` with DirAccess. |
| 3 | **DM-300 heals 3 HP every turn for the whole fight.** `destroy_pylon()` exists, but nothing calls it, and the arena "pylons" are only statue tiles. | `src/actors/mobs/bosses/dm300.gd:36,93`, `src/levels/caves_boss_level.gd:44-54` | Make the pylons actors, or at least wire `destroy_pylon`. |
| 4 | **Save files are loaded with `allow_objects=true`.** A shared or edited save can create arbitrary objects and scripts. | `src/autoloads/save_manager.gd:190,582,629` (`get_var(true)`) | Use `get_var(false)` / `store_var(x, false)`. |
| 5 | **A co-op client overwrites its own solo save.** The autosave on close or pause and "Save & Quit" never check whether the game is a client. | `save_manager.gd:36-39,81-89`, `wnd_game.gd:101` | Skip saving when `NetworkManager.is_client()`. |
| 6 | **A partial save loads and reports success.** The current run is wiped first, missing sections default to `{}`, and the function always returns `true`. | `save_manager.gd:94-165` | Validate first, then apply. Fall back to the backup on any missing section. |
| 7 | **The web build can't do multiplayer.** ENet doesn't exist on the Web platform, but the web export is the one that gets deployed. | `src/autoloads/network_manager.gd:64,88` | Use WebSocket/WebRTC peers on web, or hide the lobby there. |
| 8 | **Debug text is shown to players** on every floor load ("Level loaded: entrance=...", "FOV: N cells"). | `src/scenes/game_scene.gd:664-682` | Remove it or gate it behind a debug flag. |

## Tier 2: Big gameplay and feature impact

| # | Gap | Evidence |
|---|---|---|
| 9 | **The bosses from Tengu on are placeholders** of about 110 lines each. Tengu has no prison phases. The Dwarf King's summons stop for good after 4, because `undead_count` is never decremented. Yog's "destroy the fists first" rule isn't enforced, and its laser does a flat 25 damage with no accuracy roll. | `bosses/tengu.gd`, `king.gd:50,88`, `yog.gd:9,83` |
| 10 | **No class armor, armor abilities, King's Crown or tier-4 talents.** The whole depth-20+ progression layer is missing. | No matches for crown/armor_ability; `talent_data.gd` has no tier 4 |
| 11 | **Alchemy is a fixed 22-recipe list.** There are no energy, exotics, brews, elixirs, spells or bombs. `augmentation ×2 → Scroll of Upgrade` breaks SPD's upgrade budget. Alchemy rooms can't appear before depth 5. | `src/items/recipe.gd:83-108`, `regular_level.gd:320-322` |
| 12 | **Bags, all 11 bombs and all 9 spells can't be obtained** in normal play. Nothing creates them. | `generator.gd:179,546` |
| 13 | **The shop is invented.** Its items come pre-identified, which leaks item identities. It never stocks weapons, armor or bags, and it lists healing twice. | `src/actors/npcs/shopkeeper.gd:64-120` |
| 14 | **The Blacksmith and Wandmaker quests are stubs.** The Blacksmith takes 15 ore for one reforge, with no mining. The Wandmaker has no quest rooms or elemental. | `npcs/blacksmith.gd:6-21`, `wandmaker.gd:50-63` |
| 15 | **Levels all feel alike.** There is only LoopBuilder, one rectangular StandardRoom (SPD has about 20), 1 special room per floor, and 3 of about 12 secret rooms. | `src/levels/builders/`, `regular_level.gd:20` |
| 16 | **Meta features are missing:** challenges, custom or daily seeds, and badges (26 vs about 100). | `hero.gd:2137`, `hero_select_scene.gd:988`, `badges.gd` |
| 17 | **Content is missing:** Cleric, about 20 exotic potions and scrolls, 6 weapons (Crossbow, Katana, Whip...), 10 tipped darts, and mobs such as Ripper Demon, Gnoll Geomancer and the Crystal quest mobs. | `generator.gd:212-214`, `mob_factory.gd` |

## Tier 3: Style and feel (cheap, very visible)

| # | Improvement | Evidence |
|---|---|---|
| 18 | **Use the pixel font and one global Theme.** The font is referenced nowhere. Styling is spread over about 570 color literals and 68 inline StyleBoxes. | `assets/spd/fonts/pixel_font.ttf` (0 refs), `project.godot` (no gui/theme) |
| 19 | **Use SPD chrome for every window.** Only the title screen uses the 9-slice. `UIUtils.chrome_stylebox` is never called. Windows have a text "X" close button, and Map/Menu are flat red boxes. | `ui/windows/wnd_base.gd:71-103`, `ui/ui_utils.gd:48`, `hud.gd:806-821` |
| 20 | **Use integer pixel scaling.** The current `canvas_items` + `expand` mode with no integer scale makes 16px tiles shimmer, and the HUD also gets a non-uniform scale. | `project.godot:40-41`, `hud.gd:540-545` |
| 21 | **Use raised walls, tile variation and animated water.** Terrain currently uses flat tiles only. The crystal/gnoll tiles, the terrain_features sheet and all 9 boss/quest arena tile sets go unused. | `tiles/terrain_visuals.gd:19-49`, `tile_map_manager.gd:166-190` |
| 22 | **Use real SPD effects** instead of vector dots. Particles and projectiles are `draw_circle` calls, and a thrown item shows as a beige dot. effects.png, specks.png and the fireball sheets are unused. | `effects/particle_burst.gd:22-36`, `projectile_effect.gd:60-69`, `game_scene.gd:1010` |
| 23 | **Make the fog softer.** The shader is black-only, fog snaps between states each turn, and visible cells dim with distance even on normally lit floors. | `tiles/fog_of_war.gdshader`, `fog_of_war.gd:72-100` |
| 24 | **Add fade transitions and UI sounds.** Scene changes are hard cuts, and buttons make no sound. | `autoloads/scene_manager.gd:45-53` |
| 25 | **Use real trap and plant art.** Traps are drawn procedurally, and plants reuse the seed icons. | `sprites/trap_sprite.gd`, `plant_sprite.gd:7-24` |

## Tier 4: Controls, settings, onboarding

- **Gamepad doesn't work in-game** (only in menus), and keys are hardcoded with no InputMap or rebinding. See `mechanics/input_coordinator.gd:110-190` and `game_scene.gd:394-528`.
- **Desktop hover does nothing.** `_hover_cell` is computed every frame but never read (`game_scene.gd:229-234`). There is no tile cursor or path preview.
- **Touch has no long-press to examine.** Examine is only available through the toolbar mode.
- **Settings are thin and duplicated,** with different defaults (SFX 50 vs 80: `wnd_settings.gd:310` vs `title_scene.gd:1534`). Brightness is saved but never applied at startup. There are no options for fullscreen, UI scale, screen shake or flash.
- **There is no tutorial or tips**, and no localization, even though `assets/spd/messages/` is shipped.

## Tier 5: Code health (enables everything above)

- **Co-op uses the host's field of view** to validate remote players' targeting. XP, Warlock and Monk effects only credit `GameManager.hero`, which appears 77 times across 32 files. See `hero.gd:520,635,796...` and `mob.gd:683,767,867`.
- **`run_seed` doesn't drive the global RNG.** There are 358 unseeded `randi`/`randf` calls, so runs aren't reproducible and seeds or daily runs are impossible (`game_manager.gd:143-150`).
- **Multiplayer sends the full world snapshot every turn** with no delta. A mid-run rejoin takes a slot by display name alone (`online_snapshot_utils.gd:4-16`, `network_manager.gd:470-498`).
- **There are god files and heavy duck typing.** `game_scene.gd` (2575 lines) and `hero.gd` (2653) are the worst. The code has about 1400 `has_method` and 1700 `.get("` calls, and about 210 `has_method` checks on autoloads that always exist. The coordinators take `scene: Variant` and reach into private fields 385 times.
- **Some EventBus signals are dead.** `mob_died`, `quest_updated` (11 emits), `game_saved` and `game_loaded` have no listeners, and a misspelled signal name fails silently (`game_scene.gd:1198-1202`).
- **Pathfinding is duplicated.** An O(n²) GDScript A* sits beside Godot's native AStar, which `Level.find_path` uses (`mechanics/pathfinder.gd`, `mob.gd:312`).
- **Root cleanup:** `TRUNCATED_FILES.txt` is stale, and `fix_nulls.py` hangs waiting for input.

---

## Suggested order

1. **Safety sprint (small):** items 1, 2, 4, 5, 6 and 8, plus the DM-300 pylon fix.
2. **Look-and-feel pass:** items 18, 19 and 20, then 24. Together they make every screen look like SPD.
3. **Boss overhaul:** Tengu, DM-300, King and Yog, in dungeon order.
4. **Real alchemy + loot tables:** energy, exotics, bombs, spells, bags, and the real shop.
5. **Endgame progression:** class armor, abilities and tier-4 talents.
6. **Seeded RNG, then challenges and seeded/daily runs.**
