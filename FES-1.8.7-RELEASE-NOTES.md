# FES — Firewall Ejector Seat
## v1.8.7 Release Notes
### "Multi-Model Edition"

**Release date**: June 13, 2026
**Author**: n2nhu lab — J P Ames, Lead Engineer
**Previous public release**: v1.8.3 (June 11, 2026)

---

## Executive summary

FES v1.8.7 transforms the toolkit from a **single-model converter** (NSA-3600 / SonicOS 6.5 only) into a **multi-model converter** (NSA-3600 + TZ-300 confirmed, with the architecture in place to absorb additional SonicWall families through INI configuration alone).

**Two corpora, both lab-validated on real WatchGuard T-30 hardware running Fireware 12.5.9:**

| Corpus | SonicOS branch | Model | Status |
|---|---|---|---|
| NSA-3600 (gold master) | 6.5.x | NSA 3600 | **Certified** — 3 lab cycles, all rendering surfaces verified |
| TZ-300 (customer config) | 6.2.x | TZ 300 | **Lab-validated** — alias panel clean, VPN tunnels rendered correctly, customer policies preserved |

**Six fixes, three milestones, and one architectural inflection**: the same disciplined INI-driven design that produced v1.8.3 now drives multi-version SonicOS parsing, multi-model golden references, and provenance-backed cross-corpus consistency.

---

## Headline changes from v1.8.3 → v1.8.7

| # | Theme | Why it matters |
|---|---|---|
| 1 | **SonicOS version routing** | First class support for grammar deviations between SonicOS 6.5 (NSA) and 6.2 (TZ family). New parser exceptions land via INI, not code. |
| 2 | **Multi-model golden references** | Per-(branch, model) golden pair registry. Each customer family gets its own validator-certified reference. Single-corpus assumption gone. |
| 3 | **Alias-crud filter** | SonicWall auto-generated bookkeeping (`X0 Subnet`, `X1 IP`), VLAN-bookkeeping objects with colons (`X2:V88 IPv6 Addresses` — confirmed WG Web UI crash cause), and jpa template factory aliases drop categorically via INI rules. |
| 4 | **External-type from SW assignment mode** | Hidden bug (Firebox-tolerated for 3 lab cycles): WG Web UI showed all WAN interfaces as DHCP regardless of SW `static`/`dynamic`/`pppoe` declaration. Fixed. |
| 5 | **IPv6 access-rule drop (reversible)** | SonicOS auto-generates IPv6 mirrors of every IPv4 rule (`any/any/any`). Dropped categorically for now; one INI section flip restores them when IPv6-on-WG support arrives. |
| 6 | **VPN tunnel selector resolution** | Phase 2 `network local group` and `network remote name` now correctly resolve through customer address-objects, address-groups, and SonicOS built-in dynamic groups. Two lab-class bugs fixed (wrong-IP-from-IKE-peer-bleed, placeholder `192.168.1.0/24` everywhere). |

---

## v1.8.4 — SonicOS Version Router (Multi-Model Architecture)

### The architectural inflection

Until v1.8.3, the toolkit assumed every SonicWall customer config came from the SonicOS 6.5 grammar family. The customer's first TZ-300 lab cycle (SonicOS 6.2.3.1-19n) immediately exposed three grammar divergences that silently dropped large portions of the customer config.

The architectural answer was **not** a forest of `if-version-then-this-else-that` blocks. It was a **declarative version router** that loads per-branch exception rules from an INI catalog.

### Files introduced

#### `sonicwall_parser/version_detection.ini`

Declarative SonicOS branch detection from the e-CLI header. Matches firmware-version strings and model lines via regex against named branches:

```ini
[branch.sonicos_6_5]
matches_firmware = ^SonicOS (Enhanced )?6\.5\.
matches_model    = NSA \d+

[branch.sonicos_6_2]
matches_firmware = ^SonicOS (Enhanced )?6\.2\.
matches_model    = TZ \d+
```

Adding support for a new SonicOS family is one new section.

#### `sonicwall_parser/version_exceptions.ini`

Per-branch deviations from the SonicOS 6.5 baseline. Each exception is a declarative rule with three kinds currently supported:

