# Parity checklist 04: Play tools window and domain logic

Source of truth: `Maple-Helper/maple-helper` @ 010107f (v0.18.1). Paths relative to `maplehelper/`. All formulas here must produce identical numbers (golden tests). Python semantics apply: `round()` is half-to-even, `//` floors, `{:,.0f}` formatting.

## 0. Window shell (`ui/tools.py`)
- Pages in chip order (3 per row, 6-col grid, short last row stretches): `train` Grind spots, `exp` Grind tracker, `farm` Farm, `loot` Loot Log, `quests`, `crafting`, `town` Citizenship, `build` Build & Skills, `gear`, `scrolls` Scroll Planner, `calc` Hit & damage, `prices` Item price, `more` Sell or keep, `route` How to get there, `shop` Cash Shop. `pets`/`market` open `shop`.
- CHAR_PAGES (train exp farm quests town build gear calc): no character -> "add a character" card + `add_character_requested`.
- Lazy build (requested page first, others one per tick). Redraw skipping for quests/town when state tuple unchanged (cid, level, class, job, quests_done, theme + page picks).
- Leaving a page resets it: quests mode -> "level" + search cleared; town grade + search cleared; crafting level chip + search cleared (advisor saves); farm monster pick cleared; `more` verdicts cleared; `route` destination + output cleared; gear/scrolls save.
- Signals out: sync_requested, grind_sync_requested, ask_requested(str, bool), detail_ask_requested, tag_requested(key) ("Ask in chat"), guide_requested(key), add_character_requested, loot_area_requested.
- Screenshot reads: hide window, act after 450 ms, re-show at 1500 ms.
- Cross links: `route:`, `calc:`, `track:`, `farm:`, `price:`; `zoom:` 96 px hover preview. "How to get there" from a monster -> its busiest open map (`farm._home`).
- `_num`: < 100,000 `{:,.0f}`; else if round(a/1000, 1) < 1000 `{a/1000:.1f}K`; else `{a/1e6:.2f}M`; optional +/- and `~`. Pinned: 999,960 -> "1.00M", 999,940 -> "999.9K", 123,400 -> "123.4K".
- `_clock`: < 60 min "N min" else "H:MM h".
- Persisted per character: stats{acc,dmg_min,dmg_max}, quests_done[], cycle_done{key:ts}, crafts{prof:lv}, craft_pct, craft_targets[] (duplicates = counts), craft_bag{key:n}, gear{slot:key}, ap{str,dex,int,luk}, gender, town. Settings: farm_target, grind_auto, route_taxi, loot_overlay, loot_region, craft_exp_table, market_window_pos, wishlist. Files grind.json, loot.json, mesowatch.json (+faces), osms_cash_shop.json (+images).
- EntityPicker: contains-match case-insensitive; if none and >= 4 chars, fuzzy `_close` (each typed word to a name word by prefix, or SequenceMatcher ratio >= 0.8 vs word or its first len+1 chars). Monster rows: drop special/unopened; one per name (most spawns); sort (level, name). Item rows: one per name; drop untradeable Cash. Map rows: grind_map only, sorted by "Monster levels Lv a-b" low end (999 missing), name. Route rows: towns first, then name.

## 1. Grind spots (`train`) — `combat.spots(kb, level, acc, dmg, magic, n=6)`
- Inputs: ACC 0–999, damage min/max 0–99,999 -> `c.stats`. `damage_range`: swap if min > max; missing max = min; None if min <= 0. Magician: damage ignored.
- Candidates: open monsters `level-8 <= mob.level <= level+6` (SPOT_BELOW 8, SPOT_ABOVE 6) with >= 1 grind map; exclude `special_monster` (name has "(", "Fairy N", "Dummy", or starts "Tutorial") and bosses (quickest respawn >= 1800 s); walk by total spawn desc, first per name.
- `hit = hit_chance(acc,…)` if acc else 1; `fits = not(acc and hit < 0.6)`. With damage: `hits, avg = hits_to_kill(dmin, dmax, m, level)`; `fits &= hits <= 12`; `score = exp*hit/avg`. Without: `score = exp*hit*level_scale(level, m.level)/max(1, hp)*1000`. Then `score *= min(1, top_map_count/20) ** 0.5`.
- recommended: any map prefix-matches (either way, lowercase) a map from `plan.spots_for(level, 5)`.
- Output: fitting spots by score desc, top n; none fit -> show non-fitting with "stretch" note.
- Card: picture, "Name · Lv", "map · region"; tags Best (first), From the guide, Most mesos, Hit X% (ACC set; green >= 99.9%), N hits, EXP, crowd count, community mesos, source chip, updated chip; info "ACC needed N" when hit < 0.999; kills to level.
- Most mesos: highest `kb.mesos_per_kill * hit / (avg_hits or 1)`, only if >= 2 spots have reports. `mesos_per_kill = (min+max)/2 * (chance if chance else 100)/100`.
- Kills to level `ceil(need*(1 - pct/100) / m.exp)`; REFERENCE chip when level in EXP guide "Levels a to b reproduce a historical reference".
- Links: How to get there, Hit & damage, Grind tracker, Ask in chat.

