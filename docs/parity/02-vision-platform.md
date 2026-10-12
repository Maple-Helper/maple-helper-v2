# Parity checklist 02: vision, overlays, voice, OS integration, packaging

Source of truth: `Maple-Helper/maple-helper` @ 010107f (v0.18.1). Paths relative to `maplehelper/` unless stated.

## 1. OS layer (`osapi.py` -> `winapi.py` / `macapi.py`)
Shared surface: `SCREEN_COORDS_ARE_PHYSICAL`, `activate_self`, `capture_game`, `find_game_window`, `float_over_fullscreen`, `focus_window`, `grab_screen`, `missing_permissions`, `prepare_process`, `seconds_since_self_activation`, `set_autostart`, `window_rect`, `open_url` (http/https only; anything else refused).

**Design rule (anti-cheat): nothing invasive.** No keyboard hooks, no key-state polling, no global hotkeys, no game-process access. The chat opens from tray, shortcut, second launch, or the mic button.

### Windows
- `prepare_process`: `SetCurrentProcessExplicitAppUserModelID("MapleHelper.App")`.
- `GAME_TITLES = ("MapleStory Classic","MapleStory","Classic World","MapleRoyals")`, case-insensitive contains after removing bidi marks `[\u200e\u200f\u202a-\u202e\u2066-\u2069]`; reject titles containing "maple helper".
- Never-game classes: Chrome_WidgetWin_0/1, MozillaWindowClass, MozillaDialogClass, CabinetWClass, ExploreWClass, ApplicationFrameWindow, Windows.UI.Core.CoreWindow, IEFrame, OperaWindowClass, Notepad, Notepad++, OpusApp, XLMAIN, PPTFrameClass, CASCADIA_HOSTING_WINDOW_CLASS, ConsoleWindowClass.
- `NOT_GAME_APPS` title regex: chrome, edge, firefox, opera, brave, vivaldi, discord, youtube, explorer, notepad, telegram, whatsapp, twitch, reddit, obs, vlc, acrobat, photos, paint, steam, spotify, powershell, cmd.exe, terminal, visual studio code.
- `find_game_window`: EnumWindows; visible, passes filters, not our process; minimized (IsIconic) separate list -> only minimized => `LAST_PROBLEM="minimized"`; sort exact title first, titles with " - ", " | ", " — " last; reset LAST_PROBLEM each call.
- `window_rect`: `DwmGetWindowAttribute(DWMWA_EXTENDED_FRAME_BOUNDS=9)` else GetWindowRect; physical px; None if a side <= 50.
- `covered(hwnd, rect)`: 3x3 probe grid at 0.15/0.5/0.85 with WindowFromPoint -> GetAncestor(GA_ROOT); not covering: no hit, game hwnd/process, our process, taskbar (Shell_TrayWnd/Shell_SecondaryTrayWnd), fully transparent layered windows (WS_EX_LAYERED + GetLayeredWindowAttributes LWA_ALPHA alpha 0). Exception -> treated as covered.
- `capture_game`: minimized -> None ("minimized"); covered -> None ("covered"); else `grab_jpeg(rect)`.
- `focus_window` = IsWindow + SetForegroundWindow; `activate_self` same on own hwnd. `float_over_fullscreen` no-op; `seconds_since_self_activation` = inf; `missing_permissions` = [].
- Autostart: `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` value `"Maple Helper"` = `"<exe>" --background`; skipped when not frozen; OSError -> False + log.
- `SCREEN_COORDS_ARE_PHYSICAL=True`.

