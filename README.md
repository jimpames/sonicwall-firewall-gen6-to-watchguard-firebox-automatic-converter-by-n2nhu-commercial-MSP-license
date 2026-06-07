# SonicWall → WatchGuard Migration Toolkit

**n2nhu lab — Jim Ames**

Reproducible converter that transforms a SonicWall NSA-3600 (or compatible SonicOS) configuration export into a WatchGuard Firebox T-30 (or compatible Fireware 12.5.9) XML configuration that loads on the box AND renders correctly in the Web UI.


sonicwall to watchguard automatic converter -firewall ejector seat v11 end to end cli demo -n2nhu

<img width="2816" height="1536" alt="Gemini_Generated_Image_978lkj978lkj978l" src="https://github.com/user-attachments/assets/7573b7f9-2605-470e-9647-e1e9b084e288" />

[![Watch the video - 25 mins end to end CLI demo](https://youtu.be/VcJuLtbdjyE?si=-V1HSB2V2sT3tO0J)](https://youtu.be/VcJuLtbdjyE?si=-V1HSB2V2sT3tO0J)

---

## Quick Start

```bash
# Unpack
unzip migration_toolkit_v6.zip
cd migration_toolkit

# Convert (defaults to sonicwall_parser/input/nsa3600.txt)
python3 skeleton_engine/build_pipeline.py --full-rebuild

# Convert a different SonicWall config
python3 skeleton_engine/build_pipeline.py \
    --input /path/to/customer-sonicwall-export.txt \
    --full-rebuild

# Output appears at:
#   skeleton_engine/output/configuration.xml
# Upload that file to the Firebox via Policy Manager or Web UI Restore.
```

If all 8 validators exit **0**, the file is ready for the Firebox.

---

## Architecture

The toolkit is a **learning system** with a translation layer on top, not a hand-written translator. The pipeline:

```
sonicwall_parser  →  parsed canonical JSON
        ↓
enricher          →  customer records (interfaces, services, address-groups,
                                       access-rules, VPN policies, schedules)
        ↓
immutables_classifier  →  per-jpa-list classification (STRUCTURAL_LOCK,
                                                       PRESERVE_AND_APPEND,
                                                       FROZEN, etc.)
        ↓
skeleton_extractor     →  skeleton.xml + templates.json from jpa.xml
        ↓
jpa_scrubber           →  removes jpa-customer-private entries from skeleton
                          (driven by scrub_config.ini, algebraic-style INI)
        ↓
leaves_quality_extractor  →  per-XPath leaf-quality matrix from jpa.xml
        ↓
skeleton_filler        →  fills skeleton with customer records
                          (per-list FILLERS + cross-list fillers for
                           abs-policy / abs-ipsec-action / VPN coordination)
        ↓
8 predictive validators (all must exit 0 to ship)
   - qualities_validator       (per-XPath value qualities)
   - ref_integrity_validator   (cross-references resolve)
   - immutables_validator      (hardware-locked entries unchanged)
   - customer_fidelity_validator (every enricher record present)
   - schema_shape_validator    (no invented tags)
   - private_data_validator    (no jpa-customer-private strings)
   - required_children_validator (jpa-derived, XPath-disambiguated)
   - jpa_diff                  (structural diff against jpa.xml)
        ↓
configuration.xml ready for upload to Firebox
```

---

## Key Architectural Principles

**1. No claim without provenance.**
Every architectural decision references a specific Firebox 12.5.9 import error or a verified jpa.xml shape. No guesswork; no claims of "should work" without empirical evidence.

**2. jpa.xml is ground truth.**
The reference Firebox export (`schema_learner/corpus/T30jpa1.xml`) defines the shapes, fields, and conventions. The toolkit derives rules from it empirically rather than hardcoding them.

**3. Alias IS the WatchGuard address object.**
Customer SonicWall address objects and address groups appear in `alias-list` (the user-facing list). Each alias with IP members pairs with an internal `<name>.1.alm` address-group that carries the IPs.

**4. WG Web UI panels read from ABSTRACT lists.**
- Firewall Policies → `abs-policy-list`
- Branch Office VPN > Tunnels → `abs-ipsec-action-list`
- Branch Office VPN > Gateways → `ike-policy-group-list`
- Aliases → `alias-list`

Without entries in these abstract lists, panels appear empty even if the underlying lists are populated.

**5. Convention: customer policies use `-00` suffix + `.1.from` / `.1.to` wrapper aliases.**
Verified against working policy exports from a live Firebox:
- `policy-list/policy/name` ends with `-00`
- `abs-policy-list/abs-policy/name` is the bare name (no suffix)
- Each policy has paired wrapper aliases that contain the zone routing
- abs-policy back-references the policy by its `-00` name

**6. STRUCTURAL_LOCK = hardware-immutable only.**
Physical interfaces yes; VPN-virtual interfaces are user-creatable and scrubbable.

**7. Predictive validators only.**
Hardcoded validation rules are forbidden. The required_children_validator derives rules from jpa.xml: if a child appears in 100% of jpa instances at an XPath, it's required. XPath disambiguation uses grandparent context.

**8. Algebraic-design INI configs.**
Object × Verb = Action. Used by:
- `scrub_config.ini` — factory-allowlist + private-blocklist per jpa list
- `proxy_mapping.ini` — service → proxy-action mapping
- `parser_config.ini`, `enrich_config.ini` — per-section declarative parsing

---

## Configuration Files

| File | Purpose |
|---|---|
| `skeleton_engine/scrub_config.ini` | What to keep vs scrub from jpa.xml. Per-list factory allowlists, private-string blocklist. |
| `skeleton_engine/proxy_mapping.ini` | Service → WG proxy-action mapping. Services not listed become packet-filter policies. |
| `sonicwall_parser/parser_config.ini` | SonicOS section → JSON record extraction rules. |
| `enricher/enrich_config.ini` | Per-record enrichment + normalization rules. |
| `schema_learner/corpus/T30jpa1.xml` | The reference Firebox export. Ground truth for shapes, conventions, and required-children derivation. |

To extend support for a new SonicWall config:
- New services that have WG proxies → add to `[proxy_services]` in `proxy_mapping.ini`
- New jpa.xml introduces new customer-private strings → add to `[private_strings]` in `scrub_config.ini`

---

## CLI Reference

### Run the full pipeline

```bash
python3 skeleton_engine/build_pipeline.py [--input PATH] [--full-rebuild]
```

**Flags:**
- `--input PATH` — SonicWall config text file (default: `sonicwall_parser/input/nsa3600.txt`)
- `--full-rebuild` — Force re-extraction of skeleton + matrix + classifier (run this whenever jpa.xml or scrub_config.ini changes)

**Without `--full-rebuild`**, the pipeline reuses cached `skeleton.xml` + `templates.json` + `leaves_quality.json` + `immutables.json` if they're newer than `jpa.xml` and `scrub_config.ini`. Use `--full-rebuild` for the first run or after any change to jpa or scrub config.

### Run the diff engine

Compare any two WG XML configs structurally:

```bash
python3 diff_engine.py FILE_A FILE_B \
    [--label-a LABEL] [--label-b LABEL] \
    [--output report.txt] [--json report.json]
```

Example:

```bash
python3 diff_engine.py \
    skeleton_engine/output/configuration.xml \
    schema_learner/corpus/T30jpa1.xml \
    --label-a EMIT --label-b JPA \
    --output diff_report.txt
```

Outputs: section-by-section counts, name-set diffs (only-in-A, only-in-B), shape diffs (parent/child tag pairs).

### Run individual validators

```bash
python3 skeleton_engine/qualities_validator.py            skeleton_engine/output/configuration.xml
python3 skeleton_engine/ref_integrity_validator.py        skeleton_engine/output/configuration.xml
python3 skeleton_engine/immutables_validator.py           skeleton_engine/output/configuration.xml
python3 skeleton_engine/customer_fidelity_validator.py    skeleton_engine/output/configuration.xml
python3 skeleton_engine/schema_shape_validator.py         skeleton_engine/output/configuration.xml
python3 skeleton_engine/private_data_validator.py         skeleton_engine/output/configuration.xml
python3 skeleton_engine/required_children_validator.py    skeleton_engine/output/configuration.xml
python3 wg_validator/jpa_diff.py                          skeleton_engine/output/configuration.xml
```

Each exits `0` on clean, `1` on findings, `2` on internal error.

### Run the scrubber standalone

```bash
python3 skeleton_engine/jpa_scrubber.py [--skeleton PATH] [--config PATH] [--output PATH]
```

Defaults: scrubs `skeleton_engine/output/skeleton.xml` in place using `skeleton_engine/scrub_config.ini`.

---

## Where Things Live

```
migration_toolkit/
├── README.md                           ← you are here
├── diff_engine.py                      ← side-by-side structural XML diff
├── sonicwall_parser/
│   ├── parser_config.ini               ← SonicOS section parsing rules
│   ├── input/
│   │   ├── nsa3600.txt                 ← the customer's SonicWall export
│   │   └── slimmed-sonicwallconfig.txt ← smaller test config
│   └── output/                         ← canonical JSON (one file per record type)
├── enricher/
│   ├── enrich_config.ini               ← per-record enrichment rules
│   └── output/                         ← customer records (consumed by filler)
├── schema_learner/
│   └── corpus/T30jpa1.xml              ← REFERENCE Firebox export
├── skeleton_engine/                    ← the heart of the toolkit
│   ├── build_pipeline.py               ← orchestrator (run this)
│   ├── immutables_classifier.py
│   ├── skeleton_extractor.py
│   ├── leaves_quality_extractor.py
│   ├── jpa_scrubber.py
│   ├── scrub_config.ini                ← factory allowlist + private blocklist
│   ├── skeleton_filler.py              ← FILLERS + CROSS_LIST_FILLERS
│   ├── proxy_mapping.ini               ← service → WG proxy-action map
│   ├── qualities_validator.py
│   ├── ref_integrity_validator.py
│   ├── immutables_validator.py
│   ├── customer_fidelity_validator.py
│   ├── schema_shape_validator.py
│   ├── private_data_validator.py
│   ├── required_children_validator.py
│   └── output/
│       ├── configuration.xml           ← FINAL DELIVERABLE
│       ├── skeleton.xml                ← intermediate
│       ├── templates.json              ← per-list shape variants
│       ├── leaves_quality.json         ← per-XPath leaf qualities
│       └── immutables.json             ← list classification
├── wg_validator/
│   └── jpa_diff.py                     ← structural diff vs jpa
└── docs/                               ← design decisions, principles
```

---

## Loading the output on the Firebox

1. Verify all 8 validators exit 0 (the build_pipeline final report confirms this).
2. Take the SHA256 of `skeleton_engine/output/configuration.xml`.
3. Upload via WatchGuard Policy Manager (File > Restore Configuration) or the Web UI System > Configuration File panel.
4. On Restore Success: check the Web UI panels:
   - **Firewall > Firewall Policies** — customer access rules should appear with the right zone From/To.
   - **VPN > Branch Office VPN > Gateways** — site-to-site IKE gateways.
   - **VPN > Branch Office VPN > Tunnels** — site-to-site IPsec tunnels.
   - **Firewall > Aliases** — customer named address objects/groups.

---

## Toolkit licensing

This toolkit is offered for sale.

- $10K unlimited MSP license

Contact: **Jim Ames, n2nhu lab, Newburgh NY** 
jimpames@gmail.com 

---

*"No claim without provenance. $1 on diff saves $100 on debug."*
