# miraCLI — Development Strategy

> Status: **DRAFT for review**. Nothing is implemented yet. M0 starts only after the open questions in §2 are answered (or the suggested defaults are accepted) and the plan is approved.

---

## 1. Understanding and assumptions

### 1.1 The product in my own words

miraCLI is a single full-screen, keyboard-only TUI that acts as a launchpad and "home screen" for a Linux terminal user. It opens on a main menu. Each entry is either:

- a **built-in tool** that runs inside the TUI (file browser and file management, git clone form, env-var manager, LLM chat, settings), or
- a **launcher** that hands work to an external program the user already prefers (editor, terminal emulator, network tool, any custom command).

The value is not in reinventing these tools but in **one consistent, discoverable keyboard surface** over them: the same keys everywhere, a visible hint bar, safe defaults (confirm, trash, back up), and quick access to the current directory as shared context between tools (the browser's directory is the default for "open terminal here", "clone here", "attach to chat"…).

Architecturally it is a thin TUI shell over UI-independent, unit-tested core services, with each menu entry being a self-registering feature module so that adding a tool or a custom command is cheap.

### 1.2 Assumptions

| # | Assumption | Impact if wrong |
|---|---|---|
| A1 | Single developer (you), part-time; a "stage" ≈ 1 week ≈ 6–8 focused hours. | Stage sizes/timeline must be recalibrated (Q22). |
| A2 | Python distribution name `miracli`, import package `miracli`, console command `miraCLI` (plus lowercase alias `miracli`, see Q1). | Rename cost is low only before M0. |
| A3 | Config dir is `$XDG_CONFIG_HOME/miraCLI/` (default `~/.config/miraCLI/`), state in `$XDG_STATE_HOME/miraCLI/`, data in `$XDG_DATA_HOME/miraCLI/`. The `~/.config/thub/` path in §3.4 of the brief is a leftover of an earlier name and means `~/.config/miraCLI/`. | Q18. |
| A4 | Target is Ubuntu 24.04, bash, Python 3.10–3.13. Ubuntu 24.04 ships Python 3.12, so 3.12 is the reference interpreter; 3.10 is supported because the brief requires it (implies `tomli` backport, no `tomllib`). | — |
| A5 | Terminal emulators of interest: GNOME Terminal (Ubuntu 24.04 default), kitty, Alacritty, WezTerm, Konsole, xterm, tmux/screen inside them, and SSH sessions. 256-colour or truecolour; UTF-8 locale. | Non-UTF-8 locales get ASCII fallbacks only in M7. |
| A6 | Keyboard only. **Mouse capture is disabled by default** so that native terminal text selection/copy keeps working (Q21). | — |
| A7 | Browser navigation is **logical** (like `cd` in bash): entering a symlinked dir keeps the symlink path in the header; Left goes to the logical parent, not the realpath parent. | Changes cursor-restore logic slightly. |
| A8 | v1 operates on **one selected entry** at a time (no multi-select). Multi-select is a backlog candidate (Q9). | Clipboard model is designed as a list anyway, so adding it later is cheap. |
| A9 | Only one long-running file operation at a time (no job queue in v1). Others are blocked with a message until it ends or is cancelled. | — |
| A10 | Network management = launching `nmtui` (or configured tool). No NetworkManager D-Bus integration. | — |
| A11 | Git is installed (`git` ≥ 2.34, Ubuntu 24.04 has 2.43). miraCLI never stores git credentials; it relies on git's own credential helpers / ssh-agent. | — |
| A12 | The LLM endpoint is OpenAI-compatible (OpenAI, or local servers such as Ollama / llama.cpp server / vLLM / LM Studio via `base_url`). | — |
| A13 | No telemetry, no network access except git clone and LLM calls the user triggers. | — |
| A14 | English-only UI for v1; strings kept in one place so i18n is possible later. | — |
| A15 | Repository `lando1784/miraCLI` on GitHub; GitHub Actions is available for CI; releases are git tags installed via `pipx install git+https://github.com/lando1784/miraCLI@vX.Y.Z` (PyPI publishing is optional, Q2). | — |
| A16 | Each milestone gate ends with a tag: M0 → `v0.1.0`, M1 → `v0.2.0`, … M7 → `v1.0.0`. | — |

---

## 2. Open questions

Prioritized by the milestone they block. Each has a suggested default; if you just reply "defaults OK" I will proceed with them.

| # | Question | Blocks | Suggested default |
|---|---|---|---|
| **Q1** | Command name. Linux commands are case-sensitive; `miraCLI` is awkward to type. Install `miraCLI` only, `miracli` only, or both? | M0 | Install **both** `miracli` and `miraCLI` (same entry point). Docs use `miracli`. |
| **Q2** | License and distribution: license? Publish to PyPI or only `pipx install git+…`? | M0 | MIT; git-only install until v1.0, PyPI decision at G7. |
| **Q3** | Mouse: disable mouse capture (keeps native terminal select/copy) or enable mouse as a bonus? | M0 | Disabled by default, `ui.mouse = false` config switch. |
| **Q4** | Logging: log file on by default? | M0 | On, level `WARNING`, rotating `~/.local/state/miraCLI/miracli.log` (1 MB × 3). `--log-level debug` CLI flag. Secrets redacted. |
| **Q5** | Custom menu entries (§3.8): accept the proposed schema (argv list or shell string, `mode = tui | gui | capture`, placeholders `{cwd}`, `{selected}`, `{home}`)? | M0 (S0.3) | Yes, as proposed in §4.6. |
| **Q6** | GUI launches when there is no display (SSH, TTY): error, or fallback to `tmux new-window` when inside tmux? | M0 (S0.3) | Inside tmux: fallback to `tmux new-window -c {cwd}` for the terminal entry. Otherwise a clear error explaining why. |
| **Q7** | Default terminal emulator detection order. | M0 (S0.3) | `$TERMINAL` → `x-terminal-emulator` (Debian alternatives) → `gnome-terminal` → `kitty` → `alacritty` → `wezterm` → `konsole` → `xterm`. Result written to config on first run. |
| **Q8** | Vim-style aliases (`h j k l`, `g`/`G`) in lists in addition to arrows? | M1 | Yes, on by default, `ui.vim_keys = true`. |
| **Q9** | Multi-select (mark with Space, act on all marked) in v1? | M2 | No; backlog for after v1.0. Clipboard is designed as a list so it is cheap later. |
| **Q10** | Delete semantics: freedesktop Trash by default with a separate "Delete permanently" action, or permanent only? | M2 | Trash by default (`send2trash`); "Delete permanently" is a separate context-menu item with a stronger confirmation. |
| **Q11** | Paste placement and directory actions as proposed in §5.4 (Paste item in context menu when clipboard is non-empty; `a` opens a "current directory" menu with New file / New folder / Paste here / Open terminal here; `p` pastes into the current directory)? Add **New folder**? | M2 | Yes to all, including New folder (on folders and in the directory menu). |
| **Q12** | Folder-on-folder conflict on paste: offer **Merge** in v1, or only Replace / Skip / Rename? | M2 | No merge in v1: Replace (existing goes to Trash first), Skip, Keep both (auto-rename), Cancel. |
| **Q13** | "Open with default app" (`xdg-open`) as an extra context-menu item for files? | M2 | Yes, small; GUI-detached launch via the launcher service. |
| **Q14** | Git hosts with nested owners (GitLab subgroups `group/sub`): allow `/` in Owner? | M3 | Yes; owner may contain `/`, each segment validated. |
| **Q15** | Git auth: accept the "streamed first, hand over the terminal on auth failure" strategy (§4.5)? | M3 | Yes, `git.clone_mode = "auto"`. |
| **Q16** | After a successful clone: jump to the browser in the new folder automatically, or ask? | M3 | Ask with a focused "Open in browser" button (Enter = yes, Esc = back to form). |
| **Q17** | Env-var manager design option (§3.4 options a–d) and its sub-questions. | M4 build | Decided at gate G4.D; my recommendation is option (b) (§3.3 below). |
| **Q18** | Confirm `~/.config/thub/` in the brief means `~/.config/miraCLI/`. | M4 | Yes. |
| **Q19** | LLM chat extras (persisted conversations, system prompt per profile, attach file/path, multiple profiles) and send-key choice. | M5 build | Decided at gate G5.D; recommendations in §3.4. |
| **Q20** | Start directory: launch directory (spec) or "remember last browser directory"? | M1 | Launch directory; `browser.start_dir = "launch" | "last" | "<path>"`. |
| **Q21** | On quit, should the parent shell `cd` into the browser's last directory (needs a tiny shell function, like `ranger`/`lf` do)? | M7 | Offer it in M7 as an opt-in snippet (`miracli --print-shell-hook bash`). |
| **Q22** | Realistic weekly time budget? | All | 6–8 h/week (stage = 1 week). |

---

## 3. Technology evaluation

### 3.1 TUI framework

| Criterion | **Textual** | prompt_toolkit | urwid | curses (stdlib) |
|---|---|---|---|---|
| Keyboard handling | Declarative `BINDINGS` per App/Screen/Widget with priority & inheritance; arrows/Tab/Esc/Fn/ctrl combos; key events for custom handling. | Excellent and very low-level (`KeyBindings`, vi/emacs modes, key sequences). | Good: `keypress()` chain per widget. | Raw key codes; you handle escape sequences, Alt, resize yourself. |
| **Key-hint bar** | Built in: `Footer` widget renders the active bindings of the focused screen automatically. | Hand-made toolbar. | Hand-made. | Hand-made. |
| Lists / large lists | `DataTable` (virtualized rows, columns = metadata column for free), `OptionList`, `ListView`. | `RadioList`/custom `FormattedTextControl`; virtualization by hand. | `ListBox` + `ListWalker` — genuinely virtualized, good for huge dirs. | By hand. |
| Forms | `Input` with validators, suggesters (autocomplete from history), placeholders; `Select`, `Checkbox`, `TextArea` (multiline). | `TextArea`, `Dialog`, `Checkbox`; layout via `HSplit`/`VSplit`. Workable but verbose. | `Edit`, `CheckBox`, `Columns`/`Pile`. Workable, dated look. | By hand. |
| Modal menus / dialogs | `ModalScreen` with typed result via `push_screen(..., callback)` or `push_screen_wait`. | `Float` containers; OK. | `Overlay`; OK. | By hand. |
| Async (streaming, long ops) | asyncio-native; `@work` workers (async or thread), `call_from_thread`, cancellation via worker groups. | asyncio-native. | Has an asyncio event loop adapter; less ergonomic. | None; threads + manual select loop. |
| **Suspend for external TUI app** | `with app.suspend(): subprocess.run([...])` restores alt-screen, raw mode and repaints (Unix). | `run_in_terminal()` — excellent. | `screen.stop()` / `screen.start()` manually. | `curses.endwin()` / `refresh()` — native. |
| Resize / small terminals | Automatic reflow, CSS-like layout (`min-width`, docking). | Automatic for its containers. | Automatic. | `SIGWINCH`/`KEY_RESIZE` by hand. |
| Testability | `App.run_test()` + `Pilot` (headless, press keys, await), `pytest-textual-snapshot` SVG snapshots. | `create_pipe_input()` + `DummyOutput`; works, little tooling. | Render canvases to text; no pilot. | Practically untestable without a pty harness. |
| Maturity / activity | Actively developed by Textualize, large community, frequent releases (API occasionally changes → pin). | Very mature, stable (IPython, ptpython). | Very mature, revived maintenance, smaller community. | Stable, minimal. |
| Dependencies / startup | `rich`, `markdown-it-py`, `platformdirs`, `typing-extensions`. Import cost is the heaviest of the four (order of 100–200 ms; **must be measured in S0.1**). | `wcwidth` only; fast import. | Tiny. | None; fastest. |
| Markdown rendering (LLM replies) | `Markdown` widget built in, supports incremental appends. | None. | None. | None. |
| Windows/macOS later | Supported (suspend support on Windows to verify if ever needed). | Supported. | Partial on Windows. | Needs `windows-curses`. |

**Recommendation: Textual.** It is the only option where the brief's non-trivial requirements come for free: automatic key-hint footer from bindings, modal screens returning typed results, a virtualized table with optional columns, validated inputs with history suggestions, multiline `TextArea`, `Markdown` for chat, asyncio workers for clone/copy/stream, `App.suspend()` for TUI hand-over, and a real headless test harness. With curses or urwid most of the first three milestones would be widget-building rather than product-building.

The one real risk is **startup time vs. the 300 ms budget**. Mitigations: lazy-import every feature screen and heavy dependency (openai, pydantic if used for config, tomlkit, send2trash) until first use; measure with `python -X importtime -m miracli --version` and a time-to-first-paint probe in CI. **Runner-up: prompt_toolkit** — if the spike shows Textual cannot get under ~300 ms even with lazy imports, we switch at gate G0.1 before any feature code exists.

### 3.2 Other choices

| Choice | Options considered | Recommendation & why |
|---|---|---|
| Config format | **TOML**, YAML, JSON | **TOML**: Python-native read (`tomllib`, `tomli` on 3.10), no "Norway problem"/implicit typing, comments allowed, the format of `pyproject.toml` so familiar. YAML needs PyYAML/ruamel and has typing footguns. |
| Config read/write | `tomllib`/`tomli` (read-only), **`tomlkit`** (round-trip), `tomli-w` (write, loses comments) | Read with `tomllib`/`tomli` at startup (fast); write with **`tomlkit`** (lazy import) so the in-app settings screen preserves the user's comments and formatting. |
| Config validation | **pydantic v2**, msgspec, attrs+cattrs, jsonschema, hand-rolled dataclasses | **pydantic v2**: already a transitive dependency of `openai` (zero added weight), precise error locations (`apps.editor.command: Input should be a valid list`), defaults, JSON Schema export (lets `taplo`/VS Code autocomplete the TOML). **Fallback: msgspec** if its import cost breaks the startup budget (decided in S0.1 with measurements). |
| XDG paths | `platformdirs`, hand-rolled | **`platformdirs`** (already a Textual dependency), with `appname="miraCLI"`; honours `XDG_*` on Linux and gives the macOS/Windows paths for free later. |
| Trash | **`send2trash`**, `gio trash` subprocess, own implementation of the freedesktop spec | **`send2trash`**: implements the freedesktop Trash spec including per-volume `$topdir/.Trash-$UID`, no GLib dependency, works over SSH. Wrapped behind a `TrashAdapter` so `gio` can be swapped in. |
| LLM | **`openai`** SDK (`AsyncOpenAI`) | Required by the brief; streaming via `chat.completions.create(stream=True)`; `base_url` covers local servers. |
| Secrets | env var, **`keyring`** (optional extra) | API key from env var by default; `keyring` behind an optional extra `miracli[keyring]` (Secret Service needs D-Bus, often absent over SSH). |
| Packaging | **hatchling** + `pyproject.toml`; dev with **uv** | PEP 621 metadata, `[project.scripts]`; `pipx install git+…` works with any PEP 517 backend. `uv sync` / `uv run` for a fast reproducible dev env (`uv.lock` committed). |
| Lint/format | **ruff** (format + lint), black+flake8 | ruff only: one tool, fast. Rule sets: `E,F,W,I,B,UP,SIM,RUF,PTH,ASYNC`. |
| Types | **mypy** `--strict` on `miracli.core` and `miracli.platform`, normal on UI; pyright | mypy: mature, good pydantic plugin. Strict where logic lives, lenient on Textual glue. |
| Tests | **pytest**, `pytest-asyncio`, `pytest-textual-snapshot`, `respx`, `pytest-cov` | See §8. |
| CI | **GitHub Actions** | Matrix Python 3.10/3.11/3.12/3.13 on `ubuntu-24.04`. |
| Pre-commit | **pre-commit** with ruff + mypy hooks | Same checks locally as in CI. |

### 3.3 Env-var manager options (for the M4 design gate — not decided yet)

| Option | How it works | Pros | Cons |
|---|---|---|---|
| (a) Edit `export` lines in `~/.bashrc` | Parse lines matching `^\s*export NAME=…`, edit in place. | No new files; user sees everything in one place. | Fragile: `.bashrc` contains conditionals, functions, multi-line values, `export A B`, `declare -x`; risk of corrupting the user's main shell file; hard to know which line "wins". |
| **(b) Managed file sourced from `~/.bashrc`** | miraCLI owns `~/.config/miraCLI/env.sh`; a one-time guarded block is appended to `~/.bashrc` (with backup) that sources it. | miraCLI fully owns a file with a strict format it generates; `.bashrc` touched exactly once; trivially removable; easy to test. | Variables already defined elsewhere are not editable (only shown read-only); one more file. |
| (c) `~/.bashrc.d/miracli.sh` drop-in | Same as (b) but in a drop-in dir. Ubuntu 24.04's default `.bashrc` does **not** source `~/.bashrc.d/` (Fedora does), so it still needs the one-time hook. | Conventional for people who already use `.bashrc.d`. | Equivalent to (b) with a different path; may surprise users who have no `.bashrc.d`. |
| (d) `~/.config/environment.d/*.conf` (systemd user env) | `KEY=value` files read by systemd at login. | Affects GUI apps too. | Only after re-login, not new bash shells; limited expansion semantics; not what the brief asks. |

**Recommendation: (b)**, with a structured source of truth `~/.config/miraCLI/env.toml` and a **generated** `env.sh` (header: "generated by miraCLI, edits will be overwritten"), with location configurable so (c) is just a different path. Sketch of the hook:

```bash
# >>> miraCLI env (managed, do not edit this block) >>>
[ -f "$HOME/.config/miraCLI/env.sh" ] && . "$HOME/.config/miraCLI/env.sh"
# <<< miraCLI env <<<
```

Topics the discovery stage must settle with you: literal vs expanding values (`shlex.quote` vs a double-quoted form with `$`-references allowed); PATH-like idempotent prepend/append (`case ":$PATH:" in *":/x:"*) ;; *) PATH="/x:$PATH";; esac`); ordering between variables; how to show variables defined elsewhere (static scan of `~/.bashrc`, `~/.profile`, `~/.bash_profile` for `export` lines → shown read-only with `file:line`; plus the inherited process env); `bash -n` syntax check of the generated file before replacing it atomically; backups (`~/.bashrc.miracli-bak-YYYYmmdd-HHMMSS`, keep 5); "changes only affect new shells" notice with an "Open new terminal" shortcut. The full question list is in stage D4.

### 3.4 LLM chat extras (for the M5 design gate — not decided yet)

Minimum (from the brief): configurable `base_url`/`model`/API-key source, streaming, scrollable history, multiline input, clear/new chat. Optional extras with my opinion:

| Extra | Cost | My recommendation |
|---|---|---|
| Multiple provider profiles (OpenAI, local Ollama, …) | S — the config schema is a list of profiles anyway | **Accept** (makes `base_url` config natural). |
| System prompt per profile | S | **Accept**. |
| Attach selected file / current path as context | M — size limits, binary detection, token budget | Accept, limited to text files ≤ N KB, explicit confirmation showing what is sent. |
| Persisted conversations (`$XDG_DATA_HOME/miraCLI/chats/*.json`) | M — list/resume/delete UI | Defer to post-v1 unless you want it; privacy-sensitive. |
| Markdown rendering of replies | S with Textual `Markdown` | Accept. |
| Copy last reply / code block to clipboard (OSC 52) | S | Optional. |

Send key: terminals cannot reliably distinguish Shift+Enter or Ctrl+Enter from Enter. Options: (1) **Enter sends, Alt+Enter / Ctrl+J inserts newline** (chat-like); (2) Enter inserts newline, Ctrl+S sends (editor-like). Default proposal: (1), configurable.

---

## 4. Architecture

### 4.1 Component overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│ cli.py  (argparse: --config, --start-dir, --log-level, --check-config,   │
│          --version)  ──►  bootstrap: paths, logging, config, registry    │
└───────────────────────────────┬──────────────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ UI layer (Textual) — thin                                                │
│  MiraApp ── MainMenuScreen ◄── FeatureRegistry (ordered, filtered by cfg)│
│     │                                                                    │
│     ├─ features/browser   (BrowserScreen, ContextMenu, dialogs)          │
│     ├─ features/git_clone (CloneFormScreen, CloneProgress)               │
│     ├─ features/terminal, network, custom  (launch-only, no screen)      │
│     ├─ features/env       (TBD)                                          │
│     ├─ features/chat      (TBD)                                          │
│     ├─ features/settings  (SettingsScreen, MenuEditor)                   │
│     └─ ui/widgets: KeyHints(Footer), Confirm, Prompt, Progress, Error    │
│                    ui/handover.py: TextualHandover (implements           │
│                    TerminalHandover via App.suspend)                     │
└──────────┬──────────────────────────────────────────────┬────────────────┘
           │ calls (sync or async), gets dataclasses back │ events/progress
           ▼                                              │ callbacks