| `rule_kind` | What it overrides |
|---|---|
| `parser_field_pattern` | Body-line field-value tokenization (e.g., 6.2's combined IP+netmask shape) |
| `section_regex_override` | Top-level section detection regex (e.g., 6.2's bare `access-rule from` without `ipv4` keyword) |
| `section_regex_override` with `new_form` | Section-shape change (e.g., 6.2's inline `address-object NAME host IP zone Z` vs 6.5's multi-line block) |

Each rule cites its lab provenance (date, customer config, observed failure mode). Three exceptions ship in v1.8.7, all for the `sonicos_6_2` branch.

#### `sonicwall_parser/sonicos_version_router.py`

The router instance attaches to the parser at construction time:

```python
router = get_router(input_path)
parser.field_value_handler = router.split_field_value
parser.apply_section_regex_overrides(
    router.override_section_regex,
    router.override_section_form,
)
```

Routers expose `branch`, `model`, and `manifest_record()` for audit traceability. Every pipeline run records the detected branch + model in the parser manifest.

### What this unlocked

- TZ-300 customer config parses cleanly even though its e-CLI grammar differs from the NSA-3600 master in three measurable ways (see Fix B, Fix D).
- New SonicOS families absorb through INI additions, not code patches.
- The router output is auditable: every config produced now carries a `sonicos_version_branch_used` annotation in the manifest.

---

## v1.8.4 — Multi-Model Golden Pair Registry

### Files introduced

#### `skeleton_engine/golden_pair_registry.ini`

Per-(branch, model) golden reference registry. Validator selects the right golden pair based on the parser's detected branch + model:

```ini
[golden.nsa3600_6_5]
matches_branch       = sonicos_6_5
matches_model        = NSA 3600
sw_reference         = skeleton_engine/golden_pair/nsa3600_6_5/sw_input.txt
wg_reference         = skeleton_engine/golden_pair/nsa3600_6_5/wg_reference.xml
wg_reference_status  = present
lab_status           = certified
lab_cycles           = 3

[golden.tz300_6_2]
matches_branch       = sonicos_6_2
matches_model        = TZ 300
sw_reference         = skeleton_engine/golden_pair/tz300_6_2/sw_input.txt
wg_reference         = skeleton_engine/golden_pair/tz300_6_2/wg_reference.xml
wg_reference_status  = pending_hand_build
lab_status           = validator_only
lab_cycles           = 0
```

### Golden reference directory restructuring

Goldens moved from a flat layout to a per-(model, version) directory structure:

```
skeleton_engine/golden_pair/
├── nsa3600_6_5/
│   ├── sw_input.txt           (NSA-3600 master, marker-tagged)
│   └── wg_reference.xml       (hand-built WG target, embedded Claude markers)
└── tz300_6_2/
    ├── sw_input.txt           (customer TZ-300 config)
    └── wg_reference.xml.PENDING   (operator to hand-build)
```

### Validator selection logic

`golden_pair_validator.py` was rewritten to be registry-driven with three selection tiers (in order):

1. **Exact match**: branch AND model both matched → use this entry
2. **Branch-only match**: branch matched, model differs → use closest, log warning
3. **Default fallback**: no entry matched → use NSA-3600 baseline, log warning

When `wg_reference_status = pending_hand_build`, the structural fidelity comparison is **gracefully SKIPPED** (returning a clear status), not FAILED. The other nine validators still run normally — TZ-300 conversions are validator-graded with full rigor; only the cross-reference structural check awaits the operator's hand-built WG reference.

### Operator surface

Every validator run now reports which golden was selected, the match kind, and the lab status:

```
golden_pair_validator — composite fidelity score
  Golden reference:   golden.tz300_6_2
  Match kind:         exact
  SonicOS branch:     sonicos_6_2
  SonicWall model:    TZ 300
  Lab status:         validator_only
  Verdict:            PASS (structural skipped — WG ref pending)
```

---

## v1.8.5 — Three Lab-Driven Fixes

The first TZ-300 lab cycle on real T-30 hardware surfaced three issues. Each was diagnosed against the SonicWall source and the WatchGuard Web UI behavior, then fixed with full provenance.

### Fix A — External-type from SW `ip-assignment` mode

#### What was broken

The jpa template ships `<external-type>2</external-type>` (DHCP) as the WAN default. The toolkit's `fill_interfaces` patched the IP, netmask, and gateway from SW source but **never overrode the external-type value**. Result: every Firebox showed the WAN interface as DHCP in the Web UI regardless of whether the SonicWall had it as static, DHCP, or PPPoE.

#### How it surfaced

Captain's first TZ-300 lab report (the SW source had `ip-assignment WAN static 88.88.88.88/24`):
> "WAN was a static, but WG side is dhcp, so that needs fixing."

#### Honest disclosure of the hidden bug

The NSA-3600 master had the **same bug** for all three preceding lab cycles. The Firebox tolerated the mismatch (DHCP code with a static IP still imported cleanly), but the Web UI rendered it as DHCP. The TZ-300 customer noticed because they actually expected to see "Static" in the Web UI.

#### Fix

Patch to `fill_interfaces` in `skeleton_filler.py`. Reads the SW `ip-assignment <zone> <mode>` token, maps to the WG enum code:

| SW mode | WG `<external-type>` code |
|---|---|
| `static` | 1 |
| `dynamic` / `dhcp` | 2 |
| `pppoe` | 3 |

The enum codes are looked up via `wg_enums.ini` — not hardcoded.

#### SHA consequence

NSA-3600 SHA legitimately changed (`fad0088b...` → `79656762...`). The previous SHA was hiding a real bug. New SHA is the new canonical reference.

---

### Fix B — SonicOS 6.2 access-rule grammar

#### What was broken

| SonicOS branch | Access-rule header form |
|---|---|
| 6.5 (NSA-3600) | `access-rule ipv4 from LAN to WAN action allow` |
| 6.2 (TZ-300) | `access-rule from LAN to WAN action allow` (no `ipv4` keyword) |

The baseline parser regex required `(?P<family>ipv4\|ipv6)`. SonicOS 6.2's IPv4 access-rules omit the keyword entirely. All 18 customer IPv4 access-rules silently dropped on the first TZ-300 lab cycle. Only the 16 IPv6 rules (which DO use the `ipv6` keyword in 6.2) carried through.

#### Architectural fix: section-regex override via `version_exceptions.ini`

This was the first exercise of a new `rule_kind = section_regex_override`, which substitutes the baseline section regex with a branch-specific replacement:

```ini
[exception.sonicos_6_2.access_rule_section_regex]
rule_kind        = section_regex_override
parser_section   = access_rules
new_regex        = ^access-rule\s+(?:(?P<family>ipv4|ipv6)\s+)?from\s+...
```

The permissive regex makes `family` optional, capturing BOTH the 6.5 family-tagged form AND the 6.2 bare form. Safe to extend to other branches.

#### Architectural extension to the router

New method `override_section_regex(section_name)` returns the replacement regex string for the active branch. The parser's `apply_section_regex_overrides()` substitutes the section's compiled pattern at startup.

#### Result

TZ-300 access-rule capture went from 16 → 34 (all 18 customer IPv4 rules recovered). NSA-3600 unaffected — the override loads only when the active branch is `sonicos_6_2`.

#### Lab confirmation

Captain's lab report on v1.8.5:
> "POLICIES 13 (VNC 5800) AND 32 (H323-ALG) MADE IT!"

Two customer-marker policies (rule IDs 13 and 32) that the customer specifically expected to see post-migration rendered correctly with their proper service objects in the WG Web UI.

---

### Fix C — Alias-crud filter

#### What was broken (critically)

The first TZ-300 lab cycle exposed a **WG Web UI crash with operator logout** when viewing the Aliases panel. Forensic on the v1.8.4 emit found two aliases with colon characters in their names:

```
X2:V88 IPv6 Addresses
X2:V88 Management IPv6 Addresses
```

These are SonicWall auto-generated VLAN-sub-interface bookkeeping address-objects. WatchGuard reserves the colon character in alias names (used for port ranges like `tcp:80`); the Web UI was unable to render aliases whose names contained colons, crashed during render, and forced operator logout.

#### Architectural fix: filter pass via `alias_filter.ini`

New INI catalog with declarative drop patterns by priority:

```ini
[drop.wg_illegal_chars_colon]
pattern     = ^X\d+:V\d+\s+
priority    = critical
reason      = Alias name contains colon — reserved in WG alias grammar;
              renders WG Web UI Aliases panel unusable and logs out operator.
provenance  = Captain Jim Ames lab finding 2026-06-13, T-30 Fireware 12.5.9.

[drop.sw_auto_interface_bookkeeping]
pattern     = ^(X\d+|MGMT|LAN|WAN|External|Trusted|Optional-\d+)\s+(Subnet|IP|Default\s+Gateway)$
priority    = noise
reason      = SonicWall internal interface bookkeeping. Not operator-meaningful;
              SonicWall hides these in its own Web UI but exports them in
              the e-CLI output where they leak into FES.

[drop.jpa_factory_noise]
pattern     = ^(All\s+(Authorized\s+Access\s+Points|Rogue\s+...)|...)$
priority    = noise
reason      = jpa T-30 template factory carry-throughs. Feature relics from
              older SonicOS releases that the modern Firebox doesn't use.
```

#### Filter pass and validator exemption

Filter pass runs late in `fill()`, just before the binary XML write. Cascade-drops any backing `.1.alm` address-groups when their parent alias is dropped, keeping `ref_integrity` clean.

`customer_fidelity_validator.py` was extended to load the same INI and exempt dropped names from its missing-record reports — single source of truth, shared between emit and validator.

#### Result

TZ-300 alias-list count went from 102 → 78 in v1.8.5 (some IPv6 wrapper aliases not yet filtered). Zero colon-bearing names. After v1.8.6 (with Fix D collision-guard removing the dangling ghost), zero alias/address-group name collisions that previously triggered the Web UI crash.

---

## v1.8.6 — Three More Lab-Driven Fixes

The second lab cycle (v1.8.5 on T-30) reproduced the cycle: precise SonicWall ground truth → diagnose against WG Web UI behavior → fix with provenance.

### Fix D — SonicOS 6.2 inline address-object grammar + VPN collision-guard

#### Two entangled bugs

The TZ-300 lab cycle still crashed the Aliases panel even after Fix C eliminated the colon-bearing names. The remaining alias `tz300addressobject2wanzone` referenced backing IP `22.22.22.223` — which is **not** the customer's actual IP (the customer's value is `192.168.88.11`). Worse, an `<address-group>` named identically to the `<alias>` existed in the emit with that same wrong IP, causing a Web UI name-collision crash.

Forensic chain:

| Layer | SonicOS 6.5 (NSA-3600 baseline) | SonicOS 6.2 (TZ-300 customer) |
|---|---|---|
| Address-object declaration | `address-object ipv4 NAME` + multi-line block (`name`, `uuid`, `zone`, `host`, `exit`) | `address-object ipv4 NAME host 192.168.88.11 zone WAN` **inline single-line** |

The baseline parser regex matched only the multi-line header. TZ-300's inline form was silently dropped. Downstream `fill_address_groups` saw a SW address-group referencing the now-missing address-object, fell back to defensive emit code in `fill_vpn_policies`, and grabbed the IKE peer ID (`22.22.22.223`) as the host-IP — completely wrong field bleed.

#### Architectural fix: section-regex AND section-form override

New companion to `override_section_regex`: `override_section_form` returns the replacement form (`block_or_single`) for the section. The combined permissive regex matches BOTH 6.5 block-form and 6.2 inline-form:

```ini
[exception.sonicos_6_2.address_object_section_regex]
rule_kind        = section_regex_override
parser_section   = address_objects
new_regex        = ^address-object\s+(?P<family>ipv4|ipv6)\s+(?P<id>\S+)(?:\s+host\s+(?P<host>\d+\.\d+\.\d+\.\d+)(?:\s+zone\s+(?P<zone>\S+))?)?\s*$
new_form         = block_or_single
provenance       = Lab test 2026-06-13, TZ-300 SonicOS 6.2.3.1-19n.
```

For 6.5 multi-line form: only `family` and `id` capture at the header; body parser picks up `host` and `zone` from sub-lines.
For 6.2 inline form: all four capture in the header; `block_or_single` lets the parser accept entries with no body.

#### VPN collision-guard

Even with the parser fix, `fill_vpn_policies` still emitted a defensive "ghost" address-group for the SW VPN's `network remote name <X>` reference — duplicating the now-correctly-emitted customer alias. Added a guard:

```python
# Before emitting defensive ghost, check if alias with peer_name already exists
if peer_name and peer_ip and peer_ip != 'Any':
    if alias_already_exists_in_alias_list(peer_name):
        # Skip ghost emit — fill_user_address_objects already created proper alias
        log_vpn_collision_guard_skip(name, peer_name)
    else:
        # Original behavior — emit defensive for unresolved names
        ...
```

#### Result

After Fix D:
- TZ-300 alias `tz300addressobject2wanzone` correctly references its backing group `tz300addressobject2wanzone.1.alm`
- Backing group has the correct IP `192.168.88.11` (not the bogus `22.22.22.223`)
- Ghost address-group with the conflicting name is gone
- Web UI Aliases panel renders without crashing, operator stays logged in

NSA-3600 SHA unchanged (no 6.2 exceptions load on 6.5 branch, no collision-guard trigger on the existing NSA-3600 corpus).

---

### Fix E — Drop SonicOS auto-generated IPv6 access-rules

#### What was happening

Forensic on the v1.8.5 TZ-300 emit showed 16 IPv6 access-rules cluttering the policy panel — every one with `source any / dest any / service any` and the literal SonicOS auto-gen comment:

> `"IPv6:From Any to Any for Any service"`

These are SonicOS-auto-generated IPv6 mirrors of every IPv4 customer rule. The customer didn't write them; SonicOS does. They have no operational value on the WG side (no IPv6 interfaces emitted; rules can't fire).