### macOS
- Screen Recording: `CGPreflightScreenCaptureAccess` / `CGRequestScreenCaptureAccess(request=True)`; without it refuse capture, `LAST_PROBLEM="screen_permission"`.
- Microphone: `AVCaptureDevice.authorizationStatusForMediaType_("soun") in (1,2)` => denied.
- Window list: `CGWindowListCopyWindowInfo(OnScreenOnly|ExcludeDesktopElements, kCGNullWindowID)`, front to back.
- `pick_game_window`: skip own PID, `kCGWindowLayer != 0`, owners in NOT_GAME_OWNERS (safari, google chrome, chrome, firefox, microsoft edge, arc, brave browser, opera, vivaldi, orion, discord, finder, preview, textedit, notes, telegram, whatsapp, slack, messages, mail, terminal, iterm2, vlc, steam, code, spotify), titles matching browser/chat regex, "maple helper", sides <= 50. Match GAME_TITLES against title OR owner (titles empty without permission; CrossOver/Wine). Sort (not exact, has separator " - "/" | "/" — "/" – ", front order).
- `covering_window`: windows in front; ignore own PID, game PID, layers outside 0..19, alpha 0; overlap > 1% of game area => covered.
- Rect in points; `SCREEN_COORDS_ARE_PHYSICAL=False`.
- Focus: `NSRunningApplication…activateWithOptions_(1<<1)`; `activate_self` = `activateIgnoringOtherApps_(True)` + timestamp (reopen check < 1.5 s).
- `float_over_fullscreen(win)`: collection behavior |= CanJoinAllSpaces(1<<0) | FullScreenAuxiliary(1<<8).
- Autostart LaunchAgent `~/Library/LaunchAgents/com.maplehelper.app.plist` `{Label, ProgramArguments, RunAtLoad: true, ProcessType: "Interactive"}`; refused (False) when running from `/Volumes/` or `/AppTranslocation/`.
- Dock hidden via LSUIElement.

### Capture (`capture.py`)
- `grab_image(x,y,w,h)` RGB; physical px on Windows, points on macOS (Retina image larger).
- `content_box(img)`: luma `BLACK=12` + getbbox; bars only if each side >= 2% of dimension, sides differ <= max(4, 1%), content >= 50%; all-black -> None.
- `grab_jpeg(rect)`: sets `PLAY_AREA` (screen rect w/o bars), `LAST_FULL` (full-res trimmed image), `LAST_CURSOR` (Windows GetCursorPos relative to trimmed image, else None); thumbnail to `MAX_SIDE=1568`, JPEG q82.
- `detail_tiles(img, width=1200)`: only if longest side > 1568; vertical strips 40 px overlap, each thumbnailed to 1568, JPEG q85.
- `PROBLEM_TEXT = {"covered":"shot_game_covered","screen_permission":"perm_screen_body","minimized":"shot_game_minimized"}`.

## 2. Capture regions
- Minimap box `settings.minimap_region {x,y,w,h}` (capture coords) drawn with RegionPicker from the chat card's minimap button.
- Loot box: auto `lootlog.auto_box(play)` `AUTO_BOX=(0.78, 0.57, 1.0, 0.86)` of trimmed game picture (measured at 1368x800, 1922x1112, 2734x1568); `settings.loot_region` overrides.
- HUD strip: bottom `STRIP=0.06` of LAST_FULL.
- RegionPicker: frameless, on top, tool, translucent, union of all screens, cross cursor; dim alpha 89 (~35%), box cleared; orange 2 px border, "W × H" under box, hint at top-centre of cursor's screen; **no cancel** (Esc/right-click ignored, keyboard not grabbed); box < `MIN_SIDE=12` = misclick, stays open.
- `to_capture(rect, screens, scale)`: screen containing box centre; `x' = origin.x + (x-origin.x)*dpr` (same y), w,h * dpr, rounded; macOS no scaling. `from_capture` inverse (screen whose native extent contains centre) -> (rect, ratio).
- During picking chat hidden then re-shown; result saved as ints; scanner restarts; portal dots refresh.