┌──────────────────────────────────────────────────────────────────────────┐
│ AppContext (service container, passed to every feature)                  │
│   config: ConfigStore   fs: FsService   ops: FileOps   clip: Clipboard   │
│   launcher: Launcher    git: GitService  env: EnvStore  llm: ChatClient  │
│   history: MruStore     state: SharedState (browser cwd, selection)      │
└──────────┬───────────────────────────────────────────────────────────────┘
           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Core (pure Python, no Textual import, unit-tested)                       │
│   core/config  core/fs  core/git  core/launch  core/env  core/llm        │
│   core/history core/errors                                               │
└──────────┬───────────────────────────────────────────────────────────────┘
           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Platform adapters (OS-specific, behind Protocols)                        │
│   platform/linux.py: LinuxTrash(send2trash), LinuxDisplay(DISPLAY/       │
│   WAYLAND_DISPLAY/SSH detection), TerminalTemplates (gnome-terminal,     │
│   kitty, alacritty, wezterm, konsole, xterm, tmux), ProcessSpawner       │
│   platform/detect.py: picks the adapter set for sys.platform             │
└──────────────────────────────────────────────────────────────────────────┘
```

Rules enforced in review (and by an import-linter contract in CI, `import-linter` package): `miracli.core` and `miracli.platform` must never import `textual`; features talk to core only through `AppContext`.

### 4.2 Package / directory layout

```
miraCLI/
├── pyproject.toml              # hatchling, [project.scripts] miracli/miraCLI, ruff, mypy, pytest cfg
├── uv.lock
├── README.md  CHANGELOG.md  LICENSE
├── .pre-commit-config.yaml
├── .github/workflows/ci.yml
├── docs/
│   ├── development-plan.md     # this file
│   ├── adr/0001-tui-framework.md, 0002-config-format.md, 0003-launch-strategy.md, …
│   ├── config-reference.md     # generated from the pydantic schema in M7
│   └── keybindings.md
├── src/miracli/
│   ├── __init__.py  __main__.py  cli.py  app.py  context.py
│   ├── core/
│   │   ├── errors.py           # MiraError hierarchy with user_message + hint
│   │   ├── paths.py            # XDG dirs via platformdirs
│   │   ├── logging.py          # rotating file handler + secret redaction filter
│   │   ├── history.py          # MRU store (state/history.json)
│   │   ├── config/  schema.py defaults.py loader.py store.py migrate.py
│   │   ├── fs/      listing.py ops.py conflicts.py clipboard.py progress.py names.py
│   │   ├── launch/  spec.py launcher.py placeholders.py
│   │   ├── git/     url.py clone.py progress_parser.py errors.py
│   │   ├── env/     (M4, shape decided at G4.D)
│   │   └── llm/     profiles.py client.py conversation.py
│   ├── platform/  base.py linux.py detect.py
│   ├── features/
│   │   ├── registry.py         # FeatureSpec, FeatureRegistry, built-in + entry-point discovery
│   │   ├── browser/  feature.py screen.py table.py context_menu.py dialogs.py
│   │   ├── git_clone/ feature.py form.py progress.py
│   │   ├── terminal/ feature.py   network/ feature.py   custom/ feature.py
│   │   ├── env/ …  chat/ …  settings/ …
│   └── ui/
│       ├── handover.py  theme.tcss  hints.py
│       └── widgets/ confirm.py prompt.py progress_modal.py error_modal.py key_hints.py
└── tests/
    ├── unit/        # core, no Textual
    ├── integration/ # real fs on tmp_path, real git with local bare repos, subprocess
    ├── tui/         # Textual Pilot tests + snapshots
    └── conftest.py