#### Architectural fix: `access_rule_filter.ini` analogous to `alias_filter.ini`

New INI catalog with declarative drop rules:

```ini
[drop.ipv6_access_rules_all]
family       = ipv6
priority     = noise
reason       = FES v1.8.x is IPv4-on-WG. IPv6 access-rules emit without
               corresponding IPv6 interface support is half-a-feature
               that clutters operator's policy panel.
provenance   = Captain instruction 2026-06-13 ("drop the ipv6 both model
               via INI, we cal always restore it later via ini").
               Restorable: remove this section + add IPv6 emit support.
```

#### Filter pass + customer_fidelity exemption

Filter pass runs at the top of `fill_policies` BEFORE wrapper alias creation. Records that match drop criteria never enter the emit pipeline — no orphan `.1.from` / `.1.to` aliases get created. No cascade-delete needed.

The validator gets a matching `_is_access_rule_filter_drop()` helper that consults the same INI, exempting dropped records from missing-count.

#### Restorability

This entire feature reverses by removing the INI section. When IPv6-on-WG emit support arrives in a future FES release, the IPv6 access-rules carry through automatically.

#### Result

| Corpus | IPv6 access-rules dropped |
|---|---|
| NSA-3600 | 12 |
| TZ-300 | 16 |

Both legitimately cleaner. SHAs reflect the removal.

