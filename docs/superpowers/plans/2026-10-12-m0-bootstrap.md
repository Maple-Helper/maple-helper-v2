# M0 Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the `maple-helper-v2` repo so every later milestone has a Cargo workspace, a Tauri + React app that launches, a pinned v1 reference checkout, v1's assets and fixtures, mechanically extracted v1 constants and strings, a working oracle pipeline (Python → JSON → Rust test), and green CI on Windows and macOS.

**Architecture:** A Cargo workspace with crates in `crates/` and the Tauri app in `src-tauri/`. A single Vite/React bundle routes each window by its label. Python under `tools/` and `tests/oracle/` is dev/CI-only: it reads a pinned v1 checkout in `.v1/` (gitignored) and writes committed artifacts (assets, `assets/text/v1/*.json`, `src/i18n/strings.json`, `tests/oracle/data/*.json`). CI regenerates those artifacts with `--check` to catch drift.

**Tech Stack:** Rust stable (2021 edition), Tauri 2.11.x, React 18.3, Vite 8, TypeScript 5.9 (strict), Tailwind 3.4, Zustand 4.5, Vitest 4 + Testing Library, Python 3.13 + pytest (tools only), GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-12-maple-helper-v2-design.md` (also the roadmap `docs/superpowers/plans/2026-10-12-roadmap.md`)

**M0 exit criteria (spec §13):** CI green; extracted text committed; oracles committed.

## Global Constraints

- Repo: `Maple-Helper/maple-helper-v2`, default branch `main`, public.
- v1 pin: repo `https://github.com/Maple-Helper/maple-helper.git`, commit `010107f665af3b14e18501dd4e1163c7f08b0a38`, `__version__` `0.18.1`.
- Product name `Maple Helper`; bundle identifier `com.maplehelper.app`; executable name `Maple Helper`.
- Stack versions: Tauri 2.11.x (`@tauri-apps/api` ^2.11.1, `@tauri-apps/cli` ^2.11.4), React 18.3, Vite 8, TypeScript 5.9 strict with `noUnusedLocals`/`noUnusedParameters`, Tailwind 3.4, Zustand 4.5, Vitest 4. Node 22. Rust stable, edition 2021. Python 3.13 for tools only.
- Workspace version during M0: `0.1.0`, identical in `Cargo.toml` (`[workspace.package]`), `src-tauri/tauri.conf.json` and `package.json`.
- Commands return `Result<T, String>` once fallible (M0's single command is infallible). Frontend rule: a user action's error gets a toast plus `reportInvokeFailure`; a best-effort call gets `.catch(() => {})` with a comment saying why. Never only `console.error`.
- No global hotkeys, ever (anti-cheat, parity 02 §1).
- Committed v1-derived artifacts are **generated, never hand-edited**. Each generator has a `--check` mode used by CI.
- Every commit message ends with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## Review Focus

1. **Python float semantics at the edges** (`-0.0`, exact ties like `0.125`, `2.675`, values above 1e15): `round` and `format` must match CPython exactly. These are pinned in Task 5's oracle inputs and Task 6's tests.
2. **The v1 checkout is missing, partial or at the wrong commit** (fresh clone, interrupted fetch): `ensure_checkout` must repair it, never silently use other code. This is pinned by Task 2's empty-directory and wrong-HEAD tests.
3. **Copied fixtures altered by line-ending conversion on Windows checkouts**: hashes must survive a clone on any OS. This is pinned by Task 3's `.gitattributes` `-text` rules and the manifest check that CI runs on Linux.
4. **Window labels with suffixes** (`history:<cid>`, `toast-3`) and unknown labels: routing must pick the right component, and an unknown label must render a harmless fallback instead of crashing. This is pinned by Task 7's registry tests.
5. **Version drift between the three version files**: it would break the updater later. This is pinned by Task 7's version-sync test.

---

### Task 1: Workspace foundation and `mh-text::py_round`

**Files:**
- Create: `Cargo.toml`
- Create: `.gitignore`
- Create: `.gitattributes`
- Create: `README.md`
- Create: `CLAUDE.md`
- Create: `crates/mh-text/Cargo.toml`
- Create: `crates/mh-text/src/lib.rs`
- Create: `crates/mh-text/src/pyfmt.rs`

**Interfaces:**
- Consumes: nothing.
- Produces: the workspace (`members = ["crates/*"]`), crate `mh_text` with `pub fn py_round(x: f64) -> i64` (Python's `round(x)`: half to even, on the exact binary value).

- [ ] **Step 1: Create the workspace and hygiene files**

`Cargo.toml`:
```toml
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2021"
publish = false

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
lto = true
opt-level = 3
codegen-units = 1
strip = true
panic = "abort"
```

`.gitignore`:
```gitignore
/target/
/node_modules/
/dist/
/.v1/
/.venv*/
/data/kb/
__pycache__/
*.pyc
.DS_Store
*.log
/src-tauri/gen/schemas/
```

`.gitattributes` (byte-exact v1 copies must never be converted):
```gitattributes
* text=auto eol=lf
*.png binary
*.jpg binary
*.ico binary
*.icns binary
*.ttf binary
*.webp binary
/assets/** -text
/tests/fixtures/** -text
/evals/** -text
```

`README.md`:
```markdown
# Maple Helper (v2)

Your personal MapleStory Classic World assistant. This is the Tauri + Rust rebuild of
[Maple-Helper/maple-helper](https://github.com/Maple-Helper/maple-helper) (v0.18.1).

Unofficial companion for MapleStory Classic World · Not affiliated with Nexon.

- Design: `docs/superpowers/specs/2026-10-12-maple-helper-v2-design.md`
- Parity checklists: `docs/parity/`
- Roadmap: `docs/superpowers/plans/2026-10-12-roadmap.md`
```

`CLAUDE.md`:
```markdown
# CLAUDE.md

Maple Helper v2: a Tauri 2 + Rust + React rebuild of the Python app at `Maple-Helper/maple-helper` (v1).

## The one rule
Behaviour must match v1 at commit `010107f665af3b14e18501dd4e1163c7f08b0a38` unless `docs/parity/*` records a decision.
Proof comes from oracles: `tests/oracle/gen_oracles.py` runs v1's own code and writes `tests/oracle/data/*.json`;
Rust/TS tests must reproduce them exactly. Never hand-edit generated files:
`assets/` (v1 copy), `tests/fixtures/`, `evals/`, `assets/text/v1/`, `src/i18n/strings.json`, `tests/oracle/data/`.

## Layout
- `crates/*`: pure Rust, testable without Tauri. `src-tauri/`: the app (windows, tray, commands, events).
- `src/`: one React bundle; `src/windows/registry.tsx` picks the component by window label.
- `tools/`, `tests/oracle/`: Python 3.13, dev/CI only (`python -m venv .venv && .venv/Scripts/pip install -r tools/requirements.txt`).

## Commands
- `cargo test --workspace` · `cargo clippy --workspace --all-targets -- -D warnings` · `cargo fmt --all`
- `npm run test:run` · `npm run typecheck` · `npm run tauri dev`
- `python -m pytest` (tools) · `python tests/oracle/gen_oracles.py [--check]` · `python tools/extract_v1.py [--check]` · `python tools/import_v1_files.py [--check]`

## Conventions (from agentrium)
- Commands: `Result<T, String>`, wrapped in `wrap_cmd`; user/environment errors via `user_err` (skip crash telemetry).
- Events kebab-case. Thin `invoke` wrappers in `src/lib/*.ts` with hand-written TS types.
- Frontend errors: user action → toast + `reportInvokeFailure`; best-effort → `.catch(() => {})` plus a comment. Never only `console.error`.
- Python semantics (rounding, formatting, regex, difflib) go through `mh-text`, never ad-hoc.
- No global hotkeys, keyboard hooks or game-process access, ever.
```

- [ ] **Step 2: Write the failing test**

`crates/mh-text/Cargo.toml`:
```toml
[package]
name = "mh-text"
version.workspace = true
edition.workspace = true
publish.workspace = true

[dependencies]
```

`crates/mh-text/src/lib.rs`:
```rust
//! Python-compatible text and number helpers. v1 is Python; anything that rounds, formats,
//! matches regexes or compares strings must behave exactly like CPython 3.13 did.

pub mod pyfmt;

pub use pyfmt::py_round;
```

`crates/mh-text/src/pyfmt.rs`:
```rust
//! CPython's `round()` and `format()` for floats.

/// Python's `round(x)` with no ndigits: nearest integer, ties to even, on the exact binary value.
pub fn py_round(x: f64) -> i64 {
    unimplemented!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ties_go_to_the_even_neighbour() {
        assert_eq!(py_round(0.5), 0);
        assert_eq!(py_round(1.5), 2);
        assert_eq!(py_round(2.5), 2);
        assert_eq!(py_round(-0.5), 0);
        assert_eq!(py_round(-2.5), -2);
    }

    #[test]
    fn non_ties_round_to_nearest() {
        assert_eq!(py_round(2.6), 3);
        assert_eq!(py_round(-2.4), -2);
        assert_eq!(py_round(1234567.49), 1234567);
    }
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `cargo test -p mh-text`
Expected: FAIL, panicked at `not implemented`.

- [ ] **Step 4: Implement**

Replace the body in `crates/mh-text/src/pyfmt.rs`:
```rust
pub fn py_round(x: f64) -> i64 {
    x.round_ties_even() as i64
}
```

- [ ] **Step 5: Run the tests and lints to verify they pass**

Run: `cargo test -p mh-text && cargo clippy --workspace --all-targets -- -D warnings && cargo fmt --all --check`
Expected: 2 tests pass; no clippy warnings; fmt clean.

- [ ] **Step 6: Renormalize existing docs and commit**

```bash
git add --renormalize .
git add Cargo.toml Cargo.lock .gitignore .gitattributes README.md CLAUDE.md crates/mh-text
git commit -m "Workspace foundation and mh-text py_round

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Pinned v1 reference checkout

**Files:**
- Create: `pyproject.toml`
- Create: `tools/__init__.py` (empty)
- Create: `tools/requirements.txt`
- Create: `tools/v1.py`
- Create: `tools/tests/__init__.py` (empty)
- Create: `tools/tests/conftest.py`
- Test: `tools/tests/test_v1.py`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `tools.v1`: constants `V1_REPO`, `V1_COMMIT`, `V1_VERSION`, `ROOT: Path`, `V1_DIR: Path`.
  - Functions `head(path: Path) -> str | None`, `ensure_checkout(path: Path = V1_DIR) -> Path`, `import_path(path: Path = V1_DIR) -> Path` (puts the checkout on `sys.path` so `import maplehelper` works).

- [ ] **Step 1: Set up the tools environment**

`pyproject.toml`:
```toml
[tool.pytest.ini_options]
testpaths = ["tools/tests"]
pythonpath = ["."]
addopts = "-q"
```

`tools/requirements.txt` (v1 core modules import numpy/Pillow/mss/keyring at module level; Qt, OCR and voice are imported lazily and not needed):
```text
numpy>=1.26
pillow>=12.3
mss>=10.2
keyring>=25
pytest>=9
```

Run: `python -m venv .venv && .venv/Scripts/python -m pip install -r tools/requirements.txt` (on macOS use `.venv/bin/python`).
Expected: installs without errors.

`tools/tests/conftest.py`. v1's `store.py` resolves `DATA_DIR` from `$APPDATA` at import; point it at a throwaway folder so tools never touch a real v1 data folder:
```python
import os
import tempfile

os.environ["APPDATA"] = tempfile.mkdtemp(prefix="mh-v1-appdata-")
os.environ.setdefault("MAPLEHELPER_NO_TELEMETRY", "1")
```

- [ ] **Step 2: Write the failing tests**

`tools/tests/test_v1.py`:
```python
import subprocess

from tools import v1


def test_ensure_checkout_lands_on_the_pinned_commit():
    path = v1.ensure_checkout()
    assert v1.head(path) == v1.V1_COMMIT


def test_an_empty_folder_becomes_a_pinned_checkout(tmp_path):
    target = tmp_path / "v1"
    target.mkdir()
    v1.ensure_checkout(target)
    assert v1.head(target) == v1.V1_COMMIT
    assert (target / "maplehelper" / "__init__.py").is_file()


def test_a_checkout_at_another_commit_is_reset(tmp_path):
    target = tmp_path / "v1"
    v1.ensure_checkout(target)
    (target / "maplehelper" / "__init__.py").write_text("tampered", encoding="utf-8")
    subprocess.run(["git", "-c", "user.name=t", "-c", "user.email=t@t", "commit", "-qam", "tamper"], cwd=target, check=True)
    assert v1.head(target) != v1.V1_COMMIT
    v1.ensure_checkout(target)
    assert v1.head(target) == v1.V1_COMMIT
    assert "tampered" not in (target / "maplehelper" / "__init__.py").read_text(encoding="utf-8")


def test_v1_imports_and_reports_the_pinned_version():
    v1.import_path()
    import maplehelper

    assert maplehelper.__version__ == v1.V1_VERSION
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `.venv/Scripts/python -m pytest tools/tests/test_v1.py`
Expected: FAIL, `ImportError: cannot import name 'v1' from 'tools'`.

- [ ] **Step 4: Implement `tools/v1.py`**

```python
"""The pinned v1 (Python) checkout: the source of truth that extractors and oracles read.

Network is needed only the first time (or after the pin changes). The checkout lives in .v1/ (gitignored).
"""
from __future__ import annotations

import subprocess
import sys
from pathlib import Path

V1_REPO = "https://github.com/Maple-Helper/maple-helper.git"
V1_COMMIT = "010107f665af3b14e18501dd4e1163c7f08b0a38"
V1_VERSION = "0.18.1"
ROOT = Path(__file__).resolve().parent.parent
V1_DIR = ROOT / ".v1"


def _git(*args: str, cwd: Path) -> str:
    run = subprocess.run(["git", *args], cwd=cwd, check=True, capture_output=True, text=True)
    return run.stdout.strip()


def head(path: Path) -> str | None:
    """The checked-out commit, or None when path is not a usable git checkout."""
    if not (path / ".git").exists():
        return None
    run = subprocess.run(["git", "rev-parse", "HEAD"], cwd=path, capture_output=True, text=True)
    return run.stdout.strip() if run.returncode == 0 else None


def ensure_checkout(path: Path = V1_DIR) -> Path:
    """Make path a clean checkout of V1_COMMIT (fetching it if needed) and return it."""
    path = Path(path)
    if head(path) == V1_COMMIT and not _git("status", "--porcelain", cwd=path):
        return path
    path.mkdir(parents=True, exist_ok=True)
    if not (path / ".git").exists():
        _git("init", "-q", cwd=path)
        _git("remote", "add", "origin", V1_REPO, cwd=path)
    _git("fetch", "-q", "--depth", "1", "origin", V1_COMMIT, cwd=path)
    _git("checkout", "-q", "--force", "--detach", V1_COMMIT, cwd=path)
    _git("clean", "-qfdx", cwd=path)
    if head(path) != V1_COMMIT:
        raise RuntimeError(f"v1 checkout at {path} is not {V1_COMMIT}")
    return path


def import_path(path: Path = V1_DIR) -> Path:
    """Put the pinned checkout first on sys.path so `import maplehelper` loads v1."""
    path = ensure_checkout(path)
    entry = str(path)
    if entry not in sys.path:
        sys.path.insert(0, entry)
    return path


if __name__ == "__main__":
    print(ensure_checkout())
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `.venv/Scripts/python -m pytest tools/tests/test_v1.py`
Expected: 4 passed (the first run fetches v1, roughly 20 MB).

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml tools/__init__.py tools/requirements.txt tools/v1.py tools/tests
git commit -m "Pinned v1 reference checkout for extractors and oracles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Import v1 assets, fixtures and evals with a hash manifest

**Files:**
- Create: `tools/import_v1_files.py`
- Create (generated): `assets/**`, `tests/fixtures/**`, `evals/answers.json`, `tools/v1_files.json`
- Test: `tools/tests/test_import_v1_files.py`

**Interfaces:**
- Consumes: `tools.v1.ensure_checkout`, `tools.v1.ROOT`.
- Produces:
  - `tools.import_v1_files`: `COPIES: list[tuple[str, str]]` (v1 relative path → repo relative path); `build_manifest(root: Path) -> dict[str, str]` (POSIX relpath → sha256 hex); `import_files(v1_dir: Path, root: Path) -> dict[str, str]`; `check(v1_dir: Path, root: Path) -> list[str]` (problems; empty means OK).
  - CLI: `python tools/import_v1_files.py [--check]`.

- [ ] **Step 1: Write the failing tests**

`tools/tests/test_import_v1_files.py`:
```python
import json

from tools import import_v1_files as imp
from tools import v1


def test_import_copies_every_file_byte_for_byte(tmp_path):
    v1_dir = v1.ensure_checkout()
    manifest = imp.import_files(v1_dir, tmp_path)
    assert "assets/fonts/Rubik-Variable.ttf" in manifest
    assert "tests/fixtures/minimap_perion_live.png" in manifest
    assert "evals/answers.json" in manifest
    for rel in ("assets/notes/whatsnew.json", "tests/fixtures/loot_reads_live.json"):
        assert (tmp_path / rel).read_bytes() == (v1_dir / rel).read_bytes()
    assert imp.check(v1_dir, tmp_path) == []


def test_check_reports_a_changed_file(tmp_path):
    v1_dir = v1.ensure_checkout()
    imp.import_files(v1_dir, tmp_path)
    (tmp_path / "assets" / "notes" / "whatsnew.json").write_bytes(b"[]")
    problems = imp.check(v1_dir, tmp_path)
    assert any("assets/notes/whatsnew.json" in p for p in problems)


def test_check_reports_a_missing_file(tmp_path):
    v1_dir = v1.ensure_checkout()
    imp.import_files(v1_dir, tmp_path)
    (tmp_path / "evals" / "answers.json").unlink()
    assert any("evals/answers.json" in p for p in imp.check(v1_dir, tmp_path))


def test_the_committed_copy_matches_v1():
    v1_dir = v1.ensure_checkout()
    assert imp.check(v1_dir, v1.ROOT) == []
    manifest = json.loads((v1.ROOT / "tools" / "v1_files.json").read_text(encoding="utf-8"))
    assert len(manifest) > 100
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `.venv/Scripts/python -m pytest tools/tests/test_import_v1_files.py`
Expected: FAIL, `ImportError: cannot import name 'import_v1_files'`.

- [ ] **Step 3: Implement `tools/import_v1_files.py`**

```python
"""Copy v1's assets, test fixtures and eval cases into this repo, byte for byte, and record their hashes.

python tools/import_v1_files.py           copy + write tools/v1_files.json
python tools/import_v1_files.py --check   exit 1 if anything differs from v1 (CI)
"""
from __future__ import annotations

import argparse
import hashlib
import json
import shutil
import sys
from pathlib import Path

if __package__ in (None, ""):
    sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
from tools.v1 import ROOT, ensure_checkout  # noqa: E402

COPIES: list[tuple[str, str]] = [
    ("assets", "assets"),
    ("tests/fixtures", "tests/fixtures"),
    ("evals/answers.json", "evals/answers.json"),
]
MANIFEST = Path("tools") / "v1_files.json"


def _files(base: Path):
    if base.is_file():
        yield base
        return
    for p in sorted(base.rglob("*")):
        if p.is_file() and "__pycache__" not in p.parts:
            yield p


def _sha(path: Path) -> str:
    return hashlib.sha256(path.read_bytes()).hexdigest()


def _pairs(v1_dir: Path, root: Path):
    for src_rel, dst_rel in COPIES:
        src_base = v1_dir / src_rel
        for src in _files(src_base):
            rel = src.relative_to(src_base) if src_base.is_dir() else Path()
            dst = root / dst_rel / rel
            yield src, dst, (Path(dst_rel) / rel).as_posix()


def build_manifest(v1_dir: Path) -> dict[str, str]:
    return {key: _sha(src) for src, _dst, key in _pairs(v1_dir, Path("."))}


def import_files(v1_dir: Path, root: Path) -> dict[str, str]:
    for src_rel, dst_rel in COPIES:
        dst = root / dst_rel
        if dst.is_dir():
            shutil.rmtree(dst)
        elif dst.exists():
            dst.unlink()
    manifest: dict[str, str] = {}
    for src, dst, key in _pairs(v1_dir, root):
        dst.parent.mkdir(parents=True, exist_ok=True)
        shutil.copyfile(src, dst)
        manifest[key] = _sha(dst)
    out = root / MANIFEST
    out.parent.mkdir(parents=True, exist_ok=True)
    out.write_text(json.dumps(dict(sorted(manifest.items())), indent=1) + "\n", encoding="utf-8", newline="\n")
    return manifest


def check(v1_dir: Path, root: Path) -> list[str]:
    expected = build_manifest(v1_dir)
    problems = []
    for key, sha in sorted(expected.items()):
        local = root / key
        if not local.is_file():
            problems.append(f"missing: {key}")
        elif _sha(local) != sha:
            problems.append(f"differs from v1: {key}")
    for _src_rel, dst_rel in COPIES:
        for p in _files(root / dst_rel) if (root / dst_rel).exists() else []:
            key = p.relative_to(root).as_posix()
            if key not in expected:
                problems.append(f"not in v1: {key}")
    manifest_path = root / MANIFEST
    if not manifest_path.is_file() or json.loads(manifest_path.read_text(encoding="utf-8")) != dict(sorted(expected.items())):
        problems.append(f"stale manifest: {MANIFEST.as_posix()}")
    return problems


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--check", action="store_true")
    args = parser.parse_args()
    v1_dir = ensure_checkout()
    if args.check:
        problems = check(v1_dir, ROOT)
        print("\n".join(problems) or "v1 files: OK")
        return 1 if problems else 0
    manifest = import_files(v1_dir, ROOT)
    print(f"copied {len(manifest)} files from v1")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 4: Generate the copy**

Run: `.venv/Scripts/python tools/import_v1_files.py`
Expected: `copied N files from v1`, with N > 100.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `.venv/Scripts/python -m pytest tools/tests/test_import_v1_files.py && .venv/Scripts/python tools/import_v1_files.py --check`
Expected: 4 passed; `v1 files: OK`.

- [ ] **Step 6: Commit**

```bash
git add tools/import_v1_files.py tools/v1_files.json tools/tests/test_import_v1_files.py assets tests/fixtures evals
git commit -m "Import v1 assets, fixtures and eval cases with a hash manifest

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Mechanical extraction of v1 constants and UI strings

**Files:**
- Create: `tools/extract_v1.py`
- Create (generated): `assets/text/v1/<module>.json` (one per v1 module), `src/i18n/strings.json`
- Test: `tools/tests/test_extract_v1.py`

**Interfaces:**
- Consumes: `tools.v1.import_path`, `tools.v1.V1_COMMIT`, `tools.v1.ROOT`.
- Produces:
  - `tools.extract_v1`: `REQUIRED: tuple[str, ...]`; `encode(value) -> JSON`; `extract(root: Path) -> dict[str, Path]` (written files); `check(root: Path) -> list[str]`.
  - The JSON formats every later milestone reads:
    - `assets/text/v1/<module>.json` = `{"source": "maplehelper/<module path>.py@<commit>", "constants": {NAME: encoded}, "unsupported": [NAME, ...]}`.
    - Encoding:
      - str/int/float/bool/null as themselves
      - list/tuple → JSON array
      - `re.Pattern` → `{"$regex": pattern, "flags": "imsxa"-subset}`
      - set/frozenset → `{"$set": [sorted items]}`
      - dict with all-str keys → JSON object
      - any other dict → `{"$pairs": [[k, v], ...]}`
    - `src/i18n/strings.json` = `{key: {"he": str, "en": str}}`, exactly v1 `i18n.STRINGS`.

- [ ] **Step 1: Write the failing tests**

`tools/tests/test_extract_v1.py`:
```python
import json
import re

from tools import extract_v1 as ex
from tools import v1


def _load(root, module):
    return json.loads((root / "assets" / "text" / "v1" / f"{module}.json").read_text(encoding="utf-8"))


def test_strings_are_every_v1_ui_string_in_both_languages(tmp_path):
    v1.import_path()
    from maplehelper import i18n

    ex.extract(tmp_path)
    strings = json.loads((tmp_path / "src" / "i18n" / "strings.json").read_text(encoding="utf-8"))
    assert strings == i18n.STRINGS
    assert all(set(v) >= {"he", "en"} for v in strings.values())


def test_required_modules_are_extracted(tmp_path):
    written = ex.extract(tmp_path)
    for module in ex.REQUIRED:
        assert module in written, module


def test_prompts_are_verbatim(tmp_path):
    v1.import_path()
    from maplehelper import brain

    ex.extract(tmp_path)
    constants = _load(tmp_path, "brain")["constants"]
    assert constants["SYSTEM_PROMPT"] == brain.SYSTEM_PROMPT
    assert constants["REPLY_RULES"] == brain.REPLY_RULES
    assert constants["HUD_RULE"] == brain.HUD_RULE


def test_regexes_round_trip_with_flags(tmp_path):
    v1.import_path()
    from maplehelper import brain, quick

    ex.extract(tmp_path)
    for module, mod in (("brain", brain), ("quick", quick)):
        constants = _load(tmp_path, module)["constants"]
        for name, value in constants.items():
            if isinstance(value, dict) and "$regex" in value:
                original = getattr(mod, name)
                flags = sum(ex.FLAGS[c] for c in value["flags"])
                rebuilt = re.compile(value["$regex"], flags)
                assert rebuilt.pattern == original.pattern, name
                assert rebuilt.flags == original.flags, name


def test_sets_and_non_string_keys_are_encoded_deterministically():
    assert ex.encode({"b", "a"}) == {"$set": ["a", "b"]}
    assert ex.encode(str.maketrans("ab", "cd")) == {"$pairs": [[97, 99], [98, 100]]}
    assert ex.encode((1, "x", None)) == [1, "x", None]


def test_extraction_is_deterministic(tmp_path):
    ex.extract(tmp_path / "a")
    ex.extract(tmp_path / "b")
    for path in sorted((tmp_path / "a").rglob("*.json")):
        twin = tmp_path / "b" / path.relative_to(tmp_path / "a")
        assert path.read_bytes() == twin.read_bytes(), path.name


def test_the_committed_extraction_is_current():
    assert ex.check(v1.ROOT) == []
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `.venv/Scripts/python -m pytest tools/tests/test_extract_v1.py`
Expected: FAIL, `ImportError: cannot import name 'extract_v1'`.

- [ ] **Step 3: Implement `tools/extract_v1.py`**

```python
"""Dump v1's module-level constants (prompts, regexes, word lists, tables) and UI strings to JSON by
importing v1 itself, so nothing is ever retyped.

python tools/extract_v1.py           write assets/text/v1/*.json and src/i18n/strings.json
python tools/extract_v1.py --check   exit 1 if the committed files differ from a fresh extraction (CI)
"""
from __future__ import annotations

import argparse
import importlib
import json
import os
import pkgutil
import re
import sys
import tempfile
from pathlib import Path

if __package__ in (None, ""):
    sys.path.insert(0, str(Path(__file__).resolve().parent.parent))
os.environ.setdefault("APPDATA", tempfile.mkdtemp(prefix="mh-v1-appdata-"))
os.environ.setdefault("MAPLEHELPER_NO_TELEMETRY", "1")
from tools.v1 import ROOT, V1_COMMIT, import_path  # noqa: E402

# Modules that pull in Qt, OS APIs or the voice stack at import time, or hold no portable constants.
SKIP = {"app", "__main__", "selftest", "winapi", "macapi", "osapi", "report", "voice"}
REQUIRED = ("i18n", "bidi", "brain", "quick", "planner", "kb", "glossary", "routes", "availability",
            "sources", "tables", "combat", "plan", "jobs", "quests", "crafting", "craftplan", "farm",
            "grind", "scrollplan", "gearadvice", "sellkeep", "market", "mesowatch", "osms", "sitedata",
            "recent", "news", "guides", "lootlog", "minimap", "hud", "portrait", "inventory", "capture",
            "providers.base", "providers.claude", "providers.codex", "providers.gemini", "providers.grok")
FLAGS = {"i": re.I, "m": re.M, "s": re.S, "x": re.X, "a": re.A}
_NAME = re.compile(r"^_?[A-Z][A-Z0-9_]*$")


class Unsupported(Exception):
    pass


def _key(encoded) -> str:
    return json.dumps(encoded, sort_keys=True, ensure_ascii=False)


def encode(value):
    if isinstance(value, re.Pattern):
        return {"$regex": value.pattern, "flags": "".join(c for c, f in FLAGS.items() if value.flags & f)}
    if value is None or isinstance(value, (bool, int, float, str)):
        return value
    if isinstance(value, (list, tuple)):
        return [encode(v) for v in value]
    if isinstance(value, (set, frozenset)):
        return {"$set": sorted((encode(v) for v in value), key=_key)}
    if isinstance(value, dict):
        if all(isinstance(k, str) for k in value):
            return {k: encode(v) for k, v in value.items()}
        pairs = [[encode(k), encode(v)] for k, v in value.items()]
        return {"$pairs": sorted(pairs, key=_key)}
    raise Unsupported(type(value).__name__)


def _modules():
    import maplehelper
    import maplehelper.providers

    names = []
    for info in pkgutil.iter_modules(maplehelper.__path__):
        if not info.ispkg and info.name not in SKIP:
            names.append(info.name)
    for info in pkgutil.iter_modules(maplehelper.providers.__path__):
        names.append(f"providers.{info.name}")
    return sorted(names)


def _dump(obj) -> str:
    return json.dumps(obj, ensure_ascii=False, indent=1, sort_keys=True) + "\n"


def _render() -> dict[str, str]:
    """Relative path -> file text, for everything this tool writes."""
    import_path()
    out: dict[str, str] = {}
    errors = {}
    for name in _modules():
        try:
            mod = importlib.import_module(f"maplehelper.{name}")
        except Exception as e:  # recorded, and REQUIRED modules are asserted by the tests
            errors[name] = f"{type(e).__name__}: {e}"
            continue
        constants, unsupported = {}, []
        for attr, value in vars(mod).items():
            if not _NAME.match(attr) or callable(value):
                continue
            try:
                constants[attr] = encode(value)
            except Unsupported:
                unsupported.append(attr)
        source = f"maplehelper/{name.replace('.', '/')}.py@{V1_COMMIT}"
        out[f"assets/text/v1/{name}.json"] = _dump(
            {"source": source, "constants": constants, "unsupported": sorted(unsupported)})
    if errors:
        out["assets/text/v1/_import_errors.json"] = _dump(errors)
    from maplehelper import i18n

    out["src/i18n/strings.json"] = _dump(i18n.STRINGS)
    return out


def extract(root: Path) -> dict[str, Path]:
    root = Path(root)
    target = root / "assets" / "text" / "v1"
    if target.exists():
        for old in target.glob("*.json"):
            old.unlink()
    written = {}
    for rel, text in _render().items():
        path = root / rel
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(text, encoding="utf-8", newline="\n")
        if rel.startswith("assets/text/v1/") and not rel.endswith("_import_errors.json"):
            written[Path(rel).stem] = path
    return written


def check(root: Path) -> list[str]:
    root = Path(root)
    fresh = _render()
    problems = [f"stale: {rel}" for rel, text in fresh.items()
                if not (root / rel).is_file() or (root / rel).read_text(encoding="utf-8") != text]
    committed = {p.relative_to(root).as_posix() for p in (root / "assets" / "text" / "v1").glob("*.json")}
    problems += [f"not produced by v1: {rel}" for rel in sorted(committed - set(fresh))]
    return problems


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--check", action="store_true")
    args = parser.parse_args()
    if args.check:
        problems = check(ROOT)
        print("\n".join(problems) or "v1 extraction: OK")
        return 1 if problems else 0
    written = extract(ROOT)
    print(f"extracted {len(written)} modules")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

- [ ] **Step 4: Generate the extraction**

Run: `.venv/Scripts/python tools/extract_v1.py`
Expected: `extracted N modules`, N ≥ 40. If `assets/text/v1/_import_errors.json` exists, open it. A REQUIRED module listed there means a missing dependency: add it to `tools/requirements.txt`, reinstall and rerun. Non-required modules may stay listed.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `.venv/Scripts/python -m pytest tools/tests/test_extract_v1.py && .venv/Scripts/python tools/extract_v1.py --check`
Expected: 7 passed; `v1 extraction: OK`.

- [ ] **Step 6: Commit**

```bash
git add tools/extract_v1.py tools/tests/test_extract_v1.py tools/requirements.txt assets/text src/i18n/strings.json
git commit -m "Extract v1 constants, prompts, regexes and UI strings mechanically

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Oracle pipeline (Python generator + Rust loader) with the first oracle set

**Files:**
- Create: `tests/oracle/gen_oracles.py`
- Create (generated): `tests/oracle/data/pyfmt.json`
- Create: `crates/mh-oracle/Cargo.toml`
- Create: `crates/mh-oracle/src/lib.rs`
- Test: `tools/tests/test_gen_oracles.py`
- Test: `crates/mh-oracle/src/lib.rs` (unit tests)

**Interfaces:**
- Consumes: `tools.v1.import_path`, `tools.v1.ROOT`.
- Produces:
  - `tests/oracle/gen_oracles.py`: registry `SETS: dict[str, Callable[[], dict]]`, decorator `oracle(name)`, `render(name) -> str`, `write(name) -> Path`, `check() -> list[str]`, CLI `[--only NAME] [--check]`.
  - Every oracle file is `{"cases": [...], ...}`, written with `indent=1`, sorted keys, UTF-8 and a trailing newline.
  - Oracle set `pyfmt`: cases `{"x": float, "round": int, "round1": float, "round2": float, "round3": float, "thousands0": str, "fixed1": str, "fixed2": str}`.
  - Crate `mh_oracle` (dev-dependency for every crate): `pub fn data_dir() -> PathBuf`, `pub fn load(name: &str) -> serde_json::Value`, `pub fn cases(name: &str) -> Vec<serde_json::Value>`.

- [ ] **Step 1: Write the failing Python test**

`tools/tests/test_gen_oracles.py`:
```python
import importlib.util
import json

from tools import v1


def _gen():
    spec = importlib.util.spec_from_file_location("gen_oracles", v1.ROOT / "tests" / "oracle" / "gen_oracles.py")
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    return mod


def test_pyfmt_cases_are_cpython_results():
    gen = _gen()
    data = json.loads(gen.render("pyfmt"))
    by_x = {c["x"]: c for c in data["cases"]}
    assert by_x[2.5]["round"] == 2
    assert by_x[0.125]["round2"] == 0.12
    assert by_x[2.675]["round2"] == 2.67
    assert by_x[999960.0]["thousands0"] == "999,960"
    assert by_x[-0.4]["thousands0"] == "-0"
    assert len(data["cases"]) > 600


def test_rendering_is_deterministic():
    gen = _gen()
    assert gen.render("pyfmt") == gen.render("pyfmt")


def test_committed_oracles_are_current():
    assert _gen().check() == []
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `.venv/Scripts/python -m pytest tools/tests/test_gen_oracles.py`
Expected: FAIL, `FileNotFoundError` for `tests/oracle/gen_oracles.py`.

- [ ] **Step 3: Implement the generator**

`tests/oracle/gen_oracles.py`:
```python
"""Golden outputs ("oracles") from v1 and CPython, for Rust/TS parity tests.

python tests/oracle/gen_oracles.py               write every set to tests/oracle/data/<name>.json
python tests/oracle/gen_oracles.py --only NAME   write one set
python tests/oracle/gen_oracles.py --check       exit 1 if a committed set differs (CI)

Each milestone adds its sets here (roadmap rule 1). Inputs are fixed or seeded: output must be byte-stable.
"""
from __future__ import annotations

import argparse
import json
import os
import random
import sys
import tempfile
from pathlib import Path
from typing import Callable

ROOT = Path(__file__).resolve().parent.parent.parent
sys.path.insert(0, str(ROOT))
os.environ.setdefault("APPDATA", tempfile.mkdtemp(prefix="mh-v1-appdata-"))
os.environ.setdefault("MAPLEHELPER_NO_TELEMETRY", "1")
from tools.v1 import import_path  # noqa: E402,F401  (later sets call import_path() before importing maplehelper)

DATA = ROOT / "tests" / "oracle" / "data"
SETS: dict[str, Callable[[], dict]] = {}


def oracle(name: str):
    def register(fn: Callable[[], dict]):
        SETS[name] = fn
        return fn
    return register


@oracle("pyfmt")
def pyfmt() -> dict:
    """CPython round()/format() on floats: ties, -0.0, inexact binaries, large values, v1 _num boundaries."""
    rng = random.Random(20261012)
    xs = [0.0, -0.0, 0.5, 1.5, 2.5, -0.5, -1.5, -2.5, -0.4, 0.125, 0.375, 2.675, 1.005, 0.045,
          99999.5, 100000.0, 999940.0, 999950.0, 999960.0, 123400.0, 1234567.891, -1234.5,
          1e15 + 0.5, 2.0 ** 53, 1e-7, 5e-324]
    xs += [rng.uniform(-1e7, 1e7) for _ in range(300)]
    xs += [rng.randint(-10 ** 6, 10 ** 6) / rng.choice([1, 2, 4, 8, 10, 100, 1000]) for _ in range(300)]
    cases = [{
        "x": x,
        "round": round(x),
        "round1": round(x, 1),
        "round2": round(x, 2),
        "round3": round(x, 3),
        "thousands0": format(x, ",.0f"),
        "fixed1": format(x, ".1f"),
        "fixed2": format(x, ".2f"),
    } for x in xs]
    return {"cases": cases}


def render(name: str) -> str:
    return json.dumps(SETS[name](), ensure_ascii=False, indent=1, sort_keys=True) + "\n"


def write(name: str) -> Path:
    DATA.mkdir(parents=True, exist_ok=True)
    path = DATA / f"{name}.json"
    path.write_text(render(name), encoding="utf-8", newline="\n")
    return path


def check() -> list[str]:
    problems = []
    for name in sorted(SETS):
        path = DATA / f"{name}.json"
        if not path.is_file() or path.read_text(encoding="utf-8") != render(name):
            problems.append(f"stale oracle: {name}")
    known = {f"{n}.json" for n in SETS}
    problems += [f"orphan oracle: {p.name}" for p in sorted(DATA.glob("*.json")) if p.name not in known]
    return problems


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--only")
    parser.add_argument("--check", action="store_true")
    args = parser.parse_args()
    if args.check:
        problems = check()
        print("\n".join(problems) or "oracles: OK")
        return 1 if problems else 0
    for name in [args.only] if args.only else sorted(SETS):
        print(write(name).relative_to(ROOT).as_posix())
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

Run: `.venv/Scripts/python tests/oracle/gen_oracles.py`
Expected: prints `tests/oracle/data/pyfmt.json`.

- [ ] **Step 4: Run the Python tests to verify they pass**

Run: `.venv/Scripts/python -m pytest tools/tests/test_gen_oracles.py`
Expected: 3 passed.

- [ ] **Step 5: Write the Rust loader with a failing test first**

`crates/mh-oracle/Cargo.toml`:
```toml
[package]
name = "mh-oracle"
version.workspace = true
edition.workspace = true
publish.workspace = true

[dependencies]
serde_json.workspace = true
```

`crates/mh-oracle/src/lib.rs`:
```rust
//! Loads the golden JSON written by `tests/oracle/gen_oracles.py`. Dev-dependency of every crate.

use std::path::PathBuf;

pub fn data_dir() -> PathBuf {
    PathBuf::from(env!("CARGO_MANIFEST_DIR")).join("../../tests/oracle/data")
}

pub fn load(name: &str) -> serde_json::Value {
    unimplemented!()
}

pub fn cases(name: &str) -> Vec<serde_json::Value> {
    unimplemented!()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn loads_the_pyfmt_cases() {
        let cases = cases("pyfmt");
        assert!(cases.len() > 600);
        assert!(cases[0].get("x").is_some());
    }

    #[test]
    #[should_panic(expected = "oracle no-such-set")]
    fn a_missing_set_names_itself() {
        load("no-such-set");
    }
}
```

Run: `cargo test -p mh-oracle`
Expected: FAIL, `not implemented`.

- [ ] **Step 6: Implement the loader**

Replace the two bodies:
```rust
pub fn load(name: &str) -> serde_json::Value {
    let path = data_dir().join(format!("{name}.json"));
    let text = std::fs::read_to_string(&path)
        .unwrap_or_else(|e| panic!("oracle {name}: cannot read {}: {e}", path.display()));
    serde_json::from_str(&text).unwrap_or_else(|e| panic!("oracle {name}: bad JSON: {e}"))
}

pub fn cases(name: &str) -> Vec<serde_json::Value> {
    load(name)["cases"]
        .as_array()
        .unwrap_or_else(|| panic!("oracle {name}: no \"cases\" array"))
        .clone()
}
```

Run: `cargo test -p mh-oracle`
Expected: 2 passed.

- [ ] **Step 7: Commit**

```bash
git add tests/oracle crates/mh-oracle tools/tests/test_gen_oracles.py Cargo.lock
git commit -m "Oracle pipeline: v1/CPython golden JSON and the Rust loader

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Python float formatting in `mh-text`, proven by the `pyfmt` oracle

**Files:**
- Modify: `crates/mh-text/Cargo.toml`
- Modify: `crates/mh-text/src/lib.rs`
- Modify: `crates/mh-text/src/pyfmt.rs`
- Test: `crates/mh-text/tests/pyfmt_oracle.rs`

**Interfaces:**
- Consumes: `mh_oracle::cases`.
- Produces, in `mh_text`:
  - `py_round(x: f64) -> i64` (from Task 1)
  - `py_round_n(x: f64, ndigits: u32) -> f64`: Python `round(x, n)`
  - `fmt_fixed(x: f64, decimals: u32) -> String`: Python `format(x, f".{n}f")`
  - `fmt_thousands0(x: f64) -> String`: Python `format(x, ",.0f")`

- [ ] **Step 1: Write the failing oracle test**

Add to `crates/mh-text/Cargo.toml`:
```toml
[dev-dependencies]
mh-oracle = { path = "../mh-oracle" }
serde_json.workspace = true
```

`crates/mh-text/tests/pyfmt_oracle.rs`:
```rust
use mh_text::{fmt_fixed, fmt_thousands0, py_round, py_round_n};

#[test]
fn matches_cpython_round_and_format() {
    let mut failures = Vec::new();
    for case in mh_oracle::cases("pyfmt") {
        let x = case["x"].as_f64().unwrap();
        let checks: [(&str, String, String); 7] = [
            ("round", py_round(x).to_string(), case["round"].as_i64().unwrap().to_string()),
            ("round1", format!("{:?}", py_round_n(x, 1)), format!("{:?}", case["round1"].as_f64().unwrap())),
            ("round2", format!("{:?}", py_round_n(x, 2)), format!("{:?}", case["round2"].as_f64().unwrap())),
            ("round3", format!("{:?}", py_round_n(x, 3)), format!("{:?}", case["round3"].as_f64().unwrap())),
            ("thousands0", fmt_thousands0(x), case["thousands0"].as_str().unwrap().to_string()),
            ("fixed1", fmt_fixed(x, 1), case["fixed1"].as_str().unwrap().to_string()),
            ("fixed2", fmt_fixed(x, 2), case["fixed2"].as_str().unwrap().to_string()),
        ];
        for (what, got, want) in checks {
            if got != want {
                failures.push(format!("{what}({x:?}): got {got}, want {want}"));
            }
        }
    }
    assert!(failures.is_empty(), "{} mismatches, first: {:#?}", failures.len(), &failures[..failures.len().min(10)]);
}
```

Note: `{:?}` on f64 distinguishes `-0.0` from `0.0`, which is exactly what Python's `round(-0.04, 1)` → `-0.0` requires.

- [ ] **Step 2: Run the test to verify it fails**

Run: `cargo test -p mh-text --test pyfmt_oracle`
Expected: compile error, `cannot find function py_round_n in crate mh_text`.

- [ ] **Step 3: Implement**

Append to `crates/mh-text/src/pyfmt.rs`:
```rust
/// The exact decimal expansion of |x| as (integer digits, fraction digits). f64 has at most 1074
/// fractional digits, so printing 1100 is exact.
fn exact_digits(x: f64) -> (String, String) {
    let s = format!("{:.1100}", x.abs());
    let (int, frac) = s.split_once('.').expect("fixed-point output has a point");
    (int.to_string(), frac.to_string())
}

/// Round the exact expansion of |x| to `decimals` fraction digits, ties to even (CPython's
/// correctly-rounded `format` and `round(x, n)`). Returns (integer digits, fraction digits).
fn round_half_even(x: f64, decimals: usize) -> (String, String) {
    let (int, frac) = exact_digits(x);
    let (kept, rest) = frac.split_at(decimals.min(frac.len()));
    let mut digits: Vec<u8> = int.bytes().chain(kept.bytes()).map(|b| b - b'0').collect();
    let rest = rest.as_bytes();
    let round_up = match rest.first() {
        Some(b'6'..=b'9') => true,
        Some(b'5') => {
            let above_half = rest[1..].iter().any(|&b| b != b'0');
            above_half || digits.last().is_some_and(|d| d % 2 == 1)
        }
        _ => false,
    };
    if round_up {
        let mut i = digits.len();
        loop {
            if i == 0 {
                digits.insert(0, 1);
                break;
            }
            i -= 1;
            if digits[i] == 9 {
                digits[i] = 0;
            } else {
                digits[i] += 1;
                break;
            }
        }
    }
    let split = digits.len() - decimals;
    let to_str = |ds: &[u8]| ds.iter().map(|d| char::from(b'0' + d)).collect::<String>();
    (to_str(&digits[..split]), to_str(&digits[split..]))
}

/// Python's `round(x, ndigits)` for ndigits >= 0.
pub fn py_round_n(x: f64, ndigits: u32) -> f64 {
    if !x.is_finite() {
        return x;
    }
    let (int, frac) = round_half_even(x, ndigits as usize);
    let text = if frac.is_empty() { int } else { format!("{int}.{frac}") };
    let magnitude: f64 = text.parse().expect("digits parse as f64");
    if x.is_sign_negative() {
        -magnitude
    } else {
        magnitude
    }
}

/// Python's `format(x, f".{decimals}f")`. The sign is kept for negative zero results ("-0.0").
pub fn fmt_fixed(x: f64, decimals: u32) -> String {
    if !x.is_finite() {
        return python_special(x);
    }
    let (int, frac) = round_half_even(x, decimals as usize);
    let sign = if x.is_sign_negative() { "-" } else { "" };
    if frac.is_empty() {
        format!("{sign}{int}")
    } else {
        format!("{sign}{int}.{frac}")
    }
}

/// Python's `format(x, ",.0f")`: rounded half to even, thousands separated by commas.
pub fn fmt_thousands0(x: f64) -> String {
    if !x.is_finite() {
        return python_special(x);
    }
    let (int, _) = round_half_even(x, 0);
    let mut grouped = String::with_capacity(int.len() + int.len() / 3);
    for (i, ch) in int.chars().enumerate() {
        if i > 0 && (int.len() - i) % 3 == 0 {
            grouped.push(',');
        }
        grouped.push(ch);
    }
    let sign = if x.is_sign_negative() { "-" } else { "" };
    format!("{sign}{grouped}")
}

fn python_special(x: f64) -> String {
    if x.is_nan() {
        "nan".into()
    } else if x > 0.0 {
        "inf".into()
    } else {
        "-inf".into()
    }
}
```

Update `crates/mh-text/src/lib.rs`:
```rust
pub use pyfmt::{fmt_fixed, fmt_thousands0, py_round, py_round_n};
```

Add unit tests for the special values (not in the oracle because JSON has no NaN/inf) to the `tests` module in `pyfmt.rs`:
```rust
    #[test]
    fn non_finite_values_format_like_python() {
        assert_eq!(fmt_fixed(f64::NAN, 1), "nan");
        assert_eq!(fmt_fixed(f64::INFINITY, 2), "inf");
        assert_eq!(fmt_thousands0(f64::NEG_INFINITY), "-inf");
        assert!(py_round_n(f64::NAN, 2).is_nan());
    }

    #[test]
    fn carries_across_every_digit() {
        assert_eq!(fmt_fixed(9.999, 2), "10.00");
        assert_eq!(fmt_thousands0(999999.5), "1,000,000");
    }
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cargo test -p mh-text && cargo clippy --workspace --all-targets -- -D warnings`
Expected: all pass, including `matches_cpython_round_and_format` with 0 mismatches.

- [ ] **Step 5: Commit**

```bash
git add crates/mh-text Cargo.lock
git commit -m "mh-text: CPython round() and format() for floats, oracle-verified

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Tauri + React app skeleton (window routing, theme tokens, version sync)

**Files:**
- Modify: `Cargo.toml` (members)
- Create: `package.json`, `tsconfig.json`, `tsconfig.node.json`, `vite.config.ts`, `vitest.config.ts`, `tailwind.config.js`, `postcss.config.js`, `index.html`
- Create: `src/main.tsx`, `src/vite-env.d.ts`, `src/styles/theme.css`
- Create: `src/lib/app.ts`
- Create: `src/windows/registry.tsx`, `src/windows/UnknownWindow.tsx`, `src/windows/chat/ChatWindow.tsx`
- Create: `src-tauri/Cargo.toml`, `src-tauri/build.rs`, `src-tauri/tauri.conf.json`, `src-tauri/capabilities/default.json`, `src-tauri/src/main.rs`, `src-tauri/src/commands.rs`, `src-tauri/icons/*` (generated)
- Test: `src/windows/registry.test.tsx`, `src/lib/app.test.ts`, `src/windows/chat/ChatWindow.test.tsx`, `src-tauri/src/commands.rs` (unit tests)

**Interfaces:**
- Consumes: `assets/fonts/Rubik-Variable.ttf` and `assets/brand/icon-source.png` (Task 3).
- Produces:
  - Rust command `app_version() -> String` (registered as `app_version`).
  - TS `appVersion(): Promise<string>` in `src/lib/app.ts`.
  - `windowKind(label: string): string` and `windowFor(label: string): ComponentType` in `src/windows/registry.tsx`, with registry key `chat`. Later milestones add keys here.
  - CSS tokens on `:root` / `[data-theme="dark"]` (parity 03 §10).
  - Window label `chat`.

- [ ] **Step 1: Frontend toolchain files**

`package.json`:
```json
{
  "name": "maple-helper",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "tauri": "tauri",
    "test": "vitest",
    "test:run": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@tauri-apps/api": "^2.11.1",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "zustand": "^4.5.7"
  },
  "devDependencies": {
    "@tauri-apps/cli": "^2.11.4",
    "@testing-library/jest-dom": "^6.9.1",
    "@testing-library/react": "^16.3.3",
    "@types/node": "^20.10.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^6.0.5",
    "autoprefixer": "^10.4.16",
    "jsdom": "^29.1.1",
    "postcss": "^8.4.32",
    "tailwindcss": "^3.4.19",
    "typescript": "^5.9.3",
    "vite": "^8.2.0",
    "vitest": "^4.1.10"
  }
}
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

`tsconfig.node.json`:
```json
{
  "compilerOptions": {
    "composite": true,
    "skipLibCheck": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts", "vitest.config.ts"]
}
```

`vite.config.ts` (agentrium's, plus `publicDir: 'assets'` so fonts, brand art and guide images are served from the v1 copy):
```ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  clearScreen: false,
  publicDir: 'assets',
  server: { port: 5173, strictPort: true },
  envPrefix: ['VITE_', 'TAURI_'],
  build: {
    target: process.env.TAURI_PLATFORM === 'windows' ? 'chrome105' : 'safari13',
    minify: !process.env.TAURI_DEBUG,
    sourcemap: !!process.env.TAURI_DEBUG,
    modulePreload: { polyfill: false },
  },
});
```

`vitest.config.ts`:
```ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

// Browser-ish code runs under jsdom, pure logic under node (agentrium's split).
const JSDOM_GLOBS = [
  'src/windows/**/*.test.{ts,tsx}',
  'src/components/**/*.test.{ts,tsx}',
  'src/store/**/*.test.{ts,tsx}',
  'src/lib/**/*.test.{ts,tsx}',
];

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    clearMocks: true,
    restoreMocks: true,
    setupFiles: ['./src/test-setup.ts'],
    projects: [
      { extends: true, test: { name: 'jsdom', environment: 'jsdom', include: JSDOM_GLOBS } },
      { extends: true, test: { name: 'node', environment: 'node', include: ['src/**/*.test.{ts,tsx}'], exclude: JSDOM_GLOBS } },
    ],
  },
});
```

`src/test-setup.ts`:
```ts
import '@testing-library/jest-dom/vitest';
```

`tailwind.config.js`:
```js
/** @type {import('tailwindcss').Config} */
export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        orange: { DEFAULT: 'var(--orange)', deep: 'var(--orange-deep)' },
        text: 'var(--text)',
        muted: 'var(--muted)',
        faint: 'var(--faint)',
      },
      fontFamily: { sans: ['Rubik', 'Segoe UI', 'Helvetica Neue', 'sans-serif'] },
      borderRadius: { glass: '22px' },
    },
  },
  plugins: [],
};
```

`postcss.config.js`:
```js
export default { plugins: { tailwindcss: {}, autoprefixer: {} } };
```

`index.html`:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Maple Helper</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

`src/vite-env.d.ts`:
```ts
/// <reference types="vite/client" />
```

`src/styles/theme.css` (tokens from parity 03 §10; light is the v1 default):
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@font-face {
  font-family: 'Rubik';
  src: url('/fonts/Rubik-Variable.ttf') format('truetype');
  font-weight: 300 900;
}

:root {
  --orange: #ff9533;
  --orange-deep: #f07a12;
  --cream: #fff4e6;
  --orange-text: #c9620a;
  --on-orange: #ffffff;
  --good: #2e9e5b;
  --warn: #c9620a;
  --changed: #0a6cd6;
  --danger: #ff3b30;
  --switch-on: #34c759;
  --radius: 22px;
  --shadow: 12px;
  --glass: rgb(242 242 247);
  --text: #1d1d1f;
  --muted: rgba(60, 60, 67, 0.66);
  --faint: rgba(60, 60, 67, 0.42);
  --fill1: #ffffff;
  --fill2: #ffffff;
  --fill3: #e5e5ea;
  --pressed: rgba(230, 230, 235, 0.95);
  --stroke: rgba(0, 0, 0, 0.08);
  --hair: rgba(0, 0, 0, 0.05);
  --dialog-bg: #f2f2f7;
}

[data-theme='dark'] {
  --good: #4cd07d;
  --warn: #ffb066;
  --changed: #64b5ff;
  --glass: rgb(28 28 30);
  --text: #f5f5f7;
  --muted: rgba(235, 235, 245, 0.64);
  --faint: rgba(235, 235, 245, 0.4);
  --fill1: rgba(255, 255, 255, 0.08);
  --fill2: rgba(255, 255, 255, 0.12);
  --fill3: rgba(255, 255, 255, 0.2);
  --pressed: rgba(255, 255, 255, 0.28);
  --stroke: rgba(255, 255, 255, 0.14);
  --hair: rgba(255, 255, 255, 0.07);
  --dialog-bg: #1c1c1e;
}

html,
body,
#root {
  margin: 0;
  height: 100%;
  background: transparent;
  font-family: 'Rubik', 'Segoe UI', 'Helvetica Neue', sans-serif;
  color: var(--text);
}

.glass-window {
  box-sizing: border-box;
  height: calc(100% - 2 * var(--shadow));
  margin: var(--shadow);
  border-radius: var(--radius);
  background: var(--glass);
  box-shadow: 0 3px var(--shadow) rgba(0, 0, 0, 0.28);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
```