```

### 4.3 Key interfaces (signatures only)

```python
# features/registry.py
class FeatureKind(Enum): SCREEN = "screen"; ACTION = "action"

@dataclass(frozen=True)
class FeatureSpec:
    id: str                         # stable key used in config: "browser", "git_clone", "custom:htop"
    label: str                      # default label (overridable in config)
    hotkey: str | None              # mnemonic on the main menu, e.g. "f"
    kind: FeatureKind
    open: Callable[["AppContext"], "Screen | Awaitable[None]"]   # lazy: imports the screen module inside
    available: Callable[["AppContext"], "Availability"] = always_available
    settings_section: type[BaseModel] | None = None              # contributes to the settings screen (M6)

class FeatureRegistry(Protocol):
    def register(self, spec: FeatureSpec) -> None: ...
    def menu_entries(self, menu_cfg: "MenuConfig") -> list["MenuEntry"]: ...   # applies order/hidden/labels
    # discovers built-ins + Python entry points group "miracli.features" (post-v1 plugins)

# core/launch/spec.py + launcher.py
class AppKind(StrEnum): TUI = "tui"; GUI = "gui"; CAPTURE = "capture"

@dataclass(frozen=True)
class AppSpec:
    command: list[str] | None       # argv template, e.g. ["nvim", "{file}"]
    shell: str | None               # alternative: run via ["bash", "-lc", shell]
    kind: AppKind
    cwd: str = "{cwd}"
    env: Mapping[str, str] = field(default_factory=dict)

class TerminalHandover(Protocol):          # implemented by ui/handover.py with App.suspend()
    def suspended(self) -> ContextManager[None]: ...

@dataclass(frozen=True)
class LaunchResult:
    ok: bool; exit_code: int | None; output: str | None; error: "MiraError | None"

class Launcher(Protocol):
    async def launch(self, spec: AppSpec, ctx: Mapping[str, str]) -> LaunchResult: ...
    # ctx = placeholder values {cwd, file, selected, home}; substitution per argv element, never via a shell

# core/fs/listing.py
@dataclass(frozen=True)
class Entry:
    name: str                       # raw os name (surrogateescape-safe)
    display_name: str               # sanitized for display (control chars escaped, invalid UTF-8 replaced)
    path: Path
    kind: Literal["dir", "file", "symlink_dir", "symlink_file", "broken_symlink", "other"]
    hidden: bool
    size: int | None; mtime: float | None   # filled lazily when the metadata column is on

class FsService(Protocol):
    def list_dir(self, path: Path, *, show_hidden: bool, with_stat: bool) -> list[Entry]: ...  # sorted
    def parent(self, path: Path) -> Path: ...                                                  # logical

# core/fs/ops.py
class ConflictPolicy(StrEnum): ASK = "ask"; OVERWRITE = "overwrite"; SKIP = "skip"; KEEP_BOTH = "keep_both"

class ConflictResolver(Protocol):          # UI implements it with a modal; tests with a stub
    async def resolve(self, src: Path, dst: Path) -> tuple[ConflictPolicy, bool]: ...   # (choice, apply_to_all)

@dataclass
class Progress:
    files_done: int; files_total: int | None; bytes_done: int; bytes_total: int | None; current: Path | None

class FileOps(Protocol):
    def copy(self, srcs: Sequence[Path], dst_dir: Path, *, resolver: ConflictResolver,
             progress: Callable[[Progress], None], cancel: threading.Event) -> "OpReport": ...
    def move(self, srcs: Sequence[Path], dst_dir: Path, *, resolver: ConflictResolver,
             progress: Callable[[Progress], None], cancel: threading.Event) -> "OpReport": ...
    def delete(self, paths: Sequence[Path], *, permanent: bool) -> "OpReport": ...
    def rename(self, path: Path, new_name: str) -> Path: ...
    def create_file(self, parent: Path, name: str) -> Path: ...
    def create_dir(self, parent: Path, name: str) -> Path: ...
    def count(self, path: Path, cancel: threading.Event) -> tuple[int, int]: ...  # (files, bytes) for confirm/progress

class Clipboard(Protocol):
    mode: Literal["copy", "cut"] | None
    items: tuple[Path, ...]
    def set(self, mode: Literal["copy", "cut"], items: Sequence[Path]) -> None: ...
    def clear(self) -> None: ...

# core/git
@dataclass(frozen=True)
class CloneRequest:
    site: str; owner: str; repo: str; branch: str | None; dest_parent: Path

def build_remote_url(req: CloneRequest) -> str: ...        # https://host/owner/repo.git | git@host:owner/repo.git
def validate(req: CloneRequest) -> list["FieldError"]: ... # pure; used for live form validation

class GitService(Protocol):
    async def clone(self, req: CloneRequest, *, on_progress: Callable[["CloneProgress"], None],
                    interactive: bool = False) -> "CloneResult": ...   # cancellable asyncio task

# core/config/store.py
class ConfigStore(Protocol):
    path: Path
    def load(self) -> tuple["Config", list["ConfigIssue"]]: ...   # never raises for user errors
    def save(self, cfg: "Config") -> None: ...                    # tomlkit round-trip + backup + atomic replace

# core/llm/client.py
class ChatClient(Protocol):
    def stream(self, profile: "LlmProfile", messages: Sequence["Message"]) -> AsyncIterator[str]: ...

# core/env (shape to be confirmed at G4.D)
class EnvStore(Protocol):
    def list(self) -> list["EnvVar"]: ...           # managed + read-only external
    def upsert(self, var: "EnvVar") -> None: ...
    def delete(self, name: str) -> None: ...
    def ensure_hook_installed(self) -> "HookStatus": ...
```

### 4.4 Data flow example: "Enter on a file → Edit → editor launched → view restored"

```
User presses Enter on "notes.md" in BrowserScreen
 │
 ├─1 BrowserScreen.action_context_menu()
 │     builds items from Entry.kind + Clipboard state → push_screen(ContextMenu(items), callback)
 ├─2 User presses "e" (or arrows + Enter on "Edit") → ContextMenu.dismiss("edit")
 ├─3 BrowserScreen.on_menu_result("edit")
 │     spec = ctx.config.apps.editor        # AppSpec(command=["nvim","{file}"], kind=TUI)
 │     run_worker(ctx.launcher.launch(spec, {"file": str(path), "cwd": str(cwd)}))
 ├─4 Launcher.launch()
 │     argv = substitute(spec.command, ctx)  → ["nvim", "/home/me/notes.md"] (no shell)
 │     shutil.which(argv[0]) is None? → LaunchResult(error=AppNotFound("nvim", hint="set apps.editor"))
 │     kind == TUI:
 │        with handover.suspended():           # Textual App.suspend(): leaves alt screen, cooked mode
 │            with ignore_sigint():            # Ctrl+C in nvim must not kill miraCLI
 │                code = subprocess.run(argv, cwd=cwd).returncode
 │                                             # on exit Textual re-enters alt screen, raw mode, full repaint
 │     kind == GUI:  Popen(argv, start_new_session=True, stdin/out/err=DEVNULL) + 300 ms early-exit check
 ├─5 Back in BrowserScreen (on_resume / worker result):
 │     refresh listing, keep cursor on "notes.md" (by name; nearest neighbour if gone)
 │     exit code ≠ 0 → toast "nvim exited with code 1"; error → ErrorModal(message, hint)
 └─6 Footer shows browser bindings again
```

### 4.5 Process launching strategy

| Mode | Used for | Mechanism | Notes |
|---|---|---|---|
| `tui` | `nvim`, `nmtui`, `htop`, interactive `git clone` fallback | `App.suspend()` → `subprocess.run(argv, cwd=…)` (inherits the tty) → resume. | SIGINT ignored in miraCLI while the child runs; terminal size re-read on resume; if `App.suspend` is unsupported (e.g. headless/web driver) → clear error. |
| `gui` | terminal emulator, `code`, `xdg-open` | `subprocess.Popen(argv, cwd, stdin/stdout/stderr=DEVNULL, start_new_session=True, close_fds=True)`; an asyncio task awaits it to reap the process (no zombies); if it exits non-zero within ~300 ms we report the error. | New session ⇒ survives miraCLI exit and SSH hangups are irrelevant. Before launching, check `DISPLAY`/`WAYLAND_DISPLAY`; if absent, error (or tmux fallback, Q6). |
| `capture` | short custom commands (`df -h`, `ip -br a`) | `asyncio.create_subprocess_exec` with piped output, streamed into a scrollable output screen; Esc closes, `c` cancels. | Timeout configurable; output truncated at N lines. |

Placeholders (`{cwd}`, `{file}`, `{selected}`, `{home}`) are substituted **per argv element** after the config's list form is loaded, so filenames with spaces or quotes can never become shell syntax. The `shell = "…"` form is explicit opt-in and receives values via environment variables (`MIRACLI_CWD`, `MIRACLI_SELECTED`) rather than string interpolation.

Built-in terminal templates (selected by executable name, overridable): `gnome-terminal --working-directory={cwd}`, `kitty --directory {cwd}`, `alacritty --working-directory {cwd}`, `wezterm start --cwd {cwd}`, `konsole --workdir {cwd}`, `xterm` (with `cwd=` set on the process), `tmux new-window -c {cwd}`.

Git clone uses the same split: **streamed mode** (`asyncio` subprocess, `git clone --progress`, env `GIT_TERMINAL_PROMPT=0` and, unless the user already set `GIT_SSH_COMMAND`, `GIT_SSH_COMMAND="ssh -o BatchMode=yes"`) so credential helpers and ssh-agent work but nothing blocks on a hidden prompt; on an auth-failure signature in stderr ("terminal prompts disabled", "Authentication failed", "Permission denied (publickey)", "Host key verification failed") the UI offers **"Retry interactively"**, which runs the same command in `tui` mode so git/ssh can prompt for passwords, passphrases or host-key confirmation.

### 4.6 Config schema (draft, `~/.config/miraCLI/config.toml`)

```toml
schema_version = 1

