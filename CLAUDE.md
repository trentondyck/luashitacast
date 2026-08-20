# LuAshitacast profiles — HorizonXI (FFXI, 75-era)

Gear-swap profiles for HorizonXI. This repo is **the source of truth**; the game reads a
copy. Fork of Rag's LuAshitacast (upstream: `yzyii/luashitacast`), itself based on
`GetAwayCoxn/Luashitacast-Profiles`.

## Paths

Game install — an ~18 GB Proton prefix, Steam appid `3209151842`:

```
/home/deck/.local/share/Steam/steamapps/compatdata/3209151842/pfx/drive_c/users/steamuser/AppData/Local/HorizonXI/
```

Referred to below as `$HXI`. Inside it:

| Path | What |
|---|---|
| `addons/luashitacast/` | the addon's own source — read this to answer "how does it behave?" |
| `config/addons/LuAshitacast/Trent_46293/` | live profiles: `<JOB>.lua` + `settings.lua` |
| `config/addons/LuAshitacast/common/` | shared framework (gcinclude, gcmelee, gcmage, …) |
| `config/boot/ashita.ini` | Ashita boot config |
| `resources/findall/slips.xml` | short item-name lookup (slip-storable gear only) |

Character is **Trent_46293**. `Fatso_22898` and `Jacetms_61945` exist but have only
`settings.lua`.

## Workflow — always this order

1. Edit in this repo (`Rag_5040/<JOB>.lua`, `common/*.lua`).
2. Syntax check: `luac -p <file>`. **One file per invocation** — system luac 5.4
   double-frees on multiple args. It's a luac bug, not a problem with the file.
3. Copy into the game: `Rag_5040/*` → `$HXI/config/addons/LuAshitacast/Trent_46293/`,
   `common/` → `$HXI/config/addons/LuAshitacast/common/`. Keep the existing
   `settings.lua` — it is not in the repo.
4. Tell the user to run `/lac reload`.

Never edit only the game copy — it gets overwritten. `/lac reload` re-executes the
profile chunk *and* its `gFunc.LoadFile('common\\…')` calls, so `common/` changes are
picked up without relogging.

## Item names — the biggest trap

Sets must use `resource.Name[1]`, the **abbreviated inventory name**, compared as an exact
string (`addons/luashitacast/equip.lua:252-258`; both sides lowercased, so case is safe,
spelling and punctuation are not). **A wrong name fails silently** — no error, the slot
just never swaps.

Wiki titles and `addons/simplelog/lib/res/items_grammar.lua` store the *chat-log* name,
which is a different field. Do not validate against them.

- `Shaded Spectacles` → **`Shaded Specs.`** (trailing period)
- `Royal Knight's Belt +2` → **`R.K. Belt +2`**
- but `Goldsmith's Apron` and `Rogue's Poulaines` are **not** abbreviated

Length does not predict it. **Never guess a name from a wiki.** Instead have the user
equip the gear and run `/lac addset <SetName>`, which writes every slot using the exact
name and backs the file up first (`AddSetBackups = true`). `/lac gear` and `/lac validate`
do the same but need the **Packer plugin, which is not installed**.

## How the framework layers gear

`HandleDefault` runs on a tick and builds up in this order:

1. `gcmelee.DoDefault()` → `gcinclude.DoDefaultIdle()` equips **`Idle`** — the always-on
   base, applied even when engaged. Then, if engaged, layers `TP_LowAcc` (both accuracy
   modes) and optionally `TP_HighAcc`.
2. `gcmelee.DoDefaultOverride()` → `gcinclude.DoDefaultOverride()` layers `Town` (in
   listed town areas), `DT`/`Evasion`/resist sets by toggle, and `Movement` — gated on
   `player.IsMoving`, not on status.
3. SA/TA sets, then lockable gear, then `TH`.

Consequences worth remembering:

- A slot in `Idle` **persists into combat** unless `TP_LowAcc` also sets that slot.
- `Movement` fires whenever moving, engaged or not.
- All four aketon tables in `gcinclude-rag.lua` are commented out, so nothing contests
  the body slot in town.

## Action handlers

From `addons/luashitacast/packethandlers.lua`, by action packet category:

| Category | Handler | Notes |
|---|---|---|
| `0x03` spell | `HandlePrecast` → `HandleMidcast` | spells **only** |
| `0x07` weaponskill | `HandleWeaponskill` | |
| `0x09` job ability | `HandleAbility` | **this is the JA precast** — fires on the outgoing packet |
| `0x10` ranged | `HandlePreshot` / `HandleMidshot` | |

There is no separate JA precast function. Gear reverts after `AbilityDelay` (2.5s).

**Sneak Attack / Trick Attack are special.** SA's DEX applies when the attack *lands*, not
at activation, so gear must stay on across that gap. The profile does this by setting
`saOverride = os.clock() + 2` in `HandleAbility` (equipping nothing) and equipping
`sets.SA` from `HandleDefault` while the buff or the override window is live. Put SA gear
in `sets.SA`, **not** in `HandleAbility`.

## Settings that must not change

- **`HorizonMode` must stay `false`** in `Trent_46293/settings.lua`. Enabling it makes
  `HandleDefault` return early after its first run (`state.lua:179`), silently killing
  every set that depends on it — SA/TA, Town, Movement, DT.
- `i_can_read_and_follow_instructions_test` must stay `true` in `gcmelee.lua`,
  `gcmage.lua`, `gcinclude-rag.lua`, or the framework refuses to run.
- `load_stylist = false` in `gcinclude-rag.lua` — the Stylist plugin is not installed, and
  `true` prints "Plugin file does not exist" on every profile load. LuAshitacast's own
  `/lac lockstyle <setname>` covers the same need.

## Chat spam

The profile registers **42 aliases** on every load and Ashita echoes one line each
(`Set '/bres' to execute: …`). `state.LoadProfile` never calls `OnUnload`, so they
re-register on every `/lac reload`.

Fixed via `aliases.silent = 1` in `$HXI/config/boot/ashita.ini`. That file is **CRLF** —
patch it in binary mode; a plain `sed 's/^x = 0$/…/'` matches nothing and fails silently.
It is boot config, so it needs a game restart; the in-game `/ashita` settings window has
an "Enable Silent Aliases" checkbox that applies immediately.

## Useful commands (the user runs these in-game)

`/lac reload` · `/lac load <JOB>` · `/lac unload` · `/lac addset <SetName>` ·
`/lac naked` · `/lac lockstyle <SetName>` · `/lac debug` (toggles per-item equip logging)

## Current state of THF.lua

Populated: `Idle`, `Town`, `TP_LowAcc`, `Flee`, and the `WS_*` sets that shipped with the
fork. Still empty: `Movement`, `IdleALT`, `Resting`, `DT`, `MDT`, `Evasion`, all resist
sets, `SA`/`TA`/`SATA`, and `TH`.

`sets.TH` being empty makes all five `NeedTH()` call sites no-ops. `NeedTH()` in `Auto`
mode returns true only for a mob not yet in `taggedMobs`, i.e. it pays for TH gear on the
first hit against each new mob and then stops.

Unverified item name: **`Nanaa's Lucky Charm`** (HorizonXI-custom neck, Acc+3 / TH+1). It
appears in no local resource table. If TH+1 never shows up, that string is the suspect.

## Other jobs

All 16 job files exist but only THF has been worked on. The rest are Rag's defaults with
empty gear sets.