Run: `npm install`
Expected: `package-lock.json` created, no errors.

- [ ] **Step 2: Write the failing frontend tests**

`src/windows/registry.test.tsx`:
```tsx
import { windowFor, windowKind } from './registry';
import { ChatWindow } from './chat/ChatWindow';
import { UnknownWindow } from './UnknownWindow';

describe('window registry', () => {
  it('maps a plain label to its kind', () => {
    expect(windowKind('chat')).toBe('chat');
  });

  it('strips per-character and numbered suffixes', () => {
    expect(windowKind('history:a1b2c3d4')).toBe('history');
    expect(windowKind('wishlist:a1b2c3d4')).toBe('wishlist');
    expect(windowKind('toast-3')).toBe('toast');
    expect(windowKind('ov-npc')).toBe('ov-npc');
  });

  it('renders the chat for the chat label', () => {
    expect(windowFor('chat')).toBe(ChatWindow);
  });

  it('falls back harmlessly for labels it does not know', () => {
    expect(windowFor('no-such-window')).toBe(UnknownWindow);
    expect(windowFor('')).toBe(UnknownWindow);
  });
});
```

`src/lib/app.test.ts`:
```ts
import { invoke } from '@tauri-apps/api/core';
import { appVersion } from './app';

vi.mock('@tauri-apps/api/core', () => ({ invoke: vi.fn() }));

describe('appVersion', () => {
  it('asks the backend for app_version', async () => {
    vi.mocked(invoke).mockResolvedValue('0.1.0');
    await expect(appVersion()).resolves.toBe('0.1.0');
    expect(invoke).toHaveBeenCalledWith('app_version');
  });
});
```