[ui]
mouse = false
vim_keys = true
theme = "textual-dark"

[browser]
start_dir = "launch"        # "launch" | "last" | "/abs/path"
show_hidden = false
show_metadata = false
confirm_delete = true
delete_mode = "trash"       # "trash" | "permanent"

[apps.editor]
command = ["nvim", "{file}"]
kind = "tui"

[apps.terminal]
command = ["gnome-terminal", "--working-directory={cwd}"]
kind = "gui"

[apps.network]
command = ["nmtui"]
kind = "tui"

[git]
default_site = "https://github.com"
clone_mode = "auto"         # "auto" | "streamed" | "interactive"

[menu]
order  = ["browser", "git_clone", "terminal", "env", "chat", "network", "settings"]
hidden = []
[menu.labels]
browser = "Files"

[[menu.custom]]
id = "htop"
label = "System monitor"
hotkey = "m"
command = ["htop"]
kind = "tui"

[[menu.custom]]
id = "disk"
label = "Disk usage"
shell = "df -h | sort -k5 -r"
kind = "capture"

[llm]                        # shape finalized at G5.D
default_profile = "openai"
[[llm.profiles]]
name = "openai"
base_url = "https://api.openai.com/v1"
model = "gpt-4o-mini"
api_key_env = "OPENAI_API_KEY"   # or api_key_keyring = "miraCLI/openai"
```

- **First run**: no file → write a fully commented default file (detected editor from `$VISUAL`/`$EDITOR`, terminal per Q7, `nmtui` if on PATH) and show a one-time toast with its path.
- **Validation**: load never crashes. Invalid sections fall back to defaults, the app starts, and a banner lists issues (`config.toml:14 apps.editor.kind: must be one of tui, gui, capture`) with a key to open the file in the editor and reload. `miracli --check-config` validates from the shell (non-zero exit on errors).
- **Migration**: `schema_version` integer; ordered pure functions `migrate_1_to_2(doc: tomlkit.TOMLDocument) -> None` applied in sequence; original backed up as `config.toml.bak-v1` before rewriting; newer-than-known version ⇒ read-only mode with a warning (never downgrade-write).
- **Settings must not lock you out**: `settings` cannot be hidden from the menu (validator rejects it), and `Ctrl+,` opens settings from anywhere.
- **State vs config**: MRU remotes/owners, last directory and window state go to `~/.local/state/miraCLI/state.json`, never into the user-edited config.

---

## 5. UX specification

### 5.1 Screen map

```
                     ┌──────────────┐
          launch ──► │  Main menu   │ ◄──────── Esc from any top-level screen
                     └──────┬───────┘
     ┌──────────┬───────────┼────────────┬───────────┬──────────┬──────────────┐
     ▼          ▼           ▼            ▼           ▼          ▼              ▼
 File browser  Git clone  Env vars     LLM chat   Settings   [launch]       [custom]
     │          form       (M4)         (M5)        (M6)     Terminal,      tui/gui/
     │            │                                  │       Network        capture
     ├─► Context menu ─► Confirm / Rename prompt /   ├─► Section editors   → output screen
     │                   Conflict dialog / Progress  └─► Menu editor
     ├─► Directory menu (a)
     └─► Help overlay (?)   ── available on every screen
              Git clone ─► Progress ─► Success ("Open in browser") ─► File browser
```

### 5.2 Main menu

```
┌ miraCLI ──────────────────────────────────────────────── ~/dev/miraCLI ┐
│                                                                        │
│      ▸ [f] Files                                                       │
│        [g] Clone a git repository                                      │
│        [t] Open terminal here                                          │
│        [v] Environment variables                                       │
│        [c] Chat (LLM)                                                  │
│        [n] Network                                                     │
│        [m] System monitor                       custom                 │
│        [s] Settings                                                    │
│                                                                        │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│ ↑↓ move  ⏎ open  f/g/t… jump  ? help  ctrl+, settings  q quit          │
└────────────────────────────────────────────────────────────────────────┘
```

Header right side: the shared "current directory" (the browser's cwd), so it is clear where "Open terminal here" will land.

### 5.3 File browser (metadata column on)

```
┌ /home/me/dev/miraCLI ──────────────────────────────── 14 items · hidden off ┐
│  Name                                              Size   Modified         │
│ ▸ ▣ docs/                                             –   2026-11-04 16:02 │
│   ▣ src/                                              –   2026-11-04 16:02 │
│   ▣ tests/                                            –   2026-11-03 09:12 │
│   ↪ build → /tmp/miracli-build/                       –   2026-11-01 11:40 │
│   · CHANGELOG.md                                  1.2 K   2026-11-04 16:02 │
│   · pyproject.toml                                2.4 K   2026-11-04 16:02 │
│   · README.md                                       65 B  2026-11-04 16:02 │
│   ⚠ old-link → ../gone                     (broken link)                   │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│ clipboard: copy · pyproject.toml                                           │
│ ↑↓ move ← parent → enter ⏎ actions a dir-actions . hidden i info p paste  │
│ esc menu ? help                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

Behaviour details:
- Sorting: directories (including symlinks to directories) first, then files; each group by `str.casefold()` with a natural-number key (`file2` < `file10`).
- `.` toggles hidden entries (names starting with `.`), state persisted in config only when changed via settings; header shows `hidden on/off`.
- Symlinks: `↪` prefix and `→ target`; broken links `⚠`, cannot be entered; `→` on a link to a dir enters it via the logical path.
- Unreadable folder: `→` keeps you where you are and shows an inline error toast `Permission denied: /root` (no crash). If the current directory disappears or becomes unreadable on refresh, walk up to the nearest readable ancestor and say so.
- Empty folder: centered `(empty folder)` placeholder; `⏎` opens the directory menu instead.
- Very large folders: listing via `os.scandir` in a worker thread; `stat()` only when the metadata column is on (and lazily for visible rows on > 5,000 entries); `DataTable` virtualization; target: 20,000 entries listed and navigable in < 500 ms on SSD; "loading…" indicator after 150 ms.
- `Home`/`End`/`PgUp`/`PgDn`; type-ahead: typing letters jumps to the first matching name (resets after 1 s) — note: mnemonic keys are only active when type-ahead is off, so type-ahead is behind `/` (search jump) instead, to avoid conflicts.
- `Ctrl+R` refreshes; the view also refreshes after every operation and after returning from a suspended app.
- Narrow terminals (< 80 cols): metadata column hidden automatically; key hints wrap to two lines.
- Filenames are rendered as plain `rich.Text` (never markup), control characters shown as `\n`, `\t`, `?`, so a file named `[bold]x` or containing a newline cannot break the UI.

### 5.4 Context menu and directory menu

```
 Folder selected                   File selected                 Directory menu (a, or ⏎ on empty folder)
┌ src/ ──────────────────┐        ┌ README.md ─────────────┐     ┌ in /home/me/dev/miraCLI ──┐
│ ▸ n  New file inside   │        │ ▸ e  Edit              │     │ ▸ n  New file             │
│   N  New folder inside │        │   o  Open with default │     │   N  New folder           │
│   p  Paste into src/   │◄─only  │   c  Copy              │     │   p  Paste here           │◄─only if
│   c  Copy              │ if clip│   x  Cut               │     │   t  Open terminal here   │  clipboard
│   x  Cut               │ board  │   r  Rename            │     │   g  Clone repo here      │  non-empty
│   r  Rename            │ non-   │   d  Move to Trash     │     └───────────────────────────┘
│   d  Move to Trash     │ empty  │   D  Delete permanently│
│   D  Delete permanently│        └────────────────────────┘
└────────────────────────┘          esc close · ⏎ run · letter = shortcut
```

- **Paste** lives in (1) the folder context menu ("Paste into <folder>"), (2) the directory menu ("Paste here"), (3) the `p` key in the browser = paste into the current directory. The clipboard line under the list shows what will be pasted.
- Shortcuts `c`, `x`, `r`, `d`, `p`, `e`, `n`, `N` also work directly in the browser without opening the menu (shown in `?` help, not all in the footer to keep it short).
- Delete confirmation: `Move "build/" (1,234 items, 87 MB) to Trash?  [Cancel] [Move to Trash]` — Cancel focused. Permanent: red dialog, `Permanently delete "build/" and 1,234 items? This cannot be undone. [Cancel] [Delete]`, Cancel focused, requires pressing `D` again or Tab+Enter. If the trash is unavailable (e.g. a mount without a trash dir), the dialog says so and offers permanent delete.
- Conflict dialog (paste/move):

```
┌ Conflict ───────────────────────────────────────────────────────┐
│ "report.pdf" already exists in /home/me/docs                    │
│   existing:  1.1 MB  2026-10-02 10:11                           │
│   incoming:  1.3 MB  2026-11-04 15:40                           │
│                                                                 │
│  [o] Overwrite   [s] Skip   [k] Keep both → "report (1).pdf"    │
│  [c] Cancel all                       [ ] apply to all (space)  │
└─────────────────────────────────────────────────────────────────┘
```

- Rename / new file / new folder: inline prompt modal prefilled with the current name (stem selected for files). Invalid names (`/`, NUL, empty, `.`, `..`) rejected live; existing target ⇒ error, never silent overwrite.
- Progress modal appears only for operations lasting > 300 ms: current file, `files 120/1,234`, bytes, `ProgressBar`, `Esc` = cancel (with "Cancel copy? partially copied items will be removed" confirmation).

### 5.5 Git clone form

```
┌ Clone a git repository ──────────────────────────────────────────────────┐
│                                                                          │
│  Remote site   [https://github.com                              ] ▾ recent│
│  Owner         [elandini84                                      ] ▾ recent│
│  Repo name     [test-repo                                       ]         │
│  Branch        [devel                     ]  (empty = remote default)     │
│  Destination   [~/dev                                           ]         │
│                                                                          │
│  ── Preview ───────────────────────────────────────────────────────────  │
│  Owner remote  https://github.com/elandini84                             │
│  Clone URL     https://github.com/elandini84/test-repo.git               │
│  Target path   /home/me/dev/test-repo            ✓ does not exist        │
│  Command       git clone --branch devel <url> /home/me/dev/test-repo     │
│                                                                          │
│                                         [ Check remote ]  [ Clone ]      │
├──────────────────────────────────────────────────────────────────────────┤
│ tab/↑↓ field  ctrl+↓ suggestions  ctrl+t check remote  ctrl+s clone  esc │
└──────────────────────────────────────────────────────────────────────────┘

Progress view (replaces the preview while cloning):
│  Cloning into /home/me/dev/test-repo …                                   │
│  Receiving objects   ██████████████░░░░░░░░░░  58%  (5,812/10,020) 3 MB/s │
│  Resolving deltas    ░░░░░░░░░░░░░░░░░░░░░░░░   0%                       │
│  esc cancel                                                              │
```

