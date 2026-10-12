# Maple Helper v2: Tauri + Rust rebuild, design spec

- Date: 2026-10-12
- Owner: Tal Ayash
- Repo: `Maple-Helper/maple-helper-v2` (new)
- Replaces: `Maple-Helper/maple-helper` (Python/PySide6), at v0.18.1, commit 010107f
- Status: draft, awaiting owner review

## 1. Intent

### What the owner said
- Rebuild Maple Helper from scratch in a new repo on a new stack, modelled on the owner's `agentrium` desktop app (Tauri 2, Rust, React).
- Functionality must stay the same as the current app.
- Pain with the current stack covers all four areas: installer size and startup time, Qt stability and threading bugs, packaging and signing, and developer experience.
- Existing players should migrate without noticing (decision 2 = A).
- The new repo takes over every release channel, including the app, `kb.zip` and `kb-manifest.json`. The old repo is archived after the bridge (decision 3 = B).
- KB pipeline is hybrid. A Rust crate owns KB parsing and the TSV tables. The scrapers and the translation step stay in Python (decision 4 = A).
- Rollout is an opt-in beta first, then a bridge release to everyone (decision 5 = A).
- Architecture: fully native Rust, with no Python at runtime (approach 1).

### Assumptions (owner, please correct)
- Product name, data folder, keyring service and bundle id stay `Maple Helper` / `MapleHelper` / `com.maplehelper.app`.
- Visuals match the current app: same layout, theme tokens, Rubik font, orange brand, glass look, and light/dark modes. Icons may move from Segoe Fluent glyphs to a bundled icon set so both OSes look the same; this is the only intended visual change.
- Platforms: Windows x64 and macOS arm64, as today. macOS x64 is out of scope.
- Telemetry stays on PostHog with the same key and events, plus `generation: 2`.
- The site `maplehelper.app` is a separate repo and out of scope. Only its download links change.

### Success criteria
1. Every item in `docs/parity/01..04` is implemented, or listed with a recorded decision.
2. The v1 test suite has a ported equivalent for every behaviour-pinning test, and all of them pass on Windows and macOS CI. Golden numbers match exactly.
3. The v2 app reads a v1 data folder untouched: settings, characters, history, pins, wishlist, grind and loot data, `loot.sqlite`, keyring keys. Writing it back stays readable by v1.
4. The Windows installer is at most 120 MB without the KB. Cold start to tray is under 1.5 s on a mid-range PC. A chat answer has no added latency compared with v1.
5. Bridge: a v0.18.x install auto-updates to v2 with no user action, keeps all data, and leaves no v1 files behind.

## 2. Stack (mirrors agentrium)

| Layer | Choice |
|---|---|
| Shell | Tauri 2.11.x (`macos-private-api`), Rust edition 2021, stable toolchain |
| Frontend | React 18.3, Vite 8, TypeScript 5.9 strict, Tailwind 3.4 (tokens as CSS variables), Zustand 4.5, framer-motion; no router (windows are routed by label) |
| Tauri plugins | single-instance (registered first), updater, process, clipboard-manager, dialog. Settings stay in v1's `settings.json` (see §6), not plugin-store |
| Async/HTTP | tokio (full), reqwest 0.13 (rustls, gzip, json) |
| Data | serde/serde_json, rusqlite (bundled) for `loot.sqlite`, keyring 3 (windows-native, apple-native), directories |
| Text | fancy-regex (Python `re` parity, including lookbehind), unicode-normalization, our own port of `difflib.SequenceMatcher` / `get_close_matches` |
| Imaging | image (decode/encode), ndarray, realfft, our own port of Pillow's resampler (parity), ab_glyph (name-tag rendering) |
| ML | ort (ONNX Runtime): PP-OCR det/rec, Silero VAD. whisper-rs (whisper.cpp): speech to text |
| Audio | cpal (16 kHz mono capture, device names) |
| OS | windows crate (Win32: windows, DWM, capture, display affinity, registry, mutex); objc2 / objc2-app-kit / core-graphics (macOS) |
| Logging | tracing + tracing-appender, rolling `logs/maplehelper.log` (1 MB x 2). This departs from agentrium's `eprintln!`, because "Report a problem" must zip logs |
| Tests | cargo test (unit plus golden), Vitest (jsdom and node projects), Testing Library |
| CI | GitHub Actions, the agentrium patterns (SHA-pinned actions, rust-cache, a `latest.json` job after all builds) |

