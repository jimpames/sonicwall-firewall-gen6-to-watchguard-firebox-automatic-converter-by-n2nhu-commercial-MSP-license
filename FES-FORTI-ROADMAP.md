# FES FortiGate Migration Toolkit
## Roadmap, Sprint Plan & Engineering Work Plan

**Working title**: FES-FORTI (Firewall Ejector Seat — FortiGate edition)
**Target completion**: End of summer 2026
**Author**: n2nhu lab — J P Ames, Lead Engineer
**Status**: Planning — pre-Sprint 0

---

## 1. Strategic context

FES v1.8.7 (the SonicWall→WG toolkit) is shipping with two lab-validated corpora and an architecture built on the **algebraic-design methodology**: every translation rule expressed as `Object × Verb = Action`, declared in INI catalogs, evaluated at runtime, never hardcoded.

The downstream half of the toolkit (enricher → skeleton_filler → validator gauntlet → WG XML emit) is **vendor-agnostic by construction**. It consumes a JSON intermediate representation of customer records and produces WatchGuard XML. The SonicWall side is one specific upstream that emits records into that IR.

This roadmap extends FES to a second upstream: **FortiOS configuration files** → same IR → same downstream → same WG XML output.

### Why this is the right next vendor

| Criterion | FortiGate fit |
|---|---|
| Market size | Top-3 SMB/Mid-enterprise firewall vendor by deployment count. MSP overlap with WatchGuard prospects is significant. |
| Migration pain | FortiOS patching/downgrade discipline is notoriously fragile; operators have direct motivation to migrate to platforms with cleaner release management. |
| Grammar regularity | FortiOS config grammar is uniform across major versions (5.x/6.x/7.x). No SonicOS-style 6.2 vs 6.5 grammar divergences expected. |
| Architectural fit | FortiOS records map cleanly to the existing JSON IR (interfaces, address objects, service objects, schedules, firewall policies, VPN phase1/phase2, static routes). No IR refactor needed. |

### Why this is NOT a coffee break

FortiOS introduces semantic territory that doesn't exist on the SonicWall side:
- **VDOMs (Virtual Domains)** — multiple virtual firewalls per physical device. Decision: MVP is single-VDOM only.
- **UTM profile attachment** on firewall policies (web filter, AV, IPS, application control). Decision: WG-side analogs limited; carry as `[warn]` for operator review.
- **HA pairs** with cross-references between primary and secondary. Decision: MVP assumes standalone (`vdom=0`, non-HA).
- **`vip` (Virtual IP)** DNAT entries — separate section from policies; relationship to firewall policies must be reconstructed. Decision: in-scope.

### Strategic deliverables

By end of summer 2026:

1. **Working FortiGate→WG translator** matching the v1.8.7 SonicWall toolkit's quality bar (10/10 validator gauntlet, lab-cycled on T-30, hand-built golden master with embedded markers).
2. **Updated FES product positioning**: "Multi-Vendor Firewall Migration Toolkit" — both SonicWall and FortiGate inputs supported via the same web UI and CLI.
3. **Methodology paper update**: the Golden Rosetta Stone methodology paper extended to document multi-vendor case study — strong academic + commercial defensibility signal.

---

## 2. Architecture — leveraging what's already there

```
                    ┌─────────────────────────────────────┐
SonicOS e-CLI ─────▶│ sonicwall_parser/                  │─┐
                    │   (version-router, exceptions)     │ │
                    └─────────────────────────────────────┘ │
                                                            │
                    ┌─────────────────────────────────────┐ │   ┌──────────────┐    ┌──────────────┐
FortiOS config ────▶│ fortigate_parser/      (NEW)        │─┴──▶│ JSON IR      │───▶│ enricher/    │──┐
                    │   (uniform-grammar tokenizer)       │     │ (unchanged)  │    │ (unchanged)  │  │
                    └─────────────────────────────────────┘     └──────────────┘    └──────────────┘  │
                                                                                                       │
                                                                                                       │
                                                                                                       ▼
                                                                                          ┌────────────────────┐
                                                                                          │ skeleton_engine/   │
                                                                                          │ (unchanged)        │
                                                                                          └────────────────────┘
                                                                                                       │
                                                                                                       ▼
                                                                                                  WG XML
```