`src/windows/chat/ChatWindow.test.tsx`:
```tsx
import { render, screen } from '@testing-library/react';
import { ChatWindow } from './ChatWindow';
import { appVersion } from '../../lib/app';

vi.mock('../../lib/app', () => ({ appVersion: vi.fn() }));

describe('ChatWindow', () => {
  it('shows the product name and the running version', async () => {
    vi.mocked(appVersion).mockResolvedValue('0.1.0');
    render(<ChatWindow />);
    expect(screen.getByText('Maple Helper')).toBeInTheDocument();
    expect(await screen.findByText('v0.1.0')).toBeInTheDocument();
  });

  it('still renders when the version cannot be read', async () => {
    vi.mocked(appVersion).mockRejectedValue(new Error('no backend'));
    render(<ChatWindow />);
    expect(screen.getByText('Maple Helper')).toBeInTheDocument();
    expect(screen.queryByText(/^v\d/)).not.toBeInTheDocument();
  });
});
```

Run: `npm run test:run`
Expected: FAIL, cannot resolve `./registry`, `./app` and `./ChatWindow`.

- [ ] **Step 3: Implement the frontend**

`src/lib/app.ts`:
```ts
import { invoke } from '@tauri-apps/api/core';

/** The running app version (workspace version), e.g. "0.1.0". */
export function appVersion(): Promise<string> {
  return invoke<string>('app_version');
}
```

