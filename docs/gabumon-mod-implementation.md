# Gabumon Partner Expansion — implementation notes

Target: Digimon World 3 USA (SLUS-014.36), branch `gabumon-mod-development`.

This document records verified source constraints before changing game logic or producing a patch. It is not a claim that a playable mod or patch has been built.

## Requested result

- Preserve the eight original partners and add seven partner slots (15 total).
- Add Gabumon, Garurumon, WereGarurumon, and MetalGarurumon as the first new line.
- Keep six additional partner slots available.
- Make the line work in field travel, battle, the Digimon Lab, Piximon switching, equipment, and gym training.
- Change evolution availability to Champion 5, Ultimate 20, Mega 40, without Wisdom/Fire/stat branch gates.
- Produce a patch for the verified USA disc image, not a replacement game image.

## Verified structural constraints

### Partner roster and save layout

In `include/engine/game_state.h`:

- `enum PartnerId` currently enumerates exactly eight partners and defines `PARTNER_COUNT`.
- `GameState.partners` is an inline array of `Partner[PARTNER_COUNT]` beginning at offset `0x075C`.
- The USA save-data region ends at `0x26BC`; the USA `mode` field begins there.
- The current eight `Partner` records occupy the space through offset `0x263C`, where `GameState.progress` begins.
- A `Partner` contains a full `PartnerStats`; each partner has 44 `PartnerEntry` records.
- The active battle party is separately limited to `PARTY_SIZE == 3`; this is not the same thing as the total partner roster.

Consequently, increasing `PARTNER_COUNT` from 8 to 15 changes the in-memory and saved-game layout. It cannot safely be done by only changing the enum. Every field after the partner array, save-size/checksum handling, memory-card versioning, and any hard-coded offsets must be audited. Existing saves must either be migrated explicitly or rejected safely; silently interpreting an old save using the new layout is unsafe.

### Digimon Lab

In `src/menus/digimon_lab/lab.c`:

- `STGDGLAB_partnerAnims` has exactly eight partner animation entries.
- `STGDGLAB_entries` is a separate ID-to-sprite/recipe lookup table terminated by ID zero.
- Lab file loading requests the Digimon name and information text resources as well as sprite resources.

Adding a partner therefore requires more than adding its Digimon ID: the partner animation table, sprite archive, ID lookup, and text resources must agree.

### Digimon data

In `src/engine/game/digimon.c`, `DIGIMON_DATA` is declared as 55 records: 52 Digimon plus three placeholders. Its data records carry battle stats, resistances, techniques, growth, name IDs, and family data. The four forms need valid entries and coordinated IDs/resources; inventing IDs without auditing every consumer risks table overreads and incorrect assets.

### Equipment

Weapon, armour, and accessory records each contain an 8-bit `partners` mask. This representation cannot express 15 partner bits. The equipment compatibility representation and all readers/writers must be changed together (or a backward-compatible extended mask introduced) before the requested all-equipment compatibility can work for partners 8–14.

### Gym

The gym uses partner-specific animation and growth data. The seven-frame `PartnerAnim` type is not itself the only requirement: the gym overlay's selection, growth, and result paths must all accept the expanded partner IDs and preserve the resulting stat changes.

### Evolution unlock implementation (source verified)

The exact unlock path is `STFGTREP_learnDigimon` in `src/battle/report/growth.c`. It scans 44 rows in `STFGTREP_evolutions[partner]` and, for each not-yet-owned Digimon, currently:

1. Checks up to two prerequisite Digimon IDs and their individual `PartnerEntry.level` values (`needs[2]`).
2. Checks `Evolution.stat` against one of the partner's six battle stats, the partner's level (`stat == 7`), or one of seven resistances.
3. Adds the Digimon to the partner's entries and first free one of three evolution slots.

The `Evolution` record is 16 bytes: target index, two `(digimon, level)` prerequisites, stat selector, and minimum value. The USA data defines eight lists of 44 records in `src/battle/report/data/stfgtrep.c`. Simply changing the shared function to ignore `stat` would still leave the two prerequisite-Digimon gates intact. Deleting all checks without a replacement tier rule would unlock unrelated Digimon immediately and would not implement Champion/Ultimate/Mega progression.

The safe change must preserve the target choices in all eight lists while replacing both kinds of conditions with a tier-derived partner-level threshold (5/20/40). The table's target IDs need to be classified into Champion, Ultimate, and Mega before rewriting the conditions. This classification is a required data-audit step, not something inferred from the row number. The current list's `stat == 7` records already encode some level thresholds (5/20/40), but other rows encode stat gates and prerequisite Digimon gates.

### Gym animation path (source verified)

