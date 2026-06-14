# FES FortiGate Migration Toolkit
## Roadmap, Sprint Plan & Engineering Work Plan
### v2 — "7.2 First, INI-Branched for 7.4 / 7.6"

**Working title**: FES-FORTI (Firewall Ejector Seat — FortiGate edition)
**Target completion**: End of summer 2026 — FortiOS 7.2 lab-validated and shipping
**Fall 2026 extension**: INI-branched 7.4 and 7.6 support
**Author**: n2nhu lab — J P Ames, Lead Engineer
**Status**: Planning — pre-Sprint 0
**Lab hardware**: FortiGate 60E (racked) + WatchGuard T-30 Fireware 12.5.9 (existing)

---

## 0. The strategic decision — why 7.2 first, then INI-branch

The v1 roadmap considered three options for FortiOS 7.x version coverage:

| Option | Description | Outcome |
|---|---|---|
| A | All three versions (7.2/7.4/7.6) in parallel during summer | High risk; full-throttle pace; doesn't feel like vacation |
| B | **7.2 fully validated in summer; 7.4 + 7.6 as fall INI-branched extensions** | **Selected** |
| C | All three scaffolded in summer, only 7.2 lab-validated | Violates "no claim without provenance" discipline |

**Option B is the way.** Here's why the choice is architecturally sound, not just operationally convenient:

1. **The SonicWall toolkit already proved INI-branching works.** SonicOS 6.5 was the trained-on baseline (NSA-3600 corpus, certified across 3 lab cycles). SonicOS 6.2 (TZ-300) was added as an INI-branched extension via `version_exceptions.ini`. The TZ-300 ship was incremental work, not new construction. FortiOS 7.4 and 7.6 follow the same pattern relative to 7.2.

2. **FortiOS 7.x grammar evolution is incremental.** Reading the actual FortiOS 7.2 → 7.4 → 7.6 "Changes in CLI" sections from Fortinet's release notes, the changes are predominantly field additions, occasional renames, rare deprecations. The fundamental `config / edit / set / next / end` shape is unchanged across all three. This is **structurally different** from SonicOS 6.2 vs 6.5 (which had three real grammar earthquakes). FortiOS 7.x is closer to SonicOS 6.5.x point-release evolution — extensions within a stable grammar.

3. **Lab discipline is the moat — and it doesn't parallelize.** One firewall (60E), one operator, one lab cycle at a time. Three FortiOS versions = three sets of golden-master builds + three sets of lab cycle iterations. Forcing all three into the summer compresses each cycle's depth and increases the risk of shipping unverified work.

4. **A shippable 7.2 product in summer is a milestone with teeth.** Acquirer conversations, MSP demos, methodology paper updates, LinkedIn launch — all of these benefit from a complete, lab-validated 7.2 product available before fall extensions begin.

5. **Architectural rigor compounds.** Building 7.2 carefully — version_detection.ini, version_exceptions.ini scaffolding, field_map inheritance — means 7.4 and 7.6 land as a few weeks of well-defined INI work each, not full re-engineering.

---

## 1. Summer 2026 scope: FortiOS 7.2 only

### What ships at end of summer (Definition of Done — top-level)

- [ ] Working FortiGate→WG translator validated against FortiOS 7.2 corpus
- [ ] 10/10 validator gauntlet green on FortiOS 7.2 golden master + at least one customer-realistic 7.2 corpus
- [ ] T-30 (Fireware 12.5.9) lab cycle: WG XML imports cleanly, Web UI renders all panels without crash, customer entries visible, VPN tunnels show correct selectors, login persists
- [ ] Multi-vendor portal at `bash run-dev.sh` serves both SonicWall and FortiGate-7.2 uploads
- [ ] `FES-FORTI-RELEASE-NOTES.md` complete
- [ ] Golden Rosetta Stone methodology paper updated with multi-vendor case study
- [ ] LinkedIn launch article drafted
- [ ] All SonicWall regression tests still pass (NSA-3600 + TZ-300 SHAs match v1.8.7 baseline)
- [ ] **Architecture scaffolded for 7.4 and 7.6 INI extensions** — `fortios_version_detection.ini` and `fortios_version_exceptions.ini` files exist and contain comment placeholders for 7.4 and 7.6 sections

### What does NOT ship in summer

- FortiOS 7.4 field-map deltas (deferred to Sprint 9)
- FortiOS 7.6 field-map deltas (deferred to Sprint 10)
- VDOM support (entirely out of summer + autumn scope)
- HA cluster awareness (out of scope)
- UTM profile translation (out of scope — `[warn]` carry-throughs only)
- Dynamic routing translation (out of scope — static routes only)
- IPv6 emit on WG side (out of scope — same scope decision as SonicWall toolkit)

