# Taskline: notes for AI agents

Taskline is a task manager (icons-only or labeled buttons) plasmoid for KDE Plasma 6.5+, the
user's continuation of Fancy Tasks NG (forked from upstream 2.2.0, with selected 2.3.0 changes
ported by hand). Plugin Id `io.github.divinae.taskline` (`package/metadata.json`), so it installs
side by side with Fancy Tasks NG. Owner Jakub Lipiński (GitHub `divinae`). Published on GitHub at
https://github.com/divinae/taskline-plasmoid (releases carry `Taskline.plasmoid`); no KDE Store
entry (the upstream store workflow was removed). Remotes: **`fork` = divinae/taskline-plasmoid,
the user's; push there**, branch `master`, committed directly. `origin` = upstream
DayDve/FancyTasksNG: fetch only, never push. Local `backup/*` and `fix/*` branches are the
user's; leave them. QML/JS plus two Python helpers; nothing compiled.

## Layout

- `package/metadata.json`: KPlugin metadata; `Version` is the release version (see Release flow).
- `package/contents/config/main.xml`: kcfg schema (~120 entries, one group `General`).
  `config.qml`: pages Appearance, Behavior, Indicators, Advanced, Pinned Applications
  (`Config*.qml`, built from Kirigami Addons FormCards via `ConfigFormPage.qml`/`ConfigPage.qml`).
  `ConfigPage.qml` holds the `cfg_<key>Default` properties (generated, see Gotchas).
  `LivePreview.qml` is the preview in the settings.
- `package/contents/ui/main.qml` (~1360 lines): the `PlasmoidItem`, TasksModel, sorting, layout.
  `Task.qml` (one task), `TaskList.qml`, `ContextMenu.qml`, `Indicators.qml`, `ToolTip*.qml`
  (tooltip, thumbnails via `PipeWireThumbnail`/`X11Thumbnail`, media bar), `Attention*.qml`
  (bounce scheduler, screen-edge peek window), `AudioStreamManager.qml`, `PlayerController.qml`.
- `package/contents/ui/code/`: `tools.js`, `layoutmetrics.js`, `media.js`, `FloatingLogic.js`;
  `singletones/` (sic) with `qmldir`: `Wrappers` (i18n), `BadgeManager`, `DesktopActionsManager`.
- Python helpers in `ui/code/`: `desktop_actions.py` (jump lists, recent docs, browser history,
  Open at Login, Force Quit, config export/import; D-Bus `io.github.divinae.taskline.DesktopActions`
  and `.Config`), `unity_bridge.py` (Unity LauncherEntry badges, `.Bridge`), shared
  `helper_common.py`. Started by QML through a `Plasma5Support.DataSource` "executable" engine;
  need `python3` with dbus-python and PyGObject.
- `tests/test_desktop_actions.py`: unittest for the pure functions of `desktop_actions.py`.
- `tools/`: `build.sh`, `install.sh`, `update.sh`, `uninstall.sh`, `testing.sh` (preview),
  `diag.sh`, `functions.sh` (shared bash), `sync_defaults.py`.
- `.github/workflows/ci.yml` (lint + test on every push/PR), `release.yml` (see Release flow).
- Ignored/untracked: `release/`, `build*/`, `.ruff_cache/`, `taskline-diag.txt`.

## Build, run, test

Everything via `make` (Makefile → `tools/*.sh`):

- `make test`: `python3 -m unittest discover -s tests`.
- `make lint`: ruff (`--select E9,F`) on `package/contents/ui/code tools tests`, shellcheck
  `--severity=warning tools/*.sh`, `python3 tools/sync_defaults.py --check`, JSON check of
  `metadata.json`. CI runs exactly `make lint` and `make test`.
- QML has no automated lint; the PR template asks for `qmllint` on changed files with no new
  warnings (existing `// qmllint disable` pragmas mark known noise).
- `make preview`: runs `sync_defaults.py`, then `plasmawindowed package/` from the source tree
  (QML disk cache off). Does not touch the installed copy or the panel.
- `make build`: `release/Taskline.plasmoid` (zip of a version-stamped copy).
- `make install`: first install, via `kpackagetool6 --install` of a stamped copy.
  `make update`: `kpackagetool6 --upgrade package/` **and restarts plasmashell**
  (systemd unit or `plasmashell --replace`). `make uninstall`, `make clean`, `make diag`.
- Seeing a change in the real panel needs `make update` (restarts the shell); do it only when the
  user asks. The Python helpers are long-running processes, so changes to them also need a
  plasmashell restart. QML errors: `journalctl --user -b --since "-5min" | grep -i -E "taskline|qml"`.

