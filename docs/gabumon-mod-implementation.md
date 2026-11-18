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