`include/menus/gym.h` declares `STGTRAIN_waitAnims[PARTNER_COUNT][7]`. `src/menus/gym/screen.c` indexes this table directly with the partner returned by `GAME.funcs.getPartyMember` both for the selected Digimon and the party strip. The gym therefore needs a valid seven-frame entry for every added partner; a lab sprite entry alone will not make the gym display correctly. The training result code computes gains from the selected partner's `DIGIMON_DATA` growth record and writes the result into that partner's persistent stats, so the new partner's growth record must be valid as well.

### Lab partner-screen animation path (source verified)

`include/menus/digimon_lab.h` stores the outside-party partner list as `s32 partners[PARTNER_COUNT]`, and `src/menus/digimon_lab/lab.c` supplies `STGDGLAB_partnerAnims` for the partner-indexed sprites. The lab's party screen and gym both directly index partner-sized arrays; expanding the roster requires coordinated dimensions and frame IDs in both overlays.

## Follow-up source audit (November 2026)

### Evolution records do not carry a Digimon stage field

`DigimonData` is 0x57 bytes and contains combat values, techniques, growth, blast-form IDs, name ID, and family; it has no Champion/Ultimate/Mega rank field. `Evolution` is 0x10 bytes and contains a target `DIGIMON_DATA` index plus two prerequisite Digimon/level pairs and a stat selector/value. Therefore a correct level-only policy needs an explicit, reviewed tier classification for every target in each partner's 44-row evolution list (or a carefully designed classification table). It cannot safely infer tier from an absent `DigimonData` field or from row position.

The shared `STFGTREP_learnDigimon` loop returns after learning the first eligible row, so list order is gameplay-significant. A future implementation must preserve each partner's intended target ordering while checking the target's tier threshold. It must also decide whether the existing prerequisite Digimon requirements are removed, as requested by the global stat-gate removal, rather than merely bypassing the stat selector.

### Save growth calculation for 15 partners

The current `Partner` record is 0x3DC bytes (`pad[4]`, `unlocked`, and `PartnerStats`); the eight records occupy 0x1EE0 bytes from 0x075C through 0x263B. Seven more records would add 0x1A14 bytes before `GameState.progress`, moving every subsequent saved field unless the design instead stores the extra partners in a separate extension block. The current USA save is 0x26BC bytes and version 3. This confirms that simply changing `PARTNER_COUNT` would invalidate the documented GameState offsets and the fixed-size `GameSave`/`SaveData` representation.

### More fixed-size screens and the actual animation-table footprint

The USA symbol map records `STGDGLAB_partnerAnims` as 0xE0 bytes, exactly eight 0x1C-byte `PartnerAnim` records (seven 32-bit frame IDs each). `STGTRAIN_waitAnims` is likewise 0xE0 bytes, eight rows of seven 32-bit frame IDs. Those are actual compiled-data footprints, not merely declarations. Extending either table to 15 entries changes overlay data size and potentially its linker/symbol layout; each row also requires sprite IDs that exist in that overlay's loaded sprite archive.

The Lab party screen uses `screen->partners[screen->pick]` to select an animation by partner index and also uses `GAME.funcs.getPartyPartner(i)` for the three active party slots. The three active party slots can remain three, but the partner-indexed animation storage must cover all unlocked partners.

### Initialisation constraint

`initNewGameData` loops from zero to `PARTNER_COUNT - 1`, indexes `DIGIMON_DATA[i]`, and copies the corresponding name, base stats, and resistances into each partner record. The current data declaration has 55 rows total: eight partner definitions, 44 other Digimon, and three placeholders. Expanding the partner count without first adding and validating the four new partner definitions would make new-game initialization consume unrelated existing Digimon records as partner templates.

### Decision recorded

No gameplay source file was changed in this audit pass. In particular, no speculative evolution rewrite or `PARTNER_COUNT` increase was committed: the stage classification, new Digimon records/assets, save-extension design, and equipment-mask migration are not yet defined well enough to preserve original gameplay and saves safely.

### Equipment mask audit (source verified)

`WeaponData.partners`, `ArmorData.partners`, and `AccessoryData.partners` are each one byte at offset 0x04 in their respective records. `STSTATUS_canEquip` interprets this as `1 << partner`. This is a hard data-format limit: only partner indices 0–7 can be represented. The records are loaded from the item data archive and are read byte-by-byte by overlays, so widening the C member alone would shift record layouts and break the archive parser. A 15-partner implementation needs an extension mapping or a coordinated archive-format and reader change; it must also update the equipment UI's filtering and equip validation.

### Confirmed roster consumers beyond the Lab and gym

`initNewGameData` uses `PARTNER_COUNT` to initialise one `Partner` record from each leading `DIGIMON_DATA` row. `setParty` and starter packs unlock selected indices, while other loops use `PARTNER_COUNT` to scan unlocked partners and unequip items. `BattleReport` exposes partner-indexed evolution and animation pointers, and its report panel stores only three active party members, which is consistent with keeping `PARTY_SIZE` at three. These usages confirm that adding Gabumon as a ninth partner must be paired with its new `DIGIMON_DATA` template and all corresponding partner-indexed resources; changing the enum by itself is not a safe intermediate state.