**What changes**: one new directory `fortigate_parser/`, paralleling `sonicwall_parser/`, that emits the same JSON IR.

**What does NOT change**:
- `enricher/` — already consumes the IR, doesn't care about upstream source
- `skeleton_engine/` — golden_pair_registry will get a new branch family entry, but the skeleton_filler, alias_filter, access_rule_filter, sw_builtin_dynamic_groups, validators all work as-is
- `xml_emitter/` — XML emit configuration is WG-side, no change
- Web app wrapper (`fes_webapp/`) — wraps both parsers behind a single entry point; minimal UI change to add a vendor picker
- Validator gauntlet — runs unchanged

**The universal IR is the centerpiece of the architecture.** If a sprint deliverable can't slot into the existing IR cleanly, that's a design red flag — pause, re-examine, don't paper over.

---

## 3. Universal Intermediate Representation (IR) — formalization

The IR is currently implicit: it's the JSON files in `sonicwall_parser/output/` and `enricher/output/`. Sprint 0 formalizes it.

### IR record types (already produced by SonicWall parser, will be produced by FortiGate parser)

| File | Records | Key fields | FortiOS source section |
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

## 4. Sprint breakdown

**Estimating discipline**: hours are wall-clock focused engineering, not "ideal hours." Calendar-days assume 6-8 focused hours per working day (passion-project intensity, not 9-to-5). Each sprint has an explicit Definition of Done and a Go/No-Go gate before the next sprint starts.

### Sprint 0 — Foundation & IR formalization (4-6 days)

**Goal**: De-risk the architecture before writing any FortiGate parser code. Confirm the IR is rigorously specified.

**Deliverables**:
1. `docs/IR-SCHEMA.md` — formal IR schema with field-level semantics for each record type
2. `fortigate_parser/` directory skeleton — empty but with the same internal layout as `sonicwall_parser/` (`input/`, `output/`, parser entry script, config INI placeholder)
3. `fortigate_parser/parser_config.ini` — top-level section catalog (the 14 migration-relevant FortiOS sections)
4. **Validation test**: run the existing SonicWall parser on the NSA-3600 master, dump its output to `/tmp/sonicwall_ir/`. Hand-author equivalent JSON for a tiny FortiGate snippet (3-4 interfaces, 5-6 address objects, 1 VPN). Run the existing enricher on the hand-authored FortiGate IR. Confirm: enricher produces sensible output without errors.

**Definition of Done**: enricher consumes hand-authored FortiGate IR without crashing or producing schema-shape complaints from the validator gauntlet. This proves the architectural assumption empirically before sprint 1 begins.

**Go/No-Go gate**: if the enricher does NOT cleanly consume hand-authored FortiGate IR, STOP. Either the IR isn't actually vendor-agnostic and needs refactoring, or the IR schema doc has gaps. Fix that before Sprint 1.

---

### Sprint 1 — Tokenizer & easy-section parsers (5-7 days)

**Goal**: Working FortiOS tokenizer that handles the uniform `config`/`edit`/`set`/`next`/`end` grammar, plus parsers for the four lowest-complexity sections.

**Sections covered**:
- `config system interface` (interface records)
- `config firewall address` (address objects)
- `config firewall addrgrp` (address groups)
- `config firewall schedule recurring` (schedules)

**Deliverables**:
1. `fortigate_parser/fortigate_parser.py` — tokenizer + dispatcher framework. **Algebraic-design discipline**: section definitions come from `parser_config.ini`, NOT hardcoded `if-section-name` chains. Same pattern as `sonicwall_parser.py`.
2. Field-mapping tables in `fortigate_parser/field_mappings/`:
   - `interface_field_map.ini` — FortiOS field → IR field
   - `address_field_map.ini`
   - `addrgrp_field_map.ini`
   - `schedule_field_map.ini`
3. Unit test: run parser on the GitHub sample config (`fortigate_show_full-configuration.txt`), confirm IR output for these 4 sections matches expected shape (compare against hand-authored expected output for ~10 records).

**Definition of Done**: parser produces `interfaces.json`, `address_objects.json`, `address_groups.json`, `schedules.json` from the sample config, and the enricher consumes them without complaint.