## 3. OCR (RapidOCR -> v2: `ort` + same PP-OCR ONNX models)
- Models: rapidocr >= 3.9.2 default bundled ONNX det/rec(/cls) + yaml configs. **v2 ships the identical .onnx files and replicates det preprocessing (resize, normalize), DB post-processing (thresh/box_thresh/unclip from the same yaml), rec preprocessing and CTC greedy decode with the same char dict.**
- Params (`minimap._OCR_PARAMS`, shared): intra_op_num_threads=1, inter_op_num_threads=1, `Det.limit_type="max"`, `Det.limit_side_len=960` (only shrinks).
- Minimap Locator owns an engine (lazy, locked; load failure logged once, never retried). `hud._ocr()` global engine shared by HUD and loot (`hud._engine_lock`). Every call holds its lock.
- Output used: txts + 4-point boxes -> rows (text, y-centre, y-bottom/height[, x-centre]).
- Loot: `use_cls=False`; boxes shorter than `OCR_MIN_H=450` upscaled LANCZOS to 450 tall, coords scaled back. Minimap/HUD defaults.
- Pinned: `test_engine_is_one_thread_and_never_upscales`.

## 4. Minimap locator (`minimap.py`)
Map identity ALWAYS from OCR of the header text; KB picture `img/map/<id>.png` only aligns and places the dot.

### 4.1 Header -> map id
- `_plain` (collapse ws, lowercase); `_folded` (lowercase; 0->o, 1->l, |->l, 5->s; rn->m; strip non `[a-z0-9]`); `_ratio(a,b)` = max SequenceMatcher.ratio over plain and folded. (**v2: exact port of difflib.SequenceMatcher incl. autojunk.**)
- `_candidates_for`: exact plain in `graph.known`; else best fuzzy per distinct name, accept >= `_NAME_MIN=0.8` and ahead by >= `_NAME_MARGIN=0.07` (zero-margin ties -> none); shared names + street read -> keep `_street_ok` (exact plain, exact folded, ratio >= `_STREET_MIN=0.85`); street only narrows.
- `resolve_name`: single -> it; previous map if candidate; candidates reachable via edges from previous -> deterministic; picture scores if best >= 0.5 and ahead by `_TIE_MARGIN=0.05`; `_deterministic` (in graph.maps, town, most edges, id).
- `_header`: collapsed window first: a line with y-centre < `_TOP_BAND=0.12`*h parsed by `collapsed_title` ("Street : Map" ASCII `:` or full-width `：`, or known street + space; trailing "WORLD" stripped) -> collapsed (no dot). Else drop chrome (`is_chrome`: "minimap"/"world" sans spaces, anything in top 12%); if panel y0 >= 40 drop rows whose bottom is > 8 px below panel top. Street-anchored: first kept line `_street_ok` vs any KB street; map line(s) within 3.5 half-heights below, joining same-row fragments (±half/2). Else last line = map, previous = street. Guard: exactly one line, a map known, line matches known map's street but not name -> None.
- Header crop: after full read `_header_rows = bottom + _HEADER_PAD(70)`; later reads OCR `arr[:header_rows]`, fallback whole; `_text_bottom` kept for dot search.
- Text cache `(map_text, street, prev) -> mid` unless shared names.
- Stale panel recovery: non-fresh read finds no text -> redetect panel; if top moved, re-read.

### 4.2 Panel detection `_detect_panel`
- Flood fill on shrunk copy q = clamp(round(maxside/170), 1..4); seeds edges + top `_TITLE_SEED=4%`; 4-way where max channel step <= `_PANEL_STEP=16`; retry leaving right, left, bottom edge unseeded; < 2% grown -> whole box.
- Panel = biggest unfilled hole labelled at further ½ res, >= `_MIN_AREA=0.05`.
- `_snap_out`: edges move outward up to 40 px to first line where >= 70% px step >= 40.
- Edge strip: <= 8 passes removing near-white edges (min ch > 200 on > 50%).
- Background: median of px with saturation <= `_BG_SAT=36`; if its luma >= `_BG_LIGHT=120` and >= 15% darker, use median of dark px.
- `_shown_area`: from middle outward, first run of 4 frame lines ends visible map (80% flat within 12 of median, median max >= 80, bluish B >= max(R,G)-4, spread <= 70, differs from bg by > 24).
- Marker mask (excluded): red R>200,G<120,B<120; green G>150,R<100,B<100; blue B>140,R<120,B-R>60; yellow R>220,G>190,B<120; cyan G>150,B>150,R<110.