`src/windows/UnknownWindow.tsx`:
```tsx
/** Rendered for a label no window is registered for: harmless, never a crash. */
export function UnknownWindow() {
  return <div className="glass-window" />;
}
```

`src/windows/chat/ChatWindow.tsx`:
```tsx
import { useEffect, useState } from 'react';
import { appVersion } from '../../lib/app';

export function ChatWindow() {
  const [version, setVersion] = useState('');
  useEffect(() => {
    // best-effort: the footer simply shows no version if the backend cannot answer
    appVersion().then(setVersion).catch(() => {});
  }, []);
  return (
    <div className="glass-window">
      <header className="px-4 py-3 font-medium" data-tauri-drag-region>
        Maple Helper
      </header>
      <main className="flex-1" />
      <footer className="px-4 py-2 text-xs text-muted">{version && `v${version}`}</footer>
    </div>
  );
}
```

`src/windows/registry.tsx`:
```tsx
import type { ComponentType } from 'react';
import { ChatWindow } from './chat/ChatWindow';
import { UnknownWindow } from './UnknownWindow';

const WINDOWS: Record<string, ComponentType> = {
  chat: ChatWindow,
};

/** "history:<cid>" -> "history", "toast-3" -> "toast"; other labels are their own kind. */
export function windowKind(label: string): string {
  return label.split(':')[0].replace(/-\d+$/, '');
}

export function windowFor(label: string): ComponentType {
  return WINDOWS[windowKind(label)] ?? UnknownWindow;
}
```