Agentrium conventions adopted as-is:
- `AppState` via `app.manage`.
- `db_op`-style `spawn_blocking` for blocking work.
- Commands return `Result<T, String>` and are wrapped in `wrap_cmd`.
- `user_err` marks user errors so they skip crash telemetry.
- Events are kebab-case.
- Thin `invoke` wrappers live in `src/lib/*.ts` with hand-written TS types. The frontend error rule is a toast plus `reportInvokeFailure`, never console-only.
- Zustand stores and the CSS-variable Tailwind theme.

## 3. Repository layout

```
maple-helper-v2/
  Cargo.toml                    workspace
  crates/
    mh-text/                    SequenceMatcher port, Python-compatible rounding/format, regex helpers, bidi core
    mh-store/                   DATA_DIR, atomic JSON, Settings, Profiles, History, grind/loot json, loot.sqlite, keyring
    mh-kb/                      KB load, pages, mentions, aliases, drops/community, availability, routes graph,
                                sources, sitedata, recent, news, guides, glossary, TABLES (TSV build + mark)
    mh-play/                    combat, plan, jobs, grind summarize, farm, quests, crafting, craftplan, scrollplan,
                                gearadvice, sellkeep, market (npc prices), buildplan, citizenship
    mh-brain/                   prompts (verbatim text assets), build_prompt, split_meta/post-processing,
                                quick answers, planner, reply_language, stated_level
    mh-providers/               Race/Lines process orchestration; claude, codex, gemini(agy), grok; usage; installers;
                                login flows; keyring API keys
    mh-live/                    MesoWatch, OSMS, MeowDB FM API, server status
    mh-vision/                  capture (+ content_box, tiles, JPEG), Pillow-compatible resample, OCR (ort + PP-OCR),
                                minimap locator + follow, loot parser/tracker, HUD reader, portrait, inventory reader
    mh-voice/                   cpal recorder, Silero VAD, whisper-rs, model download/consent, device list
    mh-os/                      win/ and mac/: game window discovery, cover checks, focus, window styles
                                (no-activate, alpha, display affinity, collection behavior), autostart, permissions
    mh-update/                  KB manifest fetch/verify/swap, app update state machine (wraps tauri updater)
    mh-telemetry/               PostHog batch client, presence, crash hook
    mh-guard/                   tiny bin: Grok PreToolUse read guard (replaces C#/PS1)
    mh-cli/                     bin for CI: build-tables, validate-tables, drop-keys, mention-recall, eval-quick, oracle
  src-tauri/                    app crate: main.rs, state.rs, commands/<domain>.rs, events.rs, windows.rs, tray.rs,
                                scanners (minimap, loot, grind runner), tauri.conf.json, capabilities/
  src/                          React app (one bundle; window chosen by label)
    windows/                    chat/, tools/, settings/, onboarding/, guides/, news/, patchnotes/, whatsnew/,
                                item/, wishlist/, history/, market/, map/, toasts/, glossary/, traymenu/,
                                regionpicker/, overlays/{toolbar,npc,guide,monster,loot,portaldots}
    components/ui/              glass panel, buttons, segmented, switch, stepper, chips, cards, picker, notice, toast
    lib/                        invoke wrappers per domain, i18n (t, tp, plural, mac), bidi, markdown, format (_num, _clock)
    store/                      zustand: settings, profile, chat, location, updates, toasts
    i18n/strings.json           1,554 keys {he, en}, extracted mechanically from v1 i18n.py
  assets/                       copied from v1 assets/ (brand, fonts, guides, notes/whatsnew.json, fallback, items, ...)
  models/                       PP-OCR det/rec onnx + dict, silero_vad.onnx (git LFS); whisper model is downloaded at runtime
  pipeline/                     Python: scrape_meowdb, scrape_community, scrape_news, meowdb_sections, translate_*,
                                build_guides, make_aliases, kb_release (calls mh-cli), nightly_gate, eval_answers (live)
  tests/fixtures/               v1 fixtures copied (kb/, minimap_*_live.png, loot_*, hud, portrait, inventory, community, ...)
  tests/oracle/                 JSON golden outputs generated from v1 Python (see §11)
  tools/extract_v1.py           one-shot extractor: prompts, regex tables, i18n, aliases, glossary, constants -> assets/text/*
  packaging/bridge/             Inno Setup bridge installer (v0.x -> v2)
  docs/parity/                  parity checklists (01..04)
  .github/workflows/            ci.yml, release.yml, kb-update.yml, bridge.yml
```