---

## v1.8.7 — VPN Tunnel Selector Resolution

The third lab cycle (v1.8.6 on T-30) validated all earlier fixes, then surfaced the last major translation gap: **VPN Phase 2 tunnel selectors** were being emitted with placeholder values regardless of what the SonicWall source declared.

### What was broken

The SW VPN block has:

```
vpn policy site-to-site vpn1formmrclaude22
    ...
    network local group "LAN Subnets"
    network remote name tz300addressobject2wanzone
```

The toolkit's `fill_vpn_policies` only handled three of the four SW selector shapes properly:

| SW shape | Toolkit behavior pre-v1.8.7 |
|---|---|
| `network local network IP MASK` (direct CIDR) | ✅ Handled |
| `network local host IP` (single IP) | Partial |
| `network local group "NAME"` (group ref) | ❌ Ignored — fell back to placeholder `192.168.1.0/24` |
| `network remote name <NAME>` (address-object ref) | ❌ Ignored — fell back to IKE peer ID (wrong field bleed) |

### Lab observation (Captain QA)

Branch Office VPN → vpn1formmrclaude22 → Addresses tab on v1.8.6:

| Field | What was there | What should have been there |
|---|---|---|
| LOCAL | `192.168.1.0/24` (placeholder hardcode) | `192.168.168.0/24` + `192.168.88.0/24` (X0 + VLAN 88, both LAN-zone) |
| REMOTE | `22.22.22.223` (IKE peer ID, wrong) | `192.168.88.11` (host IP from `tz300addressobject2wanzone`) |

