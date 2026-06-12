# The Golden Rosetta Stone

### Embedded Self-Describing Metadata as a Cross-Vendor Configuration Mapping and Validation Substrate

**Jim Ames** · n2nhu lab · Newburgh, NY
*Applied Algebraic Design for Agentic AI*, Book #33

Written June 2026 · Companion technical paper to the FES (Firewall Ejector Seat) v1.8.3 release

---

## Abstract

We describe a methodology developed during the construction of FES (Firewall Ejector Seat), a translator that converts SonicWall Generation 6 firewall configurations into valid WatchGuard Firebox XML configurations. The central technical contribution is not the translator itself, but the **self-describing metadata substrate** we used to derive and validate the translation rules: a pair of carefully constructed reference configurations — one SonicWall, one WatchGuard — in which **every operator-meaningful field carries an embedded marker** that allows the field's role, shape, and target translation to be recovered from the configuration text itself, with no out-of-band documentation.

We call this construction the **Golden Rosetta Stone**. It is, to our knowledge, the first published application of self-describing metadata to vendor-to-vendor firewall configuration translation. The approach reduced translation-rule derivation from a manual, error-prone cross-referencing exercise into a deterministic shape-pairing problem, made the validation step empirical rather than opinion-based, and ultimately produced a tool that passes a 10-validator gauntlet with composite fidelity 0.9122 on real-world configurations and loads cleanly on production WatchGuard hardware.

This paper documents what the methodology is, why ordinary translation approaches fail at this problem, how the markers actually encode shape information (with real examples), how the decoder consumes them, how validation is performed against the same substrate, and what generalizes to other configuration-translation problems.

---

## 1. The translation problem

SonicWall Gen 6 firewalls reached end of support on April 16, 2026. The vendor's standard remediation — migration to Gen 7 — is the exact attack vector exploited by CVE-2024-40766, which the Akira ransomware group used to compromise SonicWall devices through all of 2025 (At-Bay 2026 InsurSec Report: 86% of Akira attacks involved a SonicWall). The honest engineering answer for stranded Gen 6 fleets is to migrate to a different vendor entirely.

That motivates the technical problem this paper addresses: **convert a SonicWall Gen 6 configuration into a WatchGuard Firebox configuration with high enough fidelity that the converted output loads on real hardware and renders correctly in the target Web UI.**

The naive approach is to write translation rules by reading SonicWall's CLI grammar, reading WatchGuard's XML schema, and authoring code that maps each SonicWall keyword to its WatchGuard equivalent. This approach was attempted as our first architectural pass. It failed.

The failure modes were not minor:

1. **The shape-divergence problem.** A single SonicWall NAT policy declaration (`nat-policy ipv4 ... name foo`) becomes a **family of four coordinated WatchGuard shapes** with cross-references: an `<abs-policy>` entry, a `<policy>` entry with a `-00` suffix, a `from-zone wrapper alias` with a `.1.from` suffix, and a `to-zone wrapper alias` with a `.1.to` suffix. The relationship is not 1:1, and the suffixes are convention, not specification.

2. **The name-form divergence problem.** SonicWall configs frequently use DNS hostnames where WatchGuard configs require IP literals (or vice versa). The same operational intent ("connect to remote peer at this endpoint") is expressed in incompatible value-types, and the translation must recognize which form is which without ambiguity.

3. **The implicit-context problem.** SonicWall auto-generates dozens of internal address-objects (`X0 Subnet`, `X1 IP`, `X2:V77 IPv6 Addresses`) that the operator never sees in the Web UI but that appear verbatim in the e-CLI export. WatchGuard has no equivalent auto-generation. A translator that copies these naively pollutes the target Aliases panel; a translator that drops them silently can lose information the operator needs.

4. **The validation-trust problem.** Even when a translation rule is written, how do you know it's correct? Reading vendor documentation and concluding "this should work" is opinion. The only honest validator is a real configuration loaded onto real hardware. That validator is expensive to run on every rule change.

The first architectural attempt — hardcoded rules per SonicWall keyword — broke down at problem 1 (shape divergence). The second — "thought-out" rules with explicit grammar — broke down at problem 2 (name-form divergence). The third architectural attempt, the one that shipped, broke down none of them. The reason was the Golden Rosetta Stone.