**Risks for this sprint**:
- **Field-mapping ambiguity** for `interface` records: FortiOS has `set type physical|aggregate|vlan|loopback|tunnel|...` — only `physical` and `vlan` have direct WG analogs. Decision: emit physical + VLAN; carry others as `[warn]` for operator review.
- **`set associated-interface ''`** field on address objects: FortiOS-specific concept (binds an address object to a specific interface for routing). WG has no analog; ignore field during translation, document as FortiOS-only.

---

### Sprint 2 — Services + routing (4-5 days)

**Goal**: Service object/group parsing + static route parsing. These are mechanically simple but semantically rich (port-range syntax variants, ICMP type codes, etc.).

**Sections covered**:
- `config firewall service custom` (service objects — 87 entries in the sample, most are factory defaults)
- `config firewall service group`
- `config router static`

**Deliverables**:
1. `fortigate_parser/field_mappings/service_field_map.ini` — handles TCP/UDP/SCTP port ranges, ICMP type/code, IP protocol number, application protocol references
2. `fortigate_parser/field_mappings/service_group_field_map.ini`
3. `fortigate_parser/field_mappings/static_route_field_map.ini`
4. **Service factory-default filter**: same architectural pattern as `alias_filter.ini` — INI catalog of FortiOS factory-default service names that should be dropped (analog to jpa template factory carry-throughs). Filter pass exempted from customer_fidelity reporting.

**Definition of Done**: all four IR files populated, validator gauntlet runs the IR through enricher + skeleton_filler producing a valid WG XML (even if functionally incomplete — policies are next sprint).

**Risks**:
- FortiOS `iprange` address type (e.g., `start-ip 10.0.0.1 end-ip 10.0.0.50`) doesn't have an exact SonicOS analog; WG side has a member type for ranges. Verify against the WG CLI Reference v12.11 — already in project knowledge.

---

### Sprint 3 — Firewall policy parsing (the deepest semantic territory) (7-10 days)

**Goal**: Translate FortiOS firewall policies, including UTM-profile attachments (which become `[warn]` carry-throughs), NAT-on-policy semantics (SNAT), and reference resolution to address objects / address groups / service objects / service groups.

**Sections covered**:
- `config firewall policy`
- `config firewall vip` (DNAT objects — reference relationship to policies must be reconstructed)

**Why this sprint is the longest**: FortiOS firewall policies are semantically the richest section. Unlike SonicOS access-rules (which are mostly source/dest/service/action), FortiOS policies bundle UTM profiles, schedule references, application-control settings, AV/IPS/web-filter attachments, NAT semantics, traffic shaping, and more — all in one `config firewall policy` `edit N` block.

**Deliverables**:
1. `fortigate_parser/field_mappings/policy_field_map.ini` — comprehensive field-by-field mapping with three categories:
   - **Direct translate**: `srcintf`, `dstintf`, `srcaddr`, `dstaddr`, `service`, `action`, `schedule`, `status`, `name`
   - **WG-equivalent transform**: NAT-on-policy (`nat enable`) becomes a separate `<nat-policy>` SNAT entry
   - **No WG analog (`[warn]` carry-through)**: `utm-status`, `av-profile`, `webfilter-profile`, `application-list`, `ips-sensor`, `dlp-sensor`
2. `fortigate_parser/policy_filter.ini` — drop rules for FortiOS auto-generated policies (e.g., factory implicit-deny rules). Analog to `access_rule_filter.ini`.
3. `vip_field_map.ini` — DNAT entries
4. **Reference-resolution test**: hand-pick 5 policies from the sample config that reference `srcaddr`/`dstaddr` to specific address objects; confirm IR correctly resolves all references end-to-end.

**Definition of Done**: WG emit contains FortiOS policies with correct source/dest/service zone wiring; UTM-attached policies show `[warn]` in the pipeline log; SNAT and DNAT correctly emit `<nat-policy>` entries.

