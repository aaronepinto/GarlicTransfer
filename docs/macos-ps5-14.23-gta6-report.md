# macOS / PS5 14.23 GTA VI save edit report

## Scope and evidence

One Grand Theft Auto VI (PPSA01547) story slot was downloaded, edited and uploaded back
to the same user on an unmodified PS5 on firmware 14.23, from a macOS 27.0.1 arm64 host
(Python 3.14.7) running the GarlicTransfer v0.3.13 release DMG. Decrypt and encrypt went
through the volunteer GarlicSaves workers, with the same `/api/v1/jobs` requests the Save
Editor makes. The edit was the GarlicSaves GTA VI editor (v2.3.1) "Max shared bank"
output, cut down to an exact amount, plus its "Max health" and "Full armour" output. The
bank balance was then checked in game; health and armour were not.

No raw logs, save contents, account IDs, console identifiers, network addresses or
credentials are included.

## Transfer

- Discover found the console. **Pair this PS5** on the Console page completed on macOS:
  the console showed as paired, and later browses logged
  `Power request identity: paired console`. No account ID or USB backup was needed.
- With the release DMG, **Browse Console Saves** failed with
  `OSError: [Errno 65] No route to host` although GarlicTransfer was allowed under
  Local Network. The DMG is signed with Electron's identifier rather than the app's
  (#30). For this report the app was started from Terminal, which applies Terminal's
  Local Network permission instead.
- The download needed one press of the PS5 power button. The console beeped and
  restarted, and the app then listed the saves.
- The GTA VI entry has nine containers. Sizes and SHA-256 matched the manifest.

| Container | Size | Contents |
|---|---|---|
| `sdimg_SAVEDATASARC0000` … `0005` | 8 MiB each | Story slots |
| `sdimg_sce_sdmemory` | 20 MiB | Autosave that **Continue** loads |
| `sdimg_SAVEDATAPlayer` | 6 MiB | Player profile |
| `sdimg_SAVEDATASettingsProfileSettings` | 6 MiB | Settings |

## Identifying a slot

A decrypted story slot contains `SARC000n`, `chives.dat`, `chives.json` and `sce_sys/`.
In `sce_sys/param.sfo`, `SUBTITLE` holds the mission name and `PARAMS` holds the save
time as a little-endian Unix timestamp at offset 0x68. The times matched the saves the
player had made, so the slot to edit was found without loading the game.

## `SARC000n` layout

`SARC000n` starts with a 16-byte header: `JSAV`, u32 version (3), u32 count of
CodeSaveBlocks sections (38) and u32 count of ScriptSaveBlocks sections (44). One
20-byte entry per section follows: u32 name hash, u32 `1`, u32 `0`, u32 offset, u32 size.
The table is in file order. Each section is compact JSON with hashed keys. After eight
zero bytes, the sections sit inside `{"CodeSaveBlocks":[\r\n … ],\r\n"ScriptSaveBlocks":[\r\n … ]\r\n}`,
separated by `,\r\n`, and the file ends with two zero bytes.

When the editor changed a section, only that section and the offsets and sizes in the
table changed. A file rebuilt the same way loaded in game, so no other checksum needed
updating.

## Shared bank

The balance is the sum of money records in the inventory section (`0xd3f59a5e`). The
inventory is a set of parallel arrays under `$0x789bdd41[0]/$0xfc8f2620`; record *i* is
element *i* of each: `$0x12ae4287` type, `$0xb4bfeab1` unique ID, `$0xbe8e255c` container,
`$0xc5730137` slot and `$0xe3fead7a` value, plus the boolean list `$0x63b44f39`. Money
records are in container `2084597891`:

| Type | Slot | Value the editor fills to |
|---|---|---|
| `6837455` | 75 | 15,000 |
| `2177639769` | 103 | 85,000 |
| `2657057649` | 111 | 400,000 |
| `867169743` | 133 | 1,500,000 |

`$0x789bdd41[0]` also holds `$0xabb1e011`, which has one handle per record
(`index * 16 + 1`) among handles of other kinds, and `$0xd558e4da`, indexed by slot.

The bank history is the `$0x1e1e8a08` list in section `0x6296c122`: 100 entries, each with
a sequence number (`$0x9d8e7980`), a delta (`$0xa662d8be`) and the resulting balance
(`$0x818f6f6d`). Before editing, its last balance, $17,971, matched the money records and
the balance shown in game.

## Editor "Max shared bank" (v2.3.1)

- Moved the three existing money records to the end of the inventory with new IDs
  (`8600000000000000001` onwards) and filled them to 15,000, 85,000 and 400,000.
- Appended type `867169743` records until the total was $999,000,000. The inventory
  grew from 773 to 1,439 records.
- Appended a `$0xabb1e011` handle for each new record, and one `$0xd558e4da` entry for
  slot 133.
- Dropped the oldest history entry and appended a deposit with the difference and the
  new balance.

## Setting an exact amount

Starting from that output for the same save:

1. Keep the money records in order until the target is reached, lowering the value of
   the last one kept.
2. Drop the remaining records from every parallel array and from `$0x63b44f39`, and drop
   their `index * 16 + 1` handles.
3. Keep the slot 133 entry in `$0xd558e4da`.
4. Set the deposit's delta and balance to the target.
5. Rebuild the section table.

For $1,500,000 this kept four money records (15,000 + 85,000 + 400,000 + 1,000,000). The
other records and sections matched the original.

A separate editor run with only "Max health" and "Full armour" changed two other
sections, `0xe528fe31` and `0x9653b39d`. Copying those two sections combined both edits.

## Encrypt and upload

- Encrypting (`operation=encrypt`, with the save's own account ID) and decrypting the
  result again returned `SARC000n` and `sce_sys/` byte-identical. `chives.dat` came back
  larger (526,752 to 1,686,688 bytes), and every mode in `chives.json` became 511.
- In Library, **More → Replace with another save…** with the encrypted container
  refreshed its size and SHA-256 in the manifest.
- After the GTA VI saved data was deleted on the console, **Upload to PS5** sent all nine
  containers of the entry, and the console answered `0xB3` and `0x94` with result 0.
- Loading the edited slot showed $1,500,000 in the bank.

This is one save on one console. The exact-amount edit starts from the editor's own
output rather than building the money records itself. The autosave container was not
edited.
