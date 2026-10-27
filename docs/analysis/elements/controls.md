# Controls and settings: best path

Element 6 in [queue.md](queue.md), from Tier 4 of [improvement-plan.md](../improvement-plan.md).
Analysis of `main` at 421178c (2026-10-14). Line numbers are from that commit.

## What the code does today

**Keyboard.** Every binding is a hardcoded `KEY_*` constant in three places that must agree by hand:
- `InputCoordinator.handle_key_input` matches keycodes for every action (`src/mechanics/input_coordinator.gd:110-190`).
- `GameScene._movement_dir_for_key` maps numpad, vi-keys and arrows to directions (`src/scenes/game_scene.gd:512-531`).
- `_is_passive_hud_key` and `_is_local_action_key` repeat the same keys so co-op clients and the "not your turn" path can filter them (`game_scene.gd:392-404`).
- Held-move repeat polls `Input.is_key_pressed(_held_move_keycode)` (`game_scene.gd:570-585`), so it only works for keys.

There is no `[input]` section in `project.godot` and no `InputMap` use anywhere in `src/`.

**Gamepad.** Joypad events are handled only in menus, each with its own ad-hoc `JOY_BUTTON_*` checks (`title_scene.gd:1139-1161`, `hero_select_scene.gd:505-527`, `lobby_scene.gd:634-661`). `GameScene` ignores `InputEventJoypadButton` and `InputEventJoypadMotion` completely, so a controller can start a run but can't play it.

**Mouse.** Left click acts on a cell, and right click only cancels targeting (`game_scene.gd:344-370`). `_hover_cell` is updated every frame (`game_scene.gd:229-234`) and never read. There is no cursor, tooltip or path preview.

**Touch.** The gesture state machine handles tap, drag-to-look and pinch (`game_scene.gd:272-342`). There is no long-press, so examine is only reachable through the toolbar button or the X key (`input_coordinator.gd:13-25`).

**Pathing.** The hero moves one BFS step at a time toward a far target (`hero.gd:380-386`, `_step_toward` around `hero.gd:1100`). `Level.find_path` (AStar, `level.gd:796`) is a different algorithm, so a path preview drawn with it could disagree with where the hero actually walks.

**Settings.** These are worse than the plan said. Three things are actually broken, not just thin:
1. **Audio settings never load.** `SaveManager.save_audio_settings()` writes volume and mute to `user://settings.dat`, but `SaveManager.apply_settings_to_audio()` (`save_manager.gd:695`) has no callers, and `AudioManager._ready` (`audio_manager.gd:200`) doesn't read the file. Volume and mute reset to code defaults on every launch.
2. **Brightness is never saved.** Both settings screens call `GameManager.set("setting_brightness", ...)` (`wnd_settings.gd:267`, `title_scene.gd:1488,1510`), but `GameManager` declares no such property, so `Object.set` silently does nothing. The same goes for `setting_music_volume`, `setting_sfx_volume` and the two mute flags. Brightness is applied live by modulating `GameScene` (`wnd_settings.gd:240-252`), and that is lost on the next floor or launch.
3. **There are two settings UIs that disagree.** The title screen builds its own panel (`title_scene.gd:578-673`, handlers `1452-1580`) instead of reusing `WndSettings`. Their defaults differ (SFX fallback 50% in `wnd_settings.gd:310` vs 80% in `title_scene.gd:1534`). The title zoom picker never saves (`title_scene.gd:1477-1482`), and both fall back to the "1x" index while the real default is 1.5 (`game_manager.gd:59`).

**Missing options.** There is no fullscreen, UI scale, screen-shake toggle or flash toggle. The hooks for the last two already exist in one place each: `GameCamera.shake` (`game_camera.gd:121`), called from `game_scene.gd:2169` and `scene_feedback_coordinator.gd:105`, and `EffectManager.screen_flash` (`effect_manager.gd:120`).

**Tests that already cover this area:** `test_targeting_input`, `test_touch_gesture_routing`, `test_mobile_hud_input`, `test_stairs_click_input` and `test_game_camera_mobile_zoom` in `tests/cases/`.

## Best path

The order is chosen so each step is small, keeps behavior the same unless it says otherwise, and unblocks the next.