## 2. Combat math (`combat.py`)
- `hit_chance(acc, pl, ml, avoid)`: avoid <= 0 -> 1; `gap = max(0, ml-pl)`; `a = acc*100/(10*gap + 255)`; a <= 0 -> 0; `d = a - avoid`; `d >= 25 + gap*(0.5*pl + 15)` -> 1; `f = 0.15 + 0.20/(1 + exp(clamp(d/12, -50, 50)))`; `q = avoid/a`; q <= 1-f -> 1; q >= 1+f -> 0; else `(1 + f - q)/(2f)`.
- `acc_needed(pl, ml, avoid, target=1.0)`: smallest acc in 0..1999 with hit >= target - 1e-9, else 2000.
- `base_acc`: Magician `floor((1.2*INT + 2*L + 0.6*LUK)/5.1 + 20)`; others `floor((1.2*DEX + 2*L + 0.6*LUK)/div + add)` with Beginner (2.5,5), Warrior (2.5,10), Bowman (4.8,20), Thief (4.0,15); unknown -> Beginner. `acc_per_point = 1.2/5.1` (Magician) or `1.2/div`.
- `level_scale`: gap = ml - pl; <= 0 -> 1; < 10 -> `1/(1 + gap²*0.005)`; else `1/(1 + gap*0.05)`.
- `landed(raw, def, pl, ml) = float(min(99999, max(1, int(raw*100/(def+100)*level_scale))))`.
- `hits_to_kill(dmin, dmax, m, pl, magic=False)`: def = mdef if magic else pdef; lo = landed(dmin); avg = landed((dmin+dmax)/2); returns `(ceil(hp/lo), max(1.0, hp/avg))`.
- Monster parse `_monster`: props Level, HP, EXP numeric, hp > 0; AVOID / P.DEF / M.DEF = number on line after exact label line; maps = rows after "Map Locations" from 3rd line while lines contain " | " and col[1] digit, filtered by grind_map, first 4; respawn = min `respawn_seconds(last col)` over rows with >= 6 cols (text before "-", sum n*{h:3600,m:60,s:1}).
- `_CLOSED_MAP` (case-insensitive): `^(Warrior|Thief|Magician|Bowman|Pirate)'s |\bFor (…)\b|Accompaniment|KPQ|Party Quest|Test|Event|Exam`; `_NOT_GRIND` = that | `Hidden Street$`; `grind_map` = not _NOT_GRIND and map_open and not instance; `reachable_map` = not _CLOSED_MAP and map_open and not instance.
- `kills_to_level = max(0, ceil(need*(1 - pct/100)/exp))`.

## 3. Hit & damage (`calc`)
- Monster picker + shared stats. Empty -> `spots(level, acc, dmg, mage, n=1)[0]`. Typed -> exact name (most spawns) else contains (shortest, then lowest level) among open monsters.
- Shows picture; HP, EXP, Avoid, P.DEF; source/updated chips. Avoid <= 0 -> "never dodges"; else "ACC needed" = acc_needed(1.0) + 90% hint. ACC set -> "Hit %" round(hit*100); < 0.999 -> "N more ACC ≈ P DEX/INT" with `more = need100 - acc`, `pts = ceil(more/acc_per_point)`. Damage set -> "N basic hits" + "avg X.X" when avg < hits - 0.05. Community mesos `lo–hi` + avg or "none reported". Links route (if map), Farm it, Ask. "ACC to never miss as your level changes" for [L-5, L, L+5] (>= 1) only if avoid > 0 and values differ. Up to 3 maps with spawn count + route link.