### 4.3 Alignment (masked NCC via FFT)
- `_masked_scores`: FFT sizes `_fast_len` (5-smooth), integral images for window sums, placements may hang off edge, reject where either side's variance <= n.
- Template: KB picture composited on black -> grey -> resized bilinear to scale*k + (1-alpha)*bg luma. (**v2: Pillow-equivalent resampler.**)
- `_ncc` (template inside panel, overlap >= `_OVERLAP_MIN=0.5`); `_ncc_over` (panel inside scaled picture; picture <= `_OV=2.5`x panel; sides >= `_MIN_SIDE=8`); score = ncc*(0.5+0.5*min(1,cover)).
- Scales: fit = min(3, W/aw, H/ah)*0.999, fit/1.12, fit*1.12 (within [0.5,3]); ladder x`_LADDER=1.08` from max(fit,0.5) to cap = min(3, 2.5W/aw, 2.5H/ah); coarse on 160 px panel; refine 7 scales ±12%, then ±4% in 1% steps; accept >= `_ALIGN_MIN=0.5`.
- Fast paths: recheck locked map at locked scale (>= `_RECHECK_MIN=0.4`); last placed scale (`_scale_prior`).
- `locate()` order: see-through check at locked scale (if lock is see-through); `_align` no sweep; see-through at prior scale; `_align` with sweep; redetect panel + `_align`; `_see_align`; else `Here(mid, None)`.
- See-through: features brightness (mean RGB) and greenness G-0.5(R+B), 0.5/0.5; NCC only over template alpha > 127 and valid panel; template covers >= 30% of drawn px; FFT cache (`_see_cache`, clear > 256); sweep 0.8–3.0 x1.05 on 160 px; top 4 by score + top 4 by margin refined at ½ (x0.97/0.985/1/1.015/1.03); best by margin checked full size (x0.99/1/1.01); accept score >= `_SEE_MIN=0.25` AND margin >= `_SEE_MARGIN=0.10` (margin vs best placement > `_SEE_APART=12` px away); sub-pixel parabolic fit.

### 4.4 Player dot `_dot`
- Search: matched rect ∩ `_shown`, below header bottom. Dot px R>=200, G>=190, B<=130; 8-connected blobs size 4..800, largest first.
- Veto: art "yellow" (R,G>=150, B<=150) at matched scale; blob overlapping art yellow >= 50% (shifts r = max(8, 0.06*maxside), FFT) is art unless blob >= `_DOT_BIG=20` px AND R+G brighter by >= `_BRIGHT_MIN=40`.
- Result: blob mean as fractions of scaled picture, clamped [0,1]. `Here(map, spot|None, view|None)`; `View(x,y,w,h,panel)` box px; `View.at(spot)`.

### 4.5 Fast follow (no OCR)
- State (mid, scale, x, y, bg, panel rect, box size, see, corr, shown) under lock. Box size change -> (mid, None).
- Grey follow: panel ½; NCC near last (±`_FOLLOW_NEAR=24` px) else whole; >= `_FOLLOW_MIN=0.35`; round to 0.5 px.
- See-through follow: `_track` (3 px grid of drawn px; cross ±12 + 5x5 block; climb <= 8 steps; parabolic; at reach limit -> None); fallback see-match at ⅓ with measured `corr`, then ½.
- Jitter: change <= 0.5 px keeps old. Race: a re-lock wins (`if self._follow is snap`).

### 4.6 Scanner (`ui/minimapscan.py`)
- `INTERVAL=0.5` s fixed (stored setting ignored; pinned by test). Work off the UI thread; overlapping ticks skipped.
- Follow timer `FOLLOW_IDLE_MS=150`; while moving (change within `FOLLOW_HOLD_S=1.0`) back to back. Follow only if current map has hidden portals or NPC guide target on this map.
- Map change needs `CONFIRM=2` consecutive reads. Known map never dropped for misses; grace max(3 s, 3*interval); then back off max(2 s, 2*interval) if known, max(5 s, 5*interval) + state "unknown" if not.
- Lost follow -> one immediate full read. A delivered read starts follow only if none live for that map. Distinct errors logged once (set cleared > 100), counted as miss. `set_kb` drops locator/graph.
- Shared `LOCATION`: here, state (""/"unknown"), follow, guide; events changed/status/followed/guided.