`src/main.tsx`:
```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { getCurrentWebviewWindow } from '@tauri-apps/api/webviewWindow';
import { windowFor } from './windows/registry';
import './styles/theme.css';

const Window = windowFor(getCurrentWebviewWindow().label);

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Window />
  </React.StrictMode>,
);
```

- [ ] **Step 4: Run the frontend checks to verify they pass**

Run: `npm run typecheck && npm run test:run`
Expected: no type errors; 7 tests pass.

- [ ] **Step 5: Write the Tauri app with a failing version-sync test**

Change `members` in the root `Cargo.toml` to `["crates/*", "src-tauri"]`, and add to `[workspace.dependencies]`:
```toml
tauri = { version = "2.11", features = ["macos-private-api"] }
tauri-build = { version = "2.11", features = [] }
tauri-plugin-single-instance = "2"
```

`src-tauri/Cargo.toml`:
```toml
[package]
name = "maple-helper"
version.workspace = true
edition.workspace = true
publish.workspace = true

[[bin]]
name = "maple-helper"
path = "src/main.rs"

[dependencies]
tauri.workspace = true
tauri-plugin-single-instance.workspace = true
serde.workspace = true
serde_json.workspace = true

[build-dependencies]
tauri-build.workspace = true
```

`src-tauri/build.rs`:
```rust
fn main() {
    tauri_build::build()
}
```

