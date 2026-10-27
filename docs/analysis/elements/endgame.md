# Endgame progression: best path

Element 3 of [queue.md](queue.md), plan item Tier 2 #10 in [improvement-plan.md](../improvement-plan.md).
Scope: class armor, armor abilities, King's Crown, tier-4 talents.

## Where the code stands (checked 2026-10-14, main at 421178c)

| Piece | State | Evidence |
|---|---|---|
| King's Crown | Missing. King's death only unseals the arena; nothing drops. | `src/actors/mobs/bosses/king.gd:96-101` |
| Class armor item | Missing. `Armor` has tier 1-5, glyph, augment, seal, mastery bonus, serialize. | `src/items/armor/armor.gd:34-451` |
| Armor abilities | Missing. No ability class, no hero field, no chooser window. | no matches for `armor_ability` |
| Tier-4 talents | Point maths exist but are switched off. `TIER_LEVEL_THRESHOLDS` already has 21 and 31; `talent_points_available_for` returns 0 for tier 4 with a comment saying it waits on armor abilities. | `talent_data.gd:21`, `hero.gd:1931-1949` |
| Talent window | Already groups by tier and shows per-tier points, so a tier-4 section appears for free once talents exist. | `src/ui/windows/wnd_talents.gd:58-73` |
| Inert-talent pattern | `TalentData._make_inert` + `implemented=false` already lets a talent be listed but not upgraded. Reuse it for abilities and their talents. | `talent_data.gd:180-184` |
| Hero art | All five hero sheets are 256x128 with 15px rows, so the class-armor row (row 6, as in SPD) is already in the sheet. `HeroSprite.update_armor(tier)` picks the row. | `assets/spd/sprites/*.png`, `hero_sprite.gd:80-92`, `game_scene.gd:944` |
| Item art | `assets/spd/sprites/items.png` is the SPD sheet, which carries the five class-armor icons and the crown. | |
| Subclass flow to copy | Potion of Mastery (dropped by Tengu) opens `WndChooseSubclass`, consumes the item, applies a permanent choice. The crown flow is the same shape. | `tengu.gd:102-106`, `wnd_choose_subclass.gd` |
| Max level | 30, so tier 4 earns 10 points (levels 21-30) under the existing bucket code. | `constants.gd:227` |

Gap worth knowing but outside this element: the Duelist has only 4 of her ~11 tier 1-3 talents (`talent_data.gd:96-101`). Endgame doesn't depend on it; it belongs with content (#17) or a Duelist pass.

## Upstream shape to match

- The King of Dwarves drops **King's Crown**. Using it on worn armor turns that armor into the hero's **class armor**, which keeps level, glyph, augment, curse state and the original tier's stats, and the hero picks **one of three armor abilities** for the rest of the run.
- Class armor holds a **charge** (0-100%) that refills over time; each ability spends a share of it.
- Abilities per class: Warrior (Heroic Leap, Shockwave, Endure), Mage (Elemental Blast, Wild Magic, Warp Beacon), Rogue (Smoke Bomb, Death Mark, Shadow Clone), Huntress (Spectral Blades, Nature's Power, Spirit Hawk), Duelist (Challenge, Elemental Strike, Feint). Ratmogrify is a secret fourth option open to every class.
- **Tier-4 talents** (levels 21-30): three per ability, plus one shared "Heroic Energy"-style talent that cuts charge cost. Only the chosen ability's talents are shown. Each has 4 points.
- Exact numbers (charge regen, costs, talent values) must be read from upstream `ClassArmor.java`, `KingsCrown.java` and each `ArmorAbility` subclass at the time of each PR, as the repo already does for other ports; this note doesn't pin them.

## Best path

Build the frame once, with every ability and talent registered as inert, and then fill it in one class at a time. This is the same pattern the talent system already uses, so each PR ships something playable and nothing shows up half-wired: an unported ability is listed in the crown window but can't be picked, exactly like an unimplemented talent today.

Why this order:
- The crown, class armor and ability chooser are useless without at least one working ability, so the first PR carries one simple self-targeted ability (Endure) to prove the full loop end to end.
- Tier-4 talents only make sense after an ability exists, and their gate (`hero.armor_ability != null`) is a one-line change once the field exists, so they go second as plumbing with inert entries.
- Per-class PRs then add three abilities, and a following PR adds that class's talents. Abilities that summon allies (Shadow Clone, Spirit Hawk) are the hardest, so their classes go after the simpler ones.

Coordination: the **bosses** thread owns `king.gd`. Step 1 touches it only to add the crown drop in `_on_death` (a few lines). If a King overhaul PR is open when step 1 ships, step 1 merges main after it lands rather than editing the same block.

Risks:
- Save format: class armor, charge and the chosen ability are new save keys. Bump `SAVE_VERSION` with a no-op migration and add a round-trip test, like the belongings tests (#23).
- Seal on class armor: the Warrior's broken seal must survive the conversion (`seal_attached`), and Runic Transference rules still apply.
- Targeted abilities need the existing cell-targeting flow used by wands and thrown weapons; reuse it rather than writing a new one.

## Ordered PR-sized steps

1. **[built 2026-10-14, push blocked; patch at endgame-step1.patch] Crown, class armor and ability frame, with Endure.** `ArmorAbility` base (id, name, description, charge cost, targeting mode, `activate`), a registry listing all 15 class abilities with only Endure implemented; `ClassArmor extends Armor` (keeps level/glyph/augment/seal/curse/tier, charge with per-turn recharge, "Use ability" action, serialize); `KingsCrown` item + `WndChooseAbility` (unimplemented abilities shown greyed out); crown drop in `King._on_death`; `Hero.armor_ability` field saved and loaded; hero sprite uses the class-armor row while class armor is worn; item icons from `items.png`. Tests: conversion keeps stats, charge spend and regen, choice is permanent, save round-trip, King drops a crown.
2. **Tier-4 talent plumbing.** Lift the tier gate in `talent_points_available_for` to require an armor ability; add `TalentData._ability_talents(ability_id)` and fold them into `get_talents_for`; register the shared charge-cost talent (implemented) and the three per-ability talents for all 15 abilities as inert; tier-4 header in `WndTalents`. Tests: 10-point band, no points before an ability is chosen, only the chosen ability's talents show.
3. **Warrior finished.** Heroic Leap and Shockwave, plus all Warrior tier-4 talents (Endure's included) switched from inert to real. Also port class armor TRANSFER here (left out of step 1).
4. **Mage abilities.** Elemental Blast (effect depends on the staff's imbued wand), Wild Magic, Warp Beacon.
5. **Mage tier-4 talents.**
6. **Huntress abilities.** Spectral Blades, Nature's Power, Spirit Hawk (an allied mob, so it needs ally AI and save support).
7. **Huntress tier-4 talents.**
8. **Rogue abilities.** Smoke Bomb, Death Mark, Shadow Clone (reuses the ally plumbing from step 6).
9. **Rogue tier-4 talents.**
10. **Duelist abilities.** Challenge (arena duel), Elemental Strike (varies by weapon), Feint (decoy).
11. **Duelist tier-4 talents.**
12. **Ratmogrify (optional).** The shared fourth ability and its talents. Skip if the user doesn't want the secret option.

Each step is a standalone PR on main, with headless tests (`godot --headless --path . -s res://tests/run_tests.gd`) and a `docs/memory/change-log.md` entry. The user merges.
