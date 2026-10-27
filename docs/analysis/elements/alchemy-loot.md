# Alchemy and loot: best path

Element 2 of the [queue](queue.md). Covers [improvement-plan](../improvement-plan.md) items #11 (alchemy), #12 (bags, bombs, spells unobtainable), #13 (shop), #14 (Blacksmith and Wandmaker) and the consumable half of #17 (exotics, tipped darts).

Checked against `main` at 421178c (2026-10-14). Upstream numbers marked *verify* are from memory of SPD and must be source-checked when the step is built, as this repo's backlog convention already does.

## Where things stand

**The item classes mostly exist; what's missing is the plumbing that creates them.**

- `Bomb` (11 types, real blob effects, tests), `Spell` (9 types), `Bag` (4 types, routed pickups via `belongings.gd:57-84`) all work when spawned. Nothing spawns them: the only `create_item("bomb")` outside the generator is a Ring of Wealth bonus drop (`ring.gd:164`), and `BAGS`/`SPELLS` are referenced only by the factory (`generator.gd:529-546`) and the Recycle pool (`spell.gd:363`).
- **Alchemy has no energy at all.** `Recipe.energy_cost` is a number that is displayed (`wnd_alchemy.gd:206,235`) and passed to the toolkit's XP hook (`artifact.gd:1677`), but never charged. There's no energy counter, no energy crystals, and no way to "energize" items.
- **The recipe list is inverted and exploitable** (`recipe.gd:83-108`). SPD turns 1 scroll into 2 stones; the port turns 2 stones into 1 scroll. `augmentation ×2 → upgrade` mints Scrolls of Upgrade from shop stock (the Sewers shop sells augmentation, `shopkeeper.gd:83`). Seeds need 3 of the same type; SPD takes any 3 seeds and only guarantees the matching potion when all 3 match.
- **Labs can't appear before depth 5** (`regular_level.gd:320-322`), and the room is just a pot and embers (`laboratory_room.gd:28-37`). SPD guarantees roughly one lab per chapter from the first chapter and stocks it with energy crystals (*verify* `Dungeon.labRoomNeeded`).
- **Rosters have drifted from the shipped SPD strings.** `assets/spd/messages/items/items.properties` (the current SPD version) has 12 exotic potions, 12 exotic scrolls, 6 brews, 8 elixirs, 12 spells, a Shrapnel Bomb and Stones of Detect Magic and Aggression. The port has none of the exotics, brews or elixirs; its spell list still has the old Aqua Blast and Feather Fall (now a brew and an elixir upstream) and lacks Telekinetic Grab, Beacon of Returning, Magical Infusion and Unstable Spell; it has a Shock Bomb that the shipped strings don't.
- **The shop is invented** (`shopkeeper.gd:64-120`): fixed per-region lists with hand-set prices, every item force-identified (`:126-127`, leaks identities), no weapons, armor, missiles or bags, and a duplicate Healing (`:71` plus `:91,96,101`). Selling already uses `item.value()` correctly (backlog S26).
- **Blacksmith** (`blacksmith.gd`): 15 bat-dropped ore for one reforge. No pickaxe, mining, favor, or the four other rewards. **Wandmaker** (`wandmaker.gd:50-63`): rolls one of three quest items but only Rotberry can actually be found; Corpse Dust and Embers have no quest rooms, so two thirds of Wandmaker quests can't be completed.
- **Bag UI gap** (backlog S18 follow-up): bags route pickups, but there's no window to open one and picking a bag up doesn't sweep matching items into it. Making bags obtainable without this would be confusing.

## The path

The dependency spine is: **fix the recipe model → add energy → make labs reachable → add content that costs energy → hand it out through the shop and quests.** Energy is the keystone: bombs, spells, exotics, brews and elixirs all cost it, so nothing past step 2 is worth building on the current energy-free `Recipe`.

Two design choices to make once, in step 2, so every later step is just data:

1. **Recipes become rules, not a fixed list.** SPD's recipes are classes that match ingredient *patterns* (any 3 seeds; any potion + energy; bomb + one of N catalysts). Keep `Recipe` but give it a `matches(ingredients) -> bool`, `cost(ingredients) -> int` and `brew(ingredients) -> Item`, with today's fixed recipes as one subclass. `wnd_alchemy`'s known-recipe list then reads a `describe()` per rule.
2. **Energy lives on the hero, saved with the run.** An `alchemy_energy: int` on `Hero` (SPD keeps it on the alchemy scene's per-run state, *verify*), serialized next to gold, with energy crystals as a stackable item that adds to it when placed in the pot. Co-op: each hero's own pool, matching how gold is per-hero.

Exotic potions and scrolls should be **subclass-free variants** of `Potion`/`Scroll` keyed by id, the way `Bomb` uses `BombType`, because both files are large and the port's factories already dispatch by id. Each exotic shares its base item's identification (SPD identifies the exotic when its base is known).

## Ordered PR-sized steps

Each step is independently shippable, keeps the suite green, and adds a `tests/cases/test_*.gd` (auto-discovered since #26).

1. ✅ **Shipped in [PR #32](https://github.com/jvmisxn/jmo-pixel-dungeon/pull/32) (2026-10-14).** **Stop the Upgrade exploit and match SPD's basic recipes.** Replace the 11 stone→scroll recipes with SPD's scroll→2 stones (Upgrade→Enchantment, Transmutation→Augmentation, Remove Curse→Detect Magic stand-in, etc., *verify* map), and the 11 same-seed recipes with any-3-seeds → weighted random potion (guaranteed match when all 3 agree). Introduces the rule-based `Recipe` shape from design choice 1. Files: `recipe.gd`, `wnd_alchemy.gd`. Small.
2. **Alchemical energy.** Hero energy pool + serialization, `energy_crystal` stackable item, pot accepts crystals, `Recipe.cost()` actually charged and the Brew button disabled when short, Alchemist's Toolkit discount applied for real (`artifact.gd:1688`). Energy shown in `wnd_alchemy`. Files: `hero.gd` (append only), `recipe.gd`, `wnd_alchemy.gd`, `generator.gd`, `save_manager` round-trip test. Medium.
3. **Labs from the first chapter, stocked with crystals.** One guaranteed lab per chapter on SPD's floors (*verify*), drop the `depth >= 5` gate, paint energy crystals in the room. Files: `regular_level.gd`, `laboratory_room.gd`. Small.
4. **Bombs through alchemy and drops.** Bomb + catalyst recipes for the 10 variants with SPD energy costs; plain bombs into the shop and SPD's drop sources; rename Shock Bomb to Shrapnel Bomb (keep `BombType` int order for saves). Files: `recipe.gd`, `bomb.gd`, `generator.gd`. Medium.
5. **Exotic potions** (12) as Potion variants + potion→exotic energy recipe, sprites from the shipped item sheet, shared identification. Files: `potion.gd`, `generator.gd`, `recipe.gd`, `item_catalog`. Large; split by six if the diff runs long.
6. **Exotic scrolls** (12), same shape as step 5. Large; same split rule.
7. **Spells roster to current SPD + recipes.** Recipes for the existing 7 current-SPD spells; add Telekinetic Grab, Beacon of Returning, Magical Infusion, Unstable Spell; retire Aqua Blast and Feather Fall as spells (they return as a brew and an elixir in step 8; keep their ids loadable for old saves). Files: `spell.gd`, `generator.gd`, `recipe.gd`. Medium-large.
8. **Brews (6) and elixirs (8).** New `Brew`/`Elixir` variant files under `src/items/potions/`, recipes, effects routed through the existing blob layer (Aqua Brew → water, Caustic → `CorrosiveGas`, etc.). Large; brews and elixirs can ship as two PRs.
9. **Bag UI, then bags obtainable.** A window to open a bag and the SPD pickup sweep (backlog S18 follow-up), so bags are usable before players can get them. Medium.
10. **The real shop.** Port SPD `ShopRoom.generateItems` per shop floor: tiered weapon/armor/missiles, `ChooseBag` (the bag matching most held items, each once per run), Healing once, random unidentified potions and scrolls, stones, bomb, Ankh, Alchemize; prices from `item.value()` × SPD's depth multiplier (*verify*) instead of hand-set numbers; stop force-identifying. Depends on steps 4, 7, 9. Files: `shopkeeper.gd` (read in full first: backlog marks it as a previously truncated file), `shop_room.gd`. Medium.
11. **Wandmaker quest rooms.** Mass Grave (corpse dust + wraiths), Ritual Site (candles → newborn elemental → embers), and the existing Rot Garden for rotberry, spawned with the quest so all three quest types are completable. Files: new rooms under `src/levels/rooms/special/`, `quest_handler.gd`, `wandmaker.gd`. Medium.
12. **Blacksmith favor and rewards.** SPD favor (per-ore favor, cap, *verify* 50/2000 already noted at `blacksmith.gd:11`) and the reward menu: reforge, harden, upgrade, smith, keeping bat ore as the source for now. Files: `blacksmith.gd`, new reward window. Medium.
13. **Blacksmith pickaxe and mine.** Pickaxe item, the quest mine sub-level with crystal/gnoll/fungi variants and their quest boss, ore from mining instead of bats. Large; likely two PRs (mine + one variant, then the other variants). Ties into the bosses/mobs work (Gnoll Geomancer and the crystal mobs in #17).
14. **Tipped darts** (10): dart + seed recipe and the tipped-dart effects; unlocks the shop's tipped-dart slot. Medium.

Out of this element: the Cleric, the six missing weapons and the missing mobs in #17 belong with class/endgame and bosses/mobs work; trinkets (shipped strings include 18) are a later pass once energy exists, since trinkets are bought with energy upstream.