## 4. Grind tracker (`exp`) — `grind.py`, `ui/grindrunner.py`
- Monster picker (partial -> shortest monster name containing it); sets running session monster (picked=True) or next-session choice. Start / Update / End; auto-update switch `grind_auto`.
- Reads are AI screenshot reads; `grind.reading` sanitises: level 1..250; exp_percent 0..100 (2 dp); map after last ": "; monster <= 80 chars; mesos 0 <= m < 1e11; potions/etc/equip `{name: int 0..99999}` summed per name; inventory = `inventory_open is True` or any inventory field.
- Runner: `AUTO_MS=60000`, `READ_WAIT=90 s` (stale pending dropped), `IDLE=20 min`. `ask()` -> "read" | "joined" | None. Start needs exp_pct else note grind_no_exp/exp_failed. Notes grind_started_ok, grind_no_inv, grind_updated, grind_saved, grind_not_saved, grind_end_no_read. Auto reads kept only with exp_pct or inventory. Character switch mid-read: End still ends; Start/Update -> exp_failed.
- Store grind.json `{active:{cid: session}, recent:{cid:[records]}}`; MAX_READS 60 (keep first + newest); MAX_RECENT 20; `end()` saves only if seconds >= 60 and any of exp/mesos/potions_cost/loot_value not None.
- `summarize`:
  - EXP runs `_agreeing(reads with level+pct, _level_fits, _level_jump)`: `_level_fits(a,b)`: `0 <= b.lv - a.lv <= 1 + int(max(0, b.t - a.t)/300)`; `_level_jump`: `0 <= Δ <= 2 + int(dt/300)`; first kept = first i with fits(r[i], r[i+1]) (else 0); later reads: append if fits(last); else new run if next exists and fits(r, next) and not fits(last, next) and jump(last, r); else drop.
  - gain = Σ runs `exp_position(last) - exp_position(first)`, `exp_position = Σ table[1..lv-1] + table[lv]*pct/100`; None if level not in table -> "no_table"; same read -> "one_read"; gain <= 0 -> "no_gain" (exp = 0 if exactly 0). span >= 60 s: `exp_h = round(gain*3600/span)`, `pct_h = round(gain*3600/span/need*100, 1)`, `to_level = round(need*(1 - b.pct/100)/(gain/span))`.
  - Mesos runs `_mesos_fit(a,b)`: `lo*5 >= hi or hi < 10000`; mesos = Σ(run last - first) when >= 2 reads total; `mesos_h = _per_hour` (None < 60 s; round(v*3600/s)).
  - Potions: first vs last read with potions; n1 > n0 restocked; equal skip; else (name, n0-n1, price, src), price from `potion_price` (cheapest released NPC shop; src = shop build label or MeowDB); sort used desc; potions_cost = Σ; net = mesos - cost.
  - Kills: monster_exp from KB (first exact-name); kills = round(exp/monster_exp); kills_h = per_hour(kills, span of EXP runs).
  - Expected mesos: `kb.community_mesos` -> (lo, hi, chance, count); chance > 1 is percent; avg = (lo+hi)/2*chance; expected = round(kills*avg).
  - Loot per tab (etc, equip) first vs last; gains n1 > n0 summed by name; priced `farm.value`; sort (-(n*price), -n, name); loot_h over max span.
  - Record: start, seconds, map, monster, level_from/to, exp, exp_h, mesos, mesos_h, potions_cost, net, kills, expected, loot [[name, n]] (first 12) or None, loot_value.
- Cells: time, exp, exp_h, mesos, mesos_h, net, kills (~), kills_h (~), potions cost; lines level-up, %/h, time to level, potions used/unpriced/restocked, community estimate. Recent table: map, monster · day, time, EXP/h (best green when > 1 row), mesos/h. "Which spot paid more" sends top 6 records as bullets to chat. Clock refresh 10 s.