**Risks**:
- **UTM profile carry-through scope**: customer may legitimately need WAF / web filter / antivirus. Operator must reconstruct in WG Web UI using equivalent WG features (WG has its own AV/IPS/web blocker stack, just different config shape). MVP scope is correctly carrying the *firewall policy itself*, not the UTM profiles.
- **`set name ''` and `set name 'someName'` distinction**: FortiOS policies may have explicit names (the customer named them) or empty names (auto-numbered). Translation needs to handle both — generate WG `<name>` from policy ID number if SW name is empty.

---

### Sprint 4 — VPN (Phase 1 + Phase 2) (5-7 days)

**Goal**: Translate IPSec VPN configurations. This is where Fix F's selector resolution pattern from SonicWall pays dividends — the architecture is already there.

**Sections covered**:
- `config vpn ipsec phase1-interface` (gateway endpoint)
- `config vpn ipsec phase2-interface` (tunnel selectors)

**FortiOS specifics**:
- Phase 1 and Phase 2 are SEPARATE sections in FortiOS, joined by `phase2-interface.phase1name == phase1-interface.<id>` cross-reference
- FortiOS encodes the SonicWall `network local / remote` selectors as `phase2-interface.src-subnet` / `dst-subnet` (direct CIDR) and `src-addr-type subnet` / `name` / `range` discriminator
- PSK ciphertext (`set psksecret ENC ...`) is FortiOS-encrypted; same situation as SonicWall — opaque carry-through, operator rotates

**Deliverables**:
1. `fortigate_parser/field_mappings/vpn_phase1_field_map.ini`
2. `fortigate_parser/field_mappings/vpn_phase2_field_map.ini`
3. **Joining pass**: helper that walks both sections, joins by `phase1name`, emits a unified `vpn_policies.json` record per VPN tunnel
4. Selector resolution reuses the existing `_resolve_vpn_selector()` helper from `skeleton_filler.py`. FortiOS `dst-addr-type name` references resolve through customer address objects; `subnet` references emit direct CIDR — same shape as SonicWall after IR alignment.

**Definition of Done**: VPN tunnel from the sample FortiOS config renders correctly in WG Web UI Branch Office VPN screens (Phase 1 Gateway Endpoint + Phase 2 Addresses tab) — local/remote selectors correct, peer-addr correct, IKE proposals translated.

**Risks**:
- **IKE proposal translation**: FortiOS uses `set proposal aes256-sha1` (single string) while WG schema separates encryption and authentication algorithms. Mapping table in `vpn_phase1_field_map.ini` decomposes the FortiOS combined token into WG's separate algorithm fields.
- **DH group encoding**: FortiOS `set dhgrp 2` is a numeric code; WG schema uses string enum. Use `wg_enums.ini` for the mapping (already exists).
- **`auto-discovery`**, **`auto-negotiate`**, **`exchange-interface-ip`** — FortiOS-specific advanced features without WG analogs. `[warn]` carry-through.

---

### Sprint 5 — Golden master + WG reference build (4-6 days)

**Goal**: Hand-build the marker-tagged FortiOS golden master and corresponding WG XML target, following the Golden Rosetta Stone methodology established for SonicWall.

**Deliverables**:
1. `skeleton_engine/golden_pair/fortigate100d_5_x/sw_input.txt` — hand-crafted FortiOS config with embedded Claude markers. The marker conventions mirror the SonicWall master:
   - `*NameORAdrClaude` for fields accepting NAME-or-ADdress
   - `*OripClaude` / `*DnsOrIpClaude` for fields accepting DOMAIN-or-IP
   - `forClaude` suffix on customer-defined object names (interfaces, address objects, policies, VPN endpoints)
2. `skeleton_engine/golden_pair/fortigate100d_5_x/wg_reference.xml` — hand-built WG XML target with matching Claude markers, embedded as both element names AND domain-name-shaped values (per the proven Captain methodology)
3. New entry in `skeleton_engine/golden_pair_registry.ini`:
   ```ini
   [golden.fortigate100d_5_x]
   matches_branch       = fortios_5_x
   matches_model        = FortiGate-100D
   sw_reference         = skeleton_engine/golden_pair/fortigate100d_5_x/sw_input.txt
   wg_reference         = skeleton_engine/golden_pair/fortigate100d_5_x/wg_reference.xml
   wg_reference_status  = present
   lab_status           = validator_only
   ```