- SSH: if Remote site is `git@host` or `ssh://git@host[:port]`, the URL becomes `git@host:owner/repo.git` / `ssh://git@host:port/owner/repo.git`. HTTPS sites without scheme get `https://`.
- Validation (live, per field): site parses to a host; owner segments and repo match `^[A-Za-z0-9._-]+$` (a trailing `.git` on repo is stripped with a notice); branch validated with `git check-ref-format --branch` (cached call); destination expanded (`~`, env vars) and must be an existing writable directory (offer "Create it?" if missing); target `<dest>/<repo>` must not exist or be an empty directory.
- "Check remote" (optional, async, 10 s timeout): `git ls-remote --heads <url> <branch>` → "✓ reachable, branch exists" / readable error; never blocks typing.

### 5.6 Global keybindings

| Key | Context | Action |
|---|---|---|
| `↑` `↓` (`k` `j`) | lists, menus | move cursor |
| `PgUp` `PgDn` `Home` `End` (`g` `G`) | lists | page / jump |
| `⏎` | menu / list | open entry / open context menu |
| `→` (`l`) | browser | enter folder (no-op on files) |
| `←` (`h`) | browser | parent folder, cursor on folder just left |
| `Esc` | everywhere | close topmost modal → else back to main menu → on main menu: nothing (no accidental quit) |
| `Tab` / `Shift+Tab` | forms, dialogs | next / previous field or button |
| `?` | everywhere | help overlay listing all bindings of the current screen (generated from bindings) |
| `q` | main menu | quit (`Ctrl+Q` quits from anywhere, with confirmation if an operation is running) |
| `Ctrl+,` | everywhere | settings |
| `Ctrl+R` | browser | refresh |
| `.` | browser | toggle hidden files |
| `i` | browser | toggle size/mtime column |
| `a` | browser | directory (current folder) menu |
| `/` | browser | jump-to-name search |
| `c` `x` `p` | browser | copy / cut / paste into current dir |
| `r` / `F2` | browser | rename |
| `d` / `Del` | browser | move to Trash (with confirm) |
| `D` / `Shift+Del` | browser | delete permanently (with confirm) |
| `e` `n` `N` | browser | edit / new file / new folder |
| `t` | browser | open terminal in current dir |
| `Ctrl+S` | forms | submit (Clone, Save) |
| letter mnemonics | main menu, menus | jump/activate entry with that hotkey |

