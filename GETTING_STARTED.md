# Getting Started — SonicWall → WatchGuard Migration Toolkit

Welcome! This guide walks you through running the toolkit end-to-end on
the included sample SonicWall config, then explains how to point it at
your own config and tune the rules. You should be productive in about
20 minutes.

If you want the architectural reference, see `README.md`. This guide is
the "open the box and try it" companion.

## What this toolkit does (in one paragraph)

It converts a SonicWall CLI text export into a WatchGuard XML config
file, plus a `migration_notes.md` document that explains every decision
the tool made — what merged, what was filtered, what needs operator
review. Every translation rule lives in INI files, so you can tune the
output without touching code.

## What's in the box

```
migration_toolkit/
├── README.md                    Architecture reference
├── GETTING_STARTED.md           This file
├── sonicwall_parser/            Phase 1: SonicWall CLI → JSON
│   ├── input/                   Drop your .txt config here
│   ├── output/                  Generated JSONs land here
│   ├── parser_config.ini        Pattern matrix (one section per area)
│   └── parser_schema.ini        Schema for parser output
├── enricher/                    Phase 1.5: cross-reference / pivot / audit
│   ├── enrich_config.ini        Three pass kinds: pivot, derive, audit
│   └── output/                  Enriched JSONs
├── concept_map/                 The vendor-neutral Rosetta stone
│   └── concept_map.ini          Value-maps, crypto audit tables, terminology
├── xml_emitter/                 Phase 2: enriched JSON → WatchGuard XML
│   ├── xml_emit_config.ini      Emit rules — generic XML tree builder
│   ├── skeleton_config.ini      Master skeleton stitching
│   ├── make_skeleton.py         Skeleton builder (run once per Firebox model)
│   └── dry_run/                 Generated configuration.xml + migration_notes.md
├── wg_validator/                Standalone WatchGuard XML validator
│   ├── wg_validator.py          The validator engine
│   ├── schema_inferer.py        Bootstrap schema from reference XML
│   ├── wg_xml_schema.ini        Schema (773 element rules, hand-curated)
│   └── wg_xml_references.ini    Cross-reference rules (18 rules)
├── diff_scorer/                 Compare emit vs reference (5-dimensional score)
├── docs/                        Reference materials
│   ├── reference_exports/       Sample WatchGuard XML exports
│   ├── skeletons/               Generated master skeletons
│   ├── portshield_to_bridge.md  Operator playbook for portshield migration
│   └── PHASE2_TODO.md           Remaining work / known limitations
├── gui/                         GUI editor for INI configs
│   └── ini_editor.py            Schema-driven, type-aware
├── harness/                     Test suite
│   ├── run_all.py               Run all 5 test suites
│   └── test_*.py                Individual suites
└── schema_lib/                  The comment-preserving INI library
    ├── comment_preserving_ini.py    Round-trip writer (preserves comments)
    └── schema_lib.py                Schema loader / validator
```

## Prerequisites

- **Python 3.10 or newer** — the only requirement
- **Tkinter** (for the GUI) — usually comes with Python; on Linux you may
  need `apt install python3-tk` or `dnf install python3-tkinter`

No third-party packages are needed. The toolkit is intentionally
dependency-free.

## Quickstart: run on the included sample

The toolkit ships with a small SonicWall config in
`sonicwall_parser/input/slimmed-sonicwallconfig.txt`.

### The easy way: one-shot driver

```bash
cd migration_toolkit
python3 run_pipeline.py
```

That's it. The driver runs all four phases (parse → enrich → emit →
validate) against the bundled sample and shows you the output of each
step. Expected final lines:

```
  Pipeline complete!
  WatchGuard XML:   xml_emitter/dry_run/configuration.xml
  Migration notes:  xml_emitter/dry_run/migration_notes.md
```

To run it against your own SonicWall config:

```bash
python3 run_pipeline.py --input /path/to/your-firewall.txt
```

If you want to see what each phase does individually, the manual steps
follow.

### The manual way: step by step

### Step 1: Parse the SonicWall CLI export

```bash
cd migration_toolkit
python3 sonicwall_parser/sonicwall_parser.py \
    --config sonicwall_parser/parser_config.ini \
    --input sonicwall_parser/input/slimmed-sonicwallconfig.txt
```

Expected output (last few lines):

```
parsed 67 sections to sonicwall_parser/output/
manifest written: _manifest.json
unparsed lines: 0
```

This converts the SonicWall CLI text into about 60 JSON files, one per
logical config area (interfaces.json, address_objects.json,
access_rules.json, etc.).

### Step 2: Run the enricher

```bash
python3 enricher/enricher.py --config enricher/enrich_config.ini
```

This does three kinds of cross-reference work:

- **pivot** — derives address-objects from address-group references that
  weren't explicitly defined
- **derive** — adds computed fields (e.g. crypto findings on VPN
  policies)
- **audit** — flags problematic patterns (deprecated crypto, etc.)

Output lands in `enricher/output/`.

### Step 3: Build the master skeleton (one-time per Firebox model)

The skeleton is the empty WatchGuard config to merge into. This is
already done for the sample (file at
`docs/skeletons/master_skeleton.xml`), so you can skip this on first run.
For your own work, you'd do:

```bash
python3 xml_emitter/make_skeleton.py \
    --reference docs/reference_exports/T30jpa1__1_.xml \
    --out docs/skeletons/master_skeleton.xml
```

### Step 4: Emit the WatchGuard XML

```bash
python3 xml_emitter/xml_emitter.py --config xml_emitter/xml_emit_config.ini
```

Expected output:

```
  [confirmed ] interfaces                       emitted=   2  merged=2
  [advisory_only] portshield_advisory              advisory_for=1
  ...
Errors:           0
Warnings:         0
Info notes:       0
```

The `merged=2` means two SonicWall interfaces were merged into the
canonical WatchGuard zone interfaces (Trusted, External). The
`advisory_for=1` means one portshield directive was detected and a
migration note was written.

### Step 5: Inspect the output

Three files now exist in `xml_emitter/dry_run/`:

```bash
# The WatchGuard XML config (ready to import)
xml_emitter/dry_run/configuration.xml

# Human-readable explanation of every translation decision
xml_emitter/dry_run/migration_notes.md

# Validator report (0 errors, 0 warnings on the sample)
xml_emitter/dry_run/validation_report.txt
```

Open `migration_notes.md` first. It documents:

- What was filtered out (e.g. SonicWall-only IPv6 system aliases)
- What was merged (e.g. X0 LAN → Trusted, X1 WAN → External)
- What collided (e.g. multiple LAN ports — first-wins, others logged)
- What needs operator action (e.g. portshield → Bridge interface)
- What deprecated crypto was flagged (e.g. GVC tunnels with weak SHA1)

### Step 6: Validate the XML

The XML is automatically validated as part of step 4, but you can
re-run the validator manually:

```bash
python3 wg_validator/wg_validator.py \
    --candidate xml_emitter/dry_run/configuration.xml
```

## Try the test suite

```bash
python3 harness/run_all.py
```

Expected: **5 passed, 0 failed**. The tests cover the round-trip INI
writer, GUI save behavior, multi-line value handling, engine wiring,
and the WatchGuard XML validator.

## Try the GUI

The GUI is for editing the INI config files in a schema-aware way (it
shows you what fields are required, what types they are, what values
are valid).

```bash
python3 gui/ini_editor.py xml_emitter/xml_emit_config.ini
```

You'll see a three-pane editor:

- **Left:** sections in the file (clickable)
- **Middle:** fields in the selected section (typed widgets)
- **Right:** schema info (help text, type, required-ness)

Edit a field, hit Save, and your changes are written back with all
comments and blank lines preserved exactly. The GUI never loses
formatting.

If you don't pass a filename, the GUI shows a file picker.

## Run on your own SonicWall config

### Step 1: Drop your config in

```bash
cp /path/to/your-firewall.txt sonicwall_parser/input/
```

The parser auto-detects any `.txt` file in the input directory.

### Step 2: Re-run the pipeline

```bash
python3 sonicwall_parser/sonicwall_parser.py \
    --config sonicwall_parser/parser_config.ini \
    --input sonicwall_parser/input/your-firewall.txt
python3 enricher/enricher.py --config enricher/enrich_config.ini
python3 xml_emitter/xml_emitter.py --config xml_emitter/xml_emit_config.ini
```

### Step 3: Read the migration notes

`xml_emitter/dry_run/migration_notes.md` is your operator briefing. It
lists:

- Every decision the tool made
- Every concern it found
- Every place it needs you to do something manually

Read it cover to cover before importing the XML.

### Step 4: Check the validator

If the validator reports errors or warnings against your output XML,
the most common causes are:

- A SonicWall construct we haven't written a translation rule for yet
- An edge case the schema is too strict about
- A genuine problem with the source SonicWall config

Each finding in the validator report includes the XML path of the
affected element, so you can see exactly where to look.

### Step 5: Import to your Firebox

When you're satisfied with `configuration.xml` and have followed the
operator action items in `migration_notes.md`:

1. Log into the Firebox web UI
2. **System → Backup and Restore Image → Restore Configuration**
3. Upload `configuration.xml`
4. Verify connectivity before committing

**Always test on a lab device first.** This toolkit is in active
development; treat its output as a strong starting point that needs
operator review, not a drop-in replacement.

