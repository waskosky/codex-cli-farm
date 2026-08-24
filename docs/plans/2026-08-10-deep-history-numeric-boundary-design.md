# Deep-History Numeric Boundary Design

## Problem

The farm rewrites tmux-deep-history's absolute-top predicate from equality to
greater-than-or-equal so a pane that continues producing output can still hand
off to disk history. The rewrite currently uses:

```tmux
#{>=:#{scroll_position},#{history_size}}
```

tmux's plain comparison modifier compares strings. With 177 retained history
lines, the first Page Up can report a scroll position of 22; tmux therefore
evaluates `"22" >= "177"` as true and opens the disk-history popup long before
copy mode reaches the top.

## Chosen behavior

Keep the live-output greater-than-or-equal safeguard, but express it through
tmux's integer expression modifier:

```tmux
#{e|>=:#{scroll_position},#{history_size}}
```

Page Up continues to page through tmux's in-memory history while the numeric
scroll position is below the numeric history size. At or beyond the absolute
top, the next Page Up opens the older disk transcript.

## Integration and rollout

The compatibility launcher will continue to rewrite the pinned plugin's
equality predicate when the plugin installs its key bindings. The farm-managed
integration marker will advance from version 3 to version 4 so an existing tmux
server does not retain the old lexicographic binding after the launcher is
updated. Before enabling seamless Page Up, `codex-add` asks the installed
launcher for its farm integration version. A stale launcher cannot claim marker
4: the farm restores native Page Up in its managed copy-mode tables, uses legacy
logging for the new pane, invalidates the marker so a repaired launcher must
reinstall its bindings, and tells the user to rerun setup with deep history.

## Testing

- Exercise the compatibility wrapper against a fake upstream `Tmux.bind()` and
  assert that it emits the numeric predicate.
- Update farm setup tests to require integration marker 4.
- Start from a stale marker-3 launcher and verify it cannot claim marker 4 or
  leave the farm-managed Page Up bindings active.
- Update the deep-history tmux smoke test to require the numeric predicate in
  both copy-mode key tables.
- Run the focused installer/setup tests, shell validation, and the complete unit
  suite.

## Alternatives rejected

- Reverting to equality avoids the string-ordering bug but restores the missed
  handoff when live pane output makes `scroll_position` exceed `history_size`.
- Adding a cursor-position gate duplicates tmux state and is more fragile than
  comparing the two intended numeric coordinates directly.