### Work completed in this pass

- Added the evolution structure and gate behavior audit.
- Recorded the exact USA save growth (seven extra Partner records add 0x1A14 bytes before later GameState fields).
- Recorded compiled USA Lab/gym animation table sizes and their sprite archive dependency.
- Recorded the new-game initialization dependency on the first `PARTNER_COUNT` Digimon data rows.
- Recorded the 8-bit equipment mask limitation and why widening the struct alone is unsafe.

No executable gameplay code or game data was changed in this pass.

## Asset status

The USA data archive contains `AAA/DAT/FIELD/SPRT/P088GABS.BIN` (2,224 bytes), which is a candidate Gabumon field/NPC asset. Its exact frame set and suitability as a walking-follow sprite have not yet been established. It is not evidence of a complete walk-cycle.

No matching Gabumon/Garurumon/WereGarurumon/MetalGarurumon battle model was identified by the initial filename scan. The battle models, textures, animation data, attack scripts/effects, lab portraits, and gym sprites therefore remain asset tasks. The earlier concept-art sheet is not game-ready art and is not an approved source for WereGarurumon or MetalGarurumon.

## Build and patch gate

The upstream build workflow explicitly skips the full build on forks because the original executable and overlay inputs are private to the upstream repository. The local work environment currently has `make`, GCC, and Clang, but no detected PS1 MIPS cross-compiler. Network access from the build container is unavailable. No full game build, emulator test, or xdelta/IPS patch has been produced.

A release candidate is blocked until the PS1 toolchain and all required source/data inputs are available in a controlled build environment, the USA executable matches the verified input, and the resulting image passes emulator smoke tests.

## Safe implementation order

1. Map every consumer of `PARTNER_COUNT`, `GameState.partners`, the save-size constants, and fixed partner-index tables.
2. Design the 15-slot save layout and a versioned migration/rejection strategy before changing the struct.
3. Expand equipment compatibility beyond the current 8-bit masks.
4. Add the four Digimon data records and all ID/name/technique/evolution mappings.
5. Add field, battle, lab, Piximon, and gym assets and lookup tables.
6. Implement and test the level-only evolution thresholds without disturbing the original evolution destinations.
7. Build against the verified USA executable and original data, then test new-game, save/load, gym stat persistence, equipment, field following, battle actions, and all original partners.
8. Generate a patch against the unmodified USA BIN and publish checksums plus emulator test results.

Do not mark a step complete until its source changes and tests are present.

## Additional audit: equipment and partner portrait paths (2026-11-18)

Further source inspection confirms the eight-partner limit affects more than the save structure:

- `src/menus/field_menu/helpers.c`, `STSTATUS_canEquip`, checks `(data->weapon.partners >> partner) & 1`. The `partners` field is a `u8` shared by weapon, armor, and accessory records. Partner indices 8-14 cannot be represented by the current mask. This is a real runtime gate, not just an editor/data limitation.
- `src/menus/field_menu/helpers.c` defines `STSTATUS_partnerAnims[]` with exactly eight portrait animation records. The status/field menu therefore needs its own Gabumon-and-future-partner animation entries, separate from the gym and Digimon Lab tables.
- `include/menus/field_menu.h` exposes the field-menu helper interface; the animation pointer is part of `StatusData`, so adding entries must preserve the expected table ordering and sprite archive frame IDs.
- `src/menus/digimon_lab/party_screen.c` indexes `STGDGLAB_data.anim[member]` while drawing both the selected outside-party partner and the active party. Every partner index must have a valid animation record before the menu can safely display it.
- `include/menus/memory_card.h` confirms the USA save format is version 3 and copies the first `GAME_SAVE_SIZE` bytes of `GameState` as one contiguous block. A roster expansion changes that serialized block; it needs an explicit version bump and a deliberate old-save migration/rejection path, not merely a larger array.

### Implementation consequence

Do not widen `PARTNER_COUNT` as an isolated first edit. The same index is used to address the partner records, initialize partner data, render field-menu portraits, render Lab and gym animations, enumerate partners for events, and test equipment compatibility. The item data records are also loaded from the original game's data archives, so an extended equipment mask requires coordinated runtime and data-format work. The safe implementation should centralize partner capability checks behind a helper, define an extended mask representation, and audit every partner-indexed animation/data table before enabling indices 8-14.

### Confirmed scope expansion

Gabumon's portrait animation must be supplied in at least three separate menu contexts: field/status menu, gym, and Digimon Lab. The existing town NPC sprite is only a potential source for the overworld appearance; it does not satisfy these menu portrait requirements or prove a complete walk cycle.