## How to tune the translation rules

The toolkit's design philosophy: **every translation rule lives in an
INI file**. To change behavior, edit an INI; never edit code.

### Common tuning targets

**Add a new value mapping** (e.g. SonicWall service name → WatchGuard
service name):

Edit `concept_map/concept_map.ini` and add or extend a `value_map.*`
section.

**Filter out SonicWall-only constructs that are noise**:

Edit `xml_emitter/xml_emit_config.ini`. Find the rule for that
construct (e.g. `[emit.aliases]`), and add a regex to
`skip_name_patterns`.

**Add a new advisory** (the toolkit detects something and writes an
operator note instead of generating XML):

Add a new `[emit.<name>_advisory]` section to
`xml_emit_config.ini` with `status = advisory_only` and an
`advisory_template`. See `[emit.portshield_advisory]` for an
example.

**Change which SonicWall zone maps to which WatchGuard zone**:

Edit `concept_map/concept_map.ini`, find
`[value_map.sw_zone_to_wg_iface_name]`, and adjust the
`canonical_to_watchguard` mapping.

### The schema feedback loop

Each config file has a sibling `<config>.schema.ini` that describes
what fields are valid in that config. The schema validator runs
automatically when you load a config in the GUI, and the engines
validate their own configs at startup.

If you add a new field to a config, you may also need to add it to the
schema. The schema files are themselves just INI, so the same rules
apply.

## Common questions

**Q: Can I run this without the GUI?**
A: Absolutely. The GUI is a convenience. All engines run from the
command line.

**Q: Will it support [my SonicWall feature]?**
A: Probably, eventually. The toolkit's coverage is improving with each
release. Anything not yet supported either lands in `_unparsed.json`
(parser missed it) or generates a TODO advisory in `migration_notes.md`
(emit rule missing). Both surfaces give you visibility into what's not
covered yet.

**Q: Can I undo a translation?**
A: The output is one XML file. To undo, just don't import it. The
SonicWall config text file is never modified.

**Q: How do I see what's happening internally?**
A: Run any engine with `--verbose`:
```bash
python3 xml_emitter/xml_emitter.py --config xml_emitter/xml_emit_config.ini --verbose
```

**Q: What's the validator's job exactly?**
A: It checks that the output XML conforms to the WatchGuard schema —
that every element has the right shape, required fields, valid
references between elements, and no dangling pointers. It does NOT
check that the XML matches your SonicWall source semantically (that's
the diff-scorer's job).

**Q: What's the diff-scorer's job?**
A: It compares your emit output XML against a reference WatchGuard XML
export across 5 dimensions (structural, element_counts,
field_completeness, cross_reference, value_distribution) and produces
an aggregate similarity score. Useful for regression-testing changes
to translation rules.

**Q: Does this work for SonicOS Enhanced and Standard?**
A: The parser handles SonicOS Enhanced (CLI export from `show
running-config`). Standard (web export) is not yet supported.

**Q: My SonicWall has [VPN | NAT | wireless | application control
configs]. Will those translate?**
A: Coverage varies. VPN policies translate (with crypto deprecation
warnings for old algorithms). NAT translates partially. Wireless and
app-control are not yet emitted to XML — they appear as TODO
advisories.

## Where to learn more

- **Architecture deep-dive:** `README.md`
- **Schema curation methodology:**
  `wg_validator/SCHEMA_CURATION_NOTES.md`
- **Cross-reference rules explained:**
  `wg_validator/CROSS_REFERENCE_FINDINGS.md`
- **Portshield migration playbook:**
  `docs/portshield_to_bridge.md`
- **Multi-line INI handling:**
  `schema_lib/MULTILINE_KNOWN_LIMITATION.md`
- **Phase 2 work-in-progress:** `docs/PHASE2_TODO.md`
- **GUI features:** `gui/README.md`

## Getting unstuck

- **Test suite fails:** Make sure you're running from the toolkit root
  (the parent of `harness/`). All tests use absolute paths from there.
- **GUI won't open:** Check that Tkinter is installed
  (`python3 -c "import tkinter"`).
- **Parser produces empty output:** Check
  `sonicwall_parser/output/_unparsed.json` — if it's huge, the parser
  pattern matrix doesn't recognize your config dialect. Open an issue
  or extend `parser_config.ini` with a new pattern.
- **Emit produces 0 entries for a section:** Check
  `xml_emitter/dry_run/_emit_manifest.json` for the rule's status. If
  it says `result: skipped`, the input filter didn't match anything.

## You're ready

Run the quickstart, browse the output, then point it at your own
config and tune from there. The migration notes are your friend —
they'll tell you exactly what the tool did and what it left for you.

Welcome aboard.
