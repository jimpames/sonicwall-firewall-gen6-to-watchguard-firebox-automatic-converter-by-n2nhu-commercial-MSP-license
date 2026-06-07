# Schema-INI Dialect

Each config INI in this toolkit (e.g. `concept_map.ini`, `parser_config.ini`)
gets a sibling schema file (e.g. `concept_map.schema.ini`) that:

1. **Validates** the config — typos, missing required keys, wrong types
2. **Documents** the config — every field has a `help` string
3. **Drives the GUI** — the editor renders widgets based on field types

The schema is just another INI. Standard parser, standard comment style.

## Schema structure

A schema INI has:

### `[meta]`
Global info about the config this schema describes.

```ini
[meta]
config_name = concept_map.ini
description = Vendor-neutral concept map; the durable Rosetta Stone.
version     = 1.0
```

### `[section.<name>]` — describes a single, fixed-name section
The corresponding config file is expected to have a `[<name>]` section with
the fields declared here.

```ini
[section.meta]
description = Header info: name, version, description.
required    = true
fields      = name, version, description

[section.meta.field.name]
type     = string
required = true
help     = Human-readable name for this concept map.
```

### `[section_pattern.<name>]` — describes a family of repeating sections
For configs where users add many sections of the same shape (e.g. one
`[value_map.<X>]` per value translation table). The matching rule lives
in `match_pattern`.

```ini
[section_pattern.value_map]
match_pattern = ^value_map\.([\w_]+)$
description   = One value-translation table per section.
fields        = description, sonicwall_to_canonical, canonical_to_watchguard

[section_pattern.value_map.field.canonical_to_watchguard]
type     = kv_list
required = true
help     = Comma-separated key=value pairs translating canonical names to WG values.
```

## Field attributes

All field types support:

- `type` (required) — see types below
- `required` (default `false`) — must be present in the config
- `default` — value used if the field is absent
- `help` (recommended) — operator-facing description shown in GUI tooltip
- `since_version` (optional) — config version this field was introduced in

## Field types

| Type              | Validates                                       | GUI widget                  |
|-------------------|-------------------------------------------------|-----------------------------|
| `string`          | any single-line string                          | text entry                  |
| `multiline_string`| string that may span lines                      | text area                   |
| `int`             | parses as integer; optional `min` / `max`       | spinbox                     |
| `bool`            | `true`/`false`/`1`/`0`/`yes`/`no`              | checkbox                    |
| `enum`            | one of `enum_values`                            | dropdown                    |
| `csv`             | comma-separated list; `csv_item_type` per item  | list editor                 |
| `regex`           | compiles as a Python regex                      | text entry + live test      |
| `xpath`           | well-formed XPath expression                    | text entry                  |
| `path`            | filesystem path; `path_kind` = file / dir       | text + file picker          |
| `kv_list`         | comma- or newline-separated `k=v` pairs         | two-column grid editor      |

## Example: the smallest useful schema

```ini
[meta]
config_name = concept_map.ini

[section.meta]
required = true
fields   = name, version

[section.meta.field.name]
type     = string
required = true
help     = Human-readable name for this concept map.

[section.meta.field.version]
type     = string
default  = 1.0.0
help     = Semantic version.
```

## Validation behavior

When an engine loads a config:

1. The validator finds the schema (`<config>.schema.ini` next to it).
2. For each section in the config, the validator finds a matching `[section.X]`
   or `[section_pattern.X]` in the schema.
3. For each field in the section, the validator checks the type and any
   constraints. Required-but-missing fields produce errors.
4. Unknown sections (no schema match) and unknown fields (no field declaration)
   produce warnings — they may be operator additions ahead of the schema, or
   they may be typos.

If the schema is missing entirely, the engine logs a single warning and
proceeds without validation. The toolkit must still run on configs without
schemas (we'll be migrating gradually).

## Forward compatibility

When a new directive is added to an engine, its schema entry is added in
the same commit. CI runs schema validation against every config in the
repo as a regression test. Drift is caught immediately.
