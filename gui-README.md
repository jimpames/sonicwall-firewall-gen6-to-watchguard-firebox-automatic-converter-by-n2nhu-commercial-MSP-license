# Schema-Driven INI Editor

A GUI editor for the toolkit's INI configuration files. Reads each config's
schema sibling and produces type-aware editing widgets.

## Running

```bash
# Edit a specific config file
python3 gui/ini_editor.py concept_map/concept_map.ini

# Or open the file picker
python3 gui/ini_editor.py
```

The editor automatically looks for `<config>.schema.ini` next to the
config file. To specify explicitly:

```bash
python3 gui/ini_editor.py path/to/config.ini --schema path/to/schema.ini
```

## What you get

- **Section list (left pane)** — every section in the file, with its field
  count. Click to edit. Sections without schema declarations show "(unknown)".
- **Field editor (right pane)** — per-section form. Each field has:
  - A label (with `*` if required)
  - The schema-declared type next to it
  - The `help` text from the schema
  - A type-appropriate widget: text entry, dropdown, checkbox, multiline
    text area, etc.
- **Validation panel (bottom)** — live validation results. Click
  **Validate** to re-run; auto-runs on Save.

## Adding sections and fields

For schemas that declare `[section_pattern.X]` (e.g. one
`[value_map.<name>]` per value-map), click **Add section** and supply
the new name. The editor renders a fresh form using the pattern's
field declarations.

For schema sections with `dynamic_keys=true` (e.g. `crypto_audit.X`,
which accepts arbitrary algorithm-name keys), you'll see an "Add
dynamic key" affordance at the bottom of the section's editor. Type a
key name, click Add, and the new row appears for editing.

Removing a section: select it in the tree, click **Remove section**.

Removing a dynamic field: each dynamic field has a small `✕` button.

## What "schema-driven" means in practice

This editor knows nothing about WatchGuard, SonicWall, or any specific
config in this toolkit. It reads schemas. To add a new editable config:

1. Write its schema (`mything.schema.ini`) following
   `docs/schema_format.md`.
2. Open it in the editor: `python3 gui/ini_editor.py mything.ini`.

No GUI code changes. That's the whole point.

## Limitations (current scope)

The editor is intentionally minimal in this first cut. Known gaps:

- **CSV and kv_list use plain multiline text** rather than two-column
  grids. Functional but not pretty. Easy follow-up.
- **No undo/redo.** Use Reload to discard changes.
- **No diff view between current and saved.**

These are nice-to-haves; the editor is safe for production use of the
toolkit's INIs.

## Comment preservation

The editor uses a custom round-trip writer (`schema_lib.comment_preserving_ini`)
that preserves block comments, blank lines, inline `; comments`, and
the original `=` separator/indent style. Opening a config in the GUI
and saving with no changes is guaranteed to produce a byte-identical
file. When you mutate values, surrounding documentation comments
survive untouched, and new keys land at the section's natural end
(after the last existing key, before any trailing dividers). The
round-trip behavior is verified by `harness/test_comment_preserving_ini.py`
and `harness/test_gui_save_roundtrip.py`, which run all 11 toolkit
INIs through five mutation patterns each.

## Schema dialect reference

See `docs/schema_format.md`.