Rule: each crate is pure Rust and testable without Tauri. Only `src-tauri` knows about windows and events.

## 4. Process and threading model

- **Main thread:** Tauri event loop, windows, tray.
- **Async runtime (tokio):** provider processes (`tokio::process`, line streams, hedged `Race` with `select!`), HTTP (KB, MesoWatch, OSMS, telemetry, server status), timers.
- **Dedicated worker threads**, each owning its state and talking over channels:
  - `minimap-scanner`: 0.5 s reads, plus the follow loop at 150 ms while moving. It owns the Locator and its OCR session.
  - `loot-scanner`: 0.5 s reads. It owns the tracker and uses the shared HUD/loot OCR session behind a mutex, as in v1.
  - `grind-runner`: timer at 60 s auto reads, 90 s read wait, 20 min idle.
  - `voice`: cpal stream callback, then a VAD and whisper job on a blocking task.
- **Shared state:** `AppState { settings: RwLock<Settings>, profiles: RwLock<Profiles>, kb: ArcSwap<Kb>, location: watch::Sender<Location>, brain: Brain, ... }`. KB reload swaps the `Arc`; scanners pick up the new KB on their next tick.
- **Events to the UI**, kebab-case: `settings-changed`, `profile-changed`, `answer-delta`, `answer-stage`, `answer-done`, `location-changed`, `loot-updated`, `grind-updated`, `voice-state`, `update-state`, `kb-reloaded`, `toast`, `news-changed`, `server-status`.
- **Crash safety:** a panic hook sends `app_crashed` with a blocking flush (2 s), then aborts. No Qt-style cross-thread object lifetimes exist, which removes the v1 dialog-thread race class.

## 5. Windows (Tauri) — mapping v1 window behaviour

One React bundle. Each window is a `WebviewWindow` with a fixed label, and `main.tsx` renders the component for that label. Windows are created lazily and destroyed on close (v1 behaviour: "deleted on close", one per kind).