1. **Fix persistence before adding options.** New settings are pointless while the existing ones don't survive a restart. One settings owner with real declared fields fixes all three bugs, and every later step adds a key to it.
2. **Then collapse the two settings UIs into `WndSettings`**, so each new option is built once.
3. **Move input onto named actions before adding gamepad or rebinding.** Both depend on it. Doing gamepad first would add a fourth hand-kept list of buttons. Upstream SPD has the same split (`GameAction` names plus a `KeyBindings` table with separate key and controller defaults), and it maps cleanly onto Godot's `InputMap`. Register the actions in code from one defaults table, rather than editing `project.godot`, so rebinding can rewrite them from saved settings.
4. **Gamepad play comes next**, because it's the largest player-facing gap: controllers work in menus and then stop.
5. **Hover cursor and long-press examine are independent** of the action work and could go earlier, but they are lower impact than broken settings and a dead controller.
6. **Rebinding UI and gamepad window navigation come last.** They need steps 3 and 4 first.
7. **UI scale waits for the visuals element.** The HUD already applies a non-uniform scale (`hud.gd:540-545`), and the visuals element owns integer pixel scaling (plan #20). A UI-scale slider built before that lands would be rewritten.

Out of scope for this element: the tutorial, tips and localization bullet in Tier 4. They aren't controls, and localization touches every UI string, so it needs its own decision.

## Steps (one PR each)

1. **Settings that persist.** **Shipped as PR #28** (2026-10-14): audio volume and mute load at startup, the title zoom saves, and the dead title `setting_*` writes are gone. Brightness stays with #27, and #28 merges cleanly with it. The rest of the original scope below (a separate `GameSettings` store) was dropped as unnecessary. *Overlap note (2026-10-14):* open PR #27 (look-and-feel, branch `claude/project-thread-m6fnvm`) already declares `GameManager.setting_brightness`, saves and loads it with the display settings, adds `brightness_factor()` / `apply_brightness(scene)`, applies it on `GameScene` load, and aligns the `WndSettings` SFX fallback to 80%. It does not load audio settings at startup, and it does not touch the title screen's zoom save or its other `GameManager.set("setting_*")` calls. So this step builds on #27 (wait for it to merge, or branch from it) and keeps its brightness API rather than re-implementing it. The original scope follows: add a `GameSettings` store (declared, typed fields plus one defaults dictionary) behind `SaveManager.load_settings/save_settings`. Load and apply it at startup: audio volume and mute, brightness, zoom and orientation. Remove the dead `GameManager.set("setting_*")` calls, and apply brightness on every `GameScene` load, not only from the slider. Tests: a settings round trip, defaults for missing keys, and audio values applied after load.
2. **One settings window.** Replace the title screen's duplicate panel (`title_scene.gd:578-673,1452-1580`) with `WndSettings`, opened over the title. Hide the "Exit to Menu" button there, as it already does outside `GameScene`. Defaults now come from step 1. Tests: the window builds without a `GameScene`, and zoom saves from the title screen.
3. **New display and comfort options.** Add fullscreen (desktop only, hidden on web and mobile, using `DisplayServer.window_set_mode`), a screen-shake toggle and strength gated inside `GameCamera.shake`, and a "reduce flashes" toggle gated inside `EffectManager.screen_flash`. All three persist through step 1. Tests: shake and flash are no-ops when disabled.
4. **Input actions (no behavior change).** Add an `InputBindings` class that registers named actions (`move_n` … `move_sw`, `wait`, `search`, `examine`, `attack_adjacent`, `inventory`, `map`, `rest`, `quickslot_1-6`, `stairs`, `ascend`, `descend`, `cycle_hero`, `cancel`) from a defaults table holding today's exact keys. Rewrite `handle_key_input`, `_movement_dir_for_key`, `_is_passive_hud_key`, `_is_local_action_key` and held-move repeat to use `event.is_action_pressed` and `Input.is_action_pressed`. Tests: every key that worked before triggers the same action, and held-move still repeats and releases.
5. **Gamepad in-game.** Add controller defaults to the same table: D-pad and left stick for 8-way movement (stick angle snapped to 8 directions, with a deadzone), A to wait or confirm, B to cancel or open the menu, X to examine, Y for inventory, shoulders to cycle quickslots, and Start for settings. Make held-move repeat work for stick and D-pad. Route `InputEventJoypad*` through the same `_unhandled_input` path, including the co-op client filter. Tests: synthetic joypad events map to moves and actions, and stick diagonals snap correctly.
6. **Desktop hover cursor and path preview.** Add a small cell-cursor node drawn on `_hover_cell`, shown only while the mouse is the last-used input and hidden after touch or gamepad input. On a hovered, visited, reachable cell, draw the walk path. First expose the hero's own stepping as a `path_to(target)` helper next to `_step_toward`, so the preview matches the real route. Show a one-line name tooltip for a mob or item under the cursor using the `CellExaminer` text. Tests: path preview equals the cells the hero actually steps through.
7. **Long-press and right-click examine.** In the touch state machine, a single touch held for about 0.45 s without passing `TOUCH_LOOK_DRAG_THRESHOLD` calls `CellExaminer.examine` and swallows the release tap. On desktop, right-click examines when not targeting (it still cancels targeting first). Tests: extend `test_touch_gesture_routing` with long-press, a drag that cancels long-press, and a short tap that still acts.
8. **Rebinding window.** Add `WndKeyBindings`, opened from `WndSettings`. It lists each action with its key and its controller binding, captures the next key or button to rebind, warns on conflicts, has a reset-to-defaults option, and saves to settings so `InputBindings` re-applies the bindings at startup. Tests: a rebind round trip, and a conflict swap.
9. **Gamepad window and targeting navigation.** Set up focus neighbors in `WndBase`, inventory and quickslot windows so the D-pad and A/B can drive them. In targeting mode, the stick moves a target cell cursor (reusing step 6's cursor) and A confirms. After this, a full run is playable on a controller.
10. **UI scale** (after the visuals element's integer-scaling PR merges). Add a UI-scale option that multiplies the HUD canvas scale in `hud.gd:540-545` in integer steps, persisted through step 1.