## 5. Overlays over the game
Common flags: frameless, always on top, tool window, never takes focus (WindowDoesNotAcceptFocus + ShowWithoutActivating + NoFocus), translucent. Take clicks, never keyboard.

| Window | Click-through | Excluded from capture | Placement / persistence |
|---|---|---|---|
| PortalDots | yes | **yes** `SetWindowDisplayAffinity(WDA_EXCLUDEFROMCAPTURE=0x11)` every show | exactly over minimap box; hidden when nothing to place |
| NpcOverlay | no | no (streamers see overlays) | `npc_overlay_geom`; default right of box +`GAP=8`; 320x440, min 220x260; edge-resizable |
| GuideWindow (NPC) | no | no | docked beside NPC list (left, or right if no room / would cover box), width 270 |
| MonsterOverlay | no | no | `monster_overlay_geom`; default beside where NPC window opens |
| GameToolbar | no | no | `game_toolbar_pos`, `game_toolbar_width`; default just above box; resizable sides only |
| Toolbar ⚙ panel | no (takes focus for Esc) | no | under gear |
| LootOverlay | no | no | fixed width 280+2*SHADOW; `loot_overlay_pos` default screen right -WIDTH-40, top +160; `loot_overlay_folded` |
| Toast | no | – | see 03-shell-ux |
| Chat | no | – | see 03-shell-ux |

- Portal dots: `DOT_PX=7`/DPR, fill (40,130,255), outline (10,20,45); spots from `Graph.hidden_spots(mid)` at `y - PORTAL_MARK_RISE(50)`, fractions via `mm=[w,h,xoff,yoff]`; NPC guide ring radius 7, green pen 2 over dark rim (8,48,24) width 4; positions only from `LOCATION.follow` for current map. No setting.
- Keep off minimap (`allowed_spot`/`drag_to`/`keep_off_minimap`): panel (rect minus SHADOW=12) fully on its screen and never intersecting the minimap box; smallest shift to one of 4 sides; no room -> stays; drags stop at box edge. Pinned by test_keep_off_minimap.
- Look: `_paint_glass` shadow, radius 16, opacity `npc_overlay_opacity` (clamp 0.3–1.0, default 0.85), gradient rim. Tints app / dark (24,24,27) / midnight (17,24,43) / forest (13,31,22) / wine (38,17,27) / slate (36,39,44). Accents app(orange) / green #2E9E5B / blue #3D8BD6 / purple #8265D6 / orange #E08A3C / pink #D65C8E / teal #2FA8A0 (each with text-on colour). Opacity steps 0.3, 0.5, 0.7, 0.85, 1.0. Settings `overlay_tint`, `overlay_accent` default "app".
- Game toolbar buttons: grip, **Chat** (accent-lit, first), checkable npc_overlay, monster_overlay, loot_overlay, **Tools** popup (prices, route, scrolls, gear, crafting, quests, farm, loot + "all"), ⚙, ✕. Flow-wrap rows; width clamped [widest button, screen]. ✕ sets `game_toolbar=False` (tray item re-enables). Tool tab names via `t(f"tool_{page}")`.
- NPC overlay: `npcs_on(graph, mid)` unique NPCs by name placed with `picture_spot`; then NPCs inside buildings reached through door portals named like `in\d` that lead back, guided to the door. Click row -> `LOCATION.guide=(mid, spot, key)`. Direction text in map units: left/right from 60, up/down from 90, else "here". Guide window shows `mapview.route_picture`. Folds to header when no NPCs. Geometry saved 400 ms debounce.
- Monster overlay: `gamelookup.map_monsters(kb, mid)` rows level, HP, EXP, spawn count, most spawns first. Read-only.
- Step aside for capture: manual -> chat opacity 0, capture 120 ms later. Grind auto read (every minute): `windows_over(rect)` finds our windows over the game; chat see-through, others hidden, wait 300 ms, restore with show-without-activate. Modal dialogs/popups open -> skip that minute. `_take_front`: chat shown at opacity 0 and activated before screenshot (Windows foreground lock).