---

## 2. Fall 2026 scope: INI-branched 7.4 + 7.6

### Sprint 9 — FortiOS 7.4 INI branch (estimated 3-5 weeks elapsed; ~80-130 focused hours)

**Architecture**: Add a `[branch.fortios_7_4]` section to `fortios_version_detection.ini` and corresponding `[exception.fortios_7_4.*]` entries to `fortios_version_exceptions.ini` for the field-level deltas between 7.2 and 7.4.

**Deliverables**:
1. `fortios_version_detection.ini` updated with 7.4 branch matcher (regex against the FortiOS config-version header)
2. `fortios_version_exceptions.ini` entries documenting each 7.4 field-level delta (additions, renames, deprecations)
3. Hand-built FortiOS 7.4 golden master on the 60E (firmware-swap the 60E to 7.4, rebuild the marker-tagged golden corpus, re-export, build matching WG XML target)
4. Registry entry `[golden.fortigate60e_7_4]` in `golden_pair_registry.ini` with `lab_status = certified` after lab cycle confirms
5. Lab cycle iterations on T-30 with the 7.4 emit
6. Release notes addendum documenting the 7.4 extension

### Sprint 10 — FortiOS 7.6 INI branch (estimated 3-5 weeks elapsed; ~80-130 focused hours)

Same pattern as Sprint 9, applied to 7.6. By Sprint 10 the pattern is well-rehearsed; minor surprises only.

### Sprint 11 (optional, depending on customer pull) — autumn polish

Documentation updates, multi-version selector UX in the webapp, golden_pair_registry surface improvements, multi-version regression test automation.

---

## 3. Strategic context (unchanged from v1)

FES v1.8.7 (the SonicWall→WG toolkit) shipped with two lab-validated corpora and an architecture built on the **algebraic-design methodology**: every translation rule expressed as `Object × Verb = Action`, declared in INI catalogs, evaluated at runtime, never hardcoded.

The downstream half of the toolkit (enricher → skeleton_filler → validator gauntlet → WG XML emit) is **vendor-agnostic by construction**. It consumes a JSON intermediate representation of customer records and produces WatchGuard XML.

This roadmap extends FES to a second upstream: **FortiOS 7.2 configuration files** → same IR → same downstream → same WG XML output. Future extensions to 7.4 and 7.6 ride the same INI-branched architecture pattern proven by the SonicOS 6.5 → 6.2 work.

---

## 4. Architecture — leveraging what's already there

```
                          ┌─────────────────────────────────────────┐
SonicOS e-CLI ────────────▶│ sonicwall_parser/                       │─┐
                          │   (version-router: 6.5 baseline + 6.2   │ │
                          │    exceptions via INI)                  │ │
                          └─────────────────────────────────────────┘ │
                                                                       │
                          ┌─────────────────────────────────────────┐ │   ┌──────────────┐   ┌──────────────┐
FortiOS 7.2 config ──────▶│ fortigate_parser/    (NEW)              │─┴──▶│ JSON IR      │──▶│ enricher/    │──┐
                          │   - uniform-grammar tokenizer           │     │ (unchanged)  │   │ (unchanged)  │  │
FortiOS 7.4 (fall) ──────▶│   - fortios_version_router (analog of   │     └──────────────┘   └──────────────┘  │
FortiOS 7.6 (fall) ──────▶│     SonicOS version_router)             │                                          │
                          │   - 7.2 baseline + 7.4/7.6 exception    │                                          │
                          │     INIs (fall extension)                │                                          │
                          └─────────────────────────────────────────┘                                          ▼
                                                                                                  ┌────────────────────┐
                                                                                                  │ skeleton_engine/   │
                                                                                                  │ (unchanged)        │
                                                                                                  └────────────────────┘
                                                                                                            │
                                                                                                            ▼
                                                                                                       WG XML
```

**Why the FortiOS version_router exists at all (even in summer)**: The architecture is in place from Sprint 1 onward, even if only the 7.2 branch is populated. This is the difference between "code that's INI-branchable" and "code that retrofits later." Future you in fall thanks current you in summer.

**Same INI-branching architecture as SonicOS** — pattern proven and documented:

| Component | SonicOS analog | FortiOS analog (NEW for summer 2026) |
|---|---|---|
| Version detection | `version_detection.ini` | `fortios_version_detection.ini` |
| Per-branch exceptions | `version_exceptions.ini` | `fortios_version_exceptions.ini` |
| Router module | `sonicos_version_router.py` | `fortios_version_router.py` |
| Parser-side override hooks | `apply_section_regex_overrides()` | Same method on `fortigate_parser.py` |

---

## 5. Universal Intermediate Representation (IR) — formalization (unchanged from v1)