## 5. Farm (`farm.py`)
- Item picker: farmable = items in `kb.droppers` (one per name); typed: `farm.item_key` (exact else shortest containing), else `_item_named`; not in droppers -> shortest farmable containing folded text. Saved `farm_target[cid]`. Wishlist chips (items with droppers).
- Item card: NPC sell-back (source = build or MeowDB), "FM price" link, droppers.
- `droppers(item, level)`: home = busiest grind map else first reachable; closed = no home but has maps; sort (source != community, fit == hard, boss, level, name); fit: no level "range"; mob <= L "easy"; <= L+6 "range"; else "hard".
- `targets(level, n=6, below=20, above=6, per_monster=3, wanted)`: open non-boss non-special monsters with maps; drops kept if NPC value or needed; drop sort (not need, -price, source != community, name); same-name dedupe keeps higher (needed, best); sort (-needed, -best, no community drop, level).
- `needs()`: wishlist ("wish"); "now" quests' item needs (not Defeat/Collect; crafting quests only for owned professions) -> ("quest", name); recipe ingredients of owned professions up to level -> ("recipe", name).
- Session section = same grind session; cells time, loot value, loot/h, kills~; loot lines "every N kills" = max(1, round(kills/n)).
- `farm.records(recent)` top 20 per (monster, item) with kills > 0 and loot; `every = max(1, round(kills/got))` if got < kills else `per_kill = got/kills`.

## 6. Loot log page (`loot`)
See 02-vision-platform §6 for parsing/tracker. Cells time, exp, exp/h, mesos, mesos/h, picked, value, value/h; feed last 8; recent 10; Start ends any other running log; record kept if exp, mesos or picked non-zero; overlay switch `loot_overlay`; "Select the messages area".