---

## 2. The Golden Rosetta Stone

The methodological insight is simple to state and consequential to apply:

> **Construct a paired reference — one source-format file and one target-format file — such that the same operational intent is expressed in both, AND every operator-meaningful field on both sides carries a unique, machine-recognizable marker that encodes the field's structural role.**

A pair like this is a Rosetta Stone in the literal historical sense: a single artifact in which an unknown grammar can be decoded by reference to a known one, because the same content appears in both grammars and identifiable tokens align the two.

The crucial additional property — and the reason our approach generalizes — is that **the markers do not have to match across the two files.** They have to be **uniquely identifiable as roles** on each side. We deliberately use different marker text on each side, because what we want to encode is not "this name becomes that name," but rather **"this shape becomes that family of shapes."**

### 2.1. What a marker actually looks like

Real example from the FES golden master.

**SonicWall side** (e-CLI text export, ASCII):

```
address-object ipv4 someremotepeerfakeadrobj4Claude
    name someremotepeerfakeadrobj4Claude
    uuid 00000000-0000-0001-0100-c0eae4fb045c
    zone WAN
    host 14.14.14.14
    exit

address-group ipv4 adrgrpforClaude
    name adrgrpforClaude
    uuid 00000000-0000-0001-0200-c0eae4fb045c
    address-object ipv4 someremotepeerfakeadrobj4Claude
    exit
```

**WatchGuard side** (lab-verified hand-built reference XML, abbreviated):

```xml
<alias>
  <name>someremotepeerfakeadrobj4Claude</name>
  <alias-member-list>
    <alias-member>
      <address>someremotepeerfakeadrobj4Claude.1.alm</address>
    </alias-member>
  </alias-member-list>
</alias>

<address-group>
  <name>someremotepeerfakeadrobj4Claude.1.alm</name>
  <addr-group-member>
    <member>
      <type>1</type>
      <host-ip-addr>14.14.14.14</host-ip-addr>
    </member>
  </addr-group-member>
</address-group>
```

The marker `someremotepeerfakeadrobj4Claude` carries through. The value `14.14.14.14` carries through. But the **structure changes**: a SonicWall single-block address-object becomes a WatchGuard two-element family (an `<alias>` plus a backing `<address-group>` with a `.1.alm` suffix). The marker is what tells the decoder "this is the standalone-host-address-object shape," and the corresponding WG structure tells the decoder "this is how that shape lives in WG XML."

### 2.2. A more aggressive example: deliberate name divergence

For the IPSec site-to-site VPN, we deliberately gave the SW and WG sides **different marker names** to force structural matching:

**SonicWall side:**

```
vpn policy site-to-site FakeSite2SiteIKEClaude
    gateway primary FakePriIPSECGwNameORAdrClaude
    gateway secondary FakeSecIPSECGwNameORAdrClaude
    proposal ike phase1
        ike-id local domain-name localikedomainoripClaude
        ike-id peer domain-name peerikedomainoripClaude
        exit
    network remote name someremotepeerfakeadrobj4Claude
```

**WatchGuard side** (markers from the hand-built reference):

```xml
<ipsec-action>
  <name>tunnel.1.bovpntunn4claude</name>
  <local-addr>tunnel.1.bovpntunn4claude.1.tlc</local-addr>
  <remote-addr>tunnel.1.bovpntunn4claude.1.trm</remote-addr>
  <ike-policy-group>gateway.1.branchoffice.for.claude</ike-policy-group>
</ipsec-action>

<ike-action>
  <name>gateway.1.branchoffice.for.claude_bo</name>
  <!-- crypto parameters -->
</ike-action>

<ike-policy>
  <name>gateway.1.branchoffice.for.claude</name>
  <peer-addr>76.76.76.76</peer-addr>
  <ike-action>gateway.1.branchoffice.for.claude_bo</ike-action>
  <local-cert>
    <local-id-data>locgwendptbovpn4Claude</local-id-data>
  </local-cert>
  <peer-auth>
    <peer-auth-mask>4</peer-auth-mask>
    <domain-name>rmtdomain4gwendpt4Claude</domain-name>
  </peer-auth>
</ike-policy>
```