4. New `fortios_version_detection.ini` for the FortiOS branch detection (analog to `version_detection.ini` for SonicOS) — even though MVP is single-version, the architecture supports future FortiOS versions cleanly

**Definition of Done**: validator gauntlet runs 10/10 green against the hand-built FortiOS golden, both at the SonicWall corpus AND the FortiGate corpus. Both `lab_status=certified` (SonicWall) and `lab_status=validator_only` (FortiGate, pending lab cycle) entries co-exist cleanly.

---

### Sprint 6 — Lab cycle iterations (7-14 days, depending on findings)

**Goal**: Convert the GitHub sample FortiGate config (a real-world FG-100D dump) through the FES-FORTI pipeline; load the resulting WG XML on the T-30 Fireware 12.5.9; observe; diagnose; fix; repeat.

**Deliverables**:
1. First WG XML output from the GitHub sample config — handed off for T-30 import
2. Lab finding reports (mirroring the SonicWall TZ-300 lab cycle format) — each finding produces an INI rule + provenance citation
3. Iterative fixes following the same discipline as v1.8.4-v1.8.7: lab observation → diagnose against grammar + WG behavior → INI catalog update → validator re-run → re-handoff

**Definition of Done**:
- T-30 imports the WG XML cleanly (no `400 Invalid XML` errors)
- Web UI Interfaces panel renders correctly
- Web UI Policies panel shows translated firewall policies with correct service objects and zone wiring
- Web UI Aliases panel renders without crash, customer-defined address objects visible
- Web UI Branch Office VPN screens show Phase 1 + Phase 2 correctly
- Operator login persists across panel navigation (the v1.8.5 crash-prevention discipline)

**Number of lab cycles expected**: 3-5 iterations matching the SonicWall pattern. Each iteration ~2 days (lab observation + diagnosis + fix + re-emit + re-validate + handoff).

---

### Sprint 7 — Web app integration + product surface (3-5 days)

**Goal**: Wire the FortiGate parser into `fes_webapp/` so the existing portal/UI/API serve both vendors.

**Deliverables**:
1. Updated `app/main.py`: vendor picker in the upload form ("SonicWall" / "FortiGate" radio buttons); routes upload to the right parser based on selection
2. Auto-detection helper: if the customer doesn't pick a vendor, sniff the first few lines of the uploaded file — `config-version=FG` → FortiGate; SonicWall e-CLI header pattern → SonicWall; show the detection result and let the user confirm
3. Updated `templates/index.html` + landing page copy: "Multi-Vendor Firewall Migration Toolkit"
4. Updated `config/fes_webapp.ini`: brand strings, pricing-tier descriptions
5. Smoke test: end-to-end FortiGate upload through the webapp produces downloadable WG XML

**Definition of Done**: `bash run-dev.sh` serves a working multi-vendor portal at `localhost:8000` that accepts both SonicWall and FortiGate uploads and produces correct WG XML for both.

---

### Sprint 8 — Documentation + Release Notes + Methodology paper update (3-5 days)

**Goal**: Match the SonicWall toolkit's documentation discipline so FortiGate ships with the same level of operator + acquirer-ready polish.

**Deliverables**:
1. `FES-FORTI-RELEASE-NOTES.md` — full release notes following the v1.8.7 format (architecture, sprints, by-the-numbers, honest perimeter, file manifest)
2. Update to the Golden Rosetta Stone methodology paper: new section "Multi-Vendor Case Study" documenting the IR reuse, the FortiOS grammar regularity finding, and the lab cycle results
3. LinkedIn launch article: "FES now supports FortiGate — same toolkit, same lab-validated discipline"
4. Updated landing page hero text + features list
5. `docs/FORTIGATE-OPERATOR-GUIDE.md` — operator-facing documentation: what FortiOS features translate, what requires Web UI manual reconstruction, what `[warn]` items mean, lab-recommended import sequence

**Definition of Done**: any acquirer reviewing the project can read the docs cold and understand: what's the moat, what's the scope, what's the methodology, what's lab-confirmed, what's documented limit, what's still TODO.

---

## 5. Total budget

