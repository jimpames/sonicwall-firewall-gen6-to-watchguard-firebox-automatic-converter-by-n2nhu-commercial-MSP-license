# Migration Toolkit — Operations Manual

> **SonicWall → WatchGuard Migration Toolkit**
> Built by n2nhu lab, Newburgh NY.
> Captain: Jim Ames (`jimpames`).
> This manual: written 2026-05-13, after Phase 7 hardening landed.

---

## Table of Contents

1. [What this manual is, and what it isn't](#1-what-this-manual-is-and-what-it-isnt)
2. [File manifest and folder layout](#2-file-manifest-and-folder-layout)
3. [The problem this toolkit solves](#3-the-problem-this-toolkit-solves)
4. [Theory of design](#4-theory-of-design)
5. [Configuration files reference](#5-configuration-files-reference)
6. [Operations — running an actual migration](#6-operations--running-an-actual-migration)
7. [Operations — triage walk-through](#7-operations--triage-walk-through)
8. [Operations — corpus management](#8-operations--corpus-management)
9. [Operations — team coordination](#9-operations--team-coordination)
10. [Outputs reference](#10-outputs-reference)
11. [Troubleshooting](#11-troubleshooting)
12. [Glossary](#12-glossary)

---

## 1. What this manual is, and what it isn't

This manual is for **the next senior engineer onboarded to the n2nhu lab migration practice.** It assumes:

- 5+ years working with SonicWall and WatchGuard firewalls
- Comfort with the command line, Python virtualenvs, and INI/JSON files
- General awareness of why MSP migrations are hard

It is **not**:

- An end-customer document
- A networking tutorial
- A description of every Python function (read the source for that)
- A complete history of why each feature exists (see `schema_learner/DESIGN*.md` files for that)

If you're reading this and you are NOT a senior engineer, two pieces of friendly guidance:

1. **The toolkit honors operator authority at every gate.** It is designed to be wielded by an expert, not to be a black box that converts firewalls. If you don't know what a finding means, the toolkit can't tell you — but a senior engineer can.

2. **Do not use this on live customer infrastructure until you've walked a full migration on a lab fixture with senior supervision.** The output of this toolkit eventually becomes the running config on a Firebox. Mistakes are visible and recoverable in the lab; in production they cause customer outages.

---

## 2. File manifest and folder layout

The toolkit lives in one directory. On the dev machine it's at:

```
/mnt/user-data/outputs/migration_toolkit/
```

On the Windows deployment it's at:

```
C:\Users\jimpames\Desktop\fes-3-commercial\migration_toolkit\
```

The two should remain byte-for-byte identical except for `__pycache__/` directories and per-machine state (audit logs, lock files).

### 2.1 Top-level layout

```
migration_toolkit/
├── OPERATIONS_MANUAL.md           This file
├── README.md                       Quick orientation (older, kept for reference)
├── GETTING_STARTED.md              First-run walkthrough (older)
├── HANDOFF_TO_DEVOPS.md            Deployment notes for engineering ops
├── run_pipeline.py                 One-shot driver for the full pipeline
│
├── sonicwall_parser/               INPUT side — parses SonicWall CLI exports
├── concept_map/                    MIDDLE — algebraic translation rules
├── xml_emitter/                    MIDDLE — emits WatchGuard XML
├── wg_validator/                   OUTPUT side — validates emitted XML
│
├── schema_learner/                 LEARNING side — schema-learner subsystem
│   ├── corpus/                       approved corpus (XML configs)
│   ├── corpus_tiers.py               banned/sideline registries
│   ├── triage.py                     operator triage CLI
│   ├── schema_learner.py             the engine
│   └── learn_config.ini              schema-learner config
│
├── harness/                        REGRESSION test suite (15 suites, 130+ tests)
├── docs/                           Reference materials (vendor docs, schemas)
├── gui/                            Web-based editor for INI files
└── team_config/                    Multi-engineer roster, locks, audit logs
```

### 2.2 The parser side — `sonicwall_parser/`

The parser ingests SonicWall CLI exports and emits structured JSON. **Phase 7 relevance filter** lives here.

```
sonicwall_parser/
├── sonicwall_parser.py             The data-driven parser engine (~620 lines)
├── parser_config.ini               65 section definitions
├── parser_config.schema.ini        Validates parser_config.ini
├── parser_schema.ini               Schema for parser output
├── builtin_objects.ini             Phase 7 — operator-curated vendor built-ins
├── relevance_filter.py             Phase 7 — classifier engine (~1150 lines)
│
├── input/                          PUT your SonicWall .txt exports HERE
│   ├── nsa3600.txt                   sample customer-seeded config (615KB)
│   └── slimmed-sonicwallconfig.txt   smaller sample (104KB)
│
├── output/                         Parser writes per-section JSONs here
│   ├── access_rules.json
│   ├── address_objects.json
│   ├── service_objects.json
│   ├── ... (47 files, one per section type)
│   └── _unparsed.json               Lines that didn't match any section
│
└── filtered/                       Phase 7 writes classified output here
    ├── customer_defined.json         Translator workload
    ├── vendor_builtin.json           Pass-through (1:1 WG mapping)
    ├── dead_unreferenced.json        Skip with audit log
    ├── disabled_in_source.json       Phase 6.5 defer/auto-promote
    └── migration_scope.md            Operator-facing migration scope summary
```

### 2.3 The concept-map and emitter — `concept_map/` and `xml_emitter/`

The algebraic translation layer. The concept map declares abstract concepts (firewall_rule, vpn_site_to_site, concrete_address) independent of vendor. The emitter turns those concepts into WatchGuard XML.

```
concept_map/
├── concept_map.ini                 51 abstract concept definitions
├── concept_map.schema.ini          Validates the concept map
└── concept_map_schema.ini          (legacy filename, ignore)

xml_emitter/
├── xml_emit_config.ini             25 emit-rule definitions
├── xml_emit_config.schema.ini      Validates emit rules
├── xml_emit_schema.ini             (legacy filename, ignore)
├── skeleton_config.ini             WatchGuard XML skeleton settings
├── make_skeleton.py                Generates the skeleton scaffolding
├── output/                         Final emitted WatchGuard .xml goes here
└── dry_run/                        Dry-run outputs for testing
```

### 2.4 The validator — `wg_validator/`

The validator checks that emitted XML matches WatchGuard's schema, with cross-reference validation.

```
wg_validator/
├── wg_validator.py                 The validator CLI
├── wg_validator_config.ini         Schema configuration
├── wg_validator_config.schema.ini  Validates the validator config
├── schema_inferer.py               Infers schema from reference WatchGuard XMLs
├── output/                         Validation reports
├── CROSS_REFERENCE_FINDINGS.md     Known cross-reference rules
└── SCHEMA_CURATION_NOTES.md        Schema curation history
```

### 2.5 The schema-learner — `schema_learner/`

The heart of the principle-binding system. Learns what WatchGuard XML surfaces exist, manages the triage workflow, enforces "no claim without provenance."

```
schema_learner/
├── DESIGN.md                       Original architecture document
├── DESIGN_team_scale_addendum.md   Team-scale design decisions (LOCKED)
│
├── schema_learner.py               The learning engine (~1200 lines)
├── feature_vectors.py              Structural fingerprinting
├── merge_engine.py                 Approved-corpus merge logic
├── corpus_tiers.py                 Approved/banned/sideline machinery
├── triage.py                       Operator triage CLI (~1700 lines)
├── llm_provider.py                 LLM drafter (Anthropic + GPT4All + Stub)
├── deep_research.py                Phase 4 lab-verification workflow
├── team_coordination.py            Phase 5 multi-engineer infrastructure
│
├── learn_config.ini                Main schema-learner config
├── learn_config.schema.ini         Validates learn_config.ini
│
├── corpus/                         Approved XML configs (the learning corpus)
│   ├── T30jpa1.xml                 real customer Firebox export
│   ├── TEMPLATE_FW.xml             vendor template Firebox export
│   └── (more .xml files as the corpus grows)
│
├── proposals/                      Per-run findings + intervention markdowns
│   └── 2026-05-10_165907_TEMPLATE_FW/
│       ├── findings.json             machine-readable engine output
│       ├── findings.md               operator-readable engine output
│       ├── OPERATOR_INTERVENTION_REQUIRED.md  the file operator edits
│       ├── triage_state.json         per-run triage progress + locks
│       └── drafts/                   cached LLM drafts
│
├── audit/                          Append-only audit log (monthly files)
│   └── 2026-05.log
│
├── banned/                         Banned-corpus entries (Phase 2)
└── backups/                        Pre-merge backups of all touched INI files
```

### 2.6 The harness — `harness/`

Regression test suite. The toolkit currently has **16 test suites with 145+ individual tests**, all of which must pass before any change is committed.

```
harness/
├── run_all.py                      Run all 15 suites
├── test_comment_preserving_ini.py
├── test_comment_preserving_multiline.py
├── test_gui_save_roundtrip.py
├── test_engine_validation_wiring.py
├── test_wg_validator.py
├── test_schema_learner_provenance.py
├── test_merge_engine.py
├── test_auto_reuse.py
├── test_triage.py
├── test_phase3_llm.py
├── test_phase4_deep_research.py
├── test_phase5_team_coordination.py
├── test_phase6_tier_filtering.py
├── test_phase6_5_auto_promotion.py
└── test_phase7_relevance_filter.py
└── test_phase8_skeleton_fixes.py
```

### 2.7 Multi-engineer infrastructure — `team_config/`

Per-machine state for multi-engineer coordination (Phase 5).

```
team_config/
├── operators.ini                   Senior engineer roster
├── locks/                          Active triage locks (file-based)
└── disagreements/                  Open disagreement flags
```

---

## 3. The problem this toolkit solves

### 3.1 The business problem

An MSP that supports both SonicWall and WatchGuard environments has three operational realities:

1. **Migrations are recurring revenue.** Customers move firewall vendors for cost, compliance, vendor support, or refresh-cycle reasons.
2. **A senior engineer's time costs more than the migration tooling does.** Every hour spent re-doing manual translation work that the toolkit could have done is margin lost.
3. **Mistakes are expensive in customer trust.** A botched migration takes a customer's network down. They remember.

The traditional approach to these migrations is: a senior engineer with dual-vendor expertise reads the SonicWall config line-by-line, mentally translates each piece into the WatchGuard equivalent, manually authors the WatchGuard XML or uses the GUI, lab-tests, and deploys. **This works but doesn't scale.**

### 3.2 The technical problem

Why doesn't this scale? Three reasons:

**Reason 1 — Knowledge is non-transferable in your head.** When a senior engineer types in a Firebox policy that matches a SonicWall rule, that translation knowledge dies with the keystroke. The next migration, the engineer has to re-derive the same decision. The next engineer joining the practice has to learn it all over from scratch.

**Reason 2 — Vendor coverage is uneven.** SonicWall ships ~65 configurable section types; WatchGuard ships ~76. Most surfaces map cleanly (HTTP service = HTTP service). Some don't (SonicWall's "App Rules" vs WatchGuard's "Application Control"). The senior engineer ends up doing 80% mechanical work (the cleanly-mapping parts) so they can focus on the 20% that actually needs judgment.

**Reason 3 — Verification is manual and error-prone.** After authoring the WatchGuard config, the engineer reviews it side-by-side with the SonicWall original to confirm completeness. This is tedious; humans skip steps when tired. Subtle migration gaps (a NAT policy lost, a VPN ACL incomplete) become production incidents.

### 3.3 What the toolkit does

The toolkit is built on one founding principle:

> **No claim without provenance.**

Every translation decision the toolkit makes is either:
- (a) Operator-vouched, with the operator's name, timestamp, and citation captured in the data, OR
- (b) Mechanically derived from operator-vouched data via rules the operator has approved

The toolkit does NOT auto-translate ambiguous surfaces and hope. It identifies them, presents them to the operator with structured context, and waits for an operator-vouched declaration with a citation. **The operator's time goes to genuine decisions, not mechanical re-derivation.**

This means:

- Vendor built-ins (HTTP, LAN, default zone-to-zone rules) auto-translate without operator attention
- Customer-defined surfaces (named access-rules, custom VPN tunnels, NAT policies) go through structured triage with citation-verified LLM drafting and lab-verification gates
- Once a senior engineer has triaged a surface (e.g., "this SonicWall thing is X on WatchGuard, here's the citation"), the toolkit remembers — so future migrations involving the same surface are auto-resolved from the corpus
- Surfaces that were disabled in source are deferred from triage attention but auto-promote when a future customer config has them enabled

The toolkit also handles **multi-engineer coordination**: when the practice grows from one senior engineer to several, the corpus and audit infrastructure ensures no two engineers triage the same surface twice, no operator's decision can be silently overridden, and every migration decision is traceable to a named operator with timestamp.

---

## 4. Theory of design

The toolkit is organized around five architectural principles. Understanding these will make the operations chapter make sense.

### 4.1 Principle 1 — No claim without provenance

Every artifact the toolkit produces carries provenance markers:

- `OBSERVATION` markers wrap engine-produced structural facts
- `OPERATOR-AUTHORED-BEGIN/END` markers wrap operator-declared semantics
- `LLM-DRAFT` markers wrap LLM-produced draft text (subordinate to operator review)

The toolkit's provenance audit (test suite `test_schema_learner_provenance.py`) refuses to accept any change that doesn't carry these markers. If you can't tell who said something, you can't trust it.

### 4.2 Principle 2 — Structural observations vs semantic claims

The schema-learner engine produces **structural observations** only: "this XML element has 47 instances, with these child tags and these value patterns, similar to these N existing elements with these similarity scores."

The engine does NOT say "this element means firewall policy" or "this maps to SonicWall access-rule X." Those are **semantic claims** that require operator authority and citation.

This split is what makes the toolkit safe to wield: the engine's structural output is mechanically verifiable, and the operator's semantic claims are always trace-back-able to a named human with a citation.

### 4.3 Principle 3 — Three-tier corpus

Surfaces in the system live in one of three tiers:

- **Approved corpus** — surface has been operator-classified and is part of the toolkit's knowledge base. Future occurrences auto-reuse this classification (Phase 1).
- **Banned corpus** — surface was classified as harmful or actively wrong. Future similar surfaces get auto-suppressed before reaching operator attention (Phase 2). This is the team's negative training signal.
- **Sideline queue** — surface needs deeper research (Phase 4) or was deferred because it was disabled in source (Phase 6). These auto-resurface when conditions change.

The tier system means operator decisions **compound across the team** rather than being repeated for each migration.

### 4.4 Principle 4 — Tiered triage by relevance

Phase 6 (output side) and Phase 7 (input side) implement the same insight: **operator attention is the scarce resource, and not every surface deserves it.**

On the WatchGuard output side (Phase 6): if a surface is disabled in source XML, defer it. It auto-promotes (Phase 6.5) when a future customer config has it enabled.

On the SonicWall input side (Phase 7): if an object is a vendor built-in, dead/unreferenced, or disabled, classify it accordingly. The translator only processes the `customer_defined.json` bucket.

Real numbers from a typical customer config: **~95% of parsed entries are vendor built-ins, dead, or disabled.** The operator triages the 5% that actually matters.

### 4.5 Principle 5 — Algebraic translation

The middle layer (`concept_map/` + `xml_emitter/`) uses an **algebraic abstraction** between SonicWall vocabulary and WatchGuard vocabulary. Rather than a direct SonicWall-to-WatchGuard table, the concept map declares abstract concepts (`concrete_address`, `firewall_rule`, `vpn_site_to_site`) and rules for SonicWall → concept and concept → WatchGuard.

This is `Object × Verb = Action` (the algebraic-design document). The benefit: when a third vendor (e.g., Fortigate) eventually enters the picture, you build one mapping (Fortigate → concept) rather than one mapping per source-target pair.

---

## 5. Configuration files reference

### 5.1 `schema_learner/learn_config.ini`

This drives the schema-learner engine. Sections and key parameters:

| Section | Key | Purpose | Default |
|---|---|---|---|
| `[io]` | `corpus_dir` | Approved corpus directory | `corpus` |
| `[io]` | `existing_schema_path` | Existing WG schema INI | `../wg_validator/wg_xml_schema.ini` |
| `[io]` | `proposals_dir` | Where findings get written | `proposals` |
| `[io]` | `findings_format` | `markdown`, `json`, or `both` | `both` |
| `[thresholds]` | `auto_propose_minimum` | Confidence below which engine produces no proposal | `0.40` |
| `[thresholds]` | `high_confidence` | Confidence above which finding is "high confidence" | `0.85` |
| `[thresholds]` | `medium_confidence` | Confidence above which "medium" | `0.50` |
| `[reporting]` | `top_k_neighbors` | How many structural neighbors per finding | `10` |
| `[auto_reuse]` | `similarity_threshold` | Approved-corpus auto-reuse threshold | `0.85` |
| `[banned_suppression]` | `similarity_threshold` | Banned-corpus suppression threshold | `0.80` |
| `[mcp]` | `enabled` | Enable LLM drafting | `false` |
| `[mcp]` | `provider` | `anthropic`, `gpt4all`, or `stub` | `anthropic` |
| `[mcp]` | `model` | Model name | `claude-haiku-4-5-20251001` |
| `[mcp]` | `gpt4all_host` | GPT4All server host | `localhost` |
| `[mcp]` | `gpt4all_port` | GPT4All server port | `4891` |
| `[mcp]` | `gpt4all_model` | GPT4All model name | `Llama 3 8B Instruct` |
| `[mcp]` | `timeout_seconds` | LLM call timeout | `60` |
| `[mcp]` | `verify_citations` | Fetch + verify citation URLs | `true` |
| `[behavior]` | `require_operator_approval` | (Cannot be `false`) | `true` |
| `[behavior]` | `backup_before_merge` | Backup INIs before merge | `true` |

**Important: do NOT set `require_operator_approval = false`.** The schema validator will reject the change. The toolkit's principle binding depends on this gate being unconditional.

### 5.2 `sonicwall_parser/builtin_objects.ini`

Operator-curated registry of SonicWall vendor built-ins. Has the following sections:

- `[meta]` — version + provenance
- `[services]` — built-in service-object names (224 entries from factory NSA3600)
- `[service_groups]` — built-in service-group names (50 entries)
- `[zones]` — built-in zone names (LAN, WAN, DMZ, ...)
- `[address_objects]` — built-in address-object names (rare; mostly empty)
- `[access_rule_patterns]` — comment patterns identifying auto-generated rules
- `[schedules]` — built-in schedule names

**To extend:** add the exact name as a key with value `true`. Comments after `;` are ignored.

Example: add a new SonicOS-shipped service:

```ini
[services]
HTTP = true
MyNewBuiltIn = true               ; from SonicOS 7.0 default config
```

### 5.3 `sonicwall_parser/parser_config.ini`

Drives the SonicWall CLI parser. Each `[section.foo]` block declares one parseable section type. **65 sections currently declared.**

You generally don't edit this unless adding parser coverage for new SonicWall constructs. Schema validation (`parser_config.schema.ini`) keeps additions safe.

### 5.4 `concept_map/concept_map.ini`

Algebraic concept declarations. **51 concepts currently declared.** Each section defines one abstract concept independent of vendor.

You edit this when adding new translatable surfaces.

### 5.5 `xml_emitter/xml_emit_config.ini`

WatchGuard XML emit rules. **25 emit rules currently declared.** Each section says "concept X emits as XML structure Y."

You edit this when adding new WatchGuard XML coverage.

### 5.6 `team_config/operators.ini`

Roster of senior engineers authorized to perform triage. Format:

```ini
[operators]
jimames = senior
some_other_engineer = senior

[meta]
roster_version = 1
bootstrap_mode = false
```

When `bootstrap_mode = true`, any operator name is accepted (for first-engineer setup). Set to `false` once the roster has at least one named operator.

---

## 6. Operations — running an actual migration

This chapter walks through a complete SonicWall → WatchGuard migration for an example customer. We'll use `nsa3600.txt` as the source.

### 6.1 Prerequisites

- Python 3.10+ (the code uses dataclass features and `match` syntax in places)
- No external Python dependencies — the toolkit uses stdlib only by design
- Either: an `ANTHROPIC_API_KEY` environment variable (for cloud LLM drafting), OR a running GPT4All server with the model loaded (for local LLM drafting), OR neither (for `--no-llm` mode where the operator does all research)

### 6.2 Confirm health before starting

Always confirm the toolkit is green before starting work on a real customer config:

```bash
cd /path/to/migration_toolkit
python3 harness/run_all.py
```

Expected output ends with:

```
======================================================================
  15 passed, 0 failed, NN.Ns total wall time
======================================================================
```

If anything fails, **do not proceed.** Investigate the failure or roll back.

### 6.3 The five-step migration workflow

```
1. Place SonicWall CLI export in sonicwall_parser/input/
2. Run the parser
3. Run the relevance filter
4. Run the schema-learner for novelty detection (only if customer
   surfaces are present)
5. Run the translator (concept_map + xml_emitter)
```

Each step is one command. The full sequence:

```bash
# Step 1 — place the SonicWall CLI .txt export
cp /path/to/customer-nsa3600-export.txt sonicwall_parser/input/

# Step 2 — parse
python3 sonicwall_parser/sonicwall_parser.py \
    --config sonicwall_parser/parser_config.ini \
    --input sonicwall_parser/input/customer-nsa3600-export.txt

# Step 3 — Phase 7 relevance filter
python3 sonicwall_parser/relevance_filter.py \
    --source-label customer-nsa3600

# Step 4 — schema-learner (if customer_defined.json has entries)
# This will be expanded as the toolkit grows. For now this step
# is most relevant when adding to the WatchGuard corpus.
python3 schema_learner/schema_learner.py \
    --input some_reference.xml \
    --source-label customer-nsa3600

# Step 5 — translator (currently in development; uses run_pipeline.py)
python3 run_pipeline.py --input sonicwall_parser/input/customer-nsa3600-export.txt
```

### 6.4 Reading the parser output

After step 2, the parser writes:

- `sonicwall_parser/output/*.json` — one file per section type (47 files)
- `sonicwall_parser/output/_unparsed.json` — anything the parser didn't recognize
- `sonicwall_parser/output/_manifest.json` — what was parsed, source file metadata

**Check `_unparsed.json` first.** If it's non-empty, the parser missed something. Either:
- The customer config has a SonicOS construct the parser doesn't yet cover (add a `[section.foo]` block to `parser_config.ini`)
- The customer config has a malformed line (rare)

**Check `_meta.source_file` in each JSON.** This must match the file you actually parsed. If it shows a different filename, the output is stale — re-run the parser.

### 6.5 Reading the relevance filter output

After step 3, the filter writes:

- `sonicwall_parser/filtered/customer_defined.json` — the translator workload (this is the file that drives the rest)
- `sonicwall_parser/filtered/vendor_builtin.json` — passthrough hints (translator emits these 1:1 to WG)
- `sonicwall_parser/filtered/dead_unreferenced.json` — dead entries, skipped
- `sonicwall_parser/filtered/disabled_in_source.json` — disabled entries, deferred
- `sonicwall_parser/filtered/migration_scope.md` — the operator-facing summary (read this!)

The filter's stdout output includes a `NOTICE` listing any parser sections that have no classifier yet — those are passed through as unclassified. Track this list; as the toolkit grows, add classifiers for sections that matter for migration coverage.

### 6.6 What "ready to translate" looks like

After parser + filter, you have:

```
sonicwall_parser/output/      ← full faithful parsed representation
sonicwall_parser/filtered/    ← classified into 4 buckets + scope report
```

The `customer_defined.json` file tells the translator exactly what work to do. The `migration_scope.md` tells the operator and the customer what's being migrated and what's being skipped.

### 6.7 Regenerating the skeleton with target-Firebox values (Phase 8)

The skeleton at `docs/skeletons/master_skeleton.xml` is generated from a reference Firebox XML (`docs/reference_exports/T30jpa1__1_.xml` for the lab T-30). The reference's version stanza values (`<for-version>`, `<rs-version>`, `<product-grade>`, etc.) are NOT appropriate for arbitrary target Fireboxes — they're specific to the unit the reference came from.

Phase 8 added explicit operator-replace machinery for these values. Two modes:

**Default mode** — the skeleton emits `__OPERATOR_REPLACE__<tagname>__` sentinels for each device invariant. The operator searches the emitted XML for `__OPERATOR_REPLACE__` and replaces each sentinel with the correct value for the target Firebox. `wg_validator` will refuse to bless any XML still carrying sentinels.

**CLI-override mode** — pass explicit values when regenerating the skeleton:

```bash
cd xml_emitter
python3 make_skeleton.py --config skeleton_config.ini \
    --for-version 12.10.2 \
    --rs-version 1068452504 \
    --product-grade 2 \
    --xml-purpose 2 \
    --using-cpm-profile 0
```

To get the right values, take a Get-Configuration XML export of the target Firebox via Policy Manager or Web UI. The first 6-8 lines have the version stanza you need.

The `--rs-version` integer matters most — it's firmware-build-specific and varies between releases. The other fields are typically:

- `--product-grade 2` (T-series, M-series)
- `--xml-purpose 2` (Firebox configuration export)
- `--using-cpm-profile 0` (unless the Firebox is centrally managed via Cloud)
- `--for-version <firmware>` (e.g., 12.10.2)

After regenerating the skeleton, re-run the emit pipeline to produce an importable config:

```bash
cd xml_emitter
python3 xml_emitter.py
```

The output (`xml_emitter/dry_run/configuration.xml`) should now validate cleanly (no `operator-sentinel` errors).

### 6.8 Phase 9 — schema-aware validation (the bidirectional comparator)

Phase 9 emerged from a single lab smoke-test session (2-3 Jun 26) when seven distinct classes of WatchGuard XSD violations were caught by the Firebox over multiple import attempts. Each round produced one new validator rule, encoded in a corpus of grounded WG schema invariants.

**The $1/$100 principle.** Captain's exact words mid-session: "Every $1 we spend on diff is maybe $100 we don't spend on debug." The validator (`wg_validator`) is the $1 oracle — runs locally, milliseconds per check. The Firebox importer is the $100 oracle — requires lab time, file push, 30-60s reload, and produces terse error messages that take effort to decode. Phase 9 invests in making the $1 oracle catch what the $100 oracle would otherwise catch. See `docs/PRINCIPLE_diff_over_debug.md` for the full rationale.

**The seven rules added:**

| Class | Validator rule | Emit-side fix |
|---|---|---|
| Missing `<type>` discriminator | `POLYMORPHIC_RULES` | `@first` directive in emit config |
| Wrong `<type>` position | (same rule, ordering check) | (same) |
| `<member>` missing required payload | `_check_member_payloads` | `<server-port>0</server-port>` fallback |
| Non-numeric port (`echo-reply`) | tightened `<server-port>` to int | `input_filter` on protocol=ICMP |
| Non-numeric protocol (`IPCOMP`) | `PATH_AWARE_VALUE_TYPES` | extended `protocol_id` value_map |
| Duplicate names in unique list | `_check_name_uniqueness` | `_dedupe_unique_lists` post-emit pass |
| Child element out of canonical order | `_check_child_ordering_vs_reference` | `_reorder_children_per_reference` |

**The bidirectional comparator.** The last rule deserves attention. Earlier validator phases did back-diff only — "does our output have anything jpa.xml doesn't have?" The Firebox enforces forward-diff too — "does our output match the canonical child ordering jpa.xml shows?" Phase 9's `_check_child_ordering_vs_reference` reads jpa.xml at validate time, builds a per-tag canonical child sequence by walking every instance in the reference, and reports children that appear out of canonical order. The companion `_reorder_children_per_reference` in the emitter uses the same canonical orderings to fix the data automatically — no per-element emit-rule rewrites needed.

This shape is the same principle as the schema-learner subsystem: the reference data IS the configuration. As jpa.xml grows (vendor versions, customer-specific shapes), the validator and reorderer update for free.

**Cache freshness check.** Phase 9 also added `_check_cache_freshness` to detect a recurring pollution mode where the regression suite re-parses a different SonicWall config mid-session and pollutes the prod caches. The emitter now warns if enricher output is older than parser output, with a clear remediation command.

For the full list of validator rules and the test cases each one is anchored to, see `wg_validator/wg_validator.py` (search for `POLYMORPHIC_RULES`, `UNIQUENESS_CONSTRAINTS`, `ORDERING_CHECKED_TAGS`).

---

## 7. Operations — triage walk-through

When the schema-learner emits novel findings (new XML element types, new field shapes, new values), they go into a proposal directory under `schema_learner/proposals/`. The operator's job is to walk through these findings and classify each one.

### 7.1 Running a triage walk

```bash
# Walk through Tier 1 (critical-path) findings only
python3 schema_learner/triage.py walk \
    --proposal schema_learner/proposals/2026-05-10_165907_TEMPLATE_FW \
    --operator jimames \
    --tier 1 \
    --auto-sideline-deferred
```

Flags worth knowing:

- `--proposal` — required. Path to the proposal directory.
- `--operator` — required. Your operator name from `team_config/operators.ini`.
- `--tier 1` — Phase 6 filter. Triages only findings classified as Tier 1 (full triage needed). Other tiers can be `1,2` or `all`.
- `--auto-sideline-deferred` — Phase 6. Findings excluded by tier filter get routed to the sideline queue with reason `deferred_disabled_in_source` (Phase 6.5 will auto-promote them on a future config with the surface enabled).
- `--no-llm` — disable LLM drafting for this run (Phase 2 behavior, operator does all research).
- `--config` — alternate path to `learn_config.ini`.

### 7.2 What you see at each finding

```
======================================================================
  Surface: <Botnet>
  Source:  TEMPLATE_FW
  Category: novel_element_type
  Status:  enabled — 🔴 Tier 1 (full triage)
  Evidence: leaf value='1'
======================================================================
STRUCTURAL OBSERVATIONS (engine):
  Instance count:        1
  Distinct child tags:   0
  Distinct field shapes: 1
  Naming tokens:         ['botnet']

STRUCTURAL NEIGHBORS (research context, NOT kinship claim):
  IPS                                      similarity = 0.277
  GAV                                      similarity = 0.277
  ...

[Calling LLM drafter for <Botnet>...]
[Draft cached: Botnet.json]

LLM DRAFT (research assistant — operator must verify):
  Model: gpt4all:Llama 3 8B Instruct
  Overall confidence: low
  Q1 Purpose: This element configures botnet detection.
    Citation: https://www.watchguard.com/...
    Status:   ✗ LOW (citation not verified — operator must research)
    Reason:   URL did not fetch: HTTP 404
  ...

Action — (c)orrect, (e)dit, (h)armful, (s)ideline, (q)uit, (?)help:
```

The action keys:

- `c` — Correct. Accept the LLM draft as-is. Use only if the draft is complete and you've verified its citations.
- `e` — Edit. Opens `OPERATOR_INTERVENTION_REQUIRED.md` in your editor; you fill in Q1-Q4 with your own claims and citations.
- `h` — Harmful. The draft (or the surface itself) is actively wrong. Goes into the banned corpus to suppress similar drafts in the future.
- `s` — Sideline. Defer to deep research (Phase 4). The surface goes into the sideline queue.
- `q` — Quit. Saves your progress and exits. Resume later with the same command.
- `?` — Help.

### 7.3 The four operator declarations (Q1-Q4)

Every surface requires the operator to fill in four declarations:

- **Q1 — Purpose.** What does this surface do? One sentence + citation URL.
- **Q2 — Category.** Pick one from: `network-plumbing`, `policy-access-control`, `object-library`, `tunnel-vpn`, `html-web-app`, `threat-detection`, `authentication-infrastructure`, `device-management`, `administrative`.
- **Q3 — SonicWall equivalent.** What's the matching SonicWall feature? Or "none — net-new on WatchGuard." Include citation.
- **Q4 — Operational notes.** Security or operational considerations to flag. Or "none."

Each Q requires either a citation URL (which the toolkit fetches and verifies) or an explicit "no citation found" marker. **No claim without provenance.**

### 7.4 LLM drafting modes

The toolkit supports three drafting providers:

**Anthropic (cloud):**
```ini
[mcp]
enabled = true
provider = anthropic
api_key_env_var = ANTHROPIC_API_KEY
model = claude-haiku-4-5-20251001
```
Requires `ANTHROPIC_API_KEY` set in environment. Production-quality drafts. Costs per-call.

**GPT4All (local):**
```ini
[mcp]
enabled = true
provider = gpt4all
gpt4all_host = localhost
gpt4all_port = 4891
gpt4all_model = Llama 3 8B Instruct
```
Requires GPT4All running with the named model loaded. No API costs. Lower quality drafts but principle binding catches the failure modes (URL hallucination, etc.).

**Stub (testing only):**
```ini
[mcp]
enabled = true
provider = stub
```
Deterministic fixture responses. Never use for real triage.

**No LLM:**
```ini
[mcp]
enabled = false
```
Or pass `--no-llm` to triage walk. Operator does all research themselves.

### 7.5 Resuming an interrupted triage walk

The toolkit caches progress in `schema_learner/proposals/<proposal>/triage_state.json`. Run the same command again and the walk picks up where you left off:

```bash
python3 schema_learner/triage.py walk \
    --proposal schema_learner/proposals/2026-05-10_165907_TEMPLATE_FW \
    --operator jimames
```

You'll see `Already processed: N` showing how many surfaces you've completed.

---

## 8. Operations — corpus management

The schema-learner's corpus is the toolkit's memory. As surfaces get triaged, they enter the approved corpus and future migrations benefit automatically.

### 8.1 Adding a WatchGuard config to the corpus

```bash
# Place the XML in corpus/
cp /path/to/customer-firebox-export.xml schema_learner/corpus/

# Re-run the schema-learner to incorporate it
python3 schema_learner/schema_learner.py \
    --input schema_learner/corpus/customer-firebox-export.xml \
    --source-label customer-name
```

If the new config has surfaces already in the approved corpus, the engine emits `auto_reused` findings — those surfaces are auto-handled. Only truly novel surfaces require triage.

### 8.2 Listing sideline queue

```bash
python3 schema_learner/triage.py sideline list
```

Shows all surfaces awaiting deep research or that were Phase-6-deferred. Status field indicates lifecycle:

- `open` — operator-driven sideline, awaiting research
- `deferred_disabled_in_source` — Phase 6 deferral, will auto-promote
- `pending_triage` — Phase 6.5 auto-promoted, awaiting operator action
- `in_research` — Phase 4 deep-research in progress
- `resolved_via_triage` — completed; surface now in approved corpus

### 8.3 Claiming a sideline entry for research

```bash
python3 schema_learner/triage.py sideline claim \
    --entry-id sl_Botnet_20260510_165907 \
    --operator jimames
```

Transitions status to `in_research` and acquires a lock so no other engineer claims it simultaneously.

### 8.4 Promoting a sideline entry to approved corpus

After completing deep research (per Phase 4 workflow):

```bash
python3 schema_learner/triage.py sideline promote \
    --entry-id sl_Botnet_20260510_165907 \
    --operator jimames \
    --citations https://watchguard.com/...,https://watchguard.com/...
```

Requires `--citations` with at least two distinct URLs (Phase 4 requires two citations per claim) plus lab-verification artifacts on disk.

### 8.5 Listing banned-corpus entries

```bash
python3 schema_learner/triage.py banned list
```

Shows all entries that were marked harmful. Each entry suppresses future similar surfaces.

### 8.6 Appealing a banned entry

Sometimes a banned entry was a mistake. To appeal:

```bash
python3 schema_learner/triage.py banned appeal \
    --entry-id banned_Foo_20260501 \
    --operator other_engineer \
    --reason "After lab investigation, this surface IS legitimate. See LAB_REPORT_xyz."
```

Goes into a disagreement-flag lifecycle (Phase 5). Another operator reviews the appeal and resolves it.

---

## 9. Operations — team coordination

When more than one senior engineer uses the toolkit, Phase 5 infrastructure kicks in.

### 9.1 Setting up the roster

Edit `team_config/operators.ini`:

```ini
[operators]
jimames = senior
new_engineer_name = senior

[meta]
roster_version = 2
bootstrap_mode = false
```

`bootstrap_mode = true` allows any operator name (use during first-engineer setup). Set to `false` once the roster has named operators.

### 9.2 Locks during triage

When an operator starts a triage walk, the toolkit acquires per-surface locks in `team_config/locks/`. Another operator running `triage walk` on the same proposal will see:

```
✗ Cannot acquire lock for <Botnet>: held by jimames since 2026-05-13T10:15:00
   Lock will expire at 2026-05-13T14:15:00 (4h default timeout).
```

Locks expire automatically. If an operator's session crashes, the lock will be reclaimed after the timeout.

### 9.3 Audit log

Every triage action is appended to a monthly audit log at `schema_learner/audit/2026-05.log`. Format: one action per line, timestamp + operator + action + target.

The log is **append-only**. Even Anthropic Claude (operating in the dev sandbox) can't rewrite this — the principle binding extends to the assistant itself.

To review the log:

```bash
tail -50 schema_learner/audit/$(date +%Y-%m).log
```

### 9.4 Disagreement flags

When two operators disagree about a classification, the toolkit supports a disagreement-flag lifecycle:

```bash
# Engineer B files a disagreement on Engineer A's classification
python3 schema_learner/triage.py disagreement file \
    --against-entry-id approved_Botnet_20260510 \
    --operator new_engineer_name \
    --reason "<Botnet> should be category 'object-library' not 'threat-detection' because..."

# Engineer A or a third engineer resolves the disagreement
python3 schema_learner/triage.py disagreement resolve \
    --flag-id dis_20260513_001 \
    --operator jimames \
    --resolution keep    # or amend, or revoke
```

Disagreement flags are visible in `team_config/disagreements/` and reviewed at quarterly cadence.

---

## 10. Outputs reference

### 10.1 Parser outputs (`sonicwall_parser/output/`)

Each section JSON has the shape:

```json
{
  "entries": [ { "_section": "...", "_id": "...", ... }, ... ],
  "_meta": {
    "parser_version": "1.0.0",
    "parser_name": "SonicOS Enhanced -> Canonical JSON Parser",
    "source_file": "input/customer-nsa3600.txt",
    "generated_at": "2026-05-13T10:15:00",
    "section": "access_rules",
    "description": "Firewall access rules",
    "count": 25
  }
}
```

Every entry carries `_section`, `_id`, `_source_lines` for traceability back to the raw CLI.

### 10.2 Relevance filter outputs (`sonicwall_parser/filtered/`)

Each bucket JSON has the shape:

```json
{
  "_meta": {
    "bucket": "customer_defined",
    "generated_at": "2026-05-13T10:16:00",
    "section_count": 4,
    "entry_count": 5
  },
  "by_section": {
    "access_rules": [ {... entry with _classification added ...}, ... ],
    "nat_policies": [ ... ],
    ...
  }
}
```

Every classified entry has an added `_classification` field:

```json
{
  "_classification": {
    "bucket": "customer_defined",
    "reason": "customer access-rule 'fakepolicyforclaude'",
    "classified_at": "2026-05-13T10:16:00"
  }
}
```

### 10.3 Schema-learner outputs (`schema_learner/proposals/<run>/`)

Proposal directories contain:

| File | Purpose |
|---|---|
| `findings.json` | Machine-readable engine output. The triage CLI consumes this. |
| `findings.md` | Operator-readable engine output. Human review. |
| `OPERATOR_INTERVENTION_REQUIRED.md` | The file the operator edits during triage. |
| `proposed_diff.md` | Preview of INI changes the merge would make. |
| `merge_preview/` | Staged INI changes (applied on merge). |
| `triage_state.json` | Per-run progress + locks. |
| `drafts/<surface>.json` | Cached LLM drafts. |

### 10.4 Validator outputs (`wg_validator/output/`)

The validator emits:

| File | Purpose |
|---|---|
| `validation_report.md` | Human-readable validation report. |
| `validation_findings.json` | Machine-readable findings. |
| `cross_reference_report.md` | Cross-reference checks (e.g., does this policy's `from` zone exist?). |

### 10.5 Emitter outputs (`xml_emitter/output/`)

Final WatchGuard XML goes here, ready to load into a Firebox:

```
xml_emitter/output/
├── <customer>.xml                  the final WatchGuard config
├── emit_report.md                  what was emitted, what was skipped
└── concept_map_coverage.md         which concepts were exercised
```

---

## 11. Troubleshooting

### 11.1 "0 entries classified" from relevance filter

The parser output is stale or empty. Check:

```bash
python3 -c "
import json
data = json.load(open('sonicwall_parser/output/access_rules.json'))
print('source_file:', data['_meta']['source_file'])
print('entry count:', len(data['entries']))
"
```

If `source_file` doesn't match your input, re-run the parser.

### 11.2 LLM draft says "Turn 2 (extract) returned empty"

The LLM provider returned no usable response on the structured-extract turn. Causes:

- **GPT4All:** the named model isn't loaded in the GPT4All UI (check the UI, load the model). Or the model is too small to follow structured prompts (try Llama 3 8B or larger).
- **Anthropic:** timeout or rate limit. Check `ANTHROPIC_API_KEY` is valid and you have quota.
- **Any provider:** the prompt was too complex. Check `learn_config.ini` `[mcp].timeout_seconds` — increase if needed.

Stderr output from `llm_provider.py` now includes diagnostic messages for these. Read them.

### 11.3 "Cannot acquire lock" error during triage walk

Another operator (or a previous session that crashed) holds the lock. Options:

- Wait for the timeout to expire (default 4 hours)
- Manually clear `team_config/locks/<surface>.lock` (only if you're certain no one else is working on it)
- Contact the locking operator

### 11.4 Provenance audit failure

If `test_schema_learner_provenance.py` fails, the toolkit has produced an artifact missing required markers. Read the audit failure message carefully — it'll name the file and the missing marker. Either:

- The artifact was manually edited and lost its markers (restore from `schema_learner/backups/`)
- A new code path produced an artifact without going through the proper provenance-wrapping function (bug — fix the code path)

### 11.5 Parser drops customer surfaces

You can check this with a conservation check:

```bash
# count raw lines in the CLI export
grep -c '^access-rule' sonicwall_parser/input/customer.txt
grep -c '^address-object' sonicwall_parser/input/customer.txt

# count parsed entries
python3 -c "
import json
for s in ['access_rules', 'address_objects']:
    d = json.load(open(f'sonicwall_parser/output/{s}.json'))
    print(f'{s}: {len(d[\"entries\"])}')
"
```

If these don't match, the parser is dropping entries. Look in `_unparsed.json` first. If empty, then the parser's regex for those sections doesn't match this customer's CLI variant. Adjust `parser_config.ini`.

### 11.6 Relevance filter classifies something wrong

Operator override is always available:

- **Wrong vendor_builtin classification:** edit `sonicwall_parser/builtin_objects.ini` to remove or add the entry, then re-run the filter
- **Wrong dead_unreferenced:** check `sonicwall_parser/output/` for the section that should reference it. The reference might be in a syntax the index doesn't recognize yet — file a bug.
- **Wrong customer_defined classification:** manually move the entry to the correct bucket in `sonicwall_parser/filtered/*.json` (the translator reads these files; the in-process classification is advisory only).

---

## 12. Glossary

| Term | Meaning |
|---|---|
| **Approved corpus** | The toolkit's verified-correct knowledge base. Surfaces here auto-reuse on future migrations. |
| **Banned corpus** | Surfaces marked as actively-wrong. Future similar surfaces are auto-suppressed. |
| **Brick risk** | The danger that importing a malformed XML into a Firebox makes the device unreachable. The toolkit is conservative because of this. |
| **Conservation check** | Verification that parsed-entries-in equals classified-entries-out. Phase 7 hardening. |
| **Customer-defined** | A surface that exists in a customer's config and isn't a vendor built-in. The translator's actual workload. |
| **Deep research** | Phase 4 workflow for surfaces that need lab verification before being added to approved corpus. |
| **Disagreement flag** | A Phase 5 lifecycle for when two operators disagree about a classification. |
| **Findings** | Engine-produced structural observations. Inputs to triage. |
| **Freshness check** | Phase 7 hardening. Reads `_meta.source_file` to detect stale parser output. |
| **MSP** | Managed Service Provider — n2nhu lab's business model. |
| **Novelty detection** | The schema-learner's job: identify XML element types/shapes the corpus doesn't already cover. |
| **Operator** | A senior engineer authorized to perform triage. Named in `team_config/operators.ini`. |
| **Phase N** | A milestone in the toolkit's build history. Phase 0 = foundation, Phase 7 = current. See `DESIGN_team_scale_addendum.md`. |
| **Principle binding** | The architectural enforcement of "no claim without provenance." Cannot be disabled. |
| **Provenance markers** | Inline markers (`OBSERVATION`, `OPERATOR-AUTHORED-BEGIN`, etc.) that identify who said what. |
| **Sideline queue** | Surfaces awaiting deep research or Phase 6-deferred. |
| **Structural observation** | Engine-produced fact about XML shape (counts, child tags, similarity). |
| **Surface** | Any classifiable element in a config — an XML element type, a SonicWall section entry, etc. |
| **Tier 1/2/3** | Phase 6 finding classification: 1 = full triage, 2 = defer with config, 3 = auto-skip. |
| **Translator** | The middle layer (`concept_map/` + `xml_emitter/`) that maps SonicWall → WatchGuard. |
| **Vendor built-in** | An object shipped by the vendor by default. Maps 1:1 across vendors; no operator decision needed. |

---

## Appendix A — Health snapshot at time of writing

```
16 test suites, all green, ~33s wall time:
  test_comment_preserving_ini.py
  test_comment_preserving_multiline.py
  test_gui_save_roundtrip.py
  test_engine_validation_wiring.py
  test_wg_validator.py
  test_schema_learner_provenance.py
  test_merge_engine.py
  test_auto_reuse.py
  test_triage.py
  test_phase3_llm.py
  test_phase4_deep_research.py
  test_phase5_team_coordination.py
  test_phase6_tier_filtering.py
  test_phase6_5_auto_promotion.py
  test_phase7_relevance_filter.py   (25 tests)
  test_phase8_skeleton_fixes.py     (13 tests — post-friend-review)

Total individual tests across the toolkit: 145+
```

## Appendix B — Reading order for further learning

For someone who's read this manual and wants more depth, in order:

1. `schema_learner/DESIGN.md` — original architectural document
2. `schema_learner/DESIGN_team_scale_addendum.md` — team-scale design decisions (LOCKED)
3. `docs/algebraic-design.pdf` — Object × Verb = Action theory
4. `wg_validator/CROSS_REFERENCE_FINDINGS.md` — cross-reference rule catalog
5. The test suites in `harness/` — each one is a worked example of correct behavior

For Anthropic Claude (or other LLM assistants) being onboarded as collaborators:

- This manual
- `schema_learner/DESIGN_team_scale_addendum.md`
- `sonicwall_parser/builtin_objects.ini` (to understand operator-curated lists)
- The most recent 2-3 test suites (to understand current invariants)

---

*Document version: 1.0 — 2026-05-13*
*Captain: Jim Ames, n2nhu lab, Newburgh NY*
*Maintained alongside the codebase. If you update a behavior, update this manual.*