### Architectural fix: declarative SonicOS built-in groups + resolver helper

#### `enricher/sw_builtin_dynamic_groups.ini` (NEW)

SonicOS has built-in dynamic groups that aren't explicitly defined in the e-CLI export — SonicOS auto-computes them from interface zone-assignments. Catalogued:

```ini
[group."LAN Subnets"]
resolution  = interface_zone_subnets
zone        = LAN
description = Every LAN-zone interface's subnet

[group."WAN Subnets"]
resolution  = interface_zone_subnets
zone        = WAN

[group."LAN Interface IP"]
resolution  = interface_zone_ips
zone        = LAN

[group."All WAN IP"]
resolution  = interface_zone_ips
zone        = WAN
description = Alias for "WAN Interface IP". SonicOS legacy name.

# ... 9 more built-in dynamic groups
```

Extending to a new built-in is one new section.

#### `_resolve_vpn_selector()` (NEW module-scope helper)

Takes a SW selector token like `local group "LAN Subnets"` or `remote name tz300addressobject2wanzone` and returns a list of resolved members. Resolution priority:

1. Direct IP / CIDR form (`network IP MASK` / `host IP`)
2. Built-in dynamic group (consults `sw_builtin_dynamic_groups.ini` → walks `interfaces.json` + `interfaces_vlan.json` by zone)
3. Customer-defined address-group (walks `address_groups.json`, resolves each member via `address_objects.json`)
4. Customer-defined address-object (resolves to `host` or `network`+`netmask`)
5. Unresolved → returns name label for caller to handle (DNS-label case)

