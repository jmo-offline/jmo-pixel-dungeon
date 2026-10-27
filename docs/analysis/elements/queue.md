# Element queue

Remaining work from [improvement-plan.md](../improvement-plan.md), split into one element per thread.
The recurring run works through this list one element at a time, top to bottom, and loops back to the top after the last.
A manual test pass pinged all six threads in order at 2026-10-14T05:18Z; the scheduled rotation still runs from 08:10 UTC and ships each element's next step after that.
Each element's thread writes its best-path analysis to `analysis/elements/<slug>.md` and lists its steps there.

| Order | Slug | Element | Plan items | Thread | Last run |
|---|---|---|---|---|---|
| 1 | bosses | Boss overhaul: Tengu, DM-300, Dwarf King, Yog | Tier 2 #9 | [bosses](#cmsg_014MZ69HVHv76nJaNNHmJhb2TdfDksgaSfgAVoMtYKzRBg) session `cse_011EiquzFgdP118ZNVFDPHj3` | never |
| 2 | alchemy-loot | Real alchemy and loot tables: energy, exotics, bombs, spells, bags, real shop, quest NPCs | Tier 2 #11-14, #17 | [alchemy-loot](#cmsg_014MZ69HVHv76nJaNNHmJhb23TUjKKPXTxzbziCBcmvZSq) session `cse_016nVTWXvbp4r5oKB5zJHg9J` | never |
| 3 | endgame | Endgame progression: class armor, armor abilities, King's Crown, tier-4 talents | Tier 2 #10 | [endgame](#cmsg_014MZ69HVHv76nJaNNHmJhb2U3vxFuJLBpDszxtHqqV3ZX) session `cse_01HSv61cKy2EyUYHM43X2XFG` | never |
| 4 | seeded-runs | Seeded RNG, then challenges, custom and daily seeds, badges | Tier 2 #16, Tier 5 run_seed | [seeded-runs](#cmsg_014MZ69HVHv76nJaNNHmJhb2MiT4PZZi4KtbrUoAH81yjn) session `cse_01XrVtbN6TRJd1zKSgBMUkjj` | never |
| 5 | visuals | Visual polish: raised walls and terrain variation, real SPD effects, softer fog, trap and plant art, HUD bars | Tier 3 #20-23, #25 | [visuals](#cmsg_014MZ69HVHv76nJaNNHmJhb294VdNVMfhNf3F3ga5iWYZa) session `cse_01SYi5MWrCR2jYqmv8UVsJva` | never |
| 6 | controls | Controls and settings: in-game gamepad, InputMap rebinding, hover cursor and path preview, long-press examine, fuller settings | Tier 4 | [controls](#cmsg_014MZ69HVHv76nJaNNHmJhb2Jb9NFY3W8vGW19f7XsuWEQ) session `cse_016s3St4GdhBD2YE3rE27xLU` | never |

Not split out (no standalone value yet): level variety (#15) folds into visuals or a later pass; code health (Tier 5) is done inside whichever element touches the code; web multiplayer (#7) is its own decision for the user.