`src-tauri/tauri.conf.json`:
```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "Maple Helper",
  "mainBinaryName": "Maple Helper",
  "version": "0.1.0",
  "identifier": "com.maplehelper.app",
  "build": {
    "beforeDevCommand": "npm run dev",
    "devUrl": "http://localhost:5173",
    "beforeBuildCommand": "npm run build",
    "frontendDist": "../dist"
  },
  "app": {
    "withGlobalTauri": false,
    "macOSPrivateApi": true,
    "windows": [
      {
        "label": "chat",
        "title": "Maple Helper",
        "width": 600,
        "height": 640,
        "decorations": false,
        "transparent": true,
        "alwaysOnTop": true,
        "resizable": true,
        "center": true
      }
    ],
    "security": {
      "csp": "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; font-src 'self'; img-src 'self' data: asset: http://asset.localhost; connect-src ipc: http://ipc.localhost"
    }
  },
  "bundle": {
    "active": true,
    "targets": ["nsis", "dmg", "app"],
    "icon": ["icons/32x32.png", "icons/128x128.png", "icons/128x128@2x.png", "icons/icon.icns", "icons/icon.ico"]
  }
}
```

`src-tauri/capabilities/default.json`:
```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capabilities for Maple Helper windows",
  "windows": ["*"],
  "permissions": [
    "core:default",
    "core:window:default",
    "core:window:allow-start-dragging",
    "core:window:allow-close",
    "core:window:allow-minimize"
  ]
}
```