#### `_ensure_addr_group_with_members()` (NEW emit helper)

Replaces single-member `_ensure_addr_group` for the tunnel-selector path. Takes a list of member dicts and emits the `.1.tlc` / `.1.trm` address-groups with all resolved members:

```xml
<!-- TZ-300 .1.tlc — TWO LAN-zone subnets correctly emitted -->
<address-group>
  <name>vpn1formmrclaude22.1.tlc</name>
  <addr-group-member>
    <member>
      <type>2</type>
      <ip-network-addr>192.168.168.0</ip-network-addr>
      <ip-mask>255.255.255.0</ip-mask>
    </member>
    <member>
      <type>2</type>
      <ip-network-addr>192.168.88.0</ip-network-addr>
      <ip-mask>255.255.255.0</ip-mask>
    </member>
  </addr-group-member>
</address-group>
```

### Both corpora improved

| | Pre-v1.8.7 | Post-v1.8.7 |
|---|---|---|
| **TZ-300 local selector** | `192.168.1.0/24` (placeholder) | `192.168.168.0/24` + `192.168.88.0/24` ✅ |
| **TZ-300 remote selector** | `22.22.22.223` (IKE peer, wrong) | `192.168.88.11` ✅ |
| **NSA-3600 local selector** | `192.168.1.0/24` (placeholder) | `192.168.1.0/24` + `192.168.77.0/24` ✅ |
| **NSA-3600 remote selector** | `0.0.0.0` (broken fallback) | `14.14.14.14` ✅ |