Conflicting keys are detected at startup (custom entries' hotkeys vs built-ins) and reported as config issues.

---

## 6. Roadmap

### 6.0 Changes to the suggested skeleton, and why

| Brief | This plan | Reason |
|---|---|---|
| M0 Foundations | **M0 Foundations + launchers** (includes Open terminal, Network, custom commands from config file) | "No milestone may end with infrastructure only". The launcher service is already built in M0; wrapping it into three menu entries costs ~1 day and gives a usable app in week 3. |
| M1 Browser (read-only) | M1 unchanged | — |
| M2 File management | M2 unchanged | — |
| M3 Launchers | merged into M0 | see above |
| M4 Git clone | **M3** Git clone | renumbered |
| M5 Env vars | **M4** Env vars — discovery stage D4 **pulled forward** to run in parallel with M2/M3 | Discovery is mostly writing + your review; doing it early means the design gate is passed when build starts, no idle week. |
| M6 LLM chat | **M5** LLM chat — discovery D5 pulled forward in parallel with M3/M4 | same |
| M7 Menu configuration | **M6** In-app settings + menu editor (custom entries via file already exist from M0) | File-based configuration is needed from day one; only the in-app editor is deferred. |
| M8 Hardening | **M7** Hardening & v1.0 | renumbered |

Estimated total: ~20 build/discovery stages ≈ 5 months at 6–8 h/week.

### 6.1 Dependencies

```
M0 ──► M1 ──► M2
 │      └─(soft)──► M3   (default destination = browser cwd; "open in browser" after clone)
 ├──────────────────► M3  (can run before M1 if you want git clone first)
 ├──► D4 ──G4.D──► M4    (independent of M1–M3)
 ├──► D5 ──G5.D──► M5    (independent; the "attach selected file" extra needs M1)
 └──► M6  needs the settings sections of all features that exist; can be built after any subset
M7 needs everything planned for v1.0
```

Reorderable: M3, M4, M5 are mutually independent and can be done in any order after M0; M3 may even precede M1. M2 strictly needs M1. M6 can move earlier (its framework auto-discovers feature settings sections), at the cost of revisiting it when later features add settings.

---

### M0 — Foundations + launchers → `v0.1.0`

**Goal / increment:** `pipx install` gives a `miracli` command that opens a main menu from which I can open my terminal emulator in the current directory, open `nmtui`, and run my own commands defined in `config.toml`. Errors are readable; the config file is created on first run.

**Scope in:** framework spike & ADR; repo skeleton, packaging, CI, pre-commit; config load/validate/defaults/first-run/`--check-config`; XDG paths & logging; feature registry & main menu with hint bar, help overlay (basic), quit; launcher service (tui/gui/capture), Textual handover; features: Terminal, Network, Custom commands.
**Scope out:** browser (so "current directory" = launch directory), in-app settings, migrations beyond `schema_version = 1` scaffolding, themes.

#### S0.1 — Framework spike (M)
- **Objective:** decide the TUI framework with measured evidence.
- **Stories:**
  - *As the developer I want evidence that the framework meets the hard requirements so that I don't rewrite the UI later.*
    - Given a throwaway Textual prototype, When I open a 20,000-row table, Then scrolling with ↓/PgDn stays responsive (no visible lag) and first render < 500 ms.
    - Given the prototype, When I trigger "edit" on a file, Then `nvim` takes over the terminal and on `:q` the prototype is fully repainted with the same state.
    - Given a fake async token stream at 50 tok/s, When I type into an input at the same time, Then keystrokes are never delayed.
    - Given `python -X importtime` and a first-paint timestamp, When I launch the prototype 10 times, Then the median time-to-first-paint is reported (target < 300 ms).
- **Tasks:** Textual prototype (table, modal menu, suspend→`nvim`, detached `gnome-terminal`, streaming, Pilot test, snapshot test); same suspend + table + startup check in prompt_toolkit (half a day) for comparison; measure pydantic vs msgspec import cost; write `docs/adr/0001-tui-framework.md` and `0002-config-format.md`.
- **Deliverable:** ADRs with numbers; spike code in a `spike/` branch (not merged).
- **Mini-gate G0.1 (framework decision):** Go = Textual meets all checks (startup ≤ ~300 ms with lazy imports). Go with conditions = Textual slightly over budget with an identified fix. Rework = switch to prompt_toolkit and redo S0.1 table/form checks.

#### S0.2 — Skeleton, config, main menu (M)
- **Objective:** an installable app that shows a configurable main menu.
- **Stories:**
  - *As a user I want to install miraCLI with pipx so that I get a single command.*
    - Given a clean machine with pipx, When I run `pipx install git+https://github.com/lando1784/miraCLI`, Then `miracli --version` and `miraCLI --version` print the version.
  - *As a user I want a default config created on first run so that I can see and edit every option.*
    - Given no `~/.config/miraCLI/config.toml`, When I start miraCLI, Then the file is created with comments and detected defaults and a toast shows its path.
    - Given a config with `apps.editor.kind = "foo"`, When I start miraCLI, Then the app starts with the default for that key and a banner shows the file, line and expected values.
    - Given that config, When I run `miracli --check-config`, Then it prints the same issue and exits with code 1.
  - *As a user I want a main menu with a visible hint bar so that I always know which keys work.*
    - Given the main menu, When it is shown, Then entries appear in `menu.order`, hidden ones are absent, labels are overridden by `menu.labels`, and the footer lists the active keys.
    - Given the main menu, When I press `?`, Then a help overlay lists the bindings; Esc closes it.
    - Given an 80×24 terminal resized to 60×20, Then nothing crashes and the menu stays usable (scrolls).
- **Tasks:** `pyproject.toml` (hatchling, scripts, deps pinned to minor ranges); `uv.lock`; ruff/mypy/pytest config; pre-commit; `.github/workflows/ci.yml` (matrix 3.10–3.13, lint, types, tests, `pipx install .` smoke job, import-time report); `core/paths.py`, `core/logging.py` with redaction; `core/config` schema/defaults/loader/store (read) + `--check-config`; `features/registry.py`; `MainMenuScreen`, help overlay, error modal, toast; `import-linter` contract (core ↛ textual).
- **Deliverable:** installable app with a main menu (only "Quit" and "Help" are functional at this point; the Settings → "Edit config file" entry arrives in S0.3 once the launcher exists), CI green.

#### S0.3 — Launcher service and launch entries (M)
- **Objective:** delegate to external apps safely (TUI hand-over, GUI detached, capture).
- **Stories:**
  - *As a user I want "Open terminal" to open my terminal emulator in the current directory so that I can continue working there.*
    - Given `apps.terminal` is gnome-terminal and a graphical session, When I choose "Open terminal here", Then a new window opens in the launch directory and miraCLI keeps running and responsive.
    - Given an SSH session without `DISPLAY`/`WAYLAND_DISPLAY` and not inside tmux, When I choose it, Then an error explains that no display is available and how to configure a TUI alternative.
    - Given I am inside tmux (Q6 default), When I choose it, Then a new tmux window opens in that directory.
  - *As a user I want "Network" to open nmtui so that I manage connections without leaving the flow.*
    - Given `apps.network = nmtui (tui)`, When I choose Network, Then nmtui takes the full terminal; When I quit nmtui, Then miraCLI's main menu is redrawn exactly as before.
    - Given nmtui is not installed, When I choose Network, Then the error says "nmtui not found on PATH — set apps.network in config.toml".
  - *As a user I want to add my own commands to the menu via the config file so that the menu grows with my habits.*
    - Given a `[[menu.custom]]` with `command = ["htop"]`, `kind = "tui"`, When I restart miraCLI, Then "System monitor" appears with its hotkey and runs htop with hand-over.
    - Given a custom entry with `kind = "capture"` and `shell = "df -h"`, When I run it, Then the output is shown in a scrollable screen; Esc returns to the menu.
    - Given a custom hotkey colliding with a built-in, Then a config issue is shown and the custom hotkey is ignored.
  - *As a user I want to open the config file from the menu so that I can edit it quickly.*
    - Given the Settings entry, When I choose "Edit config file", Then the editor opens it; on exit the config is reloaded and issues (if any) are shown.
- **Tasks:** `core/launch` (spec, placeholder substitution, `which` check, run modes, early-exit detection, reaping); `ui/handover.py` (App.suspend + SIGINT guard); `platform/linux.py` display detection, terminal templates; features `terminal`, `network`, `custom`, settings stub; capture output screen; first-run detection of editor/terminal/network tool.
- **Deliverable:** usable v0.1.

#### Gate G0 — `v0.1.0`
- **Exit criteria:**
  - [ ] ADR-0001/0002/0003 merged; framework decision recorded.
  - [ ] CI green on 3.10–3.13; ruff, mypy (strict on core/platform), pytest pass; coverage of `core/` ≥ 85 %.
  - [ ] `pipx install` from the tag works on a clean Ubuntu 24.04 (VM or container) user.
  - [ ] Median time-to-first-paint ≤ 300 ms on your laptop (`miracli --bench-startup` prints it; dev-only flag).
  - [ ] No traceback reaches the screen in the demo script; every error is a modal/toast; log file written.
- **Manual demo script:**
  1. `pipx install git+https://github.com/lando1784/miraCLI@v0.1.0 && rm -rf ~/.config/miraCLI && cd ~/dev && miracli` → config-created toast.
  2. Press `?` → help; `Esc`.
  3. `t` → GNOME Terminal opens in `~/dev`; miraCLI still responsive.
  4. `n` → nmtui; quit it → menu restored, no garbage on screen; resize terminal while nmtui is open, quit → layout correct.
  5. Edit config via Settings: add the `htop` and `df -h` custom entries, misspell `kind` on one → banner shows the issue; fix it → reload works.
  6. `ssh localhost`, run `miracli`, press `t` → readable "no display" error (or tmux window if in tmux).
  7. `miracli --check-config; echo $?`.
- **Required automated tests:** config defaults/validation/issue locations; placeholder substitution (spaces, quotes, `{` in filenames); launcher modes with fake spawner + real `true`/`false`/`sh -c 'exit 3'`; registry ordering/hidden/labels/hotkey conflicts; Pilot tests for main menu navigation, help overlay, error modal; snapshot of main menu at 80×24.
- **Risks to re-check:** R1 startup budget, R2 Textual API churn (pin range), R5 hand-over glitches.
- **Decision options:** Go / Go with conditions (e.g. "startup 340 ms, fix in M7") / Rework S0.3 / Re-scope (drop capture mode).
- **Backlog refinement for M1:** answer Q8, Q20; confirm the browser mock-up (§5.3); decide whether to start D4/D5 discovery in parallel.

---

### M1 — File browser (read-only) → `v0.2.0`

**Goal / increment:** I can browse the filesystem from the menu exactly as specified in §3.1, including edge cases, and the browser's directory becomes the "current directory" for Open terminal.
**Scope in:** listing service, navigation keys, header, sorting, hidden toggle, symlinks, errors, empty/huge folders, metadata column, refresh, search-jump, start dir config, `t` open terminal here.
**Scope out:** any modification of files (M2), previews, bookmarks, multi-select.

#### S1.1 — Core listing and navigation (M)
- **Objective:** the four navigation keys + Enter/Esc work on normal directories.
- **Stories:**
  - *As a user I want to start browsing in the directory I launched miraCLI from so that I'm immediately where I work.*
    - Given I ran `miracli` in `~/dev/proj`, When I open Files, Then the header shows `/home/me/dev/proj` and the cursor is on the first entry.
  - *As a user I want folders first and alphabetical order so that I find things predictably.*
    - Given a dir with `b.txt`, `A/`, `a.txt`, `c/`, `file10`, `file2`, Then the order is `A/`, `c/`, `a.txt`, `b.txt`, `file2`, `file10`.
  - *As a user I want ← to go up with the cursor on the folder I left so that I don't lose my place.*
    - Given I am in `~/dev/proj/src`, When I press ←, Then I am in `~/dev/proj` with the cursor on `src/`.
    - Given I am in `/`, When I press ←, Then nothing changes (subtle flash/notice).
  - *As a user I want → to enter folders only.*
    - Given the cursor on a file, When I press →, Then nothing happens; Given a folder, Then I enter it with cursor on its first entry.
  - *As a user I want Esc to return to the main menu*, Given no modal is open, When I press Esc, Then the main menu shows and its header shows the browser's directory.
- **Tasks:** `core/fs/listing.py` (scandir, kinds, sort key, display-name sanitizing, logical parent); `BrowserScreen` with `DataTable`; cursor restoration by name; shared `state.cwd`; empty-folder placeholder; unit tests on `tmp_path`.
- **Deliverable:** browsing works on ordinary trees.

#### S1.2 — Edge cases, metadata and performance (M)
- **Objective:** robust behaviour on the cases listed in §3.1.
- **Stories:**
  - *As a user I want hidden files off by default with a toggle.* Given `.git/` and `.env`, When I open Files, Then they're not shown; When I press `.`, Then they appear and the cursor stays on the same entry.
  - *As a user I want symlinks to be recognisable.* Given `build → /tmp/b/` and `old → ../gone`, Then `build` shows `↪ … → /tmp/b/`, `old` shows broken; → on `build` enters it with header `/…/proj/build`; ← returns to `proj` with cursor on `build`.
  - *As a user I want readable errors for unreadable folders.* Given `/root`, When I press → on it, Then I stay and a toast says `Permission denied: /root`.
  - *As a user I want huge folders to stay fast.* Given a directory with 20,000 files, When I enter it, Then it is navigable within 500 ms and PgDn/End are instant.
  - *As a user I want optional size/mtime columns.* When I press `i`, Then Size (human-readable) and Modified columns appear; columns hide automatically under 80 columns.
  - *As a user I want to open a terminal where I am.* Given I'm in `~/dev/proj/src`, When I press `t`, Then the terminal opens there.
  - *Directory removed under me:* Given I'm in `~/tmp/x` and it is deleted from another shell, When I press Ctrl+R, Then I land in `~/tmp` with a notice.
- **Tasks:** threaded listing + loading indicator; lazy stat; `/` jump search; Home/End/PgUp/PgDn; refresh & on-resume refresh; non-UTF-8 and control-char filenames; `browser.start_dir` setting; perf test fixture generating 20k files; snapshots.
- **Deliverable:** v0.2.

#### Gate G1 — `v0.2.0`
- **Exit criteria:** [ ] all S1.x acceptance criteria demonstrated; [ ] listing unit tests cover every `Entry.kind`, sorting, hidden, errors, invalid UTF-8; [ ] Pilot tests for each navigation key and cursor restore; [ ] 20k-entry perf test < 500 ms in CI (warn-only on slow runners, hard limit locally); [ ] no crash in a 10-minute exploratory session in `/`, `/proc`, `/dev`, `~`.
- **Manual demo script:** prepare with `mkdir -p /tmp/mc/big && cd /tmp/mc && mkdir A c .hidden && touch b.txt a.txt file2 file10 $'new\nline' && ln -s /etc etc-link && ln -s ../gone broken && (cd big && seq 1 20000 | xargs touch)`, then `miracli` → Files → verify order, `.` toggle, enter `etc-link` and back, `big` speed, `i` column, `/` jump to `file10`, → on `/root`, `t` opens terminal in `/tmp/mc/big`, Esc → menu header shows the dir; resize to 70×20.
- **Automated tests:** as in exit criteria + snapshot of browser 80×24 with/without metadata.
- **Risks to re-check:** R4 large-dir performance, R9 odd filenames, R11 `/proc` special files.
- **Decision options:** Go / Go with conditions / Rework S1.2 / Re-scope (drop jump search).
- **Backlog refinement for M2:** answer Q9–Q13; approve §5.4 mock-ups; confirm conflict dialog wording.

---

### M2 — File management → `v0.3.0`

**Goal / increment:** I can copy, cut/paste, rename, delete (Trash or permanent), create files and folders, and edit files with my editor, safely and with progress for long operations.
**Scope in:** context menu, directory menu, clipboard, all operations of §3.2 + New folder + Open with default app (if Q13 yes), conflicts, progress/cancel, cross-filesystem moves.
**Scope out:** multi-select, undo (Trash is the undo), background job queue, permissions/chmod editing, archive handling.

#### S2.1 — Context menu, rename, create, edit (M)
- **Stories:**
  - *As a user I want Enter to show only the actions that apply.* Given a file, Then the menu lists Edit, (Open with default), Copy, Cut, Rename, Move to Trash, Delete permanently; Given a folder, Then New file, New folder, Copy, Cut, Rename, Move to Trash, Delete permanently (+ Paste into when clipboard non-empty); Esc closes it with no side effect.
  - *As a user I want to edit a file in my editor and come back to the same place.* Given `notes.md` selected and `apps.editor = nvim (tui)`, When I choose Edit, Then nvim opens it; When I quit, Then the browser is restored with the cursor on `notes.md` and listing refreshed.
  - *As a user I want to rename safely.* Given `a.txt` and existing `b.txt`, When I rename `a.txt` to `b.txt`, Then an error says it exists and nothing changes; When I rename to `c.txt`, Then the list refreshes with the cursor on `c.txt`.
  - *As a user I want to create a file inside a folder.* Given folder `src/` selected, When I choose New file and type `x.py`, Then `src/x.py` exists (created exclusively) and I'm told where; invalid names are rejected live.
  - *New folder*: same as above for directories, also from the directory menu (`a`).
- **Tasks:** `ContextMenu`/`DirectoryMenu` modals with mnemonic letters; `Prompt` modal with validators; `FileOps.rename/create_file/create_dir`; `core/fs/names.py` (validation, "keep both" naming `name (1).ext`); browser shortcuts; Pilot tests.

#### S2.2 — Delete and clipboard for files (M)
- **Stories:**
  - *As a user I want deleted items to go to the Trash after confirmation.* Given `old.log`, When I press `d`, Then a dialog with Cancel focused appears; Enter → cancelled; Tab+Enter → the file is in `~/.local/share/Trash/files/` and listed in GNOME Files' Trash; the cursor moves to the next entry.
  - *As a user I want a permanent delete that is hard to trigger by mistake.* When I press `D`, Then a red dialog states it cannot be undone; only explicit confirmation deletes.
  - *Trash unavailable:* Given a mount without a usable trash, When I delete, Then the dialog explains it and offers permanent deletion.
  - *As a user I want copy/cut/paste of files with conflict handling.* Given I copied `a.txt` and pressed `p` in another dir containing `a.txt`, Then the conflict dialog offers Overwrite / Skip / Keep both / Cancel; Keep both creates `a (1).txt`; paste into the same dir as the source creates `a (copy).txt` without asking.
    - Given I cut `a.txt` and pasted it elsewhere, Then it's moved and the clipboard is cleared; a copy clipboard stays filled.
    - Given the clipboard references a file deleted meanwhile, When I paste, Then an error lists the missing item and others proceed.
- **Tasks:** `TrashAdapter` (send2trash); delete with confirm + item count (`FileOps.count` in worker); `Clipboard`; file copy (chunked, `copystat`, temp file + `os.replace` for overwrite); same-fs move via `os.rename`; `ConflictResolver` modal; clipboard indicator line.

#### S2.3 — Folders, progress, cancellation, cross-filesystem (L)
- **Stories:**
  - *As a user I want to copy/move whole folders.* Given `proj/` with nested files and symlinks, When I copy-paste it, Then the tree is reproduced with symlinks copied as symlinks (not followed), modes and mtimes preserved.
  - *As a user I am prevented from pasting a folder into itself.* Given I copied `proj/` and selected `proj/src/`, When I paste into it, Then an error explains why and nothing happens.
  - *As a user I want progress and cancel for long operations.* Given a 2 GB folder, When I paste, Then a progress modal shows files/bytes/current file within 300 ms and the UI stays responsive; When I press Esc and confirm, Then copying stops within ~1 s and the partial destination is removed; the source is untouched.
  - *As a user I want moves across filesystems to be safe.* Given I cut a folder on `/home` and paste on a USB stick (`EXDEV`), Then it is copied fully, verified (file count/sizes), and only then the source is removed; on cancel/failure the source is intact.
  - *Partial failures*: Given one unreadable file inside a folder being copied, Then the operation continues, and at the end a report lists "1 item failed: Permission denied: …".
- **Tasks:** recursive walker with pre-count; progress throttling (≤ 10 UI updates/s via `call_from_thread`); cancel `threading.Event` checked between chunks; rollback bookkeeping (only items we created); EXDEV path; dir-on-dir conflict (Replace/Skip/Keep both); `OpReport` modal; integration tests on `tmp_path` with a second filesystem (tmpfs via `/dev/shm` when available, otherwise mocked `EXDEV`).

#### Gate G2 — `v0.3.0`
- **Exit criteria:** [ ] every operation × {file, folder, symlink} covered by integration tests on `tmp_path`; [ ] cancellation tests prove source integrity for copy and cross-fs move; [ ] no operation overwrites without an explicit choice (property-style test over conflict combinations); [ ] delete never runs without confirmation when `confirm_delete = true` (Pilot test); [ ] manual Trash restore works from GNOME Files; [ ] UI responsive during a 2 GB copy.
- **Manual demo script:** in `/tmp/mc2`: create file → rename → edit with nvim → copy into `sub/` twice (conflict → Keep both) → cut folder to `/dev/shm/x` (cross-fs) → copy a 2 GB folder (`fallocate -l 2G big.bin` in a folder) and cancel mid-way, check destination removed → trash a file and restore it from GNOME Files → permanent delete a folder → try pasting a folder into itself.
- **Automated tests:** listed above + snapshots of context menus and conflict dialog.
- **Risks to re-check:** R6 data loss, R7 cross-fs partial moves, R8 Trash on other volumes.
- **Decision options:** Go / Go with conditions / Rework S2.3 / Re-scope (e.g. drop cross-fs move cancel to "not cancellable after copy phase").
- **Backlog refinement for M3:** answer Q14–Q16; approve §5.5 mock-up; list your typical hosts (GitHub only? GitLab? self-hosted?).

---

### M3 — Git clone → `v0.4.0`

**Goal / increment:** I can clone a repository by filling a short form with live preview, see progress, handle auth, and jump into the cloned folder.
**Scope in:** form, URL builder for HTTPS/SSH, validation, MRU suggestions, check remote, streamed clone with progress and cancel, interactive fallback, success → browser.
**Scope out:** submodules options, shallow clones (could be a later "Advanced" toggle), forking, GitHub API.

#### S3.1 — Form, preview, validation, suggestions (M)
- **Stories:**
  - *As a user I want a live preview of URL and path.* Given site `https://github.com`, owner `elandini84`, repo `test-repo`, dest `~/dev`, Then the preview shows owner remote `https://github.com/elandini84`, URL `https://github.com/elandini84/test-repo.git`, target `/home/me/dev/test-repo`, and the git command without `--branch`; When I type branch `devel`, Then `--branch devel` appears.
  - *As a user I want SSH remotes.* Given site `git@github.com`, Then URL is `git@github.com:elandini84/test-repo.git`.
  - *As a user I want clear validation.* Given empty owner, Then the field is marked and Clone is disabled; Given repo `test-repo.git`, Then it's normalized with a notice; Given branch `bad..name`, Then it's rejected; Given `~/dev/test-repo` already exists and is non-empty, Then the target line shows "already exists" and Clone is disabled.
  - *As a user I want the destination to default to the browser directory.* Given I was browsing `~/dev`, When I open the form, Then Destination is `~/dev`.
  - *As a user I want recent sites/owners suggested.* Given I cloned from owner `elandini84` before, When I type `el`, Then `elandini84` is suggested inline (→ accepts).
- **Tasks:** `core/git/url.py` + `validate()` (pure, heavily unit-tested); form screen with Textual `Input` validators and `SuggestFromList`; `core/history.py` MRU; async "Check remote" (`git ls-remote`, timeout).

#### S3.2 — Clone execution (M)
- **Stories:**
  - *As a user I want progress while cloning.* When I press Ctrl+S, Then a progress view shows "Receiving objects" and "Resolving deltas" percentages updating at least every second, and the UI stays responsive.
  - *As a user I want to cancel.* When I press Esc during clone and confirm, Then git is terminated and the partially created target directory is removed (only if miraCLI created it).
  - *As a user I want to authenticate when needed.* Given a private HTTPS repo with no credential helper, When the streamed clone fails with an auth error, Then I'm offered "Retry interactively"; choosing it hands over the terminal, git prompts for username/token, and on completion I return to miraCLI with the result.
    - Given an SSH remote whose host key is unknown, Then the same fallback lets ssh ask for host-key confirmation.
  - *As a user I want readable errors.* Given a non-existent repo, Then the message is "Repository not found (or no access): https://…" with git's last stderr lines expandable; Given no network, Then "Could not resolve host github.com".
  - *As a user I want to jump into the result.* Given a successful clone, Then a dialog offers "Open in browser" (focused) which opens Files in `<dest>/<repo>`.
- **Tasks:** `core/git/clone.py` (asyncio subprocess, process group, env), `progress_parser.py` (split on `\r`/`\n`, regex for phases) with recorded stderr fixtures; error classifier; interactive mode via launcher `tui`; success flow; MRU update.

#### Gate G3 — `v0.4.0`
- **Exit criteria:** [ ] URL/validation unit tests incl. SSH, ports, GitLab subgroups, trailing slashes, `.git`; [ ] integration test cloning a local bare repo via `file://` with and without `--branch`; [ ] progress parser tests on fixtures from real GitHub clones; [ ] cancel test removes target; [ ] auth-failure classification tests on fixture stderr.
- **Manual demo script:** clone a public GitHub repo (default branch) → cancel a large clone (e.g. `torvalds/linux`) → clone with branch → clone a private repo over HTTPS with no helper (interactive fallback) → SSH remote with ssh-agent → wrong repo name → existing target → "Open in browser".
- **Risks to re-check:** R10 credential prompts, R12 git output format changes across versions.
- **Decision options:** Go / Go with conditions / Rework S3.2 / Re-scope (drop "Check remote").
- **Backlog refinement for M4:** confirm the outcome of G4.D is still valid (D4 has run in parallel).

---

### M4 — Environment variables [TBD] → `v0.5.0`

**Goal / increment:** I can list, create, edit and delete environment variables (including PATH entries) without editing `~/.bashrc` by hand, with backups, and see variables defined elsewhere.

#### D4 — Discovery (S–M; can run in parallel with M2/M3)
- **Objective:** a design you approve, before any build.
- **Outputs:** (1) options a–d write-up (§3.3) with a worked example of the generated file for each; (2) ASCII mock-ups of: variable list (managed vs external, with source `file:line`), edit form (name, value, literal/expanding, PATH-mode prepend/append), hook-install confirmation dialog, "changes apply to new shells" notice; (3) a throwaway parser run (read-only) against copies of your real `~/.bashrc`, `~/.profile` to see what "defined elsewhere" detection finds; (4) the questions below.
- **Questions you must answer at G4.D:**
  1. Option (a), (b), (c) or (d)? File location?
  2. Source of truth: structured `env.toml` → generated `env.sh`, or a strictly formatted `env.sh` parsed back?
  3. Values literal by default (`'…'`, no `$` expansion) or expanding (`"…"` with `$HOME` allowed)?
  4. PATH-like variables: which names (PATH, LD_LIBRARY_PATH, PYTHONPATH, MANPATH, custom list)? Idempotent prepend/append only, or also remove entries?
  5. Variables defined elsewhere: show read-only, offer "override in managed file", or ignore?
  6. May miraCLI append the hook block to `~/.bashrc` automatically (with backup + confirmation), or should it print the line for you to add?
  7. Should `~/.profile`/login shells also source the file (relevant for SSH logins, which on Ubuntu read `.profile` → `.bashrc`)?
  8. Backups: how many to keep, where?
  9. Need for per-variable comments/notes, enable/disable toggle without deleting?
- **Gate G4.D:** decision options: choose an option (Go), ask for another round (Rework D4), or drop the feature (Stop).

#### S4.1 — Managed store and hook (M) · S4.2 — UI and PATH operations (M)
Stories, acceptance criteria and tasks will be written from the approved design at G4.D. Already-known non-negotiable criteria:
- Given any write, Then a timestamped backup of every file modified exists first, the new file is written to a temp file, checked with `bash -n`, then atomically replaced.
- Given a value containing `'`, `"`, `$`, backticks, spaces or newlines, Then sourcing the generated file in `bash` yields exactly that value (property test with `hypothesis` running real `bash -c`).
- Given a saved change, Then the UI states it applies to new shells and offers "Open terminal here".

#### Gate G4 — `v0.5.0`
Exit criteria include: round-trip property tests in real bash; hook install idempotent (running twice adds one block); uninstall path documented (`miracli env --uninstall-hook` or manual block removal); manual demo on a copy of your real `.bashrc` in a throwaway `HOME=$(mktemp -d)`.

---

### M5 — LLM chat [TBD] → `v0.6.0`

**Goal / increment:** a chat screen inside miraCLI talking to OpenAI or an OpenAI-compatible local server with streaming replies.

#### D5 — Discovery (S–M; can run in parallel with M3/M4)
- **Outputs:** ASCII mock-ups of the chat screen (history pane with Markdown, multiline input, profile indicator, token/streaming status), profile picker, and of any extras you're interested in; draft `[llm]` schema; a 30-line spike proving `AsyncOpenAI(base_url=…)` streaming against OpenAI and a local Ollama (`http://localhost:11434/v1`).
- **Questions you must answer at G5.D:**
  1. Which extras from §3.4 do you accept (profiles, system prompt per profile, attach file/path, persisted conversations, Markdown rendering, copy reply)?
  2. Send key: Enter-sends/Alt+Enter-newline or Enter-newline/Ctrl+S-sends?
  3. Key storage: env var only, or env var + optional `keyring`?
  4. History length handling: send full history, or truncate to last N messages / N tokens?
  5. If attaching files: max size, allowed types, always-confirm what is sent?
  6. If persisting: where, retention, ability to disable per profile?
  7. Default model/profile names to ship in the default config.
- **Gate G5.D:** choose design (Go) / Rework D5 / Stop.

#### S5.1 — Client, profiles, streaming screen (M) · S5.2 — History, input, extras (M–L)
Detailed stories written after G5.D. Known criteria:
- Given the API key env var is unset, When I open Chat, Then a readable message names the variable to set; the key never appears in config, logs or screenshots (redaction test).
- Given a streaming reply, When I press Esc (or `Ctrl+C` in chat), Then generation stops and the partial reply is kept, marked "stopped".
- Given 401/404 model/429/connection errors, Then each has a specific readable message.
- Given a long reply, Then the history auto-scrolls while streaming unless I scrolled up.
- `openai` is imported lazily on first opening of Chat (startup budget).

#### Gate G5 — `v0.6.0`
Exit criteria include: fake `ChatClient` Pilot tests; `respx`-mocked SSE stream tests for the real adapter; manual demo against OpenAI and a local Ollama; no key in `miracli.log` after a session at debug level.

---

### M6 — In-app settings and menu editor → `v0.7.0`

**Goal / increment:** everything configurable in the TOML can be changed from a settings screen, including reordering/hiding/renaming menu entries and managing custom commands, without losing comments in my file.
**Scope in:** settings screen generated from feature-contributed pydantic sections; apps editor with "Test launch"; menu editor; custom entries CRUD; save via tomlkit with backup; live reload.
**Scope out:** theming editor, keybinding remapping (post-v1 candidate).

#### S6.1 — Settings framework and default apps (M)
- *As a user I want to change my editor in the app.* Given Settings → Apps → Editor, When I set command `code {file}` kind `gui` and press "Test", Then VS Code opens a scratch file; When I save, Then `config.toml` keeps my comments, a backup `config.toml.bak-<timestamp>` exists, and Edit uses the new editor immediately.
- *Invalid input is caught before saving*: Given kind `foo`, Then the field shows the allowed values and Save is disabled.
- *External edits are respected*: Given the file changed on disk after the screen opened, When I save, Then I'm asked to reload or overwrite.

#### S6.2 — Menu editor and custom entries (M)
- *As a user I want to reorder and hide entries.* Given the menu editor, When I move "Chat" up with `Shift+↑` and hide "Network" with Space, Then the main menu reflects it after save; "Settings" cannot be hidden (control disabled with explanation).
- *As a user I want to rename entries.* When I set label "Files" → "Browse", Then the menu shows "Browse".
- *As a user I want to add a custom command.* When I add id `lazygit`, label "Lazygit", command `lazygit`, kind `tui`, cwd `{cwd}`, Then it appears in the menu and runs in the browser directory; hotkey conflicts are reported inline.

#### Gate G6 — `v0.7.0`
Exit criteria: tomlkit round-trip tests preserve comments/order; every schema field reachable from the UI (test enumerates schema vs UI); migration test from a v1 fixture; manual demo of the stories above; backups rotate (keep 10).
Backlog refinement for M7: decide Q21, PyPI (Q2), and the v1.0 cut list.

---

### M7 — Hardening and v1.0 → `v1.0.0`

**Goal / increment:** a polished, documented, fast v1.0 that I can install anywhere with one command and recommend to others.

#### S7.1 — UX polish and robustness (M)
- Help overlay complete and generated from bindings; consistent wording of errors (catalogue in `core/errors.py`); small-terminal audit at 80×24 and 60×20 on every screen; ASCII fallback when not UTF-8; theme light/dark; startup re-measured and lazy imports audited; "operation running" guard on quit; crash handler that restores the terminal and writes the traceback to the log with a one-line message.
- *As a user I want miraCLI never to leave my terminal broken.* Given an unexpected exception anywhere, Then the terminal is restored, a short message tells me where the log is, and the exit code is non-zero.

#### S7.2 — Docs, packaging, release (M)
- README (install, quick tour, screenshots via Textual SVG export), `docs/config-reference.md` generated from the schema, `docs/keybindings.md`, CHANGELOG, `miracli --print-shell-hook bash` (if Q21 yes), release checklist, tag `v1.0.0`, optional PyPI publish via GitHub Actions trusted publishing.

#### Gate G7 — `v1.0.0`
- Exit criteria: [ ] full demo scripts of G0–G6 re-run on a clean Ubuntu 24.04 VM via `pipx install git+…@v1.0.0`; [ ] also tested over SSH and inside tmux; [ ] startup ≤ 300 ms median; [ ] zero open "bug" issues of severity high; [ ] docs reviewed by you.
- Decision options: Release / Release with known issues (listed in CHANGELOG) / Rework S7.x.

---

## 7. Risk register

| # | Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | Startup exceeds 300 ms (Textual + pydantic import cost) | M | M | Measure in S0.1; lazy imports of features/openai/tomlkit/send2trash; msgspec fallback; prompt_toolkit fallback decided at G0.1; CI import-time report | M0, M7 |
| R2 | Textual API changes between releases | M | M | Pin `textual>=X.Y,<X.(Y+1)` style range; upgrade deliberately in one PR with snapshot tests | all |
| R3 | Scope creep from [TBD] features and extras | H | M | Discovery gates; explicit scope-out lists; backlog for post-v1 | M4, M5 |
| R4 | Slow or janky lists for very large directories | M | M | Threaded scandir, lazy stat, virtualized `DataTable`, 20k-entry perf test; custom Line-API widget fallback | M1 |
| R5 | Terminal hand-over glitches (garbage after return, mouse/alt-screen not restored, resize while suspended, Ctrl+C killing miraCLI) | M | H | `App.suspend`, SIGINT guard, forced full repaint & size re-read on resume, manual matrix (GNOME Terminal, kitty, tmux, SSH) | M0 |
| R6 | Data loss from file operations bugs | L | **H** | Trash default, confirmations, exclusive creates, temp+`os.replace` overwrites, never delete source before verified copy, extensive tmp_path tests incl. cancellation | M2 |
| R7 | Partial state after cross-filesystem move failure/cancel | M | H | Copy-verify-delete, rollback only of what we created, report listing | M2 |
| R8 | Trash not available on some mounts (network fs, removable media without permissions) | M | M | Detect failure, explain, offer permanent delete with second confirmation | M2 |
| R9 | Odd filenames (invalid UTF-8, newlines, Rich markup) break display or operations | M | M | Operate on raw `os` names, sanitized display names, `Text` not markup, dedicated tests | M1 |
| R10 | Git credential/host-key prompts hang or are invisible | H | M | `GIT_TERMINAL_PROMPT=0` + `BatchMode=yes` in streamed mode, error classifier, interactive fallback | M3 |
| R11 | Special filesystems (`/proc`, `/sys`, FIFOs, sockets) behave oddly (size 0, blocking reads on copy) | M | M | Kind `other` not copied by default (skipped with report); never open FIFOs for reading | M1, M2 |
| R12 | Git progress output format differences across versions | L | L | Tolerant regex, fixtures from several git versions, fall back to raw last line | M3 |
| R13 | Corrupting `~/.bashrc` | L | **H** | Option (b) touches it once, backups, `bash -n` check, atomic replace, tests with real bash in temp `HOME` | M4 |
| R14 | API key leakage (config, logs, crash dumps) | L | H | Key only from env/keyring; logging redaction filter; test that greps logs | M5 |
| R15 | Config schema churn breaking user files | M | M | `schema_version` + migrations + backups; never crash on invalid config | M0, M6 |
| R16 | GUI launches unavailable over SSH/TTY | H | L | Display detection, tmux fallback, clear message | M0 |
| R17 | Part-time capacity lower than planned | M | M | Small stages, each gate shippable, re-plan at every gate | all |

---

## 8. Testing strategy

| Layer | Tooling | What and how |
|---|---|---|
| Unit — core | `pytest`, `hypothesis` for property tests | Pure functions: sort keys, name validation, keep-both naming, placeholder substitution, URL building/validation, git progress parsing, error classification, config validation/migration, env quoting. No Textual import. Coverage gate ≥ 85 % for `miracli.core`. |
| Filesystem | `pytest` `tmp_path`, real filesystem | Every `FileOps` operation × file/dir/symlink/broken symlink/unreadable (chmod 000; skipped when running as root)/invalid UTF-8 names; cancellation via an `Event` set from a progress callback after N bytes; conflict policies with a stub `ConflictResolver`; cross-fs via `/dev/shm` when it is a different device (`os.stat().st_dev`), else monkeypatched `os.rename` raising `OSError(errno.EXDEV)`; Trash via `send2trash` with `XDG_DATA_HOME` pointed into `tmp_path`. |
| Git | real `git` binary, local repos | Fixture creates a bare repo with two branches in `tmp_path` and clones via `file://` (exercises real progress output, `--branch`, existing-target errors). Auth/network failures tested with recorded stderr fixtures against the classifier and with a fake `ProcessRunner`. No network in CI. |
| Launcher | fake `ProcessSpawner` + real tiny commands | Mode routing, `which` failures, early-exit detection, env passing; real `true`, `sh -c 'exit 3'`, `sleep 5` (GUI-mode reaping). Hand-over verified with a fake `TerminalHandover`; real suspend covered by the manual demo matrix. |
| LLM | fake `ChatClient`; `respx` for the openai adapter | UI tests use a fake async generator (configurable delays/errors). The real adapter is tested by mocking the `httpx` transport with `respx`, returning SSE `data: {...}` chunks and error status codes (401, 404, 429, 500), so the real `openai` SDK parsing runs. |
| Env (M4) | real `bash` in temp `HOME` | Generate file, `bash -c '. env.sh; printf %s "$VAR"'`, compare with hypothesis-generated values; hook idempotency. |
| TUI | Textual `App.run_test()` + `Pilot`, `pytest-textual-snapshot` | Screens instantiated with an `AppContext` of fakes; `pilot.press("down","right","enter")`, assert on state and widgets; SVG snapshots at 80×24 (and 120×40 for wide layouts) for main menu, browser, menus, dialogs, clone form; snapshots reviewed in PRs. |
| Config | `tmp_path` + fixtures | Defaults, first-run file creation, invalid values → issues with line numbers, migration fixtures per schema version, tomlkit round-trip preserving comments. |
| Performance | `pytest` marker `perf` | 20k-entry listing; startup import time (`python -X importtime`) report in CI; hard limits checked locally at gates. |
| Architecture | `import-linter` | `miracli.core` and `miracli.platform` must not import `textual` or `miracli.features`. |
| Manual | gate demo scripts | Run on GNOME Terminal, kitty, inside tmux, over `ssh localhost`, at 80×24. |

CI (`.github/workflows/ci.yml`): `ubuntu-24.04`, Python 3.10/3.11/3.12/3.13 → `uv sync --frozen` → `uv run ruff format --check` → `uv run ruff check` → `uv run mypy` → `uv run lint-imports` → `uv run pytest -m "not perf" --cov` ; separate jobs: `pipx install . && miracli --version` smoke, perf (non-blocking).

---

## 9. Definition of Done (applies to every story)

1. Acceptance criteria demonstrated (by test where automatable, otherwise in the gate demo script).
2. Core logic in `miracli.core`/`platform` with unit tests; TUI glue has at least one Pilot test for the happy path and one for the main error path.
3. `ruff format`, `ruff check`, `mypy` and `lint-imports` clean; CI green on all Python versions.
4. Type hints on all public functions; no `Any` leaking from core APIs.
5. Every user-triggerable error path shows a readable message (what happened + what to do) and is logged; no traceback on screen.
6. No destructive action without confirmation; any modified user file has a backup first and is written atomically.
7. Works at 80×24, survives resize, keyboard-only; new bindings appear in the footer or help overlay and in `docs/keybindings.md`.
8. Long operations (> 300 ms) run in workers, show progress or a spinner, and are cancellable where meaningful.
9. New config keys: in the schema with defaults, documented with a comment in the generated default config, validated, migration added if the shape changed.
10. No secrets in config, logs or snapshots.
11. Startup budget not regressed (import-time report checked when dependencies or imports change).
12. CHANGELOG entry under "Unreleased"; commit messages describe the change.

---

## Next step

Please review **§2 Open questions** (answer them or reply "defaults OK" for any subset) and approve or amend the plan — in particular the milestone restructuring in §6.0, the framework recommendation in §3.1, and the browser/context-menu key choices in §5. I will not start M0 until you confirm.