`src-tauri/src/commands.rs`:
```rust
//! Tauri commands. Each domain gets its own module as milestones land.

/// The running app version, from the workspace version.
#[tauri::command]
pub fn app_version() -> String {
    unimplemented!()
}

#[cfg(test)]
mod tests {
    use super::*;

    fn json(path: &str) -> serde_json::Value {
        let full = std::path::Path::new(env!("CARGO_MANIFEST_DIR")).join(path);
        serde_json::from_str(&std::fs::read_to_string(&full).unwrap()).unwrap()
    }

    #[test]
    fn app_version_is_the_workspace_version() {
        assert_eq!(app_version(), env!("CARGO_PKG_VERSION"));
    }

    #[test]
    fn all_three_version_files_agree() {
        let cargo = env!("CARGO_PKG_VERSION");
        assert_eq!(json("tauri.conf.json")["version"], cargo, "src-tauri/tauri.conf.json");
        assert_eq!(json("../package.json")["version"], cargo, "package.json");
    }
}
```

`src-tauri/src/main.rs`:
```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

mod commands;

use tauri::Manager;

fn main() {
    tauri::Builder::default()
        // registered first (agentrium rule): a second launch shows the chat instead of a second app
        .plugin(tauri_plugin_single_instance::init(|app, _argv, _cwd| {
            if let Some(chat) = app.get_webview_window("chat") {
                let _ = chat.unminimize();
                let _ = chat.show();
                let _ = chat.set_focus();
            }
        }))
        .invoke_handler(tauri::generate_handler![commands::app_version])
        .run(tauri::generate_context!())
        .expect("error while running Maple Helper");
}
```