Notice: **none of the marker text matches across sides**. `FakeSite2SiteIKEClaude` does not appear anywhere in the WG side. `localikedomainoripClaude` does not become `locgwendptbovpn4Claude` by any string substitution. They are operator-chosen labels expressing the same role in two different grammars.

This is by design. If we'd used the same marker on both sides, the translation could degenerate into a string-replace exercise that doesn't actually understand the structure. By forcing the marker names to differ, the **only honest way to pair them is structurally**: this SonicWall token sits in the `gateway primary` slot, that WatchGuard token sits in the `<peer-addr>` slot of the `<ike-policy>` that's referenced by the `<ipsec-action>` whose `local-addr` and `remote-addr` selector groups have `.1.tlc` and `.1.trm` suffixes. Once that structural pattern is recognized, the translation rule is mechanical.

---

## 3. What the markers encode

The marker is not a key. The marker is a **role declaration**, attached to a value, in context. It conveys, simultaneously:

1. **Identity** — that this is the field we mean to talk about (uniqueness)
2. **Role** — what conceptual position this field occupies (gateway primary, peer IKE-ID, tunnel local network, NAT-policy name, VLAN interface comment)
3. **Family** — what shape-family this field belongs to in the target grammar (1:1, 1:N, N:1, or part of a 4-shape policy family)
4. **Type-hint** — whether the value can be a name, an IP, a CIDR, an enum, etc.

The marker syntax — concatenating role-describing tokens followed by a discriminator (`Claude`) — is human-readable. A reader of the golden master can look at `FakePriIPSECGwNameORAdrClaude` and decode:

- `Fake` — placeholder value
- `Pri` — primary (as opposed to secondary)
- `IPSEC` — IPSec context
- `Gw` — gateway
- `NameORAdr` — accepts EITHER a hostname OR an IP literal at this slot
- `Claude` — discriminator for marker-search

The `NameORAdr` substring alone tells the decoder a critical fact that no SonicWall documentation states explicitly: **this slot accepts either form, so the translator must handle both.** That single substring saved hours of grammar-doc archaeology and made an entire branch of corner-case handling explicit in the data rather than implicit in the code.

---

## 4. The decoder: `marker_mapping.ini`

Once the markers carry shape information, the decoder is a flat catalog. In FES, this catalog is `skeleton_engine/marker_mapping.ini`, with one section per recognized shape. Excerpted:

```ini
[shape.ipsec_gateway_primary]
sw_detect_pattern   = ^\s+gateway primary (\S+)$
sw_marker_suffixes  = PriIPSECGwNameORAdrClaude
wg_shapes_required  = ike-policy/peer-addr, address-group:NAME
description         = SonicOS IPSec gateway primary → WG ike-policy/peer-addr
                      PLUS backing address-group entry with same name
                      (Firebox restore-time requirement, both IP and name
                      forms; lab-verified)

[shape.vlan_interface]
sw_detect_pattern   = ^interface\s+(\S+)\s+vlan\s+(\d+)\s*$
sw_marker_suffixes  = vlan77forclaude, vlanintfforClaude, vlanconfigforClaude
wg_shapes_required  = interface:NAME
description         = SonicOS `interface X2 vlan 77` → WG <interface> with
                      <vlan-if><vlan-id>77</vlan-id> + member-list back to
                      parent if-num. SW <comment> marker carries through
                      into WG <description> for loose-match pairing.
```

Each section has four fields:

- `sw_detect_pattern` — a regex that finds the shape on the SonicWall side
- `sw_marker_suffixes` — the Claude-marker substrings that confirm the shape is operator-meaningful (not auto-generated noise)
- `wg_shapes_required` — the WatchGuard structural elements that must be present in the output for this shape to be considered correctly translated
- `description` — human-readable provenance

The catalog is the entire translation rule set. There is no per-shape code. The emit-side code reads the catalog, consumes the parsed SonicWall record, and produces the required WG shapes mechanically. **Adding support for a new shape is an INI edit, not a code edit.**

---

