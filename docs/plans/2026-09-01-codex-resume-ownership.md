# Codex Resume Ownership Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Ensure farm manifests resume independently writable Codex root or fork threads, never multi-agent child threads, and handle active writer conflicts without launching a doomed TUI.

**Architecture:** Add a small Python metadata helper that reads only Codex `session_meta`, maps true `source.subagent` rollouts to their live-session root `session_id`, and leaves ordinary forks on their own thread ID. Use it at both save and restore boundaries. Before restore launches an exact Codex resume, use Codex's advisory writer-lock file for a bounded availability wait; a persistent owner is reported as an incomplete restore instead of starting a second writer.

**Tech Stack:** Python 3.10+, Bash, tmux pane metadata, `unittest`, advisory file locks.

### Task 1: Capture Codex thread semantics in tests

**Files:**
- Create: `tests/test_codex_session_meta.py`
- Modify: `tests/test_session_hook.py`

1. Add fixtures for root, independent fork, and multi-agent child `session_meta` records.
2. Assert root and fork IDs remain unchanged while child IDs resolve to `payload.session_id`.
3. Assert an active writer lock is distinguishable from a stale, unlocked lock.
4. Assert the pane hook ignores subagent `UserPromptSubmit` payloads containing `agent_id` or `agent_type`.
5. Run the focused tests and verify they fail because the new behavior is absent.

### Task 2: Implement metadata and writer ownership helper

**Files:**
- Create: `bin/codex-session-meta.py`
- Test: `tests/test_codex_session_meta.py`

1. Parse bounded `session_meta` input without reading transcript events.
2. Resolve a rollout path or UUID to its resumable identity, preserving independent forks.
3. Poll the per-thread advisory writer lock for a bounded configurable interval.
4. Run the helper tests and verify they pass.

### Task 3: Prevent child identity poisoning during save

**Files:**
- Modify: `bin/codex-session-hook.py`
- Modify: `bin/codex-save`
- Modify: `tests/test_session_hook.py`
- Modify: `tests/test_add_scripts.py`

1. Reject Codex subagent hook events before changing pane options.
2. Canonicalize Codex pane metadata and rollout-file candidates through the helper.
3. Add save regressions for a child hook ID and for a child rollout encountered before its parent.
4. Run the focused hook/save tests and verify green.

### Task 4: Repair old manifests and handle writer conflicts

**Files:**
- Modify: `bin/codex-restore`
- Modify: `tests/test_add_scripts.py`

1. Resolve exact Codex IDs before launch, so existing child-poisoned manifests resume the root.
2. Wait briefly for the exact thread's writer lock to clear, covering tmux shutdown races.
3. Skip a persistently owned thread, report actionable guidance, continue other rows, and exit nonzero to mark the restore incomplete.
4. Seed only the resolved resumable ID into the new pane.
5. Run restore regressions for repaired child manifests, delayed lock release, and persistent ownership.

### Task 5: Document, verify, and integrate

**Files:**
- Modify: `README.md`

1. Explain root/subagent identity repair and the active-writer limitation after in-TUI `/fork`.
2. Document that users can switch back with `/resume` in the same TUI or exit the owning TUI before resuming elsewhere.
3. Run focused tests, the full unit suite, shell syntax checks, and inspect the final diff.
4. Commit the implementation and integrate it into `main` after review.
