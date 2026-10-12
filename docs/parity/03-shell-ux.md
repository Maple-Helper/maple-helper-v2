# Parity checklist 03: app shell, chat, dialogs, theme, i18n, updates, telemetry

Source of truth: `Maple-Helper/maple-helper` @ 010107f (v0.18.1). Paths relative to `maplehelper/`.

## 0. Constants
- Version source `__init__.__version__`; `APP_NAME = "Maple Helper"`.
- Assets: `brand/` (app.ico, icon-256.png, icon-64.png, icon-source.png, mascot.png, wordmark.png), `fonts/` (Rubik-Variable.ttf, OFL.txt), `notes/whatsnew.json` (36 versions 0.3.0..0.18.1, `[{version, he:[...], en:[...]}]`), `fallback/*.png`, `guides/{en,he,img}`, `items/he.json`, `news/he.json`, `cashshop/he.json`, `pet_skills/he.json`, `official/facts.json`, `quest_tasks/he.json`, `skill_changes/he.json`, `skills/`.
- Logs: `logs/maplehelper.log` (rotating 1 MB x 2), `logs/update-YYYYmmdd-HHMMSS.log` (keep 2), `logs/startup-error.log`.

## 1. Settings (`store.DEFAULT_SETTINGS`) — `settings.json` format must be read AND written compatibly
| key | default | notes |
|---|---|---|
| language | None | "he"/"en"; consumers fall back to "he" |
| appearance | "light" | dark/light |
| font_size | 14 | 13/14/16 |
| answer_length | "short" | short/detailed |
| window | None | {x,y,w,h} chat |
| map_window_pos, item_window_pos, market_window_pos | None | {x,y} |
| start_with_windows | False | at login on macOS |
| voice_send_immediately | True | |
| microphone | None | device name |
| voice_language | "app" | app/auto |
| voice_last_used | None | epoch s |
| provider | "claude" | claude/codex/gemini/grok |
| model | "sonnet" | |
| codex_model / grok_model / gemini_model | None | |
| last_model | {} | provider->model |
| api_key_fallback | {} | provider->bool (legacy bool = Claude) |
| onboarding_done | False | |
| shot_hints_seen | [] | |
| tour_done | False | |
| pins | {} | cid->[{q,a,t}] max 12 |
| tips_dismissed | {} | cid->{kind: level} |
| usage | None | |
| saver_mode | False | |
| usage_warned | 0 | [resets, level] |
| wishlist | {} | cid->[keys] |
| farm_target | {} | cid->key |
| seen_version | "" | |
| news_read | [] | last 300 |
| last_session | None | |
| instant_answers | True | |
| install_id | "" | uuid4 hex, lazily created |
| presence_sent | {} | {ever, day, month} |
| route_taxi, grind_auto | True | |
| minimap_region, loot_region | None | {x,y,w,h} |
| npc_overlay, monster_overlay, loot_overlay | False | |
| npc_overlay_geom, monster_overlay_geom | None | |
| npc_overlay_opacity | 0.85 | |
| game_toolbar | True | |
| game_toolbar_pos, game_toolbar_width | None | |
| loot_overlay_pos | None | |
| loot_overlay_folded | False | |
| craft_exp_table | None | |
| overlay_tint, overlay_accent | "app" | |

API keys: keyring service `MapleHelper`, users `anthropic_api_key`, `openai_api_key`, `gemini_api_key`, `xai_api_key`. Gemini sign-in: service `gemini`, user `antigravity`.

## 2. Lifecycle
- Entry: wait for setup (v1 only); startup exception -> `logs/startup-error.log` + broken-install box (damaged install -> offer reinstall from cached installer else latest; else "couldn't start" with issues URL); macOS `osascript display alert`.
- `--selftest` path.
- AppUserModelID `MapleHelper.App`; single instance (second non-background launch -> show chat, exit 0).
- Init: Settings, Profiles, KB (fallback to bundled if unreadable), async table build, fonts, icon, no quit on last window closed.
- `start()` order:
  1. single-instance listener (during onboarding raises active modal; else `show_chat()`).
  2. macOS reopen rule (active > 1.5 s after self-activation, no visible windows -> show chat).
  3. onboarding loop if not done (RESTART on language change; cancel quits); else character-only onboarding if no active character.
  4. telemetry init + context + crash hook; presence start; `app_started`.
  5. brain, AI settings, chat overlay.
  6. GrindRunner, MinimapScanner, PortalDots, NpcOverlay, MonsterOverlay, GameToolbar, LootScanner/LootOverlay.
  7. voice controller, tray, autostart apply.
  8. timers: 4 s KB+app update check; 6 s voice preload; 8 s remove old installers; 9 s remove stray screenshots (> 1 h: `%TEMP%/maplehelper-shot-*.jpg`, `maplehelper-grok-*.txt`, Gemini/Grok `run-*`, Grok sessions); 10 s inventory warm; KB every 60 min; app every 15 min.
  9. live data: MesoWatch 15 min, OSMS hourly.
  10. `--background` -> toast (`app_tagline`/`ob_done_hint`); else chat opens.
  11. 1.5 s later `check_permissions` (macOS).
  12. `announce_whats_new`; shutdown hook.