The IR is currently implicit: it's the JSON files in `sonicwall_parser/output/` and `enricher/output/`. Sprint 0 formalizes it.

### IR record types (already produced by SonicWall parser, will be produced by FortiGate-7.2 parser)

| File | Records | Key fields | FortiOS 7.2 source section |
|---|---|---|---|
| `interfaces.json` | Physical + VLAN interface records | `_id`, `ip-assignment` (zone, ip, netmask), `description` | `config system interface` |
| `address_objects.json` | Customer address objects | `_id`, `family`, `host` OR `network`+`netmask`, `zone` | `config firewall address` |
| `address_groups.json` | Customer address groups | `_id`, `address-object` (list of member refs) | `config firewall addrgrp` |
| `service_objects.json` | Customer service objects | `_id`, protocol (tcp/udp/icmp), port range or icmp-type | `config firewall service custom` |
| `service_groups.json` | Customer service groups | `_id`, member refs | `config firewall service group` |
| `schedules.json` | Time-of-day / day-of-week schedules | `_id`, days, start/end times | `config firewall schedule recurring` |
| `access_rules.json` | Firewall policies | `_id`, from_zone, to_zone, action, source, destination, service | `config firewall policy` |
| `vpn_policies.json` | VPN gateway + tunnel config | `_id`, peer-addr, ike-id, network local/remote | `config vpn ipsec phase1-interface` + `phase2-interface` (joined by `phase1name`) |
| `nat_policies.json` | DNAT / SNAT entries | `_id`, original/translated source/dest | `config firewall vip` (DNAT) + policies with `nat enable` (SNAT) |
| `routes.json` | Static routes | `_id`, dst, gateway, device | `config router static` |

### Sprint 0 deliverable

`docs/IR-SCHEMA.md` — formal schema definition for each IR record type, with field semantics documented. Used as the contract that BOTH parsers (SonicWall + FortiGate) must satisfy.

---

## 6. Summer 2026 sprint breakdown (FortiOS 7.2 only)

**Estimating discipline**: hours are wall-clock focused engineering, not "ideal hours." Calendar-days assume 6-8 focused hours per working day. Each sprint has an explicit Definition of Done and a Go/No-Go gate before the next sprint starts.

### Sprint 0 — Foundation, IR formalization, and 60E baseline capture (5-7 days)

**Goal**: De-risk the architecture before writing any FortiGate parser code. Confirm the IR is rigorously specified. Capture 60E factory-default baseline + target 7.2 firmware export.

**60E-specific deliverables (NEW for v2 roadmap)**:
1. `docs/LAB-HARDWARE.md` — 60E specs, serial, factory state, firmware build, console access notes (analog to existing T-30 lab documentation)
2. `fortigate_parser/input/60e_factory_default_7_2.txt` — captured `show full-configuration` from the 60E with FortiOS 7.2.x firmware loaded and zero admin config (factory baseline)
3. **Critical Sprint 0 decision**: which exact FortiOS 7.2 patch level (7.2.0? 7.2.7? 7.2.13?) is the summer target? Document choice + rationale in `docs/LAB-HARDWARE.md`. **Recommendation**: pick the latest 7.2.x patch as of Sprint 0 start date — that's what customers running 7.2 are most likely on by summer's end.

**Architecture deliverables**:
1. `docs/IR-SCHEMA.md` — formal IR schema with field-level semantics for each record type
2. `fortigate_parser/` directory skeleton — empty but with the same internal layout as `sonicwall_parser/`
3. `fortigate_parser/parser_config.ini` — top-level section catalog (the 14 migration-relevant FortiOS sections)
4. **Scaffolding for future versions**: `fortios_version_detection.ini` exists with `[branch.fortios_7_2]` populated AND placeholder comment blocks for `[branch.fortios_7_4]` and `[branch.fortios_7_6]` (clearly marked "to be populated in autumn Sprint 9/10")

**Validation test**: run the existing SonicWall parser on the NSA-3600 master, dump its IR. Hand-author equivalent JSON for a tiny 60E export. Feed both through the enricher. Confirm: enricher produces sensible output without errors.

**Definition of Done**: enricher consumes hand-authored FortiGate IR without crashing or producing schema-shape complaints from the validator gauntlet. 60E baseline captured, target 7.2 patch level documented.

**Go/No-Go gate**: if the enricher does NOT cleanly consume hand-authored FortiGate IR, STOP. Either the IR isn't actually vendor-agnostic and needs refactoring, or the IR schema doc has gaps. Fix that before Sprint 1.

---

### Sprint 1 — Tokenizer, version-router, and easy-section parsers (6-8 days)

**Goal**: Working FortiOS 7.2 tokenizer + version router + parsers for the four lowest-complexity sections.

