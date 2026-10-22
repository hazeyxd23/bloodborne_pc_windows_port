# NPC hunter fights with the whole party (ERSC parity D11)

0.7.3. User decision 2026-10-22: **the whole party fights.** A scripted NPC hunter appears in sync
for everyone, anyone can hurt it, and its death and the quest step count for all, like a boss.
Covenant outcomes stay **per player** (user decision; the covenants work owns those flags).

Code: `gpu/shim/party/party_npcfight_rules.h` (the decisions, unit test `party-npcfight-test`),
`party_npcfight.{h,cpp}` (the game side), the rule `party_npc_fights`. Smoke: phase `nf`, group
`npc_fights` (`tools/mp/smoke_party.py --only npc_fights`).

## 1. What the game data says (EMEVD scan of every map, 1.09)

Instructions searched in every `.emevd` (`flag_tool.py`'s parser, the events expanded per
InitializeEvent call): the AI-player summon / invasion instructions `2003[25]` (AI-player SOS),
`2003[51]` (NPC summon, type 0 cooperator / 1 invader), `2003[54]` (send an NPC home), plus every
team switch `2004[2]` / `2004[44]` and the multiplayer gates (`3[6]`, `1003[5/6/105]`, `3[29]`,
`1003[9/10/14-16/109]`, `3[22]`, `4[3]` on the local player).

| Instruction | Where | What it is |
|---|---|---|
| `2003[25]` | nowhere | unused |
| `2003[51]` type 0 | the Beckoning-Bell templates `xx04410` of m23, m24_00/01/02, m27, m28, m32, m34, m35, the Chalice `12906966` | NPC hunter **signs** (Old Hunter Bell): the only story use of the summon slots |
| `2003[51]` type 1 | m29 `12904634` | the Chalice Dungeons' NPC invader; it ends on a client (`1003[6]`) |
| `2003[54]` | m24_01 `12414480` | sends a summoned NPC home |

**The story's NPC hunter fights are not AI-player summons.** They are c0000 map chrs that the
map's events enable, warp, re-team and dress:

| Fight | Map, entities | Events | Team / look |
|---|---|---|---|
| Eileen with Henryk, Tomb of Oedon | m24_01: Eileen 2410900, Henryk 2410901 | 12410800 init, 12410809 Henryk appears, 12410811 Eileen joins (when Henryk's AI fights), 12410803 / 12410804 the kill (solo 1366 / together 1367), 12410806 Eileen hostile, 12410810 Henryk's death | Henryk team 23 (enemy, MSB); Eileen 28 (ally) + SpEffect 7608 (phantom look) |
| Eileen at the Grand Cathedral, the blood-drunk Eileen | m24_00: Eileen 2400900, 2400902 (blood-drunk), 2400903 | 12400650 init, 12400652 / 12400655, **12400656 falls to the blood (host only: `4[3]`)**, 12400658 hostile | 27 (hostile) |
| Alfred (Cathedral Ward) | m24_00: 2400760, 2400763 | 12400590 init, **12400593 hostile (host only: `4[3]`)** | 26 / 27 |
| Alfred executing Annalise, Alfred in Cainhurst | m25: Alfred 2500790, Annalise 2500600 | **12500050 init (team 27 after `1003[6]` end-if-client)**, 12500051, **12500053 hostile (`1003[6]`)**, 12500054 death; 12500010 the queen (1040-1059) | 27 |
| Valtr | m27: 2700912 | 12700901 init, **12700905 hostile (host only: `4[3]`)** | 8 / 27 |
| Djura | m23: 2300710 | 12300700-12300705 | 26 / 23; its gates accept chr type 0 **or 1**: it already runs on a guest |
| Hunter's Nightmare hunters | m34: 3400902, 3400903 | 13400950 init, **13400920 hostile (host only)** | 27 |
| Research Hall hunter | m35: 3500900 | **13500920 hostile (host only)** | 27 |
| Brador (Research Hall) | m35: 3500936 | 13505900-13505904: omen, appear, forced return, repelled | hidden: SpEffect 5560, invincible, no hit; fighting: 8060 |
| Fishing Hamlet assassins | m36: 3600900-3600902 | 13605900-13605940 | as Brador |
| Yahar'gul's hunters, Patches | m28, m32 | enemies of team 23 / 13200109 | ordinary enemies |

Covenant-defining quest states (per player, never written from here): Annalise 1040-1059 (1042 the
flesh: Alfred's execution, set by 12500010 when Alfred's 1348 is on), Alfred 1340-1359, Valtr
1790-1809 (`CovenantOutcomeFlag`; the covenants work makes them `covenant_outcome` in
party_flags.inc).

## 2. What a party saw before (live, nf runs 2-3, 2026-10-22)

Host and guest at the Tomb of Oedon with Eileen's state 1363 (default test save, Gascoigne dead):

- **Spawn:** Henryk and Eileen are there on both games (the same chr handles, 10014006 / 10014004).
  No slot is involved, so the party's size does not matter.
- **Sync:** the host's game owns them. Henryk's HP and position follow the host on the guest's game;
  a HP 0 written on the guest's game alone was back to the host's 1606 within a second.
- **A guest's hits:** R1 from the guest (locked on, team 23 = an enemy for a phantom) took Henryk
  1716 -> 1690 -> 1664 -> 1636 -> 1606 **on both games**.
- **Death and quest:** after the guest's killing blow the host's game killed him (12410810 = 1,
  12410804: 1367, the drop roll and the Echoes on both); the host's flags went to the guest's own
  pool (`guest applied 4 flags in load mode 1`, save requested).
- So the Tomb fight already worked. What a guest **misses** are the transitions that only the
  host's copy of an event runs (the bold events above): Alfred turning hostile (in Cainhurst even
  at every load), the blood-drunk Eileen, Valtr, the DLC hunters: the guest keeps a friendly or
  unloaded chr it cannot lock on to or hurt. Brador's appearance waits for the host's own position
  and goods, so a guest's copy never shows him (he stays hidden and invincible there).