- `announce_whats_new`: `seen = seen_version or ("" if fresh else "0.3.0")`; set seen_version = current; fresh install -> nothing; changed -> `app_updated(from_version)`; notes since seen -> chat notice `whats_new_notice` + action `whats_new_show`; `--updated` -> What's new dialog after 900 ms (or after tour if tour not done).
- `open_window(kind)`: one per kind (`settings`, `whats_new`, `patch_notes`, `news`, `tools`, `guides`, `map`, `item`, `market`, `history:<cid>`, `wishlist:<cid>`); reopen raises; non-modal, deleted on close, always on top; `window_opened(window=kind)`.
- Language/appearance/font change: restyle all, re-apply language to overlays, reopen non-Settings windows on same page/guide/character, restyle live toasts; provider-only change re-applies language; tray rebuilt on every save.
- KB reload: windows other than Settings, What's new and the active/hidden one reopen on new KB.
- Idle summary: 30 min after chat close (single-shot) if still hidden -> summaries per character.
- `shutdown()`: `app_closed(minutes)`, telemetry flush (2 s), stop presence; stop grind/minimap/loot, hide loot overlay, save session summary, cancel brain; wait answer/sync threads (2 s each); brain shutdown; stop login; run pending installer silently unless setup running or OS shutting down.
- Report a problem: toast `report_preparing`; provider status on worker; zip to Desktop; reveal; 12 s toast whose click opens `https://github.com/Maple-Helper/maple-helper/issues` (**v2: maple-helper-v2/issues**).
- Delete character: confirm; remove from pins, wishlist, farm_target, tips_dismissed; none left -> add-character onboarding; feed cleared + "switched to" line.

## 3. Tray
- Tooltip "Maple Helper · " + `app_tagline`. Menu frameless, translucent, app-styled, RTL in Hebrew.
- Items exactly: "Maple Helper" (disabled header); separator; [macOS/portable when update announced: `update_available` "New version available: {v}" -> release page, right under header]; `tray_open` "Open chat"/"פתיחת הצ'אט" (never closes an open chat); `tray_settings` "Settings"; `tray_toolbar` "In-game toolbar" (checkable, synced from `game_toolbar` on open); [pending installer: `update_now_tray` "Update to {version}"]; separator; `tray_quit` "Quit"/"יציאה".
- Left click toggles chat on Windows; on macOS click opens menu.
- Rebuilt after Settings save and when an update becomes ready.

## 4. Keyboard
- **No global hotkeys** (anti-cheat).
- Chat: Esc ignored; F5 = inventory check (`sell_q` with screenshot, detail=True); input Enter sends, Shift+Enter newline; tab order by position, mirrored in RTL; paste strips formatting and trailing newlines.
- Dialogs: Esc closes except onboarding (first run) and Settings; Enter clicks the dialog's Enter button (Onboarding: Next) or a Tabbed button; no auto-default buttons.
- Tour: Esc ends; Enter next (or Tabbed Skip/Back); Left/Right step (mirrored in RTL); Tab trapped in card.
- Guides/News: Esc in article back to list; Enter in guide search opens first result.
- Cards, profile card, news strip, tip strip, history cards, guide rows open/tag on Enter/Space; glossary "?" on Enter/Space; stepper Up/Down; focus ring 2 px orange only for keyboard focus.

