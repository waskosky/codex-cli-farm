# Deep-History Numeric Boundary Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Prevent the disk-history popup from opening until copy mode has numerically reached the top of tmux's retained history.

**Architecture:** Keep the compatibility launcher's interception of the pinned plugin's Page Up binding, but replace the plugin's equality predicate with tmux's integer greater-than-or-equal expression. Advance the farm integration marker so existing tmux servers reinstall the corrected binding, and require a launcher-version handshake so stale installed launchers cannot falsely claim the new marker.

**Tech Stack:** Bash 3.2-compatible launcher and farm scripts, embedded Python 3.10+, tmux 3.2+ formats, Python `unittest`, shell integration tests.

### Task 1: Add regression coverage for the generated binding

**Files:**

- Modify: `tests/test_deep_history_installer.py`
- Modify: `tests/test_add_scripts.py`
- Modify: `tests/integration/deep_history_smoke.sh`

**Step 1: Expose fake upstream binding arguments**

Extend the fake pinned archive so its `Tmux.bind()` records arguments and its
fake CLI can ask the launcher to bind the upstream predicate:

```python
Tmux().bind("if-shell", "-F", "#{==:#{scroll_position},#{history_size}}")
```

Add a stale-launcher setup fixture whose existing tmux marker is `3:on`.

**Step 2: Assert the desired numeric rewrite**

Add a focused installer test expecting:

```tmux
#{e|>=:#{scroll_position},#{history_size}}
```

Update setup and smoke assertions to expect integration marker `4` and the same
numeric predicate in both copy-mode tables. Require the launcher to report farm
integration version 4, and verify a stale launcher cannot advance the marker.

**Step 3: Run tests to verify RED**

Run:

```bash
python3 -m unittest -v \
  tests.test_deep_history_installer \
  tests.test_add_scripts.AddScriptsTests.test_codex_add_auto_uses_deep_history_as_single_pipe_owner
```

Expected: FAIL because the launcher emits plain `#{>=:...}` and `codex-add`
still records integration marker `3`.

**Step 4: Commit the failing tests**

```bash
git add tests/test_deep_history_installer.py tests/test_add_scripts.py tests/integration/deep_history_smoke.sh
git commit -m "test: cover numeric deep-history boundary"
```

### Task 2: Generate a numeric boundary and refresh existing bindings

**Files:**

- Modify: `integrations/tmux_deep_history_launcher.sh`
- Modify: `bin/codex-add`
- Modify: `README.md`

**Step 1: Implement the minimal predicate fix**

Change the compatibility rewrite to:

```python
live_boundary = "#{e|>=:#{scroll_position},#{history_size}}"
```

Update its comment to state that the expression must use tmux integer
comparison.

**Step 2: Advance the binding integration marker**

Set:

```bash
desired_integration="4:$DEEP_HISTORY_SEAMLESS_PAGEUP"
```

This makes the next `codex-add` pass invoke plugin installation and replace
bindings previously installed with marker 3.

**Step 3: Guard the marker with a launcher capability handshake**

Have the compatibility launcher answer `codexfarm-integration-version` with
`4`. Before enabling seamless Page Up, require that response in `codex-add`.
When an older launcher is detected, do not write marker 4; disable plugin
auto-start, restore native Page Up for farm-managed bindings, use legacy logging
for the new pane, and print the `setup.sh --with-deep-history` recovery command.

**Step 4: Document the update command**

Clarify in the README that deep-history users should include
`--with-deep-history` when rerunning setup after an update.

**Step 5: Run focused tests to verify GREEN**

Run the Task 1 command again.

Expected: PASS.

**Step 6: Commit the implementation**

```bash
git add integrations/tmux_deep_history_launcher.sh bin/codex-add README.md
git commit -m "fix: compare deep-history boundary numerically"
```

### Task 3: Verify the complete change

**Files:**

- Verify only; no planned modifications.

**Step 1: Run focused integration tests**

```bash
python3 -m unittest -v tests.test_deep_history_installer tests.test_add_scripts
```

Expected: PASS.

**Step 2: Run repository checks**

```bash
ruff format --check .
ruff check .
while IFS= read -r file; do bash -n "$file"; done < <(git grep -l '^#!/usr/bin/env bash')
CODEX_ANNOTATOR_AUTOSTART=0 python3 -m unittest discover -s tests
VALIDATE_SKIP_TMUX=1 ./validate.sh
```

Expected: all checks pass.

**Step 3: Inspect the final diff**

```bash
git diff --check main...HEAD
git diff --stat main...HEAD
git status --short
```

Expected: only the planned launcher, marker, tests, and plan documents differ.