## Conventions

- QML: 4-space indent, `import org.kde.kirigami as Kirigami` style aliases, comments explain
  *why* in full sentences. Keep logic out of bindings that re-evaluate on every model change
  (several refactors were about that; see "Assign only on change" comments in `main.qml`).
- Python: stdlib + dbus/GLib only, pure helpers kept testable; add a test in `tests/` for new
  pure logic. Must pass `ruff --select E9,F`.
- Settings: a new option needs a `main.xml` entry, a control on a `Config*.qml` page with
  `cfg_<key>`, and its `cfg_<key>Default` in `ConfigPage.qml` (run `python3
  tools/sync_defaults.py`; `make lint` fails otherwise).
- i18n: the widget is English-only since 1.0.0 (catalogs and `make translate` removed), but UI
  strings still go through `Wrappers.i18n/i18nc/i18np/i18ncp(...)` (domain
  `plasma_applet_io.github.divinae.taskline`); `config.qml` uses plain `i18n()`. Don't add `po/`.
- SPDX: files carry `SPDX-FileCopyrightText` / `SPDX-License-Identifier` headers inherited from
  upstream (mostly GPL-2.0-or-later, some LGPL); keep them. New files: a
  `SPDX-License-Identifier: GPL-2.0-or-later` block like `AttentionPeek.qml`. Note README and
  metadata say GPL-3.0-or-later; don't "fix" headers without asking.
- CHANGELOG.md: Keep a Changelog. Add user-visible changes under `## [Unreleased]` as
  `- **Bold Lead:** prose sentence.` in `### Added/Changed/Fixed/Removed`.
- Commits: Conventional style with optional scope, lowercase, plain-English description:
  `fix(tooltip): ...`, `feat: ...`, `refactor(helper): ...`, `docs:`, `chore:`, `build:`,
  `release: Taskline X.Y.Z`. Body explains the why. AI-assisted commits end with
  `Co-Authored-By:`.

### Release flow

The committed `package/metadata.json` `Version` changes only in a release. `build.sh`/`install.sh`
stamp the *copy* with `git describe --tags` on non-release commits (`resolve_display_version`
in `functions.sh`); never commit a stamped version. Release: move `[Unreleased]` to
`## [X.Y.Z] - YYYY-MM-DD`, bump `Version`, commit `release: Taskline X.Y.Z`, tag `vX.Y.Z`
(existing: `v1.0.0`), publish a GitHub release; `release.yml` builds `Taskline.plasmoid` and takes
the notes from that CHANGELOG section (matched by `[X.Y.Z]`). Only when the user asks.

## Gotchas

- Differences from upstream that must stay: Id `io.github.divinae.taskline`, D-Bus names
  `io.github.divinae.taskline.*`, i18n domain, no translations, no KDE Store/nightly workflows,
  FUNDING pointing to the user's Ko-fi, FormCard settings pages. Compatibility kept on purpose:
  config import accepts Fancy Tasks NG exports (`io.github.daydve.fancytasksng` in
  `ConfigAdvanced.qml`), autostart entries with the `X-FancyTasksNG-Autostart` marker are treated
  as ours, `FancyTasksExplosion.qml` keeps its name.
- Upstream is not merged: 2.3.0 changes were ported commit by commit (some deliberately partial,
  e.g. audio attribution "without the registry"). Bring further upstream changes the same way,
  after comparing with `git log origin/master`; don't merge or rebase onto `origin`.
- `cfg_*Default` values in `ConfigPage.qml` drive the settings dirty-tracking; if they drift from
  `main.xml`, pages show wrong "changed" state and can rewrite user config on first Apply.
- `make update` upgrades straight from `package/` (no version stamp, no defaults sync).
- "Do not sort" runs on `SortManual` (comments in `main.qml`); pinned-app order and restore after
  restart went through several fixes, read the history before touching sorting.
- Helpers fail soft: without dbus-python/PyGObject the features vanish and the Advanced page says
  why. Helpers must survive install paths with spaces/quotes (QML shell-quotes the script path).
- Media in browsers is attributed by window title and site (no per-tab info); see the comments in
  `ToolTipDelegate.qml` and `AudioStreamManager.qml` before changing it.
- README's install says `cd Taskline` after cloning; the directory is `taskline-plasmoid`.

## Related

- uSwitch (`../uswitch-plasmoid`) and other `divinae` widgets share the Ko-fi
  `.github/FUNDING.yml` and README layout (badges, "Why you'll like it", Support, Bugs).
  No shared code with the sibling widgets is evident.