## 5. Chat window
- Frameless, always on top, normal window on Windows (Alt+Tab, Discord/OBS capture), tool window on macOS joining all Spaces + full-screen auxiliary. Title "Maple Helper".
- Painted glass: `RADIUS=22`, `SHADOW=12` px margin, opaque tint, sheen gradient top 170 px, rim gradient, 6-layer drop shadow offset +3 px.
- Resize from any edge (shadow + 8 px zone) with system resize; drag via title bar; geometry saved on release; resize saves debounced 300 ms.
- Default geometry w = min(600, avail-48), h = min(640, avail-80), top-right of game's screen (x = right-w-24, y = top+60); saved geometry clamped to monitor it mostly overlaps and moved inside the game's play area away from black bars (inset 8).
- Open anim: opacity 0->1 220 ms OutCubic + geometry 97%->100% 240 ms. Close: opacity ->0 150 ms then hide. Feed rows fade in 180 ms.
- `_take_front` before screenshot; close/minimize give focus back to game; minimize to taskbar (macOS: closes).
- Layout top to bottom:
  1. Header: 22 px logo, "Maple Helper", "BETA" badge (tooltip `beta_tip`), saver badge ("🍃 " + `saver_on_badge`, green); icon buttons search/history, news (orange while unread), guides, tracked items (star), play tools (game controller), settings; separator, minimize, close (orange, tooltip `close_chat`). Narrow steps: buttons 26 then 24 px, badge -> leaf only, BETA -> "β", then hidden.
  2. Update bar states available, ready, downloading (progress 0–1000), installing (busy), failed (`update_retry`); Primary "Update now"/"Try again" + ✕ (hide until restart).
  3. News strip: newest unread Global item from last 14 days (>= 2026-10-02): "News from {date} · and N more unread", title, summary tooltip; click opens News; ✕ marks read.
  4. Profile card: 46 px avatar (portrait else job/class picture), name, "Lv. N · Job" (he "רמה N"), "In {map}", EXP bar "67% · N EXP left · ~K mobs", status line; buttons minimap (pick region), refresh ⟳ (spinner frames 260 ms; sync_profile), "What now?" chip (fresh screenshot). Click -> character menu: other characters as cards with "Choose character", Edit, Delete, separator, Add, Share (copies card image). Chat behind blurred while menu open. Busy -> disabled rows + `menu_busy_note`. No character -> NoticeCard `no_char_chat` + Add.
  5. Tip strip (one plan tip; ✕ hides until next level); Pins bar "Pinned (n) ▾" (max 40% height; ✕ unpins).
  6. Feed: bottom-anchored, max 80 rows, follows unless user scrolled, long answers anchored to first line, "↓ New answer" pill when scrolled up, gutter mirrored in RTL, streaming redraw <= every 90 ms. Rows: user bubble (orange gradient, trailing side, <= 78% width), assistant bubble (fill1, max 680 px, markdown->HTML, accent links, glossary "?" up to 4, selectable), SystemLine, NoticeCard (info icon, 1–2 links, ✕), choice rows (chips, > 2 wrap), EntityCard, TileGrid (2 cols), DropGroupCard, SessionCard ("Last session · N min", per-character "Continue the chat"). Waiting bubble "Thinking" + 3 dots (400 ms), "· Ns" after 5 s, stage lines "Searching the database…"/"Taking longer than usual…". Copy + pin buttons (inline when one line < 160 chars). Errors red tint + warning icon, "Try again" (once; same question, tags, context, screenshot) and "Open Settings" for not_installed/not_logged_in/cli_outdated/no_credit (+usage_limit in key mode); question put back in input.
  7. Focus bar "asking about" + up to 5 tag chips (icon, name elided 140 px with "MapleStory Classic" removed, ✕); horizontal scroll; ✕ clears all; cleared after send.
  8. Screenshot hint states `shot_hint_no_game`/`_ready`/`_used` ("Retake it" link) + capture-problem keys; full text first time per state (`shot_hints_seen`), then `_short`.
  9. Voice chip "● Listening…" (red).
  10. Input capsule 42 px growing to 4 lines: camera (retake), field (placeholder `input_placeholder`/`_short`/`_tagged`, elided, alignment follows text direction), clear (trash; 5 s Undo; asks first while answering), mic (red while active; capsule border red), Send/Stop.
  11. Footer: scope note (`scope_note` + date + tooltip), server dot (8 px circle: green #34C759 up, orange maintenance/issues, grey otherwise; click `https://meowdb.com/msclassic/server-status`), version.
- Asking: busy -> `busy_wait`/`busy_reading` once; instant answers (no tags, `instant_answers`, not app text) "⚡ Instant answer from the database" + source chips + "Ask <Provider> anyway"; screenshot only with first question; history "[about X] q" when tagged; question removed from history on failure/stop; profile updates: lower level / other class / demotion asks (update / add as new / cancel); other character's name in screenshot offers switch/add (never silent rename); usage warnings notice with "Turn on saver mode"; profile sync timeout 60 s; server status polled every 5 min only while chat open (backoff 60 min when unreachable, >= 60 s between checks); notice when maintenance starts/ends.

## 6. First-run tour
- 700 ms after first chat show if `tour_done` false; dim overlay alpha 150 with rounded hole (pad 5, radius 12) + 2 px orange ring; card max 420 px below target (above if no room); clicks swallowed.
- Steps (hidden ones skipped): welcome, profile_card, profile_card.refresh, profile_card.now_btn, input, recapture, mic, history, news, guides, wish, tools, settings, min, close, done. Counter "i / N-2". Skip (link), Back (secondary), Next / "Let's go" / "Done". End -> `tour_done=True`, `tour_ended`. Replay from Settings ("Take the tour"; asks if unsaved changes).

## 7. Onboarding
- 600x680 glass dialog, on top, Esc disabled first run; ✕ asks "Quit setup?" (focus "Continue setup"); Quit exits app. Initial language = setting or system (he/iw -> he, else en). Nav: Back, "Step n of total", Next/Finish ("Done, let's play!"/"Add character"/"Save").
- Step 1 Language: wordmark 260 px, tagline both languages, buttons "עברית"/"English" (56 px); change restarts dialog on step 2 keeping name/provider/key.
- Step 2 Connect AI: segmented provider (Claude, ChatGPT, Gemini, Grok) with plan tooltips, `ob_plans_overview`; account status (Checking…, connected + email, logged out, not installed, outdated, offline); Install/Update (primary), Sign in, "Check again"; login polls 3 s for 180 s; dialog drops always-on-top during sign-in; Gemini code field; InstallPanel (indeterminate bar, last installer line polled 400 ms, reason on failure); "Or: use an API key" (password field, "Check key" on worker, hint; saved to keyring, key mode on); `ob_no_ai_note` + "Continue without an AI for now" + "Report a problem". Next enabled only when connected.
- Step 3 Character: name (max 24, unique case-insensitive); class tiles 3-col (Beginner, Warrior, Magician, Bowman, Thief + icons; he labels "ביגינר / לוחם / קוסם / קשת / גנב" over English); level stepper 1–250 (min = 1st job level for non-Beginners); job fixed label or dropdown "Pick a job"; next-advancement hint; info note.
- Step 4 Done: mascot 220 px, `ob_done_hint`, start-with-Windows switch, borderless tip, privacy line (names provider or `ob_privacy_none`), disclaimer, unofficial note.
- Finish: add profile, set `start_with_windows`, `onboarding_done=True`. Same dialog for Add character (character step only, Esc cancels) and Edit (prefilled).

## 8. Settings
- 500x720, Esc disabled; ✕ with unsaved changes -> "Save your changes?" (Save / Don't save; closing keeps Settings). One Save (primary, min width 180) -> hide, `changed`, `account_changed` if provider/model changed.
- Display: Appearance (Dark|Light), Font size (A/A/A = 13/14/16), Language.
- Audio: Send right after speaking; Microphone ("System default" + devices, "Microphone (" wrapper stripped); Microphone check (2 s; RMS < 0.03 too quiet; < 0 failed); Speech language (App language | Auto-detect).
- Answers: length (Short|Detailed); Instant answers (hint per provider).
- Minimap: In-game toolbar switch.
- AI account: provider picker (applies on Save), Model (Claude fixed list, "(recommended)" on Sonnet; others async; hint last + saver model), account line, timeout hint, code row, InstallPanel; Install/Update, Switch account (confirm, sign out, sign in), Sign out (red, confirm), Check again, API key entry; login polls 3 s up to 60 ticks.
- Plan usage / Saver: usage meter (connected + provider reports usage), note, Saver switch.
- Privacy & system: "Start when the computer starts" (macOS "Start at login").
- Data: Update database ("Checking…"), What's new in the database, What's new in Maple Helper, Take the tour, Report a problem, Open an issue on GitHub, Clear history (red, confirm; active character).
- Credits/unofficial/disclaimer footer.

## 9. Secondary windows (glass, frameless, on top, own title bar + orange ✕, draggable)
- Confirm 420x230 (initial focus on safe answer).
- WhatsNew 480x560 (from Settings: 6 most recent).
- PatchNotes 520x680: Changes | News tabs; "N changes affect you" first (class/level gear, monsters in range, tracked items), then by date added/changed/updated/removed; 12 cards then "Show more" +40; 8 built before show; News tab marks read.
- News 520x680: Global then China/Taiwan 30 each (+ MeowDB link); cards New/Official/Community chips, 88x50 thumb, date, summary, "Read the article" (in-app, Back, Esc), source link.
- Guides 560x760: library (search + chips for_you/classes/leveling/mechanics/general, rows with 88x46 cover, hover preview up to 480 px); reader (Back, title, meta, "Ask about this guide", web link, stale-translation note; pictures zoom 3x on hover; wide tables shrink font then padding).
- ItemDetails 480x720: right of chat (left if no room, gap 8), then `item_window_pos`; card, description, stats, Needed By, droppers with Ask, Meow Notes.
- Wishlist "Tracked items" 500x640: item search to add; per item card, "Dropped by" + source chip, note, "Farm" link, dropper rows (fit chip, votes, map, route link, Ask); rebuilds on change.
- History 540x720: search 150 ms debounce (all words in q or a), day headers (Today/Yesterday/date), 80 per page + Show more, expandable cards with "Continue in chat" and "Pin".
- Toasts: no-focus, shown without activating; 360 px + shadow, radius 18, 4 px orange accent bar, 40 px icon, "Maple Helper", title, body, ✕; bottom-right of the monitor in use (chat's, else game's, else primary), margin 16, stack upward, oldest dismissed when out of room; 220 ms fade + 12 px slide; default 5 s; click runs action + dismisses.
- Glossary popup: no-focus tooltip window, max width 320, orange 1 px border, radius 10; hides after 7 s; beside mouse or under label (keyboard). "?" badge orange 15 px circle.

## 10. Theme tokens
- Brand: ORANGE #FF9533, ORANGE_DEEP #F07A12, CREAM #FFF4E6, RADIUS 22, ORANGE_TEXT_LIGHT #C9620A, ON_ORANGE #FFFFFF. Good text #2E9E5B (light)/#4CD07D (dark); warn #C9620A/#FFB066; changed #0A6CD6/#64B5FF.
- Dark: glass (28,28,30); sheen 14; rim_top 60; rim 22; text #F5F5F7; muted rgba(235,235,245,.64); faint .40; fill1/2/3 white .08/.12/.20; pressed .28; stroke .14; hair .07; scroll .25.
- Light: glass (242,242,247); sheen 0; rim_top 40; rim 30; text #1D1D1F; muted rgba(60,60,67,.66); faint .42; fill1/2 #FFF; fill3 #E5E5EA; pressed rgba(230,230,235,.95); stroke black .08; hair .05; scroll .25.
- High contrast (Windows SPI_GETHIGHCONTRAST): appearance from window background luminance; muted -> full text, faint -> muted, stroke/hair alpha .60/.35.
- Font Rubik (bundled), fallback Segoe UI / Helvetica Neue. Base size = setting; derived s-5..s+10.
- Components: Send/Primary gradient #FFA24A->#F07A12 (hover #FFB066->#F58A2A); Danger #FF3B30; Switch green #34C759 46x28; Segmented #E3E3E8 light / rgba(118,118,128,.24) dark, checked white / #636366; menus/tooltips #2C2C2E dark / #FFF, item hover orange; scrollbars 6 px; dialog bg #1C1C1E / #F2F2F7.

## 11. i18n and bidi
- `i18n.STRINGS`: 1,554 keys (he+en). Provider variants `<key>_<provider>` (81) via `t.p(key, provider)`; `_one` singular (57) when n == 1; `_mac` variants for voice_mic_failed, copied, ob_done_hint, ob_autostart, close_chat, tour_min_body, tour_close_body, tour_done_body. Lookup: language -> English -> key; `.format(**kw)`.
- RTL when `he`: whole app, each window, tray menu, native context menus.
- Hebrew strings: "Maple Helper", "Claude Code" and English runs <= 3 words use NBSP; short English placeholder values too.
- `bidi.py` (v2 port must produce equivalent rendering in Chromium/WebKit; tests re-expressed as DOM/segment assertions):
  - paragraph direction from first strong char, but >= 40% Hebrew words -> RTL;
  - message RTL when >= 20% words Hebrew; inside RTL message a line is LTR only if 4+ English words, no Hebrew, ends `.?!`;
  - RTL paragraphs: LTR runs isolated (+ RLM);
  - registered KB names, routes (`→`/`->`), old->new pairs isolated (LRI…PDI);
  - runs <= 28 chars kept together (NBSP + word joiner after "/");
  - Hebrew prefix hyphen ("ב-", "ל-"…) + WJ (U+2060);
  - `<Area>` shown as "· Area";
  - `plain()` RLM both ends; `ltr_block`/`name_block`/`ltr_name`;
  - markdown rendered: headings -> bold, pipe tables -> "a · b", fences and rules dropped, links kept whole.
- Tests: test_bidi 19, test_i18n 8, test_texts_theme 11.

## 12. Updates
- KB: see 01-brain-data §16.
- App (v1): every 15 min `https://api.github.com/repos/Maple-Helper/maple-helper/releases/latest` (skip drafts/prereleases; tag `v?\d+(\.\d+){1,2}`); asset `MapleHelper-Setup.exe` + sha256 from `SHA256SUMS.txt` (both from that tag; HTTPS from this repo or `Amitaflalo1995/maple-helper`); download to `updates/MapleHelper-Setup-<tag>.exe` via `.part` with progress; size + hash must match; never same/older. UI: update bar available -> ready, toast, tray "Update to X". Update now: "installing", quit after 1.8 s, `installer /SILENT /NORESTART /LOG=… /LANG=english|hebrew /LAUNCHARGS=--updated`. On quit: `/VERYSILENT /SUPPRESSMSGBOXES` (relaunch `--background`). Detached. macOS/portable: toast + tray link to release `html_url`. Old installers/.part removed at +8 s.
- **v2: `tauri-plugin-updater` with signed `latest.json` from `Maple-Helper/maple-helper-v2`; same UX states (available, downloading %, ready, installing, failed), same "Update now" vs "on quit" behaviour, same 15-min cadence, same `--updated` relaunch, same tray item. See design spec §9.**

## 13. Telemetry
- PostHog key `phc_mUxe6uw9XcoyfVDMytYXY4swn7YM78B2ZmbK7awwnrJk`, host `https://us.i.posthog.com`, POST `/batch/` `{"api_key","batch":[…]}`, UA "MapleHelper", 10 s timeout. Always on; `MAPLEHELPER_NO_TELEMETRY` disables; `MAPLEHELPER_POSTHOG_KEY` overrides.
- Queue <= 500; batches of 20 or every 30 s; failures dropped; 2 s flush on quit. `distinct_id` = `install_id`.
- Base props: app_version, `$os` (Windows / Mac OS X), `$os_version`, frozen, `$lib="maplehelper"`. Context: language, provider, saver_mode, characters (count), char_class, char_job, char_level. Values scalars or strings <= 64 chars (longer dropped).
- Events: `app_started {fresh_install, background, $set: context+app_version+kb_version}`, `app_updated {from_version}`, `app_closed {minutes}`, `window_opened {window}`, `question_asked {answered_by: instant|provider, tagged, screenshot, saver, retry}` (instant: answered_by + tagged only), `voice_used`, `kb_updated {from_version, to_version, manual}`, `kb_update_failed {reason}`, `app_crashed {error (type), where (file:line in app), thread}`.
- Presence `app_running` at start + every 600 s: base + context + `run_id` (uuid per run), `first_ping`, `first_today`, `first_this_month` (UTC), `$process_person_profile: false`; `presence_sent` updated only after success.
- **v2 additions:** `frozen` stays true for installed builds; new prop `generation: 2` on every event (lets dashboards split v1/v2). Tests: test_telemetry 16, test_presence 11.

## 14. Other
- Clipboard: answer text (bold markers stripped), card image, character share card (380 px wide, solid background).
- Profile sync: `SYNC_QUESTION`/`GRIND_QUESTION` light prompt, 60 s; portrait crop on worker -> 128 px PNG.

## 15. v1 tests pinning this slice
test_chat_audit 49, test_round4_ui 41, test_overlay_fixes 29, test_onboarding_fixes 24, test_dialogs_provider 24, test_overlay_audit 20, test_bidi 19, test_dialogs_launch_audit 18, test_dialogs_audit 16, test_look_ux 16, test_telemetry 16, test_packaging 16, test_kb_update_retry 12, test_presence 11, test_texts_theme 11, test_first_run_ui 10, test_followup 9, test_i18n 8, test_perf_launch_audit 7, test_app_update_interval 6, test_dialog_threads 4.
Notable: tray item "In-game toolbar" checkable; `show_chat` never closes an open chat; Settings has no Esc; first-run ✕ asks before quitting.