## 5. Validation against the same substrate

The Rosetta Stone has a second use that ordinary translators miss: **the markers can be used as the substrate for an empirical validator.**

If the SW input contains marker `vlan77forclaude` and the WG output must contain a corresponding shape, then a validator that fails when the marker is present in the SW input but the required WG shape is absent in the output is **exactly the validator that catches translation regressions**, with no opinion-based assertions needed.

FES implements this as `marker_pairing_validator.py`. The validator:

1. Scans the SW input for every marker substring registered in `marker_mapping.ini`
2. For each marker found, looks up the required WG shape(s) from the catalog
3. Scans the WG output for those shapes
4. Reports a HARD finding for any marker that has no paired WG shape

On the canonical real-world test corpus, FES v1.8.3 achieves **24 of 24 marker pairs found** — perfect coverage of every documented shape in the golden master.

The validator is honest in a way ordinary unit tests are not. It does not assert that the output looks right according to someone's opinion. It asserts that for every input shape the operator explicitly marked as meaningful, a corresponding output shape exists. The criterion is the operator's own annotation, captured in the input, recovered structurally, paired automatically. **Validation reduces to a graph-matching problem on a substrate built from the data.**

### 5.1. The 10-validator gauntlet

`marker_pairing_validator.py` is one of ten validators that gate every FES conversion. The other nine cover orthogonal correctness properties:

- `qualities_validator` — value-type matrix (e.g. an IPv4 leaf doesn't receive a hostname)
- `ref_integrity_validator` — every named reference resolves to a defined target
- `immutables_validator` — hardware-locked structures unchanged
- `customer_fidelity_validator` — customer SW values survive into WG output
- `schema_shape_validator` — every emitted parent/child pair has been observed in real WG configs
- `private_data_validator` — no PII / credentials leak through translation
- `required_children_validator` — every required child element present
- `jpa_diff_validator` — diff against canonical reference within bounds
- `golden_pair_validator` — composite fidelity score against the lab-verified hand-built reference

All ten must return exit code 0 before any output is considered shippable. The composite golden-pair score on the canonical corpus is 0.9122 PASS (value fidelity 0.8333, structural fidelity 0.9910). Three independent lab cycles on a real WatchGuard T-30 (Fireware 12.5.9) confirm that outputs passing the gauntlet load on the hardware and render correctly in the Web UI.

---

## 6. What this method does NOT do

Honest engineering papers state what their methods do not do. The Rosetta Stone approach has real limits:

1. **It requires a hand-built target reference.** Someone has to construct the WatchGuard-side configuration by hand, in the lab, with embedded markers, and verify it loads on real hardware. There is no shortcut. For FES, this took roughly four weeks of empirical lab work before any translator code was written.

2. **It is biased by the markers the author chose to embed.** Shapes the golden master does not exercise are shapes the translator does not learn. FES does not yet translate SonicWall HA pairs, DHCP scopes, or BGP configurations — not because they're impossible, but because we didn't put markers for them in the golden master. Coverage is the author's coverage, no broader.

3. **It does not synthesize new translation rules from data alone.** The decoder (`marker_mapping.ini`) is still authored by hand. The methodology gives the author a substrate against which to write rules, not an automatic rule-induction engine. The labor is moved from "read two vendor manuals and write Python" to "annotate two configs and write an INI" — easier, more verifiable, more maintainable, but still human.

4. **The discriminator token (`Claude` in our case) is operator convention, not a standard.** Anyone applying this method to another translation problem chooses their own discriminator. The choice should be: short, unambiguous in the input grammar, and not occurring naturally in either vendor's syntax.

5. **The validation guarantees structural pairing, not semantic equivalence.** If both sides have a shape and the structural pattern matches, the validator passes. The validator cannot detect, for example, that a translated firewall policy permits more traffic than the source — that requires semantic analysis beyond shape pairing. Lab verification on real hardware remains the final validator for this reason.

---

## 7. Generalization

Although developed for SonicWall → WatchGuard, the methodology generalizes. The construction requires only:

- A **source configuration grammar** that allows operator-meaningful field annotations (comments, labels, names — any field that the operator can name freely)
- A **target configuration grammar** that allows the same
- A **shared discriminator convention** that lets a scanner find the markers cheaply
- A **shape catalog** that captures the structural rules between source and target

The same construction would apply to:

- Fortinet FortiGate → WatchGuard Firebox (or any other firewall pair)
- Cisco ASA → Palo Alto Networks
- VMware NSX → Cisco ACI policy migrations
- AWS Security Group → Azure NSG translations
- On-premise LDAP schema → cloud directory schema mappings
- Legacy ETL configurations → modern data-pipeline DSLs

In every case the same property holds: when the same operational intent is expressed in two grammars with structural markers in both, the translation rules can be derived by inspection rather than guesswork, and the validation can be made empirical rather than opinion-based.

The deeper claim — and this is the connection to *Applied Algebraic Design for Agentic AI* — is that **self-describing data substrates are a reusable engineering primitive for AI-assisted or AI-built systems**. An LLM or a rule-based translator operating on a Rosetta Stone substrate has access to ground truth about the structural roles of fields, without needing out-of-band documentation that may be incomplete, ambiguous, or absent. The methodology brings the documentation **into the data**, where it can be parsed, validated, and acted upon mechanically.

---

## 8. Conclusion

The Golden Rosetta Stone is not a new compiler technique or a new translation algorithm. It is a **discipline about input data**: when you control or can construct the reference configurations used to derive translation rules, **embedding self-describing role-markers into both sides transforms the translation problem from an opinion-based exercise into a structural pattern-matching exercise**.

The methodological breakthrough is not that we built a tool. The breakthrough is that we built a tool *by first building a substrate against which the tool could be built honestly*. The substrate took weeks to construct. The tool that the substrate made possible has 10/10 validators green, composite fidelity 0.9122 PASS, 24/24 marker pairs, and three clean lab cycles on real WatchGuard hardware.

For engineers facing similar cross-vendor or cross-format configuration-translation problems, the recommendation is: do not start by reading vendor documentation and writing code. **Start by constructing the paired reference, with markers, in the lab.** The translator that follows will be smaller, more correct, and more provably honest than one written from documentation alone.

For acquirers or licensees evaluating FES specifically: the **methodology is more defensible than the code**. The code is 36 INI files, a parser, an emitter, and ten validators. Useful, shipping, validator-graded. The methodology — the discipline of constructing Rosetta Stone substrates and deriving translation rules against them — is reapplicable to the next vendor pair, the next configuration format, the next migration crisis. It is the asset.

---

## Acknowledgments

The empirical lab loop that produced and refined the golden master configurations was the entire project. Every marker name in the SonicWall and WatchGuard reference files was placed deliberately, tested by attempting translation, and revised when the resulting rules failed to converge. The WatchGuard T-30 hardware in the n2nhu lab in Newburgh, NY, with Fireware 12.5.9, was the final validator three independent times. The Akira ransomware group, by attacking SonicWall Gen 6 firewalls in 86% of their incidents through 2025, made the timing of this work impossible to argue with.

---

## References

- At-Bay. *2026 InsurSec Report.* Analysis of 6,500+ ransomware claims and 100,000 policy years. https://www.at-bay.com/articles/2026-insursec-report
- SonicWall PSIRT Advisory SNWLID-2024-0015. CVE-2024-40766. Access-control vulnerability in SonicOS management interface. https://psirt.global.sonicwall.com/
- SonicWall PSIRT Advisory SNWLID-2026-0004. CVE-2026-0204 et al. April 29, 2026.
- ReliaQuest GreyMatter Threat Intelligence. *SonicWall Brute-Force Activity Analysis.* May 19, 2026.
- Ames, J. *Applied Algebraic Design for Agentic AI.* n2nhu lab Press, forthcoming. Book #33.
- FES (Firewall Ejector Seat) source repository, v1.8.3. github.com/jimpames/sonicwall-firewall-gen6-to-watchguard-firebox-automatic-converter-by-n2nhu-commercial-MSP-license
- FES self-service portal. https://firewallejectorseat.com

---

*Correspondence: licensing@firewallejectorseat.com*
*n2nhu lab — Newburgh, NY · June 12, 2026*
