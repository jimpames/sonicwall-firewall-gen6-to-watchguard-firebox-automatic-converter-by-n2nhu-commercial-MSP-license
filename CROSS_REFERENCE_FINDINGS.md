# Cross-Reference Validation Findings

This document captures the actionable findings surfaced by the
**cross-reference validation phase** added to wg_validator. These are
real issues in the migration pipeline output that the validator
caught — work items in priority order.

---

## Summary

Initial run on the slim sample's emit output produced **83
cross-reference findings** spanning four distinct issues. After all
remediation work, **0 cross-reference findings** remain — all 83 are
resolved through three different mechanisms.

| Finding | Initial | Current | Status |
|---|---|---|---|
| `alias_member_alias_name` dangling | 47 | 0 | ✅ FIXED via skip_name_patterns + migration notes |
| `policy_service` → 'any' | 12 | 0 | ✅ FIXED via wg_canonical_service value_map |
| `abs_policy_service` → 'any' | 12 | 0 | ✅ FIXED via wg_canonical_service value_map |
| `abs_policy_settings_schedule` → 'always-on' | 12 | 0 | ✅ FIXED via wg_canonical_schedule value_map |

---

## Finding 1: 23 SonicWall-style aliases without WG definitions ✅ FIXED

**Severity:** Migration gap. **Status:** Fixed via skip_name_patterns
filter mechanism with migration notes.

**Was:** The slim emit produced 47 alias-member references to 23
distinct alias names that weren't defined in `<alias-list>`. All
were SonicWall system aliases auto-generated for each interface.

**Root cause:** SonicWall's `address-group` table contains 15
auto-generated entries per interface (`X0 IPv6 Addresses`,
`LAN IPv6 Subnets`, etc.) plus the system-wide
`Default Geo-IP and Botnet Exclusion Group`. The toolkit's
enricher passed all 15 through to the emitter, which faithfully
emitted them as `<alias>` definitions — but each contained
recursive references to OTHER auto-generated aliases that were
NOT in the address-group table (e.g. `X0 IPv6 Primary Static
Address`). Result: 15 orphan alias definitions referencing 23
nonexistent names = 47 dangling cross-references.

Investigation revealed: **none** of these 15 host aliases were
referenced from any policy, NAT rule, or VPN configuration in
the WG-meaningful XML. They were emit-only cruft.

**Fix:** Added a generic `skip_name_patterns` mechanism to the
emitter:

```ini
[emit.aliases]
...
skip_name_patterns = .* IPv6 (Link-Local|Primary (Dynamic|Static)) Address(?: Subnet)?$
                     .* IPv6 Subnets$
                     .* IPv6 Addresses$
                     .* Management IPv6 Addresses$
                     ^Default Geo-IP and Botnet Exclusion Group$
skip_note_category = SonicWall IPv6 system aliases (no WG equivalent)
```

The mechanism:
1. Any input entry whose `name` matches a regex pattern is dropped
2. Each filtered entry is logged to `migration_notes.md` with the
   exact pattern that matched
3. The dropped count is reported in the rule manifest

**Migration notes output:**
```markdown
### Filtered entries — emit.aliases

Category: **SonicWall IPv6 system aliases (no WG equivalent)**

15 entries from `address_groups.json` matched a skip_name_patterns
regex and were not emitted to `/profile/alias-list`. These are
typically SonicWall-only constructs with no WatchGuard equivalent.

| Skipped entry | Matched pattern |
|---|---|
| `LAN IPv6 Subnets` | `.* IPv6 Subnets$` |
| `LAN Interface IPv6 Addresses` | `.* IPv6 Addresses$` |
| ... (15 total) ... |
```

**Generic mechanism:** The `skip_name_patterns` directive works on
ANY emit rule. Future SonicWall constructs that have no WG
equivalent can be filtered the same way — declarative, INI-driven,
fully audited.

**Regression test:** `test_emitter_filters_orphan_ipv6_aliases`
in `harness/test_wg_validator.py`.


## Finding 2: lowercase `any` instead of capital `Any` ✅ FIXED

**Severity:** Real emit bug. **Status:** Fixed in concept_map and emit config.

**Was:** 12 `<policy>/<service>` and 12 `<abs-policy>/<service>` elements emit
`<service>any</service>`. The WG built-in service is **`Any`** with a
capital A.

**Root cause:** The SonicWall CLI parser faithfully records the
lowercase `any` from source. The `firewall_policies` and
`firewall_policies_runtime` emit rules then dynamically map
`service = service` straight from parser output, OVERWRITING the
static_children defaults.

**Fix:** Added `[value_map.wg_canonical_service]` in
`concept_map/concept_map.ini`:

```ini
[value_map.wg_canonical_service]
sonicwall_to_canonical  = any=any
canonical_to_watchguard = any=Any
```

Updated emit rules to apply it:
```
service = service | wg_canonical_service
```

Now emits `<service>Any</service>` correctly. Other service names
(e.g. "HTTP", "Apple Bonjour") pass through unchanged because they
don't match the `any` key.


## Finding 3: `always-on` instead of `Always On` ✅ FIXED

**Severity:** Real emit bug. **Status:** Fixed in concept_map and emit config.

**Was:** 12 `<abs-policy>/<settings>/<schedule>` elements emit
`<schedule>always-on</schedule>`. The WG built-in schedule is
**`Always On`** (canonical form, capital A, capital O, space).

**Root cause:** Same pattern as Finding 2 — parser records the literal
SonicWall form, child_map overwrites the static `Always On` default.

**Fix:** Added `[value_map.wg_canonical_schedule]` in concept_map:

```ini
[value_map.wg_canonical_schedule]
sonicwall_to_canonical  = always-on=always-on
canonical_to_watchguard = always-on=Always On
```

Updated emit rule to apply it:
```
schedule = settings/schedule | wg_canonical_schedule
```

Now emits `<schedule>Always On</schedule>` correctly. The
firewall_policies_runtime rule already used the static-only default
(no dynamic schedule mapping), so no change needed there.

**Regression test:** `test_emitter_canonicalizes_service_and_schedule`
in `harness/test_wg_validator.py` verifies both forms appear in slim
emit output and the lowercase forms do NOT.


## Finding 4: 80 `value-domain` warnings (existing, unrelated to xrefs)

These are leaf elements whose text values aren't in the schema's
inferred enum. Most are real divergences (different IP addresses
between slim sample and reference), some are inference artifacts
(small reference sample → over-narrow enum).

These are not cross-reference findings; they pre-existed.

---

## Validator design notes

- All cross-reference rules have configurable severity. Currently
  most are `severity = warning`, with the IKE/IPsec stitching rules
  set to `severity = error` (those would actually break VPN import
  on a real Firebox).
- Built-in name allowances live in `[builtin.X]` sections of
  `wg_xml_references.ini`. Categories: `alias`, `schedule`,
  `service`, `app-action`. Hand-curated; can be expanded as more
  built-ins are discovered.
- Validator uses ElementTree's `findall` for xpath matching.
  Supports `.//tag/subtag` patterns; doesn't support full XPath 1.0.
  Sufficient for our purposes given the WG XML structure is
  hierarchical without ambiguity.
- The `validates_reference_cleanly` test confirms the T-30
  reference produces ZERO errors when validated against its own
  rules (including cross-references). 20 warnings remain — those
  are inference-artifact false positives we accept until
  hand-curation completes.