| Sprint | Hours (low) | Hours (high) | Days (low) | Days (high) |
|---|---|---|---|---|
| 0 — Foundation & IR | 25 | 40 | 4 | 6 |
| 1 — Tokenizer & easy sections | 35 | 50 | 5 | 7 |
| 2 — Services + routing | 25 | 35 | 4 | 5 |
| 3 — Firewall policy | 50 | 70 | 7 | 10 |
| 4 — VPN | 35 | 50 | 5 | 7 |
| 5 — Golden master | 25 | 40 | 4 | 6 |
| 6 — Lab cycles | 50 | 100 | 7 | 14 |
| 7 — Web app integration | 20 | 35 | 3 | 5 |
| 8 — Docs + Release Notes | 20 | 35 | 3 | 5 |
| **TOTAL** | **285** | **455** | **42** | **65** |

**Summer-vacation translation**:
- At 6 focused hours/day → 47-76 working days = roughly **7-11 weeks**
- At 4 focused hours/day (more sustainable for "vacation" intensity) → 71-114 working days = roughly **10-16 weeks**

For a focused summer (June-August), the high-confidence delivery is **end of August 2026** with a 2-week buffer for inevitable scope expansion / lab surprise fixes.

---

## 6. Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Customer FortiGate configs use VDOMs that MVP doesn't support | Medium | Medium | Document VDOM-not-supported as Sprint 0 honest-perimeter line; add VDOM iteration as Sprint 9+ if there's customer pull |
| UTM profile carry-throughs frustrate operators expecting auto-translation | Medium | Low | Clear operator guide + `[warn]` messages spell out what's preserved and what needs manual WG-side reconstruction |
| FortiOS 7.x adds new sections not in the 5.x sample | Low | Low | Architecture absorbs new sections via `parser_config.ini` additions; no code changes for routine cases |
| Lab cycle reveals a class of bug not solvable by INI alone | Medium | Medium | Same discipline as SonicWall: diagnose against grammar + WG behavior; INI catalog where possible, focused code fix where necessary; document provenance |
| Validator gauntlet doesn't catch FortiOS-specific failure modes | Low | High | Sprint 6 lab cycle is the safety net; if validators miss something, that's a sign to ADD a new validator (which strengthens the toolkit for BOTH vendors) |
| Existing SonicWall toolkit regresses while adding FortiGate | Low | High | Validator gauntlet runs against BOTH SonicWall corpora on every change. Sprint 0 sets up CI-style automation that re-validates the SonicWall NSA-3600 + TZ-300 SHAs after every commit |
| Customer PSK encrypted format on FortiOS isn't carryable | Low | Low | Same situation as SonicWall — opaque carry-through, operator rotates. Documented limit, not bug |
| The GitHub sample config doesn't exercise enough customer-realistic scenarios | High | Medium | Source 2-3 additional sample FortiGate configs (anonymized) from MSP forums / friendly customers; lab-cycle multiple corpora before declaring Sprint 6 done |

---

## 7. Non-goals (explicit scope discipline)

Things that are NOT in scope for the summer 2026 deliverable:

1. **VDOMs** (multiple virtual firewalls per device) — MVP is single-VDOM only
2. **HA cluster member sync** — MVP assumes standalone FortiGate
3. **FortiManager / FortiAnalyzer integration** — MVP reads from raw config dumps, not centralized management systems
4. **UTM profile translation** (web filter, antivirus, IPS, application control, DLP) — these are `[warn]` carry-throughs; operator reconstructs in WG Web UI using equivalent WG features
5. **Wireless controller config** (`wireless-controller wtp-profile`, `wireless-controller wids-profile`) — out of scope
6. **SSL VPN web portal** — out of scope (different WG product class)
7. **IPv6** — same scope decision as SonicWall toolkit; parsed but not emitted
8. **Dynamic routing** (RIP/OSPF/BGP/IS-IS) — static routes only for MVP; dynamic routing typically requires operator review on WG side anyway
9. **Traffic shaping / QoS** — Sprint 9+ if customer pull
10. **FortiOS proxy modes** (explicit proxy, transparent proxy) — out of scope (different WG product class)

Documenting non-goals up front prevents scope creep mid-sprint.

---

## 8. Lab + corpus strategy