**Architectural notes**:
- The version router scaffolding is built NOW even though it has only one branch (`fortios_7_2`). This avoids retrofit pain in Sprint 9/10.
- The router structure mirrors `sonicos_version_router.py` exactly — same `override_section_regex()`, `override_section_form()`, `split_field_value()` methods. Same patterns, same idioms.

**Sections covered (parsers)**:
- `config system interface` (interface records)
- `config firewall address` (address objects)
- `config firewall addrgrp` (address groups)
- `config firewall schedule recurring` (schedules)

**Deliverables**:
1. `fortigate_parser/fortigate_parser.py` — tokenizer + dispatcher framework + version router hookup
2. `fortigate_parser/fortios_version_router.py` — analog of `sonicos_version_router.py` with one branch (`fortios_7_2`) populated
3. Field-mapping tables (per-section INIs):
   - `field_mappings/interface_field_map_7_2.ini`
   - `field_mappings/address_field_map_7_2.ini`
   - `field_mappings/addrgrp_field_map_7_2.ini`
   - `field_mappings/schedule_field_map_7_2.ini`
4. Unit test: run parser on a hand-built minimal 60E 7.2 export, confirm IR output for these 4 sections matches expected shape.

**Naming convention rationale**: field-map INI filenames carry the version suffix (`_7_2.ini`) from day one. This makes the autumn extension trivial — Sprint 9 adds `_7_4.ini` files alongside the `_7_2.ini` files, the version router picks the right one per the detected branch.

**Definition of Done**: parser produces `interfaces.json`, `address_objects.json`, `address_groups.json`, `schedules.json` from a 7.2 60E export, and the enricher consumes them without complaint.

---

### Sprint 2 — Services + routing (4-6 days)

**Goal**: Service object/group parsing + static route parsing.

**Sections covered**:
- `config firewall service custom`
- `config firewall service group`
- `config router static`

**Deliverables**:
1. `field_mappings/service_field_map_7_2.ini`
2. `field_mappings/service_group_field_map_7_2.ini`
3. `field_mappings/static_route_field_map_7_2.ini`
4. **Service factory-default filter**: same architectural pattern as `alias_filter.ini` — INI catalog of FortiOS factory-default service names that should be dropped (analog to jpa template factory carry-throughs). Filter pass exempted from customer_fidelity reporting.

**Definition of Done**: all four IR files populated, validator gauntlet runs the IR through enricher + skeleton_filler producing valid WG XML.

---

### Sprint 3 — Firewall policy parsing (the deepest semantic territory) (8-12 days)

**Goal**: Translate FortiOS 7.2 firewall policies, including UTM-profile attachments (carry as `[warn]`), NAT-on-policy semantics, and reference resolution.

**Sections covered**:
- `config firewall policy`
- `config firewall vip` (DNAT)

**Why this sprint is the longest**: FortiOS firewall policies are semantically the richest section. Unlike SonicOS access-rules, FortiOS policies bundle UTM profiles, schedule references, application-control settings, NAT semantics, traffic shaping — all in one `edit N` block.

**Deliverables**:
1. `field_mappings/policy_field_map_7_2.ini` with three categorized field groups:
   - **Direct translate**: `srcintf`, `dstintf`, `srcaddr`, `dstaddr`, `service`, `action`, `schedule`, `status`, `name`
   - **WG-equivalent transform**: `nat enable` becomes a separate `<nat-policy>` SNAT entry
   - **No WG analog (`[warn]` carry-through)**: `utm-status`, `av-profile`, `webfilter-profile`, `application-list`, `ips-sensor`, `dlp-sensor`
2. `field_mappings/vip_field_map_7_2.ini` — DNAT entries
3. `policy_filter.ini` — drop rules for FortiOS auto-generated policies (analog to `access_rule_filter.ini`)
4. Reference-resolution test: hand-pick 5 policies, confirm IR correctly resolves all references end-to-end.

**Definition of Done**: WG emit contains FortiOS policies with correct source/dest/service zone wiring; UTM-attached policies show `[warn]` in the pipeline log; SNAT and DNAT correctly emit `<nat-policy>` entries.

**Risks**:
- UTM profile carry-through scope — customers may need WAF/web-filter/AV; operator reconstructs in WG Web UI.
- `set name ''` vs `set name 'someName'` distinction — translation handles both.

---

### Sprint 4 — VPN (Phase 1 + Phase 2) (6-8 days)

**Goal**: Translate IPSec VPN configurations. Fix F's selector resolution pattern from SonicWall pays dividends here.

**Sections covered**:
- `config vpn ipsec phase1-interface`
- `config vpn ipsec phase2-interface`

