# comment_preserving_ini Multi-line Value Handling

This document captures the multi-line value handling rules and the
work done in #10 to make them match ConfigParser's semantics exactly.

## Status: Resolved (#10 — landed)

What was previously a "known limitation" is now solid. The parser
now matches ConfigParser's `empty_lines_in_values=True` semantics
for multi-line values, including:

  * Internal blank lines kept as `\n` in the joined value
  * Trailing blank lines stripped
  * Indented `#` comment lines skipped from value (preserved in
    raw_lines for byte-identity on unchanged emit)
  * Repeated `set_value` calls correctly replace (no concatenation)
  * `delete_section` + `add_section` produces a clean section

## What rules apply to multi-line values

The parser collects continuation lines following a `key = ...` line
until it hits one of:

  * A new key (non-indented `key = ...` line)
  * A section header (`[name]`)
  * EOF

Within that range:

  * **Indented non-blank, non-comment lines** → value content
  * **Blank lines** → kept as empty value-parts (rendered as `\n`
    in the joined value), but trailing blanks are stripped
  * **Indented `#` comment lines** → SKIPPED in value, preserved
    in raw_lines so byte-identity holds on unchanged emit

This matches Python's `configparser.ConfigParser` exactly. So the
GUI-save round-trip (cpi.parse → ConfigParser-flavored dict →
cpi.apply_dict → cpi.write) is now byte-clean for multi-line
values, including those with internal blanks.

## What still doesn't work — by design

These ARE actually ConfigParser limitations, not bugs in our code:

  * **Markdown `###` headers inside values** — ConfigParser treats
    any indented line starting with `#` as a comment and drops it
    on read. Use `**bold**` or other markdown features that don't
    start with `#`.

  * **Inline `;` followed by content** — `;` is the inline-comment
    delimiter. `value ; comment` parses as value=`value` with a
    trailing inline comment. Not a multi-line concern, but worth
    noting.

## Reference: round-trip test coverage

The fix is regression-protected by:

  * `harness/test_comment_preserving_multiline.py` (10 synthetic
    edge-case tests, all passing)
  * `harness/test_gui_save_roundtrip.py` extended with
    `test_mutate_multiline_via_gui_path` that mutates the first
    multi-line value found in each toolkit INI file (22 files,
    all passing)
  * `harness/test_comment_preserving_ini.py` (existing
    identity/mutate/append/addsect/delkey suite, all passing)
  * The full toolkit's own `harness/run_all.py` (5 suites, all
    passing)

## Implementation notes

The fix touches four places in `schema_lib/comment_preserving_ini.py`:

1. **`_is_continuation`** — now returns True for blank lines (was
   only True for indented non-blank lines). The "stop" decision
   is centralized in the collection loop instead.

2. **`_is_indented_comment`** — new helper that detects indented
   `#` comments so the collection loop can skip them from value
   while preserving them in raw_lines.

3. **The collection loop in `parse()`** — rewritten to:
   - Scan forward through indented lines, blank lines, and
     indented comments
   - Stop only at a non-indented non-blank line (key, section
     header, EOF)
   - Build value_parts with blank lines as empty strings and
     indented `#` lines skipped
   - Strip trailing blank value_parts before joining
   - Track raw_lines for ALL collected lines (preserves
     byte-identity on unchanged emit)
   - Track continuation indent only from real value lines (not
     blanks/comments)

4. **`delete_section`** — now also marks all child KeyValue items
   as deleted, so a subsequent `add_section` + `set_value` chain
   produces a clean section (rather than retaining old keys).
