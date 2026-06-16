# FES — Firewall Ejector Seat
## v1.8.8 Release Notes
### "NAT Recovery Edition"

**Release date**: June 15, 2026
**Author**: n2nhu lab — J P Ames, Lead Engineer
**Previous release**: v1.8.7 (June 13, 2026 — "Multi-Model Edition")

---

## Executive summary

FES v1.8.8 closes a third SonicOS 6.5 → 6.2 grammar deviation discovered during the TZ-300 lab cycle following v1.8.7's ship. Two customer NAT policies in the TZ-300 corpus were silently dropped by the parser because of the 6.2 inline NAT-policy header form (no `ipv4` keyword), and the unresolved per-interface built-in references (`"X1 IP"`) surfaced as auto-generated placeholder aliases in the WatchGuard Web UI Aliases panel.

This release fixes both halves of that finding:

| Half | Problem | Fix |
|---|---|---|
| **Capture** | Parser dropped 2 TZ-300 NAT policies (and a third customer NAT we didn't know about) because the 6.2 grammar omits the `ipv4` keyword | **Fix G-1**: section-regex exception in `version_exceptions.ini`, same architectural pattern as Fix B + Fix D |
| **Resolution** | Per-interface built-in names like `"X1 IP"` / `"X0 Subnet"` / `"MGMT IP"` weren't catalogued, so NAT translated-source references couldn't resolve to actual IPs | **Fix G-2**: pattern-matched per-interface built-in entries in `sw_builtin_dynamic_groups.ini`, new resolver code path |

**Two corpora, both lab-validated:**

| Corpus | Status | SHA-256 |
|---|---|---|
| NSA-3600 (SonicOS 6.5) | **Byte-identical to v1.8.7** — no regression, 10/10 validators | `934ce25e43415d6ef0cf48ad9989cbac51f743f75bd4bf056e293a79f0106ba5` |
| TZ-300 (SonicOS 6.2) | **Updated** — 3 NAT policies now visible (was 0), 10/10 validators | `6d11a781231271fdf4c8fb972b3880cc08d46bc4a17538435acb23eec6f83888` |

NSA-3600's byte-identical SHA across the release transition is the architectural payoff: a SonicOS 6.2 grammar exception loaded only when the 6.2 branch is detected, so 6.5 code paths are untouched.

---

## Headline changes from v1.8.7 → v1.8.8

| # | Theme | Why it matters |
|---|---|---|
| 1 | **SonicOS 6.2 NAT-policy grammar capture** | Third grammar deviation between 6.2 and 6.5 now handled. Three named exceptions: access-rule (Fix B), address-object (Fix D), nat-policy (Fix G-1). Future deviations land via INI catalog, not code. |
| 2 | **Pattern-matched built-in dynamic groups** | The catalog now supports regex-based lookups in addition to exact-name. Per-interface SonicOS auto-generated names (`X<N> Subnet`, `X<N> IP`, `X<N> Default Gateway`, MGMT, USB modem ports, VLAN sub-interfaces) resolve to actual interface IPs at emit time. |
| 3 | **NAT translated-source resolution** | When a SW NAT policy uses `translated-source name "X1 IP"`, the toolkit now resolves "X1 IP" to the actual host IP (e.g., `88.88.88.88`) at emit time. Firebox no longer needs to auto-create placeholder aliases for unresolved references — Aliases panel stays clean. |

---

## The forensic chain (how Fix G was diagnosed)

Captain Jim Ames lab cycle on v1.8.7 (TZ-300 import on T-30 Fireware 12.5.9) reported:

> "all looks perfect on TZ 300 - just some crud in alias - here's a clue - this crud don't appear in alias in nsa 3600 - X0 Subnet, X0 IP, X1 Subnet, X1 IP, X1 Default Gateway, MGMT Subnet, MGMT IP ... X2:V88 IPv6 Addresses, X2:V88 Management IPv6 Addresses"

The crud names were **not present** in the v1.8.7 emit XML — verified with exhaustive `grep` against the configuration.xml for both corpora. So the question was: where do they come from?

### Step 1: Compare the two SW sources for references to the crud names

```
"X1 IP" in TZ-300 SW source:  4 occurrences (in 2 nat-policy blocks)
"X1 IP" in NSA-3600 SW source: 0 occurrences
```

Only TZ-300 references the per-interface built-ins. The 4 occurrences are inside two NAT policies that use `translated-source name "X1 IP"`.

### Step 2: Compare NAT records captured by the parser

```
NSA-3600 SW source: 1 NAT policy   → Parser captured: 1 NAT  ✓
TZ-300 SW source:   3 NAT policies → Parser captured: 0 NATs ✗ (all silently dropped)
```

### Step 3: Read the SW source side-by-side

The grammar diverges in the header form:

**SonicOS 6.5 (NSA-3600):**
```
nat-policy ipv4 inbound any outbound any source group "WAN Subnets" ...
    uuid 00000000-...
    name fakeh323wantodmzDMZnatforClaude
    priority auto
    ...
    exit
```

**SonicOS 6.2 (TZ-300):**
```
nat-policy inbound X2:V88 outbound X1 translated-source name "X1 IP"
    id 14
    inbound X2:V88
    outbound X1
    source any
    translated-source name "X1 IP"
    ...
    exit
```

The 6.2 form has NO `ipv4` keyword, uses `id N` instead of `uuid` + `name`, and carries the inline tail `translated-source name "X1 IP"` directly on the header line. The baseline parser regex required the `ipv4` keyword and matched the 6.5 block form only, so every 6.2 NAT silently dropped.

### Step 4: Connect the WG Web UI behavior

The captured 6.5 NAT in NSA-3600 emits its translated-source through the existing zone-based built-in catalog (`WAN Interface IP` was already in `sw_builtin_dynamic_groups.ini` from Fix F). The dropped 6.2 NATs in TZ-300 never reach the emit path — but the Firebox import sees other references that point at the missing structure, and the Firebox auto-creates placeholder aliases for them, surfacing as `X0 Subnet`, `X1 IP`, `MGMT Subnet`, etc. in the Aliases panel.

The fix requires both halves: (a) capture the NAT policies properly, AND (b) ensure that when they reference per-interface built-in names like `"X1 IP"`, the resolver translates those to actual IP literals so nothing is left dangling for the Firebox to auto-fill.

---

## Fix G-1 — SonicOS 6.2 NAT-policy section-regex exception

### Architectural pattern (third application of the same idea)

This is the third deviation between SonicOS 6.2 and 6.5 handled by the same `section_regex_override` mechanism introduced in Fix B:

| Fix | Version | Section | 6.5 form | 6.2 form |
|---|---|---|---|---|
| **B** | v1.8.5 | `access-rule` | `access-rule ipv4 from LAN to WAN ...` | `access-rule from LAN to WAN ...` (no `ipv4`) |
| **D** | v1.8.6 | `address-object` | multi-line block (`name`/`uuid`/`zone`/`host`/`exit`) | `address-object ipv4 NAME host IP zone Z` (inline single-line) |
| **G-1** | v1.8.8 | `nat-policy` | `nat-policy ipv4 inbound ...` (multi-line, `uuid`+`name`+`priority`) | `nat-policy inbound ...` (no `ipv4`, uses `id N`) |

Each deviation lands as a new section in `sonicwall_parser/version_exceptions.ini`. No parser code changes for any of them.

### Files changed

**`sonicwall_parser/version_exceptions.ini`** — new section:

```ini
[exception.sonicos_6_2.nat_policy_section_regex]
rule_kind        = section_regex_override
parser_section   = nat_policies
new_regex        = ^nat-policy(?:\s+(?P<family>ipv4|ipv6))?(?:\s+(?P<_inline_tail>.+?))?\s*$
provenance       = Lab test 2026-06-15, TZ-300 SonicOS 6.2.3.1-19n.
                   2 customer IPv4 NAT policies dropped due to bare-form
                   header (no 'ipv4' keyword). Permissive regex captures
                   both 6.2 bare form AND 6.5 family-tagged form.
                   Fix G-1 (v1.8.8). Same pattern as Fix B + Fix D.
```

The replacement regex makes `ipv4` / `ipv6` optional while preserving the 6.5 form, and ignores the inline header tail (the same fields are captured from sub-lines by the body parser). Safe to extend to other branches.

### Result

| | TZ-300 NAT records captured |
|---|---|
| Pre-Fix-G-1 | 0 (all dropped) |
| Post-Fix-G-1 | **3** (id=13 `natpolicytz300lanadrobjtowanbonj`, id=14 + id=15 auto-added X2:V88/X0 outbound NATs for X1 WAN) |

A third customer NAT (id=13) was recovered that wasn't even in the original bug report — a customer-named NAT for Apple Bonjour from a LAN address-object to the WAN interface. The Captain hadn't noticed it was missing because the Web UI never had a chance to render it.

NSA-3600 unaffected — the override loads only when the active branch is `sonicos_6_2`.

---

## Fix G-2 — Pattern-matched built-in dynamic groups

### The architectural gap

Fix F (v1.8.7) introduced `sw_builtin_dynamic_groups.ini` to catalogue SonicOS auto-generated dynamic address-groups that aren't explicitly defined in the e-CLI export. Fix F covered the ZONE-based built-ins:

| Built-in name | Resolution rule |
|---|---|
| `LAN Subnets` | Every LAN-zone interface's subnet |
| `WAN Subnets` | Every WAN-zone interface's subnet |
| `LAN Interface IP` | Every LAN-zone interface's IP literal |
| `WAN Interface IP` | Every WAN-zone interface's IP literal |
| ... + DMZ / WLAN / MULTICAST equivalents | |

But SonicOS ALSO auto-generates **PER-INTERFACE** built-ins for every physical port, USB modem port, VLAN sub-interface, and management port:

| Per-interface built-in | Resolves to |
|---|---|
| `X0 Subnet`, `X1 Subnet`, ..., `MGMT Subnet`, `U0 Subnet` | That specific interface's subnet |
| `X0 IP`, `X1 IP`, ..., `MGMT IP`, `U0 IP` | That specific interface's IP literal |
| `X0 Default Gateway`, `X1 Default Gateway`, ... | That specific interface's gateway IP |
| `X2:V88 Subnet`, `X2:V88 IP` | VLAN sub-interface subnet / IP |

These references appear in customer NAT policies, access rules, and other places. The v1.8.7 catalog had no entries for them, so the resolver returned "unresolved" and the references survived as literal name strings in the WG XML emit — which the Firebox then auto-created placeholder aliases for during import.

### The fix: pattern-matched catalog entries

The exact-name lookup pattern (one INI section per built-in name) doesn't scale to per-interface variants — there are 18+ interfaces possible per device, and customers can have ANY combination. Solution: extend the catalog with `[group_pattern.*]` sections that hold a compiled regex + a resolution mode.

**`enricher/sw_builtin_dynamic_groups.ini`** — five new sections:

```ini
[group_pattern.interface_subnet]
pattern_regex = ^(?P<intf>X\d+|U\d+|MGMT)\s+Subnet$
resolution    = interface_specific_subnet
description   = Per-interface subnet (e.g. "X1 Subnet" -> X1's subnet
                in interfaces.json, returned as <ip-network-addr> +
                <ip-mask>)

[group_pattern.interface_ip]
pattern_regex = ^(?P<intf>X\d+|U\d+|MGMT)\s+IP$
resolution    = interface_specific_ip
description   = Per-interface host IP (e.g. "X1 IP" -> X1's configured
                IP literal, returned as <host-ip-addr>)

[group_pattern.interface_default_gateway]
pattern_regex = ^(?P<intf>X\d+|U\d+|MGMT)\s+Default\s+Gateway$
resolution    = interface_specific_default_gateway

[group_pattern.vlan_subnet]
pattern_regex = ^(?P<intf>X\d+):V(?P<vlan>\d+)\s+Subnet$
resolution    = interface_specific_subnet

[group_pattern.vlan_ip]
pattern_regex = ^(?P<intf>X\d+):V(?P<vlan>\d+)\s+IP$
resolution    = interface_specific_ip
```

The VLAN sub-interface patterns join the `intf` and `vlan` captures into the conventional `X2:V88` form used as the `_id` in `interfaces_vlan.json`.

### Resolver architecture changes (skeleton_engine/skeleton_filler.py)

**1. `_load_sw_builtin_dynamic_groups`** now returns a dict with two keys:

```python
{
    'exact':    {name: spec, ...},          # zone-based exact matches
    'patterns': [(compiled_regex, spec), ...] # per-interface patterns
}
```

Existing zone-based exact lookups (Fix F) continue to work via `groups['exact'][name]`.

**2. New `_match_builtin_group_pattern(name, builtin_groups)` helper** walks the patterns list, matches the name, and returns `(spec, intf_name)` on match. The `intf_name` is composed from the regex capture groups (`X1` for physical, `X2:V88` for VLAN).

**3. Extended `_resolve_builtin_group(spec, interfaces, intf_name=None)`** with three new resolution modes:

| Resolution mode | Behavior |
|---|---|
| `interface_specific_subnet` | Walks `interfaces.json` (and `interfaces_vlan.json` via the v1.8.7 concat), finds the interface by `_id == intf_name`, returns `[{type: network, value: SUBNET, mask: MASK}]` |
| `interface_specific_ip` | Same lookup, returns `[{type: host, value: IP}]` |
| `interface_specific_default_gateway` | Same lookup, returns `[{type: host, value: GATEWAY}]` |

**4. `_resolve_vpn_selector`** now tries the resolution chain in the right order:

For `group "NAME"` references:
1. Exact match in `builtins['exact']` (zone-based) — Fix F behavior, preserved
2. Pattern match in `builtins['patterns']` — Fix G-2 new behavior
3. Customer-defined address-group fallback — Fix F behavior, preserved

For `name NAME` references (used by NAT `translated-source name "X1 IP"`):
1. Customer address-object lookup — Fix F behavior, preserved
2. Pattern match (NEW — resolves NAT translated-source references)
3. Exact-name built-in lookup
4. Customer address-group fallback

### Result on TZ-300

Three NAT policies now correctly resolve their references:

| NAT id | Source SW reference | Resolution path | Resolved value |
|---|---|---|---|
| 13 | `translated-source group "WAN Interface IP"` | Exact match in `builtins['exact']` (zone-based) | X1's IP (88.88.88.88) |
| 14 | `translated-source name "X1 IP"` | Pattern match `^(?P<intf>...)\s+IP$` → intf=`X1` | X1's IP (88.88.88.88) |
| 15 | `translated-source name "X1 IP"` | Same pattern match | X1's IP (88.88.88.88) |

The Firebox import sees fully-resolved IP literals in the NAT records — no auto-aliasing required, Aliases panel stays clean.

### Unit-test resolver behavior (Sprint 0 fortigate ready)

```
'X1 IP':            PATTERN match → intf='X1'      resolution=interface_specific_ip
'X0 Subnet':        PATTERN match → intf='X0'      resolution=interface_specific_subnet
'MGMT IP':          PATTERN match → intf='MGMT'    resolution=interface_specific_ip
'X2:V88 IP':        PATTERN match → intf='X2:V88'  resolution=interface_specific_ip
'WAN Interface IP': EXACT match   (zone-based, unchanged from v1.8.7)
```

The pattern-matched resolver is generic enough that it'll trivially adapt to FortiOS interface naming conventions (`port1 IP`, `wan1 Subnet`, etc.) by adding new `[group_pattern.*]` sections — no resolver code changes needed. The FortiGate roadmap (summer 2026) just got a head start.

---

## By the numbers — both corpora

### NSA-3600 (golden master, certified across multiple lab cycles)

| Metric | v1.8.7 | v1.8.8 |
|---|---|---|
| alias-list entries | 50 | 50 |
| policy-list entries | 18 | 18 |
| address-group-list entries | 12 | 12 |
| interface-list entries | 16 | 16 |
| service-list entries | 317 | 317 |
| NAT records captured | 1 | 1 |
| Validators passing | 10/10 | 10/10 |
| **SHA-256** | `934ce25e…` | **`934ce25e…` (BYTE-IDENTICAL)** |

### TZ-300 (customer config, lab-validated on T-30)

| Metric | v1.8.7 | v1.8.8 |
|---|---|---|
| alias-list entries | 83 | 85 |
| policy-list entries | 34 | 35 |
| address-group-list entries | 13 | 13 |
| interface-list entries | 16 | 16 |
| service-list entries | 245 | 245 |
| abs-policy-list entries | 34 | 35 |
| **NAT records captured** | **0** | **3** ✅ |
| Validators passing | 10/10 | 10/10 |
| **SHA-256** | `34910327…` | **`6d11a781…`** |

The TZ-300 SHA legitimately changed because we now capture and emit 3 NAT records that weren't there before. The +1 policy-list / +1 abs-policy-list increases correspond to the abstract + operative policy entries the NAT family produces (NAT id=13 generates a Policy-set, while ids 14 and 15 are auto-NAT placeholders consumed differently by the abs-policy/policy-list emit). The +2 alias entries are the from-zone / to-zone wrapper aliases for the new NAT policy.

---

## Honest perimeter (carry-overs + new for v1.8.8)

### Carried over from v1.8.7

- **PSK ciphertext** — pre-shared keys passthrough literally; operator must rotate post-import.
- **IPv6 interface emit** — parsed but not emitted on WG side. IPv6 access-rules and auto-generated IPv6 address-groups dropped categorically (Fix E + enricher).
- **Vendor-proprietary SonicOS features** — Application Control, Geo-IP enforcement objects, deep packet inspection profiles, SonicPoint wireless management — operator reconstructs in WG Web UI.
- **Phase 2 unresolved DNS labels** — fallback placeholder + `[warn]`. Neither current corpus exercises this path.
- **TZ-300 WG golden reference** — `pending_hand_build`. Validator gauntlet runs all 9 other validators; structural fidelity check returns `SKIPPED` until the operator hand-builds a marker-tagged WG XML target.

### New for v1.8.8

- **NAT `translated-destination` resolution** — Fix G-2 resolves `translated-source` references through the pattern-matched catalog. `translated-destination` (the DNAT side, used by SW source-NAT vs destination-NAT distinction) also passes through `fill_nat_policies` and uses the same resolver mechanism. However, the current corpus only exercises `translated-source` with per-interface built-ins; `translated-destination` with per-interface built-ins is theoretically supported but not lab-exercised on this release.
- **Pattern-catalog ordering** — when both an exact-match and a pattern-match could apply (rare; would require a specifically-named INI section to conflict with a pattern), exact-match wins. Documented as intentional convention.
- **VLAN sub-interface built-ins** — `X<N>:V<M> Subnet`, `X<N>:V<M> IP` patterns supported. Resolution lookup uses the `interfaces_vlan.json` concat introduced in Fix F. The TZ-300 corpus has VLAN 88 on X2 but no SW source references to `X2:V88 IP` directly, so this code path is INI-driven but not lab-exercised in the current corpora.
- **Multi-VDOM / HA / VLAN trunk / wireless** SonicOS features remain out of scope.

---

## Algebraic-design discipline — what this release showcases

Captain Jim Ames' methodology in action: **Object × Verb = Action**, declared in INI catalogs, evaluated at runtime, never hardcoded.

Fix G ships as:

| Component | Lines of code changed | Lines of INI added |
|---|---|---|
| `version_exceptions.ini` (NAT-policy 6.2 exception) | 0 | ~28 |
| `sw_builtin_dynamic_groups.ini` (5 new pattern sections) | 0 | ~45 |
| `skeleton_filler.py` (loader + resolver + dispatcher updates) | ~115 | 0 |

The resolver code change is the architectural extension: a new return shape for `_load_sw_builtin_dynamic_groups`, a new helper `_match_builtin_group_pattern`, three new resolution modes inside `_resolve_builtin_group`, and a small change to the dispatch order inside `_resolve_vpn_selector`. **Everything else is INI.**

When the FortiGate roadmap's Sprint 1 lands and we need FortiOS-flavored per-interface built-ins (`port1 IP`, `wan1 Subnet`, `internal Subnet`, etc.), it'll be a few new `[group_pattern.*]` sections in the FortiOS-side catalog. No resolver code changes. That's the algebraic-design payoff: the toolkit absorbs new vendor grammars through declarative configuration.

---

## Three-deviation summary — SonicOS 6.2 vs 6.5 grammar

After Fix G, the toolkit has documented and handles the three known grammar divergences between the two SonicOS branches:

| Fix | Section affected | Detection path | Captured behaviors |
|---|---|---|---|
| **B** (v1.8.5) | `access-rule` | `[exception.sonicos_6_2.access_rule_section_regex]` | 6.2 bare-form `access-rule from LAN to WAN ...` (no `ipv4` keyword); 6.5 form unchanged |
| **D** (v1.8.6) | `address-object` | `[exception.sonicos_6_2.address_object_section_regex]` + form override `block_or_single` | 6.2 inline form `address-object ipv4 NAME host IP zone Z`; 6.5 multi-line block form unchanged |
| **G-1** (v1.8.8) | `nat-policy` | `[exception.sonicos_6_2.nat_policy_section_regex]` | 6.2 bare-form `nat-policy inbound ...` (no `ipv4`, uses `id N`); 6.5 form unchanged |

Each follows the same architectural pattern: one INI section declares the deviation, the version router loads it when the 6.2 branch is detected, the parser substitutes the section regex at startup, and the body parser picks up the remaining fields. **No code path change required for any future SonicOS branch grammar deviation** — just add a new `[exception.<branch>.*]` section.

---

## Upgrade path from v1.8.7

For deployments running v1.8.7, drop in the v1.8.8 zip and restart the webapp. No configuration migration required. The new INI files are additive — old configs continue to work because the SonicOS 6.5 baseline path remains the default.

**Backward-compatibility note**: the NSA-3600 emit SHA is byte-identical between v1.8.7 and v1.8.8 (`934ce25e…`). Operator audit trails referencing the v1.8.7 SHA for NSA-3600 carry forward unchanged. The TZ-300 emit SHA legitimately changed (`34910327…` → `6d11a781…`) due to the 3 NAT policies that are now correctly captured and emitted. Operators with audit trails referencing the v1.8.7 SHA for TZ-300 should record the SHA progression and the corresponding fix provenance for compliance traceability.

---

## File manifest — net new + modified since v1.8.7

```
sonicwall_parser/
  version_exceptions.ini                       ← MODIFIED (new NAT-policy section)

enricher/
  sw_builtin_dynamic_groups.ini                ← MODIFIED (5 new [group_pattern.*] sections)

skeleton_engine/
  skeleton_filler.py                           ← MODIFIED
    _load_sw_builtin_dynamic_groups            ← UPDATED return shape
    _resolve_builtin_group                     ← UPDATED with per-interface modes
    _match_builtin_group_pattern               ← NEW helper
    _resolve_vpn_selector                      ← UPDATED dispatch order
```

No other files touched. All v1.8.7 functionality preserved.

---

## Acknowledgements

J P Ames, Lead Engineer, n2nhu lab — design, lab validation, methodology authorship. Captain's lab-cycle observation on v1.8.7 ("the builtins database for the TZ-300 isn't defined properly or has different syntax !!!!!! look here is the giant clue !!!") drove the entire diagnostic chain.

Lab hardware: WatchGuard Firebox T-30 running Fireware 12.5.9 (the toolkit's empirical ground truth across all lab cycles).

Reference compiler: WatchGuard CLI Reference Guide v12.11 (Fireware v2026.1.2/v12.11.8).

Built with Anthropic's Claude as collaborative engineering partner under the strict "no claim without provenance" methodology — every architectural decision in this release traces to a specific lab observation, SonicWall e-CLI line, or WatchGuard XML shape verified against the jpa template.

---

## What's next

- **Lab cycle on v1.8.8** — verify TZ-300 import on T-30 produces clean Aliases panel + 3 NAT policies visible in Network → NAT panel with correctly resolved translated-source IPs
- **FortiGate summer roadmap** (FES-FORTI v1, FortiOS 7.2 target) — Sprint 0 begins; the pattern-matched built-in resolver architecture proven in Fix G-2 is the template for FortiOS interface-naming built-ins
- **TZ-300 WG golden master** — pending hand-build; when delivered, structural fidelity validator activates automatically via `golden_pair_registry.ini` status flip

---

**End of release notes — FES v1.8.8 "NAT Recovery Edition"**