**FortiOS specifics**:
- Phase 1 and Phase 2 are SEPARATE sections, joined by `phase2-interface.phase1name == phase1-interface.<id>`
- FortiOS encodes selectors as `phase2-interface.src-subnet` / `dst-subnet` (direct CIDR) and `src-addr-type subnet|name|range`
- PSK ciphertext (`set psksecret ENC ...`) is opaque carry-through, operator rotates post-import

**Deliverables**:
1. `field_mappings/vpn_phase1_field_map_7_2.ini`
2. `field_mappings/vpn_phase2_field_map_7_2.ini`
3. Joining pass: walks both sections, joins by `phase1name`, emits unified `vpn_policies.json` records
4. Selector resolution reuses the existing `_resolve_vpn_selector()` helper unchanged

**Definition of Done**: VPN tunnel from the 60E 7.2 corpus renders correctly in WG Web UI Branch Office VPN screens — local/remote selectors correct, peer-addr correct, IKE proposals translated.

**Risks**:
- IKE proposal translation: FortiOS `aes256-sha1` (single token) → WG separate encryption/auth fields. Mapping in `vpn_phase1_field_map_7_2.ini`.
- DH group encoding: FortiOS numeric → WG string enum via `wg_enums.ini`.
- `auto-discovery`, `auto-negotiate`, `exchange-interface-ip` — FortiOS advanced features, `[warn]` carry-through.

---

### Sprint 5 — Golden master + WG reference build on 60E (5-7 days)

**Goal**: Hand-build the marker-tagged FortiOS 7.2 golden master ON the 60E (not in a text editor) and corresponding WG XML target, following the Golden Rosetta Stone methodology.

**60E-leveraged process**:
1. Flash 60E to the target 7.2 patch level (decided in Sprint 0)
2. Configure the 60E via Web UI to match the semantic shape of the SonicWall NSA-3600 master:
   - Same zone topology (LAN, WAN, DMZ, VLAN sub-interface analog)
   - Marker-tagged customer-defined object names (`forClaude` suffix convention)
   - `NameOrAdr` / `OrIPClaude` markers for IP-or-DNS dropdown fields (matching the Rosetta Stone methodology proven in SonicWall corpora)
   - One site-to-site VPN with the full IKE/Phase 2 selector machinery
3. `show full-configuration` on the 60E — that exported file IS the golden master `fg_input.txt`
4. Hand-build the WG XML target matching what the toolkit should produce
5. Embed Claude markers on both sides (per the proven Rosetta Stone methodology)

**Deliverables**:
1. `skeleton_engine/golden_pair/fortigate60e_7_2/fg_input.txt` — captured 60E export
2. `skeleton_engine/golden_pair/fortigate60e_7_2/wg_reference.xml` — hand-built WG XML target with embedded markers
3. New entry in `skeleton_engine/golden_pair_registry.ini`:
   ```ini
   [golden.fortigate60e_7_2]
   matches_branch       = fortios_7_2
   matches_model        = FortiGate-60E
   sw_reference         = skeleton_engine/golden_pair/fortigate60e_7_2/fg_input.txt
   wg_reference         = skeleton_engine/golden_pair/fortigate60e_7_2/wg_reference.xml
   wg_reference_status  = present
   lab_status           = validator_only
   ```

**Definition of Done**: validator gauntlet runs 10/10 green against the hand-built FortiOS 7.2 golden, AND SonicWall corpora (NSA-3600 + TZ-300) still pass with identical SHAs.

---

### Sprint 6 — Lab cycle iterations (8-14 days, depending on findings)

**Goal**: Convert the 60E 7.2 golden master through FES-FORTI, load the resulting WG XML on the T-30, observe, diagnose, fix, repeat.

**The dual-firewall lab discipline** (new for FortiGate work):

| Lab artifact | Role |
|---|---|
| **60E `show full-configuration` (FortiOS 7.2)** | Toolkit input — your source of truth for what FortiOS actually emits |
| **WG XML produced by toolkit** | Toolkit output |
| **T-30 Fireware 12.5.9 import** | WG-side empirical truth — does this load cleanly? |
| **60E Web UI** | FortiOS-side reading reference — what did FortiOS intend the config to mean? |
| **WG Web UI** | WG-side reading reference — what did the toolkit translate it to? |

Cross-comparison between 60E Web UI and WG Web UI is the visual provenance check the SonicWall work didn't have access to (the SonicWall hardware wasn't in the lab; SonicOS source was text-only).