Generate icons from the v1 brand art:
Run: `npx tauri icon assets/brand/icon-source.png`
Expected: `src-tauri/icons/` contains `32x32.png`, `128x128.png`, `128x128@2x.png`, `icon.icns`, `icon.ico` (plus extras, which are fine).

Run: `cargo test -p maple-helper`
Expected: FAIL, `app_version_is_the_workspace_version` panics with `not implemented`.

- [ ] **Step 6: Implement the command**

```rust
#[tauri::command]
pub fn app_version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}
```

- [ ] **Step 7: Verify everything, including a real debug build**

Run: `cargo test --workspace && cargo clippy --workspace --all-targets -- -D warnings && cargo fmt --all --check`
Expected: all tests pass, including `all_three_version_files_agree`.

Run: `npm run tauri build -- --debug --no-bundle`
Expected: builds `target/debug/Maple Helper.exe` (Windows) or `target/debug/Maple Helper` (macOS).

If the bundler rejects `mainBinaryName` containing a space, stop and report it. v1's executable is `Maple Helper.exe`, and the bridge (M9) relies on that name for shortcuts, so do not rename it silently.

Run the binary, then launch it a second time.
Expected: a 600×640 frameless, transparent, always-on-top window shows "Maple Helper" and "v0.1.0" in Rubik. The second launch focuses the first window instead of opening another. Close it.

- [ ] **Step 8: Commit**

```bash
git add Cargo.toml Cargo.lock package.json package-lock.json tsconfig.json tsconfig.node.json vite.config.ts vitest.config.ts tailwind.config.js postcss.config.js index.html src src-tauri
git commit -m "Tauri + React skeleton: chat window, label routing, theme tokens, version sync

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: CI on Windows, macOS and Linux (Rust, frontend, v1 parity drift)

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/dependabot.yml`

**Interfaces:**
- Consumes: every command from Tasks 1–7.
- Produces: required check names `Rust (windows-latest)`, `Rust (macos-latest)`, `Frontend`, `v1 parity`.

- [ ] **Step 1: Write the workflow**

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  CARGO_HTTP_MULTIPLEXING: 'false'
  CARGO_NET_RETRY: '10'

jobs:
  rust:
    name: Rust (${{ matrix.os }})
    strategy:
      fail-fast: false
      matrix:
        os: [windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: dtolnay/rust-toolchain@efa25f7f19611383d5b0ccf2d1c8914531636bf9 # master
        with:
          toolchain: stable
          components: rustfmt, clippy
      - uses: swatinem/rust-cache@c19371144df3bb44fab255c43d04cbc2ab54d1c4 # v2.9.1
        with:
          shared-key: ci-test
      - name: Format
        run: cargo fmt --all --check
      - name: Fetch
        shell: bash
        run: |
          for attempt in 1 2 3; do
            cargo fetch --locked && exit 0
            echo "cargo fetch failed (attempt $attempt), retrying in 15s"; sleep 15
          done
          exit 1
      - name: Clippy
        run: cargo clippy --workspace --all-targets --locked --offline -- -D warnings
      - name: Test
        run: cargo test --workspace --locked --offline

  frontend:
    name: Frontend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e # v6.4.0
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run typecheck
      - run: npm run test:run

  parity:
    name: v1 parity
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      - uses: actions/setup-python@v6
        with:
          python-version: '3.13'
          cache: pip
          cache-dependency-path: tools/requirements.txt
      - run: python -m pip install -r tools/requirements.txt
      - name: Tool tests
        run: python -m pytest
      - name: v1 files unchanged
        run: python tools/import_v1_files.py --check
      - name: v1 extraction current
        run: python tools/extract_v1.py --check
      - name: Oracles current
        run: python tests/oracle/gen_oracles.py --check
```

`.github/dependabot.yml` (also pins `actions/setup-python` to a SHA on its first PR):
```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: cargo
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: npm
    directory: /
    schedule: { interval: weekly }
```

- [ ] **Step 2: Push and watch CI**

```bash
git add .github
git commit -m "CI: Rust on Windows and macOS, frontend, v1 parity drift checks

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push origin main
gh run watch --exit-status "$(gh run list --workflow ci.yml --limit 1 --json databaseId -q '.[0].databaseId')"
```
Expected: all four jobs pass.

If `v1 parity` fails only on Linux, the likely cause is a platform-dependent v1 constant: a path, or something chosen by `sys.platform`. Open the `stale:` file named in the log and compare it with the Windows output. Then make the extractor drop that constant by adding its name to a `PLATFORM_DEPENDENT = {"<module>": {"<NAME>", ...}}` filter applied in `_render`, regenerate on Windows, and push. Record each dropped name in the commit message.

- [ ] **Step 3: Mark M0 done in the roadmap**

In `docs/superpowers/plans/2026-10-12-roadmap.md`, change the M0 row's status to `done (<merge commit sha>)` and M1's to `plan next`.

```bash
git add docs/superpowers/plans/2026-10-12-roadmap.md
git commit -m "Roadmap: M0 done

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
git push origin main
```