| Label | v1 equivalent | Tauri config / native tweaks |
|---|---|---|
| `chat` | Overlay | decorations false, transparent, always_on_top, resizable. Windows: shown in taskbar/Alt+Tab. macOS: `visible_on_all_workspaces` + FullScreenAuxiliary via objc2, Accessory activation policy (LSUIElement) |
| `tools`, `settings`, `onboarding`, `guides`, `news`, `patchnotes`, `whatsnew`, `item`, `wishlist:<cid>`, `history:<cid>`, `market`, `map` | GlassDialogs | decorations false, transparent, always_on_top, own title bar, drag via `data-tauri-drag-region` |
| `toast-<n>` | Toast | no-activate (see below), skip_taskbar, focusable false, positioned by Rust |
| `glossary` | glossary popup | no-activate, auto-hide 7 s |
| `traymenu` | styled QMenu | custom webview menu at the cursor (RTL-capable); native menu fallback |
| `regionpicker` | RegionPicker | one window per monitor (Tauri can't span monitors with mixed DPI), always_on_top, crosshair, no cancel |
| `ov-toolbar`, `ov-npc`, `ov-guide`, `ov-monster`, `ov-loot` | overlays | no-activate, always_on_top, skip_taskbar, transparent; keep-off-minimap enforced in Rust on move |
| `ov-portaldots` | PortalDots | no-activate + `set_ignore_cursor_events(true)` + `SetWindowDisplayAffinity(WDA_EXCLUDEFROMCAPTURE)` on every show |

Native helpers in `mh-os`, called with the window's HWND / NSWindow:
- **No-activate (Windows):** add `WS_EX_NOACTIVATE | WS_EX_TOOLWINDOW` and show with `SW_SHOWNOACTIVATE`.
- **No-activate (macOS):** a non-activating panel style via objc2 (`NSWindowStyleMaskNonactivatingPanel` with `canBecomeKeyWindow` false), or `orderFrontRegardless` without app activation.
- **Opacity / step aside:** `set_window_alpha(win, a)`. Windows uses `WS_EX_LAYERED` + `SetLayeredWindowAttributes`; macOS uses `NSWindow.alphaValue`. Used for the 120 ms see-through capture and for `_take_front` (show at alpha 0 + activate, then capture).
- **Edge resize:** `start_resize_dragging(direction)` from the 12 px shadow + 8 px edge zones, drawn in CSS.
- **Shadow/glass:** CSS inside the transparent window. A 12 px margin holds the shadow; radius 22; sheen and rim gradients; `backdrop-filter` blur for the character-menu cover.
- **DPI:** all geometry stored in v1's units. Logical points for window positions; capture coordinates are physical pixels on Windows and points on macOS (`SCREEN_COORDS_ARE_PHYSICAL`). `to_capture`/`from_capture` are ported, so stored `minimap_region` values keep working.
- **Focus return:** on chat close or minimize, `focus_window(game)`.

**Risk spike (first task of M3):** WebView2 transparent windows combined with `WS_EX_LAYERED` alpha and `WS_EX_NOACTIVATE`. Fallback if alpha on a layered WebView2 misbehaves: hide and show via `SetWindowPos` with `SWP_NOACTIVATE`, plus `DwmSetWindowAttribute(DWMWA_CLOAK)` for the see-through moment. A cloaked window can't be clicked and isn't captured, which matches the v1 intent ("never click-through or invisible while it still blocks clicks").

## 6. Data compatibility (the migration mechanism)

v2 uses **the same files with the same formats** in the same `DATA_DIR`. No conversion step exists, so rollback to v1 stays possible during beta.

- `settings.json`: typed struct plus `#[serde(flatten)] extra` for unknown keys, so they round-trip. Same defaults, `_sane_types` fallback, migration rule, atomic write (tmp + fsync + .bak + replace, retry 10 x 0.2 s), and `indent=1`.
- `profiles.json`: same character fields plus extras round-trip at both levels. `_repair` and `apply_update` are ported line by line.
- `history/*.jsonl`, `*.summaries.json`, `*.fresh`; `grind.json`; `loot.json`; `mesowatch.json`; `osms_cash_shop.json`; `avatars/`.
- `loot.sqlite`: identical schema and `user_version = 1`.
- Keyring: service `MapleHelper`, the same 4 user names; Gemini `gemini`/`antigravity`.
- KB folder: identical layout. TSV tables byte-identical (oracle-tested). `meta.json`, `kb_checked.txt`.
- Voice model: v1's CT2 folder is ignored. If it exists, v2 shows a one-time note that the voice model will be re-downloaded in a new format (consent flow as usual). The old folder is deleted after the new one is ready.
- `install_id` is preserved, so telemetry continuity holds.

## 7. Domain ports (crates)

Each crate's behaviour is specified by its parity file. Rules that apply everywhere:

1. **Mechanical extraction, not retyping.** `tools/extract_v1.py` runs once against v1 at commit 010107f and writes `assets/text/`. It covers:
   - prompts: SYSTEM_PROMPT, REPLY_RULES, HUD_RULE, SUMMARY_PROMPT, the notes
   - every classifier regex
   - `i18n.STRINGS`
   - the alias tables, the TERMS/APP glossary and the job dictionaries
   - planner FAMILIES/AREAS
   - `_en_sounds`, ENGLISH_WORDS, COMMON_LEAD_WORDS, PLAYER_HEBREW

   The extractor's output is committed. Rust loads it with `include_str!`.
2. **Python semantics helpers** in `mh-text`:
   - `py_round` (banker's rounding, including `round(x, n)` via repr-exact decimal)
   - `py_floor_div`
   - `fmt_thousands` (`{:,.0f}`), `fmt_fixed`
   - `casefold`, `str.lower` and Unicode `\w` (fancy-regex Unicode mode)
   - `SequenceMatcher` with autojunk, `get_close_matches`
   - `random` is never used in logic
3. **Golden parity:** every function with numeric or text output gets oracle tests (§11).
4. **v1 quirks** are kept unless a parity file records a decision otherwise. The recorded decisions: fix `craftadvisor._key_of`; keep the two meso-chance conventions; keep the NiaMeowDB scroll prices.

Crate-specific notes:
- **mh-kb:** `Kb` is immutable after load and indexes are built lazily (`OnceCell`). `warm()` runs on a blocking task. Tables are built into `.tmp` files with the mark written last. The `ensure()` lock, backoff and foreign-index guard are kept.
- **mh-brain / mh-providers:**
  - `Race` and `Lines` become async types. Hedge at 25 s, stall at 150 s, exit grace 3 s, outlive 5 s.
  - Process tree kill: Windows Job Object with `KILL_ON_JOB_CLOSE` instead of `taskkill /T` (same effect, more robust; tested); `killpg` on macOS.
  - `type_into_console` stays (AttachConsole + WriteConsoleInputW via the windows crate).
  - The Claude warm process pool is kept with the same key and max age 900 s.
  - Grok's guard becomes the `mh-guard` binary, shipped next to the app. Its self-test and contract are unchanged.
- **mh-vision:**
  - `resample.rs` is a line port of Pillow's `ImagingResample`: BILINEAR/LANCZOS/NEAREST with support scaling, plus `thumbnail` semantics.
  - JPEG via `image` (quality 82/85). Byte equality with Pillow is not required; size and quality class are.
  - OCR: PP-OCR det (DB) and rec (CTC) on `ort`, with 1 intra thread and 1 inter thread. Preprocessing and post-processing are ported from RapidOCR, with parameters read from the same yaml defaults. Parity is checked by recorded expectations: for every fixture image, the v2 OCR text lines must match v1's (stored in `tests/oracle/ocr/*.json`) after whitespace normalization, with at most 1 character of edit distance per line. Downstream minimap and loot results must match exactly.
- **mh-voice:** see parity 02 §10. **Spike in M7:** pick the GGML quantization (q5_0 or q8_0) and the Windows GPU backend (CUDA or Vulkan) by measuring WER and latency on a 40-clip Hebrew/English eval set recorded from v1 usage patterns. The acceptance bar: WER no worse than v1's CT2 int8 + 1 point.

## 8. Frontend

- `strings.json` is extracted from v1. `t(key, vars)` follows v1 lookup order (language, then en, then key), plus `tp` (provider variants), `_one` plurals and `_mac` variants. Hebrew NBSP rules are applied at extraction time for static strings and at runtime for placeholders.
- **Bidi:** `lib/bidi.ts` ports `bidi.py`'s decisions (paragraph direction, LTR-run detection, name isolation, 28-char run glue, prefix hyphen WJ, `<Area>` → "· Area"). It renders runs as `<bdi>`/`dir` spans instead of Qt control characters, because Chromium/WebKit honour HTML isolation reliably. The `test_bidi` cases are re-expressed as assertions on the segment output plus Playwright screenshot checks of 10 reference messages in both engines.
- **Markdown:** `markdown-it` with v1's transforms (headings to bold, pipe tables to "a · b", fences and rules dropped), then DOMPurify. Glossary "?" annotation (first 6 terms) and the answer link/key behaviour stay.
- **Theme:** v1 tokens become CSS variables (`--glass`, `--text`, `--muted`, `--fill1..3`, ...) for light, dark and high contrast. High contrast on Windows is detected in Rust (`SPI_GETHIGHCONTRAST`) and sent as an event. Font sizes 13/14/16 with v1's derived scale.
- **Feed:** row components match v1's row types. The scroll anchoring rules (bottom anchor, user-scrolled detection, long-answer anchor, "↓ New answer" pill), the 80-row cap and the 90 ms stream throttle are kept.
- **Keyboard:** v1's map (§4 of parity 03). No global shortcuts are registered.
- **Tour:** a spotlight overlay inside the chat window with the same 16 steps and the same keyboard handling.

## 9. Install, update and migration

### Ongoing (v2 → v2)
- Tauri bundler: **NSIS** (`installMode: currentUser`, Hebrew and English) and **DMG/app** for macOS arm64.
- `tauri-plugin-updater` with a minisign key. `latest.json` is built once after all platform builds (the agentrium `build-updater-json.mjs` pattern).
- The UX follows v1's state machine. Check every 15 min, from installed builds only. Bar states: available, downloading %, ready, installing, failed. "Update now" relaunches with `--updated`; otherwise the update installs on quit and relaunches with `--background`. The tray shows "Update to X".
- macOS gets real self-update. v1 couldn't do this, so it counts as an improvement. The app stays ad-hoc signed: the updater artifact is verified by minisign, not by Gatekeeper. Developer ID and notarization stay a later, optional step.
- Windows Authenticode signing is optional, through Azure Trusted Signing as in agentrium (`TRUSTED_SIGNING_ENABLED`). Off until the owner sets it up.

### Bridge (v0.18.x → v2), Windows
1. In the **old** repo, publish release **v1.0.0** (non-prerelease, marked latest) with assets named exactly as v1 expects: `MapleHelper-Setup.exe` and a `SHA256SUMS.txt` that covers it.
2. That `MapleHelper-Setup.exe` is an **Inno Setup bridge installer** built in `packaging/bridge/`. It:
   - keeps the v1 AppId `{4B526220-…}` and `SetupMutex=MapleHelperSetup`;
   - accepts v1's switches (`/SILENT`, `/VERYSILENT`, `/SUPPRESSMSGBOXES`, `/NORESTART`, `/LOG=`, `/LANG=`, `/LAUNCHARGS=`);
   - waits for or terminates `MapleHelperRunning` exactly as v1 did;
   - removes the v1 program files (`{app}\_internal`, the exe, the v1 uninstaller registration) **without** touching `%APPDATA%\MapleHelper` or Credential Manager;
   - runs the embedded v2 NSIS installer with `/S`;
   - removes the v1 Run value and writes v2's autostart only if `start_with_windows` is true;
   - launches v2 with `--updated` (interactive or "Update now") or `--background` (silent on-quit path).
3. v1 clients therefore update themselves into v2. From then on v2's updater reads `maple-helper-v2`.
4. **Install location:** v2's NSIS installs to `%LOCALAPPDATA%\Programs\Maple Helper` (`installDir` override) to keep Start Menu, desktop shortcut and pinned-taskbar paths stable. The executable name stays `Maple Helper.exe`.

### Bridge, macOS
v1 on macOS only shows "New version available" with a link to the old repo's latest release page. The bridge release notes link the v2 DMG. Users drag-install it, and the same data folder carries everything over. v2 removes the v1 LaunchAgent if one points to a different path and rewrites it.

### KB channel during the transition
The nightly publishes `kb.zip` and `kb-manifest.json` to **both** repos until the old one is archived. That happens 60 days after the bridge, or when PostHog shows fewer than 2% of `app_running` events from v1 (`generation` absent), whichever is later. The v2 KB updater allowlists `maple-helper-v2`, `maple-helper` and `Amitaflalo1995/maple-helper`.

### Versioning
- v2 starts at **1.0.0**. Betas are `1.0.0-beta.N` prereleases in `maple-helper-v2`, which the v1 updater never sees.
- Version files bumped by `/publish`: `Cargo.toml` (workspace), `src-tauri/tauri.conf.json`, `package.json`, `README.md`.
- `assets/notes/whatsnew.json` keeps v1's format and history. A test requires notes for the current version in both languages, as v1 did.

## 10. KB pipeline (hybrid)
- `pipeline/` holds the v1 Python scripts, moved over nearly unchanged with these modifications:
  - Table building and `drop_keys` call `mh-cli` instead of importing `maplehelper.*`.
  - `nightly_gate` runs the Rust test suite (`cargo test -p mh-kb -p mh-brain -p mh-play --features real-kb`) plus pipeline pytest, with the same drift-vs-known logic and the same 25-failure cap.
  - `release.py --kb-only` publishes to both repos during the transition window (§9).
- Python here is CI-only and never ships to players.
- `mh-cli` subcommands:
  - `build-tables <kb>`, which writes the TSVs and mark
  - `validate-tables`
  - `drop-keys --hebrew < json`
  - `mention-recall <fixture>`
  - `eval-quick <answers.json>`
  - `oracle-dump`, which emits v2 outputs for comparison with v1 oracles
- `kb-update.yml` moves to v2 with the same schedule (`23 0 * * *`, full refresh on Sundays), secrets and validation thresholds.

## 11. Testing and parity strategy
1. **Oracle fixtures from v1.** A script, `tests/oracle/gen_oracles.py`, runs only against a v1 checkout at 010107f. It calls v1's pure functions over fixture inputs and broad generated inputs (fixed seeds), then writes JSON:
   - combat: grid of acc/level/avoid
   - grind summarize
   - farm, craftplan, scrollplan, gearadvice
   - quests `for_level`
   - routes
   - `mention_spans` over the recall fixture and the Hebrew gamer sentences
   - `reply_language`, `drop_keys`, `split_meta`, `streamed_text`, `strip_lead_in`
   - `build_prompt` on the fixture KB
   - quick answers, planner output, availability, sources `page_note`
   - tables (TSV bytes) on the fixture KB
   - bidi segments, `_num`/`_clock`
   - minimap `locate` on each live frame, loot tracker on the recorded reads, HUD/portrait/inventory on their fixtures
   - OCR lines per fixture

   Rust tests load these and assert equality. Oracles are regenerated only on purpose.
2. **Ported v1 tests.** Every v1 test that pins behaviour gets a Rust or Vitest equivalent; parity files list them per area. UI audit tests (`test_chat_audit`, `test_overlay_*`, `test_round4_ui`, ...) become Vitest + Testing Library component tests, plus a WebdriverIO/tauri-driver smoke suite on Windows CI for window flags (always-on-top, no-activate, display affinity).
3. **Real-KB tests.** Behind `--features real-kb`, using the KB from the latest release. They run in CI and in the nightly gate.
4. **Evals.** `evals/answers.json` moves over. Quick mode runs in `cargo test`. Live mode (`pipeline/eval_answers.py` driving `mh-cli ask`) is run by hand before the bridge, for every signed-in provider, and must not regress recall versus v1's last report.
5. **Selftest.** `Maple Helper.exe --selftest <file> [--require-kb]` checks the bundled ONNX models load, VAD runs, the keyring backend works, assets are present, the KB count is right and game-window discovery works, then prints `SELFTEST OK`. CI runs it on the built installer (install, selftest, reinstall preserving data, uninstall), as v1's `-TestInstaller` did.
6. **Bridge test.** A Windows CI job installs the published v0.18.1, seeds a data folder, runs the bridge installer silently with v1's on-quit flags, then asserts: the v2 exe runs, the data is untouched, the v1 files are gone, one uninstall entry exists, and autostart is correct.

## 12. CI and release
- `ci.yml` (push and PR):
  - fmt and clippy (`-D warnings`)
  - `cargo test --workspace --locked` on windows-latest and macos-latest
  - Vitest and `tsc --noEmit`
  - Windows NSIS build with the fixture KB, plus the installer test and selftest
  - macOS app/DMG build and selftest on non-PR runs
- `release.yml` (tag `v*`):
  1. Check the tag equals the workspace version; a suffix makes it a prerelease, and non-prereleases must be on `main`.
  2. The `kb` job carries the latest `kb.zip` forward (agentrium's draft-release flow combined with v1's KB carry-forward and CDN hash verification).
  3. Matrix builds for windows-latest (NSIS) and macos-latest arm64.
  4. A `latest.json` job and `SHA256SUMS.txt`, then publish.
  5. Release assets: `MapleHelper-Setup.exe` (v2 NSIS), `MapleHelper-macOS.dmg`, the macOS updater archive (`.app.tar.gz` + `.sig`, named by the Tauri bundler), `latest.json`, `SHA256SUMS.txt`, `kb.zip`, `kb-manifest.json`.
- `bridge.yml` (manual dispatch): builds the Inno bridge around a given v2 NSIS installer and publishes v1.0.0 to the **old** repo with `SHA256SUMS.txt`. It needs a token with write access to `Maple-Helper/maple-helper` (secret `LEGACY_REPO_TOKEN`).
- `kb-update.yml`: nightly, as in §10.
- `/publish <version>` slash command, modelled on agentrium: bump the version files, add whatsnew notes in he and en, `cargo check`, commit "Release vX.Y.Z", tag, push.

## 13. Milestones
Each milestone gets its own implementation plan (writing-plans) and ends with its parity items ticked and tests green on both OSes.

| # | Milestone | Exit criteria |
|---|---|---|
| M0 | Bootstrap: workspace, CI skeleton, Tauri hello window, `extract_v1.py`, copy assets and fixtures, `gen_oracles.py` producing oracles | CI green; extracted text committed; oracles committed |
| M1 | `mh-text`, `mh-store`, `mh-kb` (+ tables), `mh-cli build-tables`; pipeline moved and calling mh-cli | TSVs byte-identical to v1 on the fixture KB and the real KB; mention recall ≥ 0.99, 0 false positives, < 25 ms; store round-trip tests on a real v1 data folder |
| M2 | `mh-brain`, `mh-providers`, `mh-live`, `mh-guard`; `mh-cli ask` headless | Prompt oracles equal; post-processing oracles equal; provider tests ported; live evals on Claude match v1 recall |
| M3 | App shell: window spike, tray, single instance, chat window (feed, input, cards, tags, pins, errors), onboarding, settings, toasts, tour, secondary windows, i18n and bidi, theme, KB updater, telemetry | parity 03 items done; UI tests ported; can answer questions end to end on Windows and macOS |
| M4 | `mh-play` + Tools window (15 tabs) | parity 04 golden numbers equal; every tab functional |
| M5 | `mh-os` capture and cover checks, `mh-vision` (resample, OCR, minimap and follow, HUD, portrait, inventory, loot), region picker, scanners, step-aside | Vision oracles equal on all live fixtures; OCR line parity; scanners at v1 cadences |
| M6 | In-game overlays: toolbar, NPC and guide, monster, loot overlay, portal dots, keep-off-minimap | parity 02 §5 items; overlay tests ported; display affinity verified |
| M7 | Voice: spike (quant/GPU), recorder, VAD, whisper, download consent, mic check | WER bar met; voice tests ported |
| M8 | Packaging and release: NSIS, DMG, updater, selftest, installer test, release.yml, `/publish`; beta 1.0.0-beta.1 | Installer ≤ 120 MB without KB; cold start < 1.5 s; beta published |
| M9 | Beta soak (≥ 2 weeks, ≥ 10 testers incl. Hebrew and macOS), bridge installer + bridge test, bridge release, dual KB publishing, old repo archive after the window | Zero data-loss reports; crash rate ≤ v1; bridge test green; v1.0.0 bridge live |

Milestones M1, M2 and M4 can overlap once M0 lands; M5 depends on M3's window spike. A rough effort figure: the v1 codebase is ~50K lines and v2 is likely ~35–45K lines of Rust and TS. The plan will size each milestone in tasks, not dates.

## 14. Risks
| Risk | Mitigation |
|---|---|
| WebView2 transparent + layered + no-activate windows misbehave | M3 starts with a spike; DWM cloak fallback (§5) |
| OCR port drifts from RapidOCR | Same ONNX models; recorded line oracles; downstream exact-match tests |
| Minimap thresholds depend on Pillow resampling | Line port of Pillow's resampler, verified on the 14 live frames before tuning anything |
| Whisper accuracy regresses | Eval clip set + WER bar; q8_0 fallback; CPU path always available |
| Bridge installer edge cases (running app, partial v1 installs, portable copies) | Dedicated CI bridge test; portable/macOS users get a release-page link, as v1 already did |
| Provider CLIs change flags during the port | The same `cli_outdated` detection and the evals catch it; the parity file pins the flags |
| Python-semantics bugs (rounding, regex, casefold) | `mh-text` helpers + oracles generated from the actual v1 functions |
| Large multi-window memory | Windows created lazily and destroyed on close; overlays only while enabled |

## 15. Out of scope
- New features, and any behaviour change not recorded in a parity file.
- macOS Intel, Linux.
- Notarization / Developer ID (optional later).
- The website repo (`maple-helper-site`), apart from its download links.