**Deliverables**:
1. First WG XML output from the 60E 7.2 golden — handed off for T-30 import
2. Lab finding reports following the established format (what broke / what worked / what's questionable)
3. Iterative fixes following the same INI-driven discipline as v1.8.4-v1.8.7

**Definition of Done**:
- T-30 imports cleanly
- Web UI Interfaces, Aliases, Policies, BOVPN panels all render without crash
- Customer-defined entries (from the marker-tagged 60E config) visible in WG Web UI
- VPN tunnels show correct Phase 1 + Phase 2 selectors (cross-verified against 60E Web UI)
- Operator login persists across panel navigation

**Expected lab cycles**: 3-5 iterations, ~2 days each.

---

### Sprint 7 — Web app integration + product surface (4-6 days)

**Goal**: Wire the FortiGate-7.2 parser into `fes_webapp/` so the existing portal serves both vendors.

**Deliverables**:
1. Updated `app/main.py`: vendor picker in the upload form ("SonicWall" / "FortiGate" radio buttons); routes upload to the right parser
2. Auto-detection helper: sniff the first few lines of the uploaded file — `#config-version=FG` → FortiGate; SonicWall e-CLI header pattern → SonicWall; show detection + let user confirm
3. **Version-aware detection**: when FortiGate is detected, the auto-detection ALSO reports the FortiOS version (parsed from `#config-version=FG60E-7.2.X-FW-build...`). If 7.4 or 7.6 is detected, the webapp explicitly returns "FortiOS 7.4/7.6 support coming Sept/Oct — please upload a 7.2 export for now" with the planned ship date.
4. Updated `templates/index.html` + landing page copy: "Multi-Vendor Firewall Migration Toolkit"
5. Updated `config/fes_webapp.ini`: brand strings, pricing tier descriptions
6. Smoke test: end-to-end FortiGate-7.2 upload through the webapp produces downloadable WG XML

**Definition of Done**: `bash run-dev.sh` serves multi-vendor portal at `localhost:8000` accepting SonicWall + FortiGate-7.2 uploads. 7.4/7.6 uploads detected and gracefully deferred with clear messaging.

---

### Sprint 8 — Documentation + Release Notes + Methodology paper update (4-6 days)

**Goal**: Match the SonicWall toolkit's documentation discipline.

**Deliverables**:
1. `FES-FORTI-RELEASE-NOTES.md` — full release notes following the v1.8.7 format
2. Update to the Golden Rosetta Stone methodology paper: new section "Multi-Vendor Case Study" documenting IR reuse, FortiOS grammar regularity finding, and lab cycle results
3. LinkedIn launch article
4. Updated landing page hero text + features list
5. `docs/FORTIGATE-OPERATOR-GUIDE.md` — operator-facing documentation
6. **Forward-looking documentation**: release notes explicitly call out the 7.4/7.6 autumn extension roadmap, the INI-branched architecture that enables it, and the planned ship windows. This sets expectations for early customers who run 7.4/7.6 and lets acquirers see the architectural runway.

**Definition of Done**: any acquirer reviewing the project can read the docs cold and understand: what's the moat, what's in scope, what's the methodology, what's lab-confirmed, what's documented limit, what's the near-term roadmap.

---

## 7. Total summer budget (FortiOS 7.2 only)

| Sprint | Hours (low) | Hours (high) | Days (low) | Days (high) |
|---|---|---|---|---|
| 0 — Foundation, IR, 60E baseline | 30 | 50 | 5 | 7 |
| 1 — Tokenizer, router, easy sections | 40 | 60 | 6 | 8 |
| 2 — Services + routing | 25 | 40 | 4 | 6 |
| 3 — Firewall policy | 55 | 85 | 8 | 12 |
| 4 — VPN | 40 | 60 | 6 | 8 |
| 5 — Golden master on 60E | 30 | 50 | 5 | 7 |
| 6 — Lab cycles | 55 | 110 | 8 | 14 |
| 7 — Web app integration | 25 | 45 | 4 | 6 |
| 8 — Docs + Release Notes | 25 | 45 | 4 | 6 |
| **TOTAL SUMMER (7.2 only)** | **325** | **545** | **50** | **74** |

**Summer-vacation translation**:
- At 6 focused hours/day → 54-91 working days = **8-13 weeks**
- At 4 focused hours/day → 81-136 working days = **12-20 weeks**

Summer 2026 (June 15 → August 31) = ~11 weeks. At 6 hr/day intensity, 7.2-only ships within summer with breathing room. At 4 hr/day vacation pace, 7.2-only fits the lower-bound estimate; the higher-bound estimate spills into early September.

---

## 8. Fall 2026 extension budget (7.4 and 7.6 INI-branched)

| Sprint | Hours (low) | Hours (high) | Days (low) | Days (high) |
|---|---|---|---|---|
| 9 — 7.4 INI branch | 80 | 130 | 13 | 22 |
| 10 — 7.6 INI branch | 80 | 130 | 13 | 22 |
| 11 — Autumn polish (optional) | 30 | 60 | 5 | 10 |
| **TOTAL FALL (7.4 + 7.6)** | **190** | **320** | **31** | **54** |

**Fall pace**: at vacation-end pace transitioning into MSP-work-resumption pace, 4-5 hours/day. Sprint 9 (7.4) targets mid-Sept to mid-Oct ship; Sprint 10 (7.6) targets late-Oct to mid-Nov ship.

---

## 9. Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Customer FortiGate configs use VDOMs | Medium | Medium | Document VDOM-not-supported as Sprint 0 honest-perimeter line; VDOM extension as v2027 work |
| UTM profile carry-throughs frustrate operators | Medium | Low | Operator guide + `[warn]` messages spell out what's preserved and what needs manual WG-side reconstruction |
| FortiOS 7.2.x patch level matters more than expected (different field defaults across 7.2.0 to 7.2.13) | Low | Low | Sprint 0 picks ONE patch level explicitly; document it; subsequent customer corpora that drift can be lab-cycled if needed |
| Lab cycle reveals a class of bug not solvable by INI alone | Medium | Medium | Same discipline as SonicWall: diagnose, INI catalog where possible, focused code fix where necessary, document provenance |
| Existing SonicWall toolkit regresses while adding FortiGate | Low | High | Validator gauntlet runs against BOTH SonicWall corpora on every change; CI-style automation re-validates SHAs on every commit |
| 60E firmware-swapping during Sprint 5 / Sprint 9-10 turns into a wall-clock time sink | Low | Low | Document the firmware-flash procedure in `docs/LAB-HARDWARE.md` once; subsequent reflashes are routine |
| Fall extension to 7.4 surfaces a structural FortiOS change that 7.2 architecture can't absorb cleanly | Low | Medium | The version-router scaffolding from Sprint 1 onward means a structural change becomes a new `[exception.fortios_7_4.*]` block, not a code rewrite. If the change is truly structural (analog to SonicOS 6.2 inline form), it gets the same `section_form_override` treatment we built for SonicWall Fix D |
| Multi-vendor portal UX confuses MSPs who only run one vendor | Low | Low | Vendor picker defaults to auto-detect; auto-detect is reliable for both SonicOS and FortiOS headers; manual override available |

---

## 10. Non-goals (explicit scope discipline)

### Not in summer 2026

- FortiOS 7.4 or 7.6 lab validation (those are autumn — Sprint 9/10)
- VDOMs
- HA cluster awareness
- UTM profile translation
- Wireless controller
- SSL VPN web portal
- IPv6 emit on WG side
- Dynamic routing (RIP/OSPF/BGP/IS-IS) — static routes only
- Traffic shaping / QoS
- FortiOS proxy modes (explicit/transparent)
- FortiManager / FortiAnalyzer integration

### Not in autumn 2026 either

- VDOMs (still — that's v2027 territory)
- HA cluster awareness
- UTM profile translation
- Third vendor (Cisco ASA, Palo Alto)

Documenting non-goals up front prevents scope creep mid-sprint.

---

## 11. Lab + corpus strategy

### Hardware

| Hardware | Role | Status |
|---|---|---|
| WatchGuard Firebox T-30 | WG-side empirical truth source (target Firebox for import testing) | Existing — Fireware 12.5.9 |
| FortiGate 60E | FortiOS-side empirical truth source (config generation, FortiOS Web UI reference) | Racked — firmware TBD per Sprint 0 decision |

### Corpora

| Asset | Source | Role | Sprint |
|---|---|---|---|
| GitHub FG-100D dump | Already in `/mnt/user-data/uploads/fortigate_show_full-configuration.txt` | Shape reference for parser; regression net | Sprint 0 (and unit tests throughout) |
| 60E factory-default 7.2 export | Direct hardware capture | Baseline diff target | Sprint 0 |
| 60E hand-built 7.2 golden master | Direct hardware capture after marker-tagged config build | Validator golden + Rosetta Stone documentation | Sprint 5 |
| Customer-realistic 7.2 corpora | Anonymized configs from MSP work / friendly customers | Sprint 6 lab inputs | Sprint 6 |
| 60E 7.4 golden master | Direct hardware capture after firmware swap | Sprint 9 validator golden | Fall — Sprint 9 |
| 60E 7.6 golden master | Direct hardware capture after firmware swap | Sprint 10 validator golden | Fall — Sprint 10 |

### Methodology continuity

Same Golden Rosetta Stone discipline as SonicWall:
- Hand-built marker-tagged corpus serves BOTH as validator golden AND as operator-facing documentation
- "No claim without provenance" — every architectural decision traces to a specific lab observation, FortiOS config line, or WG XML shape verified against the jpa template
- Lab as final validator — real T-30 hardware is the only acknowledged truth source for "does this actually work"

---

## 12. Communication & cadence

| Frequency | What |
|---|---|
| Per sprint | Sprint open: confirm Definition of Done + risks; Sprint close: ship deliverables, write a lab-quality findings note, decide Go/No-Go for next sprint |
| Mid-sprint | If a Sprint 3-class blocker emerges, pause the sprint and write a one-page architectural decision memo before continuing |
| Weekly | Update `FORTIGATE-PROGRESS.md` with a Friday status: sprint-in-flight, hours-burned, current blockers, lab findings count |
| Per lab cycle (Sprint 6) | Captain runs the T-30 import + 60E cross-reference; reports findings in a structured format matching the SonicWall lab cycle convention |
| End of summer | Public ship of FortiGate 7.2 — release notes, LinkedIn article, methodology paper update |
| Mid-fall | Public ship of FortiGate 7.4 INI branch (Sprint 9 close) |
| Late fall | Public ship of FortiGate 7.6 INI branch (Sprint 10 close) |

---

## 13. Definition of Done — summer 2026

The FortiGate 7.2 toolkit ships when ALL of these are true:

- [ ] Validator gauntlet 10/10 green on at least 2 FortiOS 7.2 corpora (60E golden master + customer-realistic 7.2 sample)
- [ ] T-30 lab cycle: WG XML imports cleanly, Web UI renders all panels without crash, customer-defined entries visible, VPN tunnels show correct selectors, login persists
- [ ] 60E Web UI cross-reference: visual comparison between FortiOS-side reading and WG-side translation shows semantic equivalence on the test corpus
- [ ] `bash run-dev.sh` serves a multi-vendor portal accepting SonicWall and FortiGate-7.2 uploads with version-aware messaging for 7.4/7.6 uploads
- [ ] `FES-FORTI-RELEASE-NOTES.md` complete with by-the-numbers, fix narratives, honest perimeter, fall extension roadmap
- [ ] Golden Rosetta Stone methodology paper updated with multi-vendor case study section
- [ ] LinkedIn launch article drafted and reviewed
- [ ] Operator-facing FortiGate documentation complete
- [ ] All SonicWall regression tests still pass (NSA-3600 + TZ-300 SHAs match v1.8.7 baseline)
- [ ] `fortios_version_detection.ini` + `fortios_version_exceptions.ini` scaffolding in place with placeholder 7.4/7.6 sections ready for autumn extension

---

## 14. Definition of Done — fall 2026 (7.4 + 7.6 INI extensions)

Each fall ship (Sprint 9 for 7.4, Sprint 10 for 7.6) follows the same pattern:

- [ ] `[branch.fortios_7_X]` populated in `fortios_version_detection.ini`
- [ ] `[exception.fortios_7_X.*]` entries documenting field deltas vs 7.2 baseline
- [ ] Hand-built 60E golden master at the target FortiOS version
- [ ] `golden_pair_registry.ini` entry with `lab_status = certified` after lab cycle
- [ ] Validator gauntlet 10/10 green on the new version's corpora
- [ ] T-30 lab cycle completed with full Web UI rendering verification
- [ ] All prior SonicWall + FortiOS 7.2 (and 7.4, in 7.6 case) regression tests still pass
- [ ] Release notes addendum documenting the extension

---

## 15. Algebraic-design discipline carry-over

This roadmap is built on the foundation of FES v1.8.7 — the SonicWall→WG toolkit and its proven architecture. The same disciplines apply, now extended to a multi-vendor + multi-version pattern:

- **Algebraic design**: every translation rule expressed as `Object × Verb = Action` in INI catalogs, not hardcoded
- **No claim without provenance**: every architectural decision traces to a specific lab observation, FortiOS config line, or WG XML shape verified against the jpa template
- **Golden Rosetta Stone methodology**: hand-built corpus pairs with embedded markers serve as both validator targets AND operator-facing documentation
- **Validator gauntlet as safety net**: 10 validators run on every emit; any change that breaks them gates the merge
- **Lab as final validator**: real T-30 hardware is the only acknowledged truth source for "does this actually work"
- **INI-branching for version extensions**: the version_router + version_exceptions architecture proven on SonicOS (6.5 baseline + 6.2 extension) is the template for FortiOS (7.2 baseline + 7.4/7.6 extensions)

The FortiGate toolkit will be measured against the same bar that v1.8.7 SonicWall meets.

---

**End of roadmap — FES FortiGate Migration Toolkit v2**
**Summer 2026: FortiOS 7.2 lab-validated ship**
**Autumn 2026: 7.4 and 7.6 INI-branched extensions**