## 7. Quests (`quests.py`)
- Tabs level / missed / soon / later; search (200 ms debounce; all words in name/npc/area/needs/rewards/class/gender rewards); level chips on missed/later; 40 cards + "show more" +40; Done toggle.
- `for_level` (cache 16 by state): open, `job_fits`, `craft_fits` (profession level >= required; only when crafts given), not done; `opens_at = max(level, complete_level)`; now = opens_at in [L-200, L] excluding citizenship, sort (-exp, level); level = opens_at == L; missed = opens_at < L sort (-opens_at, -exp); future non-citizenship sort (opens_at, -exp): soon = L+1, later > L+1.
- `job_fits`: Maple Island quests hidden once non-Beginner; Job Advancement with known job only for that class's 1st job and if base class in q.job; "beginner" in q.job -> Beginners only; else base class or job in q.job.
- Parse prerequisites: "<Town>: Citizenship grade N", "Level N+ to complete", "Job X", "Quest Complete X", "Profession X Lv. N+", "Fame N +", "Pay N mesos to accept.", else notes. Requirements `_ITEM = ((?:Defeat |Collect )?\S.*?) x ([\d,]+)(?!\S)`. Rewards flat; class/gender prefix splits; "Pick one (class-specific)"; "Random reward - one of:" with `x N P %`. Head: Self-Starting, Daily/Weekly, "Start:", "Turn in:". Task: "Quest journal … 02 …" else line starting "01 ". Daily cleared if another quest requires this one, or "First Greeting with X" where "Asking After X" exists.
- Daily/weekly done -> `cycle_done[key] = now`; back_at daily next local midnight, weekly +7 days; `expire_cycles` on refresh.
- Card: NPC picture (or first capitalised NPC in journal, or fallback); "Lv opens_at"; Daily/Weekly; +EXP; "back at"; town (NPC's town, else `town_of`, else area); how to get it (NPC / self-start / Community Board); finished with (turn-in or journal NPC Town Clerk role); task (Hebrew via `assets/quest_tasks/he.json`); needs with pictures, route/calc links for Defeat, droppers top 3 nearest level (hard last, |L-lv|, lv; "Show all N"; or recipe via `made_from`); gets, pick one, random, gender rewards; prerequisite quest link; hints complete level, grade, profession, fame, fee, notes; Mark done/Undo (also `c.finish_quest`); Ask.

## 8. Crafting — Professions (`crafting.py`)
- 6 professions smithing, weaponcrafting, tailoring, woodcrafting, leatherworking, arcforge; stepper 1..max_level -> `c.crafts`.
- `info()`: mastery quest "A/An <Word> in My Own Right" (Blacksmith/Weaponcrafter/Tailor/Carpenter/Leatherworker/Arcforger); teacher = its NPC; start quest = teacher's crafting quest preferring "<teacher> in Need of an Apprentice" else lowest level; station NPC (Anvil, Weaponcrafting Station, Sewing Machine, Woodcrafting Station or "Woodworking Station", Leatherworking Station, Arcane Station) with towns from "Locations" (released only).
- Recipes page `crafting/efficiency__<prof>`: "Lv. N" blocks, "needs X EXP · char Lv. Y", row `"n | Name"`, ingredient `(\d+) x (.+?)`, values `| exp | catalyst | matval | matcraft | sellback | net | exp/meso | mats`. Now = `up_to(lv)` sort (-level, -exp_per_meso, -exp), first "best"; All = (level, -exp_per_meso, -exp); search name or ingredient; level chips.
- `next_level` = (lv+1, table[lv].exp, table[lv+1].char_lv) from `formula/leveling` "Crafting Levels" (`lvl | EXP to next | Total | Char level`), fallback efficiency page.
- Card +EXP, cost = catalyst, net, needs, droppers.

## 9. Craft plan (`craftplan.py`, `ui/craftadvisor.py`)
- Tables recipes, shops, drops, equips, consumables. Modes highest, cheapest, least (default least).
- `_pick`: player's Swap wins; one variant -> it; highest min(-prof_lv, unit_cost, num); least min(unit_unfarmed, unit_cost, num); cheapest min(unit_cost, num); unit_cost = _fresh/makes; unfarmed = qty of raws with no NPC price.
- `demand(name, count)`: spend bag, then pool (surplus); crafts = ceil(count/makes); steps merge per product; pool = crafts*makes - needed; required ingredients recursive; cycles -> raw + note "cycle"; raw value price*count charged to consuming step (or orphan).
- Level-ups: disciplines needing prof_lv > current (unlearned = 1); "early" if a non-final step needs it (before prep), else "late" (between prep and final). `exp_needed = Σ table exp current..need-1 − floor(exp[current]*pct/100) − Σ EXP of plan's earlier crafts in that discipline` (floor 0); blocked "exp" if any level lacks EXP; blocked `char:N` if next level's char_level > character. Recipe = min over discipline recipes with prof_lv <= current and exp > 0 of `((fee + Σ max(0, qty - spare bag)*price)/exp, -exp, num, product)`; crafts = ceil(exp_needed/exp). `craft_exp_table` (levels <= 10) overrides KB.
- Plan stages early, prep, late, final; steps sort (discipline, -depth, prof_lv, name). Farm list sort (category order ore, gem, material, drop, other; farm <= 0; -farm; name). Notes cycle, no_price. cost = Σ(step.cost + fee*crafts) + orphan; crafts, exp, equip_lv = max req_lv of targets.
- Craftable table: products with fresh cost/unit, NPC buy, save = buy - cost; default equips the class holds; search all; sort (non-equip, lv, -class stat, name); 12 rows + "and N more".
- UI: profession strip (level 1..10, bar %), Make stepper 0..50 (duplicate keys in craft_targets), mode segment, plan totals, farm rows (top-3 droppers best-voted + cheapest seller), stages with Swap and Crafted (bag accounting), bag boxes, EXP table (folded), saves debounced 350 ms.
- Quirks: `craftadvisor._key_of` fallback builds "item/item/N" (v2: fix to "item/N"; covered by a new test). `Step.cost` = raw-mat value only; fees added in Plan.cost (keep behaviour; fix docstring).

## 10. Citizenship (`town`)
- < Lv 12 locked card ("opens in 12−L levels"). Town = c.town, else guide's recommendation (`buildplan.citizenship_advice`: first of Henesys/Kerning City named in job's then class's guide "citizenship" section), else Henesys.
- Advice: first 2 paragraphs split to sentences; reasons = sentences without "best/Pick/sign/…"; clerk = NPC in town whose page has "Town Clerk"/"City Clerk".
- `citizenship(town, level, done)`: open citizenship quests opens_at <= L and town_of == town, sort (grade, -exp); grade chips, search, 40 per page, done list. `town_of`: grade town -> "(Town)" in NPC string -> exactly one town in quest name -> earliest town on NPC page. "Next grade" hint: lowest-grade quest above top shown.

## 11. Build & Skills (`buildplan.py`, `jobs.py`, `plan.py`)
- Build tab: guide key `plan.class_guide` (job, else JOB_BEFORE 3rd->2nd, else class; slug `guide/<name lower, "/" removed, non-alnum -> "-">-class-guide`); shipped guide JSON (en + local language aligned by block index); tables under h2/h3 matching "AP allocation", "SP allocation", "Recommended equipment"; per kind tables whose heading level range contains L, else latest reached, else first; `current_row` first-column range covering L else last row with lo <= L; `split_paths` (columns whose English header's first word is a KB weapon type, >= 2 paths); HTML tables with current row highlighted, skill icons (longest name first, <= 3 per cell, short names when unambiguous), "Changed in COT2" chips (first mention, cols 1–2), gear cells link (price / who drops); tier card `sitedata.tiers_for` (own 2nd job or class branches; per column grade S/A/B and "place x of y" = 1 + count strictly greater).
- Skills tab: class chips (Beginner, Warrior, Magician, Bowman, Thief) -> job segments (1st + 2nd; TOP_TIER 2); cards icon, max lv, changed/updated chips, desc, "requires X lv N", Lv1 and Max lines (Hebrew from translations); default tab own (3rd job -> 2nd).
- `plan.py`: exp_table from `guide/exp-table-level-1-to-100` (`lvl | need`); brackets from `guide/best-grind-maps-every-level` ("Level a-b", rows `Map | Street | EXP/hr | Mob (lvN) | … | Why`, dedupe map, sort exp_hr desc); `progress`: left = max(0, round(need*(1-pct/100))); below Lv 8 island monster (strongest lv <= L+2 whose top maps end Maple Road or Rainbow Street, by (lv, exp)); kills = ceil(left/exp); `next_job` (Beginner -> 4 classes at 10; else next open tier level); `tip` (job tip when next_lv-2 <= L < next_lv+3; map tip when current map's mob >= 8 levels below); `exp_rate`.
- `jobs.py`: tree (1st 10, 2nd 30, 3rd 70); `open_tier` from availability (default 2).

## 12. Gear (`gearadvice.py`, `ui/gearadvisor.py`)
- Inputs STR/DEX/INT/LUK (`c.ap`), gender, worn item per slot (`c.gear`).
- `holds`: job column ("Any" or "/"-list) includes base class; tier >= 2 weapon slots must be in JOB_WEAPONS[job] (Fighter/Crusader swords+axes; Page/WK swords+blunt; Spearman/DK spear+polearm; Hunter/Ranger bow; Xbowman/Sniper xbow; Assassin/Hermit claw; Bandit/CB dagger; mage jobs wand/staff).
- `wearable`: blocks on level (gap = req - L), job, gender (if set), stats (if typed and below); untyped requirements -> missing `stat:<st>:<need>`.
- `rank_key(base, item)` desc: weapon: main attack (M.ATK Magician else W.ATK), main stat, secondary, speed (1, -stage) or (0,0) if unstated, crit, acc; armour: main, secondary, wdef, hp, crit, acc. MAIN_STAT Warrior str / Magician int / Bowman dex / Thief luk; SECOND_STAT dex / luk / str / dex.
- `upgrades`: per slot (class weapon slots, job's at tier >= 2, then armour Hat, Top, Bottom, Overall, Gloves, Shield, Cape, Earring, Shoes); candidates strictly beat worn rank; now = fit.ok top 3; soon = only level blocks, gap <= 5, top 2 soonest; ties name then rank; deltas over DELTA_ORDER; price/seller from equips; droppers top 3 by votes ("N up" desc), level, name.
- UI: folding slot sections (empty folded at first draw), badge upgrade count, worn picker (8 rows + "None"), "Plan scrolls" on items with slots > 0.

## 13. Scroll Planner (`scrollplan.py`, `ui/scrollplanner.py`)
- Equips joined to scrolls by slot: `_canon` lowercase, hyphens->spaces; aliases top->topwear, bottom->bottomwear, overall->overall armor. Groups by stat word between slot and " Scroll"; grades by -success then Lesser/Intermediate/Greater; "Attack" groups first then name. Effects from `"Label +N, …"`.
- `plan(n, p, d)`: `go = 1 - (1-p)*d`; `dist[k] = C(n,k) p^k ((1-p)(1-d))^(n-k)`; `destroyed = 1 - go^n`; `all_pass = p^n`; `tried = n` if go >= 1 else `(1 - go^n)/(1 - go)`; `expected_successes = p*tried`. KB scrolls d = 0.
- `plans`: expected gain per effect = exp_succ*gain; cost = price*tried; per_point = cost/(exp_succ * first gain).
- Price precedence: typed (per window) -> KB NPC buy -> NiaMeowDB `item_market(id).usual` (off-thread).
- UI: equip picker (gendered twins merged; same name in 2 slots -> "(slot)"); default first scrollable worn item; slots stepper (cap slots or 30); stat chips; card per grade: success, all-pass %, expected, ladder "+k·gain: %", expected cost, per-point. `_pct`: >= 0.9995 "100%", (0, 0.001) "<0.1%", else 1 dp.
- `find_equips`: all words; exact, prefix, rest; >= 2 chars.

## 14. Item price (`prices`) + Market details
- Lookup: exact (folding hyphens/spaces) else shortest containing; untradeable Cash -> NX price only.
- `market.npc_prices` (item page after 2nd "---"): "NPC Sell-back N"; "Where to buy" blocks `NPC [cheapest] / where / price|- / mesos [/ "<LABEL> prices [grade +]"]`; sorted by price; label = source chip; rank = citizen grade; only `released(town)`; lines NPC buys, cheapest shop (+rank), also sold by (<= 3 unpriced).
- Safe to sell (`sitedata.sell_list`): Keep (quests n, recipes n; tooltip <= 10 each) vs safe; nothing if list missing.
- MesoWatch line: `for_item` off-thread; 60 s poll on `mesowatch.version()`; "usual P, N sold (world)" + last sale ago + "N sold over D days" + few-sales warning, or offline/none. `market_ago` < 60 min "N min" (min 1), < 24 h "N h", else "N days".
- Links: Full details, who drops, way to cheapest shop.
- MarketDetailsDialog 460 x <= 820 (`market_window_pos`): header counts; median price with p25–p75; few/per-set tags; tiles last sale, last-3 avg, sold over days; tabs where to buy (<= 8 shops by price, seller avatar `face_file`, seen ago, location, stock) and recent sales (<= 10); "Open MesoWatch" -> item_url.
- `mesowatch.py`: see 01-brain-data §15. Trend: last 7 days with sold >= 3, round((last-first)/first*100). `solid_price` = price if not low_data and not scrolled_only.
- NiaMeowDB (scroll FM prices only; cache 600 s, 40 items, 60 s offline; sequential): `https://meowdb.com/msclassic/api/item-market-summary?itemId=&condition=clean`, `…/item-price-history?itemId=&condition=clean&days=30`, `…/market-listings?itemId=`; `free_market()` `/api/market-listings/browse?q=&sort=price_low` (median of exact-name rows).

## 15. Sell or keep (`more`) — `sellkeep.py`, `inventory.py`
- Flow: hide window -> capture -> `inventory.read` off-thread -> `sellkeep.classify` -> off-thread MesoWatch `solid_price` for up to 30 sell/no_price tradeable keys -> `with_market` (FM > NPC -> kind "fm").
- `classify` order: needs (quest/recipe/wish); equip (REQ LEV / JOB; Mage->Magician): other_job, not_yet, wear (with NPC price); supply: Use / Potion|Food|Buff|Return Scroll, `is_potion` ("HP/MP Recovery" line), class ammo (Thief stars except Bandit/CB; Bowman arrows matching bow/xbow), Use pages "Shared item buff slot"/"Returns you to"; sell (sell-back from first match or same-name copy) / no_price; unknown.
- Sort ORDER [quest, recipe, wish, supply, wear, not_yet, other_job, fm, sell, no_price, unknown], then -max(fm, price), then slot. Total = Σ sell prices.

## 16. How to get there (`route`)
- Graph and Dijkstra: see 01-brain-data §8 (COST portal 1, npc 2, boat 3, taxi 4).
- UI: from (auto from live minimap LOCATION else c.map), to, Take cabs (`route_taxi`), swap; tags steps, taxi/boat, walk; notes walk alternative, fares unknown, ring; step cards (number, name, street · continent, `route_says` line, minimap with target dot (orange portal / green NPC) and live blue "you" dot + legend); unknown vs "not in game yet".

## 17. Cash Shop (`shop`) + Pets
- `osms.py` (see 01-brain-data §15). `place()`: Equipment by sub-slot; Coupon -> beauty misc/faces/hair; Pet ids 5000000–5009999 -> pets, else pet_care; Megaphone -> social; Weather -> effects; name contains "crate" -> special; else convenience. Rows sort (nx, name). Crate tiles from MesoWatch crate odds, 4 cols. Credit "OSMS Data Explorer by ohmi".
- Pets (`pets.json`): filters sold / easiest (fewest commands) / all; lines lifespan, hunger (of top), commands to Lv, fastest commands, NX price, trade; pet skill cards.

## 18. Golden numbers (port as Rust tests)
- test_play_tools: `acc_needed(24|19|14, 24, 14)` = [47, 56, 65]; `base_acc("Warrior", 30, dex 30, luk 4)` = 49; `acc_per_point(Warrior)` = 0.48; `hits_to_kill(100, 300, hp1000, lv30, pdef0)` = (10, 5.0); `landed(1e6)` = 99999; `landed(0.2, 50)` = 1; damage_range cases; SMITHING page Juno (2, 40, 1200, -226, 0.177, "Farm only"), Lv2 (115, 10, 2), Lv3 (3, 199, 15); `next_level(smithing, 1)` = (2, 50, 10); `current_row([[Level],[10],[11-12],[20]], 15)` = 2; FM median 2000; `_ITEM` regex cases.
- test_grind (TABLE {20:1000, 21:1200, 22:1500, 23:2000}): 10->60% at 21 over 30 min: exp 600, exp_h 1200, pct_h 100.0, to_level 1440; 20@90 -> 22@10: exp 1450; misread cases mesos 115,433 / 50,000 / 20,000+30,000, exp 100+1350; potions Red 30 @ 50 = 1500 net 1500; Blue 20 @ 220 net -4400; kills 60, kills_h 120; expected 60*15 with chance 0.5 or 50; record exp_h 1080.
- test_farm: fit boundaries; loot [(Cap,4,50),(Leather,4,10)] value 240, loot_h 480, kills 40; every/per_kill; back_at daily/weekly; sellkeep kinds; with_market fm 900.
- test_craftplan: wand expand raws, crafts 17, cost 8400, leftover Screw 2; x2 Screw 4; swap 60 hoofs; bag cases; Arcforge exp_needed 308+450-123-12, 104 crafts; blocked char:30; override 900; Smithing 38/3; craftable cost 8400 save 13600; real KB wand (7900, 17).
- test_scrollplan: binomial 7 @ 0.6 expected 4.2; destruction closed form; Lesser 15000 -> 105000, per_point 1500; Intermediate typed 20000 -> 140000/63; UI "2.8%", "+63 Max HP".
- test_gearadvice (rank/deltas, soon horizon, delta_text, droppers order); test_routes (spot (0.9375, 0.5) "right"; boat fare 300; hidden_spots); test_mesowatch (price 375, trend -22% over 3 days, 0.76 kept, item_url); test_tools_audit (`_num`); test_lootlog (358 / 327 …); test_plan (next_job; left 77770, kills 1111; island monsters); test_inventory (grid + match).

## 19. Quirks to decide (decisions recorded)
- Meso chance conventions differ (`grind.community_mesos` chance > 1 = percent, None throws -> no estimate; `kb.mesos_per_kill` chance percent, None = 100%). **Decision: keep both as-is (parity); unify later in a separate change.**
- `craftadvisor._key_of` "item/item/N" fallback bug. **Decision: fix in v2.**
- Scroll Planner uses NiaMeowDB while item page uses MesoWatch. **Decision: keep (parity).**
