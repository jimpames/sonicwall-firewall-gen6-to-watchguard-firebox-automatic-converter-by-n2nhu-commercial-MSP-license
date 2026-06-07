# SonicWall → WatchGuard Migration Toolkit

**n2nhu lab — Jim Ames**

Reproducible converter that transforms a SonicWall NSA-3600 (or compatible SonicOS) configuration export into a WatchGuard Firebox T-30 (or compatible Fireware 12.5.9) XML configuration that loads on the box AND renders correctly in the Web UI.

7 jun 26 update

v11 is now in beta
- brings over interfaces, address objects (aliases), vpns and policies

- mail me to get a NDA/beta test license - jimpames@gmail.com

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
- pays for itself in 10 conversion or less
- stop paying engineers to read and write configs - let them work on corner cases instead
- you're printing money after the 10th conversion!
- binary license only
- no sales outside MSP arena - contact your MSP - tell them you want Firewall Ejector Seat

Contact: **Jim Ames, n2nhu lab, Newburgh NY** 
jimpames@gmail.com 

-------
supergrok analysis of INI files functions

Got it, Jim — I've mapped the full flow from the INI configs (and supporting docs). This is a beautiful algebraic/MTOR-style system: everything declarative in INI, provenance-driven, "no claim without provenance," with heavy use of Object × Verb = Action.
High-Level Pipeline (from README + configs)

SonicWall Parser (parser_config.ini) → Canonical JSON (one file per logical section).
Enricher (enrich_config.ini) → Normalizes, synthesizes missing bits, pivots, merges.
Skeleton Engine (from T30jpa1.xml reference export):
Extract/scrub skeleton (skeleton_config.ini, scrub_config.ini).
Fill with migrated data (skeleton_filler + cross-list coordination).

Emit → configuration.xml (WatchGuard Fireware XML).
8 Predictive Validators — All must pass (ref integrity, private data scrub, schema shape, diff vs. jpa, etc.).
Output ready for Policy Manager / Web UI restore.

The whole thing stays schema/config-driven. No hard-coded vendor logic in Python where it can be avoided.
Key INI "Map" Breakdown
1. Parser (parser_config.ini) — SonicOS to JSON

Declarative sections: [section.<name>] with match_regex, form (block/single_line), output_file, key_field, list_fields.
Handles major areas: address_objects, address_groups, service_objects/groups, schedules, zones, interfaces (IPv4/IPv6), access_rules, nat_policies, vpn_policies (site-to-site, group-vpn, etc.), route_policies, DHCP, Wi-Fi, firewall_global, SSLVPN, auth, admin, SD-WAN, content_filter, etc.
State machine: block_terminator=exit, indentation, quote stripping, negation handling, etc.
Output: One JSON per category under output/.

2. Enricher (enrich_config.ini) — Cleanup & Synthesis (Phase 1.5)

Multiple ordered [pass.X] (derive, pivot, transform, filter, merge, expand, audit).
Key passes:
derive_implicit_addresses: Synthesizes SonicWall's silent "X0 Subnet", "X0 IP", etc. (critical for VPNs).
pivot_vpn_proposals: Flattens IKE/IPsec proposal lines into structured phase1{} / phase2{} dicts.
normalize_address_group_members: Strips family prefixes for WG compatibility.
drop_sonicwall_default_address_groups: Cleans empty IPv6 placeholders.
merge_tcp_udp_twins + expand_service_groups: Handles SonicWall's split services/groups into WG-friendly merged services.
expand_schedules: Converts recurring specs to WG day-of-week shape.
audit_vpn_crypto: Flags weak/deprecated algos via concept_map tables.

Uses concept_map.ini heavily.

3. Concept Map (concept_map.ini) — The Rosetta Stone

Vendor-neutral concepts → SonicWall term + WatchGuard term.
Covers: Addresses (concrete vs. collections — note the vocabulary swap!), services, policies, NAT, IKE/IPsec (phase1/2, PSK), interfaces/zones, routes, schedules.
Value maps for enums (e.g., WG IPsec algos, schedule types).
Warnings for breaking/lossy translations (e.g., mobile VPN client mismatch).

4. Skeleton & Scrubbing (skeleton_config.ini + scrub_config.ini)

Skeleton: Starts from real T30jpa1.xml, scrubs to reusable base.
empty_lists: Pure customer data (e.g., policies, NAT).
additive_lists: Keep factory built-ins (aliases, proposals, schedules, etc.), append migrated content.
preserve_lists / scaffolding: Device defaults + importer-required structure.
Redact secrets (PSK, certs, passwords).

Scrubber (scrub_config.ini): Factory-allowlist + private-blocklist per list (aliases, address-groups, policies, etc.). Patterns for auto-generated .1.from/.to. Backstop [private_strings] validator (FAKECO, IPs, etc.).

5. Policy Emission Details (proxy_mapping.ini)

Maps services to WG proxy-actions (HTTP → Default-HTTP-Client, etc.) or falls back to packet-filter.
Service name normalization (case, variants like H.323).
Firewall action codes.

6. Validators & Diff (wg_validator_config.ini, etc.)

8 validators + jpa_diff for structural comparison.
Schema-driven (wg_xml_references.ini, etc.), predictive from jpa reference.
Private data, ref integrity, required children, qualities, immutables, customer fidelity.

Overall Strengths (Algebraic/MTOR Flavor)

Config = executable spec (INI as "homomorphism").
Empirical: Everything grounded in T30jpa1.xml + real exports.
Safety: Scrub → validators → diff. Relevance/dead-object pruning mentioned in your earlier work fits here.
MSP-ready: Binary license, provenance docs, side-by-side VPN demos.

This is a production-grade, auditable migration engine. v11 bringing interfaces + addresses + VPNs + policies is a big milestone.
---

*"No claim without provenance. $1 on diff saves $100 on debug."*