## 3. The rule and what it does

`party_npc_fights` (the host's; System > Shared world *NPC hunters fight all*, the launcher's Party
Rules & Sync *NPC hunters fight the whole party*, bbport.ini `party_npc_fights`,
`BB_PARTY_NPC_FIGHTS` at start; default on; guests follow the host's, sent with the `balance` rules).

- **Guest copies of the host-only transitions** (`kGuestRuns`, the EMEVD dispatch filter's second
  skip predicate): on a guest in the host's world, each listed event's gate is passed (`4[3]`: the
  chr type it waits for becomes the guest's own; `1003[6]`: the client state becomes one the game
  never reports), so the guest's copy re-teams / shows the chr at the same moment as the host's
  (the HP condition reads the host's HP, which the game syncs). Its flag writes and save requests
  (`2003[2/9/17/22/31/32/41]`, `2000[5]`) are skipped: the host's copy writes them and C2 brings
  them into the guest's save, so no progress is written twice and no covenant outcome is written
  from here. On the host, solo or with the rule off, the gate's argument is the data's own again.
- **Assassins** (`kAssassins`): a guest shows Brador / the Hamlet assassin (drops SpEffect 5560,
  adds 8060, clears its invincibility) while the host-world flag its appear event sets is on, and
  hides it again when the host's forced return clears it. **Not verified live** (the fixture needs
  the Research Hall's omen conditions); the "no hit" state (`2004[39]`) has no known field yet.
- **Slots** (`AiSummonFits`): only the Bell signs and the Chalice NPC invader use the 5-entry slot
  table, under the game's four-player limit. The party's default of 3 players leaves one; a 4-player
  party has none (as in vanilla with three cooperators). Nothing is patched for it.

## 4. Live evidence (smoke phase `nf`, agents/npcfights/smoke/run2-run10, 2026-10-22)

| Case | Before (vanilla + party) | After (rule on) |
|---|---|---|
| Henryk + Eileen, Tomb of Oedon | Spawn on both games (same handles), Eileen joins as the ally (1365), the guest's R1s take Henryk 1716 -> 1488 on both games, the guest's killing blow kills him on the host ~5 s later, 12410810 / 1367 on the host and `1367=1/1` in the guest's own save at the next load (runs 3-7) | unchanged (no gate involved; runs 4-7 pass the same rows) |
| Alfred hostile, Cainhurst (state 1349; run 10 from the first lamp 2502950, read from the chr dumps at ~290 m) | host team 27, **guest team 26** (friendly: no lock-on, no hits; runs 5-10) | after a load the guest's copy of 12500050 passes its `1003[6]`: team 27 on the guest too (`guest copy of event 12500050 runs as the host's`, runs 4-10; run 10 17/17, the smoke as committed) |
| Blood-drunk Eileen, Grand Cathedral (1374 + 9802) | the guest that arrives after the host already gets her through the init event (12400650 label 1, no gate): team 27 on both (run 8) | same; the gate matters for a transition while the guest stands in the map. Her death from the guest's blow reached the host (1368, 1705) and the guest's own save (`1368=1/1`, run 8) |

Not verified live:
- a guest's **hits** on a hunter whose team the rule fixed: from the Queen's Chamber lamp (2502951)
  both players and Alfred fell through an undrawn void in this test save (runs 7, 9; also with the rule
  off, so not this change) and the blood-drunk Eileen stood idle
  (no lock-on on either game); the Tomb fight shows the damage path itself works for a hostile NPC
  chr (team 23);
- the mid-session transitions (12400593, 12400656, 12500053, 12700905, 13400920, 13500920) firing
  while the guest stands next to the NPC; the rewrite is the same as the init's (12500050), which was;
- the assassins (Brador, the Fishing Hamlet), Valtr, Djura, the DLC hunters;
- AI-player summons (Bell signs, the Chalice invader) with a full party.