## 6. Loot log (`lootlog.py`, `ui/lootscan.py`, `lootdb.py`, `ui/lootoverlay.py`)
- Parse (whitespace collapsed): EXP = squashed lowercase has "exp" and one of "rece","eived","ceiv","recel". Mesos `me[s5][o0g]s|mesos|mes\b|mes$` or ("gained"/"have" + "mes"). Amount `_PAREN = [(\[{]\s*(?P<sign>[+t4]?)\s*(?P<num>\d[\d,.]*)\s*[)\]}]` ("+" sure; "t"/"4" misread plus not sure; strip , .). Item `_ITEM = ^(?P<name>.+?)\s*[x×X]\s*(?P<n>[\dlIi|]{1,4})\s*\.?\s*[eE]\w*[rn]\w*\b` or `_EARNED` `\s*\.?\s*ea[rm]n?ed\b.*$`; qty `lIi|` -> 1; `_QTY` strips trailing count; trim " .,:;-"; sure = matched + has name.
- `match`: items: different sure qty -> -1; missing name -> 0; ratio >= 0.75 -> 1; >= 0.5 or not both sure -> 0; else -1. Amounts: equal 1; same length >= 2 one digit differs 0; one longer by leading digit and not both sure 0; else -1.
- Tracker: pitch = min y gap smoothed 0.7/0.3 within 30% (default 1.4*median height); bottom y smoothed 0.8/0.2 (jumps lower if line > 0.5 pitch below); row = round((bottom-y)/pitch) cap `MAX_ROWS=10`; shift k in 0..top maximizing Σ match - 0.5 per unexplained line, prefer smaller k; first read baseline only; new events: r < k or sure; expire `UNSEEN_S=1.6` unseen / `LIFETIME_S=12` total; values by vote (most common, earliest tie, sure preferred); item names exact else `get_close_matches` cutoff `ITEM_FUZZY=0.8`; `item_names` lower->key and "\0"+lower->display.
- Log: `MAX_EVENTS=5000` (older folded into base); per-hour >= 60 s; value = Σ n*NPC price + mesos; store `DATA_DIR/loot.json` `{active:{cid:log}, recent:{cid:[<=20]}}`, one running log; character delete stops/removes.
- Scanner: `READ_INTERVAL=0.5` s; save every 5 s; event to DB when older than `SETTLE_S=12+3` or at end; `GameArea` cached hwnd, bars re-measured on resize or every 30 s; states covered (Windows `covered` on box -> tracker reset), no_game, no_reader; box size change -> reset.
- Overlay: title elapsed; folded shows what log made; stats exp, mesos, items (count+value) with per-hour; feed last 4; Start / Stop + Reset; fold (–) and ✕; log start turns overlay on; refresh 5 s; ≥1M "x.xxM", ≥100K "x.xK".

## 7. HUD reader (`hud.py`)
- Bottom 6% of LAST_FULL OCR'd. `HP\W*(\d+)\s*/\s*(\d+)`, `MP\W*…`, `EXP\W*(\d+)\s*\[?\s*(\d+(?:[.,]\d+)?)\s*%`. No EXP and no HP -> None.
- Level: need = exp*100/pct; exactly one level within 2% wins; pct >= 1; stylized digits count only when exp <= table[digits].
- Job/name: word lines left of HP x matching `[A-Za-z][A-Za-z0-9 ]{1,15}`, excluding cash shop, menu, short-cut, shortcut, lv; job = first `canonical_job` match; name = next line within 120 px horizontally.
- `local_grind_read`: >= 8 inventory slots visible -> defer to AI; else HUD; level+pct -> profile update + map from LOCATION.

