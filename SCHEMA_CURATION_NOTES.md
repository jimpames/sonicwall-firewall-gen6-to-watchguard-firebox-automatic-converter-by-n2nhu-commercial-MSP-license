# Schema Curation — Pass #1 Notes

This document captures the hand-curation work done in #3b (the schema
curation pass) plus follow-up findings that emerged from the
investigation but are out of scope for this pass.

---

## Summary

The auto-bootstrapped `wg_xml_schema.ini` produced 49 false-positive
warnings on the slim emit (and 20 false-positives on the reference
itself). Hand-curation of **6 element rules** brought both to 0/0/0.

| Element rule | What changed | Reason |
|---|---|---|
| `regexp` | Removed `allowed_values` enum | Free-form regex strings; observed enum was just 4 patterns from one reference |
| `proxy-type-attr` | Removed `allowed_values` enum | Comma-joined multi-values are common; the inferer wrongly split them on commas |
| `alias-name` | Removed `allowed_values` enum | Free-form name reference; cross-reference rules already validate resolution semantics |
| `exceptions` | Removed `required_fields` | `rule` was inferred as required but is actually present in only ~half of instances |
| `auth` | Removed `required_fields` | `fallthrough, rule` were inferred required but present in only 8/12 reference instances |
| `physical-if` | Removed `required_fields` | Inferer marked 22 fields as required from 5 samples — heavy overfit; many fields are interface-type-conditional (e.g. LAN interfaces don't have default-gateway) |

## Validation results after curation

| Surface | Before | After | Δ |
|---|---|---|---|
| Reference XML | 20 warnings | **0/0/0** | -20 |
| Slim sample emit | 49 warnings | **0/0/0** | -49 |

Both now validate cleanly against schema + cross-reference rules +
built-in name allowances. No errors, no warnings, no info notes.

## Toolkit-wide regression

```
test_comment_preserving_ini.py   PASS  (16 INIs round-trip)
test_gui_save_roundtrip.py       PASS  (16 INIs incl. curated schema)
test_engine_validation_wiring.py PASS  (5 engines × 3 scenarios)
test_wg_validator.py             PASS  (12/12 — including newly tightened
                                       validates_reference_cleanly and
                                       validates_slim_emit asserting 0/0/0)
                                 4/4 in 7.5s
```

Diff score: 0.9563 (unchanged from previous session).

---

## Follow-up finding: phantom-interface emit bug ✅ FIXED

Surfaced during physical-if investigation. **Resolved in a subsequent
session via merge-into-existing emit-rule semantics.**

### The bug (was)

The slim emit produced **18 `<interface>` elements** in
`<interface-list>`:

| Source | Count | Names | Status |
|---|---|---|---|
| Master skeleton (pre-populated) | 13 | `Any`, `Any-External`, ..., `External`, `Trusted`, `Optional-1/2/3` | Fully populated |
| `[emit.interfaces]` rule output | 5 | `X0`, `X1`, `X2`, `X3`, `X4` | Phantom — under-populated, missing 6 fields each |

The five `X*` entries duplicated and conflicted with the skeleton's
zone-named interfaces, using SonicWall's raw identifiers instead of
the WG-canonical zone names.

### The fix

Added a **generic merge-into-existing emit semantic** plus
**SonicWall→WG zone mapping**.

**1. Engine extension (`xml_emitter.py`):**

Added two new EmitRule fields:

- `merge_into_existing_by` — when set to a child-element tag
  (e.g. `name`), the engine looks up an existing element under
  `xml_parent_xpath` whose `<child-tag>/text` matches the resolved
  `final_name`, and applies child_map + static_children to THAT
  element instead of creating a new one. In merge mode, child_map
  OVERWRITES skeleton placeholder text rather than appending siblings.

- `derive_zone_from_first_token` — pre-emit transformation. For each
  entry, takes the first whitespace-token of the named field and
  stores it as `zone`. Robust to two SonicWall parser shapes (dict
  with `_args` or flat string).

The engine also now handles three new logging categories (which
appear in `migration_notes.md`):

- `skipped_no_name` — entries with no resolvable name (e.g. SonicWall
  interface with no IPv4 zone configured)
- `merge_collisions` — multiple source entries mapping to the same
  WG canonical interface (first-wins; later entries dropped with
  audit log)
- (existing) `skipped_by_pattern` — regex-matched filtering

**2. Concept-map addition (`concept_map.ini`):**

```ini
[value_map.sw_zone_to_wg_iface_name]
sonicwall_to_canonical  = LAN=LAN, WAN=WAN, DMZ=DMZ, VPN=VPN, ...
canonical_to_watchguard = LAN=Trusted, WAN=External, DMZ=Optional-1,
                          VPN=Optional-2, WLAN=Optional-3, ...
```

**3. Emit-rule rewire (`xml_emit_config.ini`):**

```ini
[emit.interfaces]
derive_zone_from_first_token = ip-assignment._args
name_attribute               = zone
value_maps                   = wg_bool_text, sw_zone_to_wg_iface_name
merge_into_existing_by       = name
child_map = ip-assignment.ip      = if-item-list/item/physical-if/ip
            ip-assignment.netmask = if-item-list/item/physical-if/netmask
            ip-assignment.gateway = if-item-list/item/physical-if/default-gateway
            comment               = description
            mtu                   = if-item-list/item/physical-if/mtu
```

The `id = if-dev-name` mapping was REMOVED — the skeleton already
has the correct dev-names (eth0, eth1, etc.), and the merge
semantics preserve them.

### Results

| Surface | Before | After |
|---|---|---|
| Total `<interface>` count | 18 | 13 |
| Phantom interfaces (X0-X4) | 5 | 0 |
| External (eth0) IP | (skeleton placeholder) | `141.10.150.12` (real X1 WAN data) |
| Trusted (eth1) IP | (skeleton placeholder `192.168.1.253`) | `10.12.20.177` (real X0 LAN data) |
| Validation findings | 0/0/0 | 0/0/0 (held) |

### SonicWall → WG mapping behavior

The slim sample's 5 SonicWall interfaces map as follows:

| SonicWall | Zone | WG Target | Result |
|---|---|---|---|
| X0 | LAN | Trusted (eth1) | Merged with X0's IP/netmask |
| X1 | WAN | External (eth0) | Merged with X1's IP/netmask/gateway |
| X2 | (no IPv4) | — | Skipped — logged: `Skipped IDs: X2` |
| X3 | LAN | Trusted (eth1) | Collision — first-wins; X0 kept, X3 dropped with note |
| X4 | LAN portshield | Trusted (eth1) | Collision — same as X3 |

### Known limitations (documented, not blocking)

- **Multi-port-same-zone collisions**: SonicWall lets you put
  multiple physical interfaces in the same zone (e.g. X0 and X3
  both in LAN). WatchGuard's canonical interfaces are 1:1. Current
  behavior: first-wins, others logged. A more sophisticated mapping
  would allocate excess same-zone interfaces to Optional-N slots in
  order of appearance — captured for a future session.

- **Portshield aggregation** (X4 = LAN portshield X0): SonicWall's
  portshield groups multiple physical ports into a virtual
  switch. WG's equivalent is a Bridge interface (different schema).
  Current behavior: ignored as a collision. Real bridge-mapping is
  a future-session topic.

### Regression test

`test_emitter_merges_interfaces_no_phantoms` in
`harness/test_wg_validator.py` verifies:

- No phantom `X0`-`X4` interface entries
- External has real X1 WAN IP (`141.10.150.12`)
- Trusted has real X0 LAN IP (`10.12.20.177`)
- migration_notes.md contains the X2 skip and X3/X4 collision entries

---

## Architectural note: schema curation lifecycle

The schema is auto-bootstrapped by `schema_inferer.py` from reference
XML. Hand-curation edits (like this pass) live in the resulting
`wg_xml_schema.ini` directly. Re-running the inferer would overwrite
these edits.

For now, the workflow is: **bootstrap once, curate going forward**.
If future reference exports need to be incorporated, the inferer
should be enhanced with a "merge" mode that preserves marked edits.
This is a #5/future architectural topic, not blocking.

---

## Schema state after pass #1

- **Element rules**: 773
- **Cross-reference rules**: 18
- **Built-in name categories**: 4 (alias, schedule, service, app-action)
- **Hand-curated rules**: 6 (regexp, proxy-type-attr, alias-name,
  exceptions, auth, physical-if)
- **Validation result on reference**: 0/0/0
- **Validation result on slim emit**: 0/0/0