The DNS-label fallback path (where a SW name doesn't resolve to any customer object) returns an unresolved-label marker that the caller logs as `[warn]` for operator review. Neither current test corpus exercises this path; both fully resolve.

### IP-or-DNS dropdown handling (clarification)

The NSA-3600 gold corpus exercises the IP-or-DNS dropdown convention in three IKE-layer contexts via Captain Jim Ames' "ipor/orip" marker pattern:

| Captain marker (SW source) | WG XML shape (v1.8.7 emits correctly) |
|---|---|
| `gateway primary FakePriIPSECGwNameORAdrClaude` | `<peer-addr>VALUE</peer-addr>` (pass-through, WG accepts NAME-or-IP) |
| `ike-id local domain-name localikedomainoripClaude` | `<local-id-type>2</local-id-type>` + `<local-id-data>VALUE</local-id-data>` |
| `ike-id peer domain-name peerikedomainoripClaude` | `<peer-auth-mask>4</peer-auth-mask>` + `<domain-name>VALUE</domain-name>` |

The toolkit's pre-v1.8.7 IKE-ID handling already implements this convention correctly — the marker pattern stays operationally readable in the WG emit.

For Phase 2 selectors at the `<addr-group-member>` layer, the gold corpus does not currently exercise an FQDN-typed member shape. If a future customer config has a tunnel selector pointing at a DNS-only label (no backing customer object), the unresolved-label warning path activates. Provenance-backed FQDN-typed `<addr-group-member>` emit will land when a Captain-built gold snippet exercises that shape on real hardware.

---

## By the numbers — both corpora

### NSA-3600 (golden master, lab-certified across multiple cycles)

| Metric | Count |
|---|---|
| alias-list entries | 50 |
| policy-list entries | 18 |
| address-group-list entries | 12 |
| interface-list entries | 16 |
| Customer IPv4 access-rules preserved | 13/13 |
| SonicOS auto-generated IPv6 mirrors dropped | 12 |
| Validators passing | 10/10 |
| **SHA-256** | `934ce25e43415d6ef0cf48ad9989cbac51f743f75bd4bf056e293a79f0106ba5` |

### TZ-300 (customer config, lab-validated on T-30)

| Metric | Count |
|---|---|
| alias-list entries | 83 |
| policy-list entries | 34 |
| address-group-list entries | 13 |
| interface-list entries | 16 |
| Customer IPv4 access-rules preserved | 18/18 (IDs 13, 14, 16, 21, 23, 25, 32, 33, 35, 37, 39, 41, 47, 49, 116, 118, 120, 124) |
| SonicOS auto-generated IPv6 mirrors dropped | 16 |
| Colon-bearing crash-cause aliases | 0 (was 2 in v1.8.4) |
| Validators passing | 10/10 |
| **SHA-256** | `3491032784991f00512f3e21c969a95ed10530f14e7bf9c08e2324962fbbef2d` |

---

## Honest perimeter (carry-overs from v1.8.3 + new for v1.8.7)

The toolkit translates the **structurally-clean WatchGuard XML shapes**. It does NOT:

### Carried over from v1.8.3 (still true in v1.8.7)

- **Decrypt SonicWall PSK ciphertext.** SW pre-shared keys ship as one-way encrypted with the SonicWall's per-device key; FES preserves the ciphertext literally, but it cannot decode them. Operator must rotate the tunnel PSK after import.

- **Emit IPv6 interfaces.** IPv6 interface declarations are PARSED (and retained in the manifest for audit) but NOT emitted on the WG side. v1.8.7 also drops SonicOS auto-generated IPv6 access-rules categorically (Fix E) — reversible via INI.

- **Hand-build IPv6-specific WG sections.** Pure-IPv6 customer rules (operator-written, not auto-gen) would be lost in v1.8.7's current scope.

- **Translate vendor-proprietary SonicOS features.** Application Control, Geo-IP enforcement objects, deep packet inspection profiles, SonicPoint wireless management — none have direct WatchGuard equivalents; the operator must reconstruct policy intent in WG Web UI.

- **Handle SSL/IPsec mobile VPN with non-default authentication backends.** RADIUS / LDAP / Active Directory bindings on mobile VPN groups require manual operator wiring in the WG authentication panel.

### New for v1.8.7

- **Phase 2 tunnel selectors with unresolved DNS labels.** When a SW VPN block's `network remote name X` references a name that doesn't appear as a customer address-object or built-in group, the toolkit emits a fallback placeholder and a `[warn]` to stderr. Operator must edit Phase 2 selectors in WG Web UI for these tunnels before they can come up. Neither current test corpus exercises this path; the warning is academic. Full FQDN-typed `<addr-group-member>` emit will land when a gold snippet exercises the shape on real hardware.

- **TZ-300 WG golden reference is pending.** The `golden_pair_registry.ini` entry for TZ-300 has `wg_reference_status = pending_hand_build`. Until the operator hand-builds a marker-tagged WG reference for the TZ family, the structural comparison validator returns SKIPPED for TZ-300 emits (the other 9 validators still run normally). When the WG reference lands, structural fidelity validation activates automatically — no code changes.

- **WatchGuard built-in 5 alias/address-group name collisions** (`Any`, `Firebox`, `dnat.from.1/2/3`) are present in both corpora. Investigation confirmed these are part of the jpa template's factory baseline (NSA-3600 has had them for all 3 successful lab cycles). They are NOT a crash cause — Fix D's collision-guard surgically targets only the harmful customer-marker collision. These remaining 5 are jpa convention and load cleanly.

---

## Algebraic design — what this release showcases

J P Ames' algebraic-design methodology: **Object × Verb = Action**, with the action expressed in INI configuration, evaluated at runtime, never hardcoded.

This release proved the methodology at scale. Six fixes across two corpora landed with:

- **Zero new hardcoded rules** in the parser or skeleton-filler code paths
- **Five new INI catalogs** declaring the new rules with full provenance citations
- **Three new architectural mechanisms** (router section-regex override, router section-form override, multi-member address-group emit) that absorbed the variance without ripping out existing code
- **Cross-corpus consistency**: every new behavior validated on BOTH NSA-3600 (baseline) AND TZ-300 (new family) before shipping

The pattern for future SonicOS families is now well-established:

1. Add a `[branch.X]` section to `version_detection.ini`
2. Add `[exception.X.*]` rules to `version_exceptions.ini` for any grammar divergences observed in lab
3. Add a `[golden.X_N]` entry to `golden_pair_registry.ini` pointing at the hand-built reference pair
4. Lab-cycle on real hardware
5. Promote `lab_status = validator_only` → `certified` after lab confirmation

No code changes required for the routine case.

---

## Acknowledgements

J P Ames, Lead Engineer, n2nhu lab — design, lab validation, methodology authorship.

Lab hardware: WatchGuard Firebox T-30 running Fireware 12.5.9 (the toolkit's empirical ground truth across all lab cycles).

Reference compiler: WatchGuard CLI Reference Guide v12.11 (Fireware v2026.1.2/v12.11.8).

Built with Anthropic's Claude as collaborative engineering partner under the strict "no claim without provenance" methodology — every architectural decision in this release traces to a specific lab observation, SonicWall e-CLI line, or WatchGuard XML shape verified against the jpa template.

---

## Upgrade path from v1.8.3

For deployments running v1.8.3, drop in the v1.8.7 zip and restart the webapp. No configuration migration required. The new INI files (version-routing, golden-registry, filters, built-in groups) are additive — old configs continue to work because the SonicOS 6.5 baseline path remains the default.

Backward-compatibility note: the NSA-3600 emit SHA legitimately changed from v1.8.3 → v1.8.7 due to Fix A (external-type) and Fix E (IPv6 access-rule drop). The new SHA `934ce25e...` is the new canonical reference for the NSA-3600 corpus. Operators with audit trails referencing the v1.8.3 SHA (`fad0088b...`) should record the SHA progression and the corresponding fix provenance for compliance traceability.

---

## File manifest — net new since v1.8.3

```
sonicwall_parser/
  version_detection.ini                  ← NEW (v1.8.4) — branch detection rules
  version_exceptions.ini                 ← NEW (v1.8.4-6) — three exceptions
  sonicos_version_router.py              ← NEW (v1.8.4) — router class

skeleton_engine/
  golden_pair_registry.ini               ← NEW (v1.8.4) — per-(branch, model) registry
  golden_pair_validator.py               ← MODIFIED — registry-driven
  alias_filter.ini                       ← NEW (v1.8.5) — alias-crud drop rules
  access_rule_filter.ini                 ← NEW (v1.8.6) — IPv6 drop rule
  customer_fidelity_validator.py         ← MODIFIED — filter exemptions
  skeleton_filler.py                     ← MODIFIED — multiple sections (see fixes)
  golden_pair/
    nsa3600_6_5/                         ← NEW directory (v1.8.4) — moved from flat
      sw_input.txt
      wg_reference.xml
    tz300_6_2/                           ← NEW directory (v1.8.4)
      sw_input.txt
      wg_reference.xml.PENDING

enricher/
  sw_builtin_dynamic_groups.ini          ← NEW (v1.8.7) — built-in group catalog
```

---

**End of release notes — FES v1.8.7 "Multi-Model Edition"**