## 8. Portrait (`portrait.py`, `overlay.crop_portrait`)
- Name tags: luma = 0.3R+0.59G+0.11B+1; ratio row y+1 / y; top edge runs 30..420 px ratio 0.22–0.65; height 9..60, bottom where >= 55% ratio 1.5–4.5; aspect 2.2..14; white letters (min >= `WHITE=200`, spread < 30) cover 4–50%; >= 3 tags drop height off median > max(3, 20%).
- Whose tag: draw name in Arial (`C:/Windows/Fonts/arial.ttf`, macOS Supplemental/Arial, /Library/Fonts, tahoma) 48 pt; plate letters luma > bg + 0.5*(max-bg) cropped; NCC after stretching >= `DRAWN_MIN=0.40` and ahead by >= 0.12; learned letters saved per name `avatars/names/<name>.png`; `find_learned` FFT NCC over whiteness map (min >= 170, spread < 45) >= `LEARNED_MIN=0.55`, confirm plate `_plate_around`, <= 4 tries. (**v2: rasterize Arial with `ab_glyph`; parity checked on fixtures, thresholds may need re-measurement.**)
- Crop: side 5.4*tag h centred, from tag_y - side to tag_y + 0.2*th; reject past left/top/right.
- No name: AI rough box (search ±3w, ±2.5h, nearest tag to feet); with name width alone never counts.
- Sprite check `sprite_mask`: flood from edges stopped by dark (luma < 70); remove pockets with bg quantized colour (/24); keep largest; `_without_scenery` removes leafy dark-green chunks at edge; accept 0.12–0.8 coverage else `_soft_outline_mask` (outline dilate 2, core peel 3; >= 70% height, 0.2–0.6 coverage); `mask_reaches_feet` (bottom third).
- Output 128x128 PNG (NEAREST up, LANCZOS down), keep bg; failure keeps old.

## 9. Inventory reader (`inventory.py`)
Slot beige (224,222,212) ±14; icons 24x24 on beige, shadow removed, 3x3 blur, ±1 px shift refine on 40 nearest; stack count box masked. `ICON_DIFF=40`, `TWIN=3.0`, rival < d*1.08+0.5 -> unknown, `UNKNOWN=25`, `UNKNOWN_ALIKE=28`, `REFINE=40`, `SMALL_SLOT=70` (read doubled), `CLEAR=12`, `COVERED=0.4`. Index of KB item icons warmed in background. Shared pictures -> "ambiguous", ask to hover; slot under `LAST_CURSOR` = hovered. 18 tests.

## 10. Voice (`voice.py`) -> v2: whisper.cpp (`whisper-rs`) + Silero VAD (`ort`) + `cpal`
- v1 model `ivrit-ai/whisper-large-v3-turbo-ct2` rev `72ad623a37947395efcc3933132353790e5a12f5`, ~1.62 GB, in `DATA_DIR/models/...`; consent before any download (`need_download(size)`); progress from bytes on disk every 500 ms capped 99; cancel; loads offline once present.
- **v2 model:** GGML conversion of the same ivrit-ai weights (q5_0 or q8_0, chosen by WER on the eval clip set), hosted as a release asset `whisper-ivrit-large-v3-turbo-<quant>.bin` with SHA-256; same consent/progress/cancel UX; stored under `DATA_DIR/models/ggml/`. GPU: Metal (macOS), CUDA or Vulkan build on Windows (decided in M7 spike); CPU fallback.
- Recording: 16 kHz mono f32; click mic to toggle (no hotkey); < `MIN_SECONDS=0.4` dropped; auto-stop `MAX_SECONDS=60`; macOS mic-denied check first, all-zero audio = permission problem.
- Transcribe: beam 5, VAD on, language he/en or auto (`voice_language` app/auto), `initial_prompt` he: "MapleStory Classic, Henesys, Ellinia, Perion, Kerning City, Red Snail, Orange Mushroom, לבל, ג'וב"; en: same without Hebrew words.
- Mics: default host API inputs (exclude "Sound Mapper"), full WASAPI names; saved by name; None = default.
- Preload 6 s after start only if ready and used in last `RECENT_DAYS=14` (`voice_last_used`).
- States listening, transcribing, loading, downloading, idle. Errors prefixed `mic:`, `download:`, `nospace:`.
- After text: `kb.resolve_names`, into input; sent if `voice_send_immediately`.