The SonicWall toolkit's success traces directly to lab discipline:
- Real T-30 Fireware 12.5.9 hardware as the empirical truth source
- Hand-built golden master with embedded markers for traceability
- Validator gauntlet as the safety net between code change and lab handoff
- "No claim without provenance" enforced on every architectural decision

**FortiGate roadmap inherits this discipline:**

| Asset | What | Source |
|---|---|---|
| Lab hardware | T-30 Fireware 12.5.9 | Existing (no new purchases) |
| Reference FortiGate config | `fortigate_show_full-configuration.txt` (GitHub) | Already in `/mnt/user-data/uploads/` |
| Customer-realistic corpus | 2-3 additional anonymized FortiGate configs | Source from MSP community / friendly customers during Sprint 0 |
| Golden master corpus | Hand-built FG-100D / FortiOS 5.x master + WG XML target | Sprint 5 deliverable |
| Validator gauntlet | Reuses 10 existing validators unchanged | Sprint 0 verifies compatibility |

---

## 9. Communication & cadence

| Frequency | What |
|---|---|
| Per sprint | Sprint open: confirm Definition of Done + risks; Sprint close: ship deliverables, write a lab-quality findings note, decide Go/No-Go for next sprint |
| Mid-sprint | If a Sprint 3-class blocker emerges (FortiOS UTM semantic mismatch, etc.), pause the sprint and write a one-page architectural decision memo before continuing |
| Weekly | Update `FORTIGATE-PROGRESS.md` with a Friday status: sprint-in-flight, hours-burned, current blockers, lab findings count |
| Per lab cycle (Sprint 6) | Captain runs the T-30 import; reports findings in a structured "what broke / what worked / what's questionable" format matching the SonicWall lab cycle convention |

---

## 10. Definition of Done — toolkit-wide (Sprint 8 close)

The FortiGate toolkit ships when ALL of these are true:

- [ ] Validator gauntlet 10/10 green on at least 2 FortiGate corpora (golden master + real-world sample)
- [ ] T-30 lab cycle: WG XML imports cleanly, Web UI renders all panels without crash, customer-defined entries visible, VPN tunnels show correct selectors, login persists
- [ ] `bash run-dev.sh` serves a multi-vendor portal accepting both SonicWall and FortiGate uploads
- [ ] `FES-FORTI-RELEASE-NOTES.md` complete with by-the-numbers, fix narratives, honest perimeter
- [ ] Golden Rosetta Stone methodology paper updated with multi-vendor case study section
- [ ] LinkedIn launch article drafted and reviewed
- [ ] Operator-facing FortiGate documentation complete
- [ ] All SonicWall regression tests still pass (NSA-3600 + TZ-300 SHAs unchanged where expected; documented where intentionally changed)

---

## 11. Post-summer roadmap (out of scope for v2026.summer; on the radar for v2027)

| Initiative | Trigger |
|---|---|
| Sprint 9+ — VDOM support | Customer pull from multi-VDOM FortiGate deployments |
| Sprint 9+ — HA cluster awareness | Customer pull |
| Sprint 10+ — third vendor (Cisco ASA, Palo Alto?) | Market signal; methodology paper makes this conceptually straightforward |
| Sprint 10+ — Dynamic routing translation | Customer pull |
| Sprint 11+ — UTM profile translation (where WG analogs exist) | After third vendor adds enough corpus diversity |

---

## 12. Acknowledgements & methodology continuity

This roadmap is built on the foundation of FES v1.8.7 — the SonicWall→WG toolkit and its proven architecture. The same disciplines apply:

- **Algebraic design**: every translation rule expressed as `Object × Verb = Action` in INI catalogs, not hardcoded
- **No claim without provenance**: every architectural decision traces to a specific lab observation, FortiOS config line, or WG XML shape verified against the jpa template
- **Golden Rosetta Stone methodology**: hand-built corpus pairs with embedded markers serve as both validator targets AND operator-facing documentation
- **Validator gauntlet as safety net**: 10 validators run on every emit; any change that breaks them gates the merge
- **Lab as final validator**: real T-30 hardware is the only acknowledged truth source for "does this actually work"

The FortiGate toolkit will be measured against the same bar that v1.8.7 SonicWall meets.

---

**End of roadmap — FES FortiGate Migration Toolkit, summer 2026 plan**