## 11. App wiring / OS integration
- Single instance; second launch shows chat unless `--background`. macOS reopen: app becomes active > 1.5 s after own activation with nothing visible -> show chat.
- Args: `--background`, `--updated`, `--selftest <file> [--require-kb]`.
- Windows mutexes `MapleHelperRunning` (held) / `MapleHelperSetup` (wait up to 900 s, note box after 8 s) — **v2: kept only in the bridge installer; NSIS handles running-app detection afterwards.**
- Permissions check 1.5 s after start (macOS Screen Recording toast 20 s).
- Broken install report `logs/startup-error.log` + message box offering reinstall.
- Report a problem: zip to Desktop, reveal (`explorer /select,` / `open -R`).
- `OPENBLAS_NUM_THREADS=1` (v1 only, not needed in v2).
- App updates: see 03-shell-ux §12.

## 12. Packaging and CI (v1, for reference; v2 replaces with Tauri — see design spec)
- PyInstaller onedir; Inno Setup AppId `{4B526220-22E4-45D2-88B7-C62A0B7385E4}`, per-user `{localappdata}\Programs\Maple Helper`, `SetupMutex=MapleHelperSetup`, en+he, lzma2/ultra, `CloseApplications=force`, `KbNeedsInstall` (string compare meta.json versions), optional desktop icon (never on silent), run after install (interactive "Open now" or `{param:LAUNCHARGS|--background}`), uninstall removes `{app}\_internal` + Run value, uninstall TaskDialog asks to delete user data (keep default) incl. Credential Manager `MapleHelper`, `anthropic_api_key@MapleHelper`, `openai_…`, `gemini_…`, `xai_…`; silent setup waits 30 s for `MapleHelperRunning` then `taskkill /F /IM "Maple Helper.exe"`.
- macOS: ad-hoc codesign, DMG `MapleHelper-macOS.dmg` (UDZO, /Applications link), arm64 only, not notarized, `LSUIElement`, `LSMinimumSystemVersion=14.0`, bundle id `com.maplehelper.app`, `NSMicrophoneUsageDescription`.
- Release assets players depend on: `MapleHelper-Setup.exe`, `SHA256SUMS.txt`, `kb.zip`, `kb-manifest.json`, `MapleHelper-<v>-portable.zip`, `MapleHelper-macOS.dmg`.
- Selftest checks imports, assets (`brand/app.ico`, `icon-64.png`, `wordmark.png`, `fonts/Rubik-Variable.ttf`), keyring backend, VAD, font, KB count (`--require-kb`), `find_game_window`, provider CLIs; prints "SELFTEST OK".

## 13. v1 tests pinning this slice
Live fixtures in `tests/fixtures/`: minimap_*_live.png (14), loot_*_live.png + loot_reads_live.json + loot_reads_small_live.json, hud_bar_live.png, classic_status_bar.png, classic_crowd.png, classic_letters_kalimero.png, classic_shop_label.png, classic_sprite_bricks.png, classic_world_tags.png, portrait_bush.png, portrait_scenery_live.png, inventory_*_tab.png.
Tests: test_minimap 33, test_minimapscan 17, test_lootlog 17, test_lootdb 7, test_portrait 17, test_hud 6, test_capture 7, test_capture_ui 4, test_winapi 10, test_macapi 16, test_regionpick 7, test_portaldots 10, test_npcoverlay 29, test_monsteroverlay 8, test_gametoolbar 31, test_keep_off_minimap 6, test_voice 19, test_inventory 18, test_updater 36, test_packaging 16, test_selftest 3.